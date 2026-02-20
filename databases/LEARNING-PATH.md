# Database Learning Path

A structured, self-paced training guide to mastering databases — from foundational concepts and relational modeling to NoSQL trade-offs, query optimization, scaling strategies, and production-ready operations. Each phase builds on the previous one, progressing from core theory to hands-on production patterns.

> **Time Estimate:** 10–12 weeks at ~5 hours/week. Adjust pace to your experience level. Engineers with prior SQL experience may complete Phases 1–2 in half the time.

---

## How to Use This Guide

1. **Follow the phases in order** — each one builds directly on prior knowledge; jumping ahead creates gaps that slow you down later
2. **Read the linked documents** — they contain the detailed content, code examples, and reference tables
3. **Complete the exercises** — hands-on practice is how database concepts become intuition; do not skip them
4. **Check yourself** — use the knowledge check items before moving to the next phase; if you cannot answer them confidently, re-read the relevant sections
5. **Build the capstone project** — the final project ties together all six phases and produces a portfolio artifact you can reference in engineering conversations
6. **Reference DDIA** — *Designing Data-Intensive Applications* by Martin Kleppmann is cited throughout; chapters are noted where relevant

---

## Phase 1: Foundations (Week 1–2)

### Learning Objectives

- Understand what a database is and why purpose-built data storage exists beyond flat files
- Learn the difference between ACID and BASE consistency models and when each applies
- Grasp the CAP theorem and its practical implications for distributed database selection
- Survey major database categories: relational, document, key-value, column-family, graph
- Understand storage engine fundamentals: B-trees vs LSM-trees, write-ahead logs, and how data reaches disk
- Navigate the modern database landscape and make informed technology choices

> **DDIA Reference:** Chapters 1–3 (Reliable, Scalable, and Maintainable Applications; Data Models and Query Languages; Storage and Retrieval)

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | What is a database, database categories, ACID vs BASE, CAP theorem, storage engines |

### Exercises

**1. Local PostgreSQL Setup with Docker:**

Run PostgreSQL locally using Docker and connect with `psql`:

```bash
# Pull and start PostgreSQL
docker run -d \
  --name learn-postgres \
  -e POSTGRES_USER=learner \
  -e POSTGRES_PASSWORD=learner123 \
  -e POSTGRES_DB=learn_db \
  -p 5432:5432 \
  postgres:16

# Verify the container is running
docker ps

# Connect with psql
docker exec -it learn-postgres psql -U learner -d learn_db
```

Once connected, explore the environment:

```sql
-- Check PostgreSQL version
SELECT version();

-- List all databases
\l

-- Show current connection info
\conninfo

-- List all schemas
\dn

-- Show all system tables
\dt pg_catalog.*
```

**2. Create a Sample Schema and Explore Storage Concepts:**

Create a small schema to understand how PostgreSQL organizes data:

```sql
-- Create a schema for our learning exercises
CREATE SCHEMA IF NOT EXISTS bookstore;
SET search_path TO bookstore;

-- Authors table
CREATE TABLE authors (
    author_id   SERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    birth_year  INTEGER,
    country     VARCHAR(100),
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Books table with foreign key
CREATE TABLE books (
    book_id     SERIAL PRIMARY KEY,
    title       VARCHAR(500) NOT NULL,
    author_id   INTEGER NOT NULL REFERENCES authors(author_id),
    isbn        VARCHAR(13) UNIQUE,
    price       NUMERIC(10, 2),
    published   DATE,
    page_count  INTEGER,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Insert sample data
INSERT INTO authors (name, birth_year, country) VALUES
    ('Martin Kleppmann', 1984, 'United Kingdom'),
    ('Alex Petrov', NULL, 'United States'),
    ('Markus Winand', NULL, 'Austria');

INSERT INTO books (title, author_id, isbn, price, published, page_count) VALUES
    ('Designing Data-Intensive Applications', 1, '9781449373320', 44.99, '2017-03-16', 616),
    ('Database Internals', 2, '9781492040347', 49.99, '2019-10-01', 350),
    ('SQL Performance Explained', 3, '9783950307825', 34.95, '2012-09-01', 204);

-- Explore what was created
\dt bookstore.*
\d bookstore.books

-- Check table sizes
SELECT
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(oid)) AS total_size
FROM pg_class
WHERE relnamespace = 'bookstore'::regnamespace
  AND relkind = 'r';
```

**3. ACID Properties Demonstration:**

Demonstrate transaction behaviour by observing ACID properties in action:

```sql
-- Atomicity: all-or-nothing
BEGIN;
INSERT INTO bookstore.authors (name, country) VALUES ('Test Author', 'Test');
-- Simulate a failure — roll back before commit
ROLLBACK;
-- Verify: the author should NOT exist
SELECT * FROM bookstore.authors WHERE name = 'Test Author';

-- Isolation: open two psql sessions and observe concurrent behaviour
-- Session 1:
BEGIN;
UPDATE bookstore.books SET price = 99.99 WHERE book_id = 1;
-- Do NOT commit yet — switch to Session 2

-- Session 2 (new terminal):
-- docker exec -it learn-postgres psql -U learner -d learn_db
SELECT price FROM bookstore.books WHERE book_id = 1;
-- What price do you see? Why?

-- Session 1:
COMMIT;

-- Session 2:
SELECT price FROM bookstore.books WHERE book_id = 1;
-- What price do you see now?
```

After completing, answer:
- What isolation level does PostgreSQL use by default?
- What would change if you set `SERIALIZABLE` isolation in Session 2?
- How does the write-ahead log (WAL) ensure durability?

### Knowledge Check

- [ ] What are the four properties of ACID, and what does each guarantee?
- [ ] How does the CAP theorem constrain distributed database design? Give an example of a CP and an AP system
- [ ] What is the difference between a B-tree and an LSM-tree storage engine? When would you prefer each?
- [ ] What does BASE stand for, and how does it differ from ACID in practice?
- [ ] Name three database categories beyond relational and give a real-world use case for each

---

