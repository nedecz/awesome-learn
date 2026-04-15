# Redis

## Table of Contents

- [Overview](#overview)
  - [What is Redis](#what-is-redis)
  - [History](#history)
  - [Architecture — Single-Threaded Event Loop](#architecture--single-threaded-event-loop)
  - [Why Redis is Fast](#why-redis-is-fast)
  - [RESP Protocol](#resp-protocol)
  - [Licensing](#licensing)
- [Data Structures](#data-structures)
  - [Strings](#strings)
  - [Lists](#lists)
  - [Sets](#sets)
  - [Sorted Sets](#sorted-sets)
  - [Hashes](#hashes)
  - [Bitmaps](#bitmaps)
  - [HyperLogLog](#hyperloglog)
  - [Streams](#streams)
  - [Geospatial Indexes](#geospatial-indexes)
  - [Data Structures Comparison](#data-structures-comparison)
- [Persistence](#persistence)
  - [RDB Snapshots](#rdb-snapshots)
  - [AOF — Append-Only File](#aof--append-only-file)
  - [Hybrid RDB + AOF](#hybrid-rdb--aof)
  - [No Persistence](#no-persistence)
  - [Persistence Trade-offs Comparison](#persistence-trade-offs-comparison)
- [Redis Cluster](#redis-cluster)
  - [Cluster Architecture](#cluster-architecture)
  - [Hash Slots and CRC16](#hash-slots-and-crc16)
  - [Resharding](#resharding)
  - [Automatic Failover](#automatic-failover)
  - [Cluster Commands](#cluster-commands)
- [Redis Sentinel](#redis-sentinel)
  - [Sentinel Architecture](#sentinel-architecture)
  - [Monitoring and Failover](#monitoring-and-failover)
  - [Sentinel Configuration](#sentinel-configuration)
- [Pub/Sub](#pubsub)
  - [Channels and Pattern Subscriptions](#channels-and-pattern-subscriptions)
  - [Pub/Sub Limitations](#pubsub-limitations)
  - [Pub/Sub Use Cases](#pubsub-use-cases)
- [Redis Streams](#redis-streams)
  - [Stream Commands](#stream-commands)
  - [Consumer Groups](#consumer-groups)
  - [Pending Entries List](#pending-entries-list)
  - [Stream Processing Patterns](#stream-processing-patterns)
  - [Streams vs Kafka](#streams-vs-kafka)
- [Lua Scripting and Functions](#lua-scripting-and-functions)
  - [EVAL and EVALSHA](#eval-and-evalsha)
  - [Redis 7 Functions API](#redis-7-functions-api)
  - [Common Script Examples](#common-script-examples)
- [Redis Modules](#redis-modules)
  - [RedisJSON](#redisjson)
  - [RediSearch](#redisearch)
  - [RedisTimeSeries](#redistimeseries)
  - [RedisBloom](#redisbloom)
  - [RedisGraph (Deprecated)](#redisgraph-deprecated)
- [Transactions](#transactions)
  - [MULTI / EXEC / DISCARD](#multi--exec--discard)
  - [Optimistic Locking with WATCH](#optimistic-locking-with-watch)
  - [Pipeline vs Transaction](#pipeline-vs-transaction)
- [Memory Management](#memory-management)
  - [maxmemory Configuration](#maxmemory-configuration)
  - [Eviction Policies](#eviction-policies)
  - [Memory Optimization](#memory-optimization)
  - [INFO MEMORY](#info-memory)
- [Security](#security)
  - [ACLs — Access Control Lists](#acls--access-control-lists)
  - [TLS Encryption](#tls-encryption)
  - [AUTH and Protected Mode](#auth-and-protected-mode)
  - [Network Security](#network-security)
  - [RENAME-COMMAND](#rename-command)
- [Common Patterns](#common-patterns)
  - [Rate Limiting — Sliding Window](#rate-limiting--sliding-window)
  - [Distributed Locks — Redlock Algorithm](#distributed-locks--redlock-algorithm)
  - [Session Storage](#session-storage)
  - [Leaderboards](#leaderboards)
  - [Caching Layer](#caching-layer)
  - [Job Queues](#job-queues)
- [Performance Tuning](#performance-tuning)
  - [Pipelining](#pipelining)
  - [Connection Pooling](#connection-pooling)
  - [Key Design Best Practices](#key-design-best-practices)
  - [SCAN vs KEYS](#scan-vs-keys)
  - [Avoiding Large Values](#avoiding-large-values)
  - [Slow Log](#slow-log)
- [Client Libraries](#client-libraries)
- [Version History](#version-history)

---

## Overview

### What is Redis

Redis (Remote Dictionary Server) is an open-source, in-memory data structure store
used as a database, cache, message broker, and streaming engine. Unlike traditional
key-value stores that only support string values, Redis provides a rich set of data
structures including strings, lists, sets, sorted sets, hashes, bitmaps, HyperLogLog,
streams, and geospatial indexes.

```
+-----------------------------------------------------------+
|                        APPLICATION                        |
+-----------------------------------------------------------+
          |                    |                  |
          v                    v                  v
   +------------+      +------------+      +------------+
   |   Cache    |      |  Database  |      |  Message   |
   |   Layer    |      |  (Primary) |      |  Broker    |
   +------------+      +------------+      +------------+
          |                    |                  |
          +--------------------+------------------+
                               |
                    +---------------------+
                    |       REDIS         |
                    |  In-Memory Store    |
                    |                     |
                    |  - Sub-millisecond  |
                    |    latency          |
                    |  - Rich data types  |
                    |  - Persistence      |
                    |  - Replication      |
                    |  - Clustering       |
                    +---------------------+
```

### History

Redis was created by **Salvatore Sanfilippo** (antirez) in 2009. Sanfilippo initially
developed Redis to improve the scalability of his Italian startup, LLOOGG, a real-time
web analytics tool. He needed a system that could handle high write loads faster than
what traditional databases offered.

Key milestones:

- **2009** — First public release of Redis
- **2010** — VMware sponsors Redis development; Sanfilippo joins VMware
- **2013** — Pivotal takes over Redis sponsorship
- **2015** — Redis Labs (later Redis Inc.) founded; Redis Cluster released in v3.0
- **2018** — Redis 5.0 introduces Streams
- **2020** — Redis 6.0 introduces ACLs, TLS, and client-side caching
- **2022** — Redis 7.0 introduces Functions API and Sharded Pub/Sub
- **2024** — License change from BSD to RSALv2/SSPLv1 dual license

### Architecture — Single-Threaded Event Loop

Redis uses a single-threaded event loop architecture for command processing. This
is a deliberate design decision that eliminates the overhead of context switching,
locking, and synchronization that come with multi-threaded systems.

```
                    +----------------------------------+
                    |         REDIS SERVER             |
                    |                                  |
  Client A ------->|   +------------------------+     |
                    |   |   Event Loop (epoll/   |     |
  Client B ------->|   |   kqueue)               |     |
                    |   |                        |     |
  Client C ------->|   |  1. Read client request |     |
                    |   |  2. Parse command       |     |
                    |   |  3. Execute command     |     |
  Client D ------->|   |  4. Write response      |     |
                    |   |                        |     |
                    |   +------------------------+     |
                    |              |                    |
                    |              v                    |
                    |   +------------------------+     |
                    |   |    In-Memory Data       |     |
                    |   |    (Hash Tables,        |     |
                    |   |     Skip Lists, etc.)   |     |
                    |   +------------------------+     |
                    |                                  |
                    |   Background Threads:             |
                    |   - Lazy free (UNLINK)            |
                    |   - AOF fsync                     |
                    |   - RDB persistence (fork)        |
                    +----------------------------------+
```

Starting with Redis 6.0, I/O threading was introduced for network read/write
operations, but command execution still happens on the main thread. This maintains
the atomicity guarantees while improving throughput for I/O-bound workloads.

### Why Redis is Fast

Redis can execute **100,000+ operations per second** on a single node. Several
design choices contribute to this:

1. **In-memory storage** — All data resides in RAM; no disk I/O for reads/writes
2. **Single-threaded execution** — No locks, no context switches, no race conditions
3. **Efficient data structures** — Custom implementations (skip lists, zip lists, etc.)
4. **Non-blocking I/O** — Uses epoll (Linux) / kqueue (BSD) multiplexing
5. **RESP protocol** — Simple, binary-safe, fast to parse
6. **No overhead from query parsing** — Commands are simple and predefined
7. **Copy-on-write for persistence** — fork() for RDB snapshots reuses memory pages

### RESP Protocol

Redis uses the **REdis Serialization Protocol (RESP)** for client-server
communication. RESP is a binary-safe text protocol that is simple to implement
and fast to parse.

```
RESP Data Types:
  + Simple Strings   → "+OK\r\n"
  - Errors           → "-ERR unknown command\r\n"
  : Integers         → ":1000\r\n"
  $ Bulk Strings     → "$5\r\nhello\r\n"
  * Arrays           → "*2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n"

Example — SET command over RESP:

  Client sends:
    *3\r\n          ← Array of 3 elements
    $3\r\n          ← Bulk string of 3 bytes
    SET\r\n         ← "SET"
    $5\r\n          ← Bulk string of 5 bytes
    mykey\r\n       ← "mykey"
    $7\r\n          ← Bulk string of 7 bytes
    myvalue\r\n     ← "myvalue"

  Server responds:
    +OK\r\n         ← Simple string
```

RESP3 (introduced in Redis 6.0) adds additional types: Maps, Sets, Booleans,
Doubles, Big Numbers, and Verbatim Strings.

### Licensing

Redis was originally released under the **3-clause BSD license**, making it free
to use, modify, and distribute. In March 2024, Redis Inc. changed the license for
Redis versions 7.4 and later to a dual license:

- **RSALv2** — Redis Source Available License v2
- **SSPLv1** — Server Side Public License v1

This change restricts cloud providers from offering Redis as a managed service
without a commercial agreement. The community responded by creating forks such as
**Valkey** (Linux Foundation) and **Redict** (LGPL-licensed) to continue
development under open-source licenses.

---

## Data Structures

Redis supports nine primary data structures. Each is backed by one or more
optimized internal encodings that Redis selects automatically based on data size.

### Strings

The most basic Redis data type. Strings can hold any data: text, serialized
JSON, binary data, or integers (which support atomic increment/decrement).

**Max size:** 512 MB

```bash
# Basic operations
SET user:1:name "Alice"              # O(1) — Set a string value
GET user:1:name                      # O(1) — Get a string value
# → "Alice"

APPEND user:1:name " Smith"          # O(1) — Append to value
GET user:1:name
# → "Alice Smith"

# Numeric operations
SET counter 100                      # O(1)
INCR counter                         # O(1) — Atomic increment
# → 101
INCRBY counter 50                    # O(1) — Increment by N
# → 151
DECR counter                         # O(1) — Atomic decrement
# → 150
INCRBYFLOAT counter 1.5              # O(1) — Increment by float
# → "151.5"

# Expiration
SET session:abc123 "data" EX 3600    # O(1) — Set with TTL (seconds)
SET token:xyz "data" PX 5000         # O(1) — Set with TTL (milliseconds)
TTL session:abc123                   # O(1) — Check remaining TTL
# → 3598

# Conditional set
SET lock:resource "owner" NX         # O(1) — Set only if Not eXists
SET lock:resource "owner" XX         # O(1) — Set only if eXists

# Multiple keys
MSET key1 "v1" key2 "v2" key3 "v3"  # O(N) — Set multiple keys
MGET key1 key2 key3                  # O(N) — Get multiple keys
# → ["v1", "v2", "v3"]

# String range operations
SETRANGE user:1:name 0 "Bob"         # O(1) — Overwrite at offset
GETRANGE user:1:name 0 2             # O(N) — Get substring
# → "Bob"
STRLEN user:1:name                   # O(1) — Get string length
```

**Use cases:** Caching, counters, rate limiters, session tokens, feature flags.

### Lists

Ordered collections of strings implemented as linked lists (quicklist encoding).
Elements can be added or removed from both head and tail in O(1).

```bash
# Push operations
LPUSH tasks "task3" "task2" "task1"  # O(N) — Push to head
RPUSH tasks "task4" "task5"          # O(N) — Push to tail
# tasks → ["task1", "task2", "task3", "task4", "task5"]

# Pop operations
LPOP tasks                           # O(1) — Pop from head
# → "task1"
RPOP tasks                           # O(1) — Pop from tail
# → "task5"

# Blocking pop (waits up to 30s for element)
BLPOP queue:jobs 30                  # O(1) — Blocking left pop
BRPOP queue:jobs 30                  # O(1) — Blocking right pop

# Range operations
LRANGE tasks 0 -1                    # O(S+N) — Get all elements
# → ["task2", "task3", "task4"]
LRANGE tasks 0 1                     # O(S+N) — Get first 2 elements
# → ["task2", "task3"]

# Index operations
LINDEX tasks 0                       # O(N) — Get by index
# → "task2"
LSET tasks 0 "updated_task2"         # O(N) — Set by index
LLEN tasks                           # O(1) — Get list length
# → 3

# Trim (keep only range)
LTRIM tasks 0 99                     # O(N) — Keep first 100 elements

# Move between lists
LMOVE source dest LEFT RIGHT         # O(1) — Atomic move
```

**Use cases:** Message queues, activity feeds, most-recent-items lists, task queues.

### Sets

Unordered collections of unique strings. Sets support membership tests,
intersections, unions, and differences.

```bash
# Add/Remove members
SADD tags:post:1 "redis" "cache" "database"  # O(N) — Add members
SREM tags:post:1 "cache"                      # O(N) — Remove members

# Membership test
SISMEMBER tags:post:1 "redis"                 # O(1) — Check membership
# → 1 (true)

# Get all members
SMEMBERS tags:post:1                          # O(N) — Get all members
# → ["redis", "database"]

# Set size
SCARD tags:post:1                             # O(1) — Cardinality
# → 2

# Random member
SRANDMEMBER tags:post:1                       # O(1) — Random element
SPOP tags:post:1                              # O(1) — Random pop

# Set operations
SADD tags:post:2 "redis" "performance" "nosql"

SINTER tags:post:1 tags:post:2               # O(N*M) — Intersection
# → ["redis"]
SUNION tags:post:1 tags:post:2               # O(N) — Union
# → ["redis", "database", "performance", "nosql"]
SDIFF tags:post:1 tags:post:2                # O(N) — Difference
# → ["database"]

# Store results
SINTERSTORE dest tags:post:1 tags:post:2     # O(N*M) — Store intersection
```

**Use cases:** Tagging, unique visitors, mutual friends, IP allowlists.

### Sorted Sets

Like Sets, but every member has an associated floating-point score. Members are
unique; scores may repeat. Sorted sets are implemented using a **skip list** and
a **hash table**, giving O(log N) for most operations.

```bash
# Add members with scores
ZADD leaderboard 1500 "alice"        # O(log N) — Add member
ZADD leaderboard 2300 "bob"
ZADD leaderboard 1800 "charlie"
ZADD leaderboard 2100 "diana"

# Score operations
ZSCORE leaderboard "bob"             # O(1) — Get score
# → "2300"
ZINCRBY leaderboard 100 "alice"      # O(log N) — Increment score
# → "1600"

# Rank operations (0-indexed, ascending by default)
ZRANK leaderboard "bob"              # O(log N) — Rank (low to high)
# → 3
ZREVRANK leaderboard "bob"           # O(log N) — Rank (high to low)
# → 0

# Range queries
ZRANGE leaderboard 0 -1 WITHSCORES  # O(log N + M) — All by score asc
# → ["alice","1600","charlie","1800","diana","2100","bob","2300"]

ZREVRANGE leaderboard 0 2           # O(log N + M) — Top 3 by score desc
# → ["bob", "diana", "charlie"]

# Range by score
ZRANGEBYSCORE leaderboard 1500 2000 WITHSCORES   # O(log N + M)
# → ["alice","1600","charlie","1800"]

# Count in range
ZCOUNT leaderboard 1500 2000        # O(log N) — Count in score range
# → 2

# Cardinality
ZCARD leaderboard                   # O(1)
# → 4

# Remove
ZREM leaderboard "alice"            # O(log N) — Remove member
ZREMRANGEBYSCORE leaderboard 0 1000 # O(log N + M) — Remove by score
ZREMRANGEBYRANK leaderboard 0 1     # O(log N + M) — Remove by rank
```

**Use cases:** Leaderboards, priority queues, rate limiters, time-based feeds.

### Hashes

Maps of field-value pairs, ideal for representing objects. Small hashes use a
compact **listpack** encoding; larger ones use a hash table.

```bash
# Set fields
HSET user:1 name "Alice" email "alice@example.com" age 30  # O(N)

# Get fields
HGET user:1 name                     # O(1)
# → "Alice"
HMGET user:1 name email              # O(N)
# → ["Alice", "alice@example.com"]
HGETALL user:1                       # O(N) — All fields and values
# → {"name":"Alice","email":"alice@example.com","age":"30"}

# Field operations
HEXISTS user:1 name                  # O(1) — Check field exists
# → 1
HDEL user:1 age                      # O(N) — Delete fields
HLEN user:1                          # O(1) — Number of fields
# → 2

# Numeric fields
HSET user:1 login_count 0
HINCRBY user:1 login_count 1         # O(1) — Increment integer field
# → 1
HINCRBYFLOAT user:1 balance 10.50    # O(1) — Increment float field

# Get all keys or values
HKEYS user:1                         # O(N) — All field names
HVALS user:1                         # O(N) — All field values

# Conditional set
HSETNX user:1 name "Bob"             # O(1) — Set only if not exists
# → 0 (field already exists)
```

**Use cases:** Object storage, user profiles, configuration, counters per entity.

### Bitmaps

Not a separate data type — bitmaps are operations on Strings that treat them as
arrays of bits. Extremely memory-efficient for tracking boolean states.

```bash
# Set individual bits
SETBIT active:2025-04-15 1001 1      # O(1) — User 1001 was active
SETBIT active:2025-04-15 1002 1      # O(1) — User 1002 was active
SETBIT active:2025-04-15 1003 0      # O(1) — User 1003 was not active

# Get individual bits
GETBIT active:2025-04-15 1001        # O(1)
# → 1

# Count set bits
BITCOUNT active:2025-04-15           # O(N) — Count all active users
# → 2

# Bitwise operations across days
BITOP AND active:both active:2025-04-14 active:2025-04-15
# Users active on BOTH days

BITOP OR active:either active:2025-04-14 active:2025-04-15
# Users active on EITHER day

# Find first set/unset bit
BITPOS active:2025-04-15 1           # O(N) — First active user ID
BITPOS active:2025-04-15 0           # O(N) — First inactive user ID
```

**Memory efficiency:** Tracking 100 million users = 12.5 MB per day.

**Use cases:** Daily active users, feature flags, bloom filter backing, online status.

### HyperLogLog

A probabilistic data structure for estimating the cardinality (number of unique
elements) of a set. Uses ~12 KB per key regardless of the number of elements.
Standard error rate: **0.81%**.

```bash
# Add elements
PFADD visitors:2025-04-15 "user1" "user2" "user3"   # O(1)
PFADD visitors:2025-04-15 "user1" "user4"            # O(1) — Duplicates ignored

# Count unique elements
PFCOUNT visitors:2025-04-15                           # O(1)
# → 4 (approximate)

# Merge multiple HyperLogLogs
PFADD visitors:2025-04-14 "user1" "user5" "user6"
PFMERGE visitors:week visitors:2025-04-14 visitors:2025-04-15   # O(N)
PFCOUNT visitors:week
# → 6 (approximate unique visitors across both days)
```

**Use cases:** Unique visitor counting, unique search queries, cardinality estimation.

### Streams

An append-only log data structure, introduced in Redis 5.0. Streams support consumer
groups for distributed message processing. See the dedicated [Redis Streams](#redis-streams)
section for detailed coverage.

```bash
# Add entries
XADD mystream * sensor-id 1234 temperature 19.8  # O(1)
# → "1681554000000-0" (auto-generated ID: timestamp-sequence)

# Read entries
XLEN mystream                                     # O(1) — Stream length
XRANGE mystream - +                               # O(N) — Read all entries
XRANGE mystream - + COUNT 10                      # O(N) — Read first 10
```

### Geospatial Indexes

Geospatial indexes let you store coordinates (longitude/latitude) and query them.
Internally, they use Sorted Sets with geohash-encoded scores.

```bash
# Add locations
GEOADD restaurants -122.4194 37.7749 "pizza-place"    # O(log N)
GEOADD restaurants -122.4089 37.7837 "sushi-bar"
GEOADD restaurants -122.4058 37.7903 "taco-shop"

# Get position
GEOPOS restaurants "pizza-place"                       # O(1)
# → ["-122.4194", "37.7749"]

# Distance between two members
GEODIST restaurants "pizza-place" "sushi-bar" km        # O(1)
# → "1.2345"

# Search by radius
GEOSEARCH restaurants FROMMEMBER "pizza-place" BYRADIUS 2 km ASC  # O(N+log N)
# → ["pizza-place", "sushi-bar", "taco-shop"]

# Search by bounding box
GEOSEARCH restaurants FROMLONLAT -122.41 37.78 BYBOX 5 5 km ASC  # O(N+log N)

# Get geohash
GEOHASH restaurants "pizza-place"                      # O(1)
# → ["9q8yyk8yuv0"]
```

**Use cases:** Nearby search, delivery radius, store locators, geofencing.

### Data Structures Comparison

```
+----------------+----------+------------+---------------------------------+
| Type           | Max Size | Encoding   | Best For                        |
+----------------+----------+------------+---------------------------------+
| Strings        | 512 MB   | int/embstr | Caching, counters, flags        |
|                |          | /raw       |                                 |
| Lists          | 2^32 - 1 | listpack/  | Queues, feeds, recent items     |
|                | elements | quicklist  |                                 |
| Sets           | 2^32 - 1 | listpack/  | Tags, unique items, set ops     |
|                | elements | hashtable  |                                 |
| Sorted Sets    | 2^32 - 1 | listpack/  | Leaderboards, priority queues   |
|                | elements | skiplist   |                                 |
| Hashes         | 2^32 - 1 | listpack/  | Object storage, profiles        |
|                | fields   | hashtable  |                                 |
| Bitmaps        | 512 MB   | string     | Boolean states per ID           |
| HyperLogLog    | 12 KB    | sparse/    | Unique counts (approximate)     |
|                |          | dense      |                                 |
| Streams        | Unlimited| rax tree   | Event sourcing, messaging       |
| Geospatial     | 2^32 - 1 | skiplist   | Location queries                |
|                | elements |            |                                 |
+----------------+----------+------------+---------------------------------+
```

---

## Persistence

Redis provides multiple persistence options that control the trade-off between
durability, performance, and disk usage.

### RDB Snapshots

RDB (Redis Database) produces point-in-time snapshots of the dataset at specified
intervals. Redis forks a child process that writes the entire dataset to a
compact binary `.rdb` file.

```bash
# Manual snapshot commands
SAVE                     # Synchronous — blocks the server (avoid in production)
BGSAVE                   # Asynchronous — forks a child process
LASTSAVE                 # Timestamp of last successful save
```

**Configuration (redis.conf):**

```
# Save after 3600s if at least 1 key changed
# Save after 300s if at least 100 keys changed
# Save after 60s if at least 10000 keys changed
save 3600 1
save 300 100
save 60 10000

# RDB file name and directory
dbfilename dump.rdb
dir /var/lib/redis

# Compress RDB file with LZF
rdbcompression yes

# Verify RDB checksum on load
rdbchecksum yes

# Stop writes if BGSAVE fails
stop-writes-on-bgsave-error yes
```

```
RDB Snapshot Process:

  Main Process                    Child Process
       |                               
       | ---- fork() ----------------->|
       |                               |
       | (continues serving            | (writes dataset
       |  client requests)             |  to dump.rdb)
       |                               |
       | (copy-on-write:               |
       |  shared memory pages          |
       |  until modified)              |
       |                               |--- Write complete
       |<--- Signal ------------------|
       | (replace old RDB)             
       |
```

**Pros:** Compact file, fast restarts, good for backups and disaster recovery.
**Cons:** Data loss between snapshots, fork can be slow with large datasets.

### AOF — Append-Only File

AOF logs every write operation received by the server. On restart, Redis replays
the AOF to reconstruct the dataset.

**Configuration (redis.conf):**

```
appendonly yes
appendfilename "appendonly.aof"

# fsync policies:
# appendfsync always     # Every write — safest, slowest
appendfsync everysec     # Every second — good balance (recommended)
# appendfsync no         # OS decides — fastest, least safe

# AOF rewrite triggers
auto-aof-rewrite-percentage 100   # Rewrite when AOF grows 100% since last rewrite
auto-aof-rewrite-min-size 64mb    # Minimum size before rewrite triggers
```

```
AOF Write Process:

  Client          Redis Server              AOF File
    |                  |                       |
    |--- SET foo bar ->|                       |
    |                  |--- append ----------->|
    |                  |    "*3\r\n$3\r\n      |
    |                  |     SET\r\n$3\r\n     |
    |                  |     foo\r\n$3\r\n     |
    |                  |     bar\r\n"          |
    |<-- +OK ---------|                       |
    |                  |                       |
    |                  |--- fsync ------------>|
    |                  |    (per policy)       |
```

AOF rewrite compacts the file by generating the minimum set of commands to
recreate the current dataset:

```bash
BGREWRITEAOF   # Triggers background AOF rewrite
```

### Hybrid RDB + AOF

Since Redis 4.0, you can enable a hybrid persistence mode. During AOF rewrite,
Redis writes an RDB preamble followed by AOF data accumulated during the rewrite.

```
aof-use-rdb-preamble yes   # Default: yes since Redis 4.0
```

```
Hybrid AOF File Structure:

  +---------------------------+
  |     RDB Preamble          |  ← Point-in-time snapshot (binary)
  |     (compact binary)      |
  +---------------------------+
  |     AOF Tail              |  ← Commands since snapshot
  |     *3\r\n$3\r\nSET...   |
  |     *3\r\n$3\r\nSET...   |
  +---------------------------+
```

**Benefit:** Faster restarts (RDB loads quickly) with near-zero data loss
(AOF tail covers the gap).

### No Persistence

For pure caching scenarios where data loss is acceptable:

```
save ""           # Disable RDB
appendonly no     # Disable AOF
```

### Persistence Trade-offs Comparison

```
+------------------+----------+-----------+----------+----------+-------------+
| Method           | Data     | Restart   | Disk     | Perf     | Best For    |
|                  | Safety   | Speed     | Usage    | Impact   |             |
+------------------+----------+-----------+----------+----------+-------------+
| RDB only         | Minutes  | Fast      | Low      | Low      | Backups,    |
|                  | of loss  | (binary)  | (compact)|          | DR          |
+------------------+----------+-----------+----------+----------+-------------+
| AOF (everysec)   | ~1 sec   | Slow      | High     | Medium   | Durability  |
|                  | of loss  | (replay)  | (text)   |          | focus       |
+------------------+----------+-----------+----------+----------+-------------+
| AOF (always)     | Zero     | Slow      | High     | High     | Financial   |
|                  | loss     | (replay)  | (text)   |          | data        |
+------------------+----------+-----------+----------+----------+-------------+
| Hybrid RDB+AOF   | ~1 sec   | Fast      | Medium   | Medium   | Recommended |
|                  | of loss  | (RDB+tail)|          |          | default     |
+------------------+----------+-----------+----------+----------+-------------+
| None             | Total    | N/A       | None     | None     | Pure cache  |
|                  | loss     |           |          |          |             |
+------------------+----------+-----------+----------+----------+-------------+
```

---

## Redis Cluster

Redis Cluster provides automatic sharding across multiple Redis nodes with
built-in replication and high availability.

### Cluster Architecture

```
+-----------------------------------------------------------------------+
|                          REDIS CLUSTER                                |
|                                                                       |
|  +---------------------+  +---------------------+  +--------------+  |
|  |   Node A (Master)   |  |   Node B (Master)   |  | Node C       |  |
|  |   Slots: 0-5460     |  |   Slots: 5461-10922 |  | (Master)     |  |
|  |                     |  |                     |  | Slots:       |  |
|  |  +---------------+  |  |  +---------------+  |  | 10923-16383  |  |
|  |  | Data Partition|  |  |  | Data Partition|  |  |              |  |
|  |  | (1/3 of keys) |  |  |  | (1/3 of keys) |  |  | +---------+ |  |
|  |  +---------------+  |  |  +---------------+  |  | | Data    | |  |
|  +--------|------------+  +--------|------------+  | | Part.   | |  |
|           |                        |               | +---------+ |  |
|           | replication            | replication    +------|------+  |
|           v                        v                      |         |
|  +---------------------+  +---------------------+        |         |
|  |   Node A1 (Replica) |  |   Node B1 (Replica) |        v         |
|  |   Slots: 0-5460     |  |   Slots: 5461-10922 |  +-----------+   |
|  |   (read replica)    |  |   (read replica)    |  | Node C1   |   |
|  +---------------------+  +---------------------+  | (Replica) |   |
|                                                     +-----------+   |
|                                                                       |
|  All nodes communicate via Cluster Bus (port + 10000)                 |
|  using a gossip protocol for failure detection and configuration      |
+-----------------------------------------------------------------------+
```

### Hash Slots and CRC16

Redis Cluster divides the keyspace into **16,384 hash slots**. The slot for a
given key is computed as:

```
HASH_SLOT = CRC16(key) mod 16384
```

Each master node owns a subset of the 16,384 slots. When a client sends a
command for a key that belongs to a different node, it receives a
**MOVED** or **ASK** redirect:

```bash
# Client connects to Node A and requests a key on Node B
GET user:500
# → -MOVED 12345 192.168.1.2:6379
# Client must redirect to Node B (192.168.1.2:6379)

# Hash tags force keys to the same slot
SET {user:1}.profile "data"     # Slot = CRC16("user:1") mod 16384
SET {user:1}.settings "data"    # Same slot — enables multi-key operations
```

### Resharding

Resharding moves hash slots between nodes. This can be done live without
downtime.

```bash
# Using redis-cli
redis-cli --cluster reshard 192.168.1.1:6379

# Check cluster state
redis-cli --cluster check 192.168.1.1:6379

# Rebalance slots across nodes
redis-cli --cluster rebalance 192.168.1.1:6379
```

During resharding, keys in migrating slots trigger **ASK** redirects:

```
Client                   Source Node              Target Node
  |--- GET key -------->|                         |
  |                     | (slot migrating)        |
  |<- -ASK 1234        |                         |
  |    target:6379 -----|                         |
  |                                               |
  |--- ASKING ---------------------------------->|
  |--- GET key --------------------------------->|
  |<-- value -----------------------------------|
```

### Automatic Failover

When a master node fails, its replica is promoted automatically:

1. Nodes detect the failure via gossip protocol (configurable timeout)
2. Replica nodes of the failed master start an election
3. Other master nodes vote; the replica with the most up-to-date data wins
4. The winning replica is promoted to master and takes over the slots

```
cluster-node-timeout 15000   # 15 seconds to mark node as failing
cluster-replica-validity-factor 10
```

### Cluster Commands

```bash
# Create a cluster (3 masters, 3 replicas)
redis-cli --cluster create \
  192.168.1.1:6379 192.168.1.2:6379 192.168.1.3:6379 \
  192.168.1.4:6379 192.168.1.5:6379 192.168.1.6:6379 \
  --cluster-replicas 1

# Cluster info
CLUSTER INFO                    # Cluster state and statistics
CLUSTER NODES                   # List all nodes with their roles/slots
CLUSTER SLOTS                   # Mapping of slots to nodes
CLUSTER MYID                    # Current node's ID

# Node management
CLUSTER MEET 192.168.1.4 6379  # Add a node to the cluster
CLUSTER FORGET <node-id>        # Remove a node
CLUSTER REPLICATE <node-id>     # Make current node a replica

# Slot management
CLUSTER ADDSLOTS 0 1 2 3       # Assign slots to current node
CLUSTER DELSLOTS 0 1 2 3       # Remove slot assignments
CLUSTER SETSLOT 1234 MIGRATING <node-id>
CLUSTER SETSLOT 1234 IMPORTING <node-id>
CLUSTER SETSLOT 1234 NODE <node-id>

# Failover
CLUSTER FAILOVER               # Manual failover (from replica)
CLUSTER RESET HARD             # Reset node (careful!)
```

---

## Redis Sentinel

Redis Sentinel provides high availability for standalone Redis deployments
(non-cluster mode). It monitors master and replica instances, performs automatic
failover, and serves as a service discovery mechanism for clients.

### Sentinel Architecture

```
+--------------------------------------------------------------------+
|                       SENTINEL SYSTEM                              |
|                                                                    |
|    +----------------+  +----------------+  +----------------+      |
|    |  Sentinel #1   |  |  Sentinel #2   |  |  Sentinel #3   |      |
|    |  (monitoring)  |  |  (monitoring)  |  |  (monitoring)  |      |
|    +-------+--------+  +-------+--------+  +-------+--------+      |
|            |                   |                   |               |
|            +------- gossip ----+------- gossip ----+               |
|            |                   |                   |               |
|            v                   v                   v               |
|    +-------+-------------------------------------------+          |
|    |                                                   |          |
|    |              +-----------------+                  |          |
|    |              |  MASTER         |                  |          |
|    |              |  192.168.1.1    |                  |          |
|    |              |  :6379          |                  |          |
|    |              +--------+--------+                  |          |
|    |                       |                           |          |
|    |            +----------+----------+                |          |
|    |            |                     |                |          |
|    |   +--------v--------+  +--------v--------+       |          |
|    |   |  REPLICA #1     |  |  REPLICA #2     |       |          |
|    |   |  192.168.1.2    |  |  192.168.1.3    |       |          |
|    |   |  :6379          |  |  :6379          |       |          |
|    |   +-----------------+  +-----------------+       |          |
|    |                                                   |          |
|    +---------------------------------------------------+          |
+--------------------------------------------------------------------+

  Sentinel quorum: 2 of 3 must agree before failover
```

### Monitoring and Failover

Sentinel performs two types of failure detection:

- **SDOWN (Subjective Down):** A single Sentinel thinks a node is down
- **ODOWN (Objective Down):** A quorum of Sentinels agrees the master is down

Failover process:

1. Sentinels detect master is unreachable (SDOWN → ODOWN)
2. A Sentinel is elected as leader to perform the failover
3. Leader selects the best replica (based on priority, replication offset, run ID)
4. Selected replica is promoted to master via `REPLICAOF NO ONE`
5. Other replicas are reconfigured to replicate from the new master
6. Clients are notified via Pub/Sub

### Sentinel Configuration

```
# sentinel.conf
port 26379
sentinel monitor mymaster 192.168.1.1 6379 2
#                 name     host         port quorum

sentinel down-after-milliseconds mymaster 30000
# Time to mark master as SDOWN (30 seconds)

sentinel failover-timeout mymaster 180000
# Max time for failover operation (3 minutes)

sentinel parallel-syncs mymaster 1
# How many replicas can sync from new master simultaneously

sentinel auth-pass mymaster your-password
# Password for the monitored master

sentinel deny-scripts-reconfig yes
# Prevent runtime script reconfiguration
```

```bash
# Sentinel commands
SENTINEL masters                         # List monitored masters
SENTINEL master mymaster                 # Info about a specific master
SENTINEL replicas mymaster               # List replicas of a master
SENTINEL sentinels mymaster              # List other Sentinels
SENTINEL get-master-addr-by-name mymaster  # Current master address
SENTINEL failover mymaster               # Force failover
SENTINEL reset mymaster                  # Reset state
```

---

## Pub/Sub

Redis Pub/Sub implements a publish/subscribe messaging paradigm where senders
(publishers) send messages to channels without knowledge of receivers
(subscribers).

### Channels and Pattern Subscriptions

```bash
# Subscribe to specific channels
SUBSCRIBE news:sports news:tech         # Block and wait for messages

# Subscribe to pattern
PSUBSCRIBE news:*                       # Match all news channels

# Publish a message
PUBLISH news:sports "Goal scored!"      # Returns number of receivers
# → (integer) 2

# Unsubscribe
UNSUBSCRIBE news:sports
PUNSUBSCRIBE news:*
```

```
Publisher                    Redis                    Subscribers
                                                     
  PUBLISH ──────────>  +----------+  ──────────>  Sub A (SUBSCRIBE news:sports)
  news:sports          | Channel  |
  "Goal scored!"       | Routing  |  ──────────>  Sub B (PSUBSCRIBE news:*)
                       +----------+
                                     ──────────>  Sub C (SUBSCRIBE news:sports)
```

**Sharded Pub/Sub (Redis 7.0+):** In cluster mode, standard Pub/Sub broadcasts
messages to all nodes. Sharded Pub/Sub routes channels to specific slots for
better scalability:

```bash
SSUBSCRIBE news:sports          # Sharded subscribe
SPUBLISH news:sports "Goal!"    # Sharded publish
```

### Pub/Sub Limitations

- **No persistence** — Messages are fire-and-forget; missed messages are lost
- **No acknowledgment** — No way to confirm a subscriber processed a message
- **No replay** — Cannot read message history
- **No consumer groups** — All subscribers receive all messages (fan-out only)
- **Blocking** — Subscribing clients cannot execute other commands

For durable messaging with acknowledgment and consumer groups, use
[Redis Streams](#redis-streams) instead.

### Pub/Sub Use Cases

- Real-time notifications (chat, alerts)
- Cache invalidation signals
- Event broadcasting across services
- Configuration change notifications
- WebSocket fan-out backend

---

## Redis Streams

Redis Streams provide a durable, append-only log data structure with consumer
group support. Streams combine the best of Pub/Sub (real-time delivery) with
durable storage and acknowledgment.

### Stream Commands

```bash
# Add entries to a stream
XADD orders * customer "alice" product "widget" qty 3
# → "1681554000000-0"  (ID = millisecondsTimestamp-sequenceNumber)

XADD orders * customer "bob" product "gadget" qty 1
# → "1681554000001-0"

# Custom IDs (must be greater than last ID)
XADD orders 1681554000002-0 customer "charlie" product "thing" qty 5

# Stream length
XLEN orders
# → 3

# Read entries
XRANGE orders - +                     # All entries (oldest to newest)
XRANGE orders - + COUNT 10            # First 10 entries
XREVRANGE orders + -                  # All entries (newest to oldest)

# Read with blocking (wait for new entries)
XREAD COUNT 5 BLOCK 5000 STREAMS orders $
# Blocks for up to 5 seconds waiting for new entries after the last one

# Read from specific ID
XREAD COUNT 10 STREAMS orders 1681554000000-0
# Returns entries after the given ID

# Delete entries
XDEL orders 1681554000000-0

# Trim stream
XTRIM orders MAXLEN 1000             # Keep latest 1000 entries
XTRIM orders MAXLEN ~ 1000           # Approximate trim (more efficient)
XTRIM orders MINID 1681554000000-0   # Remove entries older than ID
```

### Consumer Groups

Consumer groups enable distributed processing of a stream. Each message is
delivered to exactly one consumer in the group.

```bash
# Create consumer group
XGROUP CREATE orders orderprocessors $ MKSTREAM
#                stream  group-name    start-ID

# Create group starting from beginning
XGROUP CREATE orders analytics 0

# Read as consumer in a group
XREADGROUP GROUP orderprocessors worker-1 COUNT 5 BLOCK 2000 STREAMS orders >
# ">" means only new (undelivered) messages

# Read pending (unacknowledged) messages
XREADGROUP GROUP orderprocessors worker-1 COUNT 5 STREAMS orders 0
# "0" means read pending messages for this consumer

# Acknowledge processed messages
XACK orders orderprocessors 1681554000000-0 1681554000001-0

# Consumer group info
XINFO GROUPS orders                  # List all groups for stream
XINFO CONSUMERS orders orderprocessors  # List consumers in group
XINFO STREAM orders                  # Stream metadata
XINFO STREAM orders FULL             # Detailed stream info
```

```
Stream Processing with Consumer Groups:

  Producer           Stream                Consumer Group
                                          "orderprocessors"
  XADD ──────>  +-----------+
                | Entry 1   |──────>  worker-1 (XREADGROUP)
                | Entry 2   |              │
                | Entry 3   |──────>  worker-2 (XREADGROUP)
                | Entry 4   |              │
                | Entry 5   |──────>  worker-3 (XREADGROUP)
                | ...       |
                +-----------+
                     │
                     └──────>  Consumer Group "analytics"
                                  analyst-1 (reads ALL from 0)
```

### Pending Entries List

The Pending Entries List (PEL) tracks messages delivered to consumers but not
yet acknowledged. This enables reliable processing and dead-letter handling.

```bash
# View pending entries
XPENDING orders orderprocessors - + 10
# → Shows: message-ID, consumer, idle-time, delivery-count

# Claim abandoned messages (idle > 60 seconds)
XCLAIM orders orderprocessors worker-2 60000 1681554000000-0
# Transfer ownership to worker-2

# Auto-claim (Redis 6.2+) — claim idle messages automatically
XAUTOCLAIM orders orderprocessors worker-2 60000 0-0 COUNT 10
# Scans and claims up to 10 idle messages

# Delete consumer
XGROUP DELCONSUMER orders orderprocessors worker-1
```

### Stream Processing Patterns

**Fan-out (multiple groups reading the same stream):**

```bash
XGROUP CREATE events group-notifications 0
XGROUP CREATE events group-analytics 0
XGROUP CREATE events group-archiver 0
# Each group independently processes ALL events
```

**Dead letter queue:**

```bash
# Check for messages delivered 3+ times
XPENDING orders orderprocessors - + 100

# If delivery count > 3, move to dead letter stream
XCLAIM orders orderprocessors dlq-handler 0 <message-id>
# In application: read claimed message, XADD to dead-letter stream, XACK original
```

### Streams vs Kafka

```
+------------------------+-------------------+------------------------+
| Feature                | Redis Streams     | Apache Kafka           |
+------------------------+-------------------+------------------------+
| Deployment             | Simple, built-in  | Complex, JVM-based     |
| Throughput             | ~1M msgs/sec      | ~10M+ msgs/sec         |
| Persistence            | In-memory + disk  | Disk-primary           |
| Consumer Groups        | Yes               | Yes                    |
| Exactly-once           | No (at-least-once)| Yes (with transactions)|
| Message Ordering       | Per stream        | Per partition          |
| Retention              | Memory-bound      | Disk-bound (unlimited) |
| Replay                 | Yes               | Yes                    |
| Ideal Scale            | Small-medium      | Large-scale            |
| Latency                | Sub-millisecond   | Low milliseconds       |
| Built-in Processing    | No                | Kafka Streams / ksqlDB |
+------------------------+-------------------+------------------------+
```

---

## Lua Scripting and Functions

### EVAL and EVALSHA

Redis supports server-side scripting with Lua 5.1. Scripts execute atomically —
no other command runs while a script is executing.

```bash
# Basic EVAL
EVAL "return redis.call('GET', KEYS[1])" 1 mykey
# Args: script, numkeys, key1, ...keyN, arg1, ...argN

# Set a key only if current value matches
EVAL "
  local current = redis.call('GET', KEYS[1])
  if current == ARGV[1] then
    return redis.call('SET', KEYS[1], ARGV[2])
  else
    return nil
  end
" 1 mykey "expected_value" "new_value"

# EVALSHA — call by SHA1 hash (avoids retransmitting script)
# First, load the script
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# → "a42059b356c875f0717db19a51f6aaa9161571a2"

EVALSHA "a42059b356c875f0717db19a51f6aaa9161571a2" 1 mykey

# Script management
SCRIPT EXISTS <sha1> <sha2>     # Check if scripts are cached
SCRIPT FLUSH                    # Clear script cache
```

### Redis 7 Functions API

Redis 7.0 introduced Functions as a replacement for ad-hoc EVAL scripts.
Functions are stored in the server, replicated, and persisted.

```bash
# Register a function library
FUNCTION LOAD "#!lua name=mylib
  redis.register_function('myfunc', function(keys, args)
    return redis.call('GET', keys[1])
  end)

  redis.register_function('my_hgetall', function(keys, args)
    local result = redis.call('HGETALL', keys[1])
    return result
  end)
"

# Call a function
FCALL myfunc 1 mykey
FCALL my_hgetall 1 user:1

# Read-only variant (safe on replicas)
FCALL_RO myfunc 1 mykey

# Management
FUNCTION LIST                   # List all function libraries
FUNCTION DUMP                   # Serialize all functions
FUNCTION RESTORE <payload>      # Restore serialized functions
FUNCTION DELETE mylib            # Delete a function library
FUNCTION FLUSH                  # Delete all function libraries
```

### Common Script Examples

**Rate limiter (sliding window):**

```bash
EVAL "
  local key = KEYS[1]
  local limit = tonumber(ARGV[1])
  local window = tonumber(ARGV[2])
  local now = tonumber(ARGV[3])
  
  -- Remove entries outside the window
  redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
  
  -- Count current entries
  local count = redis.call('ZCARD', key)
  
  if count < limit then
    -- Add new entry and set expiry
    redis.call('ZADD', key, now, now .. '-' .. math.random(1000000))
    redis.call('EXPIRE', key, window)
    return 1  -- Allowed
  else
    return 0  -- Rate limited
  end
" 1 ratelimit:user:123 100 60 1681554000
# Allow 100 requests per 60 seconds
```

**Atomic compare-and-swap:**

```bash
EVAL "
  local key = KEYS[1]
  local expected = ARGV[1]
  local desired = ARGV[2]
  local current = redis.call('GET', key)
  if current == expected then
    redis.call('SET', key, desired)
    return 1
  end
  return 0
" 1 mykey "old_value" "new_value"
```

---

## Redis Modules

Redis modules extend Redis with new data types, commands, and capabilities.
They are loaded as shared libraries at runtime.

### RedisJSON

Provides native JSON support with path-based access and manipulation.

```bash
# Set JSON document
JSON.SET user:1 $ '{"name":"Alice","age":30,"scores":[95,87,92]}'

# Get full document
JSON.GET user:1 $
# → [{"name":"Alice","age":30,"scores":[95,87,92]}]

# Get nested field
JSON.GET user:1 $.name
# → ["Alice"]

# Update field
JSON.SET user:1 $.age 31

# Array operations
JSON.ARRAPPEND user:1 $.scores 88
JSON.ARRLEN user:1 $.scores
# → [4]

# Numeric operations
JSON.NUMINCRBY user:1 $.age 1
# → [32]
```

### RediSearch

Full-text search and secondary indexing engine for Redis.

```bash
# Create an index on hash keys
FT.CREATE idx:users ON HASH PREFIX 1 user: SCHEMA
  name TEXT SORTABLE
  email TAG
  age NUMERIC SORTABLE
  bio TEXT WEIGHT 2.0

# Search
FT.SEARCH idx:users "@name:Alice"
FT.SEARCH idx:users "@age:[25 35]"
FT.SEARCH idx:users "@name:Ali*"            # Prefix search
FT.SEARCH idx:users "@email:{alice@*}"      # Tag filter

# Aggregate
FT.AGGREGATE idx:users "*"
  GROUPBY 1 @age
  REDUCE COUNT 0 AS count
  SORTBY 2 @count DESC
```

### RedisTimeSeries

Optimized for time-series data with downsampling, aggregation, and retention.

```bash
# Create a time series
TS.CREATE sensor:temp:1 RETENTION 86400000 LABELS type temp location office

# Add samples
TS.ADD sensor:temp:1 * 22.5     # Auto-timestamp
TS.ADD sensor:temp:1 1681554000 23.1

# Range query
TS.RANGE sensor:temp:1 - + AGGREGATION avg 3600000
# Average temperature per hour

# Create downsampling rule
TS.CREATERULE sensor:temp:1 sensor:temp:1:hourly AGGREGATION avg 3600000

# Multi-key query by labels
TS.MRANGE - + FILTER type=temp location=office
```

### RedisBloom

Probabilistic data structures: Bloom filters, Cuckoo filters, Count-Min Sketch,
and Top-K.

```bash
# Bloom Filter — check membership with no false negatives
BF.ADD filter:emails "alice@example.com"
BF.EXISTS filter:emails "alice@example.com"    # → 1
BF.EXISTS filter:emails "bob@example.com"      # → 0 (probably)

# Reserve with error rate and capacity
BF.RESERVE filter:urls 0.001 1000000           # 0.1% error, 1M capacity

# Cuckoo Filter — supports deletion
CF.ADD filter:sessions "sess:abc123"
CF.EXISTS filter:sessions "sess:abc123"        # → 1
CF.DEL filter:sessions "sess:abc123"

# Count-Min Sketch — frequency estimation
CMS.INITBYDIM sketches:urls 2000 7
CMS.INCRBY sketches:urls "/page1" 1 "/page2" 3
CMS.QUERY sketches:urls "/page1"               # → [1]

# Top-K — track most frequent items
TOPK.RESERVE trending 10
TOPK.ADD trending "redis" "kafka" "redis" "postgres"
TOPK.LIST trending
```

### RedisGraph (Deprecated)

> **Note:** RedisGraph was deprecated in early 2024. Consider alternatives like
> Neo4j, Amazon Neptune, or FalkorDB (community fork of RedisGraph).

RedisGraph provided a graph database capability using the Cypher query language
(openCypher subset) and sparse adjacency matrices (GraphBLAS).

---

## Transactions

### MULTI / EXEC / DISCARD

Redis transactions group commands for sequential, atomic execution. All commands
in a transaction are serialized and executed without interruption.

```bash
MULTI                       # Start transaction
SET account:alice 1000      # → QUEUED
SET account:bob 500         # → QUEUED
DECRBY account:alice 200    # → QUEUED
INCRBY account:bob 200      # → QUEUED
EXEC                        # Execute all commands atomically
# → [OK, OK, 800, 700]

# Cancel a transaction
MULTI
SET key1 "value1"           # → QUEUED
DISCARD                     # Cancel — no commands executed
```

**Important:** Redis transactions do NOT support rollback. If a command fails
during EXEC, other commands still execute. This differs from traditional ACID
database transactions.

```bash
MULTI
SET key1 "hello"            # → QUEUED
INCR key1                   # → QUEUED (will fail — not a number)
SET key2 "world"            # → QUEUED
EXEC
# → [OK, (error) ERR value is not an integer, OK]
# key1 = "hello", key2 = "world" — partial execution occurred
```

### Optimistic Locking with WATCH

WATCH provides a check-and-set (CAS) mechanism. If any watched key changes
before EXEC, the transaction is aborted.

```bash
WATCH account:alice         # Watch for changes
val = GET account:alice     # Read current value (in client code)

MULTI
SET account:alice (val - 100)
EXEC
# Returns nil if account:alice was modified by another client
# Returns [OK] if no concurrent modification occurred
```

```
Optimistic Locking Flow:

  Client A                    Redis                    Client B
     |                          |                          |
     |--- WATCH balance ------->|                          |
     |--- GET balance --------->|                          |
     |<-- "1000" --------------|                          |
     |                          |                          |
     |                          |<-- SET balance 500 -----|
     |                          |--- OK ----------------->|
     |                          |                          |
     |--- MULTI --------------->|                          |
     |--- SET balance 900 ----->|  (QUEUED)                |
     |--- EXEC ---------------->|                          |
     |<-- (nil) / aborted ------|  ← balance was modified! |
     |                          |                          |
     |  (Retry the operation)   |                          |
```

### Pipeline vs Transaction

```
+------------------+------------------+-------------------+
| Feature          | Pipeline         | Transaction       |
+------------------+------------------+-------------------+
| Purpose          | Reduce network   | Atomic execution  |
|                  | round trips      |                   |
| Atomicity        | No               | Yes               |
| Commands         | Sent in batch,   | Queued, executed  |
|                  | executed as      | atomically with   |
|                  | received         | EXEC              |
| Interleaving     | Other clients    | No other commands |
|                  | can interleave   | run between them  |
| Error handling   | Per-command      | All or abort      |
|                  |                  | (with WATCH)      |
| Performance      | Fastest          | Fast              |
+------------------+------------------+-------------------+
```

---

## Memory Management

### maxmemory Configuration

```
# redis.conf
maxmemory 4gb               # Hard memory limit
# OR
maxmemory 75%               # Percentage of system memory (Redis 7.0+)
```

```bash
# Runtime configuration
CONFIG SET maxmemory 4294967296       # 4 GB in bytes
CONFIG SET maxmemory-policy allkeys-lru
```

### Eviction Policies

When maxmemory is reached and a new write arrives, Redis applies the configured
eviction policy.

```
+-----------------------+------------------------------------------------------+
| Policy                | Description                                          |
+-----------------------+------------------------------------------------------+
| noeviction            | Return error on writes (default). Reads still work.  |
| allkeys-lru           | Evict least recently used key from ALL keys.         |
| allkeys-lfu           | Evict least frequently used key from ALL keys.       |
| allkeys-random        | Evict a random key from ALL keys.                    |
| volatile-lru          | Evict LRU key from keys WITH an expiration set.      |
| volatile-lfu          | Evict LFU key from keys WITH an expiration set.      |
| volatile-random       | Evict random key from keys WITH an expiration set.   |
| volatile-ttl          | Evict key with shortest TTL from expiring keys.      |
+-----------------------+------------------------------------------------------+

Recommendation:
  - Cache use case           → allkeys-lru or allkeys-lfu
  - Mixed cache + persistent → volatile-lru or volatile-lfu
  - Queue / stream use case  → noeviction (handle errors in app)
```

**LRU vs LFU:**

```
LRU (Least Recently Used):
  Evicts keys that haven't been accessed for the longest time.
  Good for: recency-based access patterns.

  Access pattern:  A  B  C  A  D  B  E  → evict C (least recent)

LFU (Least Frequently Used):
  Evicts keys that have been accessed the fewest times.
  Good for: frequency-based access patterns (hot vs cold data).

  Access count: A=50  B=5  C=200  D=1 → evict D (least frequent)
```

Redis uses **approximated LRU/LFU** — it samples a configurable number of keys
and evicts the best candidate from the sample:

```
maxmemory-samples 5          # Default: check 5 random keys
maxmemory-samples 10         # More accurate but slightly slower
```

### Memory Optimization

Redis uses specialized compact encodings for small data:

```
+--------------+------------------------+---------------------------+
| Type         | Compact Encoding       | Threshold (default)       |
+--------------+------------------------+---------------------------+
| Strings      | int (for integers)     | Automatic                 |
|              | embstr (≤ 44 bytes)    | Automatic                 |
| Lists        | listpack               | ≤ 128 entries, ≤ 64 bytes |
| Sets         | listpack               | ≤ 128 entries, ≤ 64 bytes |
|              | intset (integers only)  | ≤ 512 entries             |
| Sorted Sets  | listpack               | ≤ 128 entries, ≤ 64 bytes |
| Hashes       | listpack               | ≤ 128 entries, ≤ 64 bytes |
+--------------+------------------------+---------------------------+

# Tuning thresholds
list-max-listpack-size 128
set-max-listpack-entries 128
zset-max-listpack-entries 128
hash-max-listpack-entries 128
hash-max-listpack-value 64
```

**Tips for reducing memory usage:**

- Use hashes to group related keys (100 fields in 1 hash < 100 separate keys)
- Use short key names in high-cardinality scenarios
- Set TTLs on all cache entries
- Use integer IDs instead of UUIDs when possible
- Compress large values client-side (gzip, snappy)

### INFO MEMORY

```bash
INFO MEMORY
# used_memory:1234567            ← Bytes allocated by Redis allocator
# used_memory_human:1.18M
# used_memory_rss:2345678        ← Bytes as reported by OS (resident set)
# used_memory_peak:3456789       ← Peak memory usage
# mem_fragmentation_ratio:1.90   ← RSS / used_memory (>1.5 = fragmentation)
# mem_allocator:jemalloc-5.2.1

# Per-database key count
INFO KEYSPACE
# db0:keys=1234,expires=567,avg_ttl=3600000

# Memory usage of a specific key
MEMORY USAGE mykey
# → (integer) 72                  ← Bytes used by key + value

# Memory doctor (diagnostics)
MEMORY DOCTOR
```

---

## Security

### ACLs — Access Control Lists

Redis 6.0 introduced fine-grained ACLs to control what commands and keys each
user can access.

```bash
# Default user (backward compatible)
ACL SETUSER default on >password ~* &* +@all

# Create a read-only user
ACL SETUSER readonly on >readpass ~* &* +@read -@write -@admin

# Create a user restricted to specific key patterns
ACL SETUSER appuser on >apppass ~app:* &* +@all -@admin -@dangerous

# Create a user for specific commands only
ACL SETUSER cacheuser on >cachepass ~cache:* &* +GET +SET +DEL +EXPIRE

# List all users
ACL LIST

# Get current user info
ACL WHOAMI

# Check what commands a user can run
ACL GETUSER appuser

# Test permissions without executing
ACL DRYRUN appuser GET app:config
# → OK
ACL DRYRUN appuser FLUSHALL
# → (error) ...not allowed...

# Load ACLs from file
ACL LOAD
# Save ACLs to file
ACL SAVE

# ACL log (failed auth / permission denials)
ACL LOG 10
ACL LOG RESET
```

**ACL file (aclfile.conf):**

```
user default on >password ~* &* +@all
user readonly on >readpass ~* &* +@read
user appuser on >apppass ~app:* &* +@all -@admin -@dangerous
```

### TLS Encryption

Redis 6.0+ supports TLS for encrypted client-server and server-server
communication.

```
# redis.conf
tls-port 6380
port 0                           # Disable non-TLS port

tls-cert-file /path/to/redis.crt
tls-key-file /path/to/redis.key
tls-ca-cert-file /path/to/ca.crt

tls-auth-clients optional       # Require client certificates: yes/no/optional
tls-replication yes             # Encrypt replication traffic
tls-cluster yes                 # Encrypt cluster bus traffic

tls-protocols "TLSv1.2 TLSv1.3"
tls-ciphersuites "TLS_AES_256_GCM_SHA384"
```

```bash
# Connect with TLS using redis-cli
redis-cli --tls \
  --cert /path/to/client.crt \
  --key /path/to/client.key \
  --cacert /path/to/ca.crt \
  -p 6380
```

### AUTH and Protected Mode

```bash
# Simple password (legacy, pre-6.0)
# redis.conf:
requirepass your-strong-password

# Client authentication
AUTH your-strong-password
AUTH username password              # ACL-based auth (Redis 6.0+)
```

**Protected mode** (enabled by default) blocks external connections when no
password is set and Redis is not bound to a specific interface:

```
protected-mode yes                  # Default: yes
bind 127.0.0.1 -::1                # Bind to localhost only
```

### Network Security

```
# Bind to specific interfaces
bind 127.0.0.1 192.168.1.10

# Change default port
port 6380

# Disable commands that are dangerous in production
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG ""
rename-command DEBUG ""

# Timeout idle clients
timeout 300                         # Close idle connections after 5 min

# Limit simultaneous connections
maxclients 10000
```

### RENAME-COMMAND

Rename or disable dangerous commands to prevent accidental or malicious use:

```
# redis.conf
rename-command FLUSHALL ""              # Disable entirely
rename-command FLUSHDB ""
rename-command CONFIG "CONFIG_a1b2c3"   # Rename to obscure name
rename-command DEBUG ""
rename-command SHUTDOWN "SHUTDOWN_x9y8z7"
rename-command KEYS "KEYS_internal"     # Prevent accidental KEYS in prod
```

> **Note:** In Redis 7.0+, ACLs are preferred over RENAME-COMMAND for access
> control since ACLs provide per-user granularity.

---

## Common Patterns

### Rate Limiting — Sliding Window

```bash
# Sliding window using sorted set
# Key: ratelimit:{identifier}
# Score: timestamp
# Member: unique request ID

# Check and record request (Lua script for atomicity)
EVAL "
  local key = KEYS[1]
  local limit = tonumber(ARGV[1])
  local window = tonumber(ARGV[2])
  local now = tonumber(ARGV[3])
  local request_id = ARGV[4]
  
  -- Remove entries outside the window
  redis.call('ZREMRANGEBYSCORE', key, '-inf', now - window)
  
  -- Count requests in current window
  local count = redis.call('ZCARD', key)
  
  if count < limit then
    redis.call('ZADD', key, now, request_id)
    redis.call('EXPIRE', key, window)
    return {1, limit - count - 1}   -- {allowed, remaining}
  else
    return {0, 0}                    -- {denied, remaining}
  end
" 1 ratelimit:api:user:123 100 60 1681554000 "req-uuid-abc"
```

```
Sliding Window Visualization:

  Time ──────────────────────────────────────────────>

  |----- 60 second window -----|
  |  req req  req   req    req |  req  (new request)
  |  1   2    3     4      5   |  6
  
  Window slides with each request.
  Count requests within [now - 60s, now].
  If count >= limit → reject.
```

### Distributed Locks — Redlock Algorithm

Single-instance lock:

```bash
# Acquire lock
SET lock:resource-1 "owner-uuid" NX EX 30
# NX = only if not exists; EX 30 = auto-expire in 30 seconds

# Release lock (only if you own it — use Lua for atomicity)
EVAL "
  if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
  else
    return 0
  end
" 1 lock:resource-1 "owner-uuid"
```

**Redlock (distributed lock across N independent Redis instances):**

```
Redlock Algorithm (N = 5 instances):

  1. Get current time T1
  2. Try to acquire lock on ALL N instances (with short timeout)
  3. Calculate elapsed time T2 - T1
  4. Lock is acquired if:
     - Majority (N/2 + 1 = 3) instances granted the lock
     - Total elapsed time < lock TTL
  5. Effective TTL = initial TTL - elapsed time
  6. If lock fails, release on ALL instances

  +----------+  +----------+  +----------+  +----------+  +----------+
  | Redis 1  |  | Redis 2  |  | Redis 3  |  | Redis 4  |  | Redis 5  |
  |  LOCKED  |  |  LOCKED  |  |  LOCKED  |  |  FAILED  |  |  LOCKED  |
  +----------+  +----------+  +----------+  +----------+  +----------+
       ✓             ✓             ✓             ✗             ✓
                            4/5 = majority → LOCK ACQUIRED
```

### Session Storage

```bash
# Store session as hash
HSET session:abc123 user_id 1001 role "admin" created_at 1681554000
EXPIRE session:abc123 3600              # 1 hour TTL

# Read session
HGETALL session:abc123
# → {"user_id":"1001","role":"admin","created_at":"1681554000"}

# Refresh TTL on activity
EXPIRE session:abc123 3600

# Delete session (logout)
DEL session:abc123

# Alternative: store as JSON string
SET session:abc123 '{"user_id":1001,"role":"admin"}' EX 3600
```

### Leaderboards

```bash
# Add/update scores
ZADD leaderboard 1500 "player:alice"
ZADD leaderboard 2300 "player:bob"
ZADD leaderboard 1800 "player:charlie"
ZINCRBY leaderboard 100 "player:alice"

# Top 10 players
ZREVRANGE leaderboard 0 9 WITHSCORES

# Player rank (0-indexed from top)
ZREVRANK leaderboard "player:alice"
# → 2

# Players around a specific player (e.g., rank ±2)
# Get player rank first, then range
ZREVRANGE leaderboard 0 4 WITHSCORES    # Ranks 1-5

# Count players in score range
ZCOUNT leaderboard 1000 2000

# Periodic leaderboard reset
DEL leaderboard:weekly
ZUNIONSTORE leaderboard:weekly 1 leaderboard
```

### Caching Layer

```
Cache-Aside (Lazy Loading) Pattern:

  Application          Redis Cache          Database
       |                    |                    |
       |--- GET key ------->|                    |
       |                    |                    |
       |    [CACHE HIT]     |                    |
       |<-- value ----------|                    |
       |                    |                    |
       |    [CACHE MISS]    |                    |
       |<-- nil ------------|                    |
       |                                         |
       |--- SELECT ... ------- query ----------->|
       |<-- result --------  -------------------|
       |                                         |
       |--- SET key value -->|                    |
       |    EX ttl          |                    |
       |                    |                    |

  Write-Through Pattern:

  Application          Redis Cache          Database
       |                    |                    |
       |--- UPDATE ---------|--- write --------->|
       |                    |                    |
       |--- SET key value ->|                    |
       |                    |                    |
```

```bash
# Cache with TTL
SET cache:user:1 '{"name":"Alice","email":"alice@example.com"}' EX 300

# Cache with stale-while-revalidate pattern
SET cache:product:42 '{"name":"Widget","price":9.99}' EX 300
SET cache:product:42:stale_at 1681554300     # Soft expiry time
# App checks stale_at; if expired, serve stale + async refresh
```

### Job Queues

```bash
# Producer: push jobs to queue
LPUSH queue:emails '{"to":"alice@example.com","subject":"Welcome"}'
LPUSH queue:emails '{"to":"bob@example.com","subject":"Alert"}'

# Consumer: blocking pop (waits for jobs)
BRPOP queue:emails 30
# Blocks up to 30 seconds; returns job or nil

# Reliable queue with LMOVE (processing list)
LMOVE queue:emails queue:emails:processing RIGHT LEFT
# Process the job...
LREM queue:emails:processing 1 '<job-data>'   # Remove after success

# Priority queues (check high-priority first)
BRPOP queue:critical queue:high queue:normal queue:low 30
# Returns from first non-empty queue

# Delayed jobs using sorted sets
ZADD queue:delayed 1681557600 '{"job":"send_report"}'
# Worker polls: ZRANGEBYSCORE queue:delayed -inf <current_time> LIMIT 0 1
```

---

## Performance Tuning

### Pipelining

Pipelining sends multiple commands without waiting for individual replies,
dramatically reducing network round trips.

```
Without Pipeline:             With Pipeline:

  Client       Server          Client       Server
    |              |              |              |
    |-- CMD 1 ---->|              |-- CMD 1 ---->|
    |<-- Reply 1 --|              |-- CMD 2 ---->|
    |-- CMD 2 ---->|              |-- CMD 3 ---->|
    |<-- Reply 2 --|              |-- CMD 4 ---->|
    |-- CMD 3 ---->|              |              |
    |<-- Reply 3 --|              |<-- Reply 1 --|
    |-- CMD 4 ---->|              |<-- Reply 2 --|
    |<-- Reply 4 --|              |<-- Reply 3 --|
    |              |              |<-- Reply 4 --|
                                  
  4 round trips                1 round trip
  ~4ms latency × 4             ~4ms total
  = ~16ms total
```

```bash
# redis-cli pipeline example
printf "SET key1 val1\r\nSET key2 val2\r\nGET key1\r\nGET key2\r\n" \
  | redis-cli --pipe

# Most client libraries support pipelining natively
# Example throughput: 100K ops/sec → 500K+ ops/sec with pipelining
```

### Connection Pooling

Establishing a TCP connection for every command is expensive. Use connection
pools to reuse connections.

```
+-------------------+        +---------------------------+
|   Application     |        |       Redis Server        |
|                   |        |                           |
|  Thread 1 -----+  |        |                           |
|  Thread 2 ---+ |  |        |                           |
|  Thread 3 -+ | |  |  +--->|--- Connection 1           |
|            | | |  |  |    |                           |
|          +-v-v-v--+  |    |                           |
|          | Conn   |--+--->|--- Connection 2           |
|          | Pool   |  |    |                           |
|          | (N=5)  |--+--->|--- Connection 3           |
|          |        |  |    |                           |
|          +--------+--+--->|--- Connection 4           |
|                   |  |    |                           |
|  Thread 4 --------+  +--->|--- Connection 5           |
|  Thread 5 --------+       |                           |
+-------------------+        +---------------------------+

Typical pool sizes: 5-20 connections per application instance
```

### Key Design Best Practices

```
+----------------------------------+------------------------------------+
| Practice                         | Example                            |
+----------------------------------+------------------------------------+
| Use colons as delimiters         | user:1001:profile                  |
| Include object type              | session:abc123                     |
| Include object ID                | order:789:items                    |
| Keep keys short (high volume)    | u:1001:p instead of                |
|                                  | user:1001:profile (only if needed) |
| Use hash tags for cluster        | {user:1001}:profile                |
|                                  | {user:1001}:settings               |
| Avoid special characters         | No spaces, newlines, or nulls      |
| Set TTLs on cache keys           | Always expire what can be rebuilt  |
+----------------------------------+------------------------------------+
```

### SCAN vs KEYS

**Never use KEYS in production.** It blocks the server while scanning the
entire keyspace.

```bash
# BAD — blocks server, O(N) over entire keyspace
KEYS user:*

# GOOD — incremental iteration, non-blocking
SCAN 0 MATCH user:* COUNT 100
# → cursor: 1234, results: [user:1, user:2, ...]
SCAN 1234 MATCH user:* COUNT 100
# → cursor: 5678, results: [user:3, user:4, ...]
# Continue until cursor returns 0

# Scan specific data types
SSCAN myset 0 MATCH pattern* COUNT 100    # Sets
HSCAN myhash 0 MATCH field* COUNT 100     # Hashes
ZSCAN myzset 0 MATCH member* COUNT 100    # Sorted Sets
```

### Avoiding Large Values

Large keys and values cause latency spikes, slow persistence, and memory
fragmentation.

```
+-------------------------+-----------------------------------------------+
| Problem                 | Solution                                      |
+-------------------------+-----------------------------------------------+
| Large hash (>1000       | Split into multiple hashes with bucketing:    |
| fields)                 | hash:{id / 100} → field = id % 100            |
+-------------------------+-----------------------------------------------+
| Large list (>10K items) | Use LTRIM to cap size; archive old entries     |
+-------------------------+-----------------------------------------------+
| Large string (>100 KB)  | Compress client-side; split into chunks        |
+-------------------------+-----------------------------------------------+
| Large set (>10K         | Partition into multiple sets by category        |
| members)                |                                               |
+-------------------------+-----------------------------------------------+
| Fat keys (many per      | Use UNLINK instead of DEL (non-blocking        |
| delete)                 | deletion in background thread)                 |
+-------------------------+-----------------------------------------------+
```

```bash
# Check for large keys
redis-cli --bigkeys                      # Scan and report largest keys

# Non-blocking delete
UNLINK large-key                         # O(1) — freed in background thread
# vs
DEL large-key                            # O(N) — blocks main thread

# Monitor key sizes
MEMORY USAGE mykey                       # Bytes used by a specific key
DEBUG OBJECT mykey                       # Internal encoding details
```

### Slow Log

The slow log records commands that exceed a configurable execution time threshold.

```bash
# Configuration
CONFIG SET slowlog-log-slower-than 10000   # Log commands > 10ms (microseconds)
CONFIG SET slowlog-max-len 128             # Keep last 128 entries

# View slow log
SLOWLOG GET 10                             # Last 10 slow commands
# → ID, timestamp, execution time (μs), command, client info

SLOWLOG LEN                                # Number of entries
SLOWLOG RESET                              # Clear slow log
```

```
Example slow log entry:

  1) (integer) 42                         ← Entry ID
  2) (integer) 1681554000                 ← Unix timestamp
  3) (integer) 35217                      ← Execution time: 35.2ms
  4) 1) "KEYS"                           ← Command
     2) "user:*"
  5) "192.168.1.10:54321"                ← Client address
  6) "app-server"                         ← Client name
```

---

## Client Libraries

```
+---------------------+----------+-----------+------------------------------+
| Library             | Language | Protocol  | Notes                        |
+---------------------+----------+-----------+------------------------------+
| redis-py            | Python   | RESP2/3   | Official; async support via  |
|                     |          |           | redis.asyncio                |
+---------------------+----------+-----------+------------------------------+
| ioredis             | Node.js  | RESP2/3   | Full-featured; Cluster and   |
|                     |          |           | Sentinel support; Lua        |
+---------------------+----------+-----------+------------------------------+
| Jedis               | Java     | RESP2     | Synchronous; simple API;     |
|                     |          |           | connection pooling built-in  |
+---------------------+----------+-----------+------------------------------+
| Lettuce             | Java     | RESP2/3   | Async/reactive; Netty-based; |
|                     |          |           | Cluster & Sentinel support   |
+---------------------+----------+-----------+------------------------------+
| StackExchange.Redis | C# / .NET| RESP2/3  | High-perf; multiplexed       |
|                     |          |           | connections; used by Azure   |
+---------------------+----------+-----------+------------------------------+
| go-redis            | Go       | RESP2/3   | Full-featured; type-safe;    |
|                     |          |           | Cluster, Sentinel, Streams   |
+---------------------+----------+-----------+------------------------------+
| hiredis             | C        | RESP2     | Official low-level client;   |
|                     |          |           | often embedded in other libs |
+---------------------+----------+-----------+------------------------------+
| Predis              | PHP      | RESP2/3   | Pure PHP; flexible; Cluster  |
|                     |          |           | and replication support      |
+---------------------+----------+-----------+------------------------------+
| redis-rb            | Ruby     | RESP2/3   | Official; Cluster support;   |
|                     |          |           | simple interface             |
+---------------------+----------+-----------+------------------------------+
| redis-rs            | Rust     | RESP2/3   | Async via Tokio; type-safe;  |
|                     |          |           | Cluster and Sentinel support |
+---------------------+----------+-----------+------------------------------+
```

When choosing a client library, consider:

- **Protocol support** — RESP3 enables client-side caching and richer types
- **Cluster support** — Automatic slot routing and redirect handling
- **Connection pooling** — Built-in vs manual pool management
- **Async support** — Critical for high-concurrency applications
- **Pipelining** — Batch command support for throughput optimization

---

## Version History

| Date       | Description     | Author |
|------------|-----------------|--------|
| 2025-04-15 | Initial version | Team   |
