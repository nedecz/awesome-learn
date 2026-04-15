# Memcached

## Table of Contents

- [Overview](#overview)
  - [What is Memcached](#what-is-memcached)
  - [History](#history)
  - [Design Philosophy](#design-philosophy)
  - [Licensing](#licensing)
- [Architecture](#architecture)
  - [High-Level Architecture](#high-level-architecture)
  - [Multi-Threaded Event-Driven Model](#multi-threaded-event-driven-model)
  - [Shared-Nothing Across Nodes](#shared-nothing-across-nodes)
  - [Client-Side Distribution](#client-side-distribution)
  - [No Replication by Design](#no-replication-by-design)
- [Memory Management](#memory-management)
  - [Slab Allocator Overview](#slab-allocator-overview)
  - [Slab Classes and Chunk Sizes](#slab-classes-and-chunk-sizes)
  - [Page Allocation](#page-allocation)
  - [Growth Factor](#growth-factor)
  - [Memory Fragmentation](#memory-fragmentation)
  - [Slab Reassignment and Automove](#slab-reassignment-and-automove)
  - [Stats Slabs](#stats-slabs)
- [Consistent Hashing](#consistent-hashing)
  - [Key Distribution Problem](#key-distribution-problem)
  - [Ketama Algorithm](#ketama-algorithm)
  - [Virtual Nodes](#virtual-nodes)
  - [Node Addition and Removal](#node-addition-and-removal)
- [Protocol](#protocol)
  - [Text Protocol](#text-protocol)
  - [Binary Protocol](#binary-protocol)
  - [Meta Commands Protocol](#meta-commands-protocol)
  - [Common Commands](#common-commands)
  - [Response Codes](#response-codes)
- [Data Model](#data-model)
  - [Key-Value Store](#key-value-store)
  - [Key Constraints](#key-constraints)
  - [Value Constraints](#value-constraints)
  - [Flags Field](#flags-field)
  - [CAS Tokens and Optimistic Locking](#cas-tokens-and-optimistic-locking)
- [Eviction](#eviction)
  - [LRU Per Slab Class](#lru-per-slab-class)
  - [Segmented LRU](#segmented-lru)
  - [Tail Repair](#tail-repair)
  - [Controlling Eviction Behavior](#controlling-eviction-behavior)
- [Multi-Threading](#multi-threading)
  - [Worker Threads](#worker-threads)
  - [Connection Distribution](#connection-distribution)
  - [Lock Contention Areas](#lock-contention-areas)
  - [Scaling Across CPU Cores](#scaling-across-cpu-cores)
  - [Thread Configuration](#thread-configuration)
- [Memcached vs Redis vs Valkey](#memcached-vs-redis-vs-valkey)
  - [Feature Comparison](#feature-comparison)
  - [Performance Characteristics](#performance-characteristics)
  - [When to Choose Each](#when-to-choose-each)
- [Client Libraries](#client-libraries)
  - [Popular Clients](#popular-clients)
  - [Choosing a Client](#choosing-a-client)
- [Configuration and Tuning](#configuration-and-tuning)
  - [Essential Startup Flags](#essential-startup-flags)
  - [Advanced Options](#advanced-options)
  - [Example Configurations](#example-configurations)
- [Monitoring](#monitoring)
  - [Stats Commands](#stats-commands)
  - [Key Metrics](#key-metrics)
  - [Monitoring Dashboard Example](#monitoring-dashboard-example)
- [Common Patterns](#common-patterns)
  - [Session Caching](#session-caching)
  - [Database Query Result Caching](#database-query-result-caching)
  - [Page and Fragment Caching](#page-and-fragment-caching)
  - [Object Serialization Caching](#object-serialization-caching)
  - [Multiget Optimization](#multiget-optimization)
- [When to Choose Memcached](#when-to-choose-memcached)
- [Limitations](#limitations)

---

## Overview

### What is Memcached

Memcached is a free, open-source, high-performance, distributed **in-memory key-value store**
designed for speeding up dynamic web applications by reducing database load. It stores data
as simple key-value pairs entirely in RAM, providing sub-millisecond response times for
cached data. Memcached is used by some of the largest websites in the world — Facebook,
Wikipedia, YouTube, Twitter, and many others have relied on it to serve billions of requests
per day.

Unlike more feature-rich caching systems, Memcached deliberately keeps its feature set minimal.
It does one thing — caching opaque blobs of data keyed by a string — and it does it
exceptionally well. There are no complex data structures, no persistence, no replication,
and no built-in clustering. This simplicity is its greatest strength: every design decision
optimises for raw speed and predictable memory usage.

```
┌───────────────────────────────────────────────────────────────────┐
│                        Memcached at a Glance                      │
├───────────────────────────────────────────────────────────────────┤
│  Type .............. Distributed in-memory key-value cache        │
│  Written in ........ C                                            │
│  Protocol .......... Text, Binary, Meta Commands                  │
│  Data model ........ Key → Value (opaque bytes)                   │
│  Threading ......... Multi-threaded (libevent)                    │
│  Clustering ........ Client-side (no built-in)                    │
│  Persistence ....... None (purely ephemeral)                      │
│  Max key size ...... 250 bytes                                    │
│  Max value size .... 1 MB (default, configurable)                 │
│  License ........... Revised BSD                                  │
│  Default port ...... 11211                                        │
└───────────────────────────────────────────────────────────────────┘
```

### History

Memcached was created by **Brad Fitzpatrick** in **2003** to accelerate **LiveJournal.com**,
a popular social networking and blogging platform that was struggling under heavy database
load. Rather than scaling the database vertically (more powerful hardware) or horizontally
(read replicas), Fitzpatrick built a general-purpose caching daemon that sat between the
application and the database.

Key milestones in Memcached history:

```
2003 ─── Brad Fitzpatrick writes Memcached in Perl for LiveJournal
  │
2003 ─── Anatoly Vorobey rewrites Memcached in C for performance
  │
2005 ─── Adoption by major web companies begins (Wikipedia, Flickr)
  │
2006 ─── Facebook begins using Memcached at scale
  │
2008 ─── Binary protocol introduced alongside the text protocol
  │
2009 ─── Facebook open-sources their Memcached enhancements
  │
2014 ─── Slab rebalancing and automove features added
  │
2017 ─── Memcached 1.5: Segmented LRU, modern defaults
  │
2018 ─── Meta commands protocol introduced (extstore work)
  │
2020 ─── TLS support added
  │
2021 ─── Proxy mode and extstore (external SSD storage) mature
  │
2023 ─── 20th anniversary; continued active development
  │
2024 ─── Memcached 1.6.x with proxy improvements, io_uring support
```

The original Perl implementation was quickly rewritten in C by Anatoly Vorobey with
contributions from Brad Fitzpatrick. This C implementation became the canonical version
and remains the actively maintained codebase today. The project is maintained by
**Dormando** (dormando) and a community of contributors.

### Design Philosophy

Memcached's design philosophy can be summarised in a few principles:

1. **Simplicity over features** — Do one thing well. Every feature adds complexity,
   latency, and potential bugs. Memcached resists feature creep.

2. **Speed is paramount** — Every design decision is evaluated against its impact on
   latency and throughput. Sub-millisecond response times are the norm.

3. **Memory efficiency** — The slab allocator provides predictable memory usage with
   minimal fragmentation. Memory is the most expensive resource.

4. **The server is dumb, the client is smart** — Memcached servers know nothing about
   each other. All distribution, failover, and routing logic lives in the client.

5. **Ephemeral by design** — Data can disappear at any time (eviction, restart, crash).
   Applications must be designed to handle cache misses gracefully.

6. **Half the solution** — Memcached deliberately solves only half the problem (fast
   storage/retrieval). The other half (what to cache, when to invalidate) is left
   to the application developer.

### Licensing

Memcached is released under the **Revised BSD License** (3-clause BSD), one of the most
permissive open-source licenses available. This allows:

- Free use in commercial and proprietary software
- Modification and redistribution with minimal restrictions
- No copyleft requirements (unlike GPL)

The permissive license has been a significant factor in Memcached's widespread adoption
across both open-source projects and commercial products.

---

## Architecture

### High-Level Architecture

Memcached runs as a standalone daemon process that listens on a TCP (and optionally UDP)
port. Clients connect directly to Memcached instances and issue commands over the wire.
In a typical deployment, multiple Memcached instances run on separate machines, and the
client library distributes keys across them using consistent hashing.

```
                         ┌───────────────────────────────────┐
                         │         Application Tier           │
                         │  ┌─────┐  ┌─────┐  ┌─────┐       │
                         │  │App 1│  │App 2│  │App 3│       │
                         │  └──┬──┘  └──┬──┘  └──┬──┘       │
                         │     │        │        │           │
                         │  ┌──▼────────▼────────▼──┐       │
                         │  │   Memcached Client     │       │
                         │  │  (consistent hashing)  │       │
                         │  └──┬────────┬────────┬──┘       │
                         └─────│────────│────────│───────────┘
                               │        │        │
                ┌──────────────┼────────┼────────┼──────────────┐
                │              │        │        │   Network     │
                │     ┌────────▼──┐ ┌───▼─────┐ ┌▼────────┐    │
                │     │Memcached 1│ │Memcached│ │Memcached│    │
                │     │  node-1   │ │ node-2  │ │ node-3  │    │
                │     │ (shard A) │ │(shard B)│ │(shard C)│    │
                │     └───────────┘ └─────────┘ └─────────┘    │
                │                                               │
                │   Each node is independent — no inter-node    │
                │   communication whatsoever                     │
                └───────────────────────────────────────────────┘
```

Key points about this architecture:

- **No inter-node communication** — Nodes never talk to each other. Each node is an
  isolated key-value store that knows nothing about the rest of the cluster.
- **Client-side routing** — The client library decides which node owns a given key.
  All clients must use the same algorithm and server list.
- **Horizontal scaling** — Adding a node increases total cache capacity linearly. Each
  node contributes its memory to the overall pool.

### Multi-Threaded Event-Driven Model

Memcached uses **libevent** for its event-driven I/O model and runs **multiple worker
threads** to handle connections and requests in parallel. This is a key differentiator
from Redis, which uses a single-threaded event loop for command processing.

```
┌─────────────────────────────────────────────────────┐
│                  Memcached Process                    │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │           Main / Listener Thread              │   │
│  │  ┌──────────┐                                 │   │
│  │  │ libevent │  Accepts new TCP connections     │   │
│  │  │  loop    │  Distributes to worker threads   │   │
│  │  └──────────┘                                 │   │
│  └──────────┬───────────┬───────────┬────────────┘  │
│             │           │           │                │
│     ┌───────▼───┐ ┌─────▼─────┐ ┌──▼────────┐      │
│     │ Worker    │ │ Worker    │ │ Worker    │      │
│     │ Thread 1  │ │ Thread 2  │ │ Thread N  │      │
│     │           │ │           │ │           │      │
│     │ libevent  │ │ libevent  │ │ libevent  │      │
│     │ loop      │ │ loop      │ │ loop      │      │
│     │           │ │           │ │           │      │
│     │ Reads/    │ │ Reads/    │ │ Reads/    │      │
│     │ writes    │ │ writes    │ │ writes    │      │
│     │ on its    │ │ on its    │ │ on its    │      │
│     │ assigned  │ │ assigned  │ │ assigned  │      │
│     │ conns     │ │ conns     │ │ conns     │      │
│     └───────────┘ └───────────┘ └───────────┘      │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │          Shared Hash Table + Slab Memory       │   │
│  │  (Protected by fine-grained per-bucket locks)  │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

The main thread accepts incoming connections and distributes them round-robin across
worker threads. Each worker thread runs its own libevent loop, handling I/O for its
assigned connections independently. All threads share the same hash table and slab
memory, protected by fine-grained locking.

### Shared-Nothing Across Nodes

Each Memcached instance is completely independent. There is no:

- **Gossip protocol** — Nodes do not discover each other
- **Data replication** — Data exists on exactly one node
- **Failover mechanism** — If a node dies, its data is gone
- **Consensus algorithm** — No leader election, no quorum
- **Configuration synchronisation** — Each node runs in isolation

This shared-nothing architecture keeps the server implementation extremely simple and
eliminates entire classes of distributed systems problems (split-brain, replication lag,
consensus overhead).

### Client-Side Distribution

Since Memcached servers are unaware of each other, all intelligence lives in the client:

```
Client Library Responsibilities:
┌──────────────────────────────────────────────────┐
│                                                   │
│  1. Maintain a list of available servers          │
│  2. Hash each key to determine the target server  │
│  3. Establish and pool TCP connections             │
│  4. Serialize/deserialize values                   │
│  5. Handle server failures (remove from pool)      │
│  6. Optionally compress large values               │
│  7. Implement retry logic and timeouts             │
│                                                   │
└──────────────────────────────────────────────────┘
```

This means all application servers must use the **same hashing algorithm** and the **same
server list** in the **same order**. A mismatch causes keys to be routed to different nodes
by different clients, leading to cache misses and inconsistencies.

### No Replication by Design

Memcached intentionally does not replicate data across nodes. This is a deliberate design
choice, not a missing feature:

- **Simplicity** — Replication adds complexity, failure modes, and consistency problems.
- **Memory efficiency** — Every byte of RAM stores unique data, not copies.
- **Use case fit** — Cached data is inherently reconstructable from the source of truth.
  If a node dies, the application fetches from the database and repopulates the cache.

For use cases that require high availability of cached data, consider:

- Running Memcached on reliable infrastructure
- Using a cache-aside pattern with fast fallback to the database
- Choosing Redis/Valkey with replication if cache availability is critical

---

## Memory Management

### Slab Allocator Overview

Memcached uses a **slab allocator** to manage memory efficiently and avoid the fragmentation
problems that come with general-purpose allocators like `malloc`/`free`. Rather than
allocating memory for each item individually, Memcached pre-allocates memory in large
blocks and divides them into fixed-size slots.

The key concepts are:

- **Slab class** — A group of memory pages that store items of a similar size range
- **Page** — A 1 MB block of memory assigned to a slab class
- **Chunk** — A fixed-size slot within a page, sized for the slab class

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Memcached Slab Allocator                         │
│                                                                      │
│  When memcached starts with -m 1024 (1GB), it reserves memory and   │
│  allocates pages to slab classes on demand as items are stored.      │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Slab Class 1 (96 bytes per chunk)                           │    │
│  │  ┌──────────────────── Page 1 (1 MB) ──────────────────┐    │    │
│  │  │ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐    ┌─────┐│    │    │
│  │  │ │ 96B │ │ 96B │ │ 96B │ │ 96B │ │ 96B │....│ 96B ││    │    │
│  │  │ │chunk│ │chunk│ │chunk│ │chunk│ │chunk│    │chunk││    │    │
│  │  │ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘    └─────┘│    │    │
│  │  └────────────────────────────────────────────────────────┘  │    │
│  │                 10,922 chunks per page                        │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Slab Class 2 (120 bytes per chunk)                          │    │
│  │  ┌──────────────────── Page 1 (1 MB) ──────────────────┐    │    │
│  │  │ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐     ┌──────┐   │    │    │
│  │  │ │ 120B │ │ 120B │ │ 120B │ │ 120B │ ... │ 120B │   │    │    │
│  │  │ │chunk │ │chunk │ │chunk │ │chunk │     │chunk │   │    │    │
│  │  │ └──────┘ └──────┘ └──────┘ └──────┘     └──────┘   │    │    │
│  │  └────────────────────────────────────────────────────────┘  │    │
│  │                  8,738 chunks per page                        │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Slab Class 3 (152 bytes per chunk)                          │    │
│  │  ...                                                         │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│               ... up to Slab Class N (1 MB chunk) ...                │
│                                                                      │
│  Each slab class handles items whose total size (key + value +       │
│  overhead) fits within that chunk size.                               │
└──────────────────────────────────────────────────────────────────────┘
```

### Slab Classes and Chunk Sizes

Slab classes form a geometric series of chunk sizes, starting from a minimum size and
increasing by a **growth factor** (default 1.25). An item is stored in the smallest slab
class whose chunk size can accommodate it.

```
Example slab classes with growth factor 1.25:

  Slab Class   Chunk Size   Chunks/Page   Fits Items Up To
  ──────────   ──────────   ───────────   ─────────────────
       1          96 B        10,922        96 B - overhead
       2         120 B         8,738       120 B - overhead
       3         152 B         6,898       152 B - overhead
       4         192 B         5,461       192 B - overhead
       5         240 B         4,369       240 B - overhead
       6         304 B         3,449       304 B - overhead
       7         384 B         2,730       384 B - overhead
       8         480 B         2,184       480 B - overhead
       9         600 B         1,747       600 B - overhead
      10         752 B         1,394       752 B - overhead
      ...        ...           ...         ...
      38        512 KB            2        512 KB - overhead
      39          1 MB            1          1 MB - overhead

  overhead = 48 bytes (key + metadata + CAS + pointers)
```

Each item stored in Memcached occupies exactly one chunk. If an item is 100 bytes total
(key + value + overhead), it goes into slab class 2 (120-byte chunks), wasting 20 bytes.
This internal fragmentation is the trade-off for avoiding external fragmentation.

### Page Allocation

Memory is allocated in **1 MB pages**. When a slab class needs more room:

1. Memcached checks if there is a free page in the global page pool
2. If yes, the page is assigned to the requesting slab class and divided into chunks
3. If no free pages remain, Memcached either evicts items from the same slab class
   or (with slab reassignment enabled) steals a page from another slab class

```
Page Lifecycle:

  ┌────────────┐     Assign to      ┌──────────────────────┐
  │ Free Page  │ ──────────────────► │ Slab Class N         │
  │  Pool      │     slab class     │ (divided into chunks) │
  └────────────┘                    └──────────────────────┘
       ▲                                      │
       │                                      │
       │    slab_reassign                     │ All chunks
       │    (page moves between               │ occupied
       │     slab classes)                    │
       │                                      ▼
  ┌────────────┐                    ┌──────────────────────┐
  │ Reclaimed  │ ◄───────────────── │ LRU eviction within  │
  │  (rare)    │    restart only    │ slab class           │
  └────────────┘                    └──────────────────────┘
```

Once a page is assigned to a slab class, it **stays with that slab class** for the
lifetime of the process — unless **slab reassignment** (`-o slab_reassign`) is enabled.
This is why monitoring slab utilisation is important: memory can become trapped in slab
classes that no longer need it.

### Growth Factor

The growth factor (`-f` flag, default 1.25) controls how quickly chunk sizes increase
between slab classes. It directly impacts memory efficiency:

```
Growth Factor Trade-offs:

  Lower factor (e.g. 1.05):
  ┌──────────────────────────────────────────────┐
  │  + More slab classes, finer granularity       │
  │  + Less internal fragmentation (wasted space) │
  │  - More slab classes to manage                │
  │  - More memory spent on bookkeeping           │
  └──────────────────────────────────────────────┘

  Higher factor (e.g. 2.00):
  ┌──────────────────────────────────────────────┐
  │  + Fewer slab classes, simpler management     │
  │  - More internal fragmentation                │
  │  - Up to 50% wasted space per item            │
  └──────────────────────────────────────────────┘

  Default (1.25): Good balance for most workloads
```

### Memory Fragmentation

Memcached's slab allocator trades **internal fragmentation** for freedom from **external
fragmentation**:

- **Internal fragmentation** — Space wasted within a chunk because the item is smaller
  than the chunk size. An 80-byte item in a 96-byte chunk wastes 16 bytes (17%).
- **External fragmentation** — Eliminated. Unlike `malloc`, the slab allocator never
  creates unusable gaps between allocations because all chunks are fixed-size.

The worst case for internal fragmentation occurs when items are just slightly larger than
the previous slab class's chunk size, pushing them into the next class:

```
Item size: 97 bytes → stored in slab class 2 (120-byte chunks)
Wasted: 23 bytes per item (19% waste)

Item size: 96 bytes → stored in slab class 1 (96-byte chunks)
Wasted: 0 bytes per item (0% waste)
```

### Slab Reassignment and Automove

Starting with Memcached 1.4.11, **slab reassignment** allows pages to be moved between
slab classes at runtime to address workload changes:

- **`-o slab_reassign`** — Enables the background thread that can reassign pages
- **`-o slab_automove=1`** — Automatic, conservative page rebalancing (default when
  slab_reassign is enabled)
- **`-o slab_automove=2`** — Aggressive mode: immediately moves pages when a slab
  class has evictions and another has free chunks

```
# Manual slab reassignment via the slabs command
slabs reassign <source_class> <dest_class>

# Example: move a page from class 5 to class 10
slabs reassign 5 10
```

### Stats Slabs

The `stats slabs` command provides detailed information about each slab class:

```
STAT 1:chunk_size 96
STAT 1:chunks_per_page 10922
STAT 1:total_pages 1
STAT 1:total_chunks 10922
STAT 1:used_chunks 8451
STAT 1:free_chunks 2471
STAT 1:free_chunks_end 0
STAT 1:get_hits 1523892
STAT 1:cmd_set 9102
STAT 1:delete_hits 44
STAT 1:incr_hits 0
STAT 1:decr_hits 0
STAT 1:cas_hits 0
STAT 1:cas_badval 0
STAT 1:touch_hits 0
```

Key fields to monitor:

- **used_chunks / total_chunks** — Utilisation percentage per slab class
- **free_chunks** — Chunks available without eviction
- **evicted** — How many items were evicted from this class
- **evicted_unfetched** — Items evicted that were never read (wasted cache space)

---

## Consistent Hashing

### Key Distribution Problem

When distributing keys across N Memcached nodes, a naive approach uses modular hashing:

```
server_index = hash(key) % N
```

This works until you add or remove a node. Changing N causes **almost every key** to
map to a different server, resulting in a massive cache miss storm:

```
Naive Modular Hashing — Node Removal Impact:

  3 nodes:  hash("user:42") % 3 = 1  →  Server B
  2 nodes:  hash("user:42") % 2 = 0  →  Server A  ← MISS!

  Approximately (N-1)/N = 66% of keys remap when going from 3→2 nodes
  For 10→9 nodes: ~90% of keys remap!
```

### Ketama Algorithm

The **Ketama** consistent hashing algorithm (developed at Last.fm) solves this problem.
It maps both servers and keys onto a circular hash space (a "ring") and assigns each key
to the nearest server clockwise on the ring:

```
            Consistent Hash Ring (Ketama)

                     0 / 2^32
                       │
                  ┌────┴────┐
              S3 ●          ● S1        S = Server position
             /                  \       K = Key position
            /                    \
           /                      \
      K3  ◆                        ◆ K1
         │                          │
         │                          │
         │      Hash Ring           │
         │    (0 to 2^32)           │
         │                          │
      S2 ●                          │
          \                        /
           \                      /
            \                    /
              ◆ K2          ● S1'
              └────┬────────┘
                   │
               2^32 / 2

  K1 → S1  (nearest server clockwise)
  K2 → S1' (nearest server clockwise)
  K3 → S3  (nearest server clockwise)
```

When a node is removed, only the keys that mapped to that node are redistributed to the
next server on the ring. All other keys remain on their current servers.

### Virtual Nodes

A problem with placing only one point per server on the ring is uneven distribution.
One server might own a huge arc while another owns a tiny sliver. **Virtual nodes**
(vnodes) solve this by placing multiple points per server on the ring:

```
Virtual Nodes — Improved Distribution:

  Without vnodes (1 point per server, uneven):
  ┌──────────────────────────────────────────────┐
  │    S1        S2                   S3          │
  │     ●────────●────────────────────●           │
  │   [ 20% ]   [        55%        ][  25%  ]   │
  └──────────────────────────────────────────────┘

  With vnodes (150 points per server, balanced):
  ┌──────────────────────────────────────────────┐
  │ S1 S3 S2 S1 S3 S2 S1 S2 S3 S1 S3 S2 S1 S3  │
  │  ●  ●  ●  ●  ●  ●  ●  ●  ●  ●  ●  ●  ●  ● │
  │ [≈33% S1] [≈33% S2] [≈33% S3]               │
  └──────────────────────────────────────────────┘

  Ketama default: 100-200 virtual nodes per server
  Each point = hash(server_address + "-" + vnode_index)
```

### Node Addition and Removal

With consistent hashing and virtual nodes, adding or removing a node redistributes only
**1/N** of the keys on average (where N is the total number of nodes):

```
Adding a Node (S4):

  Before: 3 nodes, each owns ~33% of the ring
  After:  4 nodes, each owns ~25% of the ring

  Only ~25% of keys (1/4) need to move — those that now fall
  in S4's portion of the ring. The other 75% stay put.

  ┌────────────────────────────────────────────────┐
  │            Ring before adding S4                │
  │   ●S1 ─────── ●S2 ─────── ●S3 ───────         │
  │   [  33%   ]  [  33%   ]  [  33%   ]          │
  │                                                │
  │            Ring after adding S4                 │
  │   ●S1 ──── ●S4 ──── ●S2 ──── ●S3 ────         │
  │   [ 25%] [ 25%]  [ 25%]   [ 25%]             │
  │            ^^^^                                │
  │        Only these keys moved from S2 to S4     │
  └────────────────────────────────────────────────┘

Removing a Node (S2 fails):

  Before: 3 nodes, each owns ~33%
  After:  2 nodes — S2's keys go to the next node clockwise

  Only ~33% of keys (1/3) are affected. Applications see
  cache misses for those keys and repopulate from the database.
```

---

## Protocol

### Text Protocol

The original Memcached protocol is a simple line-oriented text protocol. Commands are
sent as ASCII strings terminated by `\r\n`. It is human-readable and easy to debug
with tools like `telnet` or `nc`:

```bash
# Connect and interact with Memcached via telnet
$ telnet localhost 11211
set mykey 0 300 5\r\n
hello\r\n
STORED

get mykey\r\n
VALUE mykey 0 5
hello
END

delete mykey\r\n
DELETED
```

Text protocol command format:

```
Storage:   <command> <key> <flags> <exptime> <bytes> [noreply]\r\n
           <data block>\r\n

Retrieval: get <key>*\r\n
           gets <key>*\r\n   (returns CAS token)

Deletion:  delete <key> [noreply]\r\n
```

### Binary Protocol

Introduced in Memcached 1.3, the binary protocol provides:

- **Lower parsing overhead** — Fixed-size headers instead of text parsing
- **Reduced bandwidth** — More compact wire format
- **Better error handling** — Structured error codes
- **SASL authentication** — Built-in auth support

```
Binary Protocol Header (24 bytes):

  Byte/  0       |       1       |       2       |       3
     /  |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
    +---------------+---------------+---------------+---------------+
   0| Magic         | Opcode        | Key Length                    |
    +---------------+---------------+---------------+---------------+
   4| Extras length | Data type     | Status / vbucket              |
    +---------------+---------------+---------------+---------------+
   8| Total body length                                             |
    +---------------+---------------+---------------+---------------+
  12| Opaque                                                        |
    +---------------+---------------+---------------+---------------+
  16| CAS                                                           |
    |                                                               |
    +---------------+---------------+---------------+---------------+

  Magic: 0x80 (request) | 0x81 (response)
```

### Meta Commands Protocol

Introduced in Memcached 1.6, the meta commands protocol combines the human readability
of the text protocol with the flexibility and performance of the binary protocol:

```
Meta commands use single-letter flags for maximum efficiency:

  mg <key> <flags>\r\n     # meta get
  ms <key> <datalen> <flags>\r\n   # meta set
  md <key> <flags>\r\n     # meta delete
  ma <key> <flags>\r\n     # meta arithmetic
  mn\r\n                   # meta noop (pipeline flushing)
  me <key> <datalen> <flags>\r\n   # meta debug

Common flags:
  v — return value
  t — return TTL remaining
  c — return CAS token
  f — return client flags
  l — return last access time
  h — return hit-before-expiry flag
  k — return key
  O — opaque token (for request/response matching)
  q — quiet mode (noreply for non-error responses)
  T — set TTL
  N — vivify on miss (auto-create with TTL)
  R — recache: if item is within R seconds of expiry, win a token
```

### Common Commands

```
┌──────────────┬──────────────────────────────────────────────────────┐
│ Command      │ Description                                         │
├──────────────┼──────────────────────────────────────────────────────┤
│ get          │ Retrieve one or more items by key                   │
│ gets         │ Retrieve items with CAS tokens                      │
│ set          │ Store item (overwrite if exists)                    │
│ add          │ Store item only if key does NOT exist               │
│ replace      │ Store item only if key DOES exist                   │
│ append       │ Append data to existing item's value                │
│ prepend      │ Prepend data to existing item's value               │
│ cas          │ Compare-and-swap: set only if CAS token matches     │
│ delete       │ Remove an item by key                               │
│ incr         │ Increment a numeric value                           │
│ decr         │ Decrement a numeric value                           │
│ touch        │ Update an item's expiration time                    │
│ gat          │ Get and touch (retrieve + update expiry)            │
│ flush_all    │ Invalidate all items (optionally with delay)        │
│ stats        │ Return server statistics                            │
│ version      │ Return server version string                        │
│ quit         │ Close the connection                                │
└──────────────┴──────────────────────────────────────────────────────┘
```

### Response Codes

```
Storage Responses:
  STORED        — Item successfully stored
  NOT_STORED    — Item not stored (add on existing key, replace on missing key)
  EXISTS        — CAS conflict: item has been modified since last gets
  NOT_FOUND     — CAS on a key that does not exist

Retrieval Responses:
  VALUE <key> <flags> <bytes> [<cas>]\r\n<data>\r\nEND
  END           — No more items (or key not found)

Deletion Responses:
  DELETED       — Item successfully deleted
  NOT_FOUND     — Key does not exist

Arithmetic Responses:
  <value>\r\n   — New value after incr/decr
  NOT_FOUND     — Key does not exist

Error Responses:
  ERROR                  — Unknown command
  CLIENT_ERROR <msg>     — Client sent invalid input
  SERVER_ERROR <msg>     — Server-side error
```

---

## Data Model

### Key-Value Store

Memcached is a pure key-value store. Every item consists of:

```
┌─────────────────────────────────────────────────────────────┐
│                    Memcached Item Layout                      │
│                                                              │
│  ┌───────────┬────────┬─────────┬──────┬──────┬───────────┐ │
│  │   Key     │ Flags  │  CAS    │ TTL  │ Size │   Value   │ │
│  │ (string)  │(32-bit)│(64-bit) │      │      │  (bytes)  │ │
│  └───────────┴────────┴─────────┴──────┴──────┴───────────┘ │
│                                                              │
│  The value is an opaque byte array — Memcached does not      │
│  interpret, parse, or validate the contents in any way.      │
└─────────────────────────────────────────────────────────────┘
```

Memcached has no concept of data types, collections, or nested structures. The server
treats both keys and values as opaque byte sequences (with the exception of `incr`/`decr`,
which interpret the value as a decimal integer string).

### Key Constraints

- **Maximum length**: 250 bytes
- **Character restrictions**: No whitespace or control characters (they delimit the protocol)
- **Encoding**: Any byte sequence (typically UTF-8 strings)
- **Best practices**: Use structured key names like `user:42:profile` or `product:sku:1234`

```
Good key patterns:
  user:42:profile             — Entity type + ID + field
  session:abc123def456        — Session prefix + token
  query:sha256(sql)           — Cache key from query hash
  v2:user:42:profile          — Versioned keys for cache busting

Bad key patterns:
  my key with spaces          — Spaces break the text protocol
  a                           — Too generic, collision-prone
  <250+ byte string>          — Exceeds key length limit
```

### Value Constraints

- **Maximum size**: 1 MB by default
- **Configurable**: Use `-I` flag to adjust (e.g., `-I 2m` for 2 MB, max 128 MB)
- **Content**: Any byte sequence — text, JSON, serialised objects, compressed data
- **No server-side operations**: Memcached cannot query, filter, or transform values

```bash
# Start Memcached with 2 MB max item size
memcached -I 2m -m 1024 -p 11211

# Start with 10 MB max item size (use carefully — affects slab allocation)
memcached -I 10m -m 4096 -p 11211
```

Increasing the max item size creates a slab class with large chunks, which can waste
memory if most items are small. Use this sparingly and monitor slab utilisation.

### Flags Field

Each item has a **32-bit flags field** that is stored and returned opaque — Memcached
does not interpret it. Client libraries typically use flags to store metadata:

```
Common flag conventions (library-dependent):

  Bit 0    — Serialised with pickle/marshal
  Bit 1    — Compressed with zlib/gzip
  Bit 2    — Serialised as JSON
  Bit 3    — Serialised with MessagePack
  Bits 4-7 — Reserved for library use

  Example: flags = 3 → compressed + serialised

  The application / client library decides the encoding.
  Memcached just stores and returns the 32-bit integer.
```

### CAS Tokens and Optimistic Locking

**CAS (Check And Set)** provides optimistic locking to prevent lost updates when multiple
clients modify the same key concurrently:

```
CAS Workflow:

  Client A                  Memcached                  Client B
  ────────                  ─────────                  ────────
     │                          │                          │
     │  gets mykey              │                          │
     │─────────────────────────►│                          │
     │  VALUE mykey 0 5 12345   │                          │
     │◄─────────────────────────│                          │
     │                          │  gets mykey              │
     │                          │◄─────────────────────────│
     │                          │  VALUE mykey 0 5 12345   │
     │                          │─────────────────────────►│
     │                          │                          │
     │  cas mykey 0 300 6 12345 │                          │
     │  "hello!"                │                          │
     │─────────────────────────►│                          │
     │  STORED                  │                          │
     │◄─────────────────────────│  cas mykey 0 300 5 12345 │
     │                          │  "world"                 │
     │                          │◄─────────────────────────│
     │                          │  EXISTS  ← CAS mismatch! │
     │                          │─────────────────────────►│
     │                          │                          │
     │  Client A's write wins.  │  Client B must retry     │
     │  CAS token updated.      │  with fresh gets.        │
```

The CAS token is a 64-bit unique identifier that changes every time an item is modified.
By using `gets` (get with CAS) followed by `cas` (conditional set), clients can implement
safe read-modify-write cycles without locks.

---

## Eviction

### LRU Per Slab Class

Memcached maintains a **separate LRU (Least Recently Used) list per slab class**. When a
slab class runs out of free chunks and no free pages remain, the least recently used item
in that slab class is evicted to make room:

```
Eviction Flow:

  New item (150 bytes) arrives
       │
       ▼
  Needs slab class 3 (152-byte chunks)
       │
       ▼
  Any free chunks in class 3?  ──── Yes ──► Store item
       │
       No
       │
       ▼
  Any free pages in global pool?  ── Yes ──► Allocate page to class 3
       │                                      Divide into chunks
       No                                     Store item
       │
       ▼
  Evict LRU item from class 3's tail
       │
       ▼
  Store new item in freed chunk
```

This per-slab-class eviction means that a flood of large items will only evict other large
items, not small items in a different slab class. However, it also means a mostly-empty
slab class cannot donate memory to a full one (without slab reassignment).

### Segmented LRU

Starting with **Memcached 1.5** (2017), the LRU system was upgraded to a **segmented LRU**
with three queues per slab class, significantly improving cache hit rates:

```
Segmented LRU (per slab class):

  ┌─────────────────────────────────────────────────────────────┐
  │                                                              │
  │  HOT Queue                                                   │
  │  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐                             │
  │  │ A │→│ B │→│ C │→│ D │→│ E │→  (new items land here)     │
  │  └───┘ └───┘ └───┘ └───┘ └───┘                             │
  │    Items that have been set but not yet accessed again.      │
  │    When items age out, they move to WARM (if active)         │
  │    or COLD (if never accessed since insertion).              │
  │                                                              │
  │  WARM Queue                                                  │
  │  ┌───┐ ┌───┐ ┌───┐ ┌───┐                                   │
  │  │ F │→│ G │→│ H │→│ I │→  (items accessed at least twice) │
  │  └───┘ └───┘ └───┘ └───┘                                   │
  │    Items actively being used. They stay here as long as      │
  │    they are accessed. Aging out moves them to COLD.          │
  │                                                              │
  │  COLD Queue                                                  │
  │  ┌───┐ ┌───┐ ┌───┐                                         │
  │  │ J │→│ K │→│ L │→  (candidates for eviction)             │
  │  └───┘ └───┘ └───┘                                         │
  │    Items that have not been recently accessed.               │
  │    Eviction happens here: tail items are removed first.      │
  │                                                              │
  └─────────────────────────────────────────────────────────────┘

  Flow:  SET → HOT → WARM (if accessed) → COLD (if idle) → EVICTED
                  └──────→ COLD (if never accessed) → EVICTED
```

Benefits of segmented LRU:

- **Scan resistance** — A one-time bulk read (e.g., `multiget` of many keys) does not
  flush the entire cache. Items enter HOT and move to COLD without displacing WARM items.
- **Better hit rates** — Frequently accessed items stay in WARM and are protected from
  eviction, even during load spikes.
- **Reduced lock contention** — Operations on different queues use separate locks.

### Tail Repair

**Tail repair** is a background maintenance process that walks the tails of LRU queues
to fix inconsistencies and reclaim expired items:

- Finds items with expired TTLs and frees them without counting as evictions
- Repairs linked-list inconsistencies that can occur under heavy concurrent access
- Runs as a low-priority background crawler thread

```bash
# Enable the LRU crawler (enabled by default since 1.5)
-o lru_crawler

# Configure crawl frequency (milliseconds between runs)
-o lru_crawler_sleep=100

# Trigger a manual crawl
lru_crawler crawl all
```

### Controlling Eviction Behavior

```
┌──────────────────────────────────────────────────────────────────┐
│ Option                    │ Effect                                │
├───────────────────────────┼──────────────────────────────────────┤
│ -M                        │ Return errors instead of evicting    │
│                           │ items when memory is full. Disables  │
│                           │ eviction entirely.                   │
├───────────────────────────┼──────────────────────────────────────┤
│ -o lru_maintainer         │ Enable the background LRU maintainer │
│                           │ thread (default since 1.5)           │
├───────────────────────────┼──────────────────────────────────────┤
│ -o lru_crawler            │ Enable background crawler to expire  │
│                           │ items proactively                    │
├───────────────────────────┼──────────────────────────────────────┤
│ -o slab_reassign          │ Allow pages to move between slab     │
│                           │ classes to rebalance memory           │
├───────────────────────────┼──────────────────────────────────────┤
│ -o slab_automove=1        │ Conservative automatic rebalancing   │
├───────────────────────────┼──────────────────────────────────────┤
│ -o slab_automove=2        │ Aggressive rebalancing (immediate)   │
├───────────────────────────┼──────────────────────────────────────┤
│ -o expirezero_does_not_   │ Items with TTL=0 never expire        │
│   evict                   │ (but can still be evicted by LRU)    │
└──────────────────────────────────────────────────────────────────┘
```

---

## Multi-Threading

### Worker Threads

Memcached's multi-threaded architecture allows it to utilise multiple CPU cores
simultaneously. The threading model consists of:

- **1 main thread** — Listens for new connections, accepts them, and dispatches
  to worker threads via a round-robin pipe notification mechanism
- **N worker threads** — Each runs its own libevent event loop, handling I/O for
  assigned connections, parsing commands, and executing operations
- **1 LRU maintainer thread** — Background thread that manages the segmented LRU
  queues and handles item migration between HOT, WARM, and COLD
- **1 LRU crawler thread** — Walks LRU tails to expire items and reclaim memory
- **1 slab rebalancer thread** (optional) — Moves pages between slab classes
- **1 hash table expander thread** — Grows the hash table when load factor increases

### Connection Distribution

When a new client connects, the main thread accepts the connection and assigns it to
a worker thread. The assignment is permanent — a connection stays with its worker
thread for its entire lifetime:

```
Connection Assignment:

  Client A ──────┐
  Client B ──────┤──► Main Thread ──┬──► Worker Thread 1 (clients A, D, G)
  Client C ──────┤    (accept +     ├──► Worker Thread 2 (clients B, E, H)
  Client D ──────┤     dispatch)    ├──► Worker Thread 3 (clients C, F, I)
  Client E ──────┤                  └──► Worker Thread 4 (clients ...)
  Client F ──────┤
  ...            ┘

  Assignment: round-robin via pipe notification
  Each worker handles its connections independently
  Connection-to-thread mapping is sticky (no rebalancing)
```

### Lock Contention Areas

While worker threads handle I/O independently, they share the global hash table and
slab allocator, requiring synchronisation:

```
Lock Contention Points:

  ┌──────────────────────────────────────────────────────┐
  │ Resource              │ Lock Type     │ Contention   │
  ├───────────────────────┼───────────────┼──────────────┤
  │ Hash table buckets    │ Per-bucket    │ Low          │
  │                       │ (fine-grained)│ (millions    │
  │                       │               │  of buckets) │
  ├───────────────────────┼───────────────┼──────────────┤
  │ Slab class freelist   │ Per-class     │ Low-Medium   │
  │                       │ mutex         │              │
  ├───────────────────────┼───────────────┼──────────────┤
  │ LRU lists             │ Per-class     │ Low-Medium   │
  │                       │ mutex         │              │
  ├───────────────────────┼───────────────┼──────────────┤
  │ Stats counters        │ Per-thread    │ None         │
  │                       │ (thread-local)│              │
  ├───────────────────────┼───────────────┼──────────────┤
  │ Connection counters   │ Atomic ops    │ Minimal      │
  └───────────────────────┴───────────────┴──────────────┘
```

The hash table uses fine-grained locking (one lock per hash bucket or group of buckets),
which means threads rarely contend on the same lock. Slab class and LRU locks are per-class,
so contention only occurs when multiple threads access items in the same slab class.

### Scaling Across CPU Cores

Memcached scales well with additional CPU cores up to a point. Practical guidance:

```
Threading Performance Characteristics:

  Threads    Throughput (relative)    Notes
  ───────    ────────────────────     ─────────────────────────
     1            1.0x                Single-threaded baseline
     2            ~1.8x               Near-linear scaling
     4            ~3.2x               Good scaling continues
     8            ~5.5x               Diminishing returns begin
    16            ~7.0x               Lock contention visible
    32            ~7.5x               Marginal gains at best

  Rule of thumb: set threads = number of CPU cores (up to ~8)
  Beyond 8 threads, gains are workload-dependent.
  Network I/O often becomes the bottleneck before CPU.
```

### Thread Configuration

```bash
# Set the number of worker threads (default: 4)
memcached -t 8

# Typical production configuration
memcached -t 4 -m 4096 -c 10000 -p 11211

# CPU-pinning for performance (Linux)
taskset -c 0-7 memcached -t 8 -m 8192 -p 11211

# Check current thread count
echo "stats settings" | nc localhost 11211 | grep threads
STAT threads 4
```

---

## Memcached vs Redis vs Valkey

### Feature Comparison

```
┌─────────────────────┬──────────────────┬──────────────────┬──────────────────┐
│ Feature             │ Memcached        │ Redis            │ Valkey           │
├─────────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Data types          │ Strings only     │ Strings, Lists,  │ Strings, Lists,  │
│                     │ (opaque bytes)   │ Sets, Sorted     │ Sets, Sorted     │
│                     │                  │ Sets, Hashes,    │ Sets, Hashes,    │
│                     │                  │ Streams, etc.    │ Streams, etc.    │
├─────────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Threading           │ Multi-threaded   │ Single-threaded  │ Multi-threaded   │
│                     │ (command exec)   │ (command exec)   │ (I/O + command)  │
│                     │                  │ Multi-threaded   │                  │
│                     │                  │ (I/O since 6.0)  │                  │
├─────────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Persistence         │ None             │ RDB + AOF        │ RDB + AOF        │
├─────────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Replication         │ None             │ Primary-Replica  │ Primary-Replica  │
├─────────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Clustering          │ Client-side only │ Redis Cluster    │ Valkey Cluster   │
│                     │                  │ (hash slots)     │ (hash slots)     │
├─────────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Pub/Sub             │ No               │ Yes              │ Yes              │
├─────────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Scripting           │ No               │ Lua + Functions  │ Lua + Functions  │
├─────────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Transactions        │ CAS only         │ MULTI/EXEC +     │ MULTI/EXEC +     │
│                     │                  │ WATCH            │ WATCH            │
├─────────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Max key size        │ 250 bytes        │ 512 MB           │ 512 MB           │
├─────────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Max value size      │ 1 MB (default)   │ 512 MB           │ 512 MB           │
├─────────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Memory overhead     │ ~56 bytes/key    │ ~70+ bytes/key   │ ~70+ bytes/key   │
│ per key             │                  │                  │                  │
├─────────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Memory allocator    │ Slab allocator   │ jemalloc         │ jemalloc         │
├─────────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Eviction            │ Segmented LRU    │ Approx LRU/LFU   │ Approx LRU/LFU   │
│                     │ (per slab class) │ (sampled)        │ (sampled)        │
├─────────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Protocol            │ Text / Binary /  │ RESP2 / RESP3    │ RESP2 / RESP3    │
│                     │ Meta commands    │                  │                  │
├─────────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ TLS support         │ Yes (since 2020) │ Yes              │ Yes              │
├─────────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ License             │ BSD              │ RSALv2 + SSPLv1  │ BSD (3-clause)   │
│                     │                  │ (since 7.4)      │                  │
└─────────────────────┴──────────────────┴──────────────────┴──────────────────┘
```

### Performance Characteristics

```
Benchmark Comparison (typical, varies by workload):

  ┌────────────────────────┬────────────┬────────────┬────────────┐
  │ Metric                 │ Memcached  │ Redis      │ Valkey     │
  ├────────────────────────┼────────────┼────────────┼────────────┤
  │ GET latency (p50)      │ ~100 μs    │ ~120 μs    │ ~110 μs    │
  │ SET latency (p50)      │ ~110 μs    │ ~130 μs    │ ~120 μs    │
  │ Throughput (single)    │ ~200K/s    │ ~150K/s    │ ~180K/s    │
  │ Throughput (multi-core)│ ~1M+/s     │ ~200K/s    │ ~800K+/s   │
  │ Memory efficiency      │ Higher     │ Lower      │ Lower      │
  │ (simple strings)       │            │            │            │
  └────────────────────────┴────────────┴────────────┴────────────┘

  Note: These are rough guidelines. Actual performance depends on
  hardware, network, item sizes, and access patterns. Always benchmark
  your specific workload.
```

### When to Choose Each

```
Choose Memcached when:
  ✓ You need a simple, ephemeral cache for string/blob data
  ✓ Maximum memory efficiency for storing simple key-value pairs
  ✓ Multi-threaded performance on multi-core machines matters
  ✓ You have a read-heavy workload with large fan-out (multiget)
  ✓ You do not need persistence, replication, or complex data types
  ✓ You prefer BSD-licensed software

Choose Redis when:
  ✓ You need rich data structures (sorted sets, streams, etc.)
  ✓ You need persistence (RDB/AOF) or replication
  ✓ You need Pub/Sub, Lua scripting, or transactions
  ✓ You want built-in clustering with automatic sharding
  ✓ You need features like rate limiting, leaderboards, or queues
  ✓ You accept the RSAL/SSPL licensing terms

Choose Valkey when:
  ✓ You need Redis-compatible features with BSD licensing
  ✓ You want multi-threaded command execution
  ✓ You need a community-driven, truly open-source alternative
  ✓ You require persistence, replication, and clustering
```

---

## Client Libraries

### Popular Clients

```
┌─────────────────────┬──────────┬────────────┬──────────────────────────────┐
│ Library             │ Language │ Protocol   │ Notes                        │
├─────────────────────┼──────────┼────────────┼──────────────────────────────┤
│ libmemcached        │ C/C++   │ Text +     │ Feature-rich C library;      │
│                     │          │ Binary     │ basis for many wrappers;     │
│                     │          │            │ consistent hashing built-in  │
├─────────────────────┼──────────┼────────────┼──────────────────────────────┤
│ pymemcache          │ Python  │ Text +     │ Pinterest-maintained; pure   │
│                     │          │ Meta       │ Python; HashClient for       │
│                     │          │            │ consistent hashing           │
├─────────────────────┼──────────┼────────────┼──────────────────────────────┤
│ python-memcached    │ Python  │ Text       │ Original Python client;      │
│                     │          │            │ pure Python; thread-safe     │
├─────────────────────┼──────────┼────────────┼──────────────────────────────┤
│ spymemcached        │ Java    │ Text +     │ Asynchronous NIO-based;      │
│                     │          │ Binary     │ widely used; Ketama built-in │
├─────────────────────┼──────────┼────────────┼──────────────────────────────┤
│ xmemcached          │ Java    │ Text +     │ High-performance NIO;        │
│                     │          │ Binary     │ connection pooling;          │
│                     │          │            │ consistent hashing           │
├─────────────────────┼──────────┼────────────┼──────────────────────────────┤
│ Dalli               │ Ruby    │ Binary +   │ Standard Ruby client;        │
│                     │          │ Meta       │ thread-safe; connection      │
│                     │          │            │ pooling; Rails integration   │
├─────────────────────┼──────────┼────────────┼──────────────────────────────┤
│ php-memcached       │ PHP     │ Text +     │ libmemcached wrapper;        │
│                     │          │ Binary     │ consistent hashing;          │
│                     │          │            │ session handler support      │
├─────────────────────┼──────────┼────────────┼──────────────────────────────┤
│ gomemcache          │ Go      │ Text       │ Brad Fitzpatrick's Go        │
│                     │          │            │ client; simple and fast      │
├─────────────────────┼──────────┼────────────┼──────────────────────────────┤
│ Enyim.Caching       │ .NET    │ Binary     │ Popular .NET client;         │
│                     │          │            │ Ketama hashing;              │
│                     │          │            │ connection pooling            │
├─────────────────────┼──────────┼────────────┼──────────────────────────────┤
│ ElastiCache Client  │ Java/   │ Binary     │ AWS auto-discovery;          │
│                     │ .NET    │            │ adds node discovery on top   │
│                     │          │            │ of standard clients          │
└─────────────────────┴──────────┴────────────┴──────────────────────────────┘
```

### Choosing a Client

When selecting a Memcached client library, consider:

- **Protocol support** — Meta commands protocol offers the best balance of readability
  and performance. Binary protocol is good for structured communication. Text protocol
  is simplest and most widely supported.
- **Consistent hashing** — The client must implement the same hashing algorithm (Ketama)
  as all other clients in your stack. Mismatches cause routing errors.
- **Connection pooling** — Essential for high-concurrency applications to avoid the
  overhead of establishing new TCP connections per request.
- **Serialisation** — Built-in support for JSON, pickle, MessagePack, or other formats
  reduces boilerplate code.
- **Compression** — Automatic compression for large values saves memory and bandwidth.
- **Failover handling** — How the client behaves when a node is unreachable (retry,
  skip, redistribute).

---

## Configuration and Tuning

### Essential Startup Flags

```
┌──────────┬──────────────┬──────────────────────────────────────────────────┐
│ Flag     │ Default      │ Description                                      │
├──────────┼──────────────┼──────────────────────────────────────────────────┤
│ -m       │ 64 MB        │ Maximum memory for item storage. Set to the      │
│          │              │ amount of RAM you want to dedicate to caching.   │
│          │              │ Example: -m 4096 (4 GB)                          │
├──────────┼──────────────┼──────────────────────────────────────────────────┤
│ -c       │ 1024         │ Maximum simultaneous connections. Increase for   │
│          │              │ high-concurrency workloads.                      │
│          │              │ Example: -c 10000                                │
├──────────┼──────────────┼──────────────────────────────────────────────────┤
│ -t       │ 4            │ Number of worker threads. Set to number of CPU   │
│          │              │ cores (up to ~8 for most workloads).             │
│          │              │ Example: -t 8                                    │
├──────────┼──────────────┼──────────────────────────────────────────────────┤
│ -I       │ 1 MB         │ Maximum item size. Increase only if needed.      │
│          │              │ Larger values create bigger slab classes.         │
│          │              │ Example: -I 2m (range: 1k to 128m)              │
├──────────┼──────────────┼──────────────────────────────────────────────────┤
│ -p       │ 11211        │ TCP port to listen on.                           │
│          │              │ Example: -p 11211                                │
├──────────┼──────────────┼──────────────────────────────────────────────────┤
│ -l       │ 0.0.0.0      │ Interface to listen on. Bind to specific IP for  │
│          │              │ security. Example: -l 10.0.0.5                   │
├──────────┼──────────────┼──────────────────────────────────────────────────┤
│ -d       │ (off)        │ Run as a daemon (background process).            │
├──────────┼──────────────┼──────────────────────────────────────────────────┤
│ -u       │ (required    │ Run as specified user (when started as root).    │
│          │  as root)    │ Example: -u memcached                            │
├──────────┼──────────────┼──────────────────────────────────────────────────┤
│ -f       │ 1.25         │ Slab growth factor. Lower = more slab classes,   │
│          │              │ less waste. Higher = fewer classes, more waste.   │
│          │              │ Example: -f 1.10                                 │
└──────────┴──────────────┴──────────────────────────────────────────────────┘
```

### Advanced Options

```
┌────────────────────────────────┬─────────────────────────────────────────┐
│ Flag                           │ Description                             │
├────────────────────────────────┼─────────────────────────────────────────┤
│ -o modern                      │ Enables recommended defaults for        │
│                                │ Memcached 1.5+: slab_reassign,         │
│                                │ slab_automove, lru_crawler,             │
│                                │ lru_maintainer, hash_algorithm=murmur3 │
├────────────────────────────────┼─────────────────────────────────────────┤
│ -o slab_reassign               │ Allow pages to move between slab        │
│                                │ classes at runtime                      │
├────────────────────────────────┼─────────────────────────────────────────┤
│ -o slab_automove=<0|1|2>       │ Automatic slab page rebalancing         │
│                                │ 0=off, 1=conservative, 2=aggressive     │
├────────────────────────────────┼─────────────────────────────────────────┤
│ -o lru_crawler                 │ Enable background LRU crawler to        │
│                                │ proactively expire items                │
├────────────────────────────────┼─────────────────────────────────────────┤
│ -o lru_maintainer              │ Background thread for LRU management    │
├────────────────────────────────┼─────────────────────────────────────────┤
│ -o hash_algorithm=murmur3      │ Use MurmurHash3 (better distribution    │
│                                │ than default Jenkins hash)              │
├────────────────────────────────┼─────────────────────────────────────────┤
│ -o no_hashexpand               │ Disable dynamic hash table expansion    │
│                                │ (use if memory is very constrained)     │
├────────────────────────────────┼─────────────────────────────────────────┤
│ -o expirezero_does_not_evict   │ Items with TTL=0 cannot be evicted      │
│                                │ (they become permanent until deleted)   │
├────────────────────────────────┼─────────────────────────────────────────┤
│ -o maxconns_fast               │ Immediately close new connections when  │
│                                │ max is reached (instead of queuing)     │
├────────────────────────────────┼─────────────────────────────────────────┤
│ -o idle_timeout=<seconds>      │ Close idle connections after N seconds  │
│                                │ (0 = disabled, default)                 │
├────────────────────────────────┼─────────────────────────────────────────┤
│ -Z                             │ Enable TLS                              │
├────────────────────────────────┼─────────────────────────────────────────┤
│ -o ssl_chain_cert=<file>       │ TLS certificate chain file              │
├────────────────────────────────┼─────────────────────────────────────────┤
│ -o ssl_key=<file>              │ TLS private key file                    │
└────────────────────────────────┴─────────────────────────────────────────┘
```

### Example Configurations

```bash
# Development (simple, small)
memcached -m 64 -p 11211 -t 2 -c 256

# Small production (single app server)
memcached -d -u memcached -m 1024 -p 11211 -t 4 -c 2048 \
  -o modern

# Large production (dedicated cache server)
memcached -d -u memcached -m 16384 -p 11211 -t 8 -c 10000 \
  -l 10.0.0.5 \
  -o modern,maxconns_fast,idle_timeout=600,hash_algorithm=murmur3

# High-throughput with TLS
memcached -d -u memcached -m 8192 -p 11211 -t 8 -c 10000 \
  -Z -o ssl_chain_cert=/etc/memcached/cert.pem \
  -o ssl_key=/etc/memcached/key.pem \
  -o modern

# Large item support (up to 5 MB values)
memcached -d -u memcached -m 4096 -p 11211 -t 4 -c 2048 \
  -I 5m -o modern
```

---

## Monitoring

### Stats Commands

Memcached exposes rich statistics through the `stats` command family:

```bash
# General server statistics
echo "stats" | nc localhost 11211

# Per-slab-class statistics
echo "stats slabs" | nc localhost 11211

# Per-slab-class item counts and ages
echo "stats items" | nc localhost 11211

# Dump keys from a specific slab class (debugging only)
# stats cachedump <slab_class> <limit>
echo "stats cachedump 1 100" | nc localhost 11211

# Server settings (runtime configuration)
echo "stats settings" | nc localhost 11211

# Connection details
echo "stats conns" | nc localhost 11211

# Size distribution of stored items
echo "stats sizes" | nc localhost 11211
```

### Key Metrics

```
┌──────────────────────────┬───────────────────────────────────────────────┐
│ Metric                   │ Why It Matters                                │
├──────────────────────────┼───────────────────────────────────────────────┤
│ get_hits / get_misses    │ Hit ratio = hits / (hits + misses).           │
│                          │ Target > 90%. Low ratio means cache is not    │
│                          │ effective — check TTLs, key design, capacity. │
├──────────────────────────┼───────────────────────────────────────────────┤
│ evictions                │ Items removed to make room for new ones.      │
│                          │ Rising evictions = cache is too small.        │
│                          │ Increase -m or add nodes.                     │
├──────────────────────────┼───────────────────────────────────────────────┤
│ bytes_read /             │ Network throughput. Useful for capacity        │
│ bytes_written            │ planning and detecting traffic spikes.        │
├──────────────────────────┼───────────────────────────────────────────────┤
│ curr_connections         │ Current open connections. Monitor against      │
│                          │ the -c limit to avoid connection exhaustion.  │
├──────────────────────────┼───────────────────────────────────────────────┤
│ curr_items               │ Total items currently stored.                 │
├──────────────────────────┼───────────────────────────────────────────────┤
│ total_items              │ Total items stored since server start.        │
├──────────────────────────┼───────────────────────────────────────────────┤
│ cmd_get / cmd_set        │ Command counts. The ratio indicates           │
│                          │ read/write balance.                           │
├──────────────────────────┼───────────────────────────────────────────────┤
│ bytes                    │ Current memory used for item storage.         │
│                          │ Compare to limit_maxbytes for capacity.       │
├──────────────────────────┼───────────────────────────────────────────────┤
│ limit_maxbytes           │ Maximum configured memory (-m flag).          │
├──────────────────────────┼───────────────────────────────────────────────┤
│ evicted_unfetched        │ Items evicted that were never read after      │
│                          │ being stored. High = caching wrong data.      │
├──────────────────────────┼───────────────────────────────────────────────┤
│ listen_disabled_num      │ Times server hit max connections and          │
│                          │ stopped accepting. Increase -c if > 0.       │
├──────────────────────────┼───────────────────────────────────────────────┤
│ reclaimed                │ Expired items reclaimed for new storage.      │
│                          │ Healthy — TTLs are working as intended.       │
└──────────────────────────┴───────────────────────────────────────────────┘
```

### Monitoring Dashboard Example

Key formulas for monitoring dashboards:

```
Hit Ratio (%) = (get_hits / (get_hits + get_misses)) * 100

Fill Percentage (%) = (bytes / limit_maxbytes) * 100

Eviction Rate = delta(evictions) / delta(time)

Request Rate = delta(cmd_get + cmd_set) / delta(time)

Connection Utilisation (%) = (curr_connections / max_connections) * 100

Bytes per Request = delta(bytes_read + bytes_written) / delta(cmd_get + cmd_set)

Slab Efficiency (per class) = used_chunks / total_chunks * 100
```

Alert thresholds (suggested starting points):

```
┌───────────────────────────┬────────────┬──────────────────────────┐
│ Metric                    │ Warning    │ Critical                 │
├───────────────────────────┼────────────┼──────────────────────────┤
│ Hit ratio                 │ < 85%      │ < 70%                    │
│ Fill percentage           │ > 85%      │ > 95%                    │
│ Eviction rate             │ > 100/s    │ > 1000/s                 │
│ Connection utilisation    │ > 80%      │ > 95%                    │
│ listen_disabled_num       │ > 0        │ > 10                     │
└───────────────────────────┴────────────┴──────────────────────────┘
```

---

## Common Patterns

### Session Caching

Store user session data in Memcached for fast access across application servers:

```python
import pymemcache
import json
import uuid

client = pymemcache.client.base.Client(('localhost', 11211))

def create_session(user_id, user_data):
    session_id = str(uuid.uuid4())
    session = {
        'user_id': user_id,
        'username': user_data['username'],
        'roles': user_data['roles'],
        'created_at': '2025-01-15T10:30:00Z'
    }
    # Store session with 30-minute TTL
    client.set(
        f'session:{session_id}',
        json.dumps(session),
        expire=1800
    )
    return session_id

def get_session(session_id):
    data = client.get(f'session:{session_id}')
    if data is None:
        return None  # Session expired or evicted
    return json.loads(data)

def refresh_session(session_id):
    """Extend session TTL on activity (sliding expiration)."""
    client.touch(f'session:{session_id}', 1800)

def destroy_session(session_id):
    client.delete(f'session:{session_id}')
```

### Database Query Result Caching

The classic cache-aside (lazy-loading) pattern:

```python
import pymemcache
import json
import hashlib

client = pymemcache.client.base.Client(('localhost', 11211))

def get_user_profile(user_id):
    cache_key = f'user:profile:{user_id}'

    # 1. Try the cache first
    cached = client.get(cache_key)
    if cached is not None:
        return json.loads(cached)  # Cache HIT

    # 2. Cache MISS — query the database
    profile = db.query('SELECT * FROM users WHERE id = %s', user_id)

    # 3. Store in cache for future requests (5-minute TTL)
    client.set(cache_key, json.dumps(profile), expire=300)

    return profile

def update_user_profile(user_id, new_data):
    # 1. Update the database (source of truth)
    db.execute('UPDATE users SET ... WHERE id = %s', user_id, new_data)

    # 2. Invalidate the cache
    client.delete(f'user:profile:{user_id}')

    # Alternative: update the cache directly (write-through)
    # client.set(f'user:profile:{user_id}', json.dumps(new_data), expire=300)
```

### Page and Fragment Caching

Cache rendered HTML fragments to avoid expensive template rendering:

```python
def get_product_listing_html(category_id, page=1):
    cache_key = f'html:products:{category_id}:page:{page}'

    html = client.get(cache_key)
    if html is not None:
        return html.decode('utf-8')  # Cached HTML fragment

    # Render the HTML (expensive: DB query + template rendering)
    products = db.query('SELECT * FROM products WHERE category = %s LIMIT 20 OFFSET %s',
                        category_id, (page - 1) * 20)
    html = render_template('product_list.html', products=products)

    # Cache for 2 minutes
    client.set(cache_key, html.encode('utf-8'), expire=120)
    return html

def invalidate_product_cache(category_id):
    """Invalidate all pages for a category when products change."""
    # Memcached has no wildcard delete — must track keys explicitly
    # or use a generation counter approach
    gen = client.incr(f'gen:products:{category_id}', 1)
    if gen is None:
        client.set(f'gen:products:{category_id}', '1')
```

### Object Serialization Caching

Cache serialised objects with automatic compression for large values:

```python
import pymemcache
import json
import zlib

class CacheSerializer:
    """Custom serializer with compression for large values."""

    COMPRESSION_THRESHOLD = 1024  # Compress values > 1 KB
    FLAG_RAW = 0
    FLAG_JSON = 1
    FLAG_COMPRESSED_JSON = 2

    def serialize(self, key, value):
        if isinstance(value, bytes):
            return value, self.FLAG_RAW

        json_bytes = json.dumps(value).encode('utf-8')

        if len(json_bytes) > self.COMPRESSION_THRESHOLD:
            compressed = zlib.compress(json_bytes)
            return compressed, self.FLAG_COMPRESSED_JSON

        return json_bytes, self.FLAG_JSON

    def deserialize(self, key, value, flags):
        if flags == self.FLAG_RAW:
            return value
        if flags == self.FLAG_COMPRESSED_JSON:
            return json.loads(zlib.decompress(value))
        if flags == self.FLAG_JSON:
            return json.loads(value)
        return value

client = pymemcache.client.base.Client(
    ('localhost', 11211),
    serializer=CacheSerializer().serialize,
    deserializer=CacheSerializer().deserialize,
)

# Usage — serialization and compression handled transparently
client.set('large:object', {'data': [i for i in range(10000)]}, expire=600)
obj = client.get('large:object')
```

### Multiget Optimization

Multiget retrieves multiple keys in a single round-trip, dramatically reducing latency
for batch operations:

```python
def get_user_profiles_batch(user_ids):
    """Fetch multiple user profiles in a single round-trip."""

    # Build cache keys
    cache_keys = [f'user:profile:{uid}' for uid in user_ids]

    # Single multiget request — one round-trip for all keys
    cached = client.get_many(cache_keys)

    results = {}
    missing_ids = []

    for uid in user_ids:
        key = f'user:profile:{uid}'
        if key in cached and cached[key] is not None:
            results[uid] = json.loads(cached[key])
        else:
            missing_ids.append(uid)

    # Batch-fetch missing items from the database
    if missing_ids:
        db_results = db.query(
            'SELECT * FROM users WHERE id IN %s',
            (tuple(missing_ids),)
        )
        # Populate cache for each missing item
        to_cache = {}
        for row in db_results:
            uid = row['id']
            results[uid] = row
            to_cache[f'user:profile:{uid}'] = json.dumps(row)

        # Batch-set missing items (one set per key, or use set_many)
        for key, value in to_cache.items():
            client.set(key, value, expire=300)

    return results
```

```
Multiget Performance Impact:

  Sequential gets (N round-trips):
  ┌────────┐     ┌───────────┐
  │ Client │────►│ Memcached │  × N keys = N round-trips
  │        │◄────│           │  Latency: N × RTT
  └────────┘     └───────────┘

  Multiget (1 round-trip):
  ┌────────┐     ┌───────────┐
  │ Client │═══►│ Memcached │  1 request with N keys
  │        │◄═══│           │  Latency: 1 × RTT (+ processing)
  └────────┘     └───────────┘

  For 100 keys at 0.5ms RTT:
    Sequential: 100 × 0.5ms = 50ms
    Multiget:     1 × 0.5ms =  0.5ms (+ ~1ms processing) ≈ 1.5ms
    Speedup: ~33x
```

---

## When to Choose Memcached

Memcached is the right choice when your requirements align with its strengths:

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Memcached Sweet Spot                               │
│                                                                      │
│  ✓ Simple key-value caching (strings/blobs only)                    │
│  ✓ Maximum memory efficiency for homogeneous string data            │
│  ✓ Multi-threaded performance on multi-core hardware                │
│  ✓ Purely ephemeral cache — no persistence needed                   │
│  ✓ Read-heavy workloads with high fan-out (multiget)                │
│  ✓ Predictable, flat latency under load                             │
│  ✓ Session caching, page caching, query result caching              │
│  ✓ Simple operational model — minimal configuration                 │
│  ✓ BSD license requirement                                          │
│                                                                      │
│  Common Use Cases:                                                   │
│  ─────────────────                                                   │
│  • Web application session stores                                    │
│  • Database query result caching                                     │
│  • API response caching                                              │
│  • Rendered page / HTML fragment caching                             │
│  • Serialised object caching                                         │
│  • Metadata and configuration caching                                │
│  • Reducing load on backend services during traffic spikes           │
│                                                                      │
│  Who Uses Memcached:                                                 │
│  ──────────────────                                                  │
│  Facebook (now Meta), Wikipedia, YouTube, Twitter (now X),           │
│  Reddit, Pinterest, WordPress.com, Tumblr, and many others.         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Limitations

Memcached's simplicity is both its strength and its limitation. Understand these constraints
before choosing Memcached for your architecture:

```
┌────────────────────────────┬─────────────────────────────────────────────┐
│ Limitation                 │ Details and Workarounds                     │
├────────────────────────────┼─────────────────────────────────────────────┤
│ No persistence             │ All data is lost on restart or crash. Not   │
│                            │ suitable for primary data storage. Always   │
│                            │ have a source of truth (database).          │
├────────────────────────────┼─────────────────────────────────────────────┤
│ No replication             │ Data lives on exactly one node. Node        │
│                            │ failure = data loss for that shard.         │
│                            │ Applications must handle cache misses.      │
├────────────────────────────┼─────────────────────────────────────────────┤
│ No built-in clustering     │ Clients must implement distribution logic.  │
│                            │ All clients must agree on the server list   │
│                            │ and hashing algorithm.                      │
├────────────────────────────┼─────────────────────────────────────────────┤
│ No Pub/Sub                 │ Cannot subscribe to key changes or events.  │
│                            │ Use Redis/Valkey or a message broker.       │
├────────────────────────────┼─────────────────────────────────────────────┤
│ No complex data structures │ Strings only. No lists, sets, sorted sets,  │
│                            │ hashes, or streams. Serialize on the client │
│                            │ side if needed.                             │
├────────────────────────────┼─────────────────────────────────────────────┤
│ No server-side scripting   │ Cannot run Lua scripts or stored procedures │
│                            │ on the server. All logic lives in the       │
│                            │ application layer.                          │
├────────────────────────────┼─────────────────────────────────────────────┤
│ 1 MB default value limit   │ Default max value is 1 MB. Configurable    │
│                            │ up to 128 MB with -I flag, but large items  │
│                            │ waste slab memory. Compress or split data.  │
├────────────────────────────┼─────────────────────────────────────────────┤
│ 250-byte key limit         │ Keys must be ≤ 250 bytes. Hash long keys   │
│                            │ (e.g., SHA-256 of the full key string).    │
├────────────────────────────┼─────────────────────────────────────────────┤
│ No wildcard operations     │ Cannot delete or list keys by pattern.      │
│                            │ Use generation counters or explicit key     │
│                            │ tracking for bulk invalidation.             │
├────────────────────────────┼─────────────────────────────────────────────┤
│ No authentication          │ Text protocol has no auth. Binary protocol  │
│ (text protocol)            │ supports SASL. Use network-level security   │
│                            │ (firewalls, VPCs) as the primary defense.   │
├────────────────────────────┼─────────────────────────────────────────────┤
│ No cross-slab-class        │ Memory allocated to one slab class cannot   │
│ memory sharing             │ be used by another without slab_reassign.   │
│                            │ Enable -o slab_reassign,slab_automove.      │
├────────────────────────────┼─────────────────────────────────────────────┤
│ No partial value updates   │ Cannot update part of a value. Must read,   │
│                            │ modify, and write the entire value.         │
│                            │ Use CAS for safe read-modify-write.         │
└────────────────────────────┴─────────────────────────────────────────────┘
```

Despite these limitations, Memcached remains an excellent choice for its intended use
case: a fast, simple, memory-efficient distributed cache for ephemeral data. Its
constraints force clean application design — applications that handle cache misses
gracefully, maintain a reliable source of truth, and treat the cache as a performance
optimisation rather than a data store.

---

## Version History

| Date       | Description     | Author |
|------------|-----------------|--------|
| 2025-04-15 | Initial version | Team   |
