# Schema Design Principles

## Table of Contents

1. [Overview](#overview)
   - [What Are Schema Design Principles?](#what-are-schema-design-principles)
   - [Principles vs Patterns vs Models](#principles-vs-patterns-vs-models)
   - [How to Use This Guide](#how-to-use-this-guide)
2. [Naming Conventions](#naming-conventions)
   - [Table Naming](#table-naming)
   - [Column Naming](#column-naming)
   - [Index and Constraint Naming](#index-and-constraint-naming)
   - [Foreign Key Naming](#foreign-key-naming)
   - [Boolean Column Naming](#boolean-column-naming)
   - [Reserved Words and Prefixes](#reserved-words-and-prefixes)
3. [Primary Key Design](#primary-key-design)
   - [Natural vs Surrogate Keys](#natural-vs-surrogate-keys)
   - [Auto-Increment vs UUID](#auto-increment-vs-uuid)
   - [UUID v7 for Sortable IDs](#uuid-v7-for-sortable-ids)
   - [Composite Keys](#composite-keys)
   - [Choosing the Right Strategy](#choosing-the-right-strategy)
4. [Data Type Selection](#data-type-selection)
   - [General Type Guidelines](#general-type-guidelines)
   - [Common Type Mistakes](#common-type-mistakes)
   - [Timestamps and Time Zones](#timestamps-and-time-zones)
   - [Numeric Precision for Money](#numeric-precision-for-money)
   - [JSON and JSONB Columns](#json-and-jsonb-columns)
   - [Enum vs Lookup Table](#enum-vs-lookup-table)
5. [Constraint Strategy](#constraint-strategy)
   - [NOT NULL by Default](#not-null-by-default)
   - [CHECK Constraints for Domain Rules](#check-constraints-for-domain-rules)
   - [UNIQUE Constraints](#unique-constraints)
   - [Foreign Keys and Referential Integrity](#foreign-keys-and-referential-integrity)
   - [Deferrable Constraints](#deferrable-constraints)
   - [Database vs Application Validation](#database-vs-application-validation)
6. [Normalization Decisions](#normalization-decisions)
   - [When to Normalize](#when-to-normalize)
   - [When to Denormalize](#when-to-denormalize)
   - [Practical Rules of Thumb](#practical-rules-of-thumb)
   - [Joins vs Anomalies Trade-Off](#joins-vs-anomalies-trade-off)
7. [Index Alignment](#index-alignment)
   - [Design Schemas with Indexing in Mind](#design-schemas-with-indexing-in-mind)
   - [Covering Indexes](#covering-indexes)
   - [Partial Indexes](#partial-indexes)
   - [Index-Friendly Column Types](#index-friendly-column-types)
   - [Indexing Anti-Patterns](#indexing-anti-patterns)
8. [Temporal Data Design](#temporal-data-design)
   - [Created and Updated Timestamps](#created-and-updated-timestamps)
   - [Soft Deletes vs Hard Deletes](#soft-deletes-vs-hard-deletes)
   - [Audit Trails](#audit-trails)
   - [Effective Dating](#effective-dating)
9. [Schema Evolution Principles](#schema-evolution-principles)
   - [Design for Change](#design-for-change)
   - [Additive-Only Migrations](#additive-only-migrations)
   - [Backward Compatibility](#backward-compatibility)
   - [The Expand-Contract Pattern](#the-expand-contract-pattern)
10. [Multi-Database Principles](#multi-database-principles)
    - [SQL vs NoSQL Design Differences](#sql-vs-nosql-design-differences)
    - [Document Embedding Principles](#document-embedding-principles)
    - [Key-Value Key Design](#key-value-key-design)
    - [Wide-Column Partition Key Selection](#wide-column-partition-key-selection)
11. [Principle Checklist](#principle-checklist)
12. [Version History](#version-history)

---

## Overview

### What Are Schema Design Principles?

Schema design principles are the foundational, timeless guidelines that inform every schema decision you make — regardless of the database engine, the application domain, or the specific design pattern you choose. They are the "why" and "how to think" behind good schemas.

> *"The hardest part of building a software system is deciding precisely what to build."* — Fred Brooks. The same applies to data: the hardest part of schema design is deciding precisely what constraints, types, and structures to commit to.

Principles differ from patterns and models:

- **Principles** tell you *how to think* — "prefer NOT NULL unless you have a reason for NULL"
- **Patterns** tell you *what to build* — "use a closure table for hierarchies"
- **Models** tell you *how to represent* — "this is a one-to-many relationship"

### Principles vs Patterns vs Models

This document sits between the foundational data modeling concepts in [03-DATA-MODELING.md](./03-DATA-MODELING.md) and the concrete, named patterns in [11-SCHEMA-DESIGN-PATTERNS.md](./11-SCHEMA-DESIGN-PATTERNS.md).

```
  ┌───────────────────────────────────┐
  │       Domain Requirements         │
  └───────────────┬───────────────────┘
                  │
                  ▼
  ┌───────────────────────────────────┐
  │   Data Modeling (03-*)            │  ← Entities, relationships, normal forms
  │   "How do I represent my domain?" │
  └───────────────┬───────────────────┘
                  │
                  ▼
  ┌───────────────────────────────────┐
  │   Design Principles (12-*)       │  ← This document
  │   "What rules guide every        │
  │    schema decision I make?"       │
  └───────────────┬───────────────────┘
                  │
                  ▼
  ┌───────────────────────────────────┐
  │   Design Patterns (11-*)         │  ← Named, reusable solutions
  │   "What proven structures solve   │
  │    this specific problem?"        │
  └───────────────────────────────────┘
```

Think of it this way: you use **models** to understand your domain, **principles** to make sound decisions at every step, and **patterns** to apply battle-tested solutions to specific problems.

### How to Use This Guide

1. **Read the principles once** — internalize the guidelines so they become second nature
2. **Apply them during design** — use the [Principle Checklist](#principle-checklist) as a pre-flight check before finalizing any schema
3. **Reference during review** — use specific principles to justify or challenge schema decisions in code reviews
4. **Combine with patterns** — principles apply regardless of which pattern from [11-SCHEMA-DESIGN-PATTERNS.md](./11-SCHEMA-DESIGN-PATTERNS.md) you are implementing

---

## Naming Conventions

Good naming is the cheapest form of documentation. A well-named schema is self-describing; a poorly-named one generates confusion for years.

### Table Naming

**Use snake_case, plural nouns.** This is the dominant convention in PostgreSQL and most SQL databases.

```sql
-- Good
CREATE TABLE customers (...);
CREATE TABLE order_items (...);
CREATE TABLE user_preferences (...);

-- Bad
CREATE TABLE Customer (...);       -- PascalCase — inconsistent in SQL
CREATE TABLE orderItems (...);     -- camelCase — requires quoting in PostgreSQL
CREATE TABLE tbl_order_item (...); -- Hungarian notation — adds noise
```

**Rules:**

| Guideline | Example | Rationale |
|---|---|---|
| Plural table names | `customers`, `orders` | A table is a collection of rows |
| snake_case | `order_items` | No quoting needed, universally readable |
| No prefixes like `tbl_` | `orders` not `tbl_orders` | Every object is already a table — the prefix is redundant |
| Junction tables: both entity names | `student_courses` | Clearly describes the relationship |

### Column Naming

**Be explicit, avoid abbreviations, use snake_case.**

```sql
-- Good
CREATE TABLE orders (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id     BIGINT NOT NULL REFERENCES customers(id),
    shipping_address TEXT NOT NULL,
    total_amount    NUMERIC(12,2) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Bad: abbreviations, ambiguity, inconsistent style
CREATE TABLE orders (
    oid        BIGINT PRIMARY KEY,     -- "oid" is cryptic
    cust       BIGINT NOT NULL,        -- which customer field?
    addr       TEXT,                    -- shipping? billing?
    amt        NUMERIC(12,2),          -- amount of what?
    crtd       TIMESTAMPTZ             -- unreadable
);
```

**Column naming rules:**

- Foreign keys: `<referenced_table_singular>_id` — e.g., `customer_id`, `order_id`
- Timestamps: `<action>_at` — e.g., `created_at`, `updated_at`, `deleted_at`, `shipped_at`
- Booleans: `is_<adjective>` or `has_<noun>` — e.g., `is_active`, `has_discount`
- Counts: `<noun>_count` — e.g., `item_count`, `retry_count`

### Index and Constraint Naming

Name every index and constraint explicitly. Auto-generated names like `orders_pkey1` are difficult to reference in migrations and error messages.

```sql
-- Naming pattern: <type>_<table>_<columns>
-- idx  = index
-- uq   = unique constraint
-- chk  = check constraint
-- fk   = foreign key
-- pk   = primary key

CREATE TABLE orders (
    id          BIGINT GENERATED ALWAYS AS IDENTITY,
    customer_id BIGINT NOT NULL,
    status      TEXT NOT NULL,
    total       NUMERIC(12,2) NOT NULL,

    CONSTRAINT pk_orders PRIMARY KEY (id),
    CONSTRAINT fk_orders_customer FOREIGN KEY (customer_id) REFERENCES customers(id),
    CONSTRAINT chk_orders_total CHECK (total >= 0)
);

CREATE INDEX idx_orders_customer_id ON orders (customer_id);
CREATE INDEX idx_orders_status ON orders (status) WHERE status != 'archived';
```

### Foreign Key Naming

Foreign keys deserve special attention because they encode relationships:

```
Pattern: fk_<source_table>_<target_table>[_<qualifier>]

Examples:
  fk_orders_customer         — orders.customer_id → customers.id
  fk_orders_shipping_address — orders.shipping_address_id → addresses.id
  fk_order_items_order       — order_items.order_id → orders.id
  fk_order_items_product     — order_items.product_id → products.id
```

When a table has multiple foreign keys to the same target, add a qualifier:

```sql
-- An order has both a billing and shipping address
CONSTRAINT fk_orders_billing_address
    FOREIGN KEY (billing_address_id) REFERENCES addresses(id),
CONSTRAINT fk_orders_shipping_address
    FOREIGN KEY (shipping_address_id) REFERENCES addresses(id)
```

### Boolean Column Naming

Boolean columns should read naturally in a WHERE clause:

```sql
-- Good: reads like English
SELECT * FROM users WHERE is_active = true;
SELECT * FROM orders WHERE has_discount = true;
SELECT * FROM emails WHERE is_verified = true;

-- Bad: ambiguous
SELECT * FROM users WHERE active = true;    -- is it a status? a type?
SELECT * FROM users WHERE status = true;    -- status of what?
SELECT * FROM emails WHERE verified = true; -- when? by whom?
```

### Reserved Words and Prefixes

Avoid SQL reserved words as identifiers. If you must use one, quote it — but prefer renaming:

```sql
-- Bad: "user", "order", "group", "table", "column" are reserved
CREATE TABLE "user" (...);
CREATE TABLE "order" (...);

-- Good: use plurals or qualifiers
CREATE TABLE users (...);
CREATE TABLE orders (...);
CREATE TABLE user_groups (...);
```

**Common reserved words to avoid as column names:** `user`, `order`, `group`, `table`, `column`, `index`, `type`, `key`, `value`, `name`, `date`, `time`, `timestamp`.

When you need `type` or `status` as a column name, they are generally acceptable since they rarely conflict in modern PostgreSQL — but prefix if ambiguous: `order_type`, `account_status`.

---

## Primary Key Design

The primary key is the most important design decision for a table. It affects joins, indexing, replication, partitioning, and API design.

### Natural vs Surrogate Keys

A **natural key** is a value that exists in the real world (email, ISBN, SSN). A **surrogate key** is a system-generated identifier with no business meaning (auto-increment integer, UUID).

```
  Natural Key                          Surrogate Key
  ┌─────────────────────────┐          ┌─────────────────────────┐
  │  email: alice@co.com    │          │  id: 48291              │
  │  isbn: 978-0-13-468599  │          │  id: a3f7...c9e2        │
  │  ssn: 123-45-6789       │          │                         │
  └─────────────────────────┘          └─────────────────────────┘
  ✓ Meaningful                         ✓ Immutable
  ✓ No extra column needed             ✓ Compact (for integers)
  ✗ Can change (email, name)           ✓ Universally applicable
  ✗ May be sensitive (SSN)             ✗ No business meaning
  ✗ Composite keys get complex         ✗ Extra column / storage
```

**Default recommendation:** Use surrogate keys for most tables. Add UNIQUE constraints on natural keys that need to be enforced.

```sql
CREATE TABLE customers (
    id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email TEXT NOT NULL,
    name  TEXT NOT NULL,

    CONSTRAINT uq_customers_email UNIQUE (email)
);
```

Natural keys are appropriate when:

- The value is truly immutable (ISO country codes, currency codes)
- The table is a lookup/reference table
- The value is the only column (e.g., a set of tags)

### Auto-Increment vs UUID

| Factor | Auto-Increment (BIGINT) | UUID (v4) | UUID v7 |
|---|---|---|---|
| **Size** | 8 bytes | 16 bytes | 16 bytes |
| **Sortable** | Yes (by insertion order) | No | Yes (by time) |
| **Globally unique** | No (per-table only) | Yes | Yes |
| **Guessable** | Yes (sequential) | No | Partially (timestamp prefix) |
| **Index performance** | Excellent (B-tree friendly) | Poor (random scatter) | Good (time-ordered) |
| **Distributed generation** | Requires coordination | No coordination | No coordination |
| **API exposure** | Leaks count/order | Safe | Safe |

### UUID v7 for Sortable IDs

UUID v7 (RFC 9562) embeds a Unix timestamp in the first 48 bits, making it time-ordered while remaining globally unique. This gives you the best of both worlds: distributed generation without B-tree index fragmentation.

```sql
-- PostgreSQL 17+ has built-in UUIDv7 support
-- For earlier versions, use the pg_uuidv7 extension
CREATE EXTENSION IF NOT EXISTS pg_uuidv7;

CREATE TABLE events (
    id         UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    event_type TEXT NOT NULL,
    payload    JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- UUIDv7 values sort chronologically:
-- 018f6e2c-8a00-7000-8000-000000000001  (earlier)
-- 018f6e2c-8b00-7000-8000-000000000002  (later)
```

**When to use UUIDv7:**

- Client-generated IDs (mobile apps, SPAs creating records offline)
- Distributed systems where coordinating sequences is expensive
- APIs where exposing auto-increment IDs is a security concern
- Event stores where time-ordering matters

### Composite Keys

Composite keys use two or more columns as the primary key. They are appropriate for junction tables and relationship-centric data.

```sql
-- Junction table: composite key is the natural choice
CREATE TABLE student_courses (
    student_id BIGINT NOT NULL REFERENCES students(id),
    course_id  BIGINT NOT NULL REFERENCES courses(id),
    enrolled_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT pk_student_courses PRIMARY KEY (student_id, course_id)
);

-- Avoid: adding a surrogate key to a junction table adds no value
CREATE TABLE student_courses (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY, -- unnecessary
    student_id BIGINT NOT NULL REFERENCES students(id),
    course_id  BIGINT NOT NULL REFERENCES courses(id)
);
```

**Composite key guidelines:**

- Use for junction/association tables where the combination is the natural identifier
- Keep composite keys to 2–3 columns maximum
- Beware: composite keys propagate into child tables as multi-column foreign keys
- If the composite key appears in more than one child table, consider a surrogate key instead

### Choosing the Right Strategy

```
  Is the table a junction/association?
  ├── Yes → Composite key (student_id, course_id)
  └── No
      │
      Is the system distributed or client-generates IDs?
      ├── Yes → UUID v7
      └── No
          │
          Is the ID exposed in public APIs?
          ├── Yes → UUID v7 (prevents enumeration)
          └── No → Auto-increment BIGINT (simplest, most efficient)
```

---

## Data Type Selection

Choosing the right data type prevents entire categories of bugs. The type system is your first line of defense — use it.

> *"Data outlives code."* — A schema mistake in data types will cost you a migration affecting every row, while a code mistake is a deploy away from being fixed.

### General Type Guidelines

| Data | Recommended Type | Avoid |
|---|---|---|
| Identifiers | `BIGINT` or `UUID` | `INT` (2B limit), `VARCHAR` IDs |
| Short strings | `VARCHAR(n)` with a sensible limit | Omitting the limit entirely |
| Long text | `TEXT` | `VARCHAR(10000)` — just use TEXT |
| Boolean | `BOOLEAN` | `SMALLINT`, `CHAR(1)`, `VARCHAR(5)` |
| Money | `NUMERIC(precision, scale)` | `FLOAT`, `DOUBLE PRECISION`, `MONEY` |
| Timestamps | `TIMESTAMPTZ` | `TIMESTAMP` (no timezone) |
| Dates | `DATE` | `VARCHAR(10)`, `TEXT` |
| IP addresses | `INET` | `VARCHAR(45)` |
| Email | `CITEXT` or `TEXT` + lower() index | `VARCHAR(255)` with no validation |
| JSON data | `JSONB` | `JSON` (no indexing), `TEXT` |
| Enumerations | Lookup table or `TEXT` + CHECK | `ENUM` type (hard to modify) |

### Common Type Mistakes

**Mistake 1: VARCHAR for everything**

```sql
-- Bad: stringly-typed schema
CREATE TABLE events (
    id         VARCHAR(255) PRIMARY KEY,    -- should be UUID or BIGINT
    event_date VARCHAR(255),                -- should be DATE
    is_active  VARCHAR(255),                -- should be BOOLEAN
    amount     VARCHAR(255),                -- should be NUMERIC
    ip_address VARCHAR(255)                 -- should be INET
);

-- Good: proper types enforce correctness
CREATE TABLE events (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    event_date DATE NOT NULL,
    is_active  BOOLEAN NOT NULL DEFAULT true,
    amount     NUMERIC(12,2) NOT NULL,
    ip_address INET
);
```

**Mistake 2: INT instead of BIGINT for primary keys**

An `INT` (4 bytes) maxes out at ~2.1 billion. This seems like a lot until you have a high-throughput events table. `BIGINT` (8 bytes) costs 4 extra bytes per row but eliminates a ticking time bomb.

**Mistake 3: FLOAT for money**

```sql
-- Floating-point arithmetic is fundamentally lossy for decimals
SELECT 0.1 + 0.2;  -- Returns 0.30000000000000004 in most languages

-- NUMERIC stores exact decimal values
SELECT 0.1::NUMERIC + 0.2::NUMERIC;  -- Returns 0.3 exactly
```

### Timestamps and Time Zones

Always use `TIMESTAMPTZ` (timestamp with time zone). A `TIMESTAMP` without time zone is ambiguous — it represents a wall-clock time with no indication of which wall clock.

```sql
-- Bad: timestamp without time zone
CREATE TABLE orders (
    id         BIGINT PRIMARY KEY,
    created_at TIMESTAMP DEFAULT now()  -- What timezone is this in?
);

-- Good: timestamptz stores UTC internally, converts on display
CREATE TABLE orders (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

PostgreSQL stores `TIMESTAMPTZ` as UTC internally and converts to the session's timezone on output. This means you always have an unambiguous point in time.

**Exception:** Use plain `DATE` or `TIMESTAMP` when you specifically mean "a date on a calendar" or "a time on a clock" without a timezone (e.g., a birthday, a recurring meeting time like "every Tuesday at 3pm").

### Numeric Precision for Money

Money requires exact decimal arithmetic. Use `NUMERIC(precision, scale)` where precision is total digits and scale is decimal places.

```sql
-- Common money column definitions
total_amount     NUMERIC(12,2)  -- up to 9,999,999,999.99
unit_price       NUMERIC(10,4)  -- for per-unit pricing with more precision
exchange_rate    NUMERIC(18,8)  -- for currency conversion rates
tax_rate         NUMERIC(5,4)   -- e.g., 0.0825 for 8.25%

-- Store the currency alongside the amount
CREATE TABLE transactions (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    amount        NUMERIC(12,2) NOT NULL,
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217

    CONSTRAINT chk_transactions_amount CHECK (amount >= 0),
    CONSTRAINT fk_transactions_currency
        FOREIGN KEY (currency_code) REFERENCES currencies(code)
);
```

**Alternative approach:** Store money as integer cents (or the smallest currency unit). This avoids decimal arithmetic entirely.

```sql
-- Amount in cents: $19.99 is stored as 1999
amount_cents BIGINT NOT NULL CHECK (amount_cents >= 0)
```

### JSON and JSONB Columns

JSONB is powerful for semi-structured data but should not replace proper relational modeling.

**Use JSONB when:**

- The structure genuinely varies across rows (user preferences, feature flags)
- You are storing third-party webhook payloads with unpredictable schemas
- You need a flexible "extras" or "metadata" column alongside typed columns

**Do NOT use JSONB when:**

- Every row has the same fields — those should be columns
- You need to join on values inside the JSON
- You need foreign key constraints on nested data
- You frequently filter or sort by a nested field (make it a column)

```sql
-- Good: metadata column for truly variable data
CREATE TABLE products (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name        TEXT NOT NULL,
    price       NUMERIC(10,2) NOT NULL,
    category_id BIGINT NOT NULL REFERENCES categories(id),
    metadata    JSONB DEFAULT '{}'::jsonb  -- color, weight, dimensions vary by category
);

-- Index specific JSONB paths you query frequently
CREATE INDEX idx_products_metadata_color
    ON products USING GIN ((metadata -> 'color'));
```

### Enum vs Lookup Table

PostgreSQL has a native `ENUM` type, but lookup tables are almost always the better choice.

```sql
-- Approach 1: ENUM type (problematic)
CREATE TYPE order_status AS ENUM ('pending', 'confirmed', 'shipped', 'delivered');

-- Adding a value requires ALTER TYPE ... ADD VALUE
-- Removing or renaming values is extremely difficult
-- Cannot store metadata (description, display_order, is_terminal)

-- Approach 2: Lookup table (recommended)
CREATE TABLE order_statuses (
    code         TEXT PRIMARY KEY,              -- 'pending', 'confirmed', etc.
    display_name TEXT NOT NULL,
    is_terminal  BOOLEAN NOT NULL DEFAULT false,
    sort_order   SMALLINT NOT NULL
);

CREATE TABLE orders (
    id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status TEXT NOT NULL DEFAULT 'pending',

    CONSTRAINT fk_orders_status FOREIGN KEY (status) REFERENCES order_statuses(code)
);
```

**Approach 3: TEXT + CHECK** — for small, stable sets of values that don't need metadata:

```sql
CREATE TABLE orders (
    id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status TEXT NOT NULL DEFAULT 'pending',

    CONSTRAINT chk_orders_status
        CHECK (status IN ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled'))
);
```

---

## Constraint Strategy

Constraints are executable documentation. They encode business rules directly into the schema so that no bug, no script, and no manual INSERT can violate them.

> *"Data integrity is not a feature — it is a requirement."* Every constraint you skip is a data corruption bug waiting to happen.

### NOT NULL by Default

Make columns NOT NULL unless you have an explicit reason for allowing NULL. NULL is a special value that means "unknown" or "not applicable" — it should be a deliberate choice, not a default.

```sql
-- Bad: nullable by default (the SQL standard default)
CREATE TABLE orders (
    id          BIGINT PRIMARY KEY,
    customer_id BIGINT,          -- Can this really be null?
    total       NUMERIC(12,2),   -- An order with no total?
    status      TEXT,            -- Unknown status?
    created_at  TIMESTAMPTZ      -- When was it created? ¯\_(ツ)_/¯
);

-- Good: NOT NULL by default, NULL only when justified
CREATE TABLE orders (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id),
    total       NUMERIC(12,2) NOT NULL,
    status      TEXT NOT NULL DEFAULT 'pending',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    shipped_at  TIMESTAMPTZ  -- NULL here means "not yet shipped" — intentional
);
```

**When NULL is appropriate:**

- Optional fields: `middle_name`, `phone_number`
- "Not yet" semantics: `shipped_at`, `completed_at`, `deleted_at`
- Truly unknown values: `birth_date` for legacy imported data

### CHECK Constraints for Domain Rules

CHECK constraints enforce rules that go beyond data types:

```sql
CREATE TABLE products (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name        TEXT NOT NULL,
    price       NUMERIC(10,2) NOT NULL,
    weight_kg   NUMERIC(8,3),
    sku         TEXT NOT NULL,

    CONSTRAINT chk_products_price CHECK (price > 0),
    CONSTRAINT chk_products_weight CHECK (weight_kg > 0),
    CONSTRAINT chk_products_sku CHECK (sku ~ '^[A-Z]{2,4}-[0-9]{4,8}$')
);

-- Multi-column check constraints
CREATE TABLE date_ranges (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    start_date DATE NOT NULL,
    end_date   DATE NOT NULL,

    CONSTRAINT chk_date_ranges_order CHECK (end_date >= start_date)
);

-- Status transition rules
CREATE TABLE shipments (
    id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status TEXT NOT NULL DEFAULT 'pending',
    weight NUMERIC(8,2) NOT NULL,

    CONSTRAINT chk_shipments_status
        CHECK (status IN ('pending', 'picked', 'shipped', 'delivered', 'returned')),
    CONSTRAINT chk_shipments_weight CHECK (weight > 0 AND weight < 50000)
);
```

### UNIQUE Constraints

UNIQUE constraints prevent duplicate values and also create an implicit index:

```sql
-- Single-column unique
CREATE TABLE users (
    id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email CITEXT NOT NULL,

    CONSTRAINT uq_users_email UNIQUE (email)
);

-- Multi-column unique (composite uniqueness)
CREATE TABLE team_members (
    team_id BIGINT NOT NULL REFERENCES teams(id),
    user_id BIGINT NOT NULL REFERENCES users(id),

    CONSTRAINT uq_team_members UNIQUE (team_id, user_id)
);

-- Partial unique: only enforce uniqueness for active records
CREATE UNIQUE INDEX uq_users_active_email
    ON users (email) WHERE is_active = true;
```

### Foreign Keys and Referential Integrity

Foreign keys are non-negotiable in relational databases. They prevent orphaned records and encode relationships in a machine-verifiable way.

```sql
CREATE TABLE order_items (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id   BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity   INT NOT NULL,

    CONSTRAINT fk_order_items_order
        FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    CONSTRAINT fk_order_items_product
        FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE RESTRICT,
    CONSTRAINT chk_order_items_quantity CHECK (quantity > 0)
);
```

**ON DELETE behavior guidelines:**

| Behavior | Use When | Example |
|---|---|---|
| `RESTRICT` (default) | Child data has independent value | Products referenced by order items |
| `CASCADE` | Children are meaningless without parent | Order items when an order is deleted |
| `SET NULL` | Relationship is optional | `assigned_to` when a user is deactivated |
| `SET DEFAULT` | A fallback value makes sense | Rarely used in practice |

### Deferrable Constraints

Deferrable constraints allow temporary violations within a transaction, checked at commit time. Useful for circular references or bulk operations:

```sql
-- Two tables that reference each other
CREATE TABLE departments (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name       TEXT NOT NULL,
    manager_id BIGINT,

    CONSTRAINT fk_departments_manager
        FOREIGN KEY (manager_id) REFERENCES employees(id)
        DEFERRABLE INITIALLY DEFERRED
);

CREATE TABLE employees (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name          TEXT NOT NULL,
    department_id BIGINT NOT NULL,

    CONSTRAINT fk_employees_department
        FOREIGN KEY (department_id) REFERENCES departments(id)
        DEFERRABLE INITIALLY DEFERRED
);

-- Now you can insert both in a single transaction
BEGIN;
    INSERT INTO departments (id, name, manager_id) OVERRIDING SYSTEM VALUE
        VALUES (1, 'Engineering', 100);
    INSERT INTO employees (id, name, department_id) OVERRIDING SYSTEM VALUE
        VALUES (100, 'Alice', 1);
COMMIT;  -- constraints checked here
```

### Database vs Application Validation

The database should be the last line of defense, not the only one. Use both layers:

```
  ┌──────────────────────────────────────────────────────────┐
  │  Application Layer                                        │
  │  ┌─────────────────────────────────────────────────────┐  │
  │  │ • Input validation (format, length, allowed values)  │  │
  │  │ • Business rules (can this user do this action?)     │  │
  │  │ • Better error messages for the end user             │  │
  │  │ • Fast feedback (no round trip to DB)                │  │
  │  └─────────────────────────────────────────────────────┘  │
  └──────────────────────────┬───────────────────────────────┘
                             │
                             ▼
  ┌──────────────────────────────────────────────────────────┐
  │  Database Layer                                           │
  │  ┌─────────────────────────────────────────────────────┐  │
  │  │ • NOT NULL, CHECK, UNIQUE, FOREIGN KEY               │  │
  │  │ • Data type enforcement                              │  │
  │  │ • Catches bugs in ALL clients (API, scripts, REPL)   │  │
  │  │ • Last line of defense — never trust the app alone   │  │
  │  └─────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────┘
```

**Rule of thumb:** If a constraint is about data integrity (types, nullability, relationships, value ranges), put it in the database. If it is about user experience (friendly error messages, cross-entity business rules), put it in the application. Always have both.

---

## Normalization Decisions

Normalization is covered in depth in [03-DATA-MODELING.md](./03-DATA-MODELING.md). This section focuses on the **decision-making principles** — when to normalize, when to denormalize, and how to evaluate the trade-offs.

### When to Normalize

Normalize when:

- **Data changes frequently** — normalized schemas avoid update anomalies
- **Correctness matters more than read speed** — financial systems, healthcare, legal
- **Storage is a concern** — normalization eliminates redundancy
- **You don't yet know your access patterns** — normalization preserves flexibility

> *"Normalize until it hurts, denormalize until it works."* — Common database engineering adage

### When to Denormalize

Denormalize when:

- **Read performance is critical** and joins are the bottleneck (measured, not assumed)
- **The data is effectively immutable** — historical records, audit logs, event stores
- **You need to support full-text search** — search engines work best with flat documents
- **Caching is impractical** — when materialized views or cache layers are not an option
- **You are building CQRS read models** — the read side is deliberately denormalized

### Practical Rules of Thumb

```
  ┌──────────────────────────────────────────────────────────────┐
  │                    Normalization Spectrum                      │
  │                                                                │
  │  Fully Normalized                           Fully Denormalized │
  │  ◄────────────────────────────────────────────────────────────►│
  │                                                                │
  │  ✓ No update anomalies         ✓ Fast reads (no joins)        │
  │  ✓ Minimal storage             ✓ Simple queries               │
  │  ✓ Maximum flexibility         ✓ Good for read replicas       │
  │  ✗ Many joins                  ✗ Update anomalies             │
  │  ✗ Complex queries             ✗ Wasted storage               │
  │  ✗ Slower reads                ✗ Hard to maintain consistency  │
  └──────────────────────────────────────────────────────────────┘
```

**Practical guidelines:**

| Situation | Recommendation |
|---|---|
| Write-heavy OLTP (orders, payments) | 3NF — correctness is paramount |
| Read-heavy analytics | Star schema — denormalize dimensions |
| User-facing dashboards | Materialized views or CQRS read models |
| Search / autocomplete | Flat denormalized documents in a search index |
| Audit / compliance | Fully normalized + append-only event log |
| High-throughput events | Slightly denormalized — embed commonly-read fields |

### Joins vs Anomalies Trade-Off

As DDIA Chapter 2 discusses, the cost of joins depends heavily on the database:

- **Relational databases** are optimized for joins — a well-indexed join is often cheaper than you think
- **Document databases** discourage joins — denormalization is the norm
- **Distributed SQL** makes joins across partitions expensive — co-locate related data

Before denormalizing to eliminate a join, measure the actual cost:

```sql
-- Measure join performance with EXPLAIN ANALYZE
EXPLAIN (ANALYZE, BUFFERS) SELECT
    o.id,
    o.total,
    c.name AS customer_name,
    c.email
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.created_at > now() - INTERVAL '7 days';

-- If the join is fast (indexed lookup, < 1ms per row), keep it normalized
-- If the join is slow (sequential scan, cross-partition), consider denormalization
```

---

## Index Alignment

A schema designed without considering indexes is a schema designed for full table scans. Think about indexes during design, not as an afterthought.

### Design Schemas with Indexing in Mind

Index alignment means choosing column types, orderings, and structures that work well with the database's indexing capabilities.

**Principle: The columns you filter, sort, and join on should be index-friendly.**

```sql
-- Good: status as TEXT with a small set of values — great for partial indexes
CREATE TABLE orders (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status     TEXT NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    total      NUMERIC(12,2) NOT NULL
);

-- Partial index: only index non-terminal statuses (the ones you actively query)
CREATE INDEX idx_orders_active
    ON orders (created_at DESC)
    WHERE status IN ('pending', 'confirmed', 'shipped');
```

**Column order in multi-column indexes matters:**

```sql
-- If you always filter by tenant_id first, then by created_at:
CREATE INDEX idx_orders_tenant_created
    ON orders (tenant_id, created_at DESC);

-- This index supports:
--   WHERE tenant_id = 5                            ✓ (uses first column)
--   WHERE tenant_id = 5 AND created_at > '2025-01' ✓ (uses both columns)
--   WHERE created_at > '2025-01'                    ✗ (skips first column)
```

### Covering Indexes

A covering index includes all columns needed by a query, eliminating the need to read the table itself (index-only scan):

```sql
-- Query: get order id and total for a customer
SELECT id, total FROM orders WHERE customer_id = 42;

-- Covering index: includes id and total
CREATE INDEX idx_orders_customer_covering
    ON orders (customer_id) INCLUDE (total);

-- PostgreSQL can satisfy this query entirely from the index
-- No table heap access needed = much faster
```

### Partial Indexes

Partial indexes only index rows that match a condition. They are smaller, faster, and use less storage:

```sql
-- Only 5% of orders are unshipped — index only those
CREATE INDEX idx_orders_unshipped
    ON orders (created_at DESC)
    WHERE status NOT IN ('delivered', 'cancelled');

-- Unique constraint only for active users
CREATE UNIQUE INDEX uq_users_active_email
    ON users (email) WHERE deleted_at IS NULL;

-- Index only recent data for time-series queries
CREATE INDEX idx_events_recent
    ON events (event_type, created_at DESC)
    WHERE created_at > '2025-01-01';
```

### Index-Friendly Column Types

Some data types index better than others:

| Type | B-tree | GIN | GiST | Hash | Notes |
|---|---|---|---|---|---|
| `BIGINT` | Excellent | — | — | Good | Best for PKs and FKs |
| `UUID` (v4) | Poor (random) | — | — | Good | Random scatter hurts B-tree |
| `UUID` (v7) | Good | — | — | Good | Time-ordered = B-tree friendly |
| `TEXT` | Good | Full-text | Trigram | Good | Use `CITEXT` for case-insensitive |
| `TIMESTAMPTZ` | Excellent | — | Range | — | Great for range queries |
| `JSONB` | — | Excellent | — | — | GIN for containment queries |
| `INET` | Good | — | Excellent | — | GiST for network containment |
| `BOOLEAN` | Poor (low cardinality) | — | — | — | Use partial indexes instead |

### Indexing Anti-Patterns

**Anti-pattern 1: Indexing every column**

```sql
-- Bad: indexes have write costs and storage overhead
CREATE INDEX idx_1 ON orders (id);          -- already the PK
CREATE INDEX idx_2 ON orders (customer_id);
CREATE INDEX idx_3 ON orders (status);
CREATE INDEX idx_4 ON orders (total);
CREATE INDEX idx_5 ON orders (created_at);
CREATE INDEX idx_6 ON orders (updated_at);

-- Good: index based on actual query patterns
CREATE INDEX idx_orders_customer ON orders (customer_id);
CREATE INDEX idx_orders_active ON orders (created_at DESC) WHERE status = 'pending';
```

**Anti-pattern 2: Functions that negate indexes**

```sql
-- Bad: wrapping indexed column in a function
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
-- The index on email is NOT used because of LOWER()

-- Good: use a functional index or CITEXT
CREATE INDEX idx_users_email_lower ON users (LOWER(email));
-- or
ALTER TABLE users ALTER COLUMN email TYPE CITEXT;
```

**Anti-pattern 3: Indexing low-cardinality columns alone**

```sql
-- Bad: boolean has only 2 values — a B-tree index provides no selectivity
CREATE INDEX idx_users_active ON users (is_active);

-- Good: use a partial index
CREATE INDEX idx_users_active ON users (created_at DESC) WHERE is_active = true;
```

---

## Temporal Data Design

Almost every table needs some notion of time. How you handle temporal data affects auditing, debugging, compliance, and data recovery.

### Created and Updated Timestamps

Every table that stores mutable data should have `created_at` and `updated_at` columns:

```sql
CREATE TABLE orders (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status     TEXT NOT NULL DEFAULT 'pending',
    total      NUMERIC(12,2) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Automatically update updated_at on any change
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_orders_updated_at
    BEFORE UPDATE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION set_updated_at();
```

**Principle:** `created_at` should be immutable — never update it. `updated_at` should change on every modification. Both should be `NOT NULL` and `TIMESTAMPTZ`.

### Soft Deletes vs Hard Deletes

Soft deletes mark a record as deleted without removing it from the table. Hard deletes remove the row permanently.

```
  Hard Delete                           Soft Delete
  ┌──────────────┐                      ┌──────────────────────────────┐
  │  DELETE FROM  │                      │  UPDATE orders               │
  │  orders      │                      │  SET deleted_at = now()      │
  │  WHERE id=5  │                      │  WHERE id = 5                │
  │              │                      │                              │
  │  Row is gone │                      │  Row remains, flagged        │
  └──────────────┘                      └──────────────────────────────┘
```

```sql
-- Soft delete column
CREATE TABLE orders (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status     TEXT NOT NULL,
    deleted_at TIMESTAMPTZ,  -- NULL = active, non-NULL = soft-deleted
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Default queries exclude soft-deleted records
CREATE VIEW active_orders AS
    SELECT * FROM orders WHERE deleted_at IS NULL;

-- Partial unique: prevent duplicates only among active records
CREATE UNIQUE INDEX uq_orders_reference_active
    ON orders (reference_number) WHERE deleted_at IS NULL;
```

**When to use each approach:**

| Factor | Hard Delete | Soft Delete |
|---|---|---|
| Regulatory compliance (GDPR) | Required for personal data | May violate right-to-erasure |
| Undo / recovery | Not possible | Easy |
| Query complexity | Simple | Must filter deleted_at everywhere |
| Storage | Reclaimed | Grows indefinitely |
| Referential integrity | Cascades cleanly | Orphan risk if not careful |
| Audit trail | Need separate audit log | Built-in (but limited) |

**Recommendation:** Use soft deletes sparingly. If you need undo or audit trails, consider a separate audit table or event sourcing pattern ([11-SCHEMA-DESIGN-PATTERNS.md](./11-SCHEMA-DESIGN-PATTERNS.md#event-sourcing-schema)). Soft deletes add complexity to every query.

### Audit Trails

For compliance-critical systems, maintain an audit trail that records who changed what and when:

```sql
CREATE TABLE audit_log (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name  TEXT NOT NULL,
    record_id   BIGINT NOT NULL,
    action      TEXT NOT NULL,  -- 'INSERT', 'UPDATE', 'DELETE'
    old_values  JSONB,
    new_values  JSONB,
    changed_by  TEXT NOT NULL,
    changed_at  TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_audit_log_action CHECK (action IN ('INSERT', 'UPDATE', 'DELETE'))
);

-- Generic trigger function for any audited table
CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, record_id, action, new_values, changed_by)
        VALUES (TG_TABLE_NAME, NEW.id, 'INSERT', to_jsonb(NEW),
                current_setting('app.current_user', true));
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, record_id, action, old_values, new_values, changed_by)
        VALUES (TG_TABLE_NAME, NEW.id, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW),
                current_setting('app.current_user', true));
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, record_id, action, old_values, changed_by)
        VALUES (TG_TABLE_NAME, OLD.id, 'DELETE', to_jsonb(OLD),
                current_setting('app.current_user', true));
    END IF;
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

-- Attach to any table
CREATE TRIGGER trg_orders_audit
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION audit_trigger();
```

### Effective Dating

Effective dating tracks when a value is valid — not just when it was stored. This is essential for prices, rates, configurations, and any data that changes on a schedule.

```sql
CREATE TABLE product_prices (
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_id   BIGINT NOT NULL REFERENCES products(id),
    price        NUMERIC(10,2) NOT NULL,
    effective_from DATE NOT NULL,
    effective_to   DATE,  -- NULL = currently effective

    CONSTRAINT chk_price_range CHECK (effective_to IS NULL OR effective_to > effective_from)
);

-- PostgreSQL range types make this even cleaner
CREATE TABLE product_prices (
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_id   BIGINT NOT NULL REFERENCES products(id),
    price        NUMERIC(10,2) NOT NULL,
    valid_during DATERANGE NOT NULL,

    -- Prevent overlapping price ranges for the same product
    CONSTRAINT excl_product_prices_no_overlap
        EXCLUDE USING gist (product_id WITH =, valid_during WITH &&)
);

-- Query: current price for a product
SELECT price
FROM product_prices
WHERE product_id = 42
  AND valid_during @> CURRENT_DATE;
```

---

## Schema Evolution Principles

Every schema will change. The question is not whether, but how gracefully. Design for evolution from day one.

> As DDIA Chapter 4 notes: *"In most cases, a change to an application's features also requires a change to data that it stores."* Schema evolution is an ongoing process, not a one-time event.

### Design for Change

**Leave room for growth:**

- Use `BIGINT` instead of `INT` for primary keys
- Use `TEXT` instead of `VARCHAR(n)` when the max length isn't meaningful
- Use `JSONB` metadata columns for truly variable attributes
- Add `created_at` and `updated_at` to every table from the start
- Avoid encoding business rules in column types (prefer CHECK constraints that can be altered)

**Avoid painting yourself into a corner:**

```sql
-- Bad: hard-coded enum type is painful to change
CREATE TYPE order_status AS ENUM ('new', 'paid', 'shipped');
-- Adding values requires ALTER TYPE, removing is nearly impossible

-- Good: CHECK constraint is easy to update
ALTER TABLE orders DROP CONSTRAINT chk_orders_status;
ALTER TABLE orders ADD CONSTRAINT chk_orders_status
    CHECK (status IN ('new', 'paid', 'shipped', 'refunded'));
```

### Additive-Only Migrations

The safest migrations only add — they never remove or rename in a single step.

**Safe operations (usually no downtime):**

- Adding a new nullable column
- Adding a new table
- Adding a new index (using `CONCURRENTLY`)
- Adding a new CHECK or UNIQUE constraint (using `NOT VALID` + `VALIDATE`)

**Dangerous operations (may require downtime or coordination):**

- Dropping a column or table
- Renaming a column or table
- Changing a column's data type
- Adding a NOT NULL constraint to an existing column with data

```sql
-- Safe: add a new nullable column
ALTER TABLE orders ADD COLUMN tracking_number TEXT;

-- Safe: add an index concurrently (no table lock)
CREATE INDEX CONCURRENTLY idx_orders_tracking ON orders (tracking_number);

-- Safe: add a constraint without validating existing rows
ALTER TABLE orders ADD CONSTRAINT chk_orders_tracking_format
    CHECK (tracking_number ~ '^[A-Z0-9]{10,30}$') NOT VALID;

-- Then validate separately (takes a lock but only briefly)
ALTER TABLE orders VALIDATE CONSTRAINT chk_orders_tracking_format;
```

### Backward Compatibility

Schema changes should maintain backward compatibility during the migration period. Old application code should continue to work while new code is being deployed.

**Rules for backward-compatible changes:**

1. New columns must be nullable or have a default value
2. Old columns must not be removed until all readers have been updated
3. Column renames must use a dual-write period
4. Type changes must be compatible (e.g., `INT` → `BIGINT` is safe, `BIGINT` → `INT` is not)

### The Expand-Contract Pattern

The expand-contract pattern (also called parallel change) is the standard approach for non-additive schema changes in production:

```
  Phase 1: EXPAND                    Phase 2: MIGRATE                  Phase 3: CONTRACT
  ┌──────────────────────┐           ┌──────────────────────┐          ┌──────────────────────┐
  │  Add new column      │           │  Backfill new column │          │  Drop old column     │
  │  Write to both       │    →      │  Switch reads to new │   →      │  Remove old writes   │
  │  Read from old       │           │  Keep writing both   │          │  Clean up triggers   │
  └──────────────────────┘           └──────────────────────┘          └──────────────────────┘
```

**Example: Renaming `name` to `full_name`**

```sql
-- Phase 1: EXPAND — add new column, write to both
ALTER TABLE customers ADD COLUMN full_name TEXT;

-- Trigger to keep columns in sync during transition
CREATE OR REPLACE FUNCTION sync_customer_name()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.full_name IS NULL THEN
        NEW.full_name = NEW.name;
    END IF;
    IF NEW.name IS NULL THEN
        NEW.name = NEW.full_name;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_customer_name
    BEFORE INSERT OR UPDATE ON customers
    FOR EACH ROW EXECUTE FUNCTION sync_customer_name();

-- Phase 2: MIGRATE — backfill and switch reads
UPDATE customers SET full_name = name WHERE full_name IS NULL;

-- Phase 3: CONTRACT — after all readers use full_name
ALTER TABLE customers DROP COLUMN name;
DROP TRIGGER trg_sync_customer_name ON customers;
DROP FUNCTION sync_customer_name();
```

---

## Multi-Database Principles

The principles in this document apply primarily to relational databases, but many translate to other data models. This section highlights where principles differ across database paradigms.

### SQL vs NoSQL Design Differences

```
  Relational (SQL)                      Document (NoSQL)
  ┌──────────────────────────┐          ┌──────────────────────────┐
  │ • Schema-on-write         │          │ • Schema-on-read          │
  │ • Normalize by default    │          │ • Denormalize by default  │
  │ • Joins are cheap         │          │ • Joins are expensive     │
  │ • Constraints in DB       │          │ • Constraints in app      │
  │ • ACID transactions       │          │ • Eventual consistency    │
  │ • Think in tables/rows    │          │ • Think in documents      │
  └──────────────────────────┘          └──────────────────────────┘
```

| Principle | SQL Approach | NoSQL Approach |
|---|---|---|
| **Naming** | snake_case tables and columns | camelCase fields (JSON convention) |
| **Primary keys** | Auto-increment or UUID column | Document _id (often string) |
| **Data types** | Strict typing via column types | Schema validation rules (optional) |
| **Constraints** | Database-enforced FK, CHECK, UNIQUE | Application-enforced |
| **Normalization** | 3NF for OLTP | Embed for single-query retrieval |
| **Indexing** | B-tree, GIN, GiST per column | Secondary indexes on fields |
| **Temporal data** | Trigger-based updated_at | Application-set timestamps |

### Document Embedding Principles

In document databases (MongoDB, Firestore, CouchDB), the key design decision is what to embed vs reference.

**Embed when:**

- The child data is always read with the parent
- The child data is bounded (e.g., a user's 3 addresses, not their 10,000 orders)
- The child data belongs to exactly one parent (one-to-few relationship)
- You need atomic updates of parent + children in a single write

**Reference when:**

- The child data is shared across multiple parents (many-to-many)
- The child data is unbounded (comments on a post, events in a stream)
- You need to query the child data independently
- The child data changes frequently independent of the parent

```
  Embed (one-to-few)                    Reference (one-to-many / many-to-many)
  ┌───────────────────────┐             ┌───────────────────┐
  │ { user: "alice",      │             │ { _id: "u1",      │
  │   addresses: [        │             │   name: "alice" }  │
  │     { street: "...",  │             └───────────────────┘
  │       city: "..." },  │                      │
  │     { street: "...",  │             ┌────────┴────────┐
  │       city: "..." }   │             │                 │
  │   ]                   │       ┌─────┴──────┐   ┌─────┴──────┐
  │ }                     │       │ { order_id: │   │ { order_id: │
  └───────────────────────┘       │   user: "u1"│   │   user: "u1"│
                                  │   items: ...│   │   items: ...│
  Single document read            │ }           │   │ }           │
  Atomic update                   └────────────┘   └────────────┘
                                  Separate collection / query
```

### Key-Value Key Design

In key-value stores (Redis, DynamoDB, Memcached), the key structure IS your schema. Design keys for both uniqueness and query patterns:

```
  Pattern: <entity>:<id>:<attribute>

  Examples:
    user:12345:profile          → { name: "Alice", email: "..." }
    user:12345:preferences      → { theme: "dark", lang: "en" }
    order:98765:summary         → { total: 199.99, status: "shipped" }
    session:abc123              → { user_id: 12345, expires: "..." }

  For sorted access (Redis sorted sets, DynamoDB sort keys):
    tenant:acme#order:2025-01-15#98765   → order data
    tenant:acme#order:2025-01-16#98766   → order data
```

**Key design principles:**

- Use a consistent delimiter (`:` for Redis, `#` for DynamoDB)
- Put the most-queried dimension first (tenant, user, entity type)
- Include sort-friendly values (timestamps, zero-padded numbers) for range queries
- Keep keys short but readable — keys consume memory too

### Wide-Column Partition Key Selection

In wide-column stores (Cassandra, ScyllaDB, HBase), the partition key determines data distribution and query efficiency. It is the single most important design decision.

**Good partition key:**

- High cardinality (many distinct values)
- Even distribution (no hot partitions)
- Matches your primary query pattern
- Bounded partition size (< 100MB per partition in Cassandra)

```
  Good: partition by user_id             Bad: partition by status
  ┌────────────┐ ┌────────────┐          ┌──────────────────────────┐
  │ user: 1001 │ │ user: 1002 │          │ status: "active"         │
  │ ┌────────┐ │ │ ┌────────┐ │          │  500,000 rows            │
  │ │ evt 1  │ │ │ │ evt 1  │ │          │  (hot partition!)        │
  │ │ evt 2  │ │ │ │ evt 2  │ │          └──────────────────────────┘
  │ │ evt 3  │ │ │ │ evt 3  │ │          ┌──────────────────────────┐
  │ └────────┘ │ │ └────────┘ │          │ status: "inactive"       │
  └────────────┘ └────────────┘          │  50 rows                 │
                                         └──────────────────────────┘
  Even distribution                      Massively skewed
```

---

## Principle Checklist

Use this checklist as a quick-reference before finalizing any schema design. Each item maps to a section in this document.

### Naming

- [ ] Tables use snake_case, plural nouns
- [ ] Columns use snake_case, descriptive names (no abbreviations)
- [ ] Foreign key columns follow `<entity_singular>_id` pattern
- [ ] Boolean columns use `is_` or `has_` prefix
- [ ] All indexes and constraints are explicitly named
- [ ] No SQL reserved words used as unquoted identifiers

### Primary Keys

- [ ] Every table has a primary key
- [ ] Surrogate keys used for entity tables (BIGINT or UUID v7)
- [ ] Composite keys used for junction tables
- [ ] Auto-increment BIGINT for internal-only IDs
- [ ] UUID v7 for distributed or API-exposed IDs

### Data Types

- [ ] TIMESTAMPTZ (not TIMESTAMP) for all points in time
- [ ] NUMERIC for money (not FLOAT or DOUBLE PRECISION)
- [ ] BIGINT for primary keys (not INT)
- [ ] BOOLEAN for true/false (not SMALLINT or CHAR)
- [ ] JSONB only for genuinely variable-structure data
- [ ] Lookup tables preferred over ENUM types

### Constraints

- [ ] NOT NULL on every column unless NULL has explicit meaning
- [ ] CHECK constraints for value ranges and formats
- [ ] UNIQUE constraints on natural keys and business identifiers
- [ ] Foreign keys on every reference (with appropriate ON DELETE)
- [ ] Critical constraints enforced at DB level, not just in app code

### Normalization

- [ ] Default to 3NF for OLTP workloads
- [ ] Denormalization decisions are measured, not assumed
- [ ] Denormalized data has a clear consistency strategy

### Indexes

- [ ] Indexes exist for every foreign key column
- [ ] Multi-column index order matches query filter order
- [ ] Partial indexes used for selective queries
- [ ] No redundant indexes (subset of another index)
- [ ] No indexes on very low-cardinality columns alone

### Temporal Data

- [ ] `created_at TIMESTAMPTZ NOT NULL DEFAULT now()` on every table
- [ ] `updated_at` with a trigger on mutable tables
- [ ] Soft delete strategy chosen deliberately (not by default)
- [ ] Audit trail in place for compliance-sensitive tables

### Schema Evolution

- [ ] New columns are nullable or have defaults
- [ ] No column renames in a single step — use expand-contract
- [ ] Indexes created with `CONCURRENTLY`
- [ ] Constraints added `NOT VALID` then validated separately

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial schema design principles documentation |
