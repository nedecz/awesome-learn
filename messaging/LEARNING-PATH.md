# Messaging Learning Path

A structured, self-paced training guide to mastering messaging systems — from foundational concepts and pub/sub patterns to production-grade Kafka, RabbitMQ, cloud services, and reliability engineering. Each phase builds on the previous one, progressing from core concepts to advanced production patterns.

> **Time Estimate:** 8–10 weeks at ~5 hours/week. Adjust pace to your experience level. Engineers with prior Kafka or RabbitMQ experience may complete Phases 1–2 in half the time.

---

## How to Use This Guide

1. **Follow the phases in order** — each one builds directly on prior knowledge; jumping ahead creates gaps that slow you down later
2. **Read the linked documents** — they contain the detailed content, code examples, and reference tables
3. **Complete the exercises** — hands-on practice is how messaging concepts become intuition; do not skip them
4. **Check yourself** — use the knowledge check items before moving to the next phase; if you cannot answer them confidently, re-read the relevant sections
5. **Build the capstone project** — the final project ties together all six phases and produces a portfolio artifact you can reference in engineering conversations

---

## Phase 1: Foundations (Week 1–2)

### Learning Objectives

- Understand the difference between synchronous and asynchronous communication
- Learn the core messaging primitives: queues, topics, streams, and brokers
- Grasp the fundamental messaging patterns: point-to-point, publish/subscribe, and fan-out
- Survey the messaging landscape: Kafka, RabbitMQ, cloud-native services, and when to use each
- Understand event-driven architecture and how it differs from request/response

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | Messaging fundamentals, queues vs streams, event-driven architecture, broker landscape |
| 2 | [04-PATTERNS](04-PATTERNS.md) | Pub/sub basics, point-to-point, fan-out — read introductory sections only |

### Exercises

**1. Messaging Architecture Analysis:**

Choose a system you are familiar with — a personal project, a work service, or an open-source application — and perform a messaging analysis:

- List all inter-service communication paths (HTTP calls, database polling, shared state)
- For each path, classify the communication as:
  - Synchronous (request/response, blocking)
  - Asynchronous (fire-and-forget, event-driven)
- Identify the top 3 integration points that would benefit from asynchronous messaging
- For each candidate, state: What pattern would you use? (point-to-point, pub/sub, fan-out)

```
Synchronous vs Asynchronous Communication:

  Request/Response (Synchronous)        Event-Driven (Asynchronous)
  ┌─────────┐  request  ┌─────────┐    ┌─────────┐  event  ┌─────────┐
  │Service A├──────────►│Service B│    │Service A├────────►│ Broker  │
  │         │◄──────────┤         │    │         │         │         │
  └─────────┘  response └─────────┘    └─────────┘         └────┬────┘
                                                                │
  - Tight coupling                      ┌───────────────────────┼──────┐
  - Both must be available              │                       │      │
  - Latency = A + B                ┌────▼────┐  ┌─────────┐ ┌──▼──────┐
                                   │Service B│  │Service C│ │Service D│
                                   └─────────┘  └─────────┘ └─────────┘
                                   - Loose coupling
                                   - Producer doesn't wait
                                   - Independent scaling
```

Document your findings. You will revisit this analysis after Phase 6.

**2. Local RabbitMQ Setup:**

Run RabbitMQ locally using Docker Compose and explore the management UI:

```yaml
# docker-compose.yml — RabbitMQ starter stack
services:
  rabbitmq:
    image: rabbitmq:3.13-management
    ports:
      - "5672:5672"    # AMQP protocol
      - "15672:15672"  # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
```

Once running:
- Open the Management UI at `http://localhost:15672` (guest/guest)
- Navigate to **Queues** — create a new queue called `test-queue`
- Publish a message to the queue using the Management UI
- Consume the message and verify it was received
- Explore the **Exchanges** tab — note the default exchanges (direct, fanout, topic, headers)

**3. Local Kafka Setup:**

Run Kafka locally using Docker Compose:

```yaml
# docker-compose.yml — Kafka starter stack
services:
  kafka:
    image: apache/kafka:3.8.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@localhost:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
    depends_on:
      - kafka
```

