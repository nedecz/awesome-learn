# Caching Learning Path

A structured, self-paced training guide to mastering caching — from fundamentals and strategy selection to distributed caching, CDN/edge patterns, and production-ready operations. Each phase builds on the previous one, progressing from core theory to hands-on production patterns.

> **Time Estimate:** 6–8 weeks at ~4–5 hours/week. Adjust pace to your experience level. Engineers with prior Redis experience may complete Phases 1–2 in half the time.

---

## How to Use This Guide

1. **Follow the phases in order** — each one builds directly on prior knowledge
2. **Read the linked documents** — they contain detailed content, code examples, and reference tables
3. **Complete the exercises** — hands-on practice is how caching concepts become intuition
4. **Check yourself** — use the knowledge check items before moving to the next phase
5. **Reference the docs** — each phase lists the primary documents to read

---

## Phase 1: Foundations (Week 1–2)

### Learning Objectives

- Understand what caching is and why it matters for application performance
- Learn the memory hierarchy and latency characteristics at each level
- Grasp cache fundamentals: hit/miss, TTL, eviction policies, warm-up
- Understand cache types: in-process, distributed, CDN, browser, database-level
- Survey cache topologies: embedded, client-server, sidecar, multi-tier
- Learn when to cache and when NOT to cache

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | Memory hierarchy, cache hit/miss, TTL, eviction policies, cache types, topologies |

### Knowledge Check

- [ ] What is the difference between a cache hit and a cache miss?
- [ ] What are the five most common eviction policies and when is each appropriate?
- [ ] What is cache penetration, cache avalanche, and cache stampede?
- [ ] When is caching a poor choice? Give three examples.
- [ ] What is the difference between an embedded cache and a client-server cache?

---

## Phase 2: Cache Technologies (Week 2–3)

### Learning Objectives

- Understand Redis architecture, data structures, persistence, and clustering
- Learn Valkey as a Redis-compatible alternative and when to consider it
- Understand Memcached architecture, slab allocation, and multi-threading
- Compare Redis, Valkey, and Memcached for different use cases
- Set up and interact with each technology locally

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [01-REDIS](01-REDIS.md) | Data structures, persistence (RDB/AOF), clustering, Sentinel, Lua scripting |
| 2 | [02-VALKEY](02-VALKEY.md) | Redis compatibility, migration, governance, feature differences |
| 3 | [03-MEMCACHED](03-MEMCACHED.md) | Slab allocation, consistent hashing, multi-threading, binary protocol |

### Knowledge Check

- [ ] What Redis data structures go beyond simple key-value (e.g., sorted sets, streams)?
- [ ] What is the difference between RDB and AOF persistence in Redis?
- [ ] Why was Valkey forked from Redis and what are the implications for users?
- [ ] How does Memcached's slab allocator work and what problem does it solve?
- [ ] When would you choose Memcached over Redis?

---

## Phase 3: Strategies and Invalidation (Week 3–5)

### Learning Objectives

- Master cache-aside, read-through, write-through, write-behind, and refresh-ahead
- Understand when to use each strategy based on workload characteristics
- Learn cache invalidation approaches: TTL, event-driven, versioned keys, tag-based
- Implement stampede prevention techniques
- Design combined strategy architectures for real applications

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [04-CACHE-STRATEGIES](04-CACHE-STRATEGIES.md) | Strategy comparison, combining strategies, CDC, implementation examples |
| 2 | [05-CACHE-INVALIDATION](05-CACHE-INVALIDATION.md) | TTL, event-driven, versioned keys, tag-based, stampede prevention |

### Knowledge Check

- [ ] What is the difference between cache-aside and read-through? When is each better?
- [ ] What are the trade-offs of write-behind (write-back) caching?
- [ ] Why is cache invalidation considered the hardest problem in caching?
- [ ] What is TTL jitter and why is it important?
- [ ] How does probabilistic early expiration (XFetch) prevent stampedes?

---

## Phase 4: Distributed and Edge Caching (Week 5–6)

### Learning Objectives

- Understand distributed caching patterns: consistent hashing, partitioning, replication
- Design multi-tier caching architectures (L1 + L2 + CDN)
- Learn CDN caching, HTTP cache headers, and edge computing
- Understand reverse proxy caching with Varnish, Nginx, and HAProxy
- Design cache key strategies for edge and CDN layers

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [06-DISTRIBUTED-CACHING](06-DISTRIBUTED-CACHING.md) | Consistent hashing, partitioning, replication, multi-tier |
| 2 | [07-CDN-AND-EDGE-CACHING](07-CDN-AND-EDGE-CACHING.md) | CDN providers, HTTP headers, edge computing, reverse proxy |

### Knowledge Check

