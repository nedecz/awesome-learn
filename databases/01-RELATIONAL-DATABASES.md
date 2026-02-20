# Relational Databases

## Table of Contents

1. [Overview](#overview)
2. [SQL Fundamentals](#sql-fundamentals)
   - [Data Definition Language (DDL)](#data-definition-language-ddl)
   - [Data Manipulation Language (DML)](#data-manipulation-language-dml)
   - [Data Query Language (DQL)](#data-query-language-dql)
   - [Joins](#joins)
3. [Normalization](#normalization)
   - [First Normal Form (1NF)](#first-normal-form-1nf)
   - [Second Normal Form (2NF)](#second-normal-form-2nf)
   - [Third Normal Form (3NF)](#third-normal-form-3nf)
   - [Boyce-Codd Normal Form (BCNF)](#boyce-codd-normal-form-bcnf)
   - [When to Denormalize](#when-to-denormalize)
4. [Indexing](#indexing)
   - [B-Tree Indexes](#b-tree-indexes)
   - [Hash Indexes](#hash-indexes)
   - [GIN and GiST Indexes](#gin-and-gist-indexes)
   - [Composite Indexes](#composite-indexes)
   - [Covering Indexes](#covering-indexes)
   - [Partial Indexes](#partial-indexes)
   - [When to Index](#when-to-index)
5. [Transactions](#transactions)
   - [ACID Deep-Dive](#acid-deep-dive)
   - [Isolation Levels and Anomalies](#isolation-levels-and-anomalies)
   - [Write Skew and Phantoms](#write-skew-and-phantoms)
   - [Explicit Transactions](#explicit-transactions)
6. [PostgreSQL](#postgresql)
7. [MySQL](#mysql)
8. [Comparison Table](#comparison-table)
9. [Version History](#version-history)

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
| 1.0 | 2025 | Initial relational databases documentation |