Once running:
- Open Kafka UI at `http://localhost:8080`
- Create a topic called `test-events` with 3 partitions
- Use the built-in CLI tools to produce and consume messages:

```bash
# Produce messages
docker exec -it <kafka-container> /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic test-events

# Consume messages
docker exec -it <kafka-container> /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic test-events \
  --from-beginning
```

### Knowledge Check

- [ ] What is the difference between a message queue and a message stream?
- [ ] Name three benefits of asynchronous messaging over synchronous HTTP calls
- [ ] What is publish/subscribe, and how does it differ from point-to-point messaging?
- [ ] When would you choose RabbitMQ over Kafka (and vice versa)?

---

## Phase 2: Apache Kafka Deep Dive (Week 3–4)

### Learning Objectives

- Master Kafka's core architecture: topics, partitions, offsets, and replication
- Understand consumer groups, rebalancing, and partition assignment strategies
- Learn the role of Schema Registry and schema evolution with Avro and Protobuf
- Implement producers and consumers with proper configuration for durability
- Understand Kafka Streams basics for stream processing

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [01-APACHE-KAFKA](01-APACHE-KAFKA.md) | Full document — topics, partitions, consumer groups, Kafka Streams, producer/consumer config |
| 2 | [05-SCHEMA-MANAGEMENT](05-SCHEMA-MANAGEMENT.md) | Schema Registry, Avro, Protobuf, backward/forward compatibility |

### Exercises

**1. Kafka Partitioning and Consumer Groups:**

Using the Kafka Docker Compose setup from Phase 1, explore partitions and consumer groups:

```
Kafka Topic with 3 Partitions and 2 Consumer Groups:

  Producer
    │
    ▼
┌──────────────────────────────────┐
│  Topic: order-events             │
│  ┌──────────┬──────────┬───────┐ │
│  │Partition 0│Partition 1│Partition 2│ │
│  │ offset 0 │ offset 0 │ offset 0 │ │
│  │ offset 1 │ offset 1 │ offset 1 │ │
│  │ offset 2 │          │ offset 2 │ │
│  └────┬─────┴────┬─────┴────┬───┘ │
└───────┼──────────┼──────────┼────┘
        │          │          │
   Consumer Group A (order-service)
   ┌────▼────┐┌───▼────┐┌───▼────┐
   │Consumer1││Consumer2││Consumer3│
   │  (P0)   ││  (P1)  ││  (P2)  │
   └─────────┘└────────┘└────────┘

   Consumer Group B (analytics-service)
   ┌────────────▼──────────────┐
   │   Consumer1 (P0, P1, P2)  │
   └───────────────────────────┘
```

Tasks:
- Create a topic `order-events` with 3 partitions and replication factor 1
- Start 3 consumers in the same consumer group — verify each gets a subset of partitions
- Start 1 consumer in a different consumer group — verify it receives all messages
- Produce 100 messages with keys (`order-1`, `order-2`, ..., `order-10` repeating) — observe that messages with the same key always go to the same partition
- Kill one consumer and observe rebalancing — which consumer picks up the orphaned partition?

```bash
# Create topic with 3 partitions
docker exec -it <kafka-container> /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic order-events \
  --partitions 3 --replication-factor 1

# Start consumer in group "order-service"
docker exec -it <kafka-container> /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order-events \
  --group order-service

# Describe consumer group to see partition assignments
docker exec -it <kafka-container> /opt/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group order-service --describe
```

**2. Schema Registry with Avro:**

Add Schema Registry to your Docker Compose stack and register schemas:

```yaml
# Add to docker-compose.yml
  schema-registry:
    image: confluentinc/cp-schema-registry:7.6.0
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:9092
    depends_on:
      - kafka
```

Tasks:
- Register an Avro schema for `order-events-value`:

```json
{
  "type": "record",
  "name": "OrderEvent",
  "namespace": "com.example.orders",
  "fields": [
    {"name": "order_id", "type": "string"},
    {"name": "customer_id", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "currency", "type": "string", "default": "USD"},
    {"name": "timestamp", "type": "long"}
  ]
}
```