## Phase 2: Relational Databases Deep Dive (Week 3–4)

### Learning Objectives

- Write fluent SQL: JOINs, subqueries, CTEs, window functions, and aggregate queries
- Understand and apply normalization (1NF through 3NF/BCNF) and know when to denormalize
- Design entity-relationship models and translate them to physical schemas
- Master indexing: B-tree, hash, GIN, GiST — and know when each is appropriate
- Understand transaction isolation levels and their impact on concurrent workloads
- Model real-world domains with proper constraints, foreign keys, and data types

> **DDIA Reference:** Chapter 2 (Data Models and Query Languages), Chapter 7 (Transactions)

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [01-RELATIONAL-DATABASES](01-RELATIONAL-DATABASES.md) | SQL fundamentals, PostgreSQL and MySQL specifics, indexing, transactions, constraints |
| 2 | [03-DATA-MODELING](03-DATA-MODELING.md) | Entity-relationship modeling, normalization, denormalization trade-offs, schema design |

### Exercises

**1. Design a Normalized E-Commerce Schema:**

Design and implement a fully normalized (3NF) schema for an e-commerce system. The schema must support:

- Customers with addresses (one customer can have multiple addresses: billing, shipping)
- Products with categories (many-to-many relationship)
- Orders with line items, quantities, and prices at time of purchase
- Inventory tracking per product
- Product reviews with ratings

```sql
CREATE SCHEMA IF NOT EXISTS ecommerce;
SET search_path TO ecommerce;

-- Customers
CREATE TABLE customers (
    customer_id  SERIAL PRIMARY KEY,
    email        VARCHAR(255) NOT NULL UNIQUE,
    first_name   VARCHAR(100) NOT NULL,
    last_name    VARCHAR(100) NOT NULL,
    created_at   TIMESTAMPTZ DEFAULT NOW()
);

-- Customer addresses (1:N)
CREATE TABLE customer_addresses (
    address_id    SERIAL PRIMARY KEY,
    customer_id   INTEGER NOT NULL REFERENCES customers(customer_id),
    address_type  VARCHAR(20) NOT NULL CHECK (address_type IN ('billing', 'shipping')),
    street        VARCHAR(500) NOT NULL,
    city          VARCHAR(200) NOT NULL,
    state         VARCHAR(100),
    postal_code   VARCHAR(20) NOT NULL,
    country       VARCHAR(100) NOT NULL,
    is_default    BOOLEAN DEFAULT FALSE,
    UNIQUE (customer_id, address_type, is_default) -- only one default per type
);

-- Categories
CREATE TABLE categories (
    category_id  SERIAL PRIMARY KEY,
    name         VARCHAR(200) NOT NULL,
    parent_id    INTEGER REFERENCES categories(category_id),
    slug         VARCHAR(200) NOT NULL UNIQUE
);

-- Products
CREATE TABLE products (
    product_id   SERIAL PRIMARY KEY,
    sku          VARCHAR(50) NOT NULL UNIQUE,
    name         VARCHAR(500) NOT NULL,
    description  TEXT,
    price        NUMERIC(10, 2) NOT NULL CHECK (price >= 0),
    created_at   TIMESTAMPTZ DEFAULT NOW()
);

-- Product-Category junction (M:N)
CREATE TABLE product_categories (
    product_id   INTEGER NOT NULL REFERENCES products(product_id),
    category_id  INTEGER NOT NULL REFERENCES categories(category_id),
    PRIMARY KEY (product_id, category_id)
);

-- Inventory
CREATE TABLE inventory (
    product_id      INTEGER PRIMARY KEY REFERENCES products(product_id),
    quantity         INTEGER NOT NULL DEFAULT 0 CHECK (quantity >= 0),
    reorder_level    INTEGER NOT NULL DEFAULT 10,
    last_restocked   TIMESTAMPTZ
);

-- Orders
CREATE TABLE orders (
    order_id         SERIAL PRIMARY KEY,
    customer_id      INTEGER NOT NULL REFERENCES customers(customer_id),
    shipping_address INTEGER NOT NULL REFERENCES customer_addresses(address_id),
    billing_address  INTEGER NOT NULL REFERENCES customer_addresses(address_id),
    status           VARCHAR(30) NOT NULL DEFAULT 'pending'
                     CHECK (status IN ('pending','confirmed','shipped','delivered','cancelled')),
    total_amount     NUMERIC(12, 2) NOT NULL CHECK (total_amount >= 0),
    placed_at        TIMESTAMPTZ DEFAULT NOW()
);

-- Order line items
CREATE TABLE order_items (
    order_item_id  SERIAL PRIMARY KEY,
    order_id       INTEGER NOT NULL REFERENCES orders(order_id),
    product_id     INTEGER NOT NULL REFERENCES products(product_id),
    quantity        INTEGER NOT NULL CHECK (quantity > 0),
    unit_price      NUMERIC(10, 2) NOT NULL CHECK (unit_price >= 0),
    UNIQUE (order_id, product_id)
);

-- Reviews
CREATE TABLE reviews (
    review_id    SERIAL PRIMARY KEY,
    product_id   INTEGER NOT NULL REFERENCES products(product_id),
    customer_id  INTEGER NOT NULL REFERENCES customers(customer_id),
    rating       SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    title        VARCHAR(200),
    body         TEXT,
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (product_id, customer_id) -- one review per customer per product
);
```

After creating the schema:
- Draw the entity-relationship diagram (on paper or using a tool like dbdiagram.io)
- Identify the normal form of each table — is anything below 3NF?
- Identify which tables would benefit from denormalization under high read load and explain why

**2. Complex Queries — JOINs, CTEs, and Window Functions:**

Using the e-commerce schema above, write and execute these queries:

