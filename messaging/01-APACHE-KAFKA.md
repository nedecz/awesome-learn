# Apache Kafka

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Kafka Architecture](#kafka-architecture)
   - [Brokers and Clusters](#brokers-and-clusters)
   - [ZooKeeper Mode](#zookeeper-mode)
   - [KRaft Mode](#kraft-mode)
   - [Controller and Leader Election](#controller-and-leader-election)
3. [Topics and Partitions](#topics-and-partitions)
   - [Topic Creation](#topic-creation)
   - [Partition Strategy](#partition-strategy)
   - [Replication Factor](#replication-factor)
   - [Log Segments and Retention](#log-segments-and-retention)
4. [Producers](#producers)
   - [Producer API](#producer-api)
   - [Partitioning](#partitioning)
   - [Acknowledgments](#acknowledgments)
   - [Batching and Compression](#batching-and-compression)
   - [Idempotent Producers](#idempotent-producers)
5. [Consumers and Consumer Groups](#consumers-and-consumer-groups)
   - [Consumer API](#consumer-api)
   - [Consumer Group Coordination](#consumer-group-coordination)
   - [Offset Management](#offset-management)
   - [Rebalancing](#rebalancing)
6. [Kafka Streams](#kafka-streams)
   - [Stream Processing Fundamentals](#stream-processing-fundamentals)
   - [KStreams and KTables](#kstreams-and-ktables)
   - [Windowing](#windowing)
7. [Kafka Connect](#kafka-connect)
   - [Source and Sink Connectors](#source-and-sink-connectors)
   - [Transformations](#transformations)
   - [Managed Connectors](#managed-connectors)
8. [Schema Registry](#schema-registry)
   - [Why Schema Registry](#why-schema-registry)
   - [Compatibility Modes](#compatibility-modes)
   - [Integration Workflow](#integration-workflow)
9. [Security](#security)
   - [Authentication with SASL](#authentication-with-sasl)
   - [Authorization with ACLs](#authorization-with-acls)
   - [Encryption with TLS](#encryption-with-tls)
10. [Performance Tuning](#performance-tuning)
    - [Producer Configuration](#producer-configuration)
    - [Consumer Configuration](#consumer-configuration)
    - [Partition Count](#partition-count)
    - [Batch Size and Compression](#batch-size-and-compression)
11. [Kafka in Production](#kafka-in-production)
    - [Cluster Sizing](#cluster-sizing)
    - [Monitoring](#monitoring)
    - [Upgrade Strategies](#upgrade-strategies)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

Apache Kafka is a distributed event streaming platform used for building real-time data pipelines and streaming applications. Originally developed at LinkedIn and open-sourced in 2011, Kafka has become the de facto standard for high-throughput, fault-tolerant, and durable event streaming. It underpins mission-critical systems at thousands of organizations — from real-time analytics and log aggregation to event sourcing and microservices integration.

### Target Audience

- **Developers** building event-driven applications that produce and consume messages through Kafka topics
- **Data Engineers** designing streaming data pipelines with Kafka Connect and Kafka Streams
- **DevOps Engineers** deploying, configuring, and monitoring Kafka clusters in production environments
- **Architects** evaluating Kafka as the backbone for event-driven and microservices architectures

### Scope

- Kafka architecture: brokers, clusters, ZooKeeper vs KRaft consensus
- Topics, partitions, replication, and log retention
- Producer API: partitioning, acknowledgments, batching, compression, idempotency
- Consumer API: consumer groups, offset management, rebalancing strategies
- Kafka Streams: KStreams, KTables, windowing, stateful processing
- Kafka Connect: source and sink connectors, single message transforms
- Confluent Schema Registry: Avro/Protobuf/JSON Schema integration, compatibility modes
- Security: SASL authentication, ACL authorization, TLS encryption
- Performance tuning and production operations

---

## Kafka Architecture

Kafka is designed as a distributed, partitioned, replicated commit log. Understanding its core architecture is essential before working with producers, consumers, or stream processing.

### Brokers and Clusters

A **Kafka broker** is a single server that stores data and serves client requests. A **cluster** is a group of brokers working together to provide fault tolerance and horizontal scalability.

```
Kafka Cluster (3 Brokers)
──────────────────────────

┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Broker 1   │    │   Broker 2   │    │   Broker 3   │
│              │    │              │    │              │
│  Partition 0 │    │  Partition 1 │    │  Partition 2 │
│  (Leader)    │    │  (Leader)    │    │  (Leader)    │
│              │    │              │    │              │
│  Partition 1 │    │  Partition 2 │    │  Partition 0 │
│  (Follower)  │    │  (Follower)  │    │  (Follower)  │
└──────────────┘    └──────────────┘    └──────────────┘
        │                  │                  │
        └──────────────────┼──────────────────┘
                           │
                    Internal Replication
```

**Key concepts:**

- Each broker is identified by a unique integer ID
- Brokers store partitions on local disk as append-only log files
- Clients connect to any broker to discover the full cluster topology (bootstrap servers)
- Leadership for each partition is distributed across brokers for load balancing

### ZooKeeper Mode

In traditional deployments, Kafka relies on **Apache ZooKeeper** for cluster metadata management, controller election, and broker registration.

```
ZooKeeper Mode (Legacy)
───────────────────────

┌──────────────────────────┐
│     ZooKeeper Ensemble   │
│  ┌─────┐ ┌─────┐ ┌─────┐│
│  │ ZK1 │ │ ZK2 │ │ ZK3 ││
│  └─────┘ └─────┘ └─────┘│
└────────────┬─────────────┘
             │
   ┌─────────┼─────────┐
   ▼         ▼         ▼
┌──────┐  ┌──────┐  ┌──────┐
│Broker│  │Broker│  │Broker│
│  1   │  │  2   │  │  3   │
└──────┘  └──────┘  └──────┘

ZooKeeper responsibilities:
  - Broker registration and liveness
  - Controller election
  - Topic and partition metadata
  - ACL storage
  - Consumer group offsets (legacy)
```

> **Note:** ZooKeeper mode is deprecated as of Kafka 3.5 and will be removed in Kafka 4.0. New deployments should use KRaft mode.

### KRaft Mode

**KRaft** (Kafka Raft) replaces ZooKeeper with a built-in Raft-based consensus protocol. Metadata is managed internally by a set of controller nodes, eliminating the operational overhead of running a separate ZooKeeper ensemble.

```
KRaft Mode (Recommended)
────────────────────────

┌──────────────────────────────────────────┐
│           Kafka Cluster (KRaft)          │
│                                          │
│  ┌────────────┐  ┌────────────┐          │
│  │ Controller │  │ Controller │  ...     │
│  │  (Voter)   │  │  (Voter)   │          │
│  └────────────┘  └────────────┘          │
│         │               │                │
│  ┌──────┴───────────────┴──────┐         │
│  │     Metadata Log (Raft)     │         │
│  └─────────────────────────────┘         │
│         │               │                │
│  ┌──────┴──┐     ┌──────┴──┐             │
│  │ Broker  │     │ Broker  │   ...       │
│  │   1     │     │   2     │             │
│  └─────────┘     └─────────┘             │
└──────────────────────────────────────────┘

KRaft advantages:
  - No external dependency (no ZooKeeper)
  - Faster controller failover
  - Simplified operations and deployment
  - Better scalability for metadata
```

**KRaft configuration:**

```properties
# server.properties (KRaft mode)
process.roles=broker,controller
node.id=1
controller.quorum.voters=1@controller1:9093,2@controller2:9093,3@controller3:9093
controller.listener.names=CONTROLLER
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
log.dirs=/var/kafka-logs
```

### Controller and Leader Election

The **controller** is the broker responsible for managing partition leadership assignments and replica state. In ZooKeeper mode, one broker is elected controller via ZooKeeper. In KRaft mode, controllers use the Raft protocol to maintain a replicated metadata log.

| Aspect | ZooKeeper Mode | KRaft Mode |
|--------|---------------|------------|
| Metadata storage | ZooKeeper znodes | Internal Raft log |
| Controller election | ZooKeeper ephemeral node | Raft leader election |
| Failover time | Seconds to minutes | Sub-second |
| Operational complexity | High (two systems) | Low (single system) |
| Deprecation status | Deprecated (Kafka 3.5+) | Production-ready (Kafka 3.3+) |

---

## Topics and Partitions

A **topic** is a named feed of messages. Topics are divided into **partitions** — ordered, immutable sequences of records. Partitions are the fundamental unit of parallelism and distribution in Kafka.

### Topic Creation

Topics can be created via the CLI, Admin API, or automatically on first produce (if auto-creation is enabled).

```bash
# Create a topic with 6 partitions and replication factor 3
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create \
  --topic orders \
  --partitions 6 \
  --replication-factor 3

# Describe a topic
kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe \
  --topic orders

# List all topics
kafka-topics.sh --bootstrap-server localhost:9092 --list
```

**Topic-level configuration overrides:**

```bash
# Set retention to 7 days and max segment size to 1 GB
kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter \
  --entity-type topics \
  --entity-name orders \
  --add-config retention.ms=604800000,segment.bytes=1073741824
```

### Partition Strategy

Choosing the right number of partitions affects throughput, consumer parallelism, and cluster balance.

```
Topic: "orders" — 6 Partitions
───────────────────────────────

Partition 0: [msg0, msg6, msg12, msg18, ...]  → Broker 1 (Leader)
Partition 1: [msg1, msg7, msg13, msg19, ...]  → Broker 2 (Leader)
Partition 2: [msg2, msg8, msg14, msg20, ...]  → Broker 3 (Leader)
Partition 3: [msg3, msg9, msg15, msg21, ...]  → Broker 1 (Leader)
Partition 4: [msg4, msg10, msg16, msg22, ...] → Broker 2 (Leader)
Partition 5: [msg5, msg11, msg17, msg23, ...] → Broker 3 (Leader)

  - Each partition is an ordered, append-only log
  - Ordering is guaranteed WITHIN a partition only
  - Consumer group parallelism is bounded by partition count
```

**Guidelines for partition count:**

| Factor | Low Partitions (1–6) | Medium (6–30) | High (30–100+) |
|--------|---------------------|---------------|-----------------|
| Throughput | Low | Moderate | High |
| Consumer parallelism | Limited | Good | Maximum |
| End-to-end latency | Lowest | Low | Higher |
| Leader election time | Fast | Moderate | Slower |
| Memory overhead | Minimal | Moderate | Significant |

> **Rule of thumb:** Start with the number of partitions equal to or greater than the expected maximum number of consumers in a consumer group. You can increase partitions later, but you cannot decrease them.

### Replication Factor

Each partition is replicated across multiple brokers. One replica is the **leader** (handles reads and writes), and the others are **followers** (replicate from the leader).

```
Partition 0 — Replication Factor 3
──────────────────────────────────

┌──────────┐     ┌──────────┐     ┌──────────┐
│ Broker 1 │     │ Broker 2 │     │ Broker 3 │
│          │     │          │     │          │
│ Leader   │────▶│ Follower │     │ Follower │
│          │────▶│          │     │          │
│ [0,1,2,  │     │ [0,1,2,  │     │ [0,1,2,  │
│  3,4,5]  │     │  3,4,5]  │     │  3,4,5]  │
└──────────┘     └──────────┘     └──────────┘
                       ▲                ▲
                       │                │
                       └───── Replication ──────┘

  ISR (In-Sync Replicas): {Broker 1, Broker 2, Broker 3}
  - If Leader fails, a follower in the ISR is promoted
  - Replication factor 3 tolerates 2 broker failures
```

**Key settings:**

| Setting | Description | Recommended |
|---------|-------------|-------------|
| `replication.factor` | Number of copies per partition | 3 (production) |
| `min.insync.replicas` | Minimum replicas that must ACK a write | 2 |
| `unclean.leader.election.enable` | Allow out-of-sync replica to become leader | `false` (avoid data loss) |

### Log Segments and Retention

Each partition is stored as a sequence of **log segments** on disk. Kafka retains data based on time or size, not on whether consumers have read it.

```
Partition Log on Disk
─────────────────────

/var/kafka-logs/orders-0/
├── 00000000000000000000.log      ← Active segment
├── 00000000000000000000.index    ← Offset index
├── 00000000000000000000.timeindex← Timestamp index
├── 00000000000050000000.log      ← Older segment
├── 00000000000050000000.index
└── 00000000000050000000.timeindex

Retention policies:
  retention.ms=604800000     → 7 days (default)
  retention.bytes=-1         → Unlimited (default)
  segment.bytes=1073741824   → 1 GB per segment
  cleanup.policy=delete      → Delete old segments (default)
  cleanup.policy=compact     → Keep latest value per key
```

**Log compaction** retains at least the last known value for each message key, which is essential for changelog topics and KTable backing stores.

---

## Producers

Producers are client applications that publish records to Kafka topics. The producer API handles serialization, partitioning, batching, compression, and delivery acknowledgment.

### Producer API

A basic producer workflow:

```
Producer Workflow
─────────────────

Application
    │
    ▼
┌──────────────────┐
│ ProducerRecord   │ ← (topic, key, value, headers)
└────────┬─────────┘
         │
    ┌────▼────┐
    │Serializer│ ← Key and value serializers
    └────┬────┘
         │
    ┌────▼──────┐
    │Partitioner│ ← Determines target partition
    └────┬──────┘
         │
    ┌────▼───────────┐
    │ Record Batch   │ ← Accumulates records per partition
    │ (Send Buffer)  │
    └────┬───────────┘
         │
    ┌────▼────┐
    │ Sender  │ ← Background I/O thread
    └────┬────┘
         │
         ▼
    Kafka Broker
```

**Java producer example:**

```java
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092,broker2:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("acks", "all");
props.put("enable.idempotence", "true");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);

ProducerRecord<String, String> record =
    new ProducerRecord<>("orders", "order-123", "{\"item\":\"widget\",\"qty\":5}");

producer.send(record, (metadata, exception) -> {
    if (exception == null) {
        System.out.printf("Sent to partition %d offset %d%n",
            metadata.partition(), metadata.offset());
    } else {
        exception.printStackTrace();
    }
});

producer.close();
```

### Partitioning

The producer determines which partition receives each record. The default partitioner uses the murmur2 hash of the key. Records with no key use sticky partitioning (batch-level round-robin).

| Strategy | When Used | Ordering Guarantee |
|----------|-----------|-------------------|
| Key-based (hash) | Key is present | All records with the same key go to the same partition |
| Sticky partitioning | Key is null | Records batched to the same partition, then rotated |
| Custom partitioner | `partitioner.class` set | Application-defined logic |
| Explicit partition | Partition set on ProducerRecord | Direct assignment |

### Acknowledgments

The `acks` setting controls how many replicas must acknowledge a write before the producer considers it successful.

```
Acknowledgment Levels
─────────────────────

acks=0 (Fire and Forget)
  Producer ──► Broker     (no response waited)
  - Fastest, highest risk of data loss

acks=1 (Leader Only)
  Producer ──► Broker (Leader ACK) ──► response
  - Leader writes to local log, responds immediately
  - Data loss if leader crashes before replication

acks=all / acks=-1 (All ISR)
  Producer ──► Broker (Leader) ──► Followers replicate
                                       │
                              All ISR ACK ──► response
  - Strongest durability guarantee
  - Combined with min.insync.replicas=2 for safety
```

| Setting | Durability | Latency | Throughput |
|---------|-----------|---------|------------|
| `acks=0` | None | Lowest | Highest |
| `acks=1` | Leader only | Low | High |
| `acks=all` | Full ISR | Higher | Lower |

### Batching and Compression

Kafka producers batch records before sending to amortize network overhead and enable compression.

| Setting | Description | Default | Tuning Guidance |
|---------|-------------|---------|-----------------|
| `batch.size` | Max bytes per batch (per partition) | 16384 (16 KB) | Increase for throughput (e.g., 64 KB–1 MB) |
| `linger.ms` | Time to wait for batch to fill before sending | 0 | Set 5–100 ms for better batching |
| `compression.type` | Compression codec for batches | `none` | Use `lz4` or `zstd` for throughput |
| `buffer.memory` | Total memory for send buffers | 33554432 (32 MB) | Increase for high-volume producers |
| `max.in.flight.requests.per.connection` | Max unacknowledged requests | 5 | Set to 1 for strict ordering without idempotency |

**Compression comparison:**

| Codec | Compression Ratio | CPU Cost | Speed |
|-------|-------------------|----------|-------|
| `none` | 1.0x | None | Fastest |
| `gzip` | High (~5–7x) | High | Slow |
| `snappy` | Moderate (~3–4x) | Low | Fast |
| `lz4` | Moderate (~3–4x) | Low | Fastest compressed |
| `zstd` | High (~5–7x) | Moderate | Fast |

### Idempotent Producers

When `enable.idempotence=true`, the producer assigns a sequence number to each record. The broker deduplicates records by tracking the producer ID and sequence number, preventing duplicates from retries.

```
Idempotent Producer
───────────────────

Producer (PID=1)
  │
  ├── Send: seq=0, seq=1, seq=2
  │         │
  │    (network failure on seq=1 ACK)
  │         │
  ├── Retry: seq=1     ← same PID + seq
  │         │
  │    Broker detects duplicate → discards
  │
  Result: [seq=0, seq=1, seq=2]  ← no duplicates, in order
```

**Requirements for idempotency:**

- `acks=all`
- `max.in.flight.requests.per.connection` ≤ 5
- `retries` > 0 (default is already `Integer.MAX_VALUE`)

---

## Consumers and Consumer Groups

Consumers read records from Kafka topics. **Consumer groups** enable parallel consumption: each partition in a topic is assigned to exactly one consumer within a group.

### Consumer API

```
Consumer Workflow
─────────────────

┌────────────────┐
│   Consumer     │
│                │
│  1. subscribe()│ ← Subscribe to topic(s)
│  2. poll()     │ ← Fetch records in a loop
│  3. process()  │ ← Application logic
│  4. commit()   │ ← Commit offsets
└────────────────┘
        │
        ▼
   Kafka Broker
```

**Java consumer example:**

```java
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092");
props.put("group.id", "order-processing");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("enable.auto.commit", "false");
props.put("auto.offset.reset", "earliest");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Collections.singletonList("orders"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("partition=%d offset=%d key=%s value=%s%n",
            record.partition(), record.offset(), record.key(), record.value());
        // process record
    }
    consumer.commitSync();
}
```

### Consumer Group Coordination

A consumer group ensures that each partition is consumed by exactly one consumer in the group. This provides both load balancing and fault tolerance.

```
Consumer Group: "order-processing"
──────────────────────────────────

Topic: "orders" (6 partitions)

  Consumer 1          Consumer 2          Consumer 3
  ┌──────────┐        ┌──────────┐        ┌──────────┐
  │ P0, P1   │        │ P2, P3   │        │ P4, P5   │
  └──────────┘        └──────────┘        └──────────┘

  - 6 partitions / 3 consumers = 2 partitions each
  - Adding a 4th consumer → rebalance (some get 1, some get 2)
  - Adding a 7th consumer → one consumer is idle (6 partitions max)
  - If Consumer 2 crashes → P2 and P3 reassigned to survivors
```

**Multiple consumer groups can read the same topic independently:**

```
Independent Consumer Groups
───────────────────────────

Topic: "orders"
      │
      ├──► Consumer Group A: "order-processing"
      │      (reads all partitions, tracks its own offsets)
      │
      ├──► Consumer Group B: "analytics"
      │      (reads all partitions, tracks its own offsets)
      │
      └──► Consumer Group C: "audit-log"
             (reads all partitions, tracks its own offsets)
```

### Offset Management

Offsets track each consumer group's position in each partition. Kafka stores offsets in the internal `__consumer_offsets` topic.

```
Offset Tracking
───────────────

Partition 0: [0] [1] [2] [3] [4] [5] [6] [7] [8] [9]
                               ▲              ▲
                               │              │
                        Committed         Log-End
                         Offset           Offset (LEO)
                          (3)               (9)

  Consumer lag = LEO - Committed Offset = 9 - 3 = 6 messages behind
```

| Commit Strategy | Description | Trade-off |
|----------------|-------------|-----------|
| Auto-commit (`enable.auto.commit=true`) | Commits periodically (default 5s) | Simple; risk of reprocessing or skipping on crash |
| Sync commit (`commitSync()`) | Blocks until offset is committed | Safest; higher latency |
| Async commit (`commitAsync()`) | Non-blocking commit | Faster; no retry on failure |
| Manual per-record | Commit after each record | Most control; highest overhead |
| Manual per-batch | Commit after processing a poll batch | Good balance of safety and throughput |

### Rebalancing

Rebalancing redistributes partition assignments when consumers join, leave, or crash. During a rebalance, consumption pauses on affected partitions.

**Rebalancing strategies:**

| Strategy | Description | Impact |
|----------|-------------|--------|
| Eager (Stop-the-World) | All consumers revoke all partitions, then reassign | Full pause; simple logic |
| Cooperative (Incremental) | Only affected partitions are revoked and reassigned | Minimal disruption; recommended |

```properties
# Enable cooperative rebalancing
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

**Reducing unnecessary rebalances:**

| Setting | Purpose | Recommended Value |
|---------|---------|-------------------|
| `session.timeout.ms` | Time before broker considers consumer dead | 45000 (45s) |
| `heartbeat.interval.ms` | Frequency of heartbeats to group coordinator | 15000 (15s) |
| `max.poll.interval.ms` | Max time between poll() calls | 300000 (5min) or more for long processing |
| `group.instance.id` | Static membership — avoids rebalance on restart | Unique per consumer instance |

---

## Kafka Streams

Kafka Streams is a client library for building stream processing applications that read from and write to Kafka topics. It requires no external processing cluster — the application runs as a standard Java process.

### Stream Processing Fundamentals

```
Kafka Streams Application
─────────────────────────

  Input Topic(s)          Kafka Streams App            Output Topic(s)
  ┌────────────┐      ┌─────────────────────┐      ┌────────────────┐
  │  orders    │ ──►  │  Stream Topology     │ ──►  │ order-totals   │
  │            │      │                     │      │                │
  │  payments  │ ──►  │  ┌─────┐  ┌──────┐ │ ──►  │ fraud-alerts   │
  │            │      │  │ Map │─▶│Filter│ │      │                │
  └────────────┘      │  └─────┘  └──┬───┘ │      └────────────────┘
                      │              │     │
                      │         ┌────▼───┐ │
                      │         │GroupBy/ │ │
                      │         │Aggregate│ │
                      │         └────────┘ │
                      │                     │
                      │   State Store (local)│
                      └─────────────────────┘
```

**Key properties of Kafka Streams:**

- No separate cluster — runs as a JVM application
- Exactly-once processing semantics
- Local state stores backed by RocksDB and changelog topics
- Horizontally scalable — add more instances, partitions are rebalanced
- Fault-tolerant — state is rebuilt from changelog topics on failure

### KStreams and KTables

Kafka Streams provides two core abstractions:

| Abstraction | Represents | Analogy | Example |
|-------------|-----------|---------|---------|
| **KStream** | Unbounded stream of events | Append-only log | All orders placed |
| **KTable** | Changelog with latest value per key | Materialized view / table | Current account balances |

```
KStream vs KTable
─────────────────

Input records (key=A):
  (A, 10) → (A, 20) → (A, 30)

KStream interpretation (all events):
  [A:10, A:20, A:30]  ← three separate events

KTable interpretation (latest per key):
  {A: 30}             ← only the current value
```

**Example topology:**

```java
StreamsBuilder builder = new StreamsBuilder();

// Read a stream of order events
KStream<String, Order> orders = builder.stream("orders");

// Filter for high-value orders, rekey by region, aggregate totals
KTable<String, Long> regionTotals = orders
    .filter((key, order) -> order.getAmount() > 100)
    .selectKey((key, order) -> order.getRegion())
    .groupByKey()
    .count(Materialized.as("region-counts"));

// Write results to an output topic
regionTotals.toStream().to("region-order-counts");
```

### Windowing

Windowing groups events by time for time-based aggregations.

| Window Type | Description | Use Case |
|-------------|-------------|----------|
| **Tumbling** | Fixed-size, non-overlapping intervals | Hourly counts |
| **Hopping** | Fixed-size, overlapping intervals | 5-min window advancing every 1 min |
| **Sliding** | Window defined by difference between record timestamps | Records within 10 min of each other |
| **Session** | Dynamic windows closed by inactivity gap | User activity sessions |

```
Windowing Types
───────────────

Tumbling (size=5min):
  |──── W1 ────|──── W2 ────|──── W3 ────|
  0            5            10           15 (minutes)

Hopping (size=5min, advance=2min):
  |──── W1 ────|
       |──── W2 ────|
            |──── W3 ────|
  0    2    4    6    8   10

Session (gap=3min):
  ●  ● ●        ●  ●              ●
  |── S1 ──|    |─ S2 ─|         |S3|
  (events within 3min of each other grouped)
```

---

## Kafka Connect

Kafka Connect is a framework for streaming data between Kafka and external systems (databases, search indexes, file systems, cloud storage) without writing custom code.

### Source and Sink Connectors

```
Kafka Connect Architecture
──────────────────────────

  External Systems             Kafka Connect              Kafka Cluster
  ┌──────────────┐         ┌──────────────────┐         ┌──────────────┐
  │  PostgreSQL  │ ──────► │ Source Connector  │ ──────► │   Topics     │
  │  MySQL       │ ──────► │  (JDBC, Debezium)│ ──────► │              │
  └──────────────┘         └──────────────────┘         │              │
                                                        │              │
  ┌──────────────┐         ┌──────────────────┐         │              │
  │Elasticsearch │ ◄────── │  Sink Connector  │ ◄────── │              │
  │  S3 Bucket   │ ◄────── │  (ES, S3, JDBC)  │ ◄────── │              │
  └──────────────┘         └──────────────────┘         └──────────────┘
```

**Connector configuration example (JDBC source):**

```json
{
  "name": "postgres-source",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url": "jdbc:postgresql://db:5432/mydb",
    "connection.user": "kafka_connect",
    "connection.password": "${file:/secrets/db-password.txt:password}",
    "table.whitelist": "orders,customers",
    "mode": "incrementing",
    "incrementing.column.name": "id",
    "topic.prefix": "db.",
    "poll.interval.ms": "5000"
  }
}
```

**Common connectors:**

| Connector | Type | Description |
|-----------|------|-------------|
| JDBC Source/Sink | Source + Sink | Relational databases (PostgreSQL, MySQL, SQL Server) |
| Debezium | Source | Change data capture from databases |
| Elasticsearch Sink | Sink | Index data in Elasticsearch |
| S3 Sink | Sink | Write data to Amazon S3 |
| BigQuery Sink | Sink | Load data to Google BigQuery |
| FileStream | Source + Sink | Read/write files (development/testing) |

### Transformations

**Single Message Transforms (SMTs)** modify records in-flight without custom code. They are applied in the connector pipeline before records are written.

```
SMT Pipeline
─────────────

Source Record ──► SMT 1 ──► SMT 2 ──► SMT 3 ──► Kafka Topic
                  │          │          │
               Rename     Filter     Add
               field      fields     timestamp
```

**Common SMTs:**

| Transform | Purpose |
|-----------|---------|
| `ReplaceField` | Include, exclude, or rename fields |
| `MaskField` | Mask sensitive field values |
| `TimestampConverter` | Convert timestamp formats |
| `ExtractField` | Extract a single field from a struct |
| `InsertField` | Add a static or metadata field |
| `ValueToKey` | Copy value fields to the record key |

**Configuration example:**

```json
{
  "transforms": "dropSensitive,addTimestamp",
  "transforms.dropSensitive.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
  "transforms.dropSensitive.blacklist": "ssn,credit_card",
  "transforms.addTimestamp.type": "org.apache.kafka.connect.transforms.InsertField$Value",
  "transforms.addTimestamp.timestamp.field": "ingested_at"
}
```

### Managed Connectors

Kafka Connect can run in **standalone mode** (single process, for development) or **distributed mode** (clustered, for production).

| Mode | Workers | Use Case |
|------|---------|----------|
| Standalone | 1 | Development, testing, edge deployments |
| Distributed | N (cluster) | Production — fault-tolerant, scalable |

In distributed mode, connectors and tasks are balanced across workers. If a worker fails, its tasks are restarted on surviving workers.

**Managed Kafka Connect services** (Confluent Cloud, Amazon MSK Connect, Aiven) handle infrastructure, scaling, and monitoring — reducing operational overhead.

---

## Schema Registry

### Why Schema Registry

As Kafka topics evolve, producers and consumers must agree on the data format. Without governance, schema changes can break consumers. The **Confluent Schema Registry** provides centralized schema management and compatibility enforcement.

```
Schema Registry Workflow
────────────────────────

Producer                   Schema Registry              Consumer
   │                            │                          │
   ├── 1. Register schema ────► │                          │
   │   (Avro/Protobuf/JSON)    │                          │
   │                            │                          │
   ◄── 2. Return schema ID ────┤                          │
   │                            │                          │
   ├── 3. Serialize record ──►  │                          │
   │   (schema-id + payload)    │                          │
   │         │                  │                          │
   │         ▼                  │                          │
   │    Kafka Broker            │                          │
   │         │                  │                          │
   │         ▼                  │                          │
   │                            │ ◄── 4. Fetch schema ─────┤
   │                            │                          │
   │                            ├── 5. Return schema ────► │
   │                            │                          │
   │                            │    6. Deserialize ────►  │
   │                            │       record             │
```

### Compatibility Modes

Schema Registry enforces compatibility rules when a new schema version is registered.

| Mode | Rule | Example |
|------|------|---------|
| `BACKWARD` | New schema can read data written by old schema | Add optional field with default |
| `FORWARD` | Old schema can read data written by new schema | Remove optional field |
| `FULL` | Both backward and forward compatible | Add/remove optional fields with defaults |
| `BACKWARD_TRANSITIVE` | New schema can read all previous versions | Across all historical versions |
| `FORWARD_TRANSITIVE` | All previous schemas can read new data | Across all historical versions |
| `FULL_TRANSITIVE` | Both directions across all versions | Strictest mode |
| `NONE` | No compatibility checks | Not recommended for production |

**Setting compatibility mode:**

```bash
# Set compatibility for a specific subject
curl -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"compatibility": "BACKWARD"}' \
  http://schema-registry:8081/config/orders-value
```

### Integration Workflow

**Avro schema example:**

```json
{
  "type": "record",
  "name": "Order",
  "namespace": "com.example.events",
  "fields": [
    {"name": "orderId", "type": "string"},
    {"name": "customerId", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "currency", "type": "string", "default": "USD"},
    {"name": "timestamp", "type": "long", "logicalType": "timestamp-millis"}
  ]
}
```

| Format | Schema Evolution | Payload Size | Human Readable |
|--------|-----------------|-------------|----------------|
| Avro | Excellent (built-in) | Compact (binary) | No |
| Protobuf | Excellent (built-in) | Compact (binary) | No |
| JSON Schema | Good | Larger (text) | Yes |

---

## Security

Securing a Kafka cluster involves three layers: authentication (who you are), authorization (what you can do), and encryption (protecting data in transit).

### Authentication with SASL

Kafka supports multiple SASL mechanisms for client and inter-broker authentication.

| Mechanism | Description | Use Case |
|-----------|-------------|----------|
| `SASL/PLAIN` | Username/password (plaintext) | Development (use with TLS in production) |
| `SASL/SCRAM-SHA-256/512` | Salted challenge-response | Production without Kerberos |
| `SASL/GSSAPI` | Kerberos authentication | Enterprise environments with existing KDC |
| `SASL/OAUTHBEARER` | OAuth 2.0 token-based | Cloud-native and modern identity providers |

**Broker SASL configuration:**

```properties
# server.properties
listeners=SASL_SSL://0.0.0.0:9093
security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
sasl.enabled.mechanisms=SCRAM-SHA-512

# JAAS configuration
listener.name.sasl_ssl.scram-sha-512.sasl.jaas.config=\
  org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="admin" \
  password="admin-secret";
```

### Authorization with ACLs

Kafka's built-in **Authorizer** uses Access Control Lists (ACLs) to control which principals can perform which operations on which resources.

```bash
# Grant producer access to a topic
kafka-acls.sh --bootstrap-server localhost:9093 \
  --command-config admin.properties \
  --add \
  --allow-principal User:order-service \
  --operation Write \
  --topic orders

# Grant consumer group access
kafka-acls.sh --bootstrap-server localhost:9093 \
  --command-config admin.properties \
  --add \
  --allow-principal User:analytics-service \
  --operation Read \
  --topic orders \
  --group analytics-group

# List ACLs for a topic
kafka-acls.sh --bootstrap-server localhost:9093 \
  --command-config admin.properties \
  --list \
  --topic orders
```

| Resource | Operations | Example |
|----------|-----------|---------|
| Topic | Read, Write, Create, Delete, Describe, Alter | Grant write on `orders` |
| Group | Read, Describe, Delete | Grant read on `analytics-group` |
| Cluster | Create, Alter, Describe, ClusterAction | Admin operations |
| TransactionalId | Write, Describe | Exactly-once producers |

### Encryption with TLS

TLS encrypts data in transit between clients and brokers and between brokers (inter-broker communication).

```
TLS Encryption
──────────────

Producer ══════ TLS ══════ Broker 1 ══════ TLS ══════ Broker 2
                             ║
Consumer ══════ TLS ═════════╝

  ════ = Encrypted connection (TLS 1.2 / 1.3)
```

**Broker TLS configuration:**

```properties
# server.properties
listeners=SSL://0.0.0.0:9093
ssl.keystore.location=/var/kafka/ssl/kafka.keystore.jks
ssl.keystore.password=${file:/secrets/keystore-password.txt:password}
ssl.key.password=${file:/secrets/key-password.txt:password}
ssl.truststore.location=/var/kafka/ssl/kafka.truststore.jks
ssl.truststore.password=${file:/secrets/truststore-password.txt:password}
ssl.client.auth=required
ssl.enabled.protocols=TLSv1.3,TLSv1.2
```

**Security checklist:**

- Enable `SASL_SSL` listeners (authentication + encryption)
- Set `ssl.client.auth=required` for mutual TLS (mTLS)
- Use SCRAM or OAUTHBEARER over PLAIN in production
- Enable ACLs with `authorizer.class.name=kafka.security.authorizer.AclAuthorizer`
- Disable `allow.everyone.if.no.acl.found` in production
- Encrypt inter-broker communication with `security.inter.broker.protocol=SASL_SSL`
- Rotate certificates and credentials on a regular schedule

---

## Performance Tuning

Kafka performance depends on proper configuration of producers, consumers, brokers, and infrastructure. Below are the most impactful tuning parameters.

### Producer Configuration

| Setting | Default | Throughput Tuning | Latency Tuning |
|---------|---------|-------------------|----------------|
| `acks` | `all` | `1` (less durable) | `all` (safest) |
| `batch.size` | 16384 | 65536–1048576 | 16384 (smaller) |
| `linger.ms` | 0 | 10–100 | 0 (send immediately) |
| `compression.type` | `none` | `lz4` or `zstd` | `lz4` (fast codec) |
| `buffer.memory` | 33554432 | Increase for bursts | Default |
| `max.in.flight.requests` | 5 | 5 | 1 (strict ordering) |

### Consumer Configuration

| Setting | Default | Throughput Tuning | Latency Tuning |
|---------|---------|-------------------|----------------|
| `fetch.min.bytes` | 1 | 65536+ (batch fetches) | 1 (respond immediately) |
| `fetch.max.wait.ms` | 500 | 500–1000 | 100 (low wait) |
| `max.poll.records` | 500 | 1000–5000 | 100 (faster commits) |
| `max.partition.fetch.bytes` | 1048576 | 2–10 MB | 1 MB |
| `auto.offset.reset` | `latest` | — | — |

### Partition Count

Partition count directly affects parallelism and throughput. More partitions allow more consumers to read in parallel, but increase overhead.

```
Partition Scaling
─────────────────

Target throughput: 100 MB/s
Single consumer throughput: 20 MB/s

  Minimum partitions = 100 / 20 = 5

  Recommended: 6–10 partitions (headroom for spikes)

Considerations:
  - Each partition uses file handles, memory, and CPU on the broker
  - More partitions = longer leader election time during failures
  - More partitions = larger metadata overhead
  - Kafka can handle 10,000+ partitions per broker (but monitor resources)
```

### Batch Size and Compression

The interplay between `batch.size`, `linger.ms`, and `compression.type` has the largest impact on producer throughput.

```
Batching + Compression Impact
─────────────────────────────

Scenario A: batch.size=16KB, linger.ms=0, compression=none
  → Many small network requests, low throughput

Scenario B: batch.size=64KB, linger.ms=20, compression=lz4
  → Fewer, larger compressed batches, ~3–5x throughput improvement

Scenario C: batch.size=1MB, linger.ms=100, compression=zstd
  → Maximum batching, highest throughput, ~100ms added latency
```

---

## Kafka in Production

Running Kafka reliably in production requires attention to cluster sizing, monitoring, and safe upgrade procedures.

### Cluster Sizing

| Dimension | Guidance |
|-----------|---------|
| **Broker count** | Minimum 3 for fault tolerance; scale based on throughput and storage |
| **CPU** | 8–16 cores per broker; Kafka is I/O-bound, not CPU-bound |
| **Memory** | 32–64 GB RAM; allocate 6–8 GB for JVM heap, rest for OS page cache |
| **Disk** | Use dedicated SSDs or high-throughput HDDs; JBOD (multiple disks) supported |
| **Network** | 10 Gbps+ recommended; replication multiplies network traffic |
| **Replication factor** | 3 for production (tolerates 2 failures with `min.insync.replicas=2`) |
| **Partitions per broker** | Target ≤ 4000 partitions per broker as a starting guideline |

**JVM configuration:**

```bash
# Recommended JVM settings for Kafka broker
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g"
export KAFKA_JVM_PERFORMANCE_OPTS="-server \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=20 \
  -XX:InitiatingHeapOccupancyPercent=35 \
  -XX:+ExplicitGCInvokesConcurrent"
```

### Monitoring

**Critical metrics to monitor:**

| Metric | Source | Alert Threshold |
|--------|--------|----------------|
| Under-replicated partitions | Broker JMX | > 0 for sustained period |
| Consumer group lag | Consumer JMX / `kafka-consumer-groups.sh` | Growing continuously |
| Request latency (Produce/Fetch) | Broker JMX | p99 > SLA threshold |
| Active controller count | Broker JMX | ≠ 1 |
| ISR shrink/expand rate | Broker JMX | Frequent shrinks |
| Disk utilization | OS metrics | > 80% |
| Network utilization | OS metrics | > 70% of NIC capacity |
| GC pause time | JVM metrics | > 200ms |

**Monitoring tool:**

```bash
# Check consumer group lag
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe \
  --group order-processing

# Output:
# GROUP            TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# order-processing orders   0          4500            4520            20
# order-processing orders   1          3800            3800            0
# order-processing orders   2          5100            5200            100
```

**Recommended monitoring stack:**

- **JMX Exporter** + **Prometheus** + **Grafana** for broker and client metrics
- **Burrow** or **Kafka Lag Exporter** for consumer lag monitoring
- **Cruise Control** for automated partition rebalancing and anomaly detection

### Upgrade Strategies

Kafka supports **rolling upgrades** without downtime. The process ensures compatibility by upgrading the wire protocol version separately from the software version.

```
Rolling Upgrade Process
───────────────────────

Step 1: Upgrade brokers one at a time (software only)
  Broker 1 ──► Upgrade binary, restart
  Broker 2 ──► Upgrade binary, restart
  Broker 3 ──► Upgrade binary, restart
  (inter.broker.protocol.version stays at OLD version)

Step 2: Verify cluster health
  - All brokers running new version
  - No under-replicated partitions
  - Consumer lag stable

Step 3: Update protocol version
  inter.broker.protocol.version=NEW
  log.message.format.version=NEW
  (rolling restart again)

Step 4: Verify and clean up
  - Monitor for 24–48 hours
  - Remove deprecated configs
```

**Upgrade checklist:**

- Read the release notes and migration guide for the target version
- Test the upgrade in a staging environment first
- Ensure `replication.factor` ≥ 3 and `min.insync.replicas` ≥ 2
- Monitor under-replicated partitions during rolling restart
- Upgrade clients after brokers are fully upgraded
- Keep the ability to roll back for at least 48 hours

---

## Next Steps

Continue your messaging learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | Messaging Fundamentals | Core messaging concepts, delivery guarantees, event-driven architecture |
| [02-RABBITMQ.md](02-RABBITMQ.md) | RabbitMQ | Message queues, exchanges, bindings, AMQP protocol |
| [03-PATTERNS.md](03-PATTERNS.md) | Messaging Patterns | Saga, outbox, CQRS, event sourcing, dead-letter queues |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Apache Kafka documentation |
