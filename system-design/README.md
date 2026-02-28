# System Design Learning Resources

A comprehensive guide to architecture patterns, scalability, and distributed systems — ties together multiple topics into cohesive, production-ready designs.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | System design fundamentals, trade-offs, requirements analysis | **Start here** |
| [01-SCALABILITY](01-SCALABILITY.md) | Horizontal/vertical scaling, load balancing, CDNs | When planning for growth |
| [02-DISTRIBUTED-SYSTEMS](02-DISTRIBUTED-SYSTEMS.md) | Consensus, consistency models, distributed transactions | When building distributed systems |
| [03-CACHING-STRATEGIES](03-CACHING-STRATEGIES.md) | Cache-aside, write-through, CDN caching, cache invalidation | When optimizing performance |
| [04-LOAD-BALANCING](04-LOAD-BALANCING.md) | Algorithms, Layer 4 vs Layer 7, health checks | When distributing traffic |
| [05-DATA-PARTITIONING](05-DATA-PARTITIONING.md) | Sharding strategies, consistent hashing, rebalancing | When scaling data storage |
| [06-RELIABILITY](06-RELIABILITY.md) | Redundancy, failover, disaster recovery, chaos engineering | When building resilient systems |
| [07-COMMON-DESIGNS](07-COMMON-DESIGNS.md) | URL shortener, chat system, notification service, rate limiter | When practicing system design |
| [08-BEST-PRACTICES](08-BEST-PRACTICES.md) | Design process, capacity estimation, back-of-envelope math | **Essential — design checklist** |
| [09-ANTI-PATTERNS](09-ANTI-PATTERNS.md) | Common system design mistakes and how to avoid them | **Essential — what NOT to do** |
| [LEARNING-PATH](LEARNING-PATH.md) | Structured learning guide with exercises | **Start here** after the Overview |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Understand system design fundamentals
   - Learn about functional vs. non-functional requirements
   - Explore key trade-offs (CAP theorem, latency vs. throughput)

2. **Learn Scalability** ([01-SCALABILITY](01-SCALABILITY.md))
   - Horizontal vs. vertical scaling
   - Load balancing and CDN fundamentals
   - Database replication strategies

3. **Understand Caching** ([03-CACHING-STRATEGIES](03-CACHING-STRATEGIES.md))
   - Cache-aside and write-through patterns
   - CDN caching and cache invalidation
   - When and where to cache

4. **Follow the Learning Path** ([LEARNING-PATH](LEARNING-PATH.md))
   - Structured, phased curriculum
   - Hands-on exercises and knowledge checks

### For Experienced Users

1. **Review Best Practices** ([08-BEST-PRACTICES](08-BEST-PRACTICES.md))
   - Design process and estimation framework
   - Capacity planning techniques
   - Back-of-envelope calculations

2. **Avoid Anti-Patterns** ([09-ANTI-PATTERNS](09-ANTI-PATTERNS.md))
   - Over-engineering pitfalls
   - Premature optimization traps
   - Common architectural mistakes

3. **Deep Dive into Distributed Systems** ([02-DISTRIBUTED-SYSTEMS](02-DISTRIBUTED-SYSTEMS.md))
   - Consensus algorithms (Paxos, Raft)
   - Consistency models and trade-offs
   - Distributed transaction strategies

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        Clients                              │
│              (Web, Mobile, Third-Party)                     │
└──────────────────────┬──────────────────────────────────────┘
                       │
                ┌──────▼──────┐
                │     CDN     │
                │  (static    │
                │   assets)   │
                └──────┬──────┘
                       │
                ┌──────▼──────┐
                │    Load     │
                │  Balancer   │
                └──┬───┬───┬──┘
         ┌─────────┘   │   └──────────┐
         ▼             ▼              ▼
   ┌───────────┐ ┌───────────┐ ┌───────────┐
   │   App     │ │   App     │ │   App     │
   │ Server 1  │ │ Server 2  │ │ Server N  │
   └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
         │             │              │
         └──────┬──────┴──────┬───────┘
                │             │
         ┌──────▼──────┐ ┌───▼────────┐
         │   Cache     │ │  Message   │
         │   Layer     │ │  Queue     │
         │  (Redis,    │ │  (Kafka,   │
         │  Memcached) │ │  RabbitMQ) │
         └──────┬──────┘ └───┬────────┘
                │             │
         ┌──────▼─────────────▼───────┐
         │       Data Layer           │
         │  ┌────────┐  ┌──────────┐  │
         │  │Primary │  │ Replicas │  │
         │  │  DB    │  │  (Read)  │  │
         │  └────────┘  └──────────┘  │
         └────────────────────────────┘
```

## 🔑 Key Concepts

### Core Principles

```
System Design
├── Scalability              (handle growing traffic and data)
├── Reliability              (continue operating under failures)
├── Availability             (uptime and responsiveness)
├── Performance              (low latency, high throughput)
├── Maintainability          (easy to modify and extend)
└── Cost Efficiency          (optimize resource utilization)
```

### Design Trade-offs

```
Consistency vs. Availability         Latency vs. Throughput
├── Strong Consistency               ├── Caching (reduce latency)
├── Eventual Consistency             ├── Batching (increase throughput)
└── CAP Theorem                      └── Async Processing (decouple)
```

## 📋 Topics Covered

- **Fundamentals** — System design process, trade-offs, requirements analysis
- **Scalability** — Horizontal/vertical scaling, load balancing, CDNs
- **Distributed Systems** — Consensus, consistency models, distributed transactions
- **Caching** — Cache-aside, write-through, CDN caching, cache invalidation
- **Load Balancing** — Algorithms, Layer 4 vs Layer 7, health checks
- **Data Partitioning** — Sharding strategies, consistent hashing, rebalancing
- **Reliability** — Redundancy, failover, disaster recovery, chaos engineering
- **Common Designs** — URL shortener, chat system, notification service, rate limiter
- **Best Practices** — Design process, capacity estimation, back-of-envelope math
- **Anti-Patterns** — Common system design mistakes and how to avoid them

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to system design?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md)

**Preparing for interviews?** → Practice with [07-COMMON-DESIGNS.md](07-COMMON-DESIGNS.md) and [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md)

**Building production systems?** → Review [06-RELIABILITY.md](06-RELIABILITY.md) and [09-ANTI-PATTERNS.md](09-ANTI-PATTERNS.md)

**Want a structured path?** → Follow the [Learning Path](LEARNING-PATH.md)
