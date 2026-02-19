# Microservices Learning Path

A structured, self-paced training guide to mastering microservices architecture — from fundamentals and design principles to production-grade operations. Each phase builds on the previous one, progressing from core concepts to advanced patterns.

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

- Understand what microservices are and how they differ from monoliths
- Learn the core design principles: single responsibility, loose coupling, high cohesion
- Grasp Domain-Driven Design concepts and bounded contexts
- Evaluate when microservices are appropriate

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | Core concepts, monolith vs. microservices, when to adopt |
| 2 | [01-DESIGN-PRINCIPLES](01-DESIGN-PRINCIPLES.md) | Single responsibility, DDD, bounded contexts, API-first design |

### Exercises

**1. Domain Analysis:**

Choose a domain you are familiar with (e.g., e-commerce, banking, healthcare) and:

- List 5–10 business capabilities
- Group related capabilities into bounded contexts
- Identify the ubiquitous language for each context
- Draw a context map showing relationships between contexts

**2. Monolith vs. Microservices Trade-off Analysis:**

For a system you have worked on or are familiar with:

- List the benefits microservices would provide
- List the challenges microservices would introduce
- Decide: should this system be microservices? Why or why not?

### Knowledge Check

- [ ] What are the six core characteristics of microservices?
- [ ] When should you NOT use microservices?
- [ ] What is a bounded context and how does it relate to a microservice?
- [ ] Explain Conway's Law and its impact on service boundaries

---

## Phase 2: Architecture and Communication (Week 3–4)

### Learning Objectives

- Understand architectural patterns: API gateway, BFF, service mesh, sidecar
- Learn synchronous and asynchronous communication patterns
- Choose the right communication pattern for each interaction
- Design event schemas and idempotent processing

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [02-ARCHITECTURE-PATTERNS](02-ARCHITECTURE-PATTERNS.md) | API gateway, BFF, service mesh, strangler fig, CQRS |
| 2 | [03-COMMUNICATION-PATTERNS](03-COMMUNICATION-PATTERNS.md) | REST, gRPC, async messaging, choreography vs. orchestration |

### Exercises

**1. API Design:**

Design a RESTful API for an Order Service:

- Define endpoints for CRUD operations
- Include proper HTTP status codes
- Add pagination for list endpoints
- Define an idempotency key header for POST requests
- Write an OpenAPI specification

**2. Event Schema Design:**

Design event schemas for an order workflow:

- `OrderPlaced` — what data should this event contain?
- `PaymentProcessed` — what fields are needed?
- `OrderShipped` — what metadata should be included?
- Include correlation IDs and versioning

### Knowledge Check

- [ ] When would you use a BFF instead of a single API gateway?
- [ ] What is the difference between choreography and orchestration?
- [ ] Why is gRPC preferred over REST for internal service communication?
- [ ] How do you ensure idempotent event processing?

---

## Phase 3: Data Management (Week 5–6)

### Learning Objectives

- Implement the database per service pattern
- Design sagas for distributed transactions
- Understand the transactional outbox pattern
- Learn CQRS and event sourcing fundamentals
- Configure service discovery

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [04-DATA-MANAGEMENT](04-DATA-MANAGEMENT.md) | Database per service, saga, outbox, CQRS, event sourcing |
| 2 | [05-SERVICE-DISCOVERY](05-SERVICE-DISCOVERY.md) | Client-side, server-side, DNS, registries, health checks |

### Exercises

**1. Saga Design:**

Design a saga for a checkout workflow:

```
1. Create Order (Order Service)
2. Reserve Inventory (Inventory Service)
3. Process Payment (Payment Service)
4. Confirm Order (Order Service)
```

For each step, define:
- The command/event that triggers it
- The compensation action if it fails
- Whether choreography or orchestration is more appropriate

**2. Outbox Pattern Implementation:**

Design the outbox table schema and polling/CDC mechanism for the Order Service:

- What columns does the outbox table need?
- How will unpublished events be detected?
- How will events be marked as published?
- What happens if the relay process crashes?

### Knowledge Check

- [ ] Why is two-phase commit (2PC) not recommended across microservices?
- [ ] What is the dual-write problem and how does the outbox pattern solve it?
- [ ] When should you use orchestration vs. choreography for sagas?
- [ ] How does DNS-based service discovery differ from a service registry?

---

## Phase 4: Resilience and Security (Week 7–8)

### Learning Objectives

- Implement resilience patterns: circuit breaker, retry, bulkhead, timeout
- Design security for microservices: zero trust, OAuth2, mTLS
- Configure secrets management and network policies
- Layer resilience patterns for comprehensive fault tolerance

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [06-RESILIENCE-PATTERNS](06-RESILIENCE-PATTERNS.md) | Circuit breaker, retry, timeout, bulkhead, fallback |
| 2 | [07-SECURITY](07-SECURITY.md) | Zero trust, OAuth2/OIDC, mTLS, secrets management |