- [ ] What is consistent hashing and why is it important for distributed caching?
- [ ] How does a multi-tier (L1 + L2) cache architecture reduce latency?
- [ ] What HTTP headers control CDN caching behavior?
- [ ] What is the difference between CDN purge and soft purge?
- [ ] When should you cache API responses at the edge vs at the application layer?

---

## Phase 5: Production Readiness (Week 6–8)

### Learning Objectives

- Apply caching best practices for key design, TTL strategy, and memory management
- Implement failure handling and resilience patterns for cache-dependent systems
- Configure security for caching layers (TLS, ACLs, network segmentation)
- Set up monitoring, alerting, and observability for cache infrastructure
- Test cache behavior under failure conditions and load
- Complete a production readiness checklist for caching deployments
- Design cache schemas that differ from database schemas for optimal read performance
- Choose appropriate Redis data structures for different access patterns
- Apply schema versioning strategies for cached data during deployments

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [08-BEST-PRACTICES](08-BEST-PRACTICES.md) | Key design, TTL, memory, resilience, security, monitoring, testing, checklist |
| 2 | [09-SCHEMA-DESIGN](09-SCHEMA-DESIGN.md) | Cache data modeling, Redis data structures, denormalization, granularity, versioning |

### Knowledge Check

- [ ] What makes a good cache key? Give three naming conventions.
- [ ] How do you prevent OOM (Out of Memory) in a production cache cluster?
- [ ] What should happen when the cache is down? Describe graceful degradation.
- [ ] What are the top 5 metrics to monitor for a caching layer?
- [ ] What are the most common caching mistakes and how do you avoid them?
- [ ] Why should your cache schema differ from your database schema?
- [ ] When would you use a Redis Hash vs a serialized String for a cached entity?
- [ ] What is the difference between fine-grained and coarse-grained caching?

---

## Quick Reference: Document Map

| # | Document | Phase | Key Topics |
|---|----------|-------|------------|
| 00 | [OVERVIEW](00-OVERVIEW.md) | 1 | Memory hierarchy, cache hit/miss, TTL, eviction, topologies |
| 01 | [REDIS](01-REDIS.md) | 2 | Data structures, persistence, clustering, Lua scripting |
| 02 | [VALKEY](02-VALKEY.md) | 2 | Redis compatibility, migration, community governance |
| 03 | [MEMCACHED](03-MEMCACHED.md) | 2 | Slab allocation, consistent hashing, multi-threading |
| 04 | [CACHE-STRATEGIES](04-CACHE-STRATEGIES.md) | 3 | Cache-aside, read-through, write-through, write-behind, refresh-ahead |
| 05 | [CACHE-INVALIDATION](05-CACHE-INVALIDATION.md) | 3 | TTL, event-driven, versioned keys, tag-based, stampede prevention |
| 06 | [DISTRIBUTED-CACHING](06-DISTRIBUTED-CACHING.md) | 4 | Consistent hashing, partitioning, replication, multi-tier |
| 07 | [CDN-AND-EDGE-CACHING](07-CDN-AND-EDGE-CACHING.md) | 4 | CDN, HTTP headers, edge computing, reverse proxy |
| 08 | [BEST-PRACTICES](08-BEST-PRACTICES.md) | 5 | Key design, TTL, memory, resilience, security, monitoring |
| 09 | [SCHEMA-DESIGN](09-SCHEMA-DESIGN.md) | 5 | Cache data modeling, Redis structures, denormalization, granularity, versioning |
| — | [LEARNING-PATH](LEARNING-PATH.md) | All | This document — structured 5-phase curriculum |

---

## Recommended Resources

### Books

| Book | Author | Focus |
|------|--------|-------|
| *Designing Data-Intensive Applications* | Martin Kleppmann | Caching in the context of data systems, partitioning, replication |
| *Redis in Action* | Josiah L. Carlson | Practical Redis patterns and use cases |
| *Web Scalability for Startup Engineers* | Artur Ejsmont | Caching as part of scalable web architectures |

### Online Resources

- **redis.io/docs** — Official Redis documentation and best practices
- **valkey.io/docs** — Valkey documentation and migration guides
- **memcached.org** — Memcached documentation and protocol reference
- **developer.mozilla.org/en-US/docs/Web/HTTP/Caching** — HTTP caching specification

---

## Suggested Learning Path by Role

```
Backend Developer:
  00-OVERVIEW → 04-STRATEGIES → 05-INVALIDATION → 01-REDIS → 08-BEST-PRACTICES

DevOps / SRE:
  00-OVERVIEW → 06-DISTRIBUTED → 01-REDIS → 07-CDN-EDGE → 08-BEST-PRACTICES

Frontend Developer:
  00-OVERVIEW → 07-CDN-EDGE → 04-STRATEGIES → 08-BEST-PRACTICES

Architect:
  00-OVERVIEW → All files → 08-BEST-PRACTICES
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial caching learning path |
