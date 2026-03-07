# Caching Strategies

## Table of Contents

1. [Overview](#overview)
2. [What Is Caching](#what-is-caching)
   - [Definition](#definition)
   - [Why Caching Matters](#why-caching-matters)
   - [Cache Hit, Cache Miss, and Hit Ratio](#cache-hit-cache-miss-and-hit-ratio)
3. [Cache Patterns](#cache-patterns)
   - [Cache-Aside (Lazy Loading)](#cache-aside-lazy-loading)
   - [Read-Through](#read-through)
   - [Write-Through](#write-through)
   - [Write-Behind (Write-Back)](#write-behind-write-back)
   - [Refresh-Ahead](#refresh-ahead)
   - [Pattern Comparison](#pattern-comparison)
4. [Cache Eviction Policies](#cache-eviction-policies)
   - [Least Recently Used (LRU)](#least-recently-used-lru)
   - [Least Frequently Used (LFU)](#least-frequently-used-lfu)
   - [First In First Out (FIFO)](#first-in-first-out-fifo)
   - [Time-To-Live (TTL)](#time-to-live-ttl)
   - [Eviction Policy Comparison](#eviction-policy-comparison)
5. [CDN Caching](#cdn-caching)
   - [How CDNs Cache Content](#how-cdns-cache-content)
   - [Edge Caching](#edge-caching)
   - [Origin Shield](#origin-shield)
   - [Cache-Control Headers](#cache-control-headers)
6. [Application-Level Caching](#application-level-caching)
   - [In-Memory Caches](#in-memory-caches)
   - [Distributed Caches](#distributed-caches)
   - [Local vs Distributed Comparison](#local-vs-distributed-comparison)
7. [Database Query Caching](#database-query-caching)
   - [Query Result Caching](#query-result-caching)
   - [Materialized Views](#materialized-views)
8. [Cache Invalidation](#cache-invalidation)
   - [The Hard Problem](#the-hard-problem)
   - [Time-Based Invalidation](#time-based-invalidation)
   - [Event-Based Invalidation](#event-based-invalidation)
   - [Version-Based Invalidation](#version-based-invalidation)
   - [Pub/Sub Invalidation](#pubsub-invalidation)
9. [Cache Consistency](#cache-consistency)
   - [Consistency Challenges in Distributed Caches](#consistency-challenges-in-distributed-caches)
   - [Thundering Herd Problem](#thundering-herd-problem)
   - [Cache Stampede](#cache-stampede)
10. [Monitoring and Metrics](#monitoring-and-metrics)
    - [Cache Hit Ratio](#cache-hit-ratio)
    - [Latency](#latency)
    - [Eviction Rate](#eviction-rate)
11. [Next Steps](#next-steps)
12. [Version History](#version-history)

---

## Overview

Caching is one of the most powerful tools in a system designer's toolbox. A well-placed cache can reduce latency by orders of magnitude, absorb traffic spikes, and dramatically lower infrastructure costs. A poorly designed cache, however, can introduce stale data, consistency bugs, and operational complexity that is difficult to debug.

### Target Audience

- Software engineers designing high-throughput systems
- Architects choosing caching layers for distributed applications
- SREs tuning cache performance and reliability

### Scope

- Core caching patterns and when to apply each one
- Eviction policies and their trade-offs
- CDN, application-level, and database query caching
- Cache invalidation strategies and consistency challenges
- Monitoring and operational metrics for caches

## What Is Caching

### Definition

A **cache** is a temporary storage layer that holds a subset of data so that future requests for that data can be served faster than accessing the primary source. Caches exploit **locality of reference** — the observation that recently or frequently accessed data is likely to be accessed again.

```
  Without Cache                          With Cache

  ┌─────────┐    request    ┌────────┐   ┌─────────┐  hit   ┌───────┐
  │ Client  │──────────────▶│ Server │   │ Client  │──────▶│ Cache │
  │         │◀──────────────│        │   │         │◀──────│       │
  └─────────┘   ~50-200ms   └────────┘   └─────────┘ ~1ms └───────┘
                                                 │  miss         │
                                                 ▼               │
                                           ┌────────┐   fill     │
                                           │ Server │───────────▶│
                                           │        │            │
                                           └────────┘            │
```

### Why Caching Matters

| Problem | How Caching Helps |
|---|---|
| Backend overwhelmed by repeated reads | Cache absorbs read traffic; backend handles only misses |
| High latency on complex computations | Pre-computed results served from memory in < 1 ms |
| Read-heavy workloads with skewed access | Hot data lives in cache; cold data stays in the primary store |
| Cost of scaling databases vertically | A single Redis node serves 100k+ ops/sec at a fraction of the cost |
| Cross-region latency | Edge caches or regional cache clusters reduce round-trip time |

### Cache Hit, Cache Miss, and Hit Ratio

- **Cache hit** — the requested data is found in the cache and returned directly.
- **Cache miss** — the data is not in the cache and must be fetched from the origin.
- **Hit ratio** — `hits / (hits + misses)`. A ratio of 0.95 means only 5% of requests reach the origin.

```
  Hit Ratio Impact on Origin Load
  ──────────────────────────────

  Hit Ratio    Origin Queries (per 10,000 requests)
  ──────────   ────────────────────────────────────
  0%           10,000  (no cache benefit)
  90%          1,000
  95%          500
  99%          100
  99.9%        10
```

> **Key insight:** At high hit ratios, small improvements have an outsized effect. Going from 95% to 99% cuts origin load by 80%.

---

## Cache Patterns

### Cache-Aside (Lazy Loading)

The most common pattern. The application manages the cache explicitly — it checks the cache, falls back to the data source on a miss, and populates the cache with the result.

```
  Cache-Aside Pattern
  ───────────────────

  ┌─────────┐  1. GET key   ┌───────┐
  │   App   │──────────────▶│ Cache │
  │         │◀──────────────│       │
  │         │  2a. HIT:     └───────┘
  │         │     return
  │         │
  │         │  2b. MISS:
  │         │──────────────▶┌──────┐
  │         │  3. query     │  DB  │
  │         │◀──────────────│      │
  │         │  4. result    └──────┘
  │         │
  │         │  5. SET key ─▶┌───────┐
  │         │               │ Cache │
  └─────────┘               └───────┘
```

- ✅ Only requested data is cached — efficient memory usage
- ✅ Cache failures are non-fatal — the app falls back to the DB
- ❌ First request for any key is always a miss (cold start)
- ❌ Data can become stale if the DB is updated without invalidating the cache

### Read-Through

The cache itself is responsible for loading data from the data source on a miss. The application only ever talks to the cache.

```
  Read-Through Pattern
  ────────────────────

  ┌─────────┐  1. GET key   ┌───────────┐
  │   App   │──────────────▶│   Cache   │
  │         │               │           │
  │         │               │ 2. MISS?  │
  │         │               │   ┌──▶┌──────┐
  │         │               │   │   │  DB  │
  │         │               │   ◀──┘└──────┘
  │         │◀──────────────│ 3. return  │
  └─────────┘               └───────────┘
```

- ✅ Simpler application code — the cache handles loading logic
- ✅ Consistent data-loading behavior across all consumers
- ❌ Cache library or provider must support read-through
- ❌ First request is still a miss

### Write-Through

Every write goes to the cache **and** the data source synchronously. The write is only acknowledged after both succeed.

```
  Write-Through Pattern
  ─────────────────────

  ┌─────────┐  1. write    ┌───────────┐  2. write   ┌──────┐
  │   App   │─────────────▶│   Cache   │────────────▶│  DB  │
  │         │              │           │             │      │
  │         │◀─────────────│◀────────────────────────│      │
  │         │  3. ack      └───────────┘  ack        └──────┘
  └─────────┘
```

- ✅ Cache is always consistent with the data source
- ✅ No stale reads after a write
- ❌ Higher write latency — every write must hit two stores
- ❌ Writes to data that is never read waste cache space

### Write-Behind (Write-Back)

Writes go to the cache immediately. The cache asynchronously flushes writes to the data source in batches.

```
  Write-Behind Pattern
  ────────────────────

  ┌─────────┐  1. write    ┌───────────┐
  │   App   │─────────────▶│   Cache   │
  │         │◀─────────────│           │
  │         │  2. ack      │           │
  └─────────┘              │  (async)  │
                           │  3. batch ▼
                           │  flush   ┌──────┐
                           │─────────▶│  DB  │
                           └──────────└──────┘
```

- ✅ Very low write latency — acknowledged after cache write
- ✅ Batch writes reduce DB load
- ❌ Risk of data loss if the cache crashes before flushing
- ❌ Increased complexity — must handle retry, ordering, and deduplication

### Refresh-Ahead

The cache proactively refreshes entries **before** they expire, based on predicted access patterns or a configured refresh window.

```
  Refresh-Ahead Pattern
  ─────────────────────

  Timeline for a key with TTL = 60s, refresh at 80% (48s)

  0s          48s             60s
  ├───────────┼───────────────┤
  │  serving  │  cache        │  would have
  │  from     │  refreshes    │  expired
  │  cache    │  in background│  here
  └───────────┴───────────────┘

  ┌───────────┐  background   ┌──────┐
  │   Cache   │──────────────▶│  DB  │
  │           │◀──────────────│      │
  │  refresh  │  fresh data   └──────┘
  └───────────┘
```

- ✅ Eliminates cache-miss latency for frequently accessed keys
- ✅ Smooths out load on the data source
- ❌ Wastes resources refreshing data that may not be requested again
- ❌ Requires accurate prediction of access patterns

### Pattern Comparison

| Pattern | Read Latency | Write Latency | Consistency | Complexity | Best For |
|---|---|---|---|---|---|
| **Cache-Aside** | Miss on cold | N/A (direct DB write) | Eventual | Low | General-purpose read caching |
| **Read-Through** | Miss on cold | N/A (direct DB write) | Eventual | Medium | Uniform cache loading logic |
| **Write-Through** | Always hit after write | High (sync to both) | Strong | Medium | Read-after-write consistency |
| **Write-Behind** | Always hit after write | Very low (async) | Eventual | High | Write-heavy workloads |
| **Refresh-Ahead** | No miss for hot keys | N/A | Eventual | High | Predictable, high-frequency reads |

---

## Cache Eviction Policies

When a cache reaches its memory limit, it must decide which entries to remove. The eviction policy determines this.

### Least Recently Used (LRU)

Evicts the entry that has not been accessed for the longest time. LRU assumes that recently accessed data is more likely to be accessed again.

```
  LRU Eviction Example (capacity = 3)
  ─────────────────────────────────────

  Access: A, B, C, A, D

  Step 1: [A]           — add A
  Step 2: [A, B]        — add B
  Step 3: [A, B, C]     — add C (full)
  Step 4: [B, C, A]     — access A → move to most-recent
  Step 5: [C, A, D]     — add D → evict B (least recently used)
```

- ✅ Effective for workloads with temporal locality
- ✅ Simple to implement with a hash map + doubly linked list
- ❌ A single full scan can evict all hot entries (cache pollution)

### Least Frequently Used (LFU)

Evicts the entry with the fewest accesses. LFU favors entries that have been accessed many times, regardless of recency.

```
  LFU Eviction Example (capacity = 3)
  ─────────────────────────────────────

  Access: A, A, A, B, B, C, D

  After A×3, B×2, C×1:  [A(3), B(2), C(1)]  — full
  Access D:              [A(3), B(2), D(1)]   — evict C (freq=1, least frequent)
```

- ✅ Retains genuinely popular data over a longer window
- ❌ Slow to adapt — previously popular entries linger even after patterns shift
- ❌ More complex to implement (frequency counters, aging)

### First In First Out (FIFO)

Evicts the oldest entry regardless of how often or how recently it was accessed.

```
  FIFO Eviction Example (capacity = 3)
  ──────────────────────────────────────

  Access: A, B, C, D

  Step 1: [A]
  Step 2: [A, B]
  Step 3: [A, B, C]     — full
  Step 4: [B, C, D]     — evict A (first in)
```

- ✅ Simplest to implement — just a queue
- ❌ Does not consider access patterns — may evict hot entries

### Time-To-Live (TTL)

Each entry is assigned an expiration time. Entries are evicted when they expire, regardless of access patterns.

```
  TTL Eviction Timeline
  ─────────────────────

  Key A: SET at t=0, TTL=60s  → expires at t=60s
  Key B: SET at t=10, TTL=30s → expires at t=40s
  Key C: SET at t=20, TTL=90s → expires at t=110s

  t=0    t=40   t=60   t=110
  ├──────┼──────┼──────┼──────
  │  A,B,C      │  A,C │  C   │  (empty)
  │      B expires     A expires   C expires
```

- ✅ Guarantees bounded staleness
- ✅ Combines naturally with other eviction policies
- ❌ Choosing the right TTL is difficult — too short wastes cache, too long serves stale data

### Eviction Policy Comparison

| Policy | Access-Aware | Frequency-Aware | Complexity | Best For |
|---|---|---|---|---|
| **LRU** | Yes (recency) | No | O(1) with linked list | General-purpose, temporal locality |
| **LFU** | No | Yes | O(log n) typical | Stable popularity distributions |
| **FIFO** | No | No | O(1) | Simple use cases, uniform access |
| **TTL** | No | No | O(1) per check | Time-sensitive data, freshness guarantees |
| **LRU + TTL** | Yes | No | O(1) | Most production systems (Redis default) |

---

## CDN Caching

### How CDNs Cache Content

A **Content Delivery Network (CDN)** is a geographically distributed network of proxy servers that cache content close to end users. CDNs reduce latency, offload origin servers, and improve availability.

```
  CDN Caching Flow
  ────────────────

  User (Tokyo)                User (New York)
       │                            │
       ▼                            ▼
  ┌──────────┐                ┌──────────┐
  │ CDN Edge │                │ CDN Edge │
  │ (Tokyo)  │                │ (NYC)    │
  └────┬─────┘                └────┬─────┘
       │  miss                     │  miss
       ▼                           ▼
  ┌──────────────────────────────────────┐
  │           Origin Server              │
  │         (US-East Region)             │
  └──────────────────────────────────────┘
```

### Edge Caching

Edge servers are CDN nodes located in points of presence (PoPs) around the world. When a user requests content, the nearest edge server responds — either from its local cache or by fetching from the origin.

| Scenario | Latency | Origin Load |
|---|---|---|
| Cache hit at edge | 5–20 ms | None |
| Cache miss → fetch from origin | 100–300 ms | One request |
| No CDN | 100–300 ms | Every request |

### Origin Shield

An **origin shield** is an intermediate caching layer between the edge servers and the origin. All edge cache misses are routed through the shield, which coalesces duplicate requests and reduces origin load.

```
  Without Origin Shield              With Origin Shield
  ─────────────────────              ──────────────────

  Edge A ──miss──▶ Origin           Edge A ──miss──▶ ┌────────┐
  Edge B ──miss──▶ Origin           Edge B ──miss──▶ │ Shield │──miss──▶ Origin
  Edge C ──miss──▶ Origin           Edge C ──miss──▶ └────────┘
                                                      (1 origin request
  (3 origin requests)                                  instead of 3)
```

### Cache-Control Headers

HTTP `Cache-Control` headers govern how CDNs (and browsers) cache responses.

| Header | Effect | Example |
|---|---|---|
| `Cache-Control: public, max-age=3600` | CDN and browser cache for 1 hour | Static assets |
| `Cache-Control: private, no-store` | CDN must not cache; browser may not store | User-specific data |
| `Cache-Control: s-maxage=600` | CDN caches for 10 min (overrides max-age for shared caches) | API responses |
| `Cache-Control: no-cache` | Cache must revalidate with origin before serving | Frequently updated pages |
| `Surrogate-Control: max-age=86400` | CDN-specific directive (not forwarded to browser) | CDN-only caching |

---

## Application-Level Caching

### In-Memory Caches

In-memory caches store data in the application process's heap. They provide the lowest possible latency (nanoseconds) but are limited to a single instance and lost on restart.

| Library | Language | Features |
|---|---|---|
| Caffeine | Java | Near-optimal eviction (W-TinyLFU), async loading |
| Guava Cache | Java | Expiration, size-based eviction, refresh |
| `functools.lru_cache` | Python | Simple decorator-based LRU cache |
| `IMemoryCache` | .NET | Sliding/absolute expiration, size limits |
| `node-cache` | Node.js | TTL, key-value store, event hooks |

### Distributed Caches

Distributed caches run as external services, shared across multiple application instances. They survive application restarts and support larger data sets.

**Redis**

- In-memory data structure store supporting strings, hashes, lists, sets, sorted sets
- Supports persistence (RDB snapshots, AOF log), replication, and clustering
- Pub/Sub for cache invalidation across services
- Lua scripting for atomic multi-step operations
- Typical latency: < 1 ms for simple operations

**Memcached**

- Simple key-value cache optimized for high throughput
- Multi-threaded architecture — scales well on multi-core machines
- Consistent hashing for distributing keys across nodes
- No persistence, no data structures beyond key-value strings
- Typical latency: < 1 ms

### Local vs Distributed Comparison

| Dimension | Local (In-Process) | Distributed (Redis/Memcached) |
|---|---|---|
| **Latency** | ~100 ns | ~0.5–1 ms |
| **Capacity** | Limited by process heap | Scales across cluster |
| **Shared across instances** | No | Yes |
| **Survives restarts** | No | Yes (Redis with persistence) |
| **Consistency** | Trivial (single process) | Requires invalidation strategy |
| **Operational overhead** | None | Must manage cache cluster |
| **Best for** | Hot, small, read-only data | Shared state, sessions, large datasets |

---

## Database Query Caching

### Query Result Caching

The results of expensive database queries can be cached to avoid redundant computation. The cache key is typically derived from the query text and its parameters.

```
  Query Result Caching
  ────────────────────

  cache_key = hash("SELECT ... WHERE region = ?", ["us-east"])

  ┌─────────┐  1. lookup key  ┌───────┐
  │   App   │────────────────▶│ Cache │
  │         │                 │       │
  │         │  2a. HIT ◀──────│       │
  │         │     return rows └───────┘
  │         │
  │         │  2b. MISS
  │         │────────────────▶┌──────┐
  │         │  3. execute     │  DB  │
  │         │◀────────────────│      │
  │         │  4. rows        └──────┘
  │         │
  │         │  5. SET key ───▶┌───────┐
  │         │   (with TTL)   │ Cache │
  └─────────┘                └───────┘
```

**Guidelines:**

- ✅ Cache queries that are read-heavy, slow, and produce stable results
- ✅ Include all query parameters in the cache key to avoid collisions
- ❌ Do not cache queries with highly volatile data or low reuse
- ❌ Be cautious with large result sets — they consume cache memory

### Materialized Views

A **materialized view** is a database object that stores the pre-computed result of a query. Unlike a regular view, the data is physically persisted and can be indexed.

| Feature | Regular View | Materialized View |
|---|---|---|
| **Storage** | No (query re-executed) | Yes (persisted result) |
| **Read speed** | Same as underlying query | Fast (reads pre-computed data) |
| **Freshness** | Always current | Stale until refreshed |
| **Refresh** | N/A | Manual or scheduled (`REFRESH MATERIALIZED VIEW`) |
| **Use case** | Abstraction layer | Caching expensive aggregations |

```sql
-- PostgreSQL materialized view example
CREATE MATERIALIZED VIEW daily_sales_summary AS
SELECT
    date_trunc('day', order_date) AS day,
    region,
    SUM(amount) AS total_sales,
    COUNT(*) AS order_count
FROM orders
GROUP BY 1, 2;

-- Refresh periodically (e.g., via cron job)
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales_summary;
```

---

## Cache Invalidation

### The Hard Problem

> "There are only two hard things in Computer Science: cache invalidation and naming things."
> — Phil Karlton

Cache invalidation is difficult because it requires answering: **when does cached data become stale, and how do all consumers learn about the change?** In a distributed system with multiple cache layers, this is a coordination problem.

### Time-Based Invalidation

Set a **TTL (Time-To-Live)** on each cache entry. After the TTL expires, the entry is evicted or refreshed.

| TTL Value | Trade-off |
|---|---|
| Short (5–30s) | Low staleness, higher origin load |
| Medium (1–10 min) | Balanced for most use cases |
| Long (1–24 hr) | Minimal origin load, higher staleness risk |

- ✅ Simple to implement — no coordination required
- ❌ All reads between a data change and TTL expiry receive stale data

### Event-Based Invalidation

Invalidate cache entries in response to data change events (e.g., after a database write).

```
  Event-Based Invalidation
  ────────────────────────

  ┌─────────┐  1. UPDATE user  ┌──────┐
  │   App   │─────────────────▶│  DB  │
  │         │                  └──────┘
  │         │
  │         │  2. DELETE cache key
  │         │─────────────────▶┌───────┐
  │         │                  │ Cache │
  └─────────┘                  └───────┘
```

- ✅ Cache is invalidated immediately after changes
- ❌ Requires the write path to know which cache keys to invalidate
- ❌ Risk of race conditions between concurrent reads and writes

### Version-Based Invalidation

Embed a version number or hash in the cache key. When data changes, increment the version — old keys are never read again and eventually evicted.

```
  Version-Based Keys
  ──────────────────

  Before update:  cache key = "user:42:v3"   → cached data
  After update:   cache key = "user:42:v4"   → cache miss → fetch from DB

  Old key "user:42:v3" is never accessed again → evicted by LRU/TTL
```

- ✅ No explicit deletion required — stale keys are naturally orphaned
- ✅ No race conditions — readers and writers use different keys
- ❌ Orphaned keys consume memory until evicted

### Pub/Sub Invalidation

Use a **publish/subscribe** messaging system to broadcast invalidation events to all cache-holding services.

```
  Pub/Sub Cache Invalidation
  ──────────────────────────

  ┌────────────┐  1. write  ┌──────┐
  │  Service A │───────────▶│  DB  │
  │            │            └──────┘
  │            │
  │            │  2. publish "invalidate user:42"
  │            │──────────────────────┐
  └────────────┘                      ▼
                               ┌────────────┐
                               │  Pub/Sub   │
                               │  (Redis /  │
                               │   Kafka)   │
                               └─────┬──┬───┘
                                     │  │
                           ┌─────────┘  └─────────┐
                           ▼                      ▼
                    ┌────────────┐         ┌────────────┐
                    │  Service B │         │  Service C │
                    │  (delete   │         │  (delete   │
                    │   local    │         │   local    │
                    │   cache)   │         │   cache)   │
                    └────────────┘         └────────────┘
```

- ✅ All services learn about invalidation in near real-time
- ✅ Decouples the write path from knowledge of all consumers
- ❌ Pub/Sub delivery is typically at-most-once — messages can be lost
- ❌ Adds infrastructure dependency (message broker)

---

## Cache Consistency

### Consistency Challenges in Distributed Caches

When cache and database are separate systems, there is no transactional guarantee spanning both. This creates a window for inconsistency.

```
  Race Condition: Read-After-Write Inconsistency
  ───────────────────────────────────────────────

  Thread 1 (writer)              Thread 2 (reader)
  ─────────────────              ─────────────────
  1. DELETE cache key
                                 2. GET cache key → MISS
                                 3. READ from DB → old value
  4. UPDATE DB → new value
                                 5. SET cache key → old value  ← STALE!
```

**Mitigation strategies:**

- Use write-through caching to keep cache and DB in sync
- Apply short TTLs so stale data is bounded in duration
- Use distributed locks to serialize read-modify-write sequences

### Thundering Herd Problem

When a popular cache entry expires, many concurrent requests simultaneously experience a cache miss and all hit the origin, creating a load spike.

```
  Thundering Herd
  ───────────────

  Time ─────────────────────────────────────────▶

  Cache entry expires at t=60s

  t=60.001  Request A → MISS → query DB ──┐
  t=60.002  Request B → MISS → query DB ──┤
  t=60.003  Request C → MISS → query DB ──┼──▶ DB overloaded
  t=60.004  Request D → MISS → query DB ──┤
  t=60.005  Request E → MISS → query DB ──┘
```

**Mitigation strategies:**

| Strategy | How It Works |
|---|---|
| **Locking / single-flight** | Only one request fetches from the origin; others wait for the result |
| **Stale-while-revalidate** | Serve the expired entry while one request refreshes it in the background |
| **Jittered TTLs** | Add random variation to TTLs so entries do not expire at the same time |
| **Refresh-ahead** | Proactively refresh before expiry so there is never a miss |

### Cache Stampede

A cache stampede is a broader form of the thundering herd that occurs when many **different** keys expire simultaneously (e.g., after a cache restart or a bulk invalidation).

```
  Cache Stampede (after cache restart)
  ────────────────────────────────────

  t=0: Cache comes back online, completely empty

  ┌──────────┐
  │ 10,000   │──all miss──▶┌──────┐
  │ requests │             │  DB  │ ← overwhelmed
  │          │◀────────────│      │
  └──────────┘             └──────┘
```

**Mitigation strategies:**

- **Cache warming** — pre-populate the cache with hot keys before routing traffic
- **Gradual ramp-up** — slowly increase traffic to the new cache instance
- **Circuit breaker** — shed load if the origin is overwhelmed
- **Request coalescing** — deduplicate identical in-flight requests

---

## Monitoring and Metrics

Effective cache operation requires ongoing monitoring. The following metrics are essential.

### Cache Hit Ratio

The single most important cache metric. A declining hit ratio indicates cache-sizing issues, eviction pressure, or changing access patterns.

```
  Hit Ratio = hits / (hits + misses)

  ┌─────────────────────────────────────────────────┐
  │  Hit Ratio Over Time                            │
  │                                                 │
  │  99% ┤ ·····················                    │
  │  95% ┤                     ····                 │
  │  90% ┤                         ···              │
  │  85% ┤                            ··· ← investigate│
  │  80% ┤                               ···       │
  │      └──────────────────────────────────────    │
  │       00:00   06:00   12:00   18:00   24:00     │
  └─────────────────────────────────────────────────┘
```

### Latency

Track cache read/write latency percentiles (p50, p95, p99). Latency spikes can indicate network issues, hot keys, or resource contention.

| Metric | Healthy Range | Investigate When |
|---|---|---|
| p50 read latency | < 0.5 ms | > 1 ms |
| p99 read latency | < 2 ms | > 5 ms |
| p50 write latency | < 0.5 ms | > 1 ms |
| p99 write latency | < 2 ms | > 10 ms |

### Eviction Rate

The number of entries evicted per second. A high eviction rate means the cache is too small for the working set or TTLs are misconfigured.

| Metric | What It Tells You |
|---|---|
| **Eviction rate** | How often entries are removed to make room for new ones |
| **Memory usage** | How close the cache is to its capacity limit |
| **Key count** | Total number of entries in the cache |
| **Expired key count** | Entries removed due to TTL expiry (not memory pressure) |

**Alerting guidelines:**

- ✅ Alert if hit ratio drops below baseline by more than 5 percentage points
- ✅ Alert if eviction rate spikes above historical norms
- ✅ Alert if p99 latency exceeds your SLO
- ❌ Do not alert on every individual cache miss — focus on trends

---

## Next Steps

Continue to the next topic to explore additional system design concepts including load balancing, rate limiting, and message queue architectures.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial version — cache patterns, eviction policies, CDN caching, application-level caching, query caching, invalidation, consistency, monitoring |
