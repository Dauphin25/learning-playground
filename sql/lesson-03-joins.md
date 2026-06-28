# SQL Lesson 03 — Joins

| | |
|---|---|
| **Track** | SQL |
| **Lesson** | 03 of 15 |
| **Topic** | Combining tables: inner, left/right/full outer, cross, self, and semi/anti joins |
| **Estimated time** | ~2 hours |
| **Prerequisites** | Lessons 01–02; a schema with related tables (customers ↔ orders) |

### Learning objectives
1. Explain how a join combines rows from multiple tables on a condition.
2. Choose correctly between INNER, LEFT/RIGHT, FULL OUTER, and CROSS joins.
3. Write self-joins and multi-table joins with clear aliases.
4. Implement semi-joins and anti-joins (`EXISTS`/`NOT EXISTS`) and know why they beat `IN`/`NOT IN`.
5. Avoid the classic join bugs: accidental cross products, NULLs from outer joins, fan-out duplication.

---

## 1. Why this matters
Real data is split across normalized tables (Lesson 08), so almost every useful query joins them. Joins are *the* SQL skill interviewers test most, and the most common place beginners produce wrong-but-plausible results (duplicated rows, missing rows). Master the four join types and the EXISTS pattern and you can answer the majority of analytical questions.

---

## 2. Setup data for this lesson
```sql
CREATE TABLE customers (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);
CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    customer_id INT REFERENCES customers(id),
    amount      NUMERIC(10,2) NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT now()
);

INSERT INTO customers (name) VALUES ('Ann'), ('Bob'), ('Cleo');  -- ids 1,2,3
INSERT INTO orders (customer_id, amount) VALUES
    (1, 50), (1, 75),       -- Ann has two orders
    (2, 20);                -- Bob has one; Cleo has none
```

---

## 3. Theory

### 3.1 What a join is
A join produces a result by combining each row of one table with matching rows of another based on a **join condition** (usually `a.fk = b.pk`). Conceptually it starts from the Cartesian product (every row of A × every row of B) and keeps the rows satisfying the condition.

### 3.2 The join types

**INNER JOIN** — only rows with a match in *both* tables.
```sql
SELECT c.name, o.amount
FROM customers c
INNER JOIN orders o ON o.customer_id = c.id;
-- Ann/50, Ann/75, Bob/20   (Cleo dropped: no orders)
```

**LEFT [OUTER] JOIN** — all rows from the left table; matched right columns, else NULL.
```sql
SELECT c.name, o.amount
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id;
-- Ann/50, Ann/75, Bob/20, Cleo/NULL   (Cleo kept)
```

**RIGHT [OUTER] JOIN** — mirror image: all rows from the right table. (Rare; usually rewrite as LEFT by swapping table order for readability.)

**FULL [OUTER] JOIN** — all rows from both sides; NULLs where no match on either side.

**CROSS JOIN** — Cartesian product (every combination), no condition. Useful for generating grids/combinations; dangerous by accident.
```sql
SELECT c.name, t.tier FROM customers c CROSS JOIN (VALUES ('gold'),('silver')) AS t(tier);
```

**SELF JOIN** — a table joined to itself (with two aliases), e.g. employees → managers.
```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

### 3.3 Venn intuition (and its limits)
The classic Venn diagram (INNER = intersection, LEFT = left circle, FULL = union) is a useful first picture, but it breaks down with duplicate keys (fan-out) — joins multiply matching rows, they don't deduplicate. Think in terms of *matching rows*, not sets of keys.

### 3.4 Semi-join and anti-join (EXISTS / NOT EXISTS)
- **Semi-join** = "rows in A that have *at least one* match in B" — but you don't want B's columns or duplicates.
```sql
-- customers who have placed at least one order
SELECT c.* FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```
- **Anti-join** = "rows in A with *no* match in B."
```sql
-- customers with no orders
SELECT c.* FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);  -- Cleo
```
`NOT EXISTS` is preferred over `NOT IN` because it is **NULL-safe** (recall Lesson 02's `NOT IN`+NULL trap) and often plans better.

### 3.5 The fan-out / duplication trap
If you join to a table with multiple matches and then aggregate, counts/sums inflate:
```sql
-- WRONG if a customer has multiple orders AND multiple addresses:
-- joining both multiplies rows → sums double-count.
```
Fix: aggregate each side first (subquery/CTE) before joining, or be deliberate about grain.

### 3.6 ON vs WHERE in outer joins (subtle but important)
In a LEFT JOIN, a condition in `ON` filters *what counts as a match*; the same condition in `WHERE` filters the final result and can silently turn your outer join into an inner join.
```sql
-- keeps Cleo, but only shows her if she had a big order (NULL for her)
... LEFT JOIN orders o ON o.customer_id = c.id AND o.amount > 100

