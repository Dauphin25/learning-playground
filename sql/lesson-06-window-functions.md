# SQL Lesson 06 — Window Functions

| | |
|---|---|
| **Track** | SQL (PostgreSQL-first) |
| **Lesson** | 06 of 15 |
| **Topic** | `OVER`, `PARTITION BY`, ordering, frames, ranking, offsets, and running aggregates |
| **Estimated time** | ~2 hours |
| **Prerequisites** | Lessons 01–05 (aggregation and CTEs especially) |

### Learning objectives
1. Explain what a window function *is* — and precisely how it differs from `GROUP BY`.
2. Place window functions correctly in the logical evaluation order, and explain why you can't filter on them in `WHERE`.
3. Use the ranking family (`ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE`, `PERCENT_RANK`) and know exactly how each treats ties.
4. Use the offset family (`LAG`, `LEAD`, `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE`) — including the `LAST_VALUE` frame trap.
5. Master **frames** — `ROWS` vs `RANGE` vs `GROUPS`, and the default frame that catches everyone.
6. Solve the classic problems: running totals, moving averages, top-N per group, deduplication, gaps-and-islands, period-over-period, and sessionisation.

---

## 1. Why this matters

Window functions are the single highest-leverage feature in SQL. They let you compute across a set of rows **while keeping every row** — running totals, rankings, comparisons to the previous row, percent-of-total — things that otherwise require self-joins, correlated subqueries, or a round trip to Python.

They are also, bluntly, **the dividing line in data engineering interviews**. "Write a query for the top 3 orders per customer" and "compute a 7-day moving average" are near-universal screening questions, and the expected answer is a window function. Someone who answers with a correlated subquery has revealed their level.

For your target roles:
- **Data engineering:** every fact table gets deduplicated with `ROW_NUMBER() OVER (PARTITION BY key ORDER BY updated_at DESC) = 1`. Every metric gets a period-over-period comparison with `LAG`. Every SCD Type-2 dimension is built with `LEAD` over effective dates. Sessionisation is a window function. This is not an advanced topic in data engineering; it is the daily vocabulary.
- **Backend:** leaderboards, "your rank", pagination with stable ordering, "what changed since last time" — all windows.
- **Analytics:** cohort retention, funnels, RFM segmentation, cumulative revenue. All windows.

If you learn one SQL feature properly, learn this one.

---

## 2. Theory

### 2.1 The core idea

`GROUP BY` **collapses** rows. A window function **doesn't**.

```sql
-- GROUP BY: 3 rows out (one per country)
SELECT country, COUNT(*) FROM customers GROUP BY country;

-- Window: 4 rows out (every customer, each labelled with its country's count)
SELECT name, country, COUNT(*) OVER (PARTITION BY country) AS country_size
FROM customers;
```
```
 name |  country | country_size
------+----------+--------------
 Ann  | DE       |            2
 Bob  | DE       |            2
 Cleo | FR       |            1
 Dan  | US       |            1
```

That's the whole idea: **an aggregate computed over a set of related rows, attached to each row, without collapsing anything.** The `OVER` clause defines "which related rows."

Anatomy:
```sql
function(args) OVER (
    PARTITION BY expr, ...     -- split rows into independent groups (optional)
    ORDER BY expr, ...         -- order within each partition (optional, but see §2.4)
    frame_clause               -- which rows within the partition (optional)
)
```
Omit `PARTITION BY` and the whole result set is one partition. Omit everything (`OVER ()`) and you get a grand aggregate on every row:
```sql
SELECT name, salary,
       salary - AVG(salary) OVER () AS vs_company_avg,
       ROUND(100.0 * salary / SUM(salary) OVER (), 1) AS pct_of_payroll
FROM employees;
```

### 2.2 Where window functions run — and why you can't filter on them

Extend the logical order from Lesson 04:

```
1. FROM / JOIN
2. WHERE
3. GROUP BY
4. HAVING
5. WINDOW FUNCTIONS      <-- HERE. After grouping, before DISTINCT/ORDER BY.
6. SELECT
7. DISTINCT
8. ORDER BY
9. LIMIT
```

This single fact explains the two rules everyone trips over:

**(a) You cannot reference a window function in `WHERE` or `HAVING`** — they run at step 2/4, before windows exist at step 5.
```sql
SELECT name, ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
FROM employees WHERE rn <= 3;
-- ERROR:  column "rn" does not exist
SELECT name FROM employees WHERE ROW_NUMBER() OVER (ORDER BY salary DESC) <= 3;
-- ERROR:  window functions are not allowed in WHERE
```
**The fix is always a subquery or CTE** — compute the window in an inner query, filter in the outer:
```sql
SELECT name, rn FROM (
    SELECT name, ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn FROM employees
) s WHERE rn <= 3;
```
This "compute inside, filter outside" pattern is the most common shape in all of window-function SQL. Internalise it.

**(b) Window functions see the rows that survived `WHERE`/`GROUP BY`.** So `SUM(x) OVER ()` is the total *of the filtered set*, not the table. That's usually what you want, but it means "percent of total" silently means "percent of the filtered total" — a real reporting bug when someone adds a `WHERE`.

**Windows can also aggregate aggregates**, since they run after `GROUP BY`:
```sql
SELECT dept, SUM(salary) AS dept_total,
       ROUND(100.0 * SUM(salary) / SUM(SUM(salary)) OVER (), 1) AS pct_of_payroll
FROM employees GROUP BY dept;
```
```
 dept  | dept_total | pct_of_payroll
-------+------------+----------------
 exec  |  250000.00 |           19.6
 eng   |  690000.00 |           54.2
 sales |  295000.00 |           23.1
```
`SUM(SUM(salary)) OVER ()` looks bizarre and is completely correct: the inner `SUM` aggregates within each dept (step 3), the outer `SUM(...) OVER ()` sums those results across all dept rows (step 5). Being able to read that line is a genuine marker of SQL fluency.

### 2.3 The ranking family — and ties

```sql
SELECT name, dept, salary,
       ROW_NUMBER()  OVER (ORDER BY salary DESC) AS row_num,
       RANK()        OVER (ORDER BY salary DESC) AS rank,
       DENSE_RANK()  OVER (ORDER BY salary DESC) AS dense_rank,
       NTILE(4)      OVER (ORDER BY salary DESC) AS quartile,
       ROUND(PERCENT_RANK() OVER (ORDER BY salary DESC)::numeric, 2) AS pct_rank,
       ROUND(CUME_DIST()    OVER (ORDER BY salary DESC)::numeric, 2) AS cume_dist
FROM employees;
```
Add a tie to see the difference (`UPDATE employees SET salary = 145000 WHERE name = 'Walt'`):
```
 name | dept  |  salary   | row_num | rank | dense_rank | quartile | pct_rank | cume_dist
------+-------+-----------+---------+------+------------+----------+----------+-----------
 Zara | exec  | 250000.00 |       1 |    1 |          1 |        1 |     0.00 |      0.12
 Yuri | eng   | 180000.00 |       2 |    2 |          2 |        1 |     0.14 |      0.25
 Xena | sales | 175000.00 |       3 |    3 |          3 |        2 |     0.29 |      0.38
 Vera | eng   | 145000.00 |       4 |    4 |          4 |        2 |     0.43 |      0.62
 Walt | eng   | 145000.00 |       5 |    4 |          4 |        3 |     0.43 |      0.62
 Sam  | sales | 120000.00 |       6 |    6 |          5 |        3 |     0.71 |      0.75
 Tam  | eng   | 115000.00 |       7 |    7 |          6 |        4 |     0.86 |      0.88
 Ugo  | eng   | 110000.00 |       8 |    8 |          7 |        4 |     1.00 |      1.00
```

