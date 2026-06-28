# SQL Lesson 02 — Querying: SELECT, Filtering, Sorting & Expressions

| | |
|---|---|
| **Track** | SQL |
| **Lesson** | 02 of 15 |
| **Topic** | The `SELECT` statement — projecting, filtering, sorting, limiting, and expressions |
| **Estimated time** | ~2 hours |
| **Prerequisites** | Lesson 01 (running Postgres with seed data) |

### Learning objectives
1. Write `SELECT` queries with projection, `WHERE`, `ORDER BY`, `LIMIT/OFFSET`, `DISTINCT`.
2. Use the full range of predicates: comparison, `BETWEEN`, `IN`, `LIKE`/`ILIKE`, `IS NULL`.
3. Handle `NULL` correctly (three-valued logic) — the #1 source of query bugs.
4. Use expressions, `CASE`, `COALESCE`, and common scalar functions.
5. Understand the **logical order of evaluation** of a SELECT and why it explains alias rules.

---

## 1. Why this matters
`SELECT` is 80% of the SQL you'll ever write. Doing it *correctly* — especially around `NULL` and the evaluation order — separates people who "kind of know SQL" from those who can be trusted with production data. Every join, aggregation, and window function later is a `SELECT` with more clauses.

---

## 2. Theory

### 2.1 Anatomy of a SELECT
```sql
SELECT   col1, col2, expr AS alias      -- projection (what columns)
FROM     table                          -- source
WHERE    predicate                      -- row filter (before grouping)
ORDER BY col [ASC|DESC] [NULLS LAST]    -- sort
LIMIT n OFFSET m;                       -- pagination
```

### 2.2 Logical order of evaluation (memorize this)
SQL is written `SELECT … FROM … WHERE …`, but it's *evaluated* in a different order, which explains many rules:

```
1. FROM       (and JOINs)        → assemble source rows
2. WHERE      (row filter)
3. GROUP BY                      (Lesson 04)
4. HAVING     (group filter)     (Lesson 04)
5. SELECT     (projection, compute expressions, assign aliases)
6. DISTINCT
7. ORDER BY
8. LIMIT / OFFSET
```
Consequence: you **can't use a `SELECT` alias in `WHERE`** (WHERE runs before SELECT), but you **can** use it in `ORDER BY` (which runs after SELECT). Knowing this stops a lot of confusion.

### 2.3 Predicates in WHERE
```sql
WHERE price > 100
WHERE category = 'books'
WHERE price BETWEEN 10 AND 50          -- inclusive both ends
WHERE category IN ('books', 'music')
WHERE name LIKE 'A%'                    -- case-sensitive, % = any chars, _ = one char
WHERE name ILIKE 'a%'                   -- case-INsensitive (Postgres)
WHERE deleted_at IS NULL               -- correct NULL test
WHERE price > 100 AND category = 'books'
WHERE NOT (status = 'cancelled')
```

### 2.4 NULL and three-valued logic (critical)
`NULL` means *unknown*, not zero and not empty string. Any comparison with `NULL` yields `NULL` (≈ "unknown"), which is **not** true, so the row is excluded.

```sql
SELECT NULL = NULL;     -- NULL  (not TRUE!)
SELECT NULL <> 5;       -- NULL
WHERE bonus = NULL      -- WRONG: matches nothing, ever
WHERE bonus IS NULL     -- RIGHT
```
Three-valued logic: predicates can be TRUE, FALSE, or **UNKNOWN**. `WHERE` keeps only TRUE rows. This affects `NOT IN` with NULLs, aggregates (which skip NULLs), and more.

```sql
-- Trap: NOT IN with a NULL in the list returns NO rows
WHERE id NOT IN (1, 2, NULL)   -- always UNKNOWN → empty result. Use NOT EXISTS instead.
```

### 2.5 Expressions, CASE, COALESCE
```sql
SELECT
    name,
    price,
    price * 1.2          AS price_with_tax,
    COALESCE(nickname, name) AS display_name,    -- first non-NULL
    CASE
        WHEN price > 100 THEN 'premium'
        WHEN price > 20  THEN 'standard'
        ELSE 'budget'
    END AS tier
FROM products;
```
- `COALESCE(a, b, c)` → first non-NULL argument.
- `NULLIF(a, b)` → NULL if `a = b`, else `a` (handy to avoid divide-by-zero).
- `CASE` → conditional expression, usable anywhere an expression is allowed (including `ORDER BY`).