```sql
-- Seed test data first
INSERT INTO ecommerce.customers (email, first_name, last_name) VALUES
    ('alice@example.com', 'Alice', 'Smith'),
    ('bob@example.com', 'Bob', 'Jones'),
    ('carol@example.com', 'Carol', 'Williams');

INSERT INTO ecommerce.categories (name, slug) VALUES
    ('Electronics', 'electronics'),
    ('Books', 'books'),
    ('Clothing', 'clothing');

INSERT INTO ecommerce.products (sku, name, price) VALUES
    ('ELEC-001', 'Wireless Headphones', 79.99),
    ('ELEC-002', 'USB-C Hub', 49.99),
    ('BOOK-001', 'DDIA', 44.99),
    ('BOOK-002', 'Database Internals', 49.99),
    ('CLTH-001', 'Developer T-Shirt', 24.99);

-- Query 1: Multi-table JOIN — customers with their orders and items
-- (add orders and order_items data, then join across 4 tables)

-- Query 2: CTE — monthly revenue with running total
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', o.placed_at) AS month,
        SUM(oi.quantity * oi.unit_price) AS revenue
    FROM ecommerce.orders o
    JOIN ecommerce.order_items oi ON o.order_id = oi.order_id
    WHERE o.status != 'cancelled'
    GROUP BY DATE_TRUNC('month', o.placed_at)
)
SELECT
    month,
    revenue,
    SUM(revenue) OVER (ORDER BY month) AS running_total
FROM monthly_revenue
ORDER BY month;

-- Query 3: Window function — rank products by revenue within each category
SELECT
    c.name AS category,
    p.name AS product,
    SUM(oi.quantity * oi.unit_price) AS total_revenue,
    RANK() OVER (
        PARTITION BY c.category_id
        ORDER BY SUM(oi.quantity * oi.unit_price) DESC
    ) AS revenue_rank
FROM ecommerce.products p
JOIN ecommerce.product_categories pc ON p.product_id = pc.product_id
JOIN ecommerce.categories c ON pc.category_id = c.category_id
JOIN ecommerce.order_items oi ON p.product_id = oi.product_id
GROUP BY c.category_id, c.name, p.product_id, p.name;

-- Query 4: Correlated subquery — customers who spent more than average
SELECT
    c.first_name,
    c.last_name,
    customer_total.total_spent
FROM ecommerce.customers c
JOIN (
    SELECT customer_id, SUM(total_amount) AS total_spent
    FROM ecommerce.orders
    WHERE status != 'cancelled'
    GROUP BY customer_id
) customer_total ON c.customer_id = customer_total.customer_id
WHERE customer_total.total_spent > (
    SELECT AVG(total_amount) FROM ecommerce.orders WHERE status != 'cancelled'
);

-- Query 5: Window function — find gaps in order sequence
SELECT
    order_id,
    placed_at,
    LAG(placed_at) OVER (ORDER BY placed_at) AS previous_order,
    placed_at - LAG(placed_at) OVER (ORDER BY placed_at) AS time_gap
FROM ecommerce.orders
ORDER BY placed_at;
```

For each query:
- Explain what it returns in plain English
- Identify which indexes would improve its performance
- Run `EXPLAIN` (not `EXPLAIN ANALYZE` yet — that comes in Phase 4) and note the plan

### Knowledge Check

- [ ] What is the difference between 2NF and 3NF? Give an example of a 3NF violation
- [ ] When is denormalization justified, and what are the trade-offs?
- [ ] What is the difference between `INNER JOIN`, `LEFT JOIN`, and `CROSS JOIN`?
- [ ] How does a CTE differ from a subquery? When would you prefer each?
- [ ] What are the four standard transaction isolation levels and what anomalies does each prevent?

---

## Phase 3: Beyond Relational (Week 5–6)

### Learning Objectives

- Understand the motivations for NoSQL: schema flexibility, horizontal scalability, specialized data models
- Learn the four major NoSQL categories: document, key-value, column-family, and graph
- Model data in MongoDB (document) and Redis (key-value) and understand their operational characteristics
- Compare relational and document models for the same domain — understand the trade-offs empirically
- Understand when to choose NoSQL over relational and when relational is still the right answer
- Grasp the concept of polyglot persistence — using multiple databases in a single system

> **DDIA Reference:** Chapter 2 (Data Models and Query Languages — relational vs document debate), Chapter 5 (Replication)

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [02-NOSQL-DATABASES](02-NOSQL-DATABASES.md) | Document, key-value, column-family, graph models, consistency trade-offs, use cases |
| 2 | [11-SCHEMA-DESIGN-PATTERNS](11-SCHEMA-DESIGN-PATTERNS.md) | SQL and NoSQL schema patterns: multi-tenancy, event sourcing, bucket, single-table design, CQRS |

### Exercises

**1. Set Up MongoDB and Redis with Docker:**

Run MongoDB and Redis locally alongside PostgreSQL:

```yaml
# docker-compose.yml — multi-database stack
version: "3.8"
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: learner
      POSTGRES_PASSWORD: learner123
      POSTGRES_DB: learn_db
    ports:
      - "5432:5432"

  mongodb:
    image: mongo:7
    environment:
      MONGO_INITDB_ROOT_USERNAME: learner
      MONGO_INITDB_ROOT_PASSWORD: learner123
    ports:
      - "27017:27017"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

```bash
docker compose up -d

# Connect to MongoDB
docker exec -it $(docker compose ps -q mongodb) mongosh \
  -u learner -p learner123 --authenticationDatabase admin

# Connect to Redis
docker exec -it $(docker compose ps -q redis) redis-cli
```

**2. Model the Same Data in Relational and Document Style:**

Take the e-commerce domain from Phase 2 and model it in MongoDB. Notice the design differences:

```javascript
// MongoDB — document model (denormalized)
// Products collection with embedded categories and reviews
db.products.insertMany([
  {
    sku: "ELEC-001",
    name: "Wireless Headphones",
    price: 79.99,
    categories: ["Electronics", "Audio"],
    inventory: { quantity: 150, reorderLevel: 20 },
    reviews: [
      {
        customerId: "alice@example.com",
        rating: 5,
        title: "Great sound quality",
        body: "Best headphones I have ever owned.",
        createdAt: new Date("2024-01-15")
      },
      {
        customerId: "bob@example.com",
        rating: 4,
        title: "Good value",
        body: "Solid headphones for the price.",
        createdAt: new Date("2024-02-01")
      }
    ],
    createdAt: new Date()
  },
  {
    sku: "BOOK-001",
    name: "Designing Data-Intensive Applications",
    price: 44.99,
    categories: ["Books", "Technology"],
    inventory: { quantity: 500, reorderLevel: 50 },
    reviews: [],
    createdAt: new Date()
  }
]);