- Evolve the schema by adding an optional field (`shipping_address` with a default)
- Verify that backward compatibility is maintained
- Attempt a breaking change (remove `order_id`) — confirm Schema Registry rejects it

```bash
# Register schema
curl -X POST http://localhost:8081/subjects/order-events-value/versions \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{"schema": "{...}"}'

# Check compatibility
curl -X POST http://localhost:8081/compatibility/subjects/order-events-value/versions/latest \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{"schema": "{...}"}'
```

### Knowledge Check

- [ ] What is a Kafka partition, and why does partition count affect consumer parallelism?
- [ ] What happens when there are more consumers in a group than partitions in a topic?
- [ ] What is the difference between backward and forward schema compatibility?
- [ ] Why should you always use a message key for ordering-sensitive events?

---

## Phase 3: RabbitMQ & Cloud Services (Week 5–6)

### Learning Objectives

- Master RabbitMQ's AMQP model: exchanges, queues, bindings, and routing keys
- Understand exchange types: direct, fanout, topic, and headers
- Learn reliability features: publisher confirms, consumer acknowledgements, dead letter exchanges
- Survey cloud messaging services: AWS SQS/SNS, Azure Service Bus, Google Pub/Sub
- Understand when to use managed cloud services vs self-hosted brokers

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [02-RABBITMQ](02-RABBITMQ.md) | Full document — exchanges, queues, routing, reliability, clustering |
| 2 | [03-CLOUD-SERVICES](03-CLOUD-SERVICES.md) | Full document — AWS SQS/SNS, Azure Service Bus, Google Pub/Sub, comparison |

### Exercises

**1. RabbitMQ Exchange Types:**

Using the RabbitMQ Docker Compose setup from Phase 1, implement all four exchange types:

```
RabbitMQ Exchange Types:

  Direct Exchange              Fanout Exchange
  ┌──────────────┐             ┌──────────────┐
  │   direct.ex   │             │  fanout.ex   │
  └──┬────┬────┬─┘             └─┬────┬────┬──┘
     │    │    │                  │    │    │
  key=   key=  key=           (all) (all) (all)
  info  warn  error              │    │    │
     │    │    │               ┌─▼┐ ┌─▼┐ ┌─▼┐
   ┌─▼┐ ┌▼─┐ ┌▼──┐            │Q1│ │Q2│ │Q3│
   │Q1│ │Q2│ │Q3 │            └──┘ └──┘ └──┘
   └──┘ └──┘ └───┘

  Topic Exchange               Headers Exchange
  ┌──────────────┐             ┌──────────────┐
  │   topic.ex   │             │  headers.ex  │
  └──┬────┬────┬─┘             └──┬───────┬───┘
     │    │    │                  │       │
  order. order. payment.     x-type=   x-type=
   *.     #     created      order     payment
     │    │    │                  │       │
   ┌─▼┐ ┌▼─┐ ┌▼──┐            ┌─▼┐    ┌─▼┐
   │Q1│ │Q2│ │Q3 │            │Q1│    │Q2│
   └──┘ └──┘ └───┘            └──┘    └──┘
```

Tasks:
- **Direct exchange:** Create an exchange `log.direct`, bind 3 queues with routing keys `info`, `warn`, `error`. Publish messages to each key and verify only the correct queue receives each message.
- **Fanout exchange:** Create an exchange `events.fanout`, bind 3 queues with no routing key. Publish a message and verify all 3 queues receive it.
- **Topic exchange:** Create an exchange `orders.topic`, bind queues with patterns:
  - `order.created.*` — matches `order.created.us`, `order.created.eu`
  - `order.#` — matches all order events
  - `payment.completed` — matches only payment completions
- **Dead letter exchange:** Configure a queue with `x-dead-letter-exchange` and `x-message-ttl`. Publish a message, let it expire, and verify it appears in the dead letter queue.

**2. Cloud Service Comparison Lab:**

Choose one cloud messaging service (or use a local emulator) and implement the same pub/sub scenario:

- A producer publishes order events
- Two independent subscribers process the events (email notifications, analytics)
- A dead letter queue captures failed messages

Compare your implementation with the RabbitMQ and Kafka approaches:

