# Caching Fundamentals

## Table of Contents

1. [Overview](#overview)
   - [What Is Caching?](#what-is-caching)
   - [Why Caching Matters](#why-caching-matters)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [The Memory Hierarchy](#the-memory-hierarchy)
   - [Latency Comparison](#latency-comparison)
   - [Memory Hierarchy Diagram](#memory-hierarchy-diagram)
   - [Bandwidth and Throughput](#bandwidth-and-throughput)
3. [Cache Fundamentals](#cache-fundamentals)
   - [Cache Hit and Miss](#cache-hit-and-miss)
   - [Hit Ratio Formulas](#hit-ratio-formulas)
   - [TTL — Time to Live](#ttl--time-to-live)
   - [Cache Warm-Up and Cold Start](#cache-warm-up-and-cold-start)
   - [Cache Penetration](#cache-penetration)
   - [Cache Avalanche](#cache-avalanche)
   - [Cache Stampede](#cache-stampede)
4. [Types of Caches](#types-of-caches)
   - [In-Process / Local Caches](#in-process--local-caches)
   - [Distributed / Remote Caches](#distributed--remote-caches)
   - [CDN and Edge Caches](#cdn-and-edge-caches)
   - [Browser Caches](#browser-caches)
   - [Database-Level Caches](#database-level-caches)
   - [Cache Type Comparison](#cache-type-comparison)
5. [Eviction Policies](#eviction-policies)
   - [LRU — Least Recently Used](#lru--least-recently-used)
   - [LFU — Least Frequently Used](#lfu--least-frequently-used)
   - [FIFO — First In First Out](#fifo--first-in-first-out)
   - [Random Eviction](#random-eviction)
   - [TTL-Based Eviction](#ttl-based-eviction)
   - [ARC — Adaptive Replacement Cache](#arc--adaptive-replacement-cache)
   - [W-TinyLFU](#w-tinylfu)
   - [Eviction Policy Comparison](#eviction-policy-comparison)
6. [Cache Topologies](#cache-topologies)
   - [Embedded Cache](#embedded-cache)
   - [Client-Server Cache](#client-server-cache)
   - [Sidecar Proxy Cache](#sidecar-proxy-cache)
   - [Multi-Tier Cache (L1 + L2)](#multi-tier-cache-l1--l2)
7. [When to Cache (and When NOT to)](#when-to-cache-and-when-not-to)
   - [Decision Framework](#decision-framework)
   - [Good Candidates for Caching](#good-candidates-for-caching)
   - [Poor Candidates for Caching](#poor-candidates-for-caching)
   - [Read-Heavy vs Write-Heavy Workloads](#read-heavy-vs-write-heavy-workloads)
   - [Data Freshness Requirements](#data-freshness-requirements)
8. [The Caching Landscape](#the-caching-landscape)
   - [Tool Overview](#tool-overview)
   - [Choosing the Right Tool](#choosing-the-right-tool)
9. [Prerequisites](#prerequisites)
10. [Next Steps](#next-steps)
    - [Suggested Learning Path by Role](#suggested-learning-path-by-role)

---

## Overview

### What Is Caching?

Caching is the practice of storing copies of data in a **high-speed storage layer** so that
future requests for that data can be served faster than fetching it from the original source.
Rather than re-computing an expensive calculation or re-querying a slow database every time,
a cache keeps the result readily available.

```
┌──────────┐         ┌──────────┐         ┌──────────────┐
│          │  GET    │          │  MISS   │              │
│  Client  │────────►│  Cache   │────────►│  Origin      │
│          │◄────────│          │◄────────│  (DB / API)  │
│          │   HIT   │          │  FILL   │              │
└──────────┘         └──────────┘         └──────────────┘
```

At its core, caching trades **space** (memory) for **time** (latency). This seemingly simple
idea is one of the most powerful performance optimisation techniques in all of computing,
appearing at every layer from CPU hardware to globally distributed CDN networks.

### Why Caching Matters

- **Latency** — A cache hit served from memory takes microseconds; a database query can take
  milliseconds to seconds. For user-facing applications, every millisecond counts.
- **Throughput** — Caching offloads work from backend systems, allowing them to handle
  more concurrent requests without scaling vertically.
- **Cost** — Reducing calls to databases, third-party APIs, and compute-heavy services
  directly lowers infrastructure spend.
- **Resilience** — A warm cache can continue serving stale data during partial outages,
  improving system availability.

### Target Audience

- **Backend engineers** building web services and APIs
- **Site Reliability Engineers (SREs)** tuning performance and capacity
- **Software architects** designing scalable distributed systems
- **Platform engineers** operating shared caching infrastructure

### Scope

This document covers caching **concepts, terminology, and decision frameworks**. It is
technology-agnostic where possible but references popular tools (Redis, Memcached, Caffeine,
Varnish, etc.) for concrete examples. Detailed deep-dives into individual tools and
advanced patterns are covered in companion documents listed in [Next Steps](#next-steps).

---

## The Memory Hierarchy

To understand why caching works, you must understand the **memory hierarchy** — the
layered storage architecture that trades capacity for speed.

### Memory Hierarchy Diagram

```
              Speed                           Capacity
          ◄───────────                    ───────────►

        ┌───────────────┐
        │   CPU L1 Cache│  ~1 ns       32–64 KB
        ├───────────────┤
        │   CPU L2 Cache│  ~4 ns       256 KB – 1 MB
        ├───────────────┤
        │   CPU L3 Cache│  ~10 ns      4–64 MB
        ├───────────────┤
        │      RAM      │  ~100 ns     8–512 GB
        ├───────────────┤
        │    NVMe SSD   │  ~25 μs      256 GB – 8 TB
        ├───────────────┤
        │    SATA SSD   │  ~100 μs     256 GB – 8 TB
        ├───────────────┤
        │      HDD      │  ~5 ms       1–20 TB
        ├───────────────┤
        │    Network     │  ~1–100 ms   Unlimited
        └───────────────┘

     Fastest / Smallest                 Slowest / Largest
```

### Latency Comparison

| Storage Layer | Typical Latency | Approx. Capacity | Relative Speed |
|---|---|---|---|
| **CPU L1 cache** | ~1 ns | 32–64 KB | 1× (baseline) |
| **CPU L2 cache** | ~4 ns | 256 KB – 1 MB | 4× slower |
| **CPU L3 cache** | ~10 ns | 4–64 MB | 10× slower |
| **Main memory (RAM)** | ~100 ns | 8–512 GB | 100× slower |
| **NVMe SSD** | ~25 μs | 256 GB – 8 TB | 25,000× slower |
| **SATA SSD** | ~100 μs | 256 GB – 8 TB | 100,000× slower |
| **HDD (spinning disk)** | ~5 ms | 1–20 TB | 5,000,000× slower |
| **Network round-trip (same DC)** | ~0.5 ms | — | 500,000× slower |
| **Network round-trip (cross-region)** | ~50–150 ms | — | 50,000,000× slower |

> **Key insight:** Reading from RAM is roughly **100,000× faster** than reading from an SSD
> and **5 million times faster** than a spinning disk seek. This is why in-memory caching
> delivers such dramatic performance improvements.

### Bandwidth and Throughput

Latency is only half the picture. Each layer also has different **bandwidth**:

| Layer | Typical Bandwidth |
|---|---|
| **L1 cache** | ~1 TB/s |
| **RAM (DDR5)** | ~50–100 GB/s |
| **NVMe SSD** | ~3–7 GB/s |
| **SATA SSD** | ~500 MB/s |
| **HDD** | ~100–200 MB/s |
| **1 Gbps network** | ~125 MB/s |
| **10 Gbps network** | ~1.25 GB/s |

When designing caching layers, consider **both** latency and bandwidth. A cache hit avoids
not only the slow seek time but also frees bandwidth on the slower tier.

---

## Cache Fundamentals

### Cache Hit and Miss

Every cache lookup has one of two outcomes:

```
                    ┌──────────┐
                    │  Lookup  │
                    └─────┬────┘
                          │
                    ┌─────▼─────┐
                    │ Key found? │
                    └──┬─────┬──┘
                   YES │     │ NO
                 ┌─────▼┐  ┌▼──────┐
                 │ HIT  │  │ MISS  │
                 └──────┘  └───┬───┘
                               │
                         ┌─────▼──────┐
                         │ Fetch from │
                         │   origin   │
                         └─────┬──────┘
                               │
                         ┌─────▼──────┐
                         │ Store in   │
                         │   cache    │
                         └────────────┘
```

- **Cache hit** — The requested data is found in the cache. Fast path.
- **Cache miss** — The data is not in the cache. The system must fetch it from the
  origin (database, upstream API, disk), then optionally store it in the cache for
  future requests.

### Hit Ratio Formulas

The **hit ratio** (or hit rate) is the single most important metric for evaluating
cache effectiveness:

```
                        hits
  Hit Ratio  =  ─────────────────
                  hits + misses

  Miss Ratio  =  1 − Hit Ratio

                      total latency saved by cache hits
  Effective Latency = ──────────────────────────────────
                             total requests
```

**Practical targets:**

| Use Case | Target Hit Ratio | Notes |
|---|---|---|
| CDN for static assets | > 95% | Static content should almost always be cached |
| Application data cache | 80–95% | Depends on data volatility |
| Database query cache | 70–90% | Highly query-dependent |
| DNS cache | > 99% | DNS records change infrequently |

**Example calculation:**

```
  Requests = 10,000
  Hits     = 8,500
  Misses   = 1,500

  Hit Ratio = 8,500 / 10,000 = 85%

  If cache latency  = 2 ms
  If origin latency = 50 ms

  Avg latency = (0.85 × 2) + (0.15 × 50)
              = 1.7 + 7.5
              = 9.2 ms   (vs 50 ms without cache → 5.4× improvement)
```

### TTL — Time to Live

TTL defines **how long** a cached entry remains valid before it expires automatically.

```
  ┌──────────┐     SET key=value TTL=300s     ┌──────────┐
  │  Client   │──────────────────────────────►│  Cache   │
  └──────────┘                                 └──────────┘

  Time 0s    → Entry stored, counter starts
  Time 150s  → Entry still valid (150s remaining)
  Time 300s  → Entry expires, removed on next access
  Time 301s  → Cache miss; must re-fetch from origin
```

**TTL trade-offs:**

| Short TTL (seconds) | Long TTL (hours/days) |
|---|---|
| Fresher data | Higher hit ratio |
| Lower hit ratio | Risk of stale data |
| More origin load | Lower origin load |
| Better for volatile data | Better for static data |

### Cache Warm-Up and Cold Start

- **Cold start** — When a cache is empty (after a deploy, restart, or new node),
  every request is a miss. This causes a sudden load spike on the origin.
- **Warm-up** — The process of pre-populating the cache before it receives production
  traffic. Strategies include:
  - **Passive warm-up** — Let traffic naturally fill the cache over time
  - **Active warm-up** — Run a script that pre-loads hot keys before traffic is routed
  - **Cache seeding** — Replicate data from another cache node or a snapshot

```
  Load on Origin

  │
  │ ████
  │ ████████
  │ ████████████
  │ ████████████████
  │ ████████████████████
  │ ████████████████████████   ← cold start spike
  │ ████████████████████████
  │     ████████████████       ← cache warming
  │         ████████           ← steady state (warm cache)
  │             ████
  └───────────────────────────── Time
```

### Cache Penetration

Cache penetration occurs when requests are made for keys that **do not exist** in either
the cache or the origin. Every request becomes a miss that also fails at the origin.

**Mitigation strategies:**

- **Null-object caching** — Store a sentinel value (e.g., empty string or `null`) for
  known-missing keys with a short TTL
- **Bloom filter** — Use a probabilistic data structure to quickly reject lookups for
  keys that definitely do not exist
- **Request validation** — Validate input before hitting the cache

```python
# Null-object caching example
def get_user(user_id: str) -> Optional[User]:
    cached = cache.get(f"user:{user_id}")
    if cached is not None:
        return None if cached == "__NULL__" else cached

    user = database.find_user(user_id)
    if user is None:
        cache.set(f"user:{user_id}", "__NULL__", ttl=60)  # short TTL for negatives
        return None

    cache.set(f"user:{user_id}", user, ttl=3600)
    return user
```

### Cache Avalanche

Cache avalanche happens when a **large number of cache entries expire simultaneously**,
causing a flood of requests to the origin that can overwhelm it.

**Mitigation strategies:**

- **Staggered TTLs** — Add a random jitter to TTL values so entries do not expire at the
  same time: `TTL = base_ttl + random(0, jitter_range)`
- **Circuit breaker** — If the origin is overloaded, fail fast and serve stale data
- **Locking / single-flight** — Ensure only one request fetches a missing key while
  others wait for the result

```go
// Staggered TTL with jitter (Go)
import (
    "math/rand"
    "time"
)

func setWithJitter(cache Cache, key string, value interface{}, baseTTL time.Duration) {
    jitter := time.Duration(rand.Int63n(int64(baseTTL / 5)))
    cache.Set(key, value, baseTTL+jitter)
}
```

### Cache Stampede

A cache stampede (also called **thundering herd**) occurs when a single popular key
expires and many concurrent requests all try to recompute or fetch the value simultaneously.

**Mitigation strategies:**

- **Locking / mutex** — Only the first requester computes the value; others wait
- **Probabilistic early expiration** — Recompute the value before it actually expires,
  based on a probability that increases as the TTL approaches zero
- **Lease-based approach** — Grant a short lease to one requester; others get stale data

```java
// Single-flight pattern (Java pseudocode)
public V getOrCompute(String key, Supplier<V> loader) {
    V cached = cache.get(key);
    if (cached != null) return cached;

    Lock lock = locks.get(key);
    if (lock.tryLock()) {
        try {
            // Double-check after acquiring lock
            cached = cache.get(key);
            if (cached != null) return cached;

            V value = loader.get();
            cache.set(key, value, DEFAULT_TTL);
            return value;
        } finally {
            lock.unlock();
        }
    } else {
        // Another thread is computing; wait and retry
        lock.lock();
        lock.unlock();
        return cache.get(key);
    }
}
```

---

## Types of Caches

### In-Process / Local Caches

In-process caches live **inside the application's memory space**. They are the fastest
type of cache because there is no network hop — just a hash-map lookup.

| Library | Language | Key Features |
|---|---|---|
| **Caffeine** | Java | W-TinyLFU eviction, async loading, near-optimal hit rate |
| **Guava Cache** | Java | Weight-based eviction, expiry policies, statistics |
| **IMemoryCache** | .NET | Built-in to ASP.NET Core, size limits, expiry callbacks |
| **lru-cache** | Node.js | Simple LRU with max-size and TTL support |
| **cachetools** | Python | LRU, LFU, TTL decorators for functions |

```csharp
// .NET IMemoryCache example
public class ProductService
{
    private readonly IMemoryCache _cache;
    private readonly IProductRepository _repo;

    public ProductService(IMemoryCache cache, IProductRepository repo)
    {
        _cache = cache;
        _repo = repo;
    }

    public async Task<Product> GetProductAsync(int id)
    {
        return await _cache.GetOrCreateAsync($"product:{id}", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
            entry.SlidingExpiration = TimeSpan.FromMinutes(2);
            return await _repo.GetByIdAsync(id);
        });
    }
}
```

**Trade-offs:**

- ✅ Lowest possible latency (no serialization, no network)
- ✅ No external dependency
- ❌ Cache is **per-instance** — no sharing between application replicas
- ❌ Memory is bounded by the application's heap

### Distributed / Remote Caches

Distributed caches run as a **separate service** that multiple application instances share.

| Tool | Protocol | Data Structures | Clustering | Persistence |
|---|---|---|---|---|
| **Redis** | RESP | Strings, hashes, lists, sets, sorted sets, streams | Redis Cluster / Sentinel | RDB + AOF |
| **Valkey** | RESP | Same as Redis (fork) | Same as Redis | RDB + AOF |
| **Memcached** | ASCII / binary | Key-value (strings only) | Client-side sharding | None |
| **Hazelcast** | Custom | Maps, queues, topics, locks | Built-in (CP subsystem) | Hot restart |

```bash
# Redis CLI example
$ redis-cli SET user:1001 '{"name":"Alice","role":"admin"}' EX 3600
OK
$ redis-cli GET user:1001
"{\"name\":\"Alice\",\"role\":\"admin\"}"
$ redis-cli TTL user:1001
(integer) 3594
```

**Trade-offs:**

- ✅ Shared across all application instances — consistent view
- ✅ Can scale independently of the application
- ✅ Rich data structures (Redis)
- ❌ Network latency on every access (~0.5–2 ms)
- ❌ Requires serialization / deserialization overhead
- ❌ Additional infrastructure to operate

### CDN and Edge Caches

Content Delivery Networks cache content at **points of presence (PoPs)** close to end users.
They excel at caching static assets (images, CSS, JS) and can also cache API responses.

```
                         ┌─────────┐
                         │  Origin │
                         └────┬────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
         ┌────▼────┐    ┌────▼────┐    ┌────▼────┐
         │  PoP    │    │  PoP    │    │  PoP    │
         │ US-East │    │ EU-West │    │ AP-SE   │
         └────┬────┘    └────┬────┘    └────┬────┘
              │               │               │
         ┌────▼────┐    ┌────▼────┐    ┌────▼────┐
         │  Users  │    │  Users  │    │  Users  │
         │ Americas│    │  Europe │    │   Asia  │
         └─────────┘    └─────────┘    └─────────┘
```

Examples: **CloudFront**, **Cloudflare**, **Akamai**, **Fastly**, **Azure CDN**.

### Browser Caches

The browser maintains its own cache controlled by HTTP headers:

| Header | Purpose | Example |
|---|---|---|
| `Cache-Control` | Directives for caching behaviour | `max-age=3600, public` |
| `ETag` | Version identifier for conditional requests | `"abc123"` |
| `Last-Modified` | Timestamp for conditional requests | `Wed, 15 Apr 2025 12:00:00 GMT` |
| `Vary` | Specifies which request headers affect caching | `Vary: Accept-Encoding` |
| `Expires` | Legacy absolute expiration date | `Thu, 16 Apr 2025 12:00:00 GMT` |

```
  Browser Cache Flow (Conditional Request)

  ┌─────────┐           ┌──────────┐           ┌──────────┐
  │ Browser │           │  Cache   │           │  Server  │
  └────┬────┘           └─────┬────┘           └─────┬────┘
       │  GET /style.css      │                      │
       │─────────────────────►│                      │
       │                      │  Cache expired?      │
       │                      │  YES → conditional   │
       │                      │  GET If-None-Match    │
       │                      │─────────────────────►│
       │                      │     304 Not Modified  │
       │                      │◄─────────────────────│
       │  200 (from cache)    │                      │
       │◄─────────────────────│                      │
       │                      │                      │
```

### Database-Level Caches

Databases include their own internal caching mechanisms:

| Database | Cache Type | What It Caches |
|---|---|---|
| **PostgreSQL** | Shared buffer pool | Data pages (8 KB blocks) in RAM |
| **MySQL/InnoDB** | Buffer pool | Data and index pages |
| **MySQL** | Query cache (deprecated 8.0) | Exact query result sets |
| **MongoDB** | WiredTiger cache | Documents and indexes in memory |
| **SQL Server** | Buffer pool | Data pages, plan cache for query plans |

These are automatic and managed by the database engine. They complement but do not
replace application-level caching.

### Cache Type Comparison

| Aspect | In-Process | Distributed | CDN / Edge | Browser | DB Buffer |
|---|---|---|---|---|---|
| **Latency** | ~ns–μs | ~0.5–2 ms | ~5–50 ms | ~0 ms | ~μs |
| **Shared** | No (per instance) | Yes | Yes (global) | No (per user) | No (per DB) |
| **Capacity** | MB–low GB | GB–TB | TB+ | MB | GB–TB |
| **Consistency** | Trivial | Needs strategy | Eventual | Per-user | Automatic |
| **Ops overhead** | None | Medium–High | Low (managed) | None | Low |
| **Best for** | Hot data, configs | Session, API data | Static assets | Assets, pages | Query results |

---

## Eviction Policies

When a cache reaches its capacity limit, it must decide which entries to remove.
The eviction policy determines this strategy.

### LRU — Least Recently Used

Evicts the entry that has **not been accessed for the longest time**.

```
  Access sequence: A B C D A B E

  Cache (capacity=4):

  Step 1: [A]              ← A added
  Step 2: [A, B]           ← B added
  Step 3: [A, B, C]        ← C added
  Step 4: [A, B, C, D]     ← D added (full)
  Step 5: [B, C, D, A]     ← A accessed, moved to front
  Step 6: [C, D, A, B]     ← B accessed, moved to front
  Step 7: [D, A, B, E]     ← E added, C evicted (least recent)
```

**Pros:** Simple, effective for recency-biased workloads, O(1) with hash-map + doubly-linked list.
**Cons:** Susceptible to scan pollution (a one-time sequential scan evicts hot entries).
**Use when:** Access patterns show temporal locality (recently used items are likely used again).

### LFU — Least Frequently Used

Evicts the entry with the **lowest access count**.

```
  Key  │ Access Count │ Status
  ─────┼──────────────┼─────────
  A    │     12       │ Keep
  B    │      3       │ Keep
  C    │      1       │ ← Evict (lowest frequency)
  D    │      8       │ Keep
```

**Pros:** Excellent for workloads with stable popularity (e.g., product catalog).
**Cons:** Slow to adapt — historically popular items stay cached even if no longer accessed.
         Needs frequency aging or decay to handle shifting patterns.
**Use when:** Some items are consistently more popular than others over long periods.

### FIFO — First In First Out

Evicts the **oldest entry** regardless of access pattern.

**Pros:** Simplest to implement (just a queue). Low overhead.
**Cons:** Does not consider access recency or frequency — poor hit ratio for most workloads.
**Use when:** All items have roughly equal access probability, or items have natural
time-based relevance (e.g., time-series data windows).

### Random Eviction

Evicts a **randomly chosen** entry.

**Pros:** O(1) eviction, no bookkeeping overhead, no pathological worst case.
**Cons:** Unpredictable — may evict hot entries.
**Use when:** Cache is large and access is mostly uniform, or as a fallback when
simplicity and speed matter more than optimal hit ratio.

### TTL-Based Eviction

Entries expire after a **fixed time period** from creation or last access.

- **Absolute TTL** — Expires N seconds after creation
- **Sliding TTL** — Expires N seconds after the last access (resets on each hit)

**Pros:** Ensures data freshness. Simple to reason about.
**Cons:** Does not consider cache fullness — entries may expire even when space is available,
         or the cache may fill up before TTLs expire.
**Use when:** Data has a known staleness tolerance (e.g., "product prices are valid for 5 minutes").

### ARC — Adaptive Replacement Cache

ARC maintains **two LRU lists** (recent vs frequent) and dynamically adjusts the
partition between them based on workload behaviour.

```
  ┌──────────────────────────────────────────────────┐
  │                   ARC Cache                      │
  │                                                  │
  │  Ghost list B1     List T1       List T2    Ghost list B2  │
  │  (evicted from    (recently    (frequently  (evicted from  │
  │   T1 metadata)     seen once)   seen 2+)    T2 metadata)  │
  │                                                  │
  │  ◄─── adaptive partition point p ───►            │
  │                                                  │
  │  Hit in B1 → grow T1 (more recency)             │
  │  Hit in B2 → grow T2 (more frequency)           │
  └──────────────────────────────────────────────────┘
```

**Pros:** Self-tuning — adapts to both recency and frequency patterns without manual tuning.
**Cons:** More complex to implement. Higher per-operation overhead than simple LRU.
**Use when:** Workload characteristics are unknown or change over time (e.g., general-purpose
file system caches, database buffer pools).

### W-TinyLFU

W-TinyLFU (Windowed Tiny Least Frequently Used) combines a **small admission window**
(LRU) with a **main cache** (segmented LRU) and uses a **frequency sketch** (Count-Min
Sketch) to estimate item frequency without storing full counters.

```
  ┌─────────────┐     ┌─────────────────────────────────┐
  │  Admission  │     │          Main Cache              │
  │   Window    │     │                                  │
  │   (1% LRU)  │────►│  Probation (20%)  │ Protected   │
  │             │     │   (segmented LRU) │   (80%)     │
  └─────────────┘     └─────────────────────────────────┘
         │
         │ Admission filter: is candidate more frequent
         │ than the item it would evict? (TinyLFU sketch)
         │
         └─── If NO → candidate is rejected (not cached)
```

**Pros:** Near-optimal hit ratio across diverse workloads. Resistant to scan pollution.
         Low memory overhead for frequency tracking.
**Cons:** Most complex to implement correctly. Available via libraries (Caffeine for Java).
**Use when:** You need the best possible hit ratio and can use a library like Caffeine.

### Eviction Policy Comparison

| Policy | Hit Ratio | Overhead | Scan Resistant | Adaptive | Best For |
|---|---|---|---|---|---|
| **LRU** | Good | Low (O(1)) | No | No | General-purpose, temporal locality |
| **LFU** | Good | Medium | Yes | No | Stable popularity distributions |
| **FIFO** | Fair | Very Low | No | No | Time-series, simple use cases |
| **Random** | Fair | Very Low | Partially | No | Uniform access, simplicity |
| **TTL-based** | Varies | Low | N/A | No | Freshness-critical data |
| **ARC** | Very Good | Medium | Yes | Yes | Unknown/changing workloads |
| **W-TinyLFU** | Excellent | Medium | Yes | Yes | Maximum hit ratio (use Caffeine) |

---

## Cache Topologies

### Embedded Cache

The cache runs **inside the application process** as a library.

```
  ┌─────────────────────────────────┐
  │        Application Process      │
  │                                 │
  │   ┌───────────────────────┐     │
  │   │     Business Logic    │     │
  │   └───────────┬───────────┘     │
  │               │                 │
  │   ┌───────────▼───────────┐     │
  │   │   Embedded Cache      │     │
  │   │   (Caffeine, Guava,   │     │
  │   │    IMemoryCache)      │     │
  │   └───────────────────────┘     │
  │                                 │
  └─────────────────────────────────┘
```

- **Latency:** Nanoseconds (in-process memory access)
- **Sharing:** None — each instance has its own cache
- **Failure mode:** Cache is lost when the process restarts
- **Use case:** Configuration, small reference data, function memoization

### Client-Server Cache

The cache runs as a **dedicated external service** that applications connect to over the network.

```
  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
  │   App        │     │   App        │     │   App        │
  │  Instance 1  │     │  Instance 2  │     │  Instance 3  │
  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘
         │                    │                    │
         └────────────┬───────┴──────────┬─────────┘
                      │   Network        │
               ┌──────▼──────────────────▼──────┐
               │                                │
               │     Redis / Memcached /        │
               │     Hazelcast Cluster          │
               │                                │
               │  ┌──────┐ ┌──────┐ ┌──────┐   │
               │  │Node 1│ │Node 2│ │Node 3│   │
               │  └──────┘ └──────┘ └──────┘   │
               │                                │
               └────────────────────────────────┘
```

- **Latency:** ~0.5–2 ms (network round-trip)
- **Sharing:** All application instances share the same cache
- **Failure mode:** Partial — cluster can tolerate node failures
- **Use case:** Session storage, API response caching, rate limiting

### Sidecar Proxy Cache

A cache proxy runs as a **sidecar container** alongside the application (common in Kubernetes).
The sidecar intercepts cache calls and can add features like connection pooling, local
caching, or protocol translation.

```
  ┌──────────────────────────────────────────┐
  │              Pod / Host                  │
  │                                          │
  │  ┌──────────────┐   ┌────────────────┐  │
  │  │              │   │                │  │
  │  │  Application │──►│  Cache Sidecar │──────►  Redis Cluster
  │  │              │   │  (e.g., Envoy, │  │
  │  │              │   │   Twemproxy)   │  │
  │  └──────────────┘   └────────────────┘  │
  │                                          │
  └──────────────────────────────────────────┘
```

- **Latency:** ~0.1–0.5 ms (localhost hop + network to cluster)
- **Sharing:** Depends on sidecar configuration
- **Use case:** Connection pooling, transparent sharding, protocol bridging

### Multi-Tier Cache (L1 + L2)

Combines an **L1 in-process cache** with an **L2 distributed cache** for the best of both.

```
  ┌─────────────────────────────────────┐
  │          Application Instance       │
  │                                     │
  │   Request                           │
  │      │                              │
  │      ▼                              │
  │   ┌──────────┐                      │
  │   │ L1 Cache │  (in-process)        │
  │   │ Caffeine │  Latency: ~ns        │
  │   └─────┬────┘                      │
  │         │ MISS                      │
  │         ▼                           │
  └─────────┼───────────────────────────┘
            │
            ▼
     ┌──────────────┐
     │   L2 Cache   │  (distributed)
     │    Redis     │  Latency: ~1 ms
     └──────┬───────┘
            │ MISS
            ▼
     ┌──────────────┐
     │    Origin    │  (database)
     │  PostgreSQL  │  Latency: ~5–50 ms
     └──────────────┘
```

**How it works:**

1. Check L1 (in-process). If **HIT**, return immediately (~nanoseconds).
2. If L1 MISS, check L2 (distributed). If **HIT**, populate L1, return (~1 ms).
3. If L2 MISS, query origin. Populate both L2 and L1, return (~5–50 ms).

```python
# Multi-tier cache (Python pseudocode)
class MultiTierCache:
    def __init__(self, l1: LocalCache, l2: RedisCache, origin: Database):
        self.l1 = l1
        self.l2 = l2
        self.origin = origin

    def get(self, key: str):
        # L1 lookup (in-process, nanoseconds)
        value = self.l1.get(key)
        if value is not None:
            return value

        # L2 lookup (Redis, ~1 ms)
        value = self.l2.get(key)
        if value is not None:
            self.l1.set(key, value, ttl=60)  # backfill L1
            return value

        # Origin lookup (database, ~5-50 ms)
        value = self.origin.query(key)
        if value is not None:
            self.l2.set(key, value, ttl=3600)   # backfill L2
            self.l1.set(key, value, ttl=60)     # backfill L1
        return value
```

**Invalidation challenge:** When data changes, you must invalidate **both** L1 and L2.
L2 invalidation is straightforward (delete the key). L1 invalidation across multiple
instances requires a pub/sub mechanism (e.g., Redis Pub/Sub or a message bus).

---

## When to Cache (and When NOT to)

### Decision Framework

Ask these questions before adding a cache:

```
  Is the data read significantly more often        NO
  than it is written?  ──────────────────────────────►  Caching likely won't help.
       │ YES
       ▼
  Can you tolerate stale data for some              NO
  period of time?  ──────────────────────────────────►  Caching is risky; requires
       │ YES                                            careful invalidation strategy.
       ▼
  Is the computation or I/O to produce              NO
  the data expensive?  ──────────────────────────────►  Caching adds complexity for
       │ YES                                            minimal gain.
       ▼
  Is the data accessed by many users /              NO
  with a shared access pattern?  ────────────────────►  Per-user caching may still
       │ YES                                            help but shared benefit is low.
       ▼
  ┌────────────────────┐
  │  CACHING IS LIKELY │
  │  A GOOD FIT ✅     │
  └────────────────────┘
```

### Good Candidates for Caching

| Scenario | Why Cache? | Example |
|---|---|---|
| **Product catalog** | Read 1000:1 vs writes, same data for all users | E-commerce product pages |
| **User profile / session** | Read on every request, changes rarely | Auth tokens, preferences |
| **Configuration / feature flags** | Read constantly, changes via deploy | `isFeatureEnabled("dark-mode")` |
| **Aggregated analytics** | Expensive computation, result stable for minutes | Dashboard roll-ups |
| **External API responses** | Slow third-party calls, data changes infrequently | Weather data, exchange rates |
| **Database query results** | Complex JOINs with stable data | Leaderboard, reporting queries |
| **Static assets** | Never change once deployed (versioned filenames) | CSS, JS bundles, images |
| **DNS lookups** | Very slow to resolve, records change infrequently | Internal service discovery |

### Poor Candidates for Caching

| Scenario | Why NOT Cache? | Better Approach |
|---|---|---|
| **Write-heavy data** | Cache invalidated faster than it is read | Optimize the write path |
| **Highly personalized data** | Each user sees unique content, low reuse | Compute on the fly |
| **Real-time financial data** | Staleness is unacceptable (trades, balances) | Direct database reads |
| **Large binary blobs** | Consumes excessive memory for low hit ratio | CDN or object storage |
| **Security-sensitive data** | Cached data may bypass access checks | Always check at origin |
| **One-time reads** | No temporal locality — data never re-read | Don't cache |

### Read-Heavy vs Write-Heavy Workloads

```
  Read-Heavy (cache shines)           Write-Heavy (cache struggles)
  ────────────────────────            ──────────────────────────────

  Read: ██████████████████  90%       Read: ████                20%
  Write: ██                 10%       Write: ████████████████    80%

  High cache hit ratio                Low hit ratio — constant invalidation
  Origin load reduced 10×             Cache adds overhead without benefit
  Example: product pages              Example: IoT telemetry ingestion
```

**Rule of thumb:** Caching provides the most benefit when the **read-to-write ratio** is
at least **10:1** or higher.

### Data Freshness Requirements

| Freshness | TTL Range | Examples |
|---|---|---|
| **Real-time** (no caching) | 0 s | Financial transactions, inventory counts |
| **Near-real-time** | 1–30 s | Social media feeds, notifications |
| **Short-lived** | 30 s – 5 min | Search results, API aggregations |
| **Medium-lived** | 5 min – 1 hr | Product catalog, user profiles |
| **Long-lived** | 1 hr – 24 hr | CMS content, feature flags |
| **Immutable** | Forever | Versioned static assets, build artifacts |

---

## The Caching Landscape

### Tool Overview

| Tool | Type | Language / Runtime | Key Strengths | License |
|---|---|---|---|---|
| **Redis** | Distributed KV store | C | Rich data structures, Lua scripting, pub/sub, streams | BSD-3 → SSPL (v7.4+) |
| **Valkey** | Distributed KV store | C | Redis fork, fully open-source, community-governed | BSD-3 |
| **Memcached** | Distributed KV store | C | Simple, fast, battle-tested, multi-threaded | BSD-3 |
| **Hazelcast** | In-memory data grid | Java | Distributed computing, SQL, near-cache, CP subsystem | Apache 2.0 |
| **Apache Ignite** | In-memory data grid | Java | Distributed SQL, compute grid, ACID transactions | Apache 2.0 |
| **Caffeine** | In-process cache | Java | W-TinyLFU, near-optimal hit rate, async loading | Apache 2.0 |
| **Ehcache** | In-process + tiered | Java | Heap + off-heap + disk tiers, JCache (JSR-107) | Apache 2.0 |
| **Varnish** | HTTP accelerator | C | VCL scripting, ESI, very high throughput | BSD-2 |
| **Nginx caching** | Reverse proxy cache | C | Built into Nginx, simple config, microcaching | BSD-2 |

### Choosing the Right Tool

| Need | Recommended Tool(s) | Reason |
|---|---|---|
| Simple key-value caching | **Memcached**, **Redis** | Low latency, easy to operate |
| Rich data structures | **Redis**, **Valkey** | Hashes, sorted sets, streams, HyperLogLog |
| Open-source with no license concerns | **Valkey**, **Memcached** | Permissive BSD/Apache licenses |
| In-process Java cache | **Caffeine** | Best-in-class hit ratio (W-TinyLFU) |
| Distributed compute + cache | **Hazelcast**, **Apache Ignite** | Collocated processing, distributed SQL |
| HTTP / static asset caching | **Varnish**, **Nginx** | Purpose-built HTTP accelerators |
| CDN / edge caching | **CloudFront**, **Cloudflare**, **Fastly** | Global PoP networks, managed |
| .NET in-process cache | **IMemoryCache** | Built-in, zero dependencies |
| Multi-tier (L1 + L2) | **Caffeine** (L1) + **Redis** (L2) | Combines latency of local with consistency of shared |

---

## Prerequisites

Before diving into the detailed caching guides in this series, you should be comfortable
with the following:

| Topic | Why It's Needed | Where to Learn |
|---|---|---|
| **HTTP basics** | Understanding cache headers (`Cache-Control`, `ETag`) | [Networking docs](../networking/05-HTTP-TLS-AND-LOAD-BALANCING.md) |
| **Client-server architecture** | Knowing where caches fit in request flows | [System Design docs](../system-design/00-OVERVIEW.md) |
| **Basic data structures** | Hash maps, linked lists (used in LRU caches) | Any algorithms/data structures course |
| **Databases (SQL or NoSQL)** | Understanding what the "origin" is | [Databases docs](../databases/00-OVERVIEW.md) |
| **Docker basics** | Running Redis, Memcached, Varnish locally | [Docker docs](../docker/00-OVERVIEW.md) |
| **At least one backend language** | Following code examples (Python, Java, Go, C#) | Language-specific docs in this repo |

---

## Next Steps

Explore the detailed guides in this caching series:

| File | Topic | Description |
|---|---|---|
| [01-REDIS.md](01-REDIS.md) | Redis | Data structures, commands, Lua scripting, pub/sub, persistence, clustering |
| [02-VALKEY.md](02-VALKEY.md) | Valkey | Redis fork, migration guide, community governance, compatibility |
| [03-MEMCACHED.md](03-MEMCACHED.md) | Memcached | Simple KV caching, slab allocator, multi-threading, consistent hashing |
| [04-CACHE-ASIDE.md](04-CACHE-ASIDE.md) | Cache-Aside Pattern | Lazy loading, read-through, write-through, write-behind strategies |
| [05-WRITE-STRATEGIES.md](05-WRITE-STRATEGIES.md) | Write Strategies | Write-through, write-back, write-around — trade-offs and implementation |
| [06-INVALIDATION.md](06-INVALIDATION.md) | Cache Invalidation | TTL, event-driven, versioned keys, purge APIs, the hardest problem |
| [07-DISTRIBUTED-CACHING.md](07-DISTRIBUTED-CACHING.md) | Distributed Caching | Consistent hashing, replication, partitioning, failover strategies |
| [08-CDN-AND-EDGE.md](08-CDN-AND-EDGE.md) | CDN and Edge Caching | CloudFront, Cloudflare, Varnish, cache headers, edge compute |
| [09-HTTP-CACHING.md](09-HTTP-CACHING.md) | HTTP Caching | Cache-Control, ETag, conditional requests, browser cache, proxy cache |
| [10-MONITORING.md](10-MONITORING.md) | Cache Monitoring | Hit ratio tracking, eviction metrics, memory usage, alerting |
| [11-BEST-PRACTICES.md](11-BEST-PRACTICES.md) | Best Practices | Key naming, serialization, TTL strategies, capacity planning |
| [12-ANTI-PATTERNS.md](12-ANTI-PATTERNS.md) | Anti-Patterns | Unbounded caches, cache-as-database, ignoring thundering herd |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Learning Path | Structured curriculum from beginner to advanced with hands-on labs |

### Suggested Learning Path by Role

**Backend Engineers:**
1. Start with this overview → [04-CACHE-ASIDE.md](04-CACHE-ASIDE.md) → [01-REDIS.md](01-REDIS.md)
2. Then [06-INVALIDATION.md](06-INVALIDATION.md) → [11-BEST-PRACTICES.md](11-BEST-PRACTICES.md) → [12-ANTI-PATTERNS.md](12-ANTI-PATTERNS.md)

**SREs / Platform Engineers:**
1. Start with this overview → [01-REDIS.md](01-REDIS.md) → [07-DISTRIBUTED-CACHING.md](07-DISTRIBUTED-CACHING.md)
2. Then [10-MONITORING.md](10-MONITORING.md) → [11-BEST-PRACTICES.md](11-BEST-PRACTICES.md)

**Architects:**
1. Start with this overview → [07-DISTRIBUTED-CACHING.md](07-DISTRIBUTED-CACHING.md) → [05-WRITE-STRATEGIES.md](05-WRITE-STRATEGIES.md)
2. Then [06-INVALIDATION.md](06-INVALIDATION.md) → [08-CDN-AND-EDGE.md](08-CDN-AND-EDGE.md) → [12-ANTI-PATTERNS.md](12-ANTI-PATTERNS.md)

**Frontend Engineers:**
1. Start with this overview → [09-HTTP-CACHING.md](09-HTTP-CACHING.md) → [08-CDN-AND-EDGE.md](08-CDN-AND-EDGE.md)

---

## Version History

| Date | Change | Author |
|---|---|---|
| 2025-04-15 | Initial version | Team |
