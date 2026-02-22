# System Design Fundamentals

## Table of Contents

1. [Overview](#overview)
2. [What Is System Design](#what-is-system-design)
3. [Functional vs. Non-Functional Requirements](#functional-vs-non-functional-requirements)
4. [Key Trade-offs](#key-trade-offs)
5. [The System Design Process](#the-system-design-process)
6. [Core Building Blocks](#core-building-blocks)
7. [Back-of-Envelope Estimation](#back-of-envelope-estimation)
8. [When to Scale](#when-to-scale)
9. [Prerequisites](#prerequisites)
10. [Next Steps](#next-steps)

## Overview

This documentation provides a comprehensive introduction to system design — the discipline of defining the architecture, components, modules, interfaces, and data flows of a system to satisfy specified requirements. System design is a critical skill for building reliable, scalable, and maintainable software that serves millions of users.

### Target Audience

- Software engineers preparing for system design interviews
- Backend and full-stack developers building large-scale applications
- Software architects designing distributed systems
- Engineering managers evaluating technical proposals and trade-offs
- DevOps engineers planning infrastructure for growth

### Scope

- What system design is and why it matters
- How to gather and classify requirements
- Key trade-offs that shape every architectural decision
- A step-by-step process for designing systems
- Core building blocks used in distributed architectures
- Estimation techniques for capacity planning
- Signs that indicate when and how to scale

## What Is System Design

System design is the process of defining the architecture and components of a system to meet functional and non-functional requirements. It bridges the gap between a product idea and a working, production-ready implementation that can handle real-world traffic, failures, and growth.

### Why System Design Matters

| Reason | Description |
|---|---|
| **Scalability** | Systems must handle growing users, data, and traffic without degradation |
| **Reliability** | Downtime costs money and trust — design must minimize single points of failure |
| **Maintainability** | Engineers must be able to understand, modify, and extend the system over time |
| **Performance** | Users expect fast responses — poor latency drives users away |
| **Cost efficiency** | Over-engineering wastes resources; under-engineering causes outages |
| **Team velocity** | Good architecture enables teams to work independently and ship faster |

### The System Design Landscape

```
┌──────────────────────────────────────────────────────────────────────┐
│                        System Design                                 │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │   Requirements   │  │   High-Level     │  │   Detailed       │   │
│  │   Analysis       │  │   Architecture   │  │   Component      │   │
│  │                  │  │                  │  │   Design         │   │
│  │  • Functional    │  │  • Components    │  │  • Data models   │   │
│  │  • Non-functional│  │  • Data flow     │  │  • APIs          │   │
│  │  • Constraints   │  │  • Protocols     │  │  • Algorithms    │   │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘   │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │   Scaling &      │  │   Reliability &  │  │   Monitoring &   │   │
│  │   Performance    │  │   Fault          │  │   Observability  │   │
│  │                  │  │   Tolerance      │  │                  │   │
│  │  • Horizontal    │  │  • Replication   │  │  • Metrics       │   │
│  │  • Vertical      │  │  • Redundancy    │  │  • Logging       │   │
│  │  • Caching       │  │  • Failover      │  │  • Alerting      │   │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

## Functional vs. Non-Functional Requirements

Every system design begins with clearly defining what the system must do (functional) and how well it must do it (non-functional). Confusing or omitting either category leads to systems that are incomplete or impossible to operate at scale.

### Functional Requirements

Functional requirements describe **what** the system does — the features and behaviors visible to users.

| Example System | Functional Requirements |
|---|---|
| **URL shortener** | Create short URL, redirect to original URL, track click counts |
| **Chat application** | Send messages, create groups, show online status, deliver notifications |
| **News feed** | Publish posts, follow users, generate personalized feed, like/comment |
| **File storage** | Upload files, download files, share with permissions, version history |
| **Rate limiter** | Track request counts, enforce limits per user/IP, return 429 responses |

### Non-Functional Requirements

Non-functional requirements describe **how well** the system performs — the quality attributes.

| Attribute | Description | Typical Target |
|---|---|---|
| **Availability** | Percentage of time the system is operational | 99.9% – 99.999% |
| **Latency** | Time to respond to a request | p50 < 100ms, p99 < 500ms |
| **Throughput** | Number of requests handled per second | 10K – 1M+ RPS |
| **Durability** | Guarantee that stored data is not lost | 99.999999999% (11 nines) |
| **Consistency** | All users see the same data at the same time | Strong or eventual |
| **Scalability** | Ability to handle increased load | 10x growth in 2 years |
| **Fault tolerance** | System continues operating despite failures | Survive AZ/region failure |
| **Security** | Protection against unauthorized access | Encryption at rest/in transit |

### Availability in Context

```
Availability     Downtime/Year     Downtime/Month     Downtime/Week
─────────────    ──────────────    ──────────────     ─────────────
99%              3.65 days         7.31 hours         1.68 hours
99.9%            8.77 hours        43.83 minutes      10.08 minutes
99.99%           52.6 minutes      4.38 minutes       1.01 minutes
99.999%          5.26 minutes      26.3 seconds       6.05 seconds
99.9999%         31.56 seconds     2.63 seconds       0.60 seconds
```

## Key Trade-offs

System design is fundamentally about making trade-offs. There is no perfect architecture — every decision favors one quality at the expense of another. Understanding these trade-offs is what separates good designs from poor ones.

### CAP Theorem

The CAP theorem states that a distributed data store can provide only two of the following three guarantees simultaneously:

```
                    Consistency (C)
                         ╱╲
                        ╱  ╲
                       ╱    ╲
                      ╱  CP  ╲
                     ╱  Systems╲
                    ╱          ╲
                   ╱────────────╲
                  ╱              ╲
                 ╱   You can't    ╲
                ╱   have all 3     ╲
               ╱                    ╲
              ╱──────────────────────╲
             ╱  AP          CA        ╲
            ╱  Systems     Systems     ╲
           ╱              (single      ╲
          ╱                node)        ╲
         ╱──────────────────────────────╲
   Availability (A)              Partition Tolerance (P)
```

| System Type | Guarantees | Sacrifices | Examples |
|---|---|---|---|
| **CP** | Consistency + Partition Tolerance | Availability during partitions | MongoDB, HBase, Redis Cluster |
| **AP** | Availability + Partition Tolerance | Strong consistency | Cassandra, DynamoDB, CouchDB |
| **CA** | Consistency + Availability | Partition tolerance (single node only) | Traditional RDBMS (PostgreSQL, MySQL) |

> **Note:** In practice, network partitions are inevitable in distributed systems. The real choice is between CP and AP — do you prefer consistency or availability when a partition occurs?

### Consistency vs. Availability

| Dimension | Strong Consistency | Eventual Consistency |
|---|---|---|
| **Guarantee** | All reads return the most recent write | Reads may return stale data temporarily |
| **Latency** | Higher — requires coordination | Lower — no coordination needed |
| **Availability** | Lower during network issues | Higher — serves from any replica |
| **Use cases** | Banking, inventory, booking systems | Social feeds, analytics, DNS |
| **Protocols** | Paxos, Raft, two-phase commit | Gossip, anti-entropy, read repair |

### Latency vs. Throughput

```
  High ┌──────────────────────────────────────┐
       │                                      │
       │  Optimize for         Sweet spot     │
  T    │  throughput ──►      ┌───┐           │
  h    │  (batch processing)  │ ◆ │           │
  r    │                      └───┘           │
  o    │                                      │
  u    │                         Optimize for │
  g    │                         ◄── latency  │
  h    │                    (real-time apps)   │
  p    │                                      │
  u    │                                      │
  t    │                                      │
       │                                      │
  Low  └──────────────────────────────────────┘
       Low                                High
                     Latency
```

| Scenario | Optimize For | Approach |
|---|---|---|
| **Real-time gaming** | Latency | In-memory state, UDP, edge servers |
| **Video encoding** | Throughput | Batch jobs, parallel workers, queues |
| **E-commerce checkout** | Latency | Caching, read replicas, CDN |
| **Data warehouse ETL** | Throughput | Bulk inserts, partitioned processing |
| **Chat messaging** | Both | WebSockets, message queues, fan-out |

### Cost vs. Complexity

| Approach | Cost | Complexity | When to Use |
|---|---|---|---|
| **Single server** | $ | Low | Prototypes, MVPs, < 1K users |
| **Vertical scaling** | $$ | Low | Quick wins, predictable growth |
| **Horizontal scaling** | $$$ | Medium | High traffic, need redundancy |
| **Multi-region** | $$$$ | High | Global users, regulatory compliance |
| **Multi-cloud** | $$$$$ | Very high | Vendor lock-in avoidance, max resilience |

## The System Design Process

A structured approach ensures that you cover all critical aspects of a design. Whether in an interview or a real project, follow these steps to produce a well-reasoned architecture.

### Step-by-Step Approach

```
┌─────────────────────────────────────────────────────────────────┐
│                  The System Design Process                       │
│                                                                  │
│  Step 1          Step 2          Step 3          Step 4          │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│  │Require-  │──►│High-Level│──►│Deep      │──►│Bottleneck│    │
│  │ments     │   │Design    │   │Dive      │   │& Scaling │    │
│  │Gathering │   │          │   │          │   │          │    │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘    │
│                                                                  │
│  • Clarify       • Identify     • Data models   • Single points │
│    functional      components   • API design       of failure   │
│    requirements  • Draw data    • Algorithms    • Capacity      │
│  • Define non-     flows       • Storage          planning      │
│    functional    • Choose        choices        • Caching       │
│    requirements    protocols   • Indexing         strategy      │
│  • Estimate      • Define        strategy       • Monitoring    │
│    scale           APIs                           plan          │
└─────────────────────────────────────────────────────────────────┘
```

### Step 1: Requirements Gathering

Before drawing a single box, clarify what you are building and the constraints.

| Question Category | Example Questions |
|---|---|
| **Users** | How many users? DAU/MAU? Geographic distribution? |
| **Features** | What are the core use cases? What can be deferred? |
| **Scale** | How many reads/writes per second? Data volume? |
| **Performance** | What latency is acceptable? What is the SLA? |
| **Consistency** | Is eventual consistency acceptable? |
| **Availability** | What uptime is required? Can we tolerate brief outages? |
| **Constraints** | Budget? Existing tech stack? Regulatory requirements? |

### Step 2: High-Level Design

Sketch the major components and their interactions.

```
┌────────┐      ┌───────────┐      ┌──────────────┐
│ Client │─────►│ Load      │─────►│ Application  │
│        │      │ Balancer  │      │ Servers      │
└────────┘      └───────────┘      └──────┬───────┘
                                          │
                    ┌─────────────────────┼──────────────────┐
                    │                     │                   │
              ┌─────▼─────┐        ┌─────▼─────┐     ┌──────▼──────┐
              │  Cache     │        │ Database  │     │ Message     │
              │  (Redis)   │        │ (Primary) │     │ Queue       │
              └────────────┘        └─────┬─────┘     └─────────────┘
                                          │
                                    ┌─────▼─────┐
                                    │ Database  │
                                    │ (Replica) │
                                    └───────────┘
```

### Step 3: Deep Dive

Pick the most critical components and design them in detail.

| Component | Design Decisions |
|---|---|
| **Data model** | Schema design, partitioning key, indexing strategy |
| **API design** | Endpoints, request/response format, pagination, rate limits |
| **Storage** | SQL vs. NoSQL, replication strategy, backup and recovery |
| **Caching** | Cache-aside vs. write-through, TTL, eviction policy |
| **Search** | Full-text search engine, indexing pipeline, ranking |
| **Notifications** | Push vs. pull, delivery guarantees, fan-out strategy |

### Step 4: Bottleneck Identification

Identify weaknesses and propose solutions.

```
Bottleneck                          Solution
────────────────────────────────    ────────────────────────────────
Single database under heavy load    Read replicas, sharding, caching
Slow API responses                  Caching, async processing, CDN
Single point of failure             Redundancy, failover, multi-AZ
Hot partitions in database          Better partition key, consistent hashing
Large payload transfers             Compression, pagination, streaming
Thundering herd on cache miss       Cache stampede protection, locking
```

## Core Building Blocks

Every large-scale system is composed of a common set of building blocks. Understanding each component's role and trade-offs is essential for effective system design.

### Building Block Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                     System Design Building Blocks                     │
│                                                                       │
│   ┌─────────┐                                                        │
│   │  Users  │                                                        │
│   └────┬────┘                                                        │
│        │                                                              │
│        ▼                                                              │
│   ┌─────────┐     ┌──────────────┐                                   │
│   │  DNS    │────►│  CDN         │  (static assets, edge caching)    │
│   └────┬────┘     └──────────────┘                                   │
│        │                                                              │
│        ▼                                                              │
│   ┌──────────────┐                                                   │
│   │ Load Balancer│  (distributes traffic across servers)             │
│   └──────┬───────┘                                                   │
│          │                                                            │
│    ┌─────┼─────────────────────┐                                     │
│    │     │                     │                                      │
│    ▼     ▼                     ▼                                      │
│  ┌────┐┌────┐┌────┐    ┌─────────────┐                               │
│  │ S1 ││ S2 ││ S3 │    │ API Gateway │  (auth, rate limit, routing) │
│  └──┬─┘└──┬─┘└──┬─┘    └─────────────┘                               │
│     │     │     │                                                     │
│     └─────┼─────┘                                                     │
│           │                                                           │
│     ┌─────┼──────────────────────────────┐                           │
│     │     │              │               │                            │
│     ▼     ▼              ▼               ▼                            │
│  ┌──────┐ ┌──────────┐ ┌───────────┐ ┌──────────────┐               │
│  │Cache │ │ Database │ │ Message   │ │ Object       │               │
│  │      │ │          │ │ Queue     │ │ Storage      │               │
│  │Redis │ │SQL/NoSQL │ │Kafka/SQS │ │ S3/Blob      │               │
│  └──────┘ └──────────┘ └───────────┘ └──────────────┘               │
└──────────────────────────────────────────────────────────────────────┘
```

### Component Reference

| Building Block | Purpose | Examples |
|---|---|---|
| **DNS** | Translates domain names to IP addresses | Route 53, Cloudflare DNS, Google Cloud DNS |
| **CDN** | Caches static content at edge locations close to users | CloudFront, Akamai, Fastly, Cloudflare |
| **Load Balancer** | Distributes incoming traffic across multiple servers | NGINX, HAProxy, AWS ALB/NLB, Envoy |
| **API Gateway** | Single entry point for API traffic — auth, rate limiting, routing | Kong, AWS API Gateway, Apigee |
| **Application Servers** | Run business logic, process requests, return responses | Containers (Docker/K8s), VMs, serverless |
| **Cache** | Store frequently accessed data in memory for fast reads | Redis, Memcached, Varnish |
| **Database (SQL)** | Structured data with ACID transactions and strong consistency | PostgreSQL, MySQL, SQL Server |
| **Database (NoSQL)** | Flexible schemas, horizontal scaling, high throughput | MongoDB, Cassandra, DynamoDB |
| **Message Queue** | Asynchronous communication and workload decoupling | Kafka, RabbitMQ, SQS, Pulsar |
| **Object Storage** | Store large binary objects (images, videos, backups) | S3, Azure Blob Storage, GCS |
| **Search Engine** | Full-text search and complex query capabilities | Elasticsearch, OpenSearch, Solr |
| **Coordination Service** | Distributed locking, leader election, configuration | ZooKeeper, etcd, Consul |

### Caching Strategies

| Strategy | How It Works | Pros | Cons |
|---|---|---|---|
| **Cache-aside** | App reads cache; on miss, reads DB and populates cache | Simple, app controls cache | Cache miss penalty, stale data risk |
| **Read-through** | Cache itself fetches from DB on miss | Transparent to app | Cache library dependency |
| **Write-through** | App writes to cache, cache writes to DB synchronously | Strong consistency | Higher write latency |
| **Write-behind** | App writes to cache, cache writes to DB asynchronously | Low write latency | Data loss risk on cache failure |
| **Write-around** | App writes directly to DB, cache is populated on reads | Avoids cache churn | Cache miss on recent writes |

```
Cache-Aside Pattern:

  ┌────────┐   1. Read    ┌────────┐
  │  App   │─────────────►│ Cache  │
  │ Server │◄─────────────│(Redis) │
  └───┬────┘   2. Hit?    └────────┘
      │            │
      │   3. Miss  │
      ▼            │
  ┌────────┐       │
  │Database│       │
  └───┬────┘       │
      │            │
      └────────────┘
       4. Populate cache
```

### Database Scaling Patterns

```
Vertical Scaling              Read Replicas               Sharding
(Scale Up)                    (Read Scale-Out)            (Write Scale-Out)

┌──────────┐                 ┌──────────┐               ┌────┐ ┌────┐ ┌────┐
│          │                 │ Primary  │               │Sh-1│ │Sh-2│ │Sh-3│
│  Bigger  │                 │  (Write) │               │A-H │ │I-P │ │Q-Z │
│  Server  │                 └────┬─────┘               └────┘ └────┘ └────┘
│          │                      │
│ More CPU │                 ┌────┼────┐                Data is partitioned
│ More RAM │                 ▼    ▼    ▼                across shards by key
│ More Disk│              ┌────┐┌────┐┌────┐
│          │              │ R1 ││ R2 ││ R3 │
└──────────┘              └────┘└────┘└────┘
Simple but has            Scales reads;               Scales reads and writes;
hardware limits           writes still go to primary  complex to implement
```

## Back-of-Envelope Estimation

Quick estimation skills help you make informed decisions about architecture, capacity, and cost. Memorize the key numbers below — they form the foundation for any capacity planning exercise.

### Latency Numbers Every Engineer Should Know

```
Operation                                  Time
─────────────────────────────────────────  ──────────────
L1 cache reference                         0.5 ns
Branch mispredict                          5 ns
L2 cache reference                         7 ns
Mutex lock/unlock                          25 ns
Main memory reference                      100 ns
Compress 1KB with Zippy                    3,000 ns       (3 µs)
Send 1KB over 1 Gbps network              10,000 ns      (10 µs)
Read 4KB randomly from SSD                150,000 ns      (150 µs)
Read 1MB sequentially from memory         250,000 ns      (250 µs)
Round trip within same datacenter          500,000 ns      (500 µs)
Read 1MB sequentially from SSD            1,000,000 ns    (1 ms)
HDD seek                                  10,000,000 ns   (10 ms)
Read 1MB sequentially from HDD            20,000,000 ns   (20 ms)
Send packet CA → Netherlands → CA         150,000,000 ns  (150 ms)
```

### Power-of-Two Reference

| Power | Exact Value | Approximate Size | Common Usage |
|---|---|---|---|
| 10 | 1,024 | 1 Thousand | 1 KB |
| 20 | 1,048,576 | 1 Million | 1 MB |
| 30 | 1,073,741,824 | 1 Billion | 1 GB |
| 40 | 1,099,511,627,776 | 1 Trillion | 1 TB |
| 50 | ~1.13 × 10¹⁵ | 1 Quadrillion | 1 PB |

### Common Throughput Estimates

| Component | Typical Throughput |
|---|---|
| **Single web server** | 1K – 10K requests/second |
| **Redis (single node)** | 100K – 200K operations/second |
| **PostgreSQL (single node)** | 5K – 20K queries/second |
| **Kafka (single broker)** | 100K – 200K messages/second |
| **S3 PUT requests** | 3,500 requests/second per prefix |
| **S3 GET requests** | 5,500 requests/second per prefix |

### Quick Estimation Framework

Use this template to estimate capacity for any system:

```
1. Estimate users and activity
   ─────────────────────────────────────────
   DAU (Daily Active Users)        = ?
   Actions per user per day        = ?
   Total actions per day           = DAU × actions/user
   Actions per second (avg)        = total actions / 86,400
   Peak actions per second         = avg × 2–5 (peak factor)

2. Estimate storage
   ─────────────────────────────────────────
   Data per action                 = ? bytes
   Daily new data                  = actions/day × bytes/action
   Annual storage                  = daily new data × 365
   5-year storage                  = annual × 5

3. Estimate bandwidth
   ─────────────────────────────────────────
   Incoming (write) bandwidth      = actions/sec × data/action
   Outgoing (read) bandwidth       = read QPS × response size
   Read-to-write ratio             = typically 10:1 to 100:1
```

### Example: URL Shortener Estimation

```
Given:
  100M new URLs per month
  10:1 read-to-write ratio

Writes:
  100M / (30 × 86,400) ≈ 40 writes/second

Reads:
  40 × 10 = 400 reads/second

Storage (per URL entry ≈ 500 bytes):
  100M × 500 bytes = 50 GB/month
  50 GB × 12 months × 5 years = 3 TB (5-year storage)

Bandwidth:
  Write: 40 × 500 bytes = 20 KB/s
  Read:  400 × 500 bytes = 200 KB/s
```

## When to Scale

Scaling too early wastes resources. Scaling too late causes outages. Use these signals to guide scaling decisions.

### Scaling Triggers

```
Monitor These Signals:
├── CPU utilization consistently > 70%
├── Memory usage approaching capacity
├── Response latency exceeding SLA (p99 > threshold)
├── Request queue depth growing over time
├── Database connection pool exhaustion
├── Disk I/O wait times increasing
├── Error rate (5xx responses) trending upward
└── Successful request throughput plateauing under load
```

### Vertical vs. Horizontal Scaling

| Dimension | Vertical Scaling (Scale Up) | Horizontal Scaling (Scale Out) |
|---|---|---|
| **Approach** | Add more CPU, RAM, or disk to a single machine | Add more machines to the pool |
| **Complexity** | Low — no code changes needed | Higher — requires stateless design |
| **Limit** | Hardware ceiling (biggest machine available) | Near-limitless (add more nodes) |
| **Downtime** | Often requires restart | Zero downtime with rolling deploys |
| **Cost curve** | Exponential (bigger = disproportionately expensive) | Linear (more machines = proportional cost) |
| **Fault tolerance** | Single point of failure | Built-in redundancy |
| **Best for** | Databases, legacy apps, quick fixes | Stateless web/app servers, microservices |

### Scaling Strategies by Component

| Component | Scaling Strategy |
|---|---|
| **Web servers** | Horizontal scaling behind a load balancer |
| **Application logic** | Stateless services, container orchestration (Kubernetes) |
| **Database reads** | Read replicas, caching layer (Redis), CDN for static data |
| **Database writes** | Sharding, write-ahead log optimization, CQRS |
| **File storage** | Object storage (S3), CDN for delivery |
| **Search** | Elasticsearch cluster with index sharding |
| **Real-time messaging** | Kafka partitions, consumer groups |
| **Background jobs** | Worker pools, queue-based processing |

### The Scaling Ladder

```
Stage 1: Single Server
  App + DB on one machine
  Users: < 1,000

Stage 2: Separate DB
  App server ←→ DB server
  Users: 1K – 10K

Stage 3: Add Cache + LB
  Load Balancer → App servers (×N) → Cache → DB
  Users: 10K – 100K

Stage 4: Read Replicas + CDN
  CDN + LB → App servers → Cache → Primary DB + Replicas
  Users: 100K – 1M

Stage 5: Sharding + Message Queues
  CDN + LB → App servers → Cache → Sharded DBs + Queues + Workers
  Users: 1M – 10M

Stage 6: Multi-Region
  Global LB → Regional clusters (each with full stack)
  Users: 10M+
```

## Prerequisites

### Required Knowledge

- **Programming fundamentals** — at least one backend language (Python, Java, Go, C#, etc.)
- **Networking basics** — TCP/IP, HTTP/HTTPS, DNS, sockets, REST APIs
- **Database fundamentals** — SQL, indexing, normalization, transactions, ACID properties
- **Operating systems** — processes, threads, memory management, file systems
- **Distributed systems concepts** — CAP theorem, consistency models, consensus algorithms

### Recommended Tools

| Tool | Purpose |
|---|---|
| Draw.io / Excalidraw | Architecture diagram sketching |
| Redis | In-memory caching and data structures |
| PostgreSQL / MySQL | Relational database for ACID workloads |
| MongoDB / DynamoDB | NoSQL database for flexible schemas |
| Kafka / RabbitMQ | Message brokers for async processing |
| NGINX / HAProxy | Load balancing and reverse proxying |
| Docker / Kubernetes | Containerization and orchestration |
| Prometheus / Grafana | Metrics collection and dashboarding |

## Next Steps

Continue to [Scalability](01-SCALABILITY.md) to learn about horizontal and vertical scaling, load balancing algorithms, and CDN fundamentals.

### Suggested Learning Path

1. **[Scalability](01-SCALABILITY.md)** — Horizontal/vertical scaling, load balancing, CDNs
2. **[Distributed Systems](02-DISTRIBUTED-SYSTEMS.md)** — Consensus, consistency models, distributed transactions
3. **[Caching Strategies](03-CACHING-STRATEGIES.md)** — Cache-aside, write-through, CDN caching, cache invalidation
4. **[Load Balancing](04-LOAD-BALANCING.md)** — Algorithms, Layer 4 vs Layer 7, health checks
5. **[Data Partitioning](05-DATA-PARTITIONING.md)** — Sharding strategies, consistent hashing, rebalancing
6. **[Reliability](06-RELIABILITY.md)** — Redundancy, failover, disaster recovery, chaos engineering
7. **[Common Designs](07-COMMON-DESIGNS.md)** — URL shortener, chat system, notification service, rate limiter
8. **[Best Practices](08-BEST-PRACTICES.md)** — Design process, capacity estimation, back-of-envelope math
9. **[Anti-Patterns](09-ANTI-PATTERNS.md)** — Common system design mistakes and how to avoid them

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial system design fundamentals documentation |