| Feature | Kafka | RabbitMQ | Your Cloud Service |
|---------|-------|----------|-------------------|
| Ordering guarantee | Per-partition | Per-queue | ? |
| Consumer model | Pull (poll) | Push (deliver) | ? |
| Message retention | Time/size based | Until consumed | ? |
| Replay capability | Yes (offset reset) | No (destructive read) | ? |
| Max message size | 1 MB default | 128 MB default | ? |
| Dead letter support | Manual (topic) | Built-in (DLX) | ? |

### Knowledge Check

- [ ] What are the four RabbitMQ exchange types, and when would you use each?
- [ ] What is a dead letter exchange, and name three conditions that route a message there
- [ ] How does AWS SQS differ from Kafka in terms of message retention and replay?
- [ ] What is the difference between a push-based and pull-based consumer model?

---

## Phase 4: Reliability & Patterns (Week 7–8)

### Learning Objectives

- Understand delivery guarantees: at-most-once, at-least-once, exactly-once semantics
- Implement idempotent consumers that safely handle duplicate messages
- Master advanced messaging patterns: competing consumers, saga, event sourcing
- Design dead letter queue processing and retry strategies
- Understand message ordering guarantees and how to preserve them

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [06-RELIABILITY](06-RELIABILITY.md) | Full document — delivery guarantees, idempotency, ordering, exactly-once semantics |
| 2 | [04-PATTERNS](04-PATTERNS.md) | Full document — competing consumers, saga, event sourcing, dead letter queues |

### Exercises

**1. Idempotent Consumer Implementation:**

Implement an idempotent consumer that safely processes duplicate messages. The consumer should:
- Accept order events from a Kafka topic or RabbitMQ queue
- Use an idempotency key (message ID or business key) to detect duplicates
- Store processed message IDs in a local database (SQLite or PostgreSQL)
- Skip duplicate messages without side effects

```
Idempotent Consumer Pattern:

  Message arrives
       │
       ▼
  ┌────────────┐    Yes    ┌────────────────┐
  │ Already     ├─────────►│ ACK + Skip     │
  │ processed?  │          │ (no side effect)│
  └────┬───────┘          └────────────────┘
       │ No
       ▼
  ┌────────────┐
  │ Process    │
  │ message    │
  └────┬───────┘
       │
       ▼
  ┌────────────┐
  │ Store ID   │◄── atomic: process + store in same transaction
  │ in DB      │
  └────┬───────┘
       │
       ▼
  ┌────────────┐
  │ ACK        │
  └────────────┘
```

Test your implementation:
- Send the same message 5 times with the same idempotency key
- Verify the business logic executes exactly once
- Verify all 5 messages are acknowledged successfully

**2. Dead Letter Queue Processing Pipeline:**

Build a retry pipeline with exponential backoff using dead letter queues:

```
Retry Pipeline with DLQ:

  ┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌───────────┐
  │  Main    │───►│  Consumer    │───►│  Process OK  │───►│  ACK      │
  │  Queue   │    │              │    └──────────────┘    └───────────┘
  └──────────┘    └──────┬───────┘
                         │ Failure
                         ▼
                  ┌──────────────┐     retry-count < 3?
                  │  Inspect     ├───────────────┐
                  │  headers     │    Yes         │
                  └──────┬───────┘               │
                         │ No                     │
                         ▼                        ▼
                  ┌──────────────┐    ┌───────────────────┐
                  │  Parking Lot │    │  Retry Queue      │
                  │  (manual     │    │  (TTL = 2^n * 1s) │
                  │   review)    │    └─────────┬─────────┘
                  └──────────────┘              │ TTL expires
                                                ▼
                                         Back to Main Queue
```

Tasks:
- Configure a main queue, a retry queue with TTL, and a parking lot queue
- Implement a consumer that:
  - On failure, increments a `retry-count` header and republishes to the retry queue
  - On 3rd failure, routes to the parking lot queue
  - Uses exponential backoff: 1s, 2s, 4s delay between retries
- Simulate failures by randomly rejecting 50% of messages
- Monitor the flow in the RabbitMQ Management UI

