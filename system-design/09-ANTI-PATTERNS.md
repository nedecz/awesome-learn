# System Design Anti-Patterns

## Table of Contents

1. [Overview](#overview)
2. [Premature Optimization](#premature-optimization)
3. [Single Point of Failure](#single-point-of-failure)
4. [Ignoring the CAP Theorem](#ignoring-the-cap-theorem)
5. [Monolithic Database](#monolithic-database)
6. [Synchronous Everything](#synchronous-everything)
7. [Cache Without Strategy](#cache-without-strategy)
8. [No Capacity Planning](#no-capacity-planning)
9. [Tight Coupling](#tight-coupling)
10. [Ignoring Failure Modes](#ignoring-failure-modes)
11. [Poor Data Partitioning](#poor-data-partitioning)
12. [Anti-Pattern Summary](#anti-pattern-summary)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

## Overview

This document catalogs the most common system design anti-patterns — architectural and operational mistakes that undermine the reliability, scalability, and maintainability of distributed systems. For each anti-pattern, we describe the problem, the symptoms, and the correct approach.

### Target Audience

- Software architects designing large-scale systems
- Engineers preparing for system design interviews
- Teams scaling applications from prototype to production
- Technical leads reviewing architectural proposals

### Scope

- Scalability anti-patterns (premature optimization, monolithic database)
- Availability anti-patterns (single point of failure, ignoring failure modes)
- Data anti-patterns (CAP theorem violations, poor partitioning)
- Communication anti-patterns (synchronous chains, tight coupling)
- Operational anti-patterns (no capacity planning, cache mismanagement)

## Premature Optimization

### The Problem

Designing for millions of users on day one when the system has a hundred. Over-engineering adds complexity, slows development, and solves problems that may never materialize.

```
❌ Premature Optimization

  Day 1 — 100 users:

  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
  │  CDN     │──▶│  Load    │──▶│  3 App   │──▶│  Sharded │
  │  (global)│   │  Balancer│   │  Servers  │   │  DB (4x) │
  └──────────┘   └──────────┘   └──────────┘   └──────────┘
       │                              │               │
       │         ┌──────────┐         │          ┌────▼─────┐
       └────────▶│  Redis   │◀────────┘          │  Kafka   │
                 │  Cluster │                    │  Cluster │
                 └──────────┘                    └──────────┘

  6 components to serve 100 users → months to build, hard to debug

✅ Right-Sized for Day 1

  ┌──────────┐   ┌──────────┐
  │  App     │──▶│ Database │
  │  Server  │   │ (single) │
  └──────────┘   └──────────┘

  Simple, fast to build, easy to debug
  Scale when you have evidence you need to
```

### Symptoms

- Months spent on infrastructure before the first feature ships
- Sharded databases with only a few thousand rows per shard
- Complex event-driven architectures for simple CRUD operations
- More infrastructure code than application code
- The team cannot explain which bottleneck the complexity solves

### The Correct Approach

- ✅ Start simple — a single server and a single database handle more than most teams expect
- ✅ Design for the next 10x of traffic, not the next 1000x
- ✅ Instrument first — add metrics and profiling to identify real bottlenecks
- ✅ Scale the bottleneck, not the entire system
- ❌ Don't add caching, sharding, or message queues until measurements demand them

## Single Point of Failure

### The Problem

A single point of failure (SPOF) is any component whose failure brings down the entire system. Common SPOFs include a single database instance, a single region deployment, or a single load balancer.

```
❌ Single Points of Failure              ✅ Redundancy at Every Layer

┌──────────┐                             ┌──────────┐
│  Client  │                             │  Client  │
└────┬─────┘                             └────┬─────┘
     │                                        │
┌────▼─────┐                             ┌────▼─────┐
│  Single  │   ← SPOF                   │  DNS     │──→ Region A + Region B
│  Server  │                             │  (multi) │
└────┬─────┘                             └────┬─────┘
     │                                        │
┌────▼─────┐                          ┌──────▼──────┐
│  Single  │   ← SPOF                │  LB (active/ │
│  Database│                          │  standby)    │
└──────────┘                          └──────┬───────┘
                                             │
If the server dies,               ┌──────────┼──────────┐
everything is down.               │          │          │
                                  ▼          ▼          ▼
                              Server A   Server B   Server C
                                  │          │          │
                                  └─────┬────┘──────────┘
                                        │
                                  ┌─────▼──────┐
                                  │ DB Primary │──▶ DB Replica
                                  └────────────┘
```

### Symptoms

- A single server restart causes user-facing downtime
- A database failure means total system outage with no automatic recovery
- One availability zone going offline takes down the entire application
- No failover tested — recovery procedures exist only on paper
- The DNS record points to a single IP address

### The Correct Approach

- ✅ Run at least 2 instances of every critical component
- ✅ Use database replicas with automatic failover (primary-replica)
- ✅ Deploy across multiple availability zones at minimum
- ✅ Use health checks and automated recovery (restart, replace, failover)
- ✅ Test failover regularly — an untested backup is not a backup
- ❌ Don't assume "it won't go down" — everything fails eventually

## Ignoring the CAP Theorem

### The Problem

The CAP theorem states that a distributed system can provide at most two of three guarantees: Consistency, Availability, and Partition Tolerance. Since network partitions are inevitable, every system must choose between consistency and availability during a partition. Treating both as guaranteed leads to incorrect assumptions and broken designs.

```
❌ Assuming Both C and A During a Partition

  ┌──────────┐         network         ┌──────────┐
  │  Node A  │ ──── partition ──── ✗   │  Node B  │
  │          │                         │          │
  │ balance: │    Can't replicate      │ balance: │
  │  $100    │    the write            │  $100    │
  └──────────┘                         └──────────┘

  User writes to Node A: balance = $80
  User reads from Node B: balance = $100    ← stale / inconsistent!

  You cannot have both — you must choose:

✅ CP — Reject the request until partition heals (consistent but unavailable)
✅ AP — Serve stale data and reconcile later (available but inconsistent)
```

### Symptoms

- Data discrepancies across regions that go undetected for hours
- Users see different account balances depending on which server handles the request
- The system returns success but silently drops writes during a network partition
- No explicit decision documented for consistency vs. availability per feature
- Engineers assume cross-region replication is instant

### The Correct Approach

- ✅ Acknowledge that network partitions will happen
- ✅ Decide per feature: do you need CP (banking transactions) or AP (social media likes)?
- ✅ Use strong consistency (CP) for financial data, inventory, and identity
- ✅ Use eventual consistency (AP) for analytics, recommendations, and feeds
- ✅ Document the consistency model for every data store and feature
- ❌ Don't design as if all three guarantees are always achievable

## Monolithic Database

### The Problem

As a system grows to serve multiple domains (orders, users, inventory, analytics), keeping everything in a single database creates a bottleneck. Schema changes, performance issues, and scaling limitations affect the entire system.

```
❌ Monolithic Database                    ✅ Purpose-Built Databases

┌──────────┐ ┌──────────┐ ┌──────────┐  ┌──────────┐ ┌──────────┐ ┌──────────┐
│  Order   │ │  User    │ │ Analytics│  │  Order   │ │  User    │ │ Analytics│
│  Service │ │  Service │ │  Service │  │  Service │ │  Service │ │  Service │
└────┬─────┘ └────┬─────┘ └────┬─────┘  └────┬─────┘ └────┬─────┘ └────┬─────┘
     │            │            │              │            │            │
     └────────┬───┘────────────┘         ┌────▼─────┐ ┌───▼──────┐ ┌──▼───────┐
              │                          │ Postgres │ │ Postgres │ │ ClickHouse│
       ┌──────▼──────┐                  │ (orders) │ │ (users)  │ │(analytics)│
       │  Single     │                  └──────────┘ └──────────┘ └──────────┘
       │  Giant DB   │
       │             │                  Each service owns its data.
       │ 500 tables  │                  Schema changes are isolated.
       │ 2TB data    │                  Right tool for the right job.
       └─────────────┘

  A migration locks all 500 tables.
  Analytics queries slow down orders.
```

### Symptoms

- Schema migrations require a maintenance window affecting all services
- A slow analytics query causes order processing latency to spike
- Cannot scale reads for one domain without scaling the entire database
- Teams are afraid to change the schema because it might break other services
- The database has hundreds of tables with unclear ownership

### The Correct Approach

- ✅ Decompose by domain — each service or domain owns its own database
- ✅ Choose the right database for the workload (OLTP vs. OLAP vs. search)
- ✅ Share data through APIs or events, not through shared tables
- ✅ Start with logical separation (schemas) and evolve to physical separation
- ❌ Don't let multiple services write to the same tables
- ❌ Don't use a transactional database for analytics — use a data warehouse

## Synchronous Everything

### The Problem

When every operation in the system uses synchronous request-response, a chain of blocking calls creates high latency, tight coupling, and cascading failures. One slow or failed service blocks the entire chain.

```
❌ Synchronous Chain (Blocking)

  User → API Gateway → Order Service → Payment Service → Fraud Service
                                                              │
                              Total latency: 50 + 200 + 300 + 500 = 1050ms
                              If Fraud Service is down → everything fails

✅ Async with Message Queue

  User → API Gateway → Order Service → 202 Accepted (50ms)
                             │
                             ▼
                       ┌───────────┐
                       │  Message  │
                       │  Queue    │
                       └─────┬─────┘
                             │
                    ┌────────┼────────┐
                    ▼        ▼        ▼
               Payment   Fraud    Notification
               Worker    Worker   Worker

  User gets a fast response. Processing continues in the background.
  If Fraud Worker is down, messages wait in the queue and are retried.
```

### Symptoms

- End-to-end latency grows linearly as more services are added to the chain
- A single slow downstream service causes timeout cascades across the system
- The system cannot absorb traffic spikes — no buffering between components
- Every service must be available for any request to succeed
- Users wait seconds for operations that could be acknowledged immediately

### The Correct Approach

- ✅ Use asynchronous processing for anything that doesn't need an immediate result
- ✅ Return `202 Accepted` and process in the background for long-running operations
- ✅ Use message queues (Kafka, SQS, RabbitMQ) to decouple producers and consumers
- ✅ Keep synchronous calls to 2–3 hops maximum on the critical path
- ❌ Don't chain 4+ synchronous services — latency and failure risk compound
- ❌ Don't make the user wait for operations like email, analytics, or notifications

## Cache Without Strategy

### The Problem

Adding a cache without defining invalidation rules, eviction policies, or failure handling. Caching blindly leads to stale data, thundering herd problems, and cache avalanches.

```
❌ Cache Without Strategy

  1. Cache entry expires
  2. 10,000 concurrent requests arrive
  3. ALL hit the database simultaneously (thundering herd)

  Time ─────────────────────────────────────────────▶
       ┌────────┐
       │ Cache  │ TTL expires
       │ MISS   │─────────┐
       └────────┘         │
                          ▼
                    ┌───────────┐
                    │ Database  │ ← 10,000 queries at once
                    │  (DOWN)   │
                    └───────────┘

✅ Cache with Stampede Protection

  1. Cache entry expires
  2. ONE request fetches from DB (lock/lease)
  3. Other requests wait or get stale data

  Time ─────────────────────────────────────────────▶
       ┌────────┐
       │ Cache  │ TTL expires
       │ MISS   │─────────┐
       └────────┘         │
                    ┌─────▼─────┐
                    │  Lock /   │  1 request → DB
                    │  Lease    │  9,999 wait or get stale
                    └─────┬─────┘
                          │
                    ┌─────▼─────┐
                    │ Database  │ ← 1 query
                    │  (fine)   │
                    └───────────┘
```

### Symptoms

- Users see stale data for minutes or hours after an update
- Database load spikes every time a popular cache entry expires
- Cache hit rate is low despite large cache allocation
- No monitoring on cache hit/miss ratio or eviction rate
- The application breaks when the cache is unavailable (no fallback)

### The Correct Approach

- ✅ Define a TTL for every cache entry — unbounded caches serve stale data forever
- ✅ Use cache stampede protection (locking, request coalescing, or probabilistic early expiry)
- ✅ Implement cache-aside with explicit invalidation on writes
- ✅ Monitor hit rate, miss rate, eviction rate, and latency
- ✅ Design the application to function (slower) when the cache is unavailable
- ❌ Don't cache without an invalidation strategy — "cache everything forever" is not a strategy
- ❌ Don't set the same TTL for all entries — hot keys need different treatment than cold keys

## No Capacity Planning

### The Problem

Not estimating future growth, not understanding current resource utilization, and not preparing for traffic spikes. The system works fine until it doesn't — and by then it's too late.

```
❌ No Capacity Planning

  Traffic ──────────────────────────────────────────▶

  Normal          Spike (Black Friday, viral event)
  ───────         ─────────────────────────────────
  100 req/s       10,000 req/s

  ┌──────────┐    ┌──────────┐
  │ 2 servers│    │ 2 servers│ ← same capacity
  │  (fine)  │    │  (DOWN)  │    can't handle 100x
  └──────────┘    └──────────┘

✅ With Capacity Planning

  ┌──────────────────────────────────────────────────┐
  │  Capacity Plan                                   │
  │                                                  │
  │  Current:  100 req/s    │  Headroom: 3x (300)    │
  │  Growth:   10% monthly  │  Spike: 100x planned   │
  │  Auto-scale threshold:  │  70% CPU → add nodes   │
  │  Load test quarterly    │  Chaos test monthly    │
  └──────────────────────────────────────────────────┘
```

### Symptoms

- The system goes down during predictable traffic peaks (Black Friday, product launches)
- No one knows the current CPU, memory, or database utilization baseline
- Auto-scaling is not configured or has never been tested
- Storage runs out with no warning — disks fill up overnight
- Cost surprises — bills spike because scaling is reactive, not planned

### The Correct Approach

- ✅ Estimate storage, bandwidth, and QPS for the next 6–12 months
- ✅ Run load tests regularly to find the breaking point before users do
- ✅ Configure auto-scaling with tested thresholds and limits
- ✅ Set up alerts for resource utilization (CPU > 70%, disk > 80%, memory > 75%)
- ✅ Plan for known traffic spikes — pre-scale before the event, not during
- ❌ Don't assume "the cloud scales automatically" — auto-scaling has limits and lag

## Tight Coupling

### The Problem

Services with deep dependencies on each other's internals — shared schemas, shared libraries with business logic, or direct database access. A change in one service forces changes in many others.

```
❌ Tight Coupling                         ✅ Loose Coupling

┌──────────┐    ┌──────────┐             ┌──────────┐    ┌──────────┐
│ Service A│    │ Service B│             │ Service A│    │ Service B│
│          │    │          │             │          │    │          │
│ imports  │    │ imports  │             │ calls    │    │ exposes  │
│ B's data │    │ A's data │             │ B's API  │    │ stable   │
│ model    │    │ model    │             │ contract │    │ API      │
└────┬─────┘    └────┬─────┘             └──────────┘    └──────────┘
     │               │
     └───────┬───────┘                   Services communicate through
             │                           versioned APIs or events.
      ┌──────▼──────┐                    Internal models are private.
      │  Shared DB  │
      │  Shared     │
      │  Models     │
      └─────────────┘

  Changing a column in the shared
  schema breaks both services.
```

### Symptoms

- A schema change in one service requires code changes in 3+ other services
- Services share internal data models or business logic libraries
- Deploying one service requires redeploying others at the same time
- Teams cannot work independently — every change requires cross-team coordination
- A "simple" refactor has a blast radius across the entire system

### The Correct Approach

- ✅ Communicate through versioned APIs or published events, not shared internals
- ✅ Each service owns its data model — no shared schemas or direct DB access
- ✅ Use consumer-driven contract testing to catch breaking changes early
- ✅ Apply the dependency inversion principle — depend on abstractions, not implementations
- ❌ Don't share libraries that contain business logic across services
- ❌ Don't let Service A query Service B's database directly

## Ignoring Failure Modes

### The Problem

Designing only for the happy path — when all services are up, the network is fast, and every request succeeds. In production, dependencies fail, networks partition, and disks fill up. A system without failure handling collapses at the first unexpected event.

```
❌ Happy Path Only                        ✅ Designed for Failure

  Request → Service A → Service B         Request → Service A → Circuit Breaker
                              │                                       │
                         Service B                            ┌───────┴───────┐
                         is down                              │               │
                              │                          Service B       Fallback
                         No timeout                      (healthy)      (cached /
                         No retry                                        default)
                         No fallback                          │
                              │                          Response with
                         Thread hangs                     full or partial
                         forever                          data
                              │
                         All threads
                         exhausted
                              │
                         Service A
                         is now down too
```

### Symptoms

- A single downstream failure cascades and takes down the entire system
- Services hang indefinitely when a dependency is slow (no timeouts configured)
- No circuit breakers — failed services receive traffic until they are overwhelmed
- Retries without backoff amplify failures instead of recovering from them
- No fallback behavior — partial data is never returned, only errors

### The Correct Approach

- ✅ Add timeouts to every external call — a hanging request is worse than a failed one
- ✅ Implement circuit breakers to stop calling a failing dependency
- ✅ Use retries with exponential backoff and jitter for transient failures
- ✅ Design fallback responses for non-critical dependencies (cached data, defaults)
- ✅ Use bulkheads to isolate failures — one slow dependency shouldn't consume all threads
- ✅ Practice chaos engineering — inject failures to validate resilience
- ❌ Don't assume dependencies are always available — they are not

## Poor Data Partitioning

### The Problem

Choosing the wrong partition key, creating hot partitions, or forcing frequent cross-partition queries. Poor partitioning negates the benefits of horizontal scaling and creates bottlenecks.

```
❌ Poor Partition Key (Hot Partition)

  Partition Key: date

  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ 2026-01-01   │  │ 2026-01-02   │  │ 2026-01-03   │
  │              │  │              │  │  (today)     │
  │  idle        │  │  idle        │  │  ALL traffic │ ← hot partition
  │              │  │              │  │  100% load   │
  └──────────────┘  └──────────────┘  └──────────────┘

✅ Good Partition Key (Even Distribution)

  Partition Key: user_id

  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ Users A-H    │  │ Users I-P    │  │ Users Q-Z    │
  │              │  │              │  │              │
  │  ~33% load   │  │  ~33% load   │  │  ~33% load   │
  │              │  │              │  │              │
  └──────────────┘  └──────────────┘  └──────────────┘

  Traffic is evenly distributed. Each partition handles a fair share.
```

### Symptoms

- One partition is at 100% capacity while others are nearly idle
- Cross-partition queries are frequent and slow (scatter-gather on every read)
- Resharding is needed frequently because the partition key doesn't scale
- Latency percentiles (p99) are much worse than the median (p50)
- A single celebrity user or popular item overwhelms one partition

### The Correct Approach

- ✅ Choose a partition key with high cardinality and even distribution (user_id, order_id)
- ✅ Avoid time-based keys as the sole partition key — today always gets all the traffic
- ✅ Use composite keys to break up hot partitions (user_id + timestamp)
- ✅ Design access patterns first, then choose the partition key to match
- ✅ Monitor partition-level metrics — size, throughput, and hotness per partition
- ❌ Don't use low-cardinality keys (country, status) as the partition key
- ❌ Don't design queries that must touch every partition for a single request

## Anti-Pattern Summary

| Anti-Pattern | Root Cause | Fix |
|---|---|---|
| Premature optimization | Designing for scale before measuring | Start simple, instrument, scale the bottleneck |
| Single point of failure | No redundancy in critical components | Replicate, multi-AZ, automated failover |
| Ignoring CAP theorem | Treating C + A as both guaranteed | Choose CP or AP per feature, document the model |
| Monolithic database | One DB for all domains | Database per domain, right tool for the job |
| Synchronous everything | Blocking calls on every operation | Async processing, message queues, 202 Accepted |
| Cache without strategy | No invalidation, no eviction plan | TTLs, stampede protection, hit/miss monitoring |
| No capacity planning | No growth estimates, no load tests | Estimate, load test, auto-scale, alert on utilization |
| Tight coupling | Shared schemas, shared internals | Versioned APIs, events, private data models |
| Ignoring failure modes | Happy-path-only design | Timeouts, circuit breakers, retries, fallbacks |
| Poor data partitioning | Wrong shard key, hot partitions | High-cardinality keys, composite keys, monitor hotness |

## Next Steps

Return to [Best Practices](08-BEST-PRACTICES.md) for guidance on how to design systems correctly, or revisit [Scalability](01-SCALABILITY.md), [Caching Strategies](03-CACHING-STRATEGIES.md), and [Reliability](06-RELIABILITY.md) for deeper treatment of individual topics.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-01-15 | Initial release — premature optimization, SPOF, CAP theorem, monolithic database |
| 1.1 | 2026-04-10 | Added synchronous everything, cache without strategy, no capacity planning |
| 1.2 | 2026-07-01 | Added tight coupling, ignoring failure modes, poor data partitioning |