### Exercises

**1. Resilience Configuration:**

For a service that depends on three downstream services (Payment, Inventory, Notification):

- Configure circuit breaker thresholds for each dependency
- Define retry policies (which errors to retry, max attempts, backoff)
- Set timeout values (connection timeout, read timeout)
- Design a bulkhead configuration (thread pools or semaphores)
- Define fallback strategies for each dependency

**2. Security Audit:**

Review a service and check for:

- [ ] All endpoints require authentication
- [ ] JWT tokens are validated properly
- [ ] Secrets are not hardcoded
- [ ] Input is validated at the API boundary
- [ ] Network policies restrict inter-service traffic

### Knowledge Check

- [ ] What are the three states of a circuit breaker?
- [ ] Why should retries use exponential backoff with jitter?
- [ ] What is the difference between liveness and readiness health checks?
- [ ] How does mTLS differ from standard TLS?

---

## Phase 5: Observability and Testing (Week 9–10)

### Learning Objectives

- Implement the three pillars of observability: traces, logs, metrics
- Use OpenTelemetry for vendor-neutral instrumentation
- Design a testing strategy: unit, integration, contract, E2E
- Practice chaos engineering for resilience validation

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [08-OBSERVABILITY](08-OBSERVABILITY.md) | Distributed tracing, logging, metrics, alerting, OpenTelemetry |
| 2 | [10-TESTING-STRATEGIES](10-TESTING-STRATEGIES.md) | Testing pyramid, contract testing, chaos engineering |

### Exercises

**1. Observability Setup:**

For a three-service system (API Gateway → Order Service → Payment Service):

- Define structured log format (JSON) with required fields
- List RED metrics to export from each service
- Design a trace that spans all three services
- Create an SLO: "99.9% of orders complete in < 2 seconds"
- Define alerts based on the SLO error budget

**2. Contract Test:**

Write a consumer-driven contract test:

- The Order Service (consumer) expects the Payment Service (provider) to accept a POST /payments request and return a 201 with payment ID and status
- Define the contract using Pact or a similar tool
- Explain how the provider verifies this contract in its CI pipeline

### Knowledge Check

- [ ] What is the RED method for monitoring?
- [ ] Why is structured logging important in microservices?
- [ ] What problem does contract testing solve?
- [ ] What are the principles of chaos engineering?

---

## Phase 6: Production Readiness (Week 11–12)

### Learning Objectives

- Apply the production readiness checklist to a microservice
- Design deployment strategies: canary, blue-green, feature flags
- Identify and fix common anti-patterns
- Organize teams around microservices

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [09-DEPLOYMENT-STRATEGIES](09-DEPLOYMENT-STRATEGIES.md) | Blue-green, canary, feature flags, GitOps, CI/CD |
| 2 | [11-BEST-PRACTICES](11-BEST-PRACTICES.md) | Production readiness checklist, team organization |
| 3 | [12-ANTI-PATTERNS](12-ANTI-PATTERNS.md) | Distributed monolith, shared database, chatty services |

### Exercises

**1. Production Readiness Review:**

Take a service you have built (or the capstone project) and evaluate it against the production readiness checklist:

- [ ] Health checks implemented
- [ ] Structured logging with trace IDs
- [ ] RED metrics exported
- [ ] Circuit breakers on all external calls
- [ ] Secrets managed properly
- [ ] Alerts configured with runbooks
- [ ] Deployment strategy defined
- [ ] Rollback plan documented

**2. Anti-Pattern Audit:**

Review the following system description and identify the anti-patterns:

> "We have 20 microservices that all share a PostgreSQL database. Services communicate via synchronous REST calls, typically chaining 4–5 services deep. There is no distributed tracing, and logging is ad-hoc (some services log to stdout, others to files). Deployments are coordinated — we deploy all services together every two weeks."

<details>
<summary>Anti-patterns identified (click to reveal)</summary>

1. **Shared database** — 20 services on one PostgreSQL database
2. **Chatty services** — 4–5 deep synchronous call chains
3. **No observability** — No tracing, inconsistent logging
4. **Distributed monolith** — Coordinated bi-weekly deployments
5. **No resilience** — No mention of circuit breakers, timeouts, or retries

</details>

### Knowledge Check

- [ ] What is the difference between a canary deployment and a blue-green deployment?
- [ ] What is GitOps and why is it recommended for microservices?
- [ ] What are the symptoms of a distributed monolith?
- [ ] What team topology is recommended for microservices organizations?

---

## Capstone Project

Design a **production-grade microservices system** that demonstrates everything you've learned across all six phases.

### System: Online Food Ordering Platform

