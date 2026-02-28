# Messaging Patterns

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Publish-Subscribe](#publish-subscribe)
   - [Fan-Out](#fan-out)
   - [Topic-Based Routing](#topic-based-routing)
   - [Event Notification](#event-notification)
3. [Point-to-Point](#point-to-point)
   - [Work Queues](#work-queues)
   - [Competing Consumers](#competing-consumers)
   - [Load Leveling](#load-leveling)
4. [Request-Reply](#request-reply)
   - [Synchronous Over Async](#synchronous-over-async)
   - [Correlation IDs](#correlation-ids)
   - [Reply Queues](#reply-queues)
5. [Fan-Out and Fan-In](#fan-out-and-fan-in)
   - [Broadcasting](#broadcasting)
   - [Aggregation](#aggregation)
   - [Scatter-Gather](#scatter-gather)
6. [Dead Letter Queues](#dead-letter-queues)
   - [Handling Failed Messages](#handling-failed-messages)
   - [Poison Pill Detection](#poison-pill-detection)
   - [Retry Strategies](#retry-strategies)
7. [Message Routing](#message-routing)
   - [Content-Based Routing](#content-based-routing)
   - [Header-Based Routing](#header-based-routing)
   - [Routing Slip](#routing-slip)
8. [Saga Pattern](#saga-pattern)
   - [Choreography vs Orchestration](#choreography-vs-orchestration)
   - [Compensating Transactions](#compensating-transactions)
9. [Event Sourcing](#event-sourcing)
   - [Event Store](#event-store)
   - [Projections](#projections)
   - [Snapshots](#snapshots)
   - [Replaying Events](#replaying-events)
10. [CQRS with Messaging](#cqrs-with-messaging)
    - [Command and Query Separation](#command-and-query-separation)
    - [Eventual Consistency](#eventual-consistency)
11. [Outbox Pattern](#outbox-pattern)
    - [Transactional Outbox](#transactional-outbox)
    - [Change Data Capture](#change-data-capture)
    - [Reliable Publishing](#reliable-publishing)
12. [Choosing the Right Pattern](#choosing-the-right-pattern)
    - [Decision Matrix](#decision-matrix)
    - [Combining Patterns](#combining-patterns)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

---

## Overview

Messaging patterns are reusable design solutions for communication between distributed system components. They address challenges such as decoupling, reliability, scalability, and ordering. Selecting the right pattern — or combining multiple patterns — is critical for building resilient event-driven architectures.

### Target Audience

- Backend and platform engineers designing distributed systems
- Architects evaluating asynchronous communication strategies
- Developers working with message brokers (Kafka, RabbitMQ, cloud services)
- Teams migrating from synchronous to event-driven architectures

### Scope

This document covers the most widely adopted messaging patterns, their trade-offs, and practical guidance for implementation. It is **broker-agnostic** — patterns apply whether you use Apache Kafka, RabbitMQ, AWS SQS/SNS, Azure Service Bus, or Google Cloud Pub/Sub. For broker-specific details, see the companion files listed in [Next Steps](#next-steps).

---

## Publish-Subscribe

In the **publish-subscribe** (pub/sub) pattern, producers publish messages to a topic and zero or more subscribers receive a copy of each message. Publishers and subscribers are fully decoupled — the publisher does not know how many subscribers exist or what they do with the message.

```
Publish-Subscribe Pattern
─────────────────────────

                        ┌──────────────┐
                        │    Topic     │
  ┌──────────┐          │  (orders)    │
  │ Publisher │────────▶ │              │
  └──────────┘          └──────┬───────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                 ▼
       ┌────────────┐  ┌────────────┐   ┌────────────┐
       │ Subscriber │  │ Subscriber │   │ Subscriber │
       │   (Email)  │  │ (Billing)  │   │ (Analytics)│
       └────────────┘  └────────────┘   └────────────┘

  Each subscriber receives its own copy of every message.
```

### Fan-Out

Fan-out is the core mechanism of pub/sub: a single published message is delivered to multiple subscribers. This enables independent processing pipelines to react to the same event without coordination.

**Key characteristics:**

- Subscribers process messages independently and at their own pace
- Adding a new subscriber requires no changes to the publisher
- Each subscriber maintains its own offset or acknowledgment state
- Failure in one subscriber does not affect others

### Topic-Based Routing

Topic-based routing allows subscribers to filter messages by subscribing to specific topics or using topic hierarchies. This prevents subscribers from receiving irrelevant messages.

| Approach | Example | Broker Support |
|----------|---------|---------------|
| **Flat topics** | `order.created`, `order.shipped` | All brokers |
| **Hierarchical topics** | `orders/region/us-east/created` | RabbitMQ (routing keys), MQTT |
| **Subscription filters** | SQL-like filter on message properties | Azure Service Bus, AWS SNS |
| **Tag-based filtering** | Filter by message tags/attributes | GCP Pub/Sub, AWS SNS |

### Event Notification

Event notification is a lightweight variant of pub/sub where the message contains only a reference to the changed data rather than the full payload. Consumers retrieve the current state from the source system when needed.

```
Event Notification vs Event-Carried State Transfer
───────────────────────────────────────────────────

  Event Notification:
  { "event": "order.updated", "orderId": "12345" }
  ──▶ Consumer calls Order API to get current state

  Event-Carried State Transfer:
  { "event": "order.updated", "orderId": "12345",
    "status": "shipped", "trackingNo": "ABC-789" }
  ──▶ Consumer has all data it needs in the message
```

**Trade-offs:**

- **Notification**: Smaller messages, always-current data, but adds coupling to source API
- **State transfer**: Larger messages, self-contained, enables fully autonomous consumers

---

## Point-to-Point

In the **point-to-point** pattern, each message is consumed by exactly one consumer. Messages are placed on a queue and delivered to a single receiver, making this pattern ideal for work distribution.

```
Point-to-Point Pattern
──────────────────────

  ┌──────────┐       ┌───────────────────────┐       ┌──────────┐
  │ Producer │──────▶│     Queue             │──────▶│ Consumer │
  └──────────┘       │  ┌───┬───┬───┬───┐    │       └──────────┘
                     │  │ 4 │ 3 │ 2 │ 1 │    │
                     │  └───┴───┴───┴───┘    │
                     └───────────────────────┘

  Each message is delivered to exactly one consumer.
```

### Work Queues

A work queue distributes tasks across multiple worker processes. Each task is delivered to exactly one worker, providing a simple mechanism for parallelizing CPU-intensive or I/O-bound work.

**Common use cases:**

- Image or video processing pipelines
- PDF generation from templates
- Sending transactional emails or SMS
- Running background data imports

### Competing Consumers

The **competing consumers** pattern scales message processing by having multiple consumers read from the same queue. The broker distributes messages across consumers, typically using round-robin or prefetch-based allocation.

```
Competing Consumers
───────────────────

  ┌──────────┐       ┌─────────────┐       ┌────────────┐
  │ Producer │──────▶│             │──────▶│ Consumer 1 │
  └──────────┘       │   Queue     │       └────────────┘
  ┌──────────┐       │             │       ┌────────────┐
  │ Producer │──────▶│  [M5][M4]   │──────▶│ Consumer 2 │
  └──────────┘       │  [M3][M2]   │       └────────────┘
                     │  [M1]       │       ┌────────────┐
                     │             │──────▶│ Consumer 3 │
                     └─────────────┘       └────────────┘

  Messages are distributed across consumers.
  No message is processed by more than one consumer.
```

**Scaling guidance:**

| Factor | Recommendation |
|--------|---------------|
| **Consumer count** | Match the number of consumers to the expected throughput |
| **Prefetch count** | Set prefetch to balance throughput and fairness (start with 1, increase if needed) |
| **Acknowledgment** | Use manual acknowledgment to prevent message loss on consumer failure |
| **Ordering** | Competing consumers do **not** guarantee ordering; use partitions or sessions if order matters |

### Load Leveling

Load leveling uses a queue as a buffer between producers and consumers, absorbing traffic spikes that would overwhelm downstream services.

```
Load Leveling
─────────────

  Traffic                     Queue                    Processing
  (bursty)                    (buffer)                 (steady)

  ┌──┐                                                 ┌──┐
  │  │  ┌──┐               ┌──────────────┐            │  │ ┌──┐
  │  │  │  │ ┌──┐          │ Absorbs      │            │  │ │  │ ┌──┐
  │  │  │  │ │  │          │ bursts       │────────▶   │  │ │  │ │  │
  │  │  │  │ │  │  ──────▶ │              │            │  │ │  │ │  │
  └──┘  └──┘ └──┘          └──────────────┘            └──┘ └──┘ └──┘
  Time ─────▶                                          Time ─────▶

  The queue smooths bursty input into steady output.
```

**Benefits:**

- Downstream services can be scaled independently of traffic patterns
- Prevents cascading failures during traffic spikes
- Allows producers to continue sending even when consumers are temporarily unavailable
- Reduces the need to over-provision consumer infrastructure for peak load

---

## Request-Reply

The **request-reply** pattern implements synchronous-style communication over asynchronous messaging. The requester sends a message and waits for a correlated response on a dedicated reply channel.

### Synchronous Over Async

This pattern is useful when a caller needs a response but still wants the benefits of message-based communication (decoupling, load leveling, protocol bridging).

```
Request-Reply Pattern
─────────────────────

  ┌───────────┐  1. Send request     ┌──────────────┐  2. Process
  │           │─────────────────────▶│              │─────────────┐
  │ Requester │   Request Queue      │  Responder   │             │
  │           │◀─────────────────────│              │◀────────────┘
  └───────────┘  4. Receive reply    └──────────────┘  3. Send reply
                   Reply Queue
```

### Correlation IDs

A **correlation ID** is a unique identifier attached to both the request and its reply. It allows the requester to match incoming replies to outstanding requests, especially when multiple requests are in flight.

**Implementation:**

- Generate a unique correlation ID (UUID) for each request
- Include the correlation ID in the request message headers
- The responder copies the correlation ID into the reply message
- The requester filters incoming replies by correlation ID

```
Correlation Flow
────────────────

  Requester                                 Responder
  ─────────                                 ─────────
  Generate correlationId = "abc-123"
  Send { correlationId: "abc-123",    ────▶ Receive request
         body: "get-price" }                Process request
                                            Send { correlationId: "abc-123",
  Receive reply                       ◀──── body: "$42.00" }
  Match correlationId "abc-123"
  to pending request
```

### Reply Queues

The reply channel can be implemented as a **shared reply queue** or as a **temporary exclusive queue** per requester.

| Approach | Pros | Cons |
|----------|------|------|
| **Shared reply queue** | Fewer resources, simpler topology | Requires correlation ID filtering, all replies visible to all requesters |
| **Exclusive reply queue** | No filtering needed, better isolation | One queue per requester instance, more broker resources |
| **Temporary queue** | Auto-deleted when requester disconnects | May cause message loss if requester restarts |

> **Tip:** For high-throughput systems, use a shared reply queue with correlation ID filtering. For low-volume RPC-style calls, temporary exclusive queues are simpler.

---

## Fan-Out and Fan-In

Fan-out and fan-in are complementary patterns for distributing work and aggregating results.

### Broadcasting

Broadcasting is a form of fan-out where every consumer receives every message. Unlike pub/sub with independent subscribers, broadcasting typically sends the same task to all nodes (e.g., cache invalidation).

```
Broadcasting
────────────

  ┌──────────┐       ┌──────────────┐       ┌─────────────┐
  │          │       │              │──────▶│  Node 1     │
  │ Producer │──────▶│   Exchange   │──────▶│  Node 2     │
  │          │       │  (fanout)    │──────▶│  Node 3     │
  └──────────┘       └──────────────┘       └─────────────┘

  All nodes receive the same message.
  Common use: cache invalidation, config updates.
```

### Aggregation

The **aggregator** pattern collects related messages and combines them into a single composite message. It is the complement of fan-out.

```
Aggregation Pattern
───────────────────

  ┌─────────┐
  │ Part A  │──────┐
  └─────────┘      │     ┌────────────┐     ┌──────────────┐
  ┌─────────┐      ├────▶│ Aggregator │────▶│  Composite   │
  │ Part B  │──────┤     │            │     │   Message    │
  └─────────┘      │     └────────────┘     └──────────────┘
  ┌─────────┐      │
  │ Part C  │──────┘
  └─────────┘

  The aggregator waits for all parts before emitting
  a combined result. Uses a correlation key to group
  related messages.
```

**Aggregation challenges:**

- **Completeness**: How does the aggregator know when all parts have arrived? Use a known count, a timeout, or a completion marker.
- **Ordering**: Parts may arrive out of order; the aggregator must buffer and reorder if necessary.
- **Timeouts**: If a part never arrives, the aggregator must handle the incomplete set (emit partial, retry, or discard).

### Scatter-Gather

**Scatter-gather** combines fan-out and aggregation. A request is broadcast to multiple services, and the responses are aggregated into a single reply.

```
Scatter-Gather Pattern
──────────────────────

  ┌───────────┐  scatter   ┌───────────┐   ┌────────────┐
  │           │───────────▶│ Service A │──▶│            │
  │           │            └───────────┘   │            │
  │ Requester │───────────▶┌───────────┐   │ Aggregator │──▶ Combined
  │           │            │ Service B │──▶│            │    Result
  │           │            └───────────┘   │            │
  │           │───────────▶┌───────────┐   │            │
  └───────────┘            │ Service C │──▶│            │
                           └───────────┘   └────────────┘

  Example: Price comparison across multiple vendors.
  The aggregator returns the best result or all results.
```

---

## Dead Letter Queues

A **dead letter queue** (DLQ) is a holding queue for messages that cannot be processed successfully. Instead of retrying indefinitely or silently dropping failed messages, the broker redirects them to a DLQ for investigation and remediation.

```
Dead Letter Queue Flow
──────────────────────

  ┌──────────┐      ┌─────────────┐      ┌──────────┐
  │ Producer │─────▶│  Main Queue │─────▶│ Consumer │
  └──────────┘      └──────┬──────┘      └─────┬────┘
                           │                    │
                           │  Max retries       │ Processing
                           │  exceeded          │ fails
                           │                    │
                           ▼                    │
                    ┌─────────────┐             │
                    │  Dead Letter│◀────────────┘
                    │   Queue     │   (nack / reject)
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │  Alert /    │
                    │  Dashboard  │
                    └─────────────┘
```

### Handling Failed Messages

Messages end up in a DLQ for various reasons. A structured approach to handling them is essential.

| Failure Type | Example | Remediation |
|-------------|---------|-------------|
| **Deserialization error** | Malformed JSON, schema mismatch | Fix schema, republish corrected message |
| **Validation failure** | Missing required field | Correct data at source, republish |
| **Transient error** | Database timeout, network blip | Retry after delay (should not reach DLQ if retries are configured) |
| **Business logic error** | Invalid state transition | Investigate and fix logic, then reprocess |
| **Infrastructure error** | Downstream service permanently removed | Update consumer, redirect or discard messages |

### Poison Pill Detection

A **poison pill** is a message that causes the consumer to fail every time it is processed. Without detection, the consumer enters an infinite retry loop, blocking the entire queue.

**Detection strategies:**

- **Delivery count tracking**: Most brokers track how many times a message has been delivered. Route to DLQ after a threshold (e.g., 5 attempts).
- **Consumer-side counter**: Maintain a retry counter in the message headers. Increment on each processing attempt.
- **Circuit breaker**: If the same message fails repeatedly, open the circuit breaker to prevent further attempts.

```
Poison Pill Detection
─────────────────────

  Message arrives ──▶ Process ──▶ Success ──▶ Acknowledge
                        │
                        ▼ (failure)
                    Increment delivery count
                        │
                   ┌────┴─────┐
                   │ Count <  │──▶ Requeue (retry)
                   │ max?     │
                   │          │
                   │ Count >= │──▶ Move to DLQ
                   │ max      │
                   └──────────┘
```

### Retry Strategies

Before a message reaches the DLQ, configure appropriate retry behavior.

| Strategy | Description | Use Case |
|----------|-------------|----------|
| **Immediate retry** | Requeue instantly | Rare transient errors |
| **Fixed delay** | Wait a constant interval between retries | Simple, predictable failures |
| **Exponential backoff** | Double the delay each attempt (1s, 2s, 4s, 8s…) | Transient errors, rate limiting |
| **Exponential with jitter** | Backoff + random jitter to avoid thundering herd | High-concurrency systems |
| **Retry topic/queue** | Publish to a dedicated retry queue with a delay | When the broker does not support delayed redelivery natively |

> **Best practice:** Always set a maximum retry count. Combine exponential backoff with jitter for production systems and route to DLQ after the limit is reached.

---

## Message Routing

Message routing patterns direct messages to the correct destination based on message content, headers, or a predefined itinerary.

### Content-Based Routing

A **content-based router** inspects the message body and forwards it to the appropriate channel based on business rules.

```
Content-Based Router
────────────────────

  ┌──────────┐      ┌────────────────┐
  │          │      │                │───▶ [Priority Queue]
  │ Producer │─────▶│  Router        │     (if priority == "high")
  │          │      │ (inspects body)│
  └──────────┘      │                │───▶ [Standard Queue]
                    │                │     (if priority == "normal")
                    │                │
                    │                │───▶ [Bulk Queue]
                    └────────────────┘     (if priority == "low")
```

**Implementation approaches:**

- Broker-native rules (e.g., Azure Service Bus SQL filters, AWS SNS filter policies)
- Application-level routing logic in a lightweight service
- RabbitMQ exchange bindings with routing keys

### Header-Based Routing

Header-based routing uses message metadata (headers/attributes) instead of inspecting the body. This is more performant because the broker does not need to deserialize the payload.

| Broker | Header Routing Mechanism |
|--------|------------------------|
| RabbitMQ | Headers exchange with `x-match` (all/any) |
| AWS SNS | Filter policies on message attributes |
| Azure Service Bus | SQL filter expressions on message properties |
| GCP Pub/Sub | Attribute-based subscription filters |
| Apache Kafka | Custom partitioner based on headers |

### Routing Slip

A **routing slip** attaches a list of processing steps to the message itself. Each processor in the chain reads the next step from the slip, processes the message, and forwards it.

```
Routing Slip Pattern
────────────────────

  Message with routing slip:
  { "slip": ["validate", "enrich", "transform", "store"],
    "body": { ... } }

  ┌──────────┐     ┌──────────┐     ┌───────────┐     ┌───────┐
  │ Validate │────▶│ Enrich   │────▶│ Transform │────▶│ Store │
  │          │     │          │     │           │     │       │
  │ slip[0]  │     │ slip[1]  │     │ slip[2]   │     │slip[3]│
  └──────────┘     └──────────┘     └───────────┘     └───────┘

  Each step removes itself from the slip and forwards.
  The slip can be modified dynamically (steps added/skipped).
```

**Advantages over a fixed pipeline:**

- Dynamic routing: different messages can follow different paths
- Steps can be added, removed, or reordered without changing the pipeline
- Each step is independently deployable and scalable

---

## Saga Pattern

The **saga pattern** manages distributed transactions across multiple services using a sequence of local transactions. Each step publishes an event or command that triggers the next step. If a step fails, compensating transactions undo the preceding steps.

### Choreography vs Orchestration

There are two approaches to coordinating a saga:

```
Choreography (event-driven)
───────────────────────────

  ┌─────────┐  OrderCreated  ┌─────────┐  PaymentDone  ┌───────────┐
  │ Order   │───────────────▶│ Payment │───────────────▶│ Inventory │
  │ Service │                │ Service │                │ Service   │
  └─────────┘                └─────────┘                └───────────┘
       ▲                          │                          │
       │    PaymentFailed         │                          │
       └──────────────────────────┘                          │
       │    InventoryFailed                                  │
       └─────────────────────────────────────────────────────┘

  Each service listens for events and decides what to do next.


Orchestration (command-driven)
──────────────────────────────

                    ┌──────────────┐
                    │   Saga       │
                    │ Orchestrator │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌─────────┐  ┌─────────┐  ┌───────────┐
        │ Order   │  │ Payment │  │ Inventory │
        │ Service │  │ Service │  │ Service   │
        └─────────┘  └─────────┘  └───────────┘

  The orchestrator sends commands and handles responses.
```

| Aspect | Choreography | Orchestration |
|--------|-------------|---------------|
| **Coordination** | Decentralized (events) | Centralized (orchestrator) |
| **Coupling** | Low — services only know events | Medium — orchestrator knows all steps |
| **Complexity** | Hard to trace flow across services | Easy to understand the full flow |
| **Single point of failure** | None | Orchestrator (must be highly available) |
| **Best for** | Simple sagas (2-4 steps) | Complex sagas (5+ steps, branching) |

### Compensating Transactions

When a saga step fails, earlier successful steps must be reversed using **compensating transactions**.

```
Saga with Compensation
──────────────────────

  Step 1: Create Order  ──▶  Step 2: Reserve Inventory  ──▶  Step 3: Charge Payment
       ✓                           ✓                              ✗ (fails)

  Compensate:
  Step 3 failed ──▶ Undo Step 2: Release Inventory ──▶ Undo Step 1: Cancel Order
```

**Compensation guidelines:**

- Compensating actions must be **idempotent** — safe to execute more than once
- Some actions cannot be perfectly undone (e.g., a sent email); use **semantic compensation** (send a correction email)
- Log all compensation actions for auditability
- Design compensations at the same time as the forward steps

---

## Event Sourcing

**Event sourcing** persists state as an append-only sequence of events rather than storing only the current state. The current state is derived by replaying events from the beginning (or from a snapshot).

### Event Store

The event store is the single source of truth. Each entry records an immutable fact about something that happened.

```
Event Store
────────────

  Stream: Order-12345
  ┌─────┬─────────────────────┬──────────────────────────────┐
  │ Seq │ Event Type          │ Data                         │
  ├─────┼─────────────────────┼──────────────────────────────┤
  │  1  │ OrderCreated        │ { items: [...], total: 99 }  │
  │  2  │ ItemAdded           │ { sku: "ABC", qty: 2 }       │
  │  3  │ PaymentReceived     │ { amount: 99, method: "cc" } │
  │  4  │ OrderShipped        │ { trackingNo: "TRK-789" }    │
  └─────┴─────────────────────┴──────────────────────────────┘

  Current state = replay events 1 through 4.
  Events are immutable — never updated or deleted.
```

**Event store implementations:**

| Technology | Type | Notes |
|-----------|------|-------|
| EventStoreDB | Purpose-built | Native event sourcing, projections, subscriptions |
| Apache Kafka | Log-based | Compacted topics as event streams |
| PostgreSQL | Relational | Events table with optimistic concurrency |
| DynamoDB | NoSQL | Partition key = stream, sort key = sequence |
| Azure Cosmos DB | NoSQL | Change feed for projections |

### Projections

A **projection** (also called a read model or materialization) transforms the event stream into a queryable view optimized for specific read patterns.

```
Projections
───────────

  Event Store                   Projections
  ┌────────────────┐
  │ OrderCreated   │──────────▶ ┌──────────────────┐
  │ ItemAdded      │            │ Order Summary    │  (for UI)
  │ PaymentReceived│──────────▶ │ Revenue Report   │  (for analytics)
  │ OrderShipped   │            │ Shipment Tracker │  (for ops)
  └────────────────┘            └──────────────────┘

  Each projection subscribes to events and builds
  a specialized read-optimized view.
```

- Projections can be rebuilt from scratch by replaying the entire event stream
- Multiple projections can exist for the same event stream
- Projections are eventually consistent with the event store

### Snapshots

For long event streams, replaying from the beginning is expensive. **Snapshots** capture the state at a point in time so replay starts from the snapshot instead of event #1.

```
Snapshot Optimization
─────────────────────

  Without snapshot:  Replay events 1 ─▶ 2 ─▶ 3 ─▶ ... ─▶ 10,000
                     (slow)

  With snapshot:     Load snapshot @ event 9,950
                     Replay events 9,951 ─▶ 9,952 ─▶ ... ─▶ 10,000
                     (fast — only 50 events to replay)
```

**Snapshot strategies:**

- Take a snapshot every N events (e.g., every 100)
- Take a snapshot after significant state changes
- Store snapshots alongside the event stream with the sequence number

### Replaying Events

One of the most powerful benefits of event sourcing is the ability to **replay events** to rebuild state, fix bugs, or create new projections.

**Replay use cases:**

- **Rebuild a corrupted projection**: Replay all events through the projection logic
- **Create a new projection**: Deploy new read model, replay historical events to backfill
- **Debug production issues**: Replay events locally to reproduce the exact state
- **Temporal queries**: Answer "what was the state at time T?" by replaying up to that point

> **Caution:** Replaying events against external systems (e.g., sending emails, charging credit cards) requires careful idempotency handling. Use a flag or separate replay mode to skip side effects.

---

## CQRS with Messaging

**CQRS** (Command Query Responsibility Segregation) separates the write model (commands) from the read model (queries). Messaging connects the two sides by propagating changes from the write model to the read model asynchronously.

### Command and Query Separation

```
CQRS with Messaging
────────────────────

  Commands (writes)                        Queries (reads)
  ┌──────────┐      ┌──────────────┐      ┌───────────────┐
  │  Client   │─────▶│  Command     │      │  Query        │◀──── Client
  └──────────┘      │  Handler     │      │  Handler      │
                    └──────┬───────┘      └───────┬───────┘
                           │                      │
                           ▼                      ▼
                    ┌──────────────┐      ┌───────────────┐
                    │  Write DB    │      │  Read DB      │
                    │  (normalized)│      │  (denormalized│
                    └──────┬───────┘      │   projections)│
                           │              └───────────────┘
                           │                      ▲
                           │   ┌──────────────┐   │
                           └──▶│  Message Bus  │───┘
                               │  (events)     │
                               └──────────────┘

  Write side publishes domain events.
  Read side consumes events and updates projections.
```

**Benefits of CQRS with messaging:**

- Read and write models can be scaled independently
- Read models can use storage optimized for query patterns (e.g., Elasticsearch, Redis)
- Multiple read models can serve different query needs from the same event stream
- Write side remains simple and focused on business rules

### Eventual Consistency

With CQRS and messaging, the read model is **eventually consistent** with the write model. There is a propagation delay between a successful write and the read model reflecting that change.

**Managing eventual consistency:**

| Strategy | Description |
|----------|-------------|
| **Read-your-writes** | After a write, read from the write model (or cache) for the current user's session |
| **Causal consistency** | Include a version or timestamp; reject queries for stale data |
| **UI optimistic update** | Update the UI immediately with the expected state; correct if the event differs |
| **Polling / SSE** | Client polls or subscribes for updates to detect when the read model catches up |

---

## Outbox Pattern

The **outbox pattern** ensures reliable message publishing by atomically writing the message to an outbox table in the same database transaction as the business data change. A separate process reads the outbox and publishes messages to the broker.

### Transactional Outbox

```
Transactional Outbox Pattern
─────────────────────────────

  ┌───────────────────────────────────────────────┐
  │  Database Transaction                         │
  │                                               │
  │  1. UPDATE orders SET status = 'confirmed'    │
  │  2. INSERT INTO outbox (id, topic, payload)   │
  │                                               │
  │  COMMIT (atomic — both succeed or both fail)  │
  └───────────────────────────────────────────────┘
                         │
                         ▼
  ┌─────────────────────────────────────┐
  │  Outbox Relay / Poller              │
  │                                     │
  │  SELECT * FROM outbox               │
  │    WHERE published = false           │
  │                                     │
  │  Publish to broker ──▶ Mark as      │
  │                        published    │
  └─────────────────────────────────────┘
                         │
                         ▼
                  ┌──────────────┐
                  │ Message      │
                  │ Broker       │
                  └──────────────┘
```

**Why not publish directly?**

- If the database write succeeds but the broker publish fails, data is inconsistent
- If the broker publish succeeds but the database write fails, a phantom event is published
- The outbox ensures **atomicity**: the business change and the intent to publish are committed together

### Change Data Capture

**Change data capture** (CDC) is an alternative to polling the outbox table. A CDC connector monitors the database transaction log and publishes changes to the broker automatically.

```
CDC-Based Outbox
────────────────

  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
  │  Application │      │  Database    │      │  CDC Tool    │
  │              │─────▶│  (outbox     │─────▶│  (Debezium / │
  │  Write to    │      │   table)     │      │   DMS)       │
  │  outbox      │      │              │      │              │
  └──────────────┘      └──────────────┘      └──────┬───────┘
                                                     │
                                                     ▼
                                              ┌──────────────┐
                                              │  Message     │
                                              │  Broker      │
                                              └──────────────┘

  CDC reads the database transaction log (WAL / binlog).
  No polling needed — near real-time, low overhead.
```

| Approach | Latency | Complexity | Broker Dependency |
|----------|---------|-----------|-------------------|
| **Polling** | Seconds (poll interval) | Low | None (application publishes) |
| **CDC (Debezium)** | Sub-second | Medium (connector setup) | Kafka (typical) |
| **CDC (AWS DMS)** | Seconds | Medium | Kinesis / Kafka |
| **Database triggers** | Immediate | High (fragile) | Varies |

### Reliable Publishing

The outbox pattern guarantees **at-least-once** publishing. To achieve exactly-once semantics end-to-end, consumers must be idempotent.

**Ensuring idempotency on the consumer side:**

- Store a processed message ID and check for duplicates before processing
- Use database upserts (INSERT ON CONFLICT) instead of blind inserts
- Design operations to be naturally idempotent (e.g., "set status to shipped" instead of "increment shipped count")

---

## Choosing the Right Pattern

### Decision Matrix

Use this matrix to select the appropriate pattern based on your requirements:

| Requirement | Recommended Pattern(s) |
|------------|----------------------|
| One event triggers multiple independent actions | [Publish-Subscribe](#publish-subscribe) |
| Distribute work across multiple workers | [Point-to-Point](#point-to-point), [Competing Consumers](#competing-consumers) |
| Need a synchronous response over messaging | [Request-Reply](#request-reply) |
| Broadcast to all nodes (cache invalidation) | [Fan-Out](#fan-out-and-fan-in) |
| Collect partial results into one response | [Scatter-Gather](#scatter-gather) |
| Handle messages that fail repeatedly | [Dead Letter Queues](#dead-letter-queues) |
| Route messages based on content or type | [Message Routing](#message-routing) |
| Multi-service transaction without 2PC | [Saga Pattern](#saga-pattern) |
| Full audit trail / temporal queries | [Event Sourcing](#event-sourcing) |
| Separate read and write models | [CQRS with Messaging](#cqrs-with-messaging) |
| Reliable publishing from a database | [Outbox Pattern](#outbox-pattern) |
| Absorb traffic spikes | [Load Leveling](#load-leveling) |

### Combining Patterns

In practice, production systems combine multiple patterns. Here are common combinations:

```
Common Pattern Combinations
───────────────────────────

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  Event Sourcing + CQRS + Outbox                        │
  │  ─────────────────────────────────                     │
  │  Events are the write model. CQRS projections serve    │
  │  reads. The outbox ensures events are reliably          │
  │  published to the message bus.                         │
  │                                                         │
  │  Saga + Pub/Sub + DLQ                                  │
  │  ────────────────────────                              │
  │  Saga coordinates a multi-step transaction. Pub/Sub    │
  │  connects services. DLQ catches failed steps for       │
  │  investigation.                                        │
  │                                                         │
  │  Competing Consumers + Load Leveling + Routing         │
  │  ─────────────────────────────────────────────         │
  │  A queue absorbs bursts. A router directs messages     │
  │  to the right queue. Competing consumers process       │
  │  each queue in parallel.                               │
  │                                                         │
  └─────────────────────────────────────────────────────────┘
```

**Pattern selection guidance:**

- **Start simple**: Begin with pub/sub or point-to-point. Add complexity only when needed.
- **Reliability first**: Add DLQs and the outbox pattern early — retrofitting them is harder.
- **Consider consistency**: If you need strong consistency, patterns like event sourcing and sagas add complexity but provide correctness guarantees.
- **Monitor everything**: Every pattern introduces failure modes. Instrument message counts, latencies, DLQ depths, and consumer lag from day one.

---

## Next Steps

Continue your messaging learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | Messaging Fundamentals | Core messaging concepts, delivery guarantees, event-driven architecture |
| [01-APACHE-KAFKA.md](01-APACHE-KAFKA.md) | Apache Kafka | Distributed event streaming, topics, partitions, consumer groups |
| [02-RABBITMQ.md](02-RABBITMQ.md) | RabbitMQ | Message queues, exchanges, bindings, AMQP protocol |
| [03-CLOUD-SERVICES.md](03-CLOUD-SERVICES.md) | Cloud Messaging Services | AWS SQS/SNS, Azure Service Bus, Google Cloud Pub/Sub |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial messaging patterns documentation |
