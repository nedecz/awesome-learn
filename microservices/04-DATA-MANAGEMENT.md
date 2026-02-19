# Microservices Data Management

## Table of Contents

1. [Overview](#overview)
2. [Database per Service](#database-per-service)
3. [Saga Pattern](#saga-pattern)
4. [Transactional Outbox](#transactional-outbox)
5. [CQRS and Event Sourcing](#cqrs-and-event-sourcing)
6. [Data Consistency Strategies](#data-consistency-strategies)
7. [Data Replication and Caching](#data-replication-and-caching)
8. [Best Practices](#best-practices)
9. [Next Steps](#next-steps)

## Overview

Data management in microservices is fundamentally different from monolithic applications. Each service owns its data, and traditional ACID transactions across services are not possible. This document covers the patterns and strategies for managing data in a distributed system.

### Target Audience

- Developers designing data access patterns for microservices
- Architects making decisions about data storage and consistency
- Database engineers planning migration from shared databases

### Scope

- Database per service pattern and polyglot persistence
- Saga pattern for distributed transactions
- Transactional outbox for reliable event publishing
- CQRS and event sourcing for read/write separation
- Eventual consistency and conflict resolution

## Database per Service

Each microservice owns and manages its own data store. No other service may directly access another service's database.

### Why Database per Service

```
✅ Database per Service               ❌ Shared Database

┌──────────┐    ┌──────────┐         ┌──────────┐    ┌──────────┐
│ Order    │    │ Payment  │         │ Order    │    │ Payment  │
│ Service  │    │ Service  │         │ Service  │    │ Service  │
└────┬─────┘    └────┬─────┘         └────┬─────┘    └────┬─────┘
     │               │                    │               │
┌────▼─────┐    ┌────▼─────┐              └───────┬───────┘
│ Order DB │    │Payment DB│              ┌───────▼───────┐
│(Postgres)│    │ (Redis)  │              │  Shared DB    │
└──────────┘    └──────────┘              │  (coupling!)  │
                                          └───────────────┘
 Independent schema                       Schema changes
 evolution. Each service                  affect multiple
 picks the best database                 services. Tight
 for its workload.                       coupling on data.
```

### Polyglot Persistence

Different services can use different database technologies optimized for their specific workload.

| Service | Database | Reason |
|---|---|---|
| **Order Service** | PostgreSQL | Complex queries, ACID transactions |
| **Product Catalog** | Elasticsearch | Full-text search, faceted filtering |
| **Session Store** | Redis | Low-latency key-value lookups |
| **Activity Log** | MongoDB | Flexible schema, high write throughput |
| **Social Graph** | Neo4j | Relationship queries, graph traversal |
| **Time Series** | TimescaleDB | Time-ordered metrics, efficient compression |

## Saga Pattern

A saga is a sequence of local transactions across multiple services. If one step fails, compensating transactions are executed to undo the completed steps.

### Choreography Saga

Each service publishes an event after completing its local transaction. The next service reacts to that event.

```
Happy Path:

Order Service         Payment Service       Inventory Service
     │                      │                      │
     │ OrderCreated         │                      │
     ├─────────────────────▶│                      │
     │                      │ PaymentProcessed     │
     │                      ├─────────────────────▶│
     │                      │                      │ InventoryReserved
     │◀─────────────────────┼──────────────────────┤
     │ (Order Confirmed)    │                      │


Compensation (Payment fails):

Order Service         Payment Service       Inventory Service
     │                      │                      │
     │ OrderCreated         │                      │
     ├─────────────────────▶│                      │
     │                      │ PaymentFailed        │
     │◀─────────────────────┤                      │
     │ (Cancel Order)       │                      │
```

### Orchestration Saga

A central orchestrator controls the saga by sending commands to each service and handling responses.

```
┌──────────────────┐
│  Order Saga      │
│  Orchestrator    │
│                  │
│  State Machine:  │
│  1. CREATED      │
│  2. PAYMENT_PENDING
│  3. INVENTORY_PENDING
│  4. CONFIRMED    │
│  5. FAILED       │
└────┬──┬──┬───────┘
     │  │  │
     │  │  └──── ReserveInventory ──▶ Inventory Service
     │  │            ◀── InventoryReserved / InventoryFailed
     │  │
     │  └───── ProcessPayment ──▶ Payment Service
     │              ◀── PaymentProcessed / PaymentFailed
     │
     └──── (Compensate if any step fails)
```

### Choosing Choreography vs. Orchestration

| Factor | Choreography | Orchestration |
|---|---|---|
| **Number of steps** | 2–3 steps | 4+ steps |
| **Complexity** | Low workflow complexity | Complex branching and compensation |
| **Coupling** | Loosely coupled | Orchestrator couples to all participants |
| **Debugging** | Harder to trace | Easier — state machine tracks progress |

## Transactional Outbox

The transactional outbox pattern ensures reliable event publishing by writing events to an outbox table within the same database transaction as the business data change.

### Problem

```
❌ Dual-write problem (unsafe)

  ┌──────────┐     1. Write to DB      ┌──────┐
  │ Service  │ ──────────────────────▶  │  DB  │  ✅ Success
  └────┬─────┘                          └──────┘
       │
       │           2. Publish event      ┌────────┐
       └──────────────────────────────▶  │ Broker │  ❌ May fail!
                                         └────────┘

  If step 2 fails, the database has the data but no event is published.
  The system is now in an inconsistent state.
```

### Solution

```
✅ Transactional Outbox

  ┌──────────┐     1. Single transaction  ┌──────────────────┐
  │ Service  │ ──────────────────────────▶ │  Database         │
  └──────────┘                             │  ┌─────────────┐ │
                                           │  │ orders      │ │
                                           │  │ (business)  │ │
                                           │  ├─────────────┤ │
                                           │  │ outbox      │ │
                                           │  │ (events)    │ │
                                           │  └─────────────┘ │
                                           └────────┬─────────┘
                                                    │
                   ┌──────────────────┐             │ 2. Relay polls
                   │  Message Relay   │◀────────────┘    or uses CDC
                   │  (Debezium, CDC) │
                   └────────┬─────────┘
                            │ 3. Publishes to broker
                            ▼
                   ┌──────────────────┐
                   │  Message Broker  │
                   └──────────────────┘
```

### Outbox Table Schema

```sql
CREATE TABLE outbox (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(255) NOT NULL,
    aggregate_id   VARCHAR(255) NOT NULL,
    event_type     VARCHAR(255) NOT NULL,
    payload        JSONB NOT NULL,
    created_at     TIMESTAMP NOT NULL DEFAULT NOW(),
    published_at   TIMESTAMP,
    retries        INT DEFAULT 0
);

-- Index for polling unpublished events
CREATE INDEX idx_outbox_unpublished ON outbox (created_at)
  WHERE published_at IS NULL;
```

## CQRS and Event Sourcing

### CQRS in Microservices

CQRS separates the write (command) side from the read (query) side. Each side can use a different data model, database, and scaling strategy.

```
              ┌────────────────┐
              │   API Gateway  │
              └───┬────────┬───┘
                  │        │
         Commands │        │ Queries
                  ▼        ▼
        ┌──────────┐  ┌───────────┐
        │ Command  │  │  Query    │
        │ Service  │  │  Service  │
        └────┬─────┘  └─────┬─────┘
             │              │
        ┌────▼─────┐  ┌─────▼────┐
        │ Write DB │  │ Read DB  │
        │(Postgres)│  │(Elastic  │
        └────┬─────┘  │ Search)  │
             │        └─────▲────┘
             │              │
             └── Events ────┘
             (project to read model)
```

### Event Sourcing with Microservices

Instead of storing current state, store every state change as an immutable event. Rebuild current state by replaying events.

```
Event Store (append-only):

  ┌───────────────────────────────────────────────┐
  │ Stream: Order-123                              │
  │                                                │
  │  v1: OrderCreated    { items: [...], total: 99}│
  │  v2: ItemAdded       { productId: "abc" }      │
  │  v3: PaymentReceived { amount: 99 }            │
  │  v4: OrderShipped    { trackingNo: "UPS123" }  │
  └───────────────────────────────────────────────┘

  Current State (projection):
  {
    id: "Order-123",
    status: "Shipped",
    items: [...],
    total: 99,
    trackingNo: "UPS123"
  }
```

## Data Consistency Strategies

### Eventual Consistency

In a microservices system, strong consistency across services is impractical. Instead, the system achieves consistency eventually — after all events have been processed.

```
Strong Consistency (monolith):
  All data is consistent at all times within a single transaction.

Eventual Consistency (microservices):
  Data across services converges to a consistent state over time.

  T0: Order Service creates order (status: CREATED)
  T1: Payment Service processes payment (event: PaymentProcessed)
  T2: Order Service updates order (status: PAID) ← eventually consistent
```

### Conflict Resolution Strategies

| Strategy | Description | Use Case |
|---|---|---|
| **Last-write-wins** | Most recent update takes precedence | Non-critical data (user preferences) |
| **Version vectors** | Track versions per service/replica | Collaborative editing scenarios |
| **Merge functions** | Custom logic to combine conflicting updates | Shopping cart merges |
| **Operational transforms** | Transform concurrent operations | Real-time collaboration |

## Data Replication and Caching

### Read Replicas via Events

Services can maintain local read-only copies of data from other services by consuming their events.

```
┌──────────┐    CustomerUpdated    ┌──────────┐
│ Customer │ ────────────────────▶ │  Order   │
│ Service  │       (event)         │  Service │
└────┬─────┘                       └────┬─────┘
     │                                  │
┌────▼──────┐                      ┌────▼──────┐
│ Customer  │                      │ Order DB  │
│    DB     │                      │           │
│           │                      │ customers │
│ customers │                      │ (local    │
│ (source   │                      │  replica) │
│  of truth)│                      └───────────┘
└───────────┘
```

### Caching Strategies

| Strategy | Description | Consistency |
|---|---|---|
| **Cache-aside** | Application checks cache first, loads from DB on miss | Stale reads possible |
| **Write-through** | Write to cache and DB simultaneously | Strong consistency |
| **Write-behind** | Write to cache immediately, async write to DB | Eventually consistent |
| **Event-driven invalidation** | Invalidate cache when events arrive | Eventually consistent |

## Best Practices

### Data Ownership

- ✅ Each service owns exactly one data store — no sharing
- ✅ Expose data only through APIs or events — never through direct DB access
- ✅ Use the right database for each service's workload (polyglot persistence)
- ❌ Do not join data across service databases
- ❌ Do not share database schemas, tables, or connection strings

### Transactions

- ✅ Use sagas for distributed transactions across services
- ✅ Implement the transactional outbox for reliable event publishing
- ✅ Design operations to be idempotent for safe retries
- ❌ Do not use distributed two-phase commit (2PC) across microservices
- ❌ Do not assume events will arrive in order

### Consistency

- ✅ Embrace eventual consistency — design UIs and workflows to accommodate it
- ✅ Use compensation logic instead of rollbacks
- ✅ Include event IDs and timestamps for deduplication and ordering
- ❌ Do not force strong consistency where it is not required by the business

## Next Steps

Continue to [Service Discovery](05-SERVICE-DISCOVERY.md) to learn about how microservices find and communicate with each other, including client-side discovery, server-side discovery, DNS-based approaches, and service registries.
