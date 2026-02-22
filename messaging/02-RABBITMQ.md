# RabbitMQ

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [RabbitMQ Architecture](#rabbitmq-architecture)
   - [Nodes](#nodes)
   - [Clusters](#clusters)
   - [Virtual Hosts](#virtual-hosts)
   - [Connections and Channels](#connections-and-channels)
3. [Exchanges](#exchanges)
   - [Direct Exchange](#direct-exchange)
   - [Fanout Exchange](#fanout-exchange)
   - [Topic Exchange](#topic-exchange)
   - [Headers Exchange](#headers-exchange)
   - [Default Exchange](#default-exchange)
4. [Queues](#queues)
   - [Classic Queues](#classic-queues)
   - [Quorum Queues](#quorum-queues)
   - [Streams](#streams)
   - [Queue Properties](#queue-properties)
5. [Bindings and Routing](#bindings-and-routing)
   - [Binding Keys](#binding-keys)
   - [Routing Keys](#routing-keys)
   - [Routing Patterns](#routing-patterns)
6. [Publishers](#publishers)
   - [Publishing Messages](#publishing-messages)
   - [Publisher Confirms](#publisher-confirms)
   - [Mandatory Flag](#mandatory-flag)
   - [Transactions](#transactions)
7. [Consumers](#consumers)
   - [Basic.Consume](#basicconsume)
   - [Prefetch Count](#prefetch-count)
   - [Acknowledgments](#acknowledgments)
   - [Consumer Cancellation](#consumer-cancellation)
8. [Reliability](#reliability)
   - [Message Durability](#message-durability)
   - [Publisher Confirms](#publisher-confirms-1)
   - [Consumer Acknowledgments](#consumer-acknowledgments)
   - [Dead Letter Exchanges](#dead-letter-exchanges)
9. [Clustering and High Availability](#clustering-and-high-availability)
   - [Cluster Formation](#cluster-formation)
   - [Quorum Queues for HA](#quorum-queues-for-ha)
   - [Federation](#federation)
   - [Shovel](#shovel)
10. [Security](#security)
    - [Authentication](#authentication)
    - [Authorization](#authorization)
    - [TLS](#tls)
    - [Access Control](#access-control)
11. [Performance Tuning](#performance-tuning)
    - [Queue Length](#queue-length)
    - [Prefetch Count Tuning](#prefetch-count-tuning)
    - [Lazy Queues](#lazy-queues)
    - [Memory Management](#memory-management)
12. [RabbitMQ vs Kafka](#rabbitmq-vs-kafka)
    - [Architectural Differences](#architectural-differences)
    - [When to Choose RabbitMQ](#when-to-choose-rabbitmq)
    - [When to Choose Kafka](#when-to-choose-kafka)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

---

## Overview

RabbitMQ is an open-source message broker that implements the Advanced Message Queuing Protocol (AMQP 0-9-1). Originally developed by Rabbit Technologies and now maintained under the VMware umbrella, RabbitMQ is one of the most widely deployed message brokers in the world. It acts as an intermediary for messaging — accepting messages from producers, routing them through exchanges and bindings, and delivering them to consumer applications via queues.

AMQP 0-9-1 is a programmable protocol in the sense that AMQP entities and routing schemes are defined by applications themselves, not the broker administrator. This gives developers full control over exchange declarations, queue bindings, and routing topology. RabbitMQ also supports additional protocols including MQTT, STOMP, and AMQP 1.0 through plugins.

```
AMQP 0-9-1 Model (Simplified)
──────────────────────────────

  Producer                          Consumer
  ┌──────┐    ┌──────────┐    ┌─────────┐    ┌──────┐
  │ App  │───▶│ Exchange │───▶│  Queue  │───▶│ App  │
  └──────┘    └──────────┘    └─────────┘    └──────┘
                  │                ▲
                  │   Binding      │
                  └────────────────┘
              (routing key match)

  1. Producer publishes message to an Exchange
  2. Exchange routes message to Queue(s) via Bindings
  3. Consumer receives message from the Queue
```

### Target Audience

- **Backend Developers** building applications that produce and consume messages through RabbitMQ queues
- **DevOps Engineers** deploying, configuring, and monitoring RabbitMQ clusters in production environments
- **Architects** designing asynchronous, decoupled systems with reliable message delivery
- **Platform Engineers** evaluating RabbitMQ for task queues, event distribution, and inter-service communication

### Scope

- AMQP 0-9-1 protocol model: connections, channels, exchanges, queues, bindings
- RabbitMQ architecture: nodes, clusters, virtual hosts
- Exchange types: direct, fanout, topic, headers
- Queue types: classic queues, quorum queues, streams
- Publishing: confirms, mandatory flag, transactions
- Consuming: prefetch, acknowledgments, consumer cancellation
- Reliability: durability, confirms, acknowledgments, dead letter exchanges
- Clustering: formation, quorum queues, federation, shovel
- Security: authentication, authorization, TLS
- Performance tuning and memory management
- Comparison with Apache Kafka

---

## RabbitMQ Architecture

RabbitMQ is built on the Erlang/OTP platform, which provides lightweight processes, fault tolerance, and distributed computing primitives. Understanding RabbitMQ's architectural building blocks is essential before working with exchanges, queues, and bindings.

### Nodes

A **RabbitMQ node** is a single running instance of the RabbitMQ server (an Erlang VM process). Each node has its own name (e.g., `rabbit@hostname`), data directory, and configuration. A node can operate standalone or as part of a cluster.

```
RabbitMQ Node Internals
───────────────────────

┌─────────────────────────────────────────────┐
│               RabbitMQ Node                 │
│           (rabbit@hostname)                 │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Mnesia   │  │ Message  │  │ Plugin   │  │
│  │ Database │  │  Store   │  │ System   │  │
│  │ (metadata│  │ (queues, │  │ (mgmt,   │  │
│  │  schema) │  │  msgs)   │  │  auth)   │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │        Erlang/OTP Runtime            │   │
│  │   (lightweight processes, BEAM VM)   │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

Key components within a node:

- **Mnesia database** — Stores metadata: exchange definitions, queue definitions, bindings, virtual hosts, users, and permissions
- **Message store** — Persists message bodies to disk when durability is required
- **Plugin system** — Extends functionality (management UI, MQTT, STOMP, federation, shovel, etc.)

### Clusters

A **cluster** is a group of RabbitMQ nodes that share users, virtual hosts, queues, exchanges, bindings, and runtime parameters. Clustering provides both scalability and fault tolerance.

```
RabbitMQ Cluster (3 Nodes)
──────────────────────────

┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ rabbit@node1  │    │ rabbit@node2  │    │ rabbit@node3  │
│               │    │               │    │               │
│  Exchanges ◄──┼────┼── Exchanges ◄─┼────┼── Exchanges   │
│  (replicated) │    │  (replicated) │    │  (replicated) │
│               │    │               │    │               │
│  Queue A      │    │  Queue B      │    │  Queue C      │
│  (leader)     │    │  (leader)     │    │  (leader)     │
│               │    │               │    │               │
│  Queue B      │    │  Queue C      │    │  Queue A      │
│  (follower)   │    │  (follower)   │    │  (follower)   │
└───────┬───────┘    └───────┬───────┘    └───────┬───────┘
        │                    │                    │
        └────────────────────┴────────────────────┘
                  Erlang Distribution Protocol
                  (inter-node communication)
```

**Cluster facts:**

- **Metadata is replicated** across all nodes (exchanges, bindings, users, vhosts, policies)
- **Classic queue data lives on one node** by default — the node where the queue was declared
- **Quorum queues replicate data** across multiple nodes via the Raft consensus algorithm
- Nodes communicate using the **Erlang distribution protocol** (port 25672 by default)
- All nodes must run the **same RabbitMQ and Erlang versions**
- Nodes share an **Erlang cookie** for authentication between each other

### Virtual Hosts

A **virtual host** (vhost) provides logical grouping and separation of resources. Each vhost has its own set of exchanges, queues, bindings, users, and permissions. Vhosts are the primary mechanism for multi-tenancy in RabbitMQ.

```
Virtual Host Isolation
──────────────────────

┌─────────────────────────────────────────────┐
│              RabbitMQ Node                  │
│                                             │
│  ┌───────────────────┐  ┌────────────────┐  │
│  │  vhost: /prod     │  │ vhost: /staging│  │
│  │                   │  │                │  │
│  │  Exchange: orders │  │ Exchange: orders│  │
│  │  Queue: order.new │  │ Queue: order.new│  │
│  │  User: prod-app   │  │ User: stg-app  │  │
│  │                   │  │                │  │
│  │  (isolated from   │  │ (isolated from │  │
│  │   /staging)       │  │  /prod)        │  │
│  └───────────────────┘  └────────────────┘  │
└─────────────────────────────────────────────┘

  - Same exchange/queue names can exist in different vhosts
  - Users are granted permissions per vhost
  - Default vhost is "/"
```

### Connections and Channels

Clients connect to RabbitMQ over **TCP connections** (AMQP port 5672, AMQPS port 5671). Each connection can multiplex many **channels** — lightweight logical connections that share the underlying TCP socket.

```
Connection and Channel Model
─────────────────────────────

  Client Application
  ┌────────────────────────────────┐
  │                                │
  │  ┌──────────┐  ┌──────────┐   │
  │  │Channel 1 │  │Channel 2 │   │       TCP Connection
  │  │(publish) │  │(consume) │   │  ═══════════════════════▶  RabbitMQ
  │  └──────────┘  └──────────┘   │       (port 5672)
  │  ┌──────────┐                 │
  │  │Channel 3 │                 │
  │  │(consume) │                 │
  │  └──────────┘                 │
  │                                │
  └────────────────────────────────┘

  - One TCP connection per application instance (typical)
  - One channel per thread (channels are NOT thread-safe)
  - Channels are cheap to create and destroy
  - Avoid opening a new TCP connection per operation
```

**Best practices:**

- Use **one connection per application process** and **one channel per thread**
- Channels are not thread-safe — never share a channel between threads
- Close channels and connections when they are no longer needed
- Connection and channel lifecycle errors can be caught via event handlers in client libraries

---

## Exchanges

An **exchange** is a routing agent in RabbitMQ. Producers never send messages directly to queues — they publish to an exchange, which then routes the message to zero or more queues based on the exchange type, bindings, and routing key.

### Direct Exchange

A **direct exchange** routes messages to queues whose binding key exactly matches the message's routing key. This is the simplest and most common routing model.

```
Direct Exchange Routing
───────────────────────

  Producer                      Direct Exchange
  ┌──────┐  routing_key="pdf"   ┌──────────┐
  │ App  │─────────────────────▶│  docs    │
  └──────┘                      └────┬─────┘
                                     │
                    ┌────────────────┬┴────────────────┐
                    │                │                  │
              binding="pdf"   binding="html"    binding="csv"
                    │                │                  │
                    ▼                ▼                  ▼
              ┌──────────┐    ┌──────────┐      ┌──────────┐
              │ pdf_queue│    │html_queue│      │csv_queue │
              └──────────┘    └──────────┘      └──────────┘

  Message with routing_key="pdf" → delivered ONLY to pdf_queue
```

**Use cases:** Task distribution, RPC-style request/reply, targeted message routing.

### Fanout Exchange

A **fanout exchange** ignores the routing key entirely and broadcasts every message to **all queues** bound to it. This is the pub/sub (publish-subscribe) pattern.

```
Fanout Exchange Routing
───────────────────────

  Producer                      Fanout Exchange
  ┌──────┐  routing_key ignored ┌──────────┐
  │ App  │─────────────────────▶│  events  │
  └──────┘                      └────┬─────┘
                                     │
                    ┌────────────────┬┴───────────────┐
                    │                │                 │
                    ▼                ▼                 ▼
              ┌──────────┐    ┌──────────┐     ┌──────────┐
              │ email    │    │ audit    │     │ analytics│
              │ _queue   │    │ _queue   │     │ _queue   │
              └──────────┘    └──────────┘     └──────────┘

  Every message is delivered to ALL three queues
```

**Use cases:** Broadcasting events to multiple consumers, audit logging, notifications.

### Topic Exchange

A **topic exchange** routes messages based on wildcard pattern matching between the routing key and binding patterns. Routing keys are dot-delimited strings (e.g., `order.created.us`).

```
Topic Exchange Routing
──────────────────────

  Wildcard patterns:
    *  (star)  — matches exactly one word
    #  (hash)  — matches zero or more words

  Producer: routing_key = "order.created.us"

                              Topic Exchange
                              ┌──────────┐
                              │  events  │
                              └────┬─────┘
                                   │
            ┌──────────────────────┼──────────────────────┐
            │                      │                      │
   binding="order.created.*" binding="order.#"  binding="*.created.eu"
            │                      │                      │
            ▼                      ▼                      ▼
      ┌──────────┐           ┌──────────┐          ┌──────────┐
      │ new_order│           │all_orders│          │ eu_queue │
      │ _queue   │           │ _queue   │          │          │
      └──────────┘           └──────────┘          └──────────┘
         MATCH                  MATCH               NO MATCH
    ("us" matches *)       (# matches all)     ("us" ≠ "eu")
```

| Pattern | Matches | Does Not Match |
|---------|---------|----------------|
| `order.created.*` | `order.created.us`, `order.created.eu` | `order.created`, `order.created.us.east` |
| `order.#` | `order`, `order.created`, `order.created.us` | `payment.created` |
| `*.created.*` | `order.created.us`, `user.created.eu` | `order.updated.us` |
| `#` | Everything | — |

**Use cases:** Selective event routing, geographic or category-based filtering, flexible pub/sub.

### Headers Exchange

A **headers exchange** routes messages based on message header attributes instead of the routing key. The `x-match` argument controls matching behavior.

```
Headers Exchange Routing
────────────────────────

  Message headers: { format: "pdf", type: "report" }

                            Headers Exchange
                            ┌──────────┐
                            │  docs    │
                            └────┬─────┘
                                 │
               ┌─────────────────┼─────────────────┐
               │                 │                 │
         x-match: all      x-match: any      x-match: all
         format=pdf        format=pdf        format=csv
         type=report       type=invoice      type=report
               │                 │                 │
               ▼                 ▼                 ▼
         ┌──────────┐     ┌──────────┐      ┌──────────┐
         │ reports  │     │ pdf_any  │      │ csv_rpts │
         └──────────┘     └──────────┘      └──────────┘
            MATCH            MATCH           NO MATCH
         (both match)    (format matches)  (format ≠ csv)
```

| `x-match` Value | Behavior |
|------------------|----------|
| `all` | All specified headers must match |
| `any` | At least one specified header must match |

**Use cases:** Routing based on metadata rather than routing key, complex multi-attribute filtering.

### Default Exchange

The **default exchange** is a nameless direct exchange (`""`) pre-declared by the broker. Every queue is automatically bound to it with a binding key equal to the queue name. This allows publishing directly to a queue by name.

```python
# Publishing to the default exchange routes to the queue named "task_queue"
channel.basic_publish(
    exchange='',              # default exchange
    routing_key='task_queue', # queue name as routing key
    body='Hello World'
)
```

---

## Queues

Queues store messages until they are consumed. RabbitMQ supports multiple queue types, each with different trade-offs for durability, performance, and replication.

### Classic Queues

**Classic queues** are the original queue type in RabbitMQ. They store messages on a single node (the queue leader) and optionally mirror to other nodes (classic mirrored queues — now deprecated in favor of quorum queues).

```
Classic Queue (Single Node)
───────────────────────────

  ┌────────────────────────────────────┐
  │           rabbit@node1             │
  │                                    │
  │  ┌──────────────────────────────┐  │
  │  │        Classic Queue         │  │
  │  │                              │  │
  │  │  Head ──▶ [M1][M2][M3] ──▶ Tail│
  │  │                              │  │
  │  │  Messages stored in memory   │  │
  │  │  and/or paged to disk        │  │
  │  └──────────────────────────────┘  │
  │                                    │
  └────────────────────────────────────┘

  - Data lives on ONE node only
  - If that node goes down, queue is unavailable
  - Fast for single-node deployments
  - Use quorum queues for HA requirements
```

### Quorum Queues

**Quorum queues** are a replicated queue type based on the Raft consensus algorithm. They provide data safety and high availability by replicating messages across multiple nodes.

```
Quorum Queue (Raft Consensus)
─────────────────────────────

  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
  │ rabbit@node1  │    │ rabbit@node2  │    │ rabbit@node3  │
  │               │    │               │    │               │
  │  ┌─────────┐  │    │  ┌─────────┐  │    │  ┌─────────┐  │
  │  │ Leader  │  │    │  │Follower │  │    │  │Follower │  │
  │  │ [M1-M5] │◄─┼────┼─▶│ [M1-M5] │◄─┼────┼─▶│ [M1-M5] │  │
  │  └─────────┘  │    │  └─────────┘  │    │  └─────────┘  │
  │               │    │               │    │               │
  └───────────────┘    └───────────────┘    └───────────────┘

  Write path:
    1. Client publishes to leader
    2. Leader replicates to followers
    3. Majority (quorum) acknowledges → message committed
    4. Leader confirms to publisher

  - Tolerates (N-1)/2 node failures (e.g., 1 failure in a 3-node cluster)
  - Automatic leader election on failure
  - Recommended for production workloads requiring HA
```

**Key differences from classic queues:**

| Feature | Classic Queue | Quorum Queue |
|---------|--------------|--------------|
| Replication | None (or deprecated mirroring) | Raft-based consensus |
| Data safety | Single node | Majority quorum |
| Performance | Higher throughput (single node) | Slightly lower (replication overhead) |
| Message TTL | Supported | Supported (per-queue) |
| Poison message handling | Manual | Built-in delivery limit |
| Non-durable messages | Supported | Not supported (always durable) |
| Priority | Supported | Not supported |

### Streams

**RabbitMQ Streams** (introduced in 3.9) provide an append-only log data structure, similar to Apache Kafka topics. They are designed for high-throughput, replay-capable workloads.

```
RabbitMQ Stream
───────────────

  ┌────────────────────────────────────────────┐
  │                  Stream                    │
  │                                            │
  │  Offset: 0    1    2    3    4    5    6   │
  │         [M0] [M1] [M2] [M3] [M4] [M5] [M6]│
  │                              ▲              │
  │                              │              │
  │                     Consumer offset         │
  │                                            │
  │  - Append-only (no destructive consume)    │
  │  - Consumers can replay from any offset    │
  │  - Retention-based (time or size)          │
  │  - Non-destructive: messages remain after  │
  │    consumption                             │
  └────────────────────────────────────────────┘

  vs. Traditional Queue:
    Queue:  [M0][M1][M2] → consume → [M1][M2] (M0 removed)
    Stream: [M0][M1][M2] → consume → [M0][M1][M2] (M0 remains)
```

**When to use streams:**

- Fan-out to many consumers (each tracks its own offset)
- Temporal replay (re-read historical messages)
- High-throughput append-only workloads
- Large backlogs where traditional queues would struggle with memory

### Queue Properties

Queues are declared with properties that control their behavior:

| Property | Description | Default |
|----------|-------------|---------|
| `durable` | Queue survives broker restart | `false` |
| `exclusive` | Queue is used by only one connection and deleted on close | `false` |
| `auto-delete` | Queue is deleted when last consumer unsubscribes | `false` |
| `x-queue-type` | Queue type: `classic`, `quorum`, or `stream` | `classic` |
| `x-max-length` | Maximum number of messages in the queue | Unlimited |
| `x-max-length-bytes` | Maximum total size of messages in bytes | Unlimited |
| `x-message-ttl` | Per-queue message time-to-live (ms) | Unlimited |
| `x-expires` | Auto-delete queue after idle period (ms) | Never |
| `x-dead-letter-exchange` | Exchange for dead-lettered messages | None |
| `x-dead-letter-routing-key` | Routing key for dead-lettered messages | Original key |
| `x-overflow` | Behavior when max-length reached: `drop-head` or `reject-publish` | `drop-head` |

---

## Bindings and Routing

Bindings are rules that exchanges use to route messages to queues. A binding links an exchange to a queue with an optional **binding key** (also called a routing pattern).

### Binding Keys

A **binding key** is specified when creating a binding between an exchange and a queue. Its interpretation depends on the exchange type:

| Exchange Type | Binding Key Behavior |
|---------------|---------------------|
| Direct | Exact string match against routing key |
| Fanout | Ignored — all bound queues receive messages |
| Topic | Wildcard pattern match (`*` and `#`) |
| Headers | Not used — matching is based on headers |

### Routing Keys

A **routing key** is a string attached to each message by the publisher. The exchange examines the routing key and compares it against binding keys to determine which queues should receive the message.

```
Routing Key Flow
────────────────

  Publisher sets routing_key = "order.created.us"
       │
       ▼
  ┌──────────┐
  │ Exchange │ ← examines routing_key
  └────┬─────┘
       │
       ├─── binding_key = "order.created.us"  → MATCH (direct)
       ├─── binding_key = "order.created.*"   → MATCH (topic)
       ├─── binding_key = "order.#"           → MATCH (topic)
       └─── binding_key = "payment.created.*" → NO MATCH
```

### Routing Patterns

The full routing lifecycle in RabbitMQ follows a consistent pattern:

```
Complete Routing Flow
─────────────────────

  ┌──────────┐     AMQP        ┌──────────┐    Bindings    ┌──────────┐
  │Publisher │ ──────────────▶ │ Exchange │ ─────────────▶ │  Queue   │
  │          │  basic.publish  │          │  routing_key   │          │
  │          │  routing_key    │          │  ↔ binding_key │          │
  │          │  headers        │          │  ↔ headers     │          │
  └──────────┘                 └──────────┘                └────┬─────┘
                                                               │
                                                          basic.deliver
                                                               │
                                                               ▼
                                                          ┌──────────┐
                                                          │ Consumer │
                                                          └──────────┘

  Routing decisions:
    1. Publisher sends message to named exchange with a routing key
    2. Exchange evaluates routing key against all bindings
    3. Message is copied to each matching queue
    4. If no queues match and mandatory=false, message is silently dropped
    5. If no queues match and mandatory=true, message is returned to publisher
```

**Exchange-to-exchange bindings** (E2E) are also supported, allowing complex routing topologies:

```
Exchange-to-Exchange Binding
────────────────────────────

  ┌──────────┐    binding    ┌──────────┐    binding    ┌──────────┐
  │ Exchange │ ────────────▶ │ Exchange │ ────────────▶ │  Queue   │
  │ (source) │              │  (dest)  │              │          │
  └──────────┘              └──────────┘              └──────────┘

  Use case: Layer routing logic without duplicating bindings
```

---

## Publishers

Publishers are client applications that send messages to RabbitMQ exchanges. Understanding publishing semantics is critical for building reliable systems.

### Publishing Messages

Messages are published using the `basic.publish` AMQP method. Each message consists of a body (payload), routing key, and optional properties.

```
Message Structure
─────────────────

  ┌─────────────────────────────────────────┐
  │              AMQP Message               │
  │                                         │
  │  ┌─────────────────────────────────┐    │
  │  │        Properties (headers)     │    │
  │  │                                 │    │
  │  │  content_type:  "application/   │    │
  │  │                  json"          │    │
  │  │  delivery_mode: 2 (persistent)  │    │
  │  │  message_id:    "uuid-1234"     │    │
  │  │  timestamp:     1700000000      │    │
  │  │  correlation_id: "req-5678"     │    │
  │  │  reply_to:      "reply_queue"   │    │
  │  │  expiration:    "60000"         │    │
  │  └─────────────────────────────────┘    │
  │                                         │
  │  ┌─────────────────────────────────┐    │
  │  │        Body (payload)           │    │
  │  │                                 │    │
  │  │  {"order_id": 42, "total": 99}  │    │
  │  └─────────────────────────────────┘    │
  │                                         │
  └─────────────────────────────────────────┘
```

**Key message properties:**

| Property | Purpose |
|----------|---------|
| `delivery_mode` | 1 = transient, 2 = persistent (survives broker restart) |
| `content_type` | MIME type of the body (e.g., `application/json`) |
| `message_id` | Application-level unique identifier |
| `correlation_id` | Used to correlate RPC responses with requests |
| `reply_to` | Queue name for RPC reply routing |
| `expiration` | Per-message TTL in milliseconds (string) |
| `timestamp` | Unix epoch time when message was created |
| `headers` | Arbitrary key-value pairs for custom metadata |

### Publisher Confirms

**Publisher confirms** (also called confirmations) provide a mechanism for the broker to acknowledge that it has received and processed a published message. This is the primary mechanism for reliable publishing.

```
Publisher Confirm Flow
──────────────────────

  Publisher                          Broker
  ┌──────┐                          ┌──────┐
  │      │── channel.confirm_select ─▶│      │
  │      │◀── confirm_select_ok ─────│      │
  │      │                          │      │
  │      │── basic.publish (seq=1) ─▶│      │  ← message persisted
  │      │── basic.publish (seq=2) ─▶│      │    and routed
  │      │── basic.publish (seq=3) ─▶│      │
  │      │                          │      │
  │      │◀── basic.ack (seq=3,     │      │
  │      │     multiple=true) ──────│      │
  │      │                          │      │
  │      │  (confirms 1, 2, and 3)  │      │
  └──────┘                          └──────┘

  - basic.ack    → broker accepted the message
  - basic.nack   → broker could not process the message
  - multiple=true → confirms all messages up to and including seq
```

**Confirm strategies:**

| Strategy | Throughput | Latency | Complexity |
|----------|-----------|---------|------------|
| Publish and wait (sync) | Low | High | Low |
| Batch confirms | Medium | Medium | Medium |
| Async confirms (callback) | High | Low | High |

### Mandatory Flag

When `mandatory=true`, the broker returns the message to the publisher if it cannot be routed to any queue. Without it, unroutable messages are silently dropped.

```
Mandatory Flag Behavior
───────────────────────

  mandatory=false (default):
    Publisher ──▶ Exchange ──▶ (no matching queue) ──▶ MESSAGE DROPPED

  mandatory=true:
    Publisher ──▶ Exchange ──▶ (no matching queue)
                                    │
                              basic.return
                                    │
    Publisher ◀─────────────────────┘
    (message returned with reply code and text)
```

### Transactions

AMQP transactions (`tx.select`, `tx.commit`, `tx.rollback`) provide atomic publish operations but come with a **severe performance penalty** — throughput drops by a factor of ~250x compared to publisher confirms.

```
Transaction vs Confirms (Performance)
──────────────────────────────────────

  Transactions:    ~  5,000 messages/sec  (tx.select + tx.commit per msg)
  Confirms (sync): ~ 50,000 messages/sec  (wait for each ack)
  Confirms (async):~400,000 messages/sec  (fire-and-confirm asynchronously)

  Recommendation: ALWAYS prefer publisher confirms over transactions
```

---

## Consumers

Consumers are client applications that receive messages from RabbitMQ queues. The AMQP model supports both push-based and pull-based consumption.

### Basic.Consume

**Push-based consumption** (`basic.consume`) is the recommended approach. The broker delivers messages to the consumer as they become available.

```
Push-Based Consumption (basic.consume)
──────────────────────────────────────

  Consumer                          Broker
  ┌──────┐                          ┌──────┐
  │      │── basic.consume ─────────▶│      │
  │      │   (queue="orders")       │      │
  │      │◀── consume_ok ───────────│      │
  │      │                          │      │
  │      │◀── basic.deliver (msg1) ─│      │ ← broker pushes
  │      │◀── basic.deliver (msg2) ─│      │   messages to
  │      │◀── basic.deliver (msg3) ─│      │   consumer
  │      │                          │      │
  │      │── basic.ack (msg1) ──────▶│      │
  │      │── basic.ack (msg2) ──────▶│      │
  └──────┘                          └──────┘
```

**Pull-based consumption** (`basic.get`) fetches one message at a time. It is inefficient and should only be used for polling scenarios.

### Prefetch Count

The **prefetch count** (`basic.qos`) controls how many unacknowledged messages the broker will deliver to a consumer before waiting for acknowledgments. This is critical for flow control and fair distribution.

```
Prefetch Count Behavior
───────────────────────

  prefetch_count = 3

  Broker                              Consumer
  ┌──────┐                            ┌──────┐
  │      │── deliver msg1 ───────────▶│      │ unacked: 1
  │      │── deliver msg2 ───────────▶│      │ unacked: 2
  │      │── deliver msg3 ───────────▶│      │ unacked: 3
  │      │                            │      │
  │      │   (broker STOPS sending)   │      │ ← at limit
  │      │                            │      │
  │      │◀── ack msg1 ──────────────│      │ unacked: 2
  │      │── deliver msg4 ───────────▶│      │ unacked: 3
  │      │                            │      │
  └──────┘                            └──────┘

  - prefetch_count = 0 → unlimited (NOT recommended)
  - prefetch_count = 1 → fair dispatch, lowest throughput
  - prefetch_count = 10-50 → good balance for most workloads
```

### Acknowledgments

Consumers must **acknowledge** messages to tell the broker that processing is complete. Unacknowledged messages remain in the queue and may be redelivered.

| Ack Mode | Behavior | Risk |
|----------|----------|------|
| `basic.ack` | Positive ack — message removed from queue | None (safe) |
| `basic.nack` | Negative ack — message requeued or dead-lettered | Redelivery loops |
| `basic.reject` | Like nack but for single message | Redelivery loops |
| Auto-ack (`no_ack=true`) | Broker removes message on delivery | Message loss on crash |

```
Acknowledgment Modes
────────────────────

  Manual Ack (safe):
    Broker ──▶ deliver ──▶ Consumer ──▶ process ──▶ ack ──▶ Broker removes

  Auto Ack (risky):
    Broker ──▶ deliver ──▶ Broker removes immediately
                           Consumer ──▶ process ──▶ CRASH = message LOST

  Negative Ack with requeue:
    Broker ──▶ deliver ──▶ Consumer ──▶ process fails ──▶ nack(requeue=true)
                                                               │
                                                    message returned to queue
```

### Consumer Cancellation

Consumers can be cancelled in two ways:

- **Client-initiated** — The consumer calls `basic.cancel` to stop receiving messages
- **Broker-initiated** — The broker sends `basic.cancel` to the consumer when the queue is deleted, or the node hosting the queue goes down

```
Consumer Cancellation Notification
──────────────────────────────────

  Broker                              Consumer
  ┌──────┐                            ┌──────┐
  │      │  (queue deleted or         │      │
  │      │   node goes down)          │      │
  │      │                            │      │
  │      │── basic.cancel ───────────▶│      │
  │      │   (consumer_tag)           │      │
  │      │                            │      │
  │      │   Consumer should:         │      │
  │      │   1. Re-declare queue      │      │
  │      │   2. Re-bind              │      │
  │      │   3. Re-consume           │      │
  └──────┘                            └──────┘

  Enable by setting: x-cancel-on-ha-failover = true (for mirrored queues)
  Quorum queues support cancellation natively
```

---

## Reliability

Building reliable messaging systems with RabbitMQ requires a combination of durability, publisher confirms, consumer acknowledgments, and dead letter handling.

### Message Durability

For messages to survive a broker restart, **three conditions** must be met:

```
Message Durability Checklist
────────────────────────────

  ┌─────────────────────────────────────────────────────┐
  │                                                     │
  │  1. Exchange must be durable        ✓ durable=true  │
  │  2. Queue must be durable           ✓ durable=true  │
  │  3. Message must be persistent      ✓ delivery_mode=2│
  │                                                     │
  │  ALL THREE are required for durability.             │
  │  Missing any one means messages may be lost         │
  │  on broker restart.                                 │
  │                                                     │
  └─────────────────────────────────────────────────────┘

  Note: Durability has a performance cost due to disk I/O.
  Quorum queues are ALWAYS durable — they do not accept
  non-durable messages.
```

### Publisher Confirms

Publisher confirms (detailed in the [Publishers](#publisher-confirms) section) are the recommended mechanism for ensuring the broker has accepted a message. Without confirms, a publish call returns immediately with no guarantee of delivery.

```
Reliability Without vs With Confirms
─────────────────────────────────────

  Without confirms:
    Publisher ──▶ publish ──▶ (network issue) ──▶ MESSAGE LOST
                              (broker crash)

  With confirms:
    Publisher ──▶ publish ──▶ wait for ack ──▶ CONFIRMED
                                    │
                               (nack or timeout) ──▶ RETRY
```

### Consumer Acknowledgments

Consumer acknowledgments (detailed in the [Consumers](#acknowledgments) section) ensure that messages are only removed from the queue after successful processing. The combination of publisher confirms and consumer acks provides **at-least-once delivery**.

```
At-Least-Once Delivery
──────────────────────

  Publisher                 Broker                  Consumer
  ┌──────┐                ┌──────┐                ┌──────┐
  │      │── publish ────▶│      │── deliver ────▶│      │
  │      │◀── confirm ───│      │                │      │
  │      │                │      │   (processing) │      │
  │      │                │      │◀── ack ────────│      │
  │      │                │      │                │      │
  └──────┘                └──────┘                └──────┘

  Guarantees:
    ✓ Publisher knows broker received the message (confirm)
    ✓ Broker knows consumer processed the message (ack)
    ✓ Message survives broker restart (durability)
    ✗ Does NOT guarantee exactly-once (consumer may process twice)
```

### Dead Letter Exchanges

A **dead letter exchange** (DLX) receives messages that could not be processed successfully. Messages are dead-lettered when:

- Consumer sends `basic.nack` or `basic.reject` with `requeue=false`
- Message TTL expires
- Queue max-length is exceeded

```
Dead Letter Exchange Flow
─────────────────────────

  ┌──────────┐    ┌─────────────┐    ┌──────────┐
  │ Exchange │───▶│  Main Queue │───▶│ Consumer │
  └──────────┘    │             │    └────┬─────┘
                  │ x-dead-     │         │
                  │ letter-     │    nack(requeue=false)
                  │ exchange=   │         │
                  │ "dlx"       │◀────────┘
                  └──────┬──────┘
                         │
                    dead-letter
                         │
                         ▼
                  ┌──────────┐    ┌─────────────┐
                  │   DLX    │───▶│  DLQ Queue  │──▶ inspect / retry
                  │ Exchange │    │             │
                  └──────────┘    └─────────────┘

  DLX use cases:
    - Retry patterns (with TTL + re-routing)
    - Parking lot for failed messages
    - Audit trail of processing failures
```

**Retry pattern with DLX and TTL:**

```
Retry with Exponential Backoff
──────────────────────────────

  Main Queue                     Retry Queue (with TTL)
  ┌──────────┐  nack            ┌──────────────────┐
  │          │─────────────────▶│ x-message-ttl=   │
  │          │                  │   5000 (5 sec)   │
  │          │  (after TTL)     │ x-dead-letter-   │
  │          │◀─────────────────│   exchange=main  │
  └──────────┘                  └──────────────────┘

  Retry 1: 5s delay  → Retry 2: 15s delay → Retry 3: 45s delay → DLQ
```

---

## Clustering and High Availability

RabbitMQ supports several mechanisms for distributing work and ensuring availability across multiple nodes and data centers.

### Cluster Formation

Clusters can be formed using:

- **CLI** — `rabbitmqctl join_cluster rabbit@node1`
- **Config file** — `cluster_formation.peer_discovery_backend = classic_config`
- **DNS-based discovery** — Automatic via DNS A/AAAA records
- **AWS/Kubernetes discovery** — Via peer discovery plugins

```
Cluster Formation via CLI
─────────────────────────

  Node 2:
    $ rabbitmqctl stop_app
    $ rabbitmqctl join_cluster rabbit@node1
    $ rabbitmqctl start_app

  Node 3:
    $ rabbitmqctl stop_app
    $ rabbitmqctl join_cluster rabbit@node1
    $ rabbitmqctl start_app

  Verify:
    $ rabbitmqctl cluster_status

  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
  │ rabbit@node1  │◄──▶│ rabbit@node2  │◄──▶│ rabbit@node3  │
  │  (disc node)  │    │  (disc node)  │    │  (ram node)   │
  └───────────────┘    └───────────────┘    └───────────────┘
                   Erlang Distribution Protocol

  Node types:
    disc — stores metadata on disk (required for at least one node)
    ram  — stores metadata in memory only (faster joins, less safe)
```

### Quorum Queues for HA

Quorum queues are the **recommended approach for high availability** in RabbitMQ 3.8+. They replace the deprecated classic mirrored queues with a Raft-based replication model.

```
Quorum Queue HA Behavior
─────────────────────────

  Normal operation:
    ┌─────────┐    ┌─────────┐    ┌─────────┐
    │ Leader  │───▶│Follower │───▶│Follower │
    │ (node1) │    │ (node2) │    │ (node3) │
    └─────────┘    └─────────┘    └─────────┘

  Node 1 failure:
    ┌─────────┐    ┌─────────┐    ┌─────────┐
    │  DOWN   │    │ Leader  │───▶│Follower │
    │ (node1) │    │ (node2) │    │ (node3) │
    └─────────┘    └─────────┘    └─────────┘
                   (auto-elected)

  - Automatic leader election (no manual intervention)
  - Publishers and consumers reconnect to new leader
  - No message loss (committed messages are on majority of nodes)
```

**Declaring a quorum queue:**

```python
channel.queue_declare(
    queue='orders',
    durable=True,
    arguments={
        'x-queue-type': 'quorum',
        'x-quorum-initial-group-size': 3
    }
)
```

### Federation

**Federation** allows exchanges and queues to forward messages between RabbitMQ brokers that are **not in the same cluster**. It works over AMQP and tolerates unreliable networks (e.g., WAN links between data centers).

```
Federation (Cross-Datacenter)
─────────────────────────────

  Datacenter A                      Datacenter B
  ┌───────────────────┐            ┌───────────────────┐
  │  RabbitMQ Cluster │            │  RabbitMQ Cluster │
  │                   │    AMQP    │                   │
  │  ┌─────────────┐  │◄──────────▶  ┌─────────────┐  │
  │  │  Exchange:  │  │  (WAN)    │  │  Exchange:  │  │
  │  │  orders     │  │            │  │  orders     │  │
  │  │  (upstream) │  │            │  │ (downstream)│  │
  │  └─────────────┘  │            │  └─────────────┘  │
  └───────────────────┘            └───────────────────┘

  Federation properties:
    - Messages flow from upstream to downstream
    - Tolerant of network partitions and latency
    - Does NOT require Erlang clustering
    - Each broker is fully independent
    - Supports exchange federation and queue federation
```

### Shovel

**Shovel** is a simpler mechanism that continuously moves messages from a queue on one broker to an exchange on another. Unlike federation, shovel operates at the queue level and provides explicit control over message transfer.

```
Shovel (Point-to-Point Transfer)
────────────────────────────────

  Source Broker                     Destination Broker
  ┌───────────────────┐            ┌───────────────────┐
  │                   │            │                   │
  │  ┌─────────────┐  │  Shovel   │  ┌─────────────┐  │
  │  │ source_queue│  │──────────▶│  │ dest_exchange│  │
  │  │             │  │  (AMQP)   │  │             │  │
  │  └─────────────┘  │            │  └─────────────┘  │
  │                   │            │                   │
  └───────────────────┘            └───────────────────┘

  Shovel vs Federation:
    - Shovel: explicit source queue → destination exchange
    - Federation: automatic, policy-driven, exchange-to-exchange or queue-to-queue
    - Shovel: simpler conceptually, more manual configuration
    - Federation: more flexible, better for complex topologies
```

| Feature | Clustering | Federation | Shovel |
|---------|-----------|-----------|--------|
| Network | LAN only | LAN or WAN | LAN or WAN |
| Protocol | Erlang distribution | AMQP | AMQP |
| Topology | Tightly coupled | Loosely coupled | Point-to-point |
| Use case | Single datacenter HA | Cross-datacenter replication | Simple message transfer |
| Queue replication | Yes (quorum) | No (forwarding) | No (forwarding) |

---

## Security

RabbitMQ provides multiple layers of security to protect access to the broker, messages in transit, and resource permissions.

### Authentication

RabbitMQ supports multiple **authentication backends**:

| Backend | Description |
|---------|-------------|
| Internal (built-in) | Username/password stored in Mnesia database |
| LDAP | Authenticate against LDAP/Active Directory |
| HTTP | Delegate authentication to an external HTTP service |
| OAuth 2.0 | Token-based authentication (RabbitMQ 3.8+) |

```bash
# Create a user with the internal backend
rabbitmqctl add_user app_user strong_password

# Set user tags (roles)
rabbitmqctl set_user_tags app_user monitoring

# Delete the default guest user in production
rabbitmqctl delete_user guest
```

### Authorization

Permissions are granted **per virtual host** and control three operations:

| Permission | Controls |
|------------|----------|
| `configure` | Declare/delete exchanges and queues matching a pattern |
| `write` | Publish to exchanges matching a pattern |
| `read` | Consume from queues matching a pattern |

```bash
# Grant full permissions on vhost /prod for user app_user
rabbitmqctl set_permissions -p /prod app_user ".*" ".*" ".*"

# Grant limited permissions: only configure/write/read queues starting with "order"
rabbitmqctl set_permissions -p /prod order_service "^order\..*" "^order\..*" "^order\..*"
```

```
Permission Model
────────────────

  User: order_service
  Vhost: /prod
  Configure: ^order\..*
  Write:     ^order\..*
  Read:      ^order\..*

  ✓ Can declare queue "order.new"
  ✓ Can publish to exchange "order.events"
  ✓ Can consume from queue "order.processing"
  ✗ Cannot declare queue "payment.new"
  ✗ Cannot publish to exchange "payment.events"
```

### TLS

TLS encrypts all traffic between clients and the broker, and between cluster nodes.

```
TLS Configuration (rabbitmq.conf)
─────────────────────────────────

  # Enable TLS listener
  listeners.ssl.default = 5671

  # Certificate paths
  ssl_options.cacertfile = /path/to/ca_certificate.pem
  ssl_options.certfile   = /path/to/server_certificate.pem
  ssl_options.keyfile    = /path/to/server_key.pem

  # Require client certificates (mutual TLS)
  ssl_options.verify     = verify_peer
  ssl_options.fail_if_no_peer_cert = true

  # Minimum TLS version
  ssl_options.versions.1 = tlsv1.3
  ssl_options.versions.2 = tlsv1.2
```

```
TLS Connection Flow
───────────────────

  Client                              Broker (port 5671)
  ┌──────┐                            ┌──────┐
  │      │── TLS ClientHello ────────▶│      │
  │      │◀── TLS ServerHello ────────│      │
  │      │◀── Server Certificate ─────│      │
  │      │── Client Certificate ─────▶│      │  (mutual TLS)
  │      │── Key Exchange ───────────▶│      │
  │      │◀── Finished ──────────────│      │
  │      │                            │      │
  │      │══ Encrypted AMQP traffic ══│      │
  └──────┘                            └──────┘
```

### Access Control

Beyond user permissions, RabbitMQ provides additional access control mechanisms:

- **Topic-level authorization** — Fine-grained control over which routing keys a user can publish to or consume from on topic exchanges
- **Resource limits** — Limit connections and channels per user/vhost
- **IP allowlisting** — Restrict connections by source IP address via firewall or the `rabbitmq_auth_backend_ip_range` plugin

```bash
# Set connection limit for a vhost
rabbitmqctl set_vhost_limits -p /prod '{"max-connections": 500}'

# Set queue limit for a vhost
rabbitmqctl set_vhost_limits -p /prod '{"max-queues": 100}'
```

---

## Performance Tuning

RabbitMQ performance depends on proper configuration of queues, consumers, memory, and disk. Below are the most impactful tuning parameters.

### Queue Length

Long queues degrade performance. Messages in memory consume RAM; once the memory threshold is reached, messages are paged to disk (which is slow).

```
Queue Length Impact
───────────────────

  Short queue (< 10,000 messages):
    ┌────────────────────────────────┐
    │ [M1][M2][M3]...[M100]         │  ← all in memory
    │ Publish rate ≈ Consume rate    │  ← ideal state
    │ Low latency, low memory usage  │
    └────────────────────────────────┘

  Long queue (> 1,000,000 messages):
    ┌────────────────────────────────┐
    │ [M1]...[M1000000]             │  ← memory pressure
    │ Paging to disk triggered       │  ← throughput drops
    │ GC pressure increases          │  ← latency spikes
    └────────────────────────────────┘

  Best practices:
    - Keep queues short (consume as fast as you publish)
    - Set x-max-length to prevent unbounded growth
    - Use x-overflow=reject-publish to apply back-pressure
    - Monitor queue depth as a key metric
```

### Prefetch Count Tuning

The prefetch count has a direct impact on throughput and fairness:

| Prefetch Count | Throughput | Latency | Fair Distribution |
|----------------|-----------|---------|-------------------|
| 1 | Low | Lowest | Best (round-robin) |
| 10–30 | Good | Low | Good |
| 50–100 | High | Medium | Moderate |
| 250+ | Highest | Higher | Poor (one consumer may hoard) |
| 0 (unlimited) | Dangerous | Variable | Very poor |

```
Finding Optimal Prefetch
────────────────────────

  Rule of thumb:
    prefetch = round_trip_time × consumer_rate

  Example:
    Network round trip to broker: 2ms
    Consumer processing rate: 500 msg/sec
    Processing time per message: 2ms (1/500)

    prefetch ≈ (2ms + 2ms) / 2ms = 2  (minimum)
    Recommended: 10–30 for this scenario (headroom for bursts)

  Start with prefetch_count = 20, then adjust:
    - If consumers are idle → increase prefetch
    - If one consumer has too many unacked → decrease prefetch
```

### Lazy Queues

**Lazy queues** write messages directly to disk instead of keeping them in memory first. They are designed for very long queues or scenarios where memory conservation is critical.

```
Classic Queue vs Lazy Queue
───────────────────────────

  Classic Queue (default):
    Publish ──▶ [Memory Buffer] ──▶ [Disk] (when memory is full)
    Fast for short queues, memory-hungry for long queues

  Lazy Queue:
    Publish ──▶ [Disk] ──▶ [Memory] (only when consuming)
    Slower publish, but constant memory usage regardless of queue length

  ┌────────────────────────────────────────────┐
  │  Memory Usage vs Queue Length              │
  │                                            │
  │  Memory │  Classic      /                  │
  │         │  Queue      /                    │
  │         │           /                      │
  │         │         /   Lazy Queue           │
  │         │       /     ─────────────────    │
  │         │     /                            │
  │         │   /                              │
  │         │ /                                │
  │         └──────────────────────────────     │
  │              Queue Length                   │
  └────────────────────────────────────────────┘
```

**When to use lazy queues:**

- Queues that regularly exceed 100,000+ messages
- Consumers that are frequently offline or slow
- Memory-constrained environments
- Batch processing patterns where messages accumulate

**Note:** In RabbitMQ 3.12+, classic queues v2 largely replace lazy queues with a new storage engine that provides similar benefits automatically.

### Memory Management

RabbitMQ uses a **memory-based flow control** mechanism to prevent the broker from running out of memory.

```
Memory Flow Control
───────────────────

  Memory usage thresholds:

  ┌──────────────────────────────────────────────┐
  │  0%                                    100%  │
  │  ├─────────────────┬────────────┬───────┤    │
  │  │    Normal        │  Paging    │ BLOCK │    │
  │  │    operation     │  to disk   │ publishers│
  │  │                 │            │       │    │
  │  │                40%         50%      │    │
  │  │           (paging threshold)        │    │
  │  │                         (high watermark) │
  └──────────────────────────────────────────────┘

  Default high watermark: 0.4 (40% of available RAM)

  Configuration (rabbitmq.conf):
    vm_memory_high_watermark.relative = 0.4
    vm_memory_high_watermark_paging_ratio = 0.5
    disk_free_limit.relative = 1.5
```

**Key memory settings:**

| Setting | Default | Description |
|---------|---------|-------------|
| `vm_memory_high_watermark.relative` | `0.4` | Fraction of RAM at which publishers are blocked |
| `vm_memory_high_watermark_paging_ratio` | `0.5` | Fraction of watermark at which paging to disk begins |
| `disk_free_limit.relative` | `1.0` | Minimum free disk as a multiplier of RAM |
| `vm_memory_calculation_strategy` | `rss` | How to calculate memory usage (`rss` or `allocated`) |

---

## RabbitMQ vs Kafka

RabbitMQ and Apache Kafka are both messaging systems, but they have fundamentally different architectures, semantics, and optimal use cases.

### Architectural Differences

```
RabbitMQ vs Kafka Architecture
──────────────────────────────

  RabbitMQ (Smart Broker / Dumb Consumer):

    Producer ──▶ Exchange ──▶ Queue ──▶ Consumer
                                │
                           Message removed
                           after ack

  Kafka (Dumb Broker / Smart Consumer):

    Producer ──▶ Topic/Partition ──▶ Consumer
                   [M0][M1][M2][M3]
                              ▲
                         Consumer offset
                         (messages retained)
```

| Aspect | RabbitMQ | Kafka |
|--------|----------|-------|
| **Model** | Message queue (push to consumer) | Distributed log (pull by consumer) |
| **Protocol** | AMQP 0-9-1 | Custom binary protocol |
| **Message lifetime** | Removed after ack | Retained (time/size-based) |
| **Routing** | Flexible (exchange types, bindings) | Partition-based (key hashing) |
| **Consumer model** | Push-based with prefetch | Pull-based with offset tracking |
| **Ordering** | Per-queue FIFO | Per-partition FIFO |
| **Replay** | Not supported (except streams) | Native (seek to any offset) |
| **Throughput** | ~50K–100K msg/sec per node | ~500K–1M+ msg/sec per node |
| **Latency** | Lower (sub-millisecond possible) | Higher (batching-optimized) |
| **Scaling** | Add consumers to queues | Add partitions to topics |
| **Ecosystem** | Plugins, multi-protocol | Streams, Connect, Schema Registry |

### When to Choose RabbitMQ

- **Task queues and work distribution** — Route jobs to workers with acknowledgment-based processing
- **Complex routing requirements** — Topic exchanges, header-based routing, exchange-to-exchange bindings
- **Request/reply (RPC) patterns** — Built-in correlation ID and reply-to support
- **Multi-protocol environments** — AMQP, MQTT, STOMP support via plugins
- **Low-latency messaging** — When sub-millisecond delivery matters more than throughput
- **Transient messages** — Fire-and-forget patterns where durability is not required
- **Priority queues** — When messages need to be processed in priority order
- **Existing AMQP ecosystem** — When AMQP compatibility is a requirement

### When to Choose Kafka

- **Event streaming and replay** — Consumers need to re-read historical events
- **High-throughput ingestion** — Millions of events per second
- **Event sourcing and CQRS** — Append-only log as the source of truth
- **Stream processing** — Kafka Streams or ksqlDB for real-time transformations
- **Log aggregation** — Centralized log collection from many producers
- **Long-term retention** — Keep events for days, weeks, or indefinitely
- **Fan-out to many consumers** — Multiple consumer groups read the same topic independently
- **Exactly-once semantics** — Transactional producers + consumer read-committed mode

```
Decision Guide
──────────────

  Need complex routing logic?
    YES ──▶ RabbitMQ (exchanges, bindings, routing keys)
    NO  ──▶ Continue

  Need event replay / log retention?
    YES ──▶ Kafka (or RabbitMQ Streams for simpler cases)
    NO  ──▶ Continue

  Need > 500K msg/sec throughput?
    YES ──▶ Kafka (designed for high throughput)
    NO  ──▶ Continue

  Need task queue with ack-based processing?
    YES ──▶ RabbitMQ (built-in work queue semantics)
    NO  ──▶ Continue

  Need multi-protocol (AMQP + MQTT + STOMP)?
    YES ──▶ RabbitMQ (plugin ecosystem)
    NO  ──▶ Either works — choose based on team expertise
```

---

## Next Steps

Continue your messaging learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | Messaging Fundamentals | Core messaging concepts, delivery guarantees, event-driven architecture |
| [01-APACHE-KAFKA.md](01-APACHE-KAFKA.md) | Apache Kafka | Distributed event streaming, topics, partitions, consumer groups |
| [04-PATTERNS.md](04-PATTERNS.md) | Messaging Patterns | Saga, outbox, CQRS, event sourcing, dead-letter queues |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial RabbitMQ documentation |