```
                         ┌──────────────┐
                         │  API Gateway │
                         │  (auth, rate │
                         │   limiting)  │
                         └──────┬───────┘
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                  ▼
       ┌───────────┐    ┌───────────┐     ┌───────────┐
       │ Restaurant│    │   Order   │     │   User    │
       │  Service  │    │  Service  │     │  Service  │
       │           │    │           │     │           │
       │ Menu,     │    │ Cart,     │     │ Auth,     │
       │ Search    │    │ Checkout  │     │ Profile   │
       └─────┬─────┘    └─────┬─────┘     └───────────┘
             │                │
      ┌──────┴────────┬───────┘
      ▼               ▼
┌───────────┐  ┌───────────┐  ┌───────────┐
│  Payment  │  │  Delivery │  │Notification│
│  Service  │  │  Service  │  │  Service   │
└───────────┘  └───────────┘  └───────────┘
```

### Requirements

| Requirement | What to Design |
|---|---|
| Service boundaries | 6 services with clear bounded contexts and owned databases |
| API design | OpenAPI specs for at least 2 services |
| Communication | Mix of sync (REST/gRPC) and async (events) communication |
| Data management | Saga for checkout flow, outbox pattern for event publishing |
| Resilience | Circuit breakers, retries, timeouts on all external calls |
| Security | OAuth2/JWT authentication, mTLS between services |
| Observability | Structured logging, distributed tracing, RED metrics |
| Testing | Unit tests, contract tests between Order and Payment services |
| Deployment | GitOps with canary deployment for Order Service |
| Documentation | README, API contract, ADR for key decisions |

### Evaluation Criteria

| Area | What to Verify |
|------|---------------|
| Architecture | Clear service boundaries aligned with business capabilities |
| Communication | Appropriate use of sync and async patterns |
| Data | Database per service, sagas for distributed transactions |
| Resilience | Circuit breakers, retries, timeouts, fallbacks |
| Security | Authentication, authorization, encrypted communication |
| Observability | Traces, logs, metrics, alerts with runbooks |
| Testing | Unit, contract, and integration tests |
| Deployment | GitOps, canary or blue-green, automated rollback |

---

## Quick Reference: Document Map

| # | Document | Phase |
|---|----------|-------|
| 00 | [OVERVIEW](00-OVERVIEW.md) | 1 |
| 01 | [DESIGN-PRINCIPLES](01-DESIGN-PRINCIPLES.md) | 1 |
| 02 | [ARCHITECTURE-PATTERNS](02-ARCHITECTURE-PATTERNS.md) | 2 |
| 03 | [COMMUNICATION-PATTERNS](03-COMMUNICATION-PATTERNS.md) | 2 |
| 04 | [DATA-MANAGEMENT](04-DATA-MANAGEMENT.md) | 3 |
| 05 | [SERVICE-DISCOVERY](05-SERVICE-DISCOVERY.md) | 3 |
| 06 | [RESILIENCE-PATTERNS](06-RESILIENCE-PATTERNS.md) | 4 |
| 07 | [SECURITY](07-SECURITY.md) | 4 |
| 08 | [OBSERVABILITY](08-OBSERVABILITY.md) | 5 |
| 09 | [DEPLOYMENT-STRATEGIES](09-DEPLOYMENT-STRATEGIES.md) | 6 |
| 10 | [TESTING-STRATEGIES](10-TESTING-STRATEGIES.md) | 5 |
| 11 | [BEST-PRACTICES](11-BEST-PRACTICES.md) | 6 |
| 12 | [ANTI-PATTERNS](12-ANTI-PATTERNS.md) | 6 |

---

## Recommended Resources

### Books

| Book | Author | Focus |
|------|--------|-------|
| *Building Microservices* | Sam Newman | Comprehensive introduction |
| *Microservices Patterns* | Chris Richardson | Patterns and solutions |
| *Domain-Driven Design* | Eric Evans | Strategic and tactical DDD |
| *Release It!* | Michael Nygard | Resilience patterns for production |
| *Team Topologies* | Skelton & Pais | Team organization for delivery |

### Online Resources

- **microservices.io** — Pattern catalog by Chris Richardson
- **12factor.net** — Twelve-factor app methodology
- **martinfowler.com** — Articles on microservices and architecture
- **CNCF Landscape** — Cloud-native ecosystem map

### Tools

| Tool | Purpose |
|------|---------|
| Docker | Container runtime |
| Kubernetes | Container orchestration |
| Kafka | Event streaming |
| Istio / Linkerd | Service mesh |
| Jaeger | Distributed tracing |
| Prometheus + Grafana | Metrics and dashboards |
| Pact | Contract testing |
| ArgoCD / Flux | GitOps |
| Vault | Secrets management |
| k6 | Load testing |
