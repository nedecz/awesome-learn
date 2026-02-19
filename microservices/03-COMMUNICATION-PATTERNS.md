# Microservices Communication Patterns

## Table of Contents

1. [Overview](#overview)
2. [Synchronous Communication](#synchronous-communication)
   - [REST/HTTP](#resthttp)
   - [gRPC](#grpc)
   - [GraphQL](#graphql)
3. [Asynchronous Communication](#asynchronous-communication)
   - [Message Queues](#message-queues)
   - [Event Streaming](#event-streaming)
   - [Pub/Sub](#pubsub)
4. [Choreography vs. Orchestration](#choreography-vs-orchestration)
5. [Event-Driven Architecture](#event-driven-architecture)
6. [Choosing the Right Pattern](#choosing-the-right-pattern)
7. [Best Practices](#best-practices)
8. [Next Steps](#next-steps)

## Overview

This document covers the communication patterns used between microservices. Choosing the right pattern for each interaction is critical for system reliability, performance, and maintainability.

### Target Audience

- Developers implementing inter-service communication
- Architects selecting communication protocols and patterns
- Platform engineers building messaging infrastructure

### Scope

- Synchronous patterns: REST, gRPC, GraphQL
- Asynchronous patterns: message queues, event streams, pub/sub
- Choreography vs. orchestration
- Event-driven architecture fundamentals
- Choosing the right pattern for each interaction

## Synchronous Communication

In synchronous communication, the caller sends a request and waits for a response. The caller is blocked until the downstream service replies.

```
Synchronous Flow

  ┌──────────┐         ┌──────────┐
  │ Service A│───req──▶│ Service B│
  │ (blocked)│◀──res───│          │
  └──────────┘         └──────────┘

  Service A waits for Service B to respond.
```

### REST/HTTP

REST (Representational State Transfer) over HTTP is the most common synchronous communication pattern.

```
# Example REST endpoints for an Order Service

GET    /api/v1/orders              # List orders
GET    /api/v1/orders/{id}         # Get order by ID
POST   /api/v1/orders              # Create order
PUT    /api/v1/orders/{id}         # Update order
PATCH  /api/v1/orders/{id}/status  # Partial update
DELETE /api/v1/orders/{id}         # Cancel order
```

#### HTTP Status Codes

| Code | Meaning | When to Use |
|---|---|---|
| `200` | OK | Successful GET, PUT, PATCH |
| `201` | Created | Successful POST (resource created) |
| `202` | Accepted | Request accepted for async processing |
| `204` | No Content | Successful DELETE |
| `400` | Bad Request | Validation error in request body |
| `401` | Unauthorized | Missing or invalid authentication |
| `403` | Forbidden | Authenticated but not authorized |
| `404` | Not Found | Resource does not exist |
| `409` | Conflict | Duplicate or conflicting state |
| `429` | Too Many Requests | Rate limit exceeded |
| `500` | Internal Server Error | Unhandled server error |
| `503` | Service Unavailable | Service is temporarily down |

### gRPC

gRPC is a high-performance RPC framework that uses Protocol Buffers for serialization and HTTP/2 for transport. It is significantly faster than REST for service-to-service communication.

```protobuf
// order_service.proto
syntax = "proto3";
package order;

service OrderService {
  rpc CreateOrder (CreateOrderRequest) returns (Order);
  rpc GetOrder (GetOrderRequest) returns (Order);
  rpc ListOrders (ListOrdersRequest) returns (stream Order);
  rpc WatchOrderStatus (WatchRequest) returns (stream OrderStatus);
}

message CreateOrderRequest {
  string customer_id = 1;
  repeated OrderItem items = 2;
  string currency = 3;
}

message Order {
  string id = 1;
  string customer_id = 2;
  string status = 3;
  int64 total_cents = 4;
  google.protobuf.Timestamp created_at = 5;
}
```

#### REST vs. gRPC

| Dimension | REST | gRPC |
|---|---|---|
| **Protocol** | HTTP/1.1 or HTTP/2 | HTTP/2 (required) |
| **Serialization** | JSON (text) | Protocol Buffers (binary) |
| **Performance** | Good | Excellent (2–10x faster) |
| **Streaming** | Limited (SSE, WebSocket) | Native bidirectional streaming |
| **Browser support** | Native | Requires gRPC-Web proxy |
| **Contract** | OpenAPI (optional) | .proto files (required) |
| **Best for** | Public APIs, external clients | Internal service-to-service |

### GraphQL

GraphQL allows clients to request exactly the data they need. It is typically used as a BFF or API gateway pattern rather than for direct service-to-service communication.

```graphql
# Client query — only fetch what's needed
query GetOrderSummary($orderId: ID!) {
  order(id: $orderId) {
    id
    status
    total
    items {
      name
      quantity
    }
    customer {
      name
      email
    }
  }
}
```

## Asynchronous Communication

In asynchronous communication, the sender publishes a message and does not wait for an immediate response. This decouples services in time and reduces temporal coupling.

```
Asynchronous Flow

  ┌──────────┐         ┌───────────┐         ┌──────────┐
  │ Service A│──msg──▶ │  Message  │ ──msg──▶│ Service B│
  │ (returns │         │  Broker   │         │ (eventual│
  │  immediately)      │           │         │  processing)
  └──────────┘         └───────────┘         └──────────┘

  Service A publishes and moves on. Service B processes when ready.
```

### Message Queues

Message queues provide point-to-point communication. A producer sends a message to a queue, and exactly one consumer processes it.

```
Point-to-Point (Queue)

  Producer ──▶ [ Queue ] ──▶ Consumer

  Each message is processed by exactly one consumer.
  Messages are removed from the queue after processing.

  ┌──────────┐     ┌──────────────┐     ┌──────────┐
  │ Order    │────▶│  order-queue  │────▶│ Payment  │
  │ Service  │     │               │     │ Service  │
  └──────────┘     │ msg1, msg2,  │     └──────────┘
                   │ msg3          │
                   └──────────────┘
```

| Queue Technology | Key Features |
|---|---|
| **RabbitMQ** | Flexible routing, AMQP protocol, exchange types |
| **Amazon SQS** | Fully managed, dead-letter queues, FIFO support |
| **Redis Streams** | In-memory, low latency, consumer groups |

### Event Streaming

Event streaming platforms maintain an ordered, durable log of events. Multiple consumers can read from the same stream independently.

```
Event Stream (Log)

  Producer ──▶ [ Topic: orders ] ──▶ Consumer A
                                 ──▶ Consumer B
                                 ──▶ Consumer C

  Each consumer reads independently.
  Events are retained (not deleted after reading).

  ┌──────────┐     ┌────────────────────────┐
  │ Order    │────▶│  orders topic          │
  │ Service  │     │                        │
  └──────────┘     │ offset 0: OrderCreated │
                   │ offset 1: OrderPaid    │──▶ Analytics Service
                   │ offset 2: OrderShipped │──▶ Notification Service
                   │ offset 3: OrderCreated │──▶ Inventory Service
                   └────────────────────────┘
```

| Platform | Key Features |
|---|---|
| **Apache Kafka** | High throughput, durable, partitioned, exactly-once semantics |
| **Amazon Kinesis** | Managed, auto-scaling, AWS-native |
| **Apache Pulsar** | Multi-tenancy, geo-replication, tiered storage |
| **Redpanda** | Kafka-compatible, no JVM, lower operational overhead |

### Pub/Sub

Pub/Sub (publish-subscribe) decouples producers from consumers. Producers publish messages to topics, and all subscribers to that topic receive a copy.

```
Pub/Sub

  Publisher ──▶ [ Topic ] ──▶ Subscriber A (gets a copy)
                           ──▶ Subscriber B (gets a copy)
                           ──▶ Subscriber C (gets a copy)

  All subscribers receive every message.
```

| Pub/Sub Platform | Key Features |
|---|---|
| **Amazon SNS** | Managed, integrates with SQS, Lambda, HTTP |
| **Google Cloud Pub/Sub** | Global, exactly-once delivery, dead-letter topics |
| **Azure Service Bus** | Topics, sessions, message deferral |

## Choreography vs. Orchestration

### Choreography

Each service reacts to events and produces new events. No central coordinator.

```
Choreography — Event-Driven

  ┌──────────┐  OrderPlaced  ┌──────────┐  PaymentProcessed  ┌──────────┐
  │  Order   │──────────────▶│ Payment  │───────────────────▶│ Shipping │
  │  Service │               │  Service │                    │  Service │
  └──────────┘               └──────────┘                    └──────────┘
       │                          │                               │
       │                          │                               │
       ▼                          ▼                               ▼
  OrderPlaced              PaymentProcessed                OrderShipped
    (event)                   (event)                        (event)
```

**Pros:** Loose coupling, services evolve independently, no single point of failure
**Cons:** Harder to track the overall workflow, debugging distributed flows is complex

### Orchestration

A central orchestrator coordinates the workflow by sending commands to each service.

```
Orchestration — Command-Driven

                    ┌──────────────┐
                    │ Order Saga   │
                    │ Orchestrator │
                    └──┬───┬────┬──┘
         CreatePayment │   │    │ CreateShipment
                       ▼   │    ▼
                ┌────────┐ │ ┌──────────┐
                │Payment │ │ │ Shipping │
                │Service │ │ │ Service  │
                └────────┘ │ └──────────┘
                           │ UpdateInventory
                           ▼
                    ┌────────────┐
                    │ Inventory  │
                    │ Service    │
                    └────────────┘
```

**Pros:** Easy to understand the full workflow, centralized error handling, clear state management
**Cons:** Orchestrator becomes a coupling point, single point of failure

### When to Choose Each

| Factor | Choreography | Orchestration |
|---|---|---|
| **Coupling** | Low | Higher (services depend on orchestrator) |
| **Visibility** | Hard to trace full flow | Easy — the orchestrator knows the full flow |
| **Error handling** | Each service handles its own errors | Centralized compensation logic |
| **Best for** | Simple event reactions, fire-and-forget | Complex multi-step transactions (sagas) |

## Event-Driven Architecture

### Event Types

| Type | Purpose | Example |
|---|---|---|
| **Domain Events** | Record something that happened in the domain | `OrderPlaced`, `PaymentProcessed` |
| **Integration Events** | Communicate between bounded contexts | `CustomerUpdated` (shared across services) |
| **Command Events** | Request an action from another service | `ProcessPayment`, `ShipOrder` |

### Event Schema Design

```json
{
  "eventId": "evt-abc123",
  "eventType": "OrderPlaced",
  "aggregateId": "order-789",
  "aggregateType": "Order",
  "timestamp": "2025-01-15T10:30:00Z",
  "version": 1,
  "data": {
    "orderId": "order-789",
    "customerId": "cust-456",
    "items": [
      { "productId": "prod-1", "quantity": 2, "price": 29.99 }
    ],
    "totalAmount": 59.98,
    "currency": "USD"
  },
  "metadata": {
    "correlationId": "req-xyz789",
    "causationId": "cmd-create-order-1",
    "source": "order-service"
  }
}
```

### Idempotent Event Processing

Events may be delivered more than once. Consumers must be idempotent — processing the same event multiple times should produce the same result.

```
Idempotency Strategies:
├── Track processed event IDs in a deduplication table
├── Use natural idempotency (e.g., setting a status is idempotent)
├── Use database constraints (unique keys on event ID + aggregate ID)
└── Leverage message broker features (Kafka exactly-once, SQS dedup)
```

## Choosing the Right Pattern

| Interaction Type | Recommended Pattern | Reason |
|---|---|---|
| **Query data from another service** | REST or gRPC (sync) | Need immediate response |
| **Notify other services something happened** | Event/message (async) | Fire-and-forget; decouple services |
| **Multi-step business process** | Saga (choreography or orchestration) | Manage distributed transactions |
| **High-throughput internal communication** | gRPC | Binary protocol, streaming, performance |
| **Public API for external clients** | REST | Universal support, human-readable |
| **Frontend data aggregation** | GraphQL or BFF | Reduce over/under-fetching |
| **Real-time data processing** | Event streaming (Kafka) | Durable, replayable, high throughput |

## Best Practices

### General

- ✅ Prefer asynchronous communication between services to reduce temporal coupling
- ✅ Design all inter-service calls to be idempotent
- ✅ Include correlation IDs in all requests and events for distributed tracing
- ✅ Version your APIs and event schemas
- ❌ Do not create synchronous call chains longer than 2–3 hops
- ❌ Do not share internal data models across services — use dedicated DTOs

### Event Design

- ✅ Use past-tense names for events (`OrderPlaced`, not `PlaceOrder`)
- ✅ Include enough data in events for consumers to act without callbacks
- ✅ Define a dead-letter queue for events that cannot be processed
- ❌ Do not put the entire entity in every event — include only what changed
- ❌ Do not rely on event ordering across different partitions or topics

## Next Steps

Continue to [Data Management](04-DATA-MANAGEMENT.md) to learn about the database-per-service pattern, saga pattern for distributed transactions, CQRS, event sourcing, and the transactional outbox pattern.
