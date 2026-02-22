# Cloud Messaging Services

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [AWS SQS](#aws-sqs)
   - [Standard Queues](#standard-queues)
   - [FIFO Queues](#fifo-queues)
   - [Visibility Timeout](#visibility-timeout)
   - [Dead Letter Queues](#dead-letter-queues)
   - [SQS Configuration](#sqs-configuration)
3. [AWS SNS](#aws-sns)
   - [Topics](#topics)
   - [Subscriptions](#subscriptions)
   - [Fan-Out Pattern](#fan-out-pattern)
   - [Message Filtering](#message-filtering)
   - [SNS and SQS Integration](#sns-and-sqs-integration)
4. [AWS EventBridge](#aws-eventbridge)
   - [Event Buses](#event-buses)
   - [Rules](#rules)
   - [Schemas](#schemas)
   - [Cross-Account Events](#cross-account-events)
5. [Azure Service Bus](#azure-service-bus)
   - [Queues](#queues)
   - [Topics and Subscriptions](#topics-and-subscriptions)
   - [Sessions](#sessions)
   - [Message Deferral](#message-deferral)
   - [Duplicate Detection](#duplicate-detection)
6. [Azure Event Hubs](#azure-event-hubs)
   - [Partitions](#partitions)
   - [Consumer Groups](#consumer-groups)
   - [Capture](#capture)
   - [Kafka Compatibility](#kafka-compatibility)
7. [Google Cloud Pub/Sub](#google-cloud-pubsub)
   - [Pub/Sub Topics](#pubsub-topics)
   - [Pub/Sub Subscriptions](#pubsub-subscriptions)
   - [Exactly-Once Delivery](#exactly-once-delivery)
   - [Ordering Keys](#ordering-keys)
   - [Dead Letter Topics](#dead-letter-topics)
8. [Service Comparison](#service-comparison)
   - [Feature Comparison](#feature-comparison)
   - [Delivery Guarantees](#delivery-guarantees)
   - [Throughput and Latency](#throughput-and-latency)
9. [Multi-Cloud Messaging](#multi-cloud-messaging)
   - [Abstraction Patterns](#abstraction-patterns)
   - [Portability Considerations](#portability-considerations)
10. [Cost Optimization](#cost-optimization)
    - [Pricing Models](#pricing-models)
    - [Batching for Cost Reduction](#batching-for-cost-reduction)
    - [Reserved Capacity](#reserved-capacity)
11. [Migration Strategies](#migration-strategies)
    - [Moving Between Brokers](#moving-between-brokers)
    - [Hybrid Approaches](#hybrid-approaches)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

Cloud messaging services provide fully managed infrastructure for asynchronous communication between distributed applications. Unlike self-hosted brokers such as Apache Kafka or RabbitMQ, cloud-managed services eliminate operational overhead — patching, scaling, replication, and monitoring are handled by the cloud provider. Each major cloud platform (AWS, Azure, Google Cloud) offers a family of messaging services tailored to different use cases: simple queues, publish-subscribe fan-out, event routing, and high-throughput streaming.

Choosing the right service depends on delivery guarantees, ordering requirements, throughput needs, integration with existing cloud infrastructure, and cost constraints. This guide provides a comprehensive comparison of the major cloud messaging offerings, practical configuration guidance, and strategies for multi-cloud portability and cost optimization.

```
Cloud Messaging Landscape
─────────────────────────

  ┌──────────────────────────────────────────────────────────────┐
  │                     Cloud Messaging Services                  │
  ├──────────────────┬───────────────────┬───────────────────────┤
  │       AWS        │      Azure        │    Google Cloud        │
  ├──────────────────┼───────────────────┼───────────────────────┤
  │  SQS (Queues)    │ Service Bus       │ Pub/Sub               │
  │  SNS (Pub/Sub)   │  (Queues+Topics)  │  (Topics+Subs)        │
  │  EventBridge     │ Event Hubs        │ Eventarc              │
  │   (Event Router) │  (Streaming)      │  (Event Router)       │
  │  Kinesis         │ Event Grid        │ Cloud Tasks           │
  │   (Streaming)    │  (Event Router)   │  (Task Queues)        │
  └──────────────────┴───────────────────┴───────────────────────┘
```

### Target Audience

- **Cloud Engineers** deploying and managing messaging infrastructure on AWS, Azure, or Google Cloud
- **Backend Developers** building event-driven microservices using cloud-native messaging services
- **Architects** evaluating and comparing messaging solutions across cloud providers
- **DevOps Engineers** automating provisioning, monitoring, and cost management for messaging workloads
- **Platform Engineers** designing multi-cloud or hybrid messaging strategies

### Scope

- AWS SQS: standard queues, FIFO queues, visibility timeout, dead letter queues
- AWS SNS: topics, subscriptions, fan-out, message filtering, SNS+SQS integration
- AWS EventBridge: event buses, rules, schemas, cross-account event routing
- Azure Service Bus: queues, topics, sessions, deferral, duplicate detection
- Azure Event Hubs: partitions, consumer groups, capture, Kafka protocol support
- Google Cloud Pub/Sub: topics, subscriptions, exactly-once delivery, ordering keys
- Cross-provider feature comparison and selection guidance
- Multi-cloud abstraction patterns and portability
- Cost optimization strategies and migration approaches

---

## AWS SQS

Amazon Simple Queue Service (SQS) is a fully managed message queuing service that decouples producers from consumers. SQS supports two queue types — **Standard** and **FIFO** — each with different guarantees around ordering and deduplication.

```
SQS Message Flow
─────────────────

  Producer(s)                          Consumer(s)
  ┌──────┐       ┌──────────────┐       ┌──────┐
  │ App  │──────▶│              │──────▶│ App  │
  └──────┘       │   SQS Queue  │       └──────┘
  ┌──────┐       │              │       ┌──────┐
  │ App  │──────▶│  (Standard   │──────▶│ App  │
  └──────┘       │   or FIFO)   │       └──────┘
                 └──────┬───────┘
                        │ (failed messages)
                        ▼
                 ┌──────────────┐
                 │  Dead Letter  │
                 │    Queue      │
                 └──────────────┘
```

### Standard Queues

Standard queues offer **best-effort ordering** and **at-least-once delivery**. They are designed for maximum throughput — nearly unlimited transactions per second.

**Key characteristics:**

- Messages may be delivered out of order
- Messages may be delivered more than once (consumers must be idempotent)
- Nearly unlimited throughput with no pre-provisioning
- Supports up to 256 KB per message (or up to 2 GB via Extended Client Library with S3)

```bash
# Create a standard queue
aws sqs create-queue \
  --queue-name my-standard-queue \
  --attributes '{
    "MessageRetentionPeriod": "345600",
    "VisibilityTimeout": "30",
    "ReceiveMessageWaitTimeSeconds": "20"
  }'

# Send a message
aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789/my-standard-queue \
  --message-body '{"orderId": "12345", "action": "process"}'

# Receive messages (long polling)
aws sqs receive-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789/my-standard-queue \
  --max-number-of-messages 10 \
  --wait-time-seconds 20
```

### FIFO Queues

FIFO (First-In-First-Out) queues guarantee **exactly-once processing** and **strict ordering** within a message group. They are ideal when the order of operations matters.

**Key characteristics:**

- Exactly-once processing (deduplication within a 5-minute window)
- Strict ordering within each **Message Group ID**
- Throughput: 300 messages/second without batching, 3,000 with batching (high-throughput mode supports higher)
- Queue name must end with `.fifo`

```bash
# Create a FIFO queue
aws sqs create-queue \
  --queue-name my-queue.fifo \
  --attributes '{
    "FifoQueue": "true",
    "ContentBasedDeduplication": "true"
  }'

# Send with message group ID (ordering key)
aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789/my-queue.fifo \
  --message-body '{"orderId": "12345", "status": "shipped"}' \
  --message-group-id "order-12345" \
  --message-deduplication-id "ship-12345-001"
```

| Feature | Standard Queue | FIFO Queue |
|---------|---------------|------------|
| Ordering | Best-effort | Strict (per group) |
| Delivery | At-least-once | Exactly-once |
| Throughput | Unlimited | 300-3,000+ msg/s |
| Deduplication | None | 5-minute window |
| Queue Name | Any | Must end in `.fifo` |

### Visibility Timeout

When a consumer receives a message from SQS, the message becomes **invisible** to other consumers for the duration of the **visibility timeout**. This prevents duplicate processing.

```
Visibility Timeout Lifecycle
────────────────────────────

  Time ──────────────────────────────────────────────▶

  │ Message    │ Consumer     │ Visibility     │ Message
  │ arrives    │ receives     │ timeout        │ visible
  │ in queue   │ message      │ expires        │ again
  ▼            ▼              ▼                ▼
  ├────────────┤──────────────┤────────────────┤
  │  Visible   │  Invisible   │  Visible again │
  │            │  (processing)│  (if not       │
  │            │              │   deleted)     │
  └────────────┴──────────────┴────────────────┘

  Default: 30 seconds    Range: 0 seconds to 12 hours
```

**Best practices:**

- Set the visibility timeout to at least 6x the average processing time
- Use `ChangeMessageVisibility` to extend the timeout for long-running tasks
- Monitor `ApproximateAgeOfOldestMessage` to detect stuck consumers

### Dead Letter Queues

A **dead letter queue (DLQ)** captures messages that fail processing after a configured number of attempts. This prevents poison messages from blocking the queue.

```bash
# Create a DLQ
aws sqs create-queue --queue-name my-dlq

# Configure source queue to use DLQ (maxReceiveCount = 3)
aws sqs set-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789/my-queue \
  --attributes '{
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:us-east-1:123456789:my-dlq\",\"maxReceiveCount\":\"3\"}"
  }'

# Redrive messages from DLQ back to source queue
aws sqs start-message-move-task \
  --source-arn arn:aws:sqs:us-east-1:123456789:my-dlq \
  --destination-arn arn:aws:sqs:us-east-1:123456789:my-queue
```

### SQS Configuration

| Parameter | Default | Recommended | Description |
|-----------|---------|-------------|-------------|
| `VisibilityTimeout` | 30s | 6x processing time | Hide duration after receive |
| `MessageRetentionPeriod` | 4 days | 7-14 days | Max time message stays in queue |
| `ReceiveMessageWaitTimeSeconds` | 0 (short poll) | 20 (long poll) | Long polling reduces empty responses |
| `MaximumMessageSize` | 256 KB | 256 KB | Use S3 for larger payloads |
| `DelaySeconds` | 0 | Per use case | Delay before message becomes visible |

> **Tip:** Always enable **long polling** (`ReceiveMessageWaitTimeSeconds: 20`) to reduce costs and empty responses.

---

## AWS SNS

Amazon Simple Notification Service (SNS) is a fully managed pub/sub messaging service. Publishers send messages to **topics**, and SNS delivers them to all **subscribed endpoints** — SQS queues, Lambda functions, HTTP endpoints, email, SMS, and mobile push.

### Topics

An SNS **topic** is a logical channel for publishing messages. Topics can be **Standard** (high throughput, best-effort ordering) or **FIFO** (strict ordering, deduplication).

```bash
# Create a standard topic
aws sns create-topic --name order-events

# Create a FIFO topic
aws sns create-topic \
  --name order-events.fifo \
  --attributes '{
    "FifoTopic": "true",
    "ContentBasedDeduplication": "true"
  }'
```

### Subscriptions

Subscribers register to receive messages from a topic. Each subscription specifies a **protocol** and an **endpoint**.

| Protocol | Endpoint Example | Use Case |
|----------|-----------------|----------|
| `sqs` | SQS queue ARN | Async processing |
| `lambda` | Lambda function ARN | Serverless compute |
| `https` | Webhook URL | External integrations |
| `email` | user@example.com | Notifications |
| `sms` | +1234567890 | Alerts |
| `firehose` | Firehose stream ARN | Data archival |

### Fan-Out Pattern

The **fan-out pattern** uses SNS to distribute a single message to multiple SQS queues simultaneously. Each queue processes the message independently.

```
SNS Fan-Out Pattern
───────────────────

                    ┌──────────────────┐
                    │   SNS Topic      │
                    │  (order-events)  │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
     ┌────────────┐  ┌────────────┐  ┌────────────┐
     │ SQS Queue  │  │ SQS Queue  │  │ SQS Queue  │
     │ (billing)  │  │ (shipping) │  │ (analytics) │
     └─────┬──────┘  └─────┬──────┘  └─────┬──────┘
           ▼               ▼               ▼
     ┌──────────┐    ┌──────────┐    ┌──────────┐
     │ Billing  │    │ Shipping │    │Analytics │
     │ Service  │    │ Service  │    │ Service  │
     └──────────┘    └──────────┘    └──────────┘

  One message published ──▶ Three services process independently
```

### Message Filtering

SNS **subscription filter policies** allow subscribers to receive only a subset of messages, reducing downstream processing.

```json
{
  "eventType": ["order.placed", "order.cancelled"],
  "region": [{"prefix": "us-"}],
  "amount": [{"numeric": [">=", 100]}]
}
```

Filter policies support:

- **Exact match**: `["value1", "value2"]`
- **Prefix match**: `[{"prefix": "us-"}]`
- **Numeric match**: `[{"numeric": [">=", 100, "<=", 1000]}]`
- **Exists/not-exists**: `[{"exists": true}]`
- **Anything-but**: `[{"anything-but": ["internal"]}]`

### SNS and SQS Integration

The SNS+SQS combination is the most common messaging pattern in AWS. SNS handles fan-out and filtering, while SQS provides buffering, retry, and independent consumer scaling.

```bash
# Subscribe an SQS queue to an SNS topic
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789:order-events \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:123456789:billing-queue \
  --attributes '{
    "FilterPolicy": "{\"eventType\": [\"order.placed\"]}"
  }'
```

> **Important:** The SQS queue must have an **access policy** that allows the SNS topic to send messages to it.

---

## AWS EventBridge

Amazon EventBridge is a serverless event bus that connects applications using events. Unlike SNS (which focuses on message delivery), EventBridge provides **content-based routing**, **schema discovery**, and native integration with over 200 AWS services and SaaS applications.

### Event Buses

An **event bus** receives events from sources and routes them to targets based on rules.

```
EventBridge Event Flow
──────────────────────

  Event Sources                Event Bus               Targets
  ┌───────────┐          ┌──────────────────┐     ┌───────────┐
  │ AWS       │─────────▶│                  │────▶│ Lambda    │
  │ Services  │          │  ┌────────────┐  │     └───────────┘
  └───────────┘          │  │   Rule 1   │  │     ┌───────────┐
  ┌───────────┐          │  │ (pattern)  │──┼────▶│ SQS       │
  │ Custom    │─────────▶│  └────────────┘  │     └───────────┘
  │ Apps      │          │  ┌────────────┐  │     ┌───────────┐
  └───────────┘          │  │   Rule 2   │──┼────▶│ Step      │
  ┌───────────┐          │  │ (pattern)  │  │     │ Functions │
  │ SaaS      │─────────▶│  └────────────┘  │     └───────────┘
  │ Partners  │          │                  │     ┌───────────┐
  └───────────┘          └──────────────────┘     │ API       │
                                                  │ Destination│
                                                  └───────────┘
```

**Event bus types:**

- **Default bus**: Receives events from AWS services automatically
- **Custom bus**: For application-specific events
- **Partner bus**: For SaaS partner integrations (Datadog, Zendesk, etc.)

### Rules

Rules match incoming events against **event patterns** and route matching events to one or more targets.

```json
{
  "source": ["com.myapp.orders"],
  "detail-type": ["Order Placed"],
  "detail": {
    "amount": [{"numeric": [">", 100]}],
    "region": ["us-east-1", "us-west-2"]
  }
}
```

Each rule can have up to **5 targets**, and each target can receive a **transformed** version of the event using input transformers.

### Schemas

EventBridge includes a **Schema Registry** that automatically discovers event schemas from events flowing through the bus. You can also define schemas manually.

**Benefits:**

- Auto-generated code bindings (Java, Python, TypeScript)
- Schema versioning and change detection
- Schema discovery from live event traffic

### Cross-Account Events

EventBridge supports routing events between AWS accounts and regions, enabling centralized event collection in multi-account architectures.

```
Cross-Account Event Routing
────────────────────────────

  Account A                          Account B
  ┌──────────────────┐              ┌──────────────────┐
  │  Custom Bus      │              │  Custom Bus      │
  │  ┌────────────┐  │   PutEvents  │  ┌────────────┐  │
  │  │  Rule      │──┼─────────────▶│  │  Rule      │  │
  │  │ (forward)  │  │              │  │ (process)  │  │
  │  └────────────┘  │              │  └─────┬──────┘  │
  └──────────────────┘              └────────┼─────────┘
                                             ▼
                                       ┌───────────┐
                                       │  Lambda   │
                                       └───────────┘
```

**Setup requires:**

1. A rule on Account A's bus targeting Account B's bus
2. A resource policy on Account B's bus allowing Account A

---

## Azure Service Bus

Azure Service Bus is a fully managed enterprise message broker with support for **queues** (point-to-point) and **topics** (publish-subscribe). It provides advanced features including sessions, message deferral, scheduled delivery, transactions, and duplicate detection.

```
Azure Service Bus Architecture
───────────────────────────────

  ┌───────────────────────────────────────────────────┐
  │               Service Bus Namespace                │
  │                                                    │
  │  ┌──────────────┐       ┌───────────────────────┐ │
  │  │    Queue     │       │       Topic           │ │
  │  │              │       │  ┌─────────────────┐  │ │
  │  │  Point-to-   │       │  │ Subscription A  │  │ │
  │  │  Point       │       │  ├─────────────────┤  │ │
  │  │              │       │  │ Subscription B  │  │ │
  │  │              │       │  ├─────────────────┤  │ │
  │  │              │       │  │ Subscription C  │  │ │
  │  └──────────────┘       │  └─────────────────┘  │ │
  │                         └───────────────────────┘ │
  └───────────────────────────────────────────────────┘
```

### Queues

Service Bus queues provide **point-to-point** communication with guaranteed FIFO ordering and at-least-once or at-most-once delivery.

**Key features:**

- Message size up to 256 KB (Standard tier) or 100 MB (Premium tier)
- Message TTL from 1 second to unlimited
- Lock-based consumption: messages are locked to a consumer during processing
- Sessions for ordered, grouped processing
- Scheduled delivery and message deferral
- Auto-forwarding between queues

```bash
# Create a Service Bus queue (Azure CLI)
az servicebus queue create \
  --resource-group my-rg \
  --namespace-name my-namespace \
  --name my-queue \
  --max-size 5120 \
  --default-message-time-to-live P14D \
  --lock-duration PT1M \
  --dead-lettering-on-message-expiration true \
  --enable-duplicate-detection true \
  --duplicate-detection-history-time-window PT10M
```

### Topics and Subscriptions

Topics enable **publish-subscribe** messaging. Publishers send messages to a topic, and each subscription receives a copy. Subscriptions can have **SQL-like filter rules** to receive only relevant messages.

```bash
# Create a topic with subscriptions
az servicebus topic create \
  --resource-group my-rg \
  --namespace-name my-namespace \
  --name order-events \
  --max-size 5120

# Create a subscription with a filter
az servicebus topic subscription create \
  --resource-group my-rg \
  --namespace-name my-namespace \
  --topic-name order-events \
  --name high-value-orders

# Add a SQL filter rule
az servicebus topic subscription rule create \
  --resource-group my-rg \
  --namespace-name my-namespace \
  --topic-name order-events \
  --subscription-name high-value-orders \
  --name amount-filter \
  --filter-sql-expression "amount > 1000 AND region = 'US'"
```

### Sessions

**Sessions** provide ordered, grouped message processing. All messages with the same **session ID** are guaranteed to be processed in order by a single consumer.

```
Session-Based Processing
────────────────────────

  Messages arrive with Session IDs:

  ┌────────────────────────────────────────────┐
  │  Queue with Sessions Enabled               │
  │                                            │
  │  Session "order-A":  [msg1] [msg2] [msg3]  │──▶ Consumer 1
  │  Session "order-B":  [msg1] [msg2]         │──▶ Consumer 2
  │  Session "order-C":  [msg1] [msg2] [msg3]  │──▶ Consumer 3
  │                                            │
  └────────────────────────────────────────────┘

  Each session is locked to one consumer at a time.
  Messages within a session are delivered in FIFO order.
```

**Use cases:**

- Order processing (all events for an order handled sequentially)
- User activity streams (actions per user processed in order)
- Workflow steps (sequential processing of a multi-step workflow)

### Message Deferral

**Deferral** allows a consumer to set a message aside and retrieve it later by its sequence number. This is useful when messages arrive out of the expected processing order.

**Deferral workflow:**

1. Consumer receives message but cannot process it yet
2. Consumer defers the message (it remains in the queue but is no longer delivered normally)
3. Consumer later retrieves the deferred message by its sequence number
4. Consumer processes and completes the deferred message

### Duplicate Detection

Service Bus can automatically detect and discard duplicate messages within a configurable **time window** (up to 7 days). Deduplication is based on the application-set `MessageId` property.

| Feature | Standard Tier | Premium Tier |
|---------|--------------|--------------|
| Max message size | 256 KB | 100 MB |
| Transactions | Yes | Yes |
| Sessions | Yes | Yes |
| Duplicate detection | Yes | Yes |
| Auto-forwarding | Yes | Yes |
| Geo-disaster recovery | No | Yes |
| VNET integration | No | Yes |
| Dedicated capacity | Shared | Dedicated |

---

## Azure Event Hubs

Azure Event Hubs is a big data streaming platform and event ingestion service. It is designed for **high-throughput** scenarios — millions of events per second — and provides features like partitioning, consumer groups, and long-term data capture.

```
Event Hubs Architecture
───────────────────────

  Producers                  Event Hub                 Consumers
  ┌──────┐              ┌──────────────────┐
  │ App  │─────────────▶│  ┌────────────┐  │       Consumer Group 1
  └──────┘              │  │ Partition 0 │──┼──────▶ [Consumer A]
  ┌──────┐              │  ├────────────┤  │
  │ App  │─────────────▶│  │ Partition 1 │──┼──────▶ [Consumer B]
  └──────┘              │  ├────────────┤  │
  ┌──────┐              │  │ Partition 2 │──┼──────▶ [Consumer C]
  │ IoT  │─────────────▶│  ├────────────┤  │
  │Device│              │  │ Partition 3 │──┼──────▶ [Consumer D]
  └──────┘              │  └────────────┘  │
                        └────────┬─────────┘
                                 │ (Capture)
                                 ▼
                        ┌──────────────────┐
                        │  Azure Storage   │
                        │  / Data Lake     │
                        └──────────────────┘
```

### Partitions

Partitions are the **unit of parallelism** in Event Hubs. Each partition is an ordered, immutable sequence of events. Partition count is set at creation time and determines maximum consumer parallelism.

**Partition guidance:**

- Minimum: 1 partition
- Standard tier: up to 32 partitions
- Dedicated tier: up to 2,048 partitions
- Use a **partition key** to route related events to the same partition for ordering
- Events without a partition key are distributed round-robin

### Consumer Groups

A **consumer group** is a view of the entire Event Hub. Each consumer group tracks its own offsets, enabling multiple applications to read the same stream independently.

- Default consumer group: `$Default`
- Standard tier: up to 20 consumer groups
- Premium/Dedicated: up to 100 consumer groups
- Each partition within a consumer group can have at most one active consumer (for the Event Processor pattern)

### Capture

**Event Hubs Capture** automatically streams events to Azure Blob Storage or Azure Data Lake Storage in Avro format. This enables long-term retention and batch analytics without additional code.

```bash
# Enable capture on an Event Hub
az eventhubs eventhub update \
  --resource-group my-rg \
  --namespace-name my-namespace \
  --name my-eventhub \
  --enable-capture true \
  --capture-interval 300 \
  --capture-size-limit 314572800 \
  --destination-name EventHubArchive.AzureBlockBlob \
  --storage-account /subscriptions/.../storageAccounts/mystorage \
  --blob-container capture-data
```

### Kafka Compatibility

Event Hubs provides a **Kafka-compatible endpoint** that allows existing Kafka clients (producer, consumer, Kafka Connect, Kafka Streams) to work with Event Hubs without code changes. This enables migration from self-hosted Kafka to a managed service.

| Kafka Concept | Event Hubs Equivalent |
|---------------|----------------------|
| Cluster | Namespace |
| Topic | Event Hub |
| Partition | Partition |
| Consumer Group | Consumer Group |
| Offset | Offset |
| Broker | Endpoint (no direct equivalent) |

**Connection configuration for Kafka clients:**

```properties
bootstrap.servers=my-namespace.servicebus.windows.net:9093
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="$ConnectionString" \
  password="Endpoint=sb://my-namespace.servicebus.windows.net/;SharedAccessKeyName=...";
```

---

## Google Cloud Pub/Sub

Google Cloud Pub/Sub is a fully managed, real-time messaging service that enables asynchronous communication between independent applications. It provides **global** message routing, automatic scaling, and configurable delivery semantics.

```
Cloud Pub/Sub Architecture
──────────────────────────

  Publishers              Topic             Subscribers
  ┌──────┐          ┌──────────────┐
  │ App  │─────────▶│              │     Subscription A (Pull)
  └──────┘          │   Topic      │────▶ [Subscriber 1]
  ┌──────┐          │  (order-     │────▶ [Subscriber 2]
  │ App  │─────────▶│   events)    │
  └──────┘          │              │     Subscription B (Push)
  ┌──────┐          │              │────▶ https://endpoint.example.com
  │ App  │─────────▶│              │
  └──────┘          └──────────────┘     Subscription C (BigQuery)
                                    ────▶ [BigQuery Table]
```

### Pub/Sub Topics

A **topic** is a named resource to which publishers send messages. Topics are global within a Google Cloud project.

```bash
# Create a topic
gcloud pubsub topics create order-events

# Create a topic with a schema
gcloud pubsub topics create order-events \
  --schema=order-schema \
  --message-encoding=json

# Publish a message
gcloud pubsub topics publish order-events \
  --message='{"orderId": "12345", "status": "placed"}' \
  --attribute=eventType=order.placed,region=us-east1
```

### Pub/Sub Subscriptions

A **subscription** is a named resource that represents the stream of messages from a topic to a subscribing application. Each subscription receives a copy of every message published to the topic.

**Subscription types:**

| Type | Description | Use Case |
|------|-------------|----------|
| **Pull** | Subscriber polls for messages | Batch processing, variable throughput |
| **Push** | Pub/Sub delivers to HTTPS endpoint | Serverless, webhook-based processing |
| **BigQuery** | Writes directly to BigQuery | Analytics, data warehousing |
| **Cloud Storage** | Writes to GCS buckets | Archival, batch analytics |

```bash
# Create a pull subscription
gcloud pubsub subscriptions create order-processor \
  --topic=order-events \
  --ack-deadline=60 \
  --message-retention-duration=7d \
  --expiration-period=never

# Create a push subscription
gcloud pubsub subscriptions create order-webhook \
  --topic=order-events \
  --push-endpoint=https://myapp.example.com/pubsub/push \
  --ack-deadline=30
```

### Exactly-Once Delivery

Pub/Sub supports **exactly-once delivery** for pull subscriptions. When enabled, the service deduplicates messages and tracks acknowledgments to ensure each message is delivered and processed exactly once.

**Requirements for exactly-once delivery:**

- Pull subscriptions only (not push)
- Client must use the Cloud Client Library with exactly-once enabled
- Acknowledgment IDs must be used within their validity window
- Slight increase in publish and subscribe latency

### Ordering Keys

**Ordering keys** ensure that messages with the same key are delivered to subscribers in the order they were published. Without ordering keys, Pub/Sub may deliver messages out of order for higher throughput.

```
Ordering Key Behavior
─────────────────────

  Published order:                    Delivered order:

  Key "A": [A1] [A2] [A3]     ──▶    Key "A": [A1] [A2] [A3]  (ordered)
  Key "B": [B1] [B2]          ──▶    Key "B": [B1] [B2]        (ordered)
  No key:  [X1] [X2] [X3]     ──▶    No key:  [X2] [X1] [X3]  (any order)

  Ordering is guaranteed WITHIN the same key, not across keys.
```

```bash
# Create a subscription with ordering enabled
gcloud pubsub subscriptions create ordered-processor \
  --topic=order-events \
  --enable-message-ordering

# Publish with an ordering key
gcloud pubsub topics publish order-events \
  --message='{"step": 1}' \
  --ordering-key=order-12345
```

### Dead Letter Topics

A **dead letter topic** captures messages that cannot be processed after a configured number of delivery attempts. This prevents poison messages from blocking a subscription.

```bash
# Create a dead letter topic and subscription
gcloud pubsub topics create order-events-dlq
gcloud pubsub subscriptions create order-events-dlq-sub \
  --topic=order-events-dlq

# Configure a subscription with a dead letter policy
gcloud pubsub subscriptions update order-processor \
  --dead-letter-topic=order-events-dlq \
  --max-delivery-attempts=5
```

---

## Service Comparison

### Feature Comparison

| Feature | AWS SQS | AWS SNS | Azure Service Bus | Azure Event Hubs | GCP Pub/Sub |
|---------|---------|---------|-------------------|-----------------|-------------|
| **Model** | Queue | Pub/Sub | Queue + Pub/Sub | Streaming | Pub/Sub |
| **Ordering** | FIFO only | FIFO only | Sessions / FIFO | Per partition | Ordering keys |
| **Max message size** | 256 KB | 256 KB | 256 KB / 100 MB | 1 MB | 10 MB |
| **Retention** | 14 days max | None (instant) | Unlimited | 1-90 days | 31 days max |
| **Dead letter** | Yes | No (use SQS) | Yes | No (custom) | Yes |
| **Exactly-once** | FIFO only | No | Duplicate detect | No | Pull subs only |
| **Transactions** | No | No | Yes | No | No |
| **Batch send** | Up to 10 | Up to 10 | Yes (AMQP batch) | Yes (AMQP batch) | Up to 1,000 |
| **Protocol** | HTTPS/SDK | HTTPS/SDK | AMQP/HTTPS/SDK | AMQP/Kafka/HTTPS | gRPC/HTTPS |

### Delivery Guarantees

| Service | At-Least-Once | At-Most-Once | Exactly-Once |
|---------|:------------:|:------------:|:------------:|
| AWS SQS Standard | ✓ | | |
| AWS SQS FIFO | | | ✓ |
| AWS SNS | ✓ | | |
| Azure Service Bus | ✓ | ✓ (PeekLock) | ✓ (Dedup) |
| Azure Event Hubs | ✓ | | |
| GCP Pub/Sub | ✓ | | ✓ (Pull only) |

### Throughput and Latency

| Service | Throughput | Typical Latency | Scaling Model |
|---------|-----------|-----------------|---------------|
| AWS SQS Standard | Unlimited | Single-digit ms | Automatic |
| AWS SQS FIFO | 3,000 msg/s (batch) | Single-digit ms | Per queue |
| AWS SNS | Unlimited | Single-digit ms | Automatic |
| Azure Service Bus | 1-16 Messaging Units (Premium) | ~20-25 ms | Manual (MU-based) |
| Azure Event Hubs | Millions/sec | ~10 ms | Throughput Units |
| GCP Pub/Sub | Unlimited | ~50-100 ms | Automatic |

---

## Multi-Cloud Messaging

### Abstraction Patterns

When building applications that must work across multiple cloud providers, an abstraction layer decouples application code from provider-specific APIs.

```
Multi-Cloud Messaging Abstraction
──────────────────────────────────

  Application Layer
  ┌──────────────────────────────────────────┐
  │            Application Code              │
  │                                          │
  │  publish(topic, message)                 │
  │  subscribe(topic, handler)               │
  └─────────────────┬────────────────────────┘
                    │
  Abstraction Layer │
  ┌─────────────────┴────────────────────────┐
  │         Messaging Interface              │
  │                                          │
  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ │
  │  │ AWS      │ │ Azure    │ │ GCP      │ │
  │  │ Adapter  │ │ Adapter  │ │ Adapter  │ │
  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ │
  └───────┼────────────┼────────────┼────────┘
          ▼            ▼            ▼
     ┌─────────┐ ┌───────────┐ ┌─────────┐
     │ SNS+SQS │ │ Service   │ │ Pub/Sub │
     │         │ │ Bus       │ │         │
     └─────────┘ └───────────┘ └─────────┘
```

**Common abstraction approaches:**

- **Dapr**: Open-source runtime with a pub/sub building block that supports 40+ message brokers
- **CloudEvents**: CNCF specification for describing event data in a common format
- **Custom interface**: Application-specific wrapper with provider adapters
- **Apache Kafka (managed)**: Available on all clouds (MSK, Event Hubs, Confluent Cloud)

### Portability Considerations

| Consideration | Recommendation |
|---------------|---------------|
| **Message format** | Use CloudEvents or a custom envelope format with provider-agnostic fields |
| **Serialization** | Prefer JSON or Avro; avoid provider-specific binary formats |
| **Feature usage** | Limit usage to features available across target providers (e.g., avoid Azure sessions if targeting AWS) |
| **IAM and auth** | Abstract authentication behind a credential provider interface |
| **Monitoring** | Use OpenTelemetry for vendor-neutral observability |
| **Dead letters** | Implement DLQ handling in application code rather than relying on provider-specific DLQ configuration |
| **Ordering** | Design around partition/ordering key semantics (common across providers) rather than provider-specific session features |

> **Trade-off:** Abstraction layers add complexity and may prevent you from using provider-specific optimizations. Choose portability only when multi-cloud is a genuine requirement.

---

## Cost Optimization

### Pricing Models

Cloud messaging services use different pricing models. Understanding these models is critical for cost-effective architecture.

| Service | Pricing Basis | Free Tier | Key Cost Drivers |
|---------|--------------|-----------|-----------------|
| **AWS SQS** | Per request (send, receive, delete) | 1M requests/month | Number of API calls, data transfer |
| **AWS SNS** | Per publish + per delivery | 1M publishes/month | Number of publishes, SMS/email delivery |
| **AWS EventBridge** | Per event ingested | N/A | Number of events, schema discovery |
| **Azure Service Bus** | Per operation (Standard) or per MU (Premium) | N/A | Operations, message size, tier |
| **Azure Event Hubs** | Per Throughput Unit (TU) | N/A | TUs, ingress, capture |
| **GCP Pub/Sub** | Per data volume (publish + deliver) | 10 GB/month | Data volume, message storage |

### Batching for Cost Reduction

Batching reduces cost by consolidating multiple logical messages into fewer API calls.

```
Cost Impact of Batching (AWS SQS Example)
──────────────────────────────────────────

  Without Batching:
  ┌─────────┐     10,000 send calls = 10,000 requests
  │ 10,000  │     10,000 receive calls = 10,000 requests
  │ messages│     10,000 delete calls = 10,000 requests
  └─────────┘     Total: 30,000 requests

  With Batching (batch size = 10):
  ┌─────────┐     1,000 SendMessageBatch calls = 1,000 requests
  │ 10,000  │     1,000 ReceiveMessage calls = 1,000 requests
  │ messages│     1,000 DeleteMessageBatch calls = 1,000 requests
  └─────────┘     Total: 3,000 requests  (90% reduction)
```

**Batching strategies by provider:**

| Provider | Batch Operation | Max Batch Size | Recommendation |
|----------|----------------|---------------|----------------|
| AWS SQS | `SendMessageBatch` / `DeleteMessageBatch` | 10 messages | Always batch sends and deletes |
| AWS SNS | `PublishBatch` | 10 messages | Batch when publishing to same topic |
| Azure Service Bus | AMQP batch send | Limited by message size | Use AMQP client with batch mode |
| GCP Pub/Sub | Batch publish (client-side) | 1,000 messages or 10 MB | Configure client batch settings |

### Reserved Capacity

For predictable workloads, reserved capacity can significantly reduce costs.

- **Azure Service Bus Premium**: Purchase Messaging Units for dedicated capacity (1 MU ≈ $670/month)
- **Azure Event Hubs Dedicated**: Reserved Capacity Units for guaranteed throughput
- **AWS**: No reserved pricing for SQS/SNS, but consider SQS FIFO high-throughput mode
- **GCP Pub/Sub**: No reserved pricing; use committed use discounts on compute that processes messages

> **Tip:** For high-volume workloads, compare the total cost of a managed service against self-hosted Kafka or RabbitMQ on reserved compute instances.

---

## Migration Strategies

### Moving Between Brokers

Migrating from one messaging service to another (or from self-hosted to managed) requires careful planning to avoid message loss and downtime.

```
Dual-Write Migration Pattern
─────────────────────────────

  Phase 1: Dual Write                    Phase 2: Cutover

  ┌──────────┐                           ┌──────────┐
  │ Producer │                           │ Producer │
  └────┬─────┘                           └────┬─────┘
       │                                      │
       ├──────────────┐                       │ (write only to new)
       ▼              ▼                       ▼
  ┌─────────┐   ┌─────────┐            ┌─────────┐
  │ Old     │   │ New     │            │ New     │
  │ Broker  │   │ Broker  │            │ Broker  │
  └────┬────┘   └────┬────┘            └────┬────┘
       ▼              ▼                      ▼
  [Consumer A]   [Consumer B]           [Consumer B]
  (existing)     (validates)            (primary)
```

**Migration steps:**

1. **Assess**: Catalog all producers, consumers, topics, queues, and message schemas
2. **Parallel run**: Write to both old and new broker; consume from new broker for validation
3. **Verify**: Compare message counts, ordering, and processing correctness between old and new
4. **Cutover consumers**: Switch consumers to read from the new broker
5. **Cutover producers**: Switch producers to write only to the new broker
6. **Decommission**: Drain and remove the old broker after a safety period

### Hybrid Approaches

In some scenarios, running both self-hosted and cloud-managed brokers is the right choice.

**Bridge pattern:**

```
Hybrid Bridge Architecture
──────────────────────────

  On-Premises / Self-Hosted          Cloud
  ┌──────────────────────┐          ┌──────────────────────┐
  │                      │          │                      │
  │  ┌──────────────┐   │  Bridge  │  ┌──────────────┐   │
  │  │ Kafka /      │   │◄────────▶│  │ SQS / SNS /  │   │
  │  │ RabbitMQ     │   │          │  │ Pub/Sub /     │   │
  │  └──────────────┘   │          │  │ Service Bus   │   │
  │                      │          │                      │
  └──────────────────────┘          └──────────────────────┘
```

**Bridge technologies:**

| Approach | Description | Best For |
|----------|-------------|----------|
| **Kafka Connect** | Source/sink connectors for SQS, Pub/Sub, Service Bus | Kafka-centric environments |
| **Kafka MirrorMaker 2** | Replication between Kafka clusters (on-prem to MSK) | Kafka-to-Kafka migration |
| **Azure Relay** | Hybrid connections between on-premises and Azure | Azure-centric workloads |
| **Custom bridge service** | Application-level message forwarding | Full control, any broker pair |
| **Event Hubs Kafka endpoint** | Kafka protocol over Event Hubs | Kafka apps moving to Azure |

**Hybrid use cases:**

- Regulatory requirements mandate certain data stays on-premises
- Gradual cloud migration with minimal disruption
- Multi-region architectures with some regions on-premises
- Burst capacity: overflow to cloud messaging during peak load

---

## Next Steps

Continue your messaging learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | Messaging Fundamentals | Core messaging concepts, delivery guarantees, event-driven architecture |
| [01-APACHE-KAFKA.md](01-APACHE-KAFKA.md) | Apache Kafka | Distributed event streaming, topics, partitions, consumer groups |
| [02-RABBITMQ.md](02-RABBITMQ.md) | RabbitMQ | Message queues, exchanges, bindings, AMQP protocol |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial cloud messaging services documentation |
