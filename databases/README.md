# Databases Learning Resources

A comprehensive guide to database fundamentals, data modeling, storage engines, replication, partitioning, and operational best practices — grounded in concepts from *Designing Data-Intensive Applications* (Martin Kleppmann) and real-world production experience.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | Database fundamentals, CAP theorem, ACID vs BASE, storage engines | **Start here** |
| [01-RELATIONAL-DATABASES](01-RELATIONAL-DATABASES.md) | SQL, normalization, indexing, PostgreSQL, MySQL | When designing or querying relational schemas |
| [02-NOSQL-DATABASES](02-NOSQL-DATABASES.md) | Document, key-value, column-family, graph databases | When evaluating non-relational data stores |
| [03-DATA-MODELING](03-DATA-MODELING.md) | Schema design, entity relationships, denormalization strategies | When designing new schemas or refactoring existing ones |
| [04-QUERY-OPTIMIZATION](04-QUERY-OPTIMIZATION.md) | Execution plans, indexing strategies, query tuning | When diagnosing slow queries or optimizing performance |
| [05-REPLICATION-AND-SHARDING](05-REPLICATION-AND-SHARDING.md) | Read replicas, horizontal partitioning, consistency models | When scaling beyond a single node |
| [06-MIGRATIONS](06-MIGRATIONS.md) | Schema migrations, zero-downtime migrations, tooling | When evolving schemas in production |
| [07-CACHING](07-CACHING.md) | Redis, Memcached, cache strategies, invalidation | When adding a caching layer |
| [08-CONNECTION-MANAGEMENT](08-CONNECTION-MANAGEMENT.md) | Connection pooling, timeouts, resilience | When operating databases at scale |
| [09-BEST-PRACTICES](09-BEST-PRACTICES.md) | Production patterns, backup strategies, monitoring | **Essential — production checklist** |
| [10-ANTI-PATTERNS](10-ANTI-PATTERNS.md) | Common database mistakes and how to avoid them | **Essential — what NOT to do** |
| [LEARNING-PATH](LEARNING-PATH.md) | Structured 10–12 week training guide with exercises | **Start here** after the Overview |
| [README](README.md) | This index file | Start here for navigation |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Understand ACID vs BASE trade-offs
   - Learn the CAP theorem and its practical implications
   - Survey storage engine fundamentals (B-Trees, LSM-Trees)

2. **Learn Relational Databases** ([01-RELATIONAL-DATABASES](01-RELATIONAL-DATABASES.md))
   - SQL fundamentals and normalization forms
   - Index types and when to use them
   - PostgreSQL and MySQL feature comparison

3. **Explore NoSQL** ([02-NOSQL-DATABASES](02-NOSQL-DATABASES.md))
   - Understand document, key-value, column-family, and graph models
   - Learn when relational vs non-relational is the right choice

4. **Follow the Learning Path** ([LEARNING-PATH](LEARNING-PATH.md))
   - Structured 6-phase curriculum over 10–12 weeks
   - Hands-on exercises, knowledge checks, and a capstone project

### For Experienced Engineers

1. **Review Best Practices** ([09-BEST-PRACTICES](09-BEST-PRACTICES.md))
   - Backup and recovery strategies
   - Monitoring and alerting patterns
   - Capacity planning

2. **Audit Against Anti-Patterns** ([10-ANTI-PATTERNS](10-ANTI-PATTERNS.md))
   - 12 common database mistakes with real-world examples
   - Quick reference checklist for pre-production reviews

3. **Scale Your Data Layer** ([05-REPLICATION-AND-SHARDING](05-REPLICATION-AND-SHARDING.md))
   - Replication topologies and consistency trade-offs
   - Partitioning strategies and rebalancing

4. **Optimize Performance** ([04-QUERY-OPTIMIZATION](04-QUERY-OPTIMIZATION.md))
   - Reading execution plans
   - Index design for complex queries

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Application Layer                                │
│   Services, APIs, background workers, batch jobs                    │
└────────────────────────┬────────────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
┌───────────────┐ ┌─────────────┐ ┌──────────────┐
│  Connection   │ │   Cache     │ │   Message    │
│  Pool / Proxy │ │   Layer     │ │   Queue      │
│  (PgBouncer,  │ │  (Redis,    │ │  (for async  │
│   ProxySQL)   │ │  Memcached) │ │   writes)    │
└───────┬───────┘ └──────┬──────┘ └──────┬───────┘
        │                │               │
        ▼                │               ▼