// Orders collection with embedded line items (snapshot pattern)
db.orders.insertOne({
  customerId: "alice@example.com",
  status: "confirmed",
  shippingAddress: {
    street: "123 Main St",
    city: "Portland",
    state: "OR",
    postalCode: "97201",
    country: "US"
  },
  items: [
    { sku: "ELEC-001", name: "Wireless Headphones", quantity: 1, unitPrice: 79.99 },
    { sku: "BOOK-001", name: "DDIA", quantity: 1, unitPrice: 44.99 }
  ],
  totalAmount: 124.98,
  placedAt: new Date()
});
```

Now compare the two approaches by answering these questions:

| Question | Relational (PostgreSQL) | Document (MongoDB) |
|----------|------------------------|--------------------|
| How many tables/collections are needed? | | |
| How do you query "all orders for customer X"? | | |
| How do you update a product's price? | | |
| How do you find average rating per product? | | |
| What happens when a review is added? | | |
| How do you enforce "one review per customer per product"? | | |

**3. Redis Data Structures Exercise:**

Explore Redis data structures for caching and session management:

```bash
# Strings — simple caching
SET product:ELEC-001:price "79.99" EX 3600
GET product:ELEC-001:price
TTL product:ELEC-001:price

# Hashes — structured objects
HSET session:user:alice email "alice@example.com" cart_count "3" last_active "2024-01-15T10:30:00Z"
HGETALL session:user:alice
HINCRBY session:user:alice cart_count 1

# Sorted sets — leaderboards / rankings
ZADD product:ratings 4.5 "ELEC-001" 4.8 "BOOK-001" 3.9 "CLTH-001"
ZREVRANGE product:ratings 0 -1 WITHSCORES
ZRANGEBYSCORE product:ratings 4.0 5.0

# Lists — recent activity feed
LPUSH user:alice:recent_views "ELEC-001" "BOOK-001" "CLTH-001"
LRANGE user:alice:recent_views 0 4
LTRIM user:alice:recent_views 0 9  -- keep only last 10

# Sets — tags, unique visitors
SADD product:ELEC-001:tags "wireless" "bluetooth" "audio"
SADD product:BOOK-001:tags "databases" "distributed-systems"
SINTER product:ELEC-001:tags product:BOOK-001:tags  -- common tags
```

After completing, answer:
- When would you use a Redis hash vs a JSON document in MongoDB?
- What happens to Redis data if the server restarts? How can you prevent data loss?
- What is the time complexity of `ZADD`, `ZRANGEBYSCORE`, and `LPUSH`?

### Knowledge Check

- [ ] What are the four major NoSQL database categories and what is each optimized for?
- [ ] When does a document model outperform a relational model? When is it worse?
- [ ] What is polyglot persistence and what operational challenges does it introduce?
- [ ] How does MongoDB handle transactions, and how does it compare to PostgreSQL?
- [ ] What Redis data structure would you use for a rate limiter? For a leaderboard?

---

## Phase 4: Performance and Optimization (Week 7–8)

### Learning Objectives

- Read and interpret PostgreSQL `EXPLAIN ANALYZE` output: scan types, costs, row estimates, actual times
- Understand index types (B-tree, hash, GIN, GiST, BRIN) and when each is appropriate
- Diagnose and fix slow queries using systematic optimization techniques
- Understand query planner behaviour: statistics, cost estimation, and plan selection
- Implement caching strategies: cache-aside, read-through, write-through, write-behind
- Design a caching layer with Redis and understand cache invalidation challenges

> **DDIA Reference:** Chapter 3 (Storage and Retrieval — indexes, SSTables, B-trees), Chapter 5 (Replication — read scaling)

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [04-QUERY-OPTIMIZATION](04-QUERY-OPTIMIZATION.md) | EXPLAIN ANALYZE, index types, query planner, statistics, optimization techniques |
| 2 | [07-CACHING](07-CACHING.md) | Cache strategies, Redis patterns, TTL, invalidation, cache stampede, consistency |

### Exercises

**1. EXPLAIN ANALYZE Deep Dive:**

Create a larger dataset and use `EXPLAIN ANALYZE` to diagnose query performance:

```sql
-- Generate a larger dataset for meaningful performance analysis
SET search_path TO ecommerce;

-- Insert 10,000 customers
INSERT INTO customers (email, first_name, last_name)
SELECT
    'user' || i || '@example.com',
    'First' || i,
    'Last' || (i % 100)
FROM generate_series(10, 10009) AS i;

-- Insert 1,000 products
INSERT INTO products (sku, name, price)
SELECT
    'SKU-' || LPAD(i::TEXT, 5, '0'),
    'Product ' || i,
    ROUND((RANDOM() * 200 + 5)::NUMERIC, 2)
FROM generate_series(10, 1009) AS i;

-- Insert 50,000 orders
INSERT INTO orders (customer_id, shipping_address, billing_address, status, total_amount, placed_at)
SELECT
    (RANDOM() * 9999 + 1)::INT,
    1, 1,  -- placeholder addresses
    (ARRAY['pending','confirmed','shipped','delivered','cancelled'])[FLOOR(RANDOM()*5+1)::INT],
    ROUND((RANDOM() * 500 + 10)::NUMERIC, 2),
    NOW() - (RANDOM() * 365 || ' days')::INTERVAL
FROM generate_series(1, 50000);

-- Insert 150,000 order items
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT
    (RANDOM() * 49999 + 1)::INT,
    (RANDOM() * 999 + 1)::INT,
    FLOOR(RANDOM() * 5 + 1)::INT,
    ROUND((RANDOM() * 200 + 5)::NUMERIC, 2)