**3. Competing Consumers with Ordered Processing:**

Design a system where:
- 4 consumers process messages in parallel from a partitioned topic
- Messages with the same `customer_id` must be processed in order
- Messages with different `customer_id` values can be processed concurrently

Implement this using Kafka (partition key = `customer_id`) or RabbitMQ (consistent hash exchange). Verify ordering with timestamped log output.

### Knowledge Check

- [ ] What is the difference between at-least-once and exactly-once delivery?
- [ ] Why is idempotency required for at-least-once delivery systems?
- [ ] What is a dead letter queue, and when should messages be routed there?
- [ ] How do you maintain message ordering while still allowing parallel processing?

---

## Phase 5: Monitoring & Operations (Week 9–10)

### Learning Objectives

- Monitor messaging systems effectively: consumer lag, throughput, error rates
- Set up alerting for critical messaging health indicators
- Understand capacity planning for Kafka clusters and RabbitMQ nodes
- Learn partitioning strategies, retention policies, and performance tuning
- Implement operational runbooks for common messaging incidents

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [07-MONITORING](07-MONITORING.md) | Full document — consumer lag, throughput metrics, alerting, dashboards |
| 2 | [08-BEST-PRACTICES](08-BEST-PRACTICES.md) | Partitioning strategies, retention policies, capacity planning, performance tuning |

### Exercises

**1. Kafka Monitoring Dashboard:**

Set up Prometheus and Grafana to monitor your local Kafka cluster. Add JMX exporter to expose Kafka metrics:

```yaml
# Add to docker-compose.yml
  kafka-exporter:
    image: danielqsj/kafka-exporter:latest
    command:
      - --kafka.server=kafka:9092
    ports:
      - "9308:9308"
    depends_on:
      - kafka

  prometheus:
    image: prom/prometheus:v2.48.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:10.2.0
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
```

Create a Grafana dashboard with:
- **Row 1 — Cluster Health** (stat panels):
  - Active brokers count
  - Total topics / partitions
  - Under-replicated partitions (should be 0)

- **Row 2 — Consumer Lag** (time-series graph):
  - Consumer lag per consumer group
  - Consumer lag per partition
  - Alert threshold line at lag > 1000

- **Row 3 — Throughput** (time-series graph):
  - Messages in per second (by topic)
  - Bytes in/out per second
  - Request rate and latency

Key metrics to monitor:

```
Critical Kafka Metrics:

  Consumer Lag (most important)
  ┌─────────────────────────────────────┐
  │ Latest Offset:  1,000,542           │
  │ Consumer Offset:  1,000,100         │
  │ Lag: 442 messages                   │
  │                                     │
  │ ▲ lag                               │
  │ │     ╱╲                            │
  │ │    ╱  ╲    ╱╲                     │
  │ │   ╱    ╲  ╱  ╲                    │
  │ │──╱──────╲╱────╲──── threshold     │
  │ │╱                ╲                 │
  │ └───────────────────────────► time  │
  └─────────────────────────────────────┘
```

**2. Write Alerting Rules:**

Design 5 alerting rules for messaging system health:

1. **Consumer lag too high** — lag > 10,000 for 5 minutes
2. **Consumer group stopped** — no offset commits for 10 minutes
3. **Under-replicated partitions** — any partition with ISR < replication factor
4. **Queue depth growing** — RabbitMQ queue depth increasing for 15 minutes
5. **Message processing error rate** — DLQ message rate > 1% of total throughput

For each rule, write:
- Alert name and severity (critical / warning)
- The metric query (PromQL or equivalent)
- The `for` duration and threshold justification
- A 3-sentence runbook summary

**3. Capacity Planning Exercise:**

Given the following requirements, design the Kafka cluster:
- 50,000 messages/second peak throughput
- Average message size: 2 KB
- 7-day retention period
- 99.9% availability requirement

Calculate:
- Required number of partitions (target: 10,000 msg/s per partition)
- Required storage (messages × size × retention × replication factor)
- Required number of brokers (network throughput and storage constraints)
- Replication factor recommendation with justification

### Knowledge Check

