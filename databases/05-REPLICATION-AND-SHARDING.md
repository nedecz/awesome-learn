# Replication and Sharding

## Table of Contents

1. [Overview](#overview)
   - [Why Replicate and Partition Data?](#why-replicate-and-partition-data)
   - [The Spectrum: Single-Node to Distributed](#the-spectrum-single-node-to-distributed)
2. [Replication](#replication)
   - [Leader-Follower (Single-Leader) Replication](#leader-follower-single-leader-replication)
   - [Multi-Leader Replication](#multi-leader-replication)
   - [Leaderless Replication](#leaderless-replication)
3. [Consistency Models](#consistency-models)
   - [Eventual Consistency](#eventual-consistency)
   - [Causal Consistency](#causal-consistency)
   - [Linearizability](#linearizability)
   - [Total Order Broadcast and Consensus](#total-order-broadcast-and-consensus)
   - [Real-World Consistency Implementations](#real-world-consistency-implementations)
4. [Partitioning / Sharding](#partitioning--sharding)
   - [Key-Range Partitioning](#key-range-partitioning)
   - [Hash Partitioning](#hash-partitioning)
   - [Composite Partition Keys](#composite-partition-keys)
   - [Secondary Indexes and Partitioning](#secondary-indexes-and-partitioning)
   - [Rebalancing Strategies](#rebalancing-strategies)
   - [Request Routing](#request-routing)
5. [Distributed Transactions and Consensus](#distributed-transactions-and-consensus)
   - [Two-Phase Commit (2PC)](#two-phase-commit-2pc)
   - [Three-Phase Commit (3PC)](#three-phase-commit-3pc)
   - [Raft](#raft)
   - [Paxos](#paxos)
   - [Exactly-Once Semantics and Idempotency](#exactly-once-semantics-and-idempotency)
6. [Problems with Distributed Systems](#problems-with-distributed-systems)
   - [Network Partitions](#network-partitions)
   - [Clock Skew](#clock-skew)
   - [Split Brain](#split-brain)
   - [Fencing Tokens and Leader Election](#fencing-tokens-and-leader-election)
7. [Practical Patterns](#practical-patterns)
   - [PostgreSQL Streaming Replication](#postgresql-streaming-replication)
   - [Cassandra Partition Key Design](#cassandra-partition-key-design)
   - [CockroachDB Distributed Transactions](#cockroachdb-distributed-transactions)
8. [Version History](#version-history)

---

## Overview

This document covers **replication** (keeping copies of the same data on multiple nodes) and **partitioning / sharding** (splitting data across nodes so each node holds a subset). These are the two fundamental mechanisms for scaling databases beyond a single machine. The material draws heavily from *Designing Data-Intensive Applications* (DDIA) by Martin Kleppmann, particularly Chapters 5 (Replication), 6 (Partitioning), 8 (The Trouble with Distributed Systems), and 9 (Consistency and Consensus).

### Why Replicate and Partition Data?

| Goal | Mechanism | How It Helps |
|---|---|---|
| **Fault tolerance** | Replication | If one node dies, others still serve data |
| **Lower latency** | Replication | Place data geographically closer to users |
| **Read scalability** | Replication | Spread read traffic across replicas |
| **Write scalability** | Partitioning | Distribute writes across independent partitions |
| **Storage scalability** | Partitioning | Data set exceeds capacity of a single machine |

### The Spectrum: Single-Node to Distributed

```
   Complexity ──────────────────────────────────────────────────▶

   ┌────────────┐   ┌─────────────────┐   ┌──────────────────┐   ┌────────────────────┐
   │ Single     │   │ Primary +       │   │ Partitioned      │   │ Partitioned +      │
   │ Node       │   │ Read Replicas   │   │ (Sharded)        │   │ Replicated         │
   │            │   │                 │   │                  │   │                    │
   │ • Simple   │   │ • Read scale    │   │ • Write scale    │   │ • Read + write     │
   │ • ACID     │   │ • Failover      │   │ • Storage scale  │   │   scale            │
   │ • Limited  │   │ • Replication   │   │ • Routing needed │   │ • Fault-tolerant   │
   │   scale    │   │   lag           │   │ • Cross-shard    │   │ • Most complex     │
   │            │   │                 │   │   queries costly │   │                    │
   └────────────┘   └─────────────────┘   └──────────────────┘   └────────────────────┘
   PostgreSQL       PostgreSQL +           Cassandra,             CockroachDB, Spanner,
   (single)         streaming replication  Vitess, Citus          YugabyteDB
```

> **Key insight (DDIA Ch.5):** Replication and partitioning are often combined. Each partition is replicated across multiple nodes for fault tolerance, while the data set is split across partitions for scalability.

---

## Replication

Replication means keeping a copy of the same data on multiple machines (DDIA Ch.5). The three main approaches are: **single-leader**, **multi-leader**, and **leaderless**.

```
                    ┌─────────────────────────────────────────────────────────┐
                    │              Replication Topologies                      │
                    ├───────────────────┬─────────────────┬───────────────────┤
                    │   Single-Leader   │  Multi-Leader   │    Leaderless     │
                    │                   │                 │                   │
                    │  Writes → 1 node  │ Writes → N      │ Writes → any     │
                    │  Reads  → any     │ leaders         │ node (quorum)    │
                    │                   │ (usually per-DC)│                   │
                    │  PostgreSQL,      │ CouchDB,        │ Cassandra,       │
                    │  MySQL, MongoDB   │ Tungsten,       │ DynamoDB, Riak   │
                    │  (default)        │ Active Directory│                   │
                    └───────────────────┴─────────────────┴───────────────────┘
```

### Leader-Follower (Single-Leader) Replication

The most common replication approach. One node (the **leader** / primary) accepts all writes. The leader streams changes to **followers** (replicas / standbys) via a replication log.

```
     Client writes                     Client reads
         │                            ┌────┼────┐
         ▼                            ▼    ▼    ▼
    ┌──────────┐  replication   ┌───────┐ ┌───────┐ ┌───────┐
    │  Leader   │──────────────▶│Follower│ │Follower│ │Follower│
    │ (primary) │  stream       │   1    │ │   2    │ │   3    │
    └──────────┘               └───────┘ └───────┘ └───────┘
    Accepts writes              Serve reads (may be stale)
```

#### Synchronous vs Asynchronous Replication

| Mode | Behavior | Trade-off |
|---|---|---|
| **Synchronous** | Leader waits for follower to confirm write before responding to client | Strong durability guarantee, but higher write latency. A single slow follower blocks all writes. |
| **Asynchronous** | Leader responds to client immediately, streams changes in background | Lower write latency, but data may be lost if the leader fails before replication completes. |
| **Semi-synchronous** | Leader waits for **one** follower to confirm; the rest are async | Compromise: at least two copies exist before acknowledging, without waiting for all followers. PostgreSQL uses this by default when synchronous replication is enabled. |

> **DDIA Ch.5:** Fully synchronous replication is impractical — if any follower is down or slow, the leader cannot process writes. In practice, "synchronous" usually means semi-synchronous: one follower is synchronous, the rest are asynchronous.

#### Replication Lag

When followers replicate asynchronously, they may fall behind the leader. The difference between the leader's most recent write and the follower's most recent applied write is called **replication lag**.

Problems caused by replication lag:

1. **Reading your own writes** — A user writes data, then reads from a follower that hasn't caught up yet and sees stale data.
2. **Monotonic reads** — A user makes two reads that hit different followers; the second read returns older data than the first.
3. **Consistent prefix reads** — Causally related writes appear in the wrong order on a follower.

#### Read-After-Write Consistency

Strategies to ensure a user sees their own writes:

```
Strategy 1: Read from leader for "own" data
─────────────────────────────────────────────
User writes profile → reads profile → route to leader
User reads other data → route to any follower

Strategy 2: Timestamp-based routing
─────────────────────────────────────────────
Client tracks last write timestamp (t_w)
On read: pick a follower whose replication position ≥ t_w
If no follower is caught up: read from leader

Strategy 3: Logical timestamps / causal tokens
─────────────────────────────────────────────
Server returns a token with each write
Client sends token with subsequent reads
Server ensures the read reflects at least that token's state
```

### Multi-Leader Replication

In multi-leader (also called **active/active** or **master-master**) replication, more than one node accepts writes. Each leader replicates its changes to all other leaders.

#### Use Cases

- **Multi-datacenter operation** — One leader per datacenter. Writes are accepted locally and replicated asynchronously across datacenters. This reduces write latency compared to a single-leader approach where all writes must cross the WAN.
- **Offline-capable clients** — Each device has a local database that acts as a leader (e.g., CouchDB, calendar apps). Changes sync when connectivity is restored.
- **Collaborative editing** — Real-time collaboration tools (e.g., Google Docs) are conceptually multi-leader, with each user's edits applied locally and replicated.

```
     Datacenter A                    Datacenter B
    ┌──────────────────┐            ┌──────────────────┐
    │  ┌─────────┐     │    async   │     ┌─────────┐  │
    │  │ Leader  │◀────┼───────────▶┼────▶│ Leader  │  │
    │  │   A     │     │ replication│     │   B     │  │
    │  └────┬────┘     │            │     └────┬────┘  │
    │       │          │            │          │       │
    │  ┌────▼────┐     │            │     ┌────▼────┐  │
    │  │Followers│     │            │     │Followers│  │
    │  └─────────┘     │            │     └─────────┘  │
    └──────────────────┘            └──────────────────┘
    Local writes: low latency        Local writes: low latency
```

#### Conflict Resolution Strategies

When two leaders concurrently modify the same record, a **write conflict** arises. Resolution strategies (DDIA Ch.5):

| Strategy | How It Works | Drawback |
|---|---|---|
| **Last write wins (LWW)** | Attach a timestamp to each write; highest timestamp wins | Data loss — concurrent writes are silently discarded. Clock skew can make this arbitrary. |
| **Merge values** | Combine concurrent writes (e.g., union of set elements) | Only works for certain data types (CRDTs) |
| **Custom conflict handler** | Application-level code resolves the conflict (on write or on read) | Adds complexity to application logic |
| **Conflict-free replicated data types (CRDTs)** | Data structures designed so concurrent updates converge automatically (counters, sets, maps) | Limited to supported data types |
| **Operational transformation** | Transform concurrent operations to converge (used in Google Docs) | Complex to implement correctly |

### Leaderless Replication

In leaderless (Dynamo-style) replication, **any node** can accept writes. The client sends writes to multiple nodes in parallel and reads from multiple nodes, using **quorum** rules to ensure consistency.

```
     Client write (w = 2)              Client read (r = 2)
         │                                 │
    ┌────┼────┐                       ┌────┼────┐
    ▼    ▼    ▼                       ▼    ▼    ▼
  ┌───┐ ┌───┐ ┌───┐               ┌───┐ ┌───┐ ┌───┐
  │ A │ │ B │ │ C │               │ A │ │ B │ │ C │
  │ ✓ │ │ ✓ │ │ ✗ │               │ v2│ │ v2│ │ v1│
  └───┘ └───┘ └───┘               └───┘ └───┘ └───┘
  Write succeeds if               Read returns v2 (latest)
  w nodes acknowledge             from at least one of r nodes
```

#### Quorum Reads and Writes

With **n** replicas, a write requires **w** acknowledgments and a read queries **r** nodes. As long as:

```
    w + r > n
```

at least one of the nodes read from will have the latest value. Common configurations:

| n | w | r | Properties |
|---|---|---|---|
| 3 | 2 | 2 | Standard quorum. Tolerates 1 node failure for both reads and writes. |
| 3 | 3 | 1 | Fast reads (only 1 node needed), but writes require all nodes to be up. |
| 3 | 1 | 3 | Fast writes (only 1 node needed), but reads require all nodes. |
| 5 | 3 | 3 | Tolerates 2 failures. Common in larger clusters. |

> **DDIA Ch.5:** Even with quorum settings, there are edge cases where stale data is returned — for example, if a write is concurrent with a read, or if a node carrying a new value fails and is restored from a stale replica.

#### Sloppy Quorums and Hinted Handoff

When a node in the designated quorum is unreachable, a **sloppy quorum** allows writes to be temporarily accepted by a node that is not among the designated n replicas. Once the original node recovers, the temporary node sends the data via **hinted handoff**.

```
    Normal quorum: nodes {A, B, C}

    Node C is down:
    ┌───┐ ┌───┐ ┌───┐ ┌───┐
    │ A │ │ B │ │ C │ │ D │  ← D accepts writes on behalf of C
    │ ✓ │ │ ✓ │ │ ✗ │ │ ✓ │     (sloppy quorum)
    └───┘ └───┘ └───┘ └───┘

    When C recovers:
    D ───hinted handoff───▶ C    (D sends C's missed writes)
```

Sloppy quorums increase write availability but weaken the consistency guarantee: a read quorum from {A, B, C} may miss the value stored on D.

#### Read Repair and Anti-Entropy

Leaderless systems use two mechanisms to ensure replicas converge:

1. **Read repair** — When a client reads from multiple replicas and detects a stale value, it writes the latest value back to the stale replica. This is done on the read path and is opportunistic — only data that is read gets repaired.

2. **Anti-entropy process** — A background process constantly compares data between replicas and copies missing data. Unlike leader-based replication, this does not preserve ordering. Cassandra and Riak use Merkle trees to efficiently identify differences.

---

## Consistency Models

Different replication strategies offer different consistency guarantees (DDIA Ch.9). Understanding these is critical for building correct distributed systems.

```
    Weakest                                                   Strongest
    ◀──────────────────────────────────────────────────────────────▶

    Eventual         Causal           Sequential       Linearizable
    consistency      consistency      consistency      (strong)

    • All replicas   • Respects       • All ops appear • Behaves as if
      eventually       cause-and-       in the same      there is a
      converge         effect order     total order      single copy
    • No ordering   • Concurrent ops • Like a single  • Real-time
      guarantees      may be in any    sequential        ordering
                      order            processor        guarantee
```

### Eventual Consistency

The weakest useful guarantee: if no new writes are made, all replicas will **eventually** converge to the same value. There is no bound on how long "eventually" takes.

- **Used by:** Cassandra (default), DynamoDB, DNS
- **Appropriate for:** Social media feeds, analytics counters, caches — where temporarily stale data is acceptable

### Causal Consistency

Stronger than eventual: operations that are **causally related** are seen in the same order by all nodes. Concurrent (causally unrelated) operations may be seen in different orders by different nodes.

- If A happened before B (A caused B), then every node sees A before B
- If A and B are concurrent, nodes may see them in any order

Vector clocks and version vectors are common mechanisms for tracking causal dependencies.

### Linearizability

The strongest single-object consistency model. The system behaves as if there is a single copy of the data, and every operation takes effect atomically at some point between its start and end.

> **DDIA Ch.9:** Linearizability is a recency guarantee — once a write completes, all subsequent reads (from any client, on any node) must see that write or a later one.

**Where linearizability is required:**

- **Leader election** — All nodes must agree on who is the leader (e.g., via a lock in ZooKeeper)
- **Uniqueness constraints** — Two users must not register the same username concurrently
- **Cross-channel coordination** — When a side channel (e.g., a message queue) relies on the database being up-to-date

**Cost of linearizability:** According to the CAP theorem, a linearizable system must sacrifice availability during network partitions. Linearizability also adds latency because every operation must coordinate across replicas.

### Total Order Broadcast and Consensus

**Total order broadcast** (or atomic broadcast) is a protocol that guarantees:
1. **Reliable delivery** — Every message is delivered to all nodes
2. **Total ordering** — All nodes deliver messages in the same order

Total order broadcast is equivalent to consensus and can be used to implement linearizable storage (DDIA Ch.9). If you have total order broadcast, you can build a linearizable compare-and-set operation on top of it.

### Real-World Consistency Implementations

| Database | Replication Model | Default Consistency | Strongest Available |
|---|---|---|---|
| **PostgreSQL** | Single-leader (streaming) | Read committed (single node); eventual (async replica reads) | Synchronous replication provides durability, but not linearizable reads from replicas without `synchronous_commit` + reading from primary |
| **Cassandra** | Leaderless (Dynamo-style) | Eventual (ONE/ONE) | Linearizable per-partition via lightweight transactions (Paxos-based `IF NOT EXISTS`) with `SERIAL` consistency |
| **CockroachDB** | Distributed SQL (Raft-based) | Serializable | Serializable by default; linearizable reads available via `AS OF SYSTEM TIME` or closed timestamps |
| **MongoDB** | Single-leader (per shard) | Eventual (secondary reads) | Causal consistency sessions; linearizable reads with `readConcern: "linearizable"` |
| **DynamoDB** | Leaderless | Eventual | Strong consistency via `ConsistentRead: true` (reads from leader) |

---

## Partitioning / Sharding

Partitioning (DDIA Ch.6) splits a large data set into **partitions** (also called **shards**, **regions**, or **tablets**), where each partition is a subset of the data. The goal is to spread data and query load evenly across nodes.

```
    Full dataset
    ┌──────────────────────────────────────────────────────┐
    │  A-F  │  G-L  │  M-R  │  S-Z                        │
    └───┬───┴───┬───┴───┬───┴───┬──────────────────────────┘
        │       │       │       │
        ▼       ▼       ▼       ▼
    ┌───────┐┌───────┐┌───────┐┌───────┐
    │Node 1 ││Node 2 ││Node 3 ││Node 4 │    ← Each node owns
    │ A-F   ││ G-L   ││ M-R   ││ S-Z   │       a partition
    └───────┘└───────┘└───────┘└───────┘
```

### Key-Range Partitioning

Assign a continuous range of keys to each partition (like volumes of an encyclopedia).

**Advantages:**
- Efficient range queries — scanning a range of keys hits one partition (or a few adjacent ones)
- Simple to understand and implement

**Disadvantages:**
- Risk of **hot spots** — if access patterns are not uniform (e.g., time-series data where all writes go to the "today" partition)
- Partition boundaries must be chosen carefully (manually or adaptively)

**Used by:** HBase, BigTable, pre-2.x MongoDB (with range-based shard keys)

### Hash Partitioning

Apply a hash function to the key and assign hash ranges to partitions.

```
    key → hash(key) → partition

    Hash range [0x0000 ... 0x3FFF] → Partition 1
    Hash range [0x4000 ... 0x7FFF] → Partition 2
    Hash range [0x8000 ... 0xBFFF] → Partition 3
    Hash range [0xC000 ... 0xFFFF] → Partition 4
```

**Advantages:**
- Distributes keys evenly, reducing hot spots
- Works well for random access patterns

**Disadvantages:**
- Range queries are inefficient — adjacent keys by sort order are scattered across partitions
- A range scan must query all partitions (scatter-gather)

**Used by:** Cassandra (default partitioner), DynamoDB, CockroachDB, Riak

### Composite Partition Keys

A compromise between range and hash partitioning. Cassandra's data model illustrates this well:

```sql
CREATE TABLE sensor_readings (
    sensor_id   TEXT,
    reading_time TIMESTAMP,
    value       DOUBLE,
    PRIMARY KEY ((sensor_id), reading_time)
);
-- (sensor_id) is the partition key → hashed for distribution
-- reading_time is the clustering column → sorted within the partition
```

```
    Partition key: hash(sensor_id)     Clustering: reading_time (sorted)
    ┌──────────────────────────────────────────────────────────┐
    │ Partition: sensor_id = "sensor-42"                       │
    │  ┌──────────────┬──────────────┬──────────────┬────────┐ │
    │  │ 2025-01-01   │ 2025-01-02   │ 2025-01-03   │ ...    │ │
    │  │ value: 23.4  │ value: 24.1  │ value: 22.8  │        │ │
    │  └──────────────┴──────────────┴──────────────┴────────┘ │
    └──────────────────────────────────────────────────────────┘
```

This gives you hash distribution across partitions (avoiding hot spots) while preserving sort order within a partition (enabling efficient range scans on `reading_time`).

### Secondary Indexes and Partitioning

Secondary indexes are challenging in a partitioned database because the index entry may not be co-located with the data (DDIA Ch.6).

#### Document-Partitioned Index (Local Index)

Each partition maintains its own secondary index covering only the data in that partition.

```
    Partition 1 (users A-M)         Partition 2 (users N-Z)
    ┌────────────────────────┐      ┌────────────────────────┐
    │ Data: Alice, Bob, ...  │      │ Data: Nick, Oscar, ... │
    │                        │      │                        │
    │ Local index:           │      │ Local index:           │
    │  city=NYC → [Alice]    │      │  city=NYC → [Nick]     │
    │  city=LA  → [Bob]      │      │  city=LA  → [Oscar]    │
    └────────────────────────┘      └────────────────────────┘

    Query: WHERE city = 'NYC'
    → Must query ALL partitions (scatter-gather) and merge results
```

- **Pro:** Writes are local — updating an index entry only touches one partition
- **Con:** Reads by secondary index require scatter-gather across all partitions
- **Used by:** MongoDB, Cassandra, Elasticsearch, SolrCloud

#### Term-Partitioned Index (Global Index)

A global secondary index is partitioned separately from the data, typically by the indexed term.

```
    Data partitions:                Global index partitions:
    ┌──────────────┐               ┌──────────────────────────┐
    │ Partition 1  │               │ Index Partition (city A-M)│
    │ (users A-M)  │               │  city=LA → [Bob, Oscar]  │
    └──────────────┘               └──────────────────────────┘
    ┌──────────────┐               ┌──────────────────────────┐
    │ Partition 2  │               │ Index Partition (city N-Z)│
    │ (users N-Z)  │               │  city=NYC → [Alice, Nick]│
    └──────────────┘               └──────────────────────────┘

    Query: WHERE city = 'NYC'
    → Read from one index partition (fast!)
    → But writes must update the index partition (cross-partition)
```

- **Pro:** Reads by secondary index only touch one (or few) index partitions
- **Con:** Writes require distributed transactions or async index updates (may be stale)
- **Used by:** DynamoDB (global secondary indexes), Riak (search), Google Spanner

### Rebalancing Strategies

As data grows or nodes are added/removed, partitions must be **rebalanced** — moved between nodes to maintain an even distribution (DDIA Ch.6).

| Strategy | How It Works | Pros | Cons |
|---|---|---|---|
| **Fixed number of partitions** | Create many more partitions than nodes at setup (e.g., 1000 partitions for 10 nodes). When a node joins, it takes partitions from existing nodes. | Simple, well-tested. No partition splitting needed. | Must choose partition count upfront. Too few → large partitions; too many → overhead. |
| **Dynamic partitioning** | Split a partition when it exceeds a size threshold; merge when it shrinks. | Adapts to data volume. No upfront sizing decision. | Split/merge operations add complexity. |
| **Proportional to nodes** | Each node owns a fixed number of partitions. When a new node joins, it randomly splits existing partitions and takes half. | Partition count scales with cluster size. | Randomized splitting can lead to uneven distribution. |

```
    Fixed partitions (Riak, Elasticsearch, CockroachDB initial):
    ┌──────────────────────────────────────────────────────┐
    │ 1000 partitions created at start                     │
    │                                                      │
    │ 10 nodes → 100 partitions/node                       │
    │ 20 nodes → 50 partitions/node (rebalanced)           │
    └──────────────────────────────────────────────────────┘

    Dynamic partitioning (HBase, RethinkDB):
    ┌──────────────────────────────────────────────────────┐
    │ Start with 1 partition                               │
    │ Grows → splits at threshold (e.g., 10 GB)            │
    │ Shrinks → merges below threshold                     │
    └──────────────────────────────────────────────────────┘

    Proportional to nodes (Cassandra):
    ┌──────────────────────────────────────────────────────┐
    │ Each node gets 256 vnodes (virtual partitions)       │
    │ New node → takes random slices from existing nodes   │
    └──────────────────────────────────────────────────────┘
```

### Request Routing

Once data is partitioned, the system must route each request to the correct partition. Three approaches (DDIA Ch.6):

```
    Approach 1: Client-side routing
    ┌────────┐     ┌───────┐
    │ Client │────▶│ Node  │   Client knows which partition is on which node
    └────────┘     └───────┘   (via partition map / library)

    Approach 2: Routing tier (proxy)
    ┌────────┐     ┌───────────┐     ┌───────┐
    │ Client │────▶│   Proxy   │────▶│ Node  │
    └────────┘     └───────────┘     └───────┘
                   Partition-aware load balancer

    Approach 3: Any node as router
    ┌────────┐     ┌───────┐     ┌───────┐
    │ Client │────▶│Node A │────▶│Node B │   Node A forwards to correct node
    └────────┘     └───────┘     └───────┘
```

**Coordination services:**

- **ZooKeeper** — Maintains an authoritative partition-to-node mapping. Nodes register themselves. Clients or proxies subscribe to changes. Used by HBase, Kafka (older versions), and SolrCloud.
- **Gossip protocol** — Nodes exchange partition metadata with each other via gossip. No central coordination service. Any node can route requests. Used by Cassandra and Riak.

---

## Distributed Transactions and Consensus

When data is distributed across multiple nodes, coordinating writes requires **consensus** — getting all nodes to agree (DDIA Ch.9). This is one of the hardest problems in distributed systems.

### Two-Phase Commit (2PC)

The classic protocol for distributed atomic transactions.

```
    Phase 1: PREPARE                      Phase 2: COMMIT
    ┌─────────────┐                       ┌─────────────┐
    │ Coordinator │                       │ Coordinator │
    └──────┬──────┘                       └──────┬──────┘
           │ "Can you commit?"                   │ "Commit!" (or "Abort!")
           │                                     │
    ┌──────┼──────┐                       ┌──────┼──────┐
    ▼      ▼      ▼                       ▼      ▼      ▼
  ┌───┐  ┌───┐  ┌───┐                  ┌───┐  ┌───┐  ┌───┐
  │ A │  │ B │  │ C │                  │ A │  │ B │  │ C │
  │YES│  │YES│  │YES│                  │ OK│  │ OK│  │ OK│
  └───┘  └───┘  └───┘                  └───┘  └───┘  └───┘
  Each participant                     All participants commit
  votes YES or NO.                     (or all abort if any voted NO).
  Once voted YES, it                   Coordinator writes decision to
  CANNOT abort unilaterally.           its log before sending.
```

**Problems with 2PC:**

- **Blocking protocol** — If the coordinator crashes after Phase 1 but before Phase 2, participants that voted YES are stuck: they cannot commit or abort until the coordinator recovers. They must hold locks indefinitely.
- **Single point of failure** — The coordinator is a bottleneck and a failure point.
- **High latency** — Two round-trips required for every transaction.

> **DDIA Ch.9:** 2PC is not the same as two-phase locking (2PL). 2PC is a consensus protocol for distributed commits; 2PL is a concurrency control mechanism.

### Three-Phase Commit (3PC)

An attempt to make the commit protocol non-blocking by adding a **pre-commit** phase with timeouts. In theory, 3PC avoids the blocking problem of 2PC; in practice, it is rarely used because it relies on assumptions (bounded network delays, no network partitions) that do not hold in real networks (DDIA Ch.9).

### Raft

Raft is a consensus algorithm designed to be more understandable than Paxos. It elects a **leader**, and the leader manages a **replicated log**. All writes go through the leader, which replicates log entries to followers. A write is committed once a majority of nodes have persisted it.

```
    Leader election:
    ┌───────────┐
    │  Leader   │  ← Receives all client writes
    └─────┬─────┘
          │  AppendEntries (heartbeat + log replication)
    ┌─────┼─────┐
    ▼     ▼     ▼
  ┌───┐ ┌───┐ ┌───┐
  │ F │ │ F │ │ F │  ← Followers replicate the log
  └───┘ └───┘ └───┘

    If leader fails:
    • Followers detect missing heartbeats (election timeout)
    • A follower becomes a candidate and requests votes
    • Majority vote → new leader elected
    • New leader's log is the source of truth
```

**Key properties:**
- **Leader-based** — Simplifies reasoning: one leader at a time
- **Majority quorum** — A committed entry exists on a majority of nodes, so it survives any minority failure
- **Term numbers** — Monotonically increasing terms prevent stale leaders from making progress

**Used by:** etcd, CockroachDB, TiKV (TiDB), Consul, RethinkDB

### Paxos

The original consensus algorithm (Leslie Lamport). Paxos guarantees safety (no two nodes decide on different values) in any asynchronous network with crash failures (DDIA Ch.9). It is notoriously difficult to understand and implement correctly.

**Multi-Paxos** extends basic Paxos to a sequence of values (a replicated log), similar to Raft. Many systems use Multi-Paxos or Paxos variants: Google Spanner (Paxos groups), Cassandra (lightweight transactions), Apache Zab (ZooKeeper).

### Exactly-Once Semantics and Idempotency

In distributed systems, messages may be lost or duplicated. Achieving **exactly-once** processing is hard. Practical approaches:

| Approach | How It Works |
|---|---|
| **Idempotent operations** | Design operations so that applying them multiple times has the same effect as applying them once. E.g., `SET balance = 100` is idempotent; `SET balance = balance + 10` is not. |
| **Deduplication with unique IDs** | Assign a unique ID (e.g., UUID) to each operation. The receiver tracks processed IDs and ignores duplicates. |
| **Idempotency keys** | The client sends a unique key with each request. The server checks if the key was already processed and returns the cached result. Common in payment APIs (Stripe, PayPal). |
| **Transactional outbox** | Write the message to a database table (outbox) in the same transaction as the business logic. A separate process reads the outbox and publishes to the message broker. |

```
    Without idempotency:
    Client ──POST /charge──▶ Server     (timeout, no response)
    Client ──POST /charge──▶ Server     (retry → double charge!)

    With idempotency key:
    Client ──POST /charge, key=abc123──▶ Server  (timeout)
    Client ──POST /charge, key=abc123──▶ Server  (server sees abc123
                                                   already processed,
                                                   returns cached result)
```

---

## Problems with Distributed Systems

Distributed systems face fundamental challenges that single-node systems do not (DDIA Ch.8). Understanding these failure modes is essential for building reliable distributed databases.

### Network Partitions

A network partition occurs when some nodes can communicate with each other but not with other nodes. This is not a theoretical concern — it happens regularly in production.

```
    Normal operation:              Network partition:
    ┌───┐   ┌───┐   ┌───┐        ┌───┐   ┌───┐ ┃ ┌───┐
    │ A │◀─▶│ B │◀─▶│ C │        │ A │◀─▶│ B │ ┃ │ C │
    └───┘   └───┘   └───┘        └───┘   └───┘ ┃ └───┘
    All nodes connected           C is isolated from A and B
```

**Consequences:**
- CP systems (e.g., ZooKeeper, etcd) refuse operations on the minority side to preserve consistency
- AP systems (e.g., Cassandra, DynamoDB) continue to serve reads and writes on both sides, accepting potential inconsistency
- Partitions are usually transient, but the system must handle them gracefully

### Clock Skew

In a distributed system, each node has its own clock. These clocks are never perfectly synchronized, even with NTP.

**Why clock skew matters:**

- **Last-write-wins (LWW)** relies on timestamps to resolve conflicts. If node A's clock is ahead of node B's, A's writes will always "win," even if B's write was actually later.
- **Lease expiration** — A node may believe its lease is still valid while other nodes think it has expired.
- **Transaction ordering** — Timestamp-based ordering (as in Google Spanner) requires specialized hardware (TrueTime with GPS and atomic clocks) to bound clock uncertainty.

> **DDIA Ch.8:** You cannot trust timestamps for ordering events in a distributed system unless you have very tight bounds on clock skew (like Spanner's TrueTime). Use logical clocks (Lamport timestamps, vector clocks) instead.

### Split Brain

Split brain occurs when a partitioned system has **two nodes that both believe they are the leader**. Both accept writes, leading to conflicting data that is difficult to reconcile.

```
    Before partition:                After partition (split brain):
    ┌──────────┐                     ┌──────────┐    ┌──────────┐
    │  Leader  │                     │ "Leader" │    │ "Leader" │
    │  Node A  │                     │  Node A  │    │  Node B  │
    └────┬─────┘                     └──────────┘    └──────────┘
         │                            Accepts writes  Accepts writes
    ┌────▼─────┐                      (conflict!)     (conflict!)
    │ Follower │
    │  Node B  │
    └──────────┘
```

Split brain is one of the most dangerous failure modes because it can lead to **data corruption** and **data loss** that is difficult to repair.

### Fencing Tokens and Leader Election

A **fencing token** is a monotonically increasing number issued by the lock service (e.g., ZooKeeper) when granting a lock or lease. Any request to a resource must include the fencing token; the resource rejects requests with an older token than the one it has already seen.

```
    1. Client A acquires lock, gets fencing token = 33
    2. Client A pauses (GC, network delay)
    3. Lock expires, Client B acquires lock, gets fencing token = 34
    4. Client A wakes up, sends write with token = 33
    5. Storage rejects write: 33 < 34 (stale token)

    ┌──────────┐   token=33   ┌─────────┐
    │ Client A │─────────────▶│ Storage │──▶ REJECTED (33 < 34)
    └──────────┘              └─────────┘
                                   ▲
    ┌──────────┐   token=34   ─────┘
    │ Client B │─────────────▶  ACCEPTED
    └──────────┘
```

> **DDIA Ch.8:** Fencing tokens are essential whenever a distributed lock is used to protect a resource. Without them, a client that holds an expired lock can still make writes, causing data corruption.

**Leader election** in practice typically uses a consensus system:
- **ZooKeeper** — ephemeral nodes + watches for leader election
- **etcd** — Raft-based consensus with lease-based leadership
- **Consul** — Raft consensus for leader election and service discovery

---

## Practical Patterns

### PostgreSQL Streaming Replication

PostgreSQL uses **WAL (Write-Ahead Log) streaming** for single-leader replication. The primary streams WAL records to standbys, which replay them.

```bash
# Primary: postgresql.conf
wal_level = replica                    # Enable replication WAL
max_wal_senders = 10                   # Max concurrent replication connections
wal_keep_size = 1024                   # MB of WAL to retain for slow replicas
synchronous_standby_names = 'first 1 (standby1, standby2)'
                                       # Semi-sync: wait for 1 standby

# Primary: pg_hba.conf (allow replication connections)
host  replication  replicator  10.0.0.0/8  scram-sha-256
```

```bash
# Standby: create the replica from the primary
pg_basebackup -h primary-host -D /var/lib/postgresql/17/main \
  -U replicator -Fp -Xs -P -R
#  -R flag creates standby.signal and configures primary_conninfo

# Standby: postgresql.conf (auto-configured by -R, but can customize)
primary_conninfo = 'host=primary-host port=5432 user=replicator'
hot_standby = on                       # Allow read queries on standby
```

```sql
-- Monitor replication lag on the primary
SELECT client_addr,
       state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;

-- Monitor replication lag on the standby
SELECT now() - pg_last_xact_replay_timestamp() AS replication_delay;
```

**Failover considerations:**
- Use **Patroni**, **repmgr**, or **pg_auto_failover** for automated failover
- After failover, the old primary must be re-initialized as a standby (use `pg_rewind` if the timeline has not diverged too far)
- Applications should use a connection string that points to the current primary (via DNS, virtual IP, or a proxy like PgBouncer / HAProxy)

### Cassandra Partition Key Design

In Cassandra, the partition key determines which node owns the data. Good partition key design is the most important factor for performance.

```sql
-- GOOD: sensor_id distributes writes evenly, time clustering enables range queries
CREATE TABLE sensor_data (
    sensor_id    TEXT,
    bucket       TEXT,           -- e.g., '2025-01' (monthly bucket)
    event_time   TIMESTAMP,
    reading      DOUBLE,
    PRIMARY KEY ((sensor_id, bucket), event_time)
) WITH CLUSTERING ORDER BY (event_time DESC);

-- Query: latest readings for a specific sensor in a specific month
SELECT * FROM sensor_data
WHERE sensor_id = 'temp-42' AND bucket = '2025-06'
LIMIT 100;
-- Hits exactly ONE partition → fast
```

```sql
-- BAD: single partition key value → all data on one node (hot spot)
CREATE TABLE events (
    event_type TEXT,
    event_time TIMESTAMP,
    data       TEXT,
    PRIMARY KEY (event_type, event_time)
);
-- If event_type = 'page_view' has 99% of traffic → one node is overwhelmed
```

**Partition key design rules:**
1. **High cardinality** — The partition key should have many distinct values to distribute data evenly
2. **Query-driven** — Design the partition key around your primary query pattern
3. **Bounded partition size** — Keep partitions under 100 MB. Use **time bucketing** (add a date/month component to the partition key) to prevent unbounded growth
4. **Avoid scatter-gather** — Queries that do not include the full partition key must scan all partitions (expensive)

### CockroachDB Distributed Transactions

CockroachDB provides serializable distributed transactions using a combination of Raft consensus, MVCC, and a hybrid logical clock.

```sql
-- CockroachDB: distributed transaction across ranges (partitions)
BEGIN;

-- These rows may live on different nodes / Raft groups
UPDATE accounts SET balance = balance - 500 WHERE id = 'alice';
UPDATE accounts SET balance = balance + 500 WHERE id = 'bob';

COMMIT;
-- CockroachDB uses parallel commits (optimistic 2PC):
-- 1. Write intents to both ranges
-- 2. Commit transaction record
-- 3. Resolve intents asynchronously
```

```sql
-- CockroachDB: geo-partitioning for data locality
ALTER TABLE users PARTITION BY LIST (region) (
    PARTITION us_east VALUES IN ('us-east'),
    PARTITION us_west VALUES IN ('us-west'),
    PARTITION eu      VALUES IN ('eu')
);

-- Pin partition replicas to specific regions
ALTER PARTITION us_east OF TABLE users
    CONFIGURE ZONE USING constraints = '[+region=us-east]';
ALTER PARTITION eu OF TABLE users
    CONFIGURE ZONE USING constraints = '[+region=eu]';
-- Reads and writes for EU users stay in the EU region → low latency
```

```sql
-- Monitor range distribution
SELECT range_id, lease_holder, replicas, split_enforced_until
FROM [SHOW RANGES FROM TABLE accounts];

-- Check for hot ranges
SELECT range_id, qps
FROM crdb_internal.ranges
ORDER BY qps DESC
LIMIT 10;
```

**Key CockroachDB concepts:**
- **Ranges** — CockroachDB's term for partitions (default 512 MB). Automatically split and merged.
- **Leaseholder** — The replica that serves reads and coordinates writes for a range (similar to Raft leader).
- **Closed timestamps** — Allow consistent reads from followers without contacting the leaseholder, reducing read latency.
- **Parallel commits** — An optimization of 2PC that commits in a single round-trip in the common case.

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial replication and sharding documentation |