┌───────────────────────────────────────────────────────────────────┐
│                       Database Layer                               │
│                                                                   │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │ Primary     │  │ Read         │  │  Analytical /            │  │
│  │ (Leader)    │──│ Replicas     │  │  Data Warehouse          │  │
│  │             │  │ (Followers)  │  │  (OLAP)                  │  │
│  └──────┬──────┘  └──────────────┘  └──────────────────────────┘  │
│         │                                                         │
│  ┌──────▼──────────────────────────────────────────────────────┐  │
│  │                   Storage Engine                            │  │
│  │  ┌──────────────┐        ┌──────────────────────────────┐   │  │
│  │  │  B-Tree      │        │  LSM-Tree / SSTable          │   │  │
│  │  │  (PostgreSQL,│        │  (Cassandra, RocksDB,        │   │  │
│  │  │   MySQL InnoDB)       │   LevelDB)                   │   │  │
│  │  └──────────────┘        └──────────────────────────────┘   │  │
│  └─────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
```

## 📖 Designing Data-Intensive Applications (DDIA) Mapping

This topic draws heavily from Martin Kleppmann's *Designing Data-Intensive Applications*. Here is how each chapter maps to our documentation:

| DDIA Chapter | Our Document | Key Concepts |
|---|---|---|
| Ch. 1 — Reliable, Scalable, Maintainable | [00-OVERVIEW](00-OVERVIEW.md) | Reliability, scalability, maintainability trade-offs |
| Ch. 2 — Data Models & Query Languages | [01-RELATIONAL](01-RELATIONAL-DATABASES.md), [02-NOSQL](02-NOSQL-DATABASES.md) | Relational vs document vs graph models |
| Ch. 3 — Storage & Retrieval | [00-OVERVIEW](00-OVERVIEW.md), [04-QUERY-OPT](04-QUERY-OPTIMIZATION.md) | B-Trees, LSM-Trees, column-oriented storage |
| Ch. 4 — Encoding & Evolution | [06-MIGRATIONS](06-MIGRATIONS.md) | Schema evolution, forward/backward compatibility |
| Ch. 5 — Replication | [05-REPLICATION](05-REPLICATION-AND-SHARDING.md) | Leader-follower, multi-leader, leaderless replication |
| Ch. 6 — Partitioning | [05-REPLICATION](05-REPLICATION-AND-SHARDING.md) | Key-range, hash, composite partitioning |
| Ch. 7 — Transactions | [00-OVERVIEW](00-OVERVIEW.md), [01-RELATIONAL](01-RELATIONAL-DATABASES.md) | ACID, isolation levels, serializability |
| Ch. 8 — Trouble with Distributed Systems | [05-REPLICATION](05-REPLICATION-AND-SHARDING.md) | Network faults, clocks, truth and lies |
| Ch. 9 — Consistency & Consensus | [05-REPLICATION](05-REPLICATION-AND-SHARDING.md) | Linearizability, total order broadcast, consensus |

## 🔑 Key Concepts

```
Data Models
───────────
Relational → Tables, rows, SQL, ACID transactions, joins
Document   → JSON/BSON documents, flexible schema, nested data
Key-Value  → Simple lookup by key, high throughput (Redis, DynamoDB)
Column     → Wide columns, time-series, analytics (Cassandra, HBase)
Graph      → Nodes and edges, relationship traversal (Neo4j, Neptune)

Storage Engines
───────────────
B-Tree       → Balanced tree, page-oriented, in-place updates (PostgreSQL, MySQL)
LSM-Tree     → Log-structured, append-only, compaction (Cassandra, RocksDB)
Column Store → Column-oriented, compression, analytical queries (ClickHouse, Redshift)

Consistency & Availability Trade-offs (CAP / PACELC)
───────────────────────────────────────────────────
CP  → Consistent + Partition-tolerant (sacrifices availability during partitions)
AP  → Available + Partition-tolerant (sacrifices consistency during partitions)
PACELC → Extends CAP: during Partition choose A/C, Else choose Latency/Consistency

Transaction Isolation Levels (weakest → strongest)
──────────────────────────────────────────────────
Read Uncommitted → Dirty reads allowed
Read Committed   → No dirty reads, but non-repeatable reads possible
Repeatable Read  → Snapshot isolation, but phantom reads possible
Serializable     → Full isolation, as if transactions execute sequentially
```

## 📋 Topics Covered

- **Foundations** — ACID vs BASE, CAP theorem, PACELC, storage engines, data models
- **Relational Databases** — SQL, normalization, indexing, PostgreSQL, MySQL, transactions
- **NoSQL Databases** — Document, key-value, column-family, graph; when to use each
- **Data Modeling** — Schema design, entity relationships, denormalization, polymorphic patterns
- **Query Optimization** — Execution plans, index strategies, query rewriting, statistics
- **Replication & Sharding** — Leader-follower, multi-leader, leaderless; partitioning strategies
- **Migrations** — Schema evolution, zero-downtime migrations, expand-contract pattern
- **Caching** — Redis, Memcached, cache-aside, write-through, invalidation
- **Connection Management** — Pooling, proxies, timeouts, circuit breakers
- **Best Practices** — Backup, monitoring, capacity planning, security
- **Anti-Patterns** — 12 common database mistakes with examples and fixes

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to databases?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md) then follow [LEARNING-PATH.md](LEARNING-PATH.md)

**Already experienced with SQL?** → Jump to [02-NOSQL-DATABASES.md](02-NOSQL-DATABASES.md) or [05-REPLICATION-AND-SHARDING.md](05-REPLICATION-AND-SHARDING.md)

**Going to production?** → Review [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) and [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md)

**Want a structured path?** → Follow the [LEARNING-PATH.md](LEARNING-PATH.md) — 6 phases, 10–12 weeks