- [ ] What is consumer lag, and why is it the single most important metric for Kafka consumers?
- [ ] Name three operational scenarios where under-replicated partitions occur
- [ ] How do you decide the right number of partitions for a Kafka topic?
- [ ] What retention policy should you use for event sourcing vs transient notifications?

---

## Phase 6: Production Readiness (Week 11–12)

### Learning Objectives

- Identify and remediate common messaging anti-patterns
- Apply best practices for production messaging deployments
- Design a complete messaging architecture for a real-world use case
- Understand cross-cutting concerns: security, multi-tenancy, disaster recovery
- Build confidence through a capstone project that integrates all phases

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [09-ANTI-PATTERNS](09-ANTI-PATTERNS.md) | Full document — common messaging mistakes, detection, and remediation |
| 2 | [08-BEST-PRACTICES](08-BEST-PRACTICES.md) | Review — partitioning strategies, retention, capacity planning (reinforce from Phase 5) |

### Exercises

**1. Anti-Pattern Identification Exercise:**

Review the following system description and identify at least 5 anti-patterns from [09-ANTI-PATTERNS.md](09-ANTI-PATTERNS.md):

> "We run an e-commerce platform with 12 microservices. All services communicate through a single Kafka topic called `events` with 1 partition. Messages are JSON strings with no schema validation — last week a producer deployed a breaking change that crashed 3 downstream consumers. We use at-most-once delivery because 'it's faster', but we've lost payment events twice this quarter. Our consumer processes messages one at a time and restarts from offset 0 on every deployment because we never commit offsets. We have no monitoring on consumer lag — we find out about backlogs when customers complain. Messages contain full database rows (average 50 KB) including user PII. We have 90-day retention on all topics, even for ephemeral notification events."

For each anti-pattern identified:
- State the anti-pattern name
- Quote the specific evidence from the description
- Write the recommended fix in 2–3 sentences

<details>
<summary>Anti-patterns identified (click to reveal suggested answers)</summary>

1. **Single topic for everything** — all 12 services share one topic called `events`; use separate topics per domain event
2. **Single partition bottleneck** — 1 partition means zero parallelism; partition by a business key with adequate partition count
3. **No schema management** — JSON with no validation caused breaking change; adopt Schema Registry with Avro/Protobuf and compatibility checks
4. **Wrong delivery guarantee** — at-most-once for payment events; payment events require at-least-once with idempotent consumers
5. **No offset management** — restarting from offset 0 on every deploy; commit offsets after successful processing
6. **No monitoring** — no consumer lag monitoring; set up metrics and alerting for consumer lag
7. **Oversized messages** — 50 KB messages with full DB rows; send slim events with references, fetch details on demand
8. **PII in messages** — user PII in message payloads; encrypt or tokenize sensitive fields, or remove them entirely
9. **Uniform retention policy** — 90-day retention for ephemeral events wastes storage; set retention per topic based on use case

</details>

**2. Messaging Best Practices Audit:**

Select a service or the capstone project and perform a full messaging audit:

```
Messaging Health Checklist:

Producers:
  [ ] Messages have a defined schema (Avro/Protobuf/JSON Schema)
  [ ] Schema compatibility checked before deployment
  [ ] Message keys set correctly for ordering requirements
  [ ] Producer acks configured appropriately (acks=all for critical data)
  [ ] Message size within reasonable limits (< 1 MB)

Consumers:
  [ ] Consumer group ID is meaningful and stable across deployments
  [ ] Offset commit strategy is explicit (auto-commit disabled for critical paths)
  [ ] Idempotent processing implemented for at-least-once delivery
  [ ] Error handling routes failures to DLQ, not silent discard
  [ ] Consumer lag monitored and alerted

Infrastructure:
  [ ] Topics have appropriate partition count for throughput needs
  [ ] Replication factor >= 3 for production data
  [ ] Retention policy set per topic based on use case
  [ ] Monitoring covers: consumer lag, throughput, error rate, disk usage
  [ ] Dead letter queues configured with alerting

Security:
  [ ] TLS encryption for broker connections
  [ ] Authentication enabled (SASL/SCRAM, mTLS, or cloud IAM)
  [ ] No PII in message payloads without encryption
  [ ] ACLs restrict topic access per service
```

