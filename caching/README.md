# Caching Learning Resources

A comprehensive guide to caching fundamentals, technologies (Redis, Valkey, Memcached), strategies, invalidation patterns, distributed caching, CDN/edge caching, and production best practices.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | Caching fundamentals, memory hierarchy, eviction policies, cache topologies | **Start here** |
| [01-REDIS](01-REDIS.md) | Redis architecture, data structures, persistence, clustering, Lua scripting | When using or evaluating Redis |
| [02-VALKEY](02-VALKEY.md) | Valkey architecture, migration from Redis, compatibility, community governance | When evaluating Valkey as a Redis alternative |
| [03-MEMCACHED](03-MEMCACHED.md) | Memcached architecture, slab allocation, consistent hashing, multi-threading | When using or evaluating Memcached |
| [04-CACHE-STRATEGIES](04-CACHE-STRATEGIES.md) | Cache-aside, read-through, write-through, write-behind, refresh-ahead | When choosing a caching strategy |
| [05-CACHE-INVALIDATION](05-CACHE-INVALIDATION.md) | TTL, event-driven, versioned keys, tag-based, stampede prevention | When designing cache invalidation |
| [06-DISTRIBUTED-CACHING](06-DISTRIBUTED-CACHING.md) | Distributed cache topologies, consistent hashing, replication, partitioning | When scaling caching across nodes |
| [07-CDN-AND-EDGE-CACHING](07-CDN-AND-EDGE-CACHING.md) | CDN caching, edge computing, reverse proxy, HTTP cache headers | When caching at the edge or CDN layer |
| [08-BEST-PRACTICES](08-BEST-PRACTICES.md) | Key design, TTL strategy, memory management, monitoring, security, resilience | **Essential — production checklist** |
| [README](README.md) | This index file | Start here for navigation |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Understand cache hit/miss, TTL, and eviction policies
   - Learn cache types: in-process, distributed, CDN
   - Survey the caching landscape

2. **Learn Cache Strategies** ([04-CACHE-STRATEGIES](04-CACHE-STRATEGIES.md))
   - Cache-aside, read-through, write-through patterns
   - When to use each strategy
   - Combining strategies for different workloads

3. **Understand Invalidation** ([05-CACHE-INVALIDATION](05-CACHE-INVALIDATION.md))
   - TTL-based and event-driven invalidation
   - Stampede prevention techniques
   - The hardest problem in caching

### For Experienced Engineers

1. **Review Best Practices** ([08-BEST-PRACTICES](08-BEST-PRACTICES.md))
   - Cache key design and TTL strategy
   - Memory management and failure handling
   - Security, monitoring, and production readiness

2. **Scale Your Cache** ([06-DISTRIBUTED-CACHING](06-DISTRIBUTED-CACHING.md))
   - Consistent hashing and partitioning
   - Replication and high availability
   - Multi-tier caching architectures

3. **Choose a Technology** ([01-REDIS](01-REDIS.md), [02-VALKEY](02-VALKEY.md), [03-MEMCACHED](03-MEMCACHED.md))
   - Compare Redis, Valkey, and Memcached
   - Understand architecture and trade-offs
   - Evaluate for your use case

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Application Layer                             │
│   Services, APIs, background workers                            │
└────────────────────────┬────────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
┌───────────────┐ ┌─────────────┐ ┌──────────────┐
│  L1 Cache     │ │   L2 Cache  │ │   CDN /      │
│  (In-Process) │ │ (Redis,     │ │   Edge Cache │
│  (Caffeine,   │ │  Valkey,    │ │ (CloudFront, │
│   Guava)      │ │  Memcached) │ │  Fastly)     │
└───────┬───────┘ └──────┬──────┘ └──────────────┘
        │                │
        ▼                ▼
┌───────────────────────────────────────────────────────────────┐
│                    Data Store Layer                             │
│   PostgreSQL, MySQL, MongoDB, DynamoDB, etc.                   │
└───────────────────────────────────────────────────────────────┘
```

## 🔑 Key Concepts

```
Cache Strategies
────────────────
Cache-Aside    → Application manages cache; check cache, fall back to DB
Read-Through   → Cache handles loading from DB transparently
Write-Through  → Writes go to cache and DB synchronously
Write-Behind   → Writes go to cache first, DB asynchronously (batched)
Refresh-Ahead  → Proactively refresh entries before expiration

Invalidation Approaches
───────────────────────
TTL-Based         → Entries expire after a fixed time
Event-Driven      → Invalidate on data change events (CDC, pub/sub)
Versioned Keys    → New key per version; old keys expire naturally
Tag-Based         → Group entries by tags; invalidate by tag

Cache Technologies
──────────────────
Redis      → Rich data structures, Lua scripting, clustering, persistence
Valkey     → Redis-compatible fork, open-source governance
Memcached  → Simple key-value, multi-threaded, slab allocation
CDN        → Edge caching for static/dynamic content, global distribution
```

## 📋 Topics Covered

- **Fundamentals** — Memory hierarchy, hit/miss, TTL, eviction policies, cache topologies
- **Redis** — Data structures, persistence (RDB/AOF), clustering, Sentinel, Lua scripting
- **Valkey** — Redis-compatible fork, migration path, community governance
- **Memcached** — Slab allocation, consistent hashing, multi-threading, binary protocol
- **Cache Strategies** — Cache-aside, read-through, write-through, write-behind, refresh-ahead
- **Cache Invalidation** — TTL, event-driven, versioned keys, tag-based, stampede prevention
- **Distributed Caching** — Consistent hashing, partitioning, replication, multi-tier
- **CDN & Edge** — CDN providers, HTTP caching headers, edge computing, reverse proxy
- **Best Practices** — Key design, TTL, memory, resilience, security, monitoring, testing

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to caching?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md)

**Choosing a strategy?** → Read [04-CACHE-STRATEGIES.md](04-CACHE-STRATEGIES.md)

**Going to production?** → Review [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md)

**Need edge caching?** → Check [07-CDN-AND-EDGE-CACHING.md](07-CDN-AND-EDGE-CACHING.md)
