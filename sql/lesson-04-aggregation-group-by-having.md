# SQL Lesson 04 — Aggregation: GROUP BY, HAVING & Aggregate Functions

| | |
|---|---|
| **Track** | SQL (PostgreSQL-first) |
| **Lesson** | 04 of 15 |
| **Topic** | Aggregate functions, grouping, filtering groups, and grouping sets |
| **Estimated time** | ~2 hours |
| **Prerequisites** | Lessons 01–03 (SELECT, filtering, joins) |

### Learning objectives
1. State the **logical order of evaluation** of a SELECT and use it to explain why `WHERE` can't see aliases and `HAVING` can.
2. Use `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `string_agg`, `array_agg`, `bool_and/or`, and the statistical aggregates correctly — including their NULL semantics.
3. Explain the difference between `COUNT(*)`, `COUNT(col)`, and `COUNT(DISTINCT col)` and their cost.
4. Distinguish `WHERE` from `HAVING` and know which one to reach for and why it matters for performance.
5. Use `FILTER`, `GROUPING SETS`, `ROLLUP`, `CUBE`, and ordered-set aggregates (`percentile_cont`) to replace piles of subqueries.
6. Read `EXPLAIN ANALYZE` well enough to tell a **HashAggregate** from a **GroupAggregate** and know why the planner chose it.

---

## 1. Why this matters

Aggregation is the point where SQL stops being "fetch rows" and becomes "answer questions." Every dashboard, every report, every metric, every `dbt` model, and every ETL rollup is a `GROUP BY`. It is also where correctness quietly fails: aggregate + `LEFT JOIN` + NULL is a combination that produces *plausible but wrong* numbers, and plausible-but-wrong is the worst outcome in data engineering — nobody files a bug for a number that looks reasonable.

For your target roles this is central:
- **Data engineering:** the entire semantic layer is aggregation. Fact tables get rolled up by dimensions; incremental models aggregate deltas. If you can't reason about grain, you'll build a pipeline that double-counts revenue.
- **Backend:** every "stats" endpoint. Getting `COUNT(*)` vs `COUNT(col)` wrong on a `LEFT JOIN` is one of the most common real bugs in application SQL.
- **Interviews:** "what's the difference between `WHERE` and `HAVING`?" is table-stakes. "Why does `COUNT(*)` return 1 for a customer with no orders?" separates people who've *used* SQL from people who *understand* it. And `GROUP BY` + window functions (Lesson 06) is the single most common technical-screen topic for data roles.

---

## 2. Theory

### 2.1 Set-up: a schema we'll use throughout

```sql
DROP TABLE IF EXISTS order_items, orders, customers, products CASCADE;

CREATE TABLE customers (
    id         serial PRIMARY KEY,
    name       text NOT NULL,
    country    text NOT NULL,
    signup_date date NOT NULL
);

CREATE TABLE products (
    id       serial PRIMARY KEY,
    name     text NOT NULL,
    category text NOT NULL,
    price    numeric(10,2) NOT NULL
);

CREATE TABLE orders (
    id          serial PRIMARY KEY,
    customer_id int NOT NULL REFERENCES customers(id),
    status      text NOT NULL,          -- 'paid' | 'pending' | 'refunded'
    created_at  timestamptz NOT NULL
);

CREATE TABLE order_items (
    id         serial PRIMARY KEY,
    order_id   int NOT NULL REFERENCES orders(id),
    product_id int NOT NULL REFERENCES products(id),
    quantity   int NOT NULL,
    unit_price numeric(10,2) NOT NULL   -- price AT TIME OF SALE, not products.price
);

INSERT INTO customers (name, country, signup_date) VALUES
    ('Ann',  'DE', '2025-01-15'),
    ('Bob',  'DE', '2025-03-02'),
    ('Cleo', 'FR', '2025-06-20'),
    ('Dan',  'US', '2026-01-05');       -- Dan has NO orders

INSERT INTO products (name, category, price) VALUES
    ('Widget',  'hardware', 9.99),
    ('Gadget',  'hardware', 24.50),
    ('License', 'software', 99.00);

INSERT INTO orders (customer_id, status, created_at) VALUES
    (1, 'paid',     '2026-01-10 10:00+00'),
    (1, 'paid',     '2026-02-14 11:30+00'),
    (1, 'refunded', '2026-03-01 09:00+00'),
    (2, 'paid',     '2026-02-20 15:00+00'),
    (2, 'pending',  '2026-07-01 08:00+00'),
    (3, 'paid',     '2026-07-05 12:00+00');

INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 1, 2, 9.99), (1, 3, 1, 99.00),
    (2, 2, 1, 24.50),
    (3, 1, 5, 9.99),
    (4, 3, 2, 99.00),
    (5, 1, 1, 9.99),
    (6, 2, 3, 24.50);
```

Note `order_items.unit_price` duplicating `products.price`. That is **not** a normalisation error — it's deliberate: prices change, and an invoice must reflect what was actually charged. Recomputing historical revenue from `products.price` is a real, expensive, recurring bug. (Lesson 08 covers when denormalisation is correct; this is the canonical case.)

### 2.2 The logical order of evaluation — the mental model that explains everything

You *write* SQL in one order; the database *evaluates* it in another:

```
1. FROM        + JOINs      → build the working set of rows
2. WHERE                    → filter individual ROWS
3. GROUP BY                 → collapse rows into groups
4. HAVING                   → filter GROUPS
5. SELECT      (+ aggregates, expressions, aliases created HERE)
6. DISTINCT
7. ORDER BY                 → aliases from step 5 are visible
8. LIMIT / OFFSET
```

Memorise this. It answers nearly every "why doesn't this work?" question:

- **Why can't `WHERE` use a SELECT alias?** `WHERE` (2) runs before `SELECT` (5). The alias doesn't exist yet.
- **Why *can* `ORDER BY` use an alias?** `ORDER BY` (7) runs after `SELECT` (5).
- **Why can't `WHERE` contain an aggregate?** Aggregates are computed at (3)/(5); `WHERE` at (2) sees individual rows, and "the sum" doesn't exist yet.
- **Why must every non-aggregated `SELECT` column be in `GROUP BY`?** After (3) the group is *one row*; a column not in the grouping key has many possible values and the database refuses to pick arbitrarily.

```sql
-- FAILS: alias not available in WHERE
SELECT price * 1.19 AS gross FROM products WHERE gross > 50;
-- ERROR:  column "gross" does not exist

-- WORKS: repeat the expression, or use a subquery/CTE
SELECT price * 1.19 AS gross FROM products WHERE price * 1.19 > 50;
SELECT * FROM (SELECT price * 1.19 AS gross FROM products) s WHERE gross > 50;

-- WORKS: ORDER BY sees the alias (and even its ordinal position)
SELECT price * 1.19 AS gross FROM products ORDER BY gross DESC;
SELECT price * 1.19 AS gross FROM products ORDER BY 1 DESC;   -- legal, but avoid
```

**This is a *logical* order, not a physical one.** The planner reorders freely as long as results are identical — it will push a `WHERE` predicate down into an index scan, or into a join's inner side. The logical order tells you what the query *means*; `EXPLAIN` tells you what it *does*.

### 2.3 Aggregate functions and the NULL rule

**The single most important rule: aggregates ignore NULLs — except `COUNT(*)`.**

| Function | NULL behaviour | Notes |
|---|---|---|
| `COUNT(*)` | counts **rows**, NULLs included | never NULL; returns 0 for no rows |
| `COUNT(col)` | counts **non-NULL values** of `col` | the difference is the whole game |
| `COUNT(DISTINCT col)` | distinct non-NULL values | expensive — sorts or hashes |
| `SUM(col)` | ignores NULLs; **returns NULL for zero rows** | not 0! Use `COALESCE(SUM(x), 0)` |
| `AVG(col)` | ignores NULLs — **denominator excludes them** | `AVG` ≠ `SUM/COUNT(*)` when NULLs exist |
| `MIN`/`MAX` | ignore NULLs | can use an index for a single-column min/max |
| `bool_and`/`bool_or` | ignore NULLs | "all/any" over a group |
| `string_agg(col, ', ')` | ignores NULLs | takes an `ORDER BY` inside |
| `array_agg(col)` | **keeps NULLs** | the exception to the rule |
| `jsonb_agg`/`jsonb_object_agg` | keeps NULLs | for nested API responses |

```sql
CREATE TEMP TABLE t (v int);
INSERT INTO t VALUES (10), (20), (NULL);