For each unchecked item, write a one-paragraph remediation plan: what to change, estimated effort, and priority.

### Capstone Project: Event-Driven Order Processing System

Design and implement an event-driven order processing system using Kafka and/or RabbitMQ:

```
Order Processing System Architecture:

  ┌──────────────┐
  │  API Gateway  │
  │  (REST)       │
  └──────┬───────┘
         │ POST /orders
         ▼
  ┌──────────────┐    order.created     ┌──────────────────────────┐
  │ Order Service ├────────────────────►│  Kafka: order-events     │
  │              │                      │  (partitioned by         │
  └──────────────┘                      │   customer_id)           │
                                        └──┬────────┬─────────┬───┘
                                           │        │         │
                              ┌────────────▼┐ ┌─────▼──────┐ ┌▼────────────┐
                              │ Payment     │ │ Inventory  │ │ Notification │
                              │ Service     │ │ Service    │ │ Service      │
                              └──────┬──────┘ └──────┬─────┘ └─────────────┘
                                     │               │
                           payment.  │     inventory. │
                           completed │     reserved   │
                                     ▼               ▼
                              ┌──────────────────────────┐
                              │  Kafka: fulfillment-     │
                              │  events                  │
                              └────────────┬─────────────┘
                                           │
                              ┌────────────▼─────────────┐
                              │  Shipping Service        │
                              └──────────────────────────┘
```

### Requirements

| Requirement | What to Implement |
|-------------|-------------------|
| **Event schema** | Define Avro schemas for all events; register in Schema Registry with backward compatibility |
| **Producers** | Order Service publishes `order.created` with `customer_id` as partition key; acks=all |
| **Consumers** | Payment, Inventory, and Notification services consume independently with separate consumer groups |
| **Idempotency** | Payment Service is idempotent — duplicate `order.created` events do not charge twice |
| **Dead letter queue** | Failed messages route to a DLQ after 3 retries with exponential backoff |
| **Monitoring** | Consumer lag dashboard in Grafana; alerts for lag > 5,000 and DLQ message rate > 0 |
| **Schema evolution** | Add a `shipping_priority` field to `order.created` without breaking existing consumers |
| **Reliability** | At-least-once delivery for all services; exactly-once for Payment Service (idempotent consumer) |
| **Operations** | Docker Compose runs the full stack; README documents how to start, test, and monitor |
| **Anti-patterns** | Run the audit checklist; document and fix at least 3 issues found |

### Evaluation Criteria

| Area | What to Verify |
|------|----------------|
| **Architecture** | Can you describe the data flow from order creation to shipping in under 60 seconds? |
| **Schema** | Are all events registered in Schema Registry? Does adding a field pass compatibility? |
| **Reliability** | Does sending the same order event twice result in exactly one payment charge? |
| **Monitoring** | Can you see consumer lag for all groups in a dashboard? Do alerts fire on lag spikes? |
| **Error handling** | When a consumer fails, does the message retry and eventually land in the DLQ? |
| **Anti-patterns** | Are findings from the audit checklist documented with remediation plans? |
| **Documentation** | Is there a README explaining the architecture, how to run it, and how to test it? |

---

## Completion Criteria

When you have completed all six phases and the capstone project, you should be able to:

1. Explain the trade-offs between Kafka, RabbitMQ, and cloud messaging services for a given use case
2. Design a messaging architecture with appropriate topics, partitions, and consumer groups
3. Implement producers and consumers with correct delivery guarantees and idempotency
4. Register and evolve schemas without breaking downstream consumers
5. Configure dead letter queues, retry policies, and error handling pipelines
6. Monitor consumer lag, throughput, and error rates with dashboards and alerts
7. Identify and remediate common messaging anti-patterns in an existing system
8. Capacity-plan a Kafka cluster for a given throughput and retention requirement
9. Choose between self-hosted and cloud-managed messaging for a given set of constraints
10. Build and operate an event-driven system end-to-end with confidence

---

## Quick Reference: Document Map