-- DROPS Cleo (her NULL amount fails the WHERE) → effectively INNER
... LEFT JOIN orders o ON o.customer_id = c.id WHERE o.amount > 100
```

---

## 4. Official documentation quotes

> "A query that accesses multiple rows of the same or different tables at one time is called a join query. … `JOIN` types: `INNER JOIN`, `LEFT OUTER JOIN`, `RIGHT OUTER JOIN`, `FULL OUTER JOIN`, `CROSS JOIN`."
> — *PostgreSQL docs*, [2.6. Joins Between Tables](https://www.postgresql.org/docs/current/tutorial-join.html) and [7.2.1.1. Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html)

> "For each row of T1 that does not satisfy the join condition with any row in T2, a joined row is added with null values in columns of T2. … This is the converse of an inner join."
> — *PostgreSQL docs*, [7.2.1.1. Joined Tables (LEFT OUTER JOIN)](https://www.postgresql.org/docs/current/queries-table-expressions.html)

> "The subquery is evaluated to determine whether it returns any rows. If it returns at least one row, the result of `EXISTS` is 'true'."
> — *PostgreSQL docs*, [9.24. Subquery Expressions (EXISTS)](https://www.postgresql.org/docs/current/functions-subquery.html)

---

## 5. Real-world examples

### 5.1 Customers with their order totals (keep customers with zero)
```sql
SELECT c.name, COALESCE(SUM(o.amount), 0) AS total_spent
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.name
ORDER BY total_spent DESC;
-- Ann 125.00, Bob 20.00, Cleo 0.00
```

### 5.2 Multi-table join
```sql
SELECT o.id, c.name, p.name AS product, oi.qty
FROM orders o
JOIN customers   c  ON c.id = o.customer_id
JOIN order_items oi ON oi.order_id = o.id
JOIN products    p  ON p.id = oi.product_id
WHERE o.created_at >= now() - interval '30 days';
```

### 5.3 Find the gap: customers who never ordered (anti-join)
```sql
SELECT c.name
FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

---

## 6. How This Is Used In Production
- **Startups:** joins assemble the data behind every screen — an order page joins orders + customers + line items + products. ORMs generate these, but you must read them when they go wrong (N+1 queries, accidental cross joins).
- **Enterprises:** analytics/reporting joins fact and dimension tables (star schema) across millions of rows; join *strategy* (hash join vs merge join vs nested loop — chosen by the planner) determines whether a dashboard returns in 200 ms or 2 minutes. Data engineers tune these constantly.
- **Common architectures:** OLTP normalized tables joined for transactions; data warehouses pre-join/denormalize into wide tables or star schemas so analysts avoid expensive runtime joins. ETL jobs (Lesson 15) often *materialize* join results.
- **Scaling:** join order and indexes on join keys (foreign keys) are the biggest levers; the planner picks a join algorithm based on table sizes and statistics (Lesson 09/12). Big joins across shards are expensive — a key reason warehouses denormalize.
- **Monitoring/Logging:** `EXPLAIN ANALYZE` shows which join algorithm ran and where time went; slow joins on unindexed FKs are a top finding in `pg_stat_statements`.
- **Security:** row-level security policies interact with joins; least-privilege views often pre-define safe joins for analysts.
- **Real examples:** **Amazon**/**Uber** order systems join customer, order, payment, and inventory tables transactionally; **Netflix**/**Spotify** analytics join huge event-fact tables to dimension tables in warehouses (often pre-denormalized to avoid runtime join cost). The anti-join "who hasn't done X" pattern powers churn analysis everywhere.

---

## 7. Best practices
- Always qualify columns with table aliases (`c.id`, `o.amount`) in multi-table queries.
- Use explicit `JOIN ... ON`, never the old comma-join `FROM a, b WHERE a.x=b.y` (error-prone, hides cross products).
- Default to `INNER JOIN`; use `LEFT JOIN` deliberately when you must keep unmatched left rows.
- Use `NOT EXISTS` for anti-joins (NULL-safe) over `NOT IN`.
- When aggregating after joins, watch the **grain** — aggregate per side first if fan-out would double-count.
- Index foreign-key/join columns (Lesson 09).

## 8. Common mistakes & gotchas
- **Accidental cross join** from a missing/incorrect `ON` → row explosion.
- Putting a right-table filter in `WHERE` of a LEFT JOIN, silently making it INNER (put it in `ON` instead).
- **Fan-out double counting** when joining two one-to-many tables then summing.
- Forgetting `COALESCE` on outer-join NULLs (e.g., totals showing NULL instead of 0).
- `NOT IN` with a NULL-producing subquery returning empty results.
- Ambiguous column references without aliases.

