# Messaging Reliability

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Delivery Guarantees](#delivery-guarantees)
   - [At-Most-Once Delivery](#at-most-once-delivery)
   - [At-Least-Once Delivery](#at-least-once-delivery)
   - [Exactly-Once Semantics](#exactly-once-semantics)
   - [Guarantee Comparison](#guarantee-comparison)
3. [Idempotency](#idempotency)
   - [Idempotent Consumers](#idempotent-consumers)
   - [Idempotency Keys](#idempotency-keys)
   - [Deduplication Strategies](#deduplication-strategies)
4. [Message Ordering](#message-ordering)
   - [Partition Ordering](#partition-ordering)
   - [Total Ordering](#total-ordering)
   - [Causal Ordering](#causal-ordering)
   - [Out-of-Order Handling](#out-of-order-handling)
5. [Acknowledgment Strategies](#acknowledgment-strategies)
   - [Auto Acknowledgment](#auto-acknowledgment)
   - [Manual Acknowledgment](#manual-acknowledgment)
   - [Negative Acknowledgment](#negative-acknowledgment)
   - [Acknowledgment Timeout](#acknowledgment-timeout)
6. [Retry Strategies](#retry-strategies)
   - [Immediate Retry](#immediate-retry)
   - [Exponential Backoff](#exponential-backoff)
   - [Jitter](#jitter)
   - [Retry Budgets](#retry-budgets)
   - [Retry Storms](#retry-storms)
7. [Dead Letter Handling](#dead-letter-handling)
   - [DLQ Configuration](#dlq-configuration)
   - [Poison Pill Messages](#poison-pill-messages)
   - [DLQ Monitoring](#dlq-monitoring)
   - [Reprocessing](#reprocessing)
8. [Exactly-Once Processing](#exactly-once-processing)
   - [Transactional Producers](#transactional-producers)
   - [Idempotent Consumers Revisited](#idempotent-consumers-revisited)
   - [End-to-End Exactly-Once](#end-to-end-exactly-once)
9. [Durability](#durability)
   - [Message Persistence](#message-persistence)
   - [Replication](#replication)
   - [Fsync and Flush Policies](#fsync-and-flush-policies)
   - [Disaster Recovery](#disaster-recovery)
10. [Error Handling Patterns](#error-handling-patterns)
    - [Circuit Breakers](#circuit-breakers)
    - [Bulkheads](#bulkheads)
    - [Timeout Patterns](#timeout-patterns)
11. [Testing Reliability](#testing-reliability)
    - [Chaos Testing](#chaos-testing)
    - [Failure Injection](#failure-injection)
    - [Message Loss Simulation](#message-loss-simulation)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

Reliability is the cornerstone of any production messaging system. When services communicate asynchronously, messages can be lost, duplicated, or delivered out of order due to network partitions, broker failures, consumer crashes, or resource exhaustion. Building reliable messaging requires understanding the guarantees each component provides and the patterns that compensate for their limitations.

This guide covers the full spectrum of reliability concerns — from delivery guarantees and acknowledgment strategies to dead letter handling, durability, and chaos testing — giving you practical tools to build messaging systems that behave correctly even when individual components fail.

### Target Audience

- **Backend Developers** building services that produce and consume messages
- **Platform Engineers** operating message brokers and managing infrastructure
- **Software Architects** designing distributed systems with messaging backbones
- **SRE / DevOps Engineers** responsible for uptime, monitoring, and incident response

### Scope

- Delivery guarantee semantics and their trade-offs
- Idempotency patterns and deduplication techniques
- Message ordering guarantees across partitions and topics
- Acknowledgment, retry, and dead letter queue strategies
- Durability mechanisms including replication and persistence
- Error handling patterns adapted for messaging contexts
- Approaches to testing and validating reliability under failure

---

## Delivery Guarantees

Delivery guarantees define the contract between producer, broker, and consumer regarding how many times a message may be delivered. Every messaging system makes trade-offs between reliability, performance, and complexity.

```
┌──────────────────────────────────────────────────────────────┐
│                   Delivery Guarantee Spectrum                │
│                                                              │
│  ◀── Less Reliable                    More Reliable ──▶      │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ At-Most-Once │  │At-Least-Once │  │ Exactly-Once │       │
│  │              │  │              │  │              │       │
│  │ Fire & Forget│  │ Retry Until  │  │ Dedup + Ack  │       │
│  │ No Retry     │  │ Acknowledged │  │ Transactional│       │
│  │              │  │              │  │              │       │
│  │ May lose msgs│  │ May duplicate│  │ No loss/dups │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                              │
│  Throughput: HIGH       MEDIUM           LOW                 │
│  Complexity: LOW        LOW              HIGH                │
└──────────────────────────────────────────────────────────────┘
```

### At-Most-Once Delivery

Messages are sent once with no retry. If delivery fails, the message is lost. This is the simplest and fastest approach but is only appropriate when occasional data loss is acceptable.

**When to use:**

- Metrics and telemetry where individual data points are expendable
- Logging pipelines that tolerate gaps
- Real-time notifications where stale data has no value

**Implementation:** The producer sends and immediately forgets. The consumer acknowledges before processing (or auto-ack is enabled).

```
Producer                Broker               Consumer
   │                      │                     │
   │──── Send message ───▶│                     │
   │     (no ack wait)    │                     │
   │                      │──── Deliver ───────▶│
   │                      │     (auto-ack)      │
   │                      │                     │
   ✗ If broker fails,     ✗ If consumer crashes │
     message is lost        after ack, work lost│
```

### At-Least-Once Delivery

The producer retries until the broker acknowledges receipt. The consumer acknowledges only after successful processing. This guarantees no message loss but may produce duplicates.

**When to use:**

- Order processing, payment events, or any business-critical flow
- Event sourcing where every event must be captured
- Data pipelines feeding analytics or warehouses

**Implementation:** The producer waits for broker acknowledgment and retries on failure. The consumer processes first, then sends an explicit ack.

```
Producer                Broker               Consumer
   │                      │                     │
   │──── Send message ───▶│                     │
   │◀─── Ack ────────────│                     │
   │                      │──── Deliver ───────▶│
   │                      │                     │── Process
   │                      │                     │
   │                      │◀─── Ack ───────────│
   │                      │                     │
   │  On failure:         │  On consumer crash: │
   │  Retry send ──▶      │  Redeliver ──▶      │
   │  (may duplicate)     │  (may duplicate)    │
```

### Exactly-Once Semantics

Exactly-once ensures each message is processed once and only once. This is the most desirable guarantee but also the most difficult to achieve. It typically requires coordination between producer, broker, and consumer using transactions or idempotency mechanisms.

**When to use:**

- Financial transactions and billing systems
- Inventory management where double-counting is catastrophic
- State machines where duplicate transitions corrupt state

### Guarantee Comparison

| Guarantee | Message Loss | Duplicates | Complexity | Throughput | Use Case |
|-----------|-------------|------------|------------|------------|----------|
| At-most-once | Possible | None | Low | Highest | Metrics, logs |
| At-least-once | None | Possible | Medium | High | Events, orders |
| Exactly-once | None | None | High | Lower | Finance, billing |

---

## Idempotency

Idempotency means that processing a message multiple times produces the same result as processing it once. Since at-least-once delivery can produce duplicates, idempotent consumers are essential for correctness.

### Idempotent Consumers

An idempotent consumer can safely receive the same message more than once without side effects. This is the primary defense against duplicate delivery.

```
┌─────────────────────────────────────────────────────┐
│              Idempotent Consumer Pattern             │
│                                                     │
│  Message arrives                                    │
│       │                                             │
│       ▼                                             │
│  ┌─────────────────┐                                │
│  │ Extract         │                                │
│  │ Idempotency Key │                                │
│  └────────┬────────┘                                │
│           ▼                                         │
│  ┌─────────────────┐    YES   ┌───────────────┐    │
│  │ Already seen?   │────────▶ │ Skip / Return │    │
│  └────────┬────────┘         │ Previous Result│    │
│           │ NO                └───────────────┘    │
│           ▼                                         │
│  ┌─────────────────┐                                │
│  │ Process Message  │                                │
│  │ + Store Key      │                                │
│  │ (atomically)     │                                │
│  └─────────────────┘                                │
└─────────────────────────────────────────────────────┘
```

**Key principles:**

- **Use natural business keys** — order IDs, transaction IDs, or request IDs
- **Store the key atomically** with the side effect (same database transaction)
- **Return the same result** for duplicate requests without re-executing logic

### Idempotency Keys

An idempotency key is a unique identifier attached to each message that allows consumers to detect and discard duplicates.

| Key Strategy | Example | Pros | Cons |
|-------------|---------|------|------|
| Producer-assigned UUID | `msg-id: 550e8400-e29b` | Universally unique | No business meaning |
| Business entity ID | `order-id: ORD-12345` | Naturally idempotent | Scope limited to entity |
| Content hash | `sha256(payload)` | Detects true duplicates | Different messages may collide |
| Composite key | `user:42:action:pay:ref:99` | Fine-grained control | More complex to generate |

### Deduplication Strategies

- **In-memory cache** — Fast lookup using an LRU or time-based cache. Suitable for short deduplication windows but lost on restart.
- **Database-backed store** — Persist seen message IDs in a table. Durable across restarts but adds latency.
- **Broker-level deduplication** — Some brokers (e.g., Kafka with `enable.idempotence=true`) deduplicate at the producer level.
- **Bloom filters** — Space-efficient probabilistic check. Allows rare false positives (treating a new message as duplicate) but never false negatives.

```
┌──────────────────────────────────────────────────────────┐
│              Deduplication Window Strategies              │
│                                                          │
│  Time-based:                                             │
│  ┌────────────────────────────────────┐                  │
│  │  Keep IDs for last 24 hours       │                  │
│  │  ├── 0h ──── 12h ──── 24h ──▶    │                  │
│  │  │  (active)    (active)  (evict) │                  │
│  └────────────────────────────────────┘                  │
│                                                          │
│  Count-based:                                            │
│  ┌────────────────────────────────────┐                  │
│  │  Keep last 1,000,000 message IDs  │                  │
│  │  ├── newest ──────────▶ oldest    │                  │
│  │  │  (keep)              (evict)   │                  │
│  └────────────────────────────────────┘                  │
│                                                          │
│  Hybrid:                                                 │
│  ┌────────────────────────────────────┐                  │
│  │  Evict when EITHER limit reached  │                  │
│  │  ├── Time: 24h OR Count: 1M      │                  │
│  │  │  (whichever comes first)       │                  │
│  └────────────────────────────────────┘                  │
└──────────────────────────────────────────────────────────┘
```

---

## Message Ordering

Different applications require different ordering guarantees. Stronger ordering typically comes at the cost of throughput and scalability.

### Partition Ordering

Most distributed brokers (Kafka, Pulsar) guarantee order within a single partition but not across partitions. Messages with the same partition key are routed to the same partition and consumed in order.

```
Producer sends: A1, B1, A2, B2, A3

Partition Key: "A" ──▶ Partition 0: [A1, A2, A3]  (in order)
Partition Key: "B" ──▶ Partition 1: [B1, B2]       (in order)

Consumer Group:
  Consumer-1 ◀── Partition 0: reads A1, A2, A3 sequentially
  Consumer-2 ◀── Partition 1: reads B1, B2 sequentially

✓ Order guaranteed WITHIN each partition
✗ No ordering guarantee ACROSS partitions
```

**Best practice:** Choose a partition key that groups related messages — e.g., customer ID, order ID, or session ID — so that all events for one entity arrive in order.

### Total Ordering

Total ordering guarantees that all consumers see every message in the same global sequence. This is achievable with a single partition or a consensus-based log but severely limits throughput.

| Approach | Ordering | Throughput | Availability |
|----------|----------|------------|-------------|
| Single partition | Total | Low (one writer) | Limited |
| Consensus log (Raft/Paxos) | Total | Medium | High |
| Multiple partitions | Partition only | High | High |

### Causal Ordering

Causal ordering ensures that if message B was produced after the producer observed message A, then every consumer sees A before B. This is weaker than total ordering but stronger than partition ordering.

```
┌────────────────────────────────────────────────┐
│              Causal Ordering Example            │
│                                                │
│  Service X publishes:  "Order Created" (A)     │
│       │                                        │
│       ▼                                        │
│  Service Y observes A, then publishes:         │
│       "Payment Initiated" (B)                  │
│       │                                        │
│       ▼                                        │
│  Consumer Z must see A before B                │
│  (because B was causally dependent on A)       │
│                                                │
│  Implementation:                               │
│  ┌──────┐    ┌──────┐    ┌──────┐             │
│  │ Msg A│───▶│ Msg B│───▶│ Msg C│             │
│  │ v=1  │    │ v=2  │    │ v=3  │             │
│  │ dep=[]│   │dep=[1]│   │dep=[2]│            │
│  └──────┘    └──────┘    └──────┘             │
│                                                │
│  Each message carries a vector clock or        │
│  dependency list to enforce causal order.      │
└────────────────────────────────────────────────┘
```

### Out-of-Order Handling

When strict ordering cannot be guaranteed, consumers must handle out-of-order messages gracefully.

**Strategies:**

- **Sequence numbers** — Attach a monotonically increasing sequence number. Buffer messages and process in order.
- **Reorder buffer** — Hold messages in a buffer for a configurable window, sorting before processing.
- **Last-write-wins** — Use timestamps to resolve conflicts; only apply the latest version.
- **Version checks** — Reject updates with a version older than the current state.

```
Incoming:    [Msg-3] [Msg-1] [Msg-5] [Msg-2] [Msg-4]

Reorder Buffer (window = 3):
  Step 1: buffer [3]           → waiting for 1
  Step 2: buffer [3, 1]       → emit 1, waiting for 2
  Step 3: buffer [3, 5]       → waiting for 2
  Step 4: buffer [3, 5, 2]    → emit 2, emit 3, waiting for 4
  Step 5: buffer [5, 4]       → emit 4, emit 5

Output:      [Msg-1] [Msg-2] [Msg-3] [Msg-4] [Msg-5]  ✓
```

---

## Acknowledgment Strategies

Acknowledgment (ack) determines when a broker considers a message successfully delivered and can remove it from the queue or advance the consumer offset.

### Auto Acknowledgment

The broker considers the message acknowledged as soon as it is delivered to the consumer, before processing completes.

- **Pros:** Highest throughput, simplest configuration
- **Cons:** Message is lost if the consumer crashes during processing
- **Use when:** Message loss is acceptable (metrics, logs)

### Manual Acknowledgment

The consumer explicitly sends an acknowledgment after successfully processing the message.

```
┌──────────┐         ┌────────┐         ┌──────────┐
│  Broker  │         │Consumer│         │ Database │
└────┬─────┘         └───┬────┘         └────┬─────┘
     │                    │                   │
     │── Deliver msg ───▶│                   │
     │                    │── Write data ───▶│
     │                    │◀── DB Commit ────│
     │◀── Manual Ack ────│                   │
     │                    │                   │
     │  Message removed   │                   │
     │  from queue        │                   │
     └────────────────────┘                   │
```

- **Pros:** No message loss — unacked messages are redelivered
- **Cons:** Slower; risk of duplicate delivery if ack fails after processing
- **Use when:** Every message must be processed (orders, payments)

### Negative Acknowledgment

A negative ack (nack) tells the broker that the consumer could not process the message. The broker then redelivers it (possibly to another consumer) or routes it to a dead letter queue.

| Nack Behavior | Description | When to Use |
|--------------|-------------|-------------|
| Requeue | Return message to the front of the queue | Transient failure (DB timeout) |
| Requeue to back | Return message to the end of the queue | Avoid blocking other messages |
| Reject (no requeue) | Discard or route to DLQ | Permanent failure (bad format) |

### Acknowledgment Timeout

If the consumer neither acks nor nacks within a configured timeout, the broker assumes failure and redelivers the message.

```
┌───────────────────────────────────────────────────┐
│             Ack Timeout Flow                      │
│                                                   │
│  Broker delivers message                          │
│       │                                           │
│       ├── Start timeout timer (e.g., 30s)        │
│       │                                           │
│       │   Consumer processing...                  │
│       │                                           │
│       ├── Case 1: Ack received within 30s        │
│       │   └── ✓ Message complete                 │
│       │                                           │
│       ├── Case 2: Nack received within 30s       │
│       │   └── ✗ Redeliver or DLQ                 │
│       │                                           │
│       └── Case 3: No response after 30s          │
│           └── ✗ Timeout → Redeliver              │
│               (consumer presumed dead)            │
└───────────────────────────────────────────────────┘
```

**Tuning guidance:**

- Set timeout longer than your slowest expected processing time
- Include headroom for garbage collection pauses and downstream latency
- Too short: false timeouts cause duplicates; too long: slow failure recovery

---

## Retry Strategies

Retries compensate for transient failures but must be carefully designed to avoid cascading problems.

### Immediate Retry

Retry the failed operation instantly, with no delay. Suitable only when the failure is extremely transient (e.g., momentary network blip).

- **Risk:** If the downstream system is overloaded, immediate retries amplify the load.
- **Limit:** Cap at 1–2 immediate retries before switching to backoff.

### Exponential Backoff

Increase the delay between retries exponentially, giving the failing system time to recover.

```
Attempt   Delay        Cumulative Wait
───────   ─────        ───────────────
  1       1s           1s
  2       2s           3s
  3       4s           7s
  4       8s           15s
  5       16s          31s
  6       32s          63s    (cap here)
  7       32s          95s    (max delay reached)

Formula: delay = min(base * 2^attempt, max_delay)
         base = 1s, max_delay = 32s
```

### Jitter

Add randomness to the backoff delay to prevent synchronized retries from multiple consumers (the "thundering herd" problem).

| Jitter Strategy | Formula | Behavior |
|----------------|---------|----------|
| Full jitter | `random(0, base * 2^attempt)` | Widest spread, best decorrelation |
| Equal jitter | `delay/2 + random(0, delay/2)` | Moderate spread, bounded minimum |
| Decorrelated | `random(base, previous_delay * 3)` | Adaptive, good for bursty loads |

```
Without Jitter (3 consumers retry together):
  Time ──▶  0s     2s     4s     8s
  C1:       ✗──────✗──────✗──────✗
  C2:       ✗──────✗──────✗──────✗    ← all hit at same time
  C3:       ✗──────✗──────✗──────✗

With Full Jitter:
  Time ──▶  0s     2s     4s     8s
  C1:       ✗───✗────────✗──────────✗
  C2:       ✗──────✗──✗─────────✗      ← spread across time
  C3:       ✗────────✗───────✗──────✗
```

### Retry Budgets

A retry budget caps the total number or rate of retries to prevent runaway retry amplification.

- **Per-message budget** — Each message gets a maximum number of retry attempts (e.g., 5). After exhausting the budget, route to DLQ.
- **Global retry rate** — Limit the overall retry rate to a percentage of normal traffic (e.g., retries must not exceed 10% of total requests).
- **Time-based budget** — Stop retrying after a total elapsed time (e.g., 5 minutes from first attempt).

### Retry Storms

A retry storm occurs when many producers or consumers retry simultaneously, overwhelming the broker or downstream services.

```
┌────────────────────────────────────────────────────────┐
│                   Retry Storm Cascade                  │
│                                                        │
│  Step 1: Downstream DB becomes slow                    │
│       │                                                │
│       ▼                                                │
│  Step 2: Consumers start timing out                    │
│       │                                                │
│       ▼                                                │
│  Step 3: Broker redelivers unacked messages            │
│       │                                                │
│       ▼                                                │
│  Step 4: Consumers retry + receive redeliveries        │
│       │        (load doubles, then quadruples)         │
│       ▼                                                │
│  Step 5: DB collapses under amplified load             │
│       │                                                │
│       ▼                                                │
│  Step 6: Total system failure                          │
│                                                        │
│  Prevention:                                           │
│  ✓ Exponential backoff with jitter                     │
│  ✓ Retry budgets (max 10% retry rate)                  │
│  ✓ Circuit breakers on downstream calls                │
│  ✓ Back-pressure: stop consuming when overloaded       │
└────────────────────────────────────────────────────────┘
```

---

## Dead Letter Handling

A Dead Letter Queue (DLQ) captures messages that cannot be processed after exhausting all retry attempts. Without a DLQ, poison pill messages can block an entire queue.

### DLQ Configuration

```
┌──────────┐       ┌───────────┐       ┌──────────┐
│ Producer │──────▶│Main Queue │──────▶│ Consumer │
└──────────┘       └─────┬─────┘       └────┬─────┘
                         │                   │
                         │   After N failed  │
                         │   attempts        │
                         ▼                   │
                   ┌───────────┐             │
                   │   DLQ     │◀────────────┘
                   │           │  (reject / max retries)
                   └─────┬─────┘
                         │
                         ▼
                   ┌───────────┐
                   │  Alert +  │
                   │  Monitor  │
                   └───────────┘
```

| Configuration | Description | Typical Value |
|--------------|-------------|---------------|
| Max delivery attempts | Retries before routing to DLQ | 3–5 |
| DLQ retention | How long messages are kept in DLQ | 7–14 days |
| DLQ max size | Maximum number of messages in DLQ | Varies by system |
| Original metadata | Preserve original headers, timestamp, error reason | Always enabled |

### Poison Pill Messages

A poison pill is a message that consistently causes consumer failure — malformed payload, schema mismatch, or a bug triggered by specific data. Without DLQ handling, a poison pill blocks all messages behind it in an ordered queue.

**Detection strategies:**

- **Delivery count header** — Track how many times a message has been delivered. Route to DLQ after a threshold.
- **Error classification** — Distinguish transient errors (retry) from permanent errors (DLQ immediately).
- **Schema validation** — Validate messages before processing. Reject invalid messages to DLQ without retrying.

### DLQ Monitoring

- **Alert on DLQ depth** — Any message in the DLQ indicates a problem. Alert when depth > 0.
- **Track DLQ ingestion rate** — A sudden spike suggests a systemic issue (bad deployment, schema change).
- **Age of oldest message** — Stale DLQ messages indicate neglected failures.
- **Error categorization** — Group DLQ messages by error type to prioritize fixes.

### Reprocessing

After fixing the root cause, DLQ messages should be replayed back into the main queue.

```
Reprocessing Workflow:

  1. Investigate ──▶ 2. Fix Root Cause ──▶ 3. Replay
       │                    │                    │
       ▼                    ▼                    ▼
  Read DLQ messages    Deploy code fix      Move messages from
  Identify error       or fix data issue    DLQ back to main queue
  pattern                                   (with monitoring)

  4. Verify ──▶ 5. Clean Up
       │              │
       ▼              ▼
  Confirm all      Purge successfully
  messages          replayed messages
  processed         from DLQ
```

**Best practices:**

- Replay in small batches with monitoring, not all at once
- Preserve original message metadata for audit trails
- Consider a staging queue for replay validation before hitting production consumers
- Automate replay for known transient failure categories

---

## Exactly-Once Processing

True exactly-once processing requires coordination across the entire pipeline — producer, broker, and consumer must all participate.

### Transactional Producers

A transactional producer groups multiple messages into an atomic batch. Either all messages in the transaction are committed to the broker, or none are.

```
┌──────────────────────────────────────────────────────┐
│           Transactional Producer Flow                │
│                                                      │
│  Producer                        Broker              │
│     │                              │                 │
│     │── beginTransaction() ──────▶│                 │
│     │                              │                 │
│     │── send(topic-A, msg-1) ───▶│ (buffered)      │
│     │── send(topic-B, msg-2) ───▶│ (buffered)      │
│     │── send(topic-A, msg-3) ───▶│ (buffered)      │
│     │                              │                 │
│     │── commitTransaction() ────▶│                 │
│     │                              │── All 3 msgs   │
│     │◀── Commit OK ──────────────│   visible to    │
│     │                              │   consumers    │
│     │                              │                 │
│     │  On failure:                 │                 │
│     │── abortTransaction() ─────▶│                 │
│     │                              │── All 3 msgs   │
│     │                              │   discarded    │
└──────────────────────────────────────────────────────┘
```

### Idempotent Consumers Revisited

Even with transactional producers, the consumer side must be idempotent. The consume-process-commit cycle has a window where the consumer may crash after processing but before committing the offset.

**Pattern: Transactional outbox with offset storage**

1. Begin database transaction
2. Process message and write results to database
3. Store the consumer offset in the same database transaction
4. Commit database transaction
5. The next read starts from the stored offset — no reprocessing needed

### End-to-End Exactly-Once

End-to-end exactly-once combines transactional producers, broker-side deduplication, and idempotent consumers.

```
┌──────────────────────────────────────────────────────────┐
│            End-to-End Exactly-Once Pipeline               │
│                                                          │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────┐   │
│  │ Producer │    │  Broker  │    │    Consumer       │   │
│  │          │    │          │    │                    │   │
│  │ Txn +    │───▶│ Dedup +  │───▶│ Idempotent +     │   │
│  │ Sequence │    │ Txn Log  │    │ Offset-in-DB     │   │
│  │ Numbers  │    │          │    │                    │   │
│  └──────────┘    └──────────┘    └──────────────────┘   │
│                                                          │
│  Producer Side:          Broker Side:                    │
│  - Idempotent producer   - Sequence number tracking      │
│  - Transaction API       - Transaction coordinator       │
│  - Retry safely          - Atomic multi-partition write   │
│                                                          │
│  Consumer Side:                                          │
│  - Read committed isolation                              │
│  - Process + offset in one DB transaction                │
│  - Exactly-once state updates                            │
└──────────────────────────────────────────────────────────┘
```

---

## Durability

Durability ensures messages survive broker restarts, disk failures, and data center outages.

### Message Persistence

Messages can be stored in memory (fast, volatile) or on disk (slower, durable). Production systems should always persist messages to disk.

| Storage Mode | Latency | Durability | Use Case |
|-------------|---------|------------|----------|
| In-memory only | ~μs | None (lost on restart) | Caching, ephemeral signals |
| Async disk write | ~ms | Survives restart, may lose recent msgs | High-throughput logs |
| Sync disk write (fsync) | ~ms–10ms | Survives restart + power loss | Financial, transactional |
| Replicated + sync disk | ~10ms+ | Survives node + disk failure | Mission-critical systems |

### Replication

Replication copies messages to multiple broker nodes so that data survives individual node failures.

```
┌───────────────────────────────────────────────────┐
│          Replication (factor = 3)                  │
│                                                   │
│  Producer ──▶ Broker 1 (Leader)                   │
│                  │                                │
│                  ├──▶ Broker 2 (Follower)         │
│                  │       │                        │
│                  └──▶ Broker 3 (Follower)         │
│                          │                        │
│  Write acknowledged when:                         │
│                                                   │
│  acks=0  : Immediately (no wait)                  │
│  acks=1  : Leader writes to local log             │
│  acks=all: Leader + all in-sync replicas write    │
│                                                   │
│  If Leader fails:                                 │
│  ┌──────────┐                                     │
│  │ Broker 2 │ ← promoted to Leader                │
│  └──────────┘                                     │
│  Broker 3 follows new Leader                      │
│  No data loss (if acks=all was used)              │
└───────────────────────────────────────────────────┘
```

### Fsync and Flush Policies

`fsync` forces the operating system to flush data from its page cache to physical disk. Without fsync, data acknowledged by the broker may still be in volatile memory.

| Policy | Behavior | Trade-off |
|--------|----------|-----------|
| fsync every message | Flush after each write | Safest, slowest |
| fsync every N messages | Batch flush (e.g., every 1000) | Balanced |
| fsync on interval | Flush every T seconds (e.g., 1s) | Good throughput |
| OS-managed flush | Let OS decide when to flush | Fastest, least durable |

**Recommendation:** For most systems, rely on replication for durability and use interval-based fsync. Reserve per-message fsync for single-node deployments or regulatory requirements.

### Disaster Recovery

- **Cross-datacenter replication** — Mirror topics to a secondary datacenter using tools like Kafka MirrorMaker or broker-native geo-replication.
- **Backup and restore** — Periodically snapshot topic data and consumer offsets.
- **RPO and RTO targets** — Define how much data loss (Recovery Point Objective) and downtime (Recovery Time Objective) are acceptable.

```
┌──────────────────────────────────────────────────┐
│         Cross-Datacenter Replication             │
│                                                  │
│  Datacenter A (Primary)    Datacenter B (DR)     │
│  ┌──────────────────┐     ┌──────────────────┐  │
│  │ Broker Cluster   │     │ Broker Cluster   │  │
│  │ ┌──────────────┐ │     │ ┌──────────────┐ │  │
│  │ │ Topic: orders│─┼────▶│ │ Topic: orders│ │  │
│  │ └──────────────┘ │     │ └──────────────┘ │  │
│  │ ┌──────────────┐ │     │ ┌──────────────┐ │  │
│  │ │ Topic: events│─┼────▶│ │ Topic: events│ │  │
│  │ └──────────────┘ │     │ └──────────────┘ │  │
│  └──────────────────┘     └──────────────────┘  │
│                                                  │
│  Replication Lag: 50ms – 5s (typical)            │
│  Failover: Manual or automated with health checks│
└──────────────────────────────────────────────────┘
```

---

## Error Handling Patterns

Messaging systems benefit from the same resilience patterns used in synchronous systems, adapted for asynchronous contexts.

### Circuit Breakers

A circuit breaker prevents a consumer from repeatedly calling a failing downstream service. After a threshold of failures, the circuit "opens" and calls are short-circuited.

```
┌─────────────────────────────────────────────────────┐
│              Circuit Breaker States                  │
│                                                     │
│  ┌────────┐   Failure threshold   ┌────────┐       │
│  │ CLOSED │ ─────────────────────▶│  OPEN  │       │
│  │        │   reached             │        │       │
│  │ Normal │                       │ Reject │       │
│  │ flow   │                       │  all   │       │
│  └────┬───┘                       └───┬────┘       │
│       ▲                               │            │
│       │   Success                     │ Timeout    │
│       │                               ▼            │
│       │                         ┌───────────┐      │
│       └─────────────────────────│ HALF-OPEN │      │
│           Reset on success      │           │      │
│                                 │ Allow one │      │
│                                 │ test call │      │
│                                 └───────────┘      │
│                                                     │
│  In Messaging Context:                              │
│  CLOSED   → Process messages normally               │
│  OPEN     → Nack messages (requeue for later)       │
│  HALF-OPEN→ Try one message, if OK → close circuit  │
└─────────────────────────────────────────────────────┘
```

**Configuration parameters:**

| Parameter | Description | Typical Value |
|-----------|-------------|---------------|
| Failure threshold | Consecutive failures to open circuit | 5 |
| Timeout duration | Time in open state before half-open | 30–60s |
| Success threshold | Successes in half-open to close circuit | 2–3 |
| Monitoring window | Time window for counting failures | 60s |

### Bulkheads

Bulkheads isolate different message processing pipelines so that a failure in one does not starve resources from others.

```
┌────────────────────────────────────────────────────┐
│              Bulkhead Isolation                     │
│                                                    │
│  Without Bulkheads:                                │
│  ┌────────────────────────────────────┐            │
│  │        Shared Thread Pool (20)     │            │
│  │  Orders ──▶ ████████████████       │            │
│  │  Emails ──▶ ████                   │ ← starved  │
│  │  Analytics──▶                      │ ← starved  │
│  └────────────────────────────────────┘            │
│                                                    │
│  With Bulkheads:                                   │
│  ┌──────────────────┐                              │
│  │ Orders Pool (10) │                              │
│  │ ████████████████ │  ← isolated                  │
│  └──────────────────┘                              │
│  ┌──────────────────┐                              │
│  │ Emails Pool (5)  │                              │
│  │ █████            │  ← isolated                  │
│  └──────────────────┘                              │
│  ┌──────────────────┐                              │
│  │ Analytics Pool(5)│                              │
│  │ █████            │  ← isolated                  │
│  └──────────────────┘                              │
└────────────────────────────────────────────────────┘
```

**Implementation approaches:**

- **Separate consumer groups** per message type
- **Dedicated thread pools** for different processing pipelines
- **Separate broker clusters** for critical vs. non-critical traffic
- **Queue-level isolation** with independent scaling

### Timeout Patterns

Timeouts prevent consumers from waiting indefinitely for downstream responses.

| Timeout Type | Description | Guidance |
|-------------|-------------|----------|
| Processing timeout | Max time to process one message | Set above P99 processing latency |
| Connection timeout | Max time to establish a connection | 1–5s typical |
| Socket read timeout | Max time waiting for data on an open connection | 5–30s typical |
| End-to-end timeout | Max total time for the entire message pipeline | Sum of all stage timeouts |

**Best practice:** Combine timeouts with circuit breakers. A timeout triggers a failure count; enough timeouts open the circuit, preventing further calls until the downstream recovers.

---

## Testing Reliability

Reliability claims are only as strong as the tests that verify them. Systematic failure injection reveals weaknesses before production does.

### Chaos Testing

Chaos testing deliberately introduces failures into a running system to verify that reliability mechanisms work correctly.

```
┌──────────────────────────────────────────────────────┐
│              Chaos Testing Approach                   │
│                                                      │
│  1. Define steady state                              │
│     └── "All messages processed within 5s,           │
│          zero messages in DLQ"                       │
│                                                      │
│  2. Hypothesize                                      │
│     └── "If we kill one broker, steady state holds"  │
│                                                      │
│  3. Inject failure                                   │
│     └── Kill broker node, partition network,         │
│          exhaust disk, spike CPU                     │
│                                                      │
│  4. Observe                                          │
│     └── Monitor lag, DLQ depth, error rates,         │
│          processing latency                          │
│                                                      │
│  5. Verify or learn                                  │
│     └── Did steady state hold? If not, improve.      │
└──────────────────────────────────────────────────────┘
```

**Areas to test:**

- Broker node failure during message production
- Consumer crash mid-processing (before ack)
- Network partition between producer and broker
- Disk full on broker nodes
- Clock skew between nodes

### Failure Injection

Failure injection is the controlled introduction of specific faults into the messaging pipeline.

| Injection Type | Method | What It Tests |
|---------------|--------|---------------|
| Broker kill | Stop broker process | Failover, replication, producer retry |
| Network partition | iptables / tc rules | Split-brain handling, timeout behavior |
| Slow consumer | Add artificial delay | Back-pressure, queue depth monitoring |
| Corrupt message | Inject malformed payload | Schema validation, DLQ routing |
| Disk failure | Fill disk / mount read-only | Persistence, alerts, graceful degradation |
| Consumer crash | Kill consumer mid-batch | Redelivery, offset management, idempotency |

### Message Loss Simulation

Validate that your system detects and recovers from message loss.

```
┌──────────────────────────────────────────────────────┐
│          Message Loss Detection Test                 │
│                                                      │
│  Producer                                            │
│     │                                                │
│     │── Send 10,000 messages ──▶ Broker              │
│     │   (each with sequence number)                  │
│     │                                                │
│     │                           Broker               │
│     │                              │                 │
│     │           Inject: drop 1% ──┤                  │
│     │           of messages        │                 │
│     │                              │──▶ Consumer     │
│     │                                    │           │
│     │                              Verify:           │
│     │                              - Received 9,900? │
│     │                              - Detected gaps?  │
│     │                              - Alerted?        │
│     │                              - Recovered?      │
│                                                      │
│  Success Criteria:                                   │
│  ✓ Gap detection fires within 30s                    │
│  ✓ Alert triggers to on-call                         │
│  ✓ Missing messages identified by sequence number    │
│  ✓ Recovery procedure replays missing messages       │
└──────────────────────────────────────────────────────┘
```

**Testing checklist:**

- [ ] Producer retries work correctly under broker failure
- [ ] Consumer idempotency handles duplicate delivery
- [ ] DLQ captures poison pill messages
- [ ] Circuit breaker opens on downstream failure
- [ ] Replication prevents data loss on single-node failure
- [ ] Monitoring alerts fire on message loss or lag
- [ ] Replay from DLQ succeeds after root cause fix

---

## Next Steps

Continue your messaging learning journey:

| File | Topic | Description |
|------|-------|-------------|
| [00-OVERVIEW.md](00-OVERVIEW.md) | Messaging Overview | Foundational concepts, patterns, and broker comparison |
| [01-APACHE-KAFKA.md](01-APACHE-KAFKA.md) | Apache Kafka | Brokers, topics, partitions, producers, consumers |
| [02-RABBITMQ.md](02-RABBITMQ.md) | RabbitMQ | AMQP model, exchanges, queues, reliability |
| [03-AWS-MESSAGING.md](03-AWS-MESSAGING.md) | AWS Messaging | SQS, SNS, EventBridge, Kinesis |
| [04-AZURE-MESSAGING.md](04-AZURE-MESSAGING.md) | Azure Messaging | Service Bus, Event Hubs, Event Grid |
| [05-SCHEMA-MANAGEMENT.md](05-SCHEMA-MANAGEMENT.md) | Schema Management | Schema registries, evolution, compatibility |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial messaging reliability documentation |