FROM generate_series(1, 150000)
ON CONFLICT DO NOTHING;

-- Update table statistics
ANALYZE;
```

Now diagnose these queries:

```sql
-- Query 1: Sequential scan — no index
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'shipped' AND placed_at > NOW() - INTERVAL '30 days';

-- Note the scan type, estimated vs actual rows, and execution time
-- Then add an index and re-run:
CREATE INDEX idx_orders_status_placed ON orders (status, placed_at);

EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'shipped' AND placed_at > NOW() - INTERVAL '30 days';

-- Compare: What changed? How much faster is it?

-- Query 2: JOIN performance
EXPLAIN ANALYZE
SELECT
    c.email,
    COUNT(o.order_id) AS order_count,
    SUM(o.total_amount) AS total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.placed_at > NOW() - INTERVAL '90 days'
GROUP BY c.customer_id, c.email
HAVING COUNT(o.order_id) > 5
ORDER BY total_spent DESC
LIMIT 20;

-- Query 3: Subquery vs JOIN — compare plans
EXPLAIN ANALYZE
SELECT * FROM products
WHERE product_id IN (
    SELECT product_id FROM order_items
    GROUP BY product_id
    HAVING SUM(quantity) > 100
);

-- Rewrite as JOIN and compare:
EXPLAIN ANALYZE
SELECT DISTINCT p.*
FROM products p
JOIN (
    SELECT product_id FROM order_items
    GROUP BY product_id
    HAVING SUM(quantity) > 100
) top_products ON p.product_id = top_products.product_id;
```

For each query pair, document:
- The plan before and after optimization (scan type, cost, time)
- Which indexes were used and why the planner chose them
- The speedup factor achieved

**2. Indexing Strategy Exercise:**

Design an optimal indexing strategy for the e-commerce schema. For each index, justify why it exists:

```sql
-- Analyze current index usage
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan AS times_used,
    idx_tup_read AS rows_read,
    idx_tup_fetch AS rows_fetched
FROM pg_stat_user_indexes
WHERE schemaname = 'ecommerce'
ORDER BY idx_scan DESC;

-- Find missing indexes — sequential scans on large tables
SELECT
    relname AS table_name,
    seq_scan,
    seq_tup_read,
    idx_scan,
    CASE WHEN seq_scan > 0
         THEN ROUND(seq_tup_read::NUMERIC / seq_scan, 0)
         ELSE 0
    END AS avg_rows_per_seq_scan
FROM pg_stat_user_tables
WHERE schemaname = 'ecommerce'
ORDER BY seq_tup_read DESC;

-- Create indexes based on your findings
-- Example: partial index for active orders only
CREATE INDEX idx_orders_active ON orders (customer_id, placed_at)
WHERE status NOT IN ('cancelled', 'delivered');

-- Example: covering index to avoid table lookups
CREATE INDEX idx_order_items_covering ON order_items (order_id)
INCLUDE (product_id, quantity, unit_price);
```

**3. Redis Caching Layer Design:**

Design and implement a caching strategy for the e-commerce product catalog:

```python
# cache_strategy.py — cache-aside pattern with Redis
import json
import redis
import psycopg2

redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)
CACHE_TTL = 3600  # 1 hour

def get_product(product_id: int) -> dict:
    """Cache-aside pattern: check cache first, fall back to database."""
    cache_key = f"product:{product_id}"

    # Step 1: Check cache
    cached = redis_client.get(cache_key)
    if cached:
        return json.loads(cached)

    # Step 2: Cache miss — query database
    conn = psycopg2.connect(
        host="localhost", dbname="learn_db",
        user="learner", password="learner123"
    )
    cur = conn.cursor()
    cur.execute(
        "SELECT product_id, sku, name, price FROM ecommerce.products WHERE product_id = %s",
        (product_id,)
    )
    row = cur.fetchone()
    cur.close()
    conn.close()

    if not row:
        return None

    product = {
        "product_id": row[0],
        "sku": row[1],
        "name": row[2],
        "price": float(row[3])
    }

    # Step 3: Populate cache
    redis_client.setex(cache_key, CACHE_TTL, json.dumps(product))
    return product


def invalidate_product(product_id: int):
    """Invalidate cache when product is updated."""
    redis_client.delete(f"product:{product_id}")


def update_product_price(product_id: int, new_price: float):
    """Update database, then invalidate cache."""
    conn = psycopg2.connect(
        host="localhost", dbname="learn_db",
        user="learner", password="learner123"
    )
    cur = conn.cursor()
    cur.execute(
        "UPDATE ecommerce.products SET price = %s WHERE product_id = %s",
        (new_price, product_id)
    )
    conn.commit()
    cur.close()
    conn.close()

    # Invalidate — do NOT update cache (avoids race conditions)
    invalidate_product(product_id)
