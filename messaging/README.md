# Messaging Learning Resources

A comprehensive guide to message queues and event streaming — Kafka, RabbitMQ, cloud services (AWS SQS/SNS, Azure Service Bus, Google Pub/Sub), messaging patterns, schema management, reliability, and monitoring.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | Messaging fundamentals, queues vs streams, event-driven architecture | **Start here** |
| [01-APACHE-KAFKA](01-APACHE-KAFKA.md) | Topics, partitions, consumer groups, Kafka Streams | When building high-throughput event streaming pipelines |
| [02-RABBITMQ](02-RABBITMQ.md) | Exchanges, queues, routing, reliability | When implementing traditional message queuing |
| [03-CLOUD-SERVICES](03-CLOUD-SERVICES.md) | AWS SQS/SNS, Azure Service Bus, Google Pub/Sub | When using managed messaging on cloud platforms |
| [04-PATTERNS](04-PATTERNS.md) | Pub/sub, fan-out, competing consumers, dead letter queues | When designing messaging architectures |
| [05-SCHEMA-MANAGEMENT](05-SCHEMA-MANAGEMENT.md) | Schema Registry, Avro, Protobuf, backward compatibility | When managing message contracts across services |
| [06-RELIABILITY](06-RELIABILITY.md) | At-least-once, exactly-once, idempotency, ordering | **Essential — every messaging system** |
| [07-MONITORING](07-MONITORING.md) | Consumer lag, throughput, alerting | When operating messaging systems in production |
| [08-BEST-PRACTICES](08-BEST-PRACTICES.md) | Partitioning strategies, retention, capacity planning | **Essential — production checklist** |
| [09-ANTI-PATTERNS](09-ANTI-PATTERNS.md) | Common messaging mistakes and how to avoid them | **Essential — what NOT to do** |
| [LEARNING-PATH](LEARNING-PATH.md) | Structured learning guide with exercises | **Start here** after the Overview |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Understand the difference between message queues and event streams
   - Learn core messaging concepts: producers, consumers, brokers
   - Explore event-driven architecture and when to use it

2. **Learn Apache Kafka** ([01-APACHE-KAFKA](01-APACHE-KAFKA.md))
   - Understand topics, partitions, and consumer groups
   - Learn how Kafka provides durable, high-throughput streaming
   - Explore Kafka Streams for stream processing

3. **Understand Reliability** ([06-RELIABILITY](06-RELIABILITY.md))
   - Learn delivery guarantees: at-least-once, at-most-once, exactly-once
   - Implement idempotent consumers to handle duplicates
   - Understand message ordering and how to preserve it

4. **Follow the Learning Path** ([LEARNING-PATH](LEARNING-PATH.md))
   - Structured curriculum with hands-on exercises
   - Progressive skill building from basics to production

### For Experienced Engineers

1. **Review Best Practices** ([08-BEST-PRACTICES](08-BEST-PRACTICES.md))
   - Production-ready partitioning and retention strategies
   - Capacity planning and scaling considerations
   - Operational excellence for messaging infrastructure

2. **Avoid Anti-Patterns** ([09-ANTI-PATTERNS](09-ANTI-PATTERNS.md))
   - Common messaging mistakes in design, implementation, and operations
   - Pitfalls with consumer groups, offset management, and backpressure

3. **Master Messaging Patterns** ([04-PATTERNS](04-PATTERNS.md))
   - Pub/sub, fan-out, competing consumers, and dead letter queues
   - Saga pattern and event sourcing with messaging
   - Choosing the right pattern for your use case