## 9. Where AI helps (and where it hurts)
- **Helps:** translating "show me X with their Y, including those with no Y" into the right LEFT JOIN, spotting fan-out, converting `IN` to `EXISTS`, and explaining an `EXPLAIN` plan's join nodes.
- **Hurts:** AI may default to INNER when you needed LEFT (dropping rows), reproduce the `NOT IN`/NULL bug, or write fan-out-prone aggregations. Always check row counts against a known expectation.

## 10. Learn independently
- [PostgreSQL Joins tutorial](https://www.postgresql.org/docs/current/tutorial-join.html) and [Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html).
- [PGExercises — Joins and subqueries](https://pgexercises.com/questions/joins/).
- Markus Winand, *Use The Index, Luke* — joins & indexing.
- Practice self-joins and anti-joins specifically; they're the interview favorites.

## 11. Interview preparation

**Q1. Difference between INNER and LEFT JOIN?**
> INNER returns only rows with a match in both tables. LEFT returns all rows from the left table, filling right-side columns with NULL where there's no match. Use LEFT to keep "the ones with none."

**Q2. What is a self-join and when do you use one?**
> Joining a table to itself with two aliases, to relate rows within the same table — e.g., employees to their managers, or comparing a row to other rows in the same table.

**Q3. Why prefer `NOT EXISTS` over `NOT IN`?**
> `NOT IN` returns no rows if the subquery yields any NULL (three-valued logic), a silent correctness bug. `NOT EXISTS` is NULL-safe and typically optimizes as an efficient anti-join.

**Q4. You join customers to orders and SUM amounts, but totals look doubled. Why?**
> Fan-out: if you also joined another one-to-many table (e.g., order items or addresses), each order row got multiplied, inflating the sum. Aggregate each one-to-many relationship separately (subquery/CTE) before combining, or be explicit about the grain.

**Q5. In a LEFT JOIN, what's the difference between a condition in `ON` vs `WHERE`?**
> A condition in `ON` affects which right rows are considered a match (unmatched left rows still appear with NULLs). The same condition in `WHERE` is applied after the join and will drop the NULL-filled left rows, effectively turning the LEFT JOIN into an INNER JOIN.

**Q6. What is a CROSS JOIN and a legit use for it?**
> The Cartesian product of two tables (every combination). Legit uses: generating a grid of all combinations (e.g., every product × every region for a coverage report), or pairing a single-row config with each row.

**Q7. Write a query for "customers who placed an order in the last 30 days but have no order in the prior 30."**
> Use a semi-join for recent and an anti-join for prior:
> ```sql
> SELECT c.* FROM customers c
> WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id=c.id AND o.created_at >= now()-interval '30 days')
>   AND NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id=c.id
>                   AND o.created_at >= now()-interval '60 days'
>                   AND o.created_at <  now()-interval '30 days');
> ```

## 12. Homework
> Build the `customers`/`orders` (+ add `order_items`, `products`) schema; save scripts in `sql/solutions/lesson-03/`.

**Easy**
1. Write (a) an INNER JOIN listing each order with its customer name and amount, and (b) a LEFT JOIN listing every customer and their order amount, including customers with none (show NULL).

**Medium**
2. Produce a report of each customer's name and `total_spent` that includes customers who spent nothing (showing `0.00`, not NULL). Then list customers who have never placed an order using `NOT EXISTS`.

**Hard**
3. Add `order_items(order_id, product_id, qty)` and `products(id, name, price)`. Write a query for total revenue per customer computed as `SUM(qty * price)`. Deliberately first write a version that double-counts via fan-out (join orders→items and also join a second one-to-many table), observe the inflated numbers, then fix it by aggregating at the correct grain. Explain the fix in comments. Also write a self-join example using an `employees(id, name, manager_id)` table to list each employee with their manager.

**Stretch:** Reproduce the `ON` vs `WHERE` outer-join pitfall: write a LEFT JOIN with a right-side filter in `WHERE` that accidentally drops unmatched rows, then move the condition into `ON` to fix it, showing both result sets.

## 13. Key takeaways
- Joins combine rows on a condition; think "matching rows," not Venn sets (duplicates fan out).
- INNER = matches only; LEFT = keep all left rows with NULLs for non-matches; FULL = both; CROSS = all combinations.
- Self-joins relate rows within one table (employee→manager).
- Use `EXISTS`/`NOT EXISTS` for semi/anti joins — NULL-safe and efficient.
- Beware accidental cross products, fan-out double counting, and right-side filters in `WHERE` of outer joins.