```

After implementing, answer:
- What happens if Redis is down? Does the application still work?
- What is a cache stampede and how would you prevent it?
- Why does `update_product_price` invalidate the cache instead of updating it?

### Knowledge Check

- [ ] What is the difference between a sequential scan, an index scan, and an index-only scan?
- [ ] When does PostgreSQL choose a sequential scan even when an index exists?
- [ ] What is a partial index and when is it more efficient than a full index?
- [ ] What are the trade-offs between cache-aside and read-through caching patterns?
- [ ] How do you handle cache invalidation in a system with multiple application servers?

---

## Phase 5: Scaling and Operations (Week 9–10)

### Learning Objectives

- Understand replication topologies: single-leader, multi-leader, and leaderless
- Grasp the differences between synchronous and asynchronous replication and their consistency implications
- Learn sharding strategies: hash-based, range-based, and directory-based partitioning
- Plan and execute zero-downtime database migrations using the expand-contract pattern
- Understand connection pooling, PgBouncer, and connection management in production
- Design for high availability with automatic failover

> **DDIA Reference:** Chapter 5 (Replication), Chapter 6 (Partitioning), Chapter 7 (Transactions — distributed)

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [05-REPLICATION-AND-SHARDING](05-REPLICATION-AND-SHARDING.md) | Replication topologies, leader election, sharding strategies, rebalancing |
| 2 | [06-MIGRATIONS](06-MIGRATIONS.md) | Schema evolution, expand-contract pattern, zero-downtime migrations, rollback |
| 3 | [08-CONNECTION-MANAGEMENT](08-CONNECTION-MANAGEMENT.md) | Connection pooling, PgBouncer, timeouts, connection limits, health checks |

### Exercises

**1. Set Up PostgreSQL Streaming Replication:**

Configure a primary-replica PostgreSQL setup using Docker:

```yaml
# docker-compose-replication.yml
version: "3.8"
services:
  pg-primary:
    image: postgres:16
    environment:
      POSTGRES_USER: replicator
      POSTGRES_PASSWORD: replicator123
      POSTGRES_DB: app_db
    ports:
      - "5432:5432"
    command: >
      postgres
        -c wal_level=replica
        -c max_wal_senders=3
        -c max_replication_slots=3
        -c hot_standby=on
    volumes:
      - pg_primary_data:/var/lib/postgresql/data

  pg-replica:
    image: postgres:16
    environment:
      PGUSER: replicator
      PGPASSWORD: replicator123
    ports:
      - "5433:5432"
    depends_on:
      - pg-primary
    volumes:
      - pg_replica_data:/var/lib/postgresql/data

volumes:
  pg_primary_data:
  pg_replica_data:
```

After setup:

```sql
-- On primary (port 5432): check replication status
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    replay_lsn,
    replay_lag
FROM pg_stat_replication;

-- On replica (port 5433): verify it is in recovery mode
SELECT pg_is_in_recovery();

-- Test replication: write on primary, read on replica
-- Primary:
CREATE TABLE replication_test (id SERIAL, message TEXT, created_at TIMESTAMPTZ DEFAULT NOW());
INSERT INTO replication_test (message) VALUES ('Hello from primary');

-- Replica (after a moment):
SELECT * FROM replication_test;
```

Experiment with replication lag:
- Insert 10,000 rows on the primary in a transaction
- Immediately query the replica — do you see all rows? Why or why not?
- Monitor `replay_lag` during the insertion

**2. Zero-Downtime Migration with Expand-Contract:**

Practice the expand-contract pattern for a schema migration that renames a column without downtime:

```sql
-- Scenario: rename orders.placed_at to orders.ordered_at
-- This must be done with zero downtime — the application is actively reading/writing

-- Phase 1: EXPAND — add new column
ALTER TABLE ecommerce.orders ADD COLUMN ordered_at TIMESTAMPTZ;

-- Phase 2: MIGRATE — backfill existing data
UPDATE ecommerce.orders SET ordered_at = placed_at WHERE ordered_at IS NULL;

-- Phase 3: DUAL-WRITE — application writes to both columns
-- (simulate with a trigger)
CREATE OR REPLACE FUNCTION sync_ordered_at()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.ordered_at IS NULL THEN
        NEW.ordered_at := NEW.placed_at;
    END IF;
    IF NEW.placed_at IS NULL THEN
        NEW.placed_at := NEW.ordered_at;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_ordered_at
    BEFORE INSERT OR UPDATE ON ecommerce.orders
    FOR EACH ROW EXECUTE FUNCTION sync_ordered_at();

-- Phase 4: SWITCH READS — application now reads from ordered_at
-- Verify: both columns are in sync
SELECT COUNT(*) FROM ecommerce.orders WHERE placed_at != ordered_at;

-- Phase 5: CONTRACT — drop old column (only after all consumers migrated)
-- DROP TRIGGER trg_sync_ordered_at ON ecommerce.orders;
-- ALTER TABLE ecommerce.orders DROP COLUMN placed_at;
```

Document each phase:
- What is the risk at each step?
- What happens if you need to roll back at Phase 3? At Phase 5?
- Why can you not simply `ALTER TABLE RENAME COLUMN` in a zero-downtime deployment?

**3. Connection Pooling with PgBouncer:**

Set up PgBouncer in front of PostgreSQL and measure the difference:

```ini
; pgbouncer.ini
[databases]
app_db = host=pg-primary port=5432 dbname=app_db

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
default_pool_size = 20
max_client_conn = 200
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 3
server_idle_timeout = 300
log_connections = 1
log_disconnections = 1
```

After setup, compare:
- Direct connection: `psql -h localhost -p 5432 -U replicator -d app_db`
- Pooled connection: `psql -h localhost -p 6432 -U replicator -d app_db`

```sql
-- Monitor connections through PgBouncer
SHOW pools;
SHOW stats;
SHOW clients;

-- Compare connection overhead
-- Direct: each new psql session = new PostgreSQL backend process
-- Pooled: PgBouncer reuses existing connections
```

### Knowledge Check

- [ ] What is the difference between synchronous and asynchronous replication? What are the trade-offs?
- [ ] What is replication lag and how does it affect read-after-write consistency?
- [ ] Explain the expand-contract migration pattern. Why is it safer than a single-step migration?
- [ ] What is the difference between PgBouncer's `session`, `transaction`, and `statement` pool modes?
- [ ] How do you determine the right `max_connections` setting for PostgreSQL?

---

## Phase 6: Production Readiness (Week 11–12)

### Learning Objectives

- Apply database best practices for schema design, queries, indexing, and operations
- Identify and remediate common database anti-patterns
- Implement monitoring and alerting for database health: connections, replication lag, query performance
- Design a backup and recovery strategy with RPO/RTO targets
- Understand point-in-time recovery (PITR) using WAL archiving
- Build a production-readiness checklist for database deployments

> **DDIA Reference:** Chapter 7 (Transactions — serializability), Chapter 8 (The Trouble with Distributed Systems), Chapter 9 (Consistency and Consensus)

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [09-BEST-PRACTICES](09-BEST-PRACTICES.md) | Production patterns, naming conventions, backup strategy, monitoring, security |
| 2 | [10-ANTI-PATTERNS](10-ANTI-PATTERNS.md) | Common mistakes, N+1 queries, God tables, soft deletes, EAV pattern |

### Exercises

**1. Database Best Practices Audit:**

Select a database schema (your own, the e-commerce capstone, or an open-source project) and perform a full audit against the best practices checklist from [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md):

```
Schema Design:
  [ ] All tables have a primary key
  [ ] Foreign keys defined for all relationships
  [ ] Appropriate data types (no VARCHAR for dates, no TEXT for booleans)
  [ ] CHECK constraints for domain validation
  [ ] Naming convention consistent (snake_case, singular/plural)
  [ ] No reserved words used as column names

