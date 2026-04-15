# Cache Strategies

## Table of Contents

1. [Overview](#overview)
   - [Why Strategy Selection Matters](#why-strategy-selection-matters)
   - [Strategy Landscape](#strategy-landscape)
2. [Cache-Aside (Lazy Loading)](#cache-aside-lazy-loading)
   - [How Cache-Aside Works](#how-cache-aside-works)
   - [Cache-Aside Flow Diagram](#cache-aside-flow-diagram)
   - [Cache-Aside Pseudocode](#cache-aside-pseudocode)
   - [Cache-Aside Pros and Cons](#cache-aside-pros-and-cons)
   - [When to Use Cache-Aside](#when-to-use-cache-aside)
3. [Read-Through](#read-through)
   - [How Read-Through Works](#how-read-through-works)
   - [Read-Through Flow Diagram](#read-through-flow-diagram)
   - [Read-Through vs Cache-Aside](#read-through-vs-cache-aside)
   - [Read-Through Pros and Cons](#read-through-pros-and-cons)
4. [Write-Through](#write-through)
   - [How Write-Through Works](#how-write-through-works)
   - [Write-Through Flow Diagram](#write-through-flow-diagram)
   - [Write-Through Consistency Guarantee](#write-through-consistency-guarantee)
   - [Write-Through Pros and Cons](#write-through-pros-and-cons)
5. [Write-Behind (Write-Back)](#write-behind-write-back)
   - [How Write-Behind Works](#how-write-behind-works)
   - [Write-Behind Flow Diagram](#write-behind-flow-diagram)
   - [Batching and Coalescing](#batching-and-coalescing)
   - [Write-Behind Pros and Cons](#write-behind-pros-and-cons)
6. [Write-Around](#write-around)
   - [How Write-Around Works](#how-write-around-works)
   - [Write-Around Flow Diagram](#write-around-flow-diagram)
   - [Write-Around Pros and Cons](#write-around-pros-and-cons)
7. [Refresh-Ahead](#refresh-ahead)
   - [How Refresh-Ahead Works](#how-refresh-ahead-works)
   - [Refresh-Ahead Flow Diagram](#refresh-ahead-flow-diagram)
   - [Prediction Algorithms](#prediction-algorithms)
   - [Refresh-Ahead Pros and Cons](#refresh-ahead-pros-and-cons)
8. [Strategy Comparison](#strategy-comparison)
   - [Comprehensive Comparison Table](#comprehensive-comparison-table)
   - [Latency Profiles](#latency-profiles)
9. [Combining Strategies](#combining-strategies)
   - [Common Combinations](#common-combinations)
   - [Patterns by Data Type](#patterns-by-data-type)
   - [Combination Architecture Diagram](#combination-architecture-diagram)
10. [Cache-Aside with Events](#cache-aside-with-events)
    - [Change Data Capture (CDC)](#change-data-capture-cdc)
    - [Event-Driven Invalidation with Kafka](#event-driven-invalidation-with-kafka)
    - [CDC Flow Diagram](#cdc-flow-diagram)
11. [Implementation Examples](#implementation-examples)
    - [Python — Cache-Aside with Redis](#python--cache-aside-with-redis)
    - [Python — Write-Through with Redis](#python--write-through-with-redis)
    - [C# — Cache-Aside with Redis](#c--cache-aside-with-redis)
    - [C# — Write-Through with Redis](#c--write-through-with-redis)
    - [Go — Cache-Aside with Redis](#go--cache-aside-with-redis)
12. [Choosing the Right Strategy](#choosing-the-right-strategy)
    - [Decision Matrix](#decision-matrix)
    - [Decision Flowchart](#decision-flowchart)
    - [Key Questions to Ask](#key-questions-to-ask)

---

## Overview

### Why Strategy Selection Matters

A cache is only as effective as the **strategy** governing how data flows between the
application, the cache layer, and the persistent data store. Choosing the wrong strategy
can lead to stale reads, unnecessary writes, wasted memory, or — worst of all — data loss.

The right strategy depends on your workload characteristics:

| Workload Characteristic   | Primary Concern                                    |
|---------------------------|----------------------------------------------------|
| Read-heavy, rarely updated | Minimise cache misses and read latency             |
| Write-heavy               | Avoid write amplification and cache pollution       |
| Mixed read/write          | Balance consistency with throughput                 |
| Strong consistency needed | Guarantee cache and DB always agree                 |
| Eventual consistency OK   | Optimise for speed; tolerate brief staleness        |
| High availability critical| Survive cache failures without service degradation  |

There is no single "best" strategy. Real-world systems almost always **combine** multiple
strategies for different data types. This guide covers six core strategies, compares them
head-to-head, and shows how to combine them effectively.

### Strategy Landscape

```
                       ┌──────────────────────────────────────────┐
                       │          CACHE STRATEGY LANDSCAPE        │
                       └──────────────────────────────────────────┘

         READ STRATEGIES                      WRITE STRATEGIES
    ┌─────────────────────┐              ┌─────────────────────────┐
    │                     │              │                         │
    │  ┌───────────────┐  │              │  ┌───────────────────┐  │
    │  │  Cache-Aside   │  │              │  │  Write-Through    │  │
    │  │  (Lazy Load)   │  │              │  │  (Synchronous)    │  │
    │  └───────────────┘  │              │  └───────────────────┘  │
    │                     │              │                         │
    │  ┌───────────────┐  │              │  ┌───────────────────┐  │
    │  │  Read-Through  │  │              │  │  Write-Behind     │  │
    │  │  (Inline)      │  │              │  │  (Asynchronous)   │  │
    │  └───────────────┘  │              │  └───────────────────┘  │
    │                     │              │                         │
    │  ┌───────────────┐  │              │  ┌───────────────────┐  │
    │  │  Refresh-Ahead │  │              │  │  Write-Around     │  │
    │  │  (Proactive)   │  │              │  │  (Bypass Cache)   │  │
    │  └───────────────┘  │              │  └───────────────────┘  │
    │                     │              │                         │
    └─────────────────────┘              └─────────────────────────┘
```

---

## Cache-Aside (Lazy Loading)

Cache-aside is the most widely used caching strategy. The **application** is responsible for
all interactions with both the cache and the database. The cache itself has no awareness of
the underlying data store.

### How Cache-Aside Works

1. The application receives a read request.
2. It checks the cache first.
3. **Cache hit** — return the cached value directly.
4. **Cache miss** — read the value from the database.
5. Write the value into the cache for future requests.
6. Return the value to the caller.

The word "aside" reflects that the cache sits to the side; the application must explicitly
manage cache reads and writes rather than the cache doing it automatically.

### Cache-Aside Flow Diagram

```
    Cache Hit Path                          Cache Miss Path
    ══════════════                          ═══════════════

 ┌──────────┐  1. GET key  ┌──────────┐   ┌──────────┐  1. GET key  ┌──────────┐
 │          │─────────────►│          │   │          │─────────────►│          │
 │  Client  │              │  Cache   │   │  Client  │              │  Cache   │
 │          │◄─────────────│          │   │          │              │  (MISS)  │
 └──────────┘  2. Return   └──────────┘   └──────────┘              └──────────┘
                  value                        │                         │
                                               │ 2. Query               │
                                               ▼                        │
                                        ┌──────────┐                    │
                                        │          │                    │
                                        │ Database │                    │
                                        │          │                    │
                                        └──────────┘                    │
                                               │                        │
                                               │ 3. Return data         │
                                               ▼                        │
                                        ┌──────────┐  4. SET key  ┌────┘
                                        │          │─────────────►│
                                        │  Client  │              │  Cache
                                        │          │              │
                                        └──────────┘              └──────────┘
                                               │
                                               │ 5. Return data to caller
                                               ▼
```

### Cache-Aside Pseudocode

```
function get(key):
    value = cache.get(key)

    if value is not None:           # Cache hit
        return value

    value = database.query(key)     # Cache miss — read from DB

    if value is not None:
        cache.set(key, value, ttl=300)   # Populate cache with TTL
    
    return value

function update(key, new_value):
    database.update(key, new_value)
    cache.delete(key)               # Invalidate cache entry
```

> **Note:** On writes, we **delete** the cached entry rather than updating it. This avoids
> a race condition where two concurrent writes could leave the cache with a stale value.
> The next read will trigger a cache miss and fetch fresh data.

### Cache-Aside Pros and Cons

| Pros | Cons |
|------|------|
| Only requested data is cached — no wasted memory | Every cache miss incurs extra latency (DB read + cache write) |
| Application remains functional if cache is unavailable | Risk of stale data if DB is updated outside the app |
| Simple to implement and reason about | Application code must manage both cache and DB logic |
| Works with any cache technology (Redis, Memcached, etc.) | Cache stampede possible on popular keys after expiry |
| Fine-grained control over TTL per data type | No built-in consistency guarantee between cache and DB |

### When to Use Cache-Aside

- **Read-heavy workloads** with a high read-to-write ratio (e.g., product catalogs, user profiles)
- Systems where **cache failure should not break functionality** — the app falls back to DB
- When you need **per-key TTL control** and want full flexibility
- When the data store is not tightly integrated with the cache library

---

## Read-Through

Read-through looks similar to cache-aside from the caller's perspective, but the **cache
library itself** is responsible for loading data from the database on a miss. The application
only ever talks to the cache.

### How Read-Through Works

1. The application requests data from the cache.
2. **Cache hit** — return the cached value.
3. **Cache miss** — the cache library invokes a configured **loader function** that reads from
   the database, stores the result in the cache, and returns it to the application.

The key difference from cache-aside: the application does **not** contain any database read
logic for cached data. That logic lives inside the cache provider or a wrapper library.

### Read-Through Flow Diagram

```
 ┌──────────┐  1. GET key  ┌──────────────────────────────────────────────────┐
 │          │─────────────►│                  CACHE LAYER                     │
 │  Client  │              │                                                  │
 │          │              │  ┌──────────┐  2. MISS  ┌──────────────────────┐ │
 │          │              │  │  Cache   │──────────►│  Loader Function     │ │
 │          │              │  │  Store   │           │  (reads from DB)     │ │
 │          │              │  │          │◄──────────│                      │ │
 │          │              │  │          │  3. Data  └──────────────────────┘ │
 │          │              │  │          │                     │              │
 │          │              │  │          │                     │ 2a. Query    │
 │          │              │  └──────────┘                     ▼              │
 │          │              │       │               ┌──────────────────┐       │
 │          │              │       │               │    Database      │       │
 │          │◄─────────────│       │               └──────────────────┘       │
 └──────────┘  4. Return   └───────┼──────────────────────────────────────────┘
                  value            │
                          (value stored in
                           cache for next read)
```

### Read-Through vs Cache-Aside

| Aspect | Cache-Aside | Read-Through |
|--------|-------------|--------------|
| Who loads data on miss? | Application code | Cache library / loader |
| Application awareness of DB | Full awareness | No awareness (for reads) |
| Code duplication risk | Higher — every caller must implement miss logic | Lower — loader is defined once |
| Flexibility | Maximum — different logic per call site | Moderate — single loader per cache region |
| Cache provider coupling | Low — works with any cache | Higher — requires cache with loader support |

### Read-Through Pros and Cons

| Pros | Cons |
|------|------|
| Cleaner application code — no miss-handling logic scattered everywhere | Requires a cache library that supports loader/read-through callbacks |
| Consistent loading logic — defined once, used everywhere | Less flexibility for per-call-site customization |
| Easier to enforce TTL and eviction policies uniformly | Cold cache still has the same miss penalty |
| Reduces risk of inconsistent miss-handling across services | Loader errors can be harder to debug — hidden behind abstraction |

---

## Write-Through

Write-through ensures that **every write goes to both the cache and the database
synchronously** before the write is considered complete. The cache is always up to date.

### How Write-Through Works

1. The application sends a write request.
2. The data is written to the **cache** first.
3. The cache (or middleware) writes the same data to the **database**.
4. Both writes must succeed before the caller receives confirmation.

This provides a **strong consistency guarantee**: any subsequent read from the cache will
return the value that was just written.

### Write-Through Flow Diagram

```
 ┌──────────┐  1. WRITE   ┌──────────────────────────────────────────────┐
 │          │─────────────►│               CACHE LAYER                   │
 │  Client  │              │                                              │
 │          │              │  ┌──────────┐  2. Write  ┌────────────────┐ │
 │          │              │  │  Cache   │───────────►│   Database     │ │
 │          │              │  │  Store   │            │                │ │
 │          │              │  │          │◄───────────│                │ │
 │          │              │  │ (updated)│  3. ACK    └────────────────┘ │
 │          │              │  └──────────┘                               │
 │          │◄─────────────│                                              │
 └──────────┘  4. ACK      └──────────────────────────────────────────────┘
              (both writes
               confirmed)
```

### Write-Through Consistency Guarantee

Write-through eliminates the window of inconsistency that exists in cache-aside. Compare:

```
  Cache-Aside (invalidate on write):       Write-Through:
  ════════════════════════════════          ═══════════════

  T1: App writes to DB ───────────►OK     T1: App writes cache + DB ──►OK
  T2: App deletes cache key ──────►OK     T2: Cache and DB are in sync
  T3: Reader hits cache ──────────►MISS   T3: Reader hits cache ──────►HIT
  T4: Reader loads from DB ───────►OK         (returns latest value)
  T5: Reader populates cache ─────►OK
                                           No window of inconsistency!
  Between T1 and T4, another reader
  could see stale cached data if
  delete fails or is delayed.
```

The trade-off is **write latency**: every write must wait for both the cache and the database
to confirm, making writes slower than cache-aside.

### Write-Through Pros and Cons

| Pros | Cons |
|------|------|
| Strong consistency — cache always reflects DB state | Higher write latency — two synchronous writes per operation |
| Simplifies read logic — cache hits are always fresh | Write amplification — every write touches two stores |
| Eliminates stale-data bugs for actively written keys | Cache may store data that is never read (wasted memory) |
| Pairs excellently with read-through for full inline caching | Requires cache infrastructure that supports write-through |

---

## Write-Behind (Write-Back)

Write-behind (also called write-back) is an **asynchronous** write strategy. The application
writes to the cache, and the cache **asynchronously flushes** those writes to the database
in the background, often in batches.

### How Write-Behind Works

1. The application sends a write request.
2. The data is written to the **cache** immediately.
3. The caller receives confirmation — the write is "done" from the app's perspective.
4. A background process periodically flushes dirty cache entries to the database.
5. Writes may be **batched** and **coalesced** to reduce database load.

### Write-Behind Flow Diagram

```
 ┌──────────┐  1. WRITE   ┌──────────┐
 │          │─────────────►│          │
 │  Client  │              │  Cache   │
 │          │◄─────────────│  (dirty) │
 └──────────┘  2. ACK      └────┬─────┘
              (immediate)       │
                                │  3. Async flush
                                │     (batched)
                                ▼
                  ┌──────────────────────────┐
                  │     Background Worker    │
                  │                          │
                  │  • Collects dirty keys   │
                  │  • Batches writes        │
                  │  • Coalesces duplicates  │
                  │  • Retries on failure    │
                  │                          │
                  └────────────┬─────────────┘
                               │
                               │  4. Batch write
                               ▼
                        ┌──────────────┐
                        │   Database   │
                        └──────────────┘
```

### Batching and Coalescing

Write-behind shines when the same key is updated frequently. Rather than writing to the
database on every update, the background worker can:

```
  Without coalescing:                 With coalescing:
  ═══════════════════                 ════════════════

  T1: SET counter 1  ──► DB write    T1: SET counter 1  ──► (queued)
  T2: SET counter 2  ──► DB write    T2: SET counter 2  ──► (queued, replaces T1)
  T3: SET counter 3  ──► DB write    T3: SET counter 3  ──► (queued, replaces T2)
  T4: SET counter 4  ──► DB write    T4: SET counter 4  ──► (queued, replaces T3)
  T5: SET counter 5  ──► DB write    T5: Flush ────────────► DB write (counter=5)

  5 database writes                  1 database write
```

This dramatically reduces database load for high-frequency update patterns like counters,
session data, and real-time analytics.

### Write-Behind Pros and Cons

| Pros | Cons |
|------|------|
| Extremely low write latency — only cache write is synchronous | **Durability risk** — cache crash before flush = data loss |
| Reduces database write load through batching and coalescing | Eventual consistency — DB lags behind cache |
| Absorbs write spikes without overwhelming the database | Complex failure handling — retry logic, dead-letter queues |
| Excellent for high-frequency updates (counters, analytics) | Harder to debug — writes are decoupled from the request |
| Database write order can be optimized for throughput | Not suitable for data requiring strong transactional guarantees |

> **Warning:** Write-behind introduces a **durability gap**. If the cache node crashes before
> dirty entries are flushed, those writes are lost. Use write-behind only for data where
> brief loss is acceptable, or pair it with a persistent queue (e.g., Kafka) to guarantee
> delivery.

---

## Write-Around

Write-around is the simplest write strategy: data is written **directly to the database**,
completely bypassing the cache. The cache is only populated on subsequent reads (via
cache-aside or read-through).

### How Write-Around Works

1. The application sends a write request.
2. The data is written **only** to the database.
3. The cache is **not** updated or invalidated.
4. On the next read for that key, a cache miss occurs and the cache is populated from the DB.

### Write-Around Flow Diagram

```
  Write Path (bypasses cache):          Read Path (populates cache):
  ════════════════════════════          ════════════════════════════

 ┌──────────┐  1. WRITE   ┌──────────┐   ┌──────────┐  1. GET  ┌──────────┐
 │          │─────────────►│          │   │          │────────►│          │
 │  Client  │              │ Database │   │  Client  │         │  Cache   │
 │          │◄─────────────│          │   │          │         │  (MISS)  │
 └──────────┘  2. ACK      └──────────┘   └──────────┘         └──────────┘
                                               │                     ▲
              Cache is NOT touched!            │ 2. Query DB         │
                                               ▼                     │
                                        ┌──────────┐                 │
                                        │ Database │   3. Populate   │
                                        └──────────┘────────────────►┘
```

### Write-Around Pros and Cons

| Pros | Cons |
|------|------|
| Prevents cache pollution from write-heavy data that is rarely read | Higher read latency on first access after a write (guaranteed miss) |
| Simple implementation — no cache logic on the write path | Stale data in cache if existing entries are not invalidated |
| Works well for bulk/batch data loads (ETL, imports) | Not suitable for data that is read immediately after writing |
| Reduces cache churn — only frequently-read data stays cached | Requires a separate read strategy (cache-aside) to be effective |

> **When to use write-around:** Log data, audit trails, batch imports, analytics events —
> anything that is written frequently but read rarely or only in aggregate.

---

## Refresh-Ahead

Refresh-ahead is a **proactive** caching strategy that reloads cache entries **before** they
expire. Instead of waiting for a TTL expiry and a subsequent cache miss, the system predicts
which entries will be needed soon and refreshes them in the background.

### How Refresh-Ahead Works

1. A cache entry is set with a TTL (e.g., 300 seconds).
2. A **refresh window** is defined (e.g., the last 20% of the TTL = final 60 seconds).
3. When a read occurs for a key that is inside the refresh window, the cache returns the
   current (still valid) value **and** triggers an asynchronous background reload.
4. The background reload fetches the latest value from the database and updates the cache.
5. The user never experiences a cache miss for actively-read keys.

### Refresh-Ahead Flow Diagram

```
  Timeline for a key with TTL = 300s, refresh window = 60s:

  0s              240s                    300s
  │────────────────│─────────────────────│
  │  Normal TTL    │  Refresh Window     │ Expired
  │                │                     │
  │  Read → HIT   │  Read → HIT +       │  Read → MISS
  │  (no action)   │  trigger async      │  (if not refreshed)
  │                │  background reload   │
  │                │         │            │
  │                │         ▼            │
  │                │  ┌──────────────┐   │
  │                │  │ Background   │   │
  │                │  │ reload from  │   │
  │                │  │ database     │   │
  │                │  └──────┬───────┘   │
  │                │         │            │
  │                │         ▼            │
  │                │  Cache updated       │
  │                │  with fresh data     │
  │                │  + new TTL           │
```

### Prediction Algorithms

Simple refresh-ahead triggers on any read during the refresh window. More advanced
implementations use prediction:

| Algorithm | Description | Best For |
|-----------|-------------|----------|
| **Access-frequency based** | Refresh keys that are read more than N times per TTL period | High-traffic keys |
| **Time-of-day patterns** | Pre-warm keys at times when traffic historically spikes | Predictable traffic |
| **Probabilistic early expiry** | Each read has a random chance of triggering refresh as TTL approaches | Spreading load evenly |
| **Machine-learned models** | Predict which keys will be requested in the next time window | Large-scale systems |

### Refresh-Ahead Pros and Cons

| Pros | Cons |
|------|------|
| Near-zero cache miss latency for hot keys | Wastes resources refreshing keys that may not be read again |
| Smooth latency distribution — no miss spikes at TTL boundaries | More complex to implement — needs background threads and prediction logic |
| Reduces thundering herd on expiry of popular keys | Prediction inaccuracy leads to unnecessary database reads |
| Works well with read-through for clean integration | Must tune refresh window carefully — too early wastes resources, too late misses the window |

---

## Strategy Comparison

### Comprehensive Comparison Table

| Dimension | Cache-Aside | Read-Through | Write-Through | Write-Behind | Write-Around | Refresh-Ahead |
|-----------|-------------|--------------|---------------|--------------|--------------|---------------|
| **Consistency** | Eventual | Eventual | Strong | Eventual | Eventual (stale risk) | Eventual |
| **Read latency (hit)** | Very low | Very low | Very low | Very low | Very low | Very low |
| **Read latency (miss)** | High (DB + cache write) | High (loader + cache write) | Low (always in cache) | Low (always in cache) | High (DB + cache write) | Very low (proactive) |
| **Write latency** | Low (DB only, then invalidate) | Low (DB only) | High (cache + DB sync) | Very low (cache only) | Low (DB only) | N/A (read strategy) |
| **Implementation complexity** | Low | Medium | Medium | High | Low | High |
| **Durability** | High (DB is source of truth) | High | High | **Low** (risk of data loss) | High | High |
| **Cache utilization** | Good (only requested data) | Good | Lower (all writes cached) | Lower (all writes cached) | Good | Lower (speculative refresh) |
| **Best use case** | General-purpose reads | Library-managed reads | Consistency-critical writes | High-throughput writes | Write-heavy, rarely-read data | Latency-sensitive hot data |

### Latency Profiles

```
  Read Latency Comparison (cache miss scenario):

  Cache-Aside    │████████████████████│  DB read + cache write
  Read-Through   │████████████████████│  Loader + cache write
  Write-Through  │██│                    Always in cache (no miss)
  Write-Behind   │██│                    Always in cache (no miss)
  Write-Around   │████████████████████│  DB read + cache write
  Refresh-Ahead  │████│                  Rare miss (proactive refresh)
                 0ms                 50ms

  Write Latency Comparison:

  Cache-Aside    │██████████████│       DB write + cache delete
  Write-Through  │████████████████████│  Cache write + DB write (sync)
  Write-Behind   │████│                  Cache write only (async flush)
  Write-Around   │██████████│            DB write only
                 0ms                 50ms
```

---

## Combining Strategies

Real-world systems rarely use a single caching strategy in isolation. The most effective
architectures combine a read strategy with a write strategy, chosen per data type.

### Common Combinations

**1. Cache-Aside + Write-Through (Strong Consistency)**

```
  ┌──────────┐       READS (cache-aside)        ┌──────────┐
  │          │──────────────────────────────────►│          │
  │          │                                   │  Cache   │
  │  Client  │       WRITES (write-through)      │          │
  │          │──────────────────────────────────►│          │──────►┌──────────┐
  │          │◄──────────────────────────────────│          │◄──────│ Database │
  └──────────┘                                   └──────────┘       └──────────┘
```

Best for: Data that is read frequently and must be consistent after writes (e.g., user
profile data, account balances).

**2. Read-Through + Write-Behind (Maximum Throughput)**

```
  ┌──────────┐       READS (read-through)        ┌──────────┐
  │          │──────────────────────────────────►│          │
  │          │                                   │  Cache   │      ┌──────────┐
  │  Client  │       WRITES (write-behind)       │  Layer   │─────►│ Database │
  │          │──────────────────────────────────►│          │ async└──────────┘
  │          │◄──────────────────────────────────│          │
  └──────────┘       (immediate ACK)             └──────────┘
```

Best for: High-throughput workloads where eventual consistency is acceptable (e.g.,
session stores, shopping carts, real-time counters).

**3. Cache-Aside + Write-Around (Write-Heavy, Read-Light)**

```
  READS (cache-aside):                  WRITES (write-around):

  Client ──► Cache ──► (miss) ──► DB    Client ──► DB (cache untouched)
              │                                    │
              ◄─── populate cache                  │ (no cache interaction)
```

Best for: Bulk data ingestion, ETL pipelines, audit logs — data that is written
frequently but rarely read in real time.

### Patterns by Data Type

| Data Type | Read Strategy | Write Strategy | Rationale |
|-----------|--------------|----------------|-----------|
| **User profiles** | Cache-aside | Write-through | Read-heavy; must be consistent after updates |
| **Shopping cart** | Read-through | Write-behind | Frequently updated; eventual consistency OK |
| **Product catalog** | Read-through + refresh-ahead | Write-around | Read-heavy; rarely updated; low-latency critical |
| **Session data** | Read-through | Write-behind | High frequency reads and writes; loss tolerable |
| **Inventory counts** | Cache-aside | Write-through | Must be accurate; read on checkout |
| **Audit logs** | None (rarely read) | Write-around | Write-only; read via analytics pipeline |
| **Leaderboards** | Cache-aside | Write-behind | Updated constantly; reads tolerate slight staleness |
| **Configuration** | Read-through + refresh-ahead | Write-through | Read-heavy; changes must propagate quickly |

### Combination Architecture Diagram

```
┌────────────────────────────────────────────────────────────────────────┐
│                         APPLICATION LAYER                             │
│                                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                │
│  │ User Service │  │ Cart Service │  │ Catalog Svc  │                │
│  │              │  │              │  │              │                │
│  │ Cache-Aside  │  │ Read-Through │  │ Read-Through │                │
│  │ + Write-Thru │  │ + Write-Back │  │ + Refresh    │                │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                │
│         │                 │                 │                         │
└─────────┼─────────────────┼─────────────────┼─────────────────────────┘
          │                 │                 │
          ▼                 ▼                 ▼
   ┌─────────────────────────────────────────────────┐
   │              REDIS CLUSTER                       │
   │  ┌─────────┐  ┌─────────┐  ┌─────────┐         │
   │  │ Shard 1 │  │ Shard 2 │  │ Shard 3 │         │
   │  └─────────┘  └─────────┘  └─────────┘         │
   └────────────────────┬────────────────────────────┘
                        │
          ┌─────────────┼──────────────┐
          ▼             ▼              ▼
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │ Users DB │  │ Carts DB │  │Catalog DB│
   │ (Postgres)│  │ (Postgres)│  │ (Postgres)│
   └──────────┘  └──────────┘  └──────────┘
```

---

## Cache-Aside with Events

Traditional cache-aside has a fundamental limitation: when the database is updated by
**another service** or **direct DB access**, the cache becomes stale with no way to know.
Event-driven cache invalidation solves this.

### Change Data Capture (CDC)

CDC tools like **Debezium** watch the database's transaction log (WAL in PostgreSQL,
binlog in MySQL) and emit events for every row change. These events can drive cache
invalidation or cache refresh.

```
  Traditional (blind cache):              CDC-driven (event-aware cache):
  ═══════════════════════════             ═══════════════════════════════

  Service A writes DB ──► OK              Service A writes DB ──► OK
  Cache still has old value               Debezium captures WAL event
  Service B reads cache ──► STALE!        Event published to Kafka
                                          Cache consumer invalidates key
                                          Service B reads cache ──► MISS ──► fresh data
```

### Event-Driven Invalidation with Kafka

```
  ┌────────────┐     ┌────────────┐     ┌─────────────┐     ┌───────────┐
  │ Service A  │────►│  Database  │────►│  Debezium   │────►│  Kafka    │
  │ (writes)   │     │            │     │  Connector  │     │  Topic    │
  └────────────┘     └────────────┘     └─────────────┘     └─────┬─────┘
                                                                   │
  ┌────────────┐     ┌────────────┐                                │
  │ Service B  │────►│   Cache    │◄───── Cache Invalidation ◄─────┘
  │ (reads)    │◄────│  (Redis)   │       Consumer
  └────────────┘     └────────────┘
```

### CDC Flow Diagram

```
  Detailed CDC Pipeline:
  ═════════════════════

  1. Application writes to PostgreSQL
     │
     ▼
  2. PostgreSQL writes to WAL (Write-Ahead Log)
     │
     ▼
  3. Debezium reads WAL via logical replication slot
     │
     ▼
  4. Debezium publishes change event to Kafka topic
     │
     ├──► Topic: "db.public.users"
     │    Key: {"id": 42}
     │    Value: {"before": {...}, "after": {...}, "op": "u"}
     │
     ▼
  5. Cache invalidation consumer reads from Kafka
     │
     ├──► Extracts entity key from event
     ├──► Deletes or updates corresponding cache entry
     │
     ▼
  6. Next cache read triggers fresh load from DB

  Advantages:
  • Works across multiple services (no coupling)
  • Captures ALL changes, including direct SQL updates
  • At-least-once delivery with Kafka consumer groups
  • Can replay events to rebuild cache from scratch
```

> **Tip:** For systems with many services writing to the same database, CDC-driven
> invalidation is far more reliable than each service trying to coordinate cache deletes.

---

## Implementation Examples

### Python — Cache-Aside with Redis

```python
import json
import redis
import psycopg2
from typing import Optional

class UserCache:
    """Cache-aside implementation for user data."""

    def __init__(self, redis_url: str, db_dsn: str, ttl: int = 300):
        self.cache = redis.from_url(redis_url)
        self.db = psycopg2.connect(db_dsn)
        self.ttl = ttl

    def _cache_key(self, user_id: int) -> str:
        return f"user:{user_id}"

    def get_user(self, user_id: int) -> Optional[dict]:
        key = self._cache_key(user_id)

        # Step 1: Check cache
        cached = self.cache.get(key)
        if cached is not None:
            return json.loads(cached)

        # Step 2: Cache miss — read from database
        cursor = self.db.cursor()
        cursor.execute(
            "SELECT id, name, email FROM users WHERE id = %s",
            (user_id,),
        )
        row = cursor.fetchone()
        cursor.close()

        if row is None:
            return None

        user = {"id": row[0], "name": row[1], "email": row[2]}

        # Step 3: Populate cache with TTL
        self.cache.setex(key, self.ttl, json.dumps(user))

        return user

    def update_user(self, user_id: int, name: str, email: str) -> None:
        # Step 1: Write to database
        cursor = self.db.cursor()
        cursor.execute(
            "UPDATE users SET name = %s, email = %s WHERE id = %s",
            (name, email, user_id),
        )
        self.db.commit()
        cursor.close()

        # Step 2: Invalidate cache entry
        self.cache.delete(self._cache_key(user_id))
```

### Python — Write-Through with Redis

```python
import json
import redis
import psycopg2

class UserWriteThroughCache:
    """Write-through implementation — writes go to both cache and DB."""

    def __init__(self, redis_url: str, db_dsn: str, ttl: int = 300):
        self.cache = redis.from_url(redis_url)
        self.db = psycopg2.connect(db_dsn)
        self.ttl = ttl

    def _cache_key(self, user_id: int) -> str:
        return f"user:{user_id}"

    def write_user(self, user_id: int, name: str, email: str) -> dict:
        user = {"id": user_id, "name": name, "email": email}
        key = self._cache_key(user_id)

        # Write-through: update both cache and DB synchronously
        pipe = self.cache.pipeline()
        cursor = self.db.cursor()

        try:
            # Write to database first
            cursor.execute(
                """INSERT INTO users (id, name, email) VALUES (%s, %s, %s)
                   ON CONFLICT (id) DO UPDATE SET name = %s, email = %s""",
                (user_id, name, email, name, email),
            )
            self.db.commit()

            # Write to cache (only after DB succeeds)
            pipe.setex(key, self.ttl, json.dumps(user))
            pipe.execute()

        except Exception:
            self.db.rollback()
            self.cache.delete(key)  # Ensure no stale cache on failure
            raise
        finally:
            cursor.close()

        return user

    def get_user(self, user_id: int) -> dict | None:
        key = self._cache_key(user_id)
        cached = self.cache.get(key)
        if cached is not None:
            return json.loads(cached)

        # Fallback to DB (should be rare with write-through)
        cursor = self.db.cursor()
        cursor.execute(
            "SELECT id, name, email FROM users WHERE id = %s",
            (user_id,),
        )
        row = cursor.fetchone()
        cursor.close()

        if row is None:
            return None

        user = {"id": row[0], "name": row[1], "email": row[2]}
        self.cache.setex(key, self.ttl, json.dumps(user))
        return user
```

### C# — Cache-Aside with Redis

```csharp
using System.Text.Json;
using Microsoft.Data.SqlClient;
using StackExchange.Redis;

public class UserCacheAside
{
    private readonly IDatabase _cache;
    private readonly string _connectionString;
    private readonly TimeSpan _ttl;

    public UserCacheAside(
        IConnectionMultiplexer redis,
        string connectionString,
        TimeSpan? ttl = null)
    {
        _cache = redis.GetDatabase();
        _connectionString = connectionString;
        _ttl = ttl ?? TimeSpan.FromMinutes(5);
    }

    private static string CacheKey(int userId) => $"user:{userId}";

    public async Task<User?> GetUserAsync(int userId)
    {
        var key = CacheKey(userId);

        // Step 1: Check cache
        var cached = await _cache.StringGetAsync(key);
        if (cached.HasValue)
        {
            return JsonSerializer.Deserialize<User>(cached!);
        }

        // Step 2: Cache miss — read from database
        await using var conn = new SqlConnection(_connectionString);
        await conn.OpenAsync();

        await using var cmd = new SqlCommand(
            "SELECT Id, Name, Email FROM Users WHERE Id = @Id", conn);
        cmd.Parameters.AddWithValue("@Id", userId);

        await using var reader = await cmd.ExecuteReaderAsync();
        if (!await reader.ReadAsync())
        {
            return null;
        }

        var user = new User
        {
            Id = reader.GetInt32(0),
            Name = reader.GetString(1),
            Email = reader.GetString(2),
        };

        // Step 3: Populate cache
        var json = JsonSerializer.Serialize(user);
        await _cache.StringSetAsync(key, json, _ttl);

        return user;
    }

    public async Task UpdateUserAsync(int userId, string name, string email)
    {
        // Step 1: Write to database
        await using var conn = new SqlConnection(_connectionString);
        await conn.OpenAsync();

        await using var cmd = new SqlCommand(
            "UPDATE Users SET Name = @Name, Email = @Email WHERE Id = @Id",
            conn);
        cmd.Parameters.AddWithValue("@Id", userId);
        cmd.Parameters.AddWithValue("@Name", name);
        cmd.Parameters.AddWithValue("@Email", email);
        await cmd.ExecuteNonQueryAsync();

        // Step 2: Invalidate cache
        await _cache.KeyDeleteAsync(CacheKey(userId));
    }
}

public record User
{
    public int Id { get; init; }
    public string Name { get; init; } = string.Empty;
    public string Email { get; init; } = string.Empty;
}
```

### C# — Write-Through with Redis

```csharp
using System.Text.Json;
using Microsoft.Data.SqlClient;
using StackExchange.Redis;

public class UserWriteThrough
{
    private readonly IDatabase _cache;
    private readonly string _connectionString;
    private readonly TimeSpan _ttl;

    public UserWriteThrough(
        IConnectionMultiplexer redis,
        string connectionString,
        TimeSpan? ttl = null)
    {
        _cache = redis.GetDatabase();
        _connectionString = connectionString;
        _ttl = ttl ?? TimeSpan.FromMinutes(5);
    }

    private static string CacheKey(int userId) => $"user:{userId}";

    public async Task<User> WriteUserAsync(int userId, string name, string email)
    {
        var user = new User { Id = userId, Name = name, Email = email };
        var key = CacheKey(userId);

        // Write-through: DB first, then cache
        await using var conn = new SqlConnection(_connectionString);
        await conn.OpenAsync();

        await using var cmd = new SqlCommand(
            @"MERGE Users AS target
              USING (SELECT @Id, @Name, @Email) AS source (Id, Name, Email)
              ON target.Id = source.Id
              WHEN MATCHED THEN UPDATE SET Name = @Name, Email = @Email
              WHEN NOT MATCHED THEN INSERT (Id, Name, Email) VALUES (@Id, @Name, @Email);",
            conn);
        cmd.Parameters.AddWithValue("@Id", userId);
        cmd.Parameters.AddWithValue("@Name", name);
        cmd.Parameters.AddWithValue("@Email", email);
        await cmd.ExecuteNonQueryAsync();

        // Update cache after successful DB write
        var json = JsonSerializer.Serialize(user);
        await _cache.StringSetAsync(key, json, _ttl);

        return user;
    }
}
```

### Go — Cache-Aside with Redis

```go
package cache

import (
	"context"
	"database/sql"
	"encoding/json"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

type User struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
}

type UserCacheAside struct {
	cache *redis.Client
	db    *sql.DB
	ttl   time.Duration
}

func NewUserCacheAside(cache *redis.Client, db *sql.DB, ttl time.Duration) *UserCacheAside {
	return &UserCacheAside{cache: cache, db: db, ttl: ttl}
}

func cacheKey(userID int) string {
	return fmt.Sprintf("user:%d", userID)
}

func (u *UserCacheAside) GetUser(ctx context.Context, userID int) (*User, error) {
	key := cacheKey(userID)

	// Step 1: Check cache
	cached, err := u.cache.Get(ctx, key).Result()
	if err == nil {
		var user User
		if err := json.Unmarshal([]byte(cached), &user); err == nil {
			return &user, nil
		}
	}

	// Step 2: Cache miss — read from database
	var user User
	err = u.db.QueryRowContext(ctx,
		"SELECT id, name, email FROM users WHERE id = $1", userID,
	).Scan(&user.ID, &user.Name, &user.Email)

	if err == sql.ErrNoRows {
		return nil, nil
	}
	if err != nil {
		return nil, fmt.Errorf("query user %d: %w", userID, err)
	}

	// Step 3: Populate cache
	data, err := json.Marshal(user)
	if err != nil {
		return &user, nil // Return user even if cache write fails
	}
	u.cache.Set(ctx, key, data, u.ttl)

	return &user, nil
}

func (u *UserCacheAside) UpdateUser(ctx context.Context, userID int, name, email string) error {
	// Step 1: Write to database
	_, err := u.db.ExecContext(ctx,
		"UPDATE users SET name = $1, email = $2 WHERE id = $3",
		name, email, userID,
	)
	if err != nil {
		return fmt.Errorf("update user %d: %w", userID, err)
	}

	// Step 2: Invalidate cache
	u.cache.Del(ctx, cacheKey(userID))

	return nil
}
```

---

## Choosing the Right Strategy

### Decision Matrix

Use this matrix to narrow down the best strategy based on your requirements:

| Requirement | Recommended Strategy | Avoid |
|-------------|---------------------|-------|
| Read-heavy, write-light workload | Cache-aside or read-through | Write-behind (unnecessary complexity) |
| Write-heavy, read-light workload | Write-around | Write-through (cache pollution) |
| Strong consistency required | Write-through | Write-behind (eventual consistency) |
| Maximum write throughput | Write-behind | Write-through (synchronous penalty) |
| Minimal implementation effort | Cache-aside + write-around | Refresh-ahead (complex prediction logic) |
| Ultra-low read latency | Refresh-ahead + read-through | Cache-aside alone (miss penalty) |
| Resilience to cache failures | Cache-aside | Write-behind (data loss risk) |
| Multi-service architecture | Cache-aside + CDC events | Any strategy relying on single-app invalidation |

### Decision Flowchart

```
                              START
                                │
                                ▼
                   ┌──────────────────────┐
                   │  Is the data read    │
                   │  more than written?  │
                   └──────────┬───────────┘
                        │           │
                       YES         NO
                        │           │
                        ▼           ▼
              ┌─────────────┐  ┌──────────────────┐
              │ Is ultra-low │  │ Is the data read │
              │ read latency │  │ at all after     │
              │ critical?    │  │ writing?         │
              └──────┬──────┘  └────────┬─────────┘
                │        │         │          │
               YES      NO       YES        NO
                │        │         │          │
                ▼        ▼         ▼          ▼
          ┌──────────┐ ┌──────┐ ┌──────┐  ┌────────────┐
          │ Refresh- │ │Cache-│ │Strong│  │Write-Around│
          │ Ahead +  │ │Aside │ │consis│  │            │
          │ Read-Thru│ │      │ │needed│  └────────────┘
          └──────────┘ └──────┘ │ ?    │
                                └──┬───┘
                              │        │
                             YES      NO
                              │        │
                              ▼        ▼
                        ┌──────────┐ ┌──────────┐
                        │ Write-   │ │ Write-   │
                        │ Through  │ │ Behind   │
                        └──────────┘ └──────────┘
```

### Key Questions to Ask

Before choosing a caching strategy, answer these questions about your workload:

```
┌────────────────────────────────────────────────────────────────────────┐
│                    STRATEGY SELECTION CHECKLIST                        │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  1. READ / WRITE RATIO                                                │
│     □ Read-heavy (>10:1 read-to-write) → Cache-aside, Read-through   │
│     □ Write-heavy (<1:1 read-to-write) → Write-around                │
│     □ Balanced (1:1 to 10:1)          → Write-through + Cache-aside  │
│                                                                        │
│  2. CONSISTENCY REQUIREMENTS                                          │
│     □ Strong (reads must reflect latest write) → Write-through       │
│     □ Eventual (seconds of staleness OK)       → Cache-aside + TTL   │
│     □ Best-effort (minutes of staleness OK)    → Write-behind        │
│                                                                        │
│  3. LATENCY TOLERANCE                                                 │
│     □ Sub-millisecond reads required  → Refresh-ahead + local cache  │
│     □ Single-digit ms reads OK        → Cache-aside with Redis       │
│     □ Tens of ms reads OK             → Cache-aside (generous TTL)   │
│                                                                        │
│  4. DATA VOLATILITY                                                   │
│     □ Rarely changes (config, catalog) → Long TTL + refresh-ahead    │
│     □ Changes frequently (cart, session)→ Short TTL + write-through  │
│     □ Changes constantly (counters)    → Write-behind with batching  │
│                                                                        │
│  5. FAILURE TOLERANCE                                                 │
│     □ Cache down = OK (degrade gracefully) → Cache-aside             │
│     □ Cache down = unacceptable latency    → Multi-tier caching      │
│     □ Data loss on crash = unacceptable    → Write-through (not WB)  │
│     □ Brief data loss on crash = acceptable→ Write-behind            │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Version History

| Date       | Change          | Author |
|------------|-----------------|--------|
| 2025-04-15 | Initial version | Team   |
