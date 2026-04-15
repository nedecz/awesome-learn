# Distributed Caching

## Table of Contents

1. [Overview](#overview)
   - [What Is Distributed Caching](#what-is-distributed-caching)
   - [Why Single-Node Caching Is Not Enough](#why-single-node-caching-is-not-enough)
   - [Horizontal Scalability Needs](#horizontal-scalability-needs)
   - [Core Properties of Distributed Caches](#core-properties-of-distributed-caches)
2. [Local vs Distributed Caching](#local-vs-distributed-caching)
   - [Comparison Table](#comparison-table)
   - [When to Use Each](#when-to-use-each)
   - [Hybrid Approaches — The Near-Cache Pattern](#hybrid-approaches--the-near-cache-pattern)
   - [Near-Cache Invalidation Strategies](#near-cache-invalidation-strategies)
3. [Consistent Hashing](#consistent-hashing)
   - [How It Works](#how-it-works)
   - [The Hash Ring](#the-hash-ring)
   - [Virtual Nodes](#virtual-nodes)
   - [Node Join and Leave Behavior](#node-join-and-leave-behavior)
4. [Hash Slot Partitioning](#hash-slot-partitioning)
   - [The Redis and Valkey Approach — 16384 Slots](#the-redis-and-valkey-approach--16384-slots)
   - [Slot Assignment](#slot-assignment)
   - [Slot Migration](#slot-migration)
   - [Resharding Without Downtime](#resharding-without-downtime)
5. [Replication](#replication)
   - [Leader-Follower Replication in Cache Clusters](#leader-follower-replication-in-cache-clusters)
   - [Synchronous vs Asynchronous Replication](#synchronous-vs-asynchronous-replication)
   - [Replication Lag](#replication-lag)
   - [Read-From-Replica Patterns](#read-from-replica-patterns)
6. [Cluster Topologies](#cluster-topologies)
   - [Client-Side Partitioning](#client-side-partitioning)
   - [Proxy-Based Partitioning](#proxy-based-partitioning)
   - [Cluster-Native Partitioning](#cluster-native-partitioning)
   - [Topology Comparison Table](#topology-comparison-table)
7. [Consistency Models](#consistency-models)
   - [Eventual Consistency](#eventual-consistency)
   - [Strong Consistency](#strong-consistency)
   - [Read-Your-Writes Consistency](#read-your-writes-consistency)
   - [How Consistency Models Apply to Caching](#how-consistency-models-apply-to-caching)
8. [Split-Brain and Network Partitions](#split-brain-and-network-partitions)
   - [What Happens During Partitions](#what-happens-during-partitions)
   - [Quorum-Based Approaches](#quorum-based-approaches)
   - [Minimum Replicas to Write](#minimum-replicas-to-write)
   - [CLUSTER-REQUIRE-FULL-COVERAGE](#cluster-require-full-coverage)
9. [Cross-Region and Multi-Datacenter](#cross-region-and-multi-datacenter)
   - [Active-Active Replication](#active-active-replication)
   - [Active-Passive Replication](#active-passive-replication)
   - [CRDT-Based Approaches](#crdt-based-approaches)
   - [Latency Considerations](#latency-considerations)
   - [Geo-Replication Patterns](#geo-replication-patterns)
10. [Data Serialization](#data-serialization)
    - [JSON vs MessagePack vs Protocol Buffers](#json-vs-messagepack-vs-protocol-buffers)
    - [Compression Strategies](#compression-strategies)
    - [Impact on Cache Efficiency](#impact-on-cache-efficiency)
    - [Serialization Comparison Table](#serialization-comparison-table)
11. [Connection Management](#connection-management)
    - [Connection Pooling](#connection-pooling)
    - [Multiplexing](#multiplexing)
    - [Keep-Alive and Timeouts](#keep-alive-and-timeouts)
    - [Circuit Breakers for Connections](#circuit-breakers-for-connections)
12. [Proxy Layers](#proxy-layers)
    - [Twemproxy (Nutcracker)](#twemproxy-nutcracker)
    - [Mcrouter](#mcrouter)
    - [Envoy Proxy](#envoy-proxy)
    - [HAProxy for Caching](#haproxy-for-caching)
    - [Proxy Comparison Table](#proxy-comparison-table)
13. [Scaling Strategies](#scaling-strategies)
    - [Vertical Scaling — Bigger Nodes](#vertical-scaling--bigger-nodes)
    - [Horizontal Scaling — More Nodes](#horizontal-scaling--more-nodes)
    - [Read Scaling with Replicas](#read-scaling-with-replicas)
    - [Hot Key Mitigation](#hot-key-mitigation)
14. [Failure Handling](#failure-handling)
    - [Graceful Degradation Patterns](#graceful-degradation-patterns)
    - [Circuit Breaker to Database](#circuit-breaker-to-database)
    - [Fallback Strategies](#fallback-strategies)
    - [Client-Side Retry with Backoff](#client-side-retry-with-backoff)
    - [Failure Handling Decision Tree](#failure-handling-decision-tree)

---

## Overview

### What Is Distributed Caching

Distributed caching spreads cached data across multiple nodes (servers or processes)
that work together as a single logical cache. Instead of each application instance
maintaining its own isolated in-memory store, a distributed cache provides a shared,
network-accessible layer that all instances read from and write to.

```
┌─────────────────────────────────────────────────────────────────┐
│                       APPLICATION TIER                          │
│                                                                 │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐ │
│   │  App #1  │    │  App #2  │    │  App #3  │    │  App #N  │ │
│   └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘ │
│        │               │               │               │        │
└────────┼───────────────┼───────────────┼───────────────┼────────┘
         │               │               │               │
         ▼               ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                   DISTRIBUTED CACHE CLUSTER                     │
│                                                                 │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐ │
│   │  Node A  │◄──►│  Node B  │◄──►│  Node C  │◄──►│  Node D  │ │
│   │  Keys:   │    │  Keys:   │    │  Keys:   │    │  Keys:   │ │
│   │  0-4095  │    │ 4096-8191│    │8192-12287│    │12288-    │ │
│   │          │    │          │    │          │    │   16383   │ │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Key characteristics:

- **Shared state** — all application instances see the same data
- **Partitioned storage** — data is distributed across nodes by key
- **Fault tolerant** — the cluster survives individual node failures
- **Elastically scalable** — add or remove nodes to adjust capacity

### Why Single-Node Caching Is Not Enough

A single cache node works well for small systems but breaks down as scale increases:

| Limitation             | Impact                                                  |
|------------------------|---------------------------------------------------------|
| **Memory ceiling**     | One machine has a fixed RAM limit (e.g. 64–256 GB)     |
| **Single point of failure** | Node crash = 100% cache miss rate                 |
| **Throughput bottleneck** | One CPU/NIC saturates under high request volume      |
| **Cold restart**       | Rebooting a single node wipes the entire cache          |
| **No read scaling**    | All reads funneled through one node                     |
| **Vertical cost curve** | Doubling RAM on one machine ≠ 2× the cost             |

When your system serves thousands of requests per second, stores tens of gigabytes
of cached data, or requires high availability — a single cache node is insufficient.

### Horizontal Scalability Needs

Modern applications require caching that scales horizontally:

```
Single Node (vertical scaling):       Distributed (horizontal scaling):

┌──────────────────────┐               ┌────────┐ ┌────────┐ ┌────────┐
│                      │               │ Node 1 │ │ Node 2 │ │ Node 3 │
│   256 GB RAM         │               │ 64 GB  │ │ 64 GB  │ │ 64 GB  │
│   1 million ops/sec  │      vs       │ 333K   │ │ 333K   │ │ 333K   │
│   $$$$$              │               │ ops/s  │ │ ops/s  │ │ ops/s  │
│                      │               │ $$     │ │ $$     │ │ $$     │
│   SPOF               │               └────────┘ └────────┘ └────────┘
└──────────────────────┘               Total: 192 GB, 1M ops/s, $$$$$$
                                       Fault tolerant: lose 1 node = 66%
```

Horizontal scaling provides:

- **Linear capacity growth** — each node adds memory and throughput
- **Fault isolation** — losing one node affects only a fraction of keys
- **Cost efficiency** — commodity hardware instead of expensive large instances
- **Geographic distribution** — nodes in multiple regions reduce latency

### Core Properties of Distributed Caches

Every distributed cache must address these fundamental concerns:

1. **Partitioning** — how keys map to specific nodes
2. **Replication** — how data is copied for durability and read scaling
3. **Consistency** — what guarantees clients get about freshness
4. **Failure detection** — how the cluster identifies and reacts to node failures
5. **Rebalancing** — how data moves when the cluster topology changes

---

## Local vs Distributed Caching

### Comparison Table

| Aspect                | Local (In-Process)              | Distributed (Network)            |
|-----------------------|---------------------------------|----------------------------------|
| **Latency**           | Nanoseconds (memory access)     | Microseconds–milliseconds (network) |
| **Consistency**       | Per-instance only               | Shared across all instances      |
| **Capacity**          | Limited to process heap         | Aggregate of all cluster nodes   |
| **Failure domain**    | Process crash loses cache       | Node crash loses partition       |
| **Serialization**     | Not required (native objects)   | Required (network transfer)      |
| **Warm-up**           | Each instance warms separately  | Shared warm-up across instances  |
| **Memory overhead**   | Duplicated across N instances   | Single copy per partition        |
| **Complexity**        | Minimal (library only)          | Moderate (cluster management)    |
| **GC pressure**       | Adds to application GC          | Offloaded to cache process       |
| **Invalidation**      | Local only, no coordination     | Coordinated across all clients   |

### When to Use Each

**Use local caching when:**

- Data is small, rarely changes, and tolerates staleness
- Ultra-low latency is critical (sub-microsecond)
- Each instance can independently compute/fetch the data
- Examples: configuration, static reference data, compiled templates

**Use distributed caching when:**

- Multiple application instances must share state
- Dataset exceeds single-process memory
- Cache consistency matters across instances
- Examples: session data, API responses, database query results

### Hybrid Approaches — The Near-Cache Pattern

The near-cache pattern combines both: a small local L1 cache in front of a
distributed L2 cache. This gets near-local latency for hot keys while maintaining
shared state.

```
┌───────────────────────────────────────────────────┐
│              Application Instance                  │
│                                                    │
│   ┌─────────────────────────┐                      │
│   │   L1 Near-Cache (local) │  ◄── Hit? Return     │
│   │   - 1000 entries max    │                      │
│   │   - 5 sec TTL           │                      │
│   └───────────┬─────────────┘                      │
│               │ Miss                               │
│               ▼                                    │
│   ┌─────────────────────────┐                      │
│   │   Cache Client Library  │                      │
│   └───────────┬─────────────┘                      │
└───────────────┼────────────────────────────────────┘
                │ Network call
                ▼
┌───────────────────────────────────────────────────┐
│        L2 Distributed Cache Cluster                │
│   ┌────────┐  ┌────────┐  ┌────────┐              │
│   │ Node A │  │ Node B │  │ Node C │              │
│   └────────┘  └────────┘  └────────┘              │
└───────────────────────────────────────────────────┘
```

### Near-Cache Invalidation Strategies

Keeping the L1 near-cache consistent with L2 is the main challenge:

1. **Short TTL** — L1 entries expire quickly (1–10 seconds), limiting staleness
2. **Pub/Sub invalidation** — L2 publishes invalidation events; L1 subscribes and evicts
3. **Version check** — L1 stores a version; periodically checks L2 for version bump
4. **Write-through** — writes go to L2 and invalidate all L1 instances via broadcast

```python
# Near-cache with TTL-based invalidation
class NearCache:
    def __init__(self, distributed_client, max_size=1000, l1_ttl=5):
        self.l1 = LRUCache(max_size)
        self.l2 = distributed_client
        self.l1_ttl = l1_ttl

    def get(self, key):
        # Check L1 first
        value = self.l1.get(key)
        if value is not None:
            return value

        # Fall back to L2
        value = self.l2.get(key)
        if value is not None:
            self.l1.put(key, value, ttl=self.l1_ttl)
        return value

    def put(self, key, value, ttl=None):
        # Write to L2 first (source of truth)
        self.l2.set(key, value, ttl=ttl)
        # Invalidate L1 (don't populate — let reads populate)
        self.l1.delete(key)
```

---

## Consistent Hashing

### How It Works

Consistent hashing maps both cache keys and cache nodes onto the same circular
hash space (0 to 2^32 - 1). A key is stored on the first node encountered when
walking clockwise around the ring from the key's hash position.

This approach ensures that when a node is added or removed, only a fraction of
keys (approximately 1/N) need to be remapped, rather than rehashing everything.

**Traditional hashing problem:**

```
node = hash(key) % N

If N changes from 3 to 4:
  hash("user:1001") % 3 = 2  →  hash("user:1001") % 4 = 1  ← REMAPPED
  hash("user:1002") % 3 = 0  →  hash("user:1002") % 4 = 2  ← REMAPPED
  Nearly ALL keys remap when N changes!
```

### The Hash Ring

```
                         0 / 2^32
                           │
                     ┌─────┴─────┐
                ─────             ─────
            ──                         ──
          ─                               ─
        ─        key:"session:42"           ─
       ─            ●                        ─
      ─                                       ─
     │                                         │
     │   ┌─────────┐                           │
     │   │ Node A  │◄── owns keys from         │
     │   └─────────┘    Node C clockwise       │
     │       ■              to Node A           │
      ─                                       ─
       ─                               ■     ─
        ─                          ┌───────┐─
          ─                        │Node B │
            ──                     └───────┘
                ─────         ─────
                     └───┬───┘
                         │
                    ■ Node C
                 ┌───────────┐
                 │  Node C   │
                 └───────────┘

    key:"session:42" hashes between Node C and Node A
    → stored on Node A (first node clockwise)
```

### Virtual Nodes

A problem with basic consistent hashing is uneven distribution. If nodes
hash to nearby positions on the ring, one node may be responsible for a
disproportionate key range. Virtual nodes solve this.

Each physical node is mapped to multiple positions (virtual nodes) on the ring:

```
Physical Nodes:  A, B, C
Virtual Nodes:   A1, A2, A3, A4, B1, B2, B3, B4, C1, C2, C3, C4

                         0
                         │
                    ─────┴─────
                ──  B2       A1  ──
            ── C3                A3 ──
          ─                            ─
        A2                              B4
       ─                                  ─
      C1                                   B1
     │                                      │
     │                                      │
      A4                                   C4
       ─                                  ─
        B3                              C2
          ─                            ─
            ──  A1               B2 ──
                ──── ─────── ────
                         │
                        C3

With 100+ virtual nodes per physical node, the key distribution
becomes nearly uniform (standard deviation < 5% of mean).
```

**Impact of virtual node count:**

| Virtual Nodes/Physical | Load Std Deviation | Memory Overhead |
|------------------------|--------------------|-----------------|
| 1                      | ~50% of mean       | Minimal         |
| 50                     | ~10% of mean       | Low             |
| 150                    | ~5% of mean        | Moderate        |
| 500                    | ~2% of mean        | Higher          |

### Node Join and Leave Behavior

**When a node joins:**

```
BEFORE (3 nodes):                    AFTER (4 nodes, D joins):

     A ■───────── B                      A ■────── D ■── B
     │             │                     │    (D takes    │
     │             │                     │    some of     │
     C ■───────────┘                     C ■──B's range)─┘

Only keys in the range between A and D (that previously
mapped to B) move to D. Nodes A and C are unaffected.
Keys remapped: ~1/4 (only B → D transfers)
```

**When a node leaves:**

```
BEFORE (4 nodes):                    AFTER (D leaves):

     A ■────── D ■── B                  A ■───────── B
     │                │                  │             │
     C ■──────────────┘                  C ■───────────┘

D's keys are absorbed by B (next clockwise node).
Nodes A and C are unaffected.
Keys remapped: ~1/4 (only D → B transfers)
```

---

## Hash Slot Partitioning

### The Redis and Valkey Approach — 16384 Slots

Instead of consistent hashing, Redis Cluster and Valkey use a fixed number of
hash slots (16384). Each key is mapped to a slot using:

```
slot = CRC16(key) mod 16384
```

Each node in the cluster is assigned a subset of the 16384 slots. This provides
a deterministic, pre-defined mapping from keys to nodes.

```
┌──────────────────────────────────────────────────────────┐
│                    16384 Hash Slots                       │
│                                                          │
│  0          4095  4096        8191  8192       12287     │
│  ├───────────┤    ├───────────┤    ├───────────┤        │
│  │  Node A   │    │  Node B   │    │  Node C   │        │
│  │  (leader) │    │  (leader) │    │  (leader) │        │
│  └─────┬─────┘    └─────┬─────┘    └─────┬─────┘        │
│        │               │               │                │
│  ┌─────┴─────┐    ┌─────┴─────┐    ┌─────┴─────┐        │
│  │  Node A'  │    │  Node B'  │    │  Node C'  │        │
│  │ (replica) │    │ (replica) │    │ (replica) │        │
│  └───────────┘    └───────────┘    └───────────┘        │
│                                                          │
│  12288      16383                                        │
│  ├───────────┤                                           │
│  │  Node D   │                                           │
│  │  (leader) │                                           │
│  └─────┬─────┘                                           │
│  ┌─────┴─────┐                                           │
│  │  Node D'  │                                           │
│  │ (replica) │                                           │
│  └───────────┘                                           │
└──────────────────────────────────────────────────────────┘
```

### Slot Assignment

Slots can be manually or automatically assigned:

```bash
# Manual slot assignment via redis-cli
redis-cli --cluster create \
  node-a:6379 node-b:6379 node-c:6379 \
  --cluster-replicas 1

# Check slot distribution
redis-cli cluster slots
# 1) 1) (integer) 0
#    2) (integer) 5460
#    3) 1) "node-a"
#       2) (integer) 6379
# 2) 1) (integer) 5461
#    2) (integer) 10922
#    3) 1) "node-b"
#       2) (integer) 6379
# ...
```

**Hash tags** allow related keys to land on the same slot:

```
user:{1001}:profile  → CRC16("1001") → same slot
user:{1001}:session  → CRC16("1001") → same slot
user:{1001}:cart     → CRC16("1001") → same slot

# This enables multi-key operations within a slot
```

### Slot Migration

Slot migration moves a slot from one node to another while the cluster
remains operational. The process is:

```
Step 1: Mark slot as MIGRATING on source, IMPORTING on target
Step 2: For each key in the slot:
  - DUMP key on source → serialized payload
  - RESTORE key on target with payload
  - DEL key on source
Step 3: Update cluster slot configuration

During migration:
  - Reads to existing keys → source node serves them
  - Reads to migrated keys → source returns ASK redirect → client asks target
  - Writes → redirected to target via ASK
```

### Resharding Without Downtime

```
BEFORE:  3 nodes, 16384 slots evenly distributed

  Node A: slots 0-5460      (5461 slots)
  Node B: slots 5461-10922  (5462 slots)
  Node C: slots 10923-16383 (5461 slots)

ADD Node D → reshard to redistribute:

  Node A: slots 0-4095      (4096 slots)  ← gave 1365 slots
  Node B: slots 4096-8191   (4096 slots)  ← gave 1366 slots
  Node C: slots 8192-12287  (4096 slots)  ← gave 1365 slots
  Node D: slots 12288-16383 (4096 slots)  ← received 4096 slots

Resharding command:
  redis-cli --cluster reshard node-a:6379 \
    --cluster-from <node-a-id>,<node-b-id>,<node-c-id> \
    --cluster-to <node-d-id> \
    --cluster-slots 4096
```

---

## Replication

### Leader-Follower Replication in Cache Clusters

In a distributed cache cluster, each partition (set of hash slots) has one
leader and zero or more followers (replicas). Writes always go to the leader,
which then propagates changes to followers.

```
                    ┌──────────────┐
                    │   Client     │
                    └──────┬───────┘
                           │
                    Write  │  Read
                    ┌──────┴───────┐
                    │              │
                    ▼              ▼
             ┌────────────┐  ┌────────────┐
             │   Leader   │─►│  Replica 1 │
             │  (writes)  │  │  (reads)   │
             └─────┬──────┘  └────────────┘
                   │
                   │ Replication stream
                   ▼
             ┌────────────┐
             │  Replica 2 │
             │  (reads)   │
             └────────────┘
```

### Synchronous vs Asynchronous Replication

| Aspect               | Synchronous                    | Asynchronous                  |
|-----------------------|--------------------------------|-------------------------------|
| **Write latency**    | Higher (wait for replica ACK)  | Lower (return immediately)    |
| **Data safety**      | No data loss on leader crash   | Potential data loss window     |
| **Throughput**        | Lower (bounded by slowest)     | Higher (leader not blocked)   |
| **Availability**     | Lower (replica down = stall)   | Higher (replicas independent)  |
| **Common usage**     | Rare in caching                | Default for Redis/Valkey/Memcached |

Redis and Valkey use **asynchronous replication** by default. The leader does not
wait for replicas to acknowledge writes:

```
Timeline:

  Client          Leader           Replica
    │                │                │
    │── SET k v ────►│                │
    │◄── OK ─────────│                │
    │                │── replicate ──►│  ← async, may lag
    │                │                │
    │                │   (leader crash before replication)
    │                │        ✗       │
    │                │                │
    │  ← data loss: SET k v was acknowledged but not replicated
```

### Replication Lag

Replication lag is the delay between a write on the leader and its visibility on
replicas. Monitoring lag is critical for read-from-replica architectures.

```bash
# Check replication offset on Redis/Valkey
redis-cli info replication
# master_repl_offset:1234567
# slave0:ip=10.0.0.2,port=6379,state=online,offset=1234500,lag=0

# Lag in bytes: 1234567 - 1234500 = 67 bytes behind
```

Typical replication lag sources:

- **Network latency** between leader and replica (cross-AZ: 1–2ms)
- **Leader write volume** — high write throughput can outpace replication
- **Replica load** — replica busy serving reads delays applying writes
- **Large keys** — multi-MB values take longer to transfer

### Read-From-Replica Patterns

Reading from replicas increases throughput but introduces staleness risk:

```python
# Pattern 1: Always read from leader (consistent but doesn't scale reads)
value = leader_client.get("user:1001")

# Pattern 2: Read from any replica (scales reads but may be stale)
value = replica_client.get("user:1001")

# Pattern 3: Read-your-writes — route to leader after own writes
class ReadYourWritesClient:
    def __init__(self, leader, replicas):
        self.leader = leader
        self.replicas = replicas
        self.recent_writes = {}  # key -> write_timestamp

    def set(self, key, value, ttl=None):
        self.leader.set(key, value, ex=ttl)
        self.recent_writes[key] = time.time()

    def get(self, key):
        # If we wrote recently, read from leader
        if key in self.recent_writes:
            if time.time() - self.recent_writes[key] < 2.0:  # 2s window
                return self.leader.get(key)
            del self.recent_writes[key]
        # Otherwise, read from replica
        return random.choice(self.replicas).get(key)
```

---

## Cluster Topologies

### Client-Side Partitioning

The application directly determines which cache node to contact for each key.
The partitioning logic lives in the client library.

```
┌──────────────────────────────────────────┐
│            Application Server             │
│                                           │
│  ┌─────────────────────────────────────┐  │
│  │       Client Library (Jedis,        │  │
│  │       Lettuce, ioredis, etc.)       │  │
│  │                                     │  │
│  │  key:"user:42"                      │  │
│  │  → hash("user:42") mod 3 = 1       │  │
│  │  → route to Node B                 │  │
│  └────┬──────────┬──────────┬─────────┘  │
│       │          │          │             │
└───────┼──────────┼──────────┼─────────────┘
        │          │          │
        ▼          ▼          ▼
   ┌────────┐ ┌────────┐ ┌────────┐
   │ Node A │ │ Node B │ │ Node C │
   └────────┘ └────────┘ └────────┘
```

**Pros:** No extra hop, lowest latency, no proxy bottleneck
**Cons:** All clients must agree on topology, harder to change partitioning scheme

### Proxy-Based Partitioning

A proxy sits between clients and cache nodes, handling routing transparently.
Clients connect to the proxy as if it were a single cache instance.

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  App #1  │  │  App #2  │  │  App #3  │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │
     └─────────────┼─────────────┘
                   │
                   ▼
          ┌────────────────┐
          │     Proxy       │
          │  (Twemproxy /   │
          │   Envoy /       │
          │   mcrouter)     │
          └───┬────┬────┬───┘
              │    │    │
              ▼    ▼    ▼
         ┌──────┐┌──────┐┌──────┐
         │Node A││Node B││Node C│
         └──────┘└──────┘└──────┘
```

**Pros:** Clients are simple, topology changes are transparent, connection pooling
**Cons:** Extra network hop, proxy is a potential bottleneck/SPOF

### Cluster-Native Partitioning

The cache nodes themselves form a cluster and handle routing via redirections.
Redis Cluster uses MOVED and ASK redirects to guide clients to the correct node.

```
┌──────────────────────────────────────────────────────┐
│              Application Server                       │
│  ┌────────────────────────────────────────────────┐   │
│  │         Cluster-Aware Client Library            │   │
│  │   (caches slot → node mapping locally)          │   │
│  └──────────┬──────────┬──────────┬───────────────┘   │
│             │          │          │                    │
└─────────────┼──────────┼──────────┼────────────────────┘
              │          │          │
              ▼          ▼          ▼
   ┌─────────────────────────────────────────────────┐
   │             Cache Cluster (gossip protocol)      │
   │                                                  │
   │  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
   │  │  Node A  │◄─┤  Node B  │◄─┤  Node C  │       │
   │  │slots 0-  │─►│slots 5461│─►│slots 10923│      │
   │  │    5460  │  │  -10922  │  │  -16383  │       │
   │  └──────────┘  └──────────┘  └──────────┘       │
   │                                                  │
   │  Client sends GET to wrong node:                 │
   │  → MOVED 3999 node-a:6379                        │
   │  → Client updates slot map, retries to Node A    │
   └──────────────────────────────────────────────────┘
```

**Pros:** No proxy needed, automatic failover, official support
**Cons:** Client must be cluster-aware, multi-key operations limited to same slot

### Topology Comparison Table

| Feature               | Client-Side        | Proxy-Based         | Cluster-Native     |
|-----------------------|--------------------|---------------------|--------------------|
| **Extra hop**         | No                 | Yes (proxy)         | No (after slot map)|
| **Client complexity** | High               | Low                 | Medium             |
| **Topology changes**  | Client redeploy    | Proxy config reload | Automatic gossip   |
| **Connection count**  | N clients × M nodes | N clients × 1 proxy | N × M (pooled)   |
| **Failover**          | Client-managed     | Proxy-managed       | Cluster-managed    |
| **Multi-key ops**     | Yes (same node)    | Limited             | Same slot only     |
| **Examples**          | Jedis, Lettuce     | Twemproxy, Envoy    | Redis Cluster      |

---

## Consistency Models

### Eventual Consistency

Most distributed caches provide eventual consistency by default. After a write,
replicas will eventually converge to the same value, but reads from different
nodes may return different values in the interim.

```
Timeline with eventual consistency:

  T0: Client A writes key="X", value="v2" to Leader
  T1: Leader acknowledges write to Client A
  T2: Client B reads key="X" from Replica → gets "v1" (stale!)
  T3: Replication propagates "v2" to Replica
  T4: Client B reads key="X" from Replica → gets "v2" (converged)

  Inconsistency window: T1 → T3 (typically milliseconds)
```

### Strong Consistency

Strong consistency guarantees that after a write is acknowledged, all subsequent
reads return the updated value. This is expensive in distributed caches:

- Requires synchronous replication (write waits for all replicas)
- Dramatically increases write latency
- Reduces availability (replica failure blocks writes)

**Strong consistency is rarely used in caching** because the purpose of a cache
is to trade consistency for performance. If you need strong consistency, the
source of truth (database) should be the authority.

### Read-Your-Writes Consistency

A middle ground: a client always sees its own writes, even if other clients may
see stale data. This is often sufficient for user-facing applications.

Implementation strategies:

1. **Sticky sessions** — route a user's reads to the same node they wrote to
2. **Write timestamp tracking** — after writing, read from leader until replication
   catches up
3. **Version vectors** — client sends its last-known version; node checks if it has
   that version

### How Consistency Models Apply to Caching

| Model                | Cache Use Case                                           |
|----------------------|----------------------------------------------------------|
| **Eventual**         | Product catalogs, content feeds, non-critical counters   |
| **Read-your-writes** | User profiles, shopping carts, session data              |
| **Strong**           | Inventory counts, financial balances (usually use DB)    |

**Practical guidance:** For most caching use cases, eventual consistency with short
TTLs is sufficient. Reserve stronger guarantees for the source-of-truth datastore.

---

## Split-Brain and Network Partitions

### What Happens During Partitions

A network partition divides the cluster into two or more groups of nodes that
cannot communicate with each other. This creates a split-brain scenario.

```
NORMAL:                          PARTITIONED:

┌──────┐  ┌──────┐  ┌──────┐   ┌──────┐  ┌──────┐ ║ ┌──────┐
│Node A│──│Node B│──│Node C│   │Node A│──│Node B│ ║ │Node C│
└──────┘  └──────┘  └──────┘   └──────┘  └──────┘ ║ └──────┘
                                                   ║
                                Partition A         ║ Partition B
                                (majority)          ║ (minority)

In Partition A: Nodes A and B can serve reads and writes
In Partition B: Node C is isolated

Which side should accept writes?
→ If both sides accept writes, data diverges (split-brain)
→ If only majority side accepts, minority side becomes read-only or down
```

### Quorum-Based Approaches

Quorum-based systems require a majority of nodes to agree before accepting writes:

```
Cluster of 5 nodes (quorum = 3):

  Write succeeds:  3 or more nodes acknowledge
  Write fails:     fewer than 3 nodes reachable

  During partition:
  ┌─────────────────────┐     ┌──────────────┐
  │ Partition A          │     │ Partition B   │
  │ Nodes: A, B, C (3)  │     │ Nodes: D, E   │
  │ Quorum: YES (3 ≥ 3) │     │ Quorum: NO    │
  │ → accepts writes     │     │ → rejects     │
  └─────────────────────┘     └──────────────┘
```

**Note:** Redis Cluster and Memcached do not use quorum-based writes. Redis Cluster
uses a gossip protocol and epoch-based leader election, not Raft or Paxos.

### Minimum Replicas to Write

Redis and Valkey provide the `min-replicas-to-write` configuration to prevent
writes when insufficient replicas are available:

```bash
# Require at least 1 replica to acknowledge within 10 seconds
min-replicas-to-write 1
min-replicas-max-lag 10

# Effect during partition:
# If leader cannot reach any replica → rejects writes
# This prevents data loss but reduces availability
```

```
Leader with min-replicas-to-write = 1:

  Case 1: Replica reachable           Case 2: Replica unreachable
  ┌────────┐    ┌─────────┐           ┌────────┐    ┌─────────┐
  │ Leader │───►│ Replica │           │ Leader │ ╳  │ Replica │
  │  SET OK│    │ (lag<10)│           │SET FAIL│    │(offline)│
  └────────┘    └─────────┘           └────────┘    └─────────┘
```

### CLUSTER-REQUIRE-FULL-COVERAGE

By default, Redis Cluster refuses to serve any queries if one or more hash slot
ranges are uncovered (no available leader). This is controlled by:

```bash
# Default: cluster requires full slot coverage
cluster-require-full-coverage yes

# If Node A (slots 0-5460) goes down with no replica:
# → ENTIRE cluster returns CLUSTERDOWN errors
# → Even keys on healthy nodes are inaccessible

# Alternative: allow partial operations
cluster-require-full-coverage no

# → Keys on healthy nodes remain accessible
# → Keys on down node's slots return errors
# → Better availability, but clients must handle partial failures
```

**Trade-off:**

| Setting          | `yes` (default)          | `no`                          |
|------------------|--------------------------|-------------------------------|
| **Availability** | All-or-nothing           | Partial (healthy slots work)  |
| **Consistency**  | Prevents partial state   | Clients may see gaps          |
| **Use case**     | Data integrity critical  | Availability-first systems    |

---

## Cross-Region and Multi-Datacenter

### Active-Active Replication

Both (or all) datacenters accept writes. Writes in one region replicate
asynchronously to others. Conflicts must be resolved.

```
          Region US-East                    Region EU-West
   ┌─────────────────────────┐       ┌─────────────────────────┐
   │  ┌────────┐ ┌────────┐  │       │  ┌────────┐ ┌────────┐  │
   │  │Leader A│ │Leader B│  │  ◄──► │  │Leader C│ │Leader D│  │
   │  │slots   │ │slots   │  │ async │  │slots   │ │slots   │  │
   │  │0-8191  │ │8192-   │  │ repli-│  │0-8191  │ │8192-   │  │
   │  │        │ │16383   │  │ cation│  │        │ │16383   │  │
   │  └────────┘ └────────┘  │       │  └────────┘ └────────┘  │
   │                          │       │                          │
   │  Clients write locally   │       │  Clients write locally   │
   └─────────────────────────┘       └─────────────────────────┘
```

**Conflict resolution strategies:**

- **Last-writer-wins (LWW)** — timestamp determines winner (clock skew risk)
- **CRDTs** — conflict-free data types that auto-merge
- **Application-level resolution** — custom merge logic per data type

### Active-Passive Replication

One region is the primary (accepts writes); others are read-only replicas.
Simpler but introduces write latency for remote users.

```
   Primary (US-East)                 Passive (EU-West)
   ┌────────────────┐               ┌────────────────┐
   │   Read/Write   │──── async ───►│   Read-Only    │
   │   Cache Cluster│   replication │   Cache Cluster│
   └────────────────┘               └────────────────┘
         ▲                                  ▲
         │                                  │
    US clients write here            EU clients read here
    EU clients write here too        (writes proxied to US)
    (higher latency for EU)
```

### CRDT-Based Approaches

Conflict-Free Replicated Data Types (CRDTs) enable automatic merging without
coordination. Redis Enterprise and some Valkey distributions support CRDTs for:

| CRDT Type        | Cache Use Case                     | Merge Behavior            |
|------------------|------------------------------------|---------------------------|
| **G-Counter**    | View counts, like counts           | Sum of increments         |
| **PN-Counter**   | Inventory stock levels             | Positive + negative counts|
| **LWW-Register** | User profile fields                | Last write wins           |
| **OR-Set**       | Shopping cart items                 | Union of additions        |

### Latency Considerations

```
Latency between regions (approximate):

  Same AZ:           0.1 - 0.5 ms
  Cross-AZ (same region): 1 - 2 ms
  US-East ↔ US-West:      60 - 80 ms
  US-East ↔ EU-West:      80 - 120 ms
  US-East ↔ AP-Southeast: 200 - 300 ms

Impact on cross-region caching:
  - Synchronous replication: write latency += cross-region RTT
  - Asynchronous replication: inconsistency window = replication lag
  - Local reads: always fast (same region cache hit)
```

### Geo-Replication Patterns

**Pattern 1: Follow-the-sun**
Write to the region where the user is; replicate to others.

**Pattern 2: Region-local caching with global invalidation**
Each region has its own cache. Writes invalidate across regions via a message bus.

```
   US-East Cache          EU-West Cache
   ┌────────────┐         ┌────────────┐
   │  Local     │         │  Local     │
   │  Cache     │         │  Cache     │
   └─────┬──────┘         └──────┬─────┘
         │                       │
         └───────────┬───────────┘
                     │
              ┌──────┴───────┐
              │  Global      │
              │  Invalidation│
              │  Bus (Kafka) │
              └──────────────┘
              
  Write in US-East:
  1. Update DB
  2. Update US-East cache
  3. Publish invalidation event to Kafka
  4. EU-West consumer deletes stale key
  5. Next EU-West read populates from local DB replica
```

---

## Data Serialization

### JSON vs MessagePack vs Protocol Buffers

Choosing the right serialization format impacts cache memory usage, network
bandwidth, and CPU time for encode/decode.

```python
# Example: serializing the same object

user = {
    "id": 1001,
    "name": "Jane Smith",
    "email": "jane@example.com",
    "roles": ["admin", "user"],
    "active": True,
    "login_count": 4287
}

# JSON:         ~120 bytes, human-readable
# MessagePack:   ~85 bytes, binary
# Protobuf:      ~62 bytes, binary (with schema)
```

### Compression Strategies

For large values, compression reduces memory and network usage at the cost of CPU:

```
Value Size    | Uncompressed | gzip    | lz4     | zstd
------------- |------------- |---------|---------|--------
1 KB          | 1024 B       | 650 B   | 780 B   | 620 B
10 KB         | 10240 B      | 3200 B  | 4800 B  | 3000 B
100 KB        | 102400 B     | 18000 B | 32000 B | 16000 B

Rule of thumb:
- Don't compress values < 1 KB (overhead exceeds savings)
- LZ4 for latency-sensitive workloads (fastest decompression)
- Zstd for memory-sensitive workloads (best ratio)
- gzip for compatibility (widely supported)
```

### Impact on Cache Efficiency

```
Scenario: 10 million cached objects, avg 2 KB each

Serialization      | Avg Size | Total Memory | Encode Time | Decode Time
-------------------|----------|--------------|-------------|------------
JSON               | 2.0 KB  | 19.1 GB      | 15 μs       | 12 μs
JSON + gzip        | 0.8 KB  | 7.6 GB       | 45 μs       | 30 μs
MessagePack        | 1.4 KB  | 13.4 GB      | 8 μs        | 6 μs
Protobuf           | 1.0 KB  | 9.5 GB       | 5 μs        | 3 μs
Protobuf + lz4     | 0.6 KB  | 5.7 GB       | 12 μs       | 7 μs
```

### Serialization Comparison Table

| Format          | Size   | Speed    | Schema   | Human-Readable | Language Support |
|-----------------|--------|----------|----------|----------------|------------------|
| **JSON**        | Large  | Moderate | No       | Yes            | Universal        |
| **MessagePack** | Medium | Fast     | No       | No             | Wide             |
| **Protobuf**    | Small  | Fastest  | Required | No             | Wide (codegen)   |
| **Avro**        | Small  | Fast     | Required | No             | Moderate         |
| **CBOR**        | Medium | Fast     | No       | No             | Moderate         |
| **FlatBuffers** | Small  | Fastest  | Required | No             | Moderate         |

---

## Connection Management

### Connection Pooling

Each application instance should maintain a pool of connections to the cache
cluster rather than creating a new connection per request.

```
WITHOUT pooling:                    WITH pooling:

Request 1 → connect → cmd → close  Request 1 ─┐
Request 2 → connect → cmd → close               ├─► Pool ──► Cache
Request 3 → connect → cmd → close  Request 2 ─┤   [conn1]
                                                ├─► Pool ──► Cache
Overhead: TCP handshake per req    Request 3 ─┘   [conn2]
Latency: +0.5-1ms per request
                                    Reuses connections, near-zero overhead
```

```python
# Connection pool configuration (redis-py example)
import redis

pool = redis.ConnectionPool(
    host='cache-cluster.example.com',
    port=6379,
    max_connections=50,        # Max pool size
    socket_timeout=1.0,        # Read timeout (seconds)
    socket_connect_timeout=0.5,# Connect timeout
    retry_on_timeout=True,
    health_check_interval=30,  # Periodic health check
)

client = redis.Redis(connection_pool=pool)
```

### Multiplexing

Multiplexing sends multiple commands over a single connection concurrently,
reducing the number of connections needed.

```
Without multiplexing (pipeline):    With multiplexing (RESP3):

  Conn 1: GET a → response         Conn 1: GET a  ──►  ◄── response a
  Conn 1: GET b → response                  SET b ──►  ◄── response b
  Conn 1: GET c → response                  GET c ──►  ◄── response c
                                   (all interleaved on one connection)
  Serial: 3 round trips            Concurrent: overlapping requests
```

### Keep-Alive and Timeouts

```
Timeout configuration for distributed caching:

┌─────────────────────────────────────────────────────────┐
│  connect_timeout:  200-500ms                            │
│  ├── Time to establish TCP connection                   │
│  ├── Should be short — if node is down, fail fast       │
│                                                         │
│  read_timeout (socket_timeout):  500ms-2s               │
│  ├── Time to wait for response after sending command    │
│  ├── Set based on slowest expected operation            │
│                                                         │
│  idle_timeout:  300s (server-side: timeout config)      │
│  ├── Close connections idle longer than this            │
│  ├── Prevents resource leaks                            │
│                                                         │
│  tcp_keepalive:  60s                                    │
│  ├── Send TCP keepalive probes to detect dead conns     │
│  ├── Catches half-open connections                      │
└─────────────────────────────────────────────────────────┘
```

### Circuit Breakers for Connections

When a cache node is unhealthy, a circuit breaker prevents cascading failures
by short-circuiting requests.

```
                    ┌──────────┐
                    │  CLOSED  │ ← Normal operation
                    │(allowing │   Requests pass through
                    │requests) │
                    └────┬─────┘
                         │ Failures exceed threshold
                         ▼
                    ┌──────────┐
                    │   OPEN   │ ← Fail fast
                    │(blocking │   Return error / fallback immediately
                    │requests) │   Don't send to cache
                    └────┬─────┘
                         │ After timeout period
                         ▼
                    ┌──────────┐
                    │HALF-OPEN │ ← Probe
                    │(testing) │   Allow 1 request through
                    └────┬─────┘
                    ┌────┴────┐
                Success     Failure
                    │          │
                    ▼          ▼
               ┌────────┐ ┌────────┐
               │ CLOSED │ │  OPEN  │
               └────────┘ └────────┘
```

```python
# Circuit breaker with fallback
class CacheCircuitBreaker:
    def __init__(self, cache_client, failure_threshold=5, reset_timeout=30):
        self.client = cache_client
        self.failure_threshold = failure_threshold
        self.reset_timeout = reset_timeout
        self.failure_count = 0
        self.state = "CLOSED"
        self.last_failure_time = None

    def get(self, key):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.reset_timeout:
                self.state = "HALF_OPEN"
            else:
                return None  # Fail fast, skip cache

        try:
            value = self.client.get(key)
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.failure_count = 0
            return value
        except Exception:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
            return None
```

---

## Proxy Layers

### Twemproxy (Nutcracker)

Twemproxy is a lightweight proxy for Memcached and Redis, originally built by
Twitter. It provides automatic partitioning across a pool of cache servers.

Key features:
- Consistent hashing (ketama) or modular hashing
- Connection pooling and multiplexing
- Pipelining of requests
- Supports both Memcached and Redis protocols
- No support for multi-key commands across nodes

### Mcrouter

Mcrouter is Facebook's Memcached protocol router. It handles billions of
requests per second at Facebook scale.

Key features:
- Supports complex routing (prefix-based, shadow traffic, failover)
- Warm-up from cold to hot by replicating from another pool
- Connection pooling and request batching
- Multi-cluster replication
- Route handles: AllSyncRoute, FailoverRoute, HashRoute

### Envoy Proxy

Envoy can proxy Redis and Memcached traffic with advanced observability.

Key features:
- Redis cluster support with automatic slot discovery
- Request mirroring and traffic splitting
- Built-in metrics (latency, error rates, connection count)
- mTLS between proxy and cache nodes
- Rate limiting per downstream client

### HAProxy for Caching

HAProxy can load-balance cache connections using TCP mode.

Key features:
- Health checking of cache nodes
- TCP-level load balancing
- Connection queuing during overload
- Stick tables for session affinity
- Not cache-protocol-aware (treats as raw TCP)

### Proxy Comparison Table

| Feature              | Twemproxy       | Mcrouter        | Envoy           | HAProxy         |
|----------------------|-----------------|-----------------|-----------------|-----------------|
| **Protocol**         | Redis/Memcached | Memcached       | Redis/Memcached | TCP (any)       |
| **Partitioning**     | Consistent hash | Configurable    | Cluster-aware   | Round-robin/hash|
| **Multi-key ops**    | No              | Limited         | No              | N/A             |
| **Pipelining**       | Yes             | Yes             | Yes             | N/A             |
| **Observability**    | Basic stats     | Basic stats     | Rich metrics    | Rich metrics    |
| **Config reload**    | Restart needed  | Live reload     | Live reload     | Live reload     |
| **Failover**         | Eject node      | FailoverRoute   | Health checks   | Health checks   |
| **Scale**            | Moderate        | Massive (FB)    | Large           | Large           |
| **TLS support**      | No              | No              | Yes (mTLS)      | Yes             |
| **Cluster support**  | No              | No              | Redis Cluster   | No              |

---

## Scaling Strategies

### Vertical Scaling — Bigger Nodes

Increase memory, CPU, or network bandwidth on existing nodes.

```
Before:                          After:
┌────────────────────┐           ┌────────────────────┐
│  Node A            │           │  Node A             │
│  RAM: 32 GB        │    ──►    │  RAM: 128 GB        │
│  CPU: 4 cores      │           │  CPU: 16 cores      │
│  Net: 1 Gbps       │           │  Net: 10 Gbps       │
└────────────────────┘           └─────────────────────┘
```

**When vertical scaling works:**
- Dataset fits in one large node
- Single-threaded bottleneck (Redis) means more CPU cores don't help much
- Temporary capacity need (burst traffic)

**Limits:**
- Hardware ceilings (max instance size)
- Cost grows super-linearly
- Still a single point of failure
- Downtime required to resize (usually)

### Horizontal Scaling — More Nodes

Add more nodes to the cluster and redistribute hash slots or ring positions.

```
Before (3 nodes):                    After (6 nodes):
┌────────┐┌────────┐┌────────┐      ┌────────┐┌────────┐┌────────┐
│ 32 GB  ││ 32 GB  ││ 32 GB  │      │ 32 GB  ││ 32 GB  ││ 32 GB  │
│5461 slt││5462 slt││5461 slt│      │2731 slt││2731 slt││2731 slt│
└────────┘└────────┘└────────┘      └────────┘└────────┘└────────┘
Total: 96 GB, 16384 slots            ┌────────┐┌────────┐┌────────┐
                                      │ 32 GB  ││ 32 GB  ││ 32 GB  │
                                      │2730 slt││2730 slt││2731 slt│
                                      └────────┘└────────┘└────────┘
                                      Total: 192 GB, 16384 slots
```

### Read Scaling with Replicas

Add replicas to existing leaders to handle more read traffic without resharding:

```
Before:                              After:
  Leader A (R+W)                      Leader A (Writes)
  └── Replica A1 (R)                  ├── Replica A1 (Reads)
                                      ├── Replica A2 (Reads)
                                      └── Replica A3 (Reads)

Read throughput: ~4x (1 leader + 3 replicas serving reads)
Write throughput: unchanged (leader only)
```

### Hot Key Mitigation

When a single key receives disproportionate traffic, it creates a hotspot on
one node. Strategies to mitigate:

**Strategy 1: Key splitting (fan-out)**

```
Hot key: "trending:homepage"
Split into: "trending:homepage:{1..8}"

Write: update all 8 sharded keys
Read:  randomly pick one of the 8 keys

# This spreads load across up to 8 different nodes
import random

def get_trending():
    shard = random.randint(1, 8)
    return cache.get(f"trending:homepage:{shard}")

def set_trending(data):
    for shard in range(1, 9):
        cache.set(f"trending:homepage:{shard}", data)
```

**Strategy 2: Local caching for hot keys**

```python
# Detect hot keys and cache them locally
class HotKeyAwareCache:
    def __init__(self, distributed_cache):
        self.distributed = distributed_cache
        self.local = {}
        self.access_counts = Counter()

    def get(self, key):
        self.access_counts[key] += 1

        # If key is hot, serve from local cache
        if key in self.local:
            entry = self.local[key]
            if time.time() < entry['expires']:
                return entry['value']
            del self.local[key]

        value = self.distributed.get(key)

        # Promote to local if access count exceeds threshold
        if self.access_counts[key] > 1000:
            self.local[key] = {
                'value': value,
                'expires': time.time() + 1  # 1 second local TTL
            }

        return value
```

**Strategy 3: Read from replicas**

```
Configure client to read hot keys from replicas:

  Client → hash("hot_key") → Node A (leader)
  Override: read from Node A's replicas round-robin

  This spreads read load across leader + all replicas
```

---

## Failure Handling

### Graceful Degradation Patterns

When the cache layer fails, the application should degrade gracefully rather
than cascading the failure to the database or returning errors to users.

```
┌─────────────────────────────────────────────────────────┐
│                 Degradation Ladder                       │
│                                                          │
│  Level 0: Normal operation                               │
│  ├── Cache hit rate: 95%+                                │
│  ├── DB load: minimal                                    │
│                                                          │
│  Level 1: Partial cache failure (1 node down)            │
│  ├── Cache hit rate: ~75% (affected keys miss)           │
│  ├── DB load: moderate increase                          │
│  ├── Action: redistribute slots, promote replica         │
│                                                          │
│  Level 2: Major cache failure (multiple nodes down)      │
│  ├── Cache hit rate: < 50%                               │
│  ├── DB load: high                                       │
│  ├── Action: enable request coalescing, rate limit DB    │
│                                                          │
│  Level 3: Total cache failure                            │
│  ├── Cache hit rate: 0%                                  │
│  ├── DB load: critical                                   │
│  ├── Action: serve stale data, static fallbacks,         │
│  │           shed non-critical traffic                   │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Circuit Breaker to Database

When cache is down, all traffic hits the database. A circuit breaker prevents
the database from being overwhelmed:

```
                Cache DOWN
                    │
                    ▼
┌──────────────────────────────────────────┐
│        Request Flow During Outage         │
│                                           │
│  Request ──► Cache ──► TIMEOUT            │
│              │                            │
│              ▼                            │
│  Circuit Breaker (to DB)                  │
│  ├── CLOSED: allow DB queries             │
│  ├── Track: DB response times             │
│  ├── If DB latency > threshold:           │
│  │   └── OPEN: stop DB queries            │
│  │       └── Return stale/default value   │
│  └── Periodically: HALF-OPEN test         │
│                                           │
└──────────────────────────────────────────┘
```

```python
# Database circuit breaker during cache outage
class DatabaseCircuitBreaker:
    def __init__(self, db, max_concurrent=100, latency_threshold=500):
        self.db = db
        self.max_concurrent = max_concurrent
        self.latency_threshold_ms = latency_threshold
        self.active_queries = 0
        self.state = "CLOSED"

    def query(self, sql, params=None, fallback=None):
        if self.state == "OPEN":
            return fallback  # Don't hit DB

        if self.active_queries >= self.max_concurrent:
            return fallback  # Shed load

        self.active_queries += 1
        start = time.monotonic()
        try:
            result = self.db.execute(sql, params)
            latency = (time.monotonic() - start) * 1000
            if latency > self.latency_threshold_ms:
                self._record_slow_query()
            return result
        except Exception:
            self._record_failure()
            return fallback
        finally:
            self.active_queries -= 1
```

### Fallback Strategies

| Strategy                  | Description                                | Trade-off              |
|---------------------------|--------------------------------------------|------------------------|
| **Stale data**            | Serve last-known cached value              | Data may be outdated   |
| **Default value**         | Return a sensible default                  | May not match reality  |
| **Degraded response**     | Return partial data (skip cached fields)   | Incomplete experience  |
| **Queue and retry**       | Enqueue request, retry when cache recovers | Increased latency      |
| **Direct DB (throttled)** | Hit DB with rate limiting                  | DB may still overload  |
| **Static fallback**       | Serve pre-generated static content         | Not personalized       |

### Client-Side Retry with Backoff

When cache operations fail, retries with exponential backoff prevent
overwhelming recovering nodes:

```python
import random
import time

def cache_get_with_retry(client, key, max_retries=3, base_delay=0.1):
    """
    Retry cache reads with exponential backoff and jitter.
    
    Retry 1: 100ms + jitter (0-50ms)
    Retry 2: 200ms + jitter (0-100ms)
    Retry 3: 400ms + jitter (0-200ms)
    """
    for attempt in range(max_retries + 1):
        try:
            return client.get(key)
        except ConnectionError:
            if attempt == max_retries:
                return None  # Give up, return cache miss

            delay = base_delay * (2 ** attempt)
            jitter = random.uniform(0, delay * 0.5)
            time.sleep(delay + jitter)

    return None
```

```
Backoff visualization:

  Attempt 0: immediate       ───► FAIL
  Attempt 1: ~100-150ms wait ───► FAIL
  Attempt 2: ~200-300ms wait ───► FAIL
  Attempt 3: ~400-600ms wait ───► FAIL → return None (cache miss)

  Total worst case: ~1 second before giving up
  Without jitter: all clients retry simultaneously (thundering herd)
  With jitter: retries are staggered across clients
```

### Failure Handling Decision Tree

```
Cache operation failed?
│
├── Is it a connection error?
│   ├── YES → Is circuit breaker OPEN?
│   │   ├── YES → Return fallback immediately
│   │   └── NO  → Retry with backoff (up to 3 attempts)
│   │       ├── Retry succeeded → Return value
│   │       └── All retries failed → Open circuit breaker
│   │           └── Return fallback (stale data or default)
│   │
│   └── NO (timeout, protocol error, etc.)
│       ├── Is it a read operation?
│       │   ├── YES → Return cache miss, fall through to DB
│       │   │   └── Apply DB circuit breaker if under load
│       │   └── NO (write) → Log failure, don't retry write
│       │       └── Data will be populated on next read (lazy)
│       │
│       └── Is it a cluster MOVED/ASK redirect?
│           ├── MOVED → Update slot map, retry to correct node
│           └── ASK   → Retry to indicated node (one-time)
│
└── NO → Return result normally
```

---

## Version History

| Date       | Change          | Author |
|------------|-----------------|--------|
| 2025-04-15 | Initial version | Team   |
