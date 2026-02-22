# Messaging Best Practices

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Topic and Queue Design](#topic-and-queue-design)
   - [Naming Conventions](#naming-conventions)
   - [Partitioning Strategies](#partitioning-strategies)
   - [Granularity](#granularity)
3. [Producer Best Practices](#producer-best-practices)
   - [Batching](#batching)
   - [Compression](#compression)
   - [Retries and Idempotent Producers](#retries-and-idempotent-producers)
   - [Error Handling](#error-handling)
4. [Consumer Best Practices](#consumer-best-practices)
   - [Consumer Groups](#consumer-groups)
   - [Offset Management](#offset-management)
   - [Graceful Shutdown](#graceful-shutdown)
   - [Processing Guarantees](#processing-guarantees)
5. [Schema and Contracts](#schema-and-contracts)
   - [Schema Evolution](#schema-evolution)
   - [Contract-First Design](#contract-first-design)
   - [Versioning Strategies](#versioning-strategies)
6. [Reliability Best Practices](#reliability-best-practices)
   - [Acknowledgments](#acknowledgments)
   - [Dead Letter Queues](#dead-letter-queues)
   - [Idempotency](#idempotency)
   - [Exactly-Once Patterns](#exactly-once-patterns)
7. [Performance Best Practices](#performance-best-practices)
   - [Throughput Optimization](#throughput-optimization)
   - [Latency Reduction](#latency-reduction)
   - [Resource Sizing](#resource-sizing)
8. [Security Best Practices](#security-best-practices)
   - [Encryption](#encryption)
   - [Authentication](#authentication)
   - [Authorization](#authorization)
   - [Network Segmentation](#network-segmentation)
9. [Operational Best Practices](#operational-best-practices)
   - [Monitoring and Alerting](#monitoring-and-alerting)
   - [Capacity Planning](#capacity-planning)
   - [Retention Policies](#retention-policies)
   - [Upgrade Strategies](#upgrade-strategies)
10. [Testing Best Practices](#testing-best-practices)
    - [Integration Testing](#integration-testing)
    - [Contract Testing](#contract-testing)
    - [Chaos Testing](#chaos-testing)
    - [Load Testing](#load-testing)
11. [Multi-Region and Disaster Recovery](#multi-region-and-disaster-recovery)
    - [Replication Strategies](#replication-strategies)
    - [Failover](#failover)
    - [Active-Active Patterns](#active-active-patterns)
12. [Production Readiness Checklist](#production-readiness-checklist)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

---

## Overview

This guide consolidates best practices for designing, building, and operating messaging systems in production. It draws from patterns proven across Kafka, RabbitMQ, cloud-managed services, and hybrid architectures. Rather than repeating fundamentals covered in earlier documents, this guide focuses on the practical decisions and trade-offs that separate a prototype from a production-grade system.

Messaging systems are the backbone of event-driven architectures. When configured well, they enable loose coupling, horizontal scalability, and resilient communication between services. When configured poorly, they become a source of data loss, silent failures, and operational pain. The practices here aim to prevent the latter.

```
┌────────────────────────────────────────────────────────────┐
│            Messaging Best Practices — Scope                 │
│                                                            │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐               │
│  │ Producer │──▶│  Broker  │──▶│ Consumer │               │
│  └──────────┘   └──────────┘   └──────────┘               │
│       │              │              │                      │
│       ▼              ▼              ▼                      │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐               │
│  │ Batching │   │ Topic &  │   │ Offset   │               │
│  │ Retries  │   │ Queue    │   │ Mgmt     │               │
│  │ Schemas  │   │ Design   │   │ Scaling  │               │
│  │ Idempot. │   │ Security │   │ Shutdown │               │
│  └──────────┘   └──────────┘   └──────────┘               │
│                                                            │
│  Cross-Cutting: Reliability │ Performance │ Security       │
│                 Operations  │ Testing     │ DR             │
└────────────────────────────────────────────────────────────┘
```

### Target Audience

- Backend engineers building or maintaining event-driven services
- Platform and infrastructure teams operating messaging clusters
- Architects evaluating messaging strategies for new systems
- SREs responsible for messaging system reliability

### Scope

This document covers broker-agnostic best practices applicable to Kafka, RabbitMQ, cloud-managed queues (SQS, Service Bus, Pub/Sub), and similar systems. Where a practice is broker-specific, the relevant platform is noted. For deep dives into individual brokers, see [01-APACHE-KAFKA.md](01-APACHE-KAFKA.md), [02-RABBITMQ.md](02-RABBITMQ.md), and [03-CLOUD-SERVICES.md](03-CLOUD-SERVICES.md).

---

## Topic and Queue Design

Good topic and queue design is foundational. Renaming or restructuring topics in production is painful and error-prone, so getting the design right early saves significant effort later.

### Naming Conventions

Use a consistent, hierarchical naming scheme that encodes the domain, event type, and optionally the version.

```
Naming Pattern:

  <domain>.<entity>.<event>[.v<version>]

Examples:

  orders.order.created.v1
  payments.payment.processed
  shipping.shipment.dispatched.v2
  inventory.stock.adjusted
```

**Naming rules:**

| Rule | Good | Bad |
|------|------|-----|
| Use lowercase with dots as separators | `orders.order.created` | `Orders_OrderCreated` |
| Include the domain/bounded context | `payments.refund.issued` | `refund-issued` |
| Use past tense for events | `order.completed` | `order.complete` |
| Avoid generic names | `orders.order.created` | `events` |
| Include version when schemas evolve | `user.profile.updated.v2` | `user.profile.updated.new` |
| Keep names stable — avoid dates or IDs | `orders.order.created` | `orders.order.created.2025` |

### Partitioning Strategies

Partitioning determines parallelism and ordering. The partition key is the most important design decision for Kafka topics.

```
┌────────────────────────────────────────────────────────────┐
│              Partition Key Selection                         │
│                                                            │
│  Goal: Co-locate related messages in the same partition    │
│                                                            │
│  Partition Key = order_id                                  │
│                                                            │
│  Order-123 events:   ┌───────────────┐                     │
│    created ─────────▶│  Partition 2  │  All events for     │
│    paid    ─────────▶│               │  Order-123 arrive   │
│    shipped ─────────▶│               │  in order.          │
│                      └───────────────┘                     │
│                                                            │
│  Order-456 events:   ┌───────────────┐                     │
│    created ─────────▶│  Partition 0  │  Order-456 events   │
│    cancelled ───────▶│               │  also arrive in     │
│                      └───────────────┘  order.             │
│                                                            │
│  Ordering guarantee: Within a partition, never across.     │
└────────────────────────────────────────────────────────────┘
```

**Choosing a partition key:**

| Strategy | Partition Key | Use When |
|----------|--------------|----------|
| Entity-based | `order_id`, `user_id` | Need ordering per entity |
| Tenant-based | `tenant_id` | Multi-tenant system, isolate tenants |
| Region-based | `region` | Regional processing requirements |
| Random (null key) | None | Maximum throughput, ordering not needed |

**Partition count guidelines:**

- Start with at least the number of consumers you expect at peak
- Kafka: partitions can be increased but never decreased
- More partitions = more parallelism but also more file handles and rebalance overhead
- Rule of thumb: 3–10 partitions per topic for small workloads, 30–100+ for high throughput

### Granularity

Decide whether to use one topic per event type or group related events into fewer topics.

```
┌────────────────────────────────────────────────────────────┐
│         Fine-Grained vs Coarse-Grained Topics               │
│                                                            │
│  Fine-grained (1 topic per event):                         │
│                                                            │
│    orders.order.created     ─── Topic                      │
│    orders.order.updated     ─── Topic                      │
│    orders.order.cancelled   ─── Topic                      │
│    orders.order.shipped     ─── Topic                      │
│                                                            │
│    ✓ Consumers subscribe only to what they need            │
│    ✓ Independent retention and scaling                     │
│    ✗ Many topics to manage                                 │
│    ✗ Topic proliferation in large systems                  │
│                                                            │
│  Coarse-grained (1 topic per entity):                      │
│                                                            │
│    orders.order.events      ─── Topic (all order events)   │
│      message.type = "created" | "updated" | "cancelled"    │
│                                                            │
│    ✓ Fewer topics to manage                                │
│    ✓ Full entity history in one stream                     │
│    ✗ Consumers must filter irrelevant events               │
│    ✗ Noisy neighbor effect between event types             │
└────────────────────────────────────────────────────────────┘
```

**Recommendation:** Start coarse-grained (one topic per entity) and split into fine-grained topics only when consumers have clearly divergent throughput, retention, or filtering needs.

---

## Producer Best Practices

Producers are the entry point for all data in your messaging system. A poorly configured producer can cause data loss, duplicate messages, or broker overload.

### Batching

Batching amortizes the per-message overhead of network round-trips and broker writes.

```
┌────────────────────────────────────────────────────────────┐
│                  Batching Mechanics                          │
│                                                            │
│  Without Batching:                                          │
│                                                            │
│  msg1 ──▶ network ──▶ broker ──▶ ack                       │
│  msg2 ──▶ network ──▶ broker ──▶ ack                       │
│  msg3 ──▶ network ──▶ broker ──▶ ack                       │
│  Total: 3 round-trips                                      │
│                                                            │
│  With Batching:                                             │
│                                                            │
│  msg1 ─┐                                                   │
│  msg2 ─┼──▶ batch ──▶ network ──▶ broker ──▶ ack           │
│  msg3 ─┘                                                   │
│  Total: 1 round-trip                                       │
└────────────────────────────────────────────────────────────┘
```

| Setting | Low Latency | High Throughput |
|---------|-------------|-----------------|
| `batch.size` | 16 KB | 256 KB–1 MB |
| `linger.ms` | 0–5 ms | 10–100 ms |
| `buffer.memory` | 32 MB | 64–128 MB |

**Key trade-off:** Larger batches improve throughput but increase end-to-end latency. Tune based on your latency SLA.

### Compression

Compression reduces network bandwidth and broker storage at the cost of CPU on both producer and consumer.

| Algorithm | Compression Ratio | CPU Cost | Best For |
|-----------|-------------------|----------|----------|
| `none` | 1:1 | None | Low-latency, small messages |
| `lz4` | ~2:1 | Low | Default choice — good balance |
| `snappy` | ~1.7:1 | Low | Legacy compatibility |
| `zstd` | ~3:1 | Medium | High compression needs, large messages |
| `gzip` | ~2.5:1 | High | Maximum compression, CPU is not a concern |

**Recommendation:** Use `lz4` as the default. Switch to `zstd` for large payloads or when storage costs dominate. Avoid `gzip` unless compression ratio is critical and CPU is plentiful.

### Retries and Idempotent Producers

Network failures, broker leader elections, and transient errors are normal in distributed systems. Producers must retry gracefully without creating duplicates.

```
┌────────────────────────────────────────────────────────────┐
│            Idempotent Producer Flow                          │
│                                                            │
│  Producer assigns sequence number to each message:         │
│                                                            │
│  msg(seq=1) ──▶ Broker ──▶ Stored ──▶ ACK                  │
│  msg(seq=2) ──▶ Broker ──▶ Stored ──▶ ACK lost!            │
│  msg(seq=2) ──▶ Broker ──▶ Deduplicated (seq already seen) │
│                           ──▶ ACK (no duplicate written)   │
│  msg(seq=3) ──▶ Broker ──▶ Stored ──▶ ACK                  │
│                                                            │
│  Kafka: enable.idempotence = true                           │
│  Assigns ProducerID + sequence number per partition.       │
│  Broker rejects duplicates based on (PID, partition, seq). │
└────────────────────────────────────────────────────────────┘
```

**Retry configuration:**

| Setting | Recommended Value | Rationale |
|---------|-------------------|-----------|
| `retries` | `Integer.MAX_VALUE` (or very high) | Let `delivery.timeout.ms` control total time |
| `delivery.timeout.ms` | 120000 (2 min) | Upper bound on total publish time |
| `retry.backoff.ms` | 100–500 ms | Avoid hammering broker during recovery |
| `enable.idempotence` | `true` | Prevent duplicates from retries |
| `max.in.flight.requests.per.connection` | 5 (with idempotence) | Maintains ordering with idempotent producer |

### Error Handling

Not all producer errors are retriable. Distinguish between transient and permanent failures.

| Error Type | Examples | Action |
|------------|----------|--------|
| Retriable | `NotLeaderForPartition`, timeout, network error | Retry automatically |
| Non-retriable | `MessageSizeTooLarge`, `InvalidTopic`, serialization error | Log error, send to DLQ or alert |
| Fatal | Authentication failure, authorization denied | Halt producer, alert immediately |

**Best practices:**

- Always register a callback or error handler — never fire-and-forget in production
- Log failed messages with enough context to replay them (topic, key, headers)
- Use a local fallback (file, database) if the broker is unreachable for extended periods
- Monitor the producer error rate as a key health metric

---

## Consumer Best Practices

Consumers are where the real work happens. Most messaging bugs surface on the consumer side — duplicates, missed messages, and processing failures.

### Consumer Groups

Consumer groups enable parallel processing while guaranteeing that each message is delivered to exactly one consumer within the group.

```
┌────────────────────────────────────────────────────────────┐
│               Consumer Group Scaling                        │
│                                                            │
│  Topic: orders.order.created (6 partitions)                │
│                                                            │
│  Consumer Group: order-processor                           │
│                                                            │
│  2 consumers:                                              │
│    Consumer A ──▶ P0, P1, P2  (3 partitions)               │
│    Consumer B ──▶ P3, P4, P5  (3 partitions)               │
│                                                            │
│  3 consumers (balanced):                                   │
│    Consumer A ──▶ P0, P1      (2 partitions)               │
│    Consumer B ──▶ P2, P3      (2 partitions)               │
│    Consumer C ──▶ P4, P5      (2 partitions)               │
│                                                            │
│  7 consumers (one idle):                                   │
│    Consumer A–F ──▶ P0–P5     (1 partition each)           │
│    Consumer G   ──▶ (idle)    No partition assigned         │
│                                                            │
│  Rule: consumers in group <= partition count                │
│        (extras sit idle)                                   │
└────────────────────────────────────────────────────────────┘
```

**Design guidelines:**

- Name consumer groups after the service and purpose (e.g., `payment-service.order-processor`)
- Size the group to match processing capacity, not partition count
- Monitor rebalance frequency — frequent rebalances indicate instability
- Use static group membership (`group.instance.id`) to reduce rebalance churn in Kubernetes

### Offset Management

Offset management determines whether you lose messages or process duplicates after a restart or rebalance.

| Strategy | Behavior | Risk |
|----------|----------|------|
| Auto-commit (default) | Offsets committed on a timer | Messages lost if crash between commit and processing |
| Manual commit after processing | Commit only after successful processing | Duplicates on crash (at-least-once) |
| Manual commit before processing | Commit immediately on receive | Messages lost on crash (at-most-once) |
| Transactional commit | Commit offset and side effects atomically | Complex, requires transactional support |

**Recommendation:** Use manual commit after processing for most workloads. Accept at-least-once delivery and make consumers idempotent.

```
Safe Offset Commit Pattern:

  while (running) {
      records = consumer.poll(timeout)
      for each record in records:
          process(record)            // Step 1: Do the work
      consumer.commitSync()          // Step 2: Commit AFTER processing
  }

  ✓ If crash before commit → messages redelivered (no loss)
  ✗ If crash after process but before commit → duplicates
  → Make process() idempotent to handle duplicates safely
```

### Graceful Shutdown

A consumer that shuts down abruptly causes a rebalance and potentially reprocesses messages. Graceful shutdown avoids this.

```
┌────────────────────────────────────────────────────────────┐
│             Graceful Shutdown Sequence                       │
│                                                            │
│  1. Receive SIGTERM                                        │
│  2. Set running = false (stop poll loop)                   │
│  3. Call consumer.wakeup() to interrupt current poll       │
│  4. Wait for in-flight processing to complete              │
│  5. Commit final offsets                                   │
│  6. Call consumer.close()                                  │
│     → Triggers LeaveGroup — no rebalance timeout wait      │
│  7. Exit cleanly                                           │
│                                                            │
│  Without graceful shutdown:                                 │
│    Consumer disappears → session.timeout.ms elapses        │
│    → Rebalance triggered → other consumers pause           │
│    → Processing gap of 10–45 seconds                       │
│                                                            │
│  With graceful shutdown:                                    │
│    Consumer sends LeaveGroup → immediate rebalance         │
│    → Partitions reassigned in seconds                      │
└────────────────────────────────────────────────────────────┘
```

### Processing Guarantees

Choose the guarantee level based on your business requirements.

| Guarantee | Mechanism | Trade-Off |
|-----------|-----------|-----------|
| At-most-once | Commit offset before processing | Fast, no duplicates, but may lose messages |
| At-least-once | Commit offset after processing | No data loss, but must handle duplicates |
| Exactly-once | Transactional processing + offset commit | Strongest guarantee, most complex and slowest |

**In practice:** At-least-once with idempotent consumers is the most common production pattern. Exactly-once is reserved for financial transactions or strict consistency requirements.

---

## Schema and Contracts

Schemas define the contract between producers and consumers. Without enforced schemas, any producer change can silently break downstream consumers.

### Schema Evolution

Evolve schemas safely by understanding compatibility modes.

```
┌────────────────────────────────────────────────────────────┐
│              Schema Compatibility Matrix                     │
│                                                            │
│  Backward Compatible (safe for consumers):                  │
│    ✓ Add optional field with default                       │
│    ✓ Remove a field (consumers ignore it)                  │
│    ✗ Add required field without default                    │
│    ✗ Change field type                                     │
│                                                            │
│  Forward Compatible (safe for producers):                   │
│    ✓ Remove a field (producers stop sending it)            │
│    ✓ Add a field (old consumers ignore it)                 │
│    ✗ Remove required field                                 │
│                                                            │
│  Full Compatible (safe for both):                           │
│    ✓ Add optional field with default                       │
│    ✓ Remove optional field                                 │
│    ✗ Any required field changes                            │
│                                                            │
│  Recommended: BACKWARD or FULL compatibility               │
└────────────────────────────────────────────────────────────┘
```

### Contract-First Design

Define the message schema before writing producer or consumer code. This inverts the typical workflow where the schema emerges from the code.

**Contract-first workflow:**

1. Define schema in Avro, Protobuf, or JSON Schema
2. Register schema in a schema registry
3. Generate code from the schema (producer and consumer)
4. Review schema changes in pull requests like any API change
5. Run compatibility checks in CI before merging

| Serialization Format | Schema Registry Support | Best For |
|----------------------|------------------------|----------|
| Avro | Confluent Schema Registry | Kafka-centric, compact binary |
| Protobuf | Confluent, Buf Registry | Cross-language, strong typing |
| JSON Schema | Confluent, custom | Human-readable, REST integration |

### Versioning Strategies

When breaking changes are unavoidable, manage the transition with versioning.

```
┌────────────────────────────────────────────────────────────┐
│           Topic Versioning Strategy                          │
│                                                            │
│  Phase 1: Publish to both topics                           │
│                                                            │
│  Producer ──▶ orders.order.created.v1  (old consumers)     │
│          └──▶ orders.order.created.v2  (new consumers)     │
│                                                            │
│  Phase 2: Migrate consumers to v2                          │
│                                                            │
│  Producer ──▶ orders.order.created.v1  (draining)          │
│          └──▶ orders.order.created.v2  (all consumers)     │
│                                                            │
│  Phase 3: Decommission v1                                  │
│                                                            │
│  Producer ──▶ orders.order.created.v2  (only topic)        │
│                                                            │
│  Timeline: Weeks to months depending on consumer count     │
└────────────────────────────────────────────────────────────┘
```

---

## Reliability Best Practices

Reliability means messages are not lost, not duplicated (or duplicates are handled), and the system recovers from failures without manual intervention.

### Acknowledgments

Configure acknowledgments to match your durability requirements.

| Setting (Kafka `acks`) | Behavior | Durability | Latency |
|------------------------|----------|------------|---------|
| `acks=0` | Fire and forget | Lowest — messages can be lost | Lowest |
| `acks=1` | Leader acknowledges | Medium — lost if leader fails before replication | Low |
| `acks=all` | All in-sync replicas acknowledge | Highest — survives broker failures | Higher |

**Recommendation:** Use `acks=all` with `min.insync.replicas=2` for production workloads. The latency cost is small compared to the durability gained.

### Dead Letter Queues

Dead letter queues (DLQs) capture messages that cannot be processed after exhausting retries. They prevent poison messages from blocking the entire queue.

```
┌────────────────────────────────────────────────────────────┐
│              Dead Letter Queue Flow                          │
│                                                            │
│  Main Topic                                                │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐            │
│  │ message  │────▶│ Consumer │────▶│ Process  │            │
│  └──────────┘     └──────────┘     └────┬─────┘            │
│                                         │                  │
│                                    Success?                │
│                                    │     │                 │
│                                   Yes    No                │
│                                    │     │                 │
│                                    ▼     ▼                 │
│                               ┌──────┐ ┌─────────┐        │
│                               │ Ack  │ │ Retry   │        │
│                               │      │ │ (1,2,3) │        │
│                               └──────┘ └────┬────┘        │
│                                              │             │
│                                         Still failing?     │
│                                              │             │
│                                              ▼             │
│                                    ┌──────────────┐        │
│                                    │  DLQ Topic   │        │
│                                    │  (for manual │        │
│                                    │   review)    │        │
│                                    └──────────────┘        │
└────────────────────────────────────────────────────────────┘
```

**DLQ best practices:**

- Include original headers, topic, partition, offset, and failure reason in the DLQ message
- Monitor DLQ depth — a growing DLQ indicates a systemic processing problem
- Build a replay mechanism to reprocess DLQ messages after fixing the root cause
- Set retention on the DLQ long enough for investigation (7–30 days)

### Idempotency

Idempotent consumers produce the same result regardless of how many times a message is delivered. This is essential for at-least-once delivery systems.

| Technique | Mechanism | Trade-Off |
|-----------|-----------|-----------|
| Deduplication table | Store processed `message_id` in DB; skip if seen | Requires DB lookup per message |
| Upsert operations | Use `INSERT ... ON CONFLICT UPDATE` | Only works for state-setting operations |
| Idempotency key in header | Producer assigns unique key; consumer deduplicates | Requires key generation discipline |
| Version/timestamp check | Only apply if event version > current state | Natural for entity updates |

### Exactly-Once Patterns

Exactly-once semantics eliminate both data loss and duplicates. They require coordination between the messaging system and external state stores.

```
┌────────────────────────────────────────────────────────────┐
│         Transactional Outbox Pattern                        │
│                                                            │
│  Service                                                   │
│  ┌──────────────────────────────────────────────┐          │
│  │  BEGIN TRANSACTION                           │          │
│  │    1. UPDATE orders SET status = 'paid'      │          │
│  │    2. INSERT INTO outbox (event_data)        │          │
│  │  COMMIT                                      │          │
│  └──────────────────────────────────────────────┘          │
│                        │                                   │
│                        ▼                                   │
│  ┌──────────────────────────────────────────────┐          │
│  │  Outbox Poller / CDC (Debezium)              │          │
│  │    Reads committed outbox rows               │          │
│  │    Publishes to messaging topic              │          │
│  │    Marks rows as published                   │          │
│  └──────────────────────────────────────────────┘          │
│                                                            │
│  Guarantees: DB state and message are atomic.              │
│  If DB commit fails, no message is sent.                   │
│  If publish fails, poller retries from outbox.             │
└────────────────────────────────────────────────────────────┘
```

---

## Performance Best Practices

Performance tuning depends on whether your workload is throughput-bound (batch analytics) or latency-bound (real-time event processing).

### Throughput Optimization

```
┌────────────────────────────────────────────────────────────┐
│            Throughput Optimization Levers                    │
│                                                            │
│  Producer Side:                                            │
│  ┌─────────────────────────────────────────────┐           │
│  │ ↑ batch.size        → Larger batches        │           │
│  │ ↑ linger.ms         → More time to fill     │           │
│  │ ↑ buffer.memory     → More in-flight data   │           │
│  │ ↑ compression       → Less network I/O      │           │
│  └─────────────────────────────────────────────┘           │
│                                                            │
│  Broker Side:                                              │
│  ┌─────────────────────────────────────────────┐           │
│  │ ↑ num.partitions    → More parallelism      │           │
│  │ ↑ num.io.threads    → More I/O handlers     │           │
│  │ ↑ log.segment.bytes → Less frequent rolls   │           │
│  │   Use SSD storage   → Faster disk I/O       │           │
│  └─────────────────────────────────────────────┘           │
│                                                            │
│  Consumer Side:                                            │
│  ┌─────────────────────────────────────────────┐           │
│  │ ↑ fetch.min.bytes   → Larger fetches        │           │
│  │ ↑ max.poll.records  → More records per poll │           │
│  │ ↑ consumer count    → More parallel readers │           │
│  │   Async processing  → Non-blocking I/O      │           │
│  └─────────────────────────────────────────────┘           │
└────────────────────────────────────────────────────────────┘
```

### Latency Reduction

| Technique | Latency Impact | Notes |
|-----------|---------------|-------|
| Set `linger.ms=0` | Eliminates batching delay | Sacrifices throughput |
| Reduce `fetch.min.bytes` | Consumer fetches sooner | More broker requests |
| Co-locate producers with brokers | Lower network RTT | Feasible in single-region |
| Use `acks=1` instead of `acks=all` | Skips replication wait | Reduces durability |
| Disable compression | No CPU overhead | Increases network bytes |
| Pin partitions to fast brokers | Avoids slow nodes | Manual management required |

**Key trade-off:** Most latency reductions come at the cost of throughput, durability, or operational simplicity. Benchmark before and after each change.

### Resource Sizing

| Component | Sizing Factor | Guideline |
|-----------|--------------|-----------|
| Broker CPU | Message throughput + compression | 8–16 cores per broker for moderate load |
| Broker memory | Page cache for active segments | 32–64 GB RAM; allocate 60%+ to OS page cache |
| Broker disk | Retention × throughput × replication | SSD recommended; size for peak + 30% headroom |
| Network | Total ingest + replication + fetch | 10 Gbps NIC minimum for high-throughput clusters |
| Consumer memory | In-flight messages × message size | Size based on `max.poll.records` × avg message size |

---

## Security Best Practices

Messaging systems carry sensitive business data and are attractive targets. Secure them at every layer.

### Encryption

```
┌────────────────────────────────────────────────────────────┐
│              Encryption Layers                               │
│                                                            │
│  In Transit:                                                │
│  Producer ══TLS══▶ Broker ══TLS══▶ Consumer                │
│                    Broker ══TLS══▶ Broker (replication)     │
│                                                            │
│  At Rest:                                                   │
│  Broker Disk ──▶ Filesystem encryption (dm-crypt, LUKS)    │
│              ──▶ Cloud-managed encryption (AWS EBS, etc.)   │
│                                                            │
│  End-to-End (payload):                                      │
│  Producer encrypts payload ──▶ Broker stores ciphertext    │
│  ──▶ Consumer decrypts payload                             │
│  Broker never sees plaintext data.                         │
└────────────────────────────────────────────────────────────┘
```

| Layer | Mechanism | Protects Against |
|-------|-----------|------------------|
| In-transit (TLS) | mTLS between clients and brokers | Network sniffing, man-in-the-middle |
| At-rest | Disk encryption | Physical disk theft, unauthorized disk access |
| End-to-end | Application-level encryption | Compromised broker, insider threats |

### Authentication

| Method | Broker Support | Best For |
|--------|---------------|----------|
| mTLS (mutual TLS) | Kafka, RabbitMQ | Service-to-service authentication |
| SASL/SCRAM | Kafka | Username/password with secure hashing |
| SASL/OAUTHBEARER | Kafka | Integration with OAuth2/OIDC providers |
| LDAP/AD | RabbitMQ | Enterprise directory integration |
| IAM roles | AWS MSK, SQS, Pub/Sub | Cloud-native identity |

**Recommendation:** Use mTLS for service-to-service authentication in self-managed clusters. Use IAM roles for cloud-managed services. Avoid SASL/PLAIN in production.

### Authorization

Implement least-privilege access. Each service should only access the topics it needs.

```
Authorization Matrix Example:

  Service           Topic                    Permissions
  ───────────────   ──────────────────────   ──────────────
  order-service     orders.order.created     Produce
  order-service     orders.order.events      Produce
  payment-service   orders.order.created     Consume
  payment-service   payments.payment.*       Produce
  analytics-svc     * (all topics)           Consume (read-only)
  admin-tool        * (all topics)           Produce, Consume, Admin
```

| Platform | Authorization Mechanism |
|----------|------------------------|
| Kafka | ACLs (AclAuthorizer), RBAC (Confluent) |
| RabbitMQ | Permissions per vhost (configure, write, read) |
| AWS MSK | IAM policies, Kafka ACLs |
| Azure Service Bus | RBAC (Azure AD), Shared Access Signatures |

### Network Segmentation

- Place brokers in a private subnet, not exposed to the internet
- Use VPC peering or private endpoints for cross-account access
- Restrict broker ports (9092, 9093, 9094) to known CIDR ranges
- Separate client traffic (9093 TLS) from inter-broker traffic (9094)
- Use network policies in Kubernetes to restrict pod-to-broker communication

---

## Operational Best Practices

Operating a messaging system is an ongoing responsibility. These practices keep the system healthy over time.

### Monitoring and Alerting

Instrument every layer of the messaging stack. See [07-MONITORING.md](07-MONITORING.md) for detailed monitoring guidance.

**Critical alerts for any messaging system:**

| Alert | Condition | Severity |
|-------|-----------|----------|
| Consumer lag growing | Lag increasing for > 10 min | Warning |
| Consumer lag critical | Lag > threshold for > 30 min | Critical |
| DLQ depth increasing | DLQ messages > 0 and growing | Warning |
| Broker offline | Broker unreachable for > 1 min | Critical |
| Under-replicated partitions | Count > 0 for > 5 min | Critical |
| Disk usage high | Broker disk > 80% | Warning |
| Produce error rate | Error rate > 1% | Warning |
| End-to-end latency | P99 > SLA threshold | Warning |

### Capacity Planning

Plan for growth before you hit limits.

```
Capacity Planning Cycle:

  ┌──────────────┐    ┌───────────────┐    ┌──────────────┐
  │   Measure    │───▶│   Forecast    │───▶│    Scale     │
  │              │    │               │    │              │
  │ Current peak │    │ Growth rate   │    │ Add brokers  │
  │ Utilization  │    │ Headroom calc │    │ Add storage  │
  │ Growth trend │    │ Time to limit │    │ Add partns   │
  └──────────────┘    └───────────────┘    └──────────────┘
        ▲                                        │
        └────────────────────────────────────────┘
                    Repeat quarterly
```

### Retention Policies

| Data Type | Retention | Rationale |
|-----------|-----------|-----------|
| Transactional events | 7–14 days | Sufficient for replay and debugging |
| Audit events | 90 days–7 years | Compliance (GDPR, SOX, HIPAA) |
| Compacted topics | Infinite (with compaction) | Latest state per key always available |
| DLQ messages | 14–30 days | Time for investigation and replay |
| Internal/system topics | Match broker defaults | Offset commits, transaction state |

**Key rules:**

- Set retention explicitly on every topic — do not rely on broker defaults
- Use time-based retention (`retention.ms`) for event streams
- Use size-based retention (`retention.bytes`) when disk is the constraint
- Enable log compaction for topics that represent current state (e.g., user profiles)

### Upgrade Strategies

Rolling upgrades minimize downtime for broker clusters.

```
┌────────────────────────────────────────────────────────────┐
│            Rolling Upgrade Procedure                        │
│                                                            │
│  Cluster: Broker-1, Broker-2, Broker-3                     │
│                                                            │
│  Step 1: Verify cluster is healthy                         │
│    - No under-replicated partitions                        │
│    - All brokers online                                    │
│    - Consumer lag stable                                   │
│                                                            │
│  Step 2: Upgrade Broker-1                                  │
│    - Drain leadership from Broker-1                        │
│    - Stop Broker-1                                         │
│    - Upgrade binary / config                               │
│    - Start Broker-1                                        │
│    - Wait for ISR sync                                     │
│                                                            │
│  Step 3: Verify health, then repeat for Broker-2, 3       │
│                                                            │
│  Step 4: Update inter.broker.protocol.version              │
│  Step 5: Update log.message.format.version                 │
│                                                            │
│  Total downtime: Zero (if replication factor >= 2)         │
└────────────────────────────────────────────────────────────┘
```

---

## Testing Best Practices

Test messaging systems at every level to catch issues before they reach production.

### Integration Testing

Integration tests verify that producers, brokers, and consumers work together correctly.

```
┌────────────────────────────────────────────────────────────┐
│          Integration Test Architecture                      │
│                                                            │
│  Test Environment:                                         │
│  ┌──────────────────────────────────────────────┐          │
│  │  Testcontainers / Embedded Broker             │          │
│  │  ┌────────────────────────────────┐           │          │
│  │  │  Kafka / RabbitMQ container    │           │          │
│  │  └────────────────────────────────┘           │          │
│  │  ┌────────────────────────────────┐           │          │
│  │  │  Schema Registry container     │           │          │
│  │  └────────────────────────────────┘           │          │
│  └──────────────────────────────────────────────┘          │
│                                                            │
│  Test Flow:                                                │
│    1. Start containers                                     │
│    2. Create topics                                        │
│    3. Produce test messages                                │
│    4. Consume and assert on received messages              │
│    5. Verify offset commits                                │
│    6. Tear down containers                                 │
└────────────────────────────────────────────────────────────┘
```

**Integration test guidelines:**

- Use Testcontainers or embedded brokers — never test against shared environments
- Test serialization and deserialization with the actual schema registry
- Verify consumer group behavior (rebalance, offset commit) end to end
- Include tests for failure scenarios (broker down, network timeout)

### Contract Testing

Contract tests ensure producers and consumers agree on the message schema without requiring both to run simultaneously.

| Role | What to Test | Tool |
|------|-------------|------|
| Producer | Messages conform to published schema | Pact, schema registry compatibility check |
| Consumer | Can deserialize all compatible schema versions | Pact, integration tests with versioned schemas |
| Schema | Backward/forward compatibility before merge | Schema registry CI plugin, `confluent schema-registry test-compatibility` |

### Chaos Testing

Chaos tests validate that the system behaves correctly under failure conditions.

| Scenario | Method | Expected Behavior |
|----------|--------|-------------------|
| Broker failure | Kill one broker in a multi-broker cluster | Producers retry; consumers rebalance; no data loss |
| Network partition | Block traffic between broker and consumers | Consumer reconnects; lag grows temporarily then recovers |
| Disk full | Fill broker disk to 100% | Producer receives errors; broker rejects writes; alert fires |
| Consumer crash | Kill consumer process mid-processing | Rebalance occurs; uncommitted messages redelivered |
| Schema change | Deploy incompatible schema | Schema registry rejects; produce fails; no corrupt messages |

### Load Testing

Load tests reveal performance limits and bottlenecks before production traffic does.

```
Load Testing Approach:

  1. Establish baseline
     - Current throughput, latency, resource usage

  2. Ramp to expected peak
     - Verify system handles peak with headroom

  3. Push to breaking point
     - Find the actual limit (throughput ceiling, latency spike)

  4. Sustained load test
     - Run at 80% peak for 24–72 hours
     - Watch for memory leaks, disk growth, GC issues

  5. Burst test
     - Send 10x normal load for 5 minutes
     - Verify recovery time after burst

  Key metrics to capture:
  ┌──────────────────────────────────────────┐
  │ Throughput (msgs/sec)                    │
  │ Latency (P50, P95, P99)                 │
  │ Consumer lag                             │
  │ Broker CPU, memory, disk I/O             │
  │ Error rate                               │
  │ Rebalance count                          │
  └──────────────────────────────────────────┘
```

---

## Multi-Region and Disaster Recovery

Multi-region messaging adds resilience but introduces complexity around replication, consistency, and failover.

### Replication Strategies

```
┌────────────────────────────────────────────────────────────┐
│          Cross-Region Replication Models                     │
│                                                            │
│  Model 1: Active-Passive                                   │
│                                                            │
│  Region A (Primary)        Region B (DR)                   │
│  ┌──────────┐              ┌──────────┐                    │
│  │ Cluster  │─── async ──▶│ Cluster  │                    │
│  │ (active) │  replication │ (standby)│                    │
│  └──────────┘              └──────────┘                    │
│  All produce/consume here   Read-only replica              │
│                                                            │
│  Model 2: Active-Active                                    │
│                                                            │
│  Region A                  Region B                        │
│  ┌──────────┐              ┌──────────┐                    │
│  │ Cluster  │◀── async ──▶│ Cluster  │                    │
│  │ (active) │  bidirectional│ (active)│                    │
│  └──────────┘              └──────────┘                    │
│  Local produce/consume     Local produce/consume           │
│                                                            │
│  Model 3: Aggregation                                      │
│                                                            │
│  Region A ─── replicate ──▶ ┌──────────────┐              │
│  Region B ─── replicate ──▶ │  Central     │              │
│  Region C ─── replicate ──▶ │  Aggregator  │              │
│                              └──────────────┘              │
│                              Analytics / reporting         │
└────────────────────────────────────────────────────────────┘
```

| Replication Tool | Platform | Type |
|-----------------|----------|------|
| MirrorMaker 2 | Kafka | Topic-level async replication |
| Confluent Cluster Linking | Confluent | Byte-level replication, lower lag |
| Federation Plugin | RabbitMQ | Exchange-level federation |
| Shovel Plugin | RabbitMQ | Queue-level message transfer |
| Cloud-native replication | SQS, Pub/Sub, Service Bus | Managed multi-region (varies by service) |

### Failover

| Failover Type | RTO | Mechanism |
|--------------|-----|-----------|
| Automated (DNS) | 1–5 min | Health check triggers DNS failover |
| Semi-automated | 5–15 min | Runbook-driven, human triggers |
| Manual | 15–60 min | Full manual switchover procedure |

**Failover checklist:**

1. Verify DR cluster is caught up (check replication lag)
2. Update DNS or load balancer to point to DR cluster
3. Restart producers with new bootstrap servers (or rely on DNS)
4. Verify consumers are connected and processing
5. Monitor for duplicate messages during the switchover window
6. After primary recovery, plan reverse failover or resync

### Active-Active Patterns

Active-active messaging requires careful handling of message deduplication and ordering across regions.

```
┌────────────────────────────────────────────────────────────┐
│         Active-Active Design Considerations                  │
│                                                            │
│  Challenge 1: Circular Replication                          │
│    Region A ──▶ Region B ──▶ Region A (loop!)               │
│    Solution: Message provenance header — skip messages      │
│    that originated in the local region.                    │
│                                                            │
│  Challenge 2: Ordering                                      │
│    Events for the same entity may arrive out of order      │
│    across regions.                                         │
│    Solution: Use event timestamps or vector clocks         │
│    for conflict resolution.                                │
│                                                            │
│  Challenge 3: Duplicate Messages                            │
│    Same message replicated to both regions.                │
│    Solution: Idempotent consumers with global              │
│    deduplication (message_id-based).                       │
│                                                            │
│  When to Use Active-Active:                                 │
│    ✓ Users in multiple regions need low latency            │
│    ✓ No single region can handle all traffic               │
│    ✓ Business can tolerate eventual consistency            │
│    ✗ Strong ordering across regions is required            │
│    ✗ Exactly-once across regions is required               │
└────────────────────────────────────────────────────────────┘
```

---

## Production Readiness Checklist

Use this checklist before deploying a messaging system to production.

**Topic and Queue Design:**

- [ ] Naming convention defined and enforced
- [ ] Partition keys chosen based on ordering and throughput requirements
- [ ] Partition count sized for expected consumer parallelism
- [ ] Retention policies configured explicitly per topic

**Producers:**

- [ ] Batching tuned for throughput vs. latency requirements
- [ ] Compression enabled (lz4 or zstd)
- [ ] Idempotent producer enabled (`enable.idempotence=true`)
- [ ] Retries configured with backoff and delivery timeout
- [ ] Error handler registered — no fire-and-forget
- [ ] Producer metrics exported (error rate, latency, batch size)

**Consumers:**

- [ ] Consumer group named after service and purpose
- [ ] Offset commit strategy is manual after processing
- [ ] Graceful shutdown implemented (SIGTERM handler, `consumer.close()`)
- [ ] Consumer idempotency implemented (dedup table, upsert, or version check)
- [ ] Consumer lag monitored and alerted
- [ ] Max poll interval and session timeout tuned for processing time

**Schemas:**

- [ ] Schema registered in schema registry
- [ ] Compatibility mode set (backward or full)
- [ ] Schema compatibility check runs in CI
- [ ] Schema changes reviewed in pull requests

**Reliability:**

- [ ] Acknowledgments set to `acks=all` with `min.insync.replicas=2`
- [ ] Dead letter queue configured with monitoring
- [ ] Replay mechanism available for DLQ messages
- [ ] Transactional outbox or CDC in place for exactly-once needs

**Security:**

- [ ] TLS enabled for all client-broker communication
- [ ] Authentication configured (mTLS, SASL, or IAM)
- [ ] Authorization configured with least-privilege ACLs
- [ ] Brokers in private subnet, no public exposure
- [ ] Secrets managed via vault or secrets manager (not config files)

**Operations:**

- [ ] Monitoring dashboards deployed (lag, throughput, errors, latency)
- [ ] Alerts configured for all critical conditions
- [ ] Runbooks written for common failure scenarios
- [ ] Capacity plan documented with scaling triggers
- [ ] Backup and recovery procedure tested
- [ ] Rolling upgrade procedure documented and tested

**Testing:**

- [ ] Integration tests run against containerized broker
- [ ] Contract tests validate schema compatibility
- [ ] Load test results establish performance baseline
- [ ] Chaos test results document failure behavior

---

## Next Steps

| File | Topic | Description |
|------|-------|-------------|
| [00-OVERVIEW.md](00-OVERVIEW.md) | Messaging Overview | Foundational concepts, patterns, and broker comparison |
| [01-APACHE-KAFKA.md](01-APACHE-KAFKA.md) | Apache Kafka | Brokers, topics, partitions, producers, consumers |
| [02-RABBITMQ.md](02-RABBITMQ.md) | RabbitMQ | AMQP model, exchanges, queues, reliability |
| [03-CLOUD-SERVICES.md](03-CLOUD-SERVICES.md) | Cloud Services | SQS, SNS, EventBridge, Kinesis, Service Bus |
| [04-PATTERNS.md](04-PATTERNS.md) | Messaging Patterns | Pub/sub, request-reply, event sourcing, CQRS |
| [05-SCHEMA-MANAGEMENT.md](05-SCHEMA-MANAGEMENT.md) | Schema Management | Schema registries, evolution, compatibility |
| [06-RELIABILITY.md](06-RELIABILITY.md) | Reliability | Delivery guarantees, idempotency, retries, DLQs |
| [07-MONITORING.md](07-MONITORING.md) | Monitoring | Metrics, dashboards, alerting, distributed tracing |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial messaging best practices documentation |