| Function | Ties | Gaps after ties | Use for |
|---|---|---|---|
| `ROW_NUMBER()` | **broken arbitrarily** — 4, 5 | n/a | dedup, pagination, "pick one" |
| `RANK()` | same rank — 4, 4 | **yes** — next is 6 | "Olympic" ranking, leaderboards |
| `DENSE_RANK()` | same rank — 4, 4 | **no** — next is 5 | "how many distinct values above me" |
| `NTILE(n)` | split by position | n/a | quartiles, buckets |
| `PERCENT_RANK()` | `(rank-1)/(n-1)` | — | "top 10%" thresholds |
| `CUME_DIST()` | `rows ≤ this / n` | — | cumulative distribution |

**The `ROW_NUMBER` non-determinism trap:** with tied salaries, whether Vera or Walt gets `row_num = 4` is **undefined** — it depends on the physical plan and can change between runs. If you use `ROW_NUMBER() = 1` to deduplicate and the ordering has ties, **you get a non-deterministic result set**: the same query returns different rows on different days, tests flake, and downstream diffs churn. **Always make the `ORDER BY` a total order** by appending a unique tiebreaker (`ORDER BY updated_at DESC, id DESC`). This is one of the most valuable practical points in the lesson and a great senior interview answer.

### 2.4 Frames — the part everyone gets wrong

A frame defines *which rows within the partition* the function sees, relative to the current row.

```sql
{ROWS | RANGE | GROUPS} BETWEEN <start> AND <end>
-- bounds: UNBOUNDED PRECEDING | n PRECEDING | CURRENT ROW | n FOLLOWING | UNBOUNDED FOLLOWING
```

**The default frame — memorise this:**

| `OVER` clause | Implicit frame | Effect |
|---|---|---|
| `OVER ()` | entire partition | grand total |
| `OVER (PARTITION BY x)` | entire partition | partition total |
| `OVER (ORDER BY y)` | **`RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`** | **running total** |
| `OVER (PARTITION BY x ORDER BY y)` | same, per partition | running total per partition |

**Adding `ORDER BY` silently changes an aggregate from "total" to "running total."** This is *the* window-function gotcha:

```sql
SELECT name, salary,
       SUM(salary) OVER ()                    AS total,       -- 1,165,000 on every row
       SUM(salary) OVER (ORDER BY salary)     AS running      -- accumulates!
FROM employees ORDER BY salary;
```
```
 name |  salary   |   total    |  running
------+-----------+------------+------------
 Ugo  | 110000.00 | 1165000.00 |  110000.00
 Tam  | 115000.00 | 1165000.00 |  225000.00
 Sam  | 120000.00 | 1165000.00 |  345000.00
 ...
 Zara | 250000.00 | 1165000.00 | 1165000.00
```
People add `ORDER BY` for readability, get a running total, and ship a broken "percent of total." If you want the total *with* an ordering, be explicit:
```sql
SUM(salary) OVER (ORDER BY salary ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
```

**`ROWS` vs `RANGE` vs `GROUPS`** — the distinction that separates people who've read the docs from people who haven't:

- **`ROWS`** — physical offsets. "The 2 rows before this one." Deterministic and what you almost always mean.
- **`RANGE`** — **logical/value** offsets. "All rows whose ORDER BY value is within this range of mine." Crucially, **`RANGE ... CURRENT ROW` includes all peers** — every row with the *same* ordering value.
- **`GROUPS`** (PG11+) — offsets counted in *peer groups*. "The 2 distinct value-groups before this one."

```sql
-- With Vera and Walt both at 145000:
SELECT name, salary,
       SUM(salary) OVER (ORDER BY salary ROWS  BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS by_rows,
       SUM(salary) OVER (ORDER BY salary RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS by_range
FROM employees ORDER BY salary;
```
```
 name |  salary   |  by_rows  | by_range
------+-----------+-----------+-----------
 Ugo  | 110000.00 | 110000.00 | 110000.00
 Tam  | 115000.00 | 225000.00 | 225000.00
 Sam  | 120000.00 | 345000.00 | 345000.00
 Vera | 145000.00 | 490000.00 | 635000.00   <-- RANGE already includes Walt!
 Walt | 145000.00 | 635000.00 | 635000.00
 ...
```
`RANGE` gives both tied rows the *same* running total (it includes all peers); `ROWS` gives them different ones. Since `RANGE` is the **default**, your running total over a non-unique ordering column jumps at ties. If you want a strict row-by-row accumulation, **you must say `ROWS`**. Most "my running total looks wrong" bugs are this.

