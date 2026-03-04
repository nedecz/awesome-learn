# Data Partitioning

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [What Is Data Partitioning](#what-is-data-partitioning)
   - [Definition](#definition)
   - [Why Partition Data](#why-partition-data)
3. [Horizontal vs Vertical Partitioning](#horizontal-vs-vertical-partitioning)
   - [Horizontal Partitioning (Sharding)](#horizontal-partitioning-sharding)
   - [Vertical Partitioning](#vertical-partitioning)
   - [Comparison](#comparison)
4. [Sharding Strategies](#sharding-strategies)
   - [Range-Based Partitioning](#range-based-partitioning)
   - [Hash-Based Partitioning](#hash-based-partitioning)
   - [Directory-Based Partitioning](#directory-based-partitioning)
   - [Geographic Partitioning](#geographic-partitioning)
   - [Strategy Comparison](#strategy-comparison)
5. [Consistent Hashing](#consistent-hashing)
   - [The Hash Ring](#the-hash-ring)
   - [Virtual Nodes](#virtual-nodes)
   - [Adding and Removing Nodes](#adding-and-removing-nodes)
6. [Partition Key Selection](#partition-key-selection)
   - [Choosing Good Partition Keys](#choosing-good-partition-keys)
   - [Hot Partitions](#hot-partitions)
   - [Compound Partition Keys](#compound-partition-keys)
7. [Rebalancing Partitions](#rebalancing-partitions)
   - [When to Rebalance](#when-to-rebalance)
   - [Rebalancing Strategies](#rebalancing-strategies)
8. [Cross-Partition Queries](#cross-partition-queries)
   - [Scatter-Gather](#scatter-gather)
   - [Secondary Indexes](#secondary-indexes)
   - [Materialized Views](#materialized-views)
9. [Partitioning and Replication](#partitioning-and-replication)
   - [Combining Partitioning with Replication](#combining-partitioning-with-replication)
   - [Leader Assignment](#leader-assignment)
10. [Common Challenges](#common-challenges)
    - [Hot Spots](#hot-spots)
    - [Cross-Partition Transactions](#cross-partition-transactions)
    - [Joins Across Partitions](#joins-across-partitions)
11. [Next Steps](#next-steps)
12. [Version History](#version-history)

---

## Overview

Data partitioning is the practice of dividing a large data set into smaller, independent subsets — called **partitions** or **shards** — that can be distributed across multiple nodes. It is one of the two fundamental mechanisms (alongside replication) for scaling data systems beyond a single machine. This document covers the core partitioning strategies, their trade-offs, and the practical challenges of building and operating partitioned systems.

### Target Audience

- Backend engineers designing data-intensive services
- Platform engineers building or operating distributed databases
- System designers preparing for architecture discussions and interviews

### Scope

- Horizontal and vertical partitioning concepts
- Sharding strategies: range, hash, directory, and geographic
- Consistent hashing and virtual nodes
- Partition key selection and hot-spot avoidance
- Rebalancing techniques
- Cross-partition query patterns
- Combining partitioning with replication
- Common operational challenges

---

## What Is Data Partitioning

### Definition

**Data partitioning** splits a data set so that each record lives on exactly one partition. Each partition is a self-contained subset of the data that can be stored and queried independently. The term **sharding** is often used interchangeably with horizontal partitioning.

```
  Before Partitioning                After Partitioning
  ─────────────────────              ──────────────────────────────────────

  ┌───────────────────┐              ┌──────────┐ ┌──────────┐ ┌──────────┐
  │   All Records     │              │Partition 0│ │Partition 1│ │Partition 2│
  │   (1 … 900,000)   │    ──▶      │ 1–300K   │ │ 300K–600K│ │ 600K–900K│
  │                   │              │  Node A  │ │  Node B  │ │  Node C  │
  └───────────────────┘              └──────────┘ └──────────┘ └──────────┘
       Single Node                       Three Nodes
```

### Why Partition Data

| Goal | How Partitioning Helps |
|---|---|
| **Write scalability** | Each partition handles a subset of writes independently |
| **Storage scalability** | Data set can exceed the capacity of a single machine |
| **Query performance** | Queries that target a single partition touch less data |
| **Isolation** | A failure or slow query in one partition does not affect others |
| **Compliance** | Data residency laws may require data to stay in specific regions |

> **Key insight:** Partitioning does not replace replication — it complements it. In production systems each partition is typically replicated for fault tolerance.

---

## Horizontal vs Vertical Partitioning

### Horizontal Partitioning (Sharding)

Horizontal partitioning splits rows across partitions. Every partition has the same schema but holds a different subset of rows.

```
  Orders Table — Horizontal Partitioning (by order_id range)
  ──────────────────────────────────────────────────────────

  ┌──────────────────────────┐
  │      Full Table          │
  │ order_id │ customer │ $  │
  │──────────┼──────────┼────│
  │    1     │  Alice   │ 50 │
  │    2     │  Bob     │ 30 │
  │    3     │  Carol   │ 70 │
  │    4     │  Dave    │ 20 │
  │    5     │  Eve     │ 90 │
  │    6     │  Frank   │ 40 │
  └──────────────────────────┘
                │
      ┌─────────┼──────────┐
      ▼         ▼          ▼
  ┌─────────┐ ┌─────────┐ ┌─────────┐
  │ Shard A │ │ Shard B │ │ Shard C │
  │ id 1, 2 │ │ id 3, 4 │ │ id 5, 6 │
  └─────────┘ └─────────┘ └─────────┘
```

### Vertical Partitioning

Vertical partitioning splits columns across partitions. Each partition stores a different set of columns for the same rows, joined by a shared primary key.

```
  Users Table — Vertical Partitioning
  ────────────────────────────────────

  ┌─────────────────────────────────────────┐
  │ id │ name  │ email          │ bio (4KB) │
  └─────────────────────────────────────────┘
                │
        ┌───────┴────────┐
        ▼                ▼
  ┌──────────────────┐  ┌──────────────────┐
  │  Core Partition  │  │  Extended Part.   │
  │ id │ name │ email│  │ id │ bio (4KB)   │
  └──────────────────┘  └──────────────────┘
    Hot, frequent reads    Cold, infrequent reads
```

### Comparison

| Aspect | Horizontal Partitioning | Vertical Partitioning |
|---|---|---|
| **Split axis** | Rows | Columns |
| **Schema** | Same schema in each partition | Different columns per partition |
| **Primary use** | Scale writes and storage | Separate hot/cold columns, large BLOBs |
| **Scalability** | Near-linear with more nodes | Limited by number of column groups |
| **Joins** | Cross-shard joins are expensive | Requires joining partitions by primary key |
| **Examples** | Cassandra, Vitess, CockroachDB | Columnar stores (Redshift), BLOB storage offloading |

---

## Sharding Strategies

### Range-Based Partitioning

Assign each partition a contiguous range of key values. A query that falls within one range can be served by a single partition.

```
  Range-Based Partitioning (by user_id)
  ──────────────────────────────────────

  Partition A:   user_id    1 – 10,000
  Partition B:   user_id    10,001 – 20,000
  Partition C:   user_id    20,001 – 30,000

  Query: WHERE user_id = 15,000   →   routes to Partition B
  Query: WHERE user_id BETWEEN 9,000 AND 11,000
         →   touches Partition A and Partition B
```

- ✅ Efficient range scans and sorted access
- ✅ Simple to understand and implement
- ❌ Prone to hot spots if writes cluster on recent keys (e.g., auto-increment IDs, timestamps)

### Hash-Based Partitioning

Apply a hash function to the partition key and use the hash value to determine the partition. This distributes data more uniformly than range-based partitioning.

```
  Hash-Based Partitioning (3 partitions)
  ───────────────────────────────────────

  partition = hash(key) mod 3

  hash("alice")   = 7   → 7 mod 3 = 1  → Partition 1
  hash("bob")     = 12  → 12 mod 3 = 0 → Partition 0
  hash("carol")   = 5   → 5 mod 3 = 2  → Partition 2
  hash("dave")    = 9   → 9 mod 3 = 0  → Partition 0
```

- ✅ Even distribution reduces hot spots
- ✅ Works well for point lookups (key = value)
- ❌ Range queries require scatter-gather across all partitions
- ❌ Adding or removing nodes reshuffles most keys (mitigated by consistent hashing)

### Directory-Based Partitioning

A lookup service (directory) maintains a mapping from each key (or key range) to a partition. The directory is queried before every data access to determine routing.

```
  Directory-Based Partitioning
  ────────────────────────────

  ┌──────────────────┐       ┌──────────┐
  │  Client Request  │──────▶│Directory │
  │  key = "order42" │       │ Service  │
  └──────────────────┘       └────┬─────┘
                                  │ lookup("order42") → Partition B
                                  ▼
                            ┌──────────┐
                            │Partition B│
                            └──────────┘
```

- ✅ Maximum flexibility — partitions can be reassigned without rehashing
- ✅ Supports arbitrary mapping logic
- ❌ Directory is a single point of failure (must be replicated)
- ❌ Extra network hop for every request

### Geographic Partitioning

Assign partitions based on the geographic region of the data or the user. This is common for compliance (data residency) and latency optimization.

```
  Geographic Partitioning
  ───────────────────────

  ┌────────────┐    ┌────────────┐    ┌────────────┐
  │  US-East   │    │  EU-West   │    │  AP-South  │
  │ Partition  │    │ Partition  │    │ Partition  │
  │            │    │            │    │            │
  │ US users   │    │ EU users   │    │ APAC users │
  └────────────┘    └────────────┘    └────────────┘
    Virginia           Ireland          Mumbai
```

- ✅ Lower latency — users read/write to nearest region
- ✅ Compliance with GDPR, data sovereignty laws
- ❌ Cross-region queries are expensive (high latency)
- ❌ Uneven user distribution can cause imbalanced partitions

### Strategy Comparison

| Strategy | Distribution | Range Queries | Flexibility | Hot-Spot Risk | Complexity |
|---|---|---|---|---|---|
| **Range-based** | By key order | ✅ Efficient | Low | High (sequential keys) | Low |
| **Hash-based** | By hash value | ❌ Scatter-gather | Low | Low | Low |
| **Directory-based** | By lookup table | Depends on mapping | High | Depends on mapping | High |
| **Geographic** | By region | Within-region only | Medium | Medium (popular regions) | Medium |

---

## Consistent Hashing

Simple `hash(key) mod N` breaks when the number of nodes N changes — nearly all keys are remapped. **Consistent hashing** minimizes data movement by mapping both nodes and keys onto a ring.

### The Hash Ring

Both nodes and keys are hashed onto a circular space (0 to 2³² − 1). Each key is assigned to the first node encountered when walking clockwise from the key's position on the ring.

```
  Consistent Hashing — The Ring
  ─────────────────────────────

                     Node A
                       ●
                   ╱       ╲
                 ╱           ╲
              ╱                 ╲
           ●                      ●
        Node D                  Node B
              ╲                 ╱
                ╲             ╱
                  ╲         ╱
                     ●
                   Node C

  key₁ hashes between A and B → assigned to Node B
  key₂ hashes between B and C → assigned to Node C
  key₃ hashes between D and A → assigned to Node A
```

When a node is added or removed, only the keys in the affected segment of the ring need to move — on average **K/N** keys (where K = total keys, N = total nodes), compared to nearly all keys with `mod N`.

### Virtual Nodes

With only a few physical nodes, the ring can be unevenly divided. **Virtual nodes** (vnodes) solve this by mapping each physical node to many positions on the ring.

```
  Virtual Nodes
  ─────────────

  Physical node A → vnodes: A₁, A₂, A₃, A₄
  Physical node B → vnodes: B₁, B₂, B₃, B₄
  Physical node C → vnodes: C₁, C₂, C₃, C₄

       A₁    B₂    C₁    A₃    B₁    C₃    A₂    B₃    C₂    A₄    B₄    C₄
  ──●──────●──────●──────●──────●──────●──────●──────●──────●──────●──────●──────●──
  0                                                                             2³²

  More vnodes per physical node → more uniform distribution
  A powerful node can be assigned more vnodes than a weaker one
```

| Virtual Nodes per Physical Node | Distribution Uniformity | Rebalancing Granularity |
|---|---|---|
| 1 (no vnodes) | Poor — large variance | Coarse — large data movements |
| 32–64 | Good for most workloads | Fine — small segments move |
| 256+ | Excellent | Very fine — minimal disruption |

### Adding and Removing Nodes

```
  Adding Node E to the ring
  ─────────────────────────

  Before:   ... ──── Node A ──────────────── Node B ── ...
                           key₁ key₂ key₃

  After:    ... ──── Node A ──── Node E ──── Node B ── ...
                           key₁    │   key₂ key₃
                                   │
                          key₁ moves to Node E
                          key₂, key₃ stay on Node B

  Only keys between A and E on the ring move to E.
  All other keys are unaffected.
```

---

## Partition Key Selection

The partition key determines which partition a record lives on. A poor choice leads to uneven distribution (hot spots), while a good choice spreads load evenly.

### Choosing Good Partition Keys

| Guideline | Explanation | Example |
|---|---|---|
| **High cardinality** | Many distinct values ensure records spread across partitions | `user_id` (millions of values) over `country` (< 200 values) |
| **Uniform distribution** | Values should be roughly equally frequent | `order_id` (UUID) over `status` (few values, skewed) |
| **Query alignment** | The key should match your most common access pattern | Partition by `tenant_id` if most queries filter by tenant |
| **Avoid monotonic keys** | Auto-increment IDs or timestamps push all writes to one partition | Prefix timestamps with a random shard key |

### Hot Partitions

A **hot partition** receives a disproportionate share of traffic. Common causes:

1. **Celebrity problem** — A single key (e.g., a viral user's profile) gets millions of reads/writes
2. **Temporal clustering** — All writes go to the "current" time range partition
3. **Low-cardinality key** — Few distinct values mean few partitions carry all load

Mitigation strategies:

```
  Strategy 1: Salting / Key Prefixing
  ────────────────────────────────────
  Append a random suffix (0–9) to hot keys:
    "user:12345"  →  "user:12345:7"
  Reads must scatter across all 10 salted variants and merge results.

  Strategy 2: Split Hot Partitions
  ────────────────────────────────────
  Detect hot partitions at runtime and split them dynamically.
  Some databases (e.g., CockroachDB, Spanner) do this automatically.

  Strategy 3: Application-Level Routing
  ────────────────────────────────────
  Known hot keys are routed to dedicated, over-provisioned partitions.
```

### Compound Partition Keys

A compound (composite) partition key uses multiple columns to determine the partition. This is common in wide-column stores like Cassandra.

```
  Cassandra Compound Key Example
  ──────────────────────────────

  PRIMARY KEY ((tenant_id, year), event_time, event_id)
               ─────────────────  ──────────────────────
               Partition key       Clustering columns
               (determines shard)  (sort order within partition)

  All events for tenant "acme" in year 2026 land in the same partition,
  sorted by event_time — efficient for time-range queries within a tenant.
```

---

## Rebalancing Partitions

As data grows or nodes are added and removed, partitions must be **rebalanced** — redistributed across nodes to maintain even load.

### When to Rebalance

| Trigger | Description |
|---|---|
| **Node added** | New capacity should absorb some partitions |
| **Node removed / failed** | Its partitions must move to surviving nodes |
| **Data skew** | One partition has grown much larger than others |
| **Load skew** | One partition handles disproportionate traffic |
| **Storage threshold** | A partition exceeds a size limit (e.g., 10 GB) |

### Rebalancing Strategies

| Strategy | How It Works | Pros | Cons |
|---|---|---|---|
| **Fixed number of partitions** | Create many more partitions than nodes at setup (e.g., 1,000 partitions for 10 nodes). When nodes change, move whole partitions. | Simple; no splitting needed | Must guess partition count up front; hard to change later |
| **Dynamic splitting** | Start with one partition per node. When a partition exceeds a size threshold, split it in two. Merge small partitions. | Adapts to data growth | Splitting temporarily degrades performance |
| **Proportional to nodes** | Keep a fixed number of partitions per node (e.g., 256 vnodes per node). Adding a node splits existing partitions to fill the new node. | Scales naturally with cluster size | Splitting cost on node addition |

```
  Fixed Number of Partitions — Rebalancing on Node Addition
  ─────────────────────────────────────────────────────────

  Before (10 nodes, 1000 partitions):
    Node 1: P1–P100    Node 2: P101–P200   ...   Node 10: P901–P1000

  After adding Node 11:
    ~91 partitions per node (1000 / 11)
    Some partitions move from existing nodes to Node 11

  No partition is split — whole partitions migrate.
```

> **Best practice:** Avoid `hash(key) mod N` for partition assignment — changing N forces almost all data to move. Use consistent hashing or a fixed partition count with a mapping table.

---

## Cross-Partition Queries

When a query cannot be answered by a single partition, the system must coordinate across multiple partitions.

### Scatter-Gather

The coordinator sends the query to all (or relevant) partitions, each executes it locally, and the coordinator merges the results.

```
  Scatter-Gather Query
  ────────────────────

  Client: SELECT * FROM orders WHERE amount > 100

  ┌─────────────┐
  │ Coordinator  │
  └──┬───┬───┬──┘
     │   │   │       scatter
     ▼   ▼   ▼
  ┌────┐┌────┐┌────┐
  │ P0 ││ P1 ││ P2 │   each partition filters locally
  └──┬─┘└──┬─┘└──┬─┘
     │     │     │       gather
     ▼     ▼     ▼
  ┌─────────────┐
  │ Coordinator  │       merge and return to client
  └─────────────┘
```

- ✅ Works for any query, regardless of partition key
- ❌ Latency is dominated by the slowest partition (tail latency)
- ❌ High fan-out increases network and CPU cost

### Secondary Indexes

Secondary indexes allow efficient lookups on columns that are not the partition key. Two approaches:

| Approach | How It Works | Pros | Cons |
|---|---|---|---|
| **Local (partition-level) index** | Each partition maintains its own index covering only its data | Writes are fast (index update is local) | Reads require scatter-gather across all partitions |
| **Global (term-level) index** | The index itself is partitioned — each index partition covers a range of index terms across all data partitions | Reads hit a single index partition | Writes may need to update multiple index partitions (distributed write) |

```
  Local vs Global Secondary Index
  ────────────────────────────────

  Local Index (document-partitioned):

  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
  │    Partition 0    │  │    Partition 1    │  │    Partition 2    │
  │ data + index on  │  │ data + index on  │  │ data + index on  │
  │ color:red → [3,7]│  │ color:red → [12] │  │ color:red → [21] │
  └──────────────────┘  └──────────────────┘  └──────────────────┘
  Query: color = red → must ask ALL partitions (scatter-gather)

  Global Index (term-partitioned):

  ┌──────────────────┐  ┌──────────────────┐
  │ Index Partition A │  │ Index Partition B │
  │ color:red →      │  │ color:blue →     │
  │   [3, 7, 12, 21] │  │   [1, 5, 18]     │
  └──────────────────┘  └──────────────────┘
  Query: color = red → only Index Partition A (single read)
```

### Materialized Views

Pre-computed query results stored as a separate table, often partitioned differently from the source data. Useful for frequently executed cross-partition queries.

- ✅ Reads are fast — pre-computed and co-located
- ❌ Writes are more expensive — the view must be updated on every relevant change
- ❌ Storage overhead — data is duplicated

> **Example:** An orders table partitioned by `customer_id` with a materialized view partitioned by `product_id` for product-level analytics.

---

## Partitioning and Replication

In production systems, partitioning and replication are combined. Each partition is replicated across multiple nodes so that the system tolerates node failures without data loss.

### Combining Partitioning with Replication

```
  Partitioned and Replicated System (3 partitions, replication factor 3)
  ──────────────────────────────────────────────────────────────────────

  Node 1          Node 2          Node 3          Node 4          Node 5
  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
  │ P0 (L)   │   │ P0 (F)   │   │ P1 (L)   │   │ P1 (F)   │   │ P2 (L)   │
  │ P1 (F)   │   │ P2 (F)   │   │ P2 (F)   │   │ P0 (F)   │   │ P1 (F)   │
  └──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘

  L = Leader (accepts writes)    F = Follower (serves reads, provides redundancy)

  Each partition has 1 leader and 2 followers spread across different nodes.
  No node holds two replicas of the same partition.
```

### Leader Assignment

| Strategy | Description | Example System |
|---|---|---|
| **Fixed leader per partition** | A metadata service assigns a leader node for each partition | Vitess, Citus |
| **Raft / Paxos per partition** | Each partition runs its own consensus group to elect a leader | CockroachDB, TiKV |
| **Leaderless** | No leader — reads and writes use quorum protocols | Cassandra, DynamoDB |

> **Key insight:** The combination of partitioning and replication means that the **leader for each partition** can be on a **different node**, spreading write load across the cluster even though each individual partition has a single writer.

---

## Common Challenges

### Hot Spots

Even with good partitioning, hot spots can emerge:

| Cause | Example | Mitigation |
|---|---|---|
| **Skewed key distribution** | 80% of orders come from 5% of customers | Composite keys, salting |
| **Celebrity keys** | A viral post generates millions of reads | Replicate hot partitions, in-memory caching |
| **Time-based write clustering** | All IoT sensors report at the same second | Prefix time keys with device hash |
| **Seasonal traffic** | Black Friday spikes on product catalog partitions | Auto-scaling, pre-splitting |

### Cross-Partition Transactions

Transactions that span multiple partitions require coordination, which adds latency and complexity.

```
  Cross-Partition Transaction (2PC)
  ──────────────────────────────────

  Transfer $100 from Account A (Partition 1) to Account B (Partition 2)

  ┌──────────────┐
  │ Coordinator   │
  └───┬──────┬───┘
      │      │
  Phase 1: PREPARE
      │      │
      ▼      ▼
  ┌──────┐ ┌──────┐
  │  P1  │ │  P2  │     Both vote YES
  └──┬───┘ └──┬───┘
     │        │
  Phase 2: COMMIT
     │        │
     ▼        ▼
  ┌──────┐ ┌──────┐
  │  P1  │ │  P2  │     Both commit
  │ -$100│ │ +$100│
  └──────┘ └──────┘
```

- ❌ 2PC adds latency (two round-trips to all participants)
- ❌ If the coordinator crashes between phases, participants may block
- ✅ Spanner uses synchronized clocks (TrueTime) to reduce coordination overhead
- ✅ Calvin and deterministic databases pre-order transactions to avoid 2PC

### Joins Across Partitions

Joins that cross partition boundaries are expensive because data must be shipped between nodes.

| Approach | Description | Trade-Off |
|---|---|---|
| **Denormalization** | Store redundant data to avoid joins | Faster reads, more storage, update anomalies |
| **Broadcast join** | Send the smaller table to all partitions | Works for small dimension tables; expensive for large tables |
| **Co-located partitioning** | Partition related tables by the same key so joins stay local | Requires careful schema design; limits flexibility |
| **Application-level join** | Fetch from each partition separately and join in the application | Simple to implement; high latency for large result sets |

> **Best practice:** Design your partition key so that the most frequent joins are partition-local. Denormalize or use materialized views for cross-partition access patterns.

---

## Next Steps

Continue to the next topic to explore messaging systems, event-driven architectures, and how they interact with partitioned data stores in distributed systems.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial version — horizontal/vertical partitioning, sharding strategies, consistent hashing, partition key selection, rebalancing, cross-partition queries, replication integration, common challenges |
