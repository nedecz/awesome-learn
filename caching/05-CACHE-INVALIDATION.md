# Cache Invalidation

## Table of Contents

1. [Overview](#overview)
   - [The Hardest Problem in Caching](#the-hardest-problem-in-caching)
   - [Why Invalidation Is Difficult](#why-invalidation-is-difficult)
   - [Invalidation vs Eviction](#invalidation-vs-eviction)
   - [Core Invalidation Strategies at a Glance](#core-invalidation-strategies-at-a-glance)
2. [The Consistency Problem](#the-consistency-problem)
   - [Stale Data Risks](#stale-data-risks)
   - [Eventual Consistency vs Strict Consistency](#eventual-consistency-vs-strict-consistency)
   - [The Window of Inconsistency](#the-window-of-inconsistency)
   - [Real-World Impact of Serving Stale Data](#real-world-impact-of-serving-stale-data)
3. [TTL-Based Invalidation](#ttl-based-invalidation)
   - [Time-to-Live Fundamentals](#time-to-live-fundamentals)
   - [Choosing TTL Values by Data Type](#choosing-ttl-values-by-data-type)
   - [TTL Jitter to Prevent Thundering Herd](#ttl-jitter-to-prevent-thundering-herd)
   - [Short vs Long TTL Trade-Offs](#short-vs-long-ttl-trade-offs)
   - [Sliding TTL](#sliding-ttl)
   - [Adaptive TTL](#adaptive-ttl)
4. [Event-Driven Invalidation](#event-driven-invalidation)
   - [Message Queue Broadcast Pattern](#message-queue-broadcast-pattern)
   - [CDC with Debezium](#cdc-with-debezium)
   - [Database Triggers](#database-triggers)
   - [Pub/Sub Invalidation](#pubsub-invalidation)
   - [Event-Driven Invalidation Diagram](#event-driven-invalidation-diagram)
5. [Versioned Keys](#versioned-keys)
   - [Appending Version Numbers to Keys](#appending-version-numbers-to-keys)
   - [Hash-Based Cache Keys](#hash-based-cache-keys)
   - [Schema Versioning for Rolling Deployments](#schema-versioning-for-rolling-deployments)
   - [Cache Key Namespacing](#cache-key-namespacing)
6. [Tag-Based Invalidation](#tag-based-invalidation)
   - [Grouping Cache Entries with Tags](#grouping-cache-entries-with-tags)
   - [Invalidating by Tag](#invalidating-by-tag)
   - [Implementation Approaches](#implementation-approaches)
7. [Delete vs Update Debate](#delete-vs-update-debate)
   - [When to Delete the Cache Entry](#when-to-delete-the-cache-entry)
   - [When to Update In-Place](#when-to-update-in-place)
   - [Race Conditions with Update](#race-conditions-with-update)
   - [Why Delete Is Usually Safer](#why-delete-is-usually-safer)
8. [Cache Stampede Prevention](#cache-stampede-prevention)
   - [The Thundering Herd Problem](#the-thundering-herd-problem)
   - [Mutex / Lock Approach](#mutex--lock-approach)
   - [Probabilistic Early Expiration (XFetch)](#probabilistic-early-expiration-xfetch)
   - [Request Coalescing / Single-Flight](#request-coalescing--single-flight)
   - [Stale-While-Revalidate](#stale-while-revalidate)
   - [Pre-Warming](#pre-warming)
9. [Multi-Layer Invalidation](#multi-layer-invalidation)
   - [L1 + L2 + CDN Architecture](#l1--l2--cdn-architecture)
   - [Propagation Delays](#propagation-delays)
   - [Consistency Challenges Across Layers](#consistency-challenges-across-layers)
   - [Multi-Layer Invalidation Diagram](#multi-layer-invalidation-diagram)
10. [Database-Driven Invalidation](#database-driven-invalidation)
    - [PostgreSQL LISTEN/NOTIFY](#postgresql-listennotify)
    - [MySQL Binlog](#mysql-binlog)
    - [MongoDB Change Streams](#mongodb-change-streams)
    - [Debezium CDC Pipeline](#debezium-cdc-pipeline)
11. [Invalidation Patterns Comparison](#invalidation-patterns-comparison)
    - [Comprehensive Comparison Table](#comprehensive-comparison-table)
    - [Choosing the Right Pattern](#choosing-the-right-pattern)
12. [Testing Cache Invalidation](#testing-cache-invalidation)
    - [Chaos Testing](#chaos-testing)
    - [Simulating Stale Data](#simulating-stale-data)
    - [Staleness Metrics and Monitoring](#staleness-metrics-and-monitoring)
    - [Integration Test Patterns](#integration-test-patterns)
13. [Real-World Case Studies](#real-world-case-studies)
    - [Facebook Memcache Invalidation](#facebook-memcache-invalidation)
    - [Netflix EVCache](#netflix-evcache)
14. [Version History](#version-history)

---

## Overview

### The Hardest Problem in Caching

> "There are only two hard things in Computer Science: cache invalidation
> and naming things." — Phil Karlton

This famous quote endures because it captures a genuine engineering truth. Adding a
cache is relatively easy — a simple key-value lookup in front of a slow data source.
**Knowing when to remove or refresh that cached data** is where the real complexity
lives. Every cached value is a bet that the underlying data will not change before
the cache entry expires. When that bet fails, your users see stale data — or worse,
make decisions based on it.

### Why Invalidation Is Difficult

Cache invalidation is difficult because it requires solving a **distributed coordination
problem**. The data source (database) and the cache are separate systems with no shared
transaction boundary. Any change in the source must somehow be reflected in the cache,
but the two systems cannot atomically agree on state.

```
  The Fundamental Tension
  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │   Source of Truth (DB)          Cache (Redis / CDN / etc.)   │
  │   ┌──────────────────┐         ┌──────────────────────┐     │
  │   │  price = $19.99  │──???──▶ │  price = $14.99      │     │
  │   │  (just updated)  │         │  (stale!)            │     │
  │   └──────────────────┘         └──────────────────────┘     │
  │                                                              │
  │   How does the cache learn about the update?                 │
  │   ● Wait for TTL to expire?        (stale window)           │
  │   ● Application tells it?          (coupling)               │
  │   ● Database pushes a notification?(infrastructure)         │
  │   ● Version the key?               (complexity)             │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
```

Key challenges:

| Challenge              | Description                                                    |
|------------------------|----------------------------------------------------------------|
| **No shared txn**      | DB commit and cache delete cannot be a single atomic operation |
| **Network partition**  | Invalidation message may be lost or delayed                    |
| **Race conditions**    | Concurrent read and write can re-populate stale data           |
| **Multiple consumers** | Many services cache the same data; all must be notified        |
| **Multi-layer caches** | L1 in-process, L2 distributed, CDN edge — each must be purged |
| **Ordering**           | Out-of-order invalidations can restore old values              |

### Invalidation vs Eviction

These terms are often confused but represent different mechanisms:

| Aspect          | Invalidation                              | Eviction                             |
|-----------------|-------------------------------------------|--------------------------------------|
| **Trigger**     | Data changed in the source of truth       | Cache is full or TTL expired         |
| **Intent**      | Correctness — remove stale data           | Capacity — make room for new data    |
| **Who decides** | Application logic or external event       | Cache engine (LRU, LFU, etc.)        |
| **Scope**       | Specific keys or groups of keys           | Whichever keys the policy selects    |
| **Urgency**     | Usually immediate                         | Lazy / background                    |

### Core Invalidation Strategies at a Glance

```
  ┌───────────────────────────────────────────────────────────────┐
  │               INVALIDATION STRATEGY SPECTRUM                  │
  │                                                               │
  │  Simple ◄──────────────────────────────────────────► Complex  │
  │                                                               │
  │  TTL-Based    Event-Driven    Versioned    Tag-Based    CDC   │
  │  ┌──────┐     ┌──────────┐   ┌────────┐  ┌────────┐  ┌───┐  │
  │  │Timer │     │Pub/Sub   │   │Key v2  │  │Group   │  │DB │  │
  │  │expire│     │notify    │   │Key v3  │  │purge   │  │log│  │
  │  └──────┘     └──────────┘   └────────┘  └────────┘  └───┘  │
  │                                                               │
  │  ● Passive      ● Active       ● Active    ● Active  ● Active│
  │  ● Eventual     ● Near-real    ● Instant   ● Instant ● Near  │
  │  ● Simple       ● Moderate     ● Moderate  ● Complex ● High  │
  └───────────────────────────────────────────────────────────────┘
```

---

## The Consistency Problem

### Stale Data Risks

Every cache creates a copy of data. The moment the source changes, that copy becomes
**stale**. Staleness is not always a problem — showing a blog post with a 5-minute-old
comment count is perfectly acceptable. But staleness in other contexts is dangerous:

- **E-commerce pricing** — A stale price might undercharge thousands of customers.
- **Inventory counts** — Overselling items that are out of stock.
- **Permissions / ACL** — A revoked user retains access until the cache refreshes.
- **Financial data** — Stale balances lead to overdrafts or incorrect risk calculations.
- **Feature flags** — A "kill switch" flag remains cached as "enabled" for minutes.

### Eventual Consistency vs Strict Consistency

```
  Strict Consistency                   Eventual Consistency
  ┌────────────────────┐               ┌────────────────────┐
  │ Write to DB        │               │ Write to DB        │
  │       │            │               │       │            │
  │       ▼            │               │       ▼            │
  │ Invalidate cache   │               │ (cache not updated │
  │ (synchronously)    │               │  immediately)      │
  │       │            │               │       │            │
  │       ▼            │               │       ▼            │
  │ Return to client   │               │ Return to client   │
  │                    │               │       ...          │
  │ ● Always fresh     │               │ TTL expires or     │
  │ ● Higher latency   │               │ event arrives      │
  │ ● Failure coupling │               │       │            │
  │                    │               │       ▼            │
  │                    │               │ Cache refreshed    │
  │                    │               │                    │
  │                    │               │ ● Briefly stale    │
  │                    │               │ ● Lower latency    │
  │                    │               │ ● Decoupled        │
  └────────────────────┘               └────────────────────┘
```

| Model                    | Guarantee                                      | Trade-off            |
|--------------------------|-------------------------------------------------|----------------------|
| **Strict consistency**   | Reads always see the latest write               | Higher write latency |
| **Eventual consistency** | Reads *eventually* converge to the latest write | Temporary staleness  |
| **Bounded staleness**    | Stale data is at most *N* seconds old           | Middle ground        |

Most systems choose **eventual consistency with a bounded staleness window** because
strict consistency across cache and database effectively eliminates the latency
benefit of caching.

### The Window of Inconsistency

The **window of inconsistency** is the time between a data change in the source and
the cache reflecting that change. Minimizing this window is the central goal of
cache invalidation.

```
  Timeline
  ─────────────────────────────────────────────────────────────
  t0          t1          t2          t3          t4
  │           │           │           │           │
  DB write    │           Invalidation │           Cache
  occurs      │           event sent  │           reflects
              │                       │           new value
              │◄─────────────────────►│
              │   Window of           │
              │   Inconsistency       │
              │   (stale reads here)  │
  ─────────────────────────────────────────────────────────────
```

Factors that widen this window:

- **TTL too long** — Cache entry survives well beyond data change
- **Network delays** — Invalidation message arrives late
- **Message loss** — Invalidation event dropped entirely
- **Processing lag** — Consumer backlog in event-driven systems
- **Multi-region** — Cross-region replication adds propagation time

### Real-World Impact of Serving Stale Data

| Domain          | Stale Data Scenario                                | Business Impact                        |
|-----------------|----------------------------------------------------|----------------------------------------|
| E-commerce      | Old price cached for a flash-sale item             | Revenue loss, customer complaints      |
| Social media    | Deleted post still visible from cache              | Policy violations, user trust erosion  |
| Banking         | Cached account balance after transfer              | Overdrafts, regulatory issues          |
| Healthcare      | Stale patient allergy data                         | Patient safety risk                    |
| Gaming          | Cached leaderboard showing wrong rankings          | Player frustration, support tickets    |
| Ride-sharing    | Driver location cached for 30 seconds              | Wrong ETAs, poor dispatch decisions    |
| Feature flags   | Kill switch still reads "enabled" from cache       | Continued exposure to broken feature   |

---

## TTL-Based Invalidation

### Time-to-Live Fundamentals

TTL (Time-to-Live) is the simplest invalidation mechanism. Each cache entry is stored
with an expiration timestamp. Once the TTL elapses, the entry is considered expired and
will be refreshed on the next access (lazy) or removed by a background process (active).

```python
# Redis example — set a key with a 300-second TTL
import redis

r = redis.Redis()
r.setex("product:42:price", 300, "19.99")

# After 300 seconds, GET returns None (key expired)
```

TTL is **passive invalidation** — the cache does not know whether the underlying data
has changed. It simply assumes that data older than the TTL *might* be stale.

### Choosing TTL Values by Data Type

There is no universal TTL. The right value depends on how frequently the data changes
and how tolerant your application is of staleness.

| Data Type                | Change Frequency | Staleness Tolerance | Suggested TTL     |
|--------------------------|------------------|---------------------|-------------------|
| Static assets (CSS, JS)  | Per deployment   | High                | 1 year + versioned URL |
| Site configuration       | Rarely           | Moderate            | 5–15 minutes      |
| Product catalog          | Few times/day    | Moderate            | 1–5 minutes       |
| Product price            | Frequently       | Low                 | 30–60 seconds     |
| User profile             | Occasionally     | Moderate            | 2–10 minutes      |
| Session data             | Every request    | None                | Sliding, 30 min   |
| Stock / inventory        | Real-time        | Very low            | 5–15 seconds      |
| Exchange rates           | Every minute     | Low                 | 30–60 seconds     |
| News feed / timeline     | Frequently       | Moderate            | 1–2 minutes       |
| DNS records              | Rarely           | High                | 1–24 hours        |
| Feature flags            | On deploy        | Low                 | 10–30 seconds     |
| Auth tokens / JWT        | On revocation    | None                | Match token expiry |

### TTL Jitter to Prevent Thundering Herd

When many cache entries share the same TTL and were populated at roughly the same time,
they all expire simultaneously. This creates a **thundering herd** — a burst of cache
misses that hammer the database at once.

**Solution: add random jitter to each TTL.**

```python
import random

BASE_TTL = 300  # 5 minutes
JITTER   = 60   # ± 60 seconds

def ttl_with_jitter():
    return BASE_TTL + random.randint(-JITTER, JITTER)

# Each key gets a TTL between 240 and 360 seconds
r.setex("product:42:price", ttl_with_jitter(), "19.99")
r.setex("product:43:price", ttl_with_jitter(), "29.99")
```

```
  Without Jitter                        With Jitter
  ┌──────────────────────┐              ┌──────────────────────┐
  │                      │              │                      │
  │  ████████████████    │              │  ██                  │
  │  (all expire at t=5m)│              │    ███               │
  │         │            │              │       ██             │
  │         ▼            │              │         ████         │
  │  ┌──────────────┐   │              │             ██       │
  │  │  STAMPEDE!   │   │              │  (spread over time)  │
  │  │  1000 misses │   │              │  smooth load on DB   │
  │  └──────────────┘   │              │                      │
  └──────────────────────┘              └──────────────────────┘
```

### Short vs Long TTL Trade-Offs

| Aspect              | Short TTL (seconds)                | Long TTL (minutes–hours)          |
|---------------------|------------------------------------|-----------------------------------|
| **Freshness**       | Data is almost always current      | Data can be significantly stale   |
| **Cache hit ratio** | Lower — more misses                | Higher — fewer misses             |
| **DB load**         | Higher — more regeneration         | Lower — less regeneration         |
| **Latency (p50)**   | Higher (more cold fetches)         | Lower (mostly cache hits)         |
| **Latency (p99)**   | More predictable                   | Occasional spike on expiry        |
| **Best for**        | High-change, low-staleness data    | Rarely changing, tolerant data    |

### Sliding TTL

A **sliding TTL** resets the expiration timer every time the entry is accessed. This
keeps frequently accessed ("hot") data alive in the cache while letting rarely accessed
("cold") data expire naturally.

```python
def get_with_sliding_ttl(key, ttl=300):
    value = r.get(key)
    if value is not None:
        r.expire(key, ttl)   # reset the TTL on each read
    return value
```

**Use cases:** Session data, user preferences, shopping carts.

**Warning:** Sliding TTL can keep stale data alive indefinitely if the key is
read often but never invalidated explicitly.

### Adaptive TTL

Adaptive TTL dynamically adjusts the expiration time based on observed change frequency.
If data changes often, reduce the TTL. If data is stable, extend it.

```python
def adaptive_ttl(key, last_modified_ts, now_ts, base_ttl=300, min_ttl=10, max_ttl=3600):
    age = now_ts - last_modified_ts
    if age < 60:
        # Data changed very recently — use a short TTL
        return max(min_ttl, base_ttl // 10)
    elif age < 3600:
        # Changed within the last hour — moderate TTL
        return base_ttl
    else:
        # Stable data — extend the TTL
        return min(max_ttl, base_ttl * 4)
```

---

## Event-Driven Invalidation

### Message Queue Broadcast Pattern

Instead of waiting for TTL to expire, the system that modifies data **publishes an
invalidation event** to a message queue. All services holding cached copies consume
the event and purge their local caches.

```python
# Producer — after updating the database
def update_product_price(product_id, new_price):
    db.execute("UPDATE products SET price = %s WHERE id = %s", new_price, product_id)
    # Publish invalidation event
    kafka_producer.send("cache-invalidation", {
        "entity": "product",
        "id": product_id,
        "action": "invalidate",
        "timestamp": time.time()
    })
```

```python
# Consumer — running in each service instance
for message in kafka_consumer.subscribe("cache-invalidation"):
    event = json.loads(message.value)
    cache_key = f"{event['entity']}:{event['id']}"
    redis_client.delete(cache_key)
    local_cache.pop(cache_key, None)
```

### CDC with Debezium

**Change Data Capture (CDC)** monitors the database transaction log and emits events
for every row-level change. This eliminates the need for application code to publish
invalidation events — the database itself becomes the event source.

**Debezium** is the most widely used open-source CDC platform. It reads the write-ahead
log (WAL) of PostgreSQL, the binlog of MySQL, or the oplog of MongoDB and publishes
change events to Kafka topics.

```
  ┌─────────┐      ┌───────────┐      ┌───────────┐      ┌──────────┐
  │  App     │─────▶│  Database  │─────▶│  Debezium  │─────▶│  Kafka   │
  │  writes  │      │  (WAL/    │      │  Connector │      │  Topic   │
  │          │      │   binlog) │      │            │      │          │
  └─────────┘      └───────────┘      └───────────┘      └────┬─────┘
                                                               │
                                    ┌──────────────────────────┘
                                    │
                              ┌─────▼──────┐
                              │  Consumer   │
                              │  (purge     │
                              │   cache)    │
                              └────────────┘
```

Advantages of CDC over application-level events:

| Aspect                  | Application Events           | CDC (Debezium)                 |
|-------------------------|------------------------------|--------------------------------|
| Missed events           | Possible if app crashes      | WAL is durable                 |
| Dual-write risk         | DB + queue = not atomic      | Single source (DB log)         |
| Schema coverage         | Only instrumented code paths | All writes, including ad-hoc   |
| Operational complexity  | Lower                        | Higher (Kafka + connectors)    |

### Database Triggers

Database triggers can fire invalidation signals on INSERT, UPDATE, or DELETE. This
approach keeps invalidation logic at the data layer.

```sql
-- PostgreSQL trigger example
CREATE OR REPLACE FUNCTION notify_cache_invalidation()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify('cache_invalidation',
        json_build_object(
            'table', TG_TABLE_NAME,
            'op', TG_OP,
            'id', NEW.id
        )::text
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER product_cache_trigger
AFTER INSERT OR UPDATE OR DELETE ON products
FOR EACH ROW EXECUTE FUNCTION notify_cache_invalidation();
```

**Caution:** Triggers add latency to every write and tightly couple invalidation logic
to the database. Use sparingly and avoid heavy processing inside triggers.

### Pub/Sub Invalidation

Redis itself supports pub/sub, which can be used for lightweight invalidation broadcasts
across application instances.

```python
# Publisher — after a write
redis_client.publish("invalidation:products", json.dumps({
    "key": "product:42",
    "action": "delete"
}))

# Subscriber — running in each app instance
pubsub = redis_client.pubsub()
pubsub.subscribe("invalidation:products")

for message in pubsub.listen():
    if message["type"] == "message":
        event = json.loads(message["data"])
        local_cache.pop(event["key"], None)
```

**Limitation:** Redis pub/sub is fire-and-forget. If a subscriber is disconnected when
the message is published, it will never receive it. For durable invalidation, use
Kafka or a similar persistent messaging system.

### Event-Driven Invalidation Diagram

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                  EVENT-DRIVEN INVALIDATION FLOW                     │
  │                                                                      │
  │  ┌────────┐    ┌────────┐    ┌────────────────┐    ┌─────────────┐  │
  │  │ Client │───▶│  App   │───▶│   Database     │    │  Message    │  │
  │  │        │    │ Server │    │                │    │  Broker     │  │
  │  └────────┘    └───┬────┘    └───────┬────────┘    │  (Kafka /   │  │
  │                    │                 │             │  RabbitMQ)  │  │
  │                    │                 │             └──────┬──────┘  │
  │                    │                 │                    │         │
  │           ┌────────┘        Option A │           Option B │         │
  │           │ App publishes   CDC reads │           Consume │         │
  │           │ event directly  WAL/binlog│           events  │         │
  │           │                 │         │                   │         │
  │           ▼                 ▼         │                   ▼         │
  │    ┌──────────┐      ┌──────────┐    │          ┌──────────────┐   │
  │    │  Broker  │      │ Debezium │────┘          │  Service A   │   │
  │    │  (topic) │      │ ──▶ Kafka│               │  purge cache │   │
  │    └──────────┘      └──────────┘               ├──────────────┤   │
  │                                                 │  Service B   │   │
  │                                                 │  purge cache │   │
  │                                                 ├──────────────┤   │
  │                                                 │  Service C   │   │
  │                                                 │  purge cache │   │
  │                                                 └──────────────┘   │
  └──────────────────────────────────────────────────────────────────────┘
```

---

## Versioned Keys

### Appending Version Numbers to Keys

Instead of deleting a cache entry, you change the **key itself** by incrementing a
version counter. Old entries are simply never read again and will eventually be evicted.

```python
def get_product(product_id):
    version = r.get(f"product:{product_id}:version") or "1"
    cache_key = f"product:{product_id}:v{version}"
    
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)
    
    product = db.query("SELECT * FROM products WHERE id = %s", product_id)
    r.setex(cache_key, 300, json.dumps(product))
    return product

def invalidate_product(product_id):
    # Simply bump the version — old key is orphaned
    r.incr(f"product:{product_id}:version")
```

**Pros:** No race condition between delete and re-populate.
**Cons:** Orphaned keys consume memory until eviction.

### Hash-Based Cache Keys

Instead of a numeric version, compute a hash of the data or its last-modified timestamp
and embed it in the key:

```python
import hashlib

def cache_key_with_hash(product_id, last_modified):
    data_hash = hashlib.md5(str(last_modified).encode()).hexdigest()[:8]
    return f"product:{product_id}:{data_hash}"
```

This is the strategy behind **cache-busting URLs** for static assets:

```
/static/app.js?v=a3f8b2c1
/static/styles.css?v=7d2e91f4
```

### Schema Versioning for Rolling Deployments

During rolling deployments, old and new application versions may run simultaneously.
If the new version changes the cached data schema, the old version will read
incompatible data from cache.

**Solution:** Include the schema version in the cache key namespace:

```
# Old version (v3) reads/writes:
cache_key = "schema:v3:product:42"

# New version (v4) reads/writes:
cache_key = "schema:v4:product:42"
```

Both versions operate on separate key spaces, avoiding deserialization errors.
Once the rollout completes, the old keys expire naturally.

### Cache Key Namespacing

Namespacing organizes cache keys hierarchically and allows bulk invalidation of
an entire namespace by changing the namespace version:

```python
def get_namespace_version(namespace):
    return r.get(f"ns:{namespace}:ver") or "1"

def make_key(namespace, entity_id):
    ver = get_namespace_version(namespace)
    return f"{namespace}:v{ver}:{entity_id}"

def invalidate_namespace(namespace):
    r.incr(f"ns:{namespace}:ver")  # all old keys become unreachable
```

```
  Before invalidation:         After invalidation:
  ┌────────────────────┐       ┌────────────────────┐
  │ catalog:v3:prod:1  │       │ catalog:v3:prod:1  │ ← orphaned
  │ catalog:v3:prod:2  │       │ catalog:v3:prod:2  │ ← orphaned
  │ catalog:v3:prod:3  │       │ catalog:v3:prod:3  │ ← orphaned
  └────────────────────┘       │                    │
                               │ catalog:v4:prod:1  │ ← new (on miss)
                               └────────────────────┘
```

---

## Tag-Based Invalidation

### Grouping Cache Entries with Tags

Tag-based invalidation lets you associate one or more **tags** with each cache entry.
When data changes, you invalidate all entries sharing a tag rather than knowing every
specific key.

Example: A product page cache depends on product data, category data, and pricing rules.
Tag the entry with all three so any change triggers invalidation.

```
  Cache Entry: "page:product:42"
  Tags: ["product:42", "category:electronics", "pricing:holiday-sale"]
  
  Invalidate tag "pricing:holiday-sale"
    → purges ALL entries tagged with that pricing rule
    → "page:product:42", "page:product:99", "page:category:electronics", ...
```

### Invalidating by Tag

When a tag is invalidated, every entry associated with that tag must be purged. This
requires maintaining a **reverse index** from tags to cache keys.

```python
class TaggedCache:
    def __init__(self, redis_client):
        self.r = redis_client

    def set(self, key, value, ttl, tags):
        self.r.setex(key, ttl, value)
        for tag in tags:
            self.r.sadd(f"tag:{tag}", key)   # reverse index

    def invalidate_tag(self, tag):
        tag_key = f"tag:{tag}"
        members = self.r.smembers(tag_key)
        if members:
            self.r.delete(*members)           # delete all tagged entries
            self.r.delete(tag_key)            # clean up the tag set

    def get(self, key):
        return self.r.get(key)
```

### Implementation Approaches

| Approach                  | How It Works                                   | Trade-off                          |
|---------------------------|------------------------------------------------|------------------------------------|
| **Reverse index (sets)**  | Store tag → {key1, key2, ...} in Redis sets    | Extra memory, O(n) invalidation    |
| **Tag versioning**        | Each tag has a version; key valid only if all   | No extra index, stale reads on miss|
|                           | tag versions match at read time                 |                                    |
| **Bloom filter + scan**   | Probabilistic check; SCAN for matching keys    | Approximate, high CPU on large DBs |
| **Namespace embedding**   | Embed tag versions in cache key itself          | Long keys, no explicit purge needed|

Tag versioning example (no reverse index needed):

```python
def set_with_tags(key, value, tags, ttl):
    tag_versions = {tag: r.get(f"tagver:{tag}") or "0" for tag in tags}
    meta = {"value": value, "tag_versions": tag_versions}
    r.setex(key, ttl, json.dumps(meta))

def get_with_tags(key, tags):
    raw = r.get(key)
    if not raw:
        return None
    meta = json.loads(raw)
    for tag in tags:
        current_ver = r.get(f"tagver:{tag}") or "0"
        if meta["tag_versions"].get(tag) != current_ver:
            r.delete(key)
            return None   # tag was invalidated
    return meta["value"]

def invalidate_tag(tag):
    r.incr(f"tagver:{tag}")   # bump version — all entries with old version are stale
```

---

## Delete vs Update Debate

### When to Delete the Cache Entry

Deleting the cache entry on write is the **safest** default. The next read will trigger
a cache miss, fetch the latest data from the source, and populate the cache with a
fresh value.

```
  Delete Pattern (Cache-Aside Invalidation)
  ──────────────────────────────────────────
  1. App writes new value to database
  2. App deletes cache key
  3. Next reader sees a cache miss
  4. Reader fetches from DB and populates cache
```

### When to Update In-Place

Updating the cache entry instead of deleting it can avoid the **cache miss penalty**
after a write. The writer computes the new value and writes it to both the database
and the cache.

```
  Update Pattern
  ──────────────
  1. App writes new value to database
  2. App writes new value to cache (SET with new TTL)
  3. Next reader sees a cache hit with fresh data
```

This seems appealing but introduces **race conditions**.

### Race Conditions with Update

```
  RACE CONDITION: Two writers updating cache
  ──────────────────────────────────────────────────────────────
  
  Writer A                    Writer B
  ────────                    ────────
  t1: Read DB → price $10
                              t2: Read DB → price $10
  t3: Write DB → price $12
                              t4: Write DB → price $15
  t5: Write Cache → $12
                              t6: Write Cache → $15   ← OK so far
  
  BUT what if network delays swap t5 and t6?
  
  Writer A                    Writer B
  ────────                    ────────
  t3: Write DB → $12
                              t4: Write DB → $15
                              t5: Write Cache → $15
  t6: Write Cache → $12      ← STALE! DB has $15, cache has $12
  
  The cache is now PERMANENTLY stale until TTL expires
  or another invalidation occurs.
```

### Why Delete Is Usually Safer

| Factor              | DELETE cache entry                      | UPDATE cache entry                     |
|---------------------|-----------------------------------------|----------------------------------------|
| Race conditions     | Benign — miss triggers fresh load       | Dangerous — stale value can persist    |
| Complexity          | Simple — one operation                  | Must compute new value at write time   |
| Write amplification | None                                    | Cache write on every DB write          |
| Cold read penalty   | One miss after invalidation             | None                                   |
| Consistency         | Self-healing — next read always fresh   | Can diverge permanently until TTL      |

**Rule of thumb:** Default to DELETE. Only use UPDATE when:

- Write frequency is very high AND read frequency is even higher
- You can guarantee ordering (e.g., using a single writer with sequence numbers)
- The cost of a single cache miss is extremely high (sub-ms latency requirements)

---

## Cache Stampede Prevention

### The Thundering Herd Problem

When a popular cache entry expires (or is invalidated), many concurrent requests
discover the cache miss simultaneously and all rush to the database to regenerate
the value. This is the **cache stampede** (also called thundering herd).

```
  ┌────────────────────────────────────────────────────────────────┐
  │                    CACHE STAMPEDE                               │
  │                                                                │
  │  Cache entry for "homepage:feed" expires at t=0                │
  │                                                                │
  │  t=0.001  Request A → cache MISS → query DB ─┐                │
  │  t=0.002  Request B → cache MISS → query DB ──┤                │
  │  t=0.003  Request C → cache MISS → query DB ──┤                │
  │  t=0.004  Request D → cache MISS → query DB ──┤  ALL hit the  │
  │  t=0.005  Request E → cache MISS → query DB ──┤  database!    │
  │  ...                                           │                │
  │  t=0.050  Request N → cache MISS → query DB ──┘                │
  │                                                                │
  │           ┌──────────────────────────┐                         │
  │           │      DATABASE            │                         │
  │           │  ████████████████████    │                         │
  │           │  CPU 100% — timeout!     │                         │
  │           └──────────────────────────┘                         │
  └────────────────────────────────────────────────────────────────┘
```

### Mutex / Lock Approach

Only one request is allowed to regenerate the cache entry. All other requests
wait (or get a stale value) until the lock holder populates the cache.

```python
import time

LOCK_TTL = 5  # seconds

def get_with_lock(key, ttl=300):
    value = r.get(key)
    if value is not None:
        return value

    lock_key = f"lock:{key}"
    if r.set(lock_key, "1", nx=True, ex=LOCK_TTL):
        try:
            # Won the lock — fetch from DB
            value = fetch_from_database(key)
            r.setex(key, ttl, value)
            return value
        finally:
            r.delete(lock_key)
    else:
        # Another request holds the lock — wait and retry
        time.sleep(0.05)
        return get_with_lock(key, ttl)
```

**Pros:** Only one DB query per cache miss.
**Cons:** Added latency for waiting requests; risk of deadlock if lock holder crashes.

### Probabilistic Early Expiration (XFetch)

The **XFetch** algorithm (from the paper "Optimal Probabilistic Cache Stampede
Prevention") proactively recomputes a cache entry *before* it expires. Each reader
independently decides whether to refresh based on a probabilistic function of the
remaining TTL.

```python
import math
import random
import time

def xfetch_get(key, ttl=300, beta=1.0):
    raw = r.get(key)
    if raw is None:
        return recompute_and_cache(key, ttl)
    
    entry = json.loads(raw)
    value     = entry["value"]
    delta     = entry["delta"]       # time to recompute
    expiry    = entry["expiry"]      # when the entry expires
    
    now = time.time()
    # Probabilistic early expiration
    if now - delta * beta * math.log(random.random()) >= expiry:
        return recompute_and_cache(key, ttl)
    
    return value

def recompute_and_cache(key, ttl):
    start = time.time()
    value = fetch_from_database(key)
    delta = time.time() - start
    
    entry = {
        "value": value,
        "delta": delta,
        "expiry": time.time() + ttl
    }
    r.setex(key, ttl, json.dumps(entry))
    return value
```

As the entry approaches expiration, the probability of a reader triggering a refresh
increases, so the value is almost always recomputed before the hard TTL expires.

### Request Coalescing / Single-Flight

Multiple concurrent requests for the same key are **coalesced** into a single database
fetch. Only the first request executes; the rest wait for the result.

```go
// Go — using golang.org/x/sync/singleflight
package main

import (
    "golang.org/x/sync/singleflight"
)

var group singleflight.Group

func GetProduct(id string) (Product, error) {
    result, err, _ := group.Do("product:"+id, func() (interface{}, error) {
        // Only one goroutine executes this, even if 100 call simultaneously
        return fetchProductFromDB(id)
    })
    if err != nil {
        return Product{}, err
    }
    return result.(Product), nil
}
```

```
  Without Coalescing               With Coalescing (Single-Flight)
  ┌───────────────────┐            ┌───────────────────┐
  │ Req A ──▶ DB ──▶  │            │ Req A ──▶ DB ──┐  │
  │ Req B ──▶ DB ──▶  │            │ Req B ─── wait ─┤  │
  │ Req C ──▶ DB ──▶  │            │ Req C ─── wait ─┤  │
  │ 3 DB queries       │            │ 1 DB query       │  │
  └───────────────────┘            └───────────────────┘
```

### Stale-While-Revalidate

Serve the **stale cached value** immediately while triggering an asynchronous background
refresh. The user gets a fast (but possibly stale) response, and the cache is updated
for subsequent requests.

```python
import threading

def get_stale_while_revalidate(key, ttl=300, stale_ttl=60):
    value = r.get(key)
    remaining = r.ttl(key)
    
    if value is not None:
        if remaining < stale_ttl:
            # TTL is almost up — trigger background refresh
            threading.Thread(
                target=recompute_and_cache,
                args=(key, ttl)
            ).start()
        return value   # return possibly-stale value immediately
    
    # Hard miss — must fetch synchronously
    return recompute_and_cache(key, ttl)
```

This pattern is also available as an HTTP cache directive:

```
Cache-Control: max-age=300, stale-while-revalidate=60
```

### Pre-Warming

Proactively populate the cache **before** it is needed, eliminating cold-start misses
entirely. Useful after deployments, cache flushes, or when predictable traffic spikes
are expected (e.g., a scheduled sale).

```python
def prewarm_cache(product_ids):
    """Call before a flash sale begins."""
    for pid in product_ids:
        product = fetch_from_database(f"product:{pid}")
        r.setex(f"product:{pid}", 600, json.dumps(product))
    print(f"Pre-warmed {len(product_ids)} products")
```

---

## Multi-Layer Invalidation

### L1 + L2 + CDN Architecture

Modern systems often use multiple cache layers. Invalidation must propagate through
**all** of them to prevent serving stale data from any layer.

| Layer             | Type                | Latency     | Scope                     |
|-------------------|---------------------|-------------|---------------------------|
| **L1** (in-process) | HashMap, Caffeine, Guava | ~1 μs  | Single process instance   |
| **L2** (distributed) | Redis, Memcached   | ~1 ms       | All instances in a region |
| **CDN** (edge)    | Cloudflare, Akamai  | ~10–50 ms   | Global edge locations     |

### Propagation Delays

```
  Invalidation Event Timeline
  ──────────────────────────────────────────────────────────
  t=0ms       DB updated
  t=1ms       App deletes L2 (Redis) key
  t=2ms       App publishes invalidation event
  t=5ms       Other app instances receive event → purge L1
  t=50ms      CDN purge API called
  t=200ms     CDN edge nodes worldwide purged
  
  Window of inconsistency varies by layer:
    L2:  ~1ms
    L1:  ~5ms  (depends on pub/sub latency)
    CDN: ~200ms (depends on provider and geography)
```

### Consistency Challenges Across Layers

1. **L1 stale after L2 purge** — L2 is invalidated but L1 still holds the old value.
   Subsequent reads hit L1 and never reach L2.

2. **CDN re-caches stale L2** — CDN edge expires and fetches from origin, which hits
   L2 just before L2 is invalidated. CDN now caches a fresh copy of stale data.

3. **Cross-region lag** — Region A invalidates first; Region B still serves old data
   until replication catches up.

**Mitigation strategies:**

- Keep L1 TTLs very short (1–10 seconds) as a safety net
- Use broadcast invalidation (pub/sub) to actively purge L1 across instances
- Use CDN purge APIs with "purge by surrogate key" for targeted invalidation
- Version cache keys so stale entries are never read even if not purged

### Multi-Layer Invalidation Diagram

```
  ┌──────────────────────────────────────────────────────────────────┐
  │              MULTI-LAYER INVALIDATION FLOW                       │
  │                                                                  │
  │  ┌──────┐     ┌────────────────────────────────────────────┐     │
  │  │  DB  │────▶│  Invalidation Coordinator                  │     │
  │  │update│     │                                            │     │
  │  └──────┘     │  1. Delete from L2 (Redis)                 │     │
  │               │  2. Publish event to message bus            │     │
  │               │  3. Call CDN purge API                      │     │
  │               └───────┬──────────────┬──────────────┬──────┘     │
  │                       │              │              │             │
  │                       ▼              ▼              ▼             │
  │               ┌──────────┐   ┌────────────┐   ┌──────────┐      │
  │               │ L2 Redis │   │ Msg Bus    │   │ CDN API  │      │
  │               │  (delete) │   │ (fanout)   │   │ (purge)  │      │
  │               └──────────┘   └──────┬─────┘   └──────────┘      │
  │                                     │                            │
  │                    ┌────────────────┬┴───────────────┐           │
  │                    ▼                ▼                ▼           │
  │              ┌──────────┐   ┌──────────┐    ┌──────────┐        │
  │              │Instance A│   │Instance B│    │Instance C│        │
  │              │ purge L1 │   │ purge L1 │    │ purge L1 │        │
  │              └──────────┘   └──────────┘    └──────────┘        │
  └──────────────────────────────────────────────────────────────────┘
```

---

## Database-Driven Invalidation

### PostgreSQL LISTEN/NOTIFY

PostgreSQL provides a built-in pub/sub mechanism that can push row-change notifications
to connected clients without any external infrastructure.

```sql
-- In PostgreSQL: trigger sends notification on change
CREATE OR REPLACE FUNCTION notify_product_change()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify(
        'product_changes',
        json_build_object(
            'operation', TG_OP,
            'id', COALESCE(NEW.id, OLD.id),
            'table', TG_TABLE_NAME
        )::text
    );
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_product_change
AFTER INSERT OR UPDATE OR DELETE ON products
FOR EACH ROW EXECUTE FUNCTION notify_product_change();
```

```python
# Python listener using psycopg2
import psycopg2
import psycopg2.extensions
import select
import json

conn = psycopg2.connect("dbname=mydb")
conn.set_isolation_level(psycopg2.extensions.ISOLATION_LEVEL_AUTOCOMMIT)

cur = conn.cursor()
cur.execute("LISTEN product_changes;")

while True:
    if select.select([conn], [], [], 5) != ([], [], []):
        conn.poll()
        while conn.notifies:
            notify = conn.notifies.pop(0)
            event = json.loads(notify.payload)
            cache_key = f"product:{event['id']}"
            redis_client.delete(cache_key)
            print(f"Invalidated {cache_key} due to {event['operation']}")
```

### MySQL Binlog

MySQL's binary log records all data modifications. Tools like **Maxwell** or
**Debezium** can tail the binlog and emit change events.

```python
# Using python-mysql-replication to tail binlog
from pymysqlreplication import BinLogStreamReader
from pymysqlreplication.row_event import (
    UpdateRowsEvent, WriteRowsEvent, DeleteRowsEvent
)

stream = BinLogStreamReader(
    connection_settings={"host": "127.0.0.1", "port": 3306,
                         "user": "repl", "passwd": "secret"},
    server_id=100,
    only_events=[UpdateRowsEvent, WriteRowsEvent, DeleteRowsEvent],
    only_tables=["products"],
    blocking=True
)

for event in stream:
    for row in event.rows:
        if hasattr(row, "after_values"):
            product_id = row["after_values"]["id"]
        else:
            product_id = row["values"]["id"]
        
        redis_client.delete(f"product:{product_id}")
        print(f"Invalidated product:{product_id}")

stream.close()
```

### MongoDB Change Streams

MongoDB 3.6+ natively supports **change streams**, allowing applications to subscribe
to real-time data changes.

```python
# Python — using pymongo change streams
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017")
db = client["mydb"]
collection = db["products"]

with collection.watch() as stream:
    for change in stream:
        op = change["operationType"]       # insert, update, delete, replace
        doc_id = change["documentKey"]["_id"]
        
        cache_key = f"product:{doc_id}"
        redis_client.delete(cache_key)
        print(f"Invalidated {cache_key} (op={op})")
```

### Debezium CDC Pipeline

A complete Debezium pipeline consists of:

```
  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────────┐
  │ Database │────▶│ Debezium │────▶│  Kafka   │────▶│ Cache        │
  │ (source) │     │ Connect  │     │  Topic   │     │ Invalidation │
  │          │     │ (reads   │     │          │     │ Consumer     │
  │          │     │  WAL)    │     │          │     │              │
  └──────────┘     └──────────┘     └──────────┘     └──────────────┘
```

Debezium connector configuration (JSON):

```json
{
  "name": "products-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "db-host",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "secret",
    "database.dbname": "mydb",
    "table.include.list": "public.products",
    "topic.prefix": "myapp",
    "plugin.name": "pgoutput"
  }
}
```

The consumer processes events from the `myapp.public.products` Kafka topic:

```python
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    "myapp.public.products",
    bootstrap_servers=["kafka:9092"],
    value_deserializer=lambda m: json.loads(m.decode("utf-8"))
)

for message in consumer:
    payload = message.value["payload"]
    operation = payload["op"]         # c=create, u=update, d=delete
    
    if operation in ("u", "d"):
        product_id = payload["before"]["id"]
    else:
        product_id = payload["after"]["id"]
    
    redis_client.delete(f"product:{product_id}")
    print(f"CDC invalidation: product:{product_id} (op={operation})")
```

---

## Invalidation Patterns Comparison

### Comprehensive Comparison Table

| Pattern                    | Trigger          | Latency        | Complexity | Consistency      | Failure Mode                 |
|----------------------------|------------------|----------------|------------|------------------|------------------------------|
| **TTL expiration**         | Timer            | Up to TTL      | Very low   | Eventual         | Stale reads until TTL        |
| **App-level delete**       | Write path code  | Near-instant   | Low        | Near-real-time   | Missed if app crashes        |
| **App-level event**        | Pub/sub message  | Milliseconds   | Moderate   | Near-real-time   | Lost if subscriber offline   |
| **Versioned keys**         | Key rename       | Instant        | Moderate   | Strong (reads)   | Orphaned key memory overhead |
| **Tag-based**              | Tag version bump | Instant–ms     | Moderate   | Near-real-time   | Extra storage for tag index  |
| **CDC (Debezium)**         | DB WAL/binlog    | Seconds        | High       | Near-real-time   | Connector lag or failure     |
| **DB LISTEN/NOTIFY**       | DB trigger       | Milliseconds   | Moderate   | Near-real-time   | Lost if listener disconnects |
| **CDN purge API**          | API call         | 100ms–minutes  | Moderate   | Eventual         | API failure, propagation lag |
| **XFetch (probabilistic)** | Probabilistic    | Pre-emptive    | Moderate   | Near-real-time   | Rare double-compute          |
| **Stale-while-revalidate** | Background fetch | Serve stale    | Low        | Eventual         | Stale for one request cycle  |

### Choosing the Right Pattern

```
  ┌─────────────────────────────────────────────────────────────┐
  │             DECISION GUIDE                                  │
  │                                                             │
  │  Can you tolerate minutes of staleness?                     │
  │  ├── YES → TTL-based (simplest, start here)                │
  │  └── NO                                                     │
  │       │                                                     │
  │       ├── Do you control the write path?                    │
  │       │   ├── YES → App-level delete + short TTL backup     │
  │       │   └── NO  → CDC with Debezium                      │
  │       │                                                     │
  │       ├── Is the data updated by multiple services?         │
  │       │   └── YES → Event-driven (Kafka) invalidation      │
  │       │                                                     │
  │       ├── Are rolling deployments a concern?                │
  │       │   └── YES → Versioned keys with schema prefix      │
  │       │                                                     │
  │       └── Do you need to invalidate groups of entries?      │
  │           └── YES → Tag-based invalidation                  │
  └─────────────────────────────────────────────────────────────┘
```

---

## Testing Cache Invalidation

### Chaos Testing

Cache invalidation is a prime candidate for chaos engineering. The system must behave
correctly even when components fail or experience delays.

**Scenarios to test:**

| Chaos Scenario                        | What It Validates                              |
|---------------------------------------|------------------------------------------------|
| Kill the cache (Redis down)           | Application degrades gracefully to DB          |
| Delay invalidation events by 30s      | Users see stale data; is it acceptable?        |
| Drop 50% of invalidation messages     | How long before stale data self-heals (TTL)?   |
| Simulate cache stampede (flush all)   | DB can handle the thundering herd              |
| Network partition between app & cache | Reads fall through; writes still invalidate?   |
| Kill CDC connector for 5 minutes      | Backlog replays correctly after reconnect?     |

### Simulating Stale Data

Intentionally inject stale data to verify detection and alerting:

```python
def test_stale_data_detection():
    # 1. Write known value to DB
    db.execute("UPDATE products SET price = 99.99 WHERE id = 1")
    
    # 2. Write DIFFERENT value directly to cache (simulating stale)
    redis_client.setex("product:1:price", 300, "49.99")
    
    # 3. Read through application — should it detect the mismatch?
    result = app.get_product_price(1)
    
    # 4. If using background consistency checker:
    assert staleness_checker.detected_stale("product:1:price")
```

### Staleness Metrics and Monitoring

Track these metrics to measure invalidation effectiveness:

| Metric                           | Description                                          | Alert Threshold       |
|----------------------------------|------------------------------------------------------|-----------------------|
| `cache.staleness.age_seconds`    | Time since source changed vs cached value            | > 2× expected TTL     |
| `cache.invalidation.lag_ms`      | Delay between DB write and cache purge               | > 500ms (tunable)     |
| `cache.invalidation.failures`    | Failed invalidation attempts                         | > 0 sustained         |
| `cache.stampede.concurrent_miss` | Simultaneous cache misses for same key               | > 10                  |
| `cache.hit_ratio`                | Percentage of requests served from cache             | < 80% (data-dependent)|
| `cache.stale_read_rate`          | Reads that returned data older than max allowed age  | > 1%                  |

```python
# Prometheus-style staleness tracking
from prometheus_client import Histogram, Counter

cache_staleness = Histogram(
    "cache_staleness_seconds",
    "Seconds between DB update and cache reflecting the update",
    buckets=[0.01, 0.05, 0.1, 0.5, 1, 5, 10, 30, 60, 300]
)

stale_reads = Counter(
    "cache_stale_reads_total",
    "Number of reads that returned data older than the acceptable threshold"
)
```

### Integration Test Patterns

```python
class TestCacheInvalidation:
    """Integration tests for cache invalidation correctness."""
    
    def test_write_invalidates_cache(self):
        """After a DB write, subsequent read returns fresh data."""
        # Populate cache
        product = service.get_product(42)
        assert redis_client.exists("product:42")
        
        # Update DB
        service.update_product(42, {"price": 29.99})
        
        # Cache should be invalidated
        assert not redis_client.exists("product:42")
        
        # Next read should return fresh data
        product = service.get_product(42)
        assert product["price"] == 29.99
    
    def test_concurrent_write_and_read(self):
        """Concurrent read during write does not cache stale data."""
        import threading
        
        results = []
        
        def writer():
            service.update_product(42, {"price": 99.99})
        
        def reader():
            product = service.get_product(42)
            results.append(product["price"])
        
        threads = [
            threading.Thread(target=writer),
            threading.Thread(target=reader),
        ]
        for t in threads:
            t.start()
        for t in threads:
            t.join()
        
        # After both complete, cache must reflect latest value
        final = service.get_product(42)
        assert final["price"] == 99.99
    
    def test_tag_invalidation_purges_group(self):
        """Invalidating a tag purges all entries with that tag."""
        cache.set("page:prod:1", "html1", tags=["category:electronics"])
        cache.set("page:prod:2", "html2", tags=["category:electronics"])
        cache.set("page:prod:3", "html3", tags=["category:books"])
        
        cache.invalidate_tag("category:electronics")
        
        assert cache.get("page:prod:1") is None
        assert cache.get("page:prod:2") is None
        assert cache.get("page:prod:3") is not None   # different tag
```

---

## Real-World Case Studies

### Facebook Memcache Invalidation

Facebook operates one of the largest Memcached deployments in the world, serving
billions of requests per second. Their invalidation system, described in the 2013
NSDI paper *"Scaling Memcache at Facebook"*, is a masterclass in distributed cache
invalidation.

**Architecture overview:**

```
  ┌──────────────────────────────────────────────────────────────────┐
  │              FACEBOOK MEMCACHE ARCHITECTURE                      │
  │                                                                  │
  │  ┌──────────┐    ┌─────────────┐    ┌────────────────────────┐  │
  │  │  Web     │───▶│  Memcached  │    │  MySQL (source of      │  │
  │  │  Server  │    │  Cluster    │    │  truth)                │  │
  │  └────┬─────┘    └─────────────┘    └─────────┬──────────────┘  │
  │       │                                       │                  │
  │       │          ┌─────────────┐              │                  │
  │       └─────────▶│  McRouter   │              │                  │
  │                  │  (routing   │              │                  │
  │                  │   proxy)    │              │                  │
  │                  └─────────────┘              │                  │
  │                                               ▼                  │
  │                                     ┌─────────────────┐         │
  │                                     │   McSqueal       │         │
  │                                     │  (tails MySQL    │         │
  │                                     │   commit log,    │         │
  │                                     │   sends deletes) │         │
  │                                     └─────────────────┘         │
  └──────────────────────────────────────────────────────────────────┘
```

**Key innovations:**

1. **McSqueal (SQL-based CDC):** A daemon that tails MySQL's commit log and extracts
   invalidation events. When a row changes, McSqueal determines which cache keys
   must be invalidated and sends batched delete commands to Memcached clusters. This
   is effectively CDC before Debezium existed.

2. **Lease Gets (Stampede Prevention):** When a cache miss occurs, the Memcached
   server issues a **lease token** to the requesting client. Only the client holding
   the lease can populate the cache. Other clients requesting the same key within a
   short window receive a "try again later" response, preventing thundering herds.

   ```
   Client A: GET key → MISS + lease_token=12345
   Client B: GET key → MISS + "wait, lease in progress"
   Client A: SET key value lease=12345 → SUCCESS
   Client B: GET key → HIT (value populated by A)
   ```

3. **Regional Invalidation:** Facebook runs multiple Memcached regions. Cross-region
   invalidation uses a **marker system** — when a user writes in one region, a marker
   is set that redirects reads to the source-of-truth region until invalidation
   propagates. This prevents reading stale data from the replica region.

4. **Delete over Set:** Facebook always **deletes** cache entries on write rather than
   updating them. This aligns with the "delete is safer" principle — it avoids the
   race conditions inherent in updating cache entries concurrently.

5. **Gutter Pool:** A small pool of Memcached servers acts as a fallback for keys
   whose primary server is down. Rather than flooding the database during server
   failures, reads are redirected to the gutter pool with a very short TTL.

**Scale numbers (from the paper):**

| Metric                    | Value                               |
|---------------------------|-------------------------------------|
| Memcached servers         | ~1,000+ across multiple clusters    |
| Peak requests/sec         | Billions                            |
| Invalidation events/sec   | Millions (via McSqueal)             |
| Cache hit ratio           | ~99%+                               |
| Cross-region propagation  | < 1 second (typically)              |

### Netflix EVCache

Netflix uses **EVCache** (Ephemeral Volatile Cache), a distributed caching solution
built on top of Memcached with a custom client library and replication layer.

**Key characteristics:**

```
  ┌──────────────────────────────────────────────────────────────┐
  │              NETFLIX EVCACHE ARCHITECTURE                    │
  │                                                              │
  │  ┌─────────────┐                                            │
  │  │ Application │                                            │
  │  │   Service   │                                            │
  │  └──────┬──────┘                                            │
  │         │                                                    │
  │         ▼                                                    │
  │  ┌──────────────┐   Writes replicated    ┌───────────────┐  │
  │  │  EVCache     │──────────────────────▶│  EVCache      │  │
  │  │  Zone A      │   across zones         │  Zone B       │  │
  │  │  (us-east-1a)│◀──────────────────────│  (us-east-1b) │  │
  │  └──────────────┘   Reads: local zone    └───────────────┘  │
  │                      only (low latency)                      │
  │                                                              │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │  EVCache Client Library                              │   │
  │  │  ● Zone-aware routing (read local, write all)        │   │
  │  │  ● Automatic failover to other zones                 │   │
  │  │  ● Retry with backoff                                │   │
  │  │  ● Metrics and health checks built-in                │   │
  │  └──────────────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────────┘
```

**Invalidation approach:**

1. **Write-to-all-zones:** When data is invalidated, the EVCache client deletes the
   key from **all** availability zones, not just the local one. This provides
   cross-zone consistency.

2. **TTL as safety net:** Every entry has a TTL even when explicit invalidation is used.
   This provides a fallback if an invalidation message is lost.

3. **Cache warming on deployment:** Netflix pre-warms caches during deployments by
   running a "warm-up" phase that populates frequently accessed keys before the new
   instance starts serving traffic.

4. **Moneta (global replication):** For cross-region caching, Netflix built Moneta, a
   replication pipeline that synchronizes cache state across AWS regions. Invalidation
   in one region propagates to others via Moneta.

**Netflix invalidation lessons:**

| Lesson                              | Implementation                                |
|-------------------------------------|-----------------------------------------------|
| Always have a TTL fallback          | Even with explicit invalidation               |
| Zone-aware invalidation             | Delete from all zones on write                |
| Pre-warm after deployments          | Avoid cold-start stampede                     |
| Monitor invalidation lag            | Real-time dashboards per cache cluster        |
| Graceful degradation                | If cache is down, fall through to data store  |

---

## Version History

| Date       | Change          | Author |
|------------|-----------------|--------|
| 2025-04-15 | Initial version | Team   |