SELECT COUNT(*)     AS star,     -- 3   rows
       COUNT(v)     AS non_null, -- 2   non-NULL values
       SUM(v)       AS total,    -- 30
       AVG(v)       AS avg_v,    -- 15.0  <- 30/2, NOT 30/3
       SUM(v)::numeric / COUNT(*) AS avg_wrong  -- 10.0  <- treats NULL as 0
FROM t;
```
```
 star | non_null | total | avg_v | avg_wrong
------+----------+-------+-------+-----------
    3 |        2 |    30 |  15.0 |      10.0
```

`AVG` = 15 and "sum over row count" = 10 are both defensible answers to different business questions. **Which one you want is a requirements question, not a SQL question** — and choosing accidentally is how reports go wrong. Say that out loud in an interview.

The empty-set trap:
```sql
SELECT SUM(v) FROM t WHERE v > 1000;      -- NULL, not 0
SELECT COUNT(*) FROM t WHERE v > 1000;    -- 0
SELECT COALESCE(SUM(v), 0) FROM t WHERE v > 1000;   -- 0  <- what you meant
```
`SUM` over zero rows is NULL. That NULL then propagates: `SUM(v) * 1.19` is NULL, `SUM(v) > 0` is NULL (not false), and an application expecting a number gets `None` and crashes. Wrap aggregates that feed arithmetic in `COALESCE`.

### 2.4 `GROUP BY`

```sql
SELECT country, COUNT(*) AS customers, MIN(signup_date) AS first_signup
FROM customers
GROUP BY country
ORDER BY customers DESC;
```
```
 country | customers | first_signup
---------+-----------+--------------
 DE      |         2 | 2025-01-15
 FR      |         1 | 2025-06-20
 US      |         1 | 2026-01-05
```

**The grain rule.** After `GROUP BY country`, one row *is* one country. Every selected column must either be in the `GROUP BY` or be inside an aggregate — otherwise the database can't know which value you want.

```sql
SELECT country, name FROM customers GROUP BY country;
-- ERROR:  column "customers.name" must appear in the GROUP BY clause
--         or be used in an aggregate function
```

MySQL historically **allowed** this and returned an arbitrary row's value (`ONLY_FULL_GROUP_BY` off). That's a footgun, not a feature; Postgres is right to refuse. A good interview point: "MySQL lets you, and it silently gives you a random value — I'd never rely on it."

**Postgres's functional-dependency exception:** if you group by a table's **primary key**, you may select any of that table's other columns, because the PK functionally determines them:
```sql
SELECT c.id, c.name, c.country, COUNT(o.id) AS orders
FROM customers c LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id                          -- id is the PK, so name/country are implied
ORDER BY orders DESC;
```
This is standard SQL (SQL:1999) and works in Postgres but not everywhere. It's cleaner than `GROUP BY c.id, c.name, c.country`, and it *documents* that the grain is one customer.

**`GROUP BY` puts all NULLs in one group** — unlike `=`, which never matches NULL. `GROUP BY` (and `DISTINCT`, and `UNION`) use "not distinct from" semantics, where NULL groups with NULL. This inconsistency is worth knowing cold; it's the kind of detail that comes up in senior screens.

### 2.5 `WHERE` vs `HAVING`

- **`WHERE` filters rows, before grouping.** Can use indexes. Cannot see aggregates.
- **`HAVING` filters groups, after aggregation.** Cannot use indexes on the grouped result. Can see aggregates.

```sql
-- Customers with more than 1 PAID order.
SELECT c.name, COUNT(*) AS paid_orders
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'paid'          -- ROW filter: drops non-paid rows BEFORE grouping
GROUP BY c.id, c.name
HAVING COUNT(*) > 1              -- GROUP filter: needs the aggregate
ORDER BY paid_orders DESC;
```
```
 name | paid_orders
------+-------------
 Ann  |           2
```

`HAVING o.status = 'paid'` would *also* work here (Postgres allows non-aggregate conditions in `HAVING`) but it's **wrong on principle and slower in practice**: it makes the database group every row and only then discard, instead of never reading the non-paid rows at all. Worse, it changes results when combined with aggregates — `HAVING` filters after `COUNT(*)` has already counted the rows you meant to exclude.

**The rule: if a condition doesn't mention an aggregate, it belongs in `WHERE`.**

`HAVING` without `GROUP BY` is legal — it treats the whole table as one group:
```sql
SELECT SUM(quantity) FROM order_items HAVING SUM(quantity) > 100;   -- 0 rows (total is 15)
```

### 2.6 `FILTER` — conditional aggregation done properly

Postgres implements the SQL standard `FILTER` clause, which is strictly better than the `CASE` trick everyone learns first:

```sql
SELECT
    c.country,
    COUNT(*)                                    AS all_orders,
    COUNT(*) FILTER (WHERE o.status = 'paid')     AS paid,
    COUNT(*) FILTER (WHERE o.status = 'refunded') AS refunded,
    -- the old, portable way:
    SUM(CASE WHEN o.status = 'pending' THEN 1 ELSE 0 END) AS pending_old,
    ROUND(100.0 * COUNT(*) FILTER (WHERE o.status = 'refunded') / COUNT(*), 1)
                                                 AS refund_pct
FROM customers c
JOIN orders o ON o.customer_id = c.id
GROUP BY c.country
ORDER BY c.country;
```
```
 country | all_orders | paid | refunded | pending_old | refund_pct
---------+------------+------+----------+-------------+------------
 DE      |          5 |    3 |        1 |           1 |       20.0
 FR      |          1 |    1 |        0 |           0 |        0.0
```

`FILTER` is clearer than `CASE`, works with **any** aggregate (`SUM(x) FILTER (WHERE ...)`, `array_agg(...) FILTER (...)`), and avoids the classic `COUNT(CASE WHEN ... THEN 1 ELSE 0 END)` bug — `COUNT` counts the `0`s too, because 0 is not NULL. (With `COUNT` you must use `CASE WHEN ... THEN 1 END`, leaving the implicit `ELSE NULL`. With `SUM` you want `ELSE 0`. Mixing these up is a top-5 real SQL bug.) **This one pattern — pivoting a status column into columns in a single pass — is the workhorse of every dashboard query.**

### 2.7 Aggregates with `ORDER BY` and `DISTINCT` inside

```sql
SELECT
    o.customer_id,
    string_agg(p.name, ', ' ORDER BY p.name)     AS products,
    array_agg(DISTINCT p.category ORDER BY p.category) AS categories,
    COUNT(DISTINCT p.category)                   AS n_categories
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
GROUP BY o.customer_id
ORDER BY o.customer_id;
```
```
 customer_id |          products           |     categories      | n_categories
-------------+----------------------------+---------------------+--------------
           1 | License, Widget, Widget    | {hardware,software} |            2
           2 | Gadget, License            | {hardware,software} |            2
           3 | Gadget                     | {hardware}          |            1