`RANGE` with a numeric/interval offset is genuinely powerful, though — it's how you do time-based windows:
```sql
-- Revenue in the trailing 7 DAYS (calendar), not the trailing 7 ROWS:
SUM(amount) OVER (ORDER BY day RANGE BETWEEN '6 days' PRECEDING AND CURRENT ROW)
```
That handles gaps correctly (missing days don't shift the window), which `ROWS BETWEEN 6 PRECEDING` does not. **This is the correct way to do a 7-day moving average on data with gaps**, and knowing it is a strong signal.

### 2.5 The offset family

```sql
SELECT name, salary,
       LAG(salary)        OVER (ORDER BY salary) AS prev,
       LAG(salary, 1, 0)  OVER (ORDER BY salary) AS prev_or_zero,   -- default for the 1st row
       LEAD(salary)       OVER (ORDER BY salary) AS next,
       salary - LAG(salary) OVER (ORDER BY salary) AS delta,
       FIRST_VALUE(name)  OVER (PARTITION BY dept ORDER BY salary DESC) AS dept_top_earner
FROM employees;
```

`LAG`/`LEAD` are the "compare to the previous/next row" tools — period-over-period growth, gap detection, state-change detection. They take `(expr, offset, default)`; without a default, the first row's `LAG` is NULL.

**The `LAST_VALUE` trap** — this catches literally everyone:
```sql
SELECT name, dept, salary,
       FIRST_VALUE(name) OVER (PARTITION BY dept ORDER BY salary DESC) AS top,
       LAST_VALUE(name)  OVER (PARTITION BY dept ORDER BY salary DESC) AS bottom_wrong
FROM employees;
```
```
 name | dept  |  salary   | top  | bottom_wrong
------+-------+-----------+------+--------------
 Zara | exec  | 250000.00 | Zara | Zara
 Yuri | eng   | 180000.00 | Yuri | Yuri          <-- ???
 Vera | eng   | 145000.00 | Yuri | Vera          <-- ???
 Walt | eng   | 145000.00 | Yuri | Walt
 Tam  | eng   | 115000.00 | Yuri | Tam
 Ugo  | eng   | 110000.00 | Yuri | Ugo
```
`bottom_wrong` returns the **current row** every time. Why? The default frame is `RANGE ... UNBOUNDED PRECEDING AND CURRENT ROW` — the frame *ends at the current row*, so "the last value in the frame" **is** the current row. `FIRST_VALUE` works by luck: the frame starts at `UNBOUNDED PRECEDING`, which is genuinely the first row.

The fix is an explicit frame:
```sql
LAST_VALUE(name) OVER (PARTITION BY dept ORDER BY salary DESC
                       ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS bottom
```
Or just flip the sort and use `FIRST_VALUE`. **`LAST_VALUE` without an explicit frame is always a bug** — that's a rule you can state flatly, and it's a favourite interview question because it looks like it should just work.

### 2.6 `WINDOW` clause — naming windows

When several functions share a window, name it once:
```sql
SELECT name, dept, salary,
       RANK()       OVER w AS rank,
       LAG(salary)  OVER w AS prev,
       SUM(salary)  OVER (w ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running
FROM employees
WINDOW w AS (PARTITION BY dept ORDER BY salary DESC)
ORDER BY dept, salary DESC;
```
Less repetition, fewer copy-paste errors, and it makes the *intent* ("everything is per-dept by salary") explicit. It also lets the planner reuse a single sort. Underused; use it.

### 2.7 The classic patterns

**Top-N per group** (the #1 interview question):
```sql
SELECT * FROM (
    SELECT c.name, o.id, o.created_at,
           ROW_NUMBER() OVER (PARTITION BY c.id ORDER BY o.created_at DESC, o.id DESC) AS rn
    FROM customers c JOIN orders o ON o.customer_id = c.id
) s WHERE rn <= 3;
```
Note the `, o.id DESC` tiebreaker making the order total. (And recall Lesson 05: with a good index and small N, `LATERAL` beats this.)

**Deduplication — keep the latest version of each key:**
```sql
DELETE FROM events e USING (
    SELECT id, ROW_NUMBER() OVER (PARTITION BY event_key ORDER BY updated_at DESC, id DESC) AS rn
    FROM events
) d
WHERE e.id = d.id AND d.rn > 1;
```
This is *the* data-engineering idiom. Every CDC pipeline, every "we got duplicate webhooks" cleanup, every SCD build. `ROW_NUMBER() OVER (PARTITION BY natural_key ORDER BY version DESC) = 1` is the shape.

**Running total & moving average:**
```sql
SELECT day, revenue,
       SUM(revenue) OVER (ORDER BY day ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cumulative,
       ROUND(AVG(revenue) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW), 2) AS ma7_rows,
       ROUND(AVG(revenue) OVER (ORDER BY day RANGE BETWEEN '6 days' PRECEDING AND CURRENT ROW), 2) AS ma7_days
FROM daily_revenue;
```
`ma7_rows` and `ma7_days` differ the moment a day is missing. `RANGE` with an interval is correct; `ROWS` silently averages over 7 *present* days, which might span three weeks.

**Period-over-period:**
```sql
SELECT month, revenue,
       LAG(revenue) OVER (ORDER BY month) AS prev_month,
       ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY month))
             / NULLIF(LAG(revenue) OVER (ORDER BY month), 0), 1) AS mom_pct
FROM monthly_revenue;
```

**Gaps and islands** — find consecutive runs. The classic trick:
```sql
-- Which dates did each customer order on consecutive days?
WITH days AS (
    SELECT DISTINCT customer_id, created_at::date AS d FROM orders
),
grouped AS (
    SELECT customer_id, d,
           d - (ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY d))::int AS grp
    FROM days
)
SELECT customer_id, MIN(d) AS streak_start, MAX(d) AS streak_end, COUNT(*) AS streak_len
FROM grouped GROUP BY customer_id, grp
HAVING COUNT(*) > 1
ORDER BY streak_len DESC;
```
The trick: for consecutive dates, `date - row_number` is **constant**, so it becomes a group key. Elegant, non-obvious, and a genuine "aha" — it comes up in interviews as "find consecutive login streaks."

**Sessionisation** — group events into sessions with a 30-minute inactivity gap:
```sql
WITH gaps AS (
    SELECT user_id, ts,
           CASE WHEN ts - LAG(ts) OVER (PARTITION BY user_id ORDER BY ts) > interval '30 min'
                  OR LAG(ts) OVER (PARTITION BY user_id ORDER BY ts) IS NULL
                THEN 1 ELSE 0 END AS is_new_session
    FROM events
),
sessions AS (
    SELECT user_id, ts,
           SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY ts
                                     ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS session_id
    FROM gaps
)
SELECT user_id, session_id, MIN(ts) AS started, MAX(ts) AS ended, COUNT(*) AS events
FROM sessions GROUP BY user_id, session_id;
```
Two windows: `LAG` to flag session starts, then a **running `SUM` of the flag** to turn flags into a session number. That "cumulative sum of a boolean flag as a group id" pattern is worth memorising — it's how every product-analytics tool defines a session, and it generalises to any "start a new group when X happens."

---

## 3. Official documentation quotes

> "A window function performs a calculation across a set of table rows that are somehow related to the current row. This is comparable to the type of calculation that can be done with an aggregate function. **But unlike regular aggregate functions, use of a window function does not cause rows to become grouped into a single output row — the rows retain their separate identities.** Behind the scenes, the window function is able to access more than just the current row of the query result."
> — *PostgreSQL Documentation*, [3.5. Window Functions](https://www.postgresql.org/docs/current/tutorial-window.html)

> "Window functions are permitted only in the `SELECT` list and the `ORDER BY` clause of the query. They are forbidden elsewhere, such as in `GROUP BY`, `HAVING` and `WHERE` clauses. This is because they logically execute after the processing of those clauses. Also, window functions execute after non-window aggregate functions. … **If there is a need to filter or group rows after the window calculations are performed, you can use a sub-select.**"
> — *PostgreSQL Documentation*, [3.5. Window Functions](https://www.postgresql.org/docs/current/tutorial-window.html)

> "The default framing option is `RANGE UNBOUNDED PRECEDING`, which is the same as `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. With `ORDER BY`, this sets the frame to be all rows from the partition start up through the current row's last **peer** … Without `ORDER BY`, this means all rows of the partition are included in the window frame, since all rows become peers of the current row."
> — *PostgreSQL Documentation*, [4.2.8. Window Function Calls](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS)

> "In `RANGE` mode, these options require that the `ORDER BY` clause specify exactly one column. … `GROUPS` mode … the offset again must yield a non-null, non-negative integer, and the option means that the frame starts or ends the specified number of peer groups before or after the current row's peer group."
> — *PostgreSQL Documentation*, [4.2.8. Window Function Calls](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS)

> "`rank()` — returns the rank of the current row, with gaps; that is, the `row_number` of the first row in its peer group. `dense_rank()` — returns the rank of the current row, without gaps; this function effectively counts peer groups."
> — *PostgreSQL Documentation*, [9.22. Window Functions](https://www.postgresql.org/docs/current/functions-window.html)

---

## 4. Real-world examples

### 4.1 Deduplication — the data engineer's most-used query

You receive a CDC feed with duplicate rows per key. Keep the latest.

```sql
CREATE TEMP TABLE cdc_events (
    id serial PRIMARY KEY, event_key text, payload text, updated_at timestamptz
);
INSERT INTO cdc_events (event_key, payload, updated_at) VALUES
    ('user:1', 'v1', '2026-07-01 10:00+00'),
    ('user:1', 'v2', '2026-07-01 11:00+00'),
    ('user:1', 'v3', '2026-07-01 12:00+00'),
    ('user:2', 'a',  '2026-07-01 09:00+00'),
    ('user:3', 'x',  '2026-07-01 08:00+00'),
    ('user:3', 'y',  '2026-07-01 08:00+00');   -- <-- TIE on updated_at. Deliberate.

SELECT event_key, payload, updated_at, rn FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY event_key
                                 ORDER BY updated_at DESC, id DESC) AS rn   -- tiebreaker!
    FROM cdc_events
) s WHERE rn = 1;
```
```
 event_key | payload |       updated_at       | rn
-----------+---------+------------------------+----
 user:1    | v3      | 2026-07-01 12:00:00+00 |  1
 user:2    | a       | 2026-07-01 09:00:00+00 |  1
 user:3    | y       | 2026-07-01 08:00:00+00 |  1
```

Without `, id DESC`, `user:3` returns `x` or `y` **non-deterministically** — it depends on the plan, so it can flip when statistics change or when you add a parallel worker. Your incremental model then emits a spurious change every run, downstream diffs churn, and you spend a day chasing a "bug" that's actually a missing tiebreaker. **The `ORDER BY` inside `ROW_NUMBER` must be a total order.** Say this in an interview and you're clearly senior.

Postgres also offers `DISTINCT ON`, which is a non-standard shortcut for exactly this:
```sql
SELECT DISTINCT ON (event_key) event_key, payload, updated_at
FROM cdc_events ORDER BY event_key, updated_at DESC, id DESC;
```
Shorter and often faster (one sort, no subquery), but Postgres-only. Know both; `ROW_NUMBER` ports, `DISTINCT ON` doesn't.

### 4.2 Revenue dashboard: running total, MoM, share, rank

```sql
WITH monthly AS (
    SELECT date_trunc('month', o.created_at)::date AS month,
           SUM(oi.quantity * oi.unit_price)        AS revenue
    FROM orders o JOIN order_items oi ON oi.order_id = o.id
    WHERE o.status = 'paid'
    GROUP BY 1
)
SELECT month,
       revenue,
       SUM(revenue) OVER w_cum                                   AS cumulative,
       LAG(revenue) OVER w_ord                                   AS prev_month,
       ROUND(100.0 * (revenue - LAG(revenue) OVER w_ord)
             / NULLIF(LAG(revenue) OVER w_ord, 0), 1)            AS mom_pct,
       ROUND(100.0 * revenue / SUM(revenue) OVER (), 1)          AS pct_of_total,
       ROUND(AVG(revenue) OVER (ORDER BY month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2)
                                                                 AS ma3,
       RANK() OVER (ORDER BY revenue DESC)                       AS rank
FROM monthly
WINDOW w_ord AS (ORDER BY month),
       w_cum AS (ORDER BY month ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
ORDER BY month;
```
```
   month    | revenue | cumulative | prev_month | mom_pct | pct_of_total |   ma3  | rank
------------+---------+------------+------------+---------+--------------+--------+------
 2026-01-01 |  118.98 |     118.98 |            |         |         24.4 | 118.98 |    3
 2026-02-01 |  222.50 |     341.48 |     118.98 |    87.0 |         45.6 | 170.74 |    1
 2026-07-01 |   73.50 |     414.98 |     222.50 |   -67.0 |         15.1 | 138.33 |    4
```

Six window calculations, one pass, no self-joins. The `WINDOW` clause names the two shared window definitions. `NULLIF` guards the MoM division. Note `pct_of_total` uses `OVER ()` — the total of the *filtered* set (paid orders only), which is correct here but is exactly the thing that silently changes meaning when someone adds a `WHERE`.

### 4.3 Everyone above their department average — three ways

```sql
-- Window (Lesson 06):
SELECT name, dept, salary, dept_avg FROM (
    SELECT name, dept, salary, ROUND(AVG(salary) OVER (PARTITION BY dept), 2) AS dept_avg
    FROM employees
) s WHERE salary > dept_avg;

-- Correlated subquery (Lesson 05) — same answer, N executions of the inner AVG:
SELECT e.name, e.dept, e.salary FROM employees e
WHERE e.salary > (SELECT AVG(salary) FROM employees WHERE dept = e.dept);

-- Self-join to a grouped derived table:
SELECT e.name, e.dept, e.salary FROM employees e
JOIN (SELECT dept, AVG(salary) a FROM employees GROUP BY dept) d
  ON d.dept = e.dept AND e.salary > d.a;
```
```
 name | dept  |  salary   | dept_avg
------+-------+-----------+-----------
 Zara | exec  | 250000.00 | 250000.00   <-- wait, 250000 > 250000 is false...
```
(Zara is the only exec, so she equals her dept average and is correctly excluded — a good reminder to test the single-row-partition case.)

The window version scans once and computes every partition's average in the same pass. The correlated version conceptually re-runs the aggregate per row (though Postgres often decorrelates it into the third form). The self-join is what the planner probably does anyway. On a large table the window usually wins because it's one sort + one pass rather than a hash aggregate + hash join — but *check the plan*, because with a good index the join can win. The honest interview answer is "window, for clarity; then measure."

### 4.4 Reading a window plan

```sql
EXPLAIN (ANALYZE, COSTS OFF)
SELECT name, dept, RANK() OVER (PARTITION BY dept ORDER BY salary DESC) FROM employees;
```
```
 WindowAgg (actual time=0.045..0.052 rows=8 loops=1)
   Run Condition: (rank() OVER (?) <= 3)          -- only if you filtered on it (PG15+)
   ->  Sort (actual time=0.038..0.039 rows=8 loops=1)
         Sort Key: dept, salary DESC
         Sort Method: quicksort  Memory: 25kB
         ->  Seq Scan on employees (actual rows=8 loops=1)
 Execution Time: 0.081 ms
```

Two things to note. **Every window needs its input sorted** by `PARTITION BY` then `ORDER BY` — that `Sort` node is the cost. An index on `(dept, salary DESC)` can supply the order and eliminate the sort entirely, which is the main window-function optimisation available to you. And multiple windows with **different** `OVER` clauses need **multiple sorts** — you'll see stacked `WindowAgg` nodes, each with its own `Sort`. Reusing one window definition (via the `WINDOW` clause) lets them share a single sort. That's a concrete, measurable reason to name your windows.

`Run Condition` (PG15+) is a genuinely important optimisation: when you filter `WHERE rn <= 3` outside, Postgres can push that *into* the `WindowAgg` and stop producing rows for a partition once the rank exceeds 3 — turning a full sort-everything into an early-abort. It only works for the monotonic ranking functions (`row_number`, `rank`, `dense_rank`, `count`), which is a nice illustration of why those functions are special.

---

## 5. How This Is Used In Production

- **Startups:** window functions replace the "pull it all into Python and loop" instinct, which is the single biggest performance mistake application developers make with databases. A leaderboard, a "your rank" badge, a running balance, "what changed since yesterday" — each is one query instead of a fetch-and-loop that's 100× slower over a network.
- **Enterprises:** the dedup pattern (`ROW_NUMBER() OVER (PARTITION BY key ORDER BY version DESC) = 1`) is load-bearing infrastructure. Every CDC pipeline lands duplicate/out-of-order rows and every one of them is deduplicated with this exact query — dbt ships it as a macro, Snowflake/BigQuery/Databricks all document it as *the* pattern. **SCD Type-2** dimensions (tracking history with `valid_from`/`valid_to`) are built with `LEAD(effective_date) OVER (PARTITION BY key ORDER BY effective_date)` to close each row's validity interval. Product analytics (Amplitude, Mixpanel, and every in-house clone) define sessions and funnels with exactly the `LAG` + cumulative-sum pattern in §2.7.
- **Common architectures:** the **medallion/staging→mart** pipeline runs dedup at the staging boundary — raw lands append-only and immutable, staging deduplicates with a window, marts aggregate. This is why append-only ingestion is safe: correctness is restored at read/transform time by a window function. **Incremental models** use `LAG`/watermarks to process only new data. **Streaming systems** (Flink, Kafka Streams, Materialize) implement over-windows with the same semantics — a Flink `OVER` window *is* this lesson, which makes it a direct bridge to Lesson 15.
- **Scaling:** the cost is the **sort**. `PARTITION BY a ORDER BY b` needs input sorted by `(a, b)`; a matching index eliminates it, which is often a 5–10× win and is the single highest-leverage window optimisation. If the sort spills (`Sort Method: external merge  Disk: 500MB` in `EXPLAIN ANALYZE`), you're paying disk I/O — raise `work_mem` for that query or reduce the input first. Also: **push the `WHERE` inside** the subquery, because a window over 500M rows filtered to 1,000 afterwards did 500M rows of work; filter first, window second. PG15's `Run Condition` helps for ranking filters but not for everything.
- **Monitoring:** `pg_stat_statements` will show your window queries by total time. Look for `WindowAgg` over huge row counts in plans, and for external-merge sorts. Time-series monitoring itself is windowing: Prometheus's `rate()` is `(value - LAG(value)) / (ts - LAG(ts))`, and `avg_over_time` is a moving average with a `RANGE` frame. Recognising that PromQL is a window-function DSL makes both easier.
- **Security:** window functions leak information through *ranking* — publishing "you are ranked 47 of 1,200" plus a filterable dimension lets a user binary-search the underlying values of others. The same *k*-anonymity concern from Lesson 04 applies: suppress small partitions. `NTILE` on salary data with a small partition is effectively publishing individual salaries. Also, `PERCENT_RANK` over a partition of size 2 tells one person the other's value exactly.
- **CI/CD:** the **non-determinism** point matters here more than anywhere. A `ROW_NUMBER` dedup without a total-order tiebreaker produces different output across runs, which makes snapshot tests flake, makes incremental models emit spurious changes, and makes `git diff` on materialised output churn. dbt's `unique` test on the deduplicated key is a real assertion that your window worked. Add a test that runs the dedup twice and asserts identical output — it catches missing tiebreakers immediately.
- **Common tools:** `dbt` (its `deduplicate` macro is this query), `DISTINCT ON` (Postgres's shortcut), `pg_stat_statements`, `explain.dalibo.com`. Every warehouse (Snowflake, BigQuery, Redshift, ClickHouse, DuckDB) implements the same SQL:2003 window spec, so this knowledge is 100% portable — which is unusual and worth noting.
- **Real examples:** **Netflix** and **Uber** run dedup-by-window at the ingestion boundary of essentially every dataset, because at-least-once delivery guarantees mean duplicates are *expected*, not exceptional. **Stripe**-style ledgers use running-total windows for balance reconstruction (the balance after each transaction is `SUM(amount) OVER (PARTITION BY account ORDER BY ts ROWS UNBOUNDED PRECEDING)`), which is elegant but is exactly why they also snapshot balances — recomputing from the beginning of time gets slower every day. That trade-off (recompute vs. snapshot) is the same one from Lessons 04 and 05.

---

## 6. Advanced corner

**How Postgres executes a window.** The `WindowAgg` node consumes **sorted** input and maintains a frame as it walks. For frames ending at `CURRENT ROW` with an aggregate that has an *inverse transition function* (`SUM`, `COUNT`, `AVG` — you can subtract), Postgres uses **incremental aggregation**: as the frame slides, it adds the entering row and subtracts the leaving one, giving O(n) total. For aggregates with no inverse (`MAX`, `MIN`, `string_agg`), it must **recompute the whole frame** at each row, which is O(n × frame_size) — so `MAX(x) OVER (ORDER BY t ROWS BETWEEN 1000 PRECEDING AND CURRENT ROW)` is quadratic-ish and genuinely slow, while `SUM` over the same frame is linear. Check `pg_aggregate.aggminvtransfn` to see which aggregates have one. This is a real, non-obvious performance cliff and an excellent senior answer.

**`Run Condition` (PG15+).** When you write `WHERE rn <= 3` outside a subquery, the planner can push the predicate into the `WindowAgg` for **monotonic** window functions — `row_number`, `rank`, `dense_rank`, and `count` — because it knows the value only increases within a partition, so once it exceeds 3 it can stop emitting rows for that partition. This turns "sort everything, rank everything, throw away 99%" into an early abort. It does **not** apply to `SUM`, `LAG`, `NTILE`, or `percent_rank` (non-monotonic or requiring the full partition). Before PG15 there was no such optimisation, which is why `LATERAL` beat window top-N so decisively — and it's why the honest answer to "LATERAL or window?" now includes "which version are you on?"

**Sorts are the cost, and they compose badly.** Each distinct `OVER` clause needs its own sort ordering. Three windows with three different `PARTITION BY`s = three `WindowAgg` nodes with three `Sort`s. Postgres is smart enough to reuse a sort when one ordering is a prefix of another (`PARTITION BY a` and `PARTITION BY a ORDER BY b` share the `a` sort), and it will order the `WindowAgg` nodes to maximise reuse — but arbitrary different partitions cannot share. Naming windows with the `WINDOW` clause makes the sharing explicit and obvious to reviewers.

**`RANGE` requires exactly one `ORDER BY` column** when you use an offset (`RANGE BETWEEN '7 days' PRECEDING`), because "within 7 of the current value" is meaningless over a composite key. `ROWS` has no such restriction. This is why time-based moving windows can't have a tiebreaker in the `ORDER BY` — a real constraint that occasionally forces you into `ROWS` plus a pre-deduplication step.

**Peers, and why `RANGE` is the default.** Two rows are **peers** if they have equal `ORDER BY` values. `RANGE ... CURRENT ROW` includes all peers of the current row; `ROWS ... CURRENT ROW` stops at the physical current row. The standard chose `RANGE` as the default because it's **deterministic**: with `RANGE`, tied rows all get the same answer regardless of physical order, whereas `ROWS` gives them different answers based on an arbitrary order. So the default is defensible — it's protecting you from non-determinism — but it's also the cause of the "my running total jumps at ties" confusion. Understanding *why* the default is what it is turns a gotcha into a design insight.

**`GROUPS` mode (PG11+)** counts frame bounds in peer groups rather than rows: `GROUPS BETWEEN 1 PRECEDING AND CURRENT ROW` means "this value-group and the previous distinct value-group." It's rarely needed but it's the right tool for "compare against the previous distinct price point" over data with duplicates. Almost nobody knows it exists.

**Window functions are SQL:2003 and universally portable.** Postgres, MySQL 8+, SQL Server, Oracle, SQLite 3.25+, and every warehouse implement the same spec. `DISTINCT ON` is Postgres-only; `QUALIFY` (filter on a window without a subquery — genuinely nicer syntax) is Snowflake/BigQuery/Databricks/DuckDB but **not** Postgres. Knowing which is which matters when you write dbt models targeting multiple warehouses.

**What most people get wrong at this level:**
- Adding `ORDER BY` to an aggregate window and silently getting a running total instead of a total.
- `LAST_VALUE` without an explicit frame. Universal.
- `ROW_NUMBER` dedup with a non-total `ORDER BY` — non-deterministic output, flaky tests, churning diffs.
- Not knowing `ROWS` vs `RANGE`, and so writing moving averages that break on gaps.
- Trying to filter on a window function in `WHERE` and not knowing the subquery is mandatory.
- Windowing 500M rows and filtering afterwards, instead of filtering first.
- Not realising `MAX` over a sliding frame is a different complexity class from `SUM`.
- Reaching for a window when `LATERAL` + an index is 10× faster (or vice versa) without checking the plan.

---

## 7. Best practices

- **Always make `ORDER BY` inside `ROW_NUMBER`/`RANK` a total order** — append a unique tiebreaker (`ORDER BY updated_at DESC, id DESC`). Non-determinism here is a real, expensive, hard-to-diagnose bug class.
- **Always give `LAST_VALUE` an explicit frame** (`ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`), or flip the sort and use `FIRST_VALUE`. Treat `LAST_VALUE` without a frame as a syntax error you happen to be allowed to write.
- **Say `ROWS` when you mean rows.** The default `RANGE` includes peers and will surprise you the first time your ordering column has duplicates.
- **Use `RANGE` with an interval for time-based windows** (`RANGE BETWEEN '6 days' PRECEDING AND CURRENT ROW`) — it's the only correct way to handle gaps in a moving average.
- **Filter before you window.** Push the `WHERE` into the subquery/CTE so the window runs over the smallest possible set.
- **Name shared windows with the `WINDOW` clause.** Less repetition, explicit intent, and it lets the planner reuse one sort.
- **Index for the window**: `(partition_cols, order_cols)` can eliminate the `Sort` node entirely. Check `EXPLAIN` for `Sort` above your `Seq Scan`.
- **Filter windows in an outer query** — CTE or subquery. There's no other option in Postgres (no `QUALIFY`).
- Prefer `ROW_NUMBER` for "pick exactly one", `RANK` for leaderboards where ties should tie, `DENSE_RANK` for "how many distinct values beat me."
- Guard every `LAG`-based percentage with `NULLIF(..., 0)`, and consider `LAG(x, 1, 0)`'s default for the first row.
- Use `DISTINCT ON` when you're Postgres-only and want the dedup to be short and fast; use `ROW_NUMBER` when portability matters.
- Add a **determinism test**: run your dedup twice, assert identical output. It catches missing tiebreakers instantly.

## 8. Common mistakes & gotchas

**`ORDER BY` turns a total into a running total**
```sql
SUM(x) OVER ()                 -- grand total
SUM(x) OVER (ORDER BY d)       -- RUNNING total. Default frame = UNBOUNDED PRECEDING..CURRENT ROW.
```

**`LAST_VALUE` returns the current row** (§2.5). Always.

**Window function in `WHERE`**
```sql
WHERE ROW_NUMBER() OVER (...) <= 3   -- ERROR: window functions are not allowed in WHERE
```
Wrap in a subquery. There's no `QUALIFY` in Postgres.

**Non-deterministic `ROW_NUMBER`** with tied ordering values — different results per run.

**`ROWS` vs `RANGE` on a moving average with gaps**
```sql
AVG(x) OVER (ORDER BY day ROWS  BETWEEN 6 PRECEDING AND CURRENT ROW)   -- 7 PRESENT rows
AVG(x) OVER (ORDER BY day RANGE BETWEEN '6 days' PRECEDING AND CURRENT ROW)  -- 7 CALENDAR days
```
If February 3rd is missing, the first averages Jan 28–Feb 4 and calls it "7 days."

**`RANGE` with an offset needs exactly one `ORDER BY` column** — `ERROR: RANGE with offset PRECEDING/FOLLOWING requires exactly one ORDER BY column`.

**Percent-of-total silently means percent-of-filtered-total** — `SUM(x) OVER ()` sees post-`WHERE` rows.

**`RANK` vs `ROW_NUMBER` confusion** — using `RANK() = 1` to dedup returns *all* tied rows, not one. That's `ROW_NUMBER`'s job.

**Windowing before filtering** — `SELECT * FROM (SELECT *, ROW_NUMBER() OVER (...) FROM huge) WHERE created_at > x` ranks all 500M rows first. Filter inside.

**`MAX`/`MIN` over a sliding frame is not O(n)** — no inverse transition function, so each row recomputes the frame.

**`NTILE` on small partitions** — `NTILE(4)` over 2 rows gives buckets 1 and 2, not a sensible quartile.

## 9. Where AI helps (and where it hurts)

- **Helps:** genuinely good at the classic patterns — ask for "gaps and islands" or "sessionise with a 30-minute gap" and you'll get a correct, idiomatic answer, because these are well-represented and well-discussed. Also good at explaining an unfamiliar `OVER` clause, and at translating "I want a 7-day moving average of X per customer" into the right frame syntax.
- **Hurts:** it writes **`LAST_VALUE` without a frame** routinely. It omits the **tiebreaker** in `ROW_NUMBER` dedup essentially always — the non-determinism is invisible in a toy example and it has no way to know your data has ties. It defaults to `ROWS` for time-based moving averages, which is wrong on gapped data. It suggests `QUALIFY` (which Postgres doesn't have) because Snowflake syntax is in the corpus. And it won't tell you that `LATERAL` would be 10× faster for your top-N, because window functions are the more common answer in its training data. Ask explicitly: *"is this ORDER BY a total order?"*, *"what's the frame?"*, and *"would LATERAL be faster here?"*

## 10. Learn independently

**Books (specific chapters):**
- *SQL for Smarties*, 5th ed. (Celko) — **the OLAP/windowing chapters**. Celko covers `RANK` vs `DENSE_RANK` semantics and the standard's peer/frame model with more rigour than anywhere else, and his gaps-and-islands treatment is the origin of the technique most people know.
- *The Art of PostgreSQL* (Fontaine) — **"Window Functions"** — the best practical chapter on the topic in any Postgres book, with the F1 dataset making every example concrete. Fontaine's framing of windows as "the tool that lets SQL answer questions you'd otherwise leave the database for" is the right mental model.
- *SQL Performance Explained* (Winand) — **Ch. 7 "Partial Results"** — top-N per group, and precisely when an index eliminates the sort a window needs. Winand is *the* authority on window-function performance; his site (below) extends the book.
- *Designing Data-Intensive Applications* (Kleppmann) — **Ch. 11, "Stream Processing"** — the windowing section (tumbling/hopping/sliding/session windows) is the streaming counterpart to this lesson. Read it to see that a Flink session window and §2.7's sessionisation query are the same idea at different latencies. Essential for the Kafka/CDC direction in Lesson 15.
- *PostgreSQL: Up and Running*, 3rd ed. (Obe & Hsu) — **Ch. 5** for the practical `DISTINCT ON` vs window comparison.

**Primary docs:**
- [3.5. Window Functions (tutorial)](https://www.postgresql.org/docs/current/tutorial-window.html) — the gentlest correct introduction anywhere. Read it even though you know the material; the framing is excellent.
- [4.2.8. Window Function Calls](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS) — **the normative frame specification.** This is the page to read carefully; the default-frame paragraph is the one everyone should have memorised.
- [9.22. Window Functions](https://www.postgresql.org/docs/current/functions-window.html) — the function list with exact tie semantics.
- [SELECT — WINDOW clause](https://www.postgresql.org/docs/current/sql-select.html#SQL-WINDOW) and [DISTINCT ON](https://www.postgresql.org/docs/current/sql-select.html#SQL-DISTINCT).

**High-quality links:**
- [use-the-index-luke.com — Top-N Queries](https://use-the-index-luke.com/sql/partial-results/top-n-queries) and the [window-functions section](https://use-the-index-luke.com/sql/partial-results/window-functions) — Winand on indexing for windows.
- [Modern SQL — Window Functions](https://modern-sql.com/feature/over) — the best free reference, with a database-compatibility matrix for every clause (invaluable when you target multiple warehouses).
- [Modern SQL — `RANGE` vs `ROWS`](https://modern-sql.com/feature/over/range-vs-rows) — the clearest explanation of peers that exists.
- [PostgreSQL wiki — window function examples](https://wiki.postgresql.org/wiki/Window_Functions).

## 11. Interview preparation

**Q1. What's the difference between a window function and `GROUP BY`?**
> `GROUP BY` collapses rows — N input rows become one output row per group. A window function computes over a related set of rows but **keeps every row**, attaching the result to each. So `COUNT(*) OVER (PARTITION BY country)` returns every customer labelled with their country's size, while `GROUP BY country` returns one row per country. They also run at different points: windows execute *after* `GROUP BY` and `HAVING`, which is why you can nest them — `SUM(SUM(salary)) OVER ()` sums the per-group sums, and is completely legal.

**Q2. Why can't you filter on a window function in `WHERE`?**
> Because of the logical evaluation order: `FROM → WHERE → GROUP BY → HAVING → **window functions** → SELECT → DISTINCT → ORDER BY → LIMIT`. `WHERE` runs at step 2; windows don't exist until step 5. The fix is always to compute the window in a subquery or CTE and filter in the outer query. Some warehouses (Snowflake, BigQuery, DuckDB) added a `QUALIFY` clause for exactly this, which is nicer syntax; Postgres doesn't have it. What Postgres does have, since PG15, is the `Run Condition` optimisation — it can push an outer `WHERE rn <= 3` *into* the `WindowAgg` for monotonic functions like `row_number`/`rank`, so the subquery isn't the performance penalty it used to be.

**Q3. `ROW_NUMBER` vs `RANK` vs `DENSE_RANK`?**
> They differ only on ties. `ROW_NUMBER` always produces distinct sequential numbers — ties are broken **arbitrarily**. `RANK` gives tied rows the same rank and then **skips** (1, 2, 2, 4) — Olympic-style. `DENSE_RANK` gives tied rows the same rank with **no gap** (1, 2, 2, 3) — it effectively counts distinct peer groups. Use `ROW_NUMBER` to pick exactly one row per group (dedup); `RANK` for leaderboards where ties genuinely tie; `DENSE_RANK` for "how many distinct values are above me." The critical practical point about `ROW_NUMBER`: because it breaks ties arbitrarily, its output is **non-deterministic** unless the `ORDER BY` is a total order — so always append a unique tiebreaker.

**Q4. Explain the default frame, and why it matters.**
> With `ORDER BY` in the `OVER` clause and no explicit frame, the default is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. Two consequences. First, **adding `ORDER BY` silently turns an aggregate from a total into a running total** — `SUM(x) OVER ()` is the grand total but `SUM(x) OVER (ORDER BY d)` accumulates. People add `ORDER BY` for readability and break their percent-of-total. Second, because it's `RANGE` and not `ROWS`, `CURRENT ROW` means "through the current row's **last peer**" — every row with an equal `ORDER BY` value is included. So a running total over a non-unique column jumps at ties, giving tied rows identical (not incrementing) values. If you want strict row-by-row accumulation, you must write `ROWS` explicitly. The default is defensible — `RANGE` is deterministic under ties where `ROWS` isn't — but it's the number one source of confusion.

**Q5. Why does `LAST_VALUE` return the wrong answer?**
> Because the default frame ends at the current row. `LAST_VALUE(x) OVER (ORDER BY y)` asks for the last value *in the frame*, and the frame is `UNBOUNDED PRECEDING .. CURRENT ROW` — so the last value in it **is** the current row, on every row. `FIRST_VALUE` appears to work only by luck, because the frame genuinely starts at the partition's first row. The fix is an explicit frame: `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` — or reverse the `ORDER BY` and use `FIRST_VALUE`. My rule: `LAST_VALUE` without an explicit frame is always a bug.

**Q6. Write a query for the top 3 orders per customer.**
> ```sql
> SELECT * FROM (
>   SELECT o.*, ROW_NUMBER() OVER (PARTITION BY customer_id
>                                  ORDER BY total DESC, id DESC) AS rn
>   FROM orders o
> ) s WHERE rn <= 3;
> ```
> The `, id DESC` makes the ordering total so the result is deterministic under ties. And I'd add: if there's an index on `(customer_id, total DESC)` and the table is large, `LEFT JOIN LATERAL (SELECT ... WHERE customer_id = c.id ORDER BY total DESC LIMIT 3) ON true` is likely much faster — it does ~3 index lookups per customer instead of sorting every order. The window function must materialise and rank *everything* before discarding 99% of it. On PG15+ the `Run Condition` optimisation narrows that gap for `row_number` specifically, but `LATERAL` still usually wins when N is small and the index exists. I'd measure both.

**Q7. `ROWS` vs `RANGE` — when does it matter?**
> `ROWS` counts physical rows; `RANGE` works on *values* of the single `ORDER BY` column. Two places it bites. **Ties:** `RANGE ... CURRENT ROW` includes all peers, so a running total over a duplicated column gives tied rows the same value; `ROWS` increments through them. **Time windows:** `ROWS BETWEEN 6 PRECEDING` is "the last 7 rows that exist," so if three days are missing from your data it silently averages over a 10-day span and calls it a week. `RANGE BETWEEN '6 days' PRECEDING AND CURRENT ROW` is "the last 7 calendar days," which handles gaps correctly. For a moving average over time-series data with any possibility of gaps, `RANGE` with an interval is the correct answer — and it's the version most people don't know exists. The constraint is that `RANGE` with an offset requires exactly one `ORDER BY` column, so you can't have a tiebreaker.

**Q8. (System design) You get duplicate events from an at-least-once CDC feed. How do you build a clean table?**
> Land raw append-only and immutable — never dedup on write, because you'd need read-modify-write and you'd lose the audit trail. Then dedup at the transform boundary: `ROW_NUMBER() OVER (PARTITION BY natural_key ORDER BY event_ts DESC, sequence_id DESC) = 1`, filtered in an outer query. The three things that matter: **(1) the `ORDER BY` must be a total order** — a bare `event_ts DESC` is non-deterministic when two events share a millisecond, and the symptom is that your incremental model emits spurious changes every run and your tests flake; add the source's sequence/LSN/offset as a tiebreaker. **(2) Late-arriving data** — if you dedup incrementally over the last N hours, an event arriving a day late can be missed, so you need either a lookback window wider than your worst-case lateness or a periodic full rebuild. **(3) Tombstones** — deletes arrive as events too, so `rn = 1` might be a delete, and you must filter after deduplication, not before, or you'll resurrect deleted rows. I'd add a `unique` test on the natural key of the output, and a determinism test that runs the dedup twice and asserts identical results. At scale, partition by time so the window's sort doesn't span the whole table.

**Q9. (Trade-off) Window function vs `LATERAL` vs correlated subquery for "latest record per group"?**
> **`LATERAL`** with an index on `(group_key, sort_key DESC)`: a nested loop doing one index lookup per group, stopping immediately. Cost is proportional to the number of *groups*. Best when groups are many and you want few rows each. **Window function**: one sort over the entire table, then rank, then discard. Cost is proportional to the *table*, regardless of how few rows you want — but it's a single pass with no random I/O, so it wins when you're keeping most rows or when there's no usable index (a sequential scan + sort beats a million random index probes). PG15's `Run Condition` narrows the gap for `row_number`. **Correlated subquery**: usually the worst of both, and it doesn't extend to "top 3." **`DISTINCT ON`** (Postgres-only) is the fourth option and is often the fastest for exactly-one-per-group — it's a single sort with an early skip. My default is `DISTINCT ON` for Postgres-only "latest per group," `ROW_NUMBER` when portability matters or N > 1, and `LATERAL` when the plan shows a huge sort and I have (or can add) the index. And I'd say all of that only after looking at `EXPLAIN ANALYZE`, because the crossover depends on the group-count-to-row-count ratio.

**Q10. What's the performance cost of a window function?**
> **The sort.** `PARTITION BY a ORDER BY b` requires input sorted by `(a, b)`; if no index supplies that, you pay a full sort — and if it exceeds `work_mem` you pay an external merge sort on disk, which shows in `EXPLAIN ANALYZE` as `Sort Method: external merge  Disk: 500MB` and is often a 10× penalty. So: index on `(partition_cols, order_cols)` to eliminate the sort; filter before windowing so the sort input is small; and reuse one window definition (via the `WINDOW` clause) across functions so they share a sort rather than each triggering their own. There's also a subtler cost: aggregates **without an inverse transition function** — `MAX`, `MIN`, `string_agg` — must recompute the entire frame at every row, so `MAX(x) OVER (ROWS BETWEEN 1000 PRECEDING AND CURRENT ROW)` is roughly O(n × 1000) while `SUM` over the same frame is O(n), because `SUM` can subtract the row leaving the frame. That cliff surprises people.

**Q11. Explain the gaps-and-islands trick.**
> To find runs of consecutive values, compute `value - ROW_NUMBER() OVER (ORDER BY value)`. For consecutive values, both increment in lockstep, so the difference is **constant** across a run — it becomes a group key. Then `GROUP BY` that key and take `MIN`/`MAX`/`COUNT` to get each island's start, end, and length. For dates it's `date - row_number::int`; for integers it's just subtraction. It generalises: the related pattern is a **cumulative sum of a boolean flag as a group id** — flag "a new group starts here" with `LAG`, then `SUM(flag) OVER (ORDER BY ts ROWS UNBOUNDED PRECEDING)` turns the flags into sequential group numbers. That's how sessionisation works, and how you'd group by "any change in status." Both tricks are worth memorising because they turn a procedural problem into two window functions.

**Q12. (Senior) A dashboard query with five window functions over 200M rows takes 4 minutes. Diagnose it.**
> `EXPLAIN (ANALYZE, BUFFERS)` first. I'd expect to find some combination of four things. **(1) Multiple sorts** — if the five windows have different `OVER` clauses, that's up to five `WindowAgg` nodes each with its own `Sort`; consolidating them onto one or two named windows (via the `WINDOW` clause) can collapse that, since Postgres reuses a sort when one ordering is a prefix of another. **(2) External merge sorts** — `Sort Method: external merge Disk: …` means we exceeded `work_mem`; raise it with `SET LOCAL` for this query, remembering it's per-node-per-worker so five sorts × four workers is 20× the setting. **(3) Windowing before filtering** — if there's a `WHERE` in the outer query, we ranked 200M rows to keep 10,000; push the filter into the CTE. **(4) A missing index** — `(partition_cols, order_cols)` can eliminate the sort entirely, which is usually the single biggest win. Beyond tuning, I'd question the architecture: a user-facing dashboard should not window 200M rows at read time. The right fix is almost always to precompute — a materialised daily rollup that the dashboard windows over 365 rows instead of 200M, refreshed incrementally. That's the same recompute-vs-precompute trade from the aggregation and hierarchy lessons, and at this scale it's not a close call. I'd also check whether the 200M rows are partitioned by time; if not, that's the second structural fix, because it makes every subsequent query prune.

## 12. Homework

> Work in `sql/solutions/lesson-06/`. Use the Lesson 04 seed (50k orders) plus `employees`. Capture `EXPLAIN (ANALYZE, BUFFERS)` wherever a task mentions plans.

**Easy**

1. **Window vs `GROUP BY`.** Write the same question ("how many customers per country") both ways. Then write a query that could only be done with a window: each customer's name, their country's size, and their rank by signup date *within* their country.
   *Acceptance:* three queries, with a comment on each explaining why the third is impossible with `GROUP BY` alone.

2. **The ranking family.** On `employees` (with a deliberate salary tie you introduce), output all six ranking functions side by side. Then answer in `RANKS.md`: which one would you use for a leaderboard, for dedup, for "how many people earn more than me", and for quartiles — with one sentence each.
   *Acceptance:* the comparison table, correct tie behaviour visible, and the four answers.

3. **Prove the default frame.** In one query, output `SUM(salary) OVER ()`, `SUM(salary) OVER (ORDER BY salary)`, `SUM(salary) OVER (ORDER BY salary ROWS UNBOUNDED PRECEDING)`, and `SUM(salary) OVER (ORDER BY salary RANGE UNBOUNDED PRECEDING)` — with a tie present. Explain each column's values.
   *Acceptance:* the four columns, and a written explanation of why exactly two of them differ and where.

**Medium**

4. **The `LAST_VALUE` trap.** Demonstrate `LAST_VALUE` returning the current row, then fix it two ways (explicit frame; reversed `FIRST_VALUE`). Do the same for a `MAX() OVER (ORDER BY ...)` that only sees the running max.
   *Acceptance:* broken and fixed outputs side by side, plus a one-paragraph explanation of the frame mechanic that causes both.

5. **Dedup, deterministically.** Build a `raw_events` table with 100k rows, ~20% duplicate keys, and deliberate ties on `updated_at`. Dedup with `ROW_NUMBER` **without** a tiebreaker, run it 10 times, and detect that the output changes (hash the result set). Then add the tiebreaker and prove stability across 10 runs. Also implement it with `DISTINCT ON` and benchmark.
   *Acceptance:* evidence of non-determinism (differing hashes), evidence of stability after the fix, a timing comparison of `ROW_NUMBER` vs `DISTINCT ON`, and a note on which you'd ship and why. **This is the most professionally valuable task in the lesson.**

6. **The revenue dashboard.** From the 50k orders, produce a monthly report with: revenue, cumulative revenue, previous month, MoM %, YoY % (hint: `LAG(revenue, 12)`), 3-month moving average, percent of annual total, and rank. Use a `WINDOW` clause. Handle the first months' NULLs sensibly.
   *Acceptance:* correct output for all 24 months; `NULLIF` guards present; a comment on each column stating its frame; and an `EXPLAIN` showing how many `Sort` nodes you ended up with (aim for the minimum, and say what the minimum is).

7. **`ROWS` vs `RANGE` on gapped data.** Build a daily revenue series and **delete 15% of the days at random**. Compute a 7-day moving average both ways. Show they differ, quantify by how much, and explain which is correct.
   *Acceptance:* a query showing both columns and their difference, the maximum divergence found, and a written statement of which you'd ship for a business metric and why. Bonus: fix the `ROWS` version by generating the missing days with `generate_series` + `LEFT JOIN` and show it now matches.

**Hard**

8. **Gaps and islands.** Find every customer's longest streak of consecutive days with at least one order. Output: customer, streak start, streak end, streak length. Then extend it to "streaks of consecutive *weeks*" and to "streaks allowing a 1-day gap" (harder — the naive trick doesn't work).
   *Acceptance:* correct output verified by hand against 3 customers; a written explanation of *why* `date - row_number` is constant across a run; and a working 1-day-tolerance version with an explanation of why you needed a different approach.

9. **Sessionisation.** Generate 500k events for 10k users over 30 days with realistic clustering. Group into sessions with a 30-minute inactivity timeout. Output per session: user, session number, start, end, duration, event count, and whether it's that user's first session. Then compute: median session duration, sessions per user per day, and the distribution of events per session.
   *Acceptance:* correct sessions verified by hand for 2 users; the `LAG` → flag → cumulative-`SUM` pattern used; percentiles via `percentile_cont`; and a note in `NOTES.md` on how a streaming system (Flink session windows) would compute the same thing and what it does about late-arriving events that this batch query doesn't have to.

10. **Window vs `LATERAL` vs `DISTINCT ON`, measured.** Implement "the 3 most recent orders per customer" all three ways. Benchmark each **with** and **without** an index on `orders(customer_id, created_at DESC)`, at 50k orders and at 5M orders (scale the seed). That's 12 measurements.
    *Acceptance:* a 3×2×2 results table, the plans for the four most interesting cells, the observed crossover point, and a decision rule you'd actually give a colleague. Check whether PG15's `Run Condition` appears in your window plan and report whether it changed the outcome.

**Stretch (senior)**

11. **SCD Type-2 dimension.** Given a `customer_changes (customer_id, attribute_snapshot, changed_at)` feed with duplicates and out-of-order arrivals, build a proper Type-2 dimension: one row per (customer, validity interval), with `valid_from`, `valid_to`, `is_current`, and no gaps or overlaps. Use `LEAD` to close intervals and `ROW_NUMBER` to dedup. Collapse consecutive rows where nothing actually changed.
    *Acceptance:* assertions proving no overlaps (`NOT EXISTS` an overlapping pair), no gaps, exactly one `is_current` per customer, and no two adjacent rows with identical attributes. Then feed it a **late-arriving** change dated in the middle of an existing interval and show what breaks — and fix it. The late-arrival handling is the real lesson.

12. **The inverse-transition cliff.** Empirically demonstrate that `SUM` over a sliding frame is O(n) while `MAX` over the same frame is not. Benchmark both with frames of 10, 100, 1,000, and 10,000 preceding rows over 1M rows. Then look up `aggminvtransfn` in `pg_aggregate` for both and explain the results.
    *Acceptance:* a timing table showing `SUM` roughly flat as frame size grows and `MAX` scaling with it; the `pg_aggregate` query and its output; a written explanation; and one practical recommendation for computing a rolling max efficiently anyway (hint: think about what a monotonic deque does in Python — Lesson 05).

## 13. Key takeaways

- A window function computes over related rows **without collapsing them**. `GROUP BY` collapses; `OVER` doesn't.
- Windows run at step 5 — **after** `WHERE`/`GROUP BY`/`HAVING`, **before** `ORDER BY`. So you can never filter on one in `WHERE`; **compute inside a subquery, filter outside**. (No `QUALIFY` in Postgres.)
- **The default frame is `RANGE UNBOUNDED PRECEDING AND CURRENT ROW`.** Adding `ORDER BY` silently converts a total into a **running total**, and `RANGE` includes **all peers**, so it jumps at ties. Say `ROWS` when you mean rows.
- **`LAST_VALUE` without an explicit frame is always a bug** — the frame ends at the current row.
- **`ROW_NUMBER`'s `ORDER BY` must be a total order.** Missing tiebreakers cause non-deterministic output, flaky tests, and churning incremental models. This is the highest-value practical rule in the lesson.
- `ROW_NUMBER` (arbitrary ties) vs `RANK` (ties tie, gaps follow) vs `DENSE_RANK` (ties tie, no gaps). Pick by what ties should mean.
- **`RANGE` with an interval** (`'6 days' PRECEDING`) is the correct moving average over gapped time-series; `ROWS` averages "the last 7 rows that exist."
- The cost is **the sort**. Index `(partition_cols, order_cols)` to eliminate it, filter before windowing, and share one window definition across functions.
- Memorise the patterns: dedup (`ROW_NUMBER = 1`), running total, moving average, top-N, period-over-period (`LAG`), gaps-and-islands (`value - row_number`), sessionisation (`LAG` → flag → cumulative `SUM`).
- Windows are SQL:2003 and portable across every major database and warehouse. `DISTINCT ON` (Postgres) and `QUALIFY` (Snowflake/BigQuery/DuckDB) are not.
