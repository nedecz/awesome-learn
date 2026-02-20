# NoSQL Databases

## Table of Contents

1. [Overview](#overview)
   - [What Does NoSQL Mean?](#what-does-nosql-mean)
   - [History of the NoSQL Movement](#history-of-the-nosql-movement)
   - [Motivation](#motivation)
   - [NoSQL Categories at a Glance](#nosql-categories-at-a-glance)
2. [Document Databases](#document-databases)
   - [Schema-on-Read vs Schema-on-Write](#schema-on-read-vs-schema-on-write)
   - [Document Structure and BSON](#document-structure-and-bson)
   - [Embedding vs Referencing](#embedding-vs-referencing)
   - [MongoDB](#mongodb)
   - [CouchDB](#couchdb)
   - [Firestore](#firestore)
3. [Key-Value Stores](#key-value-stores)
   - [Redis](#redis)
   - [DynamoDB](#dynamodb)
   - [etcd](#etcd)
4. [Column-Family Databases](#column-family-databases)
   - [Cassandra](#cassandra)
   - [HBase](#hbase)
   - [ScyllaDB](#scylladb)
5. [Graph Databases](#graph-databases)
   - [Property Graph Model vs RDF](#property-graph-model-vs-rdf)
   - [Neo4j and Cypher](#neo4j-and-cypher)
   - [Amazon Neptune](#amazon-neptune)
   - [When Graphs Outperform Relational Joins](#when-graphs-outperform-relational-joins)
6. [Time-Series Databases](#time-series-databases)
   - [Data Characteristics](#data-characteristics)
   - [InfluxDB](#influxdb)
   - [TimescaleDB](#timescaledb)
   - [Prometheus TSDB](#prometheus-tsdb)
   - [Retention Policies](#retention-policies)
7. [Search Engines](#search-engines)
   - [Inverted Indexes](#inverted-indexes)
   - [Elasticsearch and OpenSearch](#elasticsearch-and-opensearch)
   - [Relevance Scoring](#relevance-scoring)
8. [Choosing the Right NoSQL Database](#choosing-the-right-nosql-database)
   - [Decision Framework](#decision-framework)
   - [Comparison Table](#comparison-table)
9. [Version History](#version-history)

---

## Overview

### What Does NoSQL Mean?

The term "NoSQL" originally stood for *"Not Only SQL."* It is not a rejection of SQL or relational databases but rather an acknowledgment that no single data model fits every problem. As *Designing Data-Intensive Applications* (DDIA) Chapter 2 explains, different data models make different things easy or hard — and the relational model, while remarkably versatile, is not the best choice for every workload.

NoSQL databases share a few common traits, though no single trait is universal:

- **Non-relational data models** — documents, key-value pairs, wide columns, graphs
- **Often schema-flexible** — the database does not enforce a rigid schema at write time
- **Designed for horizontal scaling** — distribute data across commodity hardware
- **Open-source roots** — many originated as open-source projects in the 2000s–2010s

### History of the NoSQL Movement

```
Timeline of the NoSQL Movement

2004 ─── Google publishes the Bigtable paper
2006 ─── Amazon publishes the Dynamo paper
2007 ─── MongoDB development begins
2008 ─── Cassandra open-sourced by Facebook; CouchDB joins Apache
2009 ─── "NoSQL" term gains popularity at a San Francisco meetup
          Redis 1.0 released
2010 ─── Neo4j 1.0 released; HBase becomes Apache top-level project
2012 ─── DynamoDB launched by AWS
2013 ─── Elasticsearch widely adopted; InfluxDB first released
2015 ─── ScyllaDB (C++ rewrite of Cassandra) released
2017 ─── Cloud Firestore launched; Amazon Neptune announced
2023 ─── Convergence: relational adds JSON/documents, NoSQL adds transactions
```

### Motivation

Three driving forces explain why NoSQL databases emerged and persist:

| Driving Force | Problem with Relational Alone | NoSQL Approach |
|---|---|---|
| **Scaling** | Vertical scaling of a single RDBMS is expensive and has hard limits | Horizontal scaling across commodity nodes with built-in sharding |
| **Flexible schemas** | Schema migrations on large tables can be slow and risky; rapid iteration is costly | Schema-on-read allows each record to have its own shape |
| **Specialized data models** | Relational joins are inefficient for deeply nested, highly connected, or append-only data | Purpose-built models (documents, graphs, time-series) map naturally to these workloads |

> *"If the data in your application has a document-like structure (i.e., a tree of one-to-many relationships), then it's probably a good idea to use a document model."* — DDIA, Chapter 2

### NoSQL Categories at a Glance

```
┌──────────────────────────────────────────────────────────────────────┐
│                        NoSQL Database Types                          │
├─────────────────┬─────────────────┬──────────────┬──────────────────┤
│   Document      │   Key-Value     │  Column-     │   Graph          │
│                 │                 │  Family      │                  │
│  ┌───────────┐  │  ┌───────────┐  │  ┌────────┐  │  ┌────────────┐ │
│  │ { JSON }  │  │  │ key → val │  │  │ CF:col │  │  │  (n)──(n)  │ │
│  │ { JSON }  │  │  │ key → val │  │  │ CF:col │  │  │  /    \    │ │
│  │ { JSON }  │  │  │ key → val │  │  │ CF:col │  │  │ (n)──(n)  │ │
│  └───────────┘  │  └───────────┘  │  └────────┘  │  └────────────┘ │
│                 │                 │              │                  │
│  MongoDB        │  Redis          │  Cassandra   │  Neo4j           │
│  CouchDB        │  DynamoDB       │  HBase       │  Neptune         │
│  Firestore      │  etcd           │  ScyllaDB    │                  │
├─────────────────┴─────────────────┴──────────────┴──────────────────┤
│                 Also: Time-Series, Search Engines                    │
│                 InfluxDB, TimescaleDB, Elasticsearch                 │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Document Databases

Document databases store data as semi-structured documents — typically JSON, BSON, or XML. Each document is self-contained and can have its own structure, making them a natural fit for applications with one-to-many relationships (DDIA Ch. 2).

### Schema-on-Read vs Schema-on-Write

This is one of the most important distinctions in database design, discussed at length in DDIA Chapter 2.

| Approach | Description | Analogy | Trade-off |
|---|---|---|---|
| **Schema-on-write** | The database enforces a schema when data is written (relational). Every row must conform to the table definition. | Static typing | Safety at write time; migrations required for changes |
| **Schema-on-read** | The structure is interpreted when the data is read. The database stores whatever you give it. | Dynamic typing | Flexibility at write time; application must handle schema variations |

```
Schema-on-Write (Relational)              Schema-on-Read (Document)
─────────────────────────────              ──────────────────────────────
                                           
  INSERT INTO users (name, email)            db.users.insertOne({
  VALUES ('Alice', 'a@test.com');              name: "Alice",
                                               email: "a@test.com",
  -- Adding a column:                          phone: "555-0100"    ← OK, no migration
  ALTER TABLE users                          })
    ADD COLUMN phone VARCHAR(20);
  -- Then backfill existing rows
```

Schema-on-read does **not** mean "no schema." Your application code always has an implicit schema. The question is whether the database or the application enforces it.

### Document Structure and BSON

MongoDB stores documents in **BSON** (Binary JSON), which extends JSON with additional types like `Date`, `ObjectId`, `Decimal128`, and `BinData`.

```json
// A single document in a "products" collection
{
  "_id": ObjectId("64a7f2e5c1d3b4001ef2a3b7"),
  "name": "Wireless Keyboard",
  "sku": "KB-2024-WL",
  "price": {
    "amount": 79.99,
    "currency": "USD"
  },
  "tags": ["electronics", "peripherals", "wireless"],
  "specifications": {
    "connectivity": "Bluetooth 5.1",
    "battery_life_hours": 200,
    "weight_grams": 450
  },
  "reviews": [
    {
      "user": "alice",
      "rating": 5,
      "text": "Great keyboard!",
      "date": ISODate("2025-03-15T10:30:00Z")
    }
  ],
  "created_at": ISODate("2025-01-10T08:00:00Z")
}
```

Key BSON types beyond JSON:

| BSON Type | Size | Purpose |
|---|---|---|
| `ObjectId` | 12 bytes | Default `_id`; encodes timestamp + random + counter |
| `Date` | 8 bytes | Milliseconds since Unix epoch |
| `Decimal128` | 16 bytes | Exact decimal arithmetic (financial data) |
| `BinData` | Variable | Binary blobs (images, encrypted data) |
| `Timestamp` | 8 bytes | Internal replication timestamp |

### Embedding vs Referencing

The fundamental data modeling decision in document databases is whether to **embed** related data inside a document or **reference** it via an ID.

```
Embedding (denormalized)                  Referencing (normalized)
────────────────────────                  ────────────────────────

┌─ order ─────────────────┐              ┌─ order ──────────────────┐
│ _id: "ord-1"            │              │ _id: "ord-1"             │
│ customer: {             │              │ customer_id: "cust-42"   │
│   name: "Alice",        │              │ items: [                 │
│   email: "a@test.com"   │              │   { product_id: "p-10",  │
│ }                       │              │     qty: 2 }             │
│ items: [                │              │ ]                        │
│   { name: "Widget",     │              └──────────────────────────┘
│     price: 9.99,        │                       │
│     qty: 2 }            │                       ▼
│ ]                       │              ┌─ customer ───────────────┐
└─────────────────────────┘              │ _id: "cust-42"           │
                                         │ name: "Alice"            │
 ✓ Single read for full order            │ email: "a@test.com"      │
 ✗ Data duplication                      └──────────────────────────┘
 ✗ Hard to update customer name
                                          ✓ No duplication
                                          ✗ Requires application-level join
```

**When to embed:**
- The child data is always accessed with the parent (1:1 or 1:few)
- The embedded data does not change independently
- The total document stays under 16 MB (MongoDB limit)

**When to reference:**
- The related entity is shared across many documents (many:many)
- The related data changes frequently and independently
- The related collection could grow unbounded (1:many with high cardinality)

### MongoDB

MongoDB is the most widely adopted document database. Key production features:

```javascript
// Create an index for common queries
db.orders.createIndex({ customer_id: 1, created_at: -1 })

// Aggregation pipeline: revenue per customer in the last 30 days
db.orders.aggregate([
  {
    $match: {
      created_at: { $gte: ISODate("2025-05-01T00:00:00Z") }
    }
  },
  { $unwind: "$items" },
  {
    $group: {
      _id: "$customer_id",
      total_revenue: { $sum: { $multiply: ["$items.price", "$items.qty"] } },
      order_count: { $sum: 1 }
    }
  },
  { $sort: { total_revenue: -1 } },
  { $limit: 10 }
])
```

**Common aggregation pipeline stages:**

| Stage | Purpose |
|---|---|
| `$match` | Filter documents (like WHERE) |
| `$group` | Aggregate values (like GROUP BY) |
| `$project` | Reshape documents (like SELECT) |
| `$unwind` | Deconstruct arrays into individual documents |
| `$lookup` | Left outer join to another collection |
| `$sort` | Order results |
| `$limit` / `$skip` | Pagination |

**MongoDB transactions** (since 4.0 for replica sets, 4.2 for sharded clusters):

```javascript
const session = client.startSession();
session.startTransaction();
try {
  await db.accounts.updateOne(
    { _id: "acc-from" },
    { $inc: { balance: -100 } },
    { session }
  );
  await db.accounts.updateOne(
    { _id: "acc-to" },
    { $inc: { balance: 100 } },
    { session }
  );
  await session.commitTransaction();
} catch (error) {
  await session.abortTransaction();
  throw error;
} finally {
  session.endSession();
}
```

### CouchDB

Apache CouchDB uses **MVCC** and exposes a RESTful HTTP API. Its distinguishing feature is **multi-master replication** — any node can accept writes and conflicts are resolved deterministically. Queries use MapReduce views or Mango (declarative JSON). CouchDB is ideal for offline-first mobile apps (via PouchDB) and distributed systems with intermittent connectivity.

### Firestore

Google Cloud Firestore is a serverless document database for real-time applications. Documents are organized into collections and subcollections. Key features include **real-time listeners** (clients subscribe to changes via WebSocket), **offline support** (client-side caching with automatic sync), **automatic sharding**, and **serializable transactions**. Limitations: max document size 1 MB, max ~1 write/sec per document.

---

## Key-Value Stores

Key-value stores are the simplest NoSQL model. Every item is stored as a key (unique identifier) mapped to a value (opaque blob or structured data). This simplicity enables extreme performance and horizontal scalability.

```
Key-Value Store Concept

  ┌─────────────────┬──────────────────────────────────┐
  │  Key             │  Value                           │
  ├─────────────────┼──────────────────────────────────┤
  │  user:1001       │  {"name":"Alice","role":"admin"}  │
  │  session:abc123  │  {"user_id":1001,"exp":17200...}  │
  │  cache:product:5 │  <serialized product object>      │
  └─────────────────┴──────────────────────────────────┘

  Operations:  GET key  |  SET key value  |  DELETE key
```

### Redis

Redis (Remote Dictionary Server) is an in-memory data structure store used as a cache, message broker, and primary database for specific workloads.

**Data structures:**

| Structure | Commands | Use Case |
|---|---|---|
| **Strings** | `SET`, `GET`, `INCR`, `EXPIRE` | Caching, counters, rate limiting |
| **Hashes** | `HSET`, `HGET`, `HGETALL` | User profiles, object storage |
| **Lists** | `LPUSH`, `RPUSH`, `LPOP`, `LRANGE` | Message queues, activity feeds |
| **Sets** | `SADD`, `SISMEMBER`, `SINTER` | Tags, unique visitors, set operations |
| **Sorted Sets** | `ZADD`, `ZRANGE`, `ZRANGEBYSCORE` | Leaderboards, priority queues, time-series |
| **Streams** | `XADD`, `XREAD`, `XREADGROUP` | Event streaming, audit logs |
| **HyperLogLog** | `PFADD`, `PFCOUNT` | Cardinality estimation (unique counts) |

```bash
# Rate limiter: allow 100 requests per minute per user
> SET ratelimit:user:1001 0 EX 60 NX
> INCR ratelimit:user:1001
(integer) 1

# Leaderboard: top 3 players
> ZADD leaderboard 1500 "alice" 2300 "bob" 1800 "charlie"
> ZREVRANGE leaderboard 0 2 WITHSCORES
1) "bob"
2) "2300"
3) "charlie"
4) "1800"
5) "alice"
6) "1500"
```

**Persistence options:**

| Option | How It Works | Trade-off |
|---|---|---|
| **RDB (snapshotting)** | Periodic point-in-time snapshot to disk | Fast recovery; potential data loss since last snapshot |
| **AOF (Append Only File)** | Logs every write operation | Minimal data loss; larger files, slower recovery |
| **RDB + AOF** | Both enabled; AOF used for recovery | Best durability; higher disk usage |
| **No persistence** | Pure in-memory | Fastest; all data lost on restart (cache-only) |

**Pub/Sub:**

```bash
# Subscriber (terminal 1)
> SUBSCRIBE order-events

# Publisher (terminal 2)
> PUBLISH order-events '{"order_id":"ord-1","status":"shipped"}'

# The subscriber receives the message in real time.
# Note: messages are fire-and-forget — no persistence or replay.
# For durable messaging, use Redis Streams instead.
```

### DynamoDB

Amazon DynamoDB is a fully managed key-value and document database designed for single-digit-millisecond latency at any scale.

**Key concepts:**

```
DynamoDB Table Structure

  Table: Orders
  ┌──────────────────────────┬────────────────────┬───────────────────┐
  │  Partition Key (PK)      │  Sort Key (SK)     │  Attributes       │
  ├──────────────────────────┼────────────────────┼───────────────────┤
  │  CUSTOMER#cust-42        │  ORDER#2025-06-01  │  total: 149.99    │
  │  CUSTOMER#cust-42        │  ORDER#2025-06-15  │  total: 89.50     │
  │  CUSTOMER#cust-42        │  PROFILE           │  name: "Alice"    │
  │  CUSTOMER#cust-99        │  ORDER#2025-06-10  │  total: 210.00    │
  └──────────────────────────┴────────────────────┴───────────────────┘

  Partition Key  →  Determines which physical partition stores the item
  Sort Key       →  Orders items within a partition; enables range queries
  PK + SK        →  Together form the unique primary key
```

**Global Secondary Index (GSI) vs Local Secondary Index (LSI):**

| Feature | GSI | LSI |
|---|---|---|
| **Partition key** | Different from base table | Same as base table |
| **Sort key** | Any attribute | Different attribute |
| **When to create** | Anytime | At table creation only |
| **Consistency** | Eventually consistent only | Strongly or eventually consistent |
| **Size limit** | No per-partition limit | 10 GB per partition key value |
| **Cost** | Consumes its own read/write capacity | Shares base table capacity |

**Single-table design** is a DynamoDB pattern where all entity types live in one table, using generic PK/SK attributes with prefixes to distinguish entities:

```
Single-Table Design Example

  Table: ECommerceApp
  ┌────────────────────┬─────────────────────┬────────────────────────────┐
  │  PK                │  SK                 │  Attributes                │
  ├────────────────────┼─────────────────────┼────────────────────────────┤
  │  USER#alice        │  PROFILE            │  email, name, created_at   │
  │  USER#alice        │  ORDER#2025-001     │  total, status             │
  │  ORDER#2025-001    │  ITEM#sku-100       │  qty, price                │
  │  PRODUCT#sku-100   │  METADATA           │  name, category, price     │
  └────────────────────┴─────────────────────┴────────────────────────────┘

  Access Patterns:
  • Get user profile       →  PK = "USER#alice", SK = "PROFILE"
  • List user's orders     →  PK = "USER#alice", SK begins_with "ORDER#"
  • Get order items        →  PK = "ORDER#2025-001", SK begins_with "ITEM#"
```

### etcd

etcd is a strongly consistent, distributed key-value store used as the backing store for Kubernetes cluster state. It uses the Raft consensus protocol (all writes go through a leader) and provides linearizable reads and writes. Its Watch API lets clients subscribe to key changes (used by Kubernetes controllers). Best for service discovery, leader election, distributed locks, and configuration — not for general-purpose application data (optimized for small metadata).

---

## Column-Family Databases

Column-family databases (sometimes called "wide-column stores") organize data into rows and column families. Unlike relational databases where every row in a table has the same columns, column-family stores allow each row to have different columns. They are optimized for write-heavy workloads and large-scale distributed data.

```
Column-Family Model

  Row Key: "user:1001"
  ┌──────────────────────────────────────────────────────────┐
  │  Column Family: profile        │  Column Family: activity │
  ├───────────┬────────────────────┼──────────┬──────────────┤
  │  name     │  "Alice"           │  login   │  "2025-06-15"│
  │  email    │  "a@test.com"      │  page    │  "/dashboard"│
  │  role     │  "admin"           │          │              │
  └───────────┴────────────────────┴──────────┴──────────────┘

  Each row can have a different set of columns within a column family.
```

### Cassandra

Apache Cassandra is a distributed, wide-column store designed for high write throughput with no single point of failure. It was originally developed at Facebook and heavily influenced by both Google's Bigtable (data model) and Amazon's Dynamo (distribution model).

**Primary key anatomy:**

```
CREATE TABLE orders_by_customer (
    customer_id UUID,           -- Partition key: determines data placement
    order_date  TIMESTAMP,      -- Clustering column: sorts data within partition
    order_id    UUID,           -- Clustering column
    total       DECIMAL,
    status      TEXT,
    PRIMARY KEY ((customer_id), order_date, order_id)
)                  │                    │
                   │                    └── Clustering columns (sort order)
                   └── Partition key (data distribution)
WITH CLUSTERING ORDER BY (order_date DESC, order_id ASC);
```

**CQL (Cassandra Query Language) examples:**

```sql
-- Insert an order
INSERT INTO orders_by_customer (customer_id, order_date, order_id, total, status)
VALUES (550e8400-e29b-41d4-a716-446655440000, '2025-06-15 10:30:00',
        now(), 149.99, 'pending');

-- Query: all orders for a customer, newest first (efficient: single partition read)
SELECT * FROM orders_by_customer
WHERE customer_id = 550e8400-e29b-41d4-a716-446655440000
ORDER BY order_date DESC
LIMIT 20;

-- Query: orders for a customer within a date range
SELECT * FROM orders_by_customer
WHERE customer_id = 550e8400-e29b-41d4-a716-446655440000
  AND order_date >= '2025-01-01'
  AND order_date <  '2025-07-01';
```

**Data modeling rules:** (1) Model around your queries — design tables to serve specific access patterns. (2) Denormalize aggressively — duplication is expected. (3) Avoid unbounded partitions — keep under ~100 MB. (4) No joins, no subqueries — all data for a query should be in one partition.

**Compaction strategies:**

| Strategy | Description | Best For |
|---|---|---|
| **SizeTieredCompactionStrategy (STCS)** | Merges SSTables of similar size | Write-heavy workloads; general purpose |
| **LeveledCompactionStrategy (LCS)** | Organizes SSTables into levels of exponentially increasing size | Read-heavy workloads; bounded space amplification |
| **TimeWindowCompactionStrategy (TWCS)** | Groups SSTables by time window, compacts within windows | Time-series data; TTL-heavy workloads |
| **UnifiedCompactionStrategy (UCS)** | Adaptive strategy combining aspects of STCS and LCS (Cassandra 5.0+) | General purpose; replaces STCS and LCS |

### HBase

Apache HBase is a column-family store modeled after Google's Bigtable, built on top of the Hadoop ecosystem (HDFS).

- **Strong consistency** — unlike Cassandra's eventual consistency, HBase provides strong consistency per row
- **Region-based sharding** — data is split into regions served by RegionServers
- **Tight HDFS integration** — leverages Hadoop for distributed storage
- **Co-processors** — server-side computation (similar to stored procedures)
- **Use cases:** Large-scale analytics, storing crawler data, message/event storage (often used with MapReduce or Spark)

### ScyllaDB

ScyllaDB is a drop-in replacement for Cassandra written in C++ using a shard-per-core architecture.

- **API-compatible with Cassandra** — uses CQL, same drivers
- **Performance:** Up to 10x throughput vs Cassandra with lower tail latencies
- **Architecture:** Thread-per-core (Seastar framework), no JVM garbage collection pauses
- **Alternator:** DynamoDB-compatible API mode
- **Best for:** Teams already using Cassandra who need better performance without architecture changes

---

## Graph Databases

Graph databases store data as **nodes** (entities) and **edges** (relationships), with properties on both. They excel at traversing connections — the kind of queries that require many recursive joins in a relational database (DDIA Ch. 2).

```
Property Graph Model

  ┌──────────────┐     WORKS_AT      ┌──────────────────┐
  │  Person       │ ──────────────▶  │  Company          │
  │  name: Alice  │   since: 2020    │  name: Acme Corp  │
  │  age: 30      │                  │  industry: Tech   │
  └──────────────┘                  └──────────────────┘
        │                                    ▲
        │  FRIENDS_WITH                      │
        │  since: 2018                       │  WORKS_AT
        ▼                                    │  since: 2019
  ┌──────────────┐                          │
  │  Person       │ ─────────────────────────┘
  │  name: Bob    │
  │  age: 28      │
  └──────────────┘

  Nodes have labels (Person, Company) and properties (key-value pairs).
  Edges have types (WORKS_AT, FRIENDS_WITH) and properties.
  Edges are always directed but can be traversed in both directions.
```

### Property Graph Model vs RDF

| Feature | Property Graph | RDF (Resource Description Framework) |
|---|---|---|
| **Basic unit** | Nodes and edges with properties | Subject-predicate-object triples |
| **Schema** | Flexible; labels and property keys are freeform | Ontologies (OWL, RDFS) define classes and relationships |
| **Query language** | Cypher (Neo4j), Gremlin (TinkerPop) | SPARQL |
| **Identity** | Internal IDs + properties | URIs (globally unique identifiers) |
| **Strengths** | Intuitive for application data modeling | Data integration across organizations, linked data |
| **Databases** | Neo4j, Amazon Neptune, ArangoDB | Amazon Neptune, Blazegraph, Virtuoso |

### Neo4j and Cypher

Neo4j is the most widely used graph database. Cypher is its declarative query language, designed to visually represent graph patterns.

```cypher
// Create nodes and relationships
CREATE (alice:Person {name: 'Alice', age: 30})
CREATE (bob:Person {name: 'Bob', age: 28})
CREATE (acme:Company {name: 'Acme Corp', industry: 'Tech'})
CREATE (alice)-[:WORKS_AT {since: 2020}]->(acme)
CREATE (bob)-[:WORKS_AT {since: 2019}]->(acme)
CREATE (alice)-[:FRIENDS_WITH {since: 2018}]->(bob);

// Find all of Alice's coworkers (people who work at the same company)
MATCH (alice:Person {name: 'Alice'})-[:WORKS_AT]->(company)<-[:WORKS_AT]-(coworker)
RETURN coworker.name, company.name;

// Shortest path between two people through FRIENDS_WITH relationships
MATCH path = shortestPath(
  (a:Person {name: 'Alice'})-[:FRIENDS_WITH*..6]-(b:Person {name: 'Diana'})
)
RETURN path, length(path);

// Recommendation: friends-of-friends who Alice doesn't know yet
MATCH (alice:Person {name: 'Alice'})-[:FRIENDS_WITH]->(friend)-[:FRIENDS_WITH]->(fof)
WHERE NOT (alice)-[:FRIENDS_WITH]->(fof) AND alice <> fof
RETURN fof.name, COUNT(friend) AS mutual_friends
ORDER BY mutual_friends DESC
LIMIT 5;
```

### Amazon Neptune

Amazon Neptune is a fully managed graph database supporting both property graph (Gremlin/TinkerPop) and RDF (SPARQL). It offers up to 128 TB storage with 6 copies across 3 AZs, up to 15 read replicas, and a serverless mode that auto-scales compute. Common use cases include knowledge graphs, fraud detection, identity graphs, and network topology.

### When Graphs Outperform Relational Joins

As DDIA Chapter 2 explains, graph queries become dramatically more efficient than relational queries when the depth of traversal is unknown or large:

```
"Find all people within 4 degrees of connection from Alice"

Relational (SQL):                          Graph (Cypher):

WITH RECURSIVE connections AS (            MATCH (a:Person {name:'Alice'})
  SELECT friend_id, 1 AS depth                   -[:FRIENDS_WITH*1..4]-
  FROM friendships                                (connected)
  WHERE person_id = 'alice'                RETURN DISTINCT connected.name;
  UNION ALL
  SELECT f.friend_id, c.depth + 1
  FROM friendships f                        → Index-free adjacency;
  JOIN connections c                          constant time per edge
    ON c.friend_id = f.person_id             traversal regardless
  WHERE c.depth < 4                          of total dataset size
)
SELECT DISTINCT friend_id
FROM connections;

→ Multiple self-joins; performance
  degrades exponentially with depth
```

**Use a graph database when:**
- You frequently traverse relationships of variable or unknown depth
- Your queries are about paths, connectivity, or patterns
- Many-to-many relationships dominate your data model
- You need real-time recommendation or fraud-detection patterns

---

## Time-Series Databases

Time-series databases (TSDBs) are optimized for storing and querying timestamped data — measurements, events, and metrics that arrive in time order.

### Data Characteristics

Time-series data has unique properties that general-purpose databases handle poorly:

| Characteristic | Description | Implication |
|---|---|---|
| **Append-mostly** | New data is inserted at the current timestamp; old data is rarely updated | Write path can be optimized for sequential appends |
| **Time-ordered** | Queries almost always involve a time range | Storage sorted by time enables efficient range scans |
| **High cardinality** | Many unique series (e.g., one per server × metric) | Need efficient indexing of series labels/tags |
| **Downsampling** | Older data is acceptable at lower resolution | Built-in retention and rollup policies |
| **High write volume** | Millions of data points per second from IoT, infrastructure | Requires batched writes and compression |

```
Time-Series Data Model

  Series: cpu.usage{host="web-01", region="us-east"}

  ┌────────────────────┬─────────┐
  │  Timestamp          │  Value  │
  ├────────────────────┼─────────┤
  │  2025-06-15 10:00  │  42.5   │
  │  2025-06-15 10:01  │  45.2   │
  │  2025-06-15 10:02  │  38.7   │
  │  2025-06-15 10:03  │  41.0   │
  │  2025-06-15 10:04  │  55.1   │
  │  ...               │  ...    │
  └────────────────────┴─────────┘

  Tags (labels) identify the series.
  Values are numeric measurements at each point in time.
```

### InfluxDB

InfluxDB is a purpose-built time-series database with its own query language (Flux).

```
-- InfluxDB line protocol (write format)
cpu,host=web-01,region=us-east usage=42.5,system=12.3 1718445600000000000

-- Flux query: average CPU usage per host over 5-minute windows
from(bucket: "metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "cpu" and r._field == "usage")
  |> aggregateWindow(every: 5m, fn: mean)
  |> group(columns: ["host"])
```

Key concepts: **Measurement** (like a table), **Tags** (indexed metadata for filtering), **Fields** (the actual values, not indexed), and **Buckets** (database with a retention policy).

### TimescaleDB

TimescaleDB is a time-series extension for PostgreSQL. It provides time-series optimizations while maintaining full SQL compatibility.

```sql
-- Create a hypertable (time-partitioned table)
CREATE TABLE metrics (
    time        TIMESTAMPTZ NOT NULL,
    host        TEXT        NOT NULL,
    cpu_usage   DOUBLE PRECISION,
    mem_usage   DOUBLE PRECISION
);
SELECT create_hypertable('metrics', by_range('time'));

-- Query: average CPU usage per host in 5-minute buckets
SELECT
    time_bucket('5 minutes', time) AS bucket,
    host,
    AVG(cpu_usage) AS avg_cpu
FROM metrics
WHERE time > NOW() - INTERVAL '1 hour'
GROUP BY bucket, host
ORDER BY bucket DESC;
```

**Advantages:** Full SQL support (JOINs, CTEs, window functions with time-series data), compatible with the PostgreSQL ecosystem, 90%+ compression on time-series data, and automatic partitioning via hypertables.

### Prometheus TSDB

Prometheus is a monitoring-focused TSDB that uses a **pull-based model** — it scrapes HTTP endpoints (`/metrics`) on a schedule. Data is identified by metric name and key-value labels. It uses PromQL for queries and alerting.

```
# PromQL examples

# CPU usage rate per instance over 5 minutes
rate(node_cpu_seconds_total{mode="idle"}[5m])

# 99th percentile HTTP request duration
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[1h]))

# Total requests per second by status code
sum(rate(http_requests_total[5m])) by (status_code)
```

### Retention Policies

All time-series databases support automatic data expiration and downsampling:

| Database | Mechanism | Example |
|---|---|---|
| **InfluxDB** | Bucket retention period | Bucket with 30-day retention; data auto-deleted after 30 days |
| **TimescaleDB** | `drop_chunks` policy or `add_retention_policy` | `SELECT add_retention_policy('metrics', INTERVAL '90 days');` |
| **Prometheus** | `--storage.tsdb.retention.time` flag | `--storage.tsdb.retention.time=15d` |

---

## Search Engines

Search engines like Elasticsearch and OpenSearch are specialized for full-text search, log analytics, and relevance-ranked queries. While they are technically document stores, their primary purpose is indexing and searching text.

### Inverted Indexes

The core data structure behind full-text search is the **inverted index** — a mapping from terms to the documents that contain them.

```
Documents:
  doc1: "The quick brown fox jumps over the lazy dog"
  doc2: "The quick rabbit jumps over the fence"
  doc3: "A lazy dog sleeps in the sun"

Inverted Index:
  ┌───────────┬─────────────────┐
  │  Term      │  Document IDs   │
  ├───────────┼─────────────────┤
  │  a         │  [doc3]         │
  │  brown     │  [doc1]         │
  │  dog       │  [doc1, doc3]   │
  │  fence     │  [doc2]         │
  │  fox       │  [doc1]         │
  │  in        │  [doc3]         │
  │  jumps     │  [doc1, doc2]   │
  │  lazy      │  [doc1, doc3]   │
  │  over      │  [doc1, doc2]   │
  │  quick     │  [doc1, doc2]   │
  │  rabbit    │  [doc2]         │
  │  sleeps    │  [doc3]         │
  │  sun       │  [doc3]         │
  │  the       │  [doc1, doc2, doc3] │
  └───────────┴─────────────────┘

  Query: "lazy dog" → intersect([doc1, doc3], [doc1, doc3]) → [doc1, doc3]
```

In practice, inverted indexes also store term frequency, position data, and field-level information to support relevance scoring and phrase queries.

### Elasticsearch and OpenSearch

Elasticsearch (and its open-source fork OpenSearch) are distributed search and analytics engines built on Apache Lucene.

An Elasticsearch cluster distributes each index across **shards** (each shard is a self-contained Lucene index). Primary shards handle writes; replica shards provide read scaling and fault tolerance. Queries are scattered to all relevant shards and results are gathered.

```json
// Index a document
PUT /products/_doc/1
{
  "name": "Wireless Keyboard",
  "description": "Ergonomic bluetooth keyboard with backlit keys",
  "category": "electronics",
  "price": 79.99,
  "in_stock": true
}

// Full-text search with filtering and boosting
GET /products/_search
{
  "query": {
    "bool": {
      "must": {
        "multi_match": {
          "query": "ergonomic keyboard",
          "fields": ["name^3", "description"]
        }
      },
      "filter": [
        { "term": { "in_stock": true } },
        { "range": { "price": { "lte": 100 } } }
      ]
    }
  },
  "highlight": {
    "fields": { "description": {} }
  },
  "aggs": {
    "by_category": {
      "terms": { "field": "category.keyword" }
    },
    "avg_price": {
      "avg": { "field": "price" }
    }
  }
}
```

**Key Elasticsearch concepts:**

| Concept | Description |
|---|---|
| **Index** | A collection of documents (like a database table) |
| **Shard** | A subset of an index; each shard is a Lucene index |
| **Mapping** | Schema definition for fields (text, keyword, integer, date, etc.) |
| **Analyzer** | Pipeline of character filters, tokenizer, and token filters for text processing |
| **text vs keyword** | `text` fields are analyzed (tokenized) for full-text search; `keyword` fields are exact-match only |

### Relevance Scoring

Elasticsearch uses **BM25** (Best Matching 25) as its default relevance scoring algorithm, which improves on TF-IDF.

| Factor | Description | Effect |
|---|---|---|
| **Term Frequency (TF)** | How often the term appears in the document | More occurrences → higher score (with saturation) |
| **Inverse Document Frequency (IDF)** | How rare the term is across all documents | Rarer terms are worth more |
| **Field length** | How long the field is | Shorter fields with the term score higher |
| **Boosting** | Explicit weight on specific fields (e.g., `name^3`) | Prioritize matches in important fields |

BM25 uses two tuning parameters: `k1` (default 1.2) controls term frequency saturation, and `b` (default 0.75) controls field-length normalization. In practice, the defaults work well for most use cases.

---

## Choosing the Right NoSQL Database

### Decision Framework

Start by identifying your **primary access pattern**, then narrow down by operational requirements:

```
Decision Tree

  What is your primary access pattern?
  │
  ├─ Simple lookups by key, very high throughput
  │  └─ Key-Value: Redis (in-memory) / DynamoDB (managed) / etcd (coordination)
  │
  ├─ Flexible schema, nested objects, varied queries
  │  └─ Document: MongoDB (general) / CouchDB (offline sync) / Firestore (serverless)
  │
  ├─ Write-heavy, wide rows, known query patterns
  │  └─ Column-Family: Cassandra (multi-DC) / HBase (Hadoop) / ScyllaDB (performance)
  │
  ├─ Traversing relationships, path finding, patterns
  │  └─ Graph: Neo4j (Cypher, community) / Neptune (managed, multi-model)
  │
  ├─ Timestamped metrics, IoT, monitoring
  │  └─ Time-Series: InfluxDB (purpose-built) / TimescaleDB (SQL) / Prometheus (monitoring)
  │
  └─ Full-text search, log analytics, relevance
     └─ Search: Elasticsearch / OpenSearch
```

### Comparison Table

| Dimension | Document (MongoDB) | Key-Value (Redis) | Key-Value (DynamoDB) | Column-Family (Cassandra) | Graph (Neo4j) | Time-Series (TimescaleDB) | Search (Elasticsearch) |
|---|---|---|---|---|---|---|---|
| **Data model** | JSON/BSON documents | Strings, hashes, lists, sets, sorted sets | Items with attributes | Wide rows, column families | Nodes and edges | Timestamped rows | JSON documents (indexed) |
| **Query language** | MQL, aggregation pipeline | Commands (GET, SET, etc.) | PartiQL, API | CQL (SQL-like) | Cypher | SQL | Query DSL (JSON) |
| **Scaling** | Horizontal (sharding) | Vertical + Cluster mode | Horizontal (automatic) | Horizontal (peer-to-peer) | Vertical (cluster in Enterprise) | Horizontal (hypertables) | Horizontal (sharding) |
| **Consistency** | Configurable (read concern) | Strong (single node) | Configurable (strong/eventual) | Tunable (per query) | ACID per transaction | ACID (PostgreSQL) | Eventually consistent |
| **Transactions** | Multi-document ACID | MULTI/EXEC (limited) | TransactWriteItems (25 items) | Lightweight transactions (CAS) | Full ACID | Full ACID | No cross-document |
| **Replication** | Replica sets (automatic failover) | Primary-replica | Multi-region, global tables | Multi-DC, peer-to-peer | Causal clustering (Enterprise) | PostgreSQL streaming | Primary-replica shards |
| **Best for** | Flexible schema, rapid iteration | Caching, sessions, real-time | Serverless, predictable latency | High write throughput, IoT | Relationship traversals | Metrics with SQL queries | Full-text search, logs |
| **Managed options** | Atlas | ElastiCache, MemoryDB | AWS native | Astra DB, Amazon Keyspaces | AuraDB | Timescale Cloud | Elastic Cloud, Amazon OpenSearch |
| **License** | SSPL | BSD-3 / RSALv2 | Proprietary | Apache 2.0 | GPL / Commercial | Apache 2.0 | SSPL / Apache 2.0 (OpenSearch) |

> *"In general, the right choice depends on how your application uses data, not on which database is the most popular."* — a paraphrase of the principle in DDIA Chapter 2

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial NoSQL databases documentation |