Indexing:
  [ ] All foreign key columns are indexed
  [ ] Frequently filtered columns have appropriate indexes
  [ ] No duplicate or redundant indexes
  [ ] Partial indexes used where appropriate
  [ ] Index usage monitored via pg_stat_user_indexes

Queries:
  [ ] No SELECT * in application code
  [ ] N+1 query patterns eliminated (use JOINs or batch loading)
  [ ] Pagination uses keyset (cursor) instead of OFFSET
  [ ] All queries tested with EXPLAIN ANALYZE under production-like data volumes
  [ ] Prepared statements used to prevent SQL injection

Operations:
  [ ] Automated backups configured (pg_dump or WAL archiving)
  [ ] Backup restoration tested (at least quarterly)
  [ ] Monitoring: connections, replication lag, query duration, table bloat
  [ ] Connection pooling configured (PgBouncer or application-level)
  [ ] Migrations tested in staging before production
```

For each unchecked item, write a one-paragraph remediation plan: what to change, estimated effort, and priority.

**2. Anti-Pattern Identification Exercise:**

Review the following system description and identify at least 5 anti-patterns from [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md):

> "We have a monolithic application with a single PostgreSQL database. Our main `orders` table has 85 columns including `customer_name`, `customer_email`, `customer_phone`, `shipping_street`, `shipping_city` (all denormalized from customers). We use `SELECT *` everywhere because it is easier. Our API endpoint for listing orders uses `OFFSET/LIMIT` pagination — at page 500, it takes 12 seconds. We have a `data` table with columns `id`, `entity_type`, `attribute_name`, `attribute_value` for storing 'flexible' data. Soft deletes are implemented with an `is_deleted` boolean on every table, but we never actually delete the rows, so the tables keep growing. We don't use database transactions for multi-step operations — 'the application handles consistency'. Our connection string is hardcoded with `max_connections=500` even though we only have 10 application instances."

For each anti-pattern identified:
- State the anti-pattern name
- Quote the specific evidence from the description
- Write the recommended fix in 2–3 sentences

<details>
<summary>Anti-patterns identified (click to reveal suggested answers)</summary>

1. **God Table** — `orders` table with 85 columns including denormalized customer and shipping data. Split into normalized tables: `orders`, `customers`, `addresses`, with foreign key relationships.
2. **SELECT \*** — Using `SELECT *` everywhere instead of specifying columns. Replace with explicit column lists; only fetch what the application needs.
3. **OFFSET Pagination** — `OFFSET/LIMIT` at page 500 takes 12 seconds because the database must scan and discard all prior rows. Replace with keyset (cursor-based) pagination using `WHERE id > last_seen_id ORDER BY id LIMIT 20`.
4. **Entity-Attribute-Value (EAV)** — The `data` table with `entity_type`, `attribute_name`, `attribute_value` columns. Replace with proper typed tables or use PostgreSQL JSONB for semi-structured data.
5. **Unbounded Soft Deletes** — `is_deleted` boolean on every table with no cleanup. Implement a scheduled archival job that moves soft-deleted rows to archive tables after a retention period.
6. **No Transactions** — Multi-step operations without database transactions. Wrap related writes in explicit `BEGIN/COMMIT` blocks; use savepoints for partial rollback.
7. **Over-provisioned Connections** — `max_connections=500` for 10 app instances. Use PgBouncer with a pool size matched to actual concurrency needs (typically 2–4 connections per CPU core).

</details>

**3. Capstone Project — Production Database Layer:**

Design and implement a complete production-ready database layer for a task management application. This project ties together all six phases:

```
Application Requirements:
- Users can create projects and tasks
- Tasks have assignees, due dates, priorities, labels, and comments
- Users can be members of multiple projects with different roles (owner, admin, member)
- Activity log for all changes (audit trail)
- Full-text search on task titles and descriptions
- API supports cursor-based pagination, filtering, and sorting
```

| Component | What to Implement |
|-----------|-------------------|
| **Schema Design** | Normalized schema in 3NF with proper types, constraints, and foreign keys (Phase 2) |
| **Indexing** | B-tree indexes on foreign keys and filter columns; GIN index for full-text search (Phase 4) |
| **Caching** | Redis cache-aside for project and task detail endpoints with invalidation (Phase 4) |
| **Migrations** | Version-controlled migration files using a tool (Flyway, golang-migrate, or Alembic); one migration must use expand-contract (Phase 5) |
| **Connection Pooling** | PgBouncer configuration with transaction-mode pooling (Phase 5) |
| **Monitoring** | Prometheus metrics: query duration histogram, active connections gauge, cache hit ratio (Phase 6) |
| **Backups** | Automated pg_dump script with retention policy; document RPO and RTO targets (Phase 6) |
| **Documentation** | README with ER diagram, setup instructions, migration guide, and runbook for common operations |

Evaluation Criteria:

| Area | What to Verify |
|------|----------------|
| **Schema** | Is the schema in 3NF? Are all relationships properly constrained? |
| **Queries** | Do all queries use EXPLAIN ANALYZE under production-like data volume? Are there no N+1 patterns? |
| **Indexes** | Are indexes justified with usage stats? No redundant indexes? |
| **Caching** | Does the cache-aside implementation handle invalidation correctly? What happens when Redis is down? |
| **Migrations** | Can migrations run forward and backward? Is the expand-contract migration reversible? |
| **Pooling** | Is PgBouncer configured with appropriate pool sizes? Can the system handle connection spikes? |
| **Monitoring** | Can you answer "Is the database healthy right now?" from a dashboard in under 10 seconds? |
| **Backup** | Has a backup been restored successfully at least once? Is the RPO documented? |

### Knowledge Check

- [ ] What is the difference between RPO (Recovery Point Objective) and RTO (Recovery Time Objective)?
- [ ] How does PostgreSQL point-in-time recovery (PITR) work with WAL archiving?
- [ ] What are the top 3 metrics you would monitor for a production PostgreSQL database?
- [ ] What is the N+1 query problem and how do you detect it in production?
- [ ] When is the EAV pattern justified, and when is PostgreSQL JSONB a better alternative?

---

## Quick Reference: Document Map

| # | Document | Phase | Key Topics |
|---|----------|-------|------------|
| 00 | [OVERVIEW](00-OVERVIEW.md) | 1 | Database categories, ACID vs BASE, CAP theorem, storage engines |
| 01 | [RELATIONAL-DATABASES](01-RELATIONAL-DATABASES.md) | 2 | SQL, PostgreSQL, MySQL, indexing, transactions, constraints |
| 02 | [NOSQL-DATABASES](02-NOSQL-DATABASES.md) | 3 | Document, key-value, column-family, graph, consistency models |
| 03 | [DATA-MODELING](03-DATA-MODELING.md) | 2 | ER modeling, normalization, denormalization, schema design |
| 04 | [QUERY-OPTIMIZATION](04-QUERY-OPTIMIZATION.md) | 4 | EXPLAIN ANALYZE, index types, query planner, statistics |
| 05 | [REPLICATION-AND-SHARDING](05-REPLICATION-AND-SHARDING.md) | 5 | Replication topologies, sharding strategies, rebalancing |
| 06 | [MIGRATIONS](06-MIGRATIONS.md) | 5 | Schema evolution, expand-contract, zero-downtime migrations |
| 07 | [CACHING](07-CACHING.md) | 4 | Redis, cache-aside, TTL, invalidation, stampede prevention |
| 08 | [CONNECTION-MANAGEMENT](08-CONNECTION-MANAGEMENT.md) | 5 | PgBouncer, pooling modes, timeouts, connection limits |
| 09 | [BEST-PRACTICES](09-BEST-PRACTICES.md) | 6 | Production patterns, backup, monitoring, naming conventions |
| 10 | [ANTI-PATTERNS](10-ANTI-PATTERNS.md) | 6 | God tables, N+1 queries, EAV, OFFSET pagination |
| 11 | [SCHEMA-DESIGN-PATTERNS](11-SCHEMA-DESIGN-PATTERNS.md) | 3 | Multi-tenancy, event sourcing, bucket, single-table design, CQRS |
| — | [LEARNING-PATH](LEARNING-PATH.md) | All | This document — structured 6-phase curriculum |

---

## Recommended Resources

### Books

| Book | Author | Focus |
|------|--------|-------|
| *Designing Data-Intensive Applications* | Martin Kleppmann | The definitive guide — data models, storage, replication, partitioning, transactions |
| *Database Internals* | Alex Petrov | Storage engines, B-trees, LSM-trees, distributed database architecture |
| *SQL Performance Explained* | Markus Winand | Indexing, execution plans, join algorithms — practical optimization |
| *PostgreSQL 14 Internals* | Egor Rogov | Deep PostgreSQL internals: MVCC, WAL, vacuum, buffer management |
| *High Performance MySQL* | Silvia Botros, Jeremy Tinley | MySQL optimization, replication, schema design, operations |
| *Seven Databases in Seven Weeks* | Luc Perkins et al. | Survey of PostgreSQL, MongoDB, Redis, Neo4j, CouchDB, HBase, DynamoDB |

### Online Resources

- **use-the-index-luke.com** — SQL indexing and tuning tutorial by Markus Winand
- **pganalyze.com/docs** — PostgreSQL performance guides and EXPLAIN visualization
- **redis.io/docs** — Redis documentation, data structure guides, best practices
- **mongodb.com/docs/manual** — MongoDB documentation, aggregation pipeline reference
- **postgresql.org/docs/current** — Official PostgreSQL documentation
- **brandur.org/postgres** — Advanced PostgreSQL articles on MVCC, serializable isolation, advisory locks

### Tools

| Tool | Purpose | Phase |
|------|---------|-------|
| PostgreSQL | Primary relational database | 1–6 |
| MongoDB | Document database | 3 |
| Redis | Key-value store and caching | 3, 4 |
| psql | PostgreSQL CLI client | 1–6 |
| pgAdmin / DBeaver | GUI database management | 1–6 |
| PgBouncer | Connection pooling | 5, 6 |
| Flyway / golang-migrate | Database migration tools | 5, Capstone |
| pg_dump / pg_basebackup | Backup utilities | 6, Capstone |
| pgbench | PostgreSQL benchmarking | 4, 6 |
| Docker Compose | Running local database stacks | 1–5 |

---

## Suggested Learning Path by Role

```
Backend Developer:
  00-OVERVIEW → 01-RELATIONAL → 03-DATA-MODELING → 11-SCHEMA-PATTERNS → 04-QUERY-OPT → 07-CACHING → 09-BEST-PRACTICES

Data Engineer:
  00-OVERVIEW → 01-RELATIONAL → 02-NOSQL → 11-SCHEMA-PATTERNS → 05-REPLICATION → 06-MIGRATIONS → 04-QUERY-OPT

DevOps / SRE:
  00-OVERVIEW → 05-REPLICATION → 08-CONNECTION-MGMT → 06-MIGRATIONS → 07-CACHING → 09-BEST-PRACTICES

Architect:
  00-OVERVIEW → All files → 11-SCHEMA-PATTERNS → 09-BEST-PRACTICES → 10-ANTI-PATTERNS
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial database learning path |
