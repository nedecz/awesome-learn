# Distributed Systems

## Table of Contents

1. [Overview](#overview)
2. [What Are Distributed Systems](#what-are-distributed-systems)
3. [The CAP Theorem](#the-cap-theorem)
4. [Consistency Models](#consistency-models)
5. [Consensus Algorithms](#consensus-algorithms)
6. [Distributed Transactions](#distributed-transactions)
7. [Clocks and Ordering](#clocks-and-ordering)
8. [Failure Modes](#failure-modes)
9. [Replication Strategies](#replication-strategies)
10. [The FLP Impossibility](#the-flp-impossibility)
11. [Practical Considerations](#practical-considerations)
12. [Next Steps](#next-steps)

## Overview

Distributed systems are the foundation of modern large-scale applications. Any system that spans multiple networked computers — from a two-node database cluster to a planet-scale web application — must deal with the fundamental challenges of distribution: partial failure, unreliable networks, and the absence of a global clock.

### Target Audience

- Software engineers designing systems that span multiple nodes
- Architects evaluating trade-offs for distributed databases and services
- SREs operating and troubleshooting distributed infrastructure

### Scope

- Core theorems and impossibility results (CAP, FLP)
- Consistency models and their guarantees
- Consensus algorithms (Paxos, Raft)
- Distributed transaction patterns (2PC, 3PC, Saga)
- Clock synchronization and event ordering
- Failure modes and replication strategies

## What Are Distributed Systems

A distributed system is a collection of independent computers that appears to its users as a single coherent system. The nodes communicate by passing messages over a network and coordinate to achieve a common goal.

### Why Distributed Systems Exist

| Driver | Explanation |
|---|---|
| **Scalability** | A single machine has finite CPU, memory, and storage. Distribution allows horizontal scaling across many machines. |
| **Fault tolerance** | If one node fails, others continue serving requests. No single point of failure. |
| **Latency** | Placing nodes geographically close to users reduces round-trip time. |
| **Regulatory requirements** | Data residency laws may require data to be stored in specific regions. |

### Fundamental Challenges

```
 Node A                    Network                   Node B
┌──────────┐           ┌────────────┐           ┌──────────┐
│          │ ─── msg ──▶            │──── ? ───▶│          │
│          │           │  Unreliable│           │          │
│ No shared│           │  - Delay   │           │ No shared│
│ memory   │           │  - Loss    │           │ clock    │
│          │           │  - Reorder │           │          │
│          │◀─── ? ─── │  - Dupl.   │◀── msg ──│          │
└──────────┘           └────────────┘           └──────────┘
```

- **No global clock** — each node has its own clock, and they drift apart.
- **Partial failure** — some nodes can fail while others continue operating.
- **Network is unreliable** — messages can be lost, delayed, duplicated, or reordered.
- **No shared memory** — nodes must communicate exclusively through message passing.

## The CAP Theorem

The CAP theorem (Brewer, 2000) states that a distributed data store can provide at most **two out of three** guarantees simultaneously:

- **Consistency (C)** — every read receives the most recent write or an error.
- **Availability (A)** — every request receives a non-error response, without guarantee that it contains the most recent write.
- **Partition Tolerance (P)** — the system continues to operate despite network partitions between nodes.

### The CAP Triangle

```
                        Consistency (C)
                             /\
                            /  \
                           /    \
                          / CP   \
                         /systems \
                        /──────────\
                       /            \
                      /   You must   \
                     /   choose two   \
                    /                  \
                   /  CA        AP      \
                  / systems   systems    \
                 /──────────────────────── \
          Availability (A) ──────── Partition Tolerance (P)
```

Since network partitions are unavoidable in any real distributed system, the practical choice is between **CP** and **AP**.

### CAP Trade-off Examples

| System | Type | Behavior During Partition |
|---|---|---|
| **ZooKeeper** | CP | Rejects writes if a quorum is unavailable. Guarantees consistency. |
| **etcd** | CP | Raft-based consensus. Becomes unavailable without a leader quorum. |
| **Cassandra** | AP | Continues accepting reads and writes on all reachable nodes. Reconciles later. |
| **DynamoDB** | AP | Eventual consistency by default. Offers optional strongly consistent reads. |
| **Single-node PostgreSQL** | CA | Consistent and available, but not partition-tolerant (single node). |

### Beyond CAP

CAP is a useful mental model, but it oversimplifies real-world trade-offs. In practice:

- ✅ Systems make different trade-offs for different operations (e.g., DynamoDB strong vs. eventual reads)
- ✅ Partitions are rare — most of the time, all three properties hold
- ✅ Latency is a practical concern that CAP does not address (see PACELC below)

**PACELC** extends CAP: if there is a **P**artition, choose **A** or **C**; **E**lse (normal operation), choose **L**atency or **C**onsistency.

## Consistency Models

Consistency models define the rules for what values a read operation can return. Stronger models are easier to reason about but harder (and slower) to implement.

### Spectrum of Consistency

```
Strongest ◀──────────────────────────────────────────▶ Weakest

Linearizability → Sequential → Causal → Eventual
     │                │            │          │
  "Real-time       "Global      "Respects   "All replicas
   order"          total order"  causality"   converge
                                              eventually"
```

### Comparison Table

| Model | Guarantee | Performance | Example System |
|---|---|---|---|
| **Linearizability** | Reads always return the latest write; operations appear instantaneous | Slowest — requires coordination | Google Spanner (TrueTime) |
| **Sequential consistency** | All nodes see operations in the same total order, but not necessarily real-time | Slower — global ordering needed | ZooKeeper (per-session) |
| **Causal consistency** | Operations that are causally related are seen in order; concurrent operations may be seen in any order | Moderate | MongoDB (causal sessions) |
| **Read-your-writes** | A client always sees its own writes | Moderate | Session-sticky caches |
| **Eventual consistency** | All replicas converge to the same value given enough time | Fastest — no coordination | Cassandra, DynamoDB (default) |

### Strong Consistency

Strong (linearizable) consistency makes a distributed system behave as if there is only a single copy of the data. Every read returns the value of the most recent completed write.

- ✅ Easiest to reason about — behaves like a single-threaded system
- ❌ Requires coordination (consensus) on every write, adding latency
- ❌ Unavailable during network partitions (CP trade-off)

### Eventual Consistency

Eventual consistency guarantees that if no new updates are made, all replicas will eventually converge to the same value. It does **not** guarantee when.

- ✅ High availability — no coordination needed for reads or writes
- ✅ Low latency — respond immediately from the nearest replica
- ❌ Stale reads are possible — a client may see outdated data
- ❌ Conflict resolution is needed when concurrent writes diverge (last-writer-wins, CRDTs)

### Causal Consistency

Causal consistency preserves the ordering of causally related operations. If operation A could have influenced operation B, then every node sees A before B. Concurrent (unrelated) operations may be seen in any order.

- ✅ Stronger than eventual — prevents nonsensical orderings
- ✅ Does not require global coordination — more available than linearizability
- ❌ Requires tracking causal dependencies (vector clocks or similar)

## Consensus Algorithms

Consensus allows a group of distributed nodes to agree on a single value, even if some nodes fail. It is the foundation of leader election, distributed locks, and replicated state machines.

### The Consensus Problem

```
  Node 1: "propose value X"  ─┐
  Node 2: "propose value Y"  ─┤──▶  Consensus  ──▶  All agree on X (or Y)
  Node 3: "propose value X"  ─┘     Algorithm        (one value chosen)
```

Requirements:

- **Agreement** — all non-faulty nodes decide the same value
- **Validity** — the decided value was proposed by some node
- **Termination** — all non-faulty nodes eventually decide

### Paxos

Paxos (Lamport, 1998) is the foundational consensus algorithm. It operates in two phases:

1. **Prepare phase** — a proposer sends a proposal number to acceptors. Acceptors promise not to accept earlier proposals.
2. **Accept phase** — the proposer sends the value with its proposal number. Acceptors accept if they have not made a higher promise.

Paxos guarantees safety (agreement + validity) but does not guarantee liveness in all cases without additional mechanisms (leader election, timeouts).

- ✅ Proven correct — foundational result in distributed systems theory
- ❌ Notoriously difficult to understand and implement
- ❌ Multi-Paxos (repeated consensus) adds significant complexity

### Raft

Raft (Ongaro & Ousterhout, 2014) was designed as an understandable alternative to Paxos. It decomposes consensus into three sub-problems:

1. **Leader election** — one node is elected leader; it manages log replication.
2. **Log replication** — the leader appends entries to its log and replicates them to followers.
3. **Safety** — ensures that committed entries are never lost.

```
Raft Cluster (5 nodes, term 3):

  ┌──────────┐     Append     ┌──────────┐
  │ Follower │◀──── entries ──│          │
  └──────────┘                │          │
  ┌──────────┐     Append     │  Leader  │     Client
  │ Follower │◀──── entries ──│ (Node 3) │◀─── writes
  └──────────┘                │          │
  ┌──────────┐     Append     │          │
  │ Follower │◀──── entries ──│          │
  └──────────┘                └──────────┘
  ┌──────────┐                     │
  │ Follower │◀──── entries ───────┘
  └──────────┘

  Commit when majority (3 of 5) acknowledge.
```

| Property | Paxos | Raft |
|---|---|---|
| **Understandability** | Difficult | Designed for clarity |
| **Leader** | Optional (but Multi-Paxos uses one) | Required — single leader per term |
| **Log** | Not inherently log-based | Log-structured by design |
| **Adoption** | Chubby, original Spanner | etcd, CockroachDB, Consul, TiKV |

### Leader Election

Leader election ensures exactly one node acts as the coordinator at any given time:

- A node starts an election by incrementing its **term** and requesting votes.
- It wins if it receives votes from a **majority** (quorum) of nodes.
- If no winner emerges (split vote), a new election starts after a randomized timeout.

### Quorum-Based Approaches

A quorum is the minimum number of nodes that must participate in an operation for it to be valid. For a cluster of **N** nodes:

- **Write quorum (W):** number of nodes that must acknowledge a write
- **Read quorum (R):** number of nodes that must respond to a read

As long as **W + R > N**, reads are guaranteed to see the latest write.

| Configuration | W | R | Trade-off |
|---|---|---|---|
| **Strong reads** | N/2 + 1 | N/2 + 1 | Balanced — every read sees latest write |
| **Fast writes** | 1 | N | Writes are fast, reads are slow |
| **Fast reads** | N | 1 | Writes are slow, reads are fast |

## Distributed Transactions

A distributed transaction spans multiple nodes or services. Ensuring atomicity (all-or-nothing) across network boundaries is significantly harder than on a single machine.

### Two-Phase Commit (2PC)

2PC is a blocking atomic commitment protocol with two phases:

```
  Coordinator                 Participants (A, B, C)
      │                              │
      │──── 1. PREPARE ────────────▶ │  (Can you commit?)
      │                              │
      │◀─── Vote YES / NO ──────────│
      │                              │
      │  (If all vote YES)           │
      │──── 2. COMMIT ─────────────▶ │  (Do commit)
      │                              │
      │  (If any votes NO)           │
      │──── 2. ABORT ──────────────▶ │  (Roll back)
      │                              │
```

- ✅ Guarantees atomicity across multiple nodes
- ❌ **Blocking** — if the coordinator crashes after sending PREPARE but before sending COMMIT/ABORT, participants are stuck holding locks
- ❌ High latency — two round-trips minimum

### Three-Phase Commit (3PC)

3PC adds a **pre-commit** phase between the vote and the final commit to reduce blocking:

1. **Prepare** — coordinator asks participants to vote
2. **Pre-commit** — coordinator tells participants to prepare to commit (but not yet)
3. **Commit** — coordinator tells participants to finalize

- ✅ Non-blocking in theory — participants can make progress if the coordinator fails
- ❌ Still vulnerable to network partitions — may violate safety
- ❌ Rarely used in practice due to complexity and the partition problem

### Saga Pattern

The saga pattern replaces a single distributed transaction with a sequence of local transactions, each with a compensating action.

```
Saga: Place Order

  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Create   │───▶│ Reserve  │───▶│ Charge   │───▶│ Ship     │
  │ Order    │    │ Inventory│    │ Payment  │    │ Order    │
  └──────────┘    └──────────┘    └──────────┘    └──────────┘

  If "Charge Payment" fails:

  ┌──────────┐    ┌──────────┐
  │ Cancel   │◀───│ Release  │    ← Compensating transactions run in reverse
  │ Order    │    │ Inventory│
  └──────────┘    └──────────┘
```

### Comparison

| Property | 2PC | 3PC | Saga |
|---|---|---|---|
| **Atomicity** | Strong (ACID) | Strong (ACID) | Eventual (compensations) |
| **Blocking** | Yes (coordinator failure) | Reduced | No |
| **Latency** | High (2 round-trips) | Higher (3 round-trips) | Low (async steps) |
| **Isolation** | Full (locks held) | Full (locks held) | None (intermediate states visible) |
| **Partition tolerance** | Poor | Poor | Good |
| **Complexity** | Moderate | High | High (compensating logic) |
| **Practical use** | Databases (XA) | Rare | Microservices, event-driven systems |

## Clocks and Ordering

In a distributed system there is no shared global clock. Determining the order of events across nodes is a fundamental challenge.

### Physical Clocks

Physical clocks (wall clocks) on different machines drift apart due to hardware imprecision. Clock synchronization protocols like **NTP** reduce drift but cannot eliminate it.

- Typical NTP accuracy: **1–10 ms** over the internet, **< 1 ms** on a LAN
- Google's **TrueTime** provides bounded clock uncertainty (±4 ms) using GPS and atomic clocks
- ❌ Physical clocks alone are insufficient to determine causal order

### Logical Clocks (Lamport Timestamps)

Lamport timestamps (1978) assign a counter to each event. They capture the **happened-before** relation:

- Each node maintains a counter `C`.
- On a local event: `C = C + 1`.
- On sending a message: attach `C` to the message.
- On receiving a message with timestamp `T`: `C = max(C, T) + 1`.

```
  Node A          Node B          Node C
    │                │                │
  (1) a1             │                │
    │──── msg ──────▶│                │
    │              (2) b1             │
    │                │──── msg ──────▶│
    │                │              (3) c1
    │                │                │
  (4) a2             │                │

  Lamport: a1(1) → b1(2) → c1(3)
  But a2(4) and c1(3) are concurrent — Lamport cannot distinguish them.
```

- ✅ If `a → b` then `L(a) < L(b)` (happened-before implies smaller timestamp)
- ❌ If `L(a) < L(b)`, it does **not** mean `a → b` (cannot detect concurrency)

### Vector Clocks

Vector clocks assign a vector of counters (one per node) to each event. They can detect **concurrent** events.

```
  Node A              Node B              Node C
  [1,0,0] a1          [0,0,0]             [0,0,0]
    │─── msg ─────────▶│                    │
    │                [1,1,0] b1             │
    │                   │─── msg ──────────▶│
    │                   │               [1,1,1] c1
    │                   │                    │
  [2,0,0] a2           │                    │

  a2=[2,0,0] vs c1=[1,1,1] → concurrent (neither dominates)
  a1=[1,0,0] vs b1=[1,1,0] → a1 happened before b1
```

- ✅ Can determine if two events are causally related or concurrent
- ❌ Vector size grows with the number of nodes — does not scale to large clusters

### Hybrid Logical Clocks (HLC)

Hybrid logical clocks combine physical time with a logical counter. They provide the causal ordering guarantees of logical clocks while staying close to real time.

- Used in **CockroachDB** and **MongoDB** for causal consistency without TrueTime hardware
- ✅ Fixed-size (does not grow with number of nodes like vector clocks)
- ✅ Captures causality while remaining close to wall-clock time

### Clock Comparison

| Clock Type | Detects Causality | Detects Concurrency | Scalability | Real-Time Meaning |
|---|---|---|---|---|
| **Physical** | No | No | Excellent | Yes |
| **Lamport** | Partial (one direction) | No | Excellent | No |
| **Vector** | Yes | Yes | Poor (grows with N) | No |
| **HLC** | Yes | Partial | Good (fixed size) | Approximate |

## Failure Modes

Understanding how systems fail is essential for designing fault-tolerant distributed systems.

### Network Partitions

A network partition occurs when some nodes can communicate with each other but not with other nodes in the cluster.

```
  ┌──────────────────────┐       ┌──────────────────────┐
  │   Partition A        │  ✕✕✕  │   Partition B        │
  │                      │  ✕✕✕  │                      │
  │  Node 1   Node 2    │  ✕✕✕  │  Node 3   Node 4    │
  │  Node 5             │       │                      │
  │                      │       │                      │
  │  (majority: 3 nodes) │       │  (minority: 2 nodes) │
  └──────────────────────┘       └──────────────────────┘

  Majority partition can continue operating (with quorum).
  Minority partition must stop accepting writes to avoid split-brain.
```

### Byzantine Failures

A Byzantine failure occurs when a node behaves arbitrarily — it may send conflicting information to different nodes, lie about its state, or act maliciously.

- **Byzantine Fault Tolerance (BFT)** requires at least **3f + 1** nodes to tolerate **f** Byzantine failures
- Most internal distributed systems (databases, coordination services) assume non-Byzantine (crash-stop) failures
- BFT is essential in trustless environments (blockchain, multi-party systems)

### Crash Failures

A crash failure occurs when a node stops operating entirely and never recovers (crash-stop) or may eventually recover (crash-recovery).

| Failure Model | Behavior | Tolerance Requirement |
|---|---|---|
| **Crash-stop** | Node halts permanently | 2f + 1 nodes for f failures |
| **Crash-recovery** | Node halts then restarts (with persistent state) | 2f + 1 nodes for f failures |
| **Byzantine** | Node behaves arbitrarily | 3f + 1 nodes for f failures |
| **Omission** | Node fails to send or receive some messages | 2f + 1 nodes for f failures |

### Split-Brain

Split-brain occurs when a network partition causes two sub-clusters to each believe they are the primary, both accepting writes independently. This leads to data divergence.

Prevention strategies:

- ✅ **Quorum-based systems** — only the majority partition can elect a leader
- ✅ **Fencing tokens** — a monotonically increasing token invalidates stale leaders
- ✅ **STONITH (Shoot The Other Node In The Head)** — forcibly power off the other node
- ✅ **Lease-based leadership** — leader must periodically renew its lease

## Replication Strategies

Replication copies data across multiple nodes for fault tolerance and read scalability. The strategy chosen determines consistency, availability, and conflict resolution.

### Single-Leader Replication

One node (the leader) accepts all writes and replicates them to followers.

```
  Clients
    │ writes
    ▼
  ┌──────────┐   replication   ┌──────────┐
  │  Leader  │────────────────▶│ Follower │ ── reads
  │  (primary)│────────────────▶│ Follower │ ── reads
  │          │────────────────▶│ Follower │ ── reads
  └──────────┘                 └──────────┘
```

- ✅ No write conflicts — all writes go through a single node
- ✅ Simple to reason about
- ❌ Leader is a bottleneck and single point of failure (until failover)
- ❌ Failover can cause data loss if replication is asynchronous

### Multi-Leader Replication

Multiple nodes accept writes independently. Each leader replicates its changes to all other leaders.

- ✅ Tolerates the failure of any single datacenter
- ✅ Writes can be accepted in multiple geographic regions (low latency)
- ❌ Write conflicts are possible and must be resolved (last-writer-wins, merge, custom logic)
- ❌ Significantly more complex than single-leader

### Leaderless Replication

Every node accepts reads and writes. Clients send writes to multiple nodes in parallel and read from multiple nodes to detect stale data.

- ✅ No leader — no failover needed
- ✅ High availability — any node can serve requests
- ❌ Requires quorum reads/writes (W + R > N) for consistency
- ❌ Conflict resolution needed (vector clocks, read repair, anti-entropy)

### Replication Trade-offs

| Strategy | Write Latency | Read Scalability | Consistency | Conflict Handling | Example Systems |
|---|---|---|---|---|---|
| **Single-leader** | Low (local) | High (read replicas) | Strong (sync) or eventual (async) | None | PostgreSQL, MySQL, MongoDB |
| **Multi-leader** | Low (local) | High | Eventual | Required (LWW, merge) | CockroachDB, Cassandra (multi-DC) |
| **Leaderless** | Moderate (quorum) | High | Tunable (quorum) | Required (read repair) | Cassandra, DynamoDB, Riak |

### Synchronous vs. Asynchronous Replication

| Type | Behavior | Trade-off |
|---|---|---|
| **Synchronous** | Leader waits for follower acknowledgment before confirming write | Durable but slower. Follower failure blocks writes. |
| **Asynchronous** | Leader confirms write immediately; follower replicates later | Fast but risks data loss on leader failure. |
| **Semi-synchronous** | Leader waits for one follower; rest replicate async | Balance of durability and performance. |

## The FLP Impossibility

The FLP impossibility result (Fischer, Lynch, Paterson, 1985) proves that in an asynchronous distributed system where even **one** node can crash, there is **no** deterministic algorithm that guarantees consensus will be reached in bounded time.

### Implications

- No consensus algorithm can guarantee both **safety** and **liveness** in a fully asynchronous system with crash failures.
- In practice, systems work around FLP by using **timeouts** (partial synchrony assumption), **randomization**, or **failure detectors** to eventually make progress.
- Paxos and Raft guarantee safety (never decide two different values) but rely on timeouts for liveness.

### Why It Matters

- ❌ You cannot build a "perfect" consensus algorithm — there will always be edge cases where progress stalls.
- ✅ Understanding FLP helps set realistic expectations for system behavior under failure.
- ✅ Practical systems use partial synchrony (bounded but unknown message delays) to sidestep FLP.

## Practical Considerations

### Real-World Distributed System Challenges

| Challenge | Description | Mitigation |
|---|---|---|
| **Clock skew** | Node clocks drift apart | NTP, TrueTime, logical clocks |
| **Thundering herd** | Many clients retry simultaneously after a failure | Jitter, exponential backoff |
| **Hot partitions** | Uneven data distribution causes some nodes to be overloaded | Consistent hashing, partition-aware routing |
| **Cascading failures** | One failing component triggers failures in dependent systems | Circuit breakers, bulkheads, load shedding |
| **Data migration** | Moving data between nodes during rebalancing | Online migrations, dual-write strategies |
| **Observability** | Debugging across many nodes is difficult | Distributed tracing (OpenTelemetry), correlated logs |

### Guidelines

- ✅ Design for failure — assume any component can fail at any time
- ✅ Use idempotent operations — retries should be safe
- ✅ Prefer eventual consistency unless strong consistency is strictly required
- ✅ Instrument everything — distributed tracing and metrics are essential for debugging
- ✅ Test failure scenarios — use chaos engineering (network partitions, node kills, clock skew injection)
- ❌ Do not assume the network is reliable, latency is zero, or bandwidth is infinite
- ❌ Do not design for single-node semantics and hope it works distributed
- ❌ Do not ignore the Fallacies of Distributed Computing

### The Eight Fallacies of Distributed Computing

1. The network is reliable.
2. Latency is zero.
3. Bandwidth is infinite.
4. The network is secure.
5. Topology does not change.
6. There is one administrator.
7. Transport cost is zero.
8. The network is homogeneous.

## Next Steps

Continue to the next topic to explore additional system design patterns, including load balancing strategies, caching architectures, and data partitioning techniques.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-01-10 | Initial version — CAP theorem, consistency models, consensus, distributed transactions, clocks, failure modes, replication |
