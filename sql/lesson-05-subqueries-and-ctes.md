# SQL Lesson 05 — Subqueries & CTEs (Including Recursive)

| | |
|---|---|
| **Track** | SQL (PostgreSQL-first) |
| **Lesson** | 05 of 15 |
| **Topic** | Scalar/row/table subqueries, correlated subqueries, `EXISTS`/`IN`/`ANY`, `LATERAL`, CTEs and recursion |
| **Estimated time** | ~2 hours |
| **Prerequisites** | Lessons 01–04 (joins and aggregation especially) |

### Learning objectives
1. Classify every subquery by **where** it appears (`SELECT`, `FROM`, `WHERE`) and by whether it's **correlated**.
2. Choose correctly between `IN`, `EXISTS`, `ANY`, and a `JOIN` — and explain the NULL trap that makes `NOT IN` dangerous.
3. Use `LATERAL` for per-row top-N and other things a plain join cannot express.
4. Write CTEs for readability, know that Postgres 12 changed their optimisation fence, and use `MATERIALIZED`/`NOT MATERIALIZED` deliberately.
5. Write recursive CTEs for hierarchies, graphs, and sequence generation — with cycle detection that actually works.
6. Read `EXPLAIN` well enough to see when the planner rewrote your subquery into a join (and when it couldn't).

---

## 1. Why this matters

Subqueries and CTEs are how SQL composes. Without them, every non-trivial question becomes a single flat query that nobody can read or review; with them, you build a pipeline of named steps that reads top-to-bottom like a program. This is the skill that makes the jump from "I can write a SELECT" to "I can express a business question in SQL."

For your target roles:
- **Data engineering:** dbt models *are* CTE chains. A well-written dbt model is 6 CTEs, each named after what it produces, ending in a final `SELECT`. Recursive CTEs handle hierarchies (org charts, category trees, BOM explosions) and graph traversal that would otherwise require pulling data into Python.
- **Backend:** `EXISTS` vs `IN` vs `JOIN` is a real latency decision on hot endpoints, and the `NOT IN` NULL trap has caused genuine production incidents — including data deletion that shouldn't have happened.
- **Interviews:** "`EXISTS` vs `IN`" and "write a recursive CTE for an org chart" are both extremely common. The `NOT IN` NULL trap is the single best "do they actually understand SQL" question there is, because it's a correctness bug that looks like nothing.

---

## 2. Theory

### 2.1 Set-up

We continue with the Lesson 04 schema (`customers`, `products`, `orders`, `order_items`), plus a self-referencing table for the recursion section:

```sql
CREATE TABLE employees (
    id        serial PRIMARY KEY,
    name      text NOT NULL,
    manager_id int REFERENCES employees(id),
    salary    numeric(10,2) NOT NULL,
    dept      text NOT NULL
);

INSERT INTO employees (name, manager_id, salary, dept) VALUES
    ('Zara',  NULL, 250000, 'exec'),      -- 1  CEO
    ('Yuri',  1,    180000, 'eng'),       -- 2
    ('Xena',  1,    175000, 'sales'),     -- 3
    ('Walt',  2,    140000, 'eng'),       -- 4
    ('Vera',  2,    145000, 'eng'),       -- 5
    ('Ugo',   4,    110000, 'eng'),       -- 6
    ('Tam',   4,    115000, 'eng'),       -- 7
    ('Sam',   3,    120000, 'sales');     -- 8

CREATE TABLE categories (
    id        serial PRIMARY KEY,
    name      text NOT NULL,
    parent_id int REFERENCES categories(id)
);
INSERT INTO categories (name, parent_id) VALUES
    ('All', NULL),                        -- 1
    ('Hardware', 1), ('Software', 1),     -- 2, 3
    ('Laptops', 2), ('Phones', 2),        -- 4, 5
    ('Gaming Laptops', 4);                -- 6
```

### 2.2 The taxonomy: where the subquery sits

**Scalar subquery** — returns exactly one row, one column. Usable anywhere a value is.
```sql
SELECT name, salary,
       salary - (SELECT AVG(salary) FROM employees) AS vs_avg     -- in SELECT
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);               -- in WHERE
```
If it returns more than one row: `ERROR: more than one row returned by a subquery used as an expression`. If it returns **zero** rows, it silently yields `NULL` — which is worse, because it doesn't error, it just makes your arithmetic NULL.

**Table subquery (derived table)** — in `FROM`. Must be aliased in Postgres.
```sql
SELECT dept, avg_salary
FROM (SELECT dept, AVG(salary) AS avg_salary FROM employees GROUP BY dept) d   -- alias 'd' REQUIRED
WHERE avg_salary > 130000;
```

**Row subquery** — returns one row, several columns.
```sql
SELECT * FROM employees
WHERE (dept, salary) = (SELECT dept, MAX(salary) FROM employees GROUP BY dept LIMIT 1);
```
Rare, but genuinely useful for multi-column comparisons.

**Predicate subquery** — in `WHERE`, with `IN` / `EXISTS` / `ANY` / `ALL`.

### 2.3 Correlated vs uncorrelated — the distinction that governs cost

An **uncorrelated** subquery is independent — it can be computed once.
```sql
SELECT * FROM employees WHERE salary > (SELECT AVG(salary) FROM employees);
--                                                           ^ no reference to the outer query
```

A **correlated** subquery references the outer row, so *conceptually* it runs once per outer row:
```sql
SELECT e.name, e.salary
FROM employees e
WHERE e.salary > (SELECT AVG(salary) FROM employees WHERE dept = e.dept);
--                                                         ^^^^^^ outer reference
```
```
 name |  salary
------+-----------
 Zara | 250000.00
 Vera | 145000.00
 Xena | 175000.00
 Tam  | 115000.00
```
"Everyone earning above their own department's average." That's a genuinely hard question to express any other way without window functions (Lesson 06 does it in one line).

**The critical nuance:** "runs once per outer row" is the *semantics*, not necessarily the *execution*. Postgres's planner frequently **decorrelates** — rewriting a correlated subquery into a hash join or a hash semi-join. Whether it can depends on the shape. So don't assume correlated = slow; assume correlated = *check the plan*. The `EXPLAIN` tell is `loops=N` on the inner node: `loops=1` means it was decorrelated, `loops=50000` means it genuinely ran per row.

### 2.4 `IN` vs `EXISTS` vs `JOIN` — and the `NOT IN` trap

The three ways to say "customers who have ordered":

```sql
-- IN
SELECT * FROM customers c WHERE c.id IN (SELECT customer_id FROM orders);

-- EXISTS (correlated)
SELECT * FROM customers c WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);

-- JOIN — needs DISTINCT, because a customer with 3 orders appears 3 times
SELECT DISTINCT c.* FROM customers c JOIN orders o ON o.customer_id = c.id;
```

On modern Postgres all three usually produce **the same plan** — a Hash Semi Join. The planner recognises the pattern. So the choice is mostly about clarity, with two real exceptions:
- `EXISTS` **short-circuits**: it stops at the first match. That matters when the inner side is huge and unindexed.
- The `JOIN` version needs `DISTINCT`, which adds a sort/hash *and* is a bug magnet — if you later add a column from `orders`, `DISTINCT` silently stops deduplicating and your row count changes.

**Prefer `EXISTS`.** It states intent ("does at least one exist?"), it never multiplies rows, and it never has the NULL problem below.

**Now the trap. This is the most important thing in this lesson.**

```sql
-- Find customers with NO orders. Looks fine:
SELECT * FROM customers c WHERE c.id NOT IN (SELECT customer_id FROM orders);
```
```
 id | name | country | signup_date
----+------+---------+-------------
  4 | Dan  | US      | 2026-01-05
```
Correct — *today*. Now let one NULL into that subquery:
```sql
INSERT INTO orders (customer_id, status, created_at) VALUES (NULL, 'paid', now());
-- (assume customer_id were nullable)

SELECT * FROM customers c WHERE c.id NOT IN (SELECT customer_id FROM orders);
```
```
 id | name | country | signup_date
----+------+---------+-------------
(0 rows)
```

**Zero rows. Silently.** Here's why: `x NOT IN (1, 2, NULL)` expands to `x <> 1 AND x <> 2 AND x <> NULL`. That last term is **NULL**, never `TRUE`. `TRUE AND TRUE AND NULL` = `NULL`, and `WHERE NULL` filters the row out. So **a single NULL in the subquery makes `NOT IN` return nothing, ever.**

Three-valued logic, in one table:

| `x NOT IN (...)` | contains NULL? | any match? | result |
|---|---|---|---|
| `3 NOT IN (1,2)` | no | no | `TRUE` |
| `1 NOT IN (1,2)` | no | yes | `FALSE` |
| `3 NOT IN (1,2,NULL)` | **yes** | no | **`NULL`** → row dropped |
| `1 NOT IN (1,2,NULL)` | yes | yes | `FALSE` |

`NOT EXISTS` has **no such problem** — it's a genuine set operation, not a chain of comparisons:
```sql
SELECT * FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);   -- always correct
```

**Rule: never write `NOT IN` against a subquery.** Use `NOT EXISTS` (or a `LEFT JOIN ... WHERE right.id IS NULL` anti-join). `NOT IN` against a literal list you wrote yourself is fine; against a subquery it's a bug waiting for a nullable column. This has caused real incidents — a `DELETE ... WHERE id NOT IN (SELECT ...)` that deleted nothing (harmless) or, with the logic inverted, deleted everything (not harmless).

**`ANY`/`ALL`** are the general forms; `IN` is literally `= ANY`:
```sql
WHERE salary > ALL (SELECT salary FROM employees WHERE dept = 'sales')   -- higher than everyone
WHERE salary > ANY (SELECT salary FROM employees WHERE dept = 'sales')   -- higher than someone
WHERE id = ANY (ARRAY[1,2,3])          -- the array form — how ORMs bind IN lists
```
`> ALL (empty set)` is `TRUE` (vacuously), `> ANY (empty set)` is `FALSE`. And `ALL` inherits the NULL trap for the same reason `NOT IN` does.

### 2.5 `LATERAL` — the thing a plain join cannot do

A normal subquery in `FROM` can't see the other tables in `FROM` — they're evaluated independently. `LATERAL` lifts that restriction: the subquery can reference columns from tables to its **left**. It's a `for` loop in SQL.

**Top-N per group** is the canonical use:
```sql
SELECT c.name, recent.id AS order_id, recent.created_at, recent.status
FROM customers c
CROSS JOIN LATERAL (
    SELECT o.id, o.created_at, o.status
    FROM orders o
    WHERE o.customer_id = c.id        -- <- references the OUTER c. Only legal with LATERAL.
    ORDER BY o.created_at DESC
    LIMIT 2                            -- <- the 2 most recent orders FOR EACH customer
) recent
ORDER BY c.name, recent.created_at DESC;
```
```
 name | order_id |       created_at       | status
------+----------+------------------------+----------
 Ann  |        3 | 2026-03-01 09:00:00+00 | refunded
 Ann  |        2 | 2026-02-14 11:30:00+00 | paid
 Bob  |        5 | 2026-07-01 08:00:00+00 | pending
 Bob  |        4 | 2026-02-20 15:00:00+00 | paid
 Cleo |        6 | 2026-07-05 12:00:00+00 | paid
```

You cannot write this with a plain join. `LIMIT 2` in a non-lateral subquery limits the whole subquery to 2 rows, not 2 per customer. The alternatives are a window function (`ROW_NUMBER() ... <= 2` — Lesson 06) or a correlated subquery per column (ugly and slow).

Two forms:
- `CROSS JOIN LATERAL (...)` — drops outer rows where the subquery returns nothing (Dan, with no orders, disappears).
- `LEFT JOIN LATERAL (...) ON true` — keeps them, with NULLs. Usually what you want.

`LATERAL` is also how you call a set-returning function per row:
```sql
SELECT c.name, t.tag
FROM customers c, LATERAL unnest(string_to_array(c.country, ',')) AS t(tag);
```

**Performance:** `LATERAL` genuinely executes per outer row (`loops=N` in `EXPLAIN`), so it is only fast if the inner query has an index. With an index on `orders(customer_id, created_at DESC)`, each iteration is an index scan that stops after 2 rows — so 1,000 customers × ~2 index lookups is trivially fast, and it beats the window-function approach (which must sort *all* orders). Without that index, it's 1,000 sequential scans and it's catastrophic. **`LATERAL` + the right index is one of the highest-leverage patterns in Postgres**, and knowing when it beats a window function is a strong senior signal.

### 2.6 CTEs — `WITH`

A CTE names a subquery so you can read the query top-to-bottom:

```sql
WITH paid_orders AS (
    SELECT * FROM orders WHERE status = 'paid'
),
order_totals AS (
    SELECT o.id, o.customer_id, SUM(oi.quantity * oi.unit_price) AS total
    FROM paid_orders o
    JOIN order_items oi ON oi.order_id = o.id
    GROUP BY o.id, o.customer_id
),
customer_stats AS (
    SELECT customer_id,
           COUNT(*)   AS orders,
           SUM(total) AS lifetime_value,
           AVG(total) AS avg_order
    FROM order_totals
    GROUP BY customer_id
)
SELECT c.name, c.country, s.orders, ROUND(s.lifetime_value, 2) AS ltv,
       ROUND(s.avg_order, 2) AS aov
FROM customers c
JOIN customer_stats s ON s.customer_id = c.id
ORDER BY s.lifetime_value DESC;
```
```
 name | country | orders |  ltv   |  aov
------+---------+--------+--------+--------
 Ann  | DE      |      2 | 217.98 | 108.99
 Bob  | DE      |      1 | 198.00 | 198.00
 Cleo | FR      |      1 |  73.50 |  73.50
```

That is a **pipeline**: filter → aggregate to order grain → aggregate to customer grain → present. Each step is named after what it produces and independently testable (`SELECT * FROM order_totals` while developing). Compare it to the equivalent nested-subquery version — three levels of parentheses read inside-out — and the case for CTEs is obvious. **This is the dbt model shape**, and it's how you should write any query longer than 15 lines.

**The optimisation fence — the version-dependent part everyone gets wrong.**

- **Postgres ≤ 11:** every CTE was an **unconditional optimisation fence**. It was materialised into a temporary result, and predicates could **not** be pushed into it. So:
  ```sql
  WITH all_orders AS (SELECT * FROM orders)          -- PG11: materialises ALL 50M rows...
  SELECT * FROM all_orders WHERE id = 42;            -- ...then filters to 1. Catastrophic.
  ```
- **Postgres ≥ 12:** CTEs referenced **exactly once**, that are not recursive and have no side effects, are **inlined** by default — the planner treats them like a subquery and pushes predicates in. CTEs referenced **multiple times**, or recursive, or containing `INSERT`/`UPDATE`/`DELETE`, are still materialised.

You can force either:
```sql
WITH lookup AS MATERIALIZED     (SELECT ... expensive ...)   -- compute once, reuse
WITH thing  AS NOT MATERIALIZED (SELECT ... )                -- always inline, even if used twice
```

When to force **`MATERIALIZED`**: the CTE is expensive and used multiple times; or the planner's estimate is badly wrong and you want to stop it choosing a terrible pushed-down plan; or the CTE calls a volatile function you need evaluated exactly once. When to force **`NOT MATERIALIZED`**: you reference it twice but it's cheap and inlining lets an index get used.

"Are CTEs slow?" is now a **version question**, and answering it with the PG12 nuance is a reliable interview differentiator — a lot of working knowledge froze at "CTEs are a fence" around 2018.

### 2.7 Data-modifying CTEs

Postgres lets `INSERT`/`UPDATE`/`DELETE ... RETURNING` live in a CTE. This is a Postgres superpower:

```sql
WITH archived AS (
    DELETE FROM orders
    WHERE created_at < '2025-01-01'
    RETURNING *
)
INSERT INTO orders_archive
SELECT * FROM archived;
```
One statement, one transaction, atomic move. Also:
```sql
WITH updated AS (
    UPDATE orders SET status = 'refunded'
    WHERE id = 42 AND status = 'paid'
    RETURNING id, customer_id
)
INSERT INTO audit_log (order_id, action, actor)
SELECT id, 'refund', current_user FROM updated;
```

**The critical semantic:** all sub-statements see the **same snapshot** and execute concurrently in the same statement. So a data-modifying CTE **cannot see its own changes**:
```sql
WITH ins AS (INSERT INTO t VALUES (1) RETURNING *)
SELECT COUNT(*) FROM t;      -- does NOT include the new row
```
And the execution order of independent CTEs is **not guaranteed** — two CTEs updating the same row produce undefined results. Data-modifying CTEs are powerful and sharp; use them for atomic move/audit patterns, not for imperative sequencing.

### 2.8 Recursive CTEs

```sql
WITH RECURSIVE cte AS (
    <anchor query>          -- the base case; runs once
    UNION [ALL]
    <recursive query>       -- references 'cte'; runs repeatedly until it returns 0 rows
)
SELECT * FROM cte;
```

The algorithm (worth knowing precisely, it's an interview question):
1. Run the anchor. Put its rows in the **result** and in a **working table**.
2. Run the recursive term, with `cte` bound to the **working table only** (not the whole accumulated result!).
3. Append the output to the result; it becomes the new working table.
4. Repeat until step 2 returns zero rows.

Note it's really *iteration*, not recursion — the SQL standard's name is a misnomer, and Postgres's docs say so.

**Org chart — walking down:**
```sql
WITH RECURSIVE org AS (
    SELECT id, name, manager_id, 1 AS level, name::text AS path
    FROM employees WHERE manager_id IS NULL           -- anchor: the CEO
  UNION ALL
    SELECT e.id, e.name, e.manager_id, o.level + 1, o.path || ' > ' || e.name
    FROM employees e
    JOIN org o ON e.manager_id = o.id                 -- recursive: children of the working set
)
SELECT level, repeat('  ', level - 1) || name AS chart, path
FROM org ORDER BY path;
```
```
 level |    chart     |             path
-------+--------------+------------------------------
     1 | Zara         | Zara
     2 |   Xena       | Zara > Xena
     3 |     Sam      | Zara > Xena > Sam
     2 |   Yuri       | Zara > Yuri
     3 |     Walt     | Zara > Yuri > Walt
     4 |       Tam    | Zara > Yuri > Walt > Tam
     4 |       Ugo    | Zara > Yuri > Walt > Ugo
     3 |     Vera     | Zara > Yuri > Vera
```
The `path` column does double duty: it produces the correct tree ordering (`ORDER BY path` is depth-first) and it's the basis of cycle detection.

**Walking up** (find someone's whole management chain) is the same shape with the join reversed:
```sql
WITH RECURSIVE chain AS (
    SELECT id, name, manager_id, 0 AS distance FROM employees WHERE name = 'Ugo'
  UNION ALL
    SELECT e.id, e.name, e.manager_id, c.distance + 1
    FROM employees e JOIN chain c ON e.id = c.manager_id
)
SELECT * FROM chain ORDER BY distance;
```
```
 id | name | manager_id | distance
----+------+------------+----------
  6 | Ugo  |          4 |        0
  4 | Walt |          2 |        1
  2 | Yuri |          1 |        2
  1 | Zara |            |        3
```

**Cycle detection — mandatory, not optional.** If the data has a cycle (A manages B manages A — a real thing in imported org data), the query runs **forever** and eats all your memory:
```sql
WITH RECURSIVE org AS (
    SELECT id, name, manager_id, ARRAY[id] AS visited, false AS is_cycle
    FROM employees WHERE manager_id IS NULL
  UNION ALL
    SELECT e.id, e.name, e.manager_id, o.visited || e.id, e.id = ANY(o.visited)
    FROM employees e
    JOIN org o ON e.manager_id = o.id
    WHERE NOT o.is_cycle                      -- stop expanding once a cycle is found
)
SELECT * FROM org WHERE NOT is_cycle;
```
Postgres 14+ gives you this as syntax:
```sql
WITH RECURSIVE org AS ( ... )
CYCLE id SET is_cycle USING path
SELECT * FROM org;
```
`UNION` (without `ALL`) also stops simple cycles by deduplicating — but it costs a distinct check on every iteration and it won't help when you're carrying a growing path column that makes every row unique. **Always add explicit cycle detection on any recursive query over data you didn't generate.**

**Category tree with aggregation:**
```sql
WITH RECURSIVE tree AS (
    SELECT id, name, parent_id, 0 AS depth FROM categories WHERE parent_id IS NULL
  UNION ALL
    SELECT c.id, c.name, c.parent_id, t.depth + 1
    FROM categories c JOIN tree t ON c.parent_id = t.id
)
SELECT depth, repeat('— ', depth) || name AS category FROM tree ORDER BY id;
```
```
 depth |       category
-------+-----------------------
     0 | All
     1 | — Hardware
     1 | — Software
     2 | — — Laptops
     2 | — — Phones
     3 | — — — Gaming Laptops
```

**Sequence generation** (no table needed):
```sql
WITH RECURSIVE fib(n, a, b) AS (
    SELECT 1, 0::bigint, 1::bigint
  UNION ALL
    SELECT n + 1, b, a + b FROM fib WHERE n < 10
)
SELECT n, a FROM fib;
```
Cute, and it proves you understand the working-table model — but in real code use `generate_series()`.

---

## 3. Official documentation quotes

> "`WITH` provides a way to write auxiliary statements for use in a larger query. These statements, which are often referred to as Common Table Expressions or CTEs, can be thought of as defining temporary tables that exist just for one query."
> — *PostgreSQL Documentation*, [7.8. WITH Queries (Common Table Expressions)](https://www.postgresql.org/docs/current/queries-with.html)

> "A useful property of `WITH` queries is that they are normally evaluated only once per execution of the parent query, even if they are referred to more than once by the parent query or sibling `WITH` queries. … However, **if a `WITH` query is non-recursive and side-effect-free … then it can be folded into the parent query, allowing joint optimization of the two query levels.** By default, this happens if the parent query references the `WITH` query just once, but not if it references the `WITH` query more than once. You can override that decision by specifying `MATERIALIZED` to force separate calculation of the `WITH` query, or by specifying `NOT MATERIALIZED` to force it to be merged into the parent query."
> — *PostgreSQL Documentation*, [7.8.1. SELECT in WITH](https://www.postgresql.org/docs/current/queries-with.html#QUERIES-WITH-SELECT)

> "Strictly speaking, this process is iteration not recursion, but `RECURSIVE` is the terminology chosen by the SQL standards committee."
> — *PostgreSQL Documentation*, [7.8.2. Recursive Queries](https://www.postgresql.org/docs/current/queries-with.html#QUERIES-WITH-RECURSIVE)

> "The sub-statements in `WITH` are executed concurrently with each other and with the main query. Therefore, when using data-modifying statements in `WITH`, the order in which the specified updates actually happen is unpredictable. **All the statements are executed with the same snapshot**, so they cannot 'see' one another's effects on the target tables."
> — *PostgreSQL Documentation*, [7.8.3. Data-Modifying Statements in WITH](https://www.postgresql.org/docs/current/queries-with.html#QUERIES-WITH-MODIFYING)

> "Subqueries appearing in `FROM` can be preceded by the key word `LATERAL`. This allows them to reference columns provided by preceding `FROM` items. (Without `LATERAL`, each subquery is evaluated independently and so cannot cross-reference any other `FROM` item.)"
> — *PostgreSQL Documentation*, [7.2.1.5. LATERAL Subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-LATERAL)

> "Note that this form of `NOT IN` … the expression yields NULL if there is no equal right-hand value and at least one right-hand row yields null."
> — *PostgreSQL Documentation*, [9.24.3. NOT IN](https://www.postgresql.org/docs/current/functions-subquery.html#FUNCTIONS-SUBQUERY-NOTIN)

---

## 4. Real-world examples

### 4.1 The `NOT IN` incident, reproduced

```sql
CREATE TEMP TABLE parents (id int);
CREATE TEMP TABLE children (parent_id int);
INSERT INTO parents VALUES (1), (2), (3);
INSERT INTO children VALUES (1), (2);

-- "Parents with no children" — correct today:
SELECT * FROM parents WHERE id NOT IN (SELECT parent_id FROM children);
--  id
-- ----
--   3

INSERT INTO children VALUES (NULL);     -- one orphan row. Perfectly legal.

SELECT * FROM parents WHERE id NOT IN (SELECT parent_id FROM children);
--  id
-- ----
-- (0 rows)                              <-- ZERO. Silently.

SELECT * FROM parents p WHERE NOT EXISTS (SELECT 1 FROM children c WHERE c.parent_id = p.id);
--  id
-- ----
--   3                                   <-- still correct

SELECT p.* FROM parents p
LEFT JOIN children c ON c.parent_id = p.id
WHERE c.parent_id IS NULL;
--  id
-- ----
--   3                                   <-- also correct (anti-join)
```

Now imagine the query was `DELETE FROM parents WHERE id NOT IN (SELECT parent_id FROM children)` — a cleanup job that ran nightly for a year, then one nullable insert made it a no-op and nobody noticed for six months. Or the inverse (`IN`), which would have deleted the wrong set. **This is the best SQL interview question in existence** because it looks like nothing and the consequences are severe.

Note the plans differ too: `NOT IN` produces a slow `Filter: (NOT (hashed SubPlan 1))` — it can't use a hash **anti-join** because of exactly this NULL semantics, so it degrades on large inner sets. `NOT EXISTS` gets a proper `Hash Anti Join`. So `NOT EXISTS` is both correct *and* faster. There is no argument for `NOT IN` on a subquery.

### 4.2 `LATERAL` top-N vs the window function

```sql
CREATE INDEX ON orders (customer_id, created_at DESC);

EXPLAIN (ANALYZE, COSTS OFF)
SELECT c.name, r.id, r.created_at
FROM customers c
LEFT JOIN LATERAL (
    SELECT o.id, o.created_at FROM orders o
    WHERE o.customer_id = c.id ORDER BY o.created_at DESC LIMIT 3
) r ON true;
```
```
 Nested Loop Left Join (actual time=0.031..0.402 rows=2847 loops=1)
   ->  Seq Scan on customers c (actual rows=1000 loops=1)
   ->  Limit (actual time=0.001..0.001 rows=3 loops=1000)
         ->  Index Scan using orders_customer_id_created_at_idx on orders o
               (actual time=0.001..0.001 rows=3 loops=1000)
               Index Cond: (customer_id = c.id)
 Execution Time: 0.51 ms
```

`loops=1000` — it *does* run per customer — but each loop is an index scan that stops after 3 rows. Total work ≈ 3,000 index tuples. The window-function alternative:

```sql
SELECT name, id, created_at FROM (
    SELECT c.name, o.id, o.created_at,
           ROW_NUMBER() OVER (PARTITION BY c.id ORDER BY o.created_at DESC) AS rn
    FROM customers c JOIN orders o ON o.customer_id = c.id
) s WHERE rn <= 3;
```
That must scan and rank **all 50,000 orders** before discarding 47,000 of them. On this shape — few rows wanted per group, many groups, good index — **`LATERAL` wins decisively**. Flip the shape (most groups wanted, no index, need other window calculations anyway) and the window function wins. Knowing that the answer is "it depends on selectivity and indexes, here's the tell in the plan" is exactly the level this curriculum targets.

### 4.3 A real dbt-shaped model

```sql
WITH paid AS (
    SELECT id, customer_id, created_at
    FROM orders
    WHERE status = 'paid' AND created_at >= '2026-01-01'
),
line_totals AS (
    SELECT order_id, SUM(quantity * unit_price) AS order_total, SUM(quantity) AS items
    FROM order_items
    WHERE order_id IN (SELECT id FROM paid)
    GROUP BY order_id
),
enriched AS (
    SELECT p.id, p.customer_id, p.created_at, lt.order_total, lt.items
    FROM paid p JOIN line_totals lt ON lt.order_id = p.id
),
per_customer AS (
    SELECT customer_id,
           COUNT(*)              AS orders,
           SUM(order_total)      AS ltv,
           AVG(order_total)      AS aov,
           MAX(created_at)       AS last_order_at
    FROM enriched
    GROUP BY customer_id
)
SELECT c.name, c.country, pc.orders, ROUND(pc.ltv, 2) AS ltv,
       ROUND(pc.aov, 2) AS aov, pc.last_order_at::date,
       CASE WHEN pc.ltv > 200 THEN 'vip'
            WHEN pc.ltv > 100 THEN 'regular'
            ELSE 'casual' END AS segment
FROM customers c
JOIN per_customer pc ON pc.customer_id = c.id
ORDER BY pc.ltv DESC;
```
```
 name | country | orders |  ltv   |  aov   | last_order_at | segment
------+---------+--------+--------+--------+---------------+---------
 Ann  | DE      |      2 | 217.98 | 108.99 | 2026-02-14    | vip
 Bob  | DE      |      1 | 198.00 | 198.00 | 2026-02-20    | regular
 Cleo | FR      |      1 |  73.50 |  73.50 | 2026-07-05    | casual
```

Five named steps, each a testable checkpoint. During development you swap the final `SELECT` for `SELECT * FROM line_totals` and inspect. In dbt each CTE would often be its own model with its own tests. Note `paid` is referenced twice (in `line_totals` and `enriched`), so **Postgres 12+ will materialise it** — which here is exactly right, since it's a cheap filter reused twice.

### 4.4 Atomic archive with a data-modifying CTE

```sql
CREATE TABLE orders_archive (LIKE orders INCLUDING ALL);

WITH moved AS (
    DELETE FROM orders
    WHERE created_at < '2026-02-01' AND status IN ('paid', 'refunded')
    RETURNING *
),
inserted AS (
    INSERT INTO orders_archive SELECT * FROM moved RETURNING id
)
SELECT COUNT(*) AS archived FROM inserted;
```
```
 archived
----------
        1
```
One statement, one transaction, atomic. There is no window where a row exists in neither table or in both. Doing this as two statements requires an explicit transaction and still risks a partial state if the connection drops between them. (Caveat: the `RETURNING`ed rows are the *deleted* versions, and the `orders_archive` FK to `customers` must exist or be dropped — `INCLUDING ALL` copies constraints.)

---

## 5. How This Is Used In Production

- **Startups:** CTEs are how a single engineer keeps a 200-line reporting query comprehensible. `EXISTS` guards permission checks on hot endpoints. Recursive CTEs handle the category tree or the "who reports to whom" page without pulling the whole table into Python and walking it there — which is the instinct most application developers have, and it's usually 100× slower because it's N+1 queries over a network.
- **Enterprises:** the transformation layer is CTEs all the way down. A **dbt** project is hundreds of models, each a CTE chain, each materialised as a table or view, with tests on grain and referential integrity between layers. `LATERAL` shows up in the serving layer for "top N per entity" endpoints (latest orders, recent activity, top products per category) — it's the pattern that makes those endpoints fast. Recursive CTEs handle BOM explosions in manufacturing, account hierarchies in finance, and permission inheritance in RBAC systems.
- **Common architectures:** **staging → intermediate → mart** in dbt is literally the CTE pipeline pattern promoted to file boundaries. **Graph traversal in the database** (recursive CTE) vs. a graph database (Neo4j) is a real architectural fork — the honest answer is that recursive CTEs handle depth-limited traversal on modest graphs perfectly well, and you only need a graph database when traversal is the *primary* workload and the depth is unbounded. **Materialised CTEs as an optimiser hint** is a legitimate production tool for taming a planner that's misestimating.
- **Scaling:** recursive CTEs are dangerous at scale because they're unbounded by default — always add a depth limit (`WHERE level < 10`) and cycle detection. A recursive query over a cyclic graph will consume all `work_mem`, spill, and then take the box down. Deep hierarchies with frequent reads are often better served by a **materialised path** (`'1.2.4.6'` as a text column with a `text_pattern_ops` index) or a **closure table** (every ancestor-descendant pair, precomputed), both of which trade write cost for O(1) reads. That trade-off — recompute-on-read vs. maintain-on-write — is the same one as Lesson 04's rollup tables, and it's *the* recurring decision in data engineering.
- **Monitoring:** `pg_stat_statements` is where you find the CTE that's materialising 50M rows. Look for queries with high `shared_blks_read` and a plan containing `CTE Scan` on a large row count — that's a fence you didn't intend. On PG ≤ 11 this was endemic; on 12+ it means someone wrote `MATERIALIZED` or referenced the CTE twice.
- **Security:** `EXISTS`-based permission checks (`WHERE EXISTS (SELECT 1 FROM acl WHERE ...)`) are the standard row-level authorization pattern, and Postgres's **Row-Level Security** policies are essentially compiled into exactly this. The `NOT IN` trap is a genuine security issue when it's the basis of an exclusion list — "block users NOT IN (allowed list)" fails **open** if the list contains a NULL, which is precisely the wrong direction to fail. Also: recursive CTEs on user-controlled depth are a DoS vector; bound them.
- **CI/CD:** dbt tests each CTE-turned-model. The `unique` test on the final model's key is a grain assertion; `relationships` tests are `NOT EXISTS` anti-joins in disguise. Every mature pipeline has "no orphaned rows" tests, and they're all this lesson's anti-join pattern.
- **Common tools:** `dbt` (the CTE pipeline as a framework), `pg_stat_statements`, `explain.dalibo.com`, `ltree` (Postgres extension for materialised paths — a genuine alternative to recursive CTEs for trees), `pgRouting` (graph algorithms in SQL, built on recursive CTEs).
- **Real examples:** **GitLab** publishes its entire dbt project and Postgres schema publicly — it's the best available real-world example of production CTE pipelines, and worth reading. **Stack Overflow** famously runs a small number of very large SQL Server boxes with heavily CTE-structured queries. **Amazon** and **Google** both push the "precompute the hierarchy, don't traverse it at read time" pattern for anything user-facing at scale, which is the closure-table argument above.

---

## 6. Advanced corner

**How the planner decorrelates.** A correlated `EXISTS` is usually rewritten into a **semi-join** (`Hash Semi Join` / `Nested Loop Semi Join`), and `NOT EXISTS` into an **anti-join** (`Hash Anti Join`). A semi-join stops at the first match per outer row — that's the "short-circuit," implemented in the executor, not the SQL. This is why `EXISTS` and `IN` converge on the same plan: both become semi-joins. What *cannot* be decorrelated: a correlated subquery in the `SELECT` list returning a computed value, and a correlated subquery with `LIMIT` — those stay as `SubPlan`s with `loops=N`. Seeing `SubPlan` with a high `loops` in a plan is the signal to rewrite as a join or `LATERAL`.

**`SubPlan` vs `InitPlan`.** In `EXPLAIN`, an **`InitPlan`** is evaluated **once** before the main query — that's an uncorrelated subquery (`WHERE salary > (SELECT AVG(salary) FROM employees)`). A **`SubPlan`** is evaluated per outer row — that's a correlated one that wasn't decorrelated. `hashed SubPlan` means Postgres built a hash table of the inner results and probes it per row — much better than re-executing, and it's what `IN` gets. Recognising these three in a plan tells you immediately what the planner did with your subquery, and it's a genuinely useful skill that most people never acquire.

**Why `NOT IN` can't use an anti-join.** A hash anti-join emits an outer row when no match is found. But `NOT IN` semantics require emitting **nothing at all** if the inner set contains any NULL — that's not an anti-join, it's a different operator. So Postgres falls back to `Filter: (NOT (hashed SubPlan 1))`, evaluated per row, which cannot be pushed into the join and cannot use the join's optimisations. **`NOT IN` is slower *because* it's semantically broken.** That's a beautiful thing to be able to say in an interview: the correctness bug and the performance bug have the same root cause.

**The recursive working-table model.** Step 2 of the algorithm binds `cte` to the **working table** — the rows produced by the *previous* iteration only — not the accumulated result. This surprises people: you cannot write `WHERE NOT EXISTS (SELECT 1 FROM cte ...)` inside the recursive term to deduplicate against everything seen so far, because `cte` there means "last iteration's rows." That's exactly why cycle detection needs an accumulated array column (`visited`) carried along each row, rather than a lookup against the result set. Postgres also forbids more than one reference to the recursive term, aggregates over it, and `LEFT JOIN` with it on the nullable side — all because the working-table model can't express them.

**`UNION` vs `UNION ALL` in recursion.** `UNION` deduplicates each iteration against the *entire accumulated result*, which does stop simple cycles — but it costs a distinct check per iteration, and it silently fails to help when you carry a path or depth column that makes every row unique anyway. `UNION ALL` is faster and is what you want with explicit cycle detection. Default to `UNION ALL` + a `visited` array or PG14's `CYCLE` clause.

**Recursive CTEs are breadth-first.** Each iteration processes an entire level. So `level` comes out in order naturally, but the *rows within* the result are grouped by depth, not by branch — which is why you need `ORDER BY path` for a depth-first tree rendering. PG14's `SEARCH DEPTH FIRST BY id SET ordercol` gives you that as syntax:
```sql
WITH RECURSIVE t AS (...) SEARCH DEPTH FIRST BY id SET ord SELECT * FROM t ORDER BY ord;
```

**`MATERIALIZED` as a planner hint.** Postgres deliberately has no query hints (`/*+ INDEX(...) */`), which is a design stance — the argument is that hints rot as data changes. But `MATERIALIZED` is a de facto hint: it forces an optimisation fence, which lets you stop the planner from pushing a predicate into a CTE where it chooses a disastrous plan from a bad estimate. It's the one legitimate escape hatch, and using it knowingly (rather than cargo-culting `WITH` everywhere) is a mark of experience. The honest caveat: it's a fence, so you're now responsible for the plan on both sides.

**What most people get wrong at this level:**
- `NOT IN` on a subquery. Universally.
- "CTEs are always materialised" — true until PG11, false since PG12, and the version matters.
- "CTEs are slower than subqueries" — since PG12 a single-reference CTE is inlined and produces an *identical* plan. There is no performance argument against CTEs for readability anymore.
- Writing a recursive CTE with no depth limit and no cycle detection against production data.
- Using `DISTINCT` to paper over a fan-out from a semi-join that should have been `EXISTS`.
- Not knowing `LATERAL` exists, and so writing top-N-per-group with a window function that sorts the entire table.
- Expecting a data-modifying CTE to see its own writes.

---

## 7. Best practices

- **Never `NOT IN` against a subquery.** `NOT EXISTS`, or a `LEFT JOIN ... IS NULL` anti-join. Non-negotiable.
- **Prefer `EXISTS` over `IN` over `JOIN + DISTINCT`** for "does a related row exist?" — it states intent, can't multiply rows, and can't be NULL-poisoned.
- **Use CTEs for anything longer than ~15 lines.** Name each step after what it *produces* (`paid_orders`, not `step2`). The readability is free since PG12.
- **Reach for `LATERAL` for top-N-per-group** — and make sure the supporting index exists, or it's a disaster. `LEFT JOIN LATERAL (...) ON true` when you need to keep outer rows.
- **Every recursive CTE gets a depth limit and cycle detection.** `WHERE level < 20` plus a `visited` array (or PG14's `CYCLE` clause). Treat this as mandatory as `LIMIT` on an ad-hoc query against prod.
- **`UNION ALL` in recursive terms**, with explicit cycle detection, rather than relying on `UNION`'s dedup.
- Alias every derived table (Postgres requires it) and qualify every column in a multi-table query — an unqualified column that later becomes ambiguous is a silent behaviour change.
- Use `MATERIALIZED` deliberately: expensive CTE used multiple times, or to fence off a bad plan. Use `NOT MATERIALIZED` when a multiply-referenced CTE is cheap and inlining unlocks an index.
- Prefer a **scalar subquery in `SELECT`** only for genuinely one-off values; if you're writing three of them against the same table, that's a `LATERAL` or a join.
- For deep hierarchies read far more often than written, precompute — `ltree`, a materialised path, or a closure table — rather than recursing on every read.
- Data-modifying CTEs: use them for atomic move/audit patterns; never rely on ordering between them.

## 8. Common mistakes & gotchas

**`NOT IN` + NULL = zero rows** (§4.1). The big one.

**Scalar subquery returning zero rows yields NULL, not an error**
```sql
SELECT name, (SELECT price FROM products WHERE id = 999) AS p FROM customers;
-- p is NULL for every row. No error. Your arithmetic downstream silently becomes NULL.
```

**Scalar subquery returning 2+ rows *does* error** — at runtime, on the first offending row, which means it can pass in dev and fail in prod when the data grows.

**Missing alias on a derived table**
```sql
SELECT * FROM (SELECT 1);   -- ERROR: subquery in FROM must have an alias
SELECT * FROM (SELECT 1) s; -- fine
```

**Correlated subquery in `SELECT` = N+1 in SQL**
```sql
SELECT c.name,
       (SELECT COUNT(*) FROM orders WHERE customer_id = c.id) AS orders,
       (SELECT MAX(created_at) FROM orders WHERE customer_id = c.id) AS last_order
FROM customers c;
-- TWO SubPlans, each with loops=N. Rewrite as one LEFT JOIN LATERAL or a grouped join.
```

**Recursive CTE with no termination** — runs until `work_mem` and then the OOM killer. Always bound it.

**Expecting `cte` in the recursive term to mean "everything so far"** — it means the last iteration only.

**Data-modifying CTE can't see its own writes** (§2.7).

**Assuming the PG11 fence** — `WITH x AS (SELECT * FROM big) SELECT * FROM x WHERE id=1` is fast on PG12+ (inlined) and catastrophic on PG11 (materialises everything). Know your version.

**`LATERAL` without an index** — `loops=50000` × a sequential scan. Check the plan; the pattern is only good *with* the index.

**`IN` with a huge list from an ORM** — `WHERE id IN (1,2,...,50000)` blows past parameter limits and plans badly. Use `= ANY(ARRAY[...])` (one parameter) or a `VALUES`/`unnest` join.

## 9. Where AI helps (and where it hurts)

- **Helps:** writing the recursive-CTE skeleton, which is fiddly syntax you use rarely; restructuring a deeply nested subquery mess into a readable CTE chain (genuinely good at this); explaining `SubPlan`/`InitPlan` in a plan you don't recognise; remembering `LATERAL` syntax and the `ON true` idiom.
- **Hurts:** AI writes **`NOT IN`** constantly — it's the most common phrasing in its training data and it will not warn you about NULLs unless asked. It writes recursive CTEs with **no cycle detection and no depth limit**, which is fine on the toy data it imagines and lethal on real imported data. It confidently repeats the pre-PG12 "CTEs are an optimisation fence" claim as current fact, because the internet froze on that around 2018. And it will not suggest `LATERAL` unprompted, because window functions are more common in the corpus — so you'll get the correct-but-slow version. Ask it explicitly: *"is this NULL-safe?"*, *"what's the termination condition?"*, and *"what Postgres version does that apply to?"*

## 10. Learn independently

**Books (specific chapters):**
- *SQL for Smarties*, 5th ed. (Celko) — **the chapters on NULLs and on subqueries**, plus **"Trees and Hierarchies in SQL"** (Celko wrote an entire separate book on this: *Joe Celko's Trees and Hierarchies in SQL for Smarties*, which is the definitive treatment of adjacency lists, nested sets, materialised paths, and closure tables — read it if you touch hierarchies seriously). His NULL chapters are why you'll never write `NOT IN` again.
- *The Art of PostgreSQL* (Fontaine) — the **"Common Table Expressions"** and **"LATERAL"** sections, plus the extended F1-database worked examples. Fontaine is opinionated about CTEs-as-code-structure in exactly the way this lesson argues.
- *SQL Performance Explained* (Winand) — **Ch. 7 "Partial Results"** covers exactly the top-N-per-group problem and why the pipelined index approach (`LATERAL`) beats the sort-everything approach. Short and directly applicable.
- *PostgreSQL: Up and Running*, 3rd ed. (Obe & Hsu) — **Ch. 7** for `LATERAL` and CTE practicalities with runnable examples.
- *Designing Data-Intensive Applications* (Kleppmann) — **Ch. 2, "Query Languages for Data"** — the declarative-vs-imperative argument and the graph-data section (property graphs, Cypher, and the recursive-CTE equivalent) directly informs the "recursive CTE vs graph database" decision in §5.

**Primary docs:**
- [7.8. WITH Queries](https://www.postgresql.org/docs/current/queries-with.html) — read all of it, including §7.8.2 (recursion) and §7.8.3 (data-modifying). It's one of the best-written pages in the Postgres docs.
- [7.2.1.5. LATERAL Subqueries](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-LATERAL)
- [9.24. Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html) — `EXISTS`, `IN`, `NOT IN`, `ANY`, `ALL`, with the NULL semantics stated explicitly.
- [ltree](https://www.postgresql.org/docs/current/ltree.html) — the materialised-path alternative to recursion.

**High-quality links:**
- [use-the-index-luke.com — Partial Results / Top-N](https://use-the-index-luke.com/sql/partial-results/top-n-queries) — Winand on why the index-supported top-N is a different complexity class.
- [Modern SQL — WITH](https://modern-sql.com/feature/with) and [WITH RECURSIVE](https://modern-sql.com/feature/with/recursive) — cross-database compatibility and the standard's view.
- [GitLab's dbt project](https://gitlab.com/gitlab-data/analytics) — real production CTE pipelines, publicly readable.
- [Postgres wiki — Loose indexscan](https://wiki.postgresql.org/wiki/Loose_indexscan) — a recursive-CTE trick to emulate `SKIP SCAN` for `DISTINCT` on a low-cardinality leading column. Genuinely clever, occasionally a 1000× win.

## 11. Interview preparation

**Q1. `IN` vs `EXISTS` — which is faster?**
> On modern Postgres, usually **the same** — the planner rewrites both into a semi-join, so you'll see an identical `Hash Semi Join` in both plans. The real differences are semantic. `EXISTS` short-circuits at the first match (an executor property of semi-joins). `IN` materialises or hashes the inner result. And critically, the *negated* forms diverge completely: `NOT EXISTS` becomes a clean anti-join, while `NOT IN` cannot — because its NULL semantics aren't an anti-join — so it degrades to a per-row filter *and* it's semantically broken. I default to `EXISTS`/`NOT EXISTS` because it states intent, never multiplies rows, and is NULL-safe.

**Q2. Why is `NOT IN` dangerous?**
> Because of three-valued logic. `x NOT IN (a, b, c)` is defined as `x <> a AND x <> b AND x <> c`. If any element is NULL, that comparison yields NULL, and `TRUE AND NULL` is NULL — so the whole predicate is NULL, never TRUE, and **every row is filtered out**. A single NULL in the subquery makes `NOT IN` return zero rows, silently, forever. It's not an error and there's no warning. The failure is severe because these queries are usually cleanup or exclusion logic: "delete parents with no children" becomes a no-op, or an inverted version deletes everything. Use `NOT EXISTS` — it's a genuine set operation with no NULL semantics — or a `LEFT JOIN ... WHERE right.key IS NULL` anti-join. The elegant part is that `NOT IN` is also *slower* for the same reason: Postgres can't use a hash anti-join, so it falls back to a per-row filter.

**Q3. Are CTEs an optimisation fence in PostgreSQL?**
> It depends on the version, and that's the whole answer. **Through PG11**, yes — unconditionally. Every CTE was materialised and no predicate could be pushed in, so `WITH x AS (SELECT * FROM huge) SELECT * FROM x WHERE id = 42` materialised the entire table then filtered. **From PG12**, a CTE that is non-recursive, side-effect-free, and referenced **exactly once** is inlined by default and optimised jointly with the parent — identical plan to a subquery. Multiply-referenced, recursive, or data-modifying CTEs are still materialised. You can override with `AS MATERIALIZED` / `AS NOT MATERIALIZED`. So on 12+ there is no performance reason to avoid CTEs for readability, and `MATERIALIZED` becomes a legitimate hint for fencing off a bad plan.

**Q4. When would you use `LATERAL`?**
> When the subquery needs to reference a column from a table to its left — most commonly **top-N per group**. `LEFT JOIN LATERAL (SELECT ... WHERE o.customer_id = c.id ORDER BY created_at DESC LIMIT 3) ON true` gives the 3 most recent orders *per customer*; a plain subquery can't, because `LIMIT 3` would apply to the whole subquery. It's also how you call a set-returning function per row. Performance-wise it's a nested loop with `loops = outer rows`, so it lives or dies on the inner index: with `(customer_id, created_at DESC)` each iteration is an index scan that stops after 3 rows and it's extremely fast; without it, it's N sequential scans and it's a disaster.

**Q5. `LATERAL` vs `ROW_NUMBER()` for top-N per group?**
> It's a selectivity question. `ROW_NUMBER()` must scan and sort **every** row in every partition, then discard the ones ranked above N — so its cost is proportional to the whole table regardless of N. `LATERAL` with a matching index touches only ~N rows per group and stops. So: **few rows per group, many groups, index available → `LATERAL`**, often by an order of magnitude. **Large N relative to group size, no usable index, or you need other window calculations anyway → the window function**, since you're paying for the sort regardless. The tell in `EXPLAIN` is whether the `LATERAL`'s inner node is an `Index Scan` with a small `rows` per loop or a `Seq Scan`.

**Q6. Explain how a recursive CTE executes.**
> It's iteration, not recursion — the Postgres docs say so explicitly. Run the anchor term once; its rows go into both the result and a **working table**. Then repeatedly run the recursive term with the CTE name bound to **the working table only** — not the accumulated result — appending output to the result and replacing the working table with it. Stop when an iteration produces zero rows. Two consequences people miss: it's **breadth-first** (each iteration is one level), so you need an accumulated path column and `ORDER BY path` for a depth-first rendering; and because the recursive term sees only the last iteration, you **cannot** deduplicate against everything seen so far with a subquery — cycle detection must carry a `visited` array along each row (or use PG14's `CYCLE` clause).

**Q7. Your recursive CTE hangs and eats all the memory. What happened?**
> Almost certainly a **cycle** in the data — A manages B manages A, or a category whose parent chain loops. Each iteration produces new rows forever. This is common in imported or user-edited hierarchies, which is why cycle detection isn't optional. Fixes: carry a `visited` array (`o.visited || e.id`) and filter `WHERE NOT e.id = ANY(o.visited)`; use PG14+'s `CYCLE id SET is_cycle USING path`; or add a hard depth limit `WHERE level < 20`, which I'd add *regardless* as a backstop. `UNION` instead of `UNION ALL` stops simple cycles by deduplicating, but it costs a distinct check per iteration and doesn't help at all if you're carrying a path or depth column that makes every row unique. I'd also add a `statement_timeout` — an unbounded recursive query on user-controlled input is a genuine DoS vector.

**Q8. (System design) Model a category tree, 6 levels deep, read 10,000×/second and edited a few times a day. How?**
> The read/write ratio decides it. An **adjacency list** (`parent_id`) with a recursive CTE per read is correct but pays traversal cost 10,000 times a second — wrong shape. So precompute. **Materialised path** (`path text = '1.2.4.6'`, or Postgres's `ltree` type with a GiST index) makes "all descendants of X" a single prefix scan — O(1)-ish reads, and writes only touch the moved subtree's paths. **Closure table** (a row per ancestor-descendant pair) makes both directions a single indexed lookup and is the most flexible, at the cost of O(depth × subtree) rows per move and more storage. **Nested sets** are read-optimal but a single insert renumbers half the tree — disqualifying for anything with concurrent writes. I'd pick `ltree` here: 6 levels is shallow, prefix queries are the dominant access pattern, edits are rare so the write amplification is irrelevant, and it's one column plus one index rather than a whole second table. I'd keep `parent_id` as the source of truth and maintain `path` with a trigger, so the denormalisation is verifiable. The general principle is the one from Lesson 04's rollup tables: when reads vastly outnumber writes, move the work to write time.

**Q9. (Trade-off) When would you deliberately force `MATERIALIZED`?**
> Three cases. **Reuse:** the CTE is expensive and referenced multiple times — though PG12+ materialises multiply-referenced CTEs by default anyway, so this is mostly about being explicit. **Fencing a bad plan:** the planner's row estimate for the CTE is badly wrong (skewed data, an unanalysed table, a function it can't estimate), and inlining lets it push a predicate in and choose something disastrous. Materialising forces it to compute the CTE on its own terms first. This is Postgres's only real hint mechanism, since it deliberately has no `/*+ ... */` hints. **Volatile functions:** you need `random()` or `now()`-adjacent logic evaluated exactly once rather than per-reference. The cost is that you've taken responsibility for the plan — you've told the optimiser to stop optimising across that boundary, so if the data shape changes, your fence may become the problem. I'd always leave a comment saying *why*.

**Q10. What's the difference between `InitPlan` and `SubPlan` in `EXPLAIN`?**
> An **`InitPlan`** is evaluated **once**, before the main query — that's an uncorrelated subquery like `WHERE salary > (SELECT AVG(salary) FROM employees)`. A **`SubPlan`** is evaluated per outer row — a correlated subquery the planner couldn't decorrelate, and you'll see `loops=N` on it. A **`hashed SubPlan`** means Postgres built a hash table of the inner results once and probes it per row, which is what `IN` typically gets — much better than re-execution. The practical use: seeing a plain `SubPlan` with `loops=50000` is your signal that a correlated subquery is genuinely running per row, and it should probably be a join or a `LATERAL`. Most people never learn to read this, and it's the fastest way to diagnose a subquery performance problem.

**Q11. Can a data-modifying CTE see its own changes?**
> No. All sub-statements in a `WITH` execute against the **same snapshot** and run concurrently with each other and the main query, so `WITH ins AS (INSERT ... RETURNING *) SELECT COUNT(*) FROM t` does not count the new row — you must select from `ins` itself. Relatedly, the execution *order* of independent CTEs is unspecified, so two CTEs modifying the same row give undefined results. What they're excellent for is atomic move-and-audit: `WITH moved AS (DELETE ... RETURNING *) INSERT INTO archive SELECT * FROM moved` is one statement with no intermediate state where the row is in neither table or both. It's a genuine Postgres advantage over most other databases.

**Q12. (Senior) Rewrite this and explain why:** `SELECT c.*, (SELECT COUNT(*) FROM orders o WHERE o.customer_id=c.id) AS n FROM customers c;`
> That's a correlated scalar subquery in the `SELECT` list — an **N+1 inside the database**. The planner will emit a `SubPlan` with `loops = |customers|`; with an index on `orders(customer_id)` each loop is cheap, so it's often survivable, but it doesn't scale and it gets much worse the moment you add a second such column (each becomes its own `SubPlan`, each with its own N loops). I'd rewrite it as a `LEFT JOIN` to a pre-aggregated subquery — `LEFT JOIN (SELECT customer_id, COUNT(*) n FROM orders GROUP BY customer_id) o ON o.customer_id = c.id` — which is a single hash aggregate plus a hash join, and it computes *all* the per-customer aggregates in one pass. The `COALESCE(o.n, 0)` matters, since customers with no orders get NULL from the outer join. The general rule: one correlated scalar subquery is a judgement call; three of them against the same table is unambiguously a join. If I genuinely needed only a handful of rows out (say, `LIMIT 20` after ordering by something indexed), I'd reconsider — the `SubPlan` only runs for rows that survive, so on a tiny result set it's actually the cheaper plan. That's the nuance: the rewrite is right for full scans, wrong for small `LIMIT`s.

## 12. Homework

> Work in `sql/solutions/lesson-05/`. Use the Lesson 04 seed plus the `employees`/`categories` tables from §2.1. Capture `EXPLAIN (ANALYZE, BUFFERS)` for every task that mentions plans.

**Easy**

1. **Subquery taxonomy.** Write one query of each kind against the schema: scalar-in-`SELECT`, scalar-in-`WHERE`, derived table in `FROM`, row subquery, `IN`, `EXISTS`, `ANY`, `ALL`. Label each with a comment saying whether it's correlated.
   *Acceptance:* eight working queries with correct labels; for two of them, the `EXPLAIN` node type (`InitPlan` vs `SubPlan`) noted in a comment.

2. **Scalar subquery edge cases.** Demonstrate: a scalar subquery returning zero rows (silent NULL); one returning two rows (runtime error — capture the exact message); and the difference between `(SELECT MAX(x) ...)` on an empty table vs `(SELECT x ... LIMIT 1)` on an empty table.
   *Acceptance:* a `SCALARS.md` with each case, its output/error, and a one-line note on which one is more dangerous and why.

3. **Four ways to say "has ordered".** Write the same question with `IN`, `EXISTS`, `JOIN + DISTINCT`, and a `LEFT JOIN ... IS NOT NULL`. Confirm all four return identical rows, then `EXPLAIN` all four.
   *Acceptance:* a `PLANS.md` comparing node types and timings, and a statement of which you'd commit and why.

**Medium**

4. **The `NOT IN` incident.** Reproduce §4.1 on your 50k dataset: make `orders.customer_id` nullable, insert one NULL row, and show `NOT IN` collapsing to zero rows while `NOT EXISTS` and the anti-join stay correct. Then `EXPLAIN` `NOT IN` vs `NOT EXISTS` and identify the exact plan-node difference.
   *Acceptance:* the three result sets, both plans, and a written explanation connecting the *correctness* bug and the *performance* bug to the same root cause (why an anti-join is impossible under `NOT IN` semantics).

5. **CTE pipeline.** Rewrite this nested horror as a CTE chain and prove the plans are equivalent on PG12+:
   ```sql
   SELECT * FROM (SELECT customer_id, SUM(t) AS ltv FROM
     (SELECT o.customer_id, o.id, SUM(oi.quantity*oi.unit_price) AS t
      FROM orders o JOIN order_items oi ON oi.order_id=o.id
      WHERE o.status='paid' GROUP BY o.customer_id, o.id) x
     GROUP BY customer_id) y WHERE ltv > 100;
   ```
   *Acceptance:* a readable 3-CTE version with named steps; identical output; a diff of both `EXPLAIN` outputs showing they're the same plan (or an explanation of why they differ).

6. **Prove the PG12 inlining rule.** Write a CTE referenced **once** and show it's inlined (predicate pushed down — look for the `WHERE` in the scan node, and the absence of a `CTE Scan`). Then reference it **twice** and show it materialises (`CTE Scan` appears). Then force each with `MATERIALIZED`/`NOT MATERIALIZED` and capture all four plans with timings.
   *Acceptance:* `INLINING.md` with four plans, four timings, and a paragraph stating the rule from the observed evidence rather than from this lesson.

7. **`LATERAL` top-N.** For each customer, return their 3 most recent orders with the order total. Do it three ways: `LATERAL`, `ROW_NUMBER()`, and a correlated subquery per column. Benchmark all three **with** and **without** the index on `orders(customer_id, created_at DESC)`.
   *Acceptance:* a 3×2 timing table, and a written rule for when you'd pick each — derived from your own numbers, including the crossover you observed.

**Hard**

8. **Org chart, hardened.** Using `employees`, write a recursive CTE producing: `level`, an indented name, the full path, the count of direct reports, the count of *all* descendants, and total salary of the subtree. Then **inject a cycle** (`UPDATE employees SET manager_id=7 WHERE id=2`) and show the query hanging. Fix it three ways: a `visited` array, PG14's `CYCLE` clause, and a hard depth limit. Add a `statement_timeout` backstop.
   *Acceptance:* the working chart; evidence of the hang (with the timeout firing); all three fixes working on the cyclic data; a note on which you'd ship and why. The subtree aggregates are the hard part — you'll need a second recursive pass or a correlated `LATERAL`.

9. **Hierarchy shootout.** Model the same 6-level, 100k-node category tree four ways: adjacency list + recursive CTE, materialised path (text + `text_pattern_ops` index), `ltree` + GiST, and a closure table. Implement three operations on each: "all descendants of X", "full ancestor chain of X", and "move subtree X under Y". Benchmark all 12.
   *Acceptance:* a 4×3 results table (ms + rows read), the `move` implementation for each with its write amplification measured, and a recommendation paragraph for a read-heavy vs a write-heavy workload. This is the single most interview-relevant task in the lesson.

10. **Atomic archive + audit.** Build a data-modifying CTE that, in one statement: deletes orders older than a cutoff, inserts them into `orders_archive`, inserts a summary row into `archive_runs (run_at, rows_moved, oldest, newest)`, and returns the run summary. Then prove the snapshot semantics: write a CTE whose main query tries to see the CTE's own insert and show it can't.
    *Acceptance:* the working statement; a demonstration that killing the session mid-statement leaves *no* partial state (use a `pg_sleep` in a second session with an advisory lock, or just reason it through with a `RAISE`); and the snapshot demonstration with output.

**Stretch (senior)**

11. **Graph traversal.** Add a `follows (follower_id, followee_id)` table with 10k users and 200k edges including cycles. Write a recursive CTE for "everyone within 3 hops of user X", with cycle detection and a hop limit, returning the shortest hop distance per user (not just any distance — that's the hard part; `UNION ALL` will give you duplicates at multiple depths). Then compare against doing it in Python with a BFS over a fetched adjacency list.
    *Acceptance:* correct shortest-distance output (verify against a Python BFS on the same data); timings for both; and a written analysis of when you'd move this to a graph database, with the specific threshold you'd use.

12. **Loose index scan.** Postgres has no `SKIP SCAN`. On a table with 5M rows and a low-cardinality leading column (say 20 distinct `country` values), `SELECT DISTINCT country` scans everything. Implement the [recursive-CTE loose-indexscan trick](https://wiki.postgresql.org/wiki/Loose_indexscan) and benchmark it against `SELECT DISTINCT` and `GROUP BY`.
    *Acceptance:* all three timings and buffer counts; an explanation of *why* the recursive version is orders of magnitude faster (it's a beautiful demonstration of the working-table model); and a note on when the trick stops paying off as cardinality rises — find the crossover empirically.

## 13. Key takeaways

- Classify subqueries by **position** (`SELECT`/`FROM`/`WHERE`) and by **correlation**. Correlated means "per outer row" *semantically*; check `loops=` in `EXPLAIN` to see whether the planner decorrelated it.
- **Never `NOT IN` against a subquery.** One NULL makes it return zero rows, silently — and the same semantics prevent an anti-join, so it's slow *because* it's broken. Use `NOT EXISTS`.
- Prefer **`EXISTS`** over `IN` over `JOIN + DISTINCT`: it states intent, short-circuits, can't multiply rows, and is NULL-safe. Modern Postgres plans `IN` and `EXISTS` the same anyway.
- **`LATERAL` is a `for` loop in SQL** — the only clean way to do top-N per group. Fast with the right index, catastrophic without it.
- **CTEs are inlined since PG12** when referenced once and side-effect-free; multiply-referenced/recursive/data-modifying CTEs still materialise. "CTEs are a fence" is a PG11 fact. `MATERIALIZED` is Postgres's one legitimate planner hint.
- Recursive CTEs are **iteration over a working table**, breadth-first, where the recursive term sees only the *last* iteration. That's why cycle detection needs a carried `visited` array.
- **Every recursive CTE needs a depth limit and cycle detection.** Non-negotiable against data you didn't generate.
- For deep, read-heavy hierarchies, precompute (`ltree`, materialised path, closure table) instead of traversing per read — the same write-time-vs-read-time trade as rollup tables.
- Data-modifying CTEs give you atomic move-and-audit in one statement, but they share one snapshot and can't see each other's writes.
- `InitPlan` = once. `SubPlan` = per row. `hashed SubPlan` = hashed once, probed per row. Learn to spot them.
