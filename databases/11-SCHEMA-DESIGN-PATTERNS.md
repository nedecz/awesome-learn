# Schema Design Patterns

## Table of Contents

1. [Overview](#overview)
   - [What Are Schema Design Patterns?](#what-are-schema-design-patterns)
   - [Patterns vs Data Modeling](#patterns-vs-data-modeling)
   - [How to Use This Guide](#how-to-use-this-guide)
2. [SQL Schema Design Patterns](#sql-schema-design-patterns)
   - [Multi-Tenancy Patterns](#multi-tenancy-patterns)
   - [Slowly Changing Dimensions](#slowly-changing-dimensions)
   - [Event Sourcing Schema](#event-sourcing-schema)
   - [State Machine Pattern](#state-machine-pattern)
   - [Ledger / Double-Entry Pattern](#ledger-double-entry-pattern)
   - [Polymorphic Associations](#polymorphic-associations)
   - [Tagging and Categorization](#tagging-and-categorization)
   - [Closure Table for Hierarchies](#closure-table-for-hierarchies)
3. [NoSQL Schema Design Patterns](#nosql-schema-design-patterns)
   - [Attribute Pattern](#attribute-pattern)
   - [Bucket Pattern](#bucket-pattern)
   - [Computed Pattern](#computed-pattern)
   - [Extended Reference Pattern](#extended-reference-pattern)
   - [Outlier Pattern](#outlier-pattern)
   - [Subset Pattern](#subset-pattern)
   - [Schema Versioning Pattern](#schema-versioning-pattern)
   - [Single-Table Design for DynamoDB](#single-table-design-for-dynamodb)
   - [Wide Partition Pattern for Cassandra](#wide-partition-pattern-for-cassandra)
4. [Cross-Cutting Patterns](#cross-cutting-patterns)
   - [CQRS Read Models](#cqrs-read-models)
   - [Polyglot Persistence](#polyglot-persistence)
   - [Write-Optimized vs Read-Optimized Schemas](#write-optimized-vs-read-optimized-schemas)
5. [Pattern Selection Guide](#pattern-selection-guide)
   - [Decision Framework](#decision-framework)
   - [SQL vs NoSQL Pattern Comparison](#sql-vs-nosql-pattern-comparison)
6. [Version History](#version-history)

---

## Overview

### What Are Schema Design Patterns?

Schema design patterns are named, reusable solutions to recurring data-structuring problems. Just as software design patterns (Gang of Four) provide templates for object-oriented code, schema design patterns provide templates for organizing data in databases. They encode hard-won production experience into repeatable blueprints.

> *"The limits of my language mean the limits of my world."* — Ludwig Wittgenstein (as quoted in DDIA, Chapter 2). The same is true for data models: the schema patterns you know determine the solutions you can see.

Each pattern in this guide describes:

- **The problem** — the recurring situation that motivates the pattern
- **The solution** — the schema structure and associated queries
- **Trade-offs** — what you gain and what you pay
- **When to use it** — concrete criteria for choosing this pattern

### Patterns vs Data Modeling

This document complements [03-DATA-MODELING.md](./03-DATA-MODELING.md), which covers the fundamentals: ER modeling, normalization, embedding vs referencing, star/snowflake schemas, polymorphic inheritance strategies, and schema evolution theory.

**Data modeling** answers: *How do I represent my domain?*
**Schema design patterns** answer: *What proven structures solve this specific operational problem?*

```
  ┌─────────────────────────────┐
  │     Domain Requirements     │
  └─────────────┬───────────────┘
                │
                ▼
  ┌─────────────────────────────┐
  │   Data Modeling (03-*)      │  ← Conceptual / Logical / Physical
  │   ER diagrams, normal forms │
  └─────────────┬───────────────┘
                │
                ▼
  ┌─────────────────────────────┐
  │  Schema Design Patterns     │  ← This document
  │  Named, reusable solutions  │
  │  for production problems    │
  └─────────────────────────────┘
```

### How to Use This Guide

1. **Identify the problem class** — multi-tenancy, time-series, hierarchies, event logs, etc.
2. **Jump to the relevant pattern** — each section is self-contained
3. **Review the trade-offs** — every pattern has costs; pick the one whose costs you can afford
4. **Adapt the examples** — code samples are production-ready starting points, not copy-paste solutions
5. **Combine patterns** — real systems use multiple patterns together (e.g., event sourcing + CQRS)

---

## SQL Schema Design Patterns

### Multi-Tenancy Patterns

Multi-tenancy is the practice of serving multiple customers (tenants) from a single application deployment. The schema strategy you choose affects isolation, performance, operational complexity, and cost.

```
  Strategy A: Shared Schema         Strategy B: Schema-per-Tenant      Strategy C: DB-per-Tenant
  ┌──────────────────────┐          ┌──────────────────────┐           ┌────────────┐
  │   Database           │          │   Database           │           │  DB: acme  │
  │  ┌────────────────┐  │          │  ┌────────────────┐  │           │ ┌────────┐ │
  │  │  orders         │  │          │  │ tenant_acme    │  │           │ │ orders │ │
  │  │ tenant_id = 1  │  │          │  │  ┌──────────┐  │  │           │ └────────┘ │
  │  │ tenant_id = 2  │  │          │  │  │ orders   │  │  │           └────────────┘
  │  │ tenant_id = 3  │  │          │  └──────────────┘  │           ┌────────────┐
  │  └────────────────┘  │          │  ┌────────────────┐  │           │  DB: globex │
  └──────────────────────┘          │  │ tenant_globex  │  │           │ ┌────────┐ │
                                    │  │  ┌──────────┐  │  │           │ │ orders │ │
   Row-level isolation              │  │  │ orders   │  │  │           │ └────────┘ │
   via tenant_id column             │  └──────────────┘  │           └────────────┘
                                    └──────────────────────┘
                                     Schema-level isolation            Full DB isolation
```

#### Shared Schema with Row-Level Security

Every table includes a `tenant_id` column. This is the simplest and most cost-effective approach.

```sql
CREATE TABLE orders (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id   INT NOT NULL REFERENCES tenants(id),
    customer_id BIGINT NOT NULL,
    total       NUMERIC(12,2) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Composite index ensures tenant-scoped queries are fast
CREATE INDEX idx_orders_tenant ON orders (tenant_id, created_at DESC);

-- PostgreSQL Row-Level Security
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.current_tenant')::INT);
```

#### Schema-per-Tenant

Each tenant gets a dedicated schema within the same database.

```sql
-- Create a schema for a new tenant
CREATE SCHEMA tenant_acme;

CREATE TABLE tenant_acme.orders (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    total       NUMERIC(12,2) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Application sets search_path per request
SET search_path TO tenant_acme, public;
```

#### Database-per-Tenant

Each tenant has a completely separate database. Maximum isolation but highest operational cost.

#### Trade-Off Comparison

| Factor | Shared Schema | Schema-per-Tenant | DB-per-Tenant |
|---|---|---|---|
| **Isolation** | Low (row-level) | Medium (schema-level) | High (full) |
| **Cost** | Lowest | Medium | Highest |
| **Operational complexity** | Low | Medium (migrations ×N) | High (backups ×N) |
| **Cross-tenant queries** | Easy | Moderate | Hard |
| **Tenant count scale** | 10,000+ | 100–1,000 | 10–100 |
| **Noisy neighbor risk** | High | Medium | None |
| **Compliance / data residency** | Harder | Moderate | Easiest |

### Slowly Changing Dimensions

Slowly Changing Dimensions (SCDs) are patterns from data warehousing for tracking how dimension attributes change over time. The [03-DATA-MODELING](./03-DATA-MODELING.md) guide covers temporal tables at a conceptual level; here we cover the named SCD types used in practice.

#### SCD Type 1 — Overwrite

Simply overwrite the old value. No history is preserved.

```sql
-- Type 1: Overwrite in place
UPDATE dim_customer
SET city = 'San Francisco',
    updated_at = now()
WHERE customer_id = 42;
```

**Use when:** History is not needed (e.g., correcting a typo).

#### SCD Type 2 — Add New Row

Create a new row for each change, with validity date ranges. This is the most commonly used SCD type.

```sql
CREATE TABLE dim_customer (
    surrogate_key  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id    INT NOT NULL,          -- natural/business key
    name           TEXT NOT NULL,
    city           TEXT NOT NULL,
    valid_from     DATE NOT NULL,
    valid_to       DATE,                  -- NULL = current record
    is_current     BOOLEAN NOT NULL DEFAULT TRUE
);

-- Close the current record and insert a new one
BEGIN;
  UPDATE dim_customer
  SET valid_to = CURRENT_DATE,
      is_current = FALSE
  WHERE customer_id = 42 AND is_current = TRUE;

  INSERT INTO dim_customer (customer_id, name, city, valid_from, is_current)
  VALUES (42, 'Alice', 'San Francisco', CURRENT_DATE, TRUE);
COMMIT;

-- Query: What city was customer 42 in on 2024-06-15?
SELECT city FROM dim_customer
WHERE customer_id = 42
  AND valid_from <= '2024-06-15'
  AND (valid_to IS NULL OR valid_to > '2024-06-15');
```

#### SCD Type 3 — Add New Column

Store the previous value in a separate column. Tracks only one level of history.

```sql
ALTER TABLE dim_customer
    ADD COLUMN previous_city TEXT,
    ADD COLUMN city_changed_on DATE;

UPDATE dim_customer
SET previous_city = city,
    city = 'San Francisco',
    city_changed_on = CURRENT_DATE
WHERE customer_id = 42;
```

#### SCD Type 4 — History Table

Keep the current dimension clean and store all history in a separate table.

```sql
CREATE TABLE dim_customer (
    customer_id  INT PRIMARY KEY,
    name         TEXT NOT NULL,
    city         TEXT NOT NULL
);

CREATE TABLE dim_customer_history (
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id  INT NOT NULL REFERENCES dim_customer(customer_id),
    name         TEXT NOT NULL,
    city         TEXT NOT NULL,
    valid_from   DATE NOT NULL,
    valid_to     DATE NOT NULL
);
```

#### SCD Type 6 — Hybrid (1 + 2 + 3)

Combines Types 1, 2, and 3: versioned rows with a `current_` column that is overwritten on every change.

```sql
CREATE TABLE dim_customer (
    surrogate_key  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id    INT NOT NULL,
    city           TEXT NOT NULL,          -- historical value for this row
    current_city   TEXT NOT NULL,          -- always reflects latest (Type 1)
    valid_from     DATE NOT NULL,
    valid_to       DATE,
    is_current     BOOLEAN NOT NULL DEFAULT TRUE
);

-- On change: insert new row (Type 2) and update ALL rows' current_city (Type 1)
BEGIN;
  UPDATE dim_customer
  SET valid_to = CURRENT_DATE, is_current = FALSE
  WHERE customer_id = 42 AND is_current = TRUE;

  INSERT INTO dim_customer (customer_id, city, current_city, valid_from, is_current)
  VALUES (42, 'San Francisco', 'San Francisco', CURRENT_DATE, TRUE);

  UPDATE dim_customer
  SET current_city = 'San Francisco'
  WHERE customer_id = 42;
COMMIT;
```

### Event Sourcing Schema

Event sourcing stores every state change as an immutable event rather than overwriting current state. The event log is the source of truth; current state is derived by replaying events.

> *"If you are mathematically inclined, you might say that applying an event is like adding an element to a set — it is an additive, monotonically increasing operation."* — DDIA, Chapter 11

#### Event Store Table

```sql
CREATE TABLE event_store (
    event_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type TEXT NOT NULL,           -- e.g., 'Order', 'Account'
    aggregate_id   UUID NOT NULL,           -- the entity this event belongs to
    event_type     TEXT NOT NULL,           -- e.g., 'OrderPlaced', 'ItemAdded'
    event_data     JSONB NOT NULL,          -- the event payload
    metadata       JSONB,                   -- correlation IDs, user info
    version        INT NOT NULL,            -- optimistic concurrency
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Ensure events for an aggregate are strictly ordered
    UNIQUE (aggregate_id, version)
);

CREATE INDEX idx_events_aggregate ON event_store (aggregate_type, aggregate_id, version);
CREATE INDEX idx_events_type ON event_store (event_type, created_at);
```

#### Snapshot Table (Performance Optimization)

Replaying thousands of events per read is expensive. Snapshots cache the computed state at a point in time.

```sql
CREATE TABLE snapshots (
    aggregate_id   UUID NOT NULL,
    aggregate_type TEXT NOT NULL,
    version        INT NOT NULL,            -- snapshot taken at this version
    state          JSONB NOT NULL,          -- serialized aggregate state
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_id, version)
);

-- To rebuild state: load latest snapshot, then replay only newer events
-- SELECT * FROM snapshots
--   WHERE aggregate_id = $1 ORDER BY version DESC LIMIT 1;
-- SELECT * FROM event_store
--   WHERE aggregate_id = $1 AND version > $snapshot_version
--   ORDER BY version;
```

#### Projection Table

Projections are read-optimized views built by processing the event stream.

```sql
CREATE TABLE order_summary (
    order_id     UUID PRIMARY KEY,
    customer_id  UUID NOT NULL,
    status       TEXT NOT NULL,
    item_count   INT NOT NULL DEFAULT 0,
    total        NUMERIC(12,2) NOT NULL DEFAULT 0,
    updated_at   TIMESTAMPTZ NOT NULL
);

-- A projection handler processes events and updates this table
-- e.g., on 'ItemAdded' → UPDATE order_summary SET item_count = item_count + 1 ...
```

### State Machine Pattern

Many domain entities follow a lifecycle with well-defined states and valid transitions. Encoding this in the schema prevents invalid state changes.

```sql
CREATE TABLE order_statuses (
    status TEXT PRIMARY KEY
);
INSERT INTO order_statuses VALUES
    ('pending'), ('confirmed'), ('shipped'), ('delivered'), ('cancelled');

CREATE TABLE order_transitions (
    from_status TEXT NOT NULL REFERENCES order_statuses(status),
    to_status   TEXT NOT NULL REFERENCES order_statuses(status),
    PRIMARY KEY (from_status, to_status)
);
INSERT INTO order_transitions VALUES
    ('pending', 'confirmed'), ('pending', 'cancelled'),
    ('confirmed', 'shipped'), ('confirmed', 'cancelled'),
    ('shipped', 'delivered');

CREATE TABLE orders (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status     TEXT NOT NULL DEFAULT 'pending' REFERENCES order_statuses(status),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Transition function that enforces valid state changes
CREATE OR REPLACE FUNCTION transition_order(
    p_order_id BIGINT,
    p_new_status TEXT
) RETURNS VOID AS $$
DECLARE
    v_current TEXT;
BEGIN
    SELECT status INTO v_current FROM orders WHERE id = p_order_id FOR UPDATE;

    IF NOT EXISTS (
        SELECT 1 FROM order_transitions
        WHERE from_status = v_current AND to_status = p_new_status
    ) THEN
        RAISE EXCEPTION 'Invalid transition: % → %', v_current, p_new_status;
    END IF;

    UPDATE orders SET status = p_new_status, updated_at = now()
    WHERE id = p_order_id;
END;
$$ LANGUAGE plpgsql;
```

```
  State Transition Diagram:

  ┌─────────┐    confirm    ┌───────────┐    ship     ┌─────────┐
  │ pending  │──────────────▶│ confirmed │────────────▶│ shipped │
  └────┬─────┘              └─────┬──────┘             └────┬────┘
       │                          │                         │
       │ cancel                   │ cancel                  │ deliver
       ▼                          ▼                         ▼
  ┌───────────┐            ┌───────────┐            ┌───────────┐
  │ cancelled │            │ cancelled │            │ delivered  │
  └───────────┘            └───────────┘            └───────────┘
```

### Ledger / Double-Entry Pattern

Financial systems require immutable, auditable records. The double-entry bookkeeping pattern ensures that every transaction balances: total debits always equal total credits.

```sql
CREATE TABLE accounts (
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name         TEXT NOT NULL,
    account_type TEXT NOT NULL CHECK (account_type IN ('asset', 'liability', 'equity', 'revenue', 'expense')),
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ledger_entries (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    transaction_id  UUID NOT NULL,          -- groups debit/credit pairs
    account_id      BIGINT NOT NULL REFERENCES accounts(id),
    entry_type      TEXT NOT NULL CHECK (entry_type IN ('debit', 'credit')),
    amount          NUMERIC(15,2) NOT NULL CHECK (amount > 0),
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- CRITICAL: No UPDATE or DELETE allowed — append only
REVOKE UPDATE, DELETE ON ledger_entries FROM app_user;

-- Enforce balanced transactions at the application level or with a trigger
CREATE OR REPLACE FUNCTION check_balanced_transaction()
RETURNS TRIGGER AS $$
BEGIN
    IF (
        SELECT SUM(CASE WHEN entry_type = 'debit' THEN amount ELSE -amount END)
        FROM ledger_entries
        WHERE transaction_id = NEW.transaction_id
    ) <> 0 THEN
        -- Transaction is not yet balanced; allow (other leg coming)
        -- Final balance check happens at COMMIT via deferred constraint or app logic
        NULL;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Query: Account balance
SELECT
    a.name,
    SUM(CASE WHEN le.entry_type = 'debit' THEN le.amount ELSE 0 END) AS total_debits,
    SUM(CASE WHEN le.entry_type = 'credit' THEN le.amount ELSE 0 END) AS total_credits,
    SUM(CASE WHEN le.entry_type = 'debit' THEN le.amount ELSE -le.amount END) AS balance
FROM ledger_entries le
JOIN accounts a ON a.id = le.account_id
GROUP BY a.id, a.name;
```

**Key constraints:**
- **Immutability** — rows are never updated or deleted; corrections are new entries
- **Balanced transactions** — every `transaction_id` group must have equal debits and credits
- **Auditability** — full history is preserved by design

### Polymorphic Associations

When an entity (e.g., a comment) can belong to multiple different parent types (e.g., a post, a photo, or a video), you need a strategy for modeling this relationship. The [03-DATA-MODELING](./03-DATA-MODELING.md) guide covers inheritance-based polymorphism; here we focus on association-based approaches.

#### Discriminator Column Approach

```sql
CREATE TABLE comments (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    commentable_type TEXT NOT NULL,        -- 'post', 'photo', 'video'
    commentable_id   BIGINT NOT NULL,      -- FK cannot be enforced by DB
    body            TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comments_target
    ON comments (commentable_type, commentable_id);
```

**Limitation:** The database cannot enforce a foreign key because `commentable_id` points to different tables depending on `commentable_type`.

#### Exclusive Join Table Approach (Recommended)

```sql
CREATE TABLE comments (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    body       TEXT NOT NULL,
    post_id    BIGINT REFERENCES posts(id),
    photo_id   BIGINT REFERENCES photos(id),
    video_id   BIGINT REFERENCES videos(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Exactly one parent must be set
    CONSTRAINT one_parent CHECK (
        (post_id IS NOT NULL)::INT +
        (photo_id IS NOT NULL)::INT +
        (video_id IS NOT NULL)::INT = 1
    )
);
```

**Advantage:** Full foreign key enforcement. The check constraint guarantees exactly one parent.

### Tagging and Categorization

#### Toxi Pattern (Normalized Tagging)

The Toxi pattern (named after the Toxi tagging system) uses a three-table design that supports many-to-many relationships between entities and tags.

```sql
CREATE TABLE tags (
    id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL UNIQUE
);

CREATE TABLE article_tags (
    article_id BIGINT NOT NULL REFERENCES articles(id) ON DELETE CASCADE,
    tag_id     BIGINT NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (article_id, tag_id)
);

-- Find articles with ALL of the specified tags
SELECT a.id, a.title
FROM articles a
JOIN article_tags at ON a.id = at.article_id
JOIN tags t ON t.id = at.tag_id
WHERE t.name IN ('postgresql', 'performance')
GROUP BY a.id, a.title
HAVING COUNT(DISTINCT t.name) = 2;

-- Find the most popular tags
SELECT t.name, COUNT(*) AS article_count
FROM tags t
JOIN article_tags at ON t.id = at.tag_id
GROUP BY t.name
ORDER BY article_count DESC
LIMIT 20;
```

#### Materialized Path for Category Trees

For hierarchical categories (e.g., `Electronics > Computers > Laptops`), store the full path as a string.

```sql
CREATE TABLE categories (
    id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name      TEXT NOT NULL,
    path      TEXT NOT NULL,             -- e.g., '/1/5/12/'
    depth     INT NOT NULL
);

CREATE INDEX idx_categories_path ON categories USING btree (path);

-- Find all descendants of category 5
SELECT * FROM categories WHERE path LIKE '/1/5/%';

-- Find all ancestors of category 12 (path = '/1/5/12/')
SELECT * FROM categories WHERE '/1/5/12/' LIKE path || '%';
```

### Closure Table for Hierarchies

The closure table pattern stores every ancestor-descendant relationship explicitly, enabling efficient queries on trees of arbitrary depth. Unlike materialized paths, it uses a separate table and supports fast subtree queries without string operations.

```sql
CREATE TABLE nodes (
    id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name  TEXT NOT NULL
);

CREATE TABLE node_closure (
    ancestor_id   BIGINT NOT NULL REFERENCES nodes(id) ON DELETE CASCADE,
    descendant_id BIGINT NOT NULL REFERENCES nodes(id) ON DELETE CASCADE,
    depth         INT NOT NULL DEFAULT 0,
    PRIMARY KEY (ancestor_id, descendant_id)
);

CREATE INDEX idx_closure_desc ON node_closure (descendant_id, depth);
```

#### Inserting a Node

When inserting a child under a parent, copy all ancestor relationships from the parent and add the self-referencing row.

```sql
-- Insert node 'Laptops' (id=7) under 'Computers' (id=3)
INSERT INTO nodes (id, name) OVERRIDING SYSTEM VALUE VALUES (7, 'Laptops');

-- Copy all ancestor paths from parent, add one level of depth
INSERT INTO node_closure (ancestor_id, descendant_id, depth)
    SELECT ancestor_id, 7, depth + 1
    FROM node_closure
    WHERE descendant_id = 3
UNION ALL
    SELECT 7, 7, 0;  -- self-reference
```

#### Querying the Tree

```sql
-- All descendants of node 3 (subtree)
SELECT n.* FROM nodes n
JOIN node_closure c ON n.id = c.descendant_id
WHERE c.ancestor_id = 3 AND c.depth > 0;

-- All ancestors of node 7 (path to root)
SELECT n.* FROM nodes n
JOIN node_closure c ON n.id = c.ancestor_id
WHERE c.descendant_id = 7 AND c.depth > 0
ORDER BY c.depth DESC;

-- Direct children only
SELECT n.* FROM nodes n
JOIN node_closure c ON n.id = c.descendant_id
WHERE c.ancestor_id = 3 AND c.depth = 1;
```

---

## NoSQL Schema Design Patterns

The following patterns are well-established in the MongoDB, DynamoDB, and Cassandra communities. They reflect a fundamental shift from the relational mindset: **design your schema around your queries, not around your entities**.

> *"Document databases target use cases where data comes in self-contained documents and relationships between one document and another are rare."* — DDIA, Chapter 2

### Attribute Pattern

**Problem:** You have entities with varied, sparse attributes (e.g., products with different specs per category). A relational approach would require many nullable columns or an EAV table.

**Solution:** Store attributes as an array of key-value subdocuments.

```javascript
// Before: sparse fields
{
  _id: "prod_1",
  name: "Running Shoe",
  color: "Blue",
  shoe_size: 10,       // only for shoes
  wattage: null,       // only for electronics
  screen_size: null    // only for electronics
}

// After: Attribute Pattern
{
  _id: "prod_1",
  name: "Running Shoe",
  attributes: [
    { key: "color", value: "Blue", unit: null },
    { key: "shoe_size", value: 10, unit: "US" },
    { key: "weight", value: 280, unit: "grams" }
  ]
}
```

```javascript
// Create a wildcard index to query any attribute efficiently
db.products.createIndex({ "attributes.key": 1, "attributes.value": 1 });

// Find all products where shoe_size = 10
db.products.find({
  attributes: { $elemMatch: { key: "shoe_size", value: 10 } }
});
```

**Use when:** Products with varied specs, user profiles with optional fields, IoT devices with different sensor types.

### Bucket Pattern

**Problem:** High-frequency time-series data (sensor readings, logs) creates millions of tiny documents, which increases index size and wastes storage on repeated metadata.

**Solution:** Group measurements into fixed-size buckets (e.g., one document per hour).

```javascript
// Instead of one document per reading:
{ sensor_id: "s1", ts: ISODate("2025-01-15T10:00:01"), temp: 22.1 }
{ sensor_id: "s1", ts: ISODate("2025-01-15T10:00:02"), temp: 22.2 }
// ... thousands more

// Bucket Pattern: one document per hour
{
  sensor_id: "s1",
  bucket_start: ISODate("2025-01-15T10:00:00"),
  bucket_end: ISODate("2025-01-15T10:59:59"),
  count: 120,
  sum_temp: 2658.4,
  measurements: [
    { ts: ISODate("2025-01-15T10:00:01"), temp: 22.1 },
    { ts: ISODate("2025-01-15T10:00:02"), temp: 22.2 }
    // ... up to bucket size limit
  ]
}
```

```javascript
// Upsert: add a reading to the current bucket
db.sensor_data.updateOne(
  {
    sensor_id: "s1",
    count: { $lt: 200 },   // bucket size limit
    bucket_start: {
      $gte: ISODate("2025-01-15T10:00:00"),
      $lt:  ISODate("2025-01-15T11:00:00")
    }
  },
  {
    $push: { measurements: { ts: new Date(), temp: 22.3 } },
    $inc: { count: 1, sum_temp: 22.3 },
    $setOnInsert: {
      bucket_start: ISODate("2025-01-15T10:00:00"),
      bucket_end: ISODate("2025-01-15T10:59:59")
    }
  },
  { upsert: true }
);
```

**Benefits:** Reduces document count by 100–200×, pre-computes aggregates, smaller index footprint.

### Computed Pattern

**Problem:** Frequently accessed derived values (counts, averages, totals) are expensive to compute on every read with aggregation pipelines.

**Solution:** Pre-compute and store the derived values, updating them on writes.

```javascript
// Movie document with pre-computed review stats
{
  _id: "movie_42",
  title: "The Schema Strikes Back",
  review_count: 1584,
  review_sum: 6892,
  avg_rating: 4.35,        // pre-computed: review_sum / review_count
  updated_at: ISODate("2025-01-15T12:00:00")
}
```

```javascript
// When a new review (rating: 5) arrives, update in one atomic operation
db.movies.updateOne(
  { _id: "movie_42" },
  [
    {
      $set: {
        review_count: { $add: ["$review_count", 1] },
        review_sum: { $add: ["$review_sum", 5] },
        avg_rating: {
          $divide: [{ $add: ["$review_sum", 5] }, { $add: ["$review_count", 1] }]
        },
        updated_at: "$$NOW"
      }
    }
  ]
);
```

**Use when:** Dashboards showing totals/averages, leaderboards, any read-heavy scenario where computing on the fly is too slow.

### Extended Reference Pattern

**Problem:** You frequently need to display data from a related document (e.g., showing the customer name on every order). Joining (via `$lookup`) on every read is expensive.

**Solution:** Copy frequently accessed fields from the referenced document into the referencing document.

```javascript
// Order document with extended reference to customer
{
  _id: "order_99",
  customer_id: "cust_42",
  // Extended reference — duplicated from customer doc
  customer_name: "Alice Johnson",
  customer_email: "alice@example.com",
  // Order-specific data
  items: [
    { sku: "WIDGET-1", qty: 3, price: 19.99 }
  ],
  total: 59.97,
  status: "shipped"
}
```

**Key trade-off:** Reads are faster (no `$lookup`), but updates to customer name/email must propagate to all their orders. This is acceptable when the duplicated fields change rarely relative to how often they are read.

```javascript
// When customer name changes, update all their orders (async / background job)
db.orders.updateMany(
  { customer_id: "cust_42" },
  { $set: { customer_name: "Alice Smith", customer_email: "alice.smith@example.com" } }
);
```

### Outlier Pattern

**Problem:** Most documents are within a predictable size, but a small number of outliers have disproportionately large arrays (e.g., a viral post with millions of likes vs typical posts with dozens).

**Solution:** Set a threshold. Below it, store data inline. Above it, overflow into separate documents.

```javascript
// Normal book — reviews fit inline
{
  _id: "book_1",
  title: "Database Internals",
  reviews: [
    { user: "u1", rating: 5, text: "Excellent" },
    { user: "u2", rating: 4, text: "Very good" }
  ],
  has_overflow: false
}

// Outlier book — too many reviews to fit in one document
{
  _id: "book_bestseller",
  title: "The Great Novel",
  reviews: [/* first 500 reviews */],
  has_overflow: true
}

// Overflow document
{
  _id: "book_bestseller_reviews_1",
  book_id: "book_bestseller",
  overflow_page: 1,
  reviews: [/* next 500 reviews */]
}
```

```javascript
// Query: Get all reviews for a book
function getReviews(bookId) {
  const book = db.books.findOne({ _id: bookId });
  let allReviews = book.reviews;

  if (book.has_overflow) {
    const overflows = db.books.find({ book_id: bookId }).sort({ overflow_page: 1 });
    overflows.forEach(doc => { allReviews = allReviews.concat(doc.reviews); });
  }

  return allReviews;
}
```

### Subset Pattern

**Problem:** Documents contain both frequently accessed ("hot") data and rarely accessed ("cold") data. Loading the full document wastes memory and bandwidth.

**Solution:** Split the document into a hot subset (kept in the working set) and a cold archive.

```javascript
// Hot collection: product_listings (what users browse)
{
  _id: "prod_1",
  name: "Wireless Headphones",
  price: 79.99,
  image_url: "/img/prod_1.jpg",
  avg_rating: 4.5,
  recent_reviews: [
    { user: "alice", rating: 5, text: "Great!", date: ISODate("2025-01-14") },
    { user: "bob", rating: 4, text: "Good value", date: ISODate("2025-01-10") }
  ]
}

// Cold collection: product_details (loaded only on product detail page)
{
  _id: "prod_1",
  full_description: "...(2000 words)...",
  specifications: { /* detailed specs */ },
  all_reviews: [ /* hundreds of reviews */ ],
  warranty_info: "...",
  manufacturer_notes: "..."
}
```

**Use when:** E-commerce product listings, social media feeds, any page where initial display needs only a fraction of the total data.

### Schema Versioning Pattern

**Problem:** Document schemas evolve over time, but migrating millions of documents at once causes downtime and heavy I/O.

**Solution:** Embed a `schema_version` field and handle multiple versions in application code, migrating documents lazily on read or in background batches.

```javascript
// Version 1: original schema
{
  _id: "user_1",
  schema_version: 1,
  name: "Alice Johnson",
  address: "123 Main St, Springfield, IL 62701"
}

// Version 2: structured address
{
  _id: "user_2",
  schema_version: 2,
  name: "Bob Smith",
  address: {
    street: "456 Oak Ave",
    city: "Portland",
    state: "OR",
    zip: "97201"
  }
}
```

```javascript
// Application code handles both versions
function normalizeUser(user) {
  if (user.schema_version === 1) {
    // Parse and restructure on read
    const parts = user.address.split(', ');
    user.address = {
      street: parts[0],
      city: parts[1],
      state: parts[2]?.split(' ')[0],
      zip: parts[2]?.split(' ')[1]
    };
    user.schema_version = 2;
    // Optionally write back the upgraded document
    db.users.replaceOne({ _id: user._id }, user);
  }
  return user;
}
```

### Single-Table Design for DynamoDB

DynamoDB is optimized for single-table designs where all entity types are stored in one table and accessed via composite primary keys and Global Secondary Indexes (GSIs).

> *"In a document database, the data model is not predefined — each document can contain whatever keys and values the application finds useful."* — DDIA, Chapter 2

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                         Single Table                                │
  ├──────────────┬────────────────┬──────────┬──────────┬──────────────┤
  │  PK           │ SK              │ GSI1-PK   │ GSI1-SK   │ Data...     │
  ├──────────────┼────────────────┼──────────┼──────────┼──────────────┤
  │ CUST#42      │ PROFILE        │ —        │ —        │ name, email  │
  │ CUST#42      │ ORDER#001      │ ORD#001  │ 2025-01  │ total, items │
  │ CUST#42      │ ORDER#002      │ ORD#002  │ 2025-01  │ total, items │
  │ PROD#99      │ INFO           │ CAT#elec │ PROD#99  │ name, price  │
  │ PROD#99      │ REVIEW#u1      │ —        │ —        │ rating, text │
  └──────────────┴────────────────┴──────────┴──────────┴──────────────┘
```

```json
// Item: Customer profile
{
  "PK": "CUST#42",
  "SK": "PROFILE",
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "entity_type": "Customer"
}
```

```json
// Item: Order belonging to a customer
{
  "PK": "CUST#42",
  "SK": "ORDER#2025-01-15#001",
  "GSI1PK": "ORDER#001",
  "GSI1SK": "2025-01-15",
  "total": 59.97,
  "status": "shipped",
  "entity_type": "Order"
}
```

```javascript
// Access pattern 1: Get customer profile + all orders
const params = {
  TableName: "AppTable",
  KeyConditionExpression: "PK = :pk AND SK >= :sk",
  ExpressionAttributeValues: {
    ":pk": "CUST#42",
    ":sk": "ORDER#"
  }
};

// Access pattern 2: Get a specific order by order ID (via GSI)
const params2 = {
  TableName: "AppTable",
  IndexName: "GSI1",
  KeyConditionExpression: "GSI1PK = :gsi1pk",
  ExpressionAttributeValues: {
    ":gsi1pk": "ORDER#001"
  }
};
```

**Design process for DynamoDB:**
1. List all access patterns first
2. Design composite keys (PK + SK) to support the primary access pattern
3. Add GSIs for secondary access patterns
4. Use sort key overloading to store multiple entity types under one partition key

### Wide Partition Pattern for Cassandra

Cassandra distributes data across nodes by partition key. Within a partition, data is sorted by clustering columns. The **wide partition** pattern exploits this to co-locate related data for efficient range queries.

```sql
-- Time-series: sensor readings partitioned by sensor + day
CREATE TABLE sensor_readings (
    sensor_id   TEXT,
    day         DATE,
    reading_ts  TIMESTAMP,
    temperature DOUBLE,
    humidity    DOUBLE,
    PRIMARY KEY ((sensor_id, day), reading_ts)
) WITH CLUSTERING ORDER BY (reading_ts DESC);
```

```sql
-- All readings for sensor 's1' on 2025-01-15 (single partition scan)
SELECT * FROM sensor_readings
WHERE sensor_id = 's1' AND day = '2025-01-15';

-- Readings in a time range within one partition
SELECT * FROM sensor_readings
WHERE sensor_id = 's1'
  AND day = '2025-01-15'
  AND reading_ts >= '2025-01-15 10:00:00'
  AND reading_ts <= '2025-01-15 12:00:00';
```

```sql
-- User activity feed: partitioned by user, clustered by time
CREATE TABLE user_activity (
    user_id     UUID,
    activity_ts TIMESTAMP,
    activity    TEXT,
    details     MAP<TEXT, TEXT>,
    PRIMARY KEY (user_id, activity_ts)
) WITH CLUSTERING ORDER BY (activity_ts DESC)
   AND default_time_to_live = 7776000;  -- 90-day TTL
```

**Partition sizing guidelines:**

| Metric | Recommended Limit |
|---|---|
| Partition size (disk) | < 100 MB |
| Rows per partition | < 100,000 |
| Cells per partition | < 2 billion (hard limit) |

If a partition grows too large, introduce a **bucket** component in the partition key (e.g., `(sensor_id, month)` instead of just `(sensor_id)`).

---

## Cross-Cutting Patterns

### CQRS Read Models

**Command Query Responsibility Segregation** (CQRS) separates the write model (optimized for transactional integrity) from the read model (optimized for query performance). This is often combined with event sourcing.

```
  ┌──────────┐     Commands      ┌──────────────────┐
  │  Client   │─────────────────▶│  Write Model     │
  │          │                   │  (normalized,     │
  │          │                   │   event store)    │
  │          │                   └────────┬─────────┘
  │          │                            │ events
  │          │                            ▼
  │          │                   ┌──────────────────┐
  │          │     Queries       │  Read Model      │
  │          │◀─────────────────│  (denormalized,   │
  └──────────┘                   │   projections)    │
                                 └──────────────────┘
```

```sql
-- Write model: normalized, relational integrity
CREATE TABLE write_orders (
    id          UUID PRIMARY KEY,
    customer_id UUID NOT NULL REFERENCES customers(id),
    status      TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE write_order_items (
    id       UUID PRIMARY KEY,
    order_id UUID NOT NULL REFERENCES write_orders(id),
    sku      TEXT NOT NULL,
    qty      INT NOT NULL,
    price    NUMERIC(10,2) NOT NULL
);

-- Read model: denormalized, query-optimized (could be in a different database)
CREATE TABLE read_order_dashboard (
    order_id      UUID PRIMARY KEY,
    customer_name TEXT NOT NULL,
    customer_email TEXT NOT NULL,
    status        TEXT NOT NULL,
    item_count    INT NOT NULL,
    total         NUMERIC(12,2) NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL,
    last_updated  TIMESTAMPTZ NOT NULL
);
```

**Key principle:** The read model is disposable and rebuildable. If it gets corrupted, replay events (or re-query the write model) to reconstruct it.

### Polyglot Persistence

**Polyglot persistence** means using different database technologies for different parts of a system, matching each access pattern to the database best suited for it.

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                        Application Layer                        │
  └───────┬──────────┬──────────┬───────────┬──────────┬───────────┘
          │          │          │           │          │
          ▼          ▼          ▼           ▼          ▼
  ┌──────────┐ ┌──────────┐ ┌────────┐ ┌────────┐ ┌──────────┐
  │PostgreSQL│ │ MongoDB  │ │ Redis  │ │Elastic │ │ DynamoDB │
  │          │ │          │ │        │ │ search │ │          │
  │ Orders   │ │ Product  │ │Sessions│ │ Search │ │ Activity │
  │ Payments │ │ Catalog  │ │ Cache  │ │ Index  │ │ Feed     │
  │ Accounts │ │ Content  │ │ Locks  │ │ Logs   │ │ Events   │
  └──────────┘ └──────────┘ └────────┘ └────────┘ └──────────┘
  Transactions  Flexible    Sub-ms     Full-text   Unlimited
  ACID          schema      reads      queries     scale
```

| Access Pattern | Best Fit | Why |
|---|---|---|
| Transactional writes with ACID | PostgreSQL / MySQL | Strong consistency, JOIN support |
| Flexible, nested documents | MongoDB | Schema flexibility, document model |
| Sub-millisecond key lookups | Redis / Memcached | In-memory, simple data structures |
| Full-text search with facets | Elasticsearch / OpenSearch | Inverted index, relevance scoring |
| Massive write throughput | Cassandra / DynamoDB | Distributed, partition-tolerant |
| Graph traversals | Neo4j / Neptune | Native graph storage and queries |

**Trade-off:** Each additional database technology increases operational complexity (monitoring, backups, expertise). Start with one or two and add only when a clear need emerges.

### Write-Optimized vs Read-Optimized Schemas

Every schema exists on a spectrum between write optimization and read optimization.

| Aspect | Write-Optimized | Read-Optimized |
|---|---|---|
| **Structure** | Normalized (3NF+) | Denormalized (star schema, materialized views) |
| **Indexes** | Minimal (reduce write amplification) | Many (cover all query patterns) |
| **Joins on read** | Many (data is split across tables) | Few or none (pre-joined) |
| **Write speed** | Fast (single row insert) | Slower (update multiple locations) |
| **Read speed** | Slower (joins at query time) | Fast (pre-computed, pre-joined) |
| **Storage** | Minimal (no duplication) | Higher (intentional duplication) |
| **Consistency** | Easy (single source of truth) | Harder (duplicated data may drift) |

> *"In an OLTP database, we typically use an index structure like a B-tree, which seeks to minimize the time to read a particular piece of data. In a data warehouse, we might use column-oriented storage, which seeks to minimize the amount of data read from disk."* — DDIA, Chapter 3

**Practical guidance:**
- **OLTP systems** (user-facing transactions): Start write-optimized, add targeted denormalization
- **OLAP systems** (analytics, reporting): Start read-optimized with star schemas
- **Mixed workloads**: Use CQRS to have both

---

## Pattern Selection Guide

### Decision Framework

Use this table to map your problem to the right pattern:

| Problem / Use Case | Recommended Pattern | Section |
|---|---|---|
| Multiple customers in one database | Multi-Tenancy (shared schema) | [Link](#multi-tenancy-patterns) |
| Enterprise customers requiring isolation | Multi-Tenancy (DB-per-tenant) | [Link](#multi-tenancy-patterns) |
| Tracking attribute history over time | SCD Type 2 or Type 6 | [Link](#slowly-changing-dimensions) |
| Complete audit log of every state change | Event Sourcing | [Link](#event-sourcing-schema) |
| Entity with well-defined lifecycle states | State Machine Pattern | [Link](#state-machine-pattern) |
| Financial transactions, accounting | Ledger / Double-Entry | [Link](#ledger-double-entry-pattern) |
| Comments/reactions on multiple entity types | Polymorphic Associations | [Link](#polymorphic-associations) |
| User-generated tags on content | Toxi Pattern | [Link](#tagging-and-categorization) |
| Deep hierarchies with subtree queries | Closure Table | [Link](#closure-table-for-hierarchies) |
| Products with varied attributes | Attribute Pattern | [Link](#attribute-pattern) |
| High-frequency time-series data | Bucket Pattern | [Link](#bucket-pattern) |
| Dashboards with counts/averages | Computed Pattern | [Link](#computed-pattern) |
| Frequent joins to get display fields | Extended Reference | [Link](#extended-reference-pattern) |
| Occasional documents with huge arrays | Outlier Pattern | [Link](#outlier-pattern) |
| Hot/cold data split | Subset Pattern | [Link](#subset-pattern) |
| Evolving document schemas | Schema Versioning Pattern | [Link](#schema-versioning-pattern) |
| DynamoDB with multiple entity types | Single-Table Design | [Link](#single-table-design-for-dynamodb) |
| Cassandra time-series or feeds | Wide Partition | [Link](#wide-partition-pattern-for-cassandra) |
| Separate read/write performance needs | CQRS | [Link](#cqrs-read-models) |
| Different DB for each access pattern | Polyglot Persistence | [Link](#polyglot-persistence) |

### SQL vs NoSQL Pattern Comparison

Some problems have solutions in both paradigms. This table compares the approaches:

| Problem | SQL Approach | NoSQL Approach | Choose SQL When | Choose NoSQL When |
|---|---|---|---|---|
| Hierarchies | Closure Table | Embedded subdocuments / Materialized Path | Deep trees, need subtree aggregations | Shallow nesting, read-heavy |
| Polymorphic data | Exclusive join table | Flexible schema per document | Referential integrity is critical | Attributes vary significantly |
| Time-series | Partitioned tables + SCD | Bucket Pattern (MongoDB) / Wide Partition (Cassandra) | Need ACID + joins with other data | Massive write throughput required |
| Event history | Event store table | Append-only collection | Need transactions across aggregates | Single aggregate per event stream |
| Pre-computed reads | Materialized views | Computed Pattern | DB-managed refresh is acceptable | Need real-time incremental updates |
| Multi-tenancy | RLS / schema-per-tenant | Database-per-tenant (managed) | Shared infra, cost-sensitive | Strict isolation, compliance |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial schema design patterns documentation |
