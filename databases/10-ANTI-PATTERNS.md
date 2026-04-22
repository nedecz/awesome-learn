# Database Anti-Patterns

A catalogue of the most common database mistakes — what they look like, why they are harmful, and exactly how to fix them. Use this document as a code-review checklist, a pre-production gate, or a team learning resource.

---

## Table of Contents

- [Introduction](#introduction)
- [Anti-Patterns Summary Table](#anti-patterns-summary-table)
- [1. God Table](#1-god-table)
- [2. Missing Indexes](#2-missing-indexes)
- [3. N+1 Query Problem](#3-n1-query-problem)
- [4. Soft Deletes Everywhere](#4-soft-deletes-everywhere)
- [5. Storing Money as Floating Point](#5-storing-money-as-floating-point)
- [6. Entity-Attribute-Value (EAV) Abuse](#6-entity-attribute-value-eav-abuse)
- [7. No Connection Pooling](#7-no-connection-pooling)
- [8. Ignoring Replication Lag](#8-ignoring-replication-lag)
- [9. Unbounded Queries](#9-unbounded-queries)
- [10. Premature Sharding](#10-premature-sharding)
- [11. Not Using Transactions](#11-not-using-transactions)
- [12. Schema Changes Without Migration Tools](#12-schema-changes-without-migration-tools)
- [13. MySQL and SQL Server Engine-Specific Anti-Patterns](#13-mysql-and-sql-server-engine-specific-anti-patterns)
- [Quick Reference Checklist](#quick-reference-checklist)
- [Next Steps](#next-steps)
- [Version History](#version-history)

---

## Introduction

### Why Anti-Patterns Matter

Anti-patterns are recurring practices that seem reasonable at first glance but create significant problems over time. In database engineering, the consequences of anti-patterns range from slow queries and data corruption to full production outages — often surfacing under load at the worst possible moment.

The patterns documented here represent real failures observed across production systems. Each one is:

- **Seductive** — it felt like the right approach when implemented
- **Harmful** — it creates data integrity issues, performance degradation, or operational risk
- **Fixable** — there is a well-understood better approach

### How to Use This Document

1. **Pre-production review:** Use the [Quick Reference Checklist](#quick-reference-checklist) before releasing a new service
2. **Code review:** Reference specific sections when reviewing database-related PRs
3. **Incident retros:** After an incident, check which anti-patterns contributed to the outage
4. **Team onboarding:** Assign this document to new engineers before they touch production schemas
5. **Periodic audit:** Run through the checklist quarterly for existing services

---

## Anti-Patterns Summary Table

| # | Anti-Pattern | Category | Severity |
|---|--------------|----------|----------|
| 1 | God Table | Schema Design | 🔴 Critical |
| 2 | Missing Indexes | Query Performance | 🔴 Critical |
| 3 | N+1 Query Problem | Query Performance | 🟠 High |
| 4 | Soft Deletes Everywhere | Schema Design | 🟡 Medium |
| 5 | Storing Money as Floating Point | Data Integrity | 🔴 Critical |
| 6 | Entity-Attribute-Value (EAV) Abuse | Schema Design | 🟠 High |
| 7 | No Connection Pooling | Operations | 🟠 High |
| 8 | Ignoring Replication Lag | Replication | 🔴 Critical |
| 9 | Unbounded Queries | Query Performance | 🟠 High |
| 10 | Premature Sharding | Architecture | 🟡 Medium |
| 11 | Not Using Transactions | Data Integrity | 🔴 Critical |
| 12 | Schema Changes Without Migration Tools | Operations | 🟠 High |
| 13 | MySQL and SQL Server Engine-Specific Anti-Patterns | Engine-Specific | 🟠 High |

---

## 1. God Table

### Problem

A "God Table" is a single table with dozens (or hundreds) of columns that tries to represent multiple distinct entities. It typically starts as a convenient dumping ground and grows over time until no one understands what a row actually represents.

```sql
-- ❌ Anti-pattern: one table to rule them all
CREATE TABLE everything (
    id              SERIAL PRIMARY KEY,
    name            VARCHAR(255),
    email           VARCHAR(255),
    phone           VARCHAR(50),
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state           VARCHAR(50),
    zip             VARCHAR(20),
    country         VARCHAR(50),
    order_id        INTEGER,
    order_date      TIMESTAMP,
    order_status    VARCHAR(20),
    product_name    VARCHAR(255),
    product_sku     VARCHAR(50),
    product_price   NUMERIC(10,2),
    quantity        INTEGER,
    shipping_method VARCHAR(50),
    tracking_number VARCHAR(100),
    payment_method  VARCHAR(50),
    card_last_four  VARCHAR(4),
    notes           TEXT,
    created_at      TIMESTAMP,
    updated_at      TIMESTAMP,
    is_active       BOOLEAN,
    type            VARCHAR(50),   -- "customer", "vendor", "partner" ...
    extra_field_1   TEXT,
    extra_field_2   TEXT,
    extra_field_3   TEXT
    -- ... 40 more columns
);
```

**Symptoms of a God Table:**

- Most columns are NULL for any given row
- A `type` column determines which columns are relevant
- Multiple teams modify the same table for unrelated features
- ALTER TABLE takes hours because the table has millions of rows and dozens of indexes

### Why It Happens

- Quick prototyping — "just add a column" is faster than designing a new table
- Fear of JOINs — developers believe JOINs are always slow
- ORM auto-generation without schema review

### Impact

- 🔴 **Critical** — Wide rows waste I/O. Every query reads columns it does not need.
- NULL-heavy rows complicate queries with unexpected three-valued logic.
- Schema changes become high-risk, long-running operations.
- As described in *DDIA* Chapter 2, document models and relational models each have trade-offs, but cramming everything into one table gives you the worst of both worlds.

### Detection

```sql
-- Find tables with excessive column counts
SELECT table_name, COUNT(*) AS column_count
FROM information_schema.columns
WHERE table_schema = 'public'
GROUP BY table_name
HAVING COUNT(*) > 20
ORDER BY column_count DESC;
```

### Recommended Fix

Normalize into focused, single-responsibility tables:

```sql
-- ✅ Correct: normalized schema
CREATE TABLE customers (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    email       VARCHAR(255) NOT NULL UNIQUE,
    phone       VARCHAR(50),
    created_at  TIMESTAMP DEFAULT now()
);

CREATE TABLE addresses (
    id          SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    line1       VARCHAR(255) NOT NULL,
    line2       VARCHAR(255),
    city        VARCHAR(100) NOT NULL,
    state       VARCHAR(50),
    zip         VARCHAR(20) NOT NULL,
    country     VARCHAR(50) NOT NULL
);

CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    status      VARCHAR(20) NOT NULL DEFAULT 'pending',
    ordered_at  TIMESTAMP DEFAULT now()
);

CREATE TABLE order_items (
    id          SERIAL PRIMARY KEY,
    order_id    INTEGER NOT NULL REFERENCES orders(id),
    product_id  INTEGER NOT NULL REFERENCES products(id),
    quantity    INTEGER NOT NULL CHECK (quantity > 0),
    unit_price  NUMERIC(10,2) NOT NULL
);
```

```
  God Table                      Normalized
  ┌──────────────────┐           ┌───────────┐
  │   everything     │           │ customers │
  │   (60+ columns)  │    ──►    ├───────────┤
  │   mostly NULLs   │           │ addresses │
  │   mixed concerns │           ├───────────┤
  └──────────────────┘           │ orders    │
                                 ├───────────┤
                                 │order_items│
                                 └───────────┘
```

### Quick Check

- [ ] No table has more than 20–25 columns
- [ ] No `type` or `kind` column that determines which other columns apply
- [ ] NULL ratio per column is below 50% for non-optional fields
- [ ] Each table represents one entity or one relationship

---

## 2. Missing Indexes

### Problem

Queries run without supporting indexes, forcing the database to perform full sequential scans on large tables. This is the single most common cause of slow queries in production.

```sql
-- ❌ Anti-pattern: querying without an index
SELECT * FROM orders WHERE customer_id = 42 AND status = 'shipped';
```

```
-- EXPLAIN output WITHOUT index:
Seq Scan on orders  (cost=0.00..45893.00 rows=52 width=128)
  Filter: ((customer_id = 42) AND (status = 'shipped'))
  Rows Removed by Filter: 999948
-- Full table scan: reads all 1,000,000 rows to find 52 matches
```

### Why It Happens

- Developers test on small datasets where everything is fast
- ORMs generate queries; nobody checks the execution plan
- "We'll add indexes later" — later never comes
- Wrong index type — a B-tree index on a column only used with `LIKE '%term%'`

### Impact

- 🔴 **Critical** — Query latency grows linearly with table size. A query that takes 5ms on 10K rows takes 5s on 10M rows without an index.
- High CPU and I/O saturation on the database server.
- Connection pool exhaustion as queries queue up.
- *DDIA* Chapter 3 explains how B-tree and LSM-tree indexes are fundamental to any storage engine's read performance.

### Detection

```sql
-- PostgreSQL: find slow sequential scans
SELECT schemaname, relname, seq_scan, seq_tup_read,
       idx_scan, idx_tup_fetch
FROM pg_stat_user_tables
WHERE seq_scan > 1000
  AND seq_tup_read > 100000
  AND idx_scan < seq_scan
ORDER BY seq_tup_read DESC
LIMIT 20;

-- PostgreSQL: find missing indexes for foreign keys
SELECT c.conrelid::regclass AS table_name,
       a.attname AS fk_column
FROM pg_constraint c
JOIN pg_attribute a ON a.attnum = ANY(c.conkey) AND a.attrelid = c.conrelid
WHERE c.contype = 'f'
  AND NOT EXISTS (
      SELECT 1 FROM pg_index i
      WHERE i.indrelid = c.conrelid
        AND a.attnum = ANY(i.indkey)
  );
```

### Recommended Fix

```sql
-- ✅ Add a composite index matching the query pattern
CREATE INDEX idx_orders_customer_status ON orders (customer_id, status);
```

```
-- EXPLAIN output WITH index:
Index Scan using idx_orders_customer_status on orders
  (cost=0.42..8.44 rows=52 width=128)
  Index Cond: ((customer_id = 42) AND (status = 'shipped'))
-- Reads exactly 52 rows from the index — 1000x faster
```

**Before and after comparison:**

| Metric | Without Index | With Index |
|--------|---------------|------------|
| Rows scanned | 1,000,000 | 52 |
| Query time | ~850 ms | ~0.8 ms |
| I/O reads | ~45,000 pages | ~4 pages |
| CPU usage | High | Negligible |

### Quick Check

- [ ] Every foreign key column has an index
- [ ] Queries in critical paths have been verified with `EXPLAIN ANALYZE`
- [ ] No sequential scan on tables with > 10,000 rows in hot paths
- [ ] Composite indexes match the column order in WHERE and ORDER BY clauses
- [ ] Unused indexes are periodically identified and dropped

---

## 3. N+1 Query Problem

### Problem

The N+1 query problem occurs when code loads a list of parent rows (1 query), then loops through each row to load related data one at a time (N queries). This turns a single JOIN into N+1 round trips to the database.

```python
# ❌ Anti-pattern: N+1 queries
def get_orders_with_items():
    orders = db.execute("SELECT * FROM orders LIMIT 100")  # 1 query

    for order in orders:
        # This fires once PER ORDER — 100 additional queries!
        items = db.execute(
            "SELECT * FROM order_items WHERE order_id = %s", [order.id]
        )
        order.items = items

    return orders
# Total: 1 + 100 = 101 queries for 100 orders
```

```
Timeline of N+1 queries:

App            Database
 │── SELECT * FROM orders ──────────────►│
 │◄──── 100 rows ───────────────────────│
 │── SELECT * FROM order_items WHERE order_id=1 ──►│
 │◄──── 3 rows ─────────────────────────│
 │── SELECT * FROM order_items WHERE order_id=2 ──►│
 │◄──── 5 rows ─────────────────────────│
 │── ... repeated 98 more times ...     │
 │                                       │
 Total round trips: 101
 Total latency:     101 × ~1ms = ~101ms network overhead alone
```

### Why It Happens

- ORM lazy loading is the default (SQLAlchemy, ActiveRecord, Hibernate)
- Works fine in development with few rows — breaks under production data
- Code structure hides the loop — the N queries happen inside a property getter

### Impact

- 🟠 **High** — Latency scales linearly with N. 100 orders = 101 queries. 10,000 orders = 10,001 queries.
- Network round-trip overhead dominates total query time.
- Database connection is held for the entire loop duration.
- Under load, connection pool is exhausted.

### Detection

```sql
-- PostgreSQL: enable query logging and look for repeated patterns
-- In postgresql.conf:
-- log_min_duration_statement = 0  (log all queries, temporarily)

-- Then search the log for repeated query patterns:
-- grep -c "SELECT.*FROM order_items WHERE order_id" postgresql.log
-- Result: 10,000+  ← N+1 detected
```

### Recommended Fix

**Option A: JOIN in a single query**

```python
# ✅ Fix: single query with JOIN
def get_orders_with_items():
    rows = db.execute("""
        SELECT o.id, o.status, o.ordered_at,
               oi.product_id, oi.quantity, oi.unit_price
        FROM orders o
        JOIN order_items oi ON oi.order_id = o.id
        ORDER BY o.id
        LIMIT 100
    """)
    return group_by_order(rows)
# Total: 1 query
```

**Option B: batch IN query**

```python
# ✅ Fix: two queries with batch IN
def get_orders_with_items():
    orders = db.execute("SELECT * FROM orders LIMIT 100")           # 1 query
    order_ids = [o.id for o in orders]

    items = db.execute(
        "SELECT * FROM order_items WHERE order_id = ANY(%s)",       # 1 query
        [order_ids]
    )
    items_by_order = group_by(items, key=lambda i: i.order_id)

    for order in orders:
        order.items = items_by_order.get(order.id, [])

    return orders
# Total: 2 queries regardless of N
```

**Option C: ORM eager loading**

```python
# ✅ Fix: SQLAlchemy eager loading
orders = (
    session.query(Order)
    .options(joinedload(Order.items))   # emits JOIN automatically
    .limit(100)
    .all()
)
```

### Quick Check

- [ ] ORM relationships use eager loading (`joinedload`, `includes`, `with`) in list endpoints
- [ ] Application performance monitoring (APM) flags queries with > 10 repetitions per request
- [ ] Code reviews check for loops containing database calls
- [ ] Database query count per HTTP request is logged and alerted on (threshold: < 10)

---

## 4. Soft Deletes Everywhere

### Problem

Soft deletes use a flag column (e.g., `is_deleted`, `deleted_at`) instead of actually removing rows. Applied indiscriminately to every table, this adds complexity to every query, bloats table size, complicates unique constraints, and creates subtle bugs when developers forget the filter.

```sql
-- ❌ Anti-pattern: soft delete on every table
ALTER TABLE customers ADD COLUMN is_deleted BOOLEAN DEFAULT FALSE;
ALTER TABLE orders ADD COLUMN is_deleted BOOLEAN DEFAULT FALSE;
ALTER TABLE order_items ADD COLUMN is_deleted BOOLEAN DEFAULT FALSE;
ALTER TABLE audit_log ADD COLUMN is_deleted BOOLEAN DEFAULT FALSE;  -- really?
ALTER TABLE sessions ADD COLUMN is_deleted BOOLEAN DEFAULT FALSE;

-- Every query must now remember the filter:
SELECT * FROM customers WHERE is_deleted = FALSE AND email = 'user@example.com';
SELECT * FROM orders WHERE is_deleted = FALSE AND customer_id = 42;

-- Unique constraints break:
-- User "deletes" account, tries to re-register with same email → UNIQUE violation
-- because the soft-deleted row still exists
```

### Why It Happens

- "We might need to restore deleted data" — often without actual business requirements
- Legal/compliance requirements misunderstood — sometimes only specific tables need retention
- Easier than implementing proper archival

### Impact

- 🟡 **Medium** — Tables grow indefinitely since rows are never removed.
- Every query must include `WHERE is_deleted = FALSE`, increasing bug surface area.
- Unique constraints require workarounds (partial indexes, composite keys).
- JOINs on soft-deleted tables produce ghost data if any query forgets the filter.
- Index size bloated by deleted rows that are never cleaned up.

### Detection

```sql
-- Find tables with soft delete columns
SELECT table_name, column_name
FROM information_schema.columns
WHERE column_name IN ('is_deleted', 'deleted_at', 'is_active', 'soft_deleted')
  AND table_schema = 'public'
ORDER BY table_name;

-- Check the ratio of "deleted" rows
SELECT
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE is_deleted = TRUE) AS deleted,
    ROUND(100.0 * COUNT(*) FILTER (WHERE is_deleted = TRUE) / COUNT(*), 1) AS pct_deleted
FROM orders;
```

### Recommended Fix

**Use soft deletes only where business requirements demand it.** For everything else, use hard deletes with an archive table or event log.

```sql
-- ✅ Correct: archive pattern for audit needs
CREATE TABLE orders_archive (
    LIKE orders INCLUDING ALL
);

-- Move deleted rows to archive
WITH deleted AS (
    DELETE FROM orders WHERE id = 42 RETURNING *
)
INSERT INTO orders_archive SELECT * FROM deleted;

-- ✅ If soft deletes ARE required, use partial indexes
CREATE UNIQUE INDEX idx_customers_email_active
    ON customers (email)
    WHERE deleted_at IS NULL;

-- ✅ Use a view to hide the filter from application code
CREATE VIEW active_customers AS
    SELECT * FROM customers WHERE deleted_at IS NULL;
```

**Decision matrix: when to use what:**

| Strategy | Use When | Example Table |
|----------|----------|---------------|
| Hard delete | No business need to restore | `sessions`, `temp_tokens` |
| Hard delete + archive | Audit trail required | `orders`, `payments` |
| Soft delete | Undo/restore feature required | `documents`, `user_accounts` |
| Event sourcing | Full history is the product | `financial_transactions` |

### Quick Check

- [ ] Soft deletes are only on tables with an explicit business reason for restoration
- [ ] Ephemeral tables (`sessions`, `cache`, `temp_*`) use hard deletes
- [ ] Queries use views or ORM default scopes to avoid forgetting the filter
- [ ] Partial unique indexes handle unique constraints on soft-deleted tables
- [ ] Archived rows are periodically purged or moved to cold storage

---

## 5. Storing Money as Floating Point

### Problem

Using `FLOAT` or `DOUBLE` to store monetary values introduces rounding errors inherent to IEEE 754 floating-point arithmetic. These small errors accumulate over thousands of transactions, causing financial reports to be incorrect and audits to fail.

```sql
-- ❌ Anti-pattern: floating-point money
CREATE TABLE transactions (
    id      SERIAL PRIMARY KEY,
    amount  FLOAT NOT NULL   -- WRONG: IEEE 754 binary floating point
);

INSERT INTO transactions (amount) VALUES (0.1), (0.2);
SELECT SUM(amount) FROM transactions;
-- Expected: 0.3
-- Actual:   0.30000000000000004
```

```python
# The classic demonstration:
>>> 0.1 + 0.2
0.30000000000000004

>>> 0.1 + 0.2 == 0.3
False

# Accumulated over 10,000 transactions:
>>> sum([0.01] * 10000)
99.99999999999857   # Lost $0.0000000000014 — per batch
```

### Why It Happens

- Programming language defaults (`float` in Python, `double` in Java) feel natural
- "The difference is tiny" — it is, until it is multiplied by millions of transactions
- ORM schema generators sometimes default to float for decimal fields

### Impact

- 🔴 **Critical** — Financial reports become wrong. Rounding errors accumulate.
- Audit failures with regulators. Cannot reconcile books to the cent.
- Customer trust destruction when invoices show incorrect amounts.
- *DDIA* does not call this out specifically, but Chapter 5's discussion of data integrity applies: if you cannot trust the data you have stored, the system's guarantees are broken.

### Detection

```sql
-- Find float/double columns that might store money
SELECT table_name, column_name, data_type
FROM information_schema.columns
WHERE data_type IN ('real', 'double precision', 'float')
  AND table_schema = 'public'
  AND (column_name LIKE '%price%'
    OR column_name LIKE '%amount%'
    OR column_name LIKE '%cost%'
    OR column_name LIKE '%balance%'
    OR column_name LIKE '%total%');
```

### Recommended Fix

```sql
-- ✅ Correct: use NUMERIC / DECIMAL for money
CREATE TABLE transactions (
    id       SERIAL PRIMARY KEY,
    amount   NUMERIC(19,4) NOT NULL   -- exact decimal arithmetic
);

-- Or store as integer cents (avoids decimal entirely)
CREATE TABLE transactions_v2 (
    id          SERIAL PRIMARY KEY,
    amount_cents BIGINT NOT NULL       -- $19.99 stored as 1999
);
```

**Comparison of approaches:**

| Approach | Storage | Precision | Arithmetic | Best For |
|----------|---------|-----------|------------|----------|
| `FLOAT` / `DOUBLE` | 4–8 bytes | ❌ Approximate | Fast | Scientific data, never money |
| `NUMERIC(19,4)` | Variable | ✅ Exact | Slower | Financial calculations |
| `BIGINT` (cents) | 8 bytes | ✅ Exact | Fastest | High-throughput payment systems |
| `money` (Postgres) | 8 bytes | ✅ Exact | Fast | ⚠️ Locale-dependent, avoid |

```python
# ✅ Application-level: use Decimal, not float
from decimal import Decimal, ROUND_HALF_UP

price = Decimal("19.99")
tax = price * Decimal("0.08")
total = (price + tax).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
# total = Decimal('21.59') — exact
```

### Quick Check

- [ ] No `FLOAT` or `DOUBLE` columns store monetary values
- [ ] All money columns use `NUMERIC(19,4)` or `BIGINT` (cents/minor units)
- [ ] Application code uses `Decimal` / `BigDecimal`, never `float` / `double`
- [ ] Rounding mode is explicitly set (`ROUND_HALF_UP` or banker's rounding)
- [ ] Financial totals are reconciled with individual line items in tests

---

## 6. Entity-Attribute-Value (EAV) Abuse

### Problem

The Entity-Attribute-Value pattern stores data as rows of `(entity_id, attribute_name, attribute_value)` instead of typed columns. While EAV has legitimate uses, applying it as the default schema design turns the database into a glorified key-value store, losing type safety, query performance, and referential integrity.

```sql
-- ❌ Anti-pattern: EAV for everything
CREATE TABLE entity_attributes (
    entity_id   INTEGER NOT NULL,
    attr_name   VARCHAR(255) NOT NULL,
    attr_value  TEXT,  -- everything is a string
    PRIMARY KEY (entity_id, attr_name)
);

-- Storing a product:
INSERT INTO entity_attributes VALUES (1, 'name', 'Widget');
INSERT INTO entity_attributes VALUES (1, 'price', '19.99');
INSERT INTO entity_attributes VALUES (1, 'weight_kg', '0.5');
INSERT INTO entity_attributes VALUES (1, 'color', 'blue');

-- Querying becomes a nightmare:
-- "Find products under $20 that weigh less than 1kg"
SELECT e1.entity_id
FROM entity_attributes e1
JOIN entity_attributes e2 ON e1.entity_id = e2.entity_id
JOIN entity_attributes e3 ON e1.entity_id = e3.entity_id
WHERE e1.attr_name = 'name'
  AND e2.attr_name = 'price' AND CAST(e2.attr_value AS NUMERIC) < 20.0
  AND e3.attr_name = 'weight_kg' AND CAST(e3.attr_value AS NUMERIC) < 1.0;
-- Three self-joins for one simple WHERE clause!
```

### Why It Happens

- "We need a flexible schema" — true for some use cases, but EAV is not the only option
- Developers coming from key-value or document databases replicate the pattern in SQL
- Premature abstraction: "we don't know what attributes entities will have yet"

### Impact

- 🟠 **High** — No type safety. `price` is stored as `TEXT`; nothing prevents `'banana'`.
- Queries require N self-joins where a proper schema needs zero.
- Cannot use `CHECK`, `NOT NULL`, `FOREIGN KEY`, or `UNIQUE` constraints on attribute values.
- Cannot `ORDER BY` attribute values without casting.
- *DDIA* Chapter 2 discusses the relational model vs. schema-on-read approaches. EAV in SQL is schema-on-read without the benefits of a document store.

### Detection

```sql
-- Find EAV-pattern tables
SELECT table_name
FROM information_schema.columns
WHERE column_name IN ('attr_name', 'attribute_name', 'key', 'property_name')
  AND table_schema = 'public';

-- Check for attribute explosion
SELECT attr_name, COUNT(*) AS usage_count
FROM entity_attributes
GROUP BY attr_name
ORDER BY usage_count DESC;
```

### Recommended Fix

**For well-known attributes:** use proper columns.

```sql
-- ✅ Correct: typed columns for known attributes
CREATE TABLE products (
    id         SERIAL PRIMARY KEY,
    name       VARCHAR(255) NOT NULL,
    price      NUMERIC(10,2) NOT NULL CHECK (price >= 0),
    weight_kg  NUMERIC(6,3),
    color      VARCHAR(50)
);

-- Simple, type-safe, indexable:
SELECT * FROM products WHERE price < 20.0 AND weight_kg < 1.0;
```

**For truly dynamic user-defined attributes:** use JSONB (PostgreSQL) or JSON column.

```sql
-- ✅ When you genuinely need flexible attributes
CREATE TABLE products (
    id         SERIAL PRIMARY KEY,
    name       VARCHAR(255) NOT NULL,
    price      NUMERIC(10,2) NOT NULL,
    attributes JSONB DEFAULT '{}'    -- dynamic, schema-on-read
);

-- Query with indexing support:
CREATE INDEX idx_products_attrs ON products USING GIN (attributes);

SELECT * FROM products
WHERE attributes @> '{"color": "blue"}'
  AND (attributes->>'weight_kg')::numeric < 1.0;
```

**When EAV IS appropriate:**

| Use Case | EAV Appropriate? | Better Alternative |
|----------|------------------|--------------------|
| User-defined form fields | ✅ Yes | JSONB is often simpler |
| Plugin/extension systems | ✅ Yes | JSONB or separate tables |
| Product catalog with fixed attributes | ❌ No | Typed columns |
| Core business entities | ❌ No | Typed columns |
| Configuration key-value pairs | ✅ Yes | But keep it in one table |

### Quick Check

- [ ] EAV is only used for genuinely dynamic, user-defined attributes
- [ ] Core business entities use typed columns, not EAV
- [ ] JSONB columns are considered as an alternative to EAV where schema flexibility is needed
- [ ] EAV tables have a defined set of allowed attribute names (validation at app layer)
- [ ] Queries against EAV attributes are rare and not in hot paths

---

## 7. No Connection Pooling

### Problem

Opening a new database connection for every HTTP request incurs TCP handshake, TLS negotiation, and authentication overhead — typically 20–50ms per connection. Under load, this exhausts the database's `max_connections` limit and causes connection refused errors.

```python
# ❌ Anti-pattern: new connection per request
def handle_request(request):
    conn = psycopg2.connect(
        host="db.prod", dbname="myapp", user="app", password="secret"
    )
    try:
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM users WHERE id = %s", [request.user_id])
        return cursor.fetchone()
    finally:
        conn.close()  # connection discarded — all setup cost wasted
```

```
Connection lifecycle WITHOUT pooling:

Request 1:  [TCP handshake][TLS][Auth][Query][Close]  ~55ms overhead
Request 2:  [TCP handshake][TLS][Auth][Query][Close]  ~55ms overhead
Request 3:  [TCP handshake][TLS][Auth][Query][Close]  ~55ms overhead
...
Request N:  [TCP handshake][TLS][Auth][Query][Close]  ~55ms overhead

Connection lifecycle WITH pooling:

Startup:    [TCP handshake][TLS][Auth]  ← once
Request 1:  [Borrow][Query][Return]    ~0.1ms overhead
Request 2:  [Borrow][Query][Return]    ~0.1ms overhead
Request 3:  [Borrow][Query][Return]    ~0.1ms overhead
...
Request N:  [Borrow][Query][Return]    ~0.1ms overhead
```

### Why It Happens

- Default examples in documentation use direct connections
- Works fine in development where there are few concurrent requests
- Cloud-hosted databases have high connection limits — until you scale horizontally

### Impact

- 🟠 **High** — Each connection consumes ~5–10MB of RAM on the database server.
- PostgreSQL's default `max_connections = 100` is exhausted quickly.
- Connection storms after a deploy restart all app instances simultaneously.
- *DDIA* Chapter 7 discusses transaction isolation levels that require connections to maintain state. Pooling ensures connections are reused efficiently.

### Detection

```sql
-- PostgreSQL: check active connections vs limit
SELECT
    count(*) AS active_connections,
    (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max_connections
FROM pg_stat_activity;

-- Check for connection churn (high connection rate)
SELECT
    state,
    count(*) AS count,
    max(now() - backend_start) AS oldest,
    min(now() - backend_start) AS newest
FROM pg_stat_activity
GROUP BY state;
```

### Recommended Fix

**Application-level pooling:**

```python
# ✅ Correct: connection pool
from psycopg2 import pool

# Create pool once at startup
connection_pool = pool.ThreadedConnectionPool(
    minconn=5,
    maxconn=20,
    host="db.prod", dbname="myapp", user="app", password="secret"
)

def handle_request(request):
    conn = connection_pool.getconn()
    try:
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM users WHERE id = %s", [request.user_id])
        return cursor.fetchone()
    finally:
        connection_pool.putconn(conn)  # returned to pool, not closed
```

**External pooling with PgBouncer (recommended for production):**

```ini
; pgbouncer.ini
[databases]
myapp = host=db.prod port=5432 dbname=myapp

[pgbouncer]
listen_port = 6432
pool_mode = transaction          ; release connection after each transaction
max_client_conn = 1000           ; accept up to 1000 app connections
default_pool_size = 25           ; only 25 actual DB connections per pool
reserve_pool_size = 5
reserve_pool_timeout = 3
```

```
                    Without PgBouncer        With PgBouncer
                    ┌─────────┐              ┌─────────┐
  100 app servers ──┤ 100 conn├──► Database   │ 100 app │──► PgBouncer ──► Database
  (100 connections) │ per box │    (10,000    │ servers │    (25 actual    (25 conn)
                    └─────────┘    connections)│(1000    │    connections)
                                              │ conn)   │
                                              └─────────┘
```

### Quick Check

- [ ] A connection pool is configured (application-level or PgBouncer/Pgpool/ProxySQL)
- [ ] Pool size is tuned: `pool_size ≈ (2 × CPU cores) + disk spindles` per database
- [ ] Connection lifetime and idle timeout are configured to avoid stale connections
- [ ] Monitoring alert when active connections exceed 80% of `max_connections`
- [ ] After-deploy connection surge is handled (graceful startup, staggered restarts)

---

## 8. Ignoring Replication Lag

### Problem

In a primary-replica architecture, reading from a replica immediately after writing to the primary can return stale data because replication is asynchronous. This creates confusing bugs where a user creates a record but cannot see it on the next page load.

```python
# ❌ Anti-pattern: write to primary, read from replica immediately
def create_and_return_order(user_id, items):
    # Write goes to primary
    primary_db.execute(
        "INSERT INTO orders (user_id, status) VALUES (%s, 'pending') RETURNING id",
        [user_id]
    )
    order_id = cursor.fetchone()[0]

    # Read goes to replica — but replication hasn't caught up yet!
    order = replica_db.execute(
        "SELECT * FROM orders WHERE id = %s", [order_id]
    )
    return order  # Returns None! The row hasn't replicated yet.
```

```
Write-then-read with replication lag:

 Client        Primary         Replica
   │─ INSERT ──►│                │
   │◄─ OK ──────│                │
   │             │── replicate ──►│  ← takes 10-500ms
   │─ SELECT ───────────────────►│
   │◄─ NULL ────────────────────│  ← row not there yet!
   │             │── replicate ──►│  ← arrives after the read
```

### Why It Happens

- Read replicas are added for scaling, and all reads are blindly routed there
- Replication lag is usually <100ms so it "works" in testing
- Load balancers distribute reads without awareness of write recency

### Impact

- 🔴 **Critical** — Users see stale data or missing records immediately after mutations.
- Inconsistency that is intermittent and load-dependent — worst kind of bug to debug.
- Can cause data loss if application logic branches on the stale read.
- *DDIA* Chapter 5 covers replication lag extensively. The "reading your own writes" consistency guarantee is specifically designed to prevent this.

### Detection

```sql
-- PostgreSQL: check replication lag
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes,
    reply_time
FROM pg_stat_replication;

-- MySQL: check seconds behind primary
SHOW REPLICA STATUS\G
-- Look for: Seconds_Behind_Source
```

### Recommended Fix

**Pattern 1: Read-your-own-writes — route reads to primary after writes.**

```python
# ✅ Fix: read from primary after a write
class DBRouter:
    def __init__(self, primary, replica):
        self.primary = primary
        self.replica = replica
        self._wrote = threading.local()

    def write(self, query, params):
        self._wrote.last_write = time.monotonic()
        return self.primary.execute(query, params)

    def read(self, query, params):
        # If we wrote within the last 2 seconds, read from primary
        last = getattr(self._wrote, 'last_write', 0)
        if time.monotonic() - last < 2.0:
            return self.primary.execute(query, params)
        return self.replica.execute(query, params)
```

**Pattern 2: Causal consistency with LSN tracking (PostgreSQL).**

```python
# ✅ Fix: wait for replica to catch up to a specific LSN
def write_and_read(primary, replica, insert_query, select_query):
    primary.execute(insert_query)

    # Get the current WAL position after write
    primary.execute("SELECT pg_current_wal_lsn()")
    write_lsn = primary.fetchone()[0]

    # Wait until replica has replayed past that LSN
    replica.execute(
        "SELECT pg_last_wal_replay_lsn() >= %s::pg_lsn", [write_lsn]
    )
    # Poll or use pg_wal_replay_wait() in PG17+

    return replica.execute(select_query)
```

**Pattern 3: Session-level stickiness.**

```
  ┌──────────────────────────────────────────┐
  │              Load Balancer               │
  │  if cookie(recent_write):               │
  │      route → Primary                     │
  │  else:                                   │
  │      route → Replica                     │
  └──────────────────────────────────────────┘
```

### Quick Check

- [ ] Write-then-read paths are identified and route reads to primary (or wait for replication)
- [ ] Replication lag is monitored and alerted on (threshold: > 1 second)
- [ ] Application code does not assume synchronous replication unless configured
- [ ] Critical user flows (create → view) are tested for read-after-write consistency
- [ ] Database proxy or application router handles primary/replica routing

---

## 9. Unbounded Queries

### Problem

Queries without `LIMIT`, or with `OFFSET`-based pagination on large tables, fetch unbounded result sets that consume excessive memory in both the database and the application. A table that had 100 rows in development now has 10 million rows in production.

```sql
-- ❌ Anti-pattern: no LIMIT
SELECT * FROM events;
-- Returns 10,000,000 rows. Application attempts to load all into memory.

-- ❌ Anti-pattern: OFFSET pagination at depth
SELECT * FROM events ORDER BY created_at OFFSET 9999900 LIMIT 100;
-- Database must scan and discard 9,999,900 rows to return 100.
```

```python
# ❌ Anti-pattern: loading all rows into a list
def get_all_users():
    cursor.execute("SELECT * FROM users")
    return cursor.fetchall()  # 10M rows loaded into Python memory → OOM
```

### Why It Happens

- "There will only be a few hundred rows" — famous last words
- OFFSET pagination is the default tutorial approach
- Admin/reporting endpoints are forgotten when adding limits

### Impact

- 🟠 **High** — Application OOM-killed when result set exceeds available memory.
- OFFSET pagination degrades linearly: offset 10M requires scanning 10M rows.
- Database temp file spills to disk for large sorts.
- *DDIA* Chapter 3 discusses how B-tree range scans are efficient only when bounded. Without a limit, the engine reads the entire index leaf chain.

### Detection

```sql
-- PostgreSQL: find queries without LIMIT in pg_stat_statements
SELECT query, calls, mean_exec_time, rows
FROM pg_stat_statements
WHERE query NOT LIKE '%LIMIT%'
  AND rows > 10000
ORDER BY rows DESC
LIMIT 20;
```

### Recommended Fix

**Always use LIMIT and prefer cursor/keyset pagination over OFFSET:**

```sql
-- ✅ Correct: keyset (cursor) pagination
-- Page 1:
SELECT id, created_at, event_type
FROM events
WHERE created_at >= '2024-01-01'
ORDER BY created_at, id
LIMIT 100;

-- Page 2 (use the last row's values as cursor):
SELECT id, created_at, event_type
FROM events
WHERE (created_at, id) > ('2024-01-15 10:30:00', 12345)
ORDER BY created_at, id
LIMIT 100;
-- Constant-time regardless of page depth!
```

**Comparison of pagination strategies:**

| Strategy | Page 1 | Page 1000 | Page 100,000 |
|----------|--------|-----------|--------------|
| `OFFSET` | ~5ms | ~500ms | ~50,000ms |
| Keyset | ~5ms | ~5ms | ~5ms |

```
OFFSET pagination cost:

Page 1:    Scan ▓ Return ▓
Page 10:   Scan ▓▓▓▓▓▓▓▓▓▓ Return ▓
Page 1000: Scan ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓...▓▓▓ Return ▓
                ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                All scanned and discarded

Keyset pagination cost:

Page 1:    Seek ▓ Return ▓
Page 10:   Seek ▓ Return ▓
Page 1000: Seek ▓ Return ▓       ← constant cost via index seek
```

### Quick Check

- [ ] Every `SELECT` in application code has a `LIMIT` clause
- [ ] List/search API endpoints enforce a maximum page size (e.g., 100)
- [ ] Deep pagination uses keyset/cursor pagination, not OFFSET
- [ ] Background jobs processing large tables use batched reads with cursors
- [ ] Admin/reporting queries use streaming or server-side cursors

---

## 10. Premature Sharding

### Problem

Sharding (horizontal partitioning across multiple database servers) is introduced before simpler scaling strategies — vertical scaling, read replicas, query optimization, caching — have been exhausted. Sharding adds enormous operational complexity that is rarely justified for databases under 1TB.

```
When teams prematurely shard:

Before sharding:                   After premature sharding:
┌─────────────┐                    ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│  PostgreSQL  │                    │Shard1│ │Shard2│ │Shard3│ │Shard4│
│  Single node │                    └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘
│  500GB, 8CPU │                       │        │        │        │
│  Works fine  │                    ┌──┴────────┴────────┴────────┴──┐
└─────────────┘                    │       Routing / Proxy Layer     │
                                   └────────────────────────────────┘
  Ops complexity: Low                Ops complexity: Extreme
  Cross-shard JOINs: N/A            Cross-shard JOINs: Impossible
  Transactions: ACID                 Transactions: 2PC or give up
  Schema migrations: 1 database      Schema migrations: 4 databases
  Backups: 1 backup                  Backups: 4 backups + coordination
```

### Why It Happens

- Cargo-culting from FAANG blog posts ("Google shards, so should we")
- Confusing read scaling (replicas) with write scaling (sharding)
- Not profiling actual bottlenecks — the issue is often a missing index, not capacity

### Impact

- 🟡 **Medium** — Massively increased operational complexity for marginal benefit.
- Cross-shard queries are impossible or extremely expensive.
- Distributed transactions require two-phase commit (2PC), adding latency and failure modes.
- Resharding (changing the shard key or count) is one of the hardest operations in database engineering.
- *DDIA* Chapter 6 discusses partitioning in detail and emphasizes that single-node solutions should be preferred until they are truly insufficient.

### Detection

Ask these questions before sharding:

```
Sharding readiness checklist:
                                                        Yes?
1. Database size > 1TB                                  [ ]
2. Write throughput > what a single node can handle     [ ]
3. Vertical scaling exhausted (max instance size used)  [ ]
4. Read replicas exhausted                              [ ]
5. Query optimization completed (no missing indexes)    [ ]
6. Caching layer (Redis/Memcached) in place             [ ]
7. Connection pooling optimized                         [ ]
8. Table partitioning (within single node) considered   [ ]

If fewer than 5 boxes are checked → DO NOT SHARD
```

### Recommended Fix

**Exhaust these strategies first, in order:**

```
Scaling ladder (try each step before the next):

Step 1: Query optimization     ← FREE
        Add missing indexes, fix N+1 queries, EXPLAIN ANALYZE

Step 2: Connection pooling     ← LOW EFFORT
        PgBouncer / ProxySQL

Step 3: Caching                ← MODERATE EFFORT
        Redis/Memcached for hot read paths

Step 4: Read replicas          ← MODERATE EFFORT
        Route reads to replicas, keep writes on primary

Step 5: Vertical scaling       ← COSTS MONEY
        Bigger instance (more CPU, RAM, faster disks)

Step 6: Table partitioning     ← MODERATE EFFORT
        PostgreSQL declarative partitioning by date/range

Step 7: Sharding               ← LAST RESORT
        Only when steps 1-6 are exhausted
```

**If you truly need sharding**, use application-level sharding or a sharding-aware proxy:

```python
# ✅ If sharding is genuinely needed: consistent shard routing
import hashlib

SHARD_COUNT = 4
shard_connections = {
    0: connect("shard-0.db.prod"),
    1: connect("shard-1.db.prod"),
    2: connect("shard-2.db.prod"),
    3: connect("shard-3.db.prod"),
}

def get_shard(tenant_id: str) -> int:
    return int(hashlib.sha256(tenant_id.encode()).hexdigest(), 16) % SHARD_COUNT

def query_for_tenant(tenant_id, sql, params):
    shard = get_shard(tenant_id)
    return shard_connections[shard].execute(sql, params)
```

### Quick Check

- [ ] Vertical scaling and read replicas have been exhausted before considering sharding
- [ ] Query optimization (indexes, N+1 fixes) has been completed
- [ ] Caching layer is in place for hot read paths
- [ ] Database is > 1TB with proven write throughput bottleneck before sharding
- [ ] If sharded, a clear shard key is chosen that avoids hotspots and cross-shard queries

---

## 11. Not Using Transactions

### Problem

Performing multiple related writes without a transaction means a partial failure leaves the database in an inconsistent state. If step 2 of 3 fails, step 1 has already been committed and cannot be rolled back.

```python
# ❌ Anti-pattern: multiple writes without a transaction
def transfer_money(from_id, to_id, amount):
    db.execute(
        "UPDATE accounts SET balance = balance - %s WHERE id = %s",
        [amount, from_id]
    )
    # What if the app crashes HERE? Money is deducted but never credited.
    db.execute(
        "UPDATE accounts SET balance = balance + %s WHERE id = %s",
        [amount, to_id]
    )
    # What if THIS fails? Same problem.
    db.execute(
        "INSERT INTO transfers (from_id, to_id, amount) VALUES (%s, %s, %s)",
        [from_id, to_id, amount]
    )
```

```
Failure scenario without a transaction:

Step 1: Debit  $100 from Account A   ✅ Committed
Step 2: Credit $100 to Account B     ❌ Database connection lost
Step 3: Log the transfer             ⬜ Never reached

Result: $100 has vanished. Account A is debited.
        Account B never received the money.
        No transfer record exists.
```

### Why It Happens

- Auto-commit mode is the default in most database drivers
- Developers assume each statement is atomic (it is — but multiple statements are not)
- "It hasn't failed yet" — partial failures are rare but catastrophic

### Impact

- 🔴 **Critical** — Data inconsistency, lost money, orphaned records.
- Bugs that only manifest under failure conditions — extremely hard to reproduce.
- No way to recover without manual intervention or compensating transactions.
- *DDIA* Chapter 7 is entirely about transactions and why they exist. The atomicity guarantee (the A in ACID) is specifically designed to prevent partial-write scenarios.

### Detection

```python
# Code review: look for multiple db.execute() calls without a transaction
# Grep for patterns like:
#   db.execute("UPDATE ...
#   db.execute("INSERT ...
# without BEGIN/COMMIT or context manager wrapping them
```

```bash
# Search for potential violations
grep -n "db.execute\|cursor.execute" app/ -r | \
  grep -v "transaction\|BEGIN\|COMMIT\|with.*session"
```

### Recommended Fix

```python
# ✅ Correct: wrap in a transaction
def transfer_money(from_id, to_id, amount):
    with db.begin() as txn:  # BEGIN
        txn.execute(
            "UPDATE accounts SET balance = balance - %s WHERE id = %s",
            [amount, from_id]
        )
        txn.execute(
            "UPDATE accounts SET balance = balance + %s WHERE id = %s",
            [amount, to_id]
        )
        txn.execute(
            "INSERT INTO transfers (from_id, to_id, amount) VALUES (%s, %s, %s)",
            [from_id, to_id, amount]
        )
    # COMMIT — all three succeed, or all three roll back
```

```python
# ✅ SQLAlchemy pattern
from sqlalchemy.orm import Session

def transfer_money(from_id, to_id, amount):
    with Session(engine) as session:
        with session.begin():
            from_acct = session.get(Account, from_id)
            to_acct = session.get(Account, to_id)
            from_acct.balance -= amount
            to_acct.balance += amount
            session.add(Transfer(from_id=from_id, to_id=to_id, amount=amount))
        # auto-commit on exit; auto-rollback on exception
```

```
With a transaction:

Step 1: BEGIN
Step 2: Debit  $100 from Account A   (buffered, not yet visible)
Step 3: Credit $100 to Account B     ❌ Connection lost
Step 4: ROLLBACK (automatic)

Result: Account A still has its $100. Account B unchanged.
        No partial state. Fully consistent.
```

### Quick Check

- [ ] All multi-statement write operations are wrapped in explicit transactions
- [ ] ORM session/unit-of-work pattern is used consistently
- [ ] Auto-commit is disabled or explicitly managed
- [ ] Error handling includes rollback on exception
- [ ] Critical financial operations use `SELECT ... FOR UPDATE` to prevent concurrent modification

---

## 12. Schema Changes Without Migration Tools

### Problem

Running ad-hoc `ALTER TABLE` statements directly in production without version-controlled migration scripts leads to schema drift between environments, untested changes, and risky deployments that cannot be rolled back.

```sql
-- ❌ Anti-pattern: ad-hoc schema change in production
-- Developer SSHs into production database and runs:
ALTER TABLE users ADD COLUMN phone VARCHAR(50);
ALTER TABLE users DROP COLUMN legacy_field;
ALTER TABLE orders ALTER COLUMN status TYPE VARCHAR(50);

-- Problems:
-- 1. No record of what changed or when
-- 2. Dev and staging databases are now out of sync
-- 3. If the ALTER fails halfway, no rollback plan
-- 4. Large table ALTER locks the table for hours (pre-PG11 for some operations)
-- 5. No code review of the schema change
```

### Why It Happens

- "It's just one column" — quick fix mentality
- No migration tooling set up in the project
- Urgency during an incident leads to shortcuts
- Lack of staging environment that mirrors production schema

### Impact

- 🟠 **High** — Schema drift between environments causes deployment failures.
- No audit trail of schema changes.
- Rollback is impossible without a reverse migration script.
- Large `ALTER TABLE` can lock tables and cause downtime (*DDIA* Chapter 4 discusses schema evolution and the need for backward/forward compatibility).

### Detection

```sql
-- PostgreSQL: check recent schema changes via DDL event triggers
-- Or compare schemas between environments:
-- pg_dump --schema-only production > prod_schema.sql
-- pg_dump --schema-only staging > staging_schema.sql
-- diff prod_schema.sql staging_schema.sql

-- Check if migration tool tables exist:
SELECT table_name FROM information_schema.tables
WHERE table_name IN (
    'schema_migrations',     -- Rails / golang-migrate
    'alembic_version',       -- Alembic (Python)
    'flyway_schema_history', -- Flyway (Java)
    '__EFMigrationsHistory'  -- Entity Framework (.NET)
);
```

### Recommended Fix

**Use a migration tool and the expand-contract pattern for safe deployments:**

```python
# ✅ Correct: version-controlled migration (Alembic example)
# alembic revision --autogenerate -m "add_phone_to_users"

"""add phone to users

Revision ID: a1b2c3d4e5f6
Revises: 9z8y7x6w5v4u
"""
from alembic import op
import sqlalchemy as sa

def upgrade():
    op.add_column('users', sa.Column('phone', sa.String(50), nullable=True))

def downgrade():
    op.drop_column('users', 'phone')
```

**Expand-contract pattern for zero-downtime changes:**

```
Expand-Contract migration for renaming a column:

Phase 1: EXPAND — add new column, keep old one
┌──────────────────────────────────────────┐
│ ALTER TABLE users ADD COLUMN phone_number VARCHAR(50); │
│ -- Both old (phone) and new (phone_number) exist       │
│ -- App writes to BOTH columns                          │
└──────────────────────────────────────────┘

Phase 2: MIGRATE — backfill data
┌──────────────────────────────────────────┐
│ UPDATE users SET phone_number = phone WHERE phone_number IS NULL; │
│ -- Run in batches to avoid long locks                             │
└──────────────────────────────────────────┘

Phase 3: CONTRACT — remove old column (after all app instances updated)
┌──────────────────────────────────────────┐
│ ALTER TABLE users DROP COLUMN phone;     │
│ -- Only after no code references the old column │
└──────────────────────────────────────────┘
```

**Migration tools by ecosystem:**

| Language | Tool | Command |
|----------|------|---------|
| Python | Alembic | `alembic upgrade head` |
| Ruby | ActiveRecord Migrations | `rails db:migrate` |
| Go | golang-migrate | `migrate -path ./migrations -database $DB up` |
| Java | Flyway | `flyway migrate` |
| .NET | Entity Framework | `dotnet ef database update` |
| Node.js | Knex / Prisma | `knex migrate:latest` / `prisma migrate deploy` |

### Quick Check

- [ ] All schema changes go through version-controlled migration files
- [ ] Migration tool is configured and runs in CI/CD pipeline
- [ ] Every migration has a `downgrade` / `down` function for rollback
- [ ] Large table changes use online DDL tools (`pg_repack`, `pt-online-schema-change`, `gh-ost`)
- [ ] Expand-contract pattern is used for breaking schema changes
- [ ] Schema changes are tested in staging before production

---

## 13. MySQL and SQL Server Engine-Specific Anti-Patterns

Engine choice does not replace fundamentals, but it does change which mistakes are most expensive. The following anti-patterns show up repeatedly in production incidents on MySQL and SQL Server systems.

---

### MySQL Anti-Pattern: Using MyISAM for Transactional Workloads

#### Problem

MyISAM tables have no crash recovery, no foreign keys, and use table-level locking. After an unclean shutdown, they require a manual `REPAIR TABLE` that can take hours on large tables. Any concurrent writes are fully blocked during the repair.

#### Detection

```sql
-- Find all non-InnoDB tables in the application database
SELECT table_schema, table_name, engine
FROM   information_schema.tables
WHERE  table_schema NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
  AND  engine <> 'InnoDB'
ORDER BY table_schema, table_name;
```

#### Recommended Fix

```sql
-- Convert a single table online (may cause a full table rebuild; time on staging first)
ALTER TABLE legacy_table ENGINE = InnoDB, ALGORITHM = INPLACE, LOCK = NONE;

-- Confirm the conversion
SELECT table_name, engine
FROM   information_schema.tables
WHERE  table_schema = 'myapp'
  AND  table_name   = 'legacy_table';
```

---

### MySQL Anti-Pattern: Legacy `utf8` or Mixed Collations

#### Problem

MySQL's `utf8` charset is a 3-byte subset of UTF-8. It cannot store 4-byte characters
(emoji, many CJK extensions, mathematical symbols), silently truncating or erroring instead.
Mixed collations across joined columns also force index-defeating implicit conversions.

#### Detection

```sql
-- Tables not on utf8mb4
SELECT table_schema, table_name, table_collation
FROM   information_schema.tables
WHERE  table_schema NOT IN ('information_schema','mysql','performance_schema','sys')
  AND  table_collation NOT LIKE 'utf8mb4%';

-- Columns with a different character set than the rest of the application
SELECT table_schema, table_name, column_name,
       character_set_name, collation_name
FROM   information_schema.columns
WHERE  character_set_name IS NOT NULL
  AND  character_set_name <> 'utf8mb4'
  AND  table_schema NOT IN ('information_schema','mysql','performance_schema','sys');
```

#### Recommended Fix

```sql
-- Convert one table (test migration time on a staging snapshot first)
ALTER TABLE posts
  CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci,
  ALGORITHM = INPLACE, LOCK = NONE;

-- Set the default for new tables in application code / migration tool
-- my.cnf:
-- character_set_server  = utf8mb4
-- collation_server      = utf8mb4_0900_ai_ci
```

---

### MySQL Anti-Pattern: Random UUID Primary Keys on InnoDB Tables

#### Problem

InnoDB stores all rows in primary key order (clustered index). Random UUIDs cause every
insert to land in a random page, resulting in:

- Constant B-Tree page splits
- Severe fragmentation of the clustered index
- Much larger secondary indexes (each stores the full PK)
- Much worse buffer pool utilization — nearly every insert evicts a warm page

#### Detection

```sql
-- Identify UUID-like VARCHAR or CHAR(36) primary key columns
SELECT table_schema, table_name, column_name, data_type, column_key
FROM   information_schema.columns
WHERE  column_key = 'PRI'
  AND  (data_type IN ('char', 'varchar') AND character_maximum_length = 36)
  AND  table_schema NOT IN ('information_schema','mysql','performance_schema','sys');

-- Measure table fragmentation for a suspected table
SELECT
  table_name,
  ROUND(data_length   / 1024 / 1024, 2) AS data_mb,
  ROUND(index_length  / 1024 / 1024, 2) AS index_mb,
  ROUND(data_free     / 1024 / 1024, 2) AS free_mb,
  ROUND(100 * data_free / NULLIF(data_length + index_length + data_free, 0), 1) AS frag_pct
FROM   information_schema.tables
WHERE  table_schema = 'myapp'
  AND  table_name   = 'orders';
```

#### Recommended Fix

```sql
-- ❌ Anti-pattern: random UUID as PK
CREATE TABLE orders_bad (
    id         CHAR(36)      PRIMARY KEY DEFAULT (UUID()),
    customer_id BIGINT       NOT NULL,
    total_cents BIGINT       NOT NULL
) ENGINE = InnoDB;

-- ✅ Better: BIGINT IDENTITY (sequential, compact)
CREATE TABLE orders_good (
    id          BIGINT        NOT NULL AUTO_INCREMENT PRIMARY KEY,
    customer_id BIGINT        NOT NULL,
    total_cents BIGINT        NOT NULL
) ENGINE = InnoDB;

-- ✅ Also good: UUID stored as binary in time-ordered form (MySQL 8.0+)
CREATE TABLE orders_uuid (
    id          BINARY(16)    NOT NULL DEFAULT (UUID_TO_BIN(UUID(), 1)) PRIMARY KEY,
    customer_id BIGINT        NOT NULL,
    total_cents BIGINT        NOT NULL
) ENGINE = InnoDB;
-- UUID_TO_BIN(..., 1) rearranges the time component to the front,
-- producing monotonically increasing values and eliminating random insertion.
```

---

### MySQL Anti-Pattern: Statement-Based Replication with Non-Deterministic Writes

#### Problem

Statement-based replication (SBR) replays the SQL statement on each replica. If the
statement uses `NOW()`, `RAND()`, `UUID()`, or calls a stored function with side effects,
each replica can produce a different result — silently diverging from the primary.

#### Detection

```sql
-- Check the current binlog format
SHOW VARIABLES LIKE 'binlog_format';
-- If result is 'STATEMENT', investigate every write that uses non-deterministic functions.

-- Search for non-deterministic patterns in stored routines
SELECT routine_schema, routine_name, routine_definition
FROM   information_schema.routines
WHERE  routine_definition REGEXP 'NOW\(\)|RAND\(\)|UUID\(\)|SYSDATE\(\)'
  AND  routine_schema NOT IN ('information_schema','mysql','performance_schema','sys');
```

#### Recommended Fix

```ini
# my.cnf — switch to row-based replication and enable GTID
[mysqld]
binlog_format            = ROW
binlog_row_image         = FULL        # stores full before/after row images
gtid_mode                = ON
enforce_gtid_consistency = ON
```

```sql
-- Verify after restart
SHOW VARIABLES LIKE 'binlog_format';
SHOW VARIABLES LIKE 'gtid_mode';

-- Verify replica is consistent
SHOW REPLICA STATUS\G
-- Look for: Seconds_Behind_Source = 0 and no errors in Last_SQL_Error
```

---

### MySQL Anti-Pattern: Blocking `ALTER TABLE` During Peak Traffic

#### Problem

When MySQL cannot execute a DDL change as `ALGORITHM=INSTANT` or `ALGORITHM=INPLACE`, it
falls back to a full table copy — which holds a metadata lock that blocks all concurrent
reads and writes until it completes. On a 50 M row table that can mean 30–90 minutes of
downtime, even though no one asked for it.

#### Detection

```sql
-- Check what algorithm MySQL would use BEFORE running the real ALTER
EXPLAIN ALTER TABLE orders
  ADD COLUMN priority TINYINT NOT NULL DEFAULT 0,
  ALGORITHM = INSTANT;
-- If error: ER_ALTER_OPERATION_NOT_SUPPORTED, fall back to INPLACE or use gh-ost

-- While an ALTER is running, inspect metadata locks
SELECT
  r.trx_id              AS blocking_trx_id,
  r.trx_mysql_thread_id AS blocking_thread,
  r.trx_query           AS blocking_query,
  b.trx_query           AS blocked_query
FROM   information_schema.innodb_lock_waits w
JOIN   information_schema.innodb_trx b ON b.trx_id = w.requesting_trx_id
JOIN   information_schema.innodb_trx r ON r.trx_id = w.blocking_trx_id;
```

#### Recommended Fix

```sql
-- Step 1: check whether INSTANT is available (fastest, no rebuild)
ALTER TABLE orders
  ADD COLUMN priority TINYINT NOT NULL DEFAULT 0,
  ALGORITHM = INSTANT;

-- Step 2: if INSTANT fails, try INPLACE with no lock (background rebuild)
ALTER TABLE orders
  ADD COLUMN priority TINYINT NOT NULL DEFAULT 0,
  ALGORITHM = INPLACE, LOCK = NONE;
```

```bash
# Step 3: for large tables where even INPLACE causes replica lag, use gh-ost
gh-ost   --host=primary.db.internal   --database=myapp   --table=orders   --alter="ADD COLUMN priority TINYINT NOT NULL DEFAULT 0"   --execute
# gh-ost uses a shadow table and binary log streaming; the cutover is a millisecond rename.
```

---

### SQL Server Anti-Pattern: Using `NOLOCK` as a Default Performance Fix

#### Problem

`WITH (NOLOCK)` / `READ UNCOMMITTED` reads dirty (uncommitted) data and can produce:

- **Phantom rows** — rows that appear and disappear mid-read
- **Missing rows** — rows skipped due to page splits during the scan
- **Duplicate rows** — same row returned twice for the same reason
- **Non-repeatable reads** — the same row returns different values in the same query

In financial and reporting contexts, these produce wrong totals with no error to debug.

#### Detection

```sql
-- Find NOLOCK / READUNCOMMITTED hints in stored procedures, functions, and views
SELECT
    o.type_desc,
    OBJECT_SCHEMA_NAME(o.object_id) + '.' + o.name AS object_name,
    m.definition
FROM sys.sql_modules AS m
JOIN sys.objects     AS o ON o.object_id = m.object_id
WHERE m.definition LIKE '%NOLOCK%'
   OR m.definition LIKE '%READUNCOMMITTED%'
ORDER BY o.type_desc, object_name;
```

#### Recommended Fix

```sql
-- ❌ Anti-pattern: NOLOCK for "speed"
SELECT order_id, total_cents
FROM   dbo.orders WITH (NOLOCK)
WHERE  customer_id = 42;

-- ✅ Step 1: enable RCSI so reads don't block (do this once per database)
ALTER DATABASE appdb SET READ_COMMITTED_SNAPSHOT ON;

-- ✅ Step 2: remove the NOLOCK hints — READ COMMITTED now uses row versions, not locks
SELECT order_id, total_cents
FROM   dbo.orders
WHERE  customer_id = 42;
-- Reads are now non-blocking AND correct.
```

---

### SQL Server Anti-Pattern: Implicit Conversions Between Parameters and Indexed Columns

#### Problem

When a parameter's data type does not exactly match the indexed column's type, SQL Server
must convert every value in the index before it can compare — turning an index seek into a
full scan. This often shows up as "Type conversion in expression" warnings in execution
plans and is one of the most common causes of "fast in testing, slow in production" bugs.

#### Detection

```sql
-- Find implicit conversion warnings in Query Store plans
SELECT TOP 20
    qsq.query_id,
    LEFT(qsqt.query_sql_text, 200)           AS query_text,
    TRY_CAST(qsqp.query_plan AS NVARCHAR(MAX)) AS plan_xml
FROM sys.query_store_query       AS qsq
JOIN sys.query_store_query_text  AS qsqt ON qsqt.query_text_id  = qsq.query_text_id
JOIN sys.query_store_plan        AS qsqp ON qsqp.query_id        = qsq.query_id
WHERE TRY_CAST(qsqp.query_plan AS NVARCHAR(MAX)) LIKE '%PlanAffectingConvert%';
```

#### Recommended Fix

```sql
-- ❌ Anti-pattern: VARCHAR column passed a NVARCHAR literal or variable
-- Column: customer_code VARCHAR(20)
SELECT * FROM dbo.customers WHERE customer_code = N'CUST-001';
-- N'' prefix forces NVARCHAR, triggering a full scan on the VARCHAR index

-- ✅ Match the parameter type exactly
SELECT * FROM dbo.customers WHERE customer_code = 'CUST-001';

-- ❌ Anti-pattern: BIGINT column compared to INT parameter in sp_executesql
EXEC sp_executesql
    N'SELECT * FROM dbo.orders WHERE order_id = @id',
    N'@id INT',        -- column is BIGINT, parameter is INT
    @id = 1001;

-- ✅ Use the correct type
EXEC sp_executesql
    N'SELECT * FROM dbo.orders WHERE order_id = @id',
    N'@id BIGINT',
    @id = 1001;
```

---

### SQL Server Anti-Pattern: Heaps or Unstable Clustered Keys for OLTP Tables

#### Problem

A heap (table with no clustered index) stores rows in insertion order on pages without
any logical sorting. Updates that increase row size produce **forwarded records** — a
pointer chain that requires two page reads per row lookup. Under heavy write load, heap
fragmentation also causes unnecessary I/O on range queries.

An unstable clustered key (random GUID, changing business key) causes the same page-split
and fragmentation pattern that random InnoDB primary keys cause in MySQL.

#### Detection

```sql
-- Tables stored as heaps
SELECT
    OBJECT_SCHEMA_NAME(i.object_id)  AS schema_name,
    OBJECT_NAME(i.object_id)         AS table_name,
    SUM(p.rows)                      AS row_count
FROM sys.indexes    AS i
JOIN sys.partitions AS p
  ON p.object_id = i.object_id AND p.index_id = i.index_id
WHERE i.type = 0   -- HEAP
  AND OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
GROUP BY i.object_id
ORDER BY row_count DESC;

-- Forwarded record count on a specific heap
SELECT
    OBJECT_NAME(ios.object_id)   AS table_name,
    ios.forwarded_fetch_count,
    ios.page_count,
    ios.avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(
    DB_ID(), OBJECT_ID('dbo.heap_table'), NULL, NULL, 'DETAILED') AS ios
WHERE ios.index_id = 0;  -- 0 = heap
```

#### Recommended Fix

```sql
-- ❌ Anti-pattern: table with no clustered index (heap)
CREATE TABLE dbo.events_heap (
    event_id   UNIQUEIDENTIFIER NOT NULL DEFAULT NEWID(),
    event_type NVARCHAR(50)     NOT NULL,
    occurred_at DATETIME2       NOT NULL DEFAULT SYSUTCDATETIME()
);

-- ✅ Better: IDENTITY clustered key (sequential, narrow)
CREATE TABLE dbo.events_good (
    event_id    BIGINT          NOT NULL IDENTITY(1,1),
    event_guid  UNIQUEIDENTIFIER NOT NULL DEFAULT NEWID(),  -- preserve external identifier
    event_type  NVARCHAR(50)    NOT NULL,
    occurred_at DATETIME2       NOT NULL DEFAULT SYSUTCDATETIME(),
    CONSTRAINT PK_events PRIMARY KEY CLUSTERED (event_id)
);

-- ✅ If a GUID external key is required, use NEWSEQUENTIALID() to make it monotonic
CREATE TABLE dbo.events_sequential_guid (
    event_id   UNIQUEIDENTIFIER NOT NULL DEFAULT NEWSEQUENTIALID(),
    event_type NVARCHAR(50)     NOT NULL,
    occurred_at DATETIME2       NOT NULL DEFAULT SYSUTCDATETIME(),
    CONSTRAINT PK_events_sg PRIMARY KEY CLUSTERED (event_id)
);
```

---

### SQL Server Anti-Pattern: Dynamic SQL via `EXEC()` and String Concatenation

#### Problem

Building SQL by concatenating strings and executing with `EXEC()` opens three problems:

1. **SQL injection** — user-controlled input is embedded directly in executable SQL
2. **No plan reuse** — each unique string gets its own plan, filling the procedure cache
3. **Unreliable quoting** — escaping rules vary by data type; mistakes produce silent bugs

#### Detection

```sql
-- Find EXEC( patterns in stored procedures and functions
SELECT
    OBJECT_SCHEMA_NAME(m.object_id) + '.' + OBJECT_NAME(m.object_id) AS object_name,
    o.type_desc
FROM sys.sql_modules AS m
JOIN sys.objects     AS o ON o.object_id = m.object_id
WHERE m.definition LIKE '%EXEC(%'
   OR m.definition LIKE '%EXECUTE (%'
ORDER BY o.type_desc, object_name;
```

#### Recommended Fix

```sql
-- ❌ Anti-pattern: string concatenation + EXEC()
DECLARE @sql  NVARCHAR(500);
DECLARE @id   INT = 42;

SET @sql = 'SELECT * FROM dbo.orders WHERE customer_id = ' + CAST(@id AS NVARCHAR);
EXEC (@sql);   -- New plan compiled every time; injection risk if @id came from user input

-- ✅ Correct: sp_executesql with a typed parameter declaration
DECLARE @id INT = 42;

EXEC sp_executesql
    N'SELECT * FROM dbo.orders WHERE customer_id = @customer_id',
    N'@customer_id INT',
    @customer_id = @id;
-- One compiled plan, parameterized, reused on subsequent calls.

-- ✅ For dynamic table/column names (cannot be parameterized), validate against a whitelist
DECLARE @table_name SYSNAME = N'orders';

IF @table_name NOT IN (
    SELECT name FROM sys.objects WHERE type = 'U')
BEGIN
    RAISERROR('Invalid table name', 16, 1);
    RETURN;
END

EXEC sp_executesql
    N'SELECT TOP 100 * FROM ' + QUOTENAME(@table_name),  -- QUOTENAME escapes safely
    N'';
```

---

### SQL Server Anti-Pattern: Relying on Autogrowth as a Capacity Plan

#### Problem

SQL Server expands data and log files automatically when they run out of space. Each
autogrowth event requires acquiring an exclusive lock on the file while it extends, which
can stall queries for hundreds of milliseconds to several seconds during peak traffic.
Using the default 1 MB growth increment on a 100 GB log file means thousands of growth
events — each one a spike.

#### Detection

```sql
-- Autogrowth events recorded in the default trace
SELECT
    DatabaseName,
    TextData     AS file_name,
    EventClass,
    StartTime,
    DATEDIFF(ms, StartTime, EndTime)   AS growth_duration_ms,
    IntegerData * 8 / 1024             AS growth_mb
FROM fn_trace_gettable(
    (SELECT SUBSTRING(path, 1, LEN(path) - CHARINDEX('', REVERSE(path)))
     FROM   sys.traces WHERE is_default = 1) + '\log.trc', DEFAULT)
WHERE EventClass IN (92, 93)   -- 92 = Log Auto Grow, 93 = Data Auto Grow
ORDER BY StartTime DESC;

-- Current file sizes and free space
SELECT
    DB_NAME(database_id)                        AS db_name,
    name                                        AS file_name,
    type_desc,
    size * 8 / 1024                             AS size_mb,
    CAST(FILEPROPERTY(name, 'SpaceUsed') AS INT) * 8 / 1024 AS used_mb,
    (size - CAST(FILEPROPERTY(name, 'SpaceUsed') AS INT)) * 8 / 1024 AS free_mb,
    growth * 8 / 1024                           AS growth_increment_mb,
    CASE is_percent_growth WHEN 1 THEN 'percent' ELSE 'mb' END AS growth_type
FROM   sys.master_files
WHERE  database_id = DB_ID('appdb')
ORDER BY type_desc, name;
```

#### Recommended Fix

```sql
-- Pre-size the data and log files to match expected workload + 25% headroom
-- Use a single large initial size rather than letting autogrowth handle it

-- Data file: grow to 10 GB immediately, expand in 512 MB steps if needed
ALTER DATABASE appdb
  MODIFY FILE (
    NAME      = 'appdb_data',
    SIZE      = 10240MB,
    FILEGROWTH = 512MB
  );

-- Log file: size to hold at least 30 minutes of peak transaction volume
ALTER DATABASE appdb
  MODIFY FILE (
    NAME      = 'appdb_log',
    SIZE      = 2048MB,
    FILEGROWTH = 256MB
  );

-- tempdb: each file should be the same size, and pre-sized to avoid growth during load
-- Add one file per logical CPU core (up to 8)
ALTER DATABASE tempdb
  MODIFY FILE (NAME = 'tempdev', SIZE = 4096MB, FILEGROWTH = 512MB);
```

---

### Quick Detection Queries Summary

```sql
-- MySQL: all non-InnoDB tables
SELECT table_schema, table_name, engine
FROM   information_schema.tables
WHERE  table_schema NOT IN ('information_schema','mysql','performance_schema','sys')
  AND  engine <> 'InnoDB';

-- MySQL: tables not on utf8mb4
SELECT table_schema, table_name, table_collation
FROM   information_schema.tables
WHERE  table_collation NOT LIKE 'utf8mb4%'
  AND  table_schema NOT IN ('information_schema','mysql','performance_schema','sys');

-- MySQL: char(36) primary keys (likely random UUIDs)
SELECT table_schema, table_name, column_name, data_type, character_maximum_length
FROM   information_schema.columns
WHERE  column_key = 'PRI'
  AND  (data_type = 'char' AND character_maximum_length = 36)
  AND  table_schema NOT IN ('information_schema','mysql','performance_schema','sys');

-- SQL Server: objects containing NOLOCK
SELECT OBJECT_SCHEMA_NAME(m.object_id) + '.' + OBJECT_NAME(m.object_id) AS obj
FROM   sys.sql_modules AS m
WHERE  m.definition LIKE '%NOLOCK%';

-- SQL Server: heaps with significant row counts
SELECT OBJECT_SCHEMA_NAME(i.object_id) AS schema_name,
       OBJECT_NAME(i.object_id) AS table_name,
       SUM(p.rows) AS rows
FROM sys.indexes i JOIN sys.partitions p
  ON p.object_id = i.object_id AND p.index_id = i.index_id
WHERE i.type = 0 AND OBJECTPROPERTY(i.object_id,'IsUserTable') = 1
GROUP BY i.object_id HAVING SUM(p.rows) > 10000
ORDER BY rows DESC;

-- SQL Server: objects with EXEC( dynamic SQL
SELECT OBJECT_SCHEMA_NAME(m.object_id) + '.' + OBJECT_NAME(m.object_id) AS obj
FROM   sys.sql_modules AS m
WHERE  m.definition LIKE '%EXEC(%';
```

---

## Quick Reference Checklist

Use this checklist before releasing a new service or during quarterly audits.

### Schema Design

- [ ] No table has more than 20–25 columns (no God Tables)
- [ ] Each table represents one entity or one relationship
- [ ] No `FLOAT` / `DOUBLE` columns for monetary values — use `NUMERIC` or `BIGINT`
- [ ] EAV is only used for genuinely dynamic user-defined attributes
- [ ] Soft deletes are applied only where business requirements demand restoration

### Query Performance

- [ ] Every foreign key column has an index
- [ ] Critical queries have been verified with `EXPLAIN ANALYZE`
- [ ] No N+1 query patterns — ORM eager loading is used for list endpoints
- [ ] Every `SELECT` has a `LIMIT`, `TOP`, or other explicit row-boundary strategy
- [ ] MySQL primary keys and SQL Server clustered indexes are chosen intentionally for hot tables
- [ ] Deep pagination uses keyset/cursor, not OFFSET
- [ ] Database query count per request is monitored (threshold: < 10)

### Data Integrity

- [ ] All multi-statement writes are wrapped in transactions
- [ ] Auto-commit is disabled or explicitly managed
- [ ] Financial operations use `SELECT ... FOR UPDATE` / locking reads where needed
- [ ] SQL Server correctness-sensitive queries do not depend on `NOLOCK`

### Operations

- [ ] Connection pooling is configured (application-level or PgBouncer/ProxySQL)
- [ ] Pool size is tuned for the workload
- [ ] Replication lag is monitored and alerted on (> 1s threshold)
- [ ] Write-then-read paths route reads to primary or wait for replication
- [ ] MySQL uses InnoDB + `utf8mb4`, and SQL Server `tempdb` / log growth are monitored

### Schema Management

- [ ] All schema changes go through version-controlled migration files
- [ ] Every migration has a rollback function
- [ ] Large table changes use online DDL tools
- [ ] Expand-contract pattern is used for breaking changes

### Scaling

- [ ] Query optimization and caching are exhausted before considering sharding
- [ ] Read replicas are used for read-heavy workloads
- [ ] Vertical scaling is considered before horizontal partitioning

---

## Next Steps

1. **Score yourself** — Use the [Quick Reference Checklist](#quick-reference-checklist) and score your current systems. Any unchecked item is a potential anti-pattern.
2. **Fix high severity first** — Address 🔴 Critical anti-patterns (#1, #2, #5, #8, #11) before others.
3. **Read the companion guide** — [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) describes the correct patterns to replace these anti-patterns.
4. **Optimize queries** — [04-QUERY-OPTIMIZATION.md](04-QUERY-OPTIMIZATION.md) covers EXPLAIN, indexing strategies, and query tuning.
5. **Set up replication** — [05-REPLICATION-AND-SHARDING.md](05-REPLICATION-AND-SHARDING.md) covers replication topologies and consistency guarantees.
6. **Manage migrations** — [06-MIGRATIONS.md](06-MIGRATIONS.md) covers migration tooling and safe schema evolution.

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.2 | 2026 | Added detection SQL and fix examples for every MySQL and SQL Server anti-pattern |
| 1.1 | 2026 | Added MySQL and SQL Server engine-specific anti-pattern guidance |
| 1.0 | 2025 | Initial database anti-patterns documentation |
