# 🗄️ SQL Track

PostgreSQL-first SQL, taken to the point of **confidence** — querying, modeling, tuning, and operating databases — with a tilt toward **data engineering, CDC, and streaming (Kafka)**.

**Target level by the end:** can design a normalized schema, write window-function-heavy analytical queries, read `EXPLAIN ANALYZE`, reason about transactions/isolation, and understand how databases feed streaming pipelines.

## Syllabus (15 lessons)

| # | Topic | Status |
|---|-------|--------|
| 01 | [Relational fundamentals & PostgreSQL setup](lesson-01-relational-fundamentals-and-postgres-setup.md) | ✅ |
| 02 | [Querying — SELECT, filtering, sorting, expressions](lesson-02-querying-select-filtering-sorting.md) | ✅ |
| 03 | [Joins — inner, outer, self, cross, anti/semi](lesson-03-joins.md) | ✅ |
| 04 | Aggregation — GROUP BY, HAVING, aggregate functions | ⬜ |
| 05 | Subqueries & CTEs (incl. recursive) | ⬜ |
| 06 | Window functions | ⬜ |
| 07 | Data types, constraints & keys | ⬜ |
| 08 | Schema design & normalization (ER modeling) | ⬜ |
| 09 | Indexes & the query planner (`EXPLAIN`) | ⬜ |
| 10 | Transactions, ACID, isolation & locking | ⬜ |
| 11 | Views, materialized views, functions & triggers | ⬜ |
| 12 | Performance tuning & optimization | ⬜ |
| 13 | Advanced types — JSON/JSONB, arrays, full-text search | ⬜ |
| 14 | Administration — backup, replication, roles & security | ⬜ |
| 15 | Data pipelines — CDC, logical replication & Kafka | ⬜ |

Each lesson follows the format defined in [`../CLAUDE.md`](../CLAUDE.md).