### 2.6 DISTINCT, ORDER BY, pagination
```sql
SELECT DISTINCT category FROM products;          -- unique values
SELECT * FROM products ORDER BY price DESC, name ASC;
SELECT * FROM products ORDER BY price LIMIT 10 OFFSET 20;  -- page 3 of 10
```
`NULLS FIRST`/`NULLS LAST` controls where NULLs sort (default: NULLs sort as if largest, so last in ASC... actually Postgres default is NULLS LAST for ASC, NULLS FIRST for DESC — be explicit when it matters).

---

## 3. Official documentation quotes

> "The `SELECT` command is used to specify queries. … `SELECT` retrieves rows from zero or more tables."
> — *PostgreSQL docs*, [SELECT](https://www.postgresql.org/docs/current/sql-select.html)

> "Do not write `expression = NULL` because `NULL` is not 'equal to' `NULL`. (The null value represents an unknown value, and it is not known whether two unknown values are equal.) … To check whether a value is or is not null, use the predicates `IS NULL` and `IS NOT NULL`."
> — *PostgreSQL docs*, [9.2. Comparison Functions and Operators](https://www.postgresql.org/docs/current/functions-comparison.html)

> "The `COALESCE` function returns the first of its arguments that is not null. Null is returned only if all arguments are null."
> — *PostgreSQL docs*, [9.18. Conditional Expressions](https://www.postgresql.org/docs/current/functions-conditional.html)

---

## 4. Real-world examples

Assume the `products(id, name, category, price, nickname, created_at)` table from Lesson 01 (extended).

### 4.1 Filter + sort + paginate
```sql
SELECT name, category, price
FROM products
WHERE category IN ('books', 'music')
  AND price BETWEEN 10 AND 100
ORDER BY price DESC
LIMIT 5;
```

### 4.2 Safe NULL handling and computed columns
```sql
SELECT
    name,
    COALESCE(nickname, name)              AS display_name,
    price,
    ROUND(price * 0.9, 2)                 AS sale_price,
    CASE WHEN price IS NULL THEN 'no price set' ELSE 'priced' END AS status
FROM products
ORDER BY price NULLS LAST;
```

### 4.3 Why your alias fails in WHERE
```sql
-- ERROR: column "tier" does not exist
SELECT price, CASE WHEN price > 100 THEN 'premium' ELSE 'other' END AS tier
FROM products
WHERE tier = 'premium';      -- WHERE runs before SELECT defines 'tier'

-- Fix: repeat the expression, or wrap in a subquery / CTE (Lesson 05)
WHERE (CASE WHEN price > 100 THEN 'premium' ELSE 'other' END) = 'premium';
```

### 4.4 Pattern search
```sql
SELECT name FROM products WHERE name ILIKE '%pro%';   -- contains 'pro', any case
```

---

## 6. How This Is Used In Production
- **Startups:** essentially every screen in an app is a `SELECT … WHERE … ORDER BY … LIMIT` (the product list, the user's orders, search results). Keyset/cursor pagination (not just `OFFSET`) becomes important as data grows.
- **Enterprises:** read-heavy services route these `SELECT`s to **read replicas**; ad-hoc filtering powers internal admin tools and BI dashboards. Parameterized queries are mandatory (SQL injection prevention).
- **Common architectures:** API endpoint → ORM/query builder → parameterized SELECT → connection pool → Postgres replica; results cached in Redis for hot queries.
- **Scaling:** `OFFSET` pagination degrades on deep pages (the DB still scans+discards skipped rows) — production systems switch to **keyset pagination** (`WHERE id > :last_seen ORDER BY id LIMIT n`). Filters on unindexed columns trigger full scans; indexing (Lesson 09) fixes this.
- **Monitoring/Logging:** slow `SELECT`s surface in `pg_stat_statements`; APM tools (Datadog, New Relic) flag N+1 query patterns from ORMs.
- **Security:** **never** string-concatenate user input into SQL — use bound parameters. `WHERE name = '` + input is the classic SQL-injection hole.
- **Real examples:** Every product you use — **Amazon** product search, **Netflix** "your list," **GitHub** issue filters — is, at the data layer, parameterized `SELECT`s with `WHERE`/`ORDER BY`/`LIMIT`, fronted by caches and replicas. The query patterns you learn here are exactly those, minus the scale plumbing.

---

## 7. Best practices
- Select only the columns you need; avoid `SELECT *` in application code (breaks on schema change, wastes I/O).
- Always test NULL handling explicitly; use `IS NULL`/`COALESCE`, never `= NULL`.
- Prefer `ILIKE`/`= ` with proper indexing over leading-wildcard `LIKE '%x'` (can't use a btree index).
- Use **parameterized queries**; never concatenate user input.
- For pagination beyond a few pages, use keyset pagination, not large `OFFSET`s.
- Be explicit with `ORDER BY` — result order is undefined without it (even if it "looks" sorted).

## 8. Common mistakes & gotchas
- `= NULL` instead of `IS NULL` — silently matches nothing.
- `NOT IN (subquery)` where the subquery can return NULL — yields empty results; use `NOT EXISTS`.
- Relying on row order without `ORDER BY`.
- Using a `SELECT` alias in `WHERE`/`HAVING` (it's not defined yet there).
- Forgetting that `BETWEEN` is **inclusive** on both ends.
- Leading-wildcard `LIKE '%term'` can't use a normal index → full scan.
- `COUNT(column)` ignores NULLs while `COUNT(*)` counts rows (preview Lesson 04).

## 9. Where AI helps (and where it hurts)
- **Helps:** turning a plain-English filter into a correct `WHERE`, writing `CASE` expressions, explaining a three-valued-logic surprise, generating sample data, suggesting keyset pagination.
- **Hurts:** AI frequently writes `= NULL`, forgets the `NOT IN`/NULL trap, and produces non-parameterized (injectable) query strings. Treat any AI SQL with user input as insecure until you've parameterized it.

## 10. Learn independently
- [PostgreSQL SELECT reference](https://www.postgresql.org/docs/current/sql-select.html) and [comparison operators](https://www.postgresql.org/docs/current/functions-comparison.html).
- [PGExercises — Basic](https://pgexercises.com/questions/basic/) — drill these.
- *SQL for Smarties* (Celko) on NULL handling.
- Markus Winand's writing on pagination (keyset vs offset).

## 11. Interview preparation

**Q1. What is the logical order of evaluation of a SELECT?**
> FROM/JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT/OFFSET. This is why you can't reference a SELECT alias in WHERE but can in ORDER BY.

**Q2. How do you test for NULL, and why not `= NULL`?**
> Use `IS NULL` / `IS NOT NULL`. `= NULL` evaluates to UNKNOWN (not TRUE) because NULL means "unknown," so the row is never returned.

**Q3. Explain three-valued logic.**
> Predicates can be TRUE, FALSE, or UNKNOWN (when NULL is involved). `WHERE` returns only rows where the predicate is TRUE. This affects `NOT IN`, `AND`/`OR` combinations, and comparisons.

**Q4. `WHERE` vs `HAVING`?**
> `WHERE` filters individual rows before grouping; `HAVING` filters groups after `GROUP BY` (and can use aggregates). (Detail in Lesson 04.)

**Q5. Difference between `COUNT(*)`, `COUNT(col)`, `COUNT(DISTINCT col)`?**
> `COUNT(*)` counts rows; `COUNT(col)` counts non-NULL values of col; `COUNT(DISTINCT col)` counts distinct non-NULL values.

**Q6. Why is `OFFSET`-based pagination slow on deep pages, and what's the fix?**
> The database must still produce and discard all skipped rows, so cost grows with offset. Keyset pagination (`WHERE id > :last ORDER BY id LIMIT n`) uses an index to jump straight to the next page in constant time.

**Q7. What's `COALESCE` and a typical use?**
> Returns the first non-NULL argument — e.g., `COALESCE(nickname, full_name, 'Anonymous')` to provide a fallback display value.

## 12. Homework
> Use the `products`/`customers` tables; save scripts in `sql/solutions/lesson-02/`.

**Easy**
1. Write queries that: (a) list product names and prices for one category sorted by price descending; (b) return the 5 cheapest products; (c) find products whose name contains "pro" case-insensitively.

**Medium**
2. Add a `nickname` column (nullable) to `customers` and set it for some rows. Write a query returning a `display_name` that uses the nickname when present, else the name, and a `CASE`-based `tier` column based on some numeric field. Sort with `NULLS LAST`. Then demonstrate the `= NULL` vs `IS NULL` difference with two queries and comment on the row counts.

**Hard**
3. Demonstrate the `NOT IN` + NULL trap: build a small two-table example where `WHERE x NOT IN (SELECT y ...)` returns zero rows because the subquery contains a NULL, then rewrite it correctly with `NOT EXISTS` and show it returns the expected rows. Explain in comments why each behaves as it does.

**Stretch:** Implement keyset pagination over `products` ordered by `(price, id)` and write the query for "the next page after the last row I saw was (price=49.99, id=120)."

## 13. Key takeaways
- A SELECT is evaluated FROM → WHERE → GROUP/HAVING → SELECT → ORDER → LIMIT.
- `NULL` is "unknown": use `IS NULL`; comparisons with NULL are UNKNOWN, not FALSE.
- `COALESCE`, `NULLIF`, and `CASE` handle nulls and conditional logic in expressions.
- Watch the `NOT IN`+NULL trap; prefer `NOT EXISTS`.
- Always `ORDER BY` for deterministic order; use keyset pagination at scale; always parameterize input.
