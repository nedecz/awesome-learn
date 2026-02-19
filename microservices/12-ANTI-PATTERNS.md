# Microservices Anti-Patterns

## Table of Contents

1. [Overview](#overview)
2. [Distributed Monolith](#distributed-monolith)
3. [Shared Database](#shared-database)
4. [Chatty Services](#chatty-services)
5. [Wrong Service Boundaries](#wrong-service-boundaries)
6. [Inadequate Observability](#inadequate-observability)
7. [Missing Resilience Patterns](#missing-resilience-patterns)
8. [Big Bang Migration](#big-bang-migration)
9. [Premature Microservices](#premature-microservices)
10. [Ignoring Data Consistency](#ignoring-data-consistency)
11. [Security Anti-Patterns](#security-anti-patterns)
12. [Anti-Pattern Summary](#anti-pattern-summary)
13. [Next Steps](#next-steps)

## Overview

This document catalogs the most common microservices anti-patterns — architectural and operational mistakes that undermine the benefits of microservices. For each anti-pattern, we describe the problem, the symptoms, and the correct approach.

### Target Audience

- Architects reviewing microservices designs
- Developers learning from common mistakes
- Teams migrating from monolith to microservices

### Scope

- Architectural anti-patterns (distributed monolith, wrong boundaries)
- Data anti-patterns (shared database, ignoring consistency)
- Communication anti-patterns (chatty services, synchronous chains)
- Operational anti-patterns (missing observability, missing resilience)
- Migration anti-patterns (big bang, premature adoption)

## Distributed Monolith

### The Problem

A distributed monolith looks like microservices (separate deployable units) but behaves like a monolith (services must be deployed together, changes cascade across services).

```
❌ Distributed Monolith

  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Service A│───▶│ Service B│───▶│ Service C│
  └──────────┘    └──────────┘    └──────────┘
       │               │               │
       └───────┬───────┴───────┬───────┘
               │               │
        Must deploy all    Shared data
        three together     model across
        for any change     all services
```

### Symptoms

- Deploying one service requires deploying others at the same time
- Changes to one service's API break multiple other services
- Services share internal data models or libraries
- Releases require coordination across multiple teams
- The system has all the complexity of microservices with none of the benefits

### How to Fix

- ✅ Define clear, stable API contracts between services
- ✅ Use events for asynchronous communication instead of synchronous chains
- ✅ Ensure each service can be deployed, tested, and scaled independently
- ✅ Avoid shared libraries that contain business logic

## Shared Database

### The Problem

Multiple services read from and write to the same database. This creates tight coupling on schema, makes independent deployment impossible, and eliminates data ownership.

```
❌ Shared Database                      ✅ Database per Service

┌──────────┐    ┌──────────┐           ┌──────────┐    ┌──────────┐
│ Order    │    │ Payment  │           │ Order    │    │ Payment  │
│ Service  │    │ Service  │           │ Service  │    │ Service  │
└────┬─────┘    └────┬─────┘           └────┬─────┘    └────┬─────┘
     │               │                      │               │
     └───────┬───────┘                 ┌────▼─────┐    ┌────▼─────┐
             │                         │ Order DB │    │Payment DB│
      ┌──────▼──────┐                 └──────────┘    └──────────┘
      │  Shared DB  │
      │             │                  Each service owns its
      │ orders      │                  database. Changes to one
      │ payments    │                  schema don't affect others.
      │ customers   │
      └─────────────┘
```

### Symptoms

- A schema change in one service breaks another service
- Multiple services insert/update the same tables
- Database migrations require coordination across teams
- Cannot change a service's database technology independently
- Performance issues in one service's queries affect all services

### How to Fix

- ✅ Give each service its own database (schema, instance, or both)
- ✅ Share data through APIs or events, not direct database access
- ✅ Use the transactional outbox pattern for reliable event publishing
- ✅ Accept eventual consistency — not all data needs to be real-time

## Chatty Services

### The Problem

Services make too many fine-grained synchronous calls to each other. This creates high latency, tight coupling, and amplifies failures.

```
❌ Chatty Services (N+1 calls)

  Order Service needs customer + items + shipping:

  ┌──────────┐     GET /customers/123        ┌──────────┐
  │  Order   │ ──────────────────────────────▶│ Customer │
  │  Service │     GET /items/456             │ Service  │
  │          │ ──────────────────────────────▶└──────────┘
  │          │     GET /items/789             ┌──────────┐
  │          │ ──────────────────────────────▶│ Catalog  │
  │          │     GET /items/012             │ Service  │
  │          │ ──────────────────────────────▶└──────────┘
  │          │     GET /shipping/estimate     ┌──────────┐
  │          │ ──────────────────────────────▶│ Shipping │
  └──────────┘                                │ Service  │
                                              └──────────┘
  5 synchronous calls → high latency, all must succeed
```

### How to Fix

```
✅ Batch API / Event-Driven

  Option 1: Batch API
  POST /items/batch { ids: [456, 789, 012] }  → One call instead of three

  Option 2: Local replica via events
  Order Service maintains a local read-only copy of catalog data
  (updated via events from Catalog Service)

  Option 3: Aggregator / BFF
  A dedicated aggregator service composes the data in a single call
```

### Guidelines

- ✅ Use batch APIs to reduce the number of calls
- ✅ Maintain local read replicas via events for frequently accessed data
- ✅ Use the BFF pattern to aggregate data for frontends
- ❌ Do not chain more than 2–3 synchronous calls
- ❌ Do not call multiple services sequentially when parallel calls are possible

## Wrong Service Boundaries

### The Problem

Services are split along the wrong lines — typically along technical layers (UI service, business logic service, data service) instead of business capabilities.

```
❌ Technical Boundaries                ✅ Business Boundaries

┌────────────────────┐               ┌──────────┐ ┌──────────┐ ┌──────────┐
│    UI Service      │               │  Order   │ │ Payment  │ │ Catalog  │
├────────────────────┤               │ Service  │ │ Service  │ │ Service  │
│  Business Service  │               │          │ │          │ │          │
├────────────────────┤     →         │ UI + API │ │ UI + API │ │ UI + API │
│    Data Service    │               │ + Logic  │ │ + Logic  │ │ + Logic  │
└────────────────────┘               │ + Data   │ │ + Data   │ │ + Data   │
                                     └──────────┘ └──────────┘ └──────────┘
Every change touches                 Changes are contained
all three services                   within one service
```

### Symptoms

- Every feature change requires changes in multiple services
- Services map to technical layers, not business domains
- Teams cannot deliver a feature without coordinating with other teams
- "Utility" services exist that have no clear business purpose

### How to Fix

- ✅ Use Domain-Driven Design to identify bounded contexts
- ✅ Run event storming workshops with domain experts
- ✅ Organize services around business capabilities (orders, payments, shipping)
- ✅ Apply Conway's Law — align service boundaries with team boundaries

## Inadequate Observability

### The Problem

Services are deployed without proper logging, metrics, tracing, or alerting. When something goes wrong, there is no visibility into what happened or where.

### Symptoms

- "It's slow" but nobody knows where the latency is
- Errors appear in one service but the root cause is in another
- Incidents take hours to diagnose because there are no traces or correlated logs
- No dashboards or alerts — issues are discovered by users

### How to Fix

- ✅ Implement the three pillars: traces, logs, and metrics
- ✅ Use OpenTelemetry for vendor-neutral instrumentation
- ✅ Add correlation IDs and trace context to every request
- ✅ Create dashboards for every service (RED metrics, dependency health)
- ✅ Set up SLO-based alerting with runbooks

## Missing Resilience Patterns

### The Problem

Services call dependencies without timeouts, retries, or circuit breakers. A single failing dependency causes cascading failures across the entire system.

```
❌ No Resilience (Cascading Failure)

  Service A → Service B → Service C (DOWN)
                              ↓
                    Service B hangs (no timeout)
                              ↓
                    Service A hangs (no timeout)
                              ↓
                    All services are down

✅ With Resilience (Contained Failure)

  Service A → Service B → Circuit Breaker → Service C (DOWN)
                              ↓
                    Circuit opens → return fallback
                              ↓
                    Service B returns degraded response
                              ↓
                    Service A continues to function
```

### How to Fix

- ✅ Add timeouts to every external call
- ✅ Implement circuit breakers for all synchronous dependencies
- ✅ Use retries with exponential backoff for transient failures
- ✅ Add bulkheads to isolate failures per dependency
- ✅ Design fallback responses for non-critical dependencies

## Big Bang Migration

### The Problem

Attempting to rewrite an entire monolith as microservices all at once. This is a high-risk approach that often fails.

### Symptoms

- The migration takes months or years with no production value delivered
- The "new system" keeps growing in scope
- The team is maintaining two systems simultaneously
- The cutover date keeps moving

### How to Fix

- ✅ Use the strangler fig pattern — migrate one feature at a time
- ✅ Deliver value incrementally — each extracted service goes to production
- ✅ Keep the monolith running — route traffic gradually to new services
- ❌ Do not attempt to rewrite everything at once

## Premature Microservices

### The Problem

Adopting microservices before the domain is understood, the team is mature enough, or the system is complex enough to justify the overhead.

### Symptoms

- A team of 3 developers is maintaining 15 microservices
- Service boundaries keep changing because the domain is not well understood
- More time is spent on infrastructure than on features
- The team struggles with distributed debugging, data consistency, and deployment

### How to Fix

- ✅ Start with a monolith or modular monolith
- ✅ Identify domain boundaries through experience before extracting services
- ✅ Migrate to microservices only when the complexity justifies the overhead
- ✅ Build DevOps maturity (CI/CD, monitoring, container orchestration) before adopting microservices

## Ignoring Data Consistency

### The Problem

Assuming data will be strongly consistent across services, or ignoring the challenges of eventual consistency entirely.

### Symptoms

- Users see stale or inconsistent data across different parts of the application
- Duplicate records are created because events are processed more than once
- No compensation logic exists for saga failures
- Events are lost because there is no outbox pattern or dead-letter queue

### How to Fix

- ✅ Design UIs to show pending/processing states (not just success/failure)
- ✅ Use idempotent event processing (deduplicate by event ID)
- ✅ Implement the transactional outbox pattern for reliable event publishing
- ✅ Add compensation logic for every saga step
- ✅ Use dead-letter queues for unprocessable events

## Security Anti-Patterns

| Anti-Pattern | Problem | Solution |
|---|---|---|
| **Trusting the internal network** | Internal services communicate without authentication | Implement mTLS and service-level authorization |
| **Hardcoded secrets** | API keys and passwords in source code | Use a secrets manager (Vault, AWS SM) |
| **Running as root** | Containers run with root privileges | Use non-root users in Dockerfiles |
| **No input validation** | Services accept any input from other services | Validate all input at every service boundary |
| **Overly permissive RBAC** | Services have admin-level permissions | Apply least-privilege principle |
| **No image scanning** | Container images with known vulnerabilities | Scan images in CI/CD pipeline (Trivy, Grype) |

## Anti-Pattern Summary

| Anti-Pattern | Root Cause | Fix |
|---|---|---|
| Distributed monolith | Tight coupling between services | Clear contracts, async communication, independent deployability |
| Shared database | No data ownership | Database per service, share via APIs/events |
| Chatty services | Fine-grained synchronous calls | Batch APIs, events, local replicas |
| Wrong boundaries | Technical decomposition | DDD, event storming, business capabilities |
| No observability | Instrumentation not prioritized | Traces, logs, metrics from day one |
| No resilience | Missing patterns | Timeout, circuit breaker, retry, bulkhead |
| Big bang migration | All-at-once rewrite | Strangler fig, incremental extraction |
| Premature microservices | Complexity before understanding domain | Start monolith, extract when ready |
| Ignoring consistency | Assuming strong consistency | Sagas, outbox, idempotency, dead-letter queues |
| Security gaps | Trusting internal network | mTLS, secrets management, least privilege |

## Next Steps

Review the [Learning Path](LEARNING-PATH.md) for a structured guide through all the microservices documentation, from fundamentals to production readiness.