```

`string_agg`/`array_agg` without an internal `ORDER BY` produce **non-deterministic order** — the result depends on the plan, which can change when statistics change. Your test passes locally and the output reorders in production. **Always order inside the aggregate** if the order is meaningful.

### 2.8 `GROUPING SETS`, `ROLLUP`, `CUBE` — many grains in one pass

The naive way to get subtotals is `UNION ALL` of three queries, each scanning the table. `GROUPING SETS` does it in **one scan**:

```sql
SELECT
    c.country,
    o.status,
    COUNT(*) AS orders,
    GROUPING(c.country) AS g_country,   -- 1 = this column is aggregated away
    GROUPING(o.status)  AS g_status
FROM customers c JOIN orders o ON o.customer_id = c.id
GROUP BY GROUPING SETS ((c.country, o.status), (c.country), ())
ORDER BY c.country NULLS LAST, o.status NULLS LAST;
```
```
 country |  status  | orders | g_country | g_status
---------+----------+--------+-----------+----------
 DE      | paid     |      3 |         0 |        0
 DE      | pending  |      1 |         0 |        0
 DE      | refunded |      1 |         0 |        0
 DE      |          |      5 |         0 |        1   <- country subtotal
 FR      | paid     |      1 |         0 |        0
 FR      |          |      1 |         0 |        1   <- country subtotal
         |          |      6 |         1 |        1   <- grand total
```

- `ROLLUP(a, b)` ≡ `GROUPING SETS ((a,b), (a), ())` — hierarchical subtotals (year → month → day).
- `CUBE(a, b)` ≡ `GROUPING SETS ((a,b), (a), (b), ())` — every combination. Note `CUBE` of *n* columns produces 2ⁿ grouping sets; `CUBE(a,b,c,d,e)` is 32 sets, which is how people accidentally DoS their warehouse.
- **`GROUPING(col)`** returns 1 when the column was rolled up. You need it because a subtotal row has `NULL` in that column — and so might a real row with a genuine NULL. Without `GROUPING()` you cannot distinguish "total across all statuses" from "the group where status IS NULL." This is exactly the kind of subtlety that makes a good interview answer.

### 2.9 Ordered-set aggregates — percentiles

Averages lie. p95 latency is the number that matters:

```sql
SELECT
    p.category,
    COUNT(*)                                                     AS n,
    ROUND(AVG(oi.unit_price), 2)                                 AS mean,
    percentile_cont(0.5)  WITHIN GROUP (ORDER BY oi.unit_price)  AS median,
    percentile_cont(0.95) WITHIN GROUP (ORDER BY oi.unit_price)  AS p95,
    percentile_disc(0.5)  WITHIN GROUP (ORDER BY oi.unit_price)  AS median_actual,
    mode() WITHIN GROUP (ORDER BY oi.unit_price)                 AS most_common
FROM order_items oi JOIN products p ON p.id = oi.product_id
GROUP BY p.category;
```
```
 category |  n |  mean  | median | p95  | median_actual | most_common
----------+----+--------+--------+------+---------------+-------------
 hardware |  5 |  15.79 |   9.99 | 24.5 |          9.99 |        9.99
 software |  2 |  99.00 |  99.00 | 99.0 |         99.00 |       99.00
```

`percentile_cont` **interpolates** (may return a value not in the data — right for continuous measures like latency); `percentile_disc` returns an **actual** data value (right for discrete things, and for "show me the median order"). Knowing that distinction cold is a nice senior detail.

---

## 3. Official documentation quotes

> "Aggregate functions compute a single result from a set of input values. … It should be noted that except for `count`, these functions return a null value when no rows are selected. In particular, `sum` of no rows returns null, not zero as one might expect, and `array_agg` returns null rather than an empty array when there are no input rows. The `coalesce` function can be used to substitute zero or an empty array for null when necessary."
> — *PostgreSQL Documentation*, [9.21. Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html)

> "`GROUP BY` will condense into a single row all selected rows that share the same values for the grouped expressions. … When `GROUP BY` is present, or any aggregate functions are present, it is not valid for the `SELECT` list expressions to refer to ungrouped columns except within aggregate functions or when the ungrouped column is functionally dependent on the grouped columns, since there would otherwise be more than one possible value to return for an ungrouped column."
> — *PostgreSQL Documentation*, [7.2.3. The GROUP BY and HAVING Clauses](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUP)

> "`HAVING` differs from `WHERE` in that: `WHERE` filters individual rows before the application of `GROUP BY`, while `HAVING` filters group rows created by `GROUP BY`. Each column referenced in condition must unambiguously reference a grouping column, unless the reference appears within an aggregate function."
> — *PostgreSQL Documentation*, [7.2.3. The GROUP BY and HAVING Clauses](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUP)

> "If `FILTER` is specified, then only the input rows for which the `filter_clause` evaluates to true are fed to the aggregate function; other rows are discarded."
> — *PostgreSQL Documentation*, [4.2.7. Aggregate Expressions](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-AGGREGATES)

> "`GROUPING SETS`, `CUBE`, and `ROLLUP` … More complex grouping operations than those described above are possible using the concept of grouping sets. The data selected by the `FROM` and `WHERE` clauses is grouped separately by each specified grouping set, aggregates computed for each group just as for simple `GROUP BY` clauses, and then the results returned."
> — *PostgreSQL Documentation*, [7.2.4. GROUPING SETS, CUBE, and ROLLUP](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPING-SETS)

---

## 4. Real-world examples

### 4.1 The `LEFT JOIN` + `COUNT(*)` bug — the most important query in this lesson

You want every customer and their order count, including customers with none.

```sql
-- WRONG
SELECT c.name, COUNT(*) AS orders
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.name
ORDER BY c.name;
```
```
 name | orders
------+--------
 Ann  |      3
 Bob  |      2
 Cleo |      1
 Dan  |      1     <-- WRONG. Dan has ZERO orders.
```

The `LEFT JOIN` emits one row for Dan with **all order columns NULL**. `COUNT(*)` counts *rows*, and that's still a row. So Dan gets 1.

```sql
-- RIGHT
SELECT c.name, COUNT(o.id) AS orders
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.name
ORDER BY c.name;
```
```
 name | orders
------+--------
 Ann  |      3
 Bob  |      2
 Cleo |      1
 Dan  |      0     <-- correct
```

`COUNT(o.id)` counts non-NULL values of `o.id`; Dan's phantom row has `o.id IS NULL`, so it counts 0. **This is the #1 real-world aggregation bug and the #1 aggregation interview question.** The rule: **with a `LEFT JOIN`, count a NOT NULL column from the right table, never `*`.**

The same bug in its `SUM` costume:
```sql
SELECT c.name, COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS revenue
FROM customers c
LEFT JOIN orders o      ON o.customer_id = c.id AND o.status = 'paid'   -- NOTE: in the ON
LEFT JOIN order_items oi ON oi.order_id = o.id
GROUP BY c.id, c.name
ORDER BY revenue DESC;
```
```
 name | revenue
------+----------
 Ann  |  217.98
 Bob  |  198.00
 Cleo |   73.50
 Dan  |    0.00
```

Two critical details. **`o.status = 'paid'` is in the `ON`, not the `WHERE`** — putting it in `WHERE` would filter out Dan's all-NULL row and silently turn the `LEFT JOIN` back into an inner join, dropping Dan entirely. And `COALESCE(..., 0)` because `SUM` of no rows is NULL. Both mistakes produce output that looks fine.

### 4.2 Fan-out: the double-counting bug

```sql
-- WRONG: joining to order_items multiplies the order rows
SELECT c.name,
       COUNT(DISTINCT o.id) AS orders,
       COUNT(o.id)          AS orders_wrong,
       SUM(oi.quantity)     AS items
FROM customers c
JOIN orders o       ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
GROUP BY c.id, c.name;
```
```
 name | orders | orders_wrong | items
------+--------+--------------+-------
 Ann  |      3 |            4 |     8
 Bob  |      2 |            2 |     3
 Cleo |      1 |            1 |     3
