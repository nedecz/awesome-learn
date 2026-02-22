# Scalability

## Table of Contents

1. [Overview](#overview)
2. [What Is Scalability](#what-is-scalability)
3. [Vertical vs Horizontal Scaling](#vertical-vs-horizontal-scaling)
4. [Load Balancing Basics](#load-balancing-basics)
5. [Content Delivery Networks (CDNs)](#content-delivery-networks-cdns)
6. [Database Scaling](#database-scaling)
7. [Application-Level Scaling](#application-level-scaling)
8. [Caching for Scale](#caching-for-scale)
9. [Asynchronous Processing](#asynchronous-processing)
10. [Scaling Patterns](#scaling-patterns)
11. [Capacity Planning](#capacity-planning)
12. [Next Steps](#next-steps)

## Overview

This document covers the principles and techniques for building systems that handle growing amounts of traffic, data, and users. Scalability is a foundational topic in system design — it determines whether your architecture can grow with your product.

### Target Audience

- Software engineers designing systems that must handle growth
- Architects evaluating scaling strategies for new or existing platforms
- DevOps and platform engineers implementing infrastructure-level scaling

### Scope

- Vertical and horizontal scaling strategies
- Load balancing fundamentals (detailed coverage in [04-LOAD-BALANCING.md](04-LOAD-BALANCING.md))
- CDN architecture and edge caching
- Database replication, sharding, and connection pooling
- Stateless application design and autoscaling
- Caching overview (detailed coverage in [03-CACHING-STRATEGIES.md](03-CACHING-STRATEGIES.md))
- Asynchronous processing and message queues
- Scaling patterns: throttling, backpressure, circuit breakers
- Capacity planning and growth estimation

## What Is Scalability

Scalability is the ability of a system to handle increased load — whether that is more users, more requests, or more data — without degrading performance or requiring a complete redesign.

### Why Scalability Matters

- **User growth** — A system that works for 1,000 users may fail at 100,000
- **Traffic spikes** — Flash sales, viral content, and seasonal peaks demand elastic capacity
- **Data volume** — Storage and query performance degrade as data grows
- **Business continuity** — Systems that cannot scale become bottlenecks to revenue

### Key Metrics

| Metric | Description | Example |
|---|---|---|
| **Throughput** | Requests processed per unit of time | 10,000 requests/second |
| **Latency** | Time to process a single request | p99 < 200 ms |
| **Concurrency** | Number of simultaneous connections | 50,000 concurrent users |
| **Resource utilization** | CPU, memory, disk, network usage | CPU < 70% at peak |
| **Error rate** | Percentage of failed requests under load | < 0.1% at target throughput |

## Vertical vs Horizontal Scaling

There are two fundamental approaches to scaling a system: scaling up (vertical) and scaling out (horizontal).

### Vertical Scaling (Scale Up)

Add more resources — CPU, RAM, faster disks — to a single machine.

```
Before                          After

┌──────────────┐               ┌──────────────┐
│   Server     │               │   Server     │
│              │               │              │
│  4 CPU cores │               │  32 CPU cores│
│  16 GB RAM   │    ──────►    │  256 GB RAM  │
│  500 GB SSD  │               │  2 TB NVMe   │
│              │               │              │
└──────────────┘               └──────────────┘
    Small box                     Bigger box
```

### Horizontal Scaling (Scale Out)

Add more machines to distribute the load across multiple nodes.

```
Before                          After

┌──────────────┐               ┌──────────┐ ┌──────────┐ ┌──────────┐
│   Server     │               │ Server 1 │ │ Server 2 │ │ Server 3 │
│              │               │          │ │          │ │          │
│  4 CPU cores │               │  4 cores │ │  4 cores │ │  4 cores │
│  16 GB RAM   │    ──────►    │  16 GB   │ │  16 GB   │ │  16 GB   │
│              │               │          │ │          │ │          │
└──────────────┘               └──────────┘ └──────────┘ └──────────┘
    One box                        Multiple boxes behind
                                   a load balancer
```

### Comparison

| Factor | Vertical Scaling | Horizontal Scaling |
|---|---|---|
| **Complexity** | Low — no code changes required | Higher — requires distributed design |
| **Cost curve** | Exponential (premium hardware) | Linear (commodity hardware) |
| **Upper limit** | Hardware ceiling (largest available machine) | Virtually unlimited |
| **Downtime** | Usually requires restart | Zero-downtime with rolling deploys |
| **Failure impact** | Single point of failure | One node fails, others continue |
| **Data consistency** | Simple — single machine | Requires coordination (replication, consensus) |
| **Best for** | Databases, legacy apps, quick wins | Stateless services, web servers, microservices |

### When to Use Each

- ✅ **Vertical** — Database servers where sharding is complex, quick capacity boost, early-stage products
- ✅ **Horizontal** — Stateless web/API servers, read-heavy workloads, systems requiring high availability
- ❌ **Vertical** — When you are already on the largest available instance, or need fault tolerance
- ❌ **Horizontal** — When the application has heavy shared mutable state that is hard to partition

## Load Balancing Basics

A load balancer distributes incoming traffic across multiple backend servers to prevent any single server from becoming overwhelmed. It is the enabler of horizontal scaling.

> **Note:** For a deep dive into algorithms, Layer 4 vs Layer 7, health checks, and advanced configurations, see [04-LOAD-BALANCING.md](04-LOAD-BALANCING.md).

### How It Works

```
                    Incoming Requests
                          │
                   ┌──────▼──────┐
                   │    Load     │
                   │  Balancer   │
                   └──┬───┬───┬──┘
            ┌─────────┘   │   └──────────┐
            ▼             ▼              ▼
      ┌──────────┐ ┌──────────┐  ┌──────────┐
      │ Server 1 │ │ Server 2 │  │ Server 3 │
      │ (healthy)│ │ (healthy)│  │ (healthy)│
      └──────────┘ └──────────┘  └──────────┘
```

### Common Algorithms

| Algorithm | How It Works | Best For |
|---|---|---|
| **Round Robin** | Rotates through servers sequentially | Equal-capacity servers |
| **Least Connections** | Routes to the server with fewest active connections | Varying request durations |
| **Weighted Round Robin** | Assigns proportional traffic based on server capacity | Mixed-capacity clusters |
| **IP Hash** | Hashes client IP to a consistent server | Session affinity without sticky cookies |
| **Random** | Selects a server at random | Simple, stateless workloads |

### Load Balancer Types

| Type | OSI Layer | Inspects | Use Case |
|---|---|---|---|
| **Layer 4 (TCP/UDP)** | Transport | IP + port | High throughput, low overhead |
| **Layer 7 (HTTP)** | Application | URL, headers, cookies | Content-based routing, SSL termination |

## Content Delivery Networks (CDNs)

A CDN is a globally distributed network of edge servers that cache and serve content close to users. CDNs reduce latency by shortening the physical distance between the user and the server responding to their request.

### How CDNs Work

```
Without CDN:                          With CDN:

┌────────┐       3000 km             ┌────────┐       50 km
│  User  │ ─────────────────►        │  User  │ ─────────────►
│ (Tokyo)│                           │ (Tokyo)│
└────────┘                           └────────┘
                    │                                   │
             ┌──────▼──────┐                     ┌──────▼──────┐
             │   Origin    │                     │  CDN Edge   │
             │  Server     │                     │  (Tokyo)    │
             │ (US-East)   │                     └──────┬──────┘
             └─────────────┘                       cache│miss?
                                                 ┌──────▼──────┐
              Latency: ~200 ms                   │   Origin    │
                                                 │  (US-East)  │
                                                 └─────────────┘

                                                  Latency: ~30 ms
                                                  (cache hit)
```

### Push vs Pull CDN

| Aspect | Push CDN | Pull CDN |
|---|---|---|
| **How content arrives** | You upload content to edge nodes | Edge nodes fetch from origin on first request |
| **Cache population** | Proactive — content available before first request | Reactive — first request triggers a cache miss |
| **Best for** | Static assets that change infrequently | Dynamic or frequently updated content |
| **Storage cost** | Higher — content replicated to all edges upfront | Lower — only requested content is cached |
| **Staleness risk** | Low — you control when to push updates | Managed via TTL and cache invalidation |
| **Examples** | Pre-deployed firmware images, large media files | Website pages, API responses, images |

### Edge Caching Strategies

- **TTL-based expiration** — Set a Time-To-Live on cached objects; edge serves from cache until TTL expires
- **Cache-Control headers** — Use `Cache-Control: public, max-age=86400` for assets; `no-cache` for personalized content
- **Cache invalidation** — Purge specific URLs or use versioned filenames (`app.v2.3.js`) to bust cache
- **Origin shield** — Add a mid-tier cache between edges and the origin to reduce origin load

### Common CDN Providers

| Provider | Key Features |
|---|---|
| **Cloudflare** | Global network, DDoS protection, Workers (edge compute) |
| **AWS CloudFront** | Deep AWS integration, Lambda@Edge, origin failover |
| **Akamai** | Largest network, enterprise-grade, advanced security |
| **Fastly** | Real-time purging, VCL configuration, edge compute |

## Database Scaling

Databases are often the hardest component to scale because they manage persistent state. There are several complementary techniques for scaling data access.

### Read Replicas

Distribute read queries across replica nodes while writes go to the primary.

```
           Writes                    Reads
             │                    ┌────┴────┐
      ┌──────▼──────┐     ┌──────▼───┐ ┌───▼───────┐
      │   Primary   │     │ Replica 1│ │ Replica 2 │
      │   (Leader)  │────►│ (Reader) │ │ (Reader)  │
      │             │ WAL │          │ │           │
      └─────────────┘ rep └──────────┘ └───────────┘
```

- ✅ Scales read-heavy workloads (80%+ reads is common)
- ✅ Replicas can serve analytics queries without impacting production writes
- ❌ Replication lag means replicas may serve slightly stale data
- ❌ Does not help scale write throughput

### Write Scaling

When a single primary cannot keep up with write volume, consider these approaches:

| Technique | Description | Trade-Off |
|---|---|---|
| **Sharding** | Partition data across multiple databases by a shard key | Cross-shard queries are complex |
| **Multi-leader replication** | Multiple nodes accept writes and sync | Conflict resolution required |
| **Write batching** | Buffer writes and flush in bulk | Slightly delayed persistence |
| **CQRS** | Separate write-optimized store from read-optimized store | Eventual consistency between stores |

### Connection Pooling

Opening a new database connection per request is expensive. Connection pools maintain a set of reusable connections.

```
Without Pooling:                    With Pooling:

App ──► open conn ──► query         App ──► pool ──► reuse conn ──► query
App ──► open conn ──► query         App ──► pool ──► reuse conn ──► query
App ──► open conn ──► query         App ──► pool ──► reuse conn ──► query
  (3 new connections)                 (3 queries, 1–2 connections)
```

| Pool Setting | Recommended Starting Value | Notes |
|---|---|---|
| **Min connections** | 5–10 | Pre-warmed connections for baseline traffic |
| **Max connections** | 20–50 per app instance | Avoid exceeding the database max connection limit |
| **Idle timeout** | 300 seconds | Release unused connections back to the pool |
| **Connection lifetime** | 1800 seconds | Rotate connections to pick up DNS/config changes |

## Application-Level Scaling

Application servers are typically the easiest tier to scale horizontally, provided they are designed correctly.

### Stateless Services

A stateless service does not store client session data on the server. Every request contains all the information needed to process it.

```
Stateful (hard to scale):           Stateless (easy to scale):

┌──────────┐                        ┌──────────┐
│ Server A │ ◄── user session       │ Server A │
│ (session │     stored here        │ (no local│
│  in RAM) │                        │  state)  │
└──────────┘                        └──────────┘
     ▲                                   ▲
     │ must route to same server         │ any server can handle
     │                                   │   any request
┌────┴─────┐                        ┌────┴─────┐
│  Client  │                        │  Client  │
└──────────┘                        └──────────┘
```

### Session Management for Stateless Services

| Strategy | How It Works | Trade-Off |
|---|---|---|
| **Client-side tokens (JWT)** | Session data encoded in a signed token sent with each request | Token size grows with claims; revocation is complex |
| **External session store** | Store sessions in Redis or Memcached; server reads on each request | Adds a network hop; session store must be highly available |
| **Sticky sessions** | Load balancer routes same client to same server | Uneven load distribution; failover loses session |

### Autoscaling

Autoscaling automatically adjusts the number of running instances based on observed metrics.

```
                 CPU Utilization
100% ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
 80% ─ ─ ─ ─ ─ ─ ─ ─ Scale Up ─ ─ ─   ◄── high threshold
      ╱‾‾‾‾‾╲       ╱‾‾‾‾‾╲
     ╱       ╲     ╱       ╲
 40% ╱─ ─ ─ ─ ╲─ ╱─ ─ ─ ─ ─╲─ ─ ─ ─
 20% ─ ─ ─ ─ ─ Scale Down ─ ─ ─ ─ ─   ◄── low threshold
  0% ─────────────────────────────────
      06:00    12:00    18:00   00:00
```

| Scaling Type | Trigger | Response Time |
|---|---|---|
| **Reactive (metric-based)** | CPU > 70%, request queue > 100 | Minutes (provision + warm-up) |
| **Scheduled** | Cron-like rules for known traffic patterns | Instant (pre-provisioned) |
| **Predictive** | ML model forecasts load from historical data | Minutes (provisions ahead of spike) |

## Caching for Scale

Caching stores frequently accessed data in a fast-access layer to reduce load on slower backend systems. It is one of the most effective tools for scaling read-heavy workloads.

> **Note:** For a deep dive into caching patterns, eviction policies, cache invalidation strategies, and distributed caching, see [03-CACHING-STRATEGIES.md](03-CACHING-STRATEGIES.md).

### Where to Cache

```
┌────────┐    ┌───────────┐    ┌───────────┐    ┌──────────┐    ┌──────────┐
│ Client │───►│  CDN /    │───►│ App-Level │───►│ Database │───►│ Database │
│ Cache  │    │  Edge     │    │ Cache     │    │ Query    │    │          │
│(browser│    │  Cache    │    │ (Redis,   │    │ Cache    │    │          │
│ local) │    │           │    │ Memcached)│    │ (buffer  │    │          │
└────────┘    └───────────┘    └───────────┘    │  pool)   │    └──────────┘
                                                └──────────┘
 Fastest ◄──────────────────────────────────────────────────── Slowest
```

### Quick Reference

| Cache Layer | Latency | Typical Use |
|---|---|---|
| **Browser cache** | 0 ms (local) | Static assets, API responses with `Cache-Control` |
| **CDN edge cache** | 5–30 ms | Images, CSS, JS, public API responses |
| **Application cache** | 1–5 ms | Session data, database query results, computed values |
| **Database query cache** | Varies | Repeated identical queries (use with caution) |

## Asynchronous Processing

Not every operation needs to happen in real time. Offloading work to background processes reduces response times and smooths out traffic spikes.

### Message Queues

A message queue decouples producers (senders) from consumers (workers), allowing them to operate independently and at different speeds.

```
┌──────────┐    ┌──────────────────┐    ┌──────────┐
│ Producer │───►│  Message Queue   │───►│ Consumer │
│ (API     │    │                  │    │ (Worker) │
│  server) │    │ ┌──┐┌──┐┌──┐┌──┐│    │          │
│          │    │ │M4││M3││M2││M1││    │ Process  │
│ Enqueue  │    │ └──┘└──┘└──┘└──┘│    │ at own   │
│ and      │    │                  │    │ pace     │
│ respond  │    │ (Kafka, SQS,    │    │          │
│ quickly  │    │  RabbitMQ)      │    │          │
└──────────┘    └──────────────────┘    └──────────┘
```

### Common Use Cases

| Use Case | Synchronous Approach | Async Approach |
|---|---|---|
| **Email sending** | Send email inline, slow response | Enqueue message, worker sends email |
| **Image resizing** | Resize during upload, timeout risk | Enqueue job, worker processes in background |
| **Order processing** | Process payment + inventory + notification in one request | Enqueue order event, dedicated services handle each step |
| **Report generation** | Generate on request, user waits | Enqueue job, notify user when ready |

### Background Workers

Workers consume messages from queues and perform processing independently of the request/response cycle.

| Pattern | Description |
|---|---|
| **Competing consumers** | Multiple workers pull from the same queue for parallel processing |
| **Fan-out** | One message published to multiple queues; each queue has its own consumer |
| **Dead letter queue (DLQ)** | Failed messages are moved to a separate queue for inspection and retry |
| **Delayed processing** | Messages are held in the queue for a specified delay before delivery |

### Event-Driven Patterns

In an event-driven architecture, services communicate by publishing and subscribing to events rather than making direct calls.

```
┌─────────────┐   OrderPlaced    ┌───────────────┐
│ Order       │ ───────────────► │ Event Broker  │
│ Service     │                  │ (Kafka topic) │
└─────────────┘                  └───┬───┬───┬───┘
                                     │   │   │
                    ┌────────────────┘   │   └────────────────┐
                    ▼                    ▼                    ▼
             ┌────────────┐      ┌────────────┐       ┌────────────┐
             │ Payment    │      │ Inventory  │       │ Notification│
             │ Service    │      │ Service    │       │ Service     │
             └────────────┘      └────────────┘       └─────────────┘
```

- ✅ Services are loosely coupled — adding a new consumer requires no changes to the producer
- ✅ Natural fit for scaling — each consumer scales independently
- ❌ Debugging is harder — event flows are implicit, not explicit call chains
- ❌ Eventual consistency — downstream services process events asynchronously

## Scaling Patterns

These patterns protect your system from being overwhelmed and help maintain stability under load.

### Throttling (Rate Limiting)

Throttling limits the number of requests a client or service can make within a time window. It prevents any single consumer from monopolizing resources.

| Algorithm | How It Works | Characteristics |
|---|---|---|
| **Fixed window** | Count requests in fixed time intervals (e.g., 100/minute) | Simple; burst at window boundaries |
| **Sliding window** | Rolling window smooths out boundary bursts | More accurate; slightly more complex |
| **Token bucket** | Tokens added at a steady rate; each request costs a token | Allows controlled bursts |
| **Leaky bucket** | Requests enter a queue processed at a fixed rate | Smooths output regardless of input burstiness |

### Backpressure

Backpressure is a mechanism where a downstream system signals to upstream systems that it is at capacity, causing producers to slow down rather than overwhelming the consumer.

```
Normal Flow:
Producer ──── 1000 msg/s ────► Consumer (capacity: 1000 msg/s)  ✅

Without Backpressure:
Producer ──── 5000 msg/s ────► Consumer (capacity: 1000 msg/s)  ❌ OOM / crash

With Backpressure:
Producer ──── 5000 msg/s ────► Consumer signals "slow down"
Producer ──── 1000 msg/s ────► Consumer (capacity: 1000 msg/s)  ✅
```

Backpressure strategies:

- **Blocking** — Producer blocks until consumer is ready (TCP flow control)
- **Dropping** — Discard excess messages when buffer is full (acceptable for metrics)
- **Buffering** — Accumulate messages in a bounded queue; reject when full
- **Scaling consumers** — Spin up more consumers when queue depth grows

### Circuit Breakers

A circuit breaker prevents cascading failures by stopping requests to a failing downstream service and returning a fallback response.

```
┌─────────┐       ┌───────────────┐       ┌─────────────┐
│ Caller  │──────►│Circuit Breaker│──────►│ Downstream  │
│         │       │               │       │ Service     │
└─────────┘       └───────┬───────┘       └─────────────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
         ┌────────┐ ┌─────────┐ ┌─────────┐
         │ CLOSED │ │  OPEN   │ │HALF-OPEN│
         │        │ │         │ │         │
         │Requests│ │All calls│ │Allow a  │
         │flow    │ │fail fast│ │few test  │
         │normally│ │(fallback│ │requests │
         │        │ │response)│ │         │
         └────────┘ └─────────┘ └─────────┘
```

| State | Behavior |
|---|---|
| **Closed** | Requests pass through normally; failures are counted |
| **Open** | All requests fail immediately with a fallback; timer starts |
| **Half-Open** | A limited number of test requests are allowed through to check recovery |

## Capacity Planning

Capacity planning is the process of estimating the resources your system needs to handle current and future load.

### Estimating Load

Start with back-of-envelope calculations to understand the order of magnitude.

**Example: Social media photo service**

```
Given:
  - 10 million daily active users (DAU)
  - Each user uploads 2 photos/day on average
  - Each user views 50 photos/day on average
  - Average photo size: 500 KB

Write load:
  - 10M × 2 = 20M uploads/day
  - 20M / 86,400 ≈ 230 uploads/second

Read load:
  - 10M × 50 = 500M views/day
  - 500M / 86,400 ≈ 5,800 reads/second

Storage (per year):
  - 20M × 500 KB × 365 = ~3.3 PB/year

Bandwidth:
  - Writes: 230 × 500 KB ≈ 115 MB/s ingress
  - Reads: 5,800 × 500 KB ≈ 2.9 GB/s egress
```

### Growth Projections

| Planning Horizon | Multiplier | Purpose |
|---|---|---|
| **Current** | 1× | Baseline resource requirements |
| **6 months** | 1.5–2× | Next infrastructure procurement cycle |
| **1 year** | 2–3× | Annual capacity review |
| **3 years** | 5–10× | Architecture decisions and technology choices |

### Capacity Planning Checklist

- ✅ Identify the system's bottleneck (CPU, memory, disk I/O, network)
- ✅ Measure current utilization and set target thresholds (e.g., 70% CPU)
- ✅ Model read vs write ratios and how they change with growth
- ✅ Account for traffic spikes (2–5× average for most applications)
- ✅ Plan for failure scenarios — can you handle peak load with N-1 nodes?
- ✅ Include storage growth and data retention policies
- ✅ Document assumptions and revisit quarterly

## Next Steps

Continue to [Distributed Systems](02-DISTRIBUTED-SYSTEMS.md) to learn about consensus algorithms, consistency models, and distributed transaction strategies. For deeper coverage of specific scaling components, see:

- **[03-CACHING-STRATEGIES.md](03-CACHING-STRATEGIES.md)** — Cache-aside, write-through, CDN caching, cache invalidation
- **[04-LOAD-BALANCING.md](04-LOAD-BALANCING.md)** — Algorithms, Layer 4 vs Layer 7, health checks, advanced configurations
- **[05-DATA-PARTITIONING.md](05-DATA-PARTITIONING.md)** — Sharding strategies, consistent hashing, rebalancing

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial scalability documentation |
