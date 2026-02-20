# Data Modeling

## Table of Contents

1. [Overview](#overview)
   - [What Is Data Modeling?](#what-is-data-modeling)
   - [Why Data Modeling Matters](#why-data-modeling-matters)
   - [Conceptual, Logical, and Physical Models](#conceptual-logical-and-physical-models)
2. [Entity-Relationship Modeling](#entity-relationship-modeling)
   - [ER Diagrams](#er-diagrams)
   - [Cardinality](#cardinality)
   - [Primary Keys and Foreign Keys](#primary-keys-and-foreign-keys)
   - [Junction Tables](#junction-tables)
3. [Relational Schema Design](#relational-schema-design)
   - [Normalization Recap](#normalization-recap)
   - [Applied Normalization Example](#applied-normalization-example)
   - [When to Denormalize](#when-to-denormalize)
4. [Document Schema Design](#document-schema-design)
   - [Embedding vs Referencing](#embedding-vs-referencing)
   - [One-to-Many Patterns](#one-to-many-patterns)
   - [Many-to-Many in Document Databases](#many-to-many-in-document-databases)
   - [Schema Versioning](#schema-versioning)
5. [Star and Snowflake Schemas](#star-and-snowflake-schemas)
   - [Fact Tables](#fact-tables)
   - [Dimension Tables](#dimension-tables)
   - [Star vs Snowflake](#star-vs-snowflake)
6. [Polymorphic Data Patterns](#polymorphic-data-patterns)
   - [Single-Table Inheritance](#single-table-inheritance)
   - [Class-Table Inheritance](#class-table-inheritance)
   - [Concrete-Table Inheritance](#concrete-table-inheritance)
   - [Entity-Attribute-Value (EAV)](#entity-attribute-value-eav)
7. [Schema Evolution](#schema-evolution)
   - [Schema-on-Write vs Schema-on-Read](#schema-on-write-vs-schema-on-read)
   - [Backward and Forward Compatibility](#backward-and-forward-compatibility)
   - [Evolution Strategies in Practice](#evolution-strategies-in-practice)
8. [Common Modeling Patterns](#common-modeling-patterns)
   - [Soft Deletes](#soft-deletes)
   - [Audit Trails](#audit-trails)
   - [Temporal Tables](#temporal-tables)
   - [Tree and Hierarchy Patterns](#tree-and-hierarchy-patterns)
9. [Version History](#version-history)

---

## Overview

### What Is Data Modeling?

Data modeling is the process of defining the structure, relationships, and constraints of data within a system. It is the bridge between a real-world domain and the storage technology that will persist it. As *Designing Data-Intensive Applications* (DDIA) Chapter 2 emphasizes, the data model is arguably the most important part of developing software — it shapes not only how the software is written, but also how we *think* about the problem being solved.

> *"Data models are perhaps the most important part of developing software, because they have such a profound effect: not only on how the software is written, but also on how we think about the problem that we are solving."* — DDIA, Chapter 2

Every layer of a software system builds on the data model of the layer below it:

- **Application developers** model the real world in terms of objects, APIs, and data structures
- **Database engineers** express those structures as tables, documents, or graphs
- **Storage engineers** represent data as bytes on disk, in memory, or across a network

### Why Data Modeling Matters

A poor data model creates problems that compound over time:

| Problem | Symptom | Root Cause |
|---|---|---|
| Slow queries | Full table scans, timeouts | Missing relationships, poor normalization |
| Data anomalies | Conflicting values for the same entity | Update anomalies from denormalized data |
| Rigid schemas | Every feature requires a migration | Over-normalized or poorly abstracted models |
| Wasted storage | Duplicate data across tables | Unnecessary denormalization |
| Application bugs | Null pointer errors, broken joins | Ambiguous cardinality, missing constraints |

Getting the model right early saves orders of magnitude of effort later. Refactoring a data model in production — with live traffic, backfills, and zero-downtime migrations — is one of the hardest problems in software engineering.

### Conceptual, Logical, and Physical Models

Data modeling typically proceeds through three levels of abstraction:

```
    ┌──────────────────────────────────────────────────────┐
    │              Conceptual Model                        │
    │  "What entities exist and how do they relate?"       │
    │  Audience: Business stakeholders, domain experts     │
    │  Artifacts: ER diagrams, domain glossary             │
    └──────────────────────┬───────────────────────────────┘
                           │
                           ▼
    ┌──────────────────────────────────────────────────────┐
    │              Logical Model                           │
    │  "What are the attributes, keys, and constraints?"   │
    │  Audience: Application developers, data architects   │
    │  Artifacts: Table definitions, document schemas      │
    └──────────────────────┬───────────────────────────────┘
                           │
                           ▼
    ┌──────────────────────────────────────────────────────┐
    │              Physical Model                          │
    │  "How is this stored on disk? What indexes exist?"   │
    │  Audience: DBAs, storage engineers, SREs             │
    │  Artifacts: DDL scripts, index definitions, config   │
    └──────────────────────────────────────────────────────┘
```

| Level | Focus | Technology-Dependent? | Example |
|---|---|---|---|
| **Conceptual** | Entities and relationships | No | "A Customer places many Orders" |
| **Logical** | Attributes, keys, data types | Partially | `customers(id PK, name, email UNIQUE)` |
| **Physical** | Storage layout, indexes, partitions | Yes | `CREATE INDEX idx_orders_customer ON orders(customer_id)` |

In practice, these layers often blur — especially in agile teams — but understanding the distinction helps separate *what* you are modeling from *how* you are storing it.

---

## Entity-Relationship Modeling

### ER Diagrams

Entity-Relationship (ER) modeling is the most widely used technique for conceptual data modeling. An ER diagram captures **entities** (things), **attributes** (properties of things), and **relationships** (how things connect to each other).

Consider a simplified e-commerce domain:

```
    ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
    │   Customer   │         │    Order     │         │   Product    │
    ├──────────────┤         ├──────────────┤         ├──────────────┤
    │ PK id        │───┐     │ PK id        │     ┌───│ PK id        │
    │    name      │   │     │ FK customer_id│     │   │    name      │
    │    email     │   └────▶│    total      │     │   │    price     │
    │    created_at│         │    status     │     │   │    sku       │
    └──────────────┘         │    ordered_at │     │   └──────────────┘
                             └──────┬───────┘     │
                                    │             │
                                    ▼             │
                             ┌──────────────┐     │
                             │  Order_Item  │     │
                             ├──────────────┤     │
                             │ PK id        │     │
                             │ FK order_id  │     │
                             │ FK product_id│─────┘
                             │    quantity  │
                             │    unit_price│
                             └──────────────┘
```

Key conventions in this diagram:

- **PK** marks primary keys — the unique identifier for each row
- **FK** marks foreign keys — references to another entity's primary key
- Arrows show the direction of the relationship (the FK points to the PK it references)

### Cardinality

Cardinality defines how many instances of one entity can be associated with instances of another. There are three fundamental types:

**One-to-One (1:1)**

Each entity on both sides has at most one related entity on the other side. Common when splitting a table for security or performance reasons.

```
    ┌──────────┐    1:1    ┌─────────────────┐
    │   User   │──────────▶│  User_Profile   │
    │ PK id    │           │ PK user_id (FK) │
    │   email  │           │   bio           │
    │   pass   │           │   avatar_url    │
    └──────────┘           └─────────────────┘
```

**One-to-Many (1:N)**

The most common relationship. One entity owns many related entities.

```
    ┌──────────┐    1:N    ┌──────────────┐
    │ Customer │──────────▶│    Order     │
    │ PK id    │           │ PK id        │
    │   name   │           │ FK cust_id   │
    └──────────┘           │   total      │
                           └──────────────┘

    One customer can have many orders.
    Each order belongs to exactly one customer.
```

**Many-to-Many (M:N)**

Both sides can have multiple associations. In relational databases, this always requires a junction (join) table.

```
    ┌──────────┐    M:N    ┌──────────┐
    │ Student  │◀─────────▶│  Course  │
    │ PK id    │           │ PK id    │
    │   name   │           │   title  │
    └──────────┘           └──────────┘

    Resolved with a junction table:

    ┌──────────┐    1:N    ┌────────────────┐    N:1    ┌──────────┐
    │ Student  │──────────▶│  Enrollment    │◀──────────│  Course  │
    │ PK id    │           │ PK id          │           │ PK id    │
    └──────────┘           │ FK student_id  │           └──────────┘
                           │ FK course_id   │
                           │   enrolled_at  │
                           │   grade        │
                           └────────────────┘
```

### Primary Keys and Foreign Keys

**Primary keys** uniquely identify a row. Two common strategies:

| Strategy | Example | Pros | Cons |
|---|---|---|---|
| **Surrogate key** | `SERIAL`, `UUID` | Stable, no business meaning to change | Extra column, UUID can hurt index performance |
| **Natural key** | `email`, `isbn` | Meaningful, one fewer column | Can change, may be composite |

**Foreign keys** enforce referential integrity — the database guarantees that every referenced row exists.

```sql
CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    total       NUMERIC(10,2) NOT NULL,
    status      VARCHAR(20) DEFAULT 'pending',
    ordered_at  TIMESTAMPTZ DEFAULT now()
);
```

Foreign key constraints prevent orphaned records. If you delete a customer who has orders, the database will reject the operation (or cascade the delete, depending on configuration).

### Junction Tables

A junction table (also called an associative table or bridge table) resolves many-to-many relationships by introducing two foreign keys:

```sql
CREATE TABLE enrollment (
    id          SERIAL PRIMARY KEY,
    student_id  INTEGER NOT NULL REFERENCES students(id),
    course_id   INTEGER NOT NULL REFERENCES courses(id),
    enrolled_at TIMESTAMPTZ DEFAULT now(),
    grade       VARCHAR(2),
    UNIQUE (student_id, course_id)          -- prevent duplicate enrollments
);
```

The `UNIQUE` constraint on `(student_id, course_id)` ensures a student cannot enroll in the same course twice. Junction tables often carry their own attributes (here, `enrolled_at` and `grade`) — this is what distinguishes them from simple link tables.

---

## Relational Schema Design

### Normalization Recap

Normalization is the process of organizing data to reduce redundancy and prevent update anomalies. The normal forms build on each other:

| Normal Form | Rule | Eliminates |
|---|---|---|
| **1NF** | Every column holds atomic (indivisible) values; no repeating groups | Repeating groups, multi-valued columns |
| **2NF** | 1NF + every non-key column depends on the *entire* primary key | Partial dependencies (in composite keys) |
| **3NF** | 2NF + no non-key column depends on another non-key column | Transitive dependencies |
| **BCNF** | Every determinant is a candidate key | Remaining anomalies from 3NF edge cases |

Most production schemas aim for **3NF** as the default. Going beyond BCNF (to 4NF, 5NF) is rarely necessary outside academic contexts.

### Applied Normalization Example

Consider a denormalized orders table:

```sql
-- Unnormalized: violates 1NF (repeating groups), 2NF, and 3NF
CREATE TABLE orders_flat (
    order_id      INTEGER,
    customer_name VARCHAR(100),
    customer_email VARCHAR(255),
    product1_name VARCHAR(100),
    product1_qty  INTEGER,
    product1_price NUMERIC(10,2),
    product2_name VARCHAR(100),
    product2_qty  INTEGER,
    product2_price NUMERIC(10,2)
);
```

**Problems:**

- Adding a third product requires altering the table (repeating groups)
- Customer name and email are duplicated across orders (update anomaly)
- Product prices are embedded per order row (transitive dependency)

**After normalizing to 3NF:**

```sql
CREATE TABLE customers (
    id    SERIAL PRIMARY KEY,
    name  VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE
);

CREATE TABLE products (
    id    SERIAL PRIMARY KEY,
    name  VARCHAR(100) NOT NULL,
    price NUMERIC(10,2) NOT NULL
);

CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    ordered_at  TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE order_items (
    id         SERIAL PRIMARY KEY,
    order_id   INTEGER NOT NULL REFERENCES orders(id),
    product_id INTEGER NOT NULL REFERENCES products(id),
    quantity   INTEGER NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(10,2) NOT NULL        -- snapshot of price at purchase time
);
```

Note that `unit_price` on `order_items` is intentionally denormalized — it captures the price at the time of purchase, which may differ from the product's current price. This is a deliberate modeling decision, not a normalization violation.

### When to Denormalize

Normalization optimizes for **write correctness** (no anomalies), but production systems often need to optimize for **read performance**. Common denormalization strategies:

| Strategy | Description | Use Case |
|---|---|---|
| **Materialized columns** | Store a computed value alongside source data | `orders.total` instead of `SUM(order_items.unit_price * quantity)` |
| **Summary tables** | Pre-aggregated data refreshed periodically | Daily sales totals, monthly active users |
| **Materialized views** | Database-managed denormalized views | Dashboards, reporting queries |
| **Duplicate columns** | Copy a frequently-joined column into the child | `order_items.product_name` to avoid joining `products` |

**Rules of thumb for denormalization:**

1. Start normalized. Denormalize only when you have measured evidence of a performance problem
2. Denormalization trades write complexity for read speed — every denormalized column is a place where data can drift out of sync
3. Keep a single source of truth and derive denormalized copies from it (via triggers, application code, or materialized views)

> *"Denormalization is like debt: sometimes it is worth taking on, but you should be aware of the interest payments."*

---

## Document Schema Design

Document databases (MongoDB, CouchDB, Firestore) store data as nested JSON-like documents. As DDIA Chapter 2 explains, the document model works well when data has a natural tree structure — a document and all its children are loaded together in a single query.

### Embedding vs Referencing

The fundamental design decision in document databases is whether to **embed** related data inside a single document or **reference** it by ID.

| Approach | When to Use | Trade-offs |
|---|---|---|
| **Embed** | Data is read together, belongs to parent, rarely changes independently | Fast reads (single query), atomic writes; document size can grow |
| **Reference** | Data is shared across entities, changes independently, or is very large | Smaller documents, avoids duplication; requires multiple queries or `$lookup` |

**Embedded design (one-to-few):**

```json
{
  "_id": "order_001",
  "customer": {
    "name": "Alice",
    "email": "alice@example.com"
  },
  "items": [
    { "product": "Widget A", "quantity": 2, "unit_price": 9.99 },
    { "product": "Widget B", "quantity": 1, "unit_price": 24.99 }
  ],
  "total": 44.97,
  "status": "shipped"
}
```

**Referenced design (one-to-many, shared data):**

```json
// Order document
{
  "_id": "order_001",
  "customer_id": "cust_42",
  "item_ids": ["item_501", "item_502"],
  "total": 44.97,
  "status": "shipped"
}

// Separate item documents
{
  "_id": "item_501",
  "order_id": "order_001",
  "product_id": "prod_10",
  "quantity": 2,
  "unit_price": 9.99
}
```

### One-to-Many Patterns

DDIA Chapter 2 distinguishes three flavors of one-to-many relationships in document databases, each suited to a different cardinality:

| Relationship | Cardinality | Strategy | Example |
|---|---|---|---|
| **One-to-few** | ≤ a handful | Embed an array of objects in the parent | Order → shipping addresses (most people have 1–3) |
| **One-to-many** | Dozens to hundreds | Embed an array of IDs; fetch children separately | Blog post → comments |
| **One-to-squillions** | Thousands+ | Store parent ID on the child; never embed | Host → log entries |

The key insight: embedding is a spectrum. Embed when the child data is small, bounded, and always read with the parent. Reference when the child collection can grow unboundedly.

### Many-to-Many in Document Databases

Document databases have no native join mechanism (or only limited ones like MongoDB's `$lookup`). For many-to-many relationships, you have three options:

1. **Application-level joins** — Store arrays of IDs on one or both sides. The application fetches the referenced documents in a second query.

```json
// Student document
{ "_id": "stu_1", "name": "Alice", "course_ids": ["cs101", "math201"] }

// Course document
{ "_id": "cs101", "title": "Intro to CS", "student_ids": ["stu_1", "stu_2"] }
```

2. **Denormalize into one side** — Embed a summary of the related data. Accept that updates must propagate to all copies.

3. **Hybrid approach** — Use a relational database for the many-to-many relationship and a document database for the rest. This is increasingly common in polyglot persistence architectures.

As DDIA Chapter 2 notes, the more interconnected your data is, the more awkward it becomes to model in a document database — and the more a relational or graph model makes sense.

### Schema Versioning

Document databases are schema-flexible, but that does not mean "schema-less." Over time, document shapes evolve. Common versioning strategies:

```json
// Version field on every document
{
  "_id": "cust_42",
  "schema_version": 2,
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "address": {
    "street": "123 Main St",
    "city": "Portland",
    "state": "OR"
  }
}
```

**Migration approaches:**

| Approach | Description | Best For |
|---|---|---|
| **Eager migration** | Run a script to update all documents to the new shape | Small collections, infrequent changes |
| **Lazy migration** | Update documents to the new shape on read (read-then-write) | Large collections, gradual rollout |
| **Version dispatch** | Application code handles multiple schema versions simultaneously | Zero-downtime deployments |

Lazy migration is the most common pattern in production. The application reads a document, checks its `schema_version`, transforms it to the latest shape if needed, and writes it back. Over time, old versions are gradually eliminated.

---

## Star and Snowflake Schemas

Star and snowflake schemas are the standard modeling patterns for **OLAP** (Online Analytical Processing) workloads — data warehouses and analytical databases. DDIA Chapter 3 discusses these in the context of column-oriented storage and analytical query patterns.

### Fact Tables

A **fact table** records individual business events — each row represents something that happened (a sale, a click, a shipment). Fact tables tend to be very wide (many columns) and extremely tall (billions of rows).

```sql
CREATE TABLE fact_sales (
    sale_id       BIGSERIAL PRIMARY KEY,
    date_key      INTEGER NOT NULL REFERENCES dim_date(date_key),
    product_key   INTEGER NOT NULL REFERENCES dim_product(product_key),
    store_key     INTEGER NOT NULL REFERENCES dim_store(store_key),
    customer_key  INTEGER NOT NULL REFERENCES dim_customer(customer_key),
    quantity_sold INTEGER NOT NULL,
    unit_price    NUMERIC(10,2) NOT NULL,
    discount      NUMERIC(5,2) DEFAULT 0,
    total_amount  NUMERIC(12,2) NOT NULL
);
```

Fact tables contain two types of columns:

- **Foreign keys** to dimension tables (the "who, what, where, when" of the event)
- **Measures** — numeric values that can be aggregated (quantity, price, amount)

### Dimension Tables

**Dimension tables** describe the context of a fact — the *who*, *what*, *where*, and *when*. They are typically wide (many descriptive columns) but short (thousands to millions of rows).

```sql
CREATE TABLE dim_product (
    product_key   SERIAL PRIMARY KEY,
    product_id    VARCHAR(20) NOT NULL,        -- natural key from source
    name          VARCHAR(200) NOT NULL,
    category      VARCHAR(100),
    subcategory   VARCHAR(100),
    brand         VARCHAR(100),
    supplier      VARCHAR(200),
    unit_cost     NUMERIC(10,2)
);

CREATE TABLE dim_date (
    date_key      INTEGER PRIMARY KEY,          -- e.g., 20250115
    full_date     DATE NOT NULL,
    year          SMALLINT NOT NULL,
    quarter       SMALLINT NOT NULL,
    month         SMALLINT NOT NULL,
    day_of_week   VARCHAR(10) NOT NULL,
    is_weekend    BOOLEAN NOT NULL,
    is_holiday    BOOLEAN NOT NULL
);
```

A typical analytical query joins the fact table to multiple dimensions:

```sql
-- Total sales by product category and quarter
SELECT
    d.year,
    d.quarter,
    p.category,
    SUM(f.total_amount) AS total_revenue,
    SUM(f.quantity_sold) AS units_sold
FROM fact_sales f
JOIN dim_date d    ON f.date_key    = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
GROUP BY d.year, d.quarter, p.category
ORDER BY d.year, d.quarter, total_revenue DESC;
```

### Star vs Snowflake

```
    Star Schema                         Snowflake Schema
    ───────────                         ────────────────

    ┌───────────┐                       ┌───────────┐
    │ dim_date  │                       │ dim_date  │
    └─────┬─────┘                       └─────┬─────┘
          │                                   │
    ┌─────┴─────┐  ┌─────────────┐      ┌─────┴─────┐  ┌─────────────┐
    │           │──│ dim_product │      │           │──│ dim_product │
    │fact_sales │  └─────────────┘      │fact_sales │  └──────┬──────┘
    │           │──┐                    │           │──┐      │
    └─────┬─────┘  │                    └─────┬─────┘  │ ┌────┴────────┐
          │        │                          │        │ │dim_category │
    ┌─────┴─────┐  └─────────────┐      ┌─────┴─────┐ │ └─────────────┘
    │dim_custmr │  │  dim_store  │      │dim_custmr │ └─────────────┐
    └───────────┘  └─────────────┘      └───────────┘  │  dim_store  │
                                                       └──────┬──────┘
                                                              │
                                                       ┌──────┴──────┐
                                                       │ dim_region  │
                                                       └─────────────┘
```

| Property | Star Schema | Snowflake Schema |
|---|---|---|
| **Dimension structure** | Flat (denormalized) | Normalized into sub-dimensions |
| **Query simplicity** | Fewer joins, simpler SQL | More joins, more complex SQL |
| **Storage efficiency** | More redundancy in dimensions | Less redundancy |
| **Query performance** | Generally faster (fewer joins) | May be slower for complex queries |
| **Maintenance** | Easier to understand and build | Harder to maintain, more tables |
| **When to use** | Most data warehouses (default choice) | When dimension tables are very large |

As DDIA Chapter 3 notes, star schemas are the dominant pattern in data warehousing. The simplicity of a single join from fact to dimension outweighs the minor storage overhead of denormalized dimensions in nearly all practical cases.

---

## Polymorphic Data Patterns

Polymorphism arises when multiple entity types share a common interface but have different attributes. For example: `Payment` might be a credit card, bank transfer, or digital wallet — each with different fields. There are four classic approaches.

### Single-Table Inheritance

Store all types in one table. Use a discriminator column to distinguish types. Unused columns are `NULL` for types that do not need them.

```sql
CREATE TABLE payments (
    id             SERIAL PRIMARY KEY,
    order_id       INTEGER NOT NULL REFERENCES orders(id),
    type           VARCHAR(20) NOT NULL,       -- 'credit_card', 'bank_transfer', 'wallet'
    amount         NUMERIC(10,2) NOT NULL,
    -- credit card fields
    card_last_four VARCHAR(4),
    card_brand     VARCHAR(20),
    -- bank transfer fields
    bank_name      VARCHAR(100),
    account_number VARCHAR(30),
    routing_number VARCHAR(20),
    -- wallet fields
    wallet_provider VARCHAR(50),
    wallet_email    VARCHAR(255),
    created_at     TIMESTAMPTZ DEFAULT now()
);
```

| Pros | Cons |
|---|---|
| Simple queries — no joins needed | Many `NULL` columns; sparse rows |
| Easy to add new types (just add columns) | Cannot enforce NOT NULL per type at DB level |
| Good query performance (single table scan) | Table becomes wide as types diverge |

**Best for:** Types that share most attributes, with few type-specific columns.

### Class-Table Inheritance

One shared table for common attributes, plus one table per subtype for type-specific attributes.

```sql
CREATE TABLE payments (
    id         SERIAL PRIMARY KEY,
    order_id   INTEGER NOT NULL REFERENCES orders(id),
    type       VARCHAR(20) NOT NULL,
    amount     NUMERIC(10,2) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE credit_card_payments (
    payment_id     INTEGER PRIMARY KEY REFERENCES payments(id),
    card_last_four VARCHAR(4) NOT NULL,
    card_brand     VARCHAR(20) NOT NULL
);

CREATE TABLE bank_transfer_payments (
    payment_id     INTEGER PRIMARY KEY REFERENCES payments(id),
    bank_name      VARCHAR(100) NOT NULL,
    account_number VARCHAR(30) NOT NULL,
    routing_number VARCHAR(20) NOT NULL
);
```

| Pros | Cons |
|---|---|
| No `NULL` columns; clean schema | Requires a JOIN to read full payment |
| Can enforce NOT NULL per subtype | Inserting requires writing to two tables |
| Easy to add new types (add a new table) | Queries across all types need UNION or multiple joins |

**Best for:** Types with significantly different attributes but common querying needs.

### Concrete-Table Inheritance

Each type gets its own independent table. No shared base table.

```sql
CREATE TABLE credit_card_payments (
    id             SERIAL PRIMARY KEY,
    order_id       INTEGER NOT NULL REFERENCES orders(id),
    amount         NUMERIC(10,2) NOT NULL,
    card_last_four VARCHAR(4) NOT NULL,
    card_brand     VARCHAR(20) NOT NULL,
    created_at     TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE bank_transfer_payments (
    id             SERIAL PRIMARY KEY,
    order_id       INTEGER NOT NULL REFERENCES orders(id),
    amount         NUMERIC(10,2) NOT NULL,
    bank_name      VARCHAR(100) NOT NULL,
    account_number VARCHAR(30) NOT NULL,
    routing_number VARCHAR(20) NOT NULL,
    created_at     TIMESTAMPTZ DEFAULT now()
);
```

| Pros | Cons |
|---|---|
| Cleanest per-type schema | Querying "all payments" requires UNION ALL across tables |
| No NULLs, no joins | Shared columns are duplicated in every table |
| Best per-type query performance | Adding a common column means altering every table |

**Best for:** Types that are rarely queried together and have very different attributes.

### Entity-Attribute-Value (EAV)

The EAV pattern stores attributes as rows rather than columns. It is maximally flexible but sacrifices almost every benefit of a relational schema.

```sql
CREATE TABLE entity_attributes (
    id          SERIAL PRIMARY KEY,
    entity_type VARCHAR(50) NOT NULL,         -- 'product', 'user', etc.
    entity_id   INTEGER NOT NULL,
    attribute   VARCHAR(100) NOT NULL,         -- 'color', 'weight', 'size'
    value       TEXT NOT NULL,
    UNIQUE (entity_type, entity_id, attribute)
);

-- "Product 42 has color=red, weight=2.5kg"
INSERT INTO entity_attributes (entity_type, entity_id, attribute, value) VALUES
    ('product', 42, 'color', 'red'),
    ('product', 42, 'weight', '2.5kg');
```

| Pros | Cons |
|---|---|
| Infinitely flexible — any attribute on any entity | No type safety (everything is TEXT) |
| No schema changes to add attributes | Queries are complex (pivot/unpivot) |
| Good for user-defined fields | Cannot use indexes effectively |
| | Impossible to enforce constraints |

**Use EAV only as a last resort** — when attributes are truly user-defined and unknowable at design time (e.g., a form builder or CMS). For most cases, JSONB columns in PostgreSQL provide similar flexibility with much better querying:

```sql
-- Prefer JSONB over EAV in modern PostgreSQL
CREATE TABLE products (
    id         SERIAL PRIMARY KEY,
    name       VARCHAR(200) NOT NULL,
    attributes JSONB DEFAULT '{}'
);

-- Query JSONB attributes with indexes
CREATE INDEX idx_products_attrs ON products USING GIN (attributes);
SELECT * FROM products WHERE attributes @> '{"color": "red"}';
```

---

## Schema Evolution

All schemas change over time. The question is not *whether* your schema will evolve, but *how gracefully* it can do so. DDIA Chapter 4 covers this topic in depth in the context of encoding and data flow.

### Schema-on-Write vs Schema-on-Read

These two philosophies represent fundamentally different approaches to schema enforcement:

| Property | Schema-on-Write | Schema-on-Read |
|---|---|---|
| **Enforcement** | Database rejects invalid data at write time | Application interprets data at read time |
| **Examples** | Relational databases (PostgreSQL, MySQL) | Document databases (MongoDB), data lakes |
| **Analogy** | Static typing (Java, Go) | Dynamic typing (Python, JavaScript) |
| **Migration** | `ALTER TABLE` — must transform existing data | New code handles multiple document shapes |
| **Validation** | Centralized in the database | Distributed across application code |
| **Safety** | Stronger guarantees; harder to corrupt data | More flexible; easier to corrupt data silently |

As DDIA Chapter 4 observes, schema-on-read is advantageous when:

- Items in a collection do not all have the same structure (heterogeneous data)
- The structure is determined by external systems you do not control
- You want to defer schema decisions until you better understand the data

Schema-on-write is better when:

- All items share a known, stable structure
- Data integrity is critical (financial systems, healthcare)
- Multiple applications write to the same database

### Backward and Forward Compatibility

When evolving schemas, two kinds of compatibility matter — especially in systems where old and new code coexist during deployments:

```
    ┌─────────────┐                    ┌─────────────┐
    │  Old Code   │───writes data────▶│  Database   │
    │  (v1)       │◀───reads data─────│             │
    └─────────────┘                    │  Contains   │
                                       │  v1 and v2  │
    ┌─────────────┐                    │  records    │
    │  New Code   │───writes data────▶│             │
    │  (v2)       │◀───reads data─────│             │
    └─────────────┘                    └─────────────┘
```

| Compatibility | Definition | Why It Matters |
|---|---|---|
| **Backward** | New code can read data written by old code | Rolling deployments — new servers must read old data |
| **Forward** | Old code can read data written by new code | Rollbacks — old servers must handle data from new servers |

**Practical rules for maintaining compatibility:**

1. **Adding a column** — backward compatible if the new column has a DEFAULT or allows NULL. Forward compatible if old code ignores unknown columns.
2. **Removing a column** — breaking change for code that reads it. Requires coordinated deployment.
3. **Renaming a column** — equivalent to removing + adding. Avoid in hot paths.
4. **Changing a column type** — risky. Use a new column and migrate data gradually.

### Evolution Strategies in Practice

```sql
-- Strategy 1: Add column with default (backward compatible)
ALTER TABLE users ADD COLUMN phone VARCHAR(20) DEFAULT NULL;

-- Strategy 2: Expand-and-contract (safe rename)
-- Step 1: Add new column
ALTER TABLE users ADD COLUMN full_name VARCHAR(200);
-- Step 2: Backfill
UPDATE users SET full_name = name WHERE full_name IS NULL;
-- Step 3: Deploy code that writes to both columns
-- Step 4: Deploy code that reads only from new column
-- Step 5: Drop old column (after all readers migrated)
ALTER TABLE users DROP COLUMN name;

-- Strategy 3: Create new table, migrate gradually
CREATE TABLE users_v2 (
    id        SERIAL PRIMARY KEY,
    full_name VARCHAR(200) NOT NULL,
    email     VARCHAR(255) NOT NULL UNIQUE,
    phone     VARCHAR(20),
    created_at TIMESTAMPTZ DEFAULT now()
);
```

The expand-and-contract pattern (also called parallel change) is the safest approach for zero-downtime schema evolution. It is covered in more detail in [06-MIGRATIONS.md](06-MIGRATIONS.md).

---

## Common Modeling Patterns

### Soft Deletes

Instead of physically removing rows, mark them as deleted. This preserves data for auditing, undo, and referential integrity.

```sql
CREATE TABLE customers (
    id         SERIAL PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    email      VARCHAR(255) NOT NULL UNIQUE,
    deleted_at TIMESTAMPTZ DEFAULT NULL        -- NULL = active, non-NULL = soft-deleted
);

-- "Delete" a customer
UPDATE customers SET deleted_at = now() WHERE id = 42;

-- Application queries filter out deleted records
SELECT * FROM customers WHERE deleted_at IS NULL;

-- Create a view for convenience
CREATE VIEW active_customers AS
    SELECT * FROM customers WHERE deleted_at IS NULL;
```

**Important considerations:**

- Unique constraints must account for soft deletes — a deleted email should not block a new signup. Use a partial unique index:

```sql
CREATE UNIQUE INDEX idx_customers_email_active
    ON customers (email) WHERE deleted_at IS NULL;
```

- Soft deletes cause tables to grow indefinitely. Implement a reaping job that hard-deletes records past a retention period.

### Audit Trails

Track who changed what and when. Two common patterns:

**Pattern 1: Audit columns on the main table**

```sql
CREATE TABLE products (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    price       NUMERIC(10,2) NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT now(),
    created_by  INTEGER REFERENCES users(id),
    updated_at  TIMESTAMPTZ DEFAULT now(),
    updated_by  INTEGER REFERENCES users(id)
);
```

**Pattern 2: Separate audit log table (full history)**

```sql
CREATE TABLE audit_log (
    id          BIGSERIAL PRIMARY KEY,
    table_name  VARCHAR(100) NOT NULL,
    record_id   INTEGER NOT NULL,
    action      VARCHAR(10) NOT NULL,            -- 'INSERT', 'UPDATE', 'DELETE'
    old_values  JSONB,
    new_values  JSONB,
    changed_by  INTEGER REFERENCES users(id),
    changed_at  TIMESTAMPTZ DEFAULT now()
);
```

Pattern 2 is often implemented via database triggers:

```sql
CREATE OR REPLACE FUNCTION audit_trigger_fn()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (table_name, record_id, action, old_values, new_values, changed_at)
    VALUES (
        TG_TABLE_NAME,
        COALESCE(NEW.id, OLD.id),
        TG_OP,
        CASE WHEN TG_OP IN ('UPDATE', 'DELETE') THEN to_jsonb(OLD) END,
        CASE WHEN TG_OP IN ('INSERT', 'UPDATE') THEN to_jsonb(NEW) END,
        now()
    );
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_products
    AFTER INSERT OR UPDATE OR DELETE ON products
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_fn();
```

### Temporal Tables

Temporal tables track the validity period of each row, enabling "as-of" queries — *"What did this record look like on January 1st?"*

```sql
CREATE TABLE product_prices (
    id          SERIAL PRIMARY KEY,
    product_id  INTEGER NOT NULL REFERENCES products(id),
    price       NUMERIC(10,2) NOT NULL,
    valid_from  TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to    TIMESTAMPTZ DEFAULT 'infinity'    -- open-ended = currently active
);

-- Current price of a product
SELECT price
FROM product_prices
WHERE product_id = 42
  AND now() BETWEEN valid_from AND valid_to;

-- Price as of a specific date
SELECT price
FROM product_prices
WHERE product_id = 42
  AND '2025-01-01'::timestamptz BETWEEN valid_from AND valid_to;
```

When updating a price, close the old row and insert a new one:

```sql
-- Close the current price
UPDATE product_prices
SET valid_to = now()
WHERE product_id = 42 AND valid_to = 'infinity';

-- Insert new price
INSERT INTO product_prices (product_id, price, valid_from)
VALUES (42, 29.99, now());
```

Some databases support temporal tables natively — SQL Server has `SYSTEM_TIME` temporal tables, and PostgreSQL has extensions like `temporal_tables`.

### Tree and Hierarchy Patterns

Hierarchical data (org charts, file systems, category trees, threaded comments) is one of the hardest things to model in relational databases. Four patterns exist, each with different trade-offs.

**1. Adjacency List**

The simplest pattern. Each row stores a reference to its parent.

```sql
CREATE TABLE categories (
    id        SERIAL PRIMARY KEY,
    name      VARCHAR(100) NOT NULL,
    parent_id INTEGER REFERENCES categories(id)
);

INSERT INTO categories (id, name, parent_id) VALUES
    (1, 'Electronics', NULL),
    (2, 'Computers', 1),
    (3, 'Laptops', 2),
    (4, 'Desktops', 2),
    (5, 'Phones', 1);
```

```
    Electronics (1)
    ├── Computers (2)
    │   ├── Laptops (3)
    │   └── Desktops (4)
    └── Phones (5)
```

| Pros | Cons |
|---|---|
| Simple schema, easy inserts/moves | Querying subtrees requires recursive CTEs |
| Intuitive to understand | Deep trees = many recursive joins |

```sql
-- Get full subtree using recursive CTE (PostgreSQL, SQL Server, MySQL 8+)
WITH RECURSIVE category_tree AS (
    SELECT id, name, parent_id, 0 AS depth
    FROM categories WHERE id = 1               -- start at root
    UNION ALL
    SELECT c.id, c.name, c.parent_id, ct.depth + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY depth, name;
```

**2. Nested Set**

Each node stores left and right boundary numbers. All descendants of a node have boundaries within the parent's range.

```sql
CREATE TABLE categories (
    id    SERIAL PRIMARY KEY,
    name  VARCHAR(100) NOT NULL,
    lft   INTEGER NOT NULL,               -- left boundary
    rgt   INTEGER NOT NULL                -- right boundary
);

INSERT INTO categories (id, name, lft, rgt) VALUES
    (1, 'Electronics',  1, 10),
    (2, 'Computers',    2,  7),
    (3, 'Laptops',      3,  4),
    (4, 'Desktops',     5,  6),
    (5, 'Phones',       8,  9);
```

```
    1──────────────────────────────10
    │ Electronics                   │
    │  2──────────7    8────────9   │
    │  │ Computers│    │ Phones │   │
    │  │ 3──4 5──6│    │        │   │
    │  │ Lap  Des │    │        │   │
    │  └──────────┘    └────────┘   │
    └───────────────────────────────┘
```

```sql
-- Get all descendants of "Computers" (lft=2, rgt=7)
SELECT * FROM categories WHERE lft > 2 AND rgt < 7;

-- Get all ancestors of "Laptops" (lft=3, rgt=4)
SELECT * FROM categories WHERE lft < 3 AND rgt > 4;
```

| Pros | Cons |
|---|---|
| Fast subtree and ancestor queries (no recursion) | Inserts and moves require renumbering many rows |
| Single query for any depth | Complex to maintain |

**3. Materialized Path**

Store the full path from root to each node as a string.

```sql
CREATE TABLE categories (
    id    SERIAL PRIMARY KEY,
    name  VARCHAR(100) NOT NULL,
    path  VARCHAR(500) NOT NULL            -- e.g., '/1/2/3'
);

INSERT INTO categories (id, name, path) VALUES
    (1, 'Electronics', '/1'),
    (2, 'Computers',   '/1/2'),
    (3, 'Laptops',     '/1/2/3'),
    (4, 'Desktops',    '/1/2/4'),
    (5, 'Phones',      '/1/5');
```

```sql
-- All descendants of "Computers"
SELECT * FROM categories WHERE path LIKE '/1/2/%';

-- All ancestors of "Laptops" (path = '/1/2/3')
SELECT * FROM categories WHERE '/1/2/3' LIKE path || '%';

-- Depth of any node
SELECT name, (LENGTH(path) - LENGTH(REPLACE(path, '/', ''))) AS depth
FROM categories;
```

| Pros | Cons |
|---|---|
| Simple queries with LIKE | Path length limited by column size |
| Easy to read and debug | Moving a subtree requires updating all descendant paths |
| No recursion needed | Not referentially enforced by the database |

PostgreSQL's `ltree` extension provides native support for materialized paths with specialized operators and indexes.

**4. Closure Table**

Store every ancestor-descendant pair in a separate table. This is the most flexible pattern at the cost of extra storage.

```sql
CREATE TABLE categories (
    id   SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE category_tree (
    ancestor_id   INTEGER NOT NULL REFERENCES categories(id),
    descendant_id INTEGER NOT NULL REFERENCES categories(id),
    depth         INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (ancestor_id, descendant_id)
);

-- Every node is its own ancestor at depth 0
INSERT INTO category_tree (ancestor_id, descendant_id, depth) VALUES
    (1, 1, 0), (1, 2, 1), (1, 3, 2), (1, 4, 2), (1, 5, 1),
    (2, 2, 0), (2, 3, 1), (2, 4, 1),
    (3, 3, 0),
    (4, 4, 0),
    (5, 5, 0);
```

```sql
-- All descendants of "Electronics" (id=1)
SELECT c.* FROM categories c
JOIN category_tree ct ON c.id = ct.descendant_id
WHERE ct.ancestor_id = 1 AND ct.depth > 0;

-- All ancestors of "Laptops" (id=3)
SELECT c.* FROM categories c
JOIN category_tree ct ON c.id = ct.ancestor_id
WHERE ct.descendant_id = 3 AND ct.depth > 0;

-- Direct children only
SELECT c.* FROM categories c
JOIN category_tree ct ON c.id = ct.descendant_id
WHERE ct.ancestor_id = 1 AND ct.depth = 1;
```

| Pros | Cons |
|---|---|
| Fast queries for any relationship type | Extra table with O(n²) rows in worst case |
| Easy inserts and moves (modify closure rows) | More complex insert/delete logic |
| Supports multiple roots and DAGs | Storage overhead for large trees |

**Choosing a tree pattern:**

| Pattern | Read Speed | Write Speed | Complexity | Best For |
|---|---|---|---|---|
| **Adjacency List** | Moderate (recursive CTE) | Fast | Low | Shallow trees, frequent moves |
| **Nested Set** | Fast (range scan) | Slow (renumber) | High | Read-heavy, static trees |
| **Materialized Path** | Fast (LIKE/ltree) | Moderate | Low | Breadcrumbs, URL paths |
| **Closure Table** | Fast (join) | Moderate | Medium | Deep trees, complex queries |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial data modeling documentation |
