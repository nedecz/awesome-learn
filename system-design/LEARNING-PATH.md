# System Design Learning Path

A structured, self-paced training guide to mastering system design — from fundamentals and scalability to distributed systems and real-world designs. Each phase builds on the previous one, progressing from core concepts to advanced patterns.

> **Time Estimate:** 10–12 weeks at ~5 hours/week. Adjust pace to your experience level.

---

## How to Use This Guide

1. **Follow the phases in order** — each one builds on prior knowledge
2. **Read the linked documents** — they contain the detailed content
3. **Complete the exercises** — hands-on practice solidifies understanding
4. **Check yourself** — use the knowledge checks before moving on
5. **Build the capstone project** — ties everything together

---

## Phase 1: Foundations (Week 1–2)

### Learning Objectives

- Understand what system design is and why it matters
- Learn how to gather and clarify functional and non-functional requirements
- Grasp the fundamentals of vertical and horizontal scaling
- Perform back-of-envelope estimation for capacity planning
- Recognize the trade-offs between latency, throughput, and availability

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | Core concepts, design process, requirements gathering, trade-offs |
| 2 | [01-SCALABILITY](01-SCALABILITY.md) | Vertical vs. horizontal scaling, stateless design, capacity planning |

### Exercises

**1. Requirements Analysis:**

Choose a well-known application (e.g., Twitter, Uber, Netflix) and:

