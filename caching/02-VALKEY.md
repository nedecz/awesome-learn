# Valkey

## Table of Contents

- [Overview](#overview)
  - [What is Valkey](#what-is-valkey)
  - [Why Valkey Exists](#why-valkey-exists)
  - [The Redis License Change](#the-redis-license-change)
  - [Founding Contributors](#founding-contributors)
  - [Governance Model](#governance-model)
  - [BSD-3-Clause License](#bsd-3-clause-license)
- [Valkey vs Redis](#valkey-vs-redis)
  - [Licensing Comparison](#licensing-comparison)
  - [API and Protocol Compatibility](#api-and-protocol-compatibility)
  - [Command Compatibility](#command-compatibility)
  - [Feature Parity at Fork Point](#feature-parity-at-fork-point)
  - [Where They Diverge](#where-they-diverge)
  - [Side-by-Side Comparison Table](#side-by-side-comparison-table)
- [Architecture](#architecture)
  - [Core Architecture — Fork of Redis 7.2.4](#core-architecture--fork-of-redis-724)
  - [Single-Threaded Event Loop](#single-threaded-event-loop)
  - [I/O Threading Model](#io-threading-model)
  - [RESP Protocol — RESP2 and RESP3](#resp-protocol--resp2-and-resp3)
  - [Memory Management](#memory-management)
  - [Persistence — RDB and AOF](#persistence--rdb-and-aof)
- [New Features in Valkey](#new-features-in-valkey)
  - [Multi-Threaded I/O Enhancements](#multi-threaded-io-enhancements)
  - [RDMA Support](#rdma-support)
  - [Dual-Channel Replication](#dual-channel-replication)
  - [Per-Slot Dictionary](#per-slot-dictionary)
  - [Improved Memory Efficiency](#improved-memory-efficiency)
  - [Valkey 8.x Features](#valkey-8x-features)
- [Data Structures](#data-structures)
  - [Full Compatibility with Redis Data Types](#full-compatibility-with-redis-data-types)
  - [Strings](#strings)
  - [Lists](#lists)
  - [Sets](#sets)
  - [Sorted Sets](#sorted-sets)
  - [Hashes](#hashes)
  - [Streams](#streams)
  - [Bitmaps and HyperLogLog](#bitmaps-and-hyperloglog)
  - [Geospatial Indexes](#geospatial-indexes)
  - [Data Structures Summary](#data-structures-summary)
- [Cluster Mode](#cluster-mode)
  - [Hash Slot Architecture — 16384 Slots](#hash-slot-architecture--16384-slots)
  - [Cluster Topology](#cluster-topology)
  - [Replication Enhancements](#replication-enhancements)
  - [Improved Failover](#improved-failover)
  - [Cluster Configuration](#cluster-configuration)
- [Migration from Redis to Valkey](#migration-from-redis-to-valkey)
  - [Migration Assessment](#migration-assessment)
  - [Client Library Compatibility](#client-library-compatibility)
  - [Configuration Mapping](#configuration-mapping)
  - [Data Migration Strategies](#data-migration-strategies)
  - [Step-by-Step Migration Guide](#step-by-step-migration-guide)
  - [Rollback Plan](#rollback-plan)
  - [Testing Checklist](#testing-checklist)
- [Cloud Provider Support](#cloud-provider-support)
  - [AWS ElastiCache for Valkey](#aws-elasticache-for-valkey)
  - [Google Cloud Memorystore for Valkey](#google-cloud-memorystore-for-valkey)
  - [Azure Cache Compatibility](#azure-cache-compatibility)
  - [Other Providers and Self-Hosted](#other-providers-and-self-hosted)
  - [Cloud Provider Comparison](#cloud-provider-comparison)
- [Valkey Modules](#valkey-modules)
  - [Module API Compatibility](#module-api-compatibility)
  - [Community Modules](#community-modules)
  - [Loading Modules](#loading-modules)
- [Community and Governance](#community-and-governance)
  - [Linux Foundation Project Structure](#linux-foundation-project-structure)
  - [Technical Steering Committee](#technical-steering-committee)
  - [Contributing to Valkey](#contributing-to-valkey)
  - [Release Cadence](#release-cadence)
  - [Roadmap](#roadmap)
- [Performance](#performance)
  - [Benchmarks vs Redis](#benchmarks-vs-redis)
  - [Multi-Threaded I/O Gains](#multi-threaded-io-gains)
  - [Latency Characteristics](#latency-characteristics)
  - [Throughput Under Load](#throughput-under-load)
- [When to Choose Valkey](#when-to-choose-valkey)
  - [Decision Framework](#decision-framework)
  - [Open-Source Licensing Needs](#open-source-licensing-needs)
  - [Cloud Provider Strategy](#cloud-provider-strategy)
  - [Community-Driven Development](#community-driven-development)
  - [When Redis May Still Be Preferred](#when-redis-may-still-be-preferred)
- [Best Practices](#best-practices)
  - [Configuration Recommendations](#configuration-recommendations)
  - [Security Hardening](#security-hardening)
  - [Monitoring and Observability](#monitoring-and-observability)
  - [Operational Tips](#operational-tips)
  - [Upgrade Strategy](#upgrade-strategy)
- [Version History](#version-history)

---

## Overview

### What is Valkey

Valkey is a high-performance, open-source key-value data store that serves as a
drop-in replacement for Redis. It is a community-driven fork of Redis 7.2.4,
maintained under the Linux Foundation. Valkey supports the full range of Redis
data structures—strings, hashes, lists, sets, sorted sets, streams, bitmaps,
HyperLogLog, and geospatial indexes—and is fully compatible with the Redis
Serialization Protocol (RESP2 and RESP3).

```
+---------------------------------------------------------------+
|                          Valkey                                |
|                                                                |
|  Fork of Redis 7.2.4  ·  BSD-3-Clause  ·  Linux Foundation    |
|                                                                |
|  +------------------+  +------------------+  +--------------+  |
|  |  In-Memory Store |  |  Data Structures |  |  Pub/Sub     |  |
|  |  Key-Value       |  |  Strings, Lists  |  |  Streams     |  |
|  |  Sub-ms Latency  |  |  Sets, Hashes    |  |  Consumer    |  |
|  |                  |  |  Sorted Sets     |  |  Groups      |  |
|  +------------------+  +------------------+  +--------------+  |
|                                                                |
|  +------------------+  +------------------+  +--------------+  |
|  |  Clustering      |  |  Persistence     |  |  Replication |  |
|  |  16384 Slots     |  |  RDB + AOF       |  |  Dual-Channel|  |
|  |  Auto-Sharding   |  |  Hybrid Mode     |  |  RDMA Support|  |
|  +------------------+  +------------------+  +--------------+  |
+---------------------------------------------------------------+
```

Valkey maintains wire-level compatibility with Redis clients, meaning existing
applications using Redis client libraries (jedis, lettuce, ioredis, redis-py,
go-redis, and others) can switch to Valkey without code changes.

### Why Valkey Exists

In March 2024, Redis Ltd. changed the license of the Redis server from the
permissive BSD-3-Clause license to a dual-license model combining the Redis
Source Available License v2 (RSALv2) and the Server Side Public License (SSPL).
This change meant that cloud service providers could no longer offer Redis as a
managed service without entering into a commercial agreement with Redis Ltd.

The license change triggered a community response. A group of major contributors
and companies that relied on Redis forked the project at version 7.2.4—the last
BSD-licensed release—and created Valkey under the Linux Foundation to ensure the
continued availability of a truly open-source, high-performance key-value store.

```
Timeline of Events
==================

2009          Redis created by Salvatore Sanfilippo (antirez)
   |
2015          Redis Labs (later Redis Ltd.) formed
   |
2018          Redis modules begin using proprietary licenses
   |
2024-Mar-20   Redis Ltd. relicenses Redis server → RSALv2 + SSPL
   |
2024-Mar-28   Valkey project announced under Linux Foundation
   |
2024-Apr      Valkey 7.2.5 — first release (bug fixes on 7.2.4 base)
   |
2024-Sep      Valkey 8.0 — first major release with new features
   |
2025+         Continued independent development and innovation
```

### The Redis License Change

Understanding the license change is essential context for why Valkey exists:

**Before (BSD-3-Clause):**
- Anyone could use, modify, and distribute Redis freely
- Cloud providers could offer Redis-as-a-Service without restrictions
- Fully compliant with OSI open-source definition

**After (RSALv2 + SSPL):**
- Users can still run Redis for internal use
- Cloud providers CANNOT offer Redis as a managed service without a license
- Not recognized as "open source" by the OSI
- Restricts competitive commercial offerings

```
+----------------------------+----------------------------+
|     BSD-3-Clause           |     RSALv2 + SSPL          |
|     (Redis ≤ 7.2.4)       |     (Redis ≥ 7.4)          |
+----------------------------+----------------------------+
| ✓ Use for any purpose      | ✓ Use for any purpose      |
| ✓ Modify freely            | ✓ Modify freely            |
| ✓ Distribute freely        | ✗ Offer as managed service |
| ✓ Offer as managed service | ✗ Competitive SaaS use     |
| ✓ OSI-approved license     | ✗ NOT OSI-approved         |
| ✓ No usage restrictions    | ✗ Usage restrictions apply |
+----------------------------+----------------------------+
```

### Founding Contributors

Valkey was founded with backing from major technology companies:

| Organization | Role                                          |
|-------------|-----------------------------------------------|
| AWS         | Primary contributor; ElastiCache Valkey support|
| Google      | Contributor; Memorystore Valkey integration    |
| Oracle      | Contributor; OCI Cache support                 |
| Ericsson    | Contributor; telecom use cases                 |
| Snap Inc.   | Contributor; social platform scale expertise   |

Additional contributors joined from companies including Alibaba, Huawei, Aiven,
Percona, and many independent open-source developers. The breadth of the founding
coalition ensured diverse perspectives and reduced single-vendor risk.

### Governance Model

Valkey follows the Linux Foundation's open governance principles:

```
+---------------------------------------------------------------+
|                    Linux Foundation                             |
|                         |                                      |
|            +------------+------------+                         |
|            |                         |                         |
|   Technical Steering           Project                         |
|   Committee (TSC)              Governance                      |
|            |                    Board                           |
|   +--------+--------+              |                           |
|   |        |        |         Budget, Legal,                   |
|   |        |        |         Trademarks                       |
|   v        v        v                                          |
|  Core   Modules  Release                                       |
|  Team    Team    Management                                    |
|   |        |        |                                          |
|   v        v        v                                          |
|  Feature  API     Versioning                                   |
|  Dev      Compat  & Packaging                                  |
+---------------------------------------------------------------+
```

Key governance principles:
- **Open decision-making** — All technical decisions happen in public forums
- **Merit-based committers** — Contributor access based on sustained quality contributions
- **Vendor-neutral** — No single company controls the project direction
- **Transparent roadmap** — Public roadmap with community input

### BSD-3-Clause License

Valkey uses the BSD-3-Clause license, one of the most permissive open-source
licenses available:

```
BSD-3-Clause License Summary
=============================

1. Redistributions of source code must retain the copyright notice
2. Redistributions in binary form must reproduce the copyright notice
3. Neither the name of the project nor contributor names may be used
   to endorse or promote derived products without permission

That's it. No copyleft. No service restrictions. No patent traps.
```

This license ensures:
- Cloud providers can offer Valkey-as-a-Service
- Companies can embed Valkey in proprietary products
- Developers can fork and modify without restriction
- Full compliance with OSI open-source definition

---

## Valkey vs Redis

### Licensing Comparison

| Aspect                  | Valkey (BSD-3-Clause)       | Redis (RSALv2 + SSPL)        |
|------------------------|-----------------------------|-------------------------------|
| OSI-approved           | Yes                         | No                            |
| Use in proprietary apps| Unrestricted                | Unrestricted                  |
| Offer as managed SaaS  | Allowed                     | Prohibited without license    |
| Competitive use        | No restrictions             | Restricted                    |
| Modification freedom   | Full                        | Full (but distribution limits)|
| Patent grant           | Implicit (BSD)              | Explicit in RSALv2            |
| Copyleft               | None                        | SSPL is strong copyleft-like  |

### API and Protocol Compatibility

Valkey is a drop-in replacement for Redis 7.2.4. Compatibility is maintained at
multiple levels:

**Wire Protocol:** Valkey speaks RESP2 and RESP3 identically to Redis. Any client
that connects to Redis can connect to Valkey without changes.

**Command Set:** All Redis 7.2.4 commands are supported with identical semantics.
Valkey has added new commands but has not removed or altered existing ones.

**Configuration:** The `valkey.conf` file follows the same format as `redis.conf`.
Most directives are identical; renamed directives have aliases for backward
compatibility.

**Data Format:** RDB and AOF files from Redis 7.2.4 are compatible with Valkey.
You can take an RDB snapshot from Redis and load it directly into Valkey.

```
+----------------+                    +----------------+
|  Application   |                    |  Application   |
|  using Redis   |                    |  using Valkey  |
|  client lib    |                    |  (same lib!)   |
+-------+--------+                    +-------+--------+
        |                                     |
        |  RESP2/RESP3                        |  RESP2/RESP3
        |  (identical)                        |  (identical)
        v                                     v
+----------------+    RDB/AOF files   +----------------+
|  Redis 7.2.4   | ================>  |  Valkey 7.2.5+ |
|  Server        |    compatible      |  Server        |
+----------------+                    +----------------+
```

### Command Compatibility

Core command groups and their compatibility status:

```
+----------------------+------------------+----------------------------------+
| Command Group        | Status           | Notes                            |
+----------------------+------------------+----------------------------------+
| String commands      | Fully compatible | GET, SET, MGET, INCR, etc.       |
| (GET, SET, etc.)     |                  |                                  |
+----------------------+------------------+----------------------------------+
| List commands        | Fully compatible | LPUSH, RPOP, LRANGE, etc.        |
| (LPUSH, RPOP, etc.)  |                  |                                  |
+----------------------+------------------+----------------------------------+
| Set commands         | Fully compatible | SADD, SMEMBERS, SINTER, etc.     |
| (SADD, SINTER, etc.) |                  |                                  |
+----------------------+------------------+----------------------------------+
| Sorted Set commands  | Fully compatible | ZADD, ZRANGE, ZRANGEBYSCORE      |
| (ZADD, ZRANGE, etc.) |                  |                                  |
+----------------------+------------------+----------------------------------+
| Hash commands        | Fully compatible | HSET, HGET, HMGET, HGETALL      |
| (HSET, HGET, etc.)   |                  |                                  |
+----------------------+------------------+----------------------------------+
| Stream commands      | Fully compatible | XADD, XREAD, XREADGROUP, etc.   |
| (XADD, XREAD, etc.)  |                  |                                  |
+----------------------+------------------+----------------------------------+
| Pub/Sub commands     | Fully compatible | PUBLISH, SUBSCRIBE, PSUBSCRIBE   |
+----------------------+------------------+----------------------------------+
| Scripting commands   | Fully compatible | EVAL, EVALSHA, FUNCTION LOAD     |
+----------------------+------------------+----------------------------------+
| Cluster commands     | Fully compatible | CLUSTER INFO, CLUSTER NODES      |
+----------------------+------------------+----------------------------------+
| Transaction commands | Fully compatible | MULTI, EXEC, WATCH, DISCARD      |
+----------------------+------------------+----------------------------------+
| ACL commands         | Fully compatible | ACL SETUSER, ACL LIST, etc.      |
+----------------------+------------------+----------------------------------+
| Server commands      | Mostly compatible| INFO, CONFIG — some new fields   |
+----------------------+------------------+----------------------------------+
| Module commands      | Fully compatible | MODULE LOAD, MODULE LIST         |
+----------------------+------------------+----------------------------------+
```

### Feature Parity at Fork Point

At the fork point (Redis 7.2.4), Valkey inherited the complete Redis feature set:

- All data structures (Strings, Lists, Sets, Sorted Sets, Hashes, Streams, etc.)
- RDB and AOF persistence with hybrid mode
- Redis Cluster with 16384 hash slots
- Redis Sentinel for high availability
- Pub/Sub and Streams with consumer groups
- Lua scripting and Redis 7.x Functions API
- ACL-based security model
- TLS/SSL encryption
- Client-side caching support (tracking)
- I/O threading (introduced in Redis 6.0)
- Module API

### Where They Diverge

Since the fork, Valkey and Redis have begun to diverge:

```
                    Redis 7.2.4 (Fork Point)
                           |
              +------------+------------+
              |                         |
              v                         v
        Valkey 7.2.5+              Redis 7.4+
              |                         |
    +---------+---------+     +---------+---------+
    |                   |     |                   |
    v                   v     v                   v
  Multi-threaded    Dual-     Redis 8.0        Proprietary
  I/O improve-     channel   features           module
  ments            replica-  (under new         licensing
                   tion      license)           continues
    |                   |
    v                   v
  RDMA support      Per-slot
  for ultra-low     dictionary
  latency           for memory
                    efficiency
```

**Valkey innovations (not in Redis):**
- Enhanced multi-threaded I/O architecture
- RDMA (Remote Direct Memory Access) support
- Dual-channel replication
- Per-slot dictionary for improved cluster memory efficiency
- Various performance optimizations

**Redis innovations (not in Valkey):**
- Features introduced in Redis 7.4+ and 8.0+
- Redis-specific proprietary modules and extensions
- Redis Insight integrated tooling

### Side-by-Side Comparison Table

```
+------------------------+-----------------------------+-----------------------------+
| Feature                | Valkey                      | Redis (post-license change) |
+------------------------+-----------------------------+-----------------------------+
| License                | BSD-3-Clause                | RSALv2 + SSPL              |
| Governance             | Linux Foundation / TSC      | Redis Ltd.                  |
| Base version           | Fork of Redis 7.2.4        | Continuous from 7.4+        |
| RESP protocol          | RESP2 + RESP3               | RESP2 + RESP3               |
| Data structures        | All Redis types             | All Redis types + new       |
| Cluster support        | Yes (16384 slots)           | Yes (16384 slots)           |
| Multi-threaded I/O     | Enhanced beyond Redis       | Original implementation     |
| RDMA support           | Yes (Valkey 8.0+)           | No                          |
| Dual-channel repl.     | Yes (Valkey 8.0+)           | No                          |
| Per-slot dictionary    | Yes (Valkey 8.0+)           | No                          |
| Module API             | Compatible                  | Original                    |
| Cloud managed service  | AWS, GCP, Oracle, etc.      | Redis Cloud                 |
| Community governance   | Open / vendor-neutral       | Single-vendor               |
| Release cadence        | Community-driven            | Company-driven              |
+------------------------+-----------------------------+-----------------------------+
```

---

## Architecture

### Core Architecture — Fork of Redis 7.2.4

Valkey's architecture is inherited from Redis 7.2.4 and shares the same
fundamental design principles: an in-memory data store with optional persistence,
built around an efficient event-driven architecture.

```
                    Valkey Server Architecture
+---------------------------------------------------------------+
|                                                                |
|  +------------------+     +-------------------+                |
|  |   Client Conn.   |     |   Client Conn.    |                |
|  |   (TCP/TLS/RDMA) |     |   (TCP/TLS/RDMA)  |                |
|  +--------+---------+     +---------+---------+                |
|           |                         |                          |
|           +------------+------------+                          |
|                        |                                       |
|                        v                                       |
|  +-----------------------------------------------------+      |
|  |              I/O Thread Pool (read/write)            |      |
|  |   Thread 1  |  Thread 2  |  Thread 3  |  Thread N   |      |
|  +-----------------------------------------------------+      |
|                        |                                       |
|                        v                                       |
|  +-----------------------------------------------------+      |
|  |           Main Event Loop (single-threaded)          |      |
|  |                                                      |      |
|  |  +----------+  +----------+  +-----------+           |      |
|  |  | Command  |  | Key      |  | Expiry    |           |      |
|  |  | Parsing  |  | Lookup   |  | Handling  |           |      |
|  |  +----------+  +----------+  +-----------+           |      |
|  |                                                      |      |
|  |  +----------+  +----------+  +-----------+           |      |
|  |  | Data     |  | Pub/Sub  |  | Cluster   |           |      |
|  |  | Mutation  |  | Dispatch |  | Messaging |           |      |
|  |  +----------+  +----------+  +-----------+           |      |
|  +-----------------------------------------------------+      |
|                        |                                       |
|            +-----------+-----------+                           |
|            |                       |                           |
|            v                       v                           |
|  +------------------+    +------------------+                  |
|  |   RDB Snapshots  |    |   AOF Journal    |                  |
|  |   (background    |    |   (append-only   |                  |
|  |    child proc)   |    |    fsync policy)  |                  |
|  +------------------+    +------------------+                  |
|                                                                |
+---------------------------------------------------------------+
```

### Single-Threaded Event Loop

Like Redis, Valkey's core command processing is single-threaded. This design:

- **Eliminates lock contention** — No mutexes needed for data structure access
- **Ensures atomicity** — Each command executes atomically without explicit locking
- **Simplifies reasoning** — Deterministic execution order
- **Reduces context switching** — No thread scheduling overhead for core logic

The event loop uses platform-specific multiplexing:
- `epoll` on Linux (most common in production)
- `kqueue` on macOS/BSD
- `evport` on Solaris

```
Event Loop Cycle
================

  +----> Read client requests (I/O threads assist)
  |              |
  |              v
  |      Parse commands
  |              |
  |              v
  |      Execute commands (main thread — single-threaded)
  |              |
  |              v
  |      Write responses (I/O threads assist)
  |              |
  |              v
  |      Handle time events (expiry, background tasks)
  |              |
  +------<-------+
```

### I/O Threading Model

Valkey enhances Redis's I/O threading model that was introduced in Redis 6.0.
While command execution remains single-threaded, network I/O operations (reading
from and writing to client sockets) can be distributed across multiple threads.

```
Without I/O Threading              With I/O Threading
====================               ==================

  Main Thread                      Main Thread      I/O Threads
  +-----------+                    +-----------+    +-----------+
  | Read      |                    | Dispatch  |    | Read req  |
  | Parse     |                    | Parse     |    | Read req  |
  | Execute   |                    | Execute   |    | Read req  |
  | Write     |                    | Dispatch  |    | Write res |
  | Read      |                    | Parse     |    | Write res |
  | Parse     |                    | Execute   |    | Write res |
  | Execute   |                    |           |    |           |
  | Write     |                    |           |    |           |
  +-----------+                    +-----------+    +-----------+

  All work on one thread           I/O offloaded to thread pool
```

Enable I/O threading in `valkey.conf`:

```bash
# Number of I/O threads (default: 1 = disabled)
# Recommended: set to number of CPU cores / 2, but no more than 8
io-threads 4

# Enable threaded reads (off by default, writes are always threaded)
io-threads-do-reads yes
```

### RESP Protocol — RESP2 and RESP3

Valkey supports both RESP (Redis Serialization Protocol) versions:

**RESP2** — Default protocol, text-based with simple type prefixes:

```
Client sends:    *3\r\n$3\r\nSET\r\n$3\r\nfoo\r\n$3\r\nbar\r\n
Server replies:  +OK\r\n

Type Prefixes:
  +  Simple String     +OK\r\n
  -  Error             -ERR unknown command\r\n
  :  Integer           :1000\r\n
  $  Bulk String       $3\r\nfoo\r\n
  *  Array             *2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n
```

**RESP3** — Enhanced protocol with richer types (use `HELLO 3` to upgrade):

```
Additional RESP3 types:
  _  Null              _\r\n
  #  Boolean           #t\r\n  or  #f\r\n
  ,  Double            ,3.14\r\n
  (  Big Number        (3492890328409238509324850943850943825024385\r\n
  %  Map               %2\r\n+key1\r\n+val1\r\n+key2\r\n+val2\r\n
  |  Attribute         (metadata before response)
  ~  Set               ~3\r\n+a\r\n+b\r\n+c\r\n
```

RESP3 enables client-side caching via server-assisted invalidation.

### Memory Management

Valkey uses the same memory allocator strategy as Redis:

- **jemalloc** — Default allocator (recommended for production)
- **libc malloc** — Alternative; compile-time option
- **tcmalloc** — Google's allocator; compile-time option

Key memory management features:

```bash
# Set maximum memory limit
maxmemory 4gb

# Eviction policy when maxmemory is reached
maxmemory-policy allkeys-lru
```

Eviction policies (identical to Redis):

| Policy               | Description                                        |
|---------------------|----------------------------------------------------|
| `noeviction`        | Return error on writes when memory limit reached    |
| `allkeys-lru`       | Evict least recently used keys from all keys        |
| `allkeys-lfu`       | Evict least frequently used keys from all keys      |
| `volatile-lru`      | Evict LRU keys only from keys with TTL set          |
| `volatile-lfu`      | Evict LFU keys only from keys with TTL set          |
| `allkeys-random`    | Evict random keys from all keys                     |
| `volatile-random`   | Evict random keys from keys with TTL set            |
| `volatile-ttl`      | Evict keys with shortest TTL first                  |

### Persistence — RDB and AOF

Valkey supports the same persistence mechanisms as Redis:

**RDB (Redis Database) Snapshots:**

```bash
# Save snapshot every 3600 seconds if at least 1 key changed
# Save snapshot every 300 seconds if at least 100 keys changed
# Save snapshot every 60 seconds if at least 10000 keys changed
save 3600 1
save 300 100
save 60 10000
```

**AOF (Append-Only File):**

```bash
appendonly yes
appendfsync everysec    # Options: always, everysec, no

# Auto-rewrite AOF when it doubles in size
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

**Hybrid Persistence (Recommended):**

```bash
aof-use-rdb-preamble yes   # Combines RDB snapshot with AOF tail
```

---

## New Features in Valkey

### Multi-Threaded I/O Enhancements

Valkey significantly improved the I/O threading model beyond what Redis 6.0
introduced. Key improvements include:

- **Reduced lock contention** between main thread and I/O threads
- **Better work distribution** across I/O threads
- **Improved batching** of client request processing
- **Reduced wake-up latency** for I/O threads

```
Valkey I/O Threading Improvements
==================================

Redis 6.0 I/O Threading:
  +------+    +------+    +------+
  | I/O  |    | I/O  |    | I/O  |
  | Th 1 |    | Th 2 |    | Th 3 |     Threads idle-wait
  +--+---+    +--+---+    +--+---+     with spin locks
     |           |           |
     +-----+-----+-----+----+
           |           |
     +-----v-----+    Main thread
     | Main Loop |    distributes
     +-----------+    and collects

Valkey Enhanced I/O Threading:
  +------+    +------+    +------+
  | I/O  |    | I/O  |    | I/O  |     Threads use efficient
  | Th 1 |    | Th 2 |    | Th 3 |     signaling (futex)
  +--+---+    +--+---+    +--+---+
     |           |           |
     +-----+-----+-----+----+
           |           |
     +-----v-----+    Better batching
     | Main Loop |    reduces round-trips
     +-----------+    between main and I/O
```

These improvements yield measurable throughput gains, especially under high
connection counts and high request rates (see Performance section).

### RDMA Support

Valkey 8.0 introduced experimental support for RDMA (Remote Direct Memory Access),
enabling ultra-low-latency communication between Valkey instances and between
clients and servers when connected via RDMA-capable networks (e.g., InfiniBand,
RoCE).

```
Traditional TCP/IP Path:
  Application → Kernel (TCP stack) → NIC → Network → NIC → Kernel → Application
  Latency: ~50-100 μs

RDMA Path:
  Application → NIC → Network → NIC → Application
  Latency: ~1-5 μs  (bypasses kernel entirely)

+-------------+                        +-------------+
|  Valkey     |     RDMA Network       |  Valkey     |
|  Primary    |<======================>|  Replica    |
|             |  Kernel bypass         |             |
|  User-space |  Zero-copy transfers   |  User-space |
|  NIC access |  Hardware offload      |  NIC access |
+-------------+                        +-------------+
```

RDMA is particularly beneficial for:
- Replication traffic between primary and replica nodes
- Cluster bus communication between nodes
- Ultra-low-latency client access in specialized environments

### Dual-Channel Replication

Traditional Redis replication uses a single connection for both the initial full
sync (RDB transfer) and ongoing command propagation. Valkey introduces dual-channel
replication that separates these concerns:

```
Traditional Single-Channel Replication:
=======================================

  Primary                          Replica
  +----------+                     +----------+
  |          |  --- RDB sync --->  |          |
  |          |  (blocks command    |          |
  |          |   propagation)      |          |
  |          |  --- commands --->  |          |
  +----------+                     +----------+

  Problem: During full sync, the replication buffer
  must hold ALL commands. If sync is slow, buffer
  can overflow → sync restarts → infinite loop.


Valkey Dual-Channel Replication:
================================

  Primary                          Replica
  +----------+                     +----------+
  |          |  === Channel 1 ===> |          |
  |          |  RDB full sync      |          |
  |          |  (dedicated conn)   |          |
  |          |                     |          |
  |          |  === Channel 2 ===> |          |
  |          |  Command stream     |          |
  |          |  (runs in parallel) |          |
  +----------+                     +----------+

  Benefit: Commands stream continuously on Channel 2
  while RDB transfers on Channel 1. No buffer overflow
  risk. Replica becomes consistent faster.
```

Key benefits of dual-channel replication:
- Eliminates replication buffer overflow during full syncs
- Reduces time to replica consistency after a full sync
- Improves stability for large datasets with high write rates
- Reduces risk of replication loops in degraded network conditions

### Per-Slot Dictionary

In cluster mode, Valkey introduced per-slot dictionaries that organize keys by
their hash slot rather than using a single global dictionary:

```
Redis Cluster: Global Dictionary
================================

  +--------------------------------------------+
  |            Global Key Dictionary            |
  |                                             |
  |  key1 → slot 1234   key5 → slot 8901       |
  |  key2 → slot 5678   key6 → slot 1234       |
  |  key3 → slot 1234   key7 → slot 5678       |
  |  key4 → slot 9012   key8 → slot 9012       |
  +--------------------------------------------+

  Slot migration requires scanning ALL keys
  to find keys belonging to the target slot.


Valkey Cluster: Per-Slot Dictionary
====================================

  +----------+  +----------+  +----------+  +----------+
  | Slot 1234|  | Slot 5678|  | Slot 8901|  | Slot 9012|
  |          |  |          |  |          |  |          |
  | key1     |  | key2     |  | key5     |  | key4     |
  | key3     |  | key7     |  |          |  | key8     |
  | key6     |  |          |  |          |  |          |
  +----------+  +----------+  +----------+  +----------+

  Slot migration directly accesses only the
  keys in the target slot. Much faster.
```

Benefits:
- **Faster slot migration** — No need to scan the entire keyspace
- **Reduced memory overhead** during resharding operations
- **Improved cluster scaling** operations
- **Better memory locality** for slot-level operations

### Improved Memory Efficiency

Valkey includes several memory optimizations:

- **Reduced per-key overhead** through data structure optimizations
- **Better memory fragmentation handling** with improved jemalloc integration
- **Optimized internal representations** for small objects
- **Improved dict resizing** algorithm reducing memory spikes during rehashing

### Valkey 8.x Features

Valkey 8.0 was the first major release with features beyond the Redis 7.2.4 base:

| Feature                       | Description                                  |
|------------------------------|----------------------------------------------|
| Multi-threaded I/O overhaul  | Significantly improved I/O thread performance |
| RDMA support (experimental)  | Kernel-bypass networking for ultra-low latency|
| Dual-channel replication     | Separate channels for sync and propagation    |
| Per-slot dictionary          | Organize keys by hash slot in cluster mode    |
| Embedded name in client info | Clients can embed identifiers in connections  |
| Performance improvements     | Various hotpath optimizations                 |
| Extended ACL support         | Enhanced access control capabilities          |

---

## Data Structures

### Full Compatibility with Redis Data Types

Valkey supports every data structure from Redis 7.2.4 with identical semantics,
encodings, and commands. Applications do not need any changes when migrating.

### Strings

The most basic data type. Can hold any binary data up to 512 MB.

```bash
# Basic operations
SET user:1:name "Alice"
GET user:1:name                     # "Alice"

# Atomic increment/decrement
SET counter 100
INCR counter                        # 101
INCRBY counter 50                   # 151
DECRBY counter 10                   # 141

# Bit operations
SETBIT flags 7 1
GETBIT flags 7                      # 1

# Expiry
SET session:abc123 "data" EX 3600   # Expires in 1 hour
TTL session:abc123                  # 3600
```

### Lists

Ordered collection of strings, implemented as a quicklist (linked list of ziplists).

```bash
LPUSH queue:tasks "task1" "task2" "task3"
RPOP queue:tasks                    # "task1"
LRANGE queue:tasks 0 -1            # ["task3", "task2"]
LLEN queue:tasks                    # 2

# Blocking pop (for job queues)
BRPOP queue:tasks 30               # Wait up to 30 seconds
```

### Sets

Unordered collection of unique strings.

```bash
SADD tags:post:1 "valkey" "caching" "database"
SADD tags:post:2 "valkey" "linux" "open-source"

SMEMBERS tags:post:1               # {"valkey", "caching", "database"}
SINTER tags:post:1 tags:post:2     # {"valkey"}
SUNION tags:post:1 tags:post:2     # {"valkey","caching","database","linux","open-source"}
SCARD tags:post:1                  # 3
```

### Sorted Sets

Like Sets but each member has an associated score for ordering.

```bash
ZADD leaderboard 1500 "player:alice"
ZADD leaderboard 2100 "player:bob"
ZADD leaderboard 1800 "player:carol"

ZRANGE leaderboard 0 -1 WITHSCORES
# player:alice 1500
# player:carol 1800
# player:bob   2100

ZREVRANGE leaderboard 0 0          # "player:bob" (top scorer)
ZRANK leaderboard "player:carol"   # 1 (0-indexed)
```

### Hashes

Maps of field-value pairs, ideal for representing objects.

```bash
HSET user:1 name "Alice" email "alice@example.com" age 30

HGET user:1 name                   # "Alice"
HGETALL user:1                     # {name: Alice, email: alice@example.com, age: 30}
HINCRBY user:1 age 1               # 31
HDEL user:1 email
```

### Streams

Append-only log data structure with consumer groups (introduced in Redis 5.0).

```bash
# Add entries to a stream
XADD events * type "click" page "/home"
XADD events * type "view" page "/products"

# Read entries
XRANGE events - +                  # All entries
XLEN events                        # 2

# Consumer groups
XGROUP CREATE events mygroup 0
XREADGROUP GROUP mygroup consumer1 COUNT 1 BLOCK 5000 STREAMS events >
XACK events mygroup <entry-id>
```

### Bitmaps and HyperLogLog

```bash
# Bitmaps — space-efficient boolean arrays
SETBIT daily:active:2025-01-15 user_id 1
BITCOUNT daily:active:2025-01-15   # Count active users

# HyperLogLog — probabilistic cardinality estimation
PFADD unique:visitors "user1" "user2" "user3"
PFCOUNT unique:visitors             # ~3 (0.81% standard error)
```

### Geospatial Indexes

```bash
GEOADD stores -122.4194 37.7749 "San Francisco"
GEOADD stores -118.2437 34.0522 "Los Angeles"

GEODIST stores "San Francisco" "Los Angeles" km   # ~559 km
GEOSEARCH stores FROMLONLAT -122.0 37.0 BYRADIUS 100 km ASC
```

### Data Structures Summary

```
+------------------+------------------+----------------------------------+
| Data Structure   | Internal Encoding| Best For                         |
+------------------+------------------+----------------------------------+
| String           | int / embstr /   | Caching, counters, flags,        |
|                  | raw              | serialized objects                |
+------------------+------------------+----------------------------------+
| List             | listpack /       | Queues, recent items, timelines  |
|                  | quicklist        |                                  |
+------------------+------------------+----------------------------------+
| Set              | listpack /       | Tags, unique items, membership   |
|                  | hashtable        | testing                          |
+------------------+------------------+----------------------------------+
| Sorted Set       | listpack /       | Leaderboards, priority queues,   |
|                  | skiplist+ht      | range queries                    |
+------------------+------------------+----------------------------------+
| Hash             | listpack /       | Object storage, configuration,   |
|                  | hashtable        | user profiles                    |
+------------------+------------------+----------------------------------+
| Stream           | rax tree of      | Event logs, messaging,           |
|                  | listpacks        | change data capture              |
+------------------+------------------+----------------------------------+
| Bitmap           | String (SDS)     | Boolean flags, daily active      |
|                  |                  | users, feature flags             |
+------------------+------------------+----------------------------------+
| HyperLogLog      | sparse / dense   | Unique visitor counting,         |
|                  | representation   | cardinality estimation           |
+------------------+------------------+----------------------------------+
| Geospatial       | Sorted Set       | Location-based queries,          |
|                  | (encoded)        | proximity search                 |
+------------------+------------------+----------------------------------+
```

---

## Cluster Mode

### Hash Slot Architecture — 16384 Slots

Valkey Cluster uses the same 16384 hash-slot architecture as Redis Cluster:

```
Hash Slot Calculation:    slot = CRC16(key) mod 16384

Key: "user:1000"
CRC16("user:1000") = 14789
14789 mod 16384 = 14789
→ Slot 14789

                    16384 Hash Slots
  +------+------+------+------+------+------+
  | 0-   | 5461-| 10923|      |      |      |
  | 5460 | 10922| 16383|      |      |      |
  +------+------+------+------+------+------+
  Node A  Node B  Node C  (3-node cluster)
```

Hash tags allow related keys to map to the same slot:

```bash
# These keys all hash to the same slot (based on "user:1000")
SET {user:1000}:name "Alice"
SET {user:1000}:email "alice@example.com"
SET {user:1000}:prefs "{...}"
```

### Cluster Topology

```
+-------------------------------------------------------------------+
|                     Valkey Cluster (6 nodes)                       |
|                                                                   |
|  +-------------------+  +-------------------+  +---------------+  |
|  | Node A (Primary)  |  | Node B (Primary)  |  | Node C (Prim) |  |
|  | Slots: 0-5460     |  | Slots: 5461-10922 |  | Slots: 10923- |  |
|  |                   |  |                   |  |        16383  |  |
|  +--------+----------+  +--------+----------+  +-------+-------+  |
|           |                      |                      |         |
|    Replication            Replication            Replication      |
|           |                      |                      |         |
|  +--------v----------+  +--------v----------+  +-------v-------+  |
|  | Node D (Replica)  |  | Node E (Replica)  |  | Node F (Repl) |  |
|  | Slots: 0-5460     |  | Slots: 5461-10922 |  | Slots: 10923- |  |
|  | (read replicas)   |  | (read replicas)   |  |        16383  |  |
|  +-------------------+  +-------------------+  +---------------+  |
|                                                                   |
|  Cluster Bus: Each node connects to every other node (gossip)     |
+-------------------------------------------------------------------+
```

### Replication Enhancements

Valkey improves replication over Redis with dual-channel replication
(see New Features section). Additional enhancements include:

- **Faster full resync** — Optimized RDB transfer with parallel I/O
- **Reduced replication buffer pressure** — Dual-channel design prevents overflow
- **Better partial resync** — Improved backlog handling after brief disconnections

```bash
# Replication configuration
replicaof <primary-host> <primary-port>

# Enable dual-channel replication (Valkey 8.0+)
repl-dual-channel yes

# Replication backlog size
repl-backlog-size 256mb
```

### Improved Failover

Valkey cluster failover remains compatible with Redis Cluster failover but
includes stability improvements:

```
Failover Process:
=================

1. Replica detects primary is unreachable
   (after cluster-node-timeout milliseconds)

2. Replica increments currentEpoch and requests votes

3. Other primaries vote (FAILOVER_AUTH_ACK)

4. If majority of primaries vote YES:
   - Replica promotes itself to primary
   - Takes ownership of the failed primary's slots
   - Announces new configuration to cluster

5. Clients receive MOVED redirections to new primary

Timeline:
  T+0s         Primary fails
  T+15s        Detected (cluster-node-timeout default)
  T+15-17s     Election and promotion
  T+17s        New primary serving requests
```

### Cluster Configuration

```bash
# valkey.conf — Cluster settings
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 15000

# Require full slot coverage for cluster to accept writes
cluster-require-full-coverage yes

# Allow reads from replicas during failover
cluster-allow-reads-when-down yes

# Migration barrier — minimum replicas before slot migration
cluster-migration-barrier 1
```

Create a cluster:

```bash
# Start 6 Valkey instances (3 primaries + 3 replicas)
valkey-server --port 7000 --cluster-enabled yes --cluster-config-file nodes-7000.conf
valkey-server --port 7001 --cluster-enabled yes --cluster-config-file nodes-7001.conf
valkey-server --port 7002 --cluster-enabled yes --cluster-config-file nodes-7002.conf
valkey-server --port 7003 --cluster-enabled yes --cluster-config-file nodes-7003.conf
valkey-server --port 7004 --cluster-enabled yes --cluster-config-file nodes-7004.conf
valkey-server --port 7005 --cluster-enabled yes --cluster-config-file nodes-7005.conf

# Create the cluster
valkey-cli --cluster create \
  127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
  127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
  --cluster-replicas 1
```

---

## Migration from Redis to Valkey

### Migration Assessment

Before migrating, assess your environment:

```
Migration Assessment Checklist
==============================

1. Redis Version
   □ Redis 7.2.x  → Direct migration (highest compatibility)
   □ Redis 7.0.x  → Compatible (minor config differences)
   □ Redis 6.x    → Compatible (but test thoroughly)
   □ Redis 5.x    → Upgrade to 7.2 first, then migrate

2. Feature Usage
   □ Core data structures          → Fully supported
   □ Pub/Sub                       → Fully supported
   □ Streams + Consumer Groups     → Fully supported
   □ Lua Scripting / Functions     → Fully supported
   □ Redis Modules                 → Check compatibility per module
   □ ACLs                          → Fully supported
   □ Cluster Mode                  → Fully supported
   □ Sentinel                      → Fully supported

3. Client Libraries
   □ Identify all client libraries in use
   □ Verify each supports Valkey (most do out of the box)
   □ Check for Redis-version-specific feature usage

4. Managed Service
   □ Identify current hosting (self-hosted, AWS, GCP, Azure)
   □ Check target managed service Valkey support
   □ Plan data migration path
```

### Client Library Compatibility

Existing Redis client libraries work with Valkey unchanged because Valkey
speaks the same RESP protocol:

```
+---------------------+----------+------------+---------------------------+
| Library             | Language | Valkey     | Notes                     |
|                     |          | Compatible |                           |
+---------------------+----------+------------+---------------------------+
| jedis               | Java     | Yes        | Change connection URL     |
| lettuce             | Java     | Yes        | Change connection URL     |
| ioredis             | Node.js  | Yes        | Change host/port config   |
| node-redis          | Node.js  | Yes        | Change connection config  |
| redis-py            | Python   | Yes        | Change host/port          |
| go-redis            | Go       | Yes        | Change address config     |
| StackExchange.Redis | .NET     | Yes        | Change connection string  |
| hiredis             | C        | Yes        | Change connection params  |
| Predis              | PHP      | Yes        | Change connection params  |
| redis-rb            | Ruby     | Yes        | Change host/port          |
| redis-rs            | Rust     | Yes        | Change connection URL     |
+---------------------+----------+------------+---------------------------+
```

Some libraries have also released Valkey-specific clients:

```
+---------------------+----------+------------------------------------------+
| Library             | Language | Description                              |
+---------------------+----------+------------------------------------------+
| valkey-java (VALKEYJ)| Java   | Official Valkey client for Java          |
| valkey-py           | Python   | Official Valkey client for Python        |
| valkey-go (valkey-go)| Go     | Official Valkey client for Go            |
| valkey-glide        | Multi    | Valkey GLIDE client (Java, Python, Node) |
+---------------------+----------+------------------------------------------+
```

### Configuration Mapping

Most Redis configuration directives work in Valkey. Key differences:

```bash
# Binary names
redis-server    → valkey-server
redis-cli       → valkey-cli
redis-sentinel  → valkey-sentinel
redis-benchmark → valkey-benchmark

# Configuration file
redis.conf      → valkey.conf

# Default port remains the same
port 6379

# Key config directives (identical syntax):
maxmemory 4gb
maxmemory-policy allkeys-lru
appendonly yes
appendfsync everysec
save 3600 1
save 300 100
save 60 10000
cluster-enabled yes

# Valkey accepts both 'redis' and 'valkey' prefixed directives
# for backward compatibility during migration
```

### Data Migration Strategies

**Strategy 1: RDB Snapshot Migration (Simplest)**

```
Step 1: Create RDB on Redis          Step 2: Copy to Valkey
+------------------+                 +------------------+
|  Redis Primary   |                 |  Valkey Server   |
|                  |   dump.rdb      |                  |
|  BGSAVE -------> | =============>  | Load on startup  |
|                  |   (scp/rsync)   |                  |
+------------------+                 +------------------+

Downtime: Minutes (during copy + restart)
Data loss: Commands executed after BGSAVE
```

```bash
# On Redis
redis-cli BGSAVE
# Wait for completion
redis-cli LASTSAVE

# Copy RDB file
scp /var/lib/redis/dump.rdb valkey-host:/var/lib/valkey/dump.rdb

# Start Valkey (loads RDB automatically)
valkey-server /etc/valkey/valkey.conf
```

**Strategy 2: Live Replication Migration (Zero-Downtime)**

```
Phase 1: Set up Valkey as replica of Redis

  +------------------+     replication     +------------------+
  |  Redis Primary   | =================> |  Valkey Replica  |
  |  (still serving) |                    |  (syncing)       |
  +------------------+                    +------------------+

Phase 2: Promote Valkey, redirect clients

  +------------------+                    +------------------+
  |  Redis (old)     |                    |  Valkey Primary  |
  |  (decommission)  |                    |  (now serving)   |
  +------------------+                    +------------------+
                                                  ^
                                                  |
                                           All clients now
                                           connect here
```

```bash
# On Valkey server — set up as replica of Redis
valkey-cli REPLICAOF redis-primary-host 6379

# Monitor replication progress
valkey-cli INFO replication
# Wait until master_link_status:up and repl_offset matches

# Promote Valkey to primary
valkey-cli REPLICAOF NO ONE

# Update client configurations to point to Valkey
# Update DNS / load balancer / service discovery
```

**Strategy 3: Dual-Write Migration (Most Complex, Safest)**

```
+------------------+
|   Application    |
|   (dual-write)   |
+--------+---------+
         |
    +----+----+
    |         |
    v         v
+--------+ +--------+
| Redis  | | Valkey |     Both receive writes
| (old)  | | (new)  |     Read from Redis initially
+--------+ +--------+     Gradually shift reads to Valkey
                          Decommission Redis when confident
```

### Step-by-Step Migration Guide

```
Phase 1: Preparation
=====================
1. Deploy Valkey alongside Redis (same version compatibility)
2. Verify valkey-cli connects and runs basic commands
3. Run valkey-benchmark to baseline performance
4. Test client library connectivity

Phase 2: Data Sync
===================
5. Set Valkey as replica of Redis (REPLICAOF)
6. Monitor replication lag until caught up
7. Verify data integrity with spot-checks

Phase 3: Client Migration
==========================
8. Update client configurations (DNS, connection strings)
9. Redirect read traffic to Valkey first
10. Redirect write traffic to Valkey
11. Promote Valkey to primary (REPLICAOF NO ONE)

Phase 4: Validation
====================
12. Monitor error rates, latency, throughput
13. Run integration test suite against Valkey
14. Verify all features work (Pub/Sub, Streams, Lua, etc.)

Phase 5: Cleanup
================
15. Decommission old Redis instances
16. Update monitoring dashboards and alerts
17. Document the new architecture
```

### Rollback Plan

```
If issues arise after migration:
=================================

Immediate (< 1 hour):
  1. Switch client configs back to Redis
  2. If data diverged, use Redis as source of truth

Short-term (< 24 hours):
  1. Set Redis as replica of Valkey to sync new data
  2. Promote Redis back to primary
  3. Redirect clients to Redis
  4. Investigate Valkey issues

Prevention:
  - Keep Redis running in read-only mode for 24-48 hours post-migration
  - Maintain RDB snapshots from both systems
  - Monitor both systems during transition period
```

### Testing Checklist

```
Pre-Migration Tests:
  □ Connection test with all client libraries
  □ CRUD operations for all data types used
  □ Pub/Sub publish and subscribe
  □ Stream XADD and XREADGROUP
  □ Lua script execution (EVAL and FUNCTION)
  □ ACL authentication and authorization
  □ TLS/SSL connection (if used)
  □ Cluster operations (if cluster mode)
  □ Sentinel failover (if Sentinel mode)
  □ Benchmark comparison (valkey-benchmark vs redis-benchmark)

Post-Migration Tests:
  □ Application integration tests pass
  □ Latency within acceptable range (p50, p99)
  □ Throughput meets requirements
  □ Memory usage comparable to Redis
  □ Persistence working (RDB and/or AOF)
  □ Replication healthy (if applicable)
  □ Monitoring and alerting functional
  □ Backup and restore procedures verified
```

---

## Cloud Provider Support

### AWS ElastiCache for Valkey

AWS was the first major cloud provider to offer native Valkey support through
Amazon ElastiCache:

```
+-------------------------------------------------------------------+
|                  AWS ElastiCache for Valkey                         |
|                                                                   |
|  +-------------------+     +-------------------+                  |
|  | Valkey Primary    |     | Valkey Primary    |                  |
|  | (AZ-a)           |     | (AZ-b)           |                  |
|  +--------+----------+     +--------+----------+                  |
|           |                         |                             |
|  +--------v----------+     +--------v----------+                  |
|  | Valkey Replica    |     | Valkey Replica    |                  |
|  | (AZ-b)           |     | (AZ-c)           |                  |
|  +-------------------+     +-------------------+                  |
|                                                                   |
|  Features:                                                        |
|  - Cluster mode and non-cluster mode                              |
|  - Automatic failover                                             |
|  - Encryption at rest and in transit (TLS)                        |
|  - VPC support and security groups                                |
|  - CloudWatch metrics integration                                 |
|  - Backup and restore                                             |
|  - Online scaling (add/remove shards and replicas)                |
+-------------------------------------------------------------------+
```

Key AWS ElastiCache Valkey features:
- Native support (not just Redis-compatible mode)
- Serverless option for automatic scaling
- Reserved nodes for cost optimization
- Global Datastore for cross-region replication
- Data tiering (memory + SSD) for cost-effective large datasets

### Google Cloud Memorystore for Valkey

Google Cloud offers Memorystore for Valkey:

- Fully managed Valkey instances
- Cluster mode support
- Automatic failover and patching
- VPC peering for secure access
- Integration with Cloud Monitoring and Logging
- Support for Valkey 7.2 and 8.0+

### Azure Cache Compatibility

Azure Cache for Redis remains Redis-branded but is protocol-compatible with
Valkey clients. For self-hosted Valkey on Azure:

- Deploy on Azure VMs or Azure Kubernetes Service (AKS)
- Use Azure Virtual Network for isolation
- Monitor with Azure Monitor and custom dashboards
- Use Azure Managed Disks for persistence storage

### Other Providers and Self-Hosted

| Provider         | Offering                          | Valkey Support        |
|-----------------|-----------------------------------|-----------------------|
| Oracle Cloud    | OCI Cache with Valkey             | Native                |
| Aiven           | Aiven for Valkey                  | Native                |
| DigitalOcean    | Managed Databases                 | Valkey option         |
| Self-hosted     | Docker, Kubernetes, bare metal    | Full control          |

Self-hosted deployment options:

```bash
# Docker
docker run -d --name valkey -p 6379:6379 valkey/valkey:8

# Docker with persistence
docker run -d --name valkey \
  -p 6379:6379 \
  -v valkey-data:/data \
  valkey/valkey:8 \
  valkey-server --save 60 1000 --appendonly yes

# Kubernetes (Helm chart)
helm repo add valkey https://valkey-io.github.io/valkey-helm
helm install my-valkey valkey/valkey
```

### Cloud Provider Comparison

```
+-------------------+------------+-----------+----------+-----------+
| Feature           | AWS        | GCP       | Azure    | Self-Host |
|                   | ElastiCache| Memorystore| (manual)| (Docker/  |
|                   |            |           |          | K8s)      |
+-------------------+------------+-----------+----------+-----------+
| Native Valkey     | Yes        | Yes       | No       | Yes       |
| Cluster mode      | Yes        | Yes       | N/A      | Yes       |
| Auto-failover     | Yes        | Yes       | N/A      | Manual    |
| Serverless        | Yes        | No        | N/A      | No        |
| Multi-AZ          | Yes        | Yes       | N/A      | Manual    |
| TLS encryption    | Yes        | Yes       | N/A      | Manual    |
| Backup/Restore    | Managed    | Managed   | N/A      | Manual    |
| Cost optimization | Reserved   | Committed | N/A      | Full ctrl |
|                   | Nodes      | Use       |          |           |
+-------------------+------------+-----------+----------+-----------+
```

---

## Valkey Modules

### Module API Compatibility

Valkey maintains compatibility with the Redis Module API, meaning most Redis
modules can be loaded into Valkey without modification:

```
+---------------------------------------------------------------+
|                     Valkey Module System                        |
|                                                                |
|  +------------------+  +------------------+  +--------------+  |
|  |  Module API      |  |  Module API      |  |  Module API  |  |
|  |  (Redis compat)  |  |  (Valkey native) |  |  (Hybrid)    |  |
|  +------------------+  +------------------+  +--------------+  |
|           |                     |                    |          |
|           v                     v                    v          |
|  +------------------+  +------------------+  +--------------+  |
|  |  Existing Redis  |  |  New Valkey      |  |  Community   |  |
|  |  modules work    |  |  modules         |  |  modules     |  |
|  +------------------+  +------------------+  +--------------+  |
+---------------------------------------------------------------+
```

Key considerations:
- Modules compiled against Redis Module API headers will work
- The `RedisModule_` function prefix is supported alongside `ValkeyModule_`
- Module RDB format compatibility is maintained
- Module commands appear in `MODULE LIST` and `COMMAND INFO`

### Community Modules

Many community modules are compatible with Valkey:

| Module          | Purpose                  | Valkey Compatible |
|----------------|--------------------------|-------------------|
| RedisJSON      | JSON document store      | Yes (open-source) |
| RediSearch      | Full-text search engine  | Yes (open-source) |
| RedisTimeSeries | Time series database     | Yes (open-source) |
| RedisBloom      | Probabilistic structures | Yes (open-source) |
| RedisGears      | Serverless engine        | Community support |

Note: Redis Ltd. modules that were relicensed may have community forks
that are maintained under open-source licenses.

### Loading Modules

```bash
# Load module at startup (valkey.conf)
loadmodule /path/to/module.so

# Load module at runtime
valkey-cli MODULE LOAD /path/to/module.so

# List loaded modules
valkey-cli MODULE LIST

# Unload a module
valkey-cli MODULE UNLOAD modulename
```

---

## Community and Governance

### Linux Foundation Project Structure

Valkey is a Linux Foundation project, which provides:

```
+---------------------------------------------------------------+
|                   Linux Foundation                             |
|                                                                |
|  Legal Entity     Trademark       Neutral       Infrastructure |
|  & Fiscal         Protection      Governance    & CI/CD        |
|  Sponsorship                      Framework     Support        |
|                                                                |
+---------------------------------------------------------------+
                            |
                            v
+---------------------------------------------------------------+
|                     Valkey Project                              |
|                                                                |
|  +-------------------+  +--------------------+                 |
|  | Technical Steering |  | Community          |                 |
|  | Committee (TSC)    |  | Contributors       |                 |
|  |                    |  |                    |                 |
|  | - Architecture     |  | - Code reviews     |                 |
|  | - Release decisions|  | - Bug reports      |                 |
|  | - API stability    |  | - Documentation    |                 |
|  | - Security policy  |  | - Testing          |                 |
|  +-------------------+  +--------------------+                 |
|                                                                |
|  +-------------------+  +--------------------+                 |
|  | Maintainers       |  | Working Groups     |                 |
|  |                    |  |                    |                 |
|  | - Merge authority  |  | - Performance WG   |                 |
|  | - Release cutting  |  | - Cluster WG       |                 |
|  | - Security fixes   |  | - Modules WG       |                 |
|  +-------------------+  +--------------------+                 |
+---------------------------------------------------------------+
```

### Technical Steering Committee

The TSC is responsible for:

- Setting technical direction and architecture decisions
- Approving major feature proposals
- Managing release schedules
- Handling security disclosures and patches
- Maintaining backward compatibility policies

TSC members are elected from active maintainers and represent a diverse set
of companies to ensure vendor neutrality.

### Contributing to Valkey

```
Contribution Workflow
=====================

1. Find or create an issue on GitHub (github.com/valkey-io/valkey)

2. Fork the repository

3. Create a feature branch
   $ git checkout -b feature/my-improvement

4. Make changes following the coding style guide

5. Write tests for new functionality

6. Submit a pull request

7. Address review feedback

8. Maintainer merges after approval

Coding Standards:
  - C code follows the existing Redis/Valkey style
  - New features require tests in the test suite
  - Documentation updates for user-facing changes
  - Performance benchmarks for performance-related changes
```

### Release Cadence

| Release Type | Frequency      | Description                              |
|-------------|----------------|------------------------------------------|
| Major (x.0) | ~12 months     | New features, potential breaking changes  |
| Minor (x.y) | ~3-4 months    | Bug fixes, minor improvements             |
| Patch (x.y.z)| As needed     | Security fixes, critical bug fixes        |

Release support policy:
- Current major version: Full support (features + bug fixes + security)
- Previous major version: Maintenance (bug fixes + security only)
- Older versions: Security fixes only (for critical vulnerabilities)

### Roadmap

The Valkey roadmap is publicly available and community-driven. Key focus areas:

- **Performance** — Continued multi-threaded improvements
- **Scalability** — Better cluster operations and larger cluster support
- **Efficiency** — Memory optimization and resource utilization
- **Ecosystem** — Module ecosystem and client library improvements
- **Observability** — Enhanced metrics and diagnostics
- **Security** — Ongoing hardening and audit

---

## Performance

### Benchmarks vs Redis

Valkey aims to match or exceed Redis performance. Since both share the same
codebase at the fork point, baseline performance is identical. Valkey's
improvements focus on I/O threading and replication.

```
Benchmark: valkey-benchmark -t set,get -n 1000000 -c 256 -P 16 --threads 4

Note: Results vary by hardware, configuration, and workload.
      These are illustrative of relative performance characteristics.

Single-threaded mode (io-threads 1):
+------------------+-------------+-------------+
| Operation        | Redis 7.2   | Valkey 7.2  |
+------------------+-------------+-------------+
| SET (ops/sec)    | ~600,000    | ~600,000    |
| GET (ops/sec)    | ~650,000    | ~650,000    |
| p99 latency      | ~1.2 ms     | ~1.2 ms     |
+------------------+-------------+-------------+
  (Identical — same codebase)

Multi-threaded mode (io-threads 4):
+------------------+-------------+-------------+
| Operation        | Redis 7.2   | Valkey 8.0  |
+------------------+-------------+-------------+
| SET (ops/sec)    | ~900,000    | ~1,200,000  |
| GET (ops/sec)    | ~1,000,000  | ~1,400,000  |
| p99 latency      | ~1.5 ms     | ~1.1 ms     |
+------------------+-------------+-------------+
  (Valkey improved I/O threading shows gains)
```

### Multi-Threaded I/O Gains

The primary performance advantage of Valkey comes from I/O threading improvements:

```
Throughput Scaling with I/O Threads
===================================

  ops/sec
  (millions)
    |
  2.0|                                          * Valkey 8.0
    |                                    *
  1.5|                             *
    |                       o           o Redis 7.2
    |                 *           o
  1.0|           o          o
    |     *     o     o
    |     o
  0.5|
    |
  0.0+----+----+----+----+----+----+----+-->
    1    2    3    4    5    6    7    8
                  I/O Threads

  * Valkey 8.0 scales more efficiently with I/O threads
  o Redis 7.2 shows diminishing returns at higher thread counts
```

Recommendations for I/O thread configuration:

| CPU Cores | Recommended io-threads | Expected Gain |
|-----------|----------------------|---------------|
| 2         | 1 (disabled)         | Baseline      |
| 4         | 2                    | ~30-50%       |
| 8         | 4                    | ~80-120%      |
| 16+       | 6-8                  | ~150-200%     |

### Latency Characteristics

```
Latency Distribution (typical, single-threaded):
=================================================

  p50:   ~0.1 ms    (median — most requests)
  p90:   ~0.2 ms    (90th percentile)
  p99:   ~0.5 ms    (99th percentile)
  p99.9: ~1.0 ms    (99.9th — occasional jitter)

Latency Sources:
  +----------------------------------------------------------+
  | Factor              | Impact   | Mitigation              |
  +----------------------------------------------------------+
  | Network RTT         | ~0.05 ms | Co-locate client/server |
  | Command processing  | ~0.01 ms | Avoid slow commands     |
  | I/O wait            | ~0.02 ms | Enable I/O threads      |
  | Persistence (AOF)   | ~0.1 ms  | Use everysec fsync      |
  | Large keys/values   | Variable | Keep values < 10 KB     |
  | Fork (BGSAVE)       | Spikes   | Schedule during low use |
  +----------------------------------------------------------+
```

### Throughput Under Load

```bash
# Benchmark with pipelining and multiple clients
valkey-benchmark -h 127.0.0.1 -p 6379 \
  -t set,get,incr,lpush,rpush,sadd,zadd,hset \
  -n 1000000 \
  -c 256 \
  -P 16 \
  --threads 4 \
  -q

# Expected output (varies by hardware):
# SET: 1,200,000 requests per second
# GET: 1,400,000 requests per second
# INCR: 1,300,000 requests per second
# LPUSH: 1,100,000 requests per second
# RPUSH: 1,100,000 requests per second
# SADD: 1,200,000 requests per second
# ZADD: 1,000,000 requests per second
# HSET: 1,100,000 requests per second
```

---

## When to Choose Valkey

### Decision Framework

```
                    Should You Use Valkey?
                    =====================

                    Do you need an in-memory
                    key-value data store?
                          |
                         Yes
                          |
                    Are you currently
                    using Redis?
                     /          \
                   Yes           No
                   /              \
              Do you need       Does open-source
              open-source       licensing matter?
              licensing?         /         \
               /    \          Yes          No
             Yes     No        |            |
              |      |    Choose Valkey   Either works;
              |      |    (BSD-3-Clause   evaluate both
              |      |    from day one)   on features
              |      |
              |   Is Redis Ltd.
              |   licensing OK
              |   for your use?
              |    /        \
              |  Yes         No
              |   |          |
              |   Stay       |
              |   with       |
              |   Redis      |
              |              |
              +--> Migrate to Valkey <--+
```

### Open-Source Licensing Needs

Choose Valkey if:
- Your organization requires OSI-approved open-source licenses
- You are building a managed service or SaaS product that includes caching
- Your legal team has concerns about RSALv2/SSPL restrictions
- You contribute to open-source and want your contributions under BSD
- You want to avoid future license change risk (Linux Foundation governance)

### Cloud Provider Strategy

Choose Valkey if:
- You use AWS ElastiCache (native Valkey support)
- You use Google Cloud Memorystore (native Valkey support)
- You want multi-cloud portability with a truly open-source engine
- You self-host and want freedom to offer as a service

### Community-Driven Development

Choose Valkey if:
- You value vendor-neutral governance
- You want to influence the project roadmap through community participation
- You prefer Linux Foundation project structure and stability
- You want diverse contributor backing (not single-company controlled)

### When Redis May Still Be Preferred

Redis may still be the right choice if:
- You are already using Redis Cloud (managed by Redis Ltd.)
- You need specific Redis 7.4+ or 8.0+ features not yet in Valkey
- You use proprietary Redis modules (not available as open-source)
- You have a commercial agreement with Redis Ltd.
- Your organization has no licensing concerns with RSALv2/SSPL

---

## Best Practices

### Configuration Recommendations

```bash
# valkey.conf — Production recommendations

# Memory
maxmemory 4gb                          # Set explicit limit
maxmemory-policy allkeys-lru           # LRU eviction for cache use cases

# Persistence (choose based on durability needs)
save 3600 1                            # RDB: snapshot every hour if 1+ changes
save 300 100                           # RDB: snapshot every 5 min if 100+ changes
appendonly yes                         # Enable AOF for durability
appendfsync everysec                   # Balance durability and performance
aof-use-rdb-preamble yes              # Hybrid persistence (recommended)

# Networking
tcp-backlog 511                        # Increase for high connection rates
timeout 300                            # Close idle connections after 5 minutes
tcp-keepalive 60                       # Detect dead connections

# I/O Threading (Valkey enhancement)
io-threads 4                           # Match to available CPU cores / 2
io-threads-do-reads yes                # Enable threaded reads

# Slow Log
slowlog-log-slower-than 10000         # Log commands slower than 10ms
slowlog-max-len 128                    # Keep last 128 slow commands

# Disable dangerous commands in production
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command DEBUG ""
rename-command CONFIG "CONFIG_8a2f9b"  # Rename to obscure string

# Background save error handling
stop-writes-on-bgsave-error yes        # Stop writes if RDB save fails

# Lazy freeing (async deletion — reduces latency spikes)
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
lazyfree-lazy-user-del yes
```

### Security Hardening

```bash
# ACL Configuration
# Create specific users with minimal permissions
ACL SETUSER app-read on >strongpassword123 ~app:* +get +mget +hget +hgetall
ACL SETUSER app-write on >strongpassword456 ~app:* +set +hset +del +expire
ACL SETUSER admin on >adminpassword789 ~* +@all

# Disable default user or set a password
requirepass your-strong-password-here

# TLS Configuration
tls-port 6380
tls-cert-file /path/to/valkey.crt
tls-key-file /path/to/valkey.key
tls-ca-cert-file /path/to/ca.crt
tls-auth-clients optional        # or 'yes' for mutual TLS

# Network binding
bind 10.0.1.100                  # Bind to specific interface, not 0.0.0.0
protected-mode yes               # Reject connections without auth from non-loopback

# Disable commands that should not be exposed
rename-command KEYS ""           # KEYS is O(n) and dangerous in production
rename-command MIGRATE ""        # Prevent data exfiltration
rename-command SLAVEOF ""        # Prevent unauthorized replication changes
rename-command REPLICAOF ""
```

Security checklist:

```
□ Set requirepass or configure ACL users
□ Enable TLS for all connections
□ Bind to specific network interfaces
□ Enable protected-mode
□ Disable or rename dangerous commands
□ Use network firewalls / security groups
□ Rotate passwords regularly
□ Monitor AUTH failures in logs
□ Keep Valkey updated (security patches)
□ Run Valkey as non-root user
```

### Monitoring and Observability

Key metrics to monitor:

```bash
# Memory metrics
valkey-cli INFO memory
# used_memory, used_memory_rss, mem_fragmentation_ratio

# Performance metrics
valkey-cli INFO stats
# instantaneous_ops_per_sec, total_commands_processed

# Replication metrics
valkey-cli INFO replication
# master_link_status, master_last_io_seconds_ago, repl_backlog_size

# Client metrics
valkey-cli INFO clients
# connected_clients, blocked_clients, tracking_clients

# Keyspace metrics
valkey-cli INFO keyspace
# keys count, expires count per database
```

```
Essential Monitoring Dashboard
===============================

+---------------------------------------------+
|  Memory Usage          |  Operations/sec     |
|  ██████████░░ 75%      |  ▁▂▃▅▆▇▆▅▃▂▁     |
|  3.0 GB / 4.0 GB      |  125,000 ops/sec    |
+---------------------------------------------+
|  Connected Clients     |  Hit Rate           |
|  ████░░░░░░░░ 32%      |  ██████████░ 95.2%  |
|  128 / 400 max         |  Cache effectiveness|
+---------------------------------------------+
|  Replication Lag       |  Evicted Keys       |
|  0.2 seconds           |  0 (healthy)        |
|  ███░░░░░░░░░          |  ░░░░░░░░░░░░       |
+---------------------------------------------+
|  Slow Log Entries (last hour): 3             |
|  Fragmentation Ratio: 1.05 (healthy)        |
+---------------------------------------------+
```

Recommended monitoring tools:
- **Prometheus + Grafana** — Use valkey-exporter for metrics
- **Datadog** — Native Valkey integration
- **CloudWatch** — For AWS ElastiCache Valkey
- **Custom scripts** — Parse `INFO` command output

### Operational Tips

**Key naming conventions:**

```bash
# Use colons as separators
user:1000:profile
order:2024:01:15:abc123
cache:api:v2:endpoint:/users

# Use consistent prefixes for easy management
app:session:*         # All sessions
app:cache:*           # All cache entries
app:queue:*           # All queue keys
```

**Avoid common pitfalls:**

```
+-------------------------------+----------------------------------------+
| Pitfall                       | Solution                               |
+-------------------------------+----------------------------------------+
| Using KEYS in production      | Use SCAN with COUNT hint instead       |
| Large values (> 100 KB)       | Compress or split into smaller keys    |
| Missing TTLs on cache keys    | Always set EXPIRE or use EX/PX in SET |
| No maxmemory limit            | Always set maxmemory in production     |
| Single point of failure       | Use Cluster or Sentinel                |
| No persistence for important  | Enable AOF with everysec fsync         |
| data                          |                                        |
| Blocking commands on main     | Use BRPOP timeout, avoid long BLPOP    |
| Not monitoring memory         | Alert on used_memory > 80% maxmemory  |
| Ignoring fragmentation ratio  | Restart if ratio > 1.5 persistently    |
+-------------------------------+----------------------------------------+
```

**Backup strategy:**

```bash
# Automated RDB backups
# Create a cron job for periodic backups
# Example: backup every 6 hours

0 */6 * * * valkey-cli BGSAVE && \
  sleep 10 && \
  cp /var/lib/valkey/dump.rdb /backups/valkey/dump-$(date +\%Y\%m\%d-\%H\%M).rdb

# Verify backup integrity
valkey-check-rdb /backups/valkey/dump-latest.rdb
```

### Upgrade Strategy

```
Upgrade Path: Valkey 7.2.x → 8.x
==================================

1. Read release notes and breaking changes
2. Test in staging environment first
3. Back up data (RDB snapshot)
4. Upgrade replicas first, then primary
5. Monitor for issues after each node upgrade
6. Verify all features work post-upgrade

For Cluster Mode:
  - Rolling upgrade: one node at a time
  - Upgrade replicas first
  - Failover primary → upgraded replica
  - Upgrade old primary (now replica)
  - Repeat for each shard

  +--------+     +--------+     +--------+
  | Shard1 |     | Shard2 |     | Shard3 |
  | P: 7.2 |     | P: 7.2 |     | P: 7.2 |
  | R: 7.2 |     | R: 7.2 |     | R: 7.2 |
  +--------+     +--------+     +--------+
       |
       v  Upgrade replica to 8.0
  +--------+     +--------+     +--------+
  | P: 7.2 |     | P: 7.2 |     | P: 7.2 |
  | R: 8.0 |     | R: 7.2 |     | R: 7.2 |
  +--------+     +--------+     +--------+
       |
       v  Failover: replica becomes primary
  +--------+     +--------+     +--------+
  | P: 8.0 |     | P: 7.2 |     | P: 7.2 |
  | R: 7.2 |     | R: 7.2 |     | R: 7.2 |
  +--------+     +--------+     +--------+
       |
       v  Upgrade old primary (now replica)
  +--------+     +--------+     +--------+
  | P: 8.0 |     | P: 7.2 |     | P: 7.2 |
  | R: 8.0 |     | R: 7.2 |     | R: 7.2 |
  +--------+     +--------+     +--------+
       |
       v  Repeat for Shard2, then Shard3...
```

---

## Version History

| Date       | Description     | Author |
|------------|-----------------|--------|
| 2025-04-15 | Initial version | Team   |
