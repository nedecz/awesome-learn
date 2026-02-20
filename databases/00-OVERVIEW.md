# Database Fundamentals

## Table of Contents

1. [Overview](#overview)
2. [What is a Database?](#what-is-a-database)
3. [ACID vs BASE](#acid-vs-base)
4. [CAP Theorem and PACELC](#cap-theorem-and-pacelc)
5. [Storage Engines](#storage-engines)
   - [B-Trees](#b-trees)
   - [LSM-Trees and SSTables](#lsm-trees-and-sstables)
   - [Column-Oriented Storage](#column-oriented-storage)
6. [Data Models](#data-models)
7. [OLTP vs OLAP](#oltp-vs-olap)
8. [Reliability, Scalability, Maintainability](#reliability-scalability-maintainability)
9. [Database Landscape](#database-landscape)
10. [Prerequisites](#prerequisites)
11. [Next Steps](#next-steps)
12. [Version History](#version-history)

---

## Overview

This document introduces the foundational concepts every engineer needs to understand before working with databases at any scale. It draws heavily from the ideas in *Designing Data-Intensive Applications* (DDIA) by Martin Kleppmann, particularly Chapters 1 and 3.

### Target Audience

- **Backend developers** building services that read and write data
- **Data engineers** designing pipelines and analytical systems
- **SREs / DevOps engineers** operating databases in production
- **Architects** choosing data stores for new systems

### Scope

- Core properties of databases: ACID and BASE
- The CAP theorem and its practical implications (PACELC)
- How storage engines work under the hood (B-Trees, LSM-Trees)
- Data models: relational, document, graph, key-value, column-family
- OLTP vs OLAP workloads
- The three pillars of data-intensive applications: reliability, scalability, maintainability

---

## What is a Database?

A database is a system that stores, organizes, and provides access to data. At its core, every database must solve two problems:

1. **How to store data on disk** — the storage engine
2. **How to find data efficiently** — the query engine and indexes

Beyond these basics, databases provide varying levels of guarantees around:

- **Durability** — data survives crashes and power failures
- **Consistency** — the data is always in a valid state
- **Concurrency** — multiple readers and writers can operate simultaneously without corruption
- **Availability** — the system remains operational despite failures

The choice of database depends on which of these guarantees matter most for your workload, and what trade-offs you are willing to accept.

---

## ACID vs BASE

These are two contrasting models for database transaction guarantees. Understanding the trade-offs is fundamental to choosing the right data store.

### ACID (Atomicity, Consistency, Isolation, Durability)

ACID is the traditional guarantee provided by relational databases. Each property reinforces the others to provide strong correctness guarantees.

| Property | Description | What Happens Without It |
|---|---|---|
| **Atomicity** | A transaction either fully completes or fully rolls back. No partial writes. | Partial updates leave the database in an inconsistent state (e.g., money debited but not credited) |
| **Consistency** | A transaction moves the database from one valid state to another, respecting all constraints and invariants. | Foreign key violations, check constraint violations, invalid business states |
| **Isolation** | Concurrent transactions behave as if they executed sequentially. The level of isolation varies by implementation. | Dirty reads, lost updates, write skew, phantom reads |
| **Durability** | Once a transaction commits, the data is permanently stored, even if the system crashes. | Committed data lost on crash, requiring manual recovery |

### Transaction Isolation Levels

Isolation is the most nuanced ACID property. Databases implement different levels (weakest to strongest):

```
Read Uncommitted  → Dirty reads possible. Rarely used.
Read Committed    → No dirty reads. Default in PostgreSQL.
Repeatable Read   → Snapshot isolation. Default in MySQL InnoDB.
Serializable      → Full isolation. Most expensive.

                   Anomalies prevented:
                   ┌─────────────┬───────────┬──────────────┬──────────┬────────────┐
                   │ Level       │ Dirty     │ Non-repeat.  │ Phantom  │ Write      │
                   │             │ Read      │ Read         │ Read     │ Skew       │
                   ├─────────────┼───────────┼──────────────┼──────────┼────────────┤
                   │ Read Uncomm │ ✗         │ ✗            │ ✗        │ ✗          │
                   │ Read Commit │ ✓         │ ✗            │ ✗        │ ✗          │
                   │ Repeat Read │ ✓         │ ✓            │ ✗/✓*    │ ✗          │
                   │ Serializable│ ✓         │ ✓            │ ✓        │ ✓          │
                   └─────────────┴───────────┴──────────────┴──────────┴────────────┘
                   * Depends on implementation (PostgreSQL RR prevents phantoms via MVCC)
```

### BASE (Basically Available, Soft state, Eventually consistent)

BASE is the model adopted by many distributed NoSQL databases that prioritize availability and partition tolerance over immediate consistency.

| Property | Description |
|---|---|
| **Basically Available** | The system guarantees availability in the CAP sense — it will always return a response, even if the data is stale |
| **Soft state** | The state of the system may change over time, even without input, due to eventual consistency propagation |
| **Eventually consistent** | Given enough time without new updates, all replicas will converge to the same value |

### ACID vs BASE Comparison

```
    ACID                              BASE
    ┌──────────────────────┐          ┌──────────────────────┐
    │ Strong consistency   │          │ Eventual consistency  │
    │ Pessimistic          │          │ Optimistic            │
    │ Complex transactions │          │ Simple operations     │
    │ Vertical scaling     │          │ Horizontal scaling    │
    │ Lower availability   │          │ Higher availability   │
    │ during partitions    │          │ during partitions     │
    └──────────────────────┘          └──────────────────────┘

    Use when:                         Use when:
    • Correctness > availability      • Availability > consistency
    • Financial transactions          • Social media feeds
    • Inventory management            • Analytics, metrics
    • Booking / reservation systems   • Session stores, caches
```

---

## CAP Theorem and PACELC

### CAP Theorem

The CAP theorem (Eric Brewer, 2000) states that a distributed data store can provide at most **two out of three** guarantees simultaneously:

- **Consistency (C)** — Every read receives the most recent write or an error
- **Availability (A)** — Every request receives a non-error response (but not necessarily the most recent data)
- **Partition Tolerance (P)** — The system continues to operate despite network partitions between nodes

Since network partitions are unavoidable in distributed systems, the practical choice is between **CP** (consistency + partition tolerance) and **AP** (availability + partition tolerance).

```
                         Consistency (C)
                              ▲
                             / \
                            /   \
                           / CP  \
                          /       \
                         /    ✓    \
                        /           \
                       /─────────────\
                      /               \
                     /    You must     \
                    /     choose        \
                   /                     \
                  /───────────────────────\
                 /  AP               CA*   \
                /    ✓                ✗     \
               ▼──────────────────────────▶ ▼
          Availability (A)         Partition Tolerance (P)

    * CA is not achievable in distributed systems — partitions will occur
```

| Category | Behavior During Partition | Examples |
|---|---|---|
| **CP** | Rejects writes or reads to maintain consistency. System may become unavailable. | PostgreSQL (single-node), etcd, ZooKeeper, HBase, MongoDB (default config) |
| **AP** | Accepts reads and writes, but responses may return stale data. | Cassandra, DynamoDB, CouchDB, Riak |

### PACELC

Daniel Abadi extended CAP with the **PACELC** model: during a **P**artition, choose **A** or **C**; **E**lse (normal operation), choose **L**atency or **C**onsistency.

This is more practical because most of the time there is no partition, and the latency vs consistency trade-off dominates.

| System | Partition (P): A or C? | Else (E): L or C? |
|---|---|---|
| PostgreSQL (single) | N/A (not distributed) | C |
| Cassandra | A | L (tunable) |
| DynamoDB | A | L |
| MongoDB | C (default) | C (default) |
| CockroachDB | C | C |
| Spanner | C | C (with TrueTime) |

---

## Storage Engines

Understanding how databases store and retrieve data on disk is fundamental. The two dominant approaches are B-Trees and LSM-Trees. This directly maps to DDIA Chapter 3.

### B-Trees

B-Trees are the most widely used indexing structure in relational databases. They are balanced, page-oriented tree structures.

```
                            [30 | 70]                    ← Root page
                           /    |    \
                          /     |     \
                    [10|20]   [50|60]   [80|90]          ← Internal pages
                    / | \     / | \     / | \
                   /  |  \   /  |  \   /  |  \
                  ▼   ▼   ▼ ▼   ▼   ▼ ▼   ▼   ▼         ← Leaf pages
                [data pages with actual rows/pointers]

    Key properties:
    • Fixed-size pages (typically 4–16 KB)
    • Balanced — all leaf pages are at the same depth
    • In-place updates — overwrites pages on disk
    • O(log n) lookups, inserts, and deletes
    • Write-ahead log (WAL) for crash recovery
```

**Characteristics:**
- **Read-optimized** — lookups traverse O(log n) pages
- **Write amplification** — each write may update multiple pages and the WAL
- **Fragmentation** — pages may become partially filled over time
- **Used by:** PostgreSQL, MySQL (InnoDB), Oracle, SQL Server

### LSM-Trees and SSTables

Log-Structured Merge Trees write data sequentially to an in-memory buffer (memtable), then flush sorted segments (SSTables) to disk. Background compaction merges segments.

```
    Write path:
    ┌────────┐    ┌──────────────┐    ┌──────────────────────┐
    │ Write  │───▶│  Memtable    │───▶│  Flush to SSTable    │
    │        │    │  (in-memory, │    │  (sorted, immutable  │
    │        │    │   sorted)    │    │   segment on disk)   │
    └────────┘    └──────────────┘    └──────────────────────┘
                                              │
                                              ▼
                                     ┌──────────────────┐
                                     │  Compaction       │
                                     │  (merge SSTables, │
                                     │   remove tombstones│
                                     │   and duplicates)  │
                                     └──────────────────┘

    Read path:
    ┌────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐
    │ Read   │───▶│ Memtable  │───▶│ SSTable 1 │───▶│ SSTable 2 │──▶ ...
    │        │    │ (newest)  │    │ (newer)   │    │ (older)   │
    └────────┘    └───────────┘    └───────────┘    └───────────┘
                  (check bloom filters to skip SSTables)
```

**Characteristics:**
- **Write-optimized** — sequential writes are fast; no random I/O for writes
- **Read amplification** — may need to check multiple SSTables
- **Space amplification** — stale data exists until compaction
- **Bloom filters** mitigate read amplification for non-existent keys
- **Used by:** Cassandra, RocksDB, LevelDB, ScyllaDB, HBase

### B-Tree vs LSM-Tree Comparison

| Dimension | B-Tree | LSM-Tree |
|---|---|---|
| **Write performance** | Moderate (random I/O, WAL) | High (sequential, append-only) |
| **Read performance** | High (single tree traversal) | Moderate (multiple SSTables) |
| **Write amplification** | Moderate | Lower for writes, higher for compaction |
| **Space efficiency** | Moderate (fragmentation) | Good after compaction |
| **Predictable latency** | Yes (consistent read/write times) | Less predictable (compaction spikes) |
| **Best for** | OLTP, read-heavy workloads | Write-heavy, time-series, logs |

### Column-Oriented Storage

Column-oriented storage stores data by column rather than by row. This is highly efficient for analytical queries that read a subset of columns across many rows.

```
    Row-oriented (OLTP):                Column-oriented (OLAP):
    ┌────┬──────┬──────┬───────┐        ┌────────────────────────┐
    │ id │ name │ city │ sales │        │ id:   1, 2, 3, 4, ... │
    ├────┼──────┼──────┼───────┤        ├────────────────────────┤
    │ 1  │ Alice│ NYC  │ 100   │        │ name: Alice, Bob, ...  │
    │ 2  │ Bob  │ LA   │ 200   │        ├────────────────────────┤
    │ 3  │ Carol│ NYC  │ 150   │        │ city: NYC, LA, NYC,... │
    └────┴──────┴──────┴───────┘        ├────────────────────────┤
                                        │ sales: 100, 200, 150,..│
    Good for: SELECT * FROM t           └────────────────────────┘
    WHERE id = 1                        Good for: SELECT city,
                                        SUM(sales) GROUP BY city
```

**Used by:** ClickHouse, Apache Parquet, Amazon Redshift, Google BigQuery, Apache Cassandra (partially)

---

## Data Models

The choice of data model is one of the most important decisions in application design (DDIA Chapter 2).

### Comparison

| Model | Shape | Query Language | Best For | Examples |
|---|---|---|---|---|
| **Relational** | Tables with rows and columns | SQL | Structured data with relationships, transactions | PostgreSQL, MySQL, SQL Server |
| **Document** | JSON/BSON documents | MongoDB Query Language, N1QL | Semi-structured data, nested objects, rapid iteration | MongoDB, CouchDB, Firestore |
| **Key-Value** | Simple key → value pairs | GET/SET API | Caching, sessions, simple lookups | Redis, DynamoDB, etcd |
| **Column-Family** | Rows with dynamic columns | CQL, HBase API | Time-series, IoT, high-write throughput | Cassandra, HBase, ScyllaDB |
| **Graph** | Nodes and edges (relationships) | Cypher, Gremlin, SPARQL | Social networks, fraud detection, knowledge graphs | Neo4j, Amazon Neptune, JanusGraph |

### When to Use Each Model

```
Start with Relational if:
  ✓ Data has clear relationships (foreign keys, joins)
  ✓ You need ACID transactions
  ✓ Query patterns are varied and evolving
  ✓ Data consistency is critical

Consider Document if:
  ✓ Data is naturally hierarchical / nested
  ✓ Schema changes frequently (schema-on-read)
  ✓ Each document is relatively self-contained
  ✓ You rarely need cross-document joins

Consider Key-Value if:
  ✓ Access pattern is simple: lookup by key
  ✓ You need sub-millisecond latency
  ✓ Data is relatively small per key

Consider Column-Family if:
  ✓ Write throughput is extremely high
  ✓ Data is time-series or event-based
  ✓ Queries are predictable (designed around partition keys)

Consider Graph if:
  ✓ Relationships between entities are as important as the entities themselves
  ✓ Queries involve traversing many levels of connections
  ✓ Pattern matching across relationships is a core requirement
```

---

## OLTP vs OLAP

Understanding the difference between transactional and analytical workloads is critical for choosing the right database and storage engine.

| Dimension | OLTP (Online Transaction Processing) | OLAP (Online Analytical Processing) |
|---|---|---|
| **Primary use** | Serve application requests | Business intelligence, reporting |
| **Read pattern** | Small number of rows per query, by key | Aggregates over large numbers of rows |
| **Write pattern** | Random-access, low-latency inserts/updates | Bulk imports (ETL) or event streams |
| **Data volume** | GB to low TB | TB to PB |
| **Users** | End-users via application | Analysts, data scientists |
| **Query style** | Simple lookups and updates | Complex aggregations, joins, window functions |
| **Schema** | Normalized (3NF) | Denormalized (star/snowflake schema) |
| **Storage engine** | Row-oriented (B-Tree) | Column-oriented |
| **Examples** | PostgreSQL, MySQL, DynamoDB | ClickHouse, Redshift, BigQuery, Snowflake |

```
                OLTP                               OLAP
    ┌──────────────────────────┐      ┌──────────────────────────┐
    │  Application Database    │      │  Data Warehouse          │
    │                          │      │                          │
    │  INSERT INTO orders      │      │  SELECT region,          │
    │    (user_id, amount)     │ ───▶ │    SUM(amount),          │
    │    VALUES (42, 99.99);   │ ETL  │    COUNT(*)              │
    │                          │      │  FROM orders             │
    │  SELECT * FROM orders    │      │  WHERE date > '2025-01'  │
    │  WHERE id = 12345;       │      │  GROUP BY region;        │
    └──────────────────────────┘      └──────────────────────────┘
    Row-oriented, normalized          Column-oriented, denormalized
    Millisecond responses             Seconds to minutes, large scans
```

---

## Reliability, Scalability, Maintainability

These three properties — from DDIA Chapter 1 — are the foundation for thinking about data systems.

### Reliability

> The system continues to work correctly, even when things go wrong.

Things that can go wrong (faults):
- **Hardware faults** — disk failure, memory errors, network outages
- **Software faults** — bugs, resource leaks, cascading failures
- **Human faults** — configuration errors, bad deployments

**Strategies for reliability:**
- Redundancy: replication, RAID, multi-AZ deployments
- Testing: unit tests, integration tests, chaos engineering
- Monitoring and alerting: detect faults before users notice
- Rollback capability: blue-green deployments, feature flags

### Scalability

> The system can handle growth in data volume, traffic, or complexity.

**Scaling strategies:**

| Strategy | Description | Pros | Cons |
|---|---|---|---|
| **Vertical (scale up)** | Bigger machine (more CPU, RAM, SSD) | Simple, no code changes | Hardware limits, single point of failure |
| **Horizontal (scale out)** | More machines (sharding, replication) | Near-linear scaling | Complexity: distributed transactions, rebalancing |
| **Read replicas** | Copies of the database for read traffic | Offloads read load | Replication lag, eventual consistency |
| **Caching** | In-memory layer (Redis, Memcached) | Sub-millisecond reads | Cache invalidation complexity, stale data |

### Maintainability

> The system is easy to operate, understand, and evolve over time.

Three design principles for maintainability:
- **Operability** — make it easy for operations teams to keep the system running
- **Simplicity** — manage complexity through abstraction and clean interfaces
- **Evolvability** — make it easy to change the system as requirements change

---

## Database Landscape

| Database | Type | License | Primary Use Case |
|---|---|---|---|
| **PostgreSQL** | Relational | PostgreSQL License (OSS) | General-purpose OLTP, JSON support, extensibility |
| **MySQL** | Relational | GPL v2 (OSS) / Commercial | Web applications, read-heavy workloads |
| **SQL Server** | Relational | Commercial | Enterprise .NET applications |
| **Oracle** | Relational | Commercial | Enterprise, financial systems |
| **MongoDB** | Document | SSPL / Commercial | Flexible schema, rapid development |
| **Redis** | Key-Value / Multi-model | BSD 3-Clause | Caching, sessions, pub/sub, leaderboards |
| **Cassandra** | Column-Family | Apache 2.0 | High-write throughput, time-series, IoT |
| **DynamoDB** | Key-Value / Document | Commercial (AWS) | Serverless, single-digit millisecond latency |
| **Neo4j** | Graph | GPL v3 (Community) / Commercial | Social networks, fraud detection, knowledge graphs |
| **ClickHouse** | Column-Oriented (OLAP) | Apache 2.0 | Real-time analytics, log analysis |
| **CockroachDB** | Distributed SQL | BSL / Commercial | Globally distributed ACID transactions |
| **Elasticsearch** | Search / Document | Elastic License 2.0 | Full-text search, log analytics |
| **Snowflake** | Cloud Data Warehouse | Commercial | Analytics, data sharing |
| **SQLite** | Embedded Relational | Public Domain | Mobile apps, embedded systems, testing |

---

## Prerequisites

### Required Knowledge

| Topic | Why It Matters |
|---|---|
| **SQL basics** | The lingua franca of databases; needed for relational databases and many analytical tools |
| **Basic data structures** | Hash tables, trees, linked lists — understanding these helps you understand indexes |
| **Linux command line** | Running databases locally, reading logs, using CLI tools (psql, mysql, redis-cli) |
| **Docker** | Running database instances locally in containers |
| **Networking basics** | TCP connections, ports, DNS — databases communicate over the network |

### Recommended Tools

```bash
# PostgreSQL client
brew install postgresql    # macOS
sudo apt-get install postgresql-client  # Ubuntu

# MySQL client
brew install mysql-client  # macOS
sudo apt-get install mysql-client  # Ubuntu

# Redis CLI
brew install redis         # macOS
sudo apt-get install redis-tools  # Ubuntu

# MongoDB Shell
brew install mongosh       # macOS
npm install -g mongosh     # via npm

# Docker (for running databases locally)
brew install --cask docker  # macOS
# See Docker topic for Linux installation

# pgcli / mycli (enhanced interactive CLI)
pip install pgcli mycli

# DBeaver (GUI database tool)
brew install --cask dbeaver-community  # macOS
```

### Quick Docker Setup

```bash
# PostgreSQL
docker run -d --name pg -e POSTGRES_PASSWORD=secret -p 5432:5432 postgres:17

# MySQL
docker run -d --name mysql -e MYSQL_ROOT_PASSWORD=secret -p 3306:3306 mysql:8

# Redis
docker run -d --name redis -p 6379:6379 redis:7

# MongoDB
docker run -d --name mongo -p 27017:27017 mongo:7

# Verify
docker ps
psql -h localhost -U postgres -c "SELECT version();"
```

---

## Next Steps

| File | Topic | Description |
|---|---|---|
| [01-RELATIONAL-DATABASES.md](01-RELATIONAL-DATABASES.md) | Relational Databases | SQL, normalization, indexing, PostgreSQL, MySQL |
| [02-NOSQL-DATABASES.md](02-NOSQL-DATABASES.md) | NoSQL Databases | Document, key-value, column-family, graph |
| [03-DATA-MODELING.md](03-DATA-MODELING.md) | Data Modeling | Schema design, entity relationships, denormalization |
| [04-QUERY-OPTIMIZATION.md](04-QUERY-OPTIMIZATION.md) | Query Optimization | Execution plans, indexing strategies, tuning |
| [05-REPLICATION-AND-SHARDING.md](05-REPLICATION-AND-SHARDING.md) | Replication & Sharding | Replicas, partitioning, consistency |
| [06-MIGRATIONS.md](06-MIGRATIONS.md) | Migrations | Schema evolution, zero-downtime migrations |
| [07-CACHING.md](07-CACHING.md) | Caching | Redis, Memcached, cache strategies |
| [08-CONNECTION-MANAGEMENT.md](08-CONNECTION-MANAGEMENT.md) | Connection Management | Pooling, proxies, timeouts |
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Best Practices | Production patterns, backup, monitoring |
| [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md) | Anti-Patterns | Common mistakes and how to avoid them |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Learning Path | Structured learning guide with exercises |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial database fundamentals documentation |
