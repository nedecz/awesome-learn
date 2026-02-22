# Messaging Monitoring

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Key Metrics](#key-metrics)
   - [Consumer Lag](#consumer-lag)
   - [Throughput](#throughput)
   - [Latency](#latency)
   - [Error Rates](#error-rates)
   - [Metrics Summary](#metrics-summary)
3. [Kafka Monitoring](#kafka-monitoring)
   - [Broker Metrics](#broker-metrics)
   - [Producer Metrics](#producer-metrics)
   - [Consumer Group Metrics](#consumer-group-metrics)
   - [JMX Monitoring](#jmx-monitoring)
   - [Kafka Exporter](#kafka-exporter)
4. [RabbitMQ Monitoring](#rabbitmq-monitoring)
   - [Queue Depth](#queue-depth)
   - [Message Rates](#message-rates)
   - [Connection and Channel Counts](#connection-and-channel-counts)
   - [Management Plugin](#management-plugin)
5. [Cloud Service Monitoring](#cloud-service-monitoring)
   - [CloudWatch for SQS and SNS](#cloudwatch-for-sqs-and-sns)
   - [Azure Monitor for Service Bus](#azure-monitor-for-service-bus)
   - [Cloud Monitoring for Pub/Sub](#cloud-monitoring-for-pubsub)
6. [Alerting Strategies](#alerting-strategies)
   - [Threshold-Based Alerts](#threshold-based-alerts)
   - [Rate-of-Change Alerts](#rate-of-change-alerts)
   - [Anomaly Detection](#anomaly-detection)
   - [Alert Fatigue Prevention](#alert-fatigue-prevention)
7. [Dashboards](#dashboards)
   - [Essential Dashboards](#essential-dashboards)
   - [PromQL and Grafana Examples](#promql-and-grafana-examples)
   - [Dashboard Design Principles](#dashboard-design-principles)
8. [Distributed Tracing](#distributed-tracing)
   - [Trace Context Propagation](#trace-context-propagation)
   - [Correlation IDs](#correlation-ids)
   - [OpenTelemetry Integration](#opentelemetry-integration)
9. [Log Management](#log-management)
   - [Structured Logging for Messaging](#structured-logging-for-messaging)
   - [Log Correlation](#log-correlation)
   - [Centralized Logging](#centralized-logging)
10. [Health Checks](#health-checks)
    - [Broker Health](#broker-health)
    - [Consumer Health](#consumer-health)
    - [Connectivity Checks](#connectivity-checks)
11. [Capacity Planning](#capacity-planning)
    - [Throughput Estimation](#throughput-estimation)
    - [Storage Planning](#storage-planning)
    - [Scaling Triggers](#scaling-triggers)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

Monitoring is the foundation of reliable messaging infrastructure. Without visibility into what your brokers, producers, and consumers are doing, failures go undetected, performance degrades silently, and capacity problems surface only when systems are already overwhelmed.

Messaging systems are inherently asynchronous. Unlike synchronous HTTP calls where a failure returns an immediate error, a dropped or delayed message can go unnoticed for hours. Monitoring closes this gap by making the invisible visible — consumer lag, processing latency, error rates, and broker health all become actionable signals.

```
┌────────────────────────────────────────────────────────────┐
│              Why Monitoring Messaging Matters               │
│                                                            │
│  Without Monitoring:                                       │
│                                                            │
│  Producer ──▶ Broker ──▶ Consumer ──▶ ???                  │
│                                                            │
│  - Did the message arrive?          Unknown                │
│  - Is the consumer keeping up?      Unknown                │
│  - How long did processing take?    Unknown                │
│  - Are errors increasing?           Unknown                │
│                                                            │
│  With Monitoring:                                          │
│                                                            │
│  Producer ──▶ Broker ──▶ Consumer ──▶ Database             │
│      │           │           │                             │
│      ▼           ▼           ▼                             │
│  ┌─────────────────────────────────┐                       │
│  │       Observability Stack       │                       │
│  │  Metrics │ Traces │ Logs        │                       │
│  └─────────────┬───────────────────┘                       │
│                │                                           │
│                ▼                                           │
│  ┌─────────────────────────────────┐                       │
│  │  Dashboards  │  Alerts  │  SLOs │                       │
│  └─────────────────────────────────┘                       │
└────────────────────────────────────────────────────────────┘
```

### Target Audience

- **Platform engineers** responsible for messaging infrastructure
- **Application developers** building producer and consumer services
- **SREs and on-call engineers** responding to messaging incidents
- **DevOps engineers** managing monitoring and alerting pipelines

### Scope

This document covers monitoring strategies for messaging systems including Apache Kafka, RabbitMQ, and cloud-managed services (AWS SQS/SNS, Azure Service Bus, Google Pub/Sub). It addresses metrics collection, alerting, dashboards, distributed tracing, log management, health checks, and capacity planning.

---

## Key Metrics

Effective messaging monitoring starts with the right metrics. These four categories cover the vast majority of operational concerns.

### Consumer Lag

Consumer lag is the most critical metric for any messaging system. It measures how far behind a consumer is from the latest message produced. Rising lag means consumers are not keeping up with the production rate.

```
┌───────────────────────────────────────────────────────┐
│                   Consumer Lag                         │
│                                                       │
│  Topic Partition:                                     │
│                                                       │
│  Messages: [0][1][2][3][4][5][6][7][8][9][10][11]    │
│                          ▲                    ▲       │
│                          │                    │       │
│                   Consumer Offset      Latest Offset  │
│                     (position 5)        (position 11) │
│                                                       │
│                   Lag = 11 - 5 = 6 messages           │
│                                                       │
│  Healthy:    Lag stable near 0                        │
│  Warning:    Lag increasing steadily                  │
│  Critical:   Lag growing unbounded                    │
└───────────────────────────────────────────────────────┘
```

**What consumer lag tells you:**

- **Lag = 0** — Consumer is fully caught up
- **Lag stable but non-zero** — Consumer processes at the same rate as the producer but with a fixed delay
- **Lag increasing** — Consumer is slower than the producer; investigate processing bottlenecks or scale consumers
- **Lag spike then recovery** — Transient issue (deployment, GC pause, downstream slowdown)

### Throughput

Throughput measures the rate of messages flowing through the system, typically expressed in messages per second or bytes per second.

```
Throughput Measurement Points:

  Producer Rate          Broker Ingestion Rate        Consumer Rate
  (msgs/sec out)         (msgs/sec in + out)          (msgs/sec in)
       │                        │                          │
       ▼                        ▼                          ▼
  ┌──────────┐           ┌──────────┐              ┌──────────┐
  │ Producer │ ────────▶ │  Broker  │ ───────────▶ │ Consumer │
  └──────────┘           └──────────┘              └──────────┘

  Key throughput metrics:
  - Messages produced per second (per topic, per partition)
  - Messages consumed per second (per consumer group)
  - Bytes in/out per second (network saturation)
  - Batch size (messages per produce/fetch request)
```

### Latency

Latency in messaging systems has multiple dimensions. Each must be measured independently.

| Latency Type | Definition | Typical Range |
|-------------|-----------|---------------|
| Produce latency | Time to acknowledge a produced message | 1–50ms |
| End-to-end latency | Time from produce to consumer processing complete | 10ms–10s |
| Processing latency | Time the consumer spends processing one message | Varies by workload |
| Commit latency | Time to commit offsets or acknowledge | 1–20ms |
| Broker propagation | Time for message to replicate across brokers | 1–100ms |

```
┌──────────────────────────────────────────────────────────┐
│               End-to-End Latency Breakdown               │
│                                                          │
│  t0          t1          t2          t3          t4      │
│  │           │           │           │           │       │
│  ▼           ▼           ▼           ▼           ▼       │
│  Produce ──▶ Broker ──▶ Fetch ──▶ Process ──▶ Commit    │
│  Request     Ack        Delivered   Complete    Offset   │
│                                                          │
│  ├── Produce ──┤                                         │
│  │   Latency   │                                         │
│  │             ├── Queue ──┤                              │
│  │             │   Wait    │                              │
│  │             │           ├── Processing ──┤             │
│  │             │           │    Latency     │             │
│  ├──────────── End-to-End Latency ─────────┤             │
└──────────────────────────────────────────────────────────┘
```

### Error Rates

Track errors at every stage of the messaging pipeline.

- **Producer errors** — Send failures, serialization errors, timeout errors
- **Broker errors** — Replication failures, disk errors, authorization failures
- **Consumer errors** — Deserialization errors, processing exceptions, commit failures
- **DLQ ingestion rate** — Messages routed to dead letter queues per minute

### Metrics Summary

| Metric | Signal | Alert Threshold (Example) |
|--------|--------|--------------------------|
| Consumer lag | Consumers falling behind | > 10,000 messages for 5 min |
| Messages/sec (in) | Production rate | Drop > 50% from baseline |
| Messages/sec (out) | Consumption rate | Drop > 50% from baseline |
| End-to-end latency P99 | Pipeline slowness | > 5s |
| Processing latency P99 | Consumer bottleneck | > 2s |
| Error rate | System failures | > 1% of messages |
| DLQ depth | Persistent failures | > 0 messages |

---

## Kafka Monitoring

Kafka exposes hundreds of metrics through JMX. Focus on the metrics that directly indicate operational health.

### Broker Metrics

```
┌────────────────────────────────────────────────────────────┐
│                  Kafka Broker Metrics                       │
│                                                            │
│  Resource Metrics:                                         │
│  ┌──────────────────────────────────────────────────┐      │
│  │ CPU Usage              [████████░░] 78%          │      │
│  │ Disk Usage             [██████░░░░] 60%          │      │
│  │ Network In             245 MB/s                  │      │
│  │ Network Out            512 MB/s                  │      │
│  │ Open File Descriptors  12,401 / 65,536           │      │
│  └──────────────────────────────────────────────────┘      │
│                                                            │
│  Kafka-Specific Metrics:                                   │
│  ┌──────────────────────────────────────────────────┐      │
│  │ Under-replicated partitions    2                 │      │
│  │ Offline partitions             0                 │      │
│  │ Active controller count        1                 │      │
│  │ ISR shrink rate                0.1/min           │      │
│  │ Request handler idle %         45%               │      │
│  │ Log flush latency (ms)         3.2               │      │
│  └──────────────────────────────────────────────────┘      │
└────────────────────────────────────────────────────────────┘
```

| Metric | JMX Bean | Critical When |
|--------|----------|---------------|
| Under-replicated partitions | `kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions` | > 0 for sustained period |
| Offline partitions | `kafka.controller:type=KafkaController,name=OfflinePartitionsCount` | > 0 (data unavailable) |
| Active controller count | `kafka.controller:type=KafkaController,name=ActiveControllerCount` | != 1 on exactly one broker |
| ISR shrink rate | `kafka.server:type=ReplicaManager,name=IsrShrinksPerSec` | Sustained shrinkage |
| Request handler idle % | `kafka.server:type=KafkaRequestHandlerPool,name=RequestHandlerAvgIdlePercent` | < 30% (overloaded) |
| Log flush latency | `kafka.log:type=LogFlushStats,name=LogFlushRateAndTimeMs` | P99 > 100ms |

### Producer Metrics

| Metric | Description | Watch For |
|--------|-------------|-----------|
| `record-send-rate` | Messages sent per second | Unexpected drops |
| `record-error-rate` | Failed sends per second | Any non-zero value |
| `request-latency-avg` | Average produce request latency | Increasing trend |
| `batch-size-avg` | Average batch size in bytes | Too small = inefficient |
| `buffer-available-bytes` | Free buffer memory | Approaching zero |
| `records-per-request-avg` | Messages per produce request | Decreasing = smaller batches |

### Consumer Group Metrics

| Metric | Description | Watch For |
|--------|-------------|-----------|
| `records-lag-max` | Maximum lag across all partitions | Sustained increase |
| `records-consumed-rate` | Messages consumed per second | Unexpected drops |
| `fetch-latency-avg` | Average time per fetch request | Increasing trend |
| `commit-latency-avg` | Average offset commit time | Spikes |
| `rebalance-rate` | Consumer group rebalances per minute | Frequent rebalances |
| `assigned-partitions` | Partitions assigned to consumer | Uneven distribution |

### JMX Monitoring

Kafka brokers expose metrics via Java Management Extensions (JMX). A typical monitoring pipeline scrapes JMX with a Prometheus JMX exporter and visualizes with Grafana.

```
┌──────────────────────────────────────────────────────────┐
│              Kafka JMX Monitoring Pipeline                │
│                                                          │
│  ┌───────────────┐     ┌──────────────────┐              │
│  │ Kafka Broker  │     │ JMX Exporter     │              │
│  │               │     │ (Java Agent)     │              │
│  │  JMX Beans ───┼────▶│                  │              │
│  │               │     │ :7071/metrics    │              │
│  └───────────────┘     └────────┬─────────┘              │
│                                 │                        │
│                                 │ HTTP scrape            │
│                                 ▼                        │
│                        ┌──────────────────┐              │
│                        │   Prometheus     │              │
│                        │                  │              │
│                        │ Scrape interval: │              │
│                        │   15–30s         │              │
│                        └────────┬─────────┘              │
│                                 │                        │
│                                 │ Query                  │
│                                 ▼                        │
│                        ┌──────────────────┐              │
│                        │    Grafana       │              │
│                        │   Dashboards     │              │
│                        └──────────────────┘              │
└──────────────────────────────────────────────────────────┘
```

**JMX exporter configuration (excerpt):**

```
# jmx-exporter-config.yaml
rules:
  - pattern: kafka.server<type=BrokerTopicMetrics,
             name=(MessagesInPerSec|BytesInPerSec|BytesOutPerSec)>
    name: kafka_server_broker_topic_metrics_$1
    type: GAUGE

  - pattern: kafka.server<type=ReplicaManager,
             name=(UnderReplicatedPartitions|PartitionCount)>
    name: kafka_server_replica_manager_$1
    type: GAUGE
```

### Kafka Exporter

The Kafka Exporter is a standalone tool that exposes consumer group lag and topic metadata as Prometheus metrics, complementing the JMX exporter.

| Metric Exported | Description |
|----------------|-------------|
| `kafka_consumergroup_lag` | Per-partition consumer lag |
| `kafka_consumergroup_current_offset` | Current consumer offset |
| `kafka_topic_partition_current_offset` | Latest offset per partition |
| `kafka_topic_partitions` | Number of partitions per topic |
| `kafka_brokers` | Number of brokers in the cluster |

---

## RabbitMQ Monitoring

RabbitMQ provides built-in monitoring through its management plugin and Prometheus-compatible metrics endpoint.

### Queue Depth

Queue depth (the number of messages waiting in a queue) is the RabbitMQ equivalent of consumer lag.

```
┌───────────────────────────────────────────────────────┐
│                Queue Depth Over Time                   │
│                                                       │
│  Messages                                             │
│  in Queue                                             │
│    5000 │          ╱╲                                  │
│    4000 │         ╱  ╲        ← Consumer restarted    │
│    3000 │        ╱    ╲                               │
│    2000 │       ╱      ╲                              │
│    1000 │ ─────╱        ╲──────── ← Normal            │
│       0 │─────────────────────────────────            │
│         └──────────────────────────────── Time        │
│         6:00   6:15   6:30   6:45   7:00              │
│                                                       │
│  Healthy: Queue depth stays near zero                 │
│  Warning: Queue depth growing over minutes            │
│  Critical: Queue depth growing over hours             │
└───────────────────────────────────────────────────────┘
```

### Message Rates

| Metric | Description | Healthy Signal |
|--------|-------------|----------------|
| Publish rate | Messages entering the queue per second | Stable or predictable pattern |
| Deliver rate | Messages delivered to consumers per second | Matches publish rate |
| Acknowledge rate | Messages acknowledged per second | Close to deliver rate |
| Redelivery rate | Messages redelivered after nack/timeout | Near zero |
| Unacked messages | Messages delivered but not yet acknowledged | Low and stable |

### Connection and Channel Counts

RabbitMQ connections and channels consume broker resources. Unbounded growth indicates connection leaks.

```
┌───────────────────────────────────────────────────────┐
│          Connection and Channel Monitoring             │
│                                                       │
│  Connections:   152 / 500 (limit)                     │
│  Channels:      304 / 2000 (limit)                    │
│  Channels/Conn: 2.0 (avg)                             │
│                                                       │
│  Watch for:                                           │
│  ✗ Connection count growing without new consumers     │
│  ✗ Channels per connection > 10 (possible leak)       │
│  ✗ Connections from single IP dominating              │
│  ✗ Connections in "blocking" state                    │
│                                                       │
│  Resource Alarms:                                     │
│  ┌─────────────────────────────────────────────┐      │
│  │ Memory alarm:  Triggered when > high_watermark│     │
│  │ Disk alarm:    Triggered when free < limit    │     │
│  │ Effect:        Producers blocked from sending │     │
│  └─────────────────────────────────────────────┘      │
└───────────────────────────────────────────────────────┘
```

### Management Plugin

The RabbitMQ management plugin provides an HTTP API and web UI for monitoring.

| Endpoint | Description |
|----------|-------------|
| `GET /api/overview` | Cluster-wide stats (message rates, queue totals, node info) |
| `GET /api/queues` | All queues with depth, rates, consumer counts |
| `GET /api/queues/{vhost}/{name}` | Detailed stats for a specific queue |
| `GET /api/connections` | Active connections with traffic stats |
| `GET /api/nodes` | Node health, memory, disk, file descriptors |
| `GET /api/healthchecks/node` | Node health check (HTTP 200 if healthy) |

**Prometheus integration:** RabbitMQ 3.8+ includes a built-in Prometheus metrics endpoint at `/metrics` that exposes all key metrics without additional plugins.

---

## Cloud Service Monitoring

Managed messaging services provide built-in monitoring through their respective cloud platforms.

### CloudWatch for SQS and SNS

```
┌───────────────────────────────────────────────────────────┐
│           AWS CloudWatch Metrics for SQS                   │
│                                                           │
│  Queue Health:                                            │
│  ┌─────────────────────────────────────────────────┐      │
│  │ ApproximateNumberOfMessagesVisible    1,234     │      │
│  │ ApproximateNumberOfMessagesNotVisible   456     │      │
│  │ ApproximateAgeOfOldestMessage         120s      │      │
│  │ NumberOfMessagesSent                  500/min   │      │
│  │ NumberOfMessagesReceived              480/min   │      │
│  │ NumberOfMessagesDeleted               475/min   │      │
│  └─────────────────────────────────────────────────┘      │
│                                                           │
│  DLQ Metrics:                                             │
│  ┌─────────────────────────────────────────────────┐      │
│  │ ApproximateNumberOfMessagesVisible (DLQ)    12  │      │
│  │ ApproximateAgeOfOldestMessage (DLQ)      3600s  │      │
│  └─────────────────────────────────────────────────┘      │
│                                                           │
│  SNS Metrics:                                             │
│  ┌─────────────────────────────────────────────────┐      │
│  │ NumberOfMessagesPublished             600/min   │      │
│  │ NumberOfNotificationsDelivered        590/min   │      │
│  │ NumberOfNotificationsFailed            10/min   │      │
│  └─────────────────────────────────────────────────┘      │
└───────────────────────────────────────────────────────────┘
```

| SQS Metric | Alert When |
|-----------|------------|
| `ApproximateAgeOfOldestMessage` | > threshold (consumers too slow) |
| `ApproximateNumberOfMessagesVisible` | Growing unbounded |
| `NumberOfMessagesSent` vs `NumberOfMessagesDeleted` | Diverging (processing gap) |
| DLQ `ApproximateNumberOfMessagesVisible` | > 0 |

### Azure Monitor for Service Bus

| Metric | Description | Alert Trigger |
|--------|-------------|---------------|
| Active Messages | Messages in queue/subscription awaiting delivery | Sustained growth |
| Dead-lettered Messages | Messages moved to DLQ | > 0 |
| Incoming Messages | Messages sent to entity per period | Unexpected drop |
| Outgoing Messages | Messages completed by receivers per period | Divergence from incoming |
| Server Errors | 5xx errors from Service Bus | Any occurrence |
| Throttled Requests | Requests throttled due to quota | Any occurrence |

### Cloud Monitoring for Pub/Sub

| Metric | Description | Alert Trigger |
|--------|-------------|---------------|
| `subscription/num_undelivered_messages` | Backlog size | Sustained growth |
| `subscription/oldest_unacked_message_age` | Age of oldest unacked message | > threshold |
| `topic/send_request_count` | Publish rate | Unexpected drop |
| `subscription/pull_request_count` | Pull rate | Drop below baseline |
| `subscription/dead_letter_message_count` | DLQ messages | > 0 |
| `subscription/ack_latencies` | Time to acknowledge | P99 > threshold |

---

## Alerting Strategies

Good alerting detects real problems without overwhelming operators with noise.

### Threshold-Based Alerts

The simplest alerting strategy: fire when a metric crosses a fixed boundary.

```
┌───────────────────────────────────────────────────────┐
│             Threshold-Based Alert Example              │
│                                                       │
│  Consumer Lag                                         │
│    15K │                    ╱╲                         │
│        │                   ╱  ╲                       │
│  ──────│─ ─ ─ ─ ─ ─ ─ ─ ╱─ ─ ╲─ ─ ─  Critical     │
│    10K │                ╱      ╲      (10,000)       │
│        │               ╱        ╲                     │
│  ──────│─ ─ ─ ─ ─ ─ ╱─ ─ ─ ─ ─ ╲─ ─  Warning      │
│     5K │            ╱              ╲   (5,000)       │
│        │           ╱                ╲                 │
│      0 │──────────╱                  ╲────           │
│        └──────────────────────────────── Time        │
│                                                       │
│  Alert fires when lag > 10K for 5 minutes             │
│  Warning fires when lag > 5K for 10 minutes           │
└───────────────────────────────────────────────────────┘
```

**Recommended thresholds:**

| Metric | Warning | Critical | Duration |
|--------|---------|----------|----------|
| Consumer lag | > 5,000 msgs | > 10,000 msgs | 5 min |
| End-to-end latency P99 | > 2s | > 5s | 5 min |
| Error rate | > 0.5% | > 2% | 5 min |
| DLQ depth | > 0 | > 100 | 1 min |
| Broker disk usage | > 75% | > 90% | 10 min |
| Under-replicated partitions | > 0 | > 0 for 10 min | Immediate |

### Rate-of-Change Alerts

Rate-of-change alerts fire when a metric is increasing or decreasing faster than expected, catching problems before thresholds are breached.

```
Rate-of-Change Example:

  Consumer Lag
    10K │
        │                         ╱  ← Slope = +2000/min
     5K │                      ╱     (alert: rate > 500/min)
        │                   ╱
     2K │               ╱
        │           ╱
      0 │──────────
        └──────────────────────── Time

  Alert: "Consumer lag increasing at 2000 msgs/min"
  (fires before reaching the 10K threshold)
```

### Anomaly Detection

Anomaly detection learns normal patterns and alerts on deviations, useful for metrics with variable baselines.

- **Seasonal patterns** — Throughput follows daily/weekly cycles; a fixed threshold produces false alerts during off-peak and misses issues during peak
- **Statistical methods** — Standard deviation bands, moving averages, EWMA
- **Machine learning** — Trained models that adapt to changing patterns

```
┌───────────────────────────────────────────────────────┐
│            Anomaly Detection vs Fixed Threshold        │
│                                                       │
│  Messages/sec                                         │
│    1000 │    ╱╲       ╱╲       ╱╲                     │
│         │   ╱  ╲     ╱  ╲     ╱  ╲    Normal daily   │
│     500 │──╱────╲───╱────╲───╱────╲── pattern        │
│         │ ╱      ╲ ╱      ╲ ╱      ╲                 │
│     200 │╱        ╲╱        ╲╱       ╲                │
│       0 │───────────────────────────── Time           │
│         Mon      Tue      Wed      Thu                │
│                                                       │
│  Fixed threshold at 300/sec:                          │
│    ✗ Alerts every night (false positive)              │
│    ✗ Misses a 50% drop during peak (false negative)   │
│                                                       │
│  Anomaly detection (learns the pattern):              │
│    ✓ No alert at night (expected low)                 │
│    ✓ Alerts on 50% peak-hour drop (unexpected)        │
└───────────────────────────────────────────────────────┘
```

### Alert Fatigue Prevention

Alert fatigue occurs when too many noisy or low-priority alerts desensitize on-call engineers.

**Prevention strategies:**

- **Alert on symptoms, not causes** — Alert on consumer lag (symptom), not CPU usage (cause), unless CPU is the actionable signal
- **Group related alerts** — A broker outage triggers dozens of downstream alerts; group them into one incident
- **Require duration** — Avoid alerting on momentary spikes; require the condition to persist (e.g., 5 min)
- **Escalation tiers** — Warning → Slack notification; Critical → PagerDuty page
- **Regular review** — Audit alerts monthly; delete alerts that never fire or always fire
- **Runbook links** — Every alert should link to a runbook with investigation steps

| Practice | Benefit |
|----------|---------|
| Severity levels (info/warn/critical) | Prioritize response effort |
| Alert deduplication | Prevent duplicate pages for same issue |
| Auto-resolve on recovery | Clear alerts when metrics return to normal |
| Maintenance windows | Suppress alerts during planned downtime |
| Alert-to-incident ratio | Track how many alerts create actionable incidents |

---

## Dashboards

Dashboards translate metrics into actionable visual context.

### Essential Dashboards

Every messaging monitoring setup should include these dashboards:

```
┌────────────────────────────────────────────────────────────┐
│                 Dashboard Hierarchy                         │
│                                                            │
│  Level 1: Executive Overview                               │
│  ┌──────────────────────────────────────────────────┐      │
│  │ Total messages/sec │ Overall error rate │ SLO %   │      │
│  └──────────────────────────────────────────────────┘      │
│                          │                                 │
│  Level 2: Service Dashboard                                │
│  ┌──────────────────────────────────────────────────┐      │
│  │ Per-topic throughput │ Consumer lag │ Latency P99 │      │
│  │ Error rates by type  │ DLQ depth   │ Active cons │      │
│  └──────────────────────────────────────────────────┘      │
│                          │                                 │
│  Level 3: Deep Dive                                        │
│  ┌──────────────────────────────────────────────────┐      │
│  │ Per-partition lag  │ Broker resource usage        │      │
│  │ Individual consumer metrics │ Network stats       │      │
│  │ JMX details │ Replication status │ GC metrics     │      │
│  └──────────────────────────────────────────────────┘      │
└────────────────────────────────────────────────────────────┘
```

### PromQL and Grafana Examples

**Consumer lag by consumer group:**

```
# Total lag per consumer group
sum by (consumergroup) (
  kafka_consumergroup_lag
)

# Lag rate of change (messages/sec falling behind)
rate(kafka_consumergroup_lag[5m])
```

**Throughput (messages/sec):**

```
# Messages produced per second per topic
sum by (topic) (
  rate(kafka_server_broker_topic_metrics_MessagesInPerSec[5m])
)

# Messages consumed per second per consumer group
sum by (consumergroup) (
  rate(kafka_consumer_fetch_manager_records_consumed_total[5m])
)
```

**End-to-end latency (histogram):**

```
# P99 end-to-end latency
histogram_quantile(0.99,
  sum by (le) (
    rate(messaging_e2e_latency_seconds_bucket[5m])
  )
)
```

**Under-replicated partitions:**

```
# Under-replicated partitions across all brokers
sum(kafka_server_replica_manager_UnderReplicatedPartitions)
```

### Dashboard Design Principles

- **Top-down layout** — Place the most important metrics (lag, error rate) at the top
- **Time range consistency** — Default to the same time range across all panels (e.g., last 6 hours)
- **Color coding** — Green for healthy, yellow for warning, red for critical; avoid using color as the only signal
- **Annotations** — Mark deployments, incidents, and configuration changes on the timeline
- **Variable templates** — Allow filtering by topic, consumer group, broker, or environment
- **Link to alerts** — Each dashboard panel should link to its corresponding alert rule

---

## Distributed Tracing

Tracing follows a request across services. In messaging systems, the challenge is propagating trace context through asynchronous message boundaries.

### Trace Context Propagation

```
┌──────────────────────────────────────────────────────────────┐
│           Trace Context Through Messages                      │
│                                                              │
│  Service A (Producer)                                        │
│  ┌──────────────────────────────────┐                        │
│  │ Span: "publish order.created"   │                        │
│  │ TraceID: abc-123                │                        │
│  │ SpanID:  span-001              │                        │
│  │                                 │                        │
│  │ Inject context into message     │                        │
│  │ headers:                        │                        │
│  │   traceparent: 00-abc123-001-01 │                        │
│  └──────────┬───────────────────────┘                        │
│             │                                                │
│             ▼  Message with trace headers                    │
│  ┌──────────────────────────────────┐                        │
│  │          Broker                 │                        │
│  │  Headers:                       │                        │
│  │    traceparent: 00-abc123-001-01│                        │
│  │  Body: { order data }           │                        │
│  └──────────┬───────────────────────┘                        │
│             │                                                │
│             ▼                                                │
│  Service B (Consumer)                                        │
│  ┌──────────────────────────────────┐                        │
│  │ Extract context from headers    │                        │
│  │ Span: "process order.created"   │                        │
│  │ TraceID: abc-123  (same trace!) │                        │
│  │ SpanID:  span-002              │                        │
│  │ ParentSpanID: span-001         │                        │
│  └──────────────────────────────────┘                        │
└──────────────────────────────────────────────────────────────┘
```

### Correlation IDs

Correlation IDs link related messages across topics and services when a single business operation produces multiple messages.

```
Order Processing Flow with Correlation ID:

  ┌──────────────┐    correlation_id: order-7890
  │ Order Service│───────────────────────────────────┐
  └──────┬───────┘                                   │
         │ Publishes: order.created                  │
         ▼                                           │
  ┌──────────────┐    correlation_id: order-7890     │
  │Payment Service│───────────────────────────────┐  │
  └──────┬───────┘                                │  │
         │ Publishes: payment.processed           │  │
         ▼                                        │  │
  ┌──────────────┐    correlation_id: order-7890  │  │
  │Shipping Svc  │                                │  │
  └──────────────┘                                │  │
                                                  │  │
  Query by correlation_id: order-7890             │  │
  → Returns all messages and traces for this order│  │
```

**Best practices for correlation IDs:**

- Generate at the entry point (first producer) and propagate through all downstream messages
- Store in message headers, not the payload
- Include in all log entries for the business operation
- Use UUIDs or structured IDs (e.g., `order-{uuid}`) for easy filtering

### OpenTelemetry Integration

OpenTelemetry provides standardized instrumentation for tracing across messaging systems.

```
┌──────────────────────────────────────────────────────────┐
│          OpenTelemetry Messaging Pipeline                 │
│                                                          │
│  Application Code                                        │
│  ┌─────────────────────────────────────────────┐         │
│  │  OTel SDK + Messaging Instrumentation       │         │
│  │                                             │         │
│  │  Producer: span kind = PRODUCER             │         │
│  │  Consumer: span kind = CONSUMER             │         │
│  │  Attributes:                                │         │
│  │    messaging.system = "kafka"               │         │
│  │    messaging.destination.name = "orders"    │         │
│  │    messaging.operation = "publish"          │         │
│  │    messaging.kafka.partition = 3            │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                        │
│                 ▼                                        │
│  ┌─────────────────────────────────────────────┐         │
│  │  OTel Collector                             │         │
│  │  - Receives spans from all services         │         │
│  │  - Batches and exports                      │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                        │
│            ┌────┴────┐                                   │
│            ▼         ▼                                   │
│       ┌────────┐ ┌────────┐                              │
│       │ Jaeger │ │ Tempo  │                              │
│       └────────┘ └────────┘                              │
└──────────────────────────────────────────────────────────┘
```

**Key OpenTelemetry semantic conventions for messaging:**

| Attribute | Description | Example |
|-----------|-------------|---------|
| `messaging.system` | Messaging system identifier | `kafka`, `rabbitmq`, `sqs` |
| `messaging.destination.name` | Topic or queue name | `order.events` |
| `messaging.operation` | Operation type | `publish`, `receive`, `process` |
| `messaging.message.id` | Unique message identifier | `msg-abc-123` |
| `messaging.consumer.group.name` | Consumer group | `order-processor` |
| `messaging.batch.message_count` | Messages in batch | `50` |

---

## Log Management

Logs complement metrics and traces by providing detailed context for debugging specific message processing issues.

### Structured Logging for Messaging

Use structured (JSON) logging with consistent fields across all producers and consumers.

```
// Producer log entry
{
  "timestamp": "2025-01-15T10:30:45.123Z",
  "level": "INFO",
  "service": "order-service",
  "action": "message_published",
  "topic": "order.created",
  "partition": 3,
  "offset": 15234,
  "message_id": "msg-abc-123",
  "correlation_id": "order-7890",
  "trace_id": "abc123def456",
  "latency_ms": 12
}

// Consumer log entry
{
  "timestamp": "2025-01-15T10:30:45.456Z",
  "level": "INFO",
  "service": "payment-service",
  "action": "message_processed",
  "topic": "order.created",
  "partition": 3,
  "offset": 15234,
  "message_id": "msg-abc-123",
  "correlation_id": "order-7890",
  "trace_id": "abc123def456",
  "processing_time_ms": 85,
  "result": "success"
}
```

**Standard fields to include:**

| Field | Purpose |
|-------|---------|
| `message_id` | Unique message identifier for deduplication tracking |
| `correlation_id` | Links related messages in a business flow |
| `trace_id` | Connects to distributed trace |
| `topic` / `queue` | Message destination |
| `partition` / `offset` | Exact position in the log (Kafka) |
| `consumer_group` | Which consumer group processed the message |
| `action` | What happened (`published`, `received`, `processed`, `failed`) |
| `latency_ms` | Processing or publish duration |

### Log Correlation

Correlating logs across producers, brokers, and consumers is essential for debugging message flow issues.

```
┌───────────────────────────────────────────────────────────┐
│               Log Correlation Flow                         │
│                                                           │
│  Search by correlation_id = "order-7890":                 │
│                                                           │
│  10:30:45.100  order-service     message_published        │
│                topic=order.created offset=15234            │
│                                                           │
│  10:30:45.200  kafka-broker      message_replicated       │
│                topic=order.created partition=3             │
│                                                           │
│  10:30:45.350  payment-service   message_received         │
│                topic=order.created offset=15234            │
│                                                           │
│  10:30:45.456  payment-service   message_processed        │
│                result=success processing_time_ms=85       │
│                                                           │
│  Complete timeline for a single business operation        │
│  reconstructed from logs across three services.           │
└───────────────────────────────────────────────────────────┘
```

### Centralized Logging

Centralize logs from all messaging components into a single searchable store.

```
┌────────────────────────────────────────────────────────────┐
│            Centralized Logging Architecture                 │
│                                                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│  │ Producer │  │  Broker  │  │ Consumer │                 │
│  │  Logs    │  │  Logs    │  │  Logs    │                 │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                 │
│       │              │              │                      │
│       ▼              ▼              ▼                      │
│  ┌──────────────────────────────────────────┐              │
│  │         Log Shipper / Agent              │              │
│  │   (Fluentd, Filebeat, Vector, OTel)     │              │
│  └─────────────────┬────────────────────────┘              │
│                    │                                       │
│                    ▼                                       │
│  ┌──────────────────────────────────────────┐              │
│  │       Log Aggregation Platform           │              │
│  │   (Elasticsearch, Loki, CloudWatch)     │              │
│  └─────────────────┬────────────────────────┘              │
│                    │                                       │
│                    ▼                                       │
│  ┌──────────────────────────────────────────┐              │
│  │          Kibana / Grafana                │              │
│  │   Search, filter, visualize logs        │              │
│  └──────────────────────────────────────────┘              │
└────────────────────────────────────────────────────────────┘
```

**Log retention guidelines:**

| Log Type | Retention | Reason |
|----------|-----------|--------|
| Application logs (info) | 7–14 days | Debugging recent issues |
| Error logs | 30–90 days | Post-incident analysis |
| Audit logs | 1–7 years | Compliance requirements |
| Broker logs | 7–30 days | Infrastructure debugging |

---

## Health Checks

Health checks provide a binary signal: is this component healthy or not? They feed into load balancers, orchestrators, and alerting systems.

### Broker Health

```
┌───────────────────────────────────────────────────────────┐
│                  Broker Health Checks                      │
│                                                           │
│  Check                    Method              Pass When   │
│  ─────────────────────    ─────────────────   ──────────  │
│  Broker reachable         TCP connect :9092   Connection  │
│                                               established │
│                                                           │
│  Cluster metadata         Metadata request    Returns     │
│                                               topic list  │
│                                                           │
│  Controller available     Describe cluster    Exactly 1   │
│                                               controller  │
│                                                           │
│  Under-replicated         JMX / metrics       Count = 0   │
│  partitions                                               │
│                                                           │
│  Disk space               OS check            > 20% free  │
│                                                           │
│  ISR status               Describe topics     All         │
│                                               partitions  │
│                                               in-sync     │
└───────────────────────────────────────────────────────────┘
```

### Consumer Health

Consumer health checks verify that consumers are running, connected, and making progress.

| Check | How to Verify | Unhealthy Signal |
|-------|--------------|-----------------|
| Consumer running | Process/container status | Process not found or crash loop |
| Group membership | Describe consumer group | Consumer not in group |
| Partition assignment | Consumer group describe | Zero assigned partitions |
| Lag trend | Compare lag over time | Lag increasing continuously |
| Last commit time | Track offset commit timestamps | No commits in > 5 min |
| Heartbeat | Consumer heartbeat interval | Missed heartbeats |

```
┌───────────────────────────────────────────────────────┐
│            Consumer Health State Machine               │
│                                                       │
│  ┌──────────┐   Connected    ┌──────────┐             │
│  │  STARTING│ ─────────────▶│  HEALTHY │             │
│  └──────────┘               └─────┬────┘             │
│       ▲                           │                  │
│       │ Restart                   │ Lag increasing   │
│       │                           │ or errors > 0    │
│  ┌──────────┐                     ▼                  │
│  │   DEAD   │◀───────────── ┌──────────┐             │
│  │          │  No heartbeat │ DEGRADED │             │
│  └──────────┘  for > 30s    └──────────┘             │
│                                                       │
│  HEALTHY:  Lag stable, errors = 0, heartbeat OK      │
│  DEGRADED: Lag growing or intermittent errors        │
│  DEAD:     No heartbeat, process crashed             │
└───────────────────────────────────────────────────────┘
```

### Connectivity Checks

Verify connectivity between all components in the messaging pipeline.

```
Connectivity Check Matrix:

  From / To       Broker    Schema Registry    Database    Downstream API
  ──────────      ──────    ───────────────    ────────    ──────────────
  Producer        ✓ :9092   ✓ :8081            -           -
  Consumer        ✓ :9092   ✓ :8081            ✓ :5432     ✓ :443
  Monitoring      ✓ :9092   -                  -           -
                  ✓ :7071
                  (JMX)

  Each ✓ should be a periodic health check (every 10–30s)
  Alert if any check fails for > 1 minute
```

---

## Capacity Planning

Capacity planning uses monitoring data to predict when you need to scale and by how much.

### Throughput Estimation

```
┌───────────────────────────────────────────────────────────┐
│              Throughput Capacity Planning                   │
│                                                           │
│  Current State:                                           │
│  - Peak throughput:       50,000 msgs/sec                 │
│  - Average message size:  1 KB                            │
│  - Peak bandwidth:        50 MB/sec                       │
│  - Broker count:          3                               │
│  - Per-broker capacity:   25,000 msgs/sec                 │
│  - Current utilization:   67% at peak                     │
│                                                           │
│  Growth Projection:                                       │
│  - Monthly growth rate:   15%                             │
│  - Months to 80% util:   ~2 months                       │
│  - Months to 100% util:  ~4 months                       │
│                                                           │
│  Throughput Over Time:                                    │
│                                                           │
│  msgs/sec                                                 │
│   75K │                              ╱  ← 100% capacity  │
│       │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ╱─ ─                    │
│   60K │                       ╱      ← 80% (scale here)  │
│       │─ ─ ─ ─ ─ ─ ─ ─ ─ ╱─ ─ ─                        │
│   50K │ ──────────────── ╱  ← Current peak                │
│       │                ╱                                  │
│   25K │              ╱                                    │
│       └──────────────────────────── Time                  │
│       Now    +1mo   +2mo   +3mo   +4mo                   │
│                                                           │
│  Action: Add broker at +2 months (before 80%)             │
└───────────────────────────────────────────────────────────┘
```

### Storage Planning

| Factor | Calculation |
|--------|------------|
| Daily ingest | Messages/day × avg message size |
| Retention storage | Daily ingest × retention days |
| Replication overhead | Retention storage × replication factor |
| Total disk needed | Replication overhead + 20% headroom |
| Growth buffer | Add projected growth for next 6 months |

```
Example Calculation:

  Messages/day:       100,000,000 (100M)
  Avg message size:   1 KB
  Daily ingest:       100 GB
  Retention:          7 days
  Retention storage:  700 GB
  Replication factor: 3
  Replicated storage: 2,100 GB (2.1 TB)
  Headroom (20%):     420 GB
  Total needed:       2,520 GB (~2.5 TB)

  With 3 brokers:     ~840 GB per broker
  Recommended disk:   1 TB per broker (allows growth)
```

### Scaling Triggers

Define clear, metrics-based triggers for scaling decisions.

| Trigger | Metric | Threshold | Action |
|---------|--------|-----------|--------|
| CPU saturation | Broker CPU usage | > 70% sustained | Add brokers |
| Disk pressure | Disk usage | > 75% | Add storage or reduce retention |
| Consumer lag | Consumer lag (messages) | Growing for > 30 min | Scale consumers |
| Network saturation | Network bytes in/out | > 80% NIC capacity | Upgrade NIC or add brokers |
| Partition hotspot | Per-partition throughput | Single partition > 50% of broker capacity | Increase partitions |
| Memory pressure | JVM heap usage | > 85% | Increase heap or add brokers |

```
┌───────────────────────────────────────────────────────┐
│            Scaling Decision Flowchart                   │
│                                                       │
│  Consumer lag increasing?                             │
│       │                                               │
│       ├── Yes ─▶ Are consumers at max CPU?            │
│       │              │                                │
│       │              ├── Yes ─▶ Scale consumer         │
│       │              │          instances              │
│       │              │                                │
│       │              └── No ──▶ Check downstream      │
│       │                         dependencies          │
│       │                         (DB, API, network)    │
│       │                                               │
│       └── No ──▶ Broker under pressure?               │
│                      │                                │
│                      ├── CPU > 70% ──▶ Add broker     │
│                      │                                │
│                      ├── Disk > 75% ─▶ Add storage    │
│                      │                 or reduce      │
│                      │                 retention      │
│                      │                                │
│                      └── Network > 80% ─▶ Add broker  │
│                                           or upgrade  │
│                                           network     │
└───────────────────────────────────────────────────────┘
```

---

## Next Steps

Continue your messaging learning journey:

| File | Topic | Description |
|------|-------|-------------|
| [00-OVERVIEW.md](00-OVERVIEW.md) | Messaging Overview | Foundational concepts, patterns, and broker comparison |
| [01-APACHE-KAFKA.md](01-APACHE-KAFKA.md) | Apache Kafka | Brokers, topics, partitions, producers, consumers |
| [02-RABBITMQ.md](02-RABBITMQ.md) | RabbitMQ | AMQP model, exchanges, queues, reliability |
| [03-CLOUD-SERVICES.md](03-CLOUD-SERVICES.md) | Cloud Services | SQS, SNS, EventBridge, Kinesis, Service Bus |
| [04-PATTERNS.md](04-PATTERNS.md) | Messaging Patterns | Pub/sub, request-reply, event sourcing, CQRS |
| [05-SCHEMA-MANAGEMENT.md](05-SCHEMA-MANAGEMENT.md) | Schema Management | Schema registries, evolution, compatibility |
| [06-RELIABILITY.md](06-RELIABILITY.md) | Reliability | Delivery guarantees, idempotency, retries, DLQs |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial messaging monitoring documentation |
