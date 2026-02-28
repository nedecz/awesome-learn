# Messaging Anti-Patterns

A catalogue of the most common messaging mistakes — what they look like, why they are harmful, and exactly how to fix them. Use this document as a code-review checklist, a pre-production gate, or a team learning resource.

---

## Table of Contents

- [Introduction](#introduction)
- [Anti-Patterns Summary Table](#anti-patterns-summary-table)
- [1. Using Messaging as a Database](#1-using-messaging-as-a-database)
- [2. Unbounded Queue Growth](#2-unbounded-queue-growth)
- [3. Missing Dead Letter Queue](#3-missing-dead-letter-queue)
- [4. Ignoring Message Ordering](#4-ignoring-message-ordering)
- [5. Giant Messages](#5-giant-messages)
- [6. Missing Idempotency](#6-missing-idempotency)
- [7. Tight Coupling Through Messages](#7-tight-coupling-through-messages)
- [8. No Schema Evolution Strategy](#8-no-schema-evolution-strategy)
- [9. Ignoring Backpressure](#9-ignoring-backpressure)
- [10. Missing Monitoring and Alerting](#10-missing-monitoring-and-alerting)
- [11. Synchronous Mindset with Async Tools](#11-synchronous-mindset-with-async-tools)
- [12. No Message Versioning](#12-no-message-versioning)
- [Quick Reference Checklist](#quick-reference-checklist)
- [Next Steps](#next-steps)
- [Version History](#version-history)

---

## Introduction

### Why Anti-Patterns Matter

Messaging anti-patterns are recurring practices that seem reasonable at first but create significant reliability, scalability, and maintainability problems over time. They persist because they are often the easiest path — storing state in a queue avoids designing a proper data model, skipping dead letter queues means fewer components to manage, and sending giant payloads avoids the complexity of claim-check patterns.

The patterns documented here represent real messaging failures seen across production systems. Each one is:

- **Seductive** — it felt like the right approach when first implemented
- **Harmful** — it creates failures that are difficult to diagnose and recover from
- **Fixable** — there is a well-understood better approach

Messaging systems are the nervous system of distributed architectures. When they fail, the cascading effects are often far worse than a single service outage. A misconfigured queue can silently drop messages for days before anyone notices. A missing idempotency guarantee can corrupt data across dozens of downstream services. An overwhelmed broker can bring an entire platform to its knees.

Unlike API failures that return immediate errors, messaging failures are **delayed and silent**. A producer publishes a message and assumes it will be processed. If the consumer is down, misconfigured, or overwhelmed, the failure may not surface for hours, days, or weeks. This delayed feedback loop makes messaging anti-patterns especially dangerous — by the time you detect the problem, significant damage has already occurred.

Understanding these anti-patterns is not optional for teams building event-driven or message-based systems — it is a prerequisite for operating them successfully.

The cost of messaging failures is uniquely high because they are **invisible by default**. A broken REST endpoint returns a 500 error immediately. A broken consumer silently stops processing messages, and nobody knows until a customer complains days later. The anti-patterns in this document are ordered roughly by how often they appear in real systems and how much damage they cause.

### How to Use This Document

1. **Code review** — Reference specific anti-patterns when reviewing pull requests that involve messaging infrastructure
2. **Pre-production checklist** — Use the [Quick Reference Checklist](#quick-reference-checklist) before deploying messaging consumers or producers to production
3. **Incident retros** — After a messaging-related incident, check which anti-patterns contributed
4. **Team onboarding** — Assign this document to new engineers before they work on messaging systems
5. **Periodic audit** — Run through the checklist quarterly for existing messaging infrastructure
6. **Architecture review** — Use the summary table during design reviews to catch problems early

---

## Anti-Patterns Summary Table

| # | Anti-Pattern | Severity | Impact |
|---|-------------|----------|--------|
| 1 | Using Messaging as a Database | 🔴 Critical | Data loss, unreliable state, operational fragility |
| 2 | Unbounded Queue Growth | 🔴 Critical | Broker crash, disk exhaustion, cascading outages |
| 3 | Missing Dead Letter Queue | 🔴 Critical | Silent data loss, unrecoverable message failures |
| 4 | Ignoring Message Ordering | 🟠 High | Data corruption, inconsistent state across services |
| 5 | Giant Messages | 🟠 High | Broker instability, throughput degradation, OOM errors |
| 6 | Missing Idempotency | 🔴 Critical | Duplicate processing, data corruption, financial errors |
| 7 | Tight Coupling Through Messages | 🟠 High | Cascading deployment failures, reduced autonomy |
| 8 | No Schema Evolution Strategy | 🟠 High | Broken consumers, deployment coordination nightmares |
| 9 | Ignoring Backpressure | 🔴 Critical | Consumer crashes, broker overload, data loss |
| 10 | Missing Monitoring and Alerting | 🟠 High | Undetected failures, delayed incident response |
| 11 | Synchronous Mindset with Async Tools | 🟡 Medium | Blocked threads, false timeouts, wasted infrastructure |
| 12 | No Message Versioning | 🟠 High | Breaking changes, tightly coupled deployments |

---

## 1. Using Messaging as a Database

### The Anti-Pattern

Storing application state in message queues instead of purpose-built databases. Relying on messages as the system of record — peeking at queue contents to answer queries, keeping messages indefinitely as a data store, or re-reading consumed messages to reconstruct state.

### What It Looks Like

- A service queries the queue to check whether an order has been placed instead of querying a database
- Messages are consumed, processed, and then re-published to the same queue for later reference
- The team uses queue depth as a proxy for "how many items are pending" in business logic
- Consumers peek at messages without consuming them, treating the queue as a read-only store
- Services delay consuming messages because they "need the data to stay in the queue"
- Infrastructure teams are told the broker "can never be restarted" because it holds critical data
- Message retention is set to "infinite" because the application depends on reading old messages

### Why It's Harmful

- **Message brokers are not databases** — they lack indexing, query capability, ACID transactions, and efficient random access. Attempting to use them as data stores leads to brittle, unscalable architectures.
- **Retention is not durable storage** — queue retention policies (TTL, max size) can silently delete messages that the application depends on. A configuration change or broker restart can wipe the "database."
- **No query capability** — retrieving a specific message from a queue of millions requires consuming every message, which is orders of magnitude slower than a database index lookup.
- **Operational fragility** — broker maintenance (upgrades, migrations, rebalancing) becomes terrifying when the broker holds irreplaceable state. Teams delay critical updates because they are afraid of losing data.
- **Performance degradation** — brokers optimize for sequential reads and writes, not random access patterns. Using them as databases degrades throughput for all consumers.

### The Fix

- Use a proper database (PostgreSQL, DynamoDB, Cosmos DB) as the system of record for application state
- Treat messages as notifications or commands — events that trigger state changes in a database, not the state itself
- If you need event sourcing, use a purpose-built event store (EventStoreDB, Marten) or an append-only database table
- Implement the Outbox Pattern to reliably publish events after database writes
- Use Kafka's compacted topics only when you explicitly need a changelog and understand the retention semantics — this is the exception, not a general-purpose database replacement
- Design for the rule: messages are ephemeral signals; databases are durable state

> **Rule of thumb:** If you need to query the data, index it, update it in place, or keep it for more than one processing cycle, it belongs in a database — not a message queue.

---

## 2. Unbounded Queue Growth

### The Anti-Pattern

Deploying message queues without retention policies, size limits, or monitoring — allowing queues to grow without bound until they exhaust disk, memory, or broker resources.

### What It Looks Like

- Queues configured with no TTL, no max-length, and no max-size
- A consumer goes down for 48 hours and the queue accumulates millions of messages with no alarm
- The broker's disk usage grows steadily and no one notices until it hits 100%
- RabbitMQ enters flow control or Kafka brokers start rejecting produce requests because log segments have consumed all available storage
- Teams notice problems only when the broker stops accepting new messages

### Why It's Harmful

- **Broker crash** — when the broker runs out of disk or memory, it crashes. This affects every queue and every service using that broker, not just the one with the runaway queue.
- **Recovery is painful** — restarting a broker with millions of backlogged messages causes long recovery times, rebalancing storms, and potential data loss.
- **Cascading failures** — a single unbounded queue can take down an entire message bus, causing widespread outages across unrelated services.
- **Consumer overwhelm** — when a consumer comes back online, it faces a massive backlog. Processing millions of stale messages can take hours and often triggers secondary failures.
- **Cost explosion** — in cloud-managed brokers (Amazon SQS, Azure Service Bus), unbounded queues translate directly to escalating storage and throughput costs.

### The Fix

- Set explicit retention policies on every queue: TTL for message age, max-length for message count, max-size for bytes
- Configure dead letter queues for messages that expire or exceed retry limits — do not silently discard them
- Monitor queue depth and set alerts for abnormal growth (e.g., depth exceeds 10,000 messages or grows faster than 1,000/minute)
- Set disk usage alerts on the broker at 70%, 85%, and 95%
- Define a backlog recovery strategy: can consumers safely skip stale messages, or must they process every one?
- For Kafka, configure `log.retention.hours`, `log.retention.bytes`, and `log.segment.bytes` on every topic
- For RabbitMQ, set `x-max-length`, `x-max-length-bytes`, and `x-message-ttl` on every queue
- Test your retention configuration by intentionally stopping a consumer and verifying that messages are handled correctly (dead-lettered, expired, or alerting fires)

> **Rule of thumb:** Every queue should have an answer to the question: "What happens when 10 million messages pile up here?" If the answer is "we don't know," you have this anti-pattern.

---

## 3. Missing Dead Letter Queue

### The Anti-Pattern

Not configuring a dead letter queue (DLQ) for messages that fail processing. Failed messages are either retried forever (consuming resources), silently discarded (losing data), or block the queue (halting all consumers).

### What It Looks Like

- A consumer encounters a malformed message and retries it indefinitely, logging the same error thousands of times
- A "poison pill" message blocks a FIFO queue — every consumer crashes on the same message, and no other messages can be processed
- Failed messages are caught in a try/catch block and silently swallowed with a log statement
- The team discovers weeks later that thousands of order events were lost because a schema change made them unparseable
- Consumers have no retry limit configured — a transient failure causes infinite retry loops
- A single poison pill message causes consumer restarts in a loop, triggering Kubernetes pod restart limits and paging the on-call engineer for the wrong reason

### Why It's Harmful

- **Silent data loss** — without a DLQ, failed messages vanish. There is no record of what failed, no way to investigate, and no way to reprocess.
- **Poison pill blocking** — in ordered queues, a single bad message can block all processing. Every consumer stops until someone manually removes the message.
- **Infinite retry storms** — without a max retry limit, consumers waste CPU and I/O retrying the same unfixable message, reducing throughput for healthy messages.
- **Delayed discovery** — without DLQ monitoring, it can take days or weeks to discover that messages are being dropped. By then, the data is gone.
- **No forensics** — when an incident occurs, the team cannot determine which messages failed, why they failed, or how many were lost.

### The Fix

- Configure a DLQ on every queue and topic — this is non-negotiable for production systems
- Set a maximum retry count (typically 3–5 retries with exponential backoff) before routing to the DLQ
- Monitor DLQ depth and alert immediately when messages arrive (DLQ depth > 0 is always worth investigating)
- Build tooling to inspect, replay, and reprocess DLQ messages
- Log the failure reason alongside the dead-lettered message for post-incident analysis
- For RabbitMQ, use `x-dead-letter-exchange` and `x-dead-letter-routing-key`
- For Kafka, implement DLQ logic in the consumer (Kafka does not natively DLQ — frameworks like Spring Kafka provide this)
- For SQS, configure a redrive policy with `maxReceiveCount` and a designated DLQ
- Treat the DLQ as an operational inbox, not a dumping ground — review it daily, resolve issues, and keep it empty

> **Rule of thumb:** A DLQ with messages in it is a to-do list, not a trash can. If you are not regularly reviewing and reprocessing DLQ messages, you are losing data.

---

## 4. Ignoring Message Ordering

### The Anti-Pattern

Assuming messages will arrive at consumers in the exact order they were published, without designing the system to handle out-of-order delivery. Or conversely, requiring strict ordering everywhere when the business logic does not demand it.

### What It Looks Like

- A consumer processes "order shipped" before "order created" and crashes because the order does not exist in the database
- Services assume sequential processing but use a topic with multiple partitions and no partition key strategy
- A consumer applies state updates sequentially without checking event timestamps or version numbers
- The team adds more consumer instances for throughput and is surprised when events arrive out of order
- All messages are funneled through a single partition "for ordering" — destroying parallelism and throughput

### Why It's Harmful

- **Data corruption** — processing events out of order can produce incorrect state. An "update" arriving before a "create" causes errors or silent data loss.
- **Race conditions** — multiple consumers processing related events concurrently can overwrite each other's state changes.
- **Artificial bottlenecks** — forcing strict global ordering when only per-entity ordering is needed destroys parallelism. A single partition or queue becomes a throughput ceiling.
- **Fragile recovery** — when consumers restart and reprocess messages, out-of-order delivery becomes even more likely. Systems that assume ordering break during recovery.

### The Fix

- Design consumers to handle out-of-order messages — use event timestamps, sequence numbers, or version vectors to detect and resolve ordering conflicts
- Use partition keys (Kafka) or message groups (SQS FIFO) to guarantee ordering per entity, not globally
- Implement idempotent consumers so that replayed or reordered messages produce the correct result
- Use the "last-writer-wins" pattern with timestamps when strict ordering is not required
- Accept eventual consistency where possible — not every workflow needs strict ordering
- If global ordering is truly required, use a single partition (accepting the throughput limitation) or implement a resequencer pattern in the consumer

> **Rule of thumb:** Require ordering per entity (per customer, per order), not globally. Global ordering is the enemy of parallelism. If you need to process events about Order #12345 in order, partition by order ID — do not force every event in the system through a single queue.

---

## 5. Giant Messages

### The Anti-Pattern

Sending large payloads (megabytes or more) through the message broker instead of using the broker for lightweight event notifications and storing large data externally.

### What It Looks Like

- Messages contain base64-encoded files, full database snapshots, or serialized object graphs
- A single message is 10 MB, 50 MB, or larger
- The message broker's maximum message size has been increased repeatedly to accommodate ever-growing payloads
- Consumers regularly run out of memory when deserializing messages
- Network throughput between broker and consumers is saturated by a small number of messages
- Serialization and deserialization dominate consumer processing time

### Why It's Harmful

- **Broker instability** — large messages consume disproportionate memory, disk I/O, and network bandwidth on the broker. A burst of giant messages can starve other queues.
- **Memory pressure** — consumers must buffer the entire message in memory for deserialization. A few concurrent large messages can trigger out-of-memory errors.
- **Throughput degradation** — message throughput is measured in messages per second, but large messages turn this into an effective bytes-per-second bottleneck. A few giant messages can block thousands of small ones.
- **Replication lag** — in clustered brokers (Kafka, RabbitMQ mirrored queues), large messages increase replication lag and increase the window for data loss during a failover.
- **Retry cost** — when a large message fails, retrying it wastes significant network and compute resources. Each retry re-transmits the entire payload.
- **Serialization overhead** — serializing and deserializing multi-megabyte payloads adds latency to every consumer and increases garbage collection pressure in managed runtimes.

### The Fix

- Use the **Claim Check Pattern** — store large payloads in object storage (S3, Azure Blob Storage, GCS) and send a reference (URL, key) in the message
- Set and enforce maximum message size limits at the broker level — a reasonable default is 256 KB to 1 MB
- Send events, not data dumps — include only the fields consumers need to act, not the entire entity
- If a consumer needs more data, let it fetch from an API or database using an identifier from the message
- Compress message payloads (gzip, snappy, LZ4) when they are borderline — but prefer reducing payload size over compressing large ones
- Monitor average and P99 message sizes to detect payload creep early

> **Rule of thumb:** If a message is too large to comfortably print on a screen, it is too large for the broker. Messages should be kilobytes, not megabytes. If you are sending megabytes, you need the Claim Check Pattern.

---

## 6. Missing Idempotency

### The Anti-Pattern

Building consumers that break, produce incorrect results, or corrupt data when they receive the same message more than once. Assuming the broker delivers each message exactly once, when in reality most brokers guarantee at-least-once delivery.

### What It Looks Like

- A consumer charges a customer twice because it processed a duplicate payment message
- A consumer creates duplicate database records every time a message is redelivered after a timeout
- The team sees duplicate entries in the database and "fixes" the problem by adding deduplication in the UI layer
- Consumers acknowledge messages before processing completes — a crash between ack and processing loses the message
- Consumers acknowledge messages after processing — a crash after processing but before ack causes redelivery and duplicate processing
- A Kafka consumer rebalance triggers reprocessing of an entire partition, and every message creates duplicate records in the database

### Why It's Harmful

- **Financial errors** — duplicate payment processing, double-charging customers, or double-crediting accounts can have serious business and legal consequences.
- **Data corruption** — duplicate inserts, duplicate state transitions, or duplicate side effects produce inconsistent data that is difficult to detect and correct.
- **At-least-once is the norm** — Kafka, RabbitMQ, SQS, and most brokers guarantee at-least-once delivery by default. Exactly-once delivery is either unsupported or comes with severe performance and complexity costs.
- **Failures guarantee duplicates** — network timeouts, consumer crashes, broker failovers, and rebalancing all cause redeliveries. In a high-volume system, duplicates are not a possibility — they are a certainty.
- **Cascading duplicates** — a duplicate message processed by one service often triggers duplicate messages to downstream services, amplifying the problem across the architecture.
- **Hard to detect** — duplicate processing often produces results that look correct individually. Only when you compare records side-by-side do you see the duplicates, and by then reconciliation is expensive.

### The Fix

- Design every consumer to be idempotent — processing the same message twice must produce the same result as processing it once
- Use a deduplication store: store message IDs in a database or cache, and check before processing
- Use natural idempotency keys from the business domain (order ID, payment ID, transaction ID) rather than broker-assigned message IDs
- For database writes, use upserts (`INSERT ... ON CONFLICT DO UPDATE`) instead of inserts
- For state transitions, check the current state before applying the transition — reject invalid transitions
- Consider the Idempotent Consumer Pattern: combine the message ID check and the business operation in a single database transaction
- For Kafka, consider enabling `enable.idempotence=true` on producers to prevent duplicate publishes

> **Rule of thumb:** Assume every message will be delivered at least twice. Design accordingly. If your consumer cannot safely process the same message 10 times in a row and produce the correct result, it is not ready for production.

---

## 7. Tight Coupling Through Messages

### The Anti-Pattern

Using messaging infrastructure in a way that creates tight coupling between services — leaking internal schemas into messages, using point-to-point messaging when pub-sub is appropriate, or requiring consumers to know about producer internals.

### What It Looks Like

- Message payloads mirror the producer's internal database schema, including internal IDs, audit columns, and implementation details
- Changing a column name in the producer's database requires updating every consumer's deserialization code
- A producer publishes messages directly to consumer-specific queues instead of a shared topic or exchange
- Adding a new consumer requires changes to the producer's configuration
- Messages contain implementation-specific types (Java class names, .NET assembly-qualified types) in headers or payloads
- Consumer and producer must be deployed simultaneously because of shared message contracts
- The producer maintains a growing list of consumer-specific queues, and the configuration file grows with every new downstream service
- Removing a consumer requires a producer deployment to stop publishing to that consumer's queue

### Why It's Harmful

- **Defeats the purpose of messaging** — the primary benefit of messaging is decoupling. If every producer change requires consumer changes, you have synchronous coupling with extra infrastructure.
- **Deployment coordination** — tightly coupled message contracts force coordinated deployments across services. A producer cannot deploy without confirming every consumer can handle the new format.
- **Reduced autonomy** — teams cannot independently evolve their services. A database migration in one service becomes a cross-team project.
- **Fragile architecture** — point-to-point messaging means the producer must know about every consumer. Adding, removing, or modifying consumers requires producer changes.
- **Technology lock-in** — serialization-specific metadata in messages (Java serialization, .NET binary formatters) prevents consumers from using different technology stacks.

### The Fix

- Design message contracts as a public API — include only business-relevant fields, never internal implementation details
- Use pub-sub (Kafka topics, RabbitMQ exchanges, SNS topics) instead of point-to-point for events
- Define messages as domain events (`OrderPlaced`, `PaymentReceived`) not data dumps (`orders_table_row_v2`)
- Use language-agnostic serialization formats (JSON, Avro, Protobuf) — never language-specific binary serialization
- Maintain message contracts in a shared schema registry, versioned independently of producer code
- Apply the principle: producers own the event contract, consumers adapt to it
- Use consumer-driven contract testing (Pact) to verify compatibility without tight coupling

> **Rule of thumb:** If adding a new consumer requires changing the producer, you do not have a decoupled system — you have a distributed monolith with a message broker in the middle.

---

## 8. No Schema Evolution Strategy

### The Anti-Pattern

Changing message schemas (adding fields, removing fields, renaming fields, changing types) without a strategy for maintaining backward or forward compatibility. Breaking existing consumers with every schema change.

### What It Looks Like

- A producer adds a required field to a message, and every consumer crashes because they do not send the new field
- A field is renamed from `user_id` to `userId` and half the consumers fail to parse messages
- A field type changes from string to integer, and consumers that expected a string throw deserialization errors
- The team must deploy all consumers simultaneously whenever the producer changes a message format
- There is no schema registry — the message format is defined implicitly by whatever the producer happens to send

### Why It's Harmful

- **Consumer outages** — incompatible schema changes crash consumers. In a large system, a single schema change can break dozens of services simultaneously.
- **Deployment coordination** — without schema compatibility, every producer change requires synchronized deployment of all consumers. This defeats the operational independence that messaging is supposed to provide.
- **Silent data loss** — some deserialization libraries silently drop unknown fields or default missing fields to null/zero. Consumers may continue "working" while processing incorrect data.
- **Rollback failures** — if a producer is rolled back to an older version, it publishes the old schema. Consumers that were updated for the new schema may break in the opposite direction.
- **Growing technical debt** — without a schema strategy, teams avoid changing message formats at all, leading to schemas that accumulate workaround fields and never get cleaned up.
- **Cross-team friction** — schema disagreements between producing and consuming teams become a recurring source of conflict when there is no agreed-upon evolution policy.

### The Fix

- Use a schema registry (Confluent Schema Registry, AWS Glue Schema Registry, Apicurio) to manage and validate message schemas
- Define and enforce compatibility modes: backward compatible (new schema can read old data), forward compatible (old schema can read new data), or full compatibility (both)
- Follow safe evolution rules:
  - **Adding a field** — always make it optional with a sensible default
  - **Removing a field** — deprecate first, stop publishing it, then remove from the schema after all consumers have stopped reading it
  - **Renaming a field** — treat as adding a new field and removing the old one
  - **Changing a type** — almost always breaking; add a new field with the new type instead
- Use schema validation in CI/CD pipelines to catch incompatible changes before deployment
- Prefer Avro or Protobuf over JSON for schema evolution — they have built-in support for compatibility checks and default values
- Document the schema evolution policy and ensure all teams follow it

> **Rule of thumb:** Every schema change should be classifiable as backward compatible or breaking. If you cannot classify it, you do not have a schema evolution strategy — you have hope.

---

## 9. Ignoring Backpressure

### The Anti-Pattern

Producers publish messages without regard for consumer processing capacity. No flow control, no rate limiting, and no strategy for handling situations where producers generate messages faster than consumers can process them.

### What It Looks Like

- A batch job publishes 10 million messages in 5 minutes, and consumers that process 100 messages per second fall hopelessly behind
- A marketing campaign triggers a spike in events that overwhelms the order processing pipeline
- Queue depth grows linearly during peak hours and never recovers before the next peak
- Consumers scale horizontally but still cannot keep up because each consumer has a fixed processing rate limited by downstream dependencies (databases, APIs)
- The broker starts rejecting messages because memory or disk limits are reached, and the producer has no retry or buffering logic
- Auto-scaling is configured for consumers, but it reacts too slowly — by the time new instances are ready, the backlog has already caused downstream failures
- Producers publish messages at wire speed during batch imports with no throttling, saturating broker I/O for all tenants

### Why It's Harmful

- **Consumer crashes** — consumers overwhelmed with messages can run out of memory, exhaust database connection pools, or exceed rate limits on downstream services.
- **Broker overload** — unchecked message production can exhaust broker resources (memory, disk, network), affecting all queues and all services on the shared broker.
- **Stale data processing** — a growing backlog means consumers are processing increasingly stale messages. In real-time systems, processing a message from 4 hours ago may be worse than not processing it at all.
- **Cascading failures** — overwhelmed consumers may fail in ways that produce error messages, retry storms, or duplicate processing — each of which generates additional load.
- **Unpredictable latency** — end-to-end message processing latency becomes a function of queue depth. A queue with 1 million backlogged messages adds hours of latency to every new message.

### The Fix

- Implement consumer-side rate limiting — control how many messages each consumer processes per second to protect downstream dependencies
- Use prefetch/QoS settings (RabbitMQ `prefetch_count`, Kafka `max.poll.records`) to limit how many messages a consumer receives at once
- Implement producer-side rate limiting for batch jobs and bulk operations — spread load over time instead of publishing everything at once
- Monitor consumer lag (Kafka consumer group lag, RabbitMQ queue depth) and auto-scale consumers based on lag thresholds
- Define a shedding strategy for extreme backpressure: can old messages be skipped? Can processing be degraded (skip non-critical steps)?
- Use separate queues or topics for high-priority and low-priority messages so that critical messages are not blocked behind bulk operations
- Implement circuit breakers in consumers — if a downstream dependency is failing, stop consuming instead of filling the DLQ

> **Rule of thumb:** If your producers can publish 10x faster than your consumers can process, you need backpressure. The question is not whether a traffic spike will happen — it is whether your system will handle it gracefully when it does.

---

## 10. Missing Monitoring and Alerting

### The Anti-Pattern

Operating messaging infrastructure without monitoring consumer lag, queue depth, broker health, message processing rates, or error rates. Having no alerts for abnormal conditions.

### What It Looks Like

- The team discovers a consumer has been down for 3 days only when a customer reports missing data
- No dashboards exist for queue depth, consumer lag, or message throughput
- Broker disk usage hits 100% with no prior warning
- The team has no idea what "normal" queue depth looks like and cannot distinguish a backlog from a traffic spike
- Failed messages are logged to application logs but no one monitors those logs
- DLQ messages accumulate for weeks without anyone noticing
- A schema change breaks consumers, but the only signal is a slowly increasing error count that nobody sees because there are no alerts

### Why It's Harmful

- **Silent failures** — messaging failures are often invisible to end users. A consumer can stop processing for hours before anyone notices, and by then significant data may be lost or stale.
- **Delayed incident response** — without alerts, the mean time to detection (MTTD) for messaging problems is measured in hours or days instead of minutes. Every hour of delay increases the blast radius.
- **No capacity planning** — without historical metrics on message throughput and consumer lag, the team cannot predict when infrastructure needs scaling.
- **Undetectable degradation** — consumer processing time may slowly increase (due to GC pressure, downstream slowdowns, or resource contention) without monitoring to surface the trend.
- **Incident investigation is blind** — when something goes wrong, there is no data to determine what happened, when it started, or what caused it.
- **Toil accumulates** — without proactive monitoring, every problem requires manual investigation. Teams spend hours debugging issues that good alerting would have surfaced in minutes.

### The Fix

- Monitor these metrics as a minimum for every messaging system:
  - **Consumer lag** — messages produced minus messages consumed, per consumer group (Kafka) or queue depth (RabbitMQ, SQS)
  - **Message throughput** — messages published and consumed per second
  - **Processing latency** — time between message publish and consumer acknowledgment
  - **Error rate** — consumer processing failures per second
  - **DLQ depth** — messages in dead letter queues (alert when > 0)
  - **Broker health** — disk usage, memory usage, CPU, network I/O, replication lag
- Set alerts for:
  - Consumer lag exceeding a threshold (e.g., > 10,000 messages or > 5 minutes)
  - Consumer group losing active members (consumer crash)
  - DLQ receiving messages
  - Broker disk or memory above 80%
  - Message throughput dropping to zero (producer or consumer failure)
- Use dedicated monitoring tools: Kafka — Burrow, AKHQ, Confluent Control Center; RabbitMQ — built-in management UI, Prometheus exporter; Cloud — native monitoring (CloudWatch, Azure Monitor)
- Build dashboards that show the full message flow: producer → broker → consumer → downstream, with latency at each stage
- Conduct regular "messaging health checks" — review dashboards, verify alerts fire correctly, and confirm DLQs are empty

> **Rule of thumb:** If you cannot answer "what is the current consumer lag for service X?" within 30 seconds, your monitoring is insufficient. Messaging systems fail silently — monitoring is how you give them a voice.

---

## 11. Synchronous Mindset with Async Tools

### The Anti-Pattern

Using asynchronous messaging infrastructure but forcing synchronous request-response patterns — blocking a thread while waiting for a response message, implementing RPC over message queues, or requiring immediate consistency in an eventually consistent system.

### What It Looks Like

- A web API publishes a message and then polls a response queue, blocking the HTTP request until a reply arrives (or times out after 30 seconds)
- Services create temporary "reply-to" queues for every request, creating thousands of short-lived queues that the broker must manage
- A saga or workflow blocks at each step, waiting for the next event before proceeding — turning an async pipeline into a slow synchronous chain
- Business stakeholders require "instant" confirmation of actions that flow through asynchronous pipelines, so developers add blocking waits
- The team uses messaging between services that sit next to each other and need sub-millisecond responses
- HTTP 504 Gateway Timeout errors are common because the API is blocking on a message response from a consumer that is processing a backlog
- Every service call adds 200ms+ of latency because it round-trips through a message broker instead of making a direct call

### Why It's Harmful

- **Thread exhaustion** — blocking a web server thread while waiting for an async response ties up limited resources. Under load, all threads are blocked waiting for messaging responses, and the service stops accepting new requests.
- **False timeouts** — synchronous waits introduce artificial timeout requirements. If the consumer is slow or temporarily backlogged, the producer times out and treats a successful-but-slow operation as a failure.
- **Wasted infrastructure** — messaging adds latency (serialization, network hop, broker routing, deserialization) compared to a direct HTTP or gRPC call. If you need synchronous request-response, use a synchronous protocol.
- **Temporary queue explosion** — creating reply-to queues per request puts significant load on the broker (queue creation, metadata management, cleanup) and does not scale.
- **Negates messaging benefits** — the decoupling, buffering, and resilience advantages of messaging are lost when you force synchronous behavior on top.

### The Fix

- Accept eventual consistency — design UIs and APIs to acknowledge receipt asynchronously ("Your order has been received") rather than blocking for the final result
- Use webhooks or server-sent events (SSE) to notify clients when async processing completes
- For request-response patterns, use synchronous protocols (HTTP, gRPC) instead of messaging
- If async request-response is truly needed, use correlation IDs and callback mechanisms rather than blocking threads
- Design frontends for optimistic UI updates — show the user a pending state and update when the event confirms completion
- Use the Async API pattern for inter-service communication: publish a command, return 202 Accepted, and let the consumer publish a completion event
- Reserve messaging for use cases that benefit from decoupling: event notification, event-carried state transfer, async command processing, and workload distribution

> **Rule of thumb:** If you are awaiting a message response with a timeout, you are doing synchronous communication with extra steps and extra latency. Use HTTP/gRPC for request-response. Use messaging for fire-and-forget, fan-out, and workload buffering.

---

## 12. No Message Versioning

### The Anti-Pattern

Publishing messages without version identifiers. Adding, changing, or removing fields in message payloads without any mechanism for consumers to detect which version of the message they are processing.

### What It Looks Like

- A new field is added to the `OrderCreated` event, and consumers that do not know about the field silently ignore it — or crash trying to parse it
- A producer starts sending a `v2` format but there is no version field, so consumers cannot distinguish `v1` from `v2` messages
- During a rollback, the producer reverts to the old message format, and consumers that were updated for the new format break
- The team wants to change a message structure but has no idea which consumers read which fields, because there is no versioning to track
- Multiple versions of the same event coexist in a topic (due to different producer versions), and consumers have no way to handle them differently
- A rollback of the producer to a prior release causes all consumers that upgraded for the new format to fail

### Why It's Harmful

- **Breaking changes are undetectable** — without a version field, consumers cannot determine whether a message matches their expected format. Deserialization may silently produce incorrect data.
- **Impossible rollbacks** — if producers and consumers are not versioned, rolling back one without the other can create incompatible message formats with no way to detect or handle the mismatch.
- **No migration path** — without version information, it is impossible to write consumers that handle both old and new message formats during a transition period.
- **Coordination tax** — every message format change requires synchronizing deployments across all producers and consumers. This scales badly as the number of services grows.
- **Lost audit trail** — when investigating incidents, the team cannot determine which version of a message a consumer processed, making root cause analysis significantly harder.

### The Fix

- Include a version identifier in every message — either in a header (`message-version: 2`) or in the payload (`"version": 2`)
- Follow semantic versioning conventions for message contracts:
  - **Patch** — documentation or metadata changes, no payload change
  - **Minor** — new optional fields added (backward compatible)
  - **Major** — breaking changes (field removed, type changed, required field added)
- For major version changes, publish to a new topic or include the version so consumers can route accordingly (`orders.v1`, `orders.v2`)
- Write consumers that handle multiple versions: check the version field and apply the appropriate deserialization/processing logic
- Maintain a message catalog that documents every event type, its current version, its schema, and which services produce and consume it
- Use a schema registry to enforce compatibility rules and reject incompatible schema changes at publish time
- When deprecating a message version, communicate a timeline: announce deprecation, stop producing the old version, give consumers time to migrate, then remove support

> **Rule of thumb:** Every message should carry its version. A consumer that receives a message without a version field is flying blind — it cannot adapt, cannot migrate, and cannot safely evolve.

---

## Quick Reference Checklist

Use this checklist for pre-production messaging reviews:

### Critical (Must Fix Before Production)

- [ ] No application state stored exclusively in message queues — queues are for transit, databases are for storage
- [ ] Every queue has retention policies configured (TTL, max-length, max-size)
- [ ] Dead letter queues configured on every queue and topic
- [ ] DLQ depth monitored and alerted on (alert when depth > 0)
- [ ] Maximum retry count configured on every consumer (3–5 retries with exponential backoff)
- [ ] Every consumer is idempotent — processing the same message twice produces the same result
- [ ] Deduplication mechanism in place (message ID store, upserts, or natural idempotency keys)
- [ ] Producers and consumers handle backpressure (rate limiting, prefetch limits, circuit breakers)

### High (Fix Before Production or Immediately After)

- [ ] Message ordering strategy defined — per-entity ordering via partition keys or message groups
- [ ] Maximum message size enforced at the broker level (≤ 1 MB recommended)
- [ ] Large payloads use the Claim Check Pattern (store in object storage, send reference)
- [ ] Message contracts are versioned with a version field in every message
- [ ] Schema evolution strategy defined and enforced (backward/forward compatibility)
- [ ] Messages use language-agnostic serialization (JSON, Avro, Protobuf)
- [ ] Pub-sub used for events; point-to-point used only for directed commands

### Medium (Address Within First Sprint)

- [ ] Consumer lag, throughput, error rate, and broker health monitored with dashboards
- [ ] Alerts configured for consumer lag, DLQ depth, broker resource usage, and throughput drops
- [ ] No synchronous blocking on message responses — async patterns used throughout
- [ ] Message contracts documented in a schema registry or message catalog
- [ ] Consumers handle out-of-order messages gracefully (timestamps, sequence numbers, version checks)
- [ ] Backlog recovery strategy defined (skip stale messages vs. process all)
- [ ] Load testing performed with realistic message volumes and failure scenarios
- [ ] Runbooks documented for common messaging failures (consumer down, broker full, DLQ overflow)

---

## Next Steps

| Step | Action | Resource |
|------|--------|----------|
| 1 | **Score yourself** — Use the [Quick Reference Checklist](#quick-reference-checklist) to assess your current messaging systems | This document |
| 2 | **Fix critical severity first** — Address 🔴 Critical anti-patterns (#1, #2, #3, #6, #9) before others | This document |
| 3 | **Read the companion guide** — Review the correct messaging patterns and best practices | [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md) |
| 4 | **Review messaging patterns** — Understand common patterns for reliable messaging | [04-PATTERNS.md](04-PATTERNS.md) |
| 5 | **Review schema management** — Learn how to evolve message schemas safely | [05-SCHEMA-MANAGEMENT.md](05-SCHEMA-MANAGEMENT.md) |
| 6 | **Review reliability** — Understand delivery guarantees and failure handling | [06-RELIABILITY.md](06-RELIABILITY.md) |
| 7 | **Review monitoring** — Set up proper observability for messaging infrastructure | [07-MONITORING.md](07-MONITORING.md) |
| 8 | **Follow the learning path** — Work through the messaging curriculum from the beginning | [00-OVERVIEW.md](00-OVERVIEW.md) |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Messaging Anti-Patterns documentation |
