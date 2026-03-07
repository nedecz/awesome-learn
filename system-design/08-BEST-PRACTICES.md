# System Design Best Practices

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [The Design Process](#the-design-process)
   - [Step 1: Clarify Requirements](#step-1-clarify-requirements)
   - [Step 2: Estimate Scale](#step-2-estimate-scale)
   - [Step 3: Define APIs](#step-3-define-apis)
   - [Step 4: Design High-Level Architecture](#step-4-design-high-level-architecture)
   - [Step 5: Deep Dive into Components](#step-5-deep-dive-into-components)
   - [Step 6: Identify Bottlenecks](#step-6-identify-bottlenecks)
3. [Requirements Gathering](#requirements-gathering)
   - [Functional vs Non-Functional](#functional-vs-non-functional)
   - [Questions to Ask](#questions-to-ask)
   - [Prioritization](#prioritization)
4. [Capacity Estimation](#capacity-estimation)
   - [Back-of-Envelope Math](#back-of-envelope-math)
   - [Storage Estimation](#storage-estimation)
   - [Bandwidth Estimation](#bandwidth-estimation)
   - [QPS Estimation](#qps-estimation)
5. [API Design](#api-design)
   - [RESTful Principles](#restful-principles)
   - [Pagination](#pagination)
   - [Rate Limiting](#rate-limiting)
   - [Versioning](#versioning)
6. [Database Design](#database-design)
   - [Schema Design Principles](#schema-design-principles)
   - [Indexing Strategy](#indexing-strategy)
   - [Choosing SQL vs NoSQL](#choosing-sql-vs-nosql)
7. [Scalability Best Practices](#scalability-best-practices)
   - [Stateless Services](#stateless-services)
   - [Horizontal Scaling](#horizontal-scaling)
   - [Caching Layers](#caching-layers)
   - [Async Processing](#async-processing)
8. [Reliability Best Practices](#reliability-best-practices)
   - [Redundancy](#redundancy)
   - [Graceful Degradation](#graceful-degradation)
   - [Health Checks](#health-checks)
   - [Circuit Breakers](#circuit-breakers)
9. [Security Best Practices](#security-best-practices)
   - [Defense in Depth](#defense-in-depth)
   - [Authentication and Authorization](#authentication-and-authorization)
   - [Encryption](#encryption)
   - [Least Privilege](#least-privilege)
10. [Monitoring and Observability](#monitoring-and-observability)
    - [Key Metrics](#key-metrics)
    - [Alerting](#alerting)
    - [Dashboards](#dashboards)
11. [Documentation Standards](#documentation-standards)
    - [Architecture Decision Records](#architecture-decision-records)
    - [System Diagrams](#system-diagrams)
    - [Runbooks](#runbooks)
12. [Interview Tips](#interview-tips)
    - [System Design Interview Approach](#system-design-interview-approach)
    - [Common Mistakes](#common-mistakes)
    - [Time Management](#time-management)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

## Overview

This document compiles the essential best practices for designing, building, and operating large-scale systems. Use it as a reference when starting a new design, as a checklist before launching to production, and as a study guide for system design interviews.

### Target Audience

- Software engineers preparing for system design interviews
- Backend and full-stack developers building production systems
- Software architects designing distributed architectures
- Tech leads conducting architecture reviews
- SREs validating operational readiness of new systems

### Scope

- A structured approach to the design process
- Requirements gathering and capacity estimation techniques
- API, database, and schema design principles
- Scalability, reliability, and security patterns
- Monitoring, documentation, and interview strategies

## The Design Process

A structured approach prevents you from jumping into details before understanding the problem. Follow these six steps in order.

```
┌─────────────────────────────────────────────────────────────────┐
│                    The Design Process                           │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ Clarify  │→ │ Estimate │→ │  Define  │→ │  High-Level   │  │
│  │   Reqs   │  │  Scale   │  │   APIs   │  │  Architecture │  │
│  └──────────┘  └──────────┘  └──────────┘  └───────┬───────┘  │
│                                                     │          │
│                                                     ▼          │
│                              ┌──────────┐  ┌───────────────┐  │
│                              │ Identify │← │  Deep Dive    │  │
│                              │Bottleneck│  │  Components   │  │
│                              └──────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Step 1: Clarify Requirements

Before designing anything, make sure you understand **what** you are building and **why**.

- ✅ Ask about the core use cases and user workflows
- ✅ Identify who the users are and how many there will be
- ✅ Determine read-to-write ratio
- ✅ Establish consistency vs availability requirements
- ❌ Don't assume requirements — always ask

### Step 2: Estimate Scale

Back-of-envelope calculations drive every architectural decision. See [Capacity Estimation](#capacity-estimation) for detailed examples.

- ✅ Estimate daily active users (DAU)
- ✅ Calculate queries per second (QPS)
- ✅ Estimate storage needs for 5 years
- ✅ Determine peak-to-average traffic ratio

### Step 3: Define APIs

Define the external contract before internal design.

- ✅ List the primary API endpoints
- ✅ Define request and response schemas
- ✅ Identify which endpoints are read-heavy vs write-heavy
- ✅ Specify authentication and rate limiting requirements

### Step 4: Design High-Level Architecture

Draw the major components and how they communicate.

```
┌────────┐     ┌─────────────┐     ┌───────────┐     ┌──────────┐
│ Client │────→│ Load Balancer│────→│ App Server│────→│ Database │
└────────┘     └─────────────┘     └─────┬─────┘     └──────────┘
                                         │
                                   ┌─────▼─────┐
                                   │   Cache    │
                                   └───────────┘
```

- ✅ Show data flow direction between components
- ✅ Separate read and write paths if ratios differ significantly
- ✅ Include CDN, load balancer, cache, queue, and database where needed

### Step 5: Deep Dive into Components

Pick the most critical or complex component and design it in detail.

- ✅ Discuss database schema and indexing
- ✅ Explain data partitioning strategy (hash-based, range-based)
- ✅ Walk through a request lifecycle end to end
- ✅ Justify trade-offs (e.g., consistency vs latency)

### Step 6: Identify Bottlenecks

No design is complete without addressing failure modes and bottlenecks.

- ✅ Single points of failure — add redundancy
- ✅ Hot partitions — rebalance or add salt keys
- ✅ Slow queries — add indexes or denormalize
- ✅ Thundering herd — use request coalescing or jitter

## Requirements Gathering

### Functional vs Non-Functional

| Category | Examples | Questions |
|---|---|---|
| **Functional** | Create, read, update, delete operations; search; notifications | What can users do? What are the core workflows? |
| **Non-Functional** | Latency, throughput, availability, durability, consistency | How fast? How reliable? How much data? |
| **Constraints** | Budget, team size, timeline, regulatory compliance | What are the hard limits we cannot change? |

### Questions to Ask

Before starting any design, work through this checklist:

- [ ] Who are the users? How many are there?
- [ ] What are the core features (MVP)?
- [ ] What is the expected read-to-write ratio?
- [ ] What latency is acceptable (p50, p99)?
- [ ] What is the availability target (99.9%, 99.99%)?
- [ ] Is strong consistency required, or is eventual consistency acceptable?
- [ ] What is the expected data growth rate?
- [ ] Are there regulatory or compliance requirements (GDPR, HIPAA)?
- [ ] What is the budget and team size?
- [ ] What existing infrastructure must we integrate with?

### Prioritization

Not all requirements are equal. Use the MoSCoW method:

| Priority | Meaning | Example |
|---|---|---|
| **Must have** | System is unusable without this | User authentication, core CRUD |
| **Should have** | Important but not critical for launch | Search, notifications |
| **Could have** | Nice to have, implement if time permits | Analytics dashboard, recommendations |
| **Won't have** | Explicitly out of scope for now | Multi-language support, mobile app |

## Capacity Estimation

### Back-of-Envelope Math

Memorize these reference numbers for quick estimation:

| Unit | Value |
|---|---|
| 1 million | 10^6 |
| 1 billion | 10^9 |
| 1 KB | 1,000 bytes |
| 1 MB | 10^6 bytes |
| 1 GB | 10^9 bytes |
| 1 TB | 10^12 bytes |
| Seconds in a day | ~86,400 ≈ 10^5 |
| Seconds in a month | ~2.5 × 10^6 |

### Storage Estimation

**Example — URL Shortener (5-year projection):**

```
New URLs per day:       100 million
Bytes per record:       500 bytes (URL + metadata)
Daily storage:          100M × 500B = 50 GB/day
Monthly storage:        50 GB × 30 = 1.5 TB/month
5-year storage:         1.5 TB × 60 = 90 TB

With replication (3x):  90 TB × 3 = 270 TB total
```

### Bandwidth Estimation

**Example — Image Hosting Service:**

```
Uploads per day:        10 million
Average image size:     200 KB
Daily inbound:          10M × 200 KB = 2 TB/day
Inbound bandwidth:      2 TB / 86,400s ≈ 23 MB/s

Read-to-write ratio:    10:1
Daily outbound:         20 TB/day
Outbound bandwidth:     20 TB / 86,400s ≈ 230 MB/s
```

### QPS Estimation

**Example — Social Media Feed:**

```
Daily active users:     50 million
Avg requests per user:  20/day
Daily requests:         50M × 20 = 1 billion/day

Average QPS:            1B / 86,400 ≈ 11,500 QPS
Peak QPS (3x average):  ~35,000 QPS
```

- ✅ Always calculate both average and peak QPS
- ✅ Plan capacity for peak, not average
- ✅ Add a safety margin of 20–30% above peak

## API Design

### RESTful Principles

| Principle | Do | Don't |
|---|---|---|
| **Resource-based URIs** | `GET /users/{id}/orders` | `GET /getUserOrders?userId=123` |
| **HTTP methods** | `DELETE /orders/{id}` | `POST /deleteOrder` |
| **Status codes** | `201 Created`, `404 Not Found` | `200 OK` with error in body |
| **Consistent naming** | `snake_case` or `camelCase` (pick one) | Mixing conventions |

### Pagination

For endpoints that return lists, always paginate:

```
Offset-based (simple, but slow for deep pages):
  GET /orders?page=3&limit=20

Cursor-based (efficient for large datasets):
  GET /orders?cursor=eyJpZCI6MTAwfQ&limit=20

Keyset-based (best for real-time feeds):
  GET /feed?after_id=9999&limit=20
```

| Strategy | Pros | Cons |
|---|---|---|
| **Offset** | Simple, supports jumping to page N | Slow on large datasets, inconsistent with inserts |
| **Cursor** | Consistent, performant | Cannot jump to arbitrary page |
| **Keyset** | Fast with indexed columns | Requires a sortable, unique column |

### Rate Limiting

Protect your APIs from abuse and overload:

- ✅ Return `429 Too Many Requests` with `Retry-After` header
- ✅ Use sliding window or token bucket algorithms
- ✅ Apply rate limits per user, per IP, and globally
- ✅ Communicate limits via response headers (`X-RateLimit-Remaining`)

### Versioning

| Strategy | Example | Trade-off |
|---|---|---|
| **URI versioning** | `/api/v1/orders` | Simple but pollutes URI space |
| **Header versioning** | `Accept: application/vnd.api.v1+json` | Clean URIs but harder to test |
| **Query param** | `/orders?version=1` | Easy but non-standard |

- ✅ Prefer URI versioning for public APIs — it's explicit and easy to route
- ✅ Support at most two versions simultaneously (current and previous)
- ✅ Deprecate old versions with a published timeline

## Database Design

### Schema Design Principles

- ✅ **Start normalized** — eliminate data redundancy in your initial schema
- ✅ **Denormalize for reads** — add redundancy when query performance demands it
- ✅ **Use UUIDs for distributed IDs** — auto-increment does not work across shards
- ✅ **Add `created_at` and `updated_at`** — every table needs audit timestamps
- ✅ **Soft delete over hard delete** — use a `deleted_at` column for recoverability
- ❌ **Don't store derived data without a refresh strategy** — stale caches cause bugs

### Indexing Strategy

| Index Type | Use Case | Example |
|---|---|---|
| **Primary key** | Unique row lookup | `id` |
| **Single-column** | Filter or sort by one field | `email`, `created_at` |
| **Composite** | Multi-column queries | `(user_id, created_at)` |
| **Covering** | Query satisfied by index alone | `(user_id, status) INCLUDE (total)` |
| **Partial** | Index a subset of rows | `WHERE status = 'active'` |

- ✅ Index columns used in `WHERE`, `JOIN`, and `ORDER BY`
- ✅ Put the most selective column first in composite indexes
- ❌ Don't over-index — each index slows down writes
- ❌ Don't index low-cardinality columns (e.g., boolean fields) in isolation

### Choosing SQL vs NoSQL

| Factor | SQL (PostgreSQL, MySQL) | NoSQL (DynamoDB, Cassandra, MongoDB) |
|---|---|---|
| **Data model** | Structured, relational | Flexible, document/key-value/wide-column |
| **Consistency** | Strong (ACID) | Eventual (tunable in some) |
| **Query flexibility** | Ad-hoc queries, JOINs | Limited query patterns, denormalized |
| **Scale model** | Vertical first, then read replicas | Horizontal by design |
| **Best for** | Transactions, complex queries | High write throughput, simple access patterns |

**Decision guide:**

- ✅ Use SQL when you need transactions, complex queries, or strong consistency
- ✅ Use NoSQL when you need massive write throughput or flexible schemas
- ✅ Use both when different parts of the system have different needs

## Scalability Best Practices

### Stateless Services

Stateless services are easier to scale because any instance can handle any request.

```
Stateful (hard to scale):         Stateless (easy to scale):
┌──────────┐                      ┌──────────┐
│ Server A │ ← session data       │ Server A │
└──────────┘                      └──────────┘
     ↑ user must return                ↑ any server works
     │ to same server            ┌─────┴──────┐
                                 │ Shared     │
                                 │ Session    │
                                 │ Store      │
                                 │ (Redis)    │
                                 └────────────┘
```

- ✅ Store session state in Redis or a database, not in application memory
- ✅ Store file uploads in object storage (S3), not on local disk
- ✅ Use environment variables or a config service for configuration
- ❌ Don't use sticky sessions — they create unbalanced load and complicate failover

### Horizontal Scaling

| Strategy | What It Scales | When to Use |
|---|---|---|
| **Add app servers** | Compute | CPU-bound workloads |
| **Read replicas** | Database reads | Read-heavy workloads |
| **Sharding** | Database writes and storage | Write-heavy or large datasets |
| **CDN** | Static content delivery | Global user base |
| **Message queues** | Async processing | Bursty or background workloads |

### Caching Layers

```
┌────────┐    ┌───────────┐    ┌──────────────┐    ┌──────────┐
│ Client │───→│ CDN Cache │───→│ App Cache    │───→│ Database │
│        │    │ (static)  │    │ (Redis/      │    │          │
│        │    │           │    │  Memcached)  │    │          │
└────────┘    └───────────┘    └──────────────┘    └──────────┘
```

| Pattern | Description | Best For |
|---|---|---|
| **Cache-aside** | App checks cache, falls back to DB | General purpose, read-heavy |
| **Write-through** | App writes to cache and DB together | Strong consistency needs |
| **Write-behind** | App writes to cache; async flush to DB | Write-heavy workloads |
| **Read-through** | Cache loads from DB on miss automatically | Simplified application logic |

- ✅ Set TTLs on every cache entry — unbounded caches lead to stale data
- ✅ Use cache stampede protection (locking or request coalescing)
- ❌ Don't cache user-specific data in shared CDN layers

### Async Processing

Move non-critical work off the request path:

- ✅ **Email / SMS notifications** — queue and process asynchronously
- ✅ **Image resizing / video encoding** — use background workers
- ✅ **Analytics and logging** — buffer and batch write
- ✅ **Third-party API calls** — decouple with a message queue

```
Synchronous (slow):                Asynchronous (fast):
User → API → Send Email → 200 OK  User → API → Queue Message → 200 OK
             (500ms)                              (10ms)
                                        Worker → Send Email
                                                  (500ms, but user isn't waiting)
```

## Reliability Best Practices

### Redundancy

Every critical component should have at least one backup:

- [ ] **Application servers** — run at least 3 instances behind a load balancer
- [ ] **Databases** — primary with synchronous or asynchronous replicas
- [ ] **Caches** — clustered Redis or Memcached with replication
- [ ] **Load balancers** — active-passive or active-active pair
- [ ] **DNS** — multiple DNS providers or multi-region DNS
- [ ] **Data centers** — multi-AZ at minimum, multi-region for critical services

### Graceful Degradation

When a non-critical dependency fails, the system should continue with reduced functionality:

```
Full functionality:                 Degraded (search service down):
┌────────────────────────────┐     ┌────────────────────────────┐
│ E-Commerce Product Page    │     │ E-Commerce Product Page    │
│                            │     │                            │
│ Product Details        ✅  │     │ Product Details        ✅  │
│ Price & Availability   ✅  │     │ Price & Availability   ✅  │
│ Personalized Recs      ✅  │     │ Personalized Recs      ⚠️  │
│ Search                 ✅  │     │ Search                 ⚠️  │
│ Reviews                ✅  │     │ Reviews                ✅  │
│                            │     │ (showing cached results    │
│                            │     │  and popular items)        │
└────────────────────────────┘     └────────────────────────────┘
```

- ✅ Classify dependencies as **critical** (must have) or **non-critical** (nice to have)
- ✅ Define fallback behavior for every non-critical dependency
- ✅ Test degraded modes regularly — don't wait for an outage to discover they're broken

### Health Checks

Implement three levels of health checks:

| Check Type | Purpose | Endpoint |
|---|---|---|
| **Liveness** | Is the process running? | `GET /health/live` |
| **Readiness** | Can the service handle traffic? | `GET /health/ready` |
| **Startup** | Has the service finished initializing? | `GET /health/startup` |

- ✅ Liveness checks should be lightweight (return 200 immediately)
- ✅ Readiness checks should verify critical dependencies (database, cache)
- ❌ Don't include non-critical dependencies in readiness checks — cascading failures

### Circuit Breakers

Prevent a failing dependency from taking down the entire system:

```
┌─────────┐     ┌─────────┐     ┌─────────┐
│ CLOSED  │────→│  OPEN   │────→│  HALF   │
│ (normal)│     │ (fail   │     │  OPEN   │
│         │     │  fast)  │     │ (probe) │
└─────────┘     └─────────┘     └────┬────┘
     ↑                               │
     └───── success ─────────────────┘
```

| State | Behavior |
|---|---|
| **Closed** | Requests flow normally; failures are counted |
| **Open** | All requests fail immediately; no calls to dependency |
| **Half-Open** | A limited number of probe requests are sent to test recovery |

- ✅ Set thresholds based on error rate, not absolute count
- ✅ Use exponential backoff for the open-to-half-open transition
- ✅ Log every state transition for debugging

## Security Best Practices

### Defense in Depth

Apply security at every layer — no single control should be the only barrier:

```
┌──────────────────────────────────────────────┐
│  Layer 1: Edge / CDN (DDoS protection, WAF)  │
├──────────────────────────────────────────────┤
│  Layer 2: API Gateway (auth, rate limiting)   │
├──────────────────────────────────────────────┤
│  Layer 3: Application (input validation)      │
├──────────────────────────────────────────────┤
│  Layer 4: Data (encryption at rest, RBAC)     │
├──────────────────────────────────────────────┤
│  Layer 5: Network (VPC, security groups)      │
└──────────────────────────────────────────────┘
```

### Authentication and Authorization

| Concern | Best Practice |
|---|---|
| **Authentication** | Use OAuth 2.0 / OpenID Connect for user-facing APIs |
| **Service-to-service** | Use mTLS or signed JWTs for internal communication |
| **API keys** | Use for machine-to-machine access with scoped permissions |
| **Authorization** | Enforce at the API gateway and again at the service level |
| **Session management** | Use short-lived tokens with refresh token rotation |

- ✅ Validate tokens on every request — don't trust the network
- ✅ Implement RBAC (Role-Based Access Control) or ABAC (Attribute-Based)
- ❌ Don't roll your own authentication — use proven identity providers

### Encryption

- ✅ **In transit** — TLS 1.2+ for all external traffic, mTLS for internal
- ✅ **At rest** — AES-256 for data stored in databases and object storage
- ✅ **Secrets** — use a secrets manager (Vault, AWS Secrets Manager) — never commit secrets to code
- ✅ **Keys** — rotate encryption keys on a regular schedule
- ❌ **Don't use** — MD5 or SHA-1 for anything security-sensitive

### Least Privilege

- ✅ Each service account has only the permissions it needs
- ✅ Database users have access to only their own schemas
- ✅ IAM policies are scoped to specific resources, not `*`
- ✅ Review permissions quarterly and revoke unused access
- ❌ Don't use root or admin accounts for application access

## Monitoring and Observability

### Key Metrics

Track the **RED** and **USE** methods:

| Method | Metric | Description |
|---|---|---|
| **RED** | Rate | Requests per second |
| **RED** | Errors | Error rate (percentage of failed requests) |
| **RED** | Duration | Latency distribution (p50, p95, p99) |
| **USE** | Utilization | Percentage of resource capacity in use |
| **USE** | Saturation | Queue depth, thread pool exhaustion |
| **USE** | Errors | Hardware or resource errors |

### Alerting

| Severity | Condition | Response | Example |
|---|---|---|---|
| **Critical** | SLO breach or data loss risk | Page on-call immediately | Error rate > 5% for 5 min |
| **Warning** | Degraded performance | Notify team in Slack | p99 latency > 2s for 10 min |
| **Info** | Unusual pattern | Review next business day | Traffic 50% above normal |

- ✅ Alert on symptoms (error rate, latency), not causes (CPU usage)
- ✅ Every alert must have a linked runbook
- ✅ Use burn-rate alerting for SLO-based monitoring
- ❌ Don't alert on metrics that don't require human action — alert fatigue kills reliability

### Dashboards

Every system should have these dashboards:

- [ ] **Service health** — QPS, error rate, latency (p50/p95/p99)
- [ ] **Infrastructure** — CPU, memory, disk, network utilization
- [ ] **Dependencies** — health and latency of downstream services
- [ ] **Business metrics** — orders/min, signups/day, revenue/hour
- [ ] **SLO burn rate** — remaining error budget over time

## Documentation Standards

### Architecture Decision Records

Document every significant architectural decision using ADRs:

```
# ADR-001: Use PostgreSQL as Primary Database

## Status
Accepted

## Context
We need a primary database that supports ACID transactions,
complex queries, and scales to 10 TB over the next 3 years.

## Decision
We will use PostgreSQL with read replicas for scaling reads
and Citus for horizontal sharding if needed.

## Consequences
- (+) Strong consistency and rich query support
- (+) Mature ecosystem with proven reliability
- (-) Vertical scaling limits may require sharding later
- (-) Team needs training on PostgreSQL-specific features
```

- ✅ Write an ADR for every decision that is hard to reverse
- ✅ Include the context, alternatives considered, and trade-offs
- ✅ Store ADRs in the repository alongside the code they describe

### System Diagrams

Every system should maintain these diagrams:

| Diagram | Purpose | Update Frequency |
|---|---|---|
| **Architecture overview** | Shows all major components and data flow | Every major change |
| **Sequence diagram** | Illustrates request lifecycle for key workflows | Per feature |
| **Data flow diagram** | Shows how data moves and transforms | Per data model change |
| **Deployment diagram** | Shows infrastructure, regions, and networking | Per infra change |

- ✅ Use a diagram-as-code tool (Mermaid, PlantUML, Structurizr)
- ✅ Keep diagrams in version control next to the code
- ❌ Don't use binary formats (Visio, Lucidchart) as the source of truth

### Runbooks

Write a runbook for every alert and common operational task:

- [ ] **Title** — clear description of the scenario
- [ ] **Symptoms** — what does the alert or user report look like?
- [ ] **Impact** — what is affected and how severely?
- [ ] **Diagnosis** — step-by-step commands to identify the root cause
- [ ] **Mitigation** — immediate actions to restore service
- [ ] **Resolution** — permanent fix after the incident
- [ ] **Escalation** — who to contact if the runbook doesn't resolve the issue

## Interview Tips

### System Design Interview Approach

Use the same structured process from [The Design Process](#the-design-process), adapted for a 45-minute interview:

| Phase | Time | Activities |
|---|---|---|
| **Requirements** | 5 min | Clarify scope, ask questions, list functional and non-functional requirements |
| **Estimation** | 5 min | Back-of-envelope QPS, storage, bandwidth |
| **High-level design** | 10 min | Draw major components, data flow, API endpoints |
| **Deep dive** | 20 min | Focus on 1–2 critical components, discuss trade-offs |
| **Wrap-up** | 5 min | Address bottlenecks, monitoring, future extensions |

### Common Mistakes

- ❌ **Jumping into solution without clarifying requirements** — always ask questions first
- ❌ **Over-engineering from the start** — design for current scale, explain how to evolve
- ❌ **Ignoring non-functional requirements** — availability, latency, and consistency matter
- ❌ **Not discussing trade-offs** — every design decision has pros and cons
- ❌ **Staying silent** — system design interviews evaluate communication as much as design
- ❌ **Trying to cover everything** — depth on key components beats breadth on all of them

### Time Management

- ✅ **Use a watch** — track time actively during the interview
- ✅ **Check in with the interviewer** — "Should I go deeper here or move on?"
- ✅ **Lead the conversation** — treat it as a collaborative design session, not a Q&A
- ✅ **Summarize before transitioning** — "We've covered the high-level design. Let me now deep dive into the database layer."
- ✅ **Save 5 minutes for wrap-up** — address bottlenecks, monitoring, and future improvements

## Next Steps

Return to [Common System Designs](07-COMMON-DESIGNS.md) to see these best practices applied to real-world systems, or revisit [Scalability](01-SCALABILITY.md) and [Reliability](06-RELIABILITY.md) for deeper treatment of individual topics.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial release — design process, requirements, estimation, API and database design |
| 1.1 | 2026 | Added scalability, reliability, and security best practices |
| 1.2 | 2026 | Added monitoring, documentation standards, and interview tips |