```

Ann's order #1 has two line items, so the join produces two rows for it — `COUNT(o.id)` sees 4. This is **fan-out**, and it's what makes multi-table aggregation dangerous: the moment you join a one-to-many, your grain changes and every aggregate over the *one* side is inflated.

`COUNT(DISTINCT o.id)` patches it, but the honest fix is to **aggregate at the right grain first**:

```sql
SELECT c.name, s.orders, s.items, s.revenue
FROM customers c
JOIN LATERAL (
    SELECT COUNT(DISTINCT o.id)                     AS orders,
           SUM(oi.quantity)                         AS items,
           SUM(oi.quantity * oi.unit_price)         AS revenue
    FROM orders o JOIN order_items oi ON oi.order_id = o.id
    WHERE o.customer_id = c.id
) s ON true;
```

Or with a pre-aggregated CTE (Lesson 05). The general principle — **never aggregate across two different one-to-many branches in one query** — is the single most valuable thing in this section. `SUM(a.x)` and `SUM(b.y)` in the same query, where `a` and `b` both fan out from the same parent, gives you a cartesian inflation of both. Every data engineer has shipped this bug once.

### 4.3 A real dashboard query, one pass

```sql
SELECT
    date_trunc('month', o.created_at)::date                        AS month,
    COUNT(DISTINCT o.id)                                           AS orders,
    COUNT(DISTINCT o.customer_id)                                  AS buyers,
    COUNT(DISTINCT o.id) FILTER (WHERE o.status = 'refunded')      AS refunds,
    ROUND(SUM(oi.quantity * oi.unit_price) FILTER (WHERE o.status = 'paid'), 2) AS revenue,
    ROUND(AVG(oi.quantity * oi.unit_price) FILTER (WHERE o.status = 'paid'), 2) AS avg_line,
    ROUND(100.0 * COUNT(DISTINCT o.id) FILTER (WHERE o.status = 'refunded')
                / NULLIF(COUNT(DISTINCT o.id), 0), 1)              AS refund_rate_pct
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
WHERE o.created_at >= '2026-01-01'
GROUP BY 1
ORDER BY 1;
```
```
   month    | orders | buyers | refunds | revenue | avg_line | refund_rate_pct
------------+--------+--------+---------+---------+----------+-----------------
 2026-01-01 |      1 |      1 |       0 |  118.98 |    59.49 |             0.0
 2026-02-01 |      2 |      2 |       0 |  222.50 |   111.25 |             0.0
 2026-03-01 |      1 |      1 |       1 |         |          |           100.0
 2026-07-01 |      2 |      2 |       0 |   73.50 |    73.50 |             0.0
```

Note `NULLIF(x, 0)` guarding the division — **division by zero raises in Postgres**, and `NULLIF` turns the denominator into NULL, making the whole expression NULL instead of aborting the query. A dashboard that 500s because one month had no orders is a real outage. Note also March's NULL revenue: no paid orders means the `FILTER`ed `SUM` sees zero rows. Whether you want `NULL` or `0.00` there is a *product* decision — NULL says "no data," 0 says "we made nothing." Wrap in `COALESCE` if the latter.

### 4.4 `HashAggregate` vs `GroupAggregate` in `EXPLAIN`

```sql
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF)
SELECT country, COUNT(*) FROM customers GROUP BY country;
```
```
 HashAggregate (actual time=0.021..0.023 rows=3 loops=1)
   Group Key: country
   Batches: 1  Memory Usage: 24kB
   ->  Seq Scan on customers (actual time=0.008..0.010 rows=4 loops=1)
 Planning Time: 0.089 ms
 Execution Time: 0.045 ms
