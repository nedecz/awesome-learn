# Common System Designs

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [URL Shortener](#url-shortener)
   - [Requirements](#requirements)
   - [High-Level Design](#high-level-design)
   - [Database Schema](#database-schema)
   - [Hash Generation](#hash-generation)
   - [Redirection Flow](#redirection-flow)
3. [Chat System](#chat-system)
   - [Requirements](#requirements-1)
   - [WebSocket vs Polling](#websocket-vs-polling)
   - [Message Storage](#message-storage)
   - [Online Presence](#online-presence)
   - [Group Chat](#group-chat)
4. [Notification Service](#notification-service)
   - [Requirements](#requirements-2)
   - [Multi-Channel Delivery](#multi-channel-delivery)
   - [Message Queue Architecture](#message-queue-architecture)
   - [Retry and Deduplication](#retry-and-deduplication)
5. [Rate Limiter](#rate-limiter)
   - [Requirements](#requirements-3)
   - [Algorithms](#algorithms)
   - [Algorithm Comparison](#algorithm-comparison)
   - [Distributed Rate Limiting](#distributed-rate-limiting)
6. [News Feed / Timeline](#news-feed--timeline)
   - [Requirements](#requirements-4)
   - [Fanout-on-Write vs Fanout-on-Read](#fanout-on-write-vs-fanout-on-read)
   - [Hybrid Approach](#hybrid-approach)
   - [Caching Strategy](#caching-strategy)
7. [Key-Value Store](#key-value-store)
   - [Requirements](#requirements-5)
   - [Partitioning](#partitioning)
   - [Replication](#replication)
   - [Consistency and Conflict Resolution](#consistency-and-conflict-resolution)
8. [Distributed Cache](#distributed-cache)
   - [Requirements](#requirements-6)
   - [Cache Partitioning](#cache-partitioning)
   - [Cache Coherence](#cache-coherence)
   - [Eviction Policies](#eviction-policies)
9. [Design Process Template](#design-process-template)
10. [Next Steps](#next-steps)
11. [Version History](#version-history)

## Overview

This document covers common system design problems frequently encountered in interviews and real-world engineering. Each design walks through requirements, high-level architecture, key decisions, and trade-offs using practical examples and ASCII diagrams.

### Target Audience

- Software engineers preparing for system design interviews
- Backend developers building scalable distributed systems
- Architects evaluating design trade-offs for production systems

### Scope

- URL shortener, chat system, and notification service
- Rate limiter algorithms and distributed implementations
- News feed fanout strategies and caching
- Key-value store partitioning and replication
- Distributed cache coherence and eviction
- Reusable design process template

---

## URL Shortener

A URL shortener maps long URLs to short, unique aliases and redirects users from the short URL to the original destination.

### Requirements

| Type | Requirement |
|---|---|
| **Functional** | Shorten a long URL to a unique short URL |
| **Functional** | Redirect short URL to the original URL |
| **Functional** | Optional custom aliases |
| **Functional** | Link expiration with configurable TTL |
| **Non-functional** | Low-latency redirection (< 50 ms p99) |
| **Non-functional** | High availability (99.99%) |
| **Non-functional** | 100:1 read-to-write ratio |

### High-Level Design

```
┌──────────┐     ┌─────────────┐     ┌──────────────────┐
│  Client   │────▶│   API       │────▶│  Application     │
│ (Browser) │◀────│  Gateway    │◀────│  Server          │
└──────────┘     └─────────────┘     └───────┬──────────┘
                                             │
                                    ┌────────┴────────┐
                               ┌────▼─────┐    ┌─────▼─────┐
                               │  Cache   │    │ Database  │
                               │ (Redis)  │    │ (Postgres)│
                               └──────────┘    └───────────┘
```

### Database Schema

```sql
CREATE TABLE urls (
    id           BIGINT PRIMARY KEY,
    short_code   VARCHAR(7) UNIQUE NOT NULL,
    original_url TEXT NOT NULL,
    created_at   TIMESTAMP DEFAULT NOW(),
    expires_at   TIMESTAMP,
    click_count  BIGINT DEFAULT 0
);
```

### Hash Generation

| Approach | Pros | Cons |
|---|---|---|
| **Base62 encoding** | Short output, no special chars | Requires unique ID generation |
| **MD5/SHA-256 truncation** | Simple to implement | Collision risk with truncation |
| **Pre-generated key store** | Zero collision, fast lookup | Requires key management service |

Base62 uses `[0-9a-zA-Z]` (62 characters). A 7-character code yields 62^7 = 3.5 trillion unique combinations.

### Redirection Flow

```
1. Client requests GET /dK8Tn2Z
2. Server checks Redis cache
   ├── HIT  → Return 301/302 redirect
   └── MISS → Query DB → cache result → return 301/302
3. Use 302 when analytics tracking is needed (no browser caching)
```

### Key Decisions

- ✅ Use 302 redirects when analytics tracking is needed
- ✅ Cache hot URLs in Redis with TTL matching link expiration
- ✅ Use Base62 with a pre-generated key store for zero-collision guarantees

---

## Chat System

A real-time messaging system supporting one-on-one and group conversations with online presence indicators.

### Requirements

| Type | Requirement |
|---|---|
| **Functional** | One-on-one and group messaging (up to 500 members) |
| **Functional** | Online/offline presence indicators |
| **Functional** | Message history, read receipts, typing indicators |
| **Non-functional** | Message delivery latency < 200 ms |
| **Non-functional** | Message ordering guaranteed per conversation |
| **Non-functional** | At-least-once delivery with deduplication |

### WebSocket vs Polling

| Method | Latency | Server Load | Complexity | Use Case |
|---|---|---|---|---|
| **Short polling** | High (interval) | Very high | Low | Legacy fallback |
| **Long polling** | Medium | High | Medium | Low-frequency updates |
| **WebSocket** | Very low | Low | Medium | Real-time bidirectional |
| **Server-Sent Events** | Low | Medium | Low | Server-to-client only |

**WebSocket is the standard choice for chat** — persistent bidirectional connection with minimal overhead after handshake.

### Message Storage

Messages are stored in Cassandra, partitioned by `conversation_id` with a time-ordered `message_id` as the clustering key. This ensures sequential reads within a conversation. Recent messages are cached in Redis sorted sets for fast access.

### Online Presence

Clients send a heartbeat every 5 seconds via WebSocket. The server updates a `last_active` timestamp in Redis with a 30-second TTL. If the TTL expires, the user is marked offline and a status change is broadcast to friends via pub/sub.

### Group Chat

- **Small groups (< 100):** Fan out directly via WebSocket connections
- **Large groups (100–500):** Write to a per-user inbox queue; clients pull on reconnect

### Architecture

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Client A │  │ Client B │  │ Client C │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │ WS          │ WS          │ WS
┌────▼─────────────▼─────────────▼────┐
│          WebSocket Gateway          │
└──┬──────────────┬───────────────┬───┘
   │              │               │
┌──▼───┐   ┌─────▼─────┐   ┌────▼──────┐
│ Chat │   │ Presence  │   │ Group     │
│ Svc  │   │ Service   │   │ Service   │
└──┬───┘   └─────┬─────┘   └────┬──────┘
   │             │               │
┌──▼─────────────▼───────────────▼────┐
│         Message Queue (Kafka)       │
└──┬──────────────────────────────┬───┘
   │                              │
┌──▼──────────┐          ┌───────▼────┐
│ Message DB  │          │   Redis    │
│ (Cassandra) │          │ (Presence) │
└─────────────┘          └────────────┘
```

---

## Notification Service

A multi-channel notification system that delivers push notifications, emails, and SMS messages reliably at scale.

### Requirements

| Type | Requirement |
|---|---|
| **Functional** | Push notifications (iOS, Android, Web), email, and SMS |
| **Functional** | User notification preferences and templates |
| **Non-functional** | At-least-once delivery with deduplication |
| **Non-functional** | < 30-second end-to-end latency (p95) |
| **Non-functional** | 10 million notifications per day |

### Multi-Channel Delivery

| Channel | Provider | Latency | Deliverability |
|---|---|---|---|
| **Push (iOS/Android)** | APNs / FCM | ~1 s | High (if app installed) |
| **Push (Web)** | Web Push API | ~2 s | Medium (browser open) |
| **Email** | SES, SendGrid | 5–30 s | Medium (spam filters) |
| **SMS** | Twilio, SNS | 3–10 s | High (carrier dependent) |

### Message Queue Architecture

```
┌──────────────────┐
│  Caller Service  │
└────────┬─────────┘
    ┌────▼─────────────────────────────┐
    │     Notification Service API     │
    │  Validate → Preferences →        │
    │  Template → Dedup → Enqueue      │
    └──┬──────────┬──────────┬─────────┘
  ┌────▼───┐ ┌───▼────┐ ┌───▼───┐
  │ Push   │ │ Email  │ │  SMS  │
  │ Queue  │ │ Queue  │ │ Queue │
  └───┬────┘ └───┬────┘ └───┬───┘
  ┌───▼────┐ ┌──▼─────┐ ┌──▼─────┐
  │ Push   │ │ Email  │ │  SMS   │
  │ Worker │ │ Worker │ │ Worker │
  │→APNs   │ │→ SES   │ │→Twilio │
  │→FCM    │ │        │ │        │
  └───┬────┘ └───┬────┘ └───┬────┘
      └──────────┴──────────┘
         ┌───────▼───────┐
         │  Delivery DB  │
         └───────────────┘
```

### Retry and Deduplication

**Retry:** Exponential backoff — immediate, 1 min, 5 min, 30 min, 2 hours. Dead-letter queue after max retries.

**Deduplication:** Each request carries an `idempotency_key`. Before processing, check Redis with `SET notification:{key} 1 NX EX 86400`. If the key already exists, skip processing.

### Key Decisions

- ✅ Separate queues per channel for independent scaling and failure isolation
- ✅ Idempotency keys to prevent duplicate notifications
- ❌ Avoid synchronous delivery — always use queues for resilience

---

## Rate Limiter

A rate limiter controls the rate of requests a client can send to an API, protecting services from abuse and ensuring fair usage.

### Requirements

| Type | Requirement |
|---|---|
| **Functional** | Limit requests per client (by API key, IP, or user ID) |
| **Functional** | Return 429 with rate limit headers when exceeded |
| **Non-functional** | Sub-millisecond overhead per request |
| **Non-functional** | Accurate limiting in a distributed environment |

### Algorithms

**Token Bucket:**

```
Bucket capacity: 10 tokens, refill rate: 2 tokens/sec

Time 0s:  [■ ■ ■ ■ ■ ■ ■ ■ ■ ■]  10/10  → 5 requests arrive
Time 0s:  [■ ■ ■ ■ ■ □ □ □ □ □]   5/10   → 5 tokens consumed
Time 1s:  [■ ■ ■ ■ ■ ■ ■ □ □ □]   7/10   → 2 tokens refilled

✅ Allows short bursts up to bucket capacity
```

**Sliding Window Log:**

```
Window: 1 min, Limit: 5 requests — stores each request timestamp

  10:00:15 → Req 1 ✅    10:00:55 → Req 5 ✅
  10:01:05 → Req 6 ✅ (10:00:15 expired, outside window)

✅ Very accurate    ❌ High memory (stores every timestamp)
```

**Fixed Window Counter:**

```
Window: 1 min, Limit: 5
  ┌──────────────┐     ┌──────────────┐
  │ 10:00: 5/5   │     │ 10:01: 0/5   │
  └──────────────┘     └──────────────┘

⚠ Boundary problem: 5 at 10:00:59 + 5 at 10:01:00 = 10 in 2 seconds
```

**Leaky Bucket:**

```
Processes requests at a constant rate. Incoming bursts queue up
in the bucket. Overflow is dropped. Smooth output, no bursts.
```

### Algorithm Comparison

| Algorithm | Burst Handling | Memory | Accuracy | Complexity |
|---|---|---|---|---|
| **Token Bucket** | Allows bursts | Low (2 values) | Good | Low |
| **Leaky Bucket** | Smooths bursts | Low (queue size) | Good | Low |
| **Fixed Window** | Boundary spike risk | Low (1 counter) | Moderate | Very low |
| **Sliding Window Log** | No bursts | High (timestamps) | Exact | Medium |
| **Sliding Window Counter** | Minimal bursts | Low (2 counters) | Near-exact | Medium |

### Distributed Rate Limiting

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Server 1 │  │ Server 2 │  │ Server 3 │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     └──────┬──────┴──────┬───────┘
       ┌────▼─────────────▼────┐
       │    Redis Cluster      │
       │  Key: rate:{client}   │
       │  Lua script for       │
       │  atomic check-and-    │
       │  decrement            │
       └───────────────────────┘
```

### Key Decisions

- ✅ Token bucket for most APIs — simple and allows controlled bursts
- ✅ Redis with Lua scripts for distributed atomic rate checks
- ✅ Return informative headers (X-RateLimit-Limit, X-RateLimit-Remaining, Retry-After)

---

## News Feed / Timeline

A system that aggregates and displays posts from followed users in a personalized, chronological, or ranked feed.

### Requirements

| Type | Requirement |
|---|---|
| **Functional** | User publishes a post; user views a personalized feed |
| **Functional** | Feed supports text, images, and links |
| **Non-functional** | Feed loads in < 500 ms |
| **Non-functional** | Support users with millions of followers |

### Fanout-on-Write vs Fanout-on-Read

| Aspect | Fanout-on-Write (Push) | Fanout-on-Read (Pull) |
|---|---|---|
| **When work happens** | At publish time | At read time |
| **Write cost** | High (N followers × 1 write) | Low (1 write) |
| **Read cost** | Low (read pre-built feed) | High (query all followed users) |
| **Latency** | Very fast reads | Slower reads |
| **Celebrity problem** | Expensive (millions of writes) | Handles naturally |
| **Freshness** | Eventual (fanout delay) | Always fresh |

### Hybrid Approach

```
Post published:
  ├── Celebrity (> 100K followers)? → Store in posts table only
  │     (fanout-on-read at query time)
  └── Normal user? → Fanout to all followers' feed caches
        (fanout-on-write — fast reads)

At read time: Merge pre-built feed + celebrity posts → rank → return top N
```

### Caching Strategy

| Layer | What | TTL |
|---|---|---|
| **CDN** | Static assets (images, profile photos) | Long |
| **App Cache (Redis)** | Pre-built feed per user (post ID list) | 5 minutes |
| **Object Cache** | Post content by post_id | 1 hour |
| **Database** | Source of truth for posts and social graph | — |

### Architecture

```
┌──────────┐           ┌──────────────────┐
│  Client  │──────────▶│   API Gateway    │
└──────────┘           └────────┬─────────┘
                                │
                   ┌────────────┼────────────┐
                   │            │            │
             ┌─────▼──┐  ┌─────▼──┐  ┌──────▼─────┐
             │ Post   │  │ Feed   │  │ Social     │
             │ Service│  │ Service│  │ Graph Svc  │
             └───┬────┘  └───┬────┘  └──────┬─────┘
                 │           │              │
            ┌────▼────┐ ┌────▼────┐  ┌──────▼─────┐
            │ Post DB │ │ Feed    │  │ Graph DB   │
            │         │ │ Cache   │  │ (followers)│
            │         │ │ (Redis) │  │            │
            └─────────┘ └─────────┘  └────────────┘
                 │
            ┌────▼────────────────┐
            │  Fanout Workers     │
            │  (async via Kafka)  │
            └─────────────────────┘
```

---

## Key-Value Store

A distributed key-value store that provides fast reads and writes with configurable consistency guarantees.

### Requirements

| Type | Requirement |
|---|---|
| **Functional** | Put(key, value), Get(key), Delete(key) with TTL |
| **Non-functional** | Single-digit millisecond latency |
| **Non-functional** | High availability (AP or CP configurable) |
| **Non-functional** | Horizontal scalability to petabytes |

### Partitioning

```
Consistent Hashing Ring:

                    Node A
                   ╱      ╲
         Node D ◀─  ● key  ─▶ Node B
                   ╲      ╱
                    Node C

Keys hash to a position on the ring and walk clockwise to find
the responsible node. Virtual nodes (100–200 per physical node)
improve balance. Adding/removing a node only affects neighbors.
```

### Replication

```
Replication factor N = 3: Write to coordinator node, replicate
to next N-1 nodes clockwise on the ring.

  ┌────────┐     ┌────────┐     ┌────────┐
  │ Node B │────▶│ Node C │────▶│ Node D │
  │(coord.)│     │(replica)│     │(replica)│
  └────────┘     └────────┘     └────────┘

Quorum: W + R > N ensures consistency
  W=2, R=2, N=3 → strong consistency
  W=1, R=1, N=3 → eventual consistency, high availability
```

### Consistency and Conflict Resolution

| Parameter | Strong Consistency | Eventual Consistency |
|---|---|---|
| **W (write quorum)** | Majority (e.g., 2/3) | 1 |
| **R (read quorum)** | Majority (e.g., 2/3) | 1 |
| **Latency** | Higher (wait for quorum) | Lower (single ack) |
| **Availability** | Lower (need quorum nodes) | Higher (any node) |

**Conflict resolution:** Last-Writer-Wins (simple, may lose writes), vector clocks (detect conflicts), CRDTs (merge automatically), or application-level resolution (return all versions).

### Architecture

```
┌──────────────────────────────────────────┐
│       Client Library (partition-aware,   │
│       retry, consistent hashing)         │
└──────────┬──────────────────┬────────────┘
     ┌─────▼──────┐    ┌─────▼──────┐
     │  Node A    │    │  Node B    │
     │ ┌────────┐ │    │ ┌────────┐ │
     │ │MemTable│ │    │ │MemTable│ │
     │ └───┬────┘ │    │ └───┬────┘ │
     │ ┌───▼────┐ │    │ ┌───▼────┐ │
     │ │  WAL   │ │    │ │  WAL   │ │
     │ └───┬────┘ │    │ └───┬────┘ │
     │ ┌───▼────┐ │    │ ┌───▼────┐ │
     │ │SSTables│ │    │ │SSTables│ │
     │ └────────┘ │    │ └────────┘ │
     └────────────┘    └────────────┘

Write: Client → MemTable + WAL → flush to SSTable
Read:  Client → MemTable → Bloom filter → SSTables
```

---

## Distributed Cache

A caching layer distributed across multiple nodes that reduces database load and improves read latency.

### Requirements

| Type | Requirement |
|---|---|
| **Functional** | Get, Set, Delete with TTL-based expiration |
| **Functional** | Support for multiple data structures |
| **Non-functional** | Sub-millisecond read latency |
| **Non-functional** | Linear scalability; graceful node failure handling |

### Cache Partitioning

| Strategy | Description | Pros | Cons |
|---|---|---|---|
| **Modulo hashing** | key_hash % N | Simple | Reshuffles on resize |
| **Consistent hashing** | Hash ring | Minimal reshuffling | Uneven distribution |
| **Consistent hashing + vnodes** | Virtual nodes on ring | Even distribution | Slightly more complex |

### Cache Coherence

```
Cache-Aside (Lazy Loading):
  Read: Client → Cache (miss?) → DB → populate Cache
  Write: Client → DB → invalidate Cache

Write-Through:
  Client → Cache → DB (synchronous write to both)

Write-Behind (Write-Back):
  Client → Cache → async Queue → DB (batched)
```

| Pattern | Consistency | Write Latency | Complexity |
|---|---|---|---|
| **Cache-Aside** | Eventual | Low (DB only) | Low |
| **Write-Through** | Strong | Higher (both) | Medium |
| **Write-Behind** | Eventual | Very low | High |

### Eviction Policies

| Policy | Description | Best For |
|---|---|---|
| **LRU** (Least Recently Used) | Evict least recently accessed | General-purpose workloads |
| **LFU** (Least Frequently Used) | Evict least frequently accessed | Stable popularity distributions |
| **FIFO** (First In First Out) | Evict oldest entries | Short-lived data |
| **TTL-based** | Evict on expiration | Time-sensitive data |
| **Random** | Evict random entry | When access patterns are uniform |

### Architecture

```
┌──────────────────────────────────────────────────┐
│                  Application                      │
│   ┌──────────────────────────────────────────┐   │
│   │  Cache Client (consistent hashing,       │   │
│   │  connection pooling, failover)           │   │
│   └─────┬──────────────┬──────────────┬──────┘   │
└─────────┼──────────────┼──────────────┼──────────┘
     ┌────▼─────┐  ┌────▼─────┐  ┌────▼─────┐
     │ Cache    │  │ Cache    │  │ Cache    │
     │ Node 1  │  │ Node 2  │  │ Node 3  │
     │ Shard A  │  │ Shard B  │  │ Shard C  │
     └──────────┘  └──────────┘  └──────────┘

Client hashes key → target node. Failed node → keys rehash to neighbors.
```

---

## Design Process Template

A reusable step-by-step framework for approaching any system design problem.

### Step 1: Requirements Clarification (5 minutes)

- What are the core functional requirements?
- What are the non-functional requirements (latency, throughput, availability)?
- What is the expected scale (users, requests/second, data volume)?
- Are there any constraints (budget, technology, compliance)?

### Step 2: Back-of-the-Envelope Estimation (5 minutes)

Estimate users, requests/second, storage, and bandwidth. Example for a URL shortener: 100M MAU, 330K writes/day ≈ 4 writes/sec, ~60 GB/year storage.

### Step 3: High-Level Design (10 minutes)

- Draw the major components (clients, servers, databases, caches)
- Identify the data flow for primary use cases
- Choose synchronous vs asynchronous communication patterns

### Step 4: Detailed Design (15 minutes)

- Database schema and indexing strategy
- API design (endpoints, request/response formats)
- Key algorithms (hashing, ranking, routing)
- Caching strategy (what to cache, eviction, invalidation)

### Step 5: Identify Bottlenecks and Trade-Offs (5 minutes)

Ask: What are the single points of failure? Can this handle 10× traffic? Is eventual consistency acceptable? Are there hot spots? Is it cost-effective at scale?

### Step 6: Summary and Extensions

Recap the design in one or two sentences. Propose future improvements (monitoring, multi-region, ML ranking). Acknowledge known trade-offs explicitly.

---

## Next Steps

Return to [Scalability](01-SCALABILITY.md) to review horizontal and vertical scaling strategies, or revisit [Caching Strategies](03-CACHING-STRATEGIES.md) for deeper coverage of cache invalidation patterns used in these designs.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial version — URL shortener, chat system, notification service, rate limiter, news feed, key-value store, distributed cache, design process template |
