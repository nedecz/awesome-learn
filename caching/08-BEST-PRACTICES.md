# Caching Best Practices

## Table of Contents

1. [Overview](#overview)
   - [Why Best Practices Matter](#why-best-practices-matter)
   - [Target Audience](#target-audience)
   - [Relationship to Other Caching Docs](#relationship-to-other-caching-docs)
2. [Cache Key Design](#cache-key-design)
   - [Naming Conventions](#naming-conventions)
   - [Namespacing](#namespacing)
   - [Key Length Limits](#key-length-limits)
   - [Hash-Based Keys](#hash-based-keys)
   - [Key Versioning](#key-versioning)
   - [Avoiding Collisions](#avoiding-collisions)
   - [Multi-Tenant Key Design](#multi-tenant-key-design)
3. [TTL Strategy](#ttl-strategy)
   - [Choosing TTL Values by Data Type](#choosing-ttl-values-by-data-type)
   - [TTL Jitter](#ttl-jitter)
   - [Adaptive TTL](#adaptive-ttl)
   - [Sliding TTL](#sliding-ttl)
   - [TTL for Different Cache Tiers](#ttl-for-different-cache-tiers)
   - [Common TTL Mistakes](#common-ttl-mistakes)
4. [Memory Management](#memory-management)
   - [Right-Sizing Cache Instances](#right-sizing-cache-instances)
   - [Memory Budgets per Data Type](#memory-budgets-per-data-type)
   - [Eviction Policy Selection](#eviction-policy-selection)
   - [Monitoring Memory Usage](#monitoring-memory-usage)
   - [OOM Prevention](#oom-prevention)
5. [Failure Handling and Resilience](#failure-handling-and-resilience)
   - [Cache-Down Scenarios](#cache-down-scenarios)
   - [Circuit Breakers](#circuit-breakers)
   - [Fallback to Database](#fallback-to-database)
   - [Graceful Degradation](#graceful-degradation)
   - [Connection Pooling](#connection-pooling)
   - [Retry and Timeout Strategies](#retry-and-timeout-strategies)
6. [Serialization and Data Format](#serialization-and-data-format)
   - [Choosing a Serialization Format](#choosing-a-serialization-format)
   - [Compression](#compression)
   - [Schema Versioning for Cached Data](#schema-versioning-for-cached-data)
   - [Avoiding Serialization Pitfalls](#avoiding-serialization-pitfalls)
7. [Security](#security)
   - [Encryption in Transit](#encryption-in-transit)
   - [Encryption at Rest](#encryption-at-rest)
   - [Access Control](#access-control)
   - [Network Segmentation](#network-segmentation)
   - [Sensitive Data and PII](#sensitive-data-and-pii)
8. [Monitoring and Observability](#monitoring-and-observability)
   - [Key Metrics](#key-metrics)
   - [Alerting Thresholds](#alerting-thresholds)
   - [Dashboards](#dashboards)
   - [Distributed Tracing Through Cache Layers](#distributed-tracing-through-cache-layers)
9. [Testing Cache Behavior](#testing-cache-behavior)
   - [Unit Testing Cache Logic](#unit-testing-cache-logic)
   - [Integration Testing with Real Cache](#integration-testing-with-real-cache)
   - [Chaos Testing](#chaos-testing)
   - [Performance and Load Testing](#performance-and-load-testing)
   - [Verifying Invalidation](#verifying-invalidation)
10. [Production Readiness Checklist](#production-readiness-checklist)
11. [Common Mistakes and How to Avoid Them](#common-mistakes-and-how-to-avoid-them)
12. [Version History](#version-history)

---

## Overview

Caching is deceptively simple to introduce and remarkably difficult to operate well. A single `cache.set()` call can cut response times by 100×, but without disciplined practices around key design, TTL management, memory limits, failure handling, and observability, that same cache becomes a source of stale data, outages, and security vulnerabilities.

This document distills the operational, design, and architectural practices that separate fragile caches from production-grade caching infrastructure.

### Why Best Practices Matter

Consider what happens when a cache is added without planning:

```
Day 1:    "We added Redis. Latency dropped from 200 ms to 3 ms!"
Week 2:   "Some users see stale profiles. Nobody knows why."
Month 1:  "Redis hit 100% memory. Evictions spiked. Latency is worse than no cache."
Month 3:  "Redis went down for 5 minutes. Every request hit the DB. The DB crashed."
```

Every one of these failures is preventable. Best practices exist to close the gap between "it works on my laptop" and "it survives Black Friday traffic at scale."

### Target Audience

This guide is for backend engineers, SREs, and architects who already understand caching fundamentals and want a concrete reference for building production systems. Familiarity with at least one cache technology (Redis, Valkey, Memcached) is assumed.

### Relationship to Other Caching Docs

This document is a cross-cutting companion to the other guides in this series:

| Document | Focus | This Doc Adds |
|----------|-------|---------------|
| [00-OVERVIEW.md](00-OVERVIEW.md) | Caching fundamentals | Operational discipline around the fundamentals |
| [01-REDIS.md](01-REDIS.md) | Redis deep dive | Redis-agnostic patterns that apply everywhere |
| [04-CACHE-STRATEGIES.md](04-CACHE-STRATEGIES.md) | Read/write strategies | How to select and tune strategies for production |
| [05-CACHE-INVALIDATION.md](05-CACHE-INVALIDATION.md) | Invalidation patterns | TTL hygiene, versioning, and testing invalidation |
| [06-DISTRIBUTED-CACHING.md](06-DISTRIBUTED-CACHING.md) | Distributed patterns | Failure handling and resilience at scale |
| [07-CDN-AND-EDGE-CACHING.md](07-CDN-AND-EDGE-CACHING.md) | CDN and edge caching | Security and monitoring across cache tiers |

---

## Cache Key Design

A poorly designed cache key is the root cause of more caching bugs than any other single factor. Keys that collide, keys that are too long, keys that lack structure — all of these lead to stale data, wasted memory, and debugging nightmares.

### Naming Conventions

Use a consistent, human-readable naming scheme across your entire organization. A good key tells you **what** is cached, **which record** it refers to, and **which version** of the schema produced it.

**Recommended format:**

```
{service}:{entity}:{identifier}:{version}
```

**Examples:**

```
user-svc:user:42:v2
product-svc:product:sku-8812:v1
order-svc:order-summary:user-42:v3
catalog:category-tree:electronics:v1
```

**Rules:**

- Use colons (`:`) as delimiters — they are conventional in Redis and visually scannable
- Use lowercase with hyphens for multi-word segments (`order-summary`, not `OrderSummary`)
- Place the most selective segment (the identifier) near the end for efficient prefix scanning
- Never embed user input directly into keys without sanitisation

### Namespacing

Namespacing prevents collisions between services, environments, and tenants. Without it, two services caching a `user:42` key will silently overwrite each other.

```
                    Namespace Hierarchy

    ┌──────────────────────────────────────────────────┐
    │  Environment    Service       Entity    ID       │
    │  ┌─────────┐  ┌──────────┐  ┌───────┐ ┌─────┐  │
    │  │  prod   │::│ user-svc │::│ user  │:│  42 │  │
    │  └─────────┘  └──────────┘  └───────┘ └─────┘  │
    │                                                  │
    │  staging::user-svc:user:42                       │
    │  prod::user-svc:user:42                          │
    │  prod::billing-svc:user:42   (different service) │
    └──────────────────────────────────────────────────┘
```

When multiple services share a Redis cluster, prefix keys with the service name. When multiple environments share infrastructure (not recommended, but common), add an environment prefix.

### Key Length Limits

Redis keys can be up to 512 MB, but that does not mean they should be. Long keys waste memory and network bandwidth on every operation.

| Key Length | Guidance |
|------------|----------|
| < 100 bytes | Ideal for most use cases |
| 100–250 bytes | Acceptable for complex composite keys |
| 250–512 bytes | Consider hashing — the key itself is consuming significant memory |
| > 512 bytes | Always hash — use SHA-256 or xxHash and store the original as a field |

**Memory impact:** In a cache with 10 million entries, reducing average key size from 200 bytes to 80 bytes saves roughly 1.1 GB of memory.

### Hash-Based Keys

When a key must include a complex query, long URL, or GraphQL operation, hash the variable part:

```python
import hashlib

def build_cache_key(service: str, entity: str, query_params: dict) -> str:
    """Build a cache key with a hashed query component."""
    canonical = "&".join(f"{k}={v}" for k, v in sorted(query_params.items()))
    query_hash = hashlib.sha256(canonical.encode()).hexdigest()[:16]
    return f"{service}:{entity}:{query_hash}"

# Result: "search-svc:results:a1b2c3d4e5f67890"
```

**Tradeoff:** Hashed keys are compact but opaque. Store a reverse mapping (hash → original query) in a debug namespace if you need to inspect keys during incidents.

### Key Versioning

When the structure of cached data changes — new fields added, types changed, serialization format updated — old cached values become incompatible. Key versioning solves this cleanly.

```python
CACHE_VERSION = "v3"

def user_cache_key(user_id: int) -> str:
    return f"user-svc:user:{user_id}:{CACHE_VERSION}"
```

When you deploy a new version, old keys are simply never read. They expire naturally via TTL. No manual flush is needed, and rollbacks are safe because the old version's keys are still intact.

### Avoiding Collisions

Collisions happen when two logically different values map to the same key. Common causes:

- **Missing entity type:** `svc:42` — is that a user, an order, or a product?
- **Missing tenant context:** `user:42` in a multi-tenant system — which tenant?
- **Truncated hash:** Using too few hash characters (8 hex chars = 4 billion combinations, which sounds safe but isn't under high-volume key generation)

**Rule of thumb:** If two different inputs could ever produce the same key, your key scheme has a collision vulnerability. Test key uniqueness as part of your integration tests.

### Multi-Tenant Key Design

In multi-tenant systems, tenant isolation in the cache is non-negotiable. A tenant seeing another tenant's data is a security incident.

**Option 1 — Tenant prefix in key:**

```
tenant-{tenant_id}:user-svc:user:{user_id}:v2
```

**Option 2 — Separate Redis database per tenant:**

```
Redis DB 0 → Tenant A
Redis DB 1 → Tenant B
```

**Option 3 — Separate Redis cluster per tenant:**

Best isolation, highest cost. Use for regulated industries or large enterprise tenants.

| Approach | Isolation | Operational Cost | Noisy Neighbour Risk |
|----------|-----------|------------------|----------------------|
| Key prefix | Low | Low | High — one tenant can evict another's data |
| Separate DB | Medium | Medium | Medium — shared memory pool |
| Separate cluster | High | High | None |

For most SaaS applications, **key prefix with per-tenant memory monitoring** is the pragmatic choice. Escalate to separate clusters for tenants with strict compliance requirements.

---

## TTL Strategy

Every cached value should have a TTL. An entry without a TTL is a memory leak waiting to happen. But choosing the **right** TTL is a balancing act between freshness and performance.

### Choosing TTL Values by Data Type

There is no universal TTL. The right value depends on how often the underlying data changes, how costly a miss is, and how tolerant your users are of staleness.

| Data Type | Typical TTL | Rationale |
|-----------|-------------|-----------|
| Static configuration | 1–24 hours | Changes rarely, safe to cache aggressively |
| User profile | 5–15 minutes | Changes occasionally, moderate staleness tolerance |
| Session data | Match session timeout | Must expire with the session |
| Product catalog | 5–60 minutes | Changes during business hours, batch updates |
| Search results | 1–5 minutes | High variability, users expect freshness |
| Rate limit counters | Exact window (e.g., 60 s) | Must be precise — over-caching breaks rate limiting |
| Feature flags | 30–60 seconds | Changes must propagate quickly |
| Auth tokens / permissions | 1–5 minutes | Security-sensitive — shorter is safer |

### TTL Jitter

When many keys share the same TTL, they expire simultaneously. This creates a **thundering herd** — thousands of cache misses hit the database at the same instant.

```
Without jitter:                     With jitter:

  ████████████████████ expire        ██  █ ███ █  ██ █ ███  expire
  ──────────────────────────►        ──────────────────────────►
  t=300s  ALL keys expire            t=270s...330s  keys expire gradually
          at the same moment
```

**Implementation:**

```python
import random

BASE_TTL = 300  # 5 minutes

def ttl_with_jitter(base: int = BASE_TTL, jitter_pct: float = 0.1) -> int:
    """Add ±jitter_pct random variation to base TTL."""
    jitter = int(base * jitter_pct)
    return base + random.randint(-jitter, jitter)

# Returns values between 270 and 330
```

**Rule of thumb:** Add 10–20% jitter to any TTL applied to batch-populated or high-cardinality key sets.

### Adaptive TTL

Some data changes frequently during business hours but is static overnight. Adaptive TTL adjusts based on observed change frequency:

```python
def adaptive_ttl(last_modified_age_seconds: int) -> int:
    """Longer TTL for data that hasn't changed recently."""
    if last_modified_age_seconds < 60:
        return 30       # Changed in the last minute — very short TTL
    elif last_modified_age_seconds < 3600:
        return 300      # Changed in the last hour — moderate TTL
    else:
        return 3600     # Stable data — cache aggressively
```

This technique is particularly effective for content management systems and catalog data where a small percentage of records change frequently while the majority are stable.

### Sliding TTL

A sliding (or touch-extended) TTL resets the expiration every time the key is read. This keeps frequently accessed data warm while letting cold data expire.

```python
def get_with_sliding_ttl(cache, key: str, ttl: int = 300):
    """Read a key and reset its TTL on each access."""
    value = cache.get(key)
    if value is not None:
        cache.expire(key, ttl)  # Reset TTL on hit
    return value
```

**When to use:** Session data, user preferences, shopping carts — data tied to active user engagement.

**When to avoid:** Data that must expire at a fixed time (rate limit windows, time-limited offers).

### TTL for Different Cache Tiers

In a multi-tier caching architecture, TTLs must decrease as you move closer to the user to ensure that upstream changes propagate:

```
    Tier             TTL           Reason

    ┌──────────┐
    │  L1 App  │    30–60 s      Closest to user, must refresh often
    │  Cache   │
    └────┬─────┘
         │
    ┌────▼─────┐
    │  L2 Redis│    5–15 min     Shared across instances, moderate TTL
    │  Cluster │
    └────┬─────┘
         │
    ┌────▼─────┐
    │  Origin  │    Source of     No TTL — this is the truth
    │  DB      │    truth
    └──────────┘
```

**Critical rule:** L1 TTL < L2 TTL. If your in-process cache holds data for 10 minutes but Redis holds it for 5 minutes, the app cache will serve stale data that Redis has already evicted and refreshed.

### Common TTL Mistakes

- **No TTL at all** — Keys accumulate forever, memory fills, evictions become unpredictable
- **Same TTL for everything** — A 5-minute TTL is wrong for both feature flags (too long) and product images (too short)
- **TTL too long for security data** — A 1-hour TTL on permissions means a revoked user retains access for up to an hour
- **TTL on rate limit counters with jitter** — Jitter breaks the precision of rate limiting windows
- **Forgetting TTL on error responses** — Caching a "404 Not Found" with a 1-hour TTL means a newly created resource is invisible for an hour

---

## Memory Management

Cache memory is a finite, expensive resource. Treating it as unlimited leads to evictions, OOM crashes, and unpredictable latency. Managing memory well means knowing your working set, budgeting by data type, and choosing the right eviction policy.

### Right-Sizing Cache Instances

Start with your working set — the subset of data that is actively accessed within a given time window.

**Estimation method:**

```
Working set size ≈ (number of active keys) × (average value size + average key size + overhead)
```

Redis adds roughly 50–80 bytes of overhead per key (for the hash table entry, expiry metadata, and encoding). For a cache with 5 million keys averaging 500 bytes each:

```
Memory ≈ 5,000,000 × (500 + 80 + 70) = ~3.25 GB
```

**Sizing rule:** Provision 1.5× your estimated working set to accommodate spikes, fragmentation, and replication buffers.

### Memory Budgets per Data Type

Not all cached data is equally valuable. Assign memory budgets to prevent low-value data from evicting high-value data:

| Data Type | Value per Hit | Typical Size | Budget Priority |
|-----------|---------------|--------------|-----------------|
| Auth / session tokens | Very high | Small (< 1 KB) | High — protect from eviction |
| API response fragments | High | Medium (1–50 KB) | Medium |
| Full page renders | Medium | Large (50–500 KB) | Low — expensive to store |
| Precomputed reports | Low (infrequent access) | Very large (> 1 MB) | Very low — consider a different tier |

**Implementation:** Use separate Redis databases or clusters for different priority tiers. If using a single instance, monitor per-prefix memory usage and set eviction policies accordingly.

### Eviction Policy Selection

When memory is full, the eviction policy determines what gets removed. The wrong policy can evict your most valuable data.

| Policy | How It Works | Best For | Avoid When |
|--------|-------------|----------|------------|
| **allkeys-lru** | Evicts least recently used key | General-purpose workloads | You have keys that must never be evicted |
| **volatile-lru** | LRU among keys with a TTL set | Mix of persistent and expiring data | Most keys lack a TTL |
| **allkeys-lfu** | Evicts least frequently used key | Workloads with stable hot/cold split | Access patterns shift rapidly |
| **volatile-lfu** | LFU among keys with a TTL | Frequency-based with TTL keys | Same as allkeys-lfu caveats |
| **allkeys-random** | Evicts a random key | Uniform access distribution | Any workload with a hot set |
| **noeviction** | Returns error when memory is full | Critical data that must never be lost | Any workload that grows unboundedly |

**Decision framework:**

```
Do you have keys that must never be evicted?
├─ Yes → Use volatile-lru or volatile-lfu (only evict TTL keys)
│        Set TTL on all evictable keys
└─ No  → Is there a clear hot/cold split?
         ├─ Yes → allkeys-lfu
         └─ No  → allkeys-lru (safe default)
```

### Monitoring Memory Usage

Track these memory metrics continuously:

- **used_memory** — Total memory consumed by Redis (data + overhead)
- **used_memory_rss** — Resident set size as reported by the OS (includes fragmentation)
- **mem_fragmentation_ratio** — RSS / used_memory. Values > 1.5 indicate significant fragmentation
- **evicted_keys** — Number of keys evicted due to maxmemory. Any non-zero value means your cache is undersized or has runaway growth
- **expired_keys** — Keys removed by TTL. Healthy caches expire far more keys than they evict

### OOM Prevention

An out-of-memory condition in your cache can cascade into a full system outage. Prevent it:

1. **Set `maxmemory` explicitly** — Never run Redis without a memory limit. The default (no limit) lets Redis consume all system RAM
2. **Set `maxmemory-policy`** — Without a policy, Redis returns errors when full (`noeviction` is the default)
3. **Alert at 70% memory usage** — This gives you time to respond before evictions begin
4. **Alert at 90% memory usage** — Page the on-call engineer; evictions are likely already happening
5. **Reject large values at the application layer** — If a value exceeds 1 MB, compress it or store it elsewhere. A handful of 10 MB values can push a cache into OOM

---

## Failure Handling and Resilience

A cache is an optimisation, not a source of truth. When the cache is unavailable, your system must continue to function — slower, but correctly. Designing for cache failure is not optional.

### Cache-Down Scenarios

Understand the failure modes and plan for each:

```
    ┌─────────────────────────────────────────────────────────┐
    │                  CACHE FAILURE MODES                     │
    │                                                         │
    │  Partial degradation          Full outage               │
    │  ┌─────────────────┐          ┌─────────────────┐       │
    │  │ Slow responses  │          │ Cache unreachable│       │
    │  │ High latency    │          │ All misses       │       │
    │  │ Elevated misses │          │ 100% DB load     │       │
    │  └─────────────────┘          └─────────────────┘       │
    │                                                         │
    │  Data corruption              Split brain               │
    │  ┌─────────────────┐          ┌─────────────────┐       │
    │  │ Stale values    │          │ Replicas diverge │       │
    │  │ Wrong types     │          │ Inconsistent     │       │
    │  │ Silent errors   │          │ reads            │       │
    │  └─────────────────┘          └─────────────────┘       │
    └─────────────────────────────────────────────────────────┘
```

### Circuit Breakers

A circuit breaker prevents your application from repeatedly attempting to reach a failed cache, which would add latency to every request without benefit.

```python
import time

class CacheCircuitBreaker:
    """Simple circuit breaker for cache operations."""

    def __init__(self, failure_threshold: int = 5, recovery_timeout: int = 30):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure_time = 0.0
        self.state = "closed"  # closed = healthy, open = failing

    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = "open"

    def record_success(self):
        self.failure_count = 0
        self.state = "closed"

    def allow_request(self) -> bool:
        if self.state == "closed":
            return True
        if time.time() - self.last_failure_time > self.recovery_timeout:
            self.state = "half-open"
            return True  # Allow one probe request
        return False
```

**Behavior:** After 5 consecutive failures, the circuit opens. All cache operations bypass the cache and go directly to the database. After 30 seconds, a single probe request tests whether the cache has recovered.

### Fallback to Database

When the circuit breaker is open, every request hits the origin database. Your database must be able to handle this load — at least temporarily.

**Preparation steps:**

- **Load test without cache** — Run your full traffic load directly against the database. If it falls over, you need connection limits, query optimisation, or a read replica strategy
- **Query-level timeouts** — Set aggressive timeouts (1–3 seconds) on database queries during cache-down events to prevent connection pool exhaustion
- **Request shedding** — If the database cannot handle full load, shed low-priority traffic (e.g., return cached-but-stale data, disable non-essential features)

### Graceful Degradation

When the cache is partially degraded, serve the best response you can rather than failing entirely:

| Scenario | Graceful Behaviour |
|----------|-------------------|
| Cache timeout (slow, not down) | Return stale data if available, mark as potentially stale |
| Cache miss during degradation | Serve from DB with reduced TTL to avoid overloading cache on recovery |
| Cache returns error | Log, increment circuit breaker counter, proceed without cache |
| Partial data in cache | Serve partial response, lazy-load the rest |

### Connection Pooling

Cache connection management is a common source of failures. Too few connections and requests queue; too many and the cache is overwhelmed.

**Guidelines:**

- **Pool size:** 10–30 connections per application instance is typical for Redis
- **Minimum idle connections:** Keep 2–5 warm connections to avoid cold-start latency
- **Connection timeout:** 1–2 seconds for initial connection; fail fast if the cache is unreachable
- **Idle timeout:** Close connections idle for more than 5 minutes to reclaim resources
- **Max lifetime:** Rotate connections every 30–60 minutes to handle DNS changes and load balancer re-balancing

### Retry and Timeout Strategies

**Timeouts:**

| Operation | Recommended Timeout | Why |
|-----------|-------------------|-----|
| Connect | 1–2 seconds | Fail fast if cache is down |
| Read (GET) | 100–500 ms | A slow cache is worse than no cache |
| Write (SET) | 200–500 ms | Slightly more tolerance for writes |
| Pipeline / batch | 500 ms–2 seconds | Proportional to batch size |

**Retries:**

- Retry **at most once** for transient failures (connection reset, timeout)
- Use **exponential backoff** only for background operations (cache warming, batch writes)
- **Never retry on the hot path** of a user request — the user is waiting. Fall back to the database instead

---

## Serialization and Data Format

How you serialize cached data affects memory usage, read/write performance, and your ability to evolve schemas over time.

### Choosing a Serialization Format

| Format | Size | Speed | Schema Evolution | Human Readable | Best For |
|--------|------|-------|------------------|----------------|----------|
| **JSON** | Large | Moderate | Poor — no versioning built in | Yes | Debugging, simple structures |
| **MessagePack** | Small | Fast | Poor — same as JSON | No | Drop-in JSON replacement needing less space |
| **Protocol Buffers** | Smallest | Fastest | Excellent — field numbering | No | High-throughput systems with evolving schemas |
| **Pickle (Python)** | Varies | Fast | Fragile — tied to class definitions | No | Never in production caches |

**Decision framework:**

- **Start with JSON** if your cache values are small (< 5 KB) and you value debuggability
- **Move to MessagePack** when JSON size or parse time becomes a bottleneck
- **Use Protocol Buffers** when you need strict schema evolution guarantees or cross-language support

### Compression

For values larger than 1 KB, compression can significantly reduce memory usage and network transfer time:

```python
import zlib
import json

def set_compressed(cache, key: str, value: dict, ttl: int = 300):
    """Serialize and compress before caching."""
    serialized = json.dumps(value, separators=(",", ":")).encode()
    if len(serialized) > 1024:  # Only compress values > 1 KB
        compressed = zlib.compress(serialized, level=6)
        cache.setex(f"z:{key}", ttl, compressed)
    else:
        cache.setex(key, ttl, serialized)

def get_compressed(cache, key: str):
    """Decompress and deserialize on read."""
    value = cache.get(f"z:{key}")
    if value is not None:
        return json.loads(zlib.decompress(value))
    value = cache.get(key)
    if value is not None:
        return json.loads(value)
    return None
```

**Compression ratios for typical cached data:**

| Data Type | Typical Compression Ratio | Worth Compressing? |
|-----------|---------------------------|-------------------|
| JSON API response | 3–5× | Yes — significant savings |
| HTML fragments | 4–8× | Yes |
| Already-binary data (images) | 1.0–1.1× | No — already compressed |
| Small values (< 500 bytes) | 0.8–1.2× | No — overhead exceeds savings |

### Schema Versioning for Cached Data

Cached data outlives deployments. When you change the structure of a cached object, old entries in the cache will fail to deserialize correctly.

**Strategy 1 — Version in key (recommended):**

Increment the key version when the schema changes. Old keys expire naturally.

```
user-svc:user:42:v2  →  user-svc:user:42:v3
```

**Strategy 2 — Version in value:**

Embed a schema version inside the cached payload and handle migration on read:

```python
def deserialize_user(raw: bytes) -> dict:
    data = json.loads(raw)
    version = data.get("_schema_version", 1)
    if version == 1:
        data["display_name"] = data.get("name", "")  # Migrate v1 → v2
        data["_schema_version"] = 2
    return data
```

**Prefer version-in-key** for simplicity. Use version-in-value only when you cannot tolerate the temporary miss rate from switching keys.

### Avoiding Serialization Pitfalls

- **Never use language-specific serializers** (Python pickle, Java Serializable) — they are fragile, insecure, and non-portable
- **Watch for datetime serialization** — `datetime` objects serialize differently across libraries. Standardise on ISO 8601 strings or Unix timestamps
- **Beware of floating-point precision** — `0.1 + 0.2 ≠ 0.3` in IEEE 754. Use strings or integers (cents, not dollars) for monetary values
- **Set a maximum value size** — Reject values larger than a defined limit (e.g., 512 KB) to prevent a single key from consuming excessive memory

---

## Security

A cache is a high-value target. It holds frequently accessed data, often including authentication tokens, user profiles, and API responses that may contain sensitive information. Securing the cache is as important as securing the database.

### Encryption in Transit

All communication between your application and the cache must be encrypted with TLS. Unencrypted traffic is vulnerable to eavesdropping and man-in-the-middle attacks on shared networks.

- **Redis 6+** supports native TLS. Enable it via `tls-port` and `tls-cert-file` configuration
- **Memcached** requires a TLS proxy (e.g., stunnel or mcrouter with TLS) or Memcached 1.6+ with `--enable-ssl`
- **Use mutual TLS (mTLS)** in zero-trust environments where both client and server authenticate each other

### Encryption at Rest

If your cache stores sensitive data and persists to disk (Redis RDB/AOF), enable encryption at rest:

- Use encrypted volumes (AWS EBS encryption, Azure Disk Encryption)
- For managed services (ElastiCache, Azure Cache for Redis), enable the encryption-at-rest option
- Remember that Redis snapshots (RDB files) contain all cached data in plaintext by default

### Access Control

- **Enable authentication** — Redis `requirepass` or ACLs (Redis 6+). Memcached SASL authentication
- **Use ACLs for least privilege** — Grant each service only the commands and key patterns it needs

```
# Redis ACL example: billing service can only access billing keys
user billing-svc on >strong-password ~billing:* +get +set +del +expire
```

- **Rotate credentials** on a regular schedule (90 days or less)
- **Never embed cache credentials in source code** — Use environment variables or a secrets manager

### Network Segmentation

- Place cache instances in a private subnet with no public IP
- Restrict security groups / firewall rules to allow traffic only from application instances
- Use VPC endpoints or private links for managed cache services
- Block all ports except the cache port (6379 for Redis, 11211 for Memcached)

### Sensitive Data and PII

**Default stance: do not cache sensitive data.** If you must, apply strict controls:

- **PII (names, emails, addresses)** — Cache only with encryption, short TTL (< 5 minutes), and audit logging
- **Authentication tokens** — Acceptable to cache, but must expire when revoked (use event-driven invalidation from [05-CACHE-INVALIDATION.md](05-CACHE-INVALIDATION.md))
- **Payment data (card numbers, bank accounts)** — Never cache. Violates PCI-DSS requirements
- **Health records** — Never cache outside a HIPAA-compliant, encrypted, access-controlled environment
- **Secrets (API keys, passwords)** — Never cache. Use a dedicated secrets manager

**Audit question:** "If someone ran `redis-cli KEYS *` and `GET` on every key, what would they see?" If the answer includes data you would not want in a log file, reconsider caching it.

---

## Monitoring and Observability

You cannot manage what you cannot measure. Caching failures are often silent — the system still works, just slower or with stale data. Monitoring makes these issues visible before they become outages.

### Key Metrics

| Metric | What It Tells You | Healthy Range |
|--------|-------------------|---------------|
| **Hit ratio** | Percentage of reads served from cache | > 90% for most workloads |
| **Miss ratio** | Percentage of reads requiring origin fetch | < 10% (inverse of hit ratio) |
| **Latency (p50 / p99)** | Cache operation speed | p50 < 1 ms, p99 < 5 ms |
| **Eviction rate** | Keys evicted due to memory pressure | 0 in steady state |
| **Memory usage** | Current vs. max memory | < 80% of maxmemory |
| **Connection count** | Active client connections | Below max connections limit |
| **Expired keys/sec** | Rate of TTL-based expiration | Proportional to write rate |
| **Command rate** | Operations per second (GET, SET, DEL) | Stable, proportional to traffic |

### Alerting Thresholds

| Condition | Severity | Action |
|-----------|----------|--------|
| Hit ratio drops below 80% | Warning | Investigate — TTL changes? New traffic pattern? |
| Hit ratio drops below 50% | Critical | Likely cache failure or misconfiguration |
| Memory usage > 70% | Warning | Plan capacity increase |
| Memory usage > 90% | Critical | Evictions imminent — scale up or shed load |
| Eviction rate > 0 (sustained) | Warning | Cache is undersized for the working set |
| p99 latency > 10 ms | Warning | Network issue, large values, or slow commands |
| p99 latency > 50 ms | Critical | Cache is effectively down — responses are slow enough to cascade |
| Connection count > 80% of max | Warning | Connection pool leak or scaling issue |

### Dashboards

A well-designed cache dashboard provides at-a-glance health assessment. Organise it in layers:

```
    ┌─────────────────────────────────────────────────────┐
    │  CACHE HEALTH DASHBOARD                             │
    │                                                     │
    │  ┌─────────────┐  ┌─────────────┐  ┌────────────┐  │
    │  │ Hit Ratio   │  │ Latency     │  │ Memory     │  │
    │  │   94.2%     │  │ p50: 0.3 ms │  │  72% used  │  │
    │  │   ▆▆▇▇▇▆▇  │  │ p99: 2.1 ms │  │  ████████░ │  │
    │  └─────────────┘  └─────────────┘  └────────────┘  │
    │                                                     │
    │  ┌─────────────┐  ┌─────────────┐  ┌────────────┐  │
    │  │ Evictions   │  │ Connections │  │ Ops/sec    │  │
    │  │    0 /sec   │  │   127/500   │  │  45,200    │  │
    │  │   ░░░░░░░░  │  │  ████░░░░░  │  │  ▅▆▇▆▅▆▇  │  │
    │  └─────────────┘  └─────────────┘  └────────────┘  │
    └─────────────────────────────────────────────────────┘
```

**Dashboard layers:**

1. **Traffic** — Ops/sec by command type, request rate, top key prefixes
2. **Performance** — Latency percentiles, slow log entries, big keys
3. **Capacity** — Memory usage, key count, eviction rate, fragmentation ratio
4. **Availability** — Connected clients, replication lag, cluster state

### Distributed Tracing Through Cache Layers

In a multi-tier cache architecture, tracing a request through each layer reveals where time is spent and where misses cascade:

```
    Request: GET /api/users/42

    ┌──────────────────────────────────────────────────────┐
    │ Trace ID: abc-123                                    │
    │                                                      │
    │  ├─ L1 Cache (in-process)     0.1 ms   MISS         │
    │  ├─ L2 Cache (Redis)          1.2 ms   MISS         │
    │  ├─ Database query            45.3 ms  HIT          │
    │  ├─ L2 Cache SET              0.8 ms                │
    │  └─ L1 Cache SET              0.05 ms               │
    │                                                      │
    │  Total: 47.45 ms  (cache-miss path)                  │
    └──────────────────────────────────────────────────────┘
```

**Implementation:** Add cache hit/miss as span tags in your tracing system (OpenTelemetry, Jaeger, Zipkin). This lets you filter traces by cache behaviour and identify patterns like "90% of slow requests are L2 cache misses."

---

## Testing Cache Behavior

Caching bugs are subtle. A stale value looks correct — it is just old. A missing invalidation is invisible until a user complains. Testing cache behavior requires deliberate effort beyond standard unit tests.

### Unit Testing Cache Logic

Test your caching layer in isolation using a mock or fake cache:

```python
import unittest
from unittest.mock import MagicMock

class TestUserCache(unittest.TestCase):

    def test_cache_hit_returns_cached_value(self):
        mock_cache = MagicMock()
        mock_cache.get.return_value = b'{"id": 42, "name": "Alice"}'
        mock_db = MagicMock()

        result = get_user(user_id=42, cache=mock_cache, db=mock_db)

        self.assertEqual(result["name"], "Alice")
        mock_db.query.assert_not_called()  # DB should not be hit

    def test_cache_miss_fetches_from_db_and_populates_cache(self):
        mock_cache = MagicMock()
        mock_cache.get.return_value = None
        mock_db = MagicMock()
        mock_db.query.return_value = {"id": 42, "name": "Alice"}

        result = get_user(user_id=42, cache=mock_cache, db=mock_db)

        self.assertEqual(result["name"], "Alice")
        mock_db.query.assert_called_once()
        mock_cache.setex.assert_called_once()  # Verify cache population

    def test_cache_failure_falls_back_to_db(self):
        mock_cache = MagicMock()
        mock_cache.get.side_effect = ConnectionError("Redis down")
        mock_db = MagicMock()
        mock_db.query.return_value = {"id": 42, "name": "Alice"}

        result = get_user(user_id=42, cache=mock_cache, db=mock_db)

        self.assertEqual(result["name"], "Alice")  # Should still work
```

### Integration Testing with Real Cache

Unit tests with mocks do not catch serialization bugs, TTL misconfiguration, or network issues. Run integration tests against a real cache instance:

- **Use Docker** to spin up Redis/Memcached in CI pipelines
- **Test TTL behaviour** by setting short TTLs (1–2 seconds) and asserting that keys expire
- **Test eviction** by filling a small-memory instance past `maxmemory` and verifying that eviction policies behave as expected
- **Test serialization round-trips** — write a complex object, read it back, assert equality

### Chaos Testing

Deliberately break the cache to verify that your failure handling works:

| Chaos Experiment | How to Run | What to Verify |
|------------------|------------|----------------|
| Kill cache process | `kill -9 redis-server` | App falls back to DB, circuit breaker opens |
| Network partition | `iptables` rule to block cache port | Timeouts trigger fallback, no hanging requests |
| Memory exhaustion | Fill cache to `maxmemory` | Eviction policy activates, no errors returned to users |
| Slow cache | Use `tc` to add 500 ms latency | Timeout fires, stale data served or DB fallback |
| Corrupt data | Write invalid bytes to a key | Deserialization error handled, key evicted, fresh data fetched |

### Performance and Load Testing

Measure cache performance under realistic conditions:

- **Benchmark with production-like data** — Key sizes, value sizes, and access patterns matching production
- **Test the miss path** — Flush the cache and measure cold-start performance
- **Test thundering herd** — Expire a hot key under load and verify that stampede prevention (locking, early refresh) works
- **Measure tail latency** — p99 and p99.9 latencies reveal queuing and garbage collection issues that p50 hides

### Verifying Invalidation

Invalidation bugs are the most common class of caching defects. Test them explicitly:

1. Write a value to the database
2. Verify the cache contains the old value
3. Trigger the invalidation mechanism (event, TTL, API call)
4. Verify the cache returns the new value (or a miss)
5. Verify that all cache tiers are invalidated, not just L1 or L2

---

## Production Readiness Checklist

Use this checklist before deploying any new caching layer or making significant changes to an existing one.

**Key Design:**

- [ ] Keys follow a consistent naming convention with namespacing
- [ ] Keys include a version segment for schema evolution
- [ ] Key length is under 250 bytes (or hashed if longer)
- [ ] Multi-tenant keys include tenant isolation
- [ ] No user input is embedded in keys without sanitisation

**TTL:**

- [ ] Every key has a TTL — no unbounded entries
- [ ] TTL values are appropriate for each data type
- [ ] TTL jitter is applied to batch-populated keys
- [ ] L1 TTL < L2 TTL in multi-tier setups
- [ ] Error responses and negative lookups have short TTLs

**Memory:**

- [ ] `maxmemory` is set explicitly
- [ ] Eviction policy is configured and appropriate
- [ ] Memory is provisioned at 1.5× estimated working set
- [ ] Large values (> 1 MB) are compressed or stored elsewhere
- [ ] Memory usage alerts are configured (70%, 90%)

**Failure Handling:**

- [ ] Circuit breaker is in place for cache connections
- [ ] Application functions correctly (if slower) when cache is down
- [ ] Database can handle full load if cache is unavailable
- [ ] Connection pool limits, timeouts, and retries are configured
- [ ] No retry loops on the user request hot path

**Serialization:**

- [ ] Language-independent serialization format (JSON, MessagePack, Protobuf)
- [ ] Schema versioning strategy in place (key version or value version)
- [ ] Maximum value size enforced
- [ ] Compression enabled for values > 1 KB

**Security:**

- [ ] TLS enabled for all cache connections
- [ ] Authentication enabled (password or ACL)
- [ ] Cache instances are in a private subnet
- [ ] PII is not cached, or cached with encryption and short TTL
- [ ] Credentials are not in source code

**Monitoring:**

- [ ] Hit ratio, latency, memory, eviction, and connection metrics are collected
- [ ] Alerting thresholds are configured for all key metrics
- [ ] Dashboard exists with traffic, performance, capacity, and availability views
- [ ] Cache operations are included in distributed traces

**Testing:**

- [ ] Unit tests cover cache hit, miss, and failure paths
- [ ] Integration tests run against a real cache in CI
- [ ] Invalidation correctness is verified by automated tests
- [ ] Load tests include cache-down scenarios

---

## Common Mistakes and How to Avoid Them

These are the caching mistakes that show up in production incidents most frequently. Each one is preventable.

**1. No TTL on cached entries**

- **What happens:** Keys accumulate indefinitely. Memory fills. Evictions become random and uncontrolled.
- **Fix:** Set a TTL on every key. Use `SETEX` / `SET ... EX` instead of bare `SET`.

**2. Caching mutable data without an invalidation strategy**

- **What happens:** Users see stale profiles, prices, or inventory. Trust in the system erodes.
- **Fix:** Pair every cache-write with an invalidation mechanism — event-driven, TTL-based, or both. See [05-CACHE-INVALIDATION.md](05-CACHE-INVALIDATION.md).

**3. Unbounded cache growth**

- **What happens:** No `maxmemory` set. Redis grows until the OS kills it or the host runs out of RAM.
- **Fix:** Always set `maxmemory` and an eviction policy. Monitor memory usage.

**4. Cache stampede on hot keys**

- **What happens:** A popular key expires. Hundreds of concurrent requests all miss, all query the database simultaneously.
- **Fix:** Use lock-based recomputation (only one request rebuilds the value) or early/probabilistic refresh.

**5. Storing large objects in cache**

- **What happens:** A few 10 MB values crowd out thousands of smaller, more frequently accessed entries. Latency spikes on serialization.
- **Fix:** Set a maximum value size. Compress large values. Consider object storage for truly large blobs.

**6. Same TTL for all data types**

- **What happens:** Feature flags are stale for 30 minutes (too long). Static assets expire every 30 seconds (too short, wasting origin requests).
- **Fix:** Choose TTL per data type based on change frequency and staleness tolerance.

**7. No fallback when cache is down**

- **What happens:** Cache goes down. Every request fails or hangs instead of falling back to the database.
- **Fix:** Implement circuit breakers and fallback logic. Test with the cache turned off.

**8. Caching error responses with long TTL**

- **What happens:** A temporary 500 error or 404 is cached. Users see the error for the full TTL duration even after the issue is resolved.
- **Fix:** Cache error responses with a very short TTL (5–30 seconds) or do not cache them at all.

**9. Using language-specific serialization**

- **What happens:** Python pickle or Java Serializable is used. A class rename breaks deserialization. Cross-service reads fail. Security vulnerabilities emerge (pickle is unsafe for untrusted data).
- **Fix:** Use JSON, MessagePack, or Protocol Buffers.

**10. Ignoring cache metrics**

- **What happens:** Hit ratio slowly drops from 95% to 40% over months. Nobody notices until performance degrades to the point of user complaints.
- **Fix:** Monitor hit ratio, latency, memory, and evictions. Set alerts. Review dashboards weekly.

**11. Caching PII without controls**

- **What happens:** User emails, addresses, and phone numbers sit in Redis in plaintext. A network misconfiguration exposes them.
- **Fix:** Default to not caching PII. If required, encrypt, set short TTLs, and audit access.

**12. Cache key collisions in multi-tenant systems**

- **What happens:** Tenant A sees Tenant B's data because both map to `user:42` without a tenant prefix.
- **Fix:** Include tenant ID in every cache key. Test with multiple tenants.

**13. Retry storms on cache failure**

- **What happens:** Cache is slow. Every request retries 3 times with no backoff. The cache, already under load, receives 3× traffic and collapses.
- **Fix:** Do not retry on the hot path. Use a circuit breaker. Fall back to the database immediately.

**14. Forgetting to warm the cache after deployment**

- **What happens:** A new deployment flushes the cache or changes key versions. Hit ratio drops to 0%. The database is overwhelmed by the sudden load.
- **Fix:** Implement cache warming — pre-populate hot keys during deployment. Use gradual rollouts to spread the cold-start load.

**15. Ignoring serialization size growth**

- **What happens:** A cached object gains new fields over time. What started as 200 bytes is now 50 KB. Memory usage and network transfer times grow silently.
- **Fix:** Monitor average value size. Set maximum value size limits. Periodically audit what is being cached.

---

## Version History

| Date       | Change          | Author |
|------------|-----------------|--------|
| 2025-07-15 | Initial version | Team   |