- List 5–8 functional requirements (what the system does)
- List 5–8 non-functional requirements (performance, reliability, security)
- Identify the top 3 constraints (budget, latency, data volume)
- Prioritize the requirements using MoSCoW (Must, Should, Could, Won't)

**2. Back-of-Envelope Estimation:**

For a system that serves 100 million daily active users:

- Estimate the number of requests per second (peak and average)
- Calculate the storage needed for 1 year of data
- Estimate the required network bandwidth
- Document your assumptions clearly

### Knowledge Check

- [ ] What is the difference between functional and non-functional requirements?
- [ ] When is vertical scaling preferred over horizontal scaling?
- [ ] What does "stateless design" mean and why does it enable horizontal scaling?
- [ ] How do you estimate the storage requirements for a system serving millions of users?
- [ ] What is the relationship between latency and throughput?

---

## Phase 2: Distributed Systems & Caching (Week 3–4)

### Learning Objectives

- Understand the CAP theorem and its practical implications
- Learn about consistency models: strong, eventual, causal
- Grasp distributed consensus and coordination challenges
- Design effective caching strategies at multiple layers
- Evaluate cache eviction policies and invalidation approaches

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [02-DISTRIBUTED-SYSTEMS](02-DISTRIBUTED-SYSTEMS.md) | CAP theorem, consistency models, consensus algorithms, replication |
| 2 | [03-CACHING-STRATEGIES](03-CACHING-STRATEGIES.md) | Cache-aside, write-through, write-behind, eviction policies, CDN caching |

### Exercises

**1. Consistency Trade-off Analysis:**

For each of the following systems, decide where it falls on the CAP spectrum and justify your choice:

- A banking ledger system
- A social media news feed
- A multiplayer online game leaderboard
- A DNS resolver
- A shopping cart service

For each system:
- State whether you prioritize consistency or availability
- Describe the user experience impact of your choice

**2. Caching Design:**

Design a caching strategy for an e-commerce product catalog:

- Choose the appropriate caching pattern (cache-aside, write-through, write-behind)
- Define the cache key structure
- Set TTL values and justify them
- Plan a cache invalidation strategy for product price updates
- Estimate the cache size needed for 1 million products
- Design a fallback for cache failures

### Knowledge Check

- [ ] Explain the CAP theorem in your own words with a real-world example
- [ ] What is the difference between strong consistency and eventual consistency?
- [ ] When would you use cache-aside vs. write-through caching?
- [ ] What is cache stampede and how do you prevent it?
- [ ] How does a CDN cache differ from an application-level cache?

---

## Phase 3: Load Balancing & Data Partitioning (Week 5–6)

### Learning Objectives

- Understand load balancing algorithms and their trade-offs
- Learn Layer 4 vs. Layer 7 load balancing
- Grasp data partitioning strategies: horizontal, vertical, directory-based
- Design effective partition key schemes
- Handle data rebalancing and hot partitions

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [04-LOAD-BALANCING](04-LOAD-BALANCING.md) | Algorithms (round-robin, least connections, consistent hashing), L4 vs. L7, health checks |
| 2 | [05-DATA-PARTITIONING](05-DATA-PARTITIONING.md) | Horizontal/vertical partitioning, partition keys, sharding strategies, rebalancing |

### Exercises

**1. Partition Key Selection:**

For each of the following data models, select an appropriate partition key and justify your choice:

| Data Model | Candidate Keys | Access Patterns |
|---|---|---|
| User messages | user_id, message_id, timestamp | Fetch all messages for a user |
| IoT sensor data | device_id, region, timestamp | Query by device over a time range |
| E-commerce orders | order_id, customer_id, date | Fetch orders by customer, analytics by date |
| Social media posts | post_id, author_id, hashtag | Fetch feed for a user, trending hashtags |

For each model:
- Explain why you chose that partition key
- Identify potential hot partition scenarios

**2. Load Balancer Configuration:**

Design a load balancing setup for a web application with the following tiers:

```
Internet → CDN → L7 Load Balancer → Web Servers (8 instances)
                                   → API Servers (12 instances)
```

For each tier:
- Choose the load balancing algorithm and justify it
- Define health check parameters (interval, timeout, unhealthy threshold)
- Configure session affinity rules where needed
- Design the failover behavior when instances go down

### Knowledge Check

- [ ] What is the difference between Layer 4 and Layer 7 load balancing?
- [ ] When would you use consistent hashing over round-robin?
- [ ] What is a hot partition and how do you detect it?
- [ ] How does horizontal partitioning differ from vertical partitioning?
- [ ] What are the challenges of cross-partition queries?

---

## Phase 4: Reliability (Week 7–8)

### Learning Objectives

- Understand availability tiers (99.9%, 99.99%, 99.999%) and their implications
- Learn fault tolerance patterns: redundancy, failover, replication
- Design for disaster recovery: RPO, RTO, backup strategies
- Practice chaos engineering principles
- Build systems that degrade gracefully under failure

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [06-RELIABILITY](06-RELIABILITY.md) | Availability, fault tolerance, redundancy, disaster recovery, chaos engineering, graceful degradation |

### Exercises

**1. Disaster Recovery Planning:**

Design a disaster recovery plan for a multi-region e-commerce platform:

- Define the RPO (Recovery Point Objective) and RTO (Recovery Time Objective)
- Design the data replication strategy across regions
- Document the failover procedure step by step
- Plan the failback procedure when the primary region recovers
- Create a communication plan for stakeholders during an outage

**2. Chaos Engineering Experiment:**

Design three chaos experiments for a microservices-based payment system:

| Experiment | Hypothesis | Blast Radius |
|---|---|---|
| Kill a Payment Service instance | Load balancer reroutes within 5s | Single instance in staging |
| Inject 500ms latency on DB calls | Circuit breaker trips, returns cached results | Single service in staging |
| Simulate region failure | DNS failover to secondary within 30s | Full region in staging |

For each experiment:
- Describe the steady-state metrics you will monitor
- Define the rollback procedure

**3. Availability Calculation:**

A system has the following component availabilities:

```
Web Server: 99.95%   App Server: 99.9%   Database: 99.99%
Cache:      99.95%   Queue:      99.9%
```

Calculate:
- The overall availability if all components are in series
- The improvement with redundancy (2 instances) for each component
- The number of nines needed from each component to achieve 99.99% overall

### Knowledge Check

- [ ] What is the difference between RPO and RTO?
- [ ] How do you calculate the availability of a system with components in series vs. parallel?
- [ ] What are the four principles of chaos engineering?
- [ ] What does graceful degradation mean and how do you implement it?
- [ ] Why is it important to test failure scenarios regularly?

---

## Phase 5: Common Designs (Week 9–10)

### Learning Objectives

- Apply system design principles to real-world problems
- Design URL shorteners, chat systems, and rate limiters
- Practice the structured approach: requirements → estimation → design → deep dive
- Learn to identify bottlenecks and optimize designs iteratively
- Communicate design decisions and trade-offs clearly

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [07-COMMON-DESIGNS](07-COMMON-DESIGNS.md) | URL shortener, chat system, rate limiter, news feed, search autocomplete |

### Exercises

**1. Design a URL Shortener:**

Design a URL shortening service (like bit.ly) with the following requirements:

- 100 million URLs shortened per month
- 10:1 read-to-write ratio
- URLs expire after 5 years by default
- Custom aliases supported
- Analytics (click count, geographic data)

Your design should include:
- Back-of-envelope estimation (storage, bandwidth, QPS)
- Database schema and choice (SQL vs. NoSQL)
- URL encoding strategy (base62, hash, counter-based)
- Caching strategy for popular URLs
- High-level architecture diagram

**2. Design a Chat System:**

Design a real-time chat system (like Slack or WhatsApp) supporting:

- 1:1 messaging and group chats (up to 500 members)
- Message delivery status (sent, delivered, read)
- Message history and search

Your design should address:
- Connection protocol (WebSocket, long polling, SSE)
- Message delivery guarantees (at-least-once, ordering)
- Storage strategy for messages (write-heavy, time-series nature)
- Group message fan-out strategy (write fan-out vs. read fan-out)

**3. Design a Rate Limiter:**

Design a distributed rate limiter for an API gateway:

- Support multiple rate limiting algorithms (fixed window, sliding window, token bucket)
- Handle 1 million requests per second across 10 gateway instances
- Allow per-user and per-API rate limits
- Return appropriate HTTP headers (X-RateLimit-Limit, X-RateLimit-Remaining)

Your design should address:
- Algorithm selection with trade-offs
- Distributed counting strategy (Redis-based vs. local + sync)
- Race condition handling
- Graceful handling when the rate limiter itself fails

### Knowledge Check

- [ ] What are the trade-offs between base62 encoding and MD5 hashing for URL shortening?
- [ ] Why is WebSocket preferred over HTTP long polling for chat systems?
- [ ] What is the difference between fixed window and sliding window rate limiting?
- [ ] How do you handle rate limiting in a distributed system with multiple gateway instances?
- [ ] What is fan-out on write vs. fan-out on read and when do you use each?

---

## Phase 6: Best Practices & Review (Week 11–12)

### Learning Objectives

- Apply design best practices to new and existing systems
- Identify and avoid common anti-patterns in system design
- Conduct system design reviews and provide constructive feedback
- Synthesize all prior phases into a complete end-to-end design
- Prepare for system design interviews and architecture discussions

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [08-BEST-PRACTICES](08-BEST-PRACTICES.md) | Design principles, documentation, review process, evolutionary architecture |
| 2 | [09-ANTI-PATTERNS](09-ANTI-PATTERNS.md) | Over-engineering, premature optimization, single points of failure, god services |

### Exercises

**1. Design Review:**

Review the following system description and identify issues, strengths, and improvements:

> "We built a social media platform with a single MySQL database handling all reads and writes for 10 million users. All images are stored as BLOBs in the database. The API is a monolithic Node.js server with 200 endpoints. We cache everything in a single Redis instance. The entire system runs in one AWS region with no backup strategy. Deployments happen manually via SSH on Friday afternoons."

For each issue found:
- Categorize it (scalability, reliability, security, operational)
- Propose a concrete improvement
- Prioritize the fix (P0 critical, P1 high, P2 medium, P3 low)

**2. Anti-Pattern Refactoring:**

Take each of the following anti-patterns and design the corrected approach:

| Anti-Pattern | Problem | Corrected Design |
|---|---|---|
| God Service | Single service handles users, orders, payments, and notifications | ? |
| Shared Database | 5 services read/write to the same tables | ? |
| Synchronous Chain | Service A → B → C → D → E for every request | ? |
| No Caching | Every request hits the database directly | ? |
| Big Bang Deployment | All components deployed simultaneously | ? |

For each:
- Draw a before and after architecture
- Estimate the effort (small, medium, large)

**3. Capstone Preparation:**

Before starting the capstone project, create a design document with:

- Requirements (functional and non-functional)
- Back-of-envelope estimation
- High-level architecture diagram
- Data model and storage design
- API design
- Scalability, reliability, and monitoring considerations

### Knowledge Check

- [ ] What are the signs of an over-engineered system?
- [ ] Why is premature optimization considered harmful?
- [ ] What is a single point of failure and how do you eliminate it?
- [ ] How do you decide whether to use a monolith or distributed architecture?
- [ ] What should a system design review checklist include?

---

## Capstone Project

Design a **production-grade system** that demonstrates everything you've learned across all six phases.

### System: Social Media Platform (or E-Commerce System)

Choose one of the following systems to design end-to-end:

**Option A: Social Media Platform**

```
                       ┌──────────────┐
                       │  API Gateway │
                       │  (auth, rate │
                       │   limiting)  │
                       └──────┬───────┘
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                  ▼
     ┌────────────┐   ┌────────────┐    ┌────────────┐
     │   User     │   │   Post     │    │   Feed     │
     │  Service   │   │  Service   │    │  Service   │
     └──────┬─────┘   └──────┬─────┘    └────────────┘
            │                │
     ┌──────┴──────┬─────────┘
     ▼             ▼
┌──────────┐ ┌──────────┐ ┌──────────────┐
│  Search  │ │  Media   │ │ Notification │
│  Service │ │  Service │ │   Service    │
└──────────┘ └──────────┘ └──────────────┘
```

**Option B: E-Commerce System**

```
                       ┌──────────────┐
                       │  API Gateway │
                       │  (auth, rate │
                       │   limiting)  │
                       └──────┬───────┘
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                  ▼
     ┌────────────┐   ┌────────────┐    ┌────────────┐
     │  Product   │   │   Order    │    │   User     │
     │  Catalog   │   │  Service   │    │  Service   │
     └──────┬─────┘   └──────┬─────┘    └────────────┘
            │                │
     ┌──────┴──────┬─────────┘
     ▼             ▼
┌──────────┐ ┌──────────┐ ┌──────────────┐
│ Payment  │ │ Shipping │ │ Notification │
│ Service  │ │ Service  │ │   Service    │
└──────────┘ └──────────┘ └──────────────┘
```

### Requirements

| Requirement | What to Design |
|---|---|
| Requirements gathering | Functional and non-functional requirements with prioritization |
| Estimation | Back-of-envelope calculations for storage, bandwidth, and QPS |
| Data model | Database schema, choice of SQL vs. NoSQL for each service |
| API design | RESTful API endpoints for at least 3 core services |
| Scalability | Horizontal scaling, data partitioning, caching at multiple layers |
| Reliability | Redundancy, failover, disaster recovery, graceful degradation |
| Caching | Multi-level caching strategy (CDN, application, database) |
| Load balancing | L4/L7 load balancers with appropriate algorithms |
| Monitoring | Metrics, logging, tracing, alerting, and dashboards |
| Security | Authentication, authorization, encryption, rate limiting |

### Evaluation Criteria

| Area | What to Verify |
|------|---------------|
| Completeness | All major components addressed with clear justification |
| Scalability | System handles 10x growth without architectural changes |
| Reliability | No single points of failure, defined RPO/RTO |
| Trade-offs | Clear articulation of decisions and their trade-offs |
| Estimation | Realistic capacity numbers with documented assumptions |
| Data design | Appropriate storage choices and partitioning strategies |
| Caching | Multi-layer caching with invalidation strategy |
| Communication | Clean diagrams, structured document, clear writing |

---

## Quick Reference: Document Map

| # | Document | Phase |
|---|----------|-------|
| 00 | [OVERVIEW](00-OVERVIEW.md) | 1 |
| 01 | [SCALABILITY](01-SCALABILITY.md) | 1 |
| 02 | [DISTRIBUTED-SYSTEMS](02-DISTRIBUTED-SYSTEMS.md) | 2 |
| 03 | [CACHING-STRATEGIES](03-CACHING-STRATEGIES.md) | 2 |
| 04 | [LOAD-BALANCING](04-LOAD-BALANCING.md) | 3 |
| 05 | [DATA-PARTITIONING](05-DATA-PARTITIONING.md) | 3 |
| 06 | [RELIABILITY](06-RELIABILITY.md) | 4 |
| 07 | [COMMON-DESIGNS](07-COMMON-DESIGNS.md) | 5 |
| 08 | [BEST-PRACTICES](08-BEST-PRACTICES.md) | 6 |
| 09 | [ANTI-PATTERNS](09-ANTI-PATTERNS.md) | 6 |

---

## Additional Resources

### Books

| Book | Author | Focus |
|------|--------|-------|
| *Designing Data-Intensive Applications* | Martin Kleppmann | Distributed systems and data fundamentals |
| *System Design Interview* | Alex Xu | Practical system design problems and solutions |
| *System Design Interview Vol. 2* | Alex Xu & Sahn Lam | Advanced system design problems |
| *Web Scalability for Startup Engineers* | Artur Ejsmont | Scalability patterns and practices |
| *Building Microservices* | Sam Newman | Distributed architecture and service design |
| *Site Reliability Engineering* | Google (Beyer et al.) | Reliability, monitoring, and incident response |

### Online Resources

- **donnemartin/system-design-primer** — Open-source system design study guide
- **highscalability.com** — Real-world architecture case studies
- **martinfowler.com** — Articles on distributed systems and architecture
- **ByteByteGo** — Visual system design explanations

### Tools

| Tool | Purpose |
|------|---------|
| Excalidraw | Architecture diagrams and whiteboarding |
| draw.io | Detailed system diagrams |
| Redis | Caching and rate limiting experiments |
| Apache Kafka | Event streaming and message queues |
| Prometheus + Grafana | Metrics collection and dashboards |
| Locust / k6 | Load testing and capacity planning |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-07-15 | Initial release — full learning path with 6 phases, capstone project, and resources |
