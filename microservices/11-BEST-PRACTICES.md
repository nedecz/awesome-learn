# Microservices Best Practices

## Table of Contents

1. [Overview](#overview)
2. [Production Readiness Checklist](#production-readiness-checklist)
3. [Service Design](#service-design)
4. [API Design](#api-design)
5. [Data Management](#data-management)
6. [Resilience and Reliability](#resilience-and-reliability)
7. [Observability](#observability)
8. [Security](#security)
9. [Deployment and Operations](#deployment-and-operations)
10. [Team Organization](#team-organization)
11. [Documentation Standards](#documentation-standards)
12. [Next Steps](#next-steps)

## Overview

This document compiles the essential best practices for building, deploying, and operating microservices in production. Use it as a checklist before going live and as a reference for ongoing improvements.

### Target Audience

- Engineering teams preparing microservices for production
- Tech leads conducting architecture reviews
- SREs validating operational readiness

### Scope

- Production readiness checklist
- Service, API, and data design best practices
- Resilience, observability, and security standards
- Team organization and documentation patterns

## Production Readiness Checklist

### Before Launching a New Service

- [ ] **Service has a clear owner** (team and on-call rotation)
- [ ] **Service has a well-defined API contract** (OpenAPI, Protobuf, or AsyncAPI)
- [ ] **Health checks are implemented** (liveness, readiness, startup)
- [ ] **Structured logging is enabled** (JSON format, includes trace IDs)
- [ ] **Metrics are exported** (RED metrics: Rate, Errors, Duration)
- [ ] **Distributed tracing is configured** (trace context propagation)
- [ ] **Alerts are set up** (based on SLOs, not raw metrics)
- [ ] **Dashboards exist** (service health, dependencies, SLOs)
- [ ] **Runbooks are written** (for every alert and common failure scenario)
- [ ] **Resilience patterns are implemented** (circuit breaker, retry, timeout)
- [ ] **Security is configured** (authentication, authorization, mTLS)
- [ ] **Secrets are managed properly** (Vault, AWS Secrets Manager — not env vars)
- [ ] **Resource limits are set** (CPU, memory, connections)
- [ ] **Autoscaling is configured** (HPA for Kubernetes, target tracking for ECS)
- [ ] **Deployment strategy is defined** (canary, blue-green, or rolling)
- [ ] **Rollback plan is documented and tested**
- [ ] **Load testing has been performed** (under expected and peak load)
- [ ] **Contract tests exist** (for all consumed and provided APIs)
- [ ] **Data backup and recovery is configured**
- [ ] **Graceful shutdown is implemented** (drain connections, finish in-flight requests)

## Service Design

### Do

- ✅ **One service, one business capability** — services should be organized around business domains, not technical layers
- ✅ **Design for failure** — assume every dependency can fail; implement circuit breakers and fallbacks
- ✅ **Keep services independently deployable** — no coordinated multi-service releases
- ✅ **Own your data** — each service has its own database; share data via APIs or events
- ✅ **Start simple** — begin with a monolith or modular monolith; extract services when boundaries are clear
- ✅ **Design APIs first** — agree on contracts before writing implementation code

### Don't

- ❌ **Don't create nano-services** — a service that does nothing on its own is too small
- ❌ **Don't share databases** — this creates tight coupling and defeats the purpose of microservices
- ❌ **Don't build a distributed monolith** — if every change requires deploying multiple services together, you have a distributed monolith
- ❌ **Don't use microservices for simple applications** — the overhead is not worth it for small teams or simple domains

## API Design

### RESTful APIs

- ✅ Use resource-based URIs (`/orders/{id}`, not `/getOrder?id=123`)
- ✅ Use standard HTTP methods (GET, POST, PUT, PATCH, DELETE)
- ✅ Return appropriate HTTP status codes
- ✅ Version your APIs (URI versioning: `/api/v1/orders`)
- ✅ Support pagination for list endpoints (`?page=1&limit=20`)
- ✅ Use HATEOAS links for discoverability (when appropriate)

### Idempotency

All write operations should be idempotent — sending the same request multiple times produces the same result.

```
Idempotency Key Pattern:

  POST /orders
  Idempotency-Key: order-abc-123

  First call:  Creates order → 201 Created
  Second call: Returns existing order → 200 OK (or 201 with same body)
  Third call:  Returns existing order → 200 OK
```

### Backward Compatibility

- ✅ Adding new fields to a response is safe
- ✅ Adding new optional fields to a request is safe
- ❌ Removing fields from a response is a breaking change
- ❌ Making optional fields required is a breaking change
- ❌ Changing field types is a breaking change

## Data Management

- ✅ **Database per service** — each service owns its data store
- ✅ **Use sagas for distributed transactions** — not two-phase commit
- ✅ **Use the transactional outbox pattern** — for reliable event publishing
- ✅ **Accept eventual consistency** — design UIs to show pending/processing states
- ✅ **Use event-driven data replication** — for read-only copies of data from other services
- ❌ **Don't join across service databases** — compose data at the application level
- ❌ **Don't use distributed ACID transactions** — they don't scale

## Resilience and Reliability

### Every External Call Must Have

| Pattern | Purpose |
|---|---|
| **Timeout** | Prevent indefinite waiting |
| **Retry with backoff** | Handle transient failures |
| **Circuit breaker** | Stop calling a failing dependency |
| **Bulkhead** | Isolate failures per dependency |
| **Fallback** | Provide a degraded response when a dependency fails |

### Graceful Shutdown

```
1. Receive SIGTERM
2. Stop accepting new requests
3. Wait for in-flight requests to complete (with a deadline)
4. Close database connections and message consumers
5. Exit process
```

### Graceful Degradation

When a non-critical dependency is unavailable, the service should continue to function with reduced capability rather than failing entirely.

```
Full functionality:        Degraded (recommendation service down):
┌─────────────────────┐   ┌─────────────────────┐
│ Product Page         │   │ Product Page         │
│                      │   │                      │
│ Product Details  ✅  │   │ Product Details  ✅  │
│ Price            ✅  │   │ Price            ✅  │
│ Reviews          ✅  │   │ Reviews          ✅  │
│ Recommendations  ✅  │   │ Recommendations  ⚠️  │
│                      │   │ (showing popular     │
│                      │   │  items instead)      │
└─────────────────────┘   └─────────────────────┘
```

## Observability

### Every Service Must Have

- **Structured logging** — JSON format, includes trace ID, span ID, service name
- **Metrics** — RED metrics (Rate, Errors, Duration) for every endpoint
- **Distributed tracing** — W3C Trace Context propagation
- **Health endpoints** — `/health/live`, `/health/ready`
- **Dashboards** — service health, dependency health, SLO burn rate
- **Alerts** — SLO-based alerting with runbook links

### Logging Standards

```json
{
  "timestamp": "2025-01-15T10:30:00.123Z",
  "level": "INFO",
  "service": "order-service",
  "version": "2.1.0",
  "traceId": "abc123",
  "spanId": "span-1",
  "message": "Order created",
  "orderId": "order-789",
  "customerId": "cust-456",
  "duration_ms": 45
}
```

## Security

- ✅ **Zero trust** — authenticate and authorize every request, even internal
- ✅ **mTLS** — encrypt all service-to-service traffic
- ✅ **Least privilege** — each service has only the permissions it needs
- ✅ **Input validation** — validate all input at the API boundary
- ✅ **Secrets management** — use Vault, AWS Secrets Manager, or Azure Key Vault
- ✅ **Container security** — non-root users, minimal base images, vulnerability scanning
- ❌ **Don't store secrets in code** — use a secrets manager
- ❌ **Don't trust internal traffic** — assume the network is hostile

## Deployment and Operations

- ✅ **Independent deployments** — each service is deployed independently
- ✅ **Immutable artifacts** — the same container image goes from staging to production
- ✅ **GitOps** — Git is the source of truth for deployment state
- ✅ **Automated rollback** — automatically roll back if health checks fail
- ✅ **Canary deployments** — for high-risk changes
- ✅ **Feature flags** — decouple deployment from release
- ❌ **Don't deploy on Fridays** — unless you have automated rollback and on-call coverage

### Resource Limits

Always set resource requests and limits for every container:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

## Team Organization

### Team Topologies for Microservices

| Team Type | Responsibility | Example |
|---|---|---|
| **Stream-aligned team** | Owns one or more services end-to-end (dev, deploy, operate) | Order team, Payment team |
| **Platform team** | Provides shared infrastructure (CI/CD, Kubernetes, observability) | Platform engineering |
| **Enabling team** | Helps stream-aligned teams adopt new practices | SRE coaches, security advisors |
| **Complicated subsystem team** | Owns a technically complex component | ML model serving, real-time data pipeline |

### You Build It, You Run It

Each team owns the full lifecycle of their services:

- ✅ Development and testing
- ✅ Deployment and release
- ✅ Monitoring and alerting
- ✅ On-call and incident response
- ✅ Performance and cost optimization

## Documentation Standards

### Every Service Should Document

| Document | Contents |
|---|---|
| **README** | Service purpose, how to run locally, API overview |
| **API contract** | OpenAPI spec, Protobuf definitions, or AsyncAPI for events |
| **Architecture decision records (ADRs)** | Why key decisions were made |
| **Runbooks** | How to diagnose and resolve common issues |
| **SLO definitions** | Service level objectives and error budgets |
| **Dependency map** | Which services this service depends on and provides to |

## Next Steps

Continue to [Anti-Patterns](12-ANTI-PATTERNS.md) to learn about the most common microservices mistakes and how to avoid them.
