# Cache Schema Design

## Table of Contents

1. [Overview](#overview)
   - [What Is Cache Schema Design?](#what-is-cache-schema-design)
   - [Why Cache Schema Design Matters](#why-cache-schema-design-matters)
   - [Relationship to Other Caching Docs](#relationship-to-other-caching-docs)
2. [Mapping Database Schema to Cache Schema](#mapping-database-schema-to-cache-schema)
   - [Why 1:1 Mapping Is Usually Wrong](#why-11-mapping-is-usually-wrong)
   - [Denormalization for Read Performance](#denormalization-for-read-performance)
   - [Pre-Joining Related Data](#pre-joining-related-data)
   - [Flattening Nested Structures](#flattening-nested-structures)
   - [Embedding vs Referencing in Cache](#embedding-vs-referencing-in-cache)
   - [Freshness vs Flexibility Trade-Off](#freshness-vs-flexibility-trade-off)
3. [Designing Cached Objects](#designing-cached-objects)
   - [Composite Objects](#composite-objects)
   - [Partial Objects](#partial-objects)
   - [Pre-Computed Aggregates](#pre-computed-aggregates)
   - [View-Model Caching](#view-model-caching)
   - [Versioned Cache Schemas](#versioned-cache-schemas)
4. [Redis Data Structure Selection](#redis-data-structure-selection)
   - [Data Structure Overview](#data-structure-overview)
   - [Decision Matrix](#decision-matrix)
   - [Strings — Serialized Objects](#strings--serialized-objects)
   - [Hashes — User Profiles and Entities](#hashes--user-profiles-and-entities)
   - [Sorted Sets — Leaderboards and Rankings](#sorted-sets--leaderboards-and-rankings)
   - [Lists — Activity Feeds and Queues](#lists--activity-feeds-and-queues)
   - [Sets — Tags and Unique Visitors](#sets--tags-and-unique-visitors)
   - [HyperLogLog — Cardinality Estimation](#hyperloglog--cardinality-estimation)
   - [Streams — Real-Time Analytics](#streams--real-time-analytics)
   - [RedisJSON — Document Caching](#redisjson--document-caching)
   - [Storage Efficiency Comparison](#storage-efficiency-comparison)
5. [Cache Granularity](#cache-granularity)
   - [Fine-Grained vs Coarse-Grained Caching](#fine-grained-vs-coarse-grained-caching)
   - [Single-Entity vs Collection Caching](#single-entity-vs-collection-caching)
   - [Page-Level vs Fragment-Level Caching](#page-level-vs-fragment-level-caching)
   - [Granularity Trade-Offs](#granularity-trade-offs)
   - [Granularity Decision Framework](#granularity-decision-framework)
6. [Schema Versioning and Evolution](#schema-versioning-and-evolution)
   - [Why Cached Data Needs Schema Versioning](#why-cached-data-needs-schema-versioning)
   - [Version-in-Key Pattern](#version-in-key-pattern)
   - [Version-in-Value Pattern](#version-in-value-pattern)
   - [Rolling Cache Migrations](#rolling-cache-migrations)
   - [Backward-Compatible Changes](#backward-compatible-changes)
   - [Handling Schema Mismatches During Deployments](#handling-schema-mismatches-during-deployments)
7. [Relationship Patterns in Cache](#relationship-patterns-in-cache)
   - [Embedded (Denormalized) Pattern](#embedded-denormalized-pattern)
   - [Reference-Based Pattern](#reference-based-pattern)
   - [Hybrid Pattern](#hybrid-pattern)
   - [Cache Dependency Graphs](#cache-dependency-graphs)
   - [When Embedding Becomes Too Expensive](#when-embedding-becomes-too-expensive)
8. [Multi-Model Caching](#multi-model-caching)
   - [Same Data, Multiple Shapes](#same-data-multiple-shapes)
   - [Keeping Derived Views Consistent](#keeping-derived-views-consistent)
   - [Fan-Out on Write vs Fan-Out on Read](#fan-out-on-write-vs-fan-out-on-read)
9. [Patterns for Common Domains](#patterns-for-common-domains)
   - [E-Commerce](#e-commerce)
   - [Social Media](#social-media)
   - [SaaS Platforms](#saas-platforms)
   - [API Response Caching](#api-response-caching)
10. [Anti-Patterns](#anti-patterns)
    - [Over-Caching](#over-caching)
    - [Stale Graph Fragments](#stale-graph-fragments)
    - [Cache-as-Database Trap](#cache-as-database-trap)
    - [Unbounded Collection Caching](#unbounded-collection-caching)
    - [Ignoring Serialization Cost](#ignoring-serialization-cost)
    - [Tight Coupling Between Cache and DB Schema](#tight-coupling-between-cache-and-db-schema)
11. [Schema Design Checklist](#schema-design-checklist)
12. [Version History](#version-history)

---

## Overview

### What Is Cache Schema Design?

Cache schema design is the practice of deciding **what shape** your cached data takes — how
objects are structured, which fields are included, how relationships are represented, and which
Redis data structures hold them. It sits between two concerns you may already be familiar with:

- **Cache key design** (covered in [08-BEST-PRACTICES](08-BEST-PRACTICES.md)) answers *how you
  name* your cache entries.
- **Cache strategy** (covered in [04-CACHE-STRATEGIES](04-CACHE-STRATEGIES.md)) answers *when
  and how* data flows between cache and database.

Cache schema design answers a different question: **what goes inside** the cache entry and how
is it organized for optimal read performance?

```
  ┌───────────────────────────────────────────────────────────────┐
  │                   THE CACHING DECISION SPACE                  │
  │                                                               │
  │   Cache Key Design        Cache Schema Design      Strategy   │
  │   ┌──────────────┐        ┌──────────────────┐   ┌────────┐  │
  │   │ HOW you name │        │ WHAT you store   │   │ WHEN & │  │
  │   │ the entry    │        │ and HOW it is    │   │ HOW it │  │
  │   │              │        │ shaped           │   │ flows  │  │
  │   └──────────────┘        └──────────────────┘   └────────┘  │
  │                                                               │
  │   user:42:profile          { id, name, email,    cache-aside  │
  │                              avatar_url, ...}    TTL: 300s    │
  │                                                               │
  └───────────────────────────────────────────────────────────────┘
```

### Why Cache Schema Design Matters

The core insight is simple but important: **your cache schema does NOT need to mirror your
database schema**. In fact, it usually should not.

A relational database is optimized for write correctness — normalized tables, foreign keys,
referential integrity. A cache is optimized for read speed — denormalized blobs, pre-joined
data, pre-computed results. Copying your database schema into your cache means inheriting all
of the read-time complexity (joins, aggregations) that you were trying to avoid.

| Concern                  | Database Schema                          | Cache Schema                            |
|--------------------------|------------------------------------------|-----------------------------------------|
| **Optimized for**        | Write correctness and storage efficiency | Read speed and access pattern matching  |
| **Normalization**        | Highly normalized (3NF or higher)        | Denormalized, pre-joined                |
| **Relationships**        | Foreign keys and JOINs                   | Embedded or duplicated                  |
| **Data shape**           | Table rows                               | API responses, view models, aggregates  |
| **Flexibility**          | Supports ad-hoc queries                  | Optimized for known access patterns     |
| **Consistency**          | ACID transactions                        | Eventual, TTL-bounded                   |

### Relationship to Other Caching Docs

This document complements the existing caching material:

- **[04-CACHE-STRATEGIES](04-CACHE-STRATEGIES.md)** — Tells you *when* to populate and
  invalidate the cache. Schema design tells you *what* to put in it.
- **[05-CACHE-INVALIDATION](05-CACHE-INVALIDATION.md)** — Tells you how to expire stale
  data. Schema design affects how easy or hard invalidation is (e.g., embedded data is
  harder to invalidate granularly).
- **[08-BEST-PRACTICES](08-BEST-PRACTICES.md)** — Covers key naming, TTL, memory management,
  and serialization. Schema design focuses on the object structure itself.

---

## Mapping Database Schema to Cache Schema

### Why 1:1 Mapping Is Usually Wrong

A common first instinct is to cache each database table row as-is. This rarely works well
because it results in multiple round trips, wastes memory on unread columns, couples cache
to DB migrations, and skips pre-computation of derived values.

```
  BAD: 1:1 Mirror                         GOOD: Read-Optimized Shape
  ─────────────────                        ────────────────────────────

  DB: users, addresses, orders tables      Cache: user_profile
  → 3 cache reads, still need joins        ┌─────────────────────────────┐
                                           │ id, name, email, address,   │
                                           │ city, order_count, avatar   │
                                           └─────────────────────────────┘
                                           → One cache read. No joins.
```

### Denormalization for Read Performance

In a cache, denormalization is not a compromise — it is the goal. You trade write complexity
(updating multiple cache entries when source data changes) for read simplicity (one key, one
round trip, complete result).

**Guidelines for denormalizing:**

1. **Start from the access pattern.** What does the caller need in a single request?
2. **Embed related data** that is always read together.
3. **Duplicate selectively.** A user's name might appear in their profile cache *and* in
   each cached order summary — that is acceptable.
4. **Accept the invalidation cost.** More keys to update on write is the conscious trade-off.

### Pre-Joining Related Data

Pre-joining means performing the database JOIN at write time (when populating the cache)
rather than at read time. The cached object contains fields from multiple tables.

```python
# Instead of caching raw rows and joining at read time:
def get_product_display(product_id: str) -> dict:
    """Build a pre-joined product object for the cache."""
    product = db.query("SELECT * FROM products WHERE id = %s", product_id)
    category = db.query("SELECT name FROM categories WHERE id = %s", product["category_id"])
    images = db.query("SELECT url, alt FROM product_images WHERE product_id = %s", product_id)
    avg_rating = db.query(
        "SELECT AVG(score) as avg, COUNT(*) as cnt FROM reviews WHERE product_id = %s",
        product_id,
    )

    cached_object = {
        "id": product["id"],
        "name": product["name"],
        "price_cents": product["price_cents"],
        "category_name": category["name"],
        "images": [{"url": img["url"], "alt": img["alt"]} for img in images],
        "avg_rating": round(avg_rating["avg"], 1),
        "review_count": avg_rating["cnt"],
    }

    redis.setex(f"product:{product_id}", 600, json.dumps(cached_object))
    return cached_object
```

The database does the heavy lifting once. Every subsequent read is a single `GET`.

### Flattening Nested Structures

Deeply nested objects increase serialization cost and make partial updates difficult. Flatten
where possible.

```json
// BEFORE: Deeply nested
{
  "user": {
    "profile": {
      "personal": {
        "name": "Alice",
        "bio": "Engineer"
      },
      "settings": {
        "theme": "dark",
        "locale": "en-US"
      }
    }
  }
}

// AFTER: Flattened
{
  "user_name": "Alice",
  "user_bio": "Engineer",
  "pref_theme": "dark",
  "pref_locale": "en-US"
}
```

Flattening reduces serialization overhead and makes it straightforward to use Redis Hashes
where each field maps to a top-level key.

### Embedding vs Referencing in Cache

| Approach       | Description                                      | Best For                                    |
|----------------|--------------------------------------------------|---------------------------------------------|
| **Embedding**  | Include related data inline inside the object    | Data read together, low update frequency    |
| **Referencing**| Store IDs; caller fetches related objects by key | Data updated independently, shared objects  |

**Embed** when the related data is small, rarely changes independently, and the caller
always needs it. **Reference** when the entity is large, updated frequently, or shared
across many parents.

### Freshness vs Flexibility Trade-Off

Denormalized, pre-joined cache objects are fast to read but harder to keep fresh. Every time
a component piece changes (e.g., a category name), all cache entries embedding that data must
be invalidated. Most production systems land in a **hybrid** zone — embed data that rarely
changes and reference data that changes frequently.

| Approach              | Read Speed | Invalidation Cost | Duplication |
|-----------------------|------------|-------------------|-------------|
| Fully denormalized    | Fastest    | Highest           | High        |
| Hybrid (embed + refs) | Balanced   | Moderate          | Moderate    |
| Fully normalized      | Slowest    | Lowest            | None        |

---

## Designing Cached Objects

### Composite Objects

A composite cached object assembles data from multiple database tables or services into a
single cache entry tailored for a specific use case.

```python
def build_user_dashboard(user_id: int) -> dict:
    """Assemble a composite dashboard object from multiple sources."""
    user = user_service.get(user_id)
    orders = order_service.get_recent(user_id, limit=5)
    loyalty = loyalty_service.get_tier(user_id)
    notif_count = notification_service.unread_count(user_id)

    return {
        "name": user.name,
        "avatar_url": user.avatar_url,
        "recent_orders": [
            {"id": o.id, "total": o.total_cents, "status": o.status}
            for o in orders
        ],
        "loyalty_tier": loyalty.tier_name,
        "unread_notifications": notif_count,
    }
```

The database and services do the heavy lifting once. Every subsequent read is a single `GET`
from cache.

### Partial Objects

Not every field in an entity is read frequently. Caching only the **hot fields** saves memory.

```python
CACHED_USER_FIELDS = {"id", "name", "email", "avatar_url", "role", "locale"}

def cache_user_partial(user_id: int):
    user = db.get_user(user_id)
    partial = {k: v for k, v in user.items() if k in CACHED_USER_FIELDS}
    redis.hset(f"user:{user_id}", mapping=partial)
    redis.expire(f"user:{user_id}", 300)
```

**When to use:** The full entity is large (> 1 KB) but only a few fields are accessed on hot
paths; different consumers need different subsets; memory is constrained with millions of
entities.

### Pre-Computed Aggregates

Instead of recalculating aggregates on every read, compute them at write time and cache the
result.

| Aggregate                 | Computed From              | Cache Key                    | TTL  |
|---------------------------|----------------------------|------------------------------|------|
| Product average rating    | `reviews` table            | `agg:product:42:rating`      | 600s |
| User order count          | `orders` table             | `agg:user:42:order_count`    | 300s |
| Category product count    | `products` table           | `agg:category:7:product_cnt` | 900s |
| Daily active users        | `logins` table             | `agg:dau:2025-07-15`        | 3600s|

```python
def update_product_rating(product_id: int):
    """Recompute and cache the average rating after a new review."""
    result = db.query(
        "SELECT AVG(score) as avg, COUNT(*) as cnt FROM reviews WHERE product_id = %s",
        product_id,
    )
    redis.hset(f"agg:product:{product_id}:rating", mapping={
        "average": str(round(result["avg"], 2)),
        "count": str(result["cnt"]),
    })
    redis.expire(f"agg:product:{product_id}:rating", 600)
```

### View-Model Caching

Cache what the API **returns**, not the raw database row. The cached object matches the
response contract exactly, so the API handler can return it with zero transformation.

```python
def get_product_api_v2(product_id: int) -> dict:
    cache_key = f"api:v2:product:{product_id}"
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)

    product = db.get_product(product_id)
    rating = redis.hgetall(f"agg:product:{product_id}:rating")

    view_model = {
        "id": product.id,
        "title": product.name,
        "price": f"${product.price_cents / 100:.2f}",
        "in_stock": product.stock_count > 0,
        "rating_summary": f"{rating['average']} stars ({rating['count']} reviews)",
        "primary_image": f"https://cdn.example.com/img/{product.id}.jpg",
    }

    redis.setex(cache_key, 300, json.dumps(view_model))
    return view_model
```

**Why view-model caching is powerful:**

- Zero transformation on cache hit — the serialized response is ready to send.
- API versioning is handled naturally — `api:v1:product:42` vs `api:v2:product:42`.
- Backend refactoring does not affect the cache contract.

### Versioned Cache Schemas

When the shape of your cached object changes (e.g., a field is renamed or added), old entries
still have the old shape. See [Schema Versioning and Evolution](#schema-versioning-and-evolution)
for how to handle this.

---

## Redis Data Structure Selection

### Data Structure Overview

Redis offers multiple data structures, each optimized for specific access patterns. Choosing
the right structure is a critical part of cache schema design.

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                 REDIS DATA STRUCTURE LANDSCAPE                  │
  │                                                                 │
  │  Strings      Hashes       Lists        Sets        Sorted Sets │
  │  ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐    ┌──────────┐  │
  │  │"val" │    │f1: v1│    │[a,b, │    │{a,b, │    │{a:1.0,   │  │
  │  │      │    │f2: v2│    │ c,d] │    │ c}   │    │ b:2.5}   │  │
  │  └──────┘    │f3: v3│    └──────┘    └──────┘    └──────────┘  │
  │              └──────┘                                           │
  │  HyperLogLog  Streams              JSON (RedisJSON module)     │
  │  ┌──────┐    ┌──────────┐          ┌────────────────────┐      │
  │  │~count│    │ID: {k:v} │          │{"nested": {"doc"}} │      │
  │  │ ≈12KB│    │ID: {k:v} │          └────────────────────┘      │
  │  └──────┘    └──────────┘                                      │
  └─────────────────────────────────────────────────────────────────┘
```

### Decision Matrix

| Access Pattern                         | Recommended Structure | Why                                       |
|----------------------------------------|-----------------------|-------------------------------------------|
| Full object get/set (JSON blob)        | String                | Simple, one `GET`/`SET`                   |
| Individual field reads on an entity    | Hash                  | `HGET` single field without full deserialize |
| Ranked list (leaderboard, top-N)       | Sorted Set            | `ZRANGEBYSCORE`, `ZRANK` in O(log N)      |
| Ordered timeline / activity feed       | List                  | `LPUSH` + `LRANGE` for recent items       |
| Membership check / unique collection   | Set                   | `SISMEMBER` in O(1)                       |
| Approximate unique count (e.g., UV)    | HyperLogLog           | 12 KB per counter regardless of cardinality|
| Event log / time-series append         | Stream                | Consumer groups, range queries by time     |
| Nested document with partial updates   | RedisJSON             | JSONPath queries, partial updates          |
| Simple counter or flag                 | String (INCR)         | Atomic increment, no serialization         |

### Strings — Serialized Objects

The most common approach: serialize the entire object (JSON, MessagePack, Protobuf) and store
it as a single String value.

```bash
# Store a serialized product object
SET product:42 '{"id":42,"name":"Headphones","price":7999}' EX 600

# Retrieve it
GET product:42
```

**Pros:** Simple, works with any serialization format, single round trip.
**Cons:** Must deserialize the entire object to read a single field. Updates require
read-modify-write.

### Hashes — User Profiles and Entities

Hashes allow individual field access without full deserialization. Ideal for entities where
different consumers read different fields.

```bash
# Store a user profile as a Hash
HSET user:42 name "Alice" email "alice@example.com" role "admin" locale "en-US"
EXPIRE user:42 300

# Read a single field
HGET user:42 email
# "alice@example.com"

# Read multiple fields
HMGET user:42 name role
# 1) "Alice"
# 2) "admin"

# Update one field without touching others
HSET user:42 locale "fr-FR"
```

**When to use Hashes over Strings:**

- Different callers need different subsets of the object's fields.
- You need to update individual fields atomically.
- The object has fewer than ~128 fields and values under 64 bytes (Redis uses a compact
  ziplist encoding in this range — very memory efficient).

### Sorted Sets — Leaderboards and Rankings

Sorted Sets map members to floating-point scores, allowing ranked queries.

```bash
# Game leaderboard — score is the player's points
ZADD leaderboard 15200 "player:alice"
ZADD leaderboard 14800 "player:bob"
ZADD leaderboard 16500 "player:carol"

# Top 3 players (highest first)
ZREVRANGE leaderboard 0 2 WITHSCORES
# 1) "player:carol"  2) "16500"
# 3) "player:alice"  4) "15200"
# 5) "player:bob"    6) "14800"

# Alice's rank (0-based, highest first)
ZREVRANK leaderboard "player:alice"
# 1
```

Other use cases for Sorted Sets: trending topics (score = engagement count), rate limiting
windows (score = timestamp), priority queues.

### Lists — Activity Feeds and Queues

Lists maintain insertion order and support efficient push/pop and range operations.

```bash
# Push new activity to the feed (newest first)
LPUSH feed:user:42 '{"type":"comment","post_id":99,"ts":1721000000}'
LPUSH feed:user:42 '{"type":"like","post_id":101,"ts":1721003600}'

# Keep only the latest 100 items
LTRIM feed:user:42 0 99

# Read the 20 most recent items
LRANGE feed:user:42 0 19
```

**Best for:** Recent activity feeds, notification queues, and any pattern where you read the
most recent N items.

### Sets — Tags and Unique Visitors

Sets store unique members with O(1) membership checks and support set operations.

```bash
# Products tagged "wireless" and "headphones"
SADD tag:wireless "product:42" "product:55" "product:78"
SADD tag:headphones "product:42" "product:78" "product:90"

# Products matching both tags
SINTER tag:wireless tag:headphones
# 1) "product:42"  2) "product:78"

# Membership check
SISMEMBER tag:wireless "product:42"
# 1 (true)
```

### HyperLogLog — Cardinality Estimation

HyperLogLog estimates the number of unique elements using only ~12 KB of memory, regardless
of how many elements are added (standard error of 0.81%).

```bash
PFADD pageviews:home:2025-07-15 "user:42"
PFADD pageviews:home:2025-07-15 "user:99"
PFADD pageviews:home:2025-07-15 "user:42"  # duplicate, not counted twice
PFCOUNT pageviews:home:2025-07-15
# 2
```

**Use when:** You need approximate unique counts over large sets (millions of visitors) and
cannot afford the memory cost of a full Set.

### Streams — Real-Time Analytics

Streams are append-only log structures with consumer group support, ideal for caching
event-driven analytics data.

```bash
XADD analytics:pageviews * page "/products/42" user_id "42" referrer "google"
XREVRANGE analytics:pageviews + - COUNT 10
```

### RedisJSON — Document Caching

The RedisJSON module allows storing, querying, and partially updating JSON documents natively.

```bash
JSON.SET product:42 $ '{"name":"Headphones","specs":{"battery_hours":30},"tags":["wireless"]}'
JSON.GET product:42 $.specs.battery_hours       # partial read: [30]
JSON.SET product:42 $.specs.battery_hours 35    # partial update
JSON.ARRAPPEND product:42 $.tags '"bluetooth"'  # append to array
```

**When to prefer RedisJSON over Strings:** You need partial reads/updates on nested
documents, multiple writers update different parts, or you want server-side JSONPath queries.

### Storage Efficiency Comparison

| Structure   | 1M Entries Overhead | Partial Read | Partial Update | Sorted Access |
|-------------|---------------------|--------------|----------------|---------------|
| String      | Low                 | No           | No (read-modify-write) | No     |
| Hash        | Very low (ziplist)  | Yes          | Yes            | No            |
| Sorted Set  | Moderate            | By rank/score| Score only     | Yes           |
| List        | Low                 | By index     | By index       | Insertion order|
| Set         | Low                 | No           | Add/Remove     | No            |
| RedisJSON   | Moderate            | Yes (JSONPath)| Yes (JSONPath)| No            |

---

## Cache Granularity

### Fine-Grained vs Coarse-Grained Caching

Granularity determines how much data lives in a single cache entry, impacting hit ratio,
memory efficiency, and invalidation complexity.

| Example                              | Style          | Keys | GETs/read | Invalidation     |
|--------------------------------------|----------------|------|-----------|------------------|
| `user:42:name`, `user:42:email`, ... | Fine-grained   | N    | N         | Per-field        |
| `user:42` → `{name, email, ...}`     | Coarse-grained | 1    | 1         | All-or-nothing   |

### Single-Entity vs Collection Caching

| Aspect              | Single-Entity Cache                   | Collection Cache                       |
|---------------------|---------------------------------------|----------------------------------------|
| **Key example**     | `product:42`                          | `products:category:7:page:1`           |
| **Contains**        | One entity                            | A list of entities (or IDs)            |
| **Invalidation**    | Invalidate on entity update           | Invalidate on any member update        |
| **Hit ratio**       | Higher (specific lookups)             | Lower (many possible query variants)   |
| **Use case**        | Profile pages, detail views           | Search results, listing pages          |

A common hybrid approach: cache the **collection as a list of IDs**, and cache each entity
individually. The collection key stores `[42, 55, 78]` and you then `MGET product:42
product:55 product:78`. This lets you invalidate a single product without blowing away every
collection that contains it.

```python
def get_category_products(category_id: int, page: int) -> list[dict]:
    ids_key = f"category:{category_id}:page:{page}:ids"
    cached_ids = redis.lrange(ids_key, 0, -1)

    if not cached_ids:
        rows = db.query(
            "SELECT id FROM products WHERE category_id = %s LIMIT 20 OFFSET %s",
            category_id, (page - 1) * 20,
        )
        cached_ids = [str(r["id"]) for r in rows]
        redis.rpush(ids_key, *cached_ids)
        redis.expire(ids_key, 300)

    # Fetch each entity — most will be individual cache hits
    product_keys = [f"product:{pid}" for pid in cached_ids]
    return [json.loads(p) if p else populate_product_cache(pid)
            for pid, p in zip(cached_ids, redis.mget(*product_keys))]
```

### Page-Level vs Fragment-Level Caching

| Approach       | Description                         | Pros                       | Cons                              |
|----------------|-------------------------------------|----------------------------|-----------------------------------|
| **Page-level** | Cache entire rendered page/response | Fastest — one read         | Hard to personalize; any change invalidates all |
| **Fragment**   | Cache individual components         | Fine invalidation; mix cached & fresh | Multiple reads; assembly logic |

### Granularity Trade-Offs

| Factor              | Fine-Grained        | Coarse-Grained      |
|---------------------|---------------------|----------------------|
| Read latency        | Higher (more GETs)  | Lower (one GET)      |
| Memory efficiency   | Better (no duplication) | Worse (embedded data) |
| Invalidation scope  | Narrow (targeted)   | Broad (all-or-nothing)|
| Cache hit ratio     | Higher per key      | Lower per key        |
| Implementation cost | Higher              | Lower                |
| Network round trips | More                | Fewer                |

### Granularity Decision Framework

Use this framework to choose granularity for a given data type:

1. **How is the data consumed?** If always together → coarse. If in varying subsets → fine.
2. **How often do parts change independently?** High independent change rate → fine.
3. **What is the read:write ratio?** Extremely read-heavy → coarse (amortize the assembly
   cost across many reads).
4. **How many entities?** Millions of entities with many fields → fine or partial (save
   memory). Thousands of entities → coarse is fine.
5. **How sensitive is the caller to latency?** Sub-millisecond requirement → coarse (one
   round trip). Tens of milliseconds acceptable → fine is OK.

---

## Schema Versioning and Evolution

### Why Cached Data Needs Schema Versioning

Unlike a database where you run a migration and all rows are updated, cached data is
**immutable until it expires or is evicted**. After deploying a code change that expects a
new field, old cache entries remain in the old shape until their TTL expires.

```
  Timeline of a Deployment
  ────────────────────────

  T0: Deploy v2 (expects "currency" field)
  T1: Cache hit → old entry from v1 (no "currency" field) → 💥 KeyError
  T2: TTL expires → cache miss → rebuild in v2 shape → ✓ works
  T3: All entries refreshed → stable

  The gap between T0 and T2 is the danger zone.
```

Without schema versioning, you face:

- **Deserialization errors** when code expects fields that old entries lack.
- **Silent data corruption** when old entries have fields with different semantics.
- **Rollback failures** when reverting to v1 code that cannot read v2 entries.

### Version-in-Key Pattern

Embed the schema version in the cache key itself. A new schema version means new keys;
old entries are ignored and eventually evicted.

```bash
# Version 1 keys
SET api:v1:product:42 '{"id":42,"name":"Headphones","price":79.99}'

# Version 2 keys (added "currency" field)
SET api:v2:product:42 '{"id":42,"name":"Headphones","price":79.99,"currency":"USD"}'
```

```python
SCHEMA_VERSION = "v3"

def cache_key(entity: str, entity_id: int) -> str:
    return f"{entity}:{SCHEMA_VERSION}:{entity_id}"

# Produces: "product:v3:42"
```

**Pros:** Simple to implement. Old versions are completely isolated. Easy rollback — just
change the version constant back.

**Cons:** Doubles memory temporarily (v1 and v2 entries coexist until v1 TTLs expire).
Requires a coordinated version bump across all services that share the cache.

### Version-in-Value Pattern

Store the version number inside the cached value. The consumer checks the version on read
and re-populates if it does not match.

```python
CURRENT_SCHEMA_VERSION = 3

def get_product(product_id: int) -> dict:
    cached = redis.get(f"product:{product_id}")
    if cached:
        data = json.loads(cached)
        if data.get("_schema_version") == CURRENT_SCHEMA_VERSION:
            return data
        # Stale schema version — treat as cache miss

    # Rebuild with current schema
    product = build_product_object(product_id)
    product["_schema_version"] = CURRENT_SCHEMA_VERSION
    redis.setex(f"product:{product_id}", 600, json.dumps(product))
    return product
```

**Pros:** No key proliferation. Gradual migration — entries are upgraded on read.

**Cons:** Every read must check the version. Old-schema entries occupy memory until accessed
or evicted.

### Rolling Cache Migrations

For large-scale schema changes, a rolling migration avoids thundering herd problems:

1. **Deploy new code** that writes the new schema but reads both old and new.
2. **Gradually re-populate** hot keys via background workers or natural traffic.
3. **Monitor** the ratio of old-schema vs new-schema cache hits.
4. **Remove backward compatibility** once old entries have fully expired.

### Backward-Compatible Changes

Some schema changes are inherently safe:

| Change Type             | Backward Compatible? | Notes                                       |
|-------------------------|----------------------|---------------------------------------------|
| Add optional field      | ✓ Yes                | Old entries lack it; set a default on read   |
| Remove unused field     | ✓ Yes                | Old entries have it; ignore it               |
| Rename field            | ✗ No                 | Old entries use the old name                 |
| Change field type       | ✗ No                 | Old entries have the old type                |
| Change field semantics  | ✗ No                 | Same name, different meaning is dangerous    |

For backward-compatible changes, no version bump is needed — just handle the missing field
gracefully in code:

```python
currency = cached_product.get("currency", "USD")  # default for old entries
```

### Handling Schema Mismatches During Deployments

During blue-green or rolling deployments, two versions of your application run simultaneously.
Both may read and write cache entries. Strategies:

1. **Version-in-key with shared prefix.** New instances use `v2:product:42`; old instances
   use `v1:product:42`. No conflict.
2. **Additive-only changes.** Only add new fields; never remove or rename. Both versions
   can read the same entries.
3. **Feature flags.** Gate new-schema writes behind a feature flag. Enable only after all
   instances are on the new version.

---

## Relationship Patterns in Cache

### Embedded (Denormalized) Pattern

Inline all related data inside the parent cache entry.

```json
{
  "order_id": 1001,
  "customer_name": "Alice",
  "customer_email": "alice@example.com",
  "items": [
    {"product_id": 42, "product_name": "Headphones", "qty": 1, "price_cents": 7999},
    {"product_id": 55, "product_name": "USB Cable", "qty": 2, "price_cents": 999}
  ],
  "shipping_address": {
    "street": "123 Main St",
    "city": "Portland",
    "state": "OR"
  }
}
```

**Best for:** Data that is always read together and changes infrequently, such as a
completed order.

### Reference-Based Pattern

Store entity IDs and fetch related objects separately. Best for entities shared across many
parents or updated frequently.

```json
{
  "order_id": 1001,
  "customer_id": 42,
  "item_ids": [42, 55],
  "shipping_address_id": 7
}
```

The caller fetches `user:42`, `product:42`, `product:55` as separate cache reads. This
allows independent invalidation of each entity.

### Hybrid Pattern

Embed **stable, small** data; reference **volatile, large** data.

```json
{
  "order_id": 1001,
  "customer_name": "Alice",
  "customer_email": "alice@example.com",
  "item_ids": [42, 55],
  "item_count": 2,
  "total_cents": 9997,
  "shipping_city": "Portland"
}
```

Here, customer name and shipping city are embedded (unlikely to change for this order), but
product details are referenced (product name or price could be updated).

### Cache Dependency Graphs

When cache entries reference each other, invalidating one entry may require invalidating or
refreshing others. Map these dependencies explicitly.

```
  ┌──────────────┐       depends on       ┌──────────────┐
  │ order:1001   │ ──────────────────────► │ product:42   │
  │              │                         │              │
  │ (embeds name)│                         │ name changed │
  └──────────────┘                         └──────────────┘
         │                                        │
         │                                        ▼
         │  also depends on               ┌──────────────┐
         └───────────────────────────────►│ user:42      │
                                          └──────────────┘
```

Approaches for managing dependencies:

- **Tag-based invalidation** — Tag entries with their dependencies. When a dependency
  changes, invalidate all tagged entries. See
  [05-CACHE-INVALIDATION](05-CACHE-INVALIDATION.md).
- **Dependency registry** — Maintain a Set in Redis: `deps:product:42` → `{order:1001,
  order:1055, ...}`. On product update, iterate and invalidate.
- **Short TTLs** — Set short TTLs on entries with many dependencies and let them refresh
  naturally.

### When Embedding Becomes Too Expensive

Embedding is the wrong choice when the embedded entity changes frequently, is large, is
embedded in hundreds of parents (update fan-out is massive), or needs independent invalidation.

**Rule of thumb:** If updating entity B requires invalidating more than ~50 cache entries
that embed B, switch to the reference-based or hybrid pattern.

---

## Multi-Model Caching

### Same Data, Multiple Shapes

A single source entity often needs to be cached in multiple shapes to serve different access
patterns.

```
  Source: users table (single row)
  ────────────────────────────────

  Shape 1: Lookup by ID              Shape 2: Lookup by email
  user:42 → { full profile }         user_email:alice@ex.com → 42

  Shape 3: Admin user list           Shape 4: Leaderboard entry
  admin:users:page:1 → [42,99,...]   leaderboard → { "user:42": 1520, ... }
```

```python
def on_user_updated(user: dict):
    """Update all cached shapes when a user changes."""
    pipeline = redis.pipeline()

    # Shape 1: Full profile by ID
    pipeline.setex(f"user:{user['id']}", 300, json.dumps(user))

    # Shape 2: Email-to-ID index
    pipeline.setex(f"user_email:{user['email']}", 300, str(user["id"]))

    # Shape 3: Invalidate admin list (will be rebuilt on next read)
    pipeline.delete("admin:users:page:1")

    # Shape 4: Update leaderboard score if applicable
    if user.get("score") is not None:
        pipeline.zadd("leaderboard", {f"user:{user['id']}": user["score"]})

    pipeline.execute()
```

### Keeping Derived Views Consistent

When the same data exists in multiple cache shapes, an update to the source must propagate
to **all** shapes. Failure to do so results in views that disagree.

**Strategies for consistency:**

| Strategy                    | Description                                                | Trade-Off                           |
|-----------------------------|------------------------------------------------------------|-------------------------------------|
| **Atomic pipeline**         | Update all shapes in a single Redis pipeline               | All-or-nothing within one node      |
| **Event-driven fan-out**    | Publish a change event; consumers update their own shapes  | Eventual consistency, decoupled     |
| **Write-through layer**     | A single cache-write function handles all shapes           | Centralized logic, single point     |
| **TTL convergence**         | Give all shapes the same TTL; accept brief inconsistency   | Simplest, least precise             |

### Fan-Out on Write vs Fan-Out on Read

| Approach             | Write Time                               | Read Time                           | Best When                              |
|----------------------|------------------------------------------|-------------------------------------|----------------------------------------|
| **Fan-out on write** | Update all shapes at write time          | Single GET per shape                | Reads >> writes, few shapes, consistency matters |
| **Fan-out on read**  | Only primary shape cached                | Derive other shapes on read         | High write volume, many query shapes, staleness OK |

---

## Patterns for Common Domains

### E-Commerce

**Product catalog** — pre-joined from multiple tables into one cached object:

```python
product_cache = {
    "id": 42,
    "name": "Wireless Headphones",
    "price_cents": 7999,
    "currency": "USD",
    "category": "Audio",           # from categories table
    "brand": "SoundCo",           # from brands table
    "primary_image": "/img/42.jpg", # from product_images table
    "avg_rating": 4.5,            # pre-computed aggregate
    "review_count": 312,
    "in_stock": True,             # from inventory table
}
# Key: product:42   TTL: 600s
```

**Shopping cart** — Hash per session, fields are product IDs:

```bash
HSET cart:session:abc123 "product:42" 1
HSET cart:session:abc123 "product:55" 2
HINCRBY cart:session:abc123 "product:42" 1  # increment quantity
HGETALL cart:session:abc123
```

**Inventory counts** — atomic counters for high write frequency:

```bash
SET inventory:42 47
DECRBY inventory:42 1  # purchase
INCRBY inventory:42 5  # restock
```

### Social Media

**Activity feed** — List-based, bounded, newest first:

```bash
LPUSH feed:user:42 '{"actor":"user:99","type":"post","id":5001,"ts":1721000000}'
LTRIM feed:user:42 0 499  # cap at 500 items
LRANGE feed:user:42 0 19  # read latest 20
```

**Follower counts** — Hash fields with atomic increments:

```bash
HSET social:user:42 followers 15234 following 342 posts 891
HINCRBY social:user:42 followers 1
```

**Trending topics** — Sorted Set with engagement score:

```bash
ZINCRBY trending:2025-07-15 1 "topic:redis-caching"
ZREVRANGE trending:2025-07-15 0 9 WITHSCORES  # top 10
```

### SaaS Platforms

**Tenant configuration** — Hash for per-field reads:

```bash
HSET tenant:acme_corp plan "enterprise" max_users "500" features "sso,audit_log,api_access"
EXPIRE tenant:acme_corp 3600
HGET tenant:acme_corp plan
```

**Feature flags** — Set per feature, members are tenant IDs:

```bash
SADD feature:new_dashboard "tenant:acme" "tenant:globex" "tenant:initech"
SISMEMBER feature:new_dashboard "tenant:acme"  # 1 (true)
```

**Session data** — Hash with per-request field reads:

```bash
HSET session:tok_abc123 user_id "42" role "admin" tenant "acme" ip "10.0.1.5"
EXPIRE session:tok_abc123 1800  # 30-minute session
HMGET session:tok_abc123 user_id role tenant
```

### API Response Caching

**REST resource caching** — cache the exact response body:

```python
cache_key = f"api:v2:products:{product_id}"
cached_response = redis.get(cache_key)
if cached_response:
    return Response(content=cached_response, media_type="application/json")
```

**GraphQL response caching** — hash the query + variables for a deterministic key:

```python
import hashlib

def graphql_cache_key(query: str, variables: dict) -> str:
    payload = json.dumps({"query": query, "variables": variables}, sort_keys=True)
    query_hash = hashlib.sha256(payload.encode()).hexdigest()[:16]
    return f"gql:{query_hash}"
```

**Paginated list caching** — one key per page, filter hash in key:

```python
def cache_paginated_list(resource: str, filters: dict, page: int, items: list):
    filters_hash = hashlib.md5(
        json.dumps(filters, sort_keys=True).encode()
    ).hexdigest()[:8]
    key = f"list:{resource}:{filters_hash}:page:{page}"
    redis.setex(key, 120, json.dumps(items))
```

---

## Anti-Patterns

### Over-Caching

**The problem:** Caching entire database tables — including rows that are rarely or never
read. This wastes memory and increases invalidation overhead. In a 2 million-row product
table, typically only the top 5% by traffic are read frequently — caching all rows wastes
~95% of cache memory.

**Fix:** Use cache-aside (lazy loading). Only cache what is actually requested. Combine with
eviction policies (LFU or allkeys-lru) to let Redis evict cold entries naturally.

### Stale Graph Fragments

**The problem:** A cached composite object embeds data from entity B. Entity B changes, but
only B's own cache entry is invalidated — the parent entry still has the old embedded copy.

**Fix:** Track dependencies (see [Relationship Patterns](#relationship-patterns-in-cache)).
When `product:42` changes, invalidate all entries that embed it. Or use the reference-based
pattern for frequently-changing fields.

### Cache-as-Database Trap

**The problem:** Treating the cache as the primary data store. If the cache is lost (crash,
eviction, scale event), data is gone.

**Fix:** Every cached entry must be rebuildable from the source of truth. Set TTLs on all
entries. Ensure cache-miss paths exist and are tested.

### Unbounded Collection Caching

**The problem:** Caching a list or set that grows without limit. An activity feed list
that is never trimmed, a "recent searches" set with no cap.

```bash
# BAD: Unbounded list — grows forever
LPUSH searches:user:42 "redis caching"

# GOOD: Bounded list — trim to latest 100
LPUSH searches:user:42 "redis caching"
LTRIM searches:user:42 0 99
```

**Fix:** Always set a maximum size for collection-type cache entries. Use `LTRIM` for Lists,
`ZREMRANGEBYRANK` for Sorted Sets, or `SRANDMEMBER` + `SREM` for Sets.

### Ignoring Serialization Cost

**The problem:** Caching large objects (100 KB+) without considering CPU cost of
serialization. JSON is human-readable but slow and large.

| Format       | Relative Size | Relative Serialize Time | Relative Deserialize Time |
|--------------|---------------|-------------------------|---------------------------|
| JSON         | 1.0× (baseline)| 1.0×                   | 1.0×                      |
| MessagePack  | 0.67×         | 0.3×                    | 0.3×                      |
| Protobuf     | 0.49×         | 0.2×                    | 0.15×                     |

**Fix:** For hot paths with large objects, benchmark serialization formats. Use MessagePack
or Protobuf for production. Reserve JSON for debugging and small objects.

### Tight Coupling Between Cache and DB Schema

**The problem:** The cache entry structure mirrors the database table exactly. Any database
migration (add column, rename table, change type) forces a synchronized cache change.

**Symptoms:** `SELECT *` piped directly to `redis.set()`; adding a DB column breaks cache
deserialization; database refactoring blocked by cache compatibility.

**Fix:** Define an explicit cache schema that is independent of the database schema. Map
from DB to cache schema in a dedicated builder function.

```python
# BAD: Tight coupling — SELECT * piped directly to cache
def cache_user(user_id):
    row = db.query("SELECT * FROM users WHERE id = %s", user_id)
    redis.set(f"user:{user_id}", json.dumps(dict(row)))

# GOOD: Explicit cache schema — decoupled from DB columns
def cache_user(user_id):
    row = db.query("SELECT id, name, email, avatar_url FROM users WHERE id = %s", user_id)
    cache_obj = {"id": row["id"], "display_name": row["name"],
                 "email": row["email"], "avatar": row["avatar_url"]}
    redis.setex(f"user:{user_id}", 300, json.dumps(cache_obj))
```

---

## Schema Design Checklist

Use this checklist when designing a new cache schema:

**Access Pattern Analysis**

- [ ] Identified the primary access patterns (by ID, by email, by list, etc.)
- [ ] Determined which fields are needed per access pattern
- [ ] Measured read-to-write ratio for the data

**Object Shape**

- [ ] Cache schema is independent of database schema
- [ ] Pre-joined related data that is always read together
- [ ] Excluded cold fields that are not needed on hot paths
- [ ] Pre-computed aggregates where applicable
- [ ] Chosen embedding vs referencing for relationships

**Data Structure Selection**

- [ ] Selected the appropriate Redis data structure for each access pattern
- [ ] Used Hashes for entities with per-field access
- [ ] Used Sorted Sets for ranked/scored data
- [ ] Used HyperLogLog for approximate cardinality
- [ ] Benchmarked serialization format for large objects

**Granularity**

- [ ] Chosen appropriate granularity (fine vs coarse) for each data type
- [ ] Bounded all collection-type cache entries (Lists, Sets, Sorted Sets)
- [ ] Considered fragment-level caching for composite pages

**Versioning and Evolution**

- [ ] Added schema version (in key or in value) for non-trivial objects
- [ ] Ensured backward compatibility for additive changes
- [ ] Planned a migration strategy for breaking changes
- [ ] Tested cache behavior during rolling deployments

**Multi-Model**

- [ ] Identified all access patterns that need separate cached shapes
- [ ] Chosen fan-out-on-write or fan-out-on-read per shape
- [ ] Ensured all derived views are updated or invalidated on source change

**Resilience**

- [ ] Every cache entry can be rebuilt from the source of truth
- [ ] All entries have a TTL (no infinite-retention entries)
- [ ] Cache-miss code paths exist and are tested
- [ ] Serialization errors are handled gracefully (treat as cache miss)

---

## Version History

| Date       | Change          | Author |
|------------|-----------------|--------|
| 2025-07-15 | Initial version | Team   |
