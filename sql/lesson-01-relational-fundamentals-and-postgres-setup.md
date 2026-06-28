# SQL Lesson 01 — Relational Fundamentals & PostgreSQL Setup

| | |
|---|---|
| **Track** | SQL |
| **Lesson** | 01 of 15 |
| **Topic** | The relational model, what a RDBMS is, and getting PostgreSQL running |
| **Estimated time** | ~2 hours |
| **Prerequisites** | A terminal; Docker *or* a local PostgreSQL install |

### Learning objectives
1. Explain the relational model: relations (tables), tuples (rows), attributes (columns), domains.
2. Describe what a RDBMS does and where PostgreSQL fits among databases.
3. Install/run PostgreSQL (Docker recommended) and connect with `psql`.
4. Understand the client/server model, databases, schemas, and the system catalog.
5. Create your first database, table, and rows; run your first queries.

---

## 1. Why this matters
SQL is the most durable skill in this whole curriculum — the relational model is 50 years old and still runs the world's transactional systems. Data engineering, backend, and analytics all sit on it. Getting the *mental model* right (sets, not loops) and a clean local Postgres now means every later SQL lesson and every capstone has a place to run.

---

## 2. Theory

### 2.1 The relational model
Proposed by E.F. Codd (1970). Data is organized into **relations** (tables):
- A **relation/table** is a set of rows.
- A **tuple/row** is one record.
- An **attribute/column** has a name and a **domain** (its data type, e.g. `integer`, `text`).
- A **key** uniquely identifies rows (primary key); relationships between tables use **foreign keys**.

The model is built on set theory and predicate logic. The profound consequence: **SQL is declarative** — you describe *what* set of data you want, not *how* to fetch it loop-by-loop. The database's **query planner** figures out the how.

### 2.2 What a RDBMS provides
A Relational Database Management System gives you:
- **Persistence** with structure (schemas, types, constraints).
- **ACID transactions** (atomicity, consistency, isolation, durability — Lesson 10).
- **A query optimizer** that turns SQL into efficient execution plans.
- **Concurrency control** so many clients read/write safely at once.
- **Integrity enforcement** via constraints (NOT NULL, UNIQUE, FOREIGN KEY, CHECK).

### 2.3 Where PostgreSQL fits
- **PostgreSQL** — open-source, standards-compliant, extensible (JSONB, full-text search, GIS via PostGIS, custom types). The default serious choice; our dialect for this track.
- **MySQL/MariaDB** — also popular OLTP; some dialect differences.
- **SQLite** — embedded, file-based, great for local/testing, no server.
- **Cloud-managed** — Amazon RDS/Aurora, Google Cloud SQL, Azure Database (managed Postgres/MySQL).
- **OLAP/warehouses** — Snowflake, BigQuery, Redshift, ClickHouse — for analytics at scale (different trade-offs; we contrast OLTP vs OLAP later).

### 2.4 Client/server, databases, schemas
- PostgreSQL is a **server** (a process listening on port 5432). Clients (`psql`, your app, a GUI) connect over the network.
- A server hosts multiple **databases**. A database contains **schemas** (namespaces; default is `public`). A schema contains **tables**, views, functions, etc.
- Metadata lives in the **system catalog** (`pg_catalog`) and the standard `information_schema`.

```
PostgreSQL server (port 5432)
└── database: shop
    └── schema: public
        ├── table: customers
        └── table: orders
```

### 2.5 SQL sublanguages
- **DDL** (Data Definition): `CREATE`, `ALTER`, `DROP` — structure.
- **DML** (Data Manipulation): `SELECT`, `INSERT`, `UPDATE`, `DELETE` — data.
- **DCL** (Data Control): `GRANT`, `REVOKE` — permissions.
- **TCL** (Transaction Control): `BEGIN`, `COMMIT`, `ROLLBACK`.

---

## 3. Official documentation quotes

