# Microservices Design Principles

## Table of Contents

1. [Overview](#overview)
2. [Single Responsibility Principle](#single-responsibility-principle)
3. [Loose Coupling](#loose-coupling)
4. [High Cohesion](#high-cohesion)
5. [Domain-Driven Design](#domain-driven-design)
6. [Bounded Contexts](#bounded-contexts)
7. [Service Boundaries](#service-boundaries)
8. [API-First Design](#api-first-design)
9. [Autonomy and Decentralization](#autonomy-and-decentralization)
10. [Best Practices](#best-practices)
11. [Next Steps](#next-steps)

## Overview

This document covers the fundamental design principles that guide microservices architecture. These principles determine how to identify, scope, and design individual services for maintainability, scalability, and independent evolution.

### Target Audience

- Software architects defining service boundaries
- Developers designing and building microservices
- Tech leads establishing architecture standards
- Domain experts collaborating on service decomposition

### Scope

- Single responsibility and service scoping
- Loose coupling and high cohesion
- Domain-Driven Design concepts for microservices
- Bounded contexts and context mapping
- API-first design and service contracts
- Decentralized governance and team autonomy

## Single Responsibility Principle

Each microservice should have a single responsibility — it should do one thing well and own one business capability end-to-end.

### How to Apply

```
❌ Too broad                         ✅ Right-sized
┌────────────────────────┐          ┌──────────┐ ┌──────────┐ ┌──────────┐
│    E-Commerce Service  │          │ Catalog  │ │  Order   │ │ Payment  │
│                        │          │ Service  │ │ Service  │ │ Service  │
│  - Product catalog     │          │          │ │          │ │          │
│  - Order management    │    →     │ Browse   │ │ Create   │ │ Charge   │
│  - Payment processing  │          │ Search   │ │ Track    │ │ Refund   │
│  - Shipping tracking   │          │ Filter   │ │ Cancel   │ │ Payout   │
│  - User reviews        │          └──────────┘ └──────────┘ └──────────┘
└────────────────────────┘
                                    ┌──────────┐ ┌──────────┐
                                    │ Shipping │ │  Review  │
                                    │ Service  │ │ Service  │
                                    │          │ │          │
                                    │ Track    │ │ Submit   │
                                    │ Estimate │ │ Moderate │
                                    │ Label    │ │ Query    │
                                    └──────────┘ └──────────┘
```

### Sizing Guidelines

| Signal | Too Large | Too Small |
|---|---|---|
| **Team size** | More than 8–10 developers | Less than 2 developers |
| **Deploy frequency** | Changes require coordinated releases | Deployed only as part of another service |
| **Responsibility** | Multiple distinct business capabilities | Only a thin wrapper around a single function |
| **Data ownership** | Manages many unrelated data entities | Has no data of its own |

## Loose Coupling

Services should depend on each other as little as possible. A change in one service should not require changes in other services.

### Types of Coupling to Avoid

| Coupling Type | Description | How to Avoid |
|---|---|---|
| **Temporal coupling** | Service A must be running for Service B to work | Use async messaging and event-driven patterns |
| **Spatial coupling** | Service A hardcodes the network location of Service B | Use service discovery or DNS |
| **Data coupling** | Services share a database or data schema | Database per service; share data via APIs or events |
| **Implementation coupling** | Service A depends on internal details of Service B | Define clear contracts; use API versioning |
| **Behavioral coupling** | Service A dictates the workflow of Service B | Choreography over orchestration where possible |

### Coupling Indicators

```
Low Coupling (good)                    High Coupling (bad)
┌──────────┐     ┌──────────┐         ┌──────────┐     ┌──────────┐
│ Order    │     │ Payment  │         │ Order    │     │ Payment  │
│ Service  │────▶│ Service  │         │ Service  │────▶│ Service  │
└──────────┘     └──────────┘         └──────────┘     └──────────┘
 Event:            Reacts to               │                 │
 OrderPlaced       event                   └────────┬────────┘
                   independently                    │
                                              ┌─────▼─────┐
                                              │ Shared DB  │
                                              └───────────┘
```

## High Cohesion

Related functionality should be grouped together within a single service. If two pieces of logic always change together, they belong in the same service.

### Cohesion Test

Ask these questions to evaluate service cohesion:

1. **Does all the code in this service change for the same reason?** If not, split it.
2. **Would removing any part of this service make it incomplete?** If so, keep it together.
3. **Do the data entities in this service belong to the same aggregate?** If not, consider separating them.

```
High Cohesion ✅                      Low Cohesion ❌
┌────────────────────┐               ┌────────────────────┐
│   Payment Service  │               │   Utility Service  │
│                    │               │                    │
│ - CreatePayment()  │               │ - SendEmail()      │
│ - RefundPayment()  │               │ - GenerateReport() │
│ - GetPaymentStatus │               │ - ValidateAddress()│
│ - ListPayments()   │               │ - ConvertCurrency()│
│                    │               │                    │
│ All about payments │               │ Unrelated functions│
└────────────────────┘               └────────────────────┘
```

## Domain-Driven Design

Domain-Driven Design (DDD) provides the conceptual framework for identifying service boundaries in microservices architecture.

### Key DDD Concepts for Microservices

| Concept | Description | Microservices Relevance |
|---|---|---|
| **Ubiquitous Language** | A shared vocabulary between developers and domain experts | Each service speaks the language of its domain |
| **Bounded Context** | A boundary within which a domain model is consistent | Maps directly to a microservice boundary |
| **Aggregate** | A cluster of entities treated as a single unit | Defines the consistency boundary within a service |
| **Domain Event** | A record that something meaningful happened in the domain | The primary mechanism for inter-service communication |
| **Context Map** | A diagram showing relationships between bounded contexts | Guides how services interact and share data |

### Strategic Design Process

```
1. Event Storming / Domain Discovery
   └──▶ Identify domain events, commands, and aggregates

2. Identify Bounded Contexts
   └──▶ Group related concepts into contexts

3. Define Context Map
   └──▶ Map relationships between contexts

4. Map Contexts to Services
   └──▶ Each bounded context becomes a candidate microservice
```

## Bounded Contexts

A bounded context defines a clear boundary around a domain model. Within the boundary, terms have precise, unambiguous meaning. The same term may mean different things in different bounded contexts.

### Example: "Product" in Different Contexts

```
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│   Catalog Context    │  │  Inventory Context   │  │  Shipping Context    │
│                      │  │                      │  │                      │
│ Product:             │  │ Product:             │  │ Product:             │
│  - name              │  │  - SKU               │  │  - weight            │
│  - description       │  │  - quantity_on_hand  │  │  - dimensions        │
│  - price             │  │  - warehouse_location│  │  - shipping_class    │
│  - images            │  │  - reorder_threshold │  │  - fragile           │
│  - categories        │  │  - supplier          │  │  - origin_country    │
└──────────────────────┘  └──────────────────────┘  └──────────────────────┘
```

### Context Mapping Patterns

| Pattern | Description | Use Case |
|---|---|---|
| **Shared Kernel** | Two contexts share a small common model | Core domain types shared by closely related services |
| **Customer/Supplier** | Upstream context provides what downstream needs | Order service (customer) depends on Inventory (supplier) |
| **Anti-Corruption Layer** | Translates between contexts to prevent model leakage | Integrating with legacy systems or third-party APIs |
| **Open Host Service** | Exposes a well-defined protocol for consumers | Public API consumed by multiple downstream services |
| **Published Language** | Shared schema format for inter-context communication | Event schemas, OpenAPI specs, Protobuf definitions |

## Service Boundaries

### How to Identify Service Boundaries

1. **Start with the domain** — use event storming to discover bounded contexts
2. **Look for natural seams** — areas with minimal cross-cutting data dependencies
3. **Consider team structure** — Conway's Law suggests boundaries should align with teams
4. **Evaluate change frequency** — components that change together should live together
5. **Assess scaling requirements** — components with different load profiles may need separation

### Boundary Checklist

- [ ] The service owns its data and does not share a database
- [ ] The service can be deployed independently
- [ ] The service has a well-defined API contract
- [ ] The service maps to a single bounded context
- [ ] Changes to this service rarely require changes in other services
- [ ] The team can understand the entire service codebase

## API-First Design

Design the API contract before writing the implementation. This ensures clear boundaries and enables parallel development across teams.

### API Design Guidelines

```yaml
# OpenAPI example for an Order Service
openapi: 3.0.3
info:
  title: Order Service API
  version: 1.0.0
paths:
  /orders:
    post:
      summary: Create a new order
      operationId: createOrder
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        '201':
          description: Order created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '400':
          description: Invalid request
        '409':
          description: Duplicate order (idempotency key conflict)
```

### API Versioning Strategies

| Strategy | Example | Pros | Cons |
|---|---|---|---|
| **URI versioning** | `/api/v1/orders` | Simple, explicit | URL proliferation |
| **Header versioning** | `Accept: application/vnd.api.v1+json` | Clean URLs | Harder to test |
| **Query parameter** | `/orders?version=1` | Easy to implement | Not RESTful |

## Autonomy and Decentralization

### Decentralized Governance

Each team selects the technology stack best suited for their service. There is no mandate to use the same language, framework, or database across all services.

```
Order Service          Payment Service        Notification Service
┌────────────────┐    ┌────────────────┐     ┌────────────────┐
│  Java/Spring   │    │  Go/gRPC       │     │  Node.js       │
│  PostgreSQL    │    │  Redis         │     │  DynamoDB      │
│  Kafka         │    │  Kafka         │     │  SQS           │
└────────────────┘    └────────────────┘     └────────────────┘
```

### Decentralized Data

Each service manages its own data store. Data is shared through APIs or events, never through direct database access.

### Conway's Law Alignment

> "Organizations which design systems are constrained to produce designs which are copies of the communication structures of these organizations." — Melvin Conway

Structure your teams around business capabilities, and the service boundaries will follow naturally.

```
Team: Orders              Team: Payments           Team: Logistics
┌───────────────┐        ┌───────────────┐        ┌───────────────┐
│ Order Service │        │Payment Service│        │Shipping Svc   │
│ Cart Service  │        │Billing Service│        │Tracking Svc   │
└───────────────┘        └───────────────┘        └───────────────┘
```

## Best Practices

### Service Design

- ✅ Start with a monolith and extract services as boundaries become clear
- ✅ Use Domain-Driven Design to identify bounded contexts
- ✅ Design APIs first — agree on contracts before implementation
- ✅ Keep services small enough for one team to own, large enough to be meaningful
- ❌ Do not create a service for every entity or database table
- ❌ Do not share databases between services
- ❌ Do not build "utility" or "helper" services with unrelated functionality

### Communication

- ✅ Prefer asynchronous communication (events) over synchronous calls (REST/gRPC) between services
- ✅ Use well-defined schemas for events and APIs
- ✅ Make all inter-service calls idempotent
- ❌ Do not create long chains of synchronous calls (A → B → C → D)
- ❌ Do not build chatty interfaces that require many calls for a single operation

### Data Management

- ✅ Each service owns its data — no shared databases
- ✅ Use events to propagate state changes between services
- ✅ Accept eventual consistency where possible
- ❌ Do not implement distributed transactions across services
- ❌ Do not synchronously query another service's data for every request

## Next Steps

Continue to [Architecture Patterns](02-ARCHITECTURE-PATTERNS.md) to learn about API gateways, Backend-for-Frontend, service mesh, sidecar proxies, and the strangler fig migration pattern.
