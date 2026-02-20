# Query Optimization

## Table of Contents

1. [Overview](#overview)
   - [Why Query Optimization Matters](#why-query-optimization-matters)
   - [The Query Planner / Optimizer](#the-query-planner--optimizer)
2. [Reading Execution Plans](#reading-execution-plans)
   - [EXPLAIN in PostgreSQL](#explain-in-postgresql)
   - [EXPLAIN ANALYZE in PostgreSQL](#explain-analyze-in-postgresql)
   - [EXPLAIN in MySQL](#explain-in-mysql)
   - [Scan Types](#scan-types)
   - [Cost Estimation](#cost-estimation)
3. [Indexing Strategies](#indexing-strategies)
   - [B-Tree Indexes](#b-tree-indexes)
   - [Hash Indexes](#hash-indexes)
   - [Multi-Column (Composite) Indexes](#multi-column-composite-indexes)
   - [Covering Indexes (INCLUDE)](#covering-indexes-include)
   - [Partial Indexes](#partial-indexes)
   - [Expression Indexes](#expression-indexes)
   - [Index-Only Scans](#index-only-scans)
4. [Common Query Anti-Patterns](#common-query-anti-patterns)
   - [Functions on Indexed Columns (SARGability)](#functions-on-indexed-columns-sargability)
   - [SELECT *](#select-)
   - [N+1 Queries](#n1-queries)
   - [Implicit Type Casts](#implicit-type-casts)
   - [OR on Different Columns](#or-on-different-columns)
   - [Leading Wildcard LIKE](#leading-wildcard-like)
5. [Join Optimization](#join-optimization)
   - [Nested Loop Join](#nested-loop-join)
   - [Hash Join](#hash-join)
   - [Merge Join](#merge-join)
   - [Join Strategy Comparison](#join-strategy-comparison)
   - [Join Order Matters](#join-order-matters)
   - [When to Denormalize](#when-to-denormalize)
6. [Aggregate and Window Function Optimization](#aggregate-and-window-function-optimization)
   - [GROUP BY Optimization](#group-by-optimization)
   - [Materialized Views](#materialized-views)
   - [Pre-Aggregation](#pre-aggregation)
7. [Statistics and the Query Planner](#statistics-and-the-query-planner)
   - [ANALYZE](#analyze)
   - [Histogram Statistics](#histogram-statistics)
   - [Cardinality Estimation](#cardinality-estimation)
   - [Auto-Vacuum's Role](#auto-vacuums-role)
8. [Practical Tuning Checklist](#practical-tuning-checklist)
9. [Version History](#version-history)

---

## Overview

### Why Query Optimization Matters

A single slow query can bring down an entire production system. As tables grow from thousands to millions of rows, queries that once ran in milliseconds can degrade to seconds or minutes. Query optimization is the practice of structuring queries, indexes, and schemas so the database can retrieve data with the least possible work.

This document focuses on relational databases — primarily PostgreSQL and MySQL — but the principles apply broadly. It builds on the storage engine concepts from [00-OVERVIEW.md](00-OVERVIEW.md) (B-Trees, LSM-Trees) and the schema design patterns in [03-DATA-MODELING.md](03-DATA-MODELING.md). It draws on ideas from *Designing Data-Intensive Applications* (DDIA) by Martin Kleppmann, particularly Chapter 3 on storage and retrieval, which explains how indexes trade write speed for read speed.

> *"An index is an additional structure that is derived from the primary data. Many databases allow you to add and remove indexes, and this doesn't affect the contents of the database; it only affects the performance of queries."* — DDIA, Chapter 3

### The Query Planner / Optimizer

Every SQL query goes through a multi-stage pipeline before results are returned:

```
    ┌─────────────┐     ┌─────────────┐     ┌─────────────────┐     ┌──────────────┐
    │   SQL Text   │────▶│   Parser    │────▶│  Query Planner  │────▶│   Executor   │
    │              │     │             │     │  / Optimizer     │     │              │
    └─────────────┘     └─────────────┘     └─────────────────┘     └──────────────┘
                                                    │
                                                    ▼
                                            ┌──────────────┐
                                            │  Execution   │
                                            │  Plan        │
                                            │              │
                                            │  • Scan type │
                                            │  • Join algo │
                                            │  • Sort      │
                                            │  • Filter    │
                                            └──────────────┘
```

The **query planner** (also called the optimizer) is the most critical component. It:

1. **Parses** the SQL into an abstract syntax tree
2. **Rewrites** the query (e.g., flattening subqueries, applying view definitions)
3. **Generates** multiple candidate execution plans
4. **Estimates** the cost of each plan using table statistics
5. **Selects** the plan with the lowest estimated cost

The planner does not run the query — it reasons about costs using statistics (row counts, value distributions, index availability). If those statistics are stale, the planner can choose a bad plan.

---

## Reading Execution Plans

The execution plan is your primary diagnostic tool. Learning to read plans is the single most valuable query optimization skill.

### EXPLAIN in PostgreSQL

`EXPLAIN` shows the plan the planner *would* choose, without actually running the query.

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
```

```
                                    QUERY PLAN
---------------------------------------------------------------------------
 Index Scan using idx_orders_customer_id on orders  (cost=0.43..8.45 rows=5 width=64)
   Index Cond: (customer_id = 42)
```

Key fields in the output:

| Field | Meaning |
|---|---|
| `cost=0.43..8.45` | Estimated startup cost .. total cost (in arbitrary units) |
| `rows=5` | Estimated number of rows returned |
| `width=64` | Estimated average row size in bytes |
| `Index Scan` | The access method chosen |
| `Index Cond` | The condition pushed down to the index |

### EXPLAIN ANALYZE in PostgreSQL

`EXPLAIN ANALYZE` actually executes the query and shows real timings alongside estimates. This is the gold standard for diagnosis.

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, o.total, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.created_at >= '2025-01-01'
  AND o.status = 'shipped';
```

```
                                          QUERY PLAN
---------------------------------------------------------------------------------------------
 Hash Join  (cost=12.50..245.80 rows=120 width=48)
            (actual time=0.35..2.10 rows=98 loops=1)
   Hash Cond: (o.customer_id = c.id)
   Buffers: shared hit=45 read=3
   -> Bitmap Heap Scan on orders o  (cost=4.80..220.30 rows=120 width=28)
                                     (actual time=0.20..1.60 rows=98 loops=1)
        Recheck Cond: (created_at >= '2025-01-01'::date)
        Filter: (status = 'shipped')
        Rows Removed by Filter: 42
        Buffers: shared hit=38 read=3
        -> Bitmap Index Scan on idx_orders_created_at  (cost=0.00..4.75 rows=160 width=0)
                                                        (actual time=0.10..0.10 rows=140 loops=1)
             Index Cond: (created_at >= '2025-01-01'::date)
   -> Hash  (cost=5.20..5.20 rows=200 width=24)
             (actual time=0.08..0.08 rows=200 loops=1)
        Buffers: shared hit=7
        -> Seq Scan on customers c  (cost=0.00..5.20 rows=200 width=24)
                                     (actual time=0.01..0.05 rows=200 loops=1)
 Planning Time: 0.15 ms
 Execution Time: 2.25 ms
```

**Reading the plan (bottom-up):**

1. The innermost nodes execute first
2. `Bitmap Index Scan` on `idx_orders_created_at` finds 140 rows matching the date filter
3. `Bitmap Heap Scan` fetches those rows from the heap, then applies the `status = 'shipped'` filter (removing 42 rows)
4. `Seq Scan` on `customers` loads all 200 customer rows into a hash table
5. `Hash Join` matches orders to customers

**Red flags to watch for:**

| Symptom | Likely Problem |
|---|---|
| `actual rows` >> `rows` (estimate) | Stale statistics — run `ANALYZE` |
| `Rows Removed by Filter` is very high | Missing index on the filter column |
| `Seq Scan` on a large table | Missing index or planner chose to ignore it |
| `Buffers: read` is very high | Data not in shared buffers — cold cache or table too large |
| `loops=N` where N is large | Nested loop with many iterations — consider a hash join |

### EXPLAIN in MySQL

MySQL uses a different format. The most useful variant:

```sql
EXPLAIN FORMAT=TREE
SELECT o.id, o.total, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.created_at >= '2025-01-01'
  AND o.status = 'shipped';
```

The traditional tabular `EXPLAIN` output:

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
```

| id | select_type | table | type | possible_keys | key | rows | Extra |
|---|---|---|---|---|---|---|---|
| 1 | SIMPLE | orders | ref | idx_customer_id | idx_customer_id | 5 | NULL |

Key MySQL `type` values (best to worst):

| type | Description |
|---|---|
| `system` / `const` | Single row lookup (primary key = constant) |
| `eq_ref` | Unique index lookup in a join (one row per join) |
| `ref` | Non-unique index lookup |
| `range` | Index range scan (BETWEEN, >, <) |
| `index` | Full index scan (reads entire index, no table) |
| `ALL` | Full table scan — usually a problem |

### Scan Types

Understanding scan types is fundamental. These determine how the database accesses data on disk.

```
    ┌──────────────────────────────────────────────────────────────────────────┐
    │                         Scan Types (fastest to slowest)                  │
    │                                                                          │
    │  Index Scan ──▶ Bitmap Index Scan ──▶ Index-Only Scan ──▶ Seq Scan     │
    │  (few rows)     (moderate rows)       (index covers all    (many rows   │
    │                                        needed columns)      or no index) │
    └──────────────────────────────────────────────────────────────────────────┘
```

| Scan Type | How It Works | When Used |
|---|---|---|
| **Sequential Scan** | Reads every row in the table, in storage order | No usable index, or query reads a large fraction of rows |
| **Index Scan** | Traverses the B-Tree index, then fetches each matching row from the heap (table) | Few rows match; index is selective |
| **Bitmap Index Scan** | Builds a bitmap of matching pages from the index, then fetches pages in physical order | Moderate selectivity; reduces random I/O vs index scan |
| **Index-Only Scan** | Reads data directly from the index without accessing the heap | All required columns are in the index (covering index) |

**When does the planner prefer a sequential scan?**

If the query returns more than roughly 5–15% of the table, a sequential scan is often cheaper than an index scan because sequential I/O is faster than the random I/O of hopping between index and heap.

### Cost Estimation

PostgreSQL's cost model uses configurable parameters:

| Parameter | Default | Description |
|---|---|---|
| `seq_page_cost` | 1.0 | Cost of reading a page sequentially |
| `random_page_cost` | 4.0 | Cost of reading a random page (index lookup) |
| `cpu_tuple_cost` | 0.01 | Cost of processing a row |
| `cpu_index_tuple_cost` | 0.005 | Cost of processing an index entry |
| `effective_cache_size` | 4 GB | Planner's estimate of available cache |

The planner combines these with table statistics to estimate total cost. On SSDs, you may want to lower `random_page_cost` to 1.1–1.5, since random reads are nearly as fast as sequential reads.

```sql
-- Check current settings
SHOW random_page_cost;
SHOW effective_cache_size;

-- Adjust for SSD-based storage
SET random_page_cost = 1.1;
```

---

## Indexing Strategies

Indexes are the single most impactful tool for query optimization. As DDIA Chapter 3 explains, an index is a derived data structure that speeds up reads at the cost of slower writes — every insert, update, or delete must also update every relevant index.

### B-Tree Indexes

B-Tree is the default index type in PostgreSQL and MySQL (InnoDB). It stores keys in sorted order, making it efficient for both equality and range queries.

```sql
-- Equality lookup
CREATE INDEX idx_orders_status ON orders (status);
SELECT * FROM orders WHERE status = 'shipped';

-- Range query
CREATE INDEX idx_orders_created_at ON orders (created_at);
SELECT * FROM orders WHERE created_at BETWEEN '2025-01-01' AND '2025-01-31';

-- Sorting — the index can provide pre-sorted output, avoiding a sort step
SELECT * FROM orders WHERE status = 'shipped' ORDER BY created_at DESC;
-- Optimal index for this query:
CREATE INDEX idx_orders_status_created ON orders (status, created_at DESC);
```

**B-Tree supports these operators:** `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `IN`, `IS NULL`, pattern matches with a fixed prefix (`LIKE 'abc%'`).

### Hash Indexes

Hash indexes support only equality comparisons (`=`). They are smaller and faster than B-Trees for pure equality lookups but cannot help with range queries or sorting.

```sql
-- PostgreSQL (hash indexes are WAL-logged and crash-safe since v10)
CREATE INDEX idx_sessions_token ON sessions USING hash (token);

-- Only useful for:
SELECT * FROM sessions WHERE token = 'abc123def456';

-- NOT useful for:
SELECT * FROM sessions WHERE token > 'abc';  -- range query, hash index ignored
```

In MySQL InnoDB, hash indexes exist only in the adaptive hash index (internal optimization). You cannot explicitly create hash indexes on InnoDB tables.

### Multi-Column (Composite) Indexes

A composite index covers multiple columns. Column order is critical — the index is sorted by the first column, then by the second within each first-column value, and so on.

```
    Index on (status, created_at, customer_id):

    ┌──────────────────────────────────────────────────────────┐
    │  status     │  created_at    │  customer_id  │  row ptr  │
    ├──────────────────────────────────────────────────────────┤
    │  cancelled  │  2025-01-05    │  12           │  ───▶     │
    │  cancelled  │  2025-01-09    │  37           │  ───▶     │
    │  pending    │  2025-01-02    │  5            │  ───▶     │
    │  pending    │  2025-01-08    │  22           │  ───▶     │
    │  shipped    │  2025-01-01    │  42           │  ───▶     │
    │  shipped    │  2025-01-03    │  18           │  ───▶     │
    │  shipped    │  2025-01-07    │  42           │  ───▶     │
    └──────────────────────────────────────────────────────────┘
```

**The leftmost prefix rule:** the index can be used for queries that filter on a prefix of its columns, from left to right.

```sql
CREATE INDEX idx_orders_composite ON orders (status, created_at, customer_id);

-- ✓ Uses the index (matches leftmost prefix)
SELECT * FROM orders WHERE status = 'shipped';
SELECT * FROM orders WHERE status = 'shipped' AND created_at > '2025-01-01';
SELECT * FROM orders WHERE status = 'shipped' AND created_at > '2025-01-01' AND customer_id = 42;

-- ✗ Cannot use this index efficiently
SELECT * FROM orders WHERE created_at > '2025-01-01';                     -- skips first column
SELECT * FROM orders WHERE customer_id = 42;                              -- skips first two columns
SELECT * FROM orders WHERE status = 'shipped' AND customer_id = 42;       -- gap in the middle*
```

*PostgreSQL can use an Index Scan for `status = 'shipped'` and then filter on `customer_id`, but it cannot do a range scan on `customer_id` through the index.

**Column order guidelines:**

1. Place **equality** columns first (`WHERE status = ?`)
2. Place **range** columns after equality columns (`WHERE created_at > ?`)
3. Place **sort** columns last if they align with `ORDER BY`

### Covering Indexes (INCLUDE)

A covering index contains all columns needed by a query, enabling an **index-only scan** that never touches the heap.

```sql
-- PostgreSQL 11+ INCLUDE syntax
CREATE INDEX idx_orders_covering ON orders (customer_id)
    INCLUDE (total, created_at);

-- This query can be satisfied entirely from the index:
SELECT total, created_at
FROM orders
WHERE customer_id = 42;
-- Plan: Index Only Scan using idx_orders_covering
```

The `INCLUDE` columns are stored in the index leaf pages but are not part of the sort order. This keeps the index tree efficient while still enabling index-only scans.

In MySQL, all InnoDB secondary indexes implicitly include the primary key, and you can add columns to a composite index to make it covering:

```sql
-- MySQL covering index (no INCLUDE syntax — add columns to the key)
CREATE INDEX idx_orders_covering ON orders (customer_id, total, created_at);
```

### Partial Indexes

A partial index covers only a subset of rows, defined by a `WHERE` clause. This creates a smaller, more efficient index.

```sql
-- Only index active orders (most queries filter for these)
CREATE INDEX idx_orders_active ON orders (customer_id, created_at)
    WHERE status != 'cancelled';

-- Only index unprocessed items
CREATE INDEX idx_queue_pending ON job_queue (priority, created_at)
    WHERE processed_at IS NULL;

-- Partial unique index for soft deletes (from 03-DATA-MODELING.md)
CREATE UNIQUE INDEX idx_users_email_active ON users (email)
    WHERE deleted_at IS NULL;
```

Partial indexes are a PostgreSQL feature. MySQL does not support them directly (use generated columns as a workaround).

### Expression Indexes

An expression index (also called a functional index) indexes the result of an expression, not the raw column value.

```sql
-- Index on lower-cased email for case-insensitive lookups
CREATE INDEX idx_users_email_lower ON users (LOWER(email));

-- Query must use the same expression:
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';  -- ✓ uses index
SELECT * FROM users WHERE email = 'Alice@example.com';          -- ✗ does not use index

-- Index on extracted year
CREATE INDEX idx_orders_year ON orders (EXTRACT(YEAR FROM created_at));
SELECT * FROM orders WHERE EXTRACT(YEAR FROM created_at) = 2025;

-- MySQL 8.0+ supports expression indexes:
CREATE INDEX idx_users_email_lower ON users ((LOWER(email)));
-- Note the double parentheses in MySQL
```

### Index-Only Scans

An index-only scan is the fastest access path — the database reads data directly from the index without visiting the table heap. Requirements:

1. All columns in `SELECT`, `WHERE`, `JOIN`, and `ORDER BY` must be in the index
2. In PostgreSQL, the visibility map must confirm all rows on the page are visible (all-visible). If many rows were recently updated, PostgreSQL falls back to a regular index scan to check visibility. Running `VACUUM` updates the visibility map.

```sql
-- Assume this index:
CREATE INDEX idx_orders_cust_total ON orders (customer_id) INCLUDE (total);

-- Index-only scan possible:
EXPLAIN SELECT total FROM orders WHERE customer_id = 42;
-- Index Only Scan using idx_orders_cust_total on orders

-- Index-only scan NOT possible (needs 'status' which is not in the index):
EXPLAIN SELECT total, status FROM orders WHERE customer_id = 42;
-- Index Scan using idx_orders_cust_total on orders (falls back to heap access)
```

---

## Common Query Anti-Patterns

These patterns silently kill performance. Each one prevents the database from using indexes effectively.

### Functions on Indexed Columns (SARGability)

A predicate is **SARGable** (Search ARGument able) if the database can use an index to satisfy it. Wrapping an indexed column in a function destroys SARGability.

```sql
-- ✗ Non-SARGable: function on the indexed column prevents index use
SELECT * FROM orders WHERE YEAR(created_at) = 2025;
SELECT * FROM orders WHERE UPPER(name) = 'ALICE';
SELECT * FROM orders WHERE amount + tax > 100;

-- ✓ SARGable: rewrite to keep the column bare
SELECT * FROM orders WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01';
SELECT * FROM orders WHERE name = 'ALICE';   -- with a case-insensitive collation
SELECT * FROM orders WHERE amount > 100 - tax;  -- if tax is a constant

-- ✓ Alternative: create an expression index (if rewrite is not possible)
CREATE INDEX idx_orders_year ON orders (EXTRACT(YEAR FROM created_at));
```

### SELECT *

`SELECT *` fetches every column, which:

- Prevents index-only scans (the index rarely covers all columns)
- Transfers unnecessary data over the network
- Breaks when columns are added or reordered

```sql
-- ✗ Anti-pattern
SELECT * FROM orders WHERE customer_id = 42;

-- ✓ Select only what you need
SELECT id, total, created_at FROM orders WHERE customer_id = 42;
```

### N+1 Queries

The N+1 problem occurs when application code issues one query to fetch a list, then N additional queries to fetch related data for each item. This is common in ORM code.

```sql
-- ✗ N+1 pattern (pseudocode):
orders = query("SELECT * FROM orders WHERE status = 'pending'")     -- 1 query
for order in orders:
    customer = query("SELECT * FROM customers WHERE id = ?", order.customer_id)  -- N queries

-- ✓ Fix: use a JOIN
SELECT o.id, o.total, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'pending';

-- ✓ Alternative: batch the lookups
SELECT * FROM customers WHERE id IN (1, 2, 3, 42, 99);
```

**Impact:** An N+1 pattern with 1,000 orders makes 1,001 database round-trips. The JOIN makes 1.

### Implicit Type Casts

When the column type and the value type don't match, the database may cast the column, making the predicate non-SARGable.

```sql
-- Table definition
CREATE TABLE users (
    id    SERIAL PRIMARY KEY,
    phone VARCHAR(20)
);
CREATE INDEX idx_users_phone ON users (phone);

-- ✗ Implicit cast: comparing varchar to integer forces a cast on every row
SELECT * FROM users WHERE phone = 5551234;
-- PostgreSQL: casts phone to integer → Seq Scan

-- ✓ Match types explicitly
SELECT * FROM users WHERE phone = '5551234';
-- Uses idx_users_phone → Index Scan
```

### OR on Different Columns

An `OR` across different columns often forces a sequential scan because no single index covers both conditions.

```sql
-- ✗ OR on different columns — neither index alone is sufficient
SELECT * FROM orders WHERE customer_id = 42 OR status = 'urgent';

-- ✓ Rewrite as UNION ALL (allows each branch to use its own index)
SELECT * FROM orders WHERE customer_id = 42
UNION ALL
SELECT * FROM orders WHERE status = 'urgent' AND customer_id != 42;
```

PostgreSQL may use a **BitmapOr** to combine two bitmap index scans, but this depends on the planner's cost estimate. The `UNION ALL` rewrite gives you explicit control.

### Leading Wildcard LIKE

A `LIKE` pattern with a leading wildcard (`%abc`) cannot use a B-Tree index because the index is sorted by the beginning of the string.

```sql
-- ✗ Leading wildcard — cannot use B-Tree index
SELECT * FROM products WHERE name LIKE '%widget%';

-- ✓ Trailing wildcard — uses B-Tree index
SELECT * FROM products WHERE name LIKE 'widget%';

-- ✓ For full-text search, use dedicated features:
-- PostgreSQL: GIN index with pg_trgm for substring matching
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_products_name_trgm ON products USING gin (name gin_trgm_ops);
SELECT * FROM products WHERE name LIKE '%widget%';  -- now uses GIN index

-- PostgreSQL: full-text search with tsvector
CREATE INDEX idx_products_fts ON products USING gin (to_tsvector('english', name));
SELECT * FROM products WHERE to_tsvector('english', name) @@ to_tsquery('widget');

-- MySQL: FULLTEXT index
CREATE FULLTEXT INDEX idx_products_ft ON products (name);
SELECT * FROM products WHERE MATCH(name) AGAINST('widget');
```

---

## Join Optimization

The join algorithm chosen by the planner has a significant impact on performance. PostgreSQL supports three main algorithms; MySQL InnoDB primarily uses nested loop and hash joins (hash join added in MySQL 8.0.18).

### Nested Loop Join

For each row in the outer table, scan the inner table for matching rows. Efficient when the outer side is small and the inner side has an index.

```
    Outer (orders)          Inner (customers)
    ┌────────────┐          ┌───────────────┐
    │ cust_id=42 │───scan──▶│ id=42 ✓ match │
    │ cust_id=7  │───scan──▶│ id=7  ✓ match │
    │ cust_id=42 │───scan──▶│ id=42 ✓ match │
    │ ...        │          │ ...           │
    └────────────┘          └───────────────┘

    Cost: O(N × M) without index, O(N × log M) with index on inner
    Best when: outer is small, inner is indexed
```

```sql
-- Plan output example:
-- Nested Loop  (cost=0.43..52.10 rows=5 width=48)
--   -> Seq Scan on orders o  (cost=0.00..25.00 rows=5 width=28)
--         Filter: (status = 'pending')
--   -> Index Scan using customers_pkey on customers c  (cost=0.43..5.42 rows=1 width=24)
--         Index Cond: (id = o.customer_id)
```

### Hash Join

Build a hash table from the smaller (inner) table, then probe it with each row from the larger (outer) table. No index needed.

```
    Build phase:                    Probe phase:
    ┌────────────────────┐          ┌────────────────────┐
    │ customers table    │          │ orders table        │
    │                    │          │                     │
    │ id=1  → hash(1)   │          │ cust_id=42 → hash   │──▶ lookup in
    │ id=7  → hash(7)   │          │ cust_id=7  → hash   │──▶ hash table
    │ id=42 → hash(42)  │          │ cust_id=42 → hash   │──▶ ✓ match
    │ ...                │          │ ...                 │
    └────────────────────┘          └────────────────────┘
         Hash Table                      Probe

    Cost: O(N + M) — linear in total rows
    Best when: no useful index, both tables fit in memory (or at least the smaller one)
```

**Warning:** If the hash table exceeds `work_mem`, it spills to disk and performance degrades significantly.

```sql
-- Increase work_mem for a specific query
SET work_mem = '256MB';
EXPLAIN ANALYZE SELECT ...;
RESET work_mem;
```

### Merge Join

Both inputs must be sorted on the join key. The algorithm walks through both sorted lists in tandem, merging matches. Very efficient if the data is already sorted (e.g., from an index).

```
    Sorted orders             Sorted customers
    ┌──────────────┐          ┌──────────────┐
    │ cust_id=1    │◀──┐ ┌──▶│ id=1         │
    │ cust_id=7    │   ├─┤   │ id=7         │
    │ cust_id=7    │   │ │   │ id=42        │
    │ cust_id=42   │───┘ └───│ id=99        │
    │ cust_id=42   │          │ ...          │
    └──────────────┘          └──────────────┘
    Both sides advance in sorted order — no backtracking

    Cost: O(N log N + M log M) if sorting needed, O(N + M) if pre-sorted
    Best when: both inputs already sorted (index), or very large tables
```

### Join Strategy Comparison

| Algorithm | Best For | Index Required? | Memory | Time Complexity |
|---|---|---|---|---|
| **Nested Loop** | Small outer, indexed inner | Yes (on inner) | Low | O(N × log M) |
| **Hash Join** | Medium-to-large tables, no index | No | O(smaller table) | O(N + M) |
| **Merge Join** | Large pre-sorted tables | No (but helps) | Low | O(N + M) |

### Join Order Matters

For multi-table joins, the order in which tables are joined can dramatically affect performance. The planner evaluates many orderings, but the search space grows factorially with the number of tables.

```sql
-- PostgreSQL limits exhaustive search to 12 tables by default
SHOW geqo_threshold;  -- Default: 12

-- For queries joining more than 12 tables, PostgreSQL switches to
-- a genetic algorithm (GEQO) which finds a good-but-not-optimal plan

-- Hint: if you know the best join order, you can guide the planner:
SET join_collapse_limit = 1;  -- Forces joins to execute in the written order
SELECT ...
FROM small_table s
JOIN medium_table m ON m.id = s.medium_id
JOIN large_table l ON l.id = m.large_id;
RESET join_collapse_limit;
```

**Rule of thumb:** join from the most selective (fewest rows) to the least selective, and ensure every join key is indexed.

### When to Denormalize

If a frequently executed query joins many tables and performance is unacceptable even after indexing, consider denormalization — storing redundant data to eliminate joins.

| Approach | Trade-off |
|---|---|
| **Add a redundant column** | Faster reads, must keep column in sync on writes |
| **Materialized view** | Pre-computed join result, refreshed periodically or on-demand |
| **Application-level cache** | Avoids the DB entirely; cache invalidation is the challenge |
| **JSONB column** | Store related data inline; good for read-heavy, write-light patterns |

```sql
-- Example: denormalize customer_name into orders to avoid a join
ALTER TABLE orders ADD COLUMN customer_name VARCHAR(200);

-- Keep it in sync via trigger
CREATE OR REPLACE FUNCTION sync_customer_name()
RETURNS TRIGGER AS $$
BEGIN
    NEW.customer_name := (SELECT name FROM customers WHERE id = NEW.customer_id);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_customer_name
    BEFORE INSERT OR UPDATE OF customer_id ON orders
    FOR EACH ROW EXECUTE FUNCTION sync_customer_name();
```

Denormalization trades write complexity for read speed. Only do it after proving the join is the bottleneck.

---

## Aggregate and Window Function Optimization

### GROUP BY Optimization

`GROUP BY` can be expensive because it typically requires sorting or hashing all input rows. Two planner strategies:

```
    HashAggregate:                     GroupAggregate (Sort + Aggregate):
    ┌──────────────────────┐           ┌──────────────────────┐
    │ Build hash table     │           │ Sort by group key    │
    │ keyed by group cols  │           │ (or use index order) │
    │                      │           │                      │
    │ For each row:        │           │ Walk sorted rows:    │
    │   hash(group_key)    │           │   accumulate until   │
    │   update aggregate   │           │   group key changes  │
    └──────────────────────┘           └──────────────────────┘
    Better for few groups              Better when pre-sorted
    Needs work_mem                     (e.g., index on group key)
```

```sql
-- If you frequently group by status and aggregate:
SELECT status, COUNT(*), SUM(total)
FROM orders
GROUP BY status;

-- An index on (status) lets the planner use GroupAggregate with an index scan,
-- avoiding a sort and a hash table:
CREATE INDEX idx_orders_status ON orders (status);
```

### Materialized Views

A materialized view stores the result of a query physically. It is useful for expensive aggregations that don't need real-time freshness.

```sql
-- Create a materialized view for daily sales
CREATE MATERIALIZED VIEW mv_daily_sales AS
SELECT
    DATE(created_at) AS sale_date,
    COUNT(*)         AS order_count,
    SUM(total)       AS revenue
FROM orders
WHERE status = 'completed'
GROUP BY DATE(created_at);

-- Create an index on the materialized view for fast lookups
CREATE UNIQUE INDEX idx_mv_daily_sales_date ON mv_daily_sales (sale_date);

-- Query the materialized view instead of the base table
SELECT * FROM mv_daily_sales WHERE sale_date >= '2025-01-01';

-- Refresh the data (blocks reads until complete)
REFRESH MATERIALIZED VIEW mv_daily_sales;

-- Refresh without blocking reads (requires a unique index)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_sales;
```

**Refresh strategies:**

| Strategy | Freshness | Complexity |
|---|---|---|
| Scheduled cron job (`REFRESH` every N minutes) | Minutes stale | Low |
| Trigger-based refresh on source table changes | Near real-time | High |
| Application-triggered refresh after batch writes | After batch | Medium |

### Pre-Aggregation

For high-volume event data, maintain a summary table updated by triggers or batch jobs.

```sql
-- Summary table maintained incrementally
CREATE TABLE order_stats (
    stat_date   DATE PRIMARY KEY,
    order_count INTEGER NOT NULL DEFAULT 0,
    revenue     NUMERIC(12,2) NOT NULL DEFAULT 0
);

-- Trigger to maintain the summary on each insert
CREATE OR REPLACE FUNCTION update_order_stats()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO order_stats (stat_date, order_count, revenue)
    VALUES (DATE(NEW.created_at), 1, NEW.total)
    ON CONFLICT (stat_date) DO UPDATE
    SET order_count = order_stats.order_count + 1,
        revenue     = order_stats.revenue + EXCLUDED.revenue;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_order_stats
    AFTER INSERT ON orders
    FOR EACH ROW EXECUTE FUNCTION update_order_stats();

-- Dashboard query is now instant:
SELECT * FROM order_stats WHERE stat_date >= '2025-01-01' ORDER BY stat_date;
```

Pre-aggregation is the technique behind OLAP cubes and time-series databases (e.g., TimescaleDB continuous aggregates).

---

## Statistics and the Query Planner

The query planner is only as good as its statistics. Stale or inaccurate statistics lead to bad plans — the most common cause of sudden query regressions.

### ANALYZE

`ANALYZE` collects statistics about column value distributions and stores them in `pg_statistic` (PostgreSQL) or the data dictionary (MySQL).

```sql
-- Analyze a specific table
ANALYZE orders;

-- Analyze specific columns
ANALYZE orders (customer_id, status);

-- Analyze all tables in the database
ANALYZE;

-- MySQL equivalent
ANALYZE TABLE orders;
```

**When to run ANALYZE:**

- After bulk inserts or deletes (loading data, purging old rows)
- After creating a new index
- When `EXPLAIN ANALYZE` shows large discrepancies between estimated and actual row counts
- After a major data distribution change

### Histogram Statistics

PostgreSQL samples rows and builds histograms to estimate the selectivity of predicates. The `default_statistics_target` controls the number of histogram buckets (default: 100).

```sql
-- Increase statistics granularity for a specific column
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 500;
ANALYZE orders;

-- View the statistics
SELECT
    attname,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    histogram_bounds
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

| Statistic | What It Tells the Planner |
|---|---|
| `n_distinct` | Number of distinct values (or ratio if negative) |
| `most_common_vals` | Most frequent values and their frequencies |
| `histogram_bounds` | Evenly distributed boundaries for non-MCV values |
| `correlation` | How physically ordered the column is (affects index scan cost) |

### Cardinality Estimation

Cardinality estimation — predicting how many rows a query step will produce — is the hardest problem in query optimization. Bad estimates cascade:

```
    Estimated rows: 10          Actual rows: 50,000
    ┌──────────────┐            ┌──────────────┐
    │  Nested Loop │            │  Nested Loop │  ← Should have been a Hash Join
    │  (cheap for  │            │  (disastrous │
    │   10 rows)   │            │   for 50K)   │
    └──────────────┘            └──────────────┘
         Good plan                    Bad plan
```

**Common causes of bad estimates:**

| Cause | Solution |
|---|---|
| Stale statistics | Run `ANALYZE` |
| Correlated columns (planner assumes independence) | Create extended statistics (`CREATE STATISTICS`) |
| Skewed data distributions | Increase `default_statistics_target` |
| Complex expressions in WHERE | Simplify predicates or use expression indexes |

```sql
-- PostgreSQL 10+: extended statistics for correlated columns
CREATE STATISTICS stat_orders_status_region (dependencies, ndistinct, mcv)
    ON status, region FROM orders;
ANALYZE orders;
```

### Auto-Vacuum's Role

PostgreSQL's auto-vacuum process does two critical things for query optimization:

1. **Reclaims dead tuples** — MVCC creates dead rows on UPDATE/DELETE. Without vacuum, tables bloat and sequential scans get slower.
2. **Updates statistics** — auto-vacuum runs `ANALYZE` after a threshold of row changes (default: 10% of the table has changed).

```
    Auto-vacuum triggers:
    ┌──────────────────────────────────────────────────────────┐
    │  VACUUM runs when:                                       │
    │    dead tuples > autovacuum_vacuum_threshold              │
    │                 + autovacuum_vacuum_scale_factor × n_rows │
    │    (default: 50 + 0.2 × n_rows)                         │
    │                                                          │
    │  ANALYZE runs when:                                      │
    │    rows changed > autovacuum_analyze_threshold            │
    │                  + autovacuum_analyze_scale_factor × n_rows│
    │    (default: 50 + 0.1 × n_rows)                         │
    └──────────────────────────────────────────────────────────┘
```

For large tables (hundreds of millions of rows), the default scale factor means vacuum/analyze may not run often enough. Tune per table:

```sql
-- More aggressive vacuum and analyze for a hot table
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,     -- vacuum after 1% change instead of 20%
    autovacuum_analyze_scale_factor = 0.005,   -- analyze after 0.5% change instead of 10%
    autovacuum_vacuum_cost_delay = 2           -- speed up vacuum (less throttling)
);
```

---

## Practical Tuning Checklist

Use this step-by-step approach when diagnosing a slow query.

### Step 1: Measure First

```sql
-- Get the actual execution plan with timing and buffer info
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) <your query>;

-- Record the baseline: execution time, rows scanned, buffers hit/read
```

### Step 2: Check Row Estimates

Compare `rows` (estimated) vs `actual rows` at every node. Large mismatches indicate stale statistics.

```sql
-- If estimates are off:
ANALYZE <table>;
-- Then re-run EXPLAIN ANALYZE
```

### Step 3: Look for Sequential Scans on Large Tables

If you see `Seq Scan` on a table with millions of rows and the query is selective, add an appropriate index.

### Step 4: Check for Missing Indexes

```sql
-- PostgreSQL: find tables with high sequential scan counts
SELECT
    schemaname,
    relname,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch
FROM pg_stat_user_tables
WHERE seq_scan > 100
ORDER BY seq_tup_read DESC
LIMIT 20;
```

### Step 5: Check for Unused Indexes

Indexes that are never scanned waste write performance and storage.

```sql
-- PostgreSQL: find indexes that are never used
SELECT
    indexrelname,
    relname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%pkey%'
  AND indexrelname NOT LIKE '%unique%'
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Step 6: Eliminate Anti-Patterns

Review the query for the anti-patterns in [Common Query Anti-Patterns](#common-query-anti-patterns):

- [ ] No functions on indexed columns (SARGable predicates)
- [ ] No `SELECT *` — select only needed columns
- [ ] No N+1 queries — use JOINs or batch lookups
- [ ] No implicit type casts — match column and value types
- [ ] No `OR` across different columns — use `UNION ALL` if needed
- [ ] No leading wildcard `LIKE` — use full-text search or trigram indexes

### Step 7: Consider Index Design

Use the index selection flowchart:

```
    Query has WHERE clause?
    ├── Yes
    │   ├── Equality only (=, IN)?
    │   │   ├── Single column → B-Tree or Hash index
    │   │   └── Multiple columns → Composite index (equality cols first)
    │   ├── Range (<, >, BETWEEN)?
    │   │   └── B-Tree (range column after equality columns)
    │   ├── Pattern match (LIKE 'abc%')?
    │   │   └── B-Tree with text_pattern_ops
    │   └── Substring search (LIKE '%abc%')?
    │       └── GIN with pg_trgm or full-text search
    ├── Query returns few columns?
    │   └── Consider INCLUDE to enable index-only scan
    └── Query filters on a subset of rows?
        └── Consider a partial index
```

### Step 8: Tune work_mem for Complex Queries

If the plan shows `Sort Method: external merge` or hash batches spilling to disk:

```sql
-- Increase work_mem for the session
SET work_mem = '256MB';
EXPLAIN ANALYZE <your query>;
RESET work_mem;

-- If it helps, set it for specific queries in application code,
-- not globally (global increases multiply across all connections)
```

### Step 9: Monitor Continuously

```sql
-- PostgreSQL: enable slow query logging
ALTER SYSTEM SET log_min_duration_statement = '200ms';
SELECT pg_reload_conf();

-- PostgreSQL: pg_stat_statements extension for top queries by time
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT
    query,
    calls,
    mean_exec_time,
    total_exec_time,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

### Quick Reference Summary

| Problem | Solution |
|---|---|
| Slow query, no obvious cause | Run `EXPLAIN (ANALYZE, BUFFERS)` |
| Estimated rows wildly wrong | Run `ANALYZE`; increase statistics target |
| Seq Scan on large table | Add an index on the filter column |
| Index exists but not used | Check for non-SARGable predicates, type mismatches |
| Join is slow | Ensure join keys are indexed; check join algorithm |
| Sort spills to disk | Increase `work_mem` for the session |
| Table bloat / slow scans | Check auto-vacuum; run `VACUUM FULL` if needed |
| Dashboard query too slow | Materialized view or pre-aggregation table |
| N+1 queries from ORM | Rewrite as JOIN or use eager loading |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial query optimization documentation |
