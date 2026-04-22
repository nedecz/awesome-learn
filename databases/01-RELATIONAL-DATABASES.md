# Relational Databases

## Table of Contents

1. [Overview](#overview)
2. [SQL Fundamentals](#sql-fundamentals)
   - [Data Definition Language (DDL)](#data-definition-language-ddl)
   - [Data Manipulation Language (DML)](#data-manipulation-language-dml)
   - [Data Query Language (DQL)](#data-query-language-dql)
   - [Joins](#joins)
3. [Composing Good SQL Scripts](#composing-good-sql-scripts)
4. [Normalization](#normalization)
   - [First Normal Form (1NF)](#first-normal-form-1nf)
   - [Second Normal Form (2NF)](#second-normal-form-2nf)
   - [Third Normal Form (3NF)](#third-normal-form-3nf)
   - [Boyce-Codd Normal Form (BCNF)](#boyce-codd-normal-form-bcnf)
   - [When to Denormalize](#when-to-denormalize)
5. [Indexing](#indexing)
   - [B-Tree Indexes](#b-tree-indexes)
   - [Hash Indexes](#hash-indexes)
   - [GIN and GiST Indexes](#gin-and-gist-indexes)
   - [Composite Indexes](#composite-indexes)
   - [Covering Indexes](#covering-indexes)
   - [Partial Indexes](#partial-indexes)
   - [When to Index](#when-to-index)
6. [Transactions](#transactions)
   - [ACID Deep-Dive](#acid-deep-dive)
   - [Isolation Levels and Anomalies](#isolation-levels-and-anomalies)
   - [Write Skew and Phantoms](#write-skew-and-phantoms)
   - [Explicit Transactions](#explicit-transactions)
7. [PostgreSQL](#postgresql)
8. [MySQL](#mysql)
9. [SQL Server](#sql-server)
10. [Comparison Table](#comparison-table)
11. [Version History](#version-history)

---

## Overview

The relational model was introduced by Edgar F. Codd in his 1970 paper *"A Relational Model of Data for Large Shared Data Banks."* It represents data as **relations** (tables) consisting of **tuples** (rows) with a fixed set of **attributes** (columns). The key insight is separating the logical representation of data from its physical storage, allowing the query optimizer to decide how to execute queries.

As discussed in *Designing Data-Intensive Applications* (DDIA) Chapter 2, the relational model has proven remarkably durable — outliving network and hierarchical models — because it offers a clean, declarative interface for querying data. SQL hides the complexity of joins, indexes, and storage engines behind a simple abstraction.

### When to Use a Relational Database

- **Your data has structure** — entities, relationships, and well-defined schemas
- **You need ACID transactions** — financial systems, inventory, booking
- **You query data in many ways** — ad-hoc queries, joins across entities, aggregations
- **Data integrity is critical** — foreign keys, check constraints, unique constraints enforce correctness at the database level
- **You need a mature ecosystem** — ORMs, migration tools, monitoring, managed cloud offerings

### When to Consider Alternatives

- **Very high write throughput with flexible schema** — document databases (MongoDB)
- **Simple key-value lookups at extreme scale** — key-value stores (DynamoDB, Redis)
- **Highly connected data with recursive traversals** — graph databases (Neo4j)
- **Append-only event logs** — event stores, Kafka

```
    Relational Model

    ┌─────────────────────────────────────────────────────┐
    │  Relation (Table): users                            │
    ├──────┬──────────────┬───────────────┬───────────────┤
    │  id  │  name        │  email        │  created_at   │
    ├──────┼──────────────┼───────────────┼───────────────┤
    │  1   │  Alice       │  a@test.com   │  2025-01-15   │
    │  2   │  Bob         │  b@test.com   │  2025-02-20   │
    │  3   │  Charlie     │  c@test.com   │  2025-03-10   │
    └──────┴──────────────┴───────────────┴───────────────┘
      ▲         ▲              ▲               ▲
    Tuple     Attribute      Attribute       Attribute
    (Row)     (Column)       (Column)        (Column)
```

---

## SQL Fundamentals

SQL (Structured Query Language) is the standard interface for relational databases. It is divided into sub-languages by purpose.

### Data Definition Language (DDL)

DDL defines the structure of the database: tables, columns, constraints, and indexes.

```sql
-- Create a table with constraints
CREATE TABLE users (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    email       VARCHAR(255) NOT NULL UNIQUE,
    role        VARCHAR(20) NOT NULL DEFAULT 'member',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_role CHECK (role IN ('admin', 'member', 'viewer'))
);

-- Create a related table with a foreign key
CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    user_id     INT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    total       DECIMAL(12, 2) NOT NULL CHECK (total >= 0),
    status      VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Create an index
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Alter a table
ALTER TABLE users ADD COLUMN last_login TIMESTAMPTZ;

-- Drop a table (careful!)
DROP TABLE IF EXISTS orders;
```

### Data Manipulation Language (DML)

DML writes data: INSERT, UPDATE, DELETE.

```sql
-- Insert a single row
INSERT INTO users (name, email, role)
VALUES ('Alice', 'alice@example.com', 'admin');

-- Insert multiple rows
INSERT INTO users (name, email) VALUES
    ('Bob', 'bob@example.com'),
    ('Charlie', 'charlie@example.com'),
    ('Diana', 'diana@example.com');

-- Upsert (PostgreSQL): insert or update on conflict
INSERT INTO users (name, email, role)
VALUES ('Alice', 'alice@example.com', 'admin')
ON CONFLICT (email) DO UPDATE
    SET role = EXCLUDED.role,
        name = EXCLUDED.name;

-- Update with a WHERE clause
UPDATE orders
SET status = 'shipped',
    updated_at = now()
WHERE status = 'paid'
  AND created_at < now() - INTERVAL '2 days';

-- Delete with a subquery
DELETE FROM orders
WHERE user_id IN (
    SELECT id FROM users WHERE role = 'viewer'
);
```

### Data Query Language (DQL)

DQL retrieves data. SELECT is the most-used SQL statement.

```sql
-- Basic select with filtering and sorting
SELECT id, name, email
FROM users
WHERE role = 'admin'
ORDER BY created_at DESC
LIMIT 10;

-- Aggregation
SELECT status, COUNT(*) AS order_count, SUM(total) AS revenue
FROM orders
WHERE created_at >= '2025-01-01'
GROUP BY status
HAVING COUNT(*) > 5
ORDER BY revenue DESC;

-- Subquery in WHERE
SELECT name, email
FROM users
WHERE id IN (
    SELECT DISTINCT user_id
    FROM orders
    WHERE total > 500
);

-- Common Table Expression (CTE)
WITH high_value_customers AS (
    SELECT user_id, SUM(total) AS lifetime_value
    FROM orders
    GROUP BY user_id
    HAVING SUM(total) > 10000
)
SELECT u.name, u.email, hvc.lifetime_value
FROM users u
JOIN high_value_customers hvc ON u.id = hvc.user_id
ORDER BY hvc.lifetime_value DESC;
```

### Joins

Joins combine rows from two or more tables based on a related column.

```
    INNER JOIN             LEFT JOIN              FULL OUTER JOIN
    ┌─────┬─────┐         ┌─────┬─────┐         ┌─────┬─────┐
    │  A  │  B  │         │  A  │  B  │         │  A  │  B  │
    │     ┼━━━━━┥         │━━━━━┼━━━━━┥         │━━━━━┼━━━━━│
    │     │█████│         │█████│█████│         │█████│█████│
    │     ┼━━━━━┥         │━━━━━┼━━━━━┥         │━━━━━┼━━━━━│
    │     │     │         │     │     │         │     │     │
    └─────┴─────┘         └─────┴─────┘         └─────┴─────┘
    Only matching rows    All of A, matching B   All rows from both
```

```sql
-- INNER JOIN: only rows with matches in both tables
SELECT u.name, o.id AS order_id, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN: all users, even those without orders
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.name;

-- RIGHT JOIN: all orders, even if the user was deleted (rare in practice)
SELECT u.name, o.id AS order_id
FROM users u
RIGHT JOIN orders o ON u.id = o.user_id;

-- FULL OUTER JOIN: all rows from both tables
SELECT u.name, o.id AS order_id
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;

-- CROSS JOIN: cartesian product (every combination)
SELECT u.name, p.name AS product
FROM users u
CROSS JOIN products p;

-- Self join: employees and their managers
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

---

## Composing Good SQL Scripts

A good SQL script is predictable for the operator, safe for the database, and readable for the next engineer. Whether the script is a repeatable migration, a one-off repair, or an application query file, the quality bar is the same: make the intent obvious, make the blast radius small, and make the verification steps explicit.

### Script Design Principles

| Principle | What Good Looks Like | Common Failure Mode |
|---|---|---|
| **Deterministic** | Same input and same database state produce the same result | Non-deterministic updates, missing `ORDER BY`, hidden session settings |
| **Idempotent** | Safe to re-run or clearly guarded against re-execution | Duplicate rows, duplicate indexes, repeated backfills |
| **Bounded** | Every write has a precise predicate or batch boundary | `UPDATE table SET ...` without a restrictive `WHERE` clause |
| **Observable** | Script reports what changed and how it was validated | No row counts, no post-change checks, silent failures |
| **Vendor-aware** | Dialect-specific behavior is called out where it matters | Copying PostgreSQL syntax into MySQL or SQL Server unchanged |
| **Reversible** | Rollback or forward-fix strategy is documented before execution | Hotfixes that cannot be safely undone |

### Recommended Script Anatomy

1. **State the goal and execution context first.** Name the target schema, the expected preconditions, and whether the script is intended for PostgreSQL, MySQL, SQL Server, or ANSI SQL only.
2. **Separate DDL, data changes, and verification.** Schema changes, backfills, and validation queries should be easy to identify at a glance. Avoid mixing exploration queries into a deployment script.
3. **Prefer idempotent guards.** Use `IF EXISTS`, `IF NOT EXISTS`, `WHERE NOT EXISTS`, or metadata checks so that retries do not corrupt state.
4. **Control transactions deliberately.** Keep write transactions short, understand locking side effects, and remember that many MySQL DDL statements auto-commit even when wrapped in a transaction.
5. **Make risky writes bounded.** Large repairs should run in batches, and every destructive statement should include a narrow predicate plus a pre-flight `SELECT` using the same filter.
6. **Use precise types and stable names.** Match parameter types to column types, schema-qualify objects (`dbo.orders`, `sales.orders`), and avoid ambiguous aliases.
7. **Make validation part of the script.** Add row-count checks, null checks, duplicate checks, or index verification queries immediately after the change.
8. **Document rollback or forward-fix strategy.** For additive changes, rollback may mean disabling the feature and leaving the new column in place. For destructive changes, require a tested restore path first.

### Dialect Notes That Change Script Design

| Concern | PostgreSQL | MySQL | SQL Server |
|---|---|---|---|
| **Batch separator** | Standard `;` terminator | Standard `;`; `DELIMITER` only in client tools for routines | `GO` is a client-tool batch separator, not T-SQL syntax inside application queries |
| **DDL transactions** | Most DDL is transactional | Many DDL statements cause implicit commit | Many DDL statements can participate in a transaction, but some operations and tooling batches have restrictions |
| **Online schema change** | Often metadata-only or low-lock, but verify | Use `ALGORITHM=INSTANT/INPLACE` and `LOCK=NONE` when supported | Prefer online/rebuild options supported by the edition and operation |
| **Parameterized execution** | Prepared statements / bind variables | Prepared statements / bind variables | Prefer `sp_executesql` with typed parameters |
| **Return changed rows** | `RETURNING` is first-class | Feature support is narrower; often follow with explicit verification query | `OUTPUT inserted...` / `OUTPUT deleted...` |

### Annotated Example: a Safe Backfill Script

```sql
-- Goal: add archived_at to orders and backfill completed rows.
-- Preconditions: application no longer writes archived_at directly; backfill is safe to retry.

-- 1) Pre-flight check: know exactly how many rows should change.
SELECT COUNT(*) AS rows_to_archive
FROM orders
WHERE status IN ('completed', 'cancelled')
  AND archived_at IS NULL;

-- 2) Apply the smallest schema change first.
ALTER TABLE orders
ADD COLUMN archived_at TIMESTAMP NULL;

-- 3) Backfill using a bounded predicate.
UPDATE orders
SET archived_at = updated_at
WHERE status IN ('completed', 'cancelled')
  AND archived_at IS NULL;

-- 4) Validate the outcome immediately.
SELECT COUNT(*) AS remaining_nulls
FROM orders
WHERE status IN ('completed', 'cancelled')
  AND archived_at IS NULL;
```

Why this script is good:

- **The pre-flight query and the update share the same predicate.** That is the easiest way to catch an accidental full-table update before it happens.
- **The change is additive first.** Adding a nullable column is usually safer than adding a non-null column and trying to populate it in one risky step.
- **The validation query proves completion.** You should not rely on the absence of errors as proof that the script did the right thing.
- **The script is easy to adapt per engine.** In MySQL, validate whether the `ALTER TABLE` can use `ALGORITHM=INSTANT` or `LOCK=NONE`; in SQL Server, consider `SET XACT_ABORT ON` and batch large updates to protect the log and lock manager.

### Idempotency Patterns by Engine

Writing idempotent SQL varies across engines. Here are the most reliable patterns for the three most common scenarios: adding a column, creating an index, and upserting a row.

#### PostgreSQL

```sql
-- Idempotent column addition (PostgreSQL 9.6+)
ALTER TABLE orders ADD COLUMN IF NOT EXISTS archived_at TIMESTAMPTZ;

-- Idempotent index creation
CREATE INDEX IF NOT EXISTS idx_orders_archived_at ON orders(archived_at)
  WHERE archived_at IS NOT NULL;

-- Idempotent row upsert: insert or update on conflict
INSERT INTO config (config_key, config_value)
VALUES ('feature_x', 'enabled')
ON CONFLICT (config_key)
DO UPDATE SET config_value = EXCLUDED.config_value,
              updated_at   = now();
```

#### MySQL

MySQL 8.0+ supports `IF EXISTS` / `IF NOT EXISTS` in `ALTER TABLE`, but for older versions
you must guard using `information_schema`.

```sql
-- Check whether column exists before adding (works on all MySQL versions)
SET @col_exists = (
  SELECT COUNT(*) FROM information_schema.columns
  WHERE table_schema = DATABASE()
    AND table_name   = 'orders'
    AND column_name  = 'archived_at'
);

-- Execute ALTER only when the column is absent
-- (wrap in a stored procedure or handle in application/migration tool)
-- ALTER TABLE orders ADD COLUMN archived_at TIMESTAMP NULL, ALGORITHM=INSTANT;

-- Idempotent row upsert
INSERT INTO config (config_key, config_value)
VALUES ('feature_x', 'enabled')
ON DUPLICATE KEY UPDATE config_value = VALUES(config_value);
```

#### SQL Server

SQL Server provides `IF NOT EXISTS` guards via catalog views. Use `GO` to separate batches
inside `sqlcmd` / SSMS scripts.

```sql
-- Idempotent column addition
IF NOT EXISTS (
  SELECT 1 FROM sys.columns
  WHERE object_id = OBJECT_ID('dbo.orders')
    AND name = 'archived_at')
  ALTER TABLE dbo.orders ADD archived_at DATETIME2 NULL;
GO

-- Idempotent index creation
IF NOT EXISTS (
  SELECT 1 FROM sys.indexes
  WHERE object_id = OBJECT_ID('dbo.orders')
    AND name = 'IX_orders_archived_at')
  CREATE NONCLUSTERED INDEX IX_orders_archived_at
    ON dbo.orders (archived_at)
    WHERE archived_at IS NOT NULL;
GO

-- Idempotent row upsert (MERGE)
MERGE INTO dbo.config AS target
USING (VALUES ('feature_x', 'enabled')) AS src (config_key, config_value)
ON target.config_key = src.config_key
WHEN MATCHED     THEN UPDATE SET config_value = src.config_value
WHEN NOT MATCHED THEN INSERT (config_key, config_value)
                      VALUES (src.config_key, src.config_value);
```

### Batching Large Writes

Updating or deleting millions of rows in a single statement locks too many rows, bloats the
transaction log, and blocks concurrent traffic. Use small, restart-safe batches instead.

#### MySQL — loop with LIMIT

```sql
-- Archive 1 000 rows per iteration; safe to stop and restart
archive_loop: LOOP
  UPDATE orders
  SET    archived_at = updated_at
  WHERE  status IN ('completed', 'cancelled')
    AND  archived_at IS NULL
  LIMIT  1000;

  -- Exit when nothing is left to update
  IF ROW_COUNT() = 0 THEN
    LEAVE archive_loop;
  END IF;

  -- Brief pause so replicas can catch up before the next batch
  DO SLEEP(0.1);
END LOOP archive_loop;
```

#### PostgreSQL — CTE with LIMIT

```sql
-- Batch via writable CTE; repeat until 0 rows are returned
WITH batch AS (
  SELECT id FROM orders
  WHERE  status IN ('completed', 'cancelled')
    AND  archived_at IS NULL
  LIMIT  1000
  FOR UPDATE SKIP LOCKED
)
UPDATE orders
SET    archived_at = updated_at
WHERE  id IN (SELECT id FROM batch);

-- Run in a loop from the application or a DO block until no rows are updated.
-- The FOR UPDATE SKIP LOCKED avoids blocking other concurrent batch workers.
```

#### SQL Server — TOP with WHILE loop

```sql
SET XACT_ABORT ON;

DECLARE @batch_size INT = 1000;
DECLARE @rows_updated INT = 1;

WHILE @rows_updated > 0
BEGIN
  BEGIN TRANSACTION;

  UPDATE TOP (@batch_size) dbo.orders
  SET    archived_at = updated_at
  WHERE  status IN ('completed', 'cancelled')
    AND  archived_at IS NULL;

  SET @rows_updated = @@ROWCOUNT;
  COMMIT TRANSACTION;

  -- Brief pause between batches to reduce log pressure
  WAITFOR DELAY '00:00:00.100';
END
```

### Transaction Control by Engine

Each engine handles DDL inside transactions differently. Knowing this before writing a
deployment script prevents silent partial changes.

#### PostgreSQL — DDL is transactional

```sql
-- PostgreSQL DDL participates in transactions; schema changes roll back on error.
BEGIN;

ALTER TABLE orders ADD COLUMN IF NOT EXISTS priority SMALLINT NOT NULL DEFAULT 0;

UPDATE orders
SET    priority = 1
WHERE  customer_tier = 'gold';

-- Validate inside the transaction before committing
DO $$
BEGIN
  IF (SELECT COUNT(*) FROM orders
      WHERE customer_tier = 'gold' AND priority <> 1) > 0
  THEN
    RAISE EXCEPTION 'priority backfill validation failed — rolling back';
  END IF;
END;
$$;

COMMIT;
```

#### MySQL — DDL causes an implicit commit

```sql
-- Step 1: schema change. MySQL auto-commits this regardless of any wrapping transaction.
ALTER TABLE orders
  ADD COLUMN priority TINYINT NOT NULL DEFAULT 0,
  ALGORITHM = INSTANT;   -- fastest path; falls back to INPLACE if INSTANT is unavailable

-- Step 2: data change in an explicit transaction.
START TRANSACTION;

UPDATE orders
SET    priority = 1
WHERE  customer_tier = 'gold';

-- Validate before committing
SELECT COUNT(*) AS should_be_zero
FROM   orders
WHERE  customer_tier = 'gold'
  AND  priority <> 1;
-- If result = 0, the backfill is correct.

COMMIT;
```

#### SQL Server — DDL inside a transaction with XACT_ABORT

```sql
SET XACT_ABORT ON;   -- any runtime error triggers automatic rollback

BEGIN TRANSACTION;

-- Schema change participates in the transaction
ALTER TABLE dbo.orders ADD priority TINYINT NOT NULL DEFAULT 0;

-- Data backfill
UPDATE dbo.orders
SET    priority = 1
WHERE  customer_tier = 'gold';

-- Validate before committing
IF (SELECT COUNT(*)
    FROM   dbo.orders
    WHERE  customer_tier = 'gold' AND priority <> 1) > 0
BEGIN
  ROLLBACK TRANSACTION;
  RAISERROR('priority backfill validation failed', 16, 1);
  RETURN;
END

COMMIT TRANSACTION;
```

### SQL Script Review Checklist

Before running a script in staging or production, verify that all of the following are true:

- The script declares its target database engine and any version assumptions that matter.
- Every `UPDATE` or `DELETE` has a restrictive predicate and a matching dry-run `SELECT`.
- Long-running changes are split into small, restart-safe batches.
- Object names are schema-qualified and naming is consistent.
- Application-issued SQL uses parameters instead of string interpolation.
- The script includes verification queries, not just change statements.
- The operator knows whether rollback means `ROLLBACK`, a compensating migration, or restore-from-backup.

---

## Normalization

Normalization organizes tables to reduce data redundancy and prevent update anomalies. The goal is a design where each fact is stored in exactly one place.

### First Normal Form (1NF)

**Rule:** Every column contains atomic (indivisible) values. No repeating groups or arrays.

```
❌ Violates 1NF                      ✅ Satisfies 1NF

┌────┬────────┬───────────────────┐  ┌────┬────────┬──────────────┐
│ id │ name   │ phones            │  │ id │ name   │ phone        │
├────┼────────┼───────────────────┤  ├────┼────────┼──────────────┤
│ 1  │ Alice  │ 555-0101,555-0102 │  │ 1  │ Alice  │ 555-0101     │
│ 2  │ Bob    │ 555-0201          │  │ 1  │ Alice  │ 555-0102     │
└────┴────────┴───────────────────┘  │ 2  │ Bob    │ 555-0201     │
                                     └────┴────────┴──────────────┘
  Comma-separated list in one          One value per row. Or better:
  column — not atomic.                 separate phones table with FK.
```

### Second Normal Form (2NF)

**Rule:** 1NF + every non-key column depends on the **entire** primary key (not just part of a composite key).

```
❌ Violates 2NF (composite key: student_id + course_id)

┌────────────┬───────────┬──────────────┬───────┐
│ student_id │ course_id │ course_name  │ grade │
├────────────┼───────────┼──────────────┼───────┤
│ 1          │ CS101     │ Intro to CS  │ A     │
│ 2          │ CS101     │ Intro to CS  │ B     │  ← course_name depends
│ 1          │ MA201     │ Linear Alg   │ A-    │    only on course_id
└────────────┴───────────┴──────────────┴───────┘

✅ Satisfies 2NF: split into two tables

courses                             enrollments
┌───────────┬──────────────┐        ┌────────────┬───────────┬───────┐
│ course_id │ course_name  │        │ student_id │ course_id │ grade │
├───────────┼──────────────┤        ├────────────┼───────────┼───────┤
│ CS101     │ Intro to CS  │        │ 1          │ CS101     │ A     │
│ MA201     │ Linear Alg   │        │ 2          │ CS101     │ B     │
└───────────┴──────────────┘        │ 1          │ MA201     │ A-    │
                                    └────────────┴───────────┴───────┘
```

### Third Normal Form (3NF)

**Rule:** 2NF + no transitive dependencies — non-key columns depend only on the primary key, not on other non-key columns.

```
❌ Violates 3NF

┌────┬────────┬───────────────┬──────────────┐
│ id │ name   │ department_id │ dept_name    │
├────┼────────┼───────────────┼──────────────┤
│ 1  │ Alice  │ 10            │ Engineering  │  ← dept_name depends on
│ 2  │ Bob    │ 10            │ Engineering  │    department_id, not id
│ 3  │ Charlie│ 20            │ Marketing    │
└────┴────────┴───────────────┴──────────────┘

✅ Satisfies 3NF: extract the transitive dependency

employees                           departments
┌────┬────────┬───────────────┐     ┌───────────────┬──────────────┐
│ id │ name   │ department_id │     │ department_id │ dept_name    │
├────┼────────┼───────────────┤     ├───────────────┼──────────────┤
│ 1  │ Alice  │ 10            │     │ 10            │ Engineering  │
│ 2  │ Bob    │ 10            │     │ 20            │ Marketing    │
│ 3  │ Charlie│ 20            │     └───────────────┴──────────────┘
└────┴────────┴───────────────┘
```

### Boyce-Codd Normal Form (BCNF)

**Rule:** 3NF + every determinant is a candidate key. BCNF handles the rare case where a non-candidate-key attribute determines part of a candidate key.

```sql
-- Example: A professor teaches only one subject, but a subject
-- can be taught by multiple professors.
-- Candidate keys: (student, subject) and (student, professor)
-- professor → subject is a functional dependency, but professor
-- is not a candidate key. This violates BCNF.

-- ❌ Not in BCNF
-- student_courses(student, subject, professor)

-- ✅ BCNF: decompose
-- professor_subjects(professor PK, subject)
-- student_professors(student, professor) PK(student, professor)
```

### When to Denormalize

Normalization optimizes for writes and data integrity. Denormalization optimizes for read performance by trading redundancy for fewer joins.

| Denormalize When | Stay Normalized When |
|---|---|
| Read-heavy workloads (90%+ reads) | Write-heavy workloads |
| Complex joins are killing query performance | Data integrity is paramount |
| Caching the join result is impractical | Storage cost matters |
| Reporting / OLAP queries need pre-joined data | The schema is still evolving |
| You can tolerate stale or duplicated data | You need a single source of truth |

**Common denormalization strategies:**

- **Materialized views** — precomputed join results refreshed periodically
- **Computed columns** — store `order_count` on the user row, updated by triggers
- **Embedding** — store `customer_name` on the order row (acceptable for immutable snapshots like invoices)

---

## Indexing

Indexes are separate data structures that allow the database to find rows without scanning the entire table. As DDIA Chapter 3 explains, the trade-off is always: **faster reads at the cost of slower writes and additional storage**.

### B-Tree Indexes

The default index type in nearly every relational database. B-Trees keep data sorted, enabling efficient lookups, range scans, and ordering.

```
    B-Tree Index on users(email)

              ┌─────────────────┐
              │   d@test.com    │          Root node
              └────┬───────┬────┘
                   │       │
         ┌─────────▼──┐  ┌─▼──────────┐
         │ a@  │ c@   │  │ f@  │ m@   │  Internal nodes
         └──┬────┬──┬─┘  └─┬────┬──┬──┘
            │    │  │      │    │  │
           ▼    ▼  ▼     ▼    ▼  ▼     Leaf nodes (row pointers)
```

**Characteristics:**
- **O(log n)** lookups — a table with 10 million rows needs ~24 page reads (assuming branching factor of ~500)
- Supports equality (`=`) and range queries (`<`, `>`, `BETWEEN`, `LIKE 'abc%'`)
- Keeps data sorted — useful for `ORDER BY` without a separate sort step
- Self-balancing — performance stays consistent as data grows

```sql
-- B-Tree index: the default
CREATE INDEX idx_users_email ON users(email);

-- Unique index: enforces uniqueness + provides index lookup
CREATE UNIQUE INDEX idx_users_email ON users(email);
```

### Hash Indexes

Hash indexes provide **O(1)** lookups for exact equality comparisons but cannot support range queries.

```sql
-- PostgreSQL hash index (niche use case)
CREATE INDEX idx_sessions_token ON sessions USING hash (token);
-- Only useful for: WHERE token = 'abc123'
-- Cannot do: WHERE token > 'abc' or ORDER BY token
```

**In practice:** B-Tree indexes handle equality queries nearly as fast, so hash indexes are rarely used. The query planner can use a B-Tree for both equality and range queries.

### GIN and GiST Indexes

PostgreSQL-specific index types for specialized data.

| Index Type | Best For | Example Use Cases |
|---|---|---|
| **GIN** (Generalized Inverted Index) | Multi-valued data | Full-text search, JSONB containment, array elements |
| **GiST** (Generalized Search Tree) | Spatial / geometric data | PostGIS geometry, range types, nearest-neighbor |

```sql
-- GIN index on a JSONB column
CREATE INDEX idx_products_attrs ON products USING gin (attributes);

-- Enables queries like:
SELECT * FROM products
WHERE attributes @> '{"color": "red", "size": "L"}';

-- GIN index for full-text search
CREATE INDEX idx_articles_search ON articles
USING gin (to_tsvector('english', title || ' ' || body));

-- GiST index for PostGIS spatial queries
CREATE INDEX idx_locations_geom ON locations USING gist (geom);

-- Enables queries like:
SELECT name FROM locations
WHERE ST_DWithin(geom, ST_MakePoint(-73.99, 40.73)::geography, 1000);
```

### Composite Indexes

An index on multiple columns. Column order matters — the index is useful for queries that filter on a **left prefix** of the indexed columns.

```sql
CREATE INDEX idx_orders_status_created ON orders(status, created_at DESC);

-- ✅ Uses the index (left prefix match)
SELECT * FROM orders WHERE status = 'pending';
SELECT * FROM orders WHERE status = 'pending' AND created_at > '2025-01-01';
SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at DESC;

-- ❌ Cannot use this index efficiently
SELECT * FROM orders WHERE created_at > '2025-01-01';
-- (created_at is the second column; use a separate index)
```

### Covering Indexes

A covering index includes all columns needed by a query, allowing the database to answer the query from the index alone without fetching the table row (an **index-only scan**).

```sql
-- Covering index using INCLUDE (PostgreSQL 11+)
CREATE INDEX idx_orders_covering ON orders(user_id)
INCLUDE (total, status);

-- This query is answered entirely from the index:
SELECT total, status FROM orders WHERE user_id = 42;

-- SQL Server equivalent:
-- CREATE INDEX idx_orders_covering ON orders(user_id) INCLUDE (total, status);
```

### Partial Indexes

An index that only includes rows matching a condition. Smaller, faster, and cheaper than a full index.

```sql
-- Only index active users (skip millions of inactive rows)
CREATE INDEX idx_users_active_email ON users(email)
WHERE active = true;

-- Only index unprocessed orders
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';
```

### When to Index

```
    Decision Framework

    Is the column used in WHERE, JOIN, or ORDER BY?
    │
    ├─ No  → Do not index
    │
    └─ Yes → Is the table small (< 1000 rows)?
             │
             ├─ Yes → Probably not worth it (sequential scan is fast)
             │
             └─ No  → Is the column selective (filters out > 80% of rows)?
                      │
                      ├─ Yes → CREATE INDEX ✅
                      │
                      └─ No  → Will the query run frequently?
                               │
                               ├─ Yes → Index, but monitor write overhead
                               │
                               └─ No  → Skip it; add later if needed
```

**Rules of thumb:**
- Always index foreign keys
- Always index columns used in `WHERE`, `JOIN ON`, and `ORDER BY`
- Use `EXPLAIN ANALYZE` to verify the index is actually used
- Monitor index usage with `pg_stat_user_indexes` (PostgreSQL) or `sys.dm_db_index_usage_stats` (SQL Server)
- Remove unused indexes — they slow down writes for no benefit

---

## Transactions

A transaction groups multiple operations into a single logical unit that either fully succeeds or fully fails. DDIA Chapter 7 provides the most thorough treatment of transaction isolation in practice.

### ACID Deep-Dive

**Atomicity** — "all or nothing." If any statement in a transaction fails, all changes are rolled back. The database is never left in a partially-updated state.

```sql
-- Atomicity example: transferring money
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- If the second UPDATE fails, the first is rolled back.
```

**Consistency** — the transaction moves the database from one valid state to another. All constraints (foreign keys, CHECK, UNIQUE, NOT NULL) are enforced. Note: in DDIA, Kleppmann argues that consistency is really a property of the *application*, not the database — the database can only enforce declared constraints.

**Isolation** — concurrent transactions do not interfere with each other. The level of isolation varies (see below). Perfect isolation (serializability) behaves as if transactions ran one at a time, but is expensive.

**Durability** — once COMMIT returns success, the data is safe on disk (written to the write-ahead log). Even if the server crashes immediately after, the data survives.

### Isolation Levels and Anomalies

Each isolation level prevents certain concurrency anomalies while allowing others. Understanding these anomalies is critical for building correct systems.

#### Dirty Read

A transaction reads data written by another transaction that has not yet committed.

```sql
-- Transaction A                     -- Transaction B
BEGIN;
UPDATE accounts
SET balance = 0 WHERE id = 1;
                                     BEGIN;
                                     -- Dirty read: sees balance = 0
                                     -- even though A hasn't committed
                                     SELECT balance FROM accounts
                                     WHERE id = 1;
ROLLBACK;                            -- B acted on data that was rolled back!
```

**Prevented by:** Read Committed and above.

#### Non-Repeatable Read

A transaction reads the same row twice and gets different values because another transaction committed a change in between.

```sql
-- Transaction A                     -- Transaction B
BEGIN;
SELECT balance FROM accounts
WHERE id = 1;
-- Returns: 500
                                     BEGIN;
                                     UPDATE accounts
                                     SET balance = 300 WHERE id = 1;
                                     COMMIT;
SELECT balance FROM accounts
WHERE id = 1;
-- Returns: 300 (different!)
COMMIT;
```

**Prevented by:** Repeatable Read and above (snapshot isolation).

#### Phantom Read

A transaction re-executes a range query and sees new rows that were inserted by another committed transaction.

```sql
-- Transaction A                     -- Transaction B
BEGIN;
SELECT COUNT(*) FROM bookings
WHERE room = '101'
  AND date = '2025-07-01';
-- Returns: 0
                                     BEGIN;
                                     INSERT INTO bookings
                                       (room, date, guest)
                                     VALUES ('101', '2025-07-01', 'Bob');
                                     COMMIT;
SELECT COUNT(*) FROM bookings
WHERE room = '101'
  AND date = '2025-07-01';
-- Returns: 1 (phantom row appeared!)
COMMIT;
```

**Prevented by:** Serializable isolation.

#### Write Skew (DDIA Ch. 7)

Two transactions each read the same data, make decisions based on what they read, and then write — but neither sees the other's write. The result violates an invariant that should hold.

```sql
-- Invariant: At least one doctor must be on call at all times.
-- Currently Alice and Bob are both on call.

-- Transaction A (Alice)             -- Transaction B (Bob)
BEGIN;                               BEGIN;
SELECT COUNT(*) FROM doctors         SELECT COUNT(*) FROM doctors
WHERE on_call = true;                WHERE on_call = true;
-- Returns: 2 (safe to go off)      -- Returns: 2 (safe to go off)

UPDATE doctors                       UPDATE doctors
SET on_call = false                  SET on_call = false
WHERE name = 'Alice';                WHERE name = 'Bob';
COMMIT;                              COMMIT;

-- Result: ZERO doctors on call! Both transactions saw 2, both decided
-- it was safe to remove themselves. Write skew is not prevented by
-- Repeatable Read — it requires Serializable isolation or explicit
-- locking (SELECT ... FOR UPDATE).
```

**Prevented by:** Serializable isolation or application-level locking (`SELECT ... FOR UPDATE`).

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Write Skew |
|---|---|---|---|---|
| **Read Uncommitted** | Possible | Possible | Possible | Possible |
| **Read Committed** | Prevented | Possible | Possible | Possible |
| **Repeatable Read** | Prevented | Prevented | Possible* | Possible |
| **Serializable** | Prevented | Prevented | Prevented | Prevented |

*\* PostgreSQL's Repeatable Read uses snapshot isolation, which prevents phantoms in most cases but not write skew.*

### Explicit Transactions

```sql
-- Basic transaction
BEGIN;
    INSERT INTO orders (user_id, total) VALUES (1, 99.99);
    INSERT INTO order_items (order_id, product_id, qty)
    VALUES (currval('orders_id_seq'), 42, 2);
COMMIT;

-- Savepoints: partial rollback within a transaction
BEGIN;
    INSERT INTO orders (user_id, total) VALUES (1, 99.99);
    SAVEPOINT before_items;
    INSERT INTO order_items (order_id, product_id, qty)
    VALUES (currval('orders_id_seq'), 9999, 1);  -- might fail
    -- If the insert fails:
    ROLLBACK TO before_items;
    -- Continue with alternative logic...
COMMIT;

-- SELECT FOR UPDATE: pessimistic locking to prevent write skew
BEGIN;
    SELECT * FROM doctors
    WHERE on_call = true
    FOR UPDATE;  -- locks all matching rows

    -- Now safe to check count and update
    UPDATE doctors SET on_call = false WHERE name = 'Alice';
COMMIT;

-- Advisory locks (PostgreSQL): application-level locking
SELECT pg_advisory_lock(hashtext('process-payments'));
-- ... critical section ...
SELECT pg_advisory_unlock(hashtext('process-payments'));
```

---

## PostgreSQL

PostgreSQL is the most advanced open-source relational database. It prioritizes correctness, extensibility, and standards compliance.

### Key Features

**MVCC (Multi-Version Concurrency Control):** PostgreSQL never locks rows for reading. Each transaction sees a snapshot of the database at the time it started. Writers create new row versions rather than overwriting. Old versions are cleaned up by the `VACUUM` process. This means readers never block writers, and writers never block readers.

```
    MVCC: How concurrent access works

    Transaction A (read)           Storage (heap)
    ┌──────────────────┐          ┌──────────────────────────────┐
    │ SELECT * FROM t  │          │ Row version 1 (xmin=100)     │ ← A sees this
    │ (snapshot: txn   │          │ Row version 2 (xmin=105)     │ ← B wrote this
    │  id = 103)       │          │ (invisible to A — txn 105    │
    └──────────────────┘          │  started after A's snapshot) │
                                  └──────────────────────────────┘
    Transaction B (write)
    ┌──────────────────┐
    │ UPDATE t SET ... │   Creates version 2, does NOT block A's read
    │ (txn id = 105)   │
    └──────────────────┘
```

**JSONB:** Binary JSON storage with indexing. Combines the flexibility of a document store with the power of a relational database. Use it for semi-structured data that doesn't warrant its own table.

```sql
-- Store and query JSONB
CREATE TABLE events (
    id      SERIAL PRIMARY KEY,
    type    VARCHAR(50) NOT NULL,
    payload JSONB NOT NULL DEFAULT '{}'
);

INSERT INTO events (type, payload) VALUES
('order.created', '{"order_id": 123, "items": [{"sku": "A1", "qty": 2}]}');

-- Query nested fields
SELECT payload->>'order_id' AS order_id
FROM events
WHERE type = 'order.created'
  AND payload @> '{"items": [{"sku": "A1"}]}';

-- Index JSONB for fast containment queries
CREATE INDEX idx_events_payload ON events USING gin (payload);
```

**CTEs and Window Functions:**

```sql
-- Recursive CTE: organizational hierarchy
WITH RECURSIVE org_chart AS (
    SELECT id, name, manager_id, 1 AS depth
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.id, e.name, e.manager_id, oc.depth + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT * FROM org_chart ORDER BY depth, name;

-- Window function: running total and rank
SELECT
    date,
    revenue,
    SUM(revenue) OVER (ORDER BY date) AS running_total,
    RANK() OVER (ORDER BY revenue DESC) AS revenue_rank
FROM daily_sales;

-- Window function: compare each row to the previous
SELECT
    date,
    revenue,
    revenue - LAG(revenue) OVER (ORDER BY date) AS day_over_day_change
FROM daily_sales;
```

**Extensions:**

| Extension | Purpose |
|---|---|
| **PostGIS** | Geospatial data types, spatial indexing, distance queries |
| **pg_trgm** | Trigram-based fuzzy text matching and similarity search |
| **pg_stat_statements** | Track query execution statistics for performance tuning |
| **hstore** | Key-value pairs within a single column |
| **uuid-ossp** / **pgcrypto** | UUID generation |
| **pg_partman** | Automated table partitioning management |
| **TimescaleDB** | Time-series hypertables on top of PostgreSQL |
| **Citus** | Distributed (sharded) PostgreSQL |

### Operational Notes

- **VACUUM:** MVCC creates dead row versions. `autovacuum` reclaims space and updates planner statistics. Monitor `pg_stat_user_tables.n_dead_tup` — if dead tuples pile up, autovacuum is falling behind.
- **WAL (Write-Ahead Log):** Every change is written to the WAL before being applied to data files. This guarantees durability and enables point-in-time recovery and streaming replication.
- **Connection pooling:** PostgreSQL forks a process per connection. Use PgBouncer or pgpool-II in front of it for connection pooling. A typical production PostgreSQL instance handles 200–500 direct connections comfortably.
- **Default isolation level:** Read Committed. PostgreSQL's Repeatable Read uses true snapshot isolation via MVCC (called Serializable Snapshot Isolation at the Serializable level).
- **`EXPLAIN ANALYZE`:** The most important performance tool. Always run it to verify that queries use indexes as expected.

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 42 AND status = 'pending';

-- Look for:
-- ✅ Index Scan / Index Only Scan
-- ❌ Seq Scan on large tables (missing index)
-- ❌ Nested Loop with large outer table (consider hash/merge join)
```

---

## MySQL

MySQL is the most widely deployed open-source relational database, powering much of the web (the "M" in LAMP).

### InnoDB vs MyISAM

InnoDB has been the default storage engine since MySQL 5.5. MyISAM is legacy and should not be used for new projects.

| Feature | InnoDB | MyISAM |
|---|---|---|
| **Transactions** | Full ACID | No |
| **Row-level locking** | Yes | Table-level only |
| **Foreign keys** | Yes | No |
| **Crash recovery** | Yes (redo log) | No (requires repair) |
| **MVCC** | Yes | No |
| **Full-text search** | Yes (since 5.6) | Yes |
| **Clustered index** | Yes (PK = data order) | No (heap) |
| **Count(*)** | Slow (MVCC row count) | Fast (stored metadata) |
| **Use case** | Everything | Read-only archival (avoid) |

### InnoDB Architecture

```
    InnoDB Storage Engine

    ┌──────────────────────────────────────────┐
    │              Buffer Pool                  │
    │  (caches data + index pages in RAM)       │
    │  ┌──────────┐  ┌──────────┐  ┌────────┐ │
    │  │Data Pages│  │Idx Pages │  │Undo Log│ │
    │  └──────────┘  └──────────┘  └────────┘ │
    └──────────────┬───────────────────────────┘
                   │
    ┌──────────────▼───────────────────────────┐
    │            Redo Log (WAL)                 │
    │  Guarantees durability. Sequential I/O.   │
    └──────────────┬───────────────────────────┘
                   │
    ┌──────────────▼───────────────────────────┐
    │          Tablespace Files (.ibd)          │
    │  Actual data on disk. Organized as a      │
    │  clustered index (B+ Tree on PK).         │
    └──────────────────────────────────────────┘
```

**Key point:** InnoDB stores table data in primary key order (clustered index). This means:
- Primary key lookups are extremely fast (data is right there in the leaf node)
- Auto-increment integer PKs produce sequential inserts (efficient for B-Trees)
- Random UUIDs as PKs cause page splits and fragmentation — use `uuid_to_bin()` with swap flag or ULIDs

### Key Features

- **Replication modes:**

| Mode | How It Works | Trade-off |
|---|---|---|
| **Asynchronous** | Primary doesn't wait for replicas | Fastest. Risk of data loss on failover |
| **Semi-synchronous** | Primary waits for at least one replica ACK | Balanced. Slight latency increase |
| **Group Replication** | Paxos-based consensus across nodes | Strongest consistency. Higher latency |

- **GTID (Global Transaction Identifiers):** Each transaction gets a unique ID across the cluster, making failover and replica reconfiguration reliable.
- **Online DDL:** Many ALTER TABLE operations can now run without locking the table (using `ALGORITHM=INPLACE` or `ALGORITHM=INSTANT`).
- **InnoDB Cluster:** Built-in high availability with MySQL Shell, Group Replication, and MySQL Router.

### Character Sets and Collation

A common source of bugs and performance issues in MySQL.

```sql
-- Always use utf8mb4, not utf8 (which is only 3 bytes, missing emoji and CJK)
CREATE TABLE posts (
    id      INT AUTO_INCREMENT PRIMARY KEY,
    title   VARCHAR(255) NOT NULL,
    body    TEXT NOT NULL
) ENGINE=InnoDB
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_unicode_ci;

-- Check current settings
SHOW VARIABLES LIKE 'character_set%';
SHOW VARIABLES LIKE 'collation%';

-- Fix a common mistake: convert an existing table
ALTER TABLE posts CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

| Collation | Behavior | Use Case |
|---|---|---|
| `utf8mb4_general_ci` | Fast, approximate comparisons | Legacy default |
| `utf8mb4_unicode_ci` | Correct Unicode sorting | Most applications |
| `utf8mb4_0900_ai_ci` | MySQL 8.0+ default, UCA 9.0 | New projects on MySQL 8 |
| `utf8mb4_bin` | Byte-by-byte comparison | Case-sensitive exact matching |

### MySQL Operational Notes

- **`innodb_buffer_pool_size`:** The single most important tuning parameter. Set to 70–80% of available RAM on a dedicated database server.
- **Slow query log:** Enable it and set `long_query_time = 1` (seconds) to catch problematic queries.
- **`EXPLAIN` format:** Use `EXPLAIN FORMAT=TREE` (MySQL 8.0+) or `EXPLAIN ANALYZE` (8.0.18+) for modern query plan output.
- **Default isolation level:** Repeatable Read (unlike PostgreSQL's Read Committed). This can cause unexpected locking in workloads with many concurrent writes.

---

## SQL Server

SQL Server is Microsoft's flagship relational database, widely used for enterprise OLTP, reporting, BI, and .NET-centric systems. Its strengths are strong tooling, mature operational features, and deep integration with the Microsoft ecosystem.

### Key Features

- **T-SQL:** Rich procedural SQL dialect with variables, error handling, table-valued parameters, and strong administrative tooling.
- **Query Store:** Built-in plan history, regression detection, and plan forcing for stabilizing important workloads.
- **Availability features:** Always On availability groups, failover clustering, log shipping, and replication options for different recovery objectives.
- **Indexing options:** Clustered and nonclustered rowstore indexes, filtered indexes, columnstore indexes, and indexed views.
- **Security and governance:** Row-level security, dynamic data masking, transparent data encryption, auditing, and integration with Active Directory.

### Storage and Indexing Model

SQL Server tables are typically organized around a **clustered index**. If you do not define one, the table may remain a heap, which is often undesirable for hot OLTP workloads because updates can create forwarded records and make lookups more expensive.

**Rules of thumb:**

- Choose a **narrow, stable, ever-increasing clustered key** for write-heavy OLTP tables.
- Use **nonclustered indexes** for lookup patterns and include extra columns when you want covering behavior.
- Use **columnstore indexes** for analytical queries and large aggregations, not as a default for transactional tables.
- Watch for **implicit conversions** between parameters and indexed columns, because they can turn seeks into scans.

### Concurrency and Isolation

SQL Server defaults to **Read Committed**. In many mixed read/write systems, enabling **Read Committed Snapshot Isolation (RCSI)** is an effective way to reduce reader/writer blocking without forcing every query to use unsafe hints. Snapshot-based isolation moves older row versions into `tempdb`, so monitor version-store growth and long-running transactions.

### SQL Server Operational Notes

- **Enable Query Store** in production and review regressed plans before forcing hints or rewriting queries.
- **Pre-size the transaction log and `tempdb`.** Treat autogrowth as a safety net, not a sizing strategy.
- **Use schema-qualified names** and typed parameters so execution plans are predictable.
- **Prefer `sp_executesql` over `EXEC()`** for dynamic SQL to get plan reuse and protection from SQL injection.
- **Use `SET XACT_ABORT ON`** in deployment and data-fix scripts so runtime errors abort the entire transaction instead of leaving partial changes behind.

### T-SQL Examples

**Defining a table with an explicit clustered key and a covering nonclustered index:**

```sql
-- Clustered by IDENTITY key for sequential inserts; nonclustered for customer lookups
CREATE TABLE dbo.orders (
    order_id    BIGINT        NOT NULL IDENTITY(1,1),
    customer_id BIGINT        NOT NULL,
    status      NVARCHAR(20)  NOT NULL DEFAULT N'pending',
    total_cents BIGINT        NOT NULL,
    created_at  DATETIME2     NOT NULL DEFAULT SYSUTCDATETIME(),
    CONSTRAINT PK_orders PRIMARY KEY CLUSTERED (order_id)
);

-- Covering nonclustered: avoids a key lookup for the common customer-order query
CREATE NONCLUSTERED INDEX IX_orders_customer_status
  ON dbo.orders (customer_id, status)
  INCLUDE (total_cents, created_at);

-- Filtered nonclustered: smaller and faster for the subset that matters
CREATE NONCLUSTERED INDEX IX_orders_pending
  ON dbo.orders (created_at)
  WHERE status = N'pending';
```

**Explicit transaction with structured error handling:**

```sql
SET XACT_ABORT ON;

BEGIN TRY
  BEGIN TRANSACTION;

  -- Debit sender
  UPDATE dbo.accounts
  SET    balance = balance - 100
  WHERE  account_id = 1;

  -- Credit recipient
  UPDATE dbo.accounts
  SET    balance = balance + 100
  WHERE  account_id = 2;

  COMMIT TRANSACTION;
END TRY
BEGIN CATCH
  IF @@TRANCOUNT > 0
    ROLLBACK TRANSACTION;

  -- Re-raise the original error to the caller
  THROW;
END CATCH;
```

**Enabling Query Store and RCSI:**

```sql
-- Enable Query Store (persists execution plan history across restarts)
ALTER DATABASE appdb
  SET QUERY_STORE = ON
  WITH (
    OPERATION_MODE        = READ_WRITE,
    MAX_STORAGE_SIZE_MB   = 1024,
    QUERY_CAPTURE_MODE    = AUTO
  );

-- Enable RCSI (readers no longer block on writers; requires brief exclusive lock on the DB)
ALTER DATABASE appdb SET READ_COMMITTED_SNAPSHOT ON;

-- Verify the setting
SELECT name, is_read_committed_snapshot_on
FROM   sys.databases
WHERE  name = DB_NAME();
```

**Parameterized dynamic SQL to get plan reuse:**

```sql
-- ❌ Anti-pattern: concatenated SQL gets a separate plan each time
EXEC ('SELECT total_amount FROM sales.orders WHERE customer_id = ' + @id);

-- ✅ Correct: sp_executesql with typed parameter — one plan is compiled and reused
EXEC sp_executesql
    N'SELECT total_amount FROM sales.orders WHERE customer_id = @customer_id',
    N'@customer_id BIGINT',
    @customer_id = 42;
```

### SQL Server Monitoring Queries

**Top queries by average CPU time (Query Store):**

```sql
SELECT TOP 20
    OBJECT_NAME(qsq.object_id)           AS proc_name,
    qsq.query_id,
    qsrs.count_executions,
    ROUND(qsrs.avg_cpu_time   / 1000.0, 2) AS avg_cpu_ms,
    ROUND(qsrs.avg_duration   / 1000.0, 2) AS avg_duration_ms,
    ROUND(qsrs.avg_logical_io_reads, 0)    AS avg_logical_reads,
    LEFT(qsqt.query_sql_text, 200)         AS query_text
FROM sys.query_store_runtime_stats   AS qsrs
JOIN sys.query_store_plan            AS qsqp ON qsqp.plan_id        = qsrs.plan_id
JOIN sys.query_store_query           AS qsq  ON qsq.query_id        = qsqp.query_id
JOIN sys.query_store_query_text      AS qsqt ON qsqt.query_text_id  = qsq.query_text_id
ORDER BY qsrs.avg_cpu_time DESC;
```

**Wait statistics — find the root cause of latency:**

```sql
SELECT TOP 20
    wait_type,
    waiting_tasks_count,
    ROUND(wait_time_ms / 1000.0, 1)        AS wait_time_sec,
    ROUND(signal_wait_time_ms / 1000.0, 1) AS cpu_queue_sec,
    ROUND(
        100.0 * wait_time_ms
        / NULLIF(SUM(wait_time_ms) OVER (), 0), 2)  AS pct_of_total
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
    'SLEEP_TASK','BROKER_TO_FLUSH','BROKER_EVENTHANDLER',
    'REQUEST_FOR_DEADLOCK_SEARCH','CHECKPOINT_QUEUE',
    'SQLTRACE_BUFFER_FLUSH','CLR_AUTO_EVENT',
    'DISPATCHER_QUEUE_SEMAPHORE','XE_TIMER_EVENT',
    'FT_IFTS_SCHEDULER_IDLE_WAIT','WAITFOR',
    'LAZYWRITER_SLEEP','LOGMGR_QUEUE',
    'ONDEMAND_TASK_QUEUE','XE_DISPATCHER_WAIT')
ORDER BY wait_time_ms DESC;
```

**Find tables stored as heaps (no clustered index):**

```sql
SELECT
    OBJECT_SCHEMA_NAME(i.object_id)  AS schema_name,
    OBJECT_NAME(i.object_id)         AS table_name,
    SUM(p.rows)                      AS row_count
FROM sys.indexes    AS i
JOIN sys.partitions AS p
  ON p.object_id = i.object_id AND p.index_id = i.index_id
WHERE i.type = 0   -- 0 = HEAP (no clustered index)
  AND OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
GROUP BY i.object_id
ORDER BY row_count DESC;
```

---

## Comparison Table

| Feature | PostgreSQL | MySQL (InnoDB) | SQL Server |
|---|---|---|---|
| **License** | PostgreSQL License (OSS) | GPL v2 / Commercial | Commercial |
| **Default isolation** | Read Committed | Repeatable Read | Read Committed |
| **MVCC** | Yes (tuple versioning) | Yes (undo log) | Yes (row versioning in RCSI) |
| **JSON support** | JSONB (binary, indexed) | JSON (text-based, since 5.7) | JSON functions (since 2016) |
| **Full-text search** | Built-in (`tsvector/tsquery`) | Built-in (InnoDB) | Built-in (full-text catalog) |
| **Geospatial** | PostGIS extension | Spatial indexes (basic) | Spatial types (built-in) |
| **Partitioning** | Declarative (10+) | Range, List, Hash, Key | Partition functions + schemes |
| **Window functions** | Full support | Since 8.0 | Full support |
| **CTEs (WITH)** | Full (including recursive) | Since 8.0 | Full (including recursive) |
| **Materialized views** | Yes | No (use workarounds) | Yes (indexed views) |
| **Logical replication** | Yes (pub/sub, since 10) | Binlog-based | Always On, transactional |
| **Extensions** | Rich ecosystem (PostGIS, TimescaleDB, Citus) | Plugins (limited) | CLR integration |
| **Stored procedures** | PL/pgSQL, PL/Python, PL/V8 | SQL/PSM | T-SQL |
| **Max DB size** | Unlimited | 256 TB per table | 524 PB |
| **Connection model** | Process per connection | Thread per connection | Thread per connection |
| **Managed cloud** | AWS RDS/Aurora, GCP Cloud SQL, Azure | AWS RDS/Aurora, GCP Cloud SQL, Azure | Azure SQL, AWS RDS |
| **Best for** | Complex queries, extensibility, correctness | Web apps, read-heavy, wide ecosystem | Enterprise .NET, BI, SSRS/SSIS |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.1 | 2026 | Added comprehensive SQL script composition guidance and SQL Server coverage |
| 1.0 | 2025 | Initial relational databases documentation |
