# Microservices Architecture Patterns

## Table of Contents

1. [Overview](#overview)
2. [API Gateway Pattern](#api-gateway-pattern)
3. [Backend for Frontend (BFF)](#backend-for-frontend-bff)
4. [Service Mesh](#service-mesh)
5. [Sidecar Pattern](#sidecar-pattern)
6. [Strangler Fig Pattern](#strangler-fig-pattern)
7. [Aggregator Pattern](#aggregator-pattern)
8. [CQRS Pattern](#cqrs-pattern)
9. [Event Sourcing](#event-sourcing)
10. [Choosing the Right Pattern](#choosing-the-right-pattern)
11. [Next Steps](#next-steps)

## Overview

This document covers the architectural patterns commonly used in microservices systems. These patterns address cross-cutting concerns like routing, security, observability, and migration from monoliths.

### Target Audience

- Software architects designing microservices systems
- Platform engineers building shared infrastructure
- Developers implementing service-level patterns

### Scope

- API Gateway and request routing
- Backend-for-Frontend composition
- Service mesh and sidecar proxy
- Strangler fig migration pattern
- Aggregator, CQRS, and event sourcing patterns

## API Gateway Pattern

An API Gateway is a single entry point for all client requests. It routes requests to the appropriate microservice, handles cross-cutting concerns, and can aggregate responses.

### Responsibilities

```
┌─────────────────────────────────────────────────┐
│                  API Gateway                     │
│                                                   │
│  ┌───────────┐ ┌──────────┐ ┌────────────────┐  │
│  │ Routing   │ │ Auth     │ │ Rate Limiting  │  │
│  │           │ │ (JWT,    │ │                │  │
│  │ Path →    │ │  OAuth2) │ │ Per-client     │  │
│  │ Service   │ │          │ │ throttling     │  │
│  └───────────┘ └──────────┘ └────────────────┘  │
│                                                   │
│  ┌───────────┐ ┌──────────┐ ┌────────────────┐  │
│  │ TLS       │ │ Request/ │ │ Response       │  │
│  │ Termin-   │ │ Response │ │ Caching        │  │
│  │ ation     │ │ Transform│ │                │  │
│  └───────────┘ └──────────┘ └────────────────┘  │
└───────────────────────┬─────────────────────────┘
         ┌──────────────┼──────────────┐
         ▼              ▼              ▼
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │ Order    │  │ Payment  │  │ Catalog  │
   │ Service  │  │ Service  │  │ Service  │
   └──────────┘  └──────────┘  └──────────┘
```

### Common Implementations

| Gateway | Type | Key Features |
|---|---|---|
| **Kong** | Open source | Plugin ecosystem, gRPC, declarative config |
| **AWS API Gateway** | Managed | Lambda integration, WebSocket support, usage plans |
| **NGINX** | Open source | High performance, Lua scripting, upstream management |
| **Envoy** | Open source | Service mesh proxy, gRPC-native, observability built-in |
| **Spring Cloud Gateway** | Framework | Java-native, predicate-based routing, filters |

### Gateway Routing Example

```yaml
# Kong declarative configuration example
services:
  - name: order-service
    url: http://order-svc:8080
    routes:
      - name: order-routes
        paths:
          - /api/v1/orders
        methods:
          - GET
          - POST
        plugins:
          - name: rate-limiting
            config:
              minute: 100
          - name: jwt

  - name: catalog-service
    url: http://catalog-svc:8080
    routes:
      - name: catalog-routes
        paths:
          - /api/v1/products
        methods:
          - GET
```

## Backend for Frontend (BFF)

The BFF pattern creates a dedicated backend service for each frontend type (web, mobile, IoT). Each BFF aggregates and transforms data from downstream microservices into a format optimized for its specific frontend.

### Architecture

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│   Web    │  │  Mobile  │  │   IoT    │
│  Client  │  │   App    │  │ Devices  │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │              │
┌────▼─────┐ ┌────▼─────┐ ┌─────▼────┐
│ Web BFF  │ │Mobile BFF│ │ IoT BFF  │
│          │ │          │ │          │
│ Rich     │ │ Compact  │ │ Minimal  │
│ payloads │ │ payloads │ │ payloads │
│ Complex  │ │ Optimized│ │ Binary   │
│ queries  │ │ for 4G   │ │ protocol │
└────┬─────┘ └────┬─────┘ └────┬─────┘
     │             │              │
     └──────┬──────┴──────┬───────┘
            │             │
      ┌─────▼──────┐ ┌───▼────────┐
      │ Order Svc  │ │ Catalog Svc│
      └────────────┘ └────────────┘
```

### When to Use BFF

- ✅ Multiple frontend types with different data requirements
- ✅ Mobile apps needing smaller payloads than web
- ✅ Different authentication flows per frontend
- ❌ A single frontend type — use a standard API gateway instead
- ❌ When all frontends need the same data shape

## Service Mesh

A service mesh is a dedicated infrastructure layer that handles service-to-service communication. It is implemented as a set of sidecar proxies deployed alongside each service instance.

### Architecture

```
┌──────────────────────────────────────────────────────┐
│                    Control Plane                      │
│         (Istiod, Linkerd Control Plane)               │
│                                                       │
│  ┌──────────┐  ┌──────────────┐  ┌────────────────┐ │
│  │ Config   │  │ Certificate  │  │ Service        │ │
│  │ (routing │  │ Authority    │  │ Discovery      │ │
│  │  rules)  │  │ (mTLS certs) │  │                │ │
│  └──────────┘  └──────────────┘  └────────────────┘ │
└──────────────────────┬───────────────────────────────┘
                       │ Pushes config
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ ┌─────────┐ │ │ ┌─────────┐ │ │ ┌─────────┐ │
│ │  Proxy  │ │ │ │  Proxy  │ │ │ │  Proxy  │ │
│ │ (Envoy) │ │ │ │ (Envoy) │ │ │ │ (Envoy) │ │
│ ├─────────┤ │ │ ├─────────┤ │ │ ├─────────┤ │
│ │ Order   │ │ │ │ Payment │ │ │ │ Catalog │ │
│ │ Service │ │ │ │ Service │ │ │ │ Service │ │
│ └─────────┘ │ │ └─────────┘ │ │ └─────────┘ │
└─────────────┘ └─────────────┘ └─────────────┘
      Pod              Pod              Pod
```

### Service Mesh Features

| Feature | Description |
|---|---|
| **mTLS** | Automatic mutual TLS encryption between all services |
| **Traffic management** | Canary releases, traffic splitting, retries, timeouts |
| **Observability** | Distributed tracing, metrics, and access logs without code changes |
| **Circuit breaking** | Automatically stop sending traffic to failing services |
| **Access control** | Fine-grained authorization policies between services |

### When to Use a Service Mesh

- ✅ Large number of services (> 10) with complex communication patterns
- ✅ Need for mTLS without application code changes
- ✅ Consistent observability across all services
- ❌ Small systems (< 5 services) — overhead not justified
- ❌ Teams unfamiliar with Kubernetes and proxy configuration

## Sidecar Pattern

The sidecar pattern deploys a helper container alongside the main application container within the same Pod or host. The sidecar handles cross-cutting concerns like logging, monitoring, networking, or configuration.

### Common Sidecar Use Cases

```
┌─────────────────────────────────────────┐
│                   Pod                    │
│                                          │
│  ┌───────────────┐  ┌────────────────┐  │
│  │ Application   │  │ Sidecar        │  │
│  │ Container     │  │ Container      │  │
│  │               │  │                │  │
│  │ Business      │  │ - Log shipper  │  │
│  │ Logic         │  │ - Proxy (Envoy)│  │
│  │               │  │ - Config agent │  │
│  │               │  │ - Cert rotator │  │
│  └───────────────┘  └────────────────┘  │
│        │                    │            │
│        └────── localhost ───┘            │
│             (shared network)             │
└──────────────────────────────────────────┘
```

| Sidecar Type | Examples | Responsibility |
|---|---|---|
| **Proxy sidecar** | Envoy, Linkerd proxy | Traffic management, mTLS, load balancing |
| **Log shipper** | Fluentd, Filebeat | Collect and forward application logs |
| **Config agent** | Consul agent, Vault agent | Dynamic configuration and secret injection |
| **Monitoring agent** | Datadog agent, OTEL collector | Metrics collection and export |

## Strangler Fig Pattern

The strangler fig pattern enables incremental migration from a monolith to microservices by gradually routing traffic from the old system to new services.

### Migration Process

```
Step 1: Identify                Step 2: Build              Step 3: Route
a feature to                    the new service            traffic to the
extract                                                    new service

┌──────────────┐               ┌──────────────┐           ┌──────────────┐
│  Monolith    │               │  Monolith    │           │  Monolith    │
│              │               │              │           │  (smaller)   │
│ ┌──────────┐ │               │  ┌────────┐  │           │              │
│ │ Feature  │ │               │  │Remaining│  │           │              │
│ │ to       │ │               │  │features │  │           └──────────────┘
│ │ extract  │ │               │  └────────┘  │
│ └──────────┘ │               └──────────────┘           ┌──────────────┐
└──────────────┘                                          │  New Service │
                               ┌──────────────┐           │  (extracted  │
                               │  New Service │           │   feature)   │
                               │  (parallel)  │           └──────────────┘
                               └──────────────┘
```

### Implementation Steps

1. **Identify** — Choose a well-bounded feature with clear inputs/outputs
2. **Build** — Create the new microservice implementing the extracted feature
3. **Proxy** — Add a routing layer (API gateway or reverse proxy) in front of the monolith
4. **Route** — Redirect traffic for the extracted feature to the new service
5. **Verify** — Run both implementations in parallel, comparing results
6. **Cut over** — Remove the old code from the monolith once confident
7. **Repeat** — Extract the next feature

## Aggregator Pattern

The aggregator pattern composes data from multiple microservices into a single response for the client. This avoids requiring clients to make multiple calls and assemble data themselves.

### Architecture

```
Client Request: GET /dashboard
         │
    ┌────▼─────┐
    │Aggregator│
    │ Service  │
    └──┬─┬──┬──┘
       │ │  │
  ┌────┘ │  └────┐
  ▼      ▼       ▼
┌────┐ ┌────┐ ┌────┐
│User│ │Order│ │Inv │
│Svc │ │Svc  │ │Svc │
└────┘ └────┘ └────┘

Aggregator Response:
{
  "user": { ... },
  "recentOrders": [ ... ],
  "inventory": { ... }
}
```

### Considerations

- ✅ Reduces client-side complexity and network calls
- ✅ Can optimize payloads for specific use cases
- ❌ Adds latency (waits for all downstream responses)
- ❌ Creates a coupling point — changes in downstream services may require aggregator changes

## CQRS Pattern

Command Query Responsibility Segregation (CQRS) separates read and write operations into different models. This is particularly useful in microservices where read and write workloads have different scaling and performance characteristics.

### Architecture

```
                    ┌─────────────┐
                    │   Client    │
                    └──┬───────┬──┘
                       │       │
              Commands │       │ Queries
                       ▼       ▼
              ┌─────────┐ ┌─────────┐
              │ Command  │ │  Query  │
              │  Model   │ │  Model  │
              │ (write)  │ │  (read) │
              └────┬─────┘ └────┬────┘
                   │            │
              ┌────▼─────┐ ┌───▼────┐
              │ Write DB  │ │Read DB │
              │(PostgreSQL│ │(Elastic│
              │ normalized│ │ Search,│
              │ schema)   │ │ Redis) │
              └────┬──────┘ └───▲────┘
                   │            │
                   └─── Events ─┘
                   (sync read model)
```

### When to Use CQRS

- ✅ Read and write workloads have very different performance profiles
- ✅ Complex queries that don't fit the write-optimized schema
- ✅ Read replicas need different data formats (e.g., search indexes)
- ❌ Simple CRUD applications — CQRS adds unnecessary complexity
- ❌ When strong consistency between read and write is required

## Event Sourcing

Event sourcing stores state as an immutable sequence of events rather than current state. Instead of updating a record, you append an event describing what happened.

### How It Works

```
Traditional (mutable state):
┌─────────────────────────────────┐
│ orders table                    │
│ id: 123                         │
│ status: SHIPPED  (overwritten)  │
│ total: $99.00                   │
└─────────────────────────────────┘

Event Sourcing (immutable log):
┌───────────────────────────────────────────┐
│ Event Store                               │
│                                           │
│ 1. OrderCreated    { id:123, total:$99 }  │
│ 2. PaymentReceived { id:123, amount:$99 } │
│ 3. OrderShipped    { id:123, carrier:UPS }│
│                                           │
│ Current state = replay all events         │
└───────────────────────────────────────────┘
```

### Benefits and Trade-Offs

| Benefit | Trade-Off |
|---|---|
| Complete audit trail | Increased storage requirements |
| Can rebuild state at any point in time | Event schema evolution is complex |
| Enables temporal queries | Eventual consistency between event store and projections |
| Natural fit for event-driven microservices | Higher implementation complexity |

## Choosing the Right Pattern

| Pattern | Use When | Avoid When |
|---|---|---|
| **API Gateway** | Multiple clients need a single entry point | Only one client with direct service access |
| **BFF** | Multiple frontends with different data needs | Single frontend type |
| **Service Mesh** | > 10 services; need mTLS and observability at scale | < 5 services; adds unnecessary overhead |
| **Sidecar** | Cross-cutting concerns need separation from business logic | Simple services with minimal infrastructure needs |
| **Strangler Fig** | Migrating from monolith incrementally | Greenfield microservices projects |
| **Aggregator** | Clients need composed views from multiple services | Single-service responses suffice |
| **CQRS** | Read/write workloads differ significantly | Simple CRUD operations |
| **Event Sourcing** | Audit trail required; temporal queries needed | Simple state management with few state transitions |

## Next Steps

Continue to [Communication Patterns](03-COMMUNICATION-PATTERNS.md) to learn about synchronous communication (REST, gRPC), asynchronous messaging (event-driven, message queues), and how to choose the right communication style for each interaction.
