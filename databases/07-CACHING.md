# Caching

## Table of Contents

1. [Overview](#overview)
   - [What Is Caching?](#what-is-caching)
   - [Why Caching Matters for Databases](#why-caching-matters-for-databases)
   - [The Memory Hierarchy](#the-memory-hierarchy)
   - [Cache Hit, Cache Miss, and Hit Ratio](#cache-hit-cache-miss-and-hit-ratio)
2. [Cache Strategies](#cache-strategies)
   - [Cache-Aside (Lazy Loading)](#cache-aside-lazy-loading)
   - [Read-Through](#read-through)
   - [Write-Through](#write-through)
   - [Write-Behind (Write-Back)](#write-behind-write-back)
   - [Refresh-Ahead](#refresh-ahead)
   - [Strategy Comparison](#strategy-comparison)
3. [Cache Invalidation](#cache-invalidation)
   - [Why Invalidation Is Hard](#why-invalidation-is-hard)
   - [TTL-Based Invalidation](#ttl-based-invalidation)
   - [Event-Driven Invalidation](#event-driven-invalidation)
   - [Versioned Keys](#versioned-keys)
4. [Redis](#redis)
   - [Data Structures](#data-structures)
   - [Persistence: RDB and AOF](#persistence-rdb-and-aof)
   - [Redis Cluster](#redis-cluster)
   - [Redis Sentinel](#redis-sentinel)
   - [Pub/Sub](#pubsub)
   - [Lua Scripting](#lua-scripting)
   - [Common Redis Patterns](#common-redis-patterns)
5. [Memcached](#memcached)
   - [Architecture](#architecture)
   - [Consistent Hashing](#consistent-hashing)
   - [Slab Allocation](#slab-allocation)
   - [When to Choose Memcached over Redis](#when-to-choose-memcached-over-redis)
6. [Application-Level Caching](#application-level-caching)
   - [In-Process Caches](#in-process-caches)
   - [CDN Caching for API Responses](#cdn-caching-for-api-responses)
   - [HTTP Caching Headers](#http-caching-headers)
7. [Database-Level Caching](#database-level-caching)
   - [Buffer Pool (InnoDB)](#buffer-pool-innodb)
   - [PostgreSQL Shared Buffers](#postgresql-shared-buffers)
   - [Query Cache (MySQL — Deprecated)](#query-cache-mysql--deprecated)
   - [Materialized Views as Cache](#materialized-views-as-cache)
8. [Cache Patterns at Scale](#cache-patterns-at-scale)
   - [Thundering Herd / Cache Stampede](#thundering-herd--cache-stampede)
   - [Hot Key Mitigation](#hot-key-mitigation)
   - [Multi-Tier Caching](#multi-tier-caching)
9. [Version History](#version-history)

---

## Overview

### What Is Caching?

A **cache** is a high-speed data storage layer that stores a subset of data — typically transient — so that future requests for that data are served faster than fetching it from the primary data store. Caching exploits the principle of **locality of reference**: recently or frequently accessed data is likely to be accessed again.

In the context of databases, caching sits between the application and the database to absorb repeated reads, reduce query load, and lower response latency from milliseconds to microseconds.

```
  Without Cache                          With Cache

  ┌─────────┐    query     ┌────────┐   ┌─────────┐  hit   ┌───────┐
  │  App    │───────────▶│  DB    │   │  App    │──────▶│ Cache │
  │         │◀───────────│        │   │         │◀──────│       │
  └─────────┘   ~5-50ms   └────────┘   └─────────┘ ~1ms └───────┘
                                               │  miss         │
                                               ▼               │
                                         ┌────────┐            │
                                         │  DB    │   fill     │
                                         │        │───────────▶│
                                         └────────┘            │
```

### Why Caching Matters for Databases

| Problem | How Caching Helps |
|---|---|
| Database CPU saturated by repeated reads | Cache absorbs read traffic; DB handles only cache misses |
| High tail latency on complex queries | Pre-computed results served from memory in < 1 ms |
| Read-heavy workloads with skewed access | Hot data lives in cache; cold data stays in DB |
| Cost of scaling database vertically | A single Redis instance can serve 100k+ ops/sec at a fraction of the cost |
| Cross-region latency to a centralized DB | Edge caches or regional cache clusters reduce round-trip time |

### The Memory Hierarchy

Caching is effective because of the vast speed difference across storage tiers:

```
  Access Latency (approximate)
  ───────────────────────────────────────────────────────────────

  L1 Cache (CPU)         ~1 ns        ████
  L2 Cache (CPU)         ~4 ns        ████████
  L3 Cache (CPU)        ~10 ns        ████████████████
  RAM (main memory)     ~100 ns       ████████████████████████████████
  Redis (over network)  ~0.5 ms       ██████████████████████████████████████...
  SSD (random read)     ~0.1 ms       ██████████████████████████████████████...
  HDD (random read)     ~10 ms        ██████████████████████████████████████...
  DB query (indexed)    ~1-10 ms      ██████████████████████████████████████...
  DB query (complex)    ~50-500 ms    ██████████████████████████████████████...
```

> **Key insight:** Even a network-attached cache like Redis (~0.5 ms) is 10–100x faster than a typical indexed database query. An in-process cache (~100 ns) is 10,000x faster.

### Cache Hit, Cache Miss, and Hit Ratio

- **Cache hit** — The requested data is found in the cache. Served directly, no DB query needed.
- **Cache miss** — The requested data is not in the cache. Must fall through to the database.
- **Hit ratio** — `hits / (hits + misses)`. A ratio of 0.95 (95%) means only 5% of requests reach the DB.

```
  Hit Ratio Impact on Database Load
  ────────────────────────────────

  Hit Ratio    DB Queries (per 1000 requests)
  ──────────   ──────────────────────────────
  0%           1000  (no cache benefit)
  90%          100
  95%          50
  99%          10
  99.9%        1
```

A small improvement in hit ratio at high levels has an outsized impact. Going from 95% to 99% cuts DB load by 80%.

---

## Cache Strategies

### Cache-Aside (Lazy Loading)

The most common pattern. The application manages the cache directly — it checks the cache first, falls back to the database on a miss, and populates the cache with the result.

```
  Cache-Aside Pattern
  ───────────────────

  ┌─────────┐  1. GET key   ┌───────┐
  │  App    │──────────────▶│ Cache │
  │         │◀──────────────│       │
  │         │  2a. HIT:     └───────┘
  │         │     return
  │         │
  │         │  2b. MISS:
  │         │──────────────▶┌──────┐
  │         │  3. query DB  │  DB  │
  │         │◀──────────────│      │
  │         │  4. result    └──────┘
  │         │
  │         │  5. SET key ─▶┌───────┐
  │         │               │ Cache │
  └─────────┘               └───────┘
```

```python
def get_user(user_id):
    # 1. Check cache
    cached = redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)

    # 2. Cache miss — query database
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)

    # 3. Populate cache with TTL
    redis.setex(f"user:{user_id}", 300, json.dumps(user))
    return user
```

**Pros:** Only requested data is cached (no wasted memory); cache failures are non-fatal.
**Cons:** Cache miss incurs triple penalty (cache check + DB query + cache write); data can become stale.

### Read-Through

Similar to cache-aside, but the cache itself is responsible for loading data from the database on a miss. The application only talks to the cache.

```
  Read-Through Pattern
  ────────────────────

  ┌─────────┐  1. GET key   ┌───────────────────────┐
  │  App    │──────────────▶│        Cache          │
  │         │               │                       │
  │         │               │  2. MISS? Load from:  │
  │         │               │  ┌──────┐             │
  │         │               │  │  DB  │             │
  │         │               │  └──────┘             │
  │         │◀──────────────│  3. Return + store    │
  └─────────┘               └───────────────────────┘
```

**Pros:** Simpler application code; loading logic centralized in cache layer.
**Cons:** Requires a cache library or provider that supports read-through (e.g., NCache, Hazelcast); first request for each key is always slow.

### Write-Through

Every write goes to the cache **and** the database synchronously. The cache is always consistent with the database.

```
  Write-Through Pattern
  ─────────────────────

  ┌─────────┐  1. WRITE    ┌───────┐  2. WRITE    ┌──────┐
  │  App    │─────────────▶│ Cache │─────────────▶│  DB  │
  │         │              │       │              │      │
  │         │◀─────────────│       │◀─────────────│      │
  │         │  4. ACK      │       │  3. ACK      │      │
  └─────────┘              └───────┘              └──────┘
```

**Pros:** Cache is never stale (for writes through this path); reads are always fast after a write.
**Cons:** Write latency increases (two writes on the critical path); data that is written but never read wastes cache space.

### Write-Behind (Write-Back)

The application writes to the cache, which **asynchronously** flushes writes to the database in the background. This decouples write latency from database performance.

```
  Write-Behind Pattern
  ────────────────────

  ┌─────────┐  1. WRITE    ┌───────┐
  │  App    │─────────────▶│ Cache │
  │         │◀─────────────│       │──┐
  │         │  2. ACK      │       │  │  3. Async batch
  └─────────┘              └───────┘  │     write
                                      ▼
                                ┌──────┐
                                │  DB  │
                                │      │
                                └──────┘
```

**Pros:** Lowest write latency; writes can be batched and coalesced (e.g., 10 updates to same key → 1 DB write).
**Cons:** Risk of data loss if cache crashes before flushing; complex error handling; eventual consistency.

### Refresh-Ahead

The cache proactively refreshes entries **before** they expire, predicting which keys will be needed based on recent access patterns. This avoids the latency spike of a cache miss on expiry.

```
  Refresh-Ahead Pattern
  ─────────────────────

  ┌───────┐  TTL: 300s, Refresh at 240s
  │ Cache │
  │       │─── Key accessed at t=230s (within refresh window)
  │       │
  │       │──▶ Background thread:
  │       │    ┌──────┐
  │       │    │  DB  │  Fetch fresh value
  │       │◀───│      │  Update cache before expiry
  │       │    └──────┘
  └───────┘
```

**Pros:** Eliminates cache-miss latency for hot keys; seamless for the application.
**Cons:** Wasted refreshes if predictions are wrong; added complexity; requires the cache library to support it.

### Strategy Comparison

| Strategy | Read Latency | Write Latency | Consistency | Complexity | Best For |
|---|---|---|---|---|---|
| Cache-Aside | Miss: high; Hit: low | N/A (app writes to DB) | Eventual (TTL) | Low | General-purpose read caching |
| Read-Through | Miss: high; Hit: low | N/A | Eventual (TTL) | Medium | Simplifying application code |
| Write-Through | Always low | High (2 sync writes) | Strong | Medium | Read-heavy with strict consistency |
| Write-Behind | Always low | Very low | Eventual | High | Write-heavy workloads, batch writes |
| Refresh-Ahead | Always low | N/A | Near-real-time | High | Predictable hot-key access patterns |

---

## Cache Invalidation

### Why Invalidation Is Hard

> *"There are only two hard things in Computer Science: cache invalidation and naming things."*
> — Phil Karlton

The fundamental tension: **a cache is a copy of data**, and copies can diverge from the source of truth. Every invalidation approach trades off between staleness, complexity, and performance.

```
  The Staleness Window
  ────────────────────

  Time ──────────────────────────────────────────────▶

  DB:    value=A ──────── UPDATE to B ──────────────────
  Cache: value=A ──────── value=A (stale!) ── value=B ──
                          ▲                   ▲
                          │                   │
                     DB updated          Cache invalidated
                                         (staleness window)
```

### TTL-Based Invalidation

The simplest approach. Every cached entry has a **time-to-live** (TTL). After the TTL expires, the entry is evicted and the next request triggers a fresh load from the database.

```python
# Set key with a 5-minute TTL
redis.setex("product:42", 300, serialized_product)

# For data that rarely changes, use longer TTLs
redis.setex("config:feature_flags", 3600, serialized_flags)  # 1 hour

# For rapidly changing data, use short TTLs
redis.setex("stock:AAPL:price", 5, current_price)  # 5 seconds
```

| TTL Length | Staleness Risk | Cache Efficiency | Good For |
|---|---|---|---|
| Short (1–30s) | Low | Low hit ratio | Stock prices, live scores |
| Medium (1–15 min) | Moderate | Good hit ratio | User profiles, product details |
| Long (1–24 hr) | High | High hit ratio | Config, feature flags, reference data |

**Pitfall:** If all keys share the same TTL and were populated at the same time, they expire simultaneously — causing a **cache stampede** (see [Thundering Herd](#thundering-herd--cache-stampede)). Add **jitter** to spread expirations:

```python
import random
base_ttl = 300
jitter = random.randint(-30, 30)
redis.setex(key, base_ttl + jitter, value)
```

### Event-Driven Invalidation

Instead of waiting for TTL expiry, **actively invalidate** the cache when the underlying data changes. This is done by publishing events from the write path.

```
  Event-Driven Invalidation
  ─────────────────────────

  ┌─────────┐  1. UPDATE   ┌──────┐
  │  App    │─────────────▶│  DB  │
  │         │              └──────┘
  │         │
  │         │  2. Publish event
  │         │─────────────▶┌──────────────┐
  │         │              │ Message Bus  │
  └─────────┘              │ (Kafka, SNS) │
                           └──────┬───────┘
                                  │  3. Invalidate
                                  ▼
                           ┌───────┐
                           │ Cache │  DEL "product:42"
                           └───────┘
```

Alternatively, use **database CDC** (Change Data Capture) with tools like Debezium to automatically capture row-level changes and emit invalidation events without modifying application code.

**Pros:** Near-real-time consistency; no wasted TTL staleness.
**Cons:** Added infrastructure complexity; message ordering and delivery guarantees must be handled; still pair with a TTL as a safety net.

### Versioned Keys

Embed a **version number** or **timestamp** in the cache key. When data changes, increment the version — the old cache entry is never explicitly deleted, it simply becomes unreferenced and eventually evicted by the cache's eviction policy.

```python
# Write path: increment version on update
new_version = db.execute(
    "UPDATE products SET name=%s, cache_version=cache_version+1 "
    "WHERE id=%s RETURNING cache_version", (name, product_id)
)

# Read path: include version in the cache key
cache_key = f"product:{product_id}:v{new_version}"
cached = redis.get(cache_key)
```

**Pros:** No explicit invalidation needed; no race conditions between delete and set.
**Cons:** Orphaned old-version keys consume memory until evicted; requires passing the version around.

---

## Redis

Redis (Remote Dictionary Server) is an open-source, in-memory data structure store used as a cache, message broker, and database. It is single-threaded for command execution (ensuring atomicity) and supports rich data structures beyond simple key-value pairs.

### Data Structures

| Structure | Commands | Use Case |
|---|---|---|
| **String** | `GET`, `SET`, `INCR`, `DECR`, `MGET` | Simple caching, counters, serialized objects |
| **Hash** | `HGET`, `HSET`, `HGETALL`, `HINCRBY` | Object fields (user profile with individual field access) |
| **List** | `LPUSH`, `RPUSH`, `LPOP`, `LRANGE` | Queues, activity feeds, recent items |
| **Set** | `SADD`, `SMEMBERS`, `SINTER`, `SUNION` | Tags, unique visitors, set operations |
| **Sorted Set** | `ZADD`, `ZRANGE`, `ZRANGEBYSCORE`, `ZRANK` | Leaderboards, rate limiting, priority queues |
| **Stream** | `XADD`, `XREAD`, `XREADGROUP` | Event sourcing, message queues with consumer groups |
| **HyperLogLog** | `PFADD`, `PFCOUNT`, `PFMERGE` | Cardinality estimation (unique counts) with ~0.81% error |
| **Bitmap** | `SETBIT`, `GETBIT`, `BITCOUNT` | Feature flags, daily active users, bloom filters |

```redis
-- Hash: store a user object with individual field access
HSET user:1001 name "Alice" email "alice@example.com" plan "premium"
HGET user:1001 plan          -- Returns "premium"
HINCRBY user:1001 login_count 1

-- Sorted Set: leaderboard
ZADD leaderboard 1500 "player:alice"
ZADD leaderboard 2300 "player:bob"
ZADD leaderboard 1800 "player:carol"
ZREVRANGE leaderboard 0 2 WITHSCORES  -- Top 3 players
```

### Persistence: RDB and AOF

Redis offers two persistence mechanisms that can be used independently or together:

| Feature | RDB (Snapshotting) | AOF (Append-Only File) |
|---|---|---|
| **How it works** | Point-in-time snapshot of the entire dataset | Logs every write operation |
| **Durability** | Data loss between snapshots (minutes) | Configurable: every second or every write |
| **Recovery speed** | Fast (load binary dump) | Slower (replay all operations) |
| **File size** | Compact | Larger, but AOF rewrite compacts it |
| **Performance impact** | Fork for background save (memory spike) | Continuous writes (fsync cost) |

```
# redis.conf — recommended production settings

# RDB: snapshot every 60 seconds if at least 1000 keys changed
save 60 1000

# AOF: enable with fsync every second (good durability/perf balance)
appendonly yes
appendfsync everysec

# AOF rewrite when file doubles in size
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

> **Production recommendation:** Use both RDB + AOF. RDB provides fast restarts; AOF provides durability. On restart, Redis loads the AOF (more complete) if available.

### Redis Cluster

Redis Cluster provides automatic data sharding across multiple Redis nodes with built-in replication for high availability.

```
  Redis Cluster (6 nodes: 3 masters + 3 replicas)
  ────────────────────────────────────────────────

  Hash Slots: 0 ─────────────────────────── 16383

  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
  │  Master A      │ │  Master B      │ │  Master C      │
  │  Slots 0-5460  │ │ Slots 5461-10922│ │Slots 10923-16383│
  │                │ │                │ │                │
  │  ┌──────────┐  │ │  ┌──────────┐  │ │  ┌──────────┐  │
  │  │Replica A1│  │ │  │Replica B1│  │ │  │Replica C1│  │
  │  └──────────┘  │ │  └──────────┘  │ │  └──────────┘  │
  └────────────────┘ └────────────────┘ └────────────────┘
```

**Key concepts:**
- **16,384 hash slots** — Each key is mapped to a slot via `CRC16(key) mod 16384`. Slots are distributed across masters.
- **Automatic failover** — If a master fails, its replica is promoted automatically.
- **Multi-key operations** — Only supported when all keys map to the same slot. Use **hash tags** `{user:1001}.profile` and `{user:1001}.sessions` to colocate keys.
- **Client-side routing** — Smart clients cache the slot-to-node mapping and route commands directly.

### Redis Sentinel

Redis Sentinel provides high availability for non-clustered Redis deployments via monitoring, automatic failover, and service discovery.

```
  Redis Sentinel Architecture
  ───────────────────────────

  ┌───────────┐  ┌───────────┐  ┌───────────┐
  │ Sentinel 1│  │ Sentinel 2│  │ Sentinel 3│   (quorum = 2)
  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
        │               │               │
        │    monitor     │    monitor     │    monitor
        ▼               ▼               ▼
  ┌───────────┐        ┌───────────┐
  │  Master   │───────▶│  Replica  │
  │           │  repl  │           │
  └───────────┘        └───────────┘
```

Use Sentinel when you need HA with a single master (no sharding). Use Cluster when you need both HA and horizontal scaling.

### Pub/Sub

Redis Pub/Sub enables fire-and-forget messaging between publishers and subscribers. Messages are not persisted — if no subscriber is listening, the message is lost.

```redis
-- Publisher
PUBLISH order:events '{"order_id": 42, "status": "shipped"}'

-- Subscriber
SUBSCRIBE order:events
```

For durable messaging with consumer groups and acknowledgments, use **Redis Streams** instead:

```redis
-- Producer: add event to a stream
XADD order:stream * order_id 42 status shipped

-- Consumer group: create group and read
XGROUP CREATE order:stream processors $ MKSTREAM
XREADGROUP GROUP processors worker1 COUNT 10 BLOCK 5000 STREAMS order:stream >

-- Acknowledge processing
XACK order:stream processors 1234567890-0
```

### Lua Scripting

Redis executes Lua scripts atomically — no other command runs between script instructions. This is essential for multi-step operations that must be atomic without using transactions.

```lua
-- Rate limiter: allow max 100 requests per 60-second window
-- KEYS[1] = rate limit key, ARGV[1] = max requests, ARGV[2] = window seconds
local current = redis.call('INCR', KEYS[1])
if current == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[2])
end
if current > tonumber(ARGV[1]) then
    return 0  -- rate limited
end
return 1  -- allowed
```

```python
# Execute Lua script from Python
rate_limit_script = redis.register_script(lua_source)
allowed = rate_limit_script(keys=["ratelimit:user:42"], args=[100, 60])
```

### Common Redis Patterns

**Rate Limiting (Sliding Window)**

```python
import time

def is_rate_limited(redis, user_id, max_requests=100, window_sec=60):
    key = f"ratelimit:{user_id}"
    now = time.time()
    pipe = redis.pipeline()
    pipe.zremrangebyscore(key, 0, now - window_sec)  # Remove old entries
    pipe.zadd(key, {str(now): now})                   # Add current request
    pipe.zcard(key)                                    # Count requests in window
    pipe.expire(key, window_sec)                       # Set TTL for cleanup
    _, _, count, _ = pipe.execute()
    return count > max_requests
```

**Session Storage**

```python
def create_session(redis, session_id, user_data, ttl=3600):
    redis.hset(f"session:{session_id}", mapping=user_data)
    redis.expire(f"session:{session_id}", ttl)

def get_session(redis, session_id):
    session = redis.hgetall(f"session:{session_id}")
    if session:
        redis.expire(f"session:{session_id}", 3600)  # Refresh TTL on access
    return session
```

**Leaderboard**

```python
def update_score(redis, player_id, score):
    redis.zadd("leaderboard", {player_id: score})

def get_top_players(redis, count=10):
    return redis.zrevrange("leaderboard", 0, count - 1, withscores=True)

def get_player_rank(redis, player_id):
    rank = redis.zrevrank("leaderboard", player_id)
    return rank + 1 if rank is not None else None  # 1-indexed
```

---

## Memcached

### Architecture

Memcached is a high-performance, distributed memory caching system. Unlike Redis, it is **multi-threaded** and focuses exclusively on simple key-value caching with no persistence, no data structures, and no built-in replication.

```
  Memcached Architecture
  ──────────────────────

  ┌─────────┐
  │  App    │
  │         │
  │ Client  │── Consistent hashing determines which server to use
  │ Library │
  └────┬────┘
       │
       ├──────────────────┬──────────────────┐
       ▼                  ▼                  ▼
  ┌──────────┐      ┌──────────┐      ┌──────────┐
  │ Memcached│      │ Memcached│      │ Memcached│
  │ Server 1 │      │ Server 2 │      │ Server 3 │
  │          │      │          │      │          │
  │ Slab     │      │ Slab     │      │ Slab     │
  │ Allocator│      │ Allocator│      │ Allocator│
  └──────────┘      └──────────┘      └──────────┘

  Servers are independent — no inter-node communication.
  All distribution logic lives in the client.
```

### Consistent Hashing

Memcached clients use **consistent hashing** to distribute keys across servers. When a server is added or removed, only `K/N` keys need to be remapped (where K = total keys, N = number of servers), rather than rehashing everything.

```
  Consistent Hash Ring
  ────────────────────

              Server A
                 │
         ┌───────┴───────┐
        ╱                 ╲
       ╱    key1 ●         ╲
      │                     │
  Server D       ●key2     Server B
      │                     │
       ╲    ● key3         ╱
        ╲                 ╱
         └───────┬───────┘
                 │
              Server C

  Each key maps to the next server clockwise on the ring.
  Adding/removing a server only affects keys in adjacent range.
```

### Slab Allocation

Memcached pre-allocates memory into **slabs** (fixed-size chunks) to avoid memory fragmentation. Each slab class stores items of a specific size range.

```
  Slab Classes (example)
  ──────────────────────

  Class 1:  96 bytes   ── small strings, counters
  Class 2:  120 bytes  ── short JSON objects
  Class 3:  152 bytes  ── medium objects
  ...
  Class 42: 1 MB       ── large objects (max item size)

  Each class has its own LRU eviction queue.
```

**Pitfall:** If most of your items are ~200 bytes but a few are 100 KB, the large-item slab classes consume memory that small items cannot use. This is called **slab calcification**. Memcached 1.4.11+ supports `slab_reassign` and `slab_automove` to mitigate this.

### When to Choose Memcached over Redis

| Factor | Choose Memcached | Choose Redis |
|---|---|---|
| **Data model** | Simple key-value only | Rich data structures needed |
| **Multi-threading** | Better multi-core utilization out of the box | Single-threaded command execution (I/O threads in 6.0+) |
| **Memory efficiency** | Slightly lower overhead for simple strings | Hash/set/list overhead per entry |
| **Persistence** | Not needed (pure cache) | Need durability or backup |
| **High availability** | External tooling (mcrouter) | Built-in Sentinel or Cluster |
| **Use case** | Simple HTML fragment / query result caching | Session store, rate limiting, pub/sub, queues |
| **Scale pattern** | Horizontal via consistent hashing (client-side) | Cluster with automatic sharding |

> **In practice:** Redis has largely replaced Memcached for most use cases. Memcached still shines for very large, simple caching tiers (e.g., Facebook uses Memcached at massive scale with mcrouter for routing and replication).

---

## Application-Level Caching

### In-Process Caches

In-process caches store data in the application's own memory, eliminating network round-trips entirely. They are the fastest caching layer but are limited to a single process and cannot be shared across instances.

| Language | Library | Eviction Policies | Key Features |
|---|---|---|---|
| **Java** | Caffeine | Size, time, reference | Near-optimal hit ratio (W-TinyLFU), async refresh |
| **Python** | `functools.lru_cache` | LRU (size-based) | Built-in decorator, zero dependencies |
| **Python** | `cachetools` | LRU, LFU, TTL | TTL support, multiple policies |
| **.NET** | `MemoryCache` | Size, time, priority | Built-in to .NET, sliding/absolute expiration |
| **Go** | `groupcache` / `ristretto` | LRU / TinyLFU | Distributed coordination (groupcache), high concurrency |

```java
// Java — Caffeine (near-optimal hit ratio)
LoadingCache<String, User> userCache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofMinutes(5))
    .refreshAfterWrite(Duration.ofMinutes(4))  // Refresh-ahead
    .build(userId -> userRepository.findById(userId));

User user = userCache.get("user:1001");  // Automatic load on miss
```

```python
# Python — functools.lru_cache
from functools import lru_cache

@lru_cache(maxsize=1024)
def get_config(key: str) -> str:
    return db.execute("SELECT value FROM config WHERE key = %s", key)
```

```csharp
// .NET — MemoryCache
using Microsoft.Extensions.Caching.Memory;

var cache = new MemoryCache(new MemoryCacheOptions {
    SizeLimit = 10000
});

var user = await cache.GetOrCreateAsync($"user:{userId}", async entry => {
    entry.SlidingExpiration = TimeSpan.FromMinutes(5);
    entry.Size = 1;
    return await _dbContext.Users.FindAsync(userId);
});
```

### CDN Caching for API Responses

CDN edge nodes can cache API responses close to users, reducing origin load and latency. This works best for responses that are identical across many users (e.g., product catalog, public content).

```
  CDN Caching for APIs
  ────────────────────

  User (Tokyo)               Edge Node (Tokyo)            Origin (US-East)
  ─────────────              ─────────────────            ────────────────
  GET /api/products  ──────▶  Cache HIT? ─── Yes ──▶ Return cached response
                              │
                              No (MISS)
                              │
                              ├─────────────────────────▶ GET /api/products
                              │                           │
                              │◀──────────────────────────┤ 200 OK
                              │  Cache-Control: public,   │
                              │  max-age=60               │
                              │                           │
                              Store in edge cache
                              Return to user
```

### HTTP Caching Headers

| Header | Purpose | Example |
|---|---|---|
| `Cache-Control` | Primary caching directive | `public, max-age=3600` |
| `ETag` | Content hash for conditional requests | `"abc123def456"` |
| `Last-Modified` | Timestamp for conditional requests | `Wed, 01 Jan 2025 00:00:00 GMT` |
| `Vary` | Cache varies by request header | `Vary: Accept-Encoding, Authorization` |

```
  Cache-Control values:
  ─────────────────────

  public             Any cache (CDN, browser) can store the response
  private            Only the end-user's browser can cache it
  no-cache           Must revalidate with origin before using cached copy
  no-store           Do not cache at all (sensitive data)
  max-age=N          Cache is fresh for N seconds
  s-maxage=N         Override max-age for shared caches (CDN)
  stale-while-revalidate=N   Serve stale while refreshing in background
```

```
# Example response headers for a product API
Cache-Control: public, max-age=60, stale-while-revalidate=30
ETag: "v2-product-42-hash-abc123"
Vary: Accept-Encoding
```

> **`stale-while-revalidate`** is powerful: it serves the cached (slightly stale) response immediately while fetching a fresh copy in the background. The user experiences zero latency, and the next request gets fresh data.

---

## Database-Level Caching

### Buffer Pool (InnoDB)

The InnoDB buffer pool is MySQL's primary caching mechanism — it caches table data and index pages in memory, reducing disk I/O.

```
# my.cnf — InnoDB buffer pool configuration

# Set to ~70-80% of available RAM on a dedicated DB server
innodb_buffer_pool_size = 12G

# Multiple instances reduce contention on multi-core systems
innodb_buffer_pool_instances = 8

# Dump/load buffer pool state on restart for warm-up
innodb_buffer_pool_dump_at_shutdown = ON
innodb_buffer_pool_load_at_startup = ON
```

Monitor buffer pool efficiency:

```sql
-- InnoDB buffer pool hit ratio (should be > 99%)
SELECT
    (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100
    AS buffer_pool_hit_ratio
FROM (
    SELECT
        VARIABLE_VALUE AS Innodb_buffer_pool_reads
    FROM performance_schema.global_status
    WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads'
) reads,
(
    SELECT
        VARIABLE_VALUE AS Innodb_buffer_pool_read_requests
    FROM performance_schema.global_status
    WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'
) requests;
```

### PostgreSQL Shared Buffers

PostgreSQL uses `shared_buffers` as its internal cache for table and index pages. It also relies on the OS page cache for additional caching.

```
# postgresql.conf

# Set to ~25% of available RAM (PostgreSQL relies on OS cache for the rest)
shared_buffers = 4GB

# Effective cache size hint for the query planner (RAM - shared_buffers)
effective_cache_size = 12GB

# Working memory for sorts and hash joins (per-operation, not global)
work_mem = 256MB
```

```sql
-- Check shared buffer hit ratio (should be > 99%)
SELECT
    sum(heap_blks_hit) AS hits,
    sum(heap_blks_read) AS misses,
    round(
        sum(heap_blks_hit)::numeric /
        nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100, 2
    ) AS hit_ratio
FROM pg_statio_user_tables;
```

> **Key difference:** InnoDB recommends 70–80% of RAM for buffer pool because it manages its own I/O. PostgreSQL recommends only 25% because it intentionally leverages the operating system's page cache for the remainder.

### Query Cache (MySQL — Deprecated)

MySQL's query cache stored the full result set of SELECT statements keyed by the exact query text. It was **removed in MySQL 8.0** due to severe scalability issues.

**Why it was removed:**
- Global mutex contention — every read and write acquired a single lock.
- Any write to a table invalidated **all** cached queries for that table.
- Exact text matching — `SELECT * FROM users` and `select * FROM users` were separate cache entries.
- At high concurrency, the query cache became a bottleneck rather than an optimization.

**What to use instead:** Application-level caching (Redis, Memcached) or ProxySQL's query caching feature, which avoids these problems by being external to the database engine.

### Materialized Views as Cache

A **materialized view** is a database object that stores the result of a query physically. It acts as a cache for expensive aggregations or joins — pre-computed and stored on disk.

```sql
-- PostgreSQL: create a materialized view for a dashboard query
CREATE MATERIALIZED VIEW monthly_sales_summary AS
SELECT
    date_trunc('month', order_date) AS month,
    product_category,
    count(*) AS order_count,
    sum(total_amount) AS revenue
FROM orders
JOIN products ON orders.product_id = products.id
WHERE order_date >= now() - interval '2 years'
GROUP BY 1, 2;

-- Create an index on the materialized view for fast lookups
CREATE INDEX idx_sales_summary_month ON monthly_sales_summary (month);

-- Refresh periodically (this locks the view; use CONCURRENTLY to avoid blocking)
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_sales_summary;
```

| Aspect | Materialized View | External Cache (Redis) |
|---|---|---|
| **Location** | Inside the database | External service |
| **Query support** | Full SQL (joins, filters, aggregations) | Key-value lookup only |
| **Refresh** | `REFRESH MATERIALIZED VIEW` (manual/scheduled) | TTL or event-driven |
| **Best for** | Complex aggregations, dashboard queries | Simple lookups, session data |
| **Concurrency** | `CONCURRENTLY` avoids read locks | Always non-blocking |

---

## Cache Patterns at Scale

### Thundering Herd / Cache Stampede

When a popular cache key expires, many concurrent requests simultaneously miss the cache and hit the database with the same expensive query. This can overload the database.

```
  Cache Stampede
  ──────────────

  Time ──────────────────────────────────────────────▶

                  Key "popular" expires
                          │
  Request 1 ──── MISS ────┼──── DB query ────┐
  Request 2 ──── MISS ────┼──── DB query ────┤
  Request 3 ──── MISS ────┼──── DB query ────┤  DB overloaded!
  Request 4 ──── MISS ────┼──── DB query ────┤
  Request 5 ──── MISS ────┼──── DB query ────┘
```

**Solution 1: Lock / Lease Pattern**

Only one request is allowed to recompute the value. Others wait for the result or serve a stale value.

```python
def get_with_lock(redis, key, ttl, compute_fn):
    value = redis.get(key)
    if value is not None:
        return json.loads(value)

    lock_key = f"lock:{key}"
    # Try to acquire a lock (NX = only if not exists, EX = auto-expire)
    if redis.set(lock_key, "1", nx=True, ex=5):
        try:
            # Won the lock — recompute
            value = compute_fn()
            redis.setex(key, ttl, json.dumps(value))
            return value
        finally:
            redis.delete(lock_key)
    else:
        # Another request is recomputing — wait briefly and retry
        time.sleep(0.05)
        return get_with_lock(redis, key, ttl, compute_fn)
```

**Solution 2: Probabilistic Early Recomputation (XFetch)**

Instead of waiting for the key to expire, each request has a small probability of triggering a recompute before the TTL elapses. The probability increases as the TTL approaches expiry.

```python
import math, random, time

def xfetch(redis, key, ttl, beta, compute_fn):
    """
    Probabilistic early recomputation.
    beta controls eagerness: 1.0 = standard, >1 = more eager.
    """
    value, expiry = redis.get_with_ttl(key)  # Hypothetical: get value + remaining TTL
    if value is not None:
        remaining = expiry - time.time()
        # Probability of recompute increases as TTL approaches 0
        if remaining > 0 and -beta * math.log(random.random()) < remaining:
            return json.loads(value)  # Use cached value

    # Recompute (either expired or probabilistically chosen)
    start = time.time()
    fresh_value = compute_fn()
    compute_time = time.time() - start

    redis.setex(key, ttl, json.dumps(fresh_value))
    return fresh_value
```

### Hot Key Mitigation

A **hot key** is a single cache key that receives a disproportionate volume of requests (e.g., a viral product page, a trending topic). Even Redis can struggle when millions of requests per second target one key.

```
  Hot Key Problem
  ───────────────

  1M req/s ─────────▶ ┌───────────┐
                      │  Redis    │
                      │  Master   │◀── All requests for "trending:post:42"
                      │  Node 3   │    land on the same shard
                      └───────────┘
```

**Mitigation strategies:**

| Strategy | How It Works | Trade-off |
|---|---|---|
| **Key replication** | Append a random suffix: `key:N` (N = 0..9). Read from random replica key. | Write fan-out; eventually consistent |
| **Local cache + short TTL** | Cache hot keys in application memory (L1) with a 1–5 second TTL | Staleness; memory per app instance |
| **Read replicas** | Route hot-key reads to Redis read replicas | Replication lag |
| **Client-side caching** | Redis 6.0+ server-assisted client caching with invalidation | Client memory; complexity |

```python
# Hot key replication: spread across 10 virtual keys
import random

REPLICAS = 10

def get_hot_key(redis, base_key):
    replica_key = f"{base_key}:{random.randint(0, REPLICAS - 1)}"
    return redis.get(replica_key)

def set_hot_key(redis, base_key, value, ttl):
    pipe = redis.pipeline()
    for i in range(REPLICAS):
        pipe.setex(f"{base_key}:{i}", ttl, value)
    pipe.execute()
```

### Multi-Tier Caching

Production systems often combine multiple caching layers, each optimized for different latency and capacity trade-offs.

```
  Multi-Tier Caching Architecture
  ────────────────────────────────

  ┌──────────────────────────────────────────────────────────────────┐
  │                         Request Path                             │
  │                                                                  │
  │  ┌──────────┐    ┌──────────────┐    ┌──────────┐    ┌────────┐ │
  │  │  CDN /   │───▶│  In-Process  │───▶│  Redis / │───▶│   DB   │ │
  │  │  Edge    │    │  Cache (L1)  │    │  Memcached│   │        │ │
  │  │  Cache   │    │              │    │  (L2)     │   │        │ │
  │  └──────────┘    └──────────────┘    └──────────┘    └────────┘ │
  │                                                                  │
  │  Latency:         ~1 ms              ~100 μs         ~0.5 ms    ~1-50 ms   │
  │  Capacity:        Terabytes          Megabytes       Gigabytes  Terabytes  │
  │  Scope:           Global CDN         Per-process     Shared     Source of  │
  │                                                      cluster    truth      │
  └──────────────────────────────────────────────────────────────────┘
```

**Design principles for multi-tier caching:**

1. **Shorter TTLs at higher tiers** — The CDN might cache for 60s, Redis for 300s, so stale data at the edge is limited.
2. **Invalidation cascades downward** — When invalidating, clear from L1 first, then L2. Or broadcast invalidation events that all tiers consume.
3. **Each tier has its own eviction policy** — L1 uses LRU with small size limits; L2 uses TTL + LRU with larger capacity.
4. **Monitor hit ratios at each tier** — A low L1 hit ratio with a high L2 hit ratio suggests L1 is too small. A low L2 hit ratio means data is not being cached effectively.

```python
class MultiTierCache:
    def __init__(self, local_cache, redis_client, default_ttl=300):
        self.local = local_cache    # L1: in-process (e.g., Caffeine, lru_cache)
        self.redis = redis_client   # L2: distributed (Redis)
        self.ttl = default_ttl

    def get(self, key, compute_fn):
        # L1: check in-process cache
        value = self.local.get(key)
        if value is not None:
            return value

        # L2: check Redis
        cached = self.redis.get(key)
        if cached is not None:
            value = json.loads(cached)
            self.local.set(key, value)  # Promote to L1
            return value

        # L3: compute from source (database)
        value = compute_fn()
        self.redis.setex(key, self.ttl, json.dumps(value))
        self.local.set(key, value)
        return value
```

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial caching documentation |