> "PostgreSQL is a powerful, open source object-relational database system that uses and extends the SQL language combined with many features that safely store and scale the most complicated data workloads."
> — *PostgreSQL*, [About](https://www.postgresql.org/about/)

> "PostgreSQL uses a client/server model. A PostgreSQL session consists of the following cooperating processes (programs): A server process, which manages the database files, accepts connections to the database from client applications, and performs database actions on behalf of the clients."
> — *PostgreSQL docs*, [1.2. Architectural Fundamentals](https://www.postgresql.org/docs/current/tutorial-arch.html)

> "SQL has become the standard relational database language. … SQL is a declarative language: you tell the database what you want, not how to get it."
> — paraphrasing the relational model; see *PostgreSQL docs*, [Part I. Tutorial](https://www.postgresql.org/docs/current/tutorial.html)

---

## 4. Real-world examples

### 4.1 Run PostgreSQL with Docker (recommended — clean, disposable)
```bash
docker run --name pg-lab \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_USER=learner \
  -e POSTGRES_DB=playground \
  -p 5432:5432 \
  -d postgres:16

# connect with psql running inside the container
docker exec -it pg-lab psql -U learner -d playground
```

### 4.2 Or connect with a local `psql`
```bash
psql "postgresql://learner:secret@localhost:5432/playground"
```

### 4.3 First commands inside `psql`
```sql
-- meta-commands (psql-specific, start with backslash)
\l            -- list databases
\dt           -- list tables
\d customers  -- describe a table
\q            -- quit

-- create a table
CREATE TABLE customers (
    id         SERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    email      TEXT UNIQUE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- insert rows
INSERT INTO customers (name, email) VALUES
    ('Ann Lee',  'ann@example.com'),
    ('Bob Ray',  'bob@example.com');

-- query
SELECT id, name, email FROM customers ORDER BY name;
```
```
 id |  name   |      email
----+---------+-----------------
  1 | Ann Lee | ann@example.com
  2 | Bob Ray | bob@example.com
```

### 4.4 Inspect the catalog
```sql
SELECT table_name FROM information_schema.tables
WHERE table_schema = 'public';
```

---

## 6. How This Is Used In Production
- **Startups:** PostgreSQL is the default first database — one managed instance (RDS/Cloud SQL/Supabase/Neon) backing the whole app. It's cheap, reliable, and scales surprisingly far vertically before you need anything fancier.
- **Enterprises:** run Postgres in **primary + read-replica** topologies, often multi-AZ for high availability, with connection pooling (PgBouncer), automated backups/PITR, and separate OLTP (Postgres) vs OLAP (Snowflake/BigQuery) systems fed by ETL/CDC.
- **Common architectures:** app servers → connection pooler → primary (writes) + replicas (reads); change data capture (logical replication / Debezium) streams row changes into Kafka and then into a warehouse (covered in Lesson 15).
- **Scaling:** vertical (bigger instance) first; then read replicas; then partitioning/sharding (Citus) or moving analytics off to a warehouse. Indexing and query tuning (Lessons 09/12) usually matter far more than hardware.
- **Monitoring/Logging:** `pg_stat_statements` for slow queries, `pg_stat_activity` for live sessions, exporters into Prometheus/Grafana, and `EXPLAIN ANALYZE` for plan analysis.
- **Security:** TLS connections, role-based access (Lesson 14), least-privilege app users, secrets in a vault, network isolation (no public 5432).
- **CI/CD:** schema migrations (Alembic, Flyway, Liquibase) run as a deploy step; ephemeral Postgres in CI (the exact Docker command above) for integration tests.
- **Real examples:** **Instagram** scaled on PostgreSQL (sharded) to hundreds of millions of users; **GitLab**, **Apple**, and **Cloudflare** run PostgreSQL heavily; **Amazon Aurora** is a Postgres/MySQL-compatible re-architecture for the cloud. PostgreSQL is among the most widely used databases in the world per the annual Stack Overflow Developer Survey.

---

## 7. Best practices
- Use Docker for a clean, reproducible local lab you can throw away and recreate.
- Name things in `snake_case`; tables plural (`customers`), PK column `id`.
- Always define a primary key; use `NOT NULL` aggressively; add `UNIQUE`/`CHECK` where the domain demands it.
- Prefer `TIMESTAMPTZ` over `TIMESTAMP` for points in time; store UTC.
- Keep DDL in version-controlled migration files, not typed ad-hoc into prod.
- Never connect your app as a superuser; create a least-privilege role.

## 8. Common mistakes & gotchas
- Confusing **DELETE FROM table** (removes rows) with **DROP TABLE** (removes the table).
- `psql` meta-commands (`\dt`) are *not* SQL and won't work from your app driver — use `information_schema` queries there.
- Forgetting the semicolon — `psql` waits for it; the prompt changes to `playground-#`.
- Using `TIMESTAMP` (no zone) and getting timezone bugs; prefer `TIMESTAMPTZ`.
- Storing money as `float` (rounding errors) — use `NUMERIC`.
- Treating SQL imperatively ("loop over rows") instead of thinking in sets.

## 9. Where AI helps (and where it hurts)
- **Helps:** generating the Docker run command, explaining `psql` meta-commands, drafting starter DDL, translating "I want a table of X" into a sensible schema, explaining error messages.
- **Hurts:** AI may target the wrong SQL dialect (MySQL syntax in a Postgres context — e.g. `AUTO_INCREMENT` vs `SERIAL`/`GENERATED`), or suggest schemas without constraints. Always confirm it's PostgreSQL-flavored and that keys/constraints are present.

## 10. Learn independently
- [PostgreSQL Tutorial (official)](https://www.postgresql.org/docs/current/tutorial.html) — start to finish.
- [PostgreSQL Exercises](https://pgexercises.com/) — superb interactive practice.
- *The Art of PostgreSQL* (Dimitri Fontaine).
- [Use The Index, Luke](https://use-the-index-luke.com/) (save for Lesson 09).
- The classic relational model: Codd's 1970 paper "A Relational Model of Data for Large Shared Data Banks."

## 11. Interview preparation

**Q1. What is a relational database?**
> A database that stores data in tables (relations) of rows and columns, where relationships between tables are expressed via keys, governed by the relational model and queried with SQL. It enforces structure and integrity and supports ACID transactions.

**Q2. SQL is declarative — what does that mean?**
> You specify *what* result you want; the database's query optimizer decides *how* to compute it (which indexes, join order, algorithms). You don't write the loops.

**Q3. Difference between a database and a schema in PostgreSQL?**
> A server hosts multiple databases; you connect to one at a time and can't query across them in a single query easily. A schema is a namespace *within* a database (default `public`) that groups tables/objects and lets you avoid name collisions and manage permissions.

**Q4. OLTP vs OLAP?**
> OLTP (online transaction processing) = many small, fast read/write transactions (an app backend; PostgreSQL/MySQL). OLAP (online analytical processing) = large aggregate/analytical queries over huge datasets (warehouses like Snowflake/BigQuery). Different storage layouts (row vs columnar) and trade-offs.

**Q5. What does ACID stand for?**
> Atomicity (all-or-nothing transactions), Consistency (constraints always hold), Isolation (concurrent transactions don't corrupt each other), Durability (committed data survives crashes). Detailed in Lesson 10.

**Q6. Why PostgreSQL over MySQL or SQLite (in general)?**
> PostgreSQL is highly standards-compliant and extensible (JSONB, full-text, GIS, custom types, advanced indexing), with strong concurrency (MVCC). SQLite is embedded/serverless — great for local/testing but not concurrent server workloads. MySQL is also solid OLTP; choice often comes down to ecosystem, features, and team familiarity.

## 12. Homework
> Save SQL scripts under `sql/solutions/lesson-01/`.

**Easy**
1. Stand up PostgreSQL via Docker, connect with `psql`, and run `\l`, `\dt`, `SELECT version();`. Paste the version output into a `notes.md`.

**Medium**
2. Write `schema.sql` creating two tables — `customers` and `products` — each with a primary key, appropriate `NOT NULL`/`UNIQUE` constraints, a `NUMERIC` price on products, and `created_at TIMESTAMPTZ DEFAULT now()`. Insert at least 5 rows in each. Provide the script so it can be run from scratch (`DROP TABLE IF EXISTS` at top).

**Hard**
3. Make it reproducible: write a short `README.md` + a single script that (a) creates the database objects, (b) seeds data, and (c) runs 3 verification `SELECT`s. Then prove it: drop the container, recreate it with the Docker command, and run your script against the fresh container so it rebuilds everything. Document the exact commands.

**Stretch:** Query `information_schema` to list every column, its data type, and whether it's nullable for your two tables — without using `\d`.

## 13. Key takeaways
- Relational model: tables (relations) of rows (tuples) and typed columns (attributes), linked by keys.
- SQL is **declarative** — describe the result, let the optimizer plan it.
- PostgreSQL is a client/server RDBMS; a server → databases → schemas → tables.
- DDL/DML/DCL/TCL are the four SQL sublanguages.
- Run a disposable Postgres in Docker; keep schema in version-controlled scripts.