```

Two strategies exist:
- **HashAggregate** — build a hash table keyed by the grouping columns. No sort needed, O(n), but the hash table must fit in `work_mem`.
- **GroupAggregate** — requires **sorted** input, then aggregates each run as it goes. Constant memory, but pays for the sort (or needs an index providing the order).

The planner chooses on estimated distinct groups. Watch for `Batches: 5  Memory Usage: 4096kB  Disk Usage: 12000kB` — that means the hash table **spilled to disk** (Postgres 13+ can do this; before 13 it would just blow past `work_mem`). Spilling is often a 10× slowdown, and the fix is usually to raise `work_mem` for that session/query, reduce the group cardinality, or pre-aggregate. `HashAggregate` with a bad row estimate is one of the most common causes of "the query was fast yesterday."

---

## 5. How This Is Used In Production

- **Startups:** aggregation *is* the analytics stack for the first year — a `GROUP BY` behind a `/stats` endpoint and a Metabase dashboard on the replica. The failure mode is running these on the primary, where a 30-second `GROUP BY` over the full orders table holds resources and competes with transactional traffic; the first real fix is a read replica, the second is a nightly rollup table.
- **Enterprises:** aggregation is layered. Raw events → staging → **fact tables** → aggregate/rollup tables → BI. Tools like **dbt** materialise each layer with tests on grain (`unique` on the grouping key is literally a grain assertion). Warehouses (Snowflake, BigQuery, Redshift) are columnar precisely because analytical aggregation touches few columns over many rows — a column store reads only `status` and `amount`, not the whole row, which is a 10–100× I/O reduction. This is *the* architectural reason OLAP and OLTP diverged.
- **Common architectures:** **pre-aggregation** is the standard scaling answer — materialised views (Lesson 11) or rollup tables refreshed incrementally, so the dashboard reads 365 rows instead of 500 million. **Approximate aggregation** for high-cardinality distincts: `COUNT(DISTINCT user_id)` over a billion rows is brutal, so you use HyperLogLog (`postgresql-hll`, BigQuery's `APPROX_COUNT_DISTINCT`, Redis `PFCOUNT`) and accept ~2% error for ~1/1000th the memory. **Streaming aggregation** (Kafka Streams, Flink) computes windowed rollups continuously so the warehouse never has to — that's Lesson 15's territory, and the mental link is worth making: a Flink tumbling window *is* a `GROUP BY date_trunc('minute', ts)` that never ends.
- **Scaling:** the progression is: index the `WHERE` columns → pre-aggregate into rollup tables → partition by time so old data is never scanned → approximate the expensive distincts → move to a columnar store. Each buys roughly an order of magnitude. `COUNT(*)` over a huge table is deceptively expensive in Postgres because MVCC means it must check visibility for every row — there's no O(1) row count. Teams cache it, use `reltuples` from `pg_class` for an estimate, or maintain a counter table with triggers.
- **Monitoring:** every SLO is an aggregation — `COUNT(*) FILTER (WHERE status >= 500) / COUNT(*)` over a window. Prometheus's `rate()`/`histogram_quantile()` are the time-series dialect of this lesson. Percentiles matter more than means: an average latency of 200 ms with a p99 of 8 s means 1% of users are having an awful time, and `AVG` hides it completely. Always chart p50/p95/p99, never the mean alone.
- **Security:** aggregation is the classic **inference-attack** surface. "Average salary grouped by department" is safe with 200 people per department and a data leak with 1. Real systems apply *k*-anonymity (suppress groups with `HAVING COUNT(*) >= 5`) and, in strict settings, differential privacy (adding calibrated noise). Apple, Google, and the US Census Bureau all do this. Also: a `GROUP BY` on user-supplied column names is a SQL-injection vector that parameterisation does **not** protect against — you cannot bind an identifier, only a value, so you must allowlist.
- **CI/CD:** dbt tests assert grain (`unique`, `not_null` on the group key) and reconciliation (aggregate the fact and the rollup, assert equality within tolerance). Every serious pipeline has a "does the sum of the parts equal the whole" test, because fan-out bugs are invisible otherwise.
- **Common tools:** `dbt` (SQL transformation), `Metabase`/`Superset`/`Looker` (BI, which generate the `GROUP BY` for you — and generate it badly if the model's grain is wrong), `pg_stat_statements` (find your expensive aggregations), `postgresql-hll`, `TimescaleDB` continuous aggregates (incrementally-maintained rollups for time-series — genuinely excellent).
- **Real examples:** **Netflix** and **Uber** run enormous aggregation layers (Uber's is public: Hive/Presto/Pinot serving pre-aggregated slices, because a live `GROUP BY` over trip history at their scale is impossible). **Cloudflare** publicly described moving analytics to ClickHouse with materialised rollups for exactly this reason. **Amazon** and **Google** both push the "pre-aggregate on write, not on read" pattern for anything user-facing — the general principle being that read-heavy analytics should pay its cost once, at ingestion, not on every dashboard load.

---

## 6. Advanced corner

**How Postgres actually aggregates.** Every aggregate is a state machine: an initial state, a **transition function** called once per row (`sfunc`), and an optional **final function** (`ffunc`). `AVG(int)` keeps state `(count, sum)` and the final function divides. You can see them:
```sql
SELECT aggfnoid::regproc, aggtransfn, aggfinalfn, aggcombinefn
FROM pg_aggregate WHERE aggfnoid = 'avg(int4)'::regprocedure;
```
`aggcombinefn` merges two partial states — that's what makes **parallel aggregation** possible: each worker aggregates a slice, then the leader combines. An aggregate without a combine function cannot be parallelised, which is why some custom aggregates silently force a serial plan. You can define your own aggregate with `CREATE AGGREGATE`, and this is how extensions like `hll` and `tdigest` plug in natively.

**Why `COUNT(*)` is O(n) in Postgres.** MVCC means each row version has visibility info; "how many rows are there" depends on *your* snapshot, so there's no maintainable global counter. An index-only scan helps (smaller pages to read, if the visibility map says the pages are all-visible) but it's still linear. Alternatives: `SELECT reltuples::bigint FROM pg_class WHERE relname='orders'` (an estimate, updated by `ANALYZE`/autovacuum — good enough for pagination hints), or a trigger-maintained counter table (exact, but a write-contention hotspot). MyISAM had O(1) counts because it had no MVCC; that's the trade.

**`DISTINCT` inside aggregates is expensive and blocks parallelism.** `COUNT(DISTINCT x)` forces Postgres to collect and sort/hash *all* the values per group — it cannot be computed incrementally, and (as of PG 17) it **disables parallel aggregation** for that query. Multiple `COUNT(DISTINCT ...)` on different columns are pathological. Alternatives: pre-aggregate to unique rows in a CTE and count normally; use `GROUP BY x` in a subquery then `COUNT(*)` outside; or use HLL for approximate answers. This is a genuinely useful thing to know — it's the reason a dashboard query with five `COUNT(DISTINCT)`s takes 40 seconds.

**`work_mem` and hash-agg spills.** `work_mem` is **per operation, per node, per worker** — not per query and not per connection. A query with 3 hash aggregates and 4 parallel workers can use ~12× `work_mem`. Set it globally low (4–16 MB) and raise it per-session for known-heavy analytical queries (`SET LOCAL work_mem = '256MB'` inside a transaction). Setting `work_mem = 1GB` globally on a 200-connection server is a classic self-inflicted OOM, and it's a great "what would you check?" interview answer.

**The `HAVING` optimisation fence isn't one.** The planner *can* push a `HAVING` condition into `WHERE` if it doesn't reference an aggregate — so `HAVING status='paid'` often costs nothing. But relying on that is fragile: it's a specific optimisation, not a guarantee, and the semantics change the instant an aggregate enters the predicate. Write it correctly; don't lean on the planner's charity.

**Ordered-set vs hypothetical-set aggregates.** `percentile_cont`/`percentile_disc`/`mode` are *ordered-set* aggregates (`WITHIN GROUP (ORDER BY ...)`). There's a second family — *hypothetical-set* aggregates (`rank`, `dense_rank`, `percent_rank`, `cume_dist` used with `WITHIN GROUP`) — that answer "where would this value rank if I inserted it?" without inserting it:
```sql
SELECT rank(50.0) WITHIN GROUP (ORDER BY unit_price) FROM order_items;  -- 6
```
Rarely used, genuinely useful for "what percentile would this new value land in," and a nice thing to know exists.

**`GROUPING SETS` on high cardinality.** `CUBE(a,b,c,d)` is 16 grouping sets; each needs its own hash table, all sharing `work_mem`. Postgres can sometimes share work between sets that are prefixes of each other (a `ROLLUP` needs one sort, not *n*), which is why `ROLLUP` is much cheaper than `CUBE` for the same column count. Prefer `ROLLUP` when your grouping is genuinely hierarchical (it usually is — year/month/day, country/region/city).

**What most people get wrong at this level:**
- `COUNT(*)` with a `LEFT JOIN` (§4.1). Endemic.
- Fan-out double-counting when joining two one-to-many tables (§4.2). Endemic and *silent*.
- Believing `SUM` of nothing is 0.
- Putting the outer-join condition in `WHERE` instead of `ON`, silently converting a LEFT JOIN to an INNER JOIN.
- `COUNT(CASE WHEN x THEN 1 ELSE 0 END)` — counts everything, because `0` isn't NULL.
- Thinking `HAVING` and `WHERE` are interchangeable because both "filter."
- Not knowing `AVG` excludes NULLs from the denominator, and shipping a metric that quietly means something else.
- Charting the mean when the business cares about the tail.

---

## 7. Best practices

- **Filter rows in `WHERE`, groups in `HAVING`.** If the predicate has no aggregate, it belongs in `WHERE` — always, on principle and for speed.
- **With `LEFT JOIN`, never `COUNT(*)`.** Count a NOT NULL column from the right side. Put the right table's filters in `ON`, not `WHERE`.
- **`COALESCE` any aggregate feeding arithmetic or an API response.** `SUM` of no rows is NULL, and NULL propagates through everything downstream.
- **Guard every division with `NULLIF(denominator, 0)`.** Division by zero aborts the query.
- **Use `FILTER` over `CASE`** — clearer, works with any aggregate, and dodges the `COUNT(... ELSE 0)` bug.
- **Always `ORDER BY` inside `string_agg`/`array_agg`** when order matters. Without it the order is a plan detail that will change on you.
- **`GROUP BY` the primary key** and let functional dependency carry the rest — it's less noise and it documents the grain.
- **Know your grain, and state it in a comment.** "One row per customer per month" at the top of the query prevents the fan-out bug better than any technique.
- **Never aggregate across two one-to-many branches in one query.** Pre-aggregate each branch in a CTE/lateral, then join the results.
- **Report percentiles, not just means**, for anything latency- or money-shaped.
- Round at the *presentation* layer, and use `numeric` (not `float`) for money — `float` sums are non-associative and drift.
- Add `HAVING COUNT(*) >= k` when aggregating over people. It's a correctness guard against tiny-denominator noise and a privacy guard at the same time.

## 8. Common mistakes & gotchas

**`COUNT(*)` with `LEFT JOIN` returns 1 for empty** (§4.1). The big one.

**Fan-out inflation** (§4.2) — two one-to-many joins multiply each other.

**`SUM` of no rows is NULL**
```sql
SELECT SUM(quantity) FROM order_items WHERE order_id = 999;   -- NULL
SELECT COALESCE(SUM(quantity), 0) FROM order_items WHERE order_id = 999;   -- 0
```

**`WHERE` on the outer table of a `LEFT JOIN`**
```sql
-- Silently an INNER JOIN — Dan disappears:
FROM customers c LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'paid'
-- Correct:
FROM customers c LEFT JOIN orders o ON o.customer_id = c.id AND o.status = 'paid'
```

**`COUNT` with `ELSE 0`**
```sql
COUNT(CASE WHEN status='paid' THEN 1 ELSE 0 END)   -- counts ALL rows: 0 is not NULL
COUNT(CASE WHEN status='paid' THEN 1 END)          -- correct (implicit ELSE NULL)
COUNT(*) FILTER (WHERE status='paid')              -- better
```

**`AVG` denominator excludes NULLs** — `AVG(v)` ≠ `SUM(v)/COUNT(*)` when `v` has NULLs.

**Division by zero aborts**
```sql
SELECT SUM(a)/COUNT(*) FILTER (WHERE b) FROM t;  -- ERROR: division by zero
SELECT SUM(a)/NULLIF(COUNT(*) FILTER (WHERE b), 0) FROM t;   -- NULL instead
```

**Integer division truncates** — `SELECT 1/2` is `0`. Use `1.0/2` or `::numeric`. `100 * refunds / orders` is `0` when refunds < orders.

**Non-deterministic `array_agg` order** without an internal `ORDER BY`.

**`GROUP BY` groups all NULLs together** — even though `NULL = NULL` is NULL. `DISTINCT` and `UNION` behave the same way. Inconsistent with `WHERE`, and worth knowing.

**Ordinal `GROUP BY 1, 2`** is legal and concise but breaks silently when someone reorders the `SELECT` list. Fine in exploratory SQL, dangerous in checked-in code.

**Aggregating `float` money** — `SUM` of floats isn't associative, so parallel workers can produce a different last digit than a serial plan. Use `numeric`.

## 9. Where AI helps (and where it hurts)

- **Helps:** translating a business question into a `GROUP BY` skeleton; remembering `GROUPING SETS`/`percentile_cont`/`FILTER` syntax you use twice a year; explaining an unfamiliar `EXPLAIN` plan; generating the test data for a homework exercise; converting a `CASE`-pivot to `FILTER`.
- **Hurts:** AI produces the `LEFT JOIN` + `COUNT(*)` bug **constantly** — it pattern-matches on shape, and `COUNT(*)` is the most common token sequence. It does **not** reason about grain, so it will happily join `orders` and `order_items` and `SUM` across both, giving you inflated revenue that no one catches for a quarter. It forgets `COALESCE` around `SUM`, forgets `NULLIF` around division, and puts outer-join conditions in `WHERE`. **The rule: never accept a generated aggregate query without reconciling the total against a known-correct simpler query.** Ask it "what is the grain of each row after each join?" — it answers well when asked and never volunteers it.

## 10. Learn independently

**Books (specific chapters):**
- *SQL for Smarties*, 5th ed. (Celko) — **Ch. on "Aggregate Functions"** and the NULL chapters. Celko is the authority on why `SUM` of nothing is NULL and why NULL logic is three-valued; his treatment of `COUNT(*)` vs `COUNT(col)` is definitive. Read the NULLs material even if you read nothing else.
- *The Art of PostgreSQL* (Fontaine) — **"Data Manipulation and Queries"** and the **"Aggregate Functions"**/**"Grouping Sets"** sections. Fontaine's worked examples (the F1 database) are the best demonstration of `FILTER`, `GROUPING SETS`, and ordered-set aggregates in print. The whole book is written for exactly the reader this curriculum targets.
- *SQL Performance Explained* (Winand) — **Ch. 8 "Modifying Data"** is not it; you want the discussions of **sorting and grouping** (Ch. 6, "Sorting and Grouping") — it explains precisely when an index can supply pre-sorted input to a `GroupAggregate` and eliminate the sort. Short, and directly applicable.
- *Designing Data-Intensive Applications* (Kleppmann) — **Ch. 3, "Transaction Processing or Analytics?"** and the **column-oriented storage** section. This is *why* OLAP systems exist and why your `GROUP BY` belongs on a replica or a warehouse, not the primary. Read it alongside this lesson; it converts a SQL technique into an architectural instinct.
- *PostgreSQL: Up and Running*, 3rd ed. (Obe & Hsu) — **Ch. 7 "Query Performance Tuning"** and the aggregates section of Ch. 5 for the practical Postgres-specific reference.

**Primary docs:**
- [9.21. Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html) — read the whole table once; there are more aggregates than you think.
- [7.2.3. GROUP BY and HAVING](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUP) and [7.2.4. GROUPING SETS, CUBE, ROLLUP](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPING-SETS)
- [4.2.7. Aggregate Expressions](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-AGGREGATES) — `FILTER`, `DISTINCT`, `ORDER BY` inside aggregates.
- [CREATE AGGREGATE](https://www.postgresql.org/docs/current/sql-createaggregate.html) — the state-machine model.
- [`work_mem`](https://www.postgresql.org/docs/current/runtime-config-resource.html#GUC-WORK-MEM) — read the "per operation" wording carefully.

**High-quality links:**
- [use-the-index-luke.com — Sorting and Grouping](https://use-the-index-luke.com/sql/sorting-grouping) — Winand's free companion to *SQL Performance Explained*. The pipelined-vs-blocking distinction is exactly the HashAggregate/GroupAggregate story.
- [Modern SQL — `FILTER`](https://modern-sql.com/feature/filter) and [`GROUPING SETS`](https://modern-sql.com/feature/grouping-sets) — Winand's site, with cross-database compatibility tables.
- [explain.dalibo.com](https://explain.dalibo.com/) / [explain.depesz.com](https://explain.depesz.com/) — paste your `EXPLAIN ANALYZE` output for a visual plan.

## 11. Interview preparation

**Q1. `WHERE` vs `HAVING`?**
> `WHERE` filters individual **rows** before grouping; `HAVING` filters **groups** after aggregation. It follows from the logical evaluation order — `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY` — which also explains why `WHERE` can't reference an aggregate (it doesn't exist yet) or a `SELECT` alias, while `HAVING` and `ORDER BY` can. Practically: `WHERE` can use indexes and reduces the rows that reach the aggregation, so any predicate that doesn't mention an aggregate belongs there. Postgres will often let you put a non-aggregate condition in `HAVING`, but it's semantically misleading and you shouldn't rely on the planner rescuing you.

**Q2. `COUNT(*)` vs `COUNT(col)` vs `COUNT(DISTINCT col)`?**
> `COUNT(*)` counts rows, including all-NULL ones. `COUNT(col)` counts rows where `col IS NOT NULL`. `COUNT(DISTINCT col)` counts distinct non-NULL values, and is far more expensive because it must materialise and sort/hash every value per group — and in Postgres it disables parallel aggregation. The critical case: with a `LEFT JOIN`, a parent with no children still produces one row with NULLs on the right, so `COUNT(*)` returns 1 where the answer is 0. **With an outer join, always count a NOT NULL column from the right table.**

**Q3. Why does `SUM` of an empty set return NULL rather than 0?**
> Because SQL's aggregates other than `COUNT` are defined over the *values* present, and there are none — the answer is genuinely "unknown/no data," not "zero." It's arguably debatable (the additive identity *is* 0) but it's consistent with SQL's NULL-means-absent philosophy and it's what the standard says. The practical consequence is that NULL propagates: `SUM(x) * 1.19` is NULL, `SUM(x) > 0` is NULL (not false, so a `WHERE` on it filters the row out), and your application gets `None`. Wrap it: `COALESCE(SUM(x), 0)`.

**Q4. Explain the difference between these and when each is right.**
```sql
FROM a LEFT JOIN b ON a.id = b.a_id WHERE b.status = 'x'
FROM a LEFT JOIN b ON a.id = b.a_id AND b.status = 'x'
```
> The first is an **inner join in disguise**: the `LEFT JOIN` produces NULL-filled rows for unmatched `a`s, and then `WHERE b.status = 'x'` evaluates to NULL for those rows, which isn't true, so they're filtered out. Every unmatched `a` disappears. The second filters `b` *during* the join, so unmatched `a`s survive with NULLs — which is what you almost always want. The rule: **conditions on the outer (nullable) table go in `ON`; conditions on the preserved table go in `WHERE`.** The exception is deliberate anti-joins: `LEFT JOIN ... WHERE b.id IS NULL` is the classic "find rows in a with no match in b."

**Q5. What is fan-out and how do you avoid it?**
> When you join a parent to two (or more) one-to-many children, the join produces the cartesian product of the children per parent. An order with 3 items and 2 shipments yields 6 rows, so `SUM(items.qty)` doubles and `SUM(shipments.cost)` triples. `COUNT(DISTINCT)` patches counts but nothing patches `SUM`. The fix is to **aggregate each branch to the parent's grain first** — separate CTEs or lateral subqueries — and then join the pre-aggregated results one-to-one. The discipline that prevents it: state the grain of the result at the top of the query ("one row per order") and check it after every join. This is the bug that silently inflates revenue reports, and it's invisible because the number is merely *wrong*, not absurd.

**Q6. What's the logical order of evaluation, and why does it matter?**
> `FROM/JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT`. It matters because it explains, from one rule, why aliases work in `ORDER BY` but not `WHERE`, why aggregates work in `HAVING` but not `WHERE`, and why ungrouped columns are rejected in `SELECT`. It's a *semantic* order, not an execution order — the planner reorders aggressively (predicate pushdown, join reordering) as long as the result is identical. So use the logical order to reason about *meaning*, and `EXPLAIN` to reason about *speed*.

**Q7. (System design) A dashboard runs `GROUP BY` over 500M orders and takes 45 seconds. Walk me through fixing it.**
> First I'd `EXPLAIN (ANALYZE, BUFFERS)` it and find where the time goes — usually a Seq Scan of the whole table, or a HashAggregate spilling to disk (`Batches: > 1`). Then, in escalating order of cost to implement: **(1) Reduce rows scanned** — is there a date predicate? Is it indexable? A B-tree on `(created_at)` or a composite matching the filter turns a 500M-row scan into a 5M-row range scan. **(2) Fix the spill** — if the hash table exceeds `work_mem`, raise it for that query (`SET LOCAL work_mem='256MB'`), not globally. **(3) Pre-aggregate** — the dashboard almost certainly wants daily/monthly rollups, so materialise them: a `daily_order_stats` table refreshed incrementally, or a materialised view refreshed concurrently, or a TimescaleDB continuous aggregate. The dashboard then reads a few hundred rows and it's instant. **(4) Partition** by month so old partitions are never touched and pruning is automatic. **(5) Move it off the primary** — analytics on a replica at minimum, a columnar store (ClickHouse, or a warehouse) if this is the whole workload, because a row store reads whole rows to aggregate two columns. **(6) Approximate** — if it's `COUNT(DISTINCT user_id)`, HLL turns an intractable query into a cheap one for ~2% error. The meta-answer is that you should almost never compute a user-facing aggregate at read time over the full history; you pay once at write time. And the first question I'd actually ask is "how fresh does this need to be?" — if the answer is "hourly is fine," a rollup solves it in an afternoon and everything else is unnecessary.

**Q8. (Trade-off) `COUNT(DISTINCT user_id)` over a billion rows is too slow. Options?**
> Four, with different trade-offs. **Pre-aggregate** — maintain a daily distinct-users rollup; exact and fast, but distincts don't compose (you can't sum daily uniques to get monthly uniques — the same user appears in both), so you need the raw data for every window you want, which is the real trap here. **HyperLogLog** — ~1.5 KB per sketch for ~2% error, and crucially **sketches merge**, so daily sketches *can* be unioned into monthly ones. That property is why HLL dominates this problem; it's what Redis `PFCOUNT`, BigQuery's `APPROX_COUNT_DISTINCT`, and `postgresql-hll` implement. **Bitmap/Roaring bitmaps** — exact and mergeable if your IDs are dense integers; more memory than HLL, but no error. **Sampling** — cheapest, but the error is unbounded for skewed data. I'd default to HLL for analytics dashboards and exact bitmaps for billing, and I'd push hard on the question "does this number need to be exact?" — for a "monthly active users" chart, 2% error is invisible; for an invoice, it's fraud.

**Q9. HashAggregate vs GroupAggregate — what's the difference and why does the planner pick one?**
> `HashAggregate` builds a hash table keyed by the grouping columns in one pass — O(n), no sort, but it needs the whole table of groups in `work_mem`. `GroupAggregate` requires input sorted by the grouping key and aggregates each run as it streams — constant memory, but it pays for a sort unless an index already provides the order. The planner estimates the number of distinct groups: few groups → hash; many groups (hash table won't fit) or already-sorted input available → group. The failure mode is a bad estimate: the planner thinks 1,000 groups, reality is 10 million, the hash table spills to disk, and the query goes from 2 seconds to 2 minutes. You see it in `EXPLAIN ANALYZE` as `Batches: 17  Disk Usage: 250000kB`. Fixes: `ANALYZE` the table, raise the statistics target on the skewed column, raise `work_mem` for that query, or pre-aggregate.

**Q10. When would you use `GROUPING SETS`/`ROLLUP` over `UNION ALL`?**
> Whenever you need multiple grains from the same data — detail plus subtotals plus a grand total. `UNION ALL` of three queries scans the table three times; `GROUPING SETS` scans once and computes all the sets together, which is typically 2–3× faster and, more importantly, guarantees the sets are consistent (three separate queries can drift if someone edits one predicate). `ROLLUP(a,b,c)` is the hierarchical case (year/month/day) and is much cheaper than `CUBE(a,b,c)` because the sets are prefixes and can share a single sort — `CUBE` of *n* columns is 2ⁿ sets, which is how people accidentally melt a warehouse. The one thing you must add is `GROUPING(col)`, which returns 1 for rolled-up columns — without it you can't distinguish a subtotal row's NULL from a genuine NULL in the data.

**Q11. Your report shows average salary by department. What's the risk?**
> Two. **Statistical:** the mean is destroyed by outliers and hides the distribution — one executive in a 5-person department makes the "average" meaningless. Report the median (`percentile_cont(0.5) WITHIN GROUP (ORDER BY salary)`) and the quartiles, or at least state *n* alongside. **Privacy:** aggregation over small groups is an **inference attack** surface — with `COUNT(*) = 1`, "average salary" is that person's salary, and with two queries differing by one person you can subtract to recover an individual (a differencing attack). The standard mitigations are *k*-anonymity — `HAVING COUNT(*) >= 5`, suppressing small groups — and, where the threat model warrants it, differential privacy with calibrated noise, which is what Apple and the US Census use. The `HAVING COUNT(*) >= k` guard is worth adding by reflex on any aggregate over people; it also happens to fix the tiny-denominator noise problem for free.

**Q12. Why is `COUNT(*)` slow on a large Postgres table when MySQL/MyISAM was instant?**
> MVCC. Each row version carries visibility information, and whether a row "exists" depends on your transaction's snapshot — so there is no single global row count to maintain; Postgres must check visibility for every row. An index-only scan can help when the visibility map marks pages all-visible (fewer, smaller pages to read), but it's still O(n). MyISAM was O(1) because it had no MVCC and stored a counter — and paid for it with table-level locking. If you need a fast count: `reltuples` from `pg_class` gives an estimate maintained by `ANALYZE`/autovacuum (perfect for "about 4.2M results" pagination), or maintain an exact counter table with triggers, accepting the write contention on that row. The right answer is usually "do you actually need an exact count?" — for UI pagination, almost never.

## 12. Homework

> Work in `sql/solutions/lesson-04/`. Use the schema from §2.1 plus the generator in task 1. Every query gets a comment stating **the grain of one output row**.

**Easy**

1. **Seed data.** Write `seed.sql` that generates: 1,000 customers across 8 countries, 20 products in 4 categories, 50,000 orders over 24 months with a realistic status mix (~80% paid, 15% pending, 5% refunded), and 1–5 `order_items` per order. Use `generate_series`, `random()`, and `setseed()` so it's reproducible. Ensure ~5% of customers have **zero** orders.
   *Acceptance:* re-running `seed.sql` twice produces identical data; `SELECT COUNT(*) FROM orders` = 50,000; at least 40 customers have no orders.

2. **NULL semantics.** On a temp table with NULLs, produce a single query returning `COUNT(*)`, `COUNT(col)`, `COUNT(DISTINCT col)`, `SUM`, `AVG`, `SUM/COUNT(*)`, and `array_agg`. Then run the same query with a `WHERE` clause matching zero rows and record which columns are NULL vs 0.
   *Acceptance:* a `NULLS.md` with both result sets and a one-line explanation per column.

3. **Order of evaluation.** Write five queries that each fail, one per rule: alias in `WHERE`; aggregate in `WHERE`; ungrouped column in `SELECT`; alias in `GROUP BY` (does it work? find out); `HAVING` referencing a non-grouped, non-aggregated column. Record the exact Postgres error message for each, then write the working version.
   *Acceptance:* `ORDER.md` with 5 × (broken query, error text, fixed query, one-line why).

**Medium**

4. **The `LEFT JOIN` bug, demonstrated and fixed.** Produce customer-order counts three ways: `COUNT(*)` (wrong), `COUNT(o.id)` (right), and a correlated scalar subquery. Show all three side by side for customers with and without orders. Then `EXPLAIN ANALYZE` all three and compare.
   *Acceptance:* one query with all three columns showing the discrepancy only on zero-order customers; a `PLANS.md` comparing timings and explaining why the subquery version is usually slower.

5. **Fan-out, demonstrated and fixed.** Write a query joining `customers → orders → order_items` that reports order count, item count, and revenue per customer. Show the naive version producing wrong order counts. Then fix it two ways: `COUNT(DISTINCT)`, and pre-aggregation in a CTE. Prove both give the same numbers and compare plans.
   *Acceptance:* a reconciliation query proving `SUM(revenue)` from your per-customer report equals `SUM(quantity*unit_price)` over the whole `order_items` table (filtered to paid). If they differ, you have the bug.

6. **The dashboard query.** Produce one query returning, per month: orders, unique buyers, new customers (first order that month), revenue, refund count, refund rate %, average order value, and median order value. Use `FILTER`, `NULLIF`, and `percentile_cont`. It must return a row for **every** month in the range, including months with no orders (hint: `generate_series` + `LEFT JOIN`).
   *Acceptance:* correct values, no division-by-zero on empty months, gap months present with zeros (not NULLs) where a zero is the honest answer, and a comment justifying each `COALESCE` vs. deliberate NULL.

7. **Conditional aggregation.** Pivot `order_items` into one row per category with columns per order status (`paid_qty`, `pending_qty`, `refunded_qty`, `paid_revenue`, ...). Do it once with `CASE` and once with `FILTER`. Then break the `CASE` version by using `COUNT(CASE WHEN ... THEN 1 ELSE 0 END)` and explain the wrong number.
   *Acceptance:* the two correct versions agree exactly; a written explanation of the `ELSE 0` bug with the actual wrong output.

**Hard**

8. **`GROUPING SETS` report.** Produce a single query giving revenue by `(country, category)`, subtotals per country, subtotals per category, and a grand total — with a `level` column derived from `GROUPING()` labelling each row as `'detail' | 'country_total' | 'category_total' | 'grand_total'`, and ordering that puts each subtotal directly after its group. Then write the `UNION ALL` equivalent and benchmark both on the 50k-order dataset.
   *Acceptance:* correct labels (verified by a row where the data genuinely has a NULL country — add one), a timing comparison, and a `plans/` folder with both `EXPLAIN ANALYZE` outputs. Explain the buffer-read difference.

9. **Force a hash-agg spill.** Construct a `GROUP BY` with high enough group cardinality to spill at the default `work_mem`. Capture `EXPLAIN (ANALYZE, BUFFERS)` showing `Batches > 1` and `Disk Usage`. Then fix it three ways — raise `work_mem` with `SET LOCAL`, add an index enabling a `GroupAggregate`, and pre-aggregate — and record the timing of each.
   *Acceptance:* `SPILL.md` with the spilling plan, three fixed plans, timings, and a paragraph on why you would *not* just set `work_mem = '1GB'` globally (be specific about the arithmetic: connections × nodes × workers).

10. **Cohort retention.** Build a monthly cohort table: rows = signup month, columns = months since signup (0–11), values = % of the cohort that placed an order in that month. Use `GROUP BY` + `FILTER` (no window functions — that's next lesson). Add a `cohort_size` column and suppress cohorts with fewer than 10 customers.
    *Acceptance:* a correct triangular retention matrix; month 0 is ≤100% (and you can explain why it's not always 100%); a written note on why the `HAVING cohort_size >= 10` guard is both a statistics decision and a privacy decision.

**Stretch (senior)**

11. **Reconciliation harness.** Write a `checks.sql` implementing five assertions in the dbt style, each returning zero rows on success: (a) the per-customer revenue report sums to the global total; (b) the per-month report sums to the global total; (c) `GROUPING SETS` subtotals equal the sum of their details; (d) every rollup row's `orders` equals a recomputed `COUNT(DISTINCT)`; (e) no group has `COUNT(*) = 0` where the report claims a value. Then **deliberately introduce a fan-out bug** and show which checks catch it.
    *Acceptance:* all five pass on correct data; at least two fail on the injected bug; a `WHY.md` explaining which class of bug each check catches and which bug class *none* of them catch.

12. **Approximate vs exact distinct.** Install `postgresql-hll` (or emulate with `hashtext` + a hand-rolled sketch if you can't). Compute monthly unique buyers exactly and via HLL over your 50k orders, then scale the dataset to 5M orders and compare wall time, memory, and error %. Then demonstrate the property that actually matters: **merge** 12 monthly HLL sketches into an annual figure and compare against the exact annual distinct — and show why summing 12 exact monthly counts gives a *different, wrong* answer.
    *Acceptance:* a results table (exact ms, HLL ms, error %), the merge demonstration with all three numbers (summed-monthly, merged-HLL, exact-annual), and a recommendation stating which workloads you'd ship each for.

## 13. Key takeaways

- **Logical order: `FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT`.** This one line explains aliases, aggregate placement, and the grouping rule.
- **Aggregates ignore NULLs — except `COUNT(*)`.** And **`SUM` of nothing is NULL, not 0.**
- **`COUNT(*)` with a `LEFT JOIN` is the #1 real aggregation bug.** Count a NOT NULL column from the right table, and put the right table's filters in `ON`, not `WHERE`.
- **Fan-out silently inflates every aggregate** when you join two one-to-many branches. Know your grain; pre-aggregate each branch before joining.
- `WHERE` filters **rows** (indexable, cheap); `HAVING` filters **groups**. No aggregate in the predicate ⇒ it belongs in `WHERE`.
- **`FILTER` beats `CASE`** for conditional aggregation, and sidesteps the `COUNT(... ELSE 0)` bug.
- `GROUPING SETS`/`ROLLUP` give many grains in one scan; `GROUPING()` is what distinguishes a subtotal's NULL from a real one.
- **Report percentiles, not means** — `percentile_cont` interpolates, `percentile_disc` returns a real value.
- Guard division with `NULLIF`, guard `SUM` with `COALESCE`, guard people-aggregates with `HAVING COUNT(*) >= k`.
- `HashAggregate` (fast, needs `work_mem`) vs `GroupAggregate` (sorted input, constant memory). Spills show up as `Batches > 1` and are a top cause of "it was fast yesterday."