4. **Implement Schema Management** ([05-SCHEMA-MANAGEMENT](05-SCHEMA-MANAGEMENT.md))
   - Schema Registry for centralized contract management
   - Avro and Protobuf for efficient serialization
   - Backward and forward compatibility strategies

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Messaging & Event Streaming                    │
│                                                                  │
│   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│   │  Message Queues   │  │  Event Streams   │  │ Cloud Services│  │
│   │  (Point-to-point) │  │  (Log-based)     │  │ (Managed)    │  │
│   │                   │  │                  │  │              │  │
│   │  - RabbitMQ       │  │  - Apache Kafka  │  │  - AWS SQS   │  │
│   │  - Exchanges      │  │  - Topics        │  │  - Azure SB  │  │
│   │  - Routing keys   │  │  - Partitions    │  │  - Google P/S│  │
│   └──────────────────┘  └──────────────────┘  └──────────────┘  │
│                                                                  │
│   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│   │  Patterns         │  │  Schemas         │  │ Reliability  │  │
│   │  (Design your     │  │  (Manage your    │  │ (Guarantee   │  │
│   │   messaging)      │  │   contracts)     │  │  delivery)   │  │
│   │                   │  │                  │  │              │  │
│   │  - Pub/sub        │  │  - Schema Reg.   │  │  - At-least  │  │
│   │  - Fan-out        │  │  - Avro / Proto  │  │  - Exactly   │  │
│   │  - Dead letters   │  │  - Compatibility │  │  - Idempotent│  │
│   └──────────────────┘  └──────────────────┘  └──────────────┘  │
│                                                                  │
│   ┌──────────────────┐  ┌──────────────────┐                    │
│   │  Monitoring       │  │  Operations      │                    │
│   │  (Observe your    │  │  (Run messaging  │                    │
│   │   systems)        │  │   at scale)      │                    │
│   │                   │  │                  │                    │
│   │  - Consumer lag   │  │  - Partitioning  │                    │
│   │  - Throughput     │  │  - Retention      │                    │
│   │  - Alerting       │  │  - Capacity plan │                    │
│   └──────────────────┘  └──────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
```

## 🔑 Key Concepts

```
Messaging Fundamentals
──────────────────────
Producer            → Application that sends messages to a broker
Consumer            → Application that receives and processes messages
Broker              → Intermediary that routes and stores messages
Queue               → Point-to-point channel — each message consumed once
Topic               → Publish-subscribe channel — messages delivered to all subscribers

Apache Kafka
────────────
Partition       → Ordered, immutable sequence of messages within a topic
Consumer Group  → Set of consumers that share the work of reading a topic
Offset          → Position of a consumer within a partition
Kafka Streams   → Library for building stream processing applications
Retention       → How long messages are kept on the broker

Messaging Patterns
──────────────────
Pub/Sub             → Publish once, deliver to many subscribers
Fan-out             → Broadcast a message to multiple queues or consumers
Competing Consumers → Multiple consumers share a single queue for parallel processing
Dead Letter Queue   → Queue for messages that fail processing repeatedly
Saga                → Coordinate distributed transactions via messaging

Reliability & Delivery
──────────────────────
At-most-once    → Fire and forget — messages may be lost
At-least-once   → Retry on failure — messages may be duplicated
Exactly-once    → Hardest guarantee — requires idempotency or transactions
Idempotency     → Processing the same message multiple times produces the same result
Ordering        → Guaranteeing message sequence within a partition or queue
```

## 📋 Topics Covered

- **Foundations** — Message queues vs event streams, event-driven architecture, brokers, producers, consumers
- **Apache Kafka** — Topics, partitions, consumer groups, offsets, Kafka Streams, Kafka Connect
- **RabbitMQ** — Exchanges, queues, bindings, routing keys, reliability, clustering
- **Cloud Services** — AWS SQS/SNS, Azure Service Bus, Google Pub/Sub, managed vs self-hosted
- **Patterns** — Pub/sub, fan-out, competing consumers, dead letter queues, saga, event sourcing
- **Schema Management** — Schema Registry, Avro, Protobuf, backward/forward compatibility
- **Reliability** — Delivery guarantees, idempotency, ordering, transactions, deduplication
- **Monitoring** — Consumer lag, throughput metrics, alerting, distributed tracing
- **Best Practices** — Partitioning strategies, retention policies, capacity planning, scaling
- **Anti-Patterns** — Common messaging mistakes in design, implementation, and operations

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to messaging?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md) then follow [LEARNING-PATH.md](LEARNING-PATH.md)

**Already familiar with messaging?** → Jump to [04-PATTERNS.md](04-PATTERNS.md) or [05-SCHEMA-MANAGEMENT.md](05-SCHEMA-MANAGEMENT.md)

**Going to production?** → Review [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md) and [09-ANTI-PATTERNS.md](09-ANTI-PATTERNS.md)

**Want a structured path?** → Follow the [LEARNING-PATH.md](LEARNING-PATH.md) — progressive exercises from basics to production