| # | Document | Phase | Key Topics |
|---|----------|-------|------------|
| 00 | [OVERVIEW](00-OVERVIEW.md) | 1 | Messaging fundamentals, queues vs streams, event-driven architecture |
| 01 | [APACHE-KAFKA](01-APACHE-KAFKA.md) | 2 | Topics, partitions, consumer groups, Kafka Streams, producer/consumer config |
| 02 | [RABBITMQ](02-RABBITMQ.md) | 3 | Exchanges, queues, routing, reliability, clustering |
| 03 | [CLOUD-SERVICES](03-CLOUD-SERVICES.md) | 3 | AWS SQS/SNS, Azure Service Bus, Google Pub/Sub, comparison |
| 04 | [PATTERNS](04-PATTERNS.md) | 1, 4 | Pub/sub, fan-out, competing consumers, saga, dead letter queues |
| 05 | [SCHEMA-MANAGEMENT](05-SCHEMA-MANAGEMENT.md) | 2 | Schema Registry, Avro, Protobuf, backward/forward compatibility |
| 06 | [RELIABILITY](06-RELIABILITY.md) | 4 | At-least-once, exactly-once, idempotency, ordering guarantees |
| 07 | [MONITORING](07-MONITORING.md) | 5 | Consumer lag, throughput metrics, alerting, dashboards |
| 08 | [BEST-PRACTICES](08-BEST-PRACTICES.md) | 5, 6 | Partitioning strategies, retention, capacity planning, performance tuning |
| 09 | [ANTI-PATTERNS](09-ANTI-PATTERNS.md) | 6 | Common messaging mistakes, detection, remediation |
| — | [LEARNING-PATH](LEARNING-PATH.md) | All | This document — structured 6-phase curriculum |

---

## Recommended Resources

### Books

| Book | Author | Focus |
|------|--------|-------|
| *Designing Data-Intensive Applications* | Martin Kleppmann | Messaging, streaming, replication — the definitive systems design reference |
| *Kafka: The Definitive Guide* | Gwen Shapira, Todd Palino, Rajini Sivaram, Krit Petty | Comprehensive Kafka architecture, operations, and best practices |
| *RabbitMQ in Depth* | Gavin M. Roy | AMQP model, exchange types, clustering, reliability patterns |
| *Enterprise Integration Patterns* | Gregor Hohpe, Bobby Woolf | Canonical messaging patterns — the origin of modern messaging design |
| *Building Event-Driven Microservices* | Adam Bellemare | Event-driven architecture, event sourcing, CQRS with messaging |
| *Streaming Systems* | Tyler Akidau, Slava Chernyak, Reuven Lax | Windowing, watermarks, exactly-once — stream processing theory |

### Online Resources

- **kafka.apache.org/documentation** — Official Kafka documentation, configuration reference, operations guide
- **rabbitmq.com/docs** — RabbitMQ documentation, tutorials, clustering and reliability guides
- **confluent.io/blog** — Practical Kafka articles on schema management, exactly-once, and Kafka Streams
- **cloudamqp.com/blog** — RabbitMQ tutorials, best practices, and performance tuning
- **docs.aws.amazon.com/sqs** — AWS SQS/SNS documentation and best practices
- **learn.microsoft.com/azure/service-bus-messaging** — Azure Service Bus documentation and patterns
- **cloud.google.com/pubsub/docs** — Google Pub/Sub documentation, ordering, and exactly-once delivery

### Tools

| Tool | Purpose | Phase |
|------|---------|-------|
| Apache Kafka | Distributed event streaming platform | 1, 2, 4, 5 |
| RabbitMQ | Message broker with flexible routing | 1, 3, 4 |
| Confluent Schema Registry | Schema management and compatibility enforcement | 2 |
| Kafka UI | Web UI for Kafka cluster management | 1, 2, 5 |
| Prometheus | Metrics collection and alerting | 5 |
| Grafana | Dashboards and visualisation | 5, 6 |
| kafka-exporter | Prometheus exporter for Kafka metrics | 5 |
| kcat (kafkacat) | Command-line Kafka producer/consumer/metadata tool | 2, 4 |
| Docker Compose | Running local messaging stacks | 1–6 |
| k6 | Load testing to generate message traffic | 5, Capstone |
