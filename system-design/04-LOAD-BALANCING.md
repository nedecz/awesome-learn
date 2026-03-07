# Load Balancing

## Table of Contents

1. [Overview](#overview)
2. [What Is Load Balancing](#what-is-load-balancing)
   - [Definition](#definition)
   - [Why Load Balancing Matters](#why-load-balancing-matters)
3. [Layer 4 vs Layer 7 Load Balancing](#layer-4-vs-layer-7-load-balancing)
   - [OSI Model Context](#osi-model-context)
   - [Layer 4 (Transport)](#layer-4-transport)
   - [Layer 7 (Application)](#layer-7-application)
   - [Comparison](#comparison)
4. [Load Balancing Algorithms](#load-balancing-algorithms)
   - [Round Robin](#round-robin)
   - [Weighted Round Robin](#weighted-round-robin)
   - [Least Connections](#least-connections)
   - [IP Hash](#ip-hash)
   - [Random](#random)
   - [Consistent Hashing](#consistent-hashing)
   - [Algorithm Comparison](#algorithm-comparison)
5. [Health Checks](#health-checks)
   - [Active Health Checks](#active-health-checks)
   - [Passive Health Checks](#passive-health-checks)
   - [Liveness and Readiness Probes](#liveness-and-readiness-probes)
6. [Session Persistence](#session-persistence)
   - [Sticky Sessions](#sticky-sessions)
   - [Cookie-Based Affinity](#cookie-based-affinity)
   - [Session Replication](#session-replication)
7. [Global Server Load Balancing (GSLB)](#global-server-load-balancing-gslb)
   - [DNS-Based Load Balancing](#dns-based-load-balancing)
   - [Latency-Based Routing](#latency-based-routing)
   - [Geographic Routing](#geographic-routing)
8. [Software vs Hardware Load Balancers](#software-vs-hardware-load-balancers)
   - [Software Load Balancers](#software-load-balancers)
   - [Hardware Load Balancers](#hardware-load-balancers)
   - [Cloud Load Balancers](#cloud-load-balancers)
9. [SSL/TLS Termination](#ssltls-termination)
   - [SSL Offloading](#ssl-offloading)
   - [End-to-End Encryption](#end-to-end-encryption)
   - [Re-Encryption](#re-encryption)
10. [High Availability for Load Balancers](#high-availability-for-load-balancers)
    - [Active-Passive](#active-passive)
    - [Active-Active](#active-active)
    - [Floating IPs](#floating-ips)
11. [Next Steps](#next-steps)
12. [Version History](#version-history)

## Overview

Load balancing is the practice of distributing incoming network traffic across multiple backend servers to ensure no single server bears too much demand. It is a foundational building block in scalable, reliable system design.

### Target Audience

- Backend engineers designing scalable web services
- Platform and infrastructure engineers managing traffic distribution
- DevOps engineers configuring load balancers and health checks

### Scope

- Layer 4 and Layer 7 load balancing concepts
- Common load balancing algorithms and their trade-offs
- Health checking strategies
- Session persistence techniques
- Global server load balancing (GSLB)
- Software, hardware, and cloud load balancer options
- SSL/TLS termination patterns
- High availability configurations for load balancers

---

## What Is Load Balancing

### Definition

A **load balancer** sits between clients and a pool of backend servers. It receives incoming requests and forwards each one to a server according to a routing algorithm, distributing the workload so that no single server is overwhelmed.

```
                         ┌──────────────┐
                         │   Clients    │
                         └──────┬───────┘
                                │
                                ▼
                      ┌──────────────────┐
                      │  Load Balancer   │
                      └────┬────┬────┬───┘
                           │    │    │
                    ┌──────┘    │    └──────┐
                    ▼           ▼           ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │ Server A │ │ Server B │ │ Server C │
              └──────────┘ └──────────┘ └──────────┘
```

### Why Load Balancing Matters

Without load balancing, a single server must handle all traffic. This creates a single point of failure and limits the throughput of the entire system.

| Problem | Without Load Balancing | With Load Balancing |
|---|---|---|
| **Scalability** | Vertical only (bigger machine) | Horizontal (add more servers) |
| **Availability** | One server down = total outage | Traffic shifts to healthy servers |
| **Performance** | Uneven load causes hot spots | Even distribution reduces latency |
| **Maintenance** | Downtime required for deploys | Rolling deploys with zero downtime |

---

## Layer 4 vs Layer 7 Load Balancing

### OSI Model Context

Load balancers operate at different layers of the OSI model. The two most common are Layer 4 (Transport) and Layer 7 (Application).

```
  OSI Model — Load Balancer Layers
  ─────────────────────────────────

  Layer 7 │ Application  │ HTTP, gRPC, WebSocket     ◀── L7 LB operates here
  Layer 6 │ Presentation │ SSL/TLS, compression
  Layer 5 │ Session      │ Session management
  Layer 4 │ Transport    │ TCP, UDP                   ◀── L4 LB operates here
  Layer 3 │ Network      │ IP routing
  Layer 2 │ Data Link    │ Ethernet, MAC
  Layer 1 │ Physical     │ Cables, signals
```

### Layer 4 (Transport)

A Layer 4 load balancer routes traffic based on **IP address and TCP/UDP port** without inspecting the payload. It operates on network packets and is very fast because it does not need to parse the application protocol.

```
  Layer 4 Load Balancing
  ──────────────────────

  Client ──TCP SYN──▶ ┌──────────┐ ──TCP SYN──▶ ┌──────────┐
                       │  L4 LB   │              │ Server A │
                       │          │              │ :8080    │
  Routing decision:    │ src IP   │              └──────────┘
    - Source IP/Port   │ dst port │
    - Dest IP/Port     └──────────┘
    - No payload
      inspection
```

### Layer 7 (Application)

A Layer 7 load balancer inspects the **full HTTP request** — URL path, headers, cookies, body — and makes routing decisions based on application-level content.

```
  Layer 7 Load Balancing
  ──────────────────────

  Client ──HTTP──▶ ┌──────────┐
                   │  L7 LB   │
                   │          │──▶ /api/*     ──▶ API servers
  Routing by:      │ inspects │──▶ /static/*  ──▶ CDN / static servers
    - URL path     │ headers, │──▶ /admin/*   ──▶ Admin servers
    - Host header  │ cookies, │
    - Cookies      │ body     │
    - HTTP method  └──────────┘
```

### Comparison

| Dimension | Layer 4 | Layer 7 |
|---|---|---|
| **Operates on** | TCP/UDP packets | HTTP/HTTPS requests |
| **Routing criteria** | IP, port | URL, headers, cookies, method |
| **Performance** | Very high (no payload parsing) | Lower (must parse HTTP) |
| **SSL termination** | Pass-through or terminate | Typically terminates |
| **Content-based routing** | No | Yes |
| **WebSocket support** | Yes (transparent) | Yes (protocol-aware) |
| **Use case** | High-throughput TCP services, databases | Web apps, APIs, microservices |
| **Examples** | AWS NLB, HAProxy (TCP mode) | AWS ALB, NGINX, Envoy |

**When to use each:**

- ✅ Use **Layer 4** when you need raw TCP/UDP forwarding with minimal latency (e.g., database proxies, gaming servers, non-HTTP protocols)
- ✅ Use **Layer 7** when you need content-based routing, header inspection, or path-based routing (e.g., REST APIs, web applications, microservices)

---

## Load Balancing Algorithms

### Round Robin

Requests are distributed to servers in sequential, circular order. Server A gets the first request, Server B the second, Server C the third, then back to Server A.

```
  Round Robin
  ───────────

  Request 1 ──▶ Server A
  Request 2 ──▶ Server B
  Request 3 ──▶ Server C
  Request 4 ──▶ Server A   ← cycle repeats
  Request 5 ──▶ Server B
  Request 6 ──▶ Server C
```

- ✅ Simple to implement and understand
- ❌ Does not account for server capacity or current load

### Weighted Round Robin

Each server is assigned a weight proportional to its capacity. A server with weight 3 receives three times the requests of a server with weight 1.

```
  Weighted Round Robin (A=3, B=1, C=1)
  ─────────────────────────────────────

  Request 1 ──▶ Server A  (weight 3)
  Request 2 ──▶ Server A
  Request 3 ──▶ Server A
  Request 4 ──▶ Server B  (weight 1)
  Request 5 ──▶ Server C  (weight 1)
  Request 6 ──▶ Server A  ← cycle repeats
```

- ✅ Accounts for heterogeneous server capacities
- ❌ Weights must be configured manually and updated when capacity changes

### Least Connections

Each new request is sent to the server with the fewest active connections. This naturally adapts to servers that process requests at different speeds.

```
  Least Connections
  ─────────────────

  Server A: 12 active connections
  Server B:  4 active connections  ◀── next request goes here
  Server C:  8 active connections
```

- ✅ Adapts to varying request processing times
- ✅ Prevents slow servers from being overwhelmed
- ❌ Requires the load balancer to track connection counts in real time

### IP Hash

A hash of the client's IP address determines which server receives the request. The same client IP always maps to the same server (as long as the server pool is stable).

```
  IP Hash
  ───────

  hash(10.0.0.1) mod 3 = 0 ──▶ Server A
  hash(10.0.0.2) mod 3 = 2 ──▶ Server C
  hash(10.0.0.3) mod 3 = 1 ──▶ Server B
  hash(10.0.0.1) mod 3 = 0 ──▶ Server A  ← same client, same server
```

- ✅ Provides session affinity without cookies
- ❌ Uneven distribution if client IPs are not uniformly distributed
- ❌ Adding or removing servers rehashes most clients to different servers

### Random

Each request is sent to a randomly selected server. With a sufficiently large number of requests, the distribution approaches uniform.

- ✅ No state required — extremely simple
- ❌ No guarantee of even distribution for small request counts
- ❌ Does not account for server capacity or health

### Consistent Hashing

Servers and requests are mapped onto a virtual ring. Each request is assigned to the next server clockwise on the ring. When a server is added or removed, only the adjacent segment of keys is redistributed.

```
  Consistent Hashing Ring
  ───────────────────────

            Server A
               │
        ┌──────┴──────┐
       ╱                ╲
      │    hash ring     │
      │                  │
  Server D            Server B
      │                  │
       ╲                ╱
        └──────┬──────┘
               │
            Server C

  Request hash lands between A and B ──▶ routed to Server B

  If Server B is removed:
    Only requests in the A→B segment move to Server C
    All other mappings remain unchanged
```

- ✅ Minimal redistribution when servers are added or removed
- ✅ Ideal for caching layers and stateful partitioning
- ❌ More complex to implement than simple hashing
- ❌ Can produce uneven distribution without virtual nodes

### Algorithm Comparison

| Algorithm | Statefulness | Even Distribution | Session Affinity | Best For |
|---|---|---|---|---|
| **Round Robin** | Stateless | Even (equal servers) | No | Homogeneous, stateless services |
| **Weighted Round Robin** | Stateless | Configurable | No | Heterogeneous server capacities |
| **Least Connections** | Stateful | Adaptive | No | Varying request processing times |
| **IP Hash** | Stateless | Depends on IP spread | Yes (by IP) | Simple session affinity |
| **Random** | Stateless | Probabilistic | No | Simple, low-overhead balancing |
| **Consistent Hashing** | Stateless | Good (with virtual nodes) | Yes (by key) | Distributed caches, partitioned data |

---

## Health Checks

A load balancer must detect unhealthy servers and stop sending traffic to them. Health checks are the mechanism for this.

### Active Health Checks

The load balancer periodically sends probe requests to each backend server and evaluates the response.

```
  Active Health Check
  ───────────────────

  Load Balancer                    Backend Servers
  ─────────────                    ───────────────

  every 5s ──GET /health──▶  Server A ──▶ 200 OK     ✅ healthy
  every 5s ──GET /health──▶  Server B ──▶ 200 OK     ✅ healthy
  every 5s ──GET /health──▶  Server C ──▶ timeout     ❌ unhealthy
                                                         → remove from pool
```

| Parameter | Typical Value | Purpose |
|---|---|---|
| **Interval** | 5–10 seconds | Time between consecutive probes |
| **Timeout** | 2–3 seconds | Max wait for a response before marking as failed |
| **Healthy threshold** | 2–3 consecutive successes | Passes required before re-adding a recovered server |
| **Unhealthy threshold** | 2–3 consecutive failures | Failures required before removing a server |

### Passive Health Checks

The load balancer monitors real production traffic rather than sending synthetic probes. If a server starts returning errors or timing out on actual requests, it is marked unhealthy.

```
  Passive Health Check
  ────────────────────

  Client ──request──▶ Load Balancer ──forward──▶ Server A ──▶ 200 OK  ✅
  Client ──request──▶ Load Balancer ──forward──▶ Server B ──▶ 502     ❌
  Client ──request──▶ Load Balancer ──forward──▶ Server B ──▶ 502     ❌
  Client ──request──▶ Load Balancer ──forward──▶ Server B ──▶ 502     ❌
                                                                  3 failures
                                                             → remove Server B
```

- ✅ No additional probe traffic — uses real requests
- ✅ Detects issues that synthetic probes may miss (e.g., application-level errors)
- ❌ Unhealthy servers receive some real traffic before being removed
- ❌ Low-traffic servers may not generate enough data points for detection

### Liveness and Readiness Probes

Modern platforms distinguish between two types of health:

| Probe | Question | Failure Action |
|---|---|---|
| **Liveness** | Is the process alive and not deadlocked? | Restart the instance |
| **Readiness** | Is the instance ready to accept traffic? | Remove from load balancer pool (do not restart) |

A server may be **live but not ready** — for example, during startup while it warms caches or establishes database connections. Sending traffic to such a server would result in errors.

```
  Server Lifecycle
  ────────────────

  ┌─────────┐     ┌──────────────────┐     ┌───────────────┐
  │ Starting │────▶│ Live, Not Ready  │────▶│ Live & Ready  │
  │          │     │ (warming caches) │     │ (serving)     │
  └─────────┘     └──────────────────┘     └───────────────┘
                   Liveness: ✅              Liveness: ✅
                   Readiness: ❌             Readiness: ✅
                   Traffic: none             Traffic: active
```

---

## Session Persistence

Stateless services do not require session persistence. However, some applications maintain in-memory session state and require consecutive requests from the same client to reach the same server.

### Sticky Sessions

The load balancer remembers which server handled a client's first request and routes all subsequent requests from that client to the same server.

```
  Sticky Sessions
  ───────────────

  Client X ──req 1──▶ LB ──▶ Server A  ← assigned
  Client X ──req 2──▶ LB ──▶ Server A  ← same server
  Client X ──req 3──▶ LB ──▶ Server A  ← same server

  Client Y ──req 1──▶ LB ──▶ Server B  ← different client, different server
```

- ✅ Simple — no changes required to the application
- ❌ Uneven load distribution if some clients are much more active
- ❌ If the target server fails, the session is lost

### Cookie-Based Affinity

The load balancer inserts a cookie into the HTTP response that identifies the backend server. On subsequent requests, the client sends the cookie back, and the load balancer routes to the encoded server.

```
  Cookie-Based Affinity
  ─────────────────────

  1. Client ──GET /app──▶ LB ──▶ Server B
  2. Server B ──200 OK──▶ LB ──▶ Client
                          LB adds: Set-Cookie: SERVERID=srv-b

  3. Client ──GET /app──▶ LB
     Cookie: SERVERID=srv-b
     LB reads cookie ──▶ Server B
```

- ✅ Survives client IP changes (e.g., mobile networks)
- ✅ Load balancer controls the cookie — no application changes needed
- ❌ Requires Layer 7 load balancing (must inspect HTTP headers)
- ❌ Does not work for non-HTTP protocols

### Session Replication

Instead of pinning clients to servers, replicate session data across all servers so that any server can handle any request.

| Strategy | Mechanism | Trade-Off |
|---|---|---|
| **In-memory replication** | Servers replicate sessions to peers via multicast or gossip | Low latency; does not scale beyond a small cluster |
| **External session store** | Sessions stored in Redis, Memcached, or a database | Scales well; adds a network hop and external dependency |
| **JWT / token-based** | Session state encoded in a signed token sent with each request | No server-side storage; token size grows with session data |

- ✅ Any server can handle any request — enables true horizontal scaling
- ✅ Server failure does not lose sessions
- ❌ External session stores add latency and become a dependency
- ❌ Session replication increases network and memory usage

---

## Global Server Load Balancing (GSLB)

GSLB distributes traffic across geographically dispersed data centers or cloud regions to reduce latency, improve availability, and satisfy data residency requirements.

### DNS-Based Load Balancing

The simplest form of GSLB uses DNS to return different IP addresses based on the client's location or the health of data centers.

```
  DNS-Based GSLB
  ──────────────

  User (Europe)                             User (US West)
       │                                         │
       ▼                                         ▼
  ┌──────────┐                              ┌──────────┐
  │   DNS    │                              │   DNS    │
  │ Resolver │                              │ Resolver │
  └────┬─────┘                              └────┬─────┘
       │ api.example.com?                        │ api.example.com?
       ▼                                         ▼
  ┌────────────────────────────────────────────────────┐
  │              Authoritative DNS (GSLB)              │
  │                                                    │
  │  Europe client → 203.0.113.10 (EU-West DC)        │
  │  US West client → 198.51.100.20 (US-West DC)      │
  └────────────────────────────────────────────────────┘
       │                                         │
       ▼                                         ▼
  ┌──────────┐                              ┌──────────┐
  │ EU-West  │                              │ US-West  │
  │   DC     │                              │   DC     │
  └──────────┘                              └──────────┘
```

- ✅ Works with any client — no special software required
- ❌ DNS TTL caching delays failover
- ❌ Coarse-grained — operates per DNS query, not per request

### Latency-Based Routing

The GSLB measures latency from each resolver to each data center and returns the IP of the lowest-latency data center.

| Routing Method | Decision Basis | Precision |
|---|---|---|
| **Latency-based** | Measured network latency | High — picks fastest path |
| **Geographic** | Client's geographic region (GeoIP) | Medium — region-level |
| **Geoproximity** | Distance with configurable bias | Medium — tunable |

### Geographic Routing

Traffic is routed based on the client's geographic location, determined by GeoIP databases. This ensures compliance with data residency regulations and reduces latency for regional users.

- ✅ Ensures data residency compliance (e.g., GDPR — EU traffic stays in EU)
- ✅ Predictable routing — same region always maps to same data center
- ❌ GeoIP databases are not always accurate
- ❌ Does not account for actual network conditions

---

## Software vs Hardware Load Balancers

### Software Load Balancers

Software load balancers run on commodity servers or virtual machines and are the dominant choice in modern infrastructure.

| Software LB | Layer | Key Features |
|---|---|---|
| **NGINX** | L4 / L7 | HTTP/2, gRPC, WebSocket, built-in caching, widespread adoption |
| **HAProxy** | L4 / L7 | Very high performance, advanced health checks, detailed stats |
| **Envoy** | L4 / L7 | Service mesh sidecar, xDS API for dynamic config, observability |
| **Traefik** | L7 | Auto-discovery, Let's Encrypt integration, Kubernetes-native |

- ✅ Low cost — runs on standard hardware or cloud VMs
- ✅ Highly configurable — open source with large communities
- ✅ Easy to automate — configuration as code, API-driven
- ❌ Throughput limited by the underlying host's CPU and network

### Hardware Load Balancers

Hardware load balancers are purpose-built appliances with custom ASICs for packet processing. They deliver extremely high throughput and deterministic latency.

| Vendor | Product | Throughput |
|---|---|---|
| **F5** | BIG-IP | Up to 300+ Gbps |
| **Citrix** | ADC (NetScaler) | Up to 200+ Gbps |
| **A10** | Thunder ADC | Up to 220+ Gbps |

- ✅ Extreme throughput and low, predictable latency
- ✅ Dedicated hardware offloads SSL and compression
- ❌ Very expensive — capital expenditure model
- ❌ Difficult to scale — requires purchasing additional appliances
- ❌ Vendor lock-in — proprietary configuration and management

### Cloud Load Balancers

Cloud providers offer managed load balancers that are fully integrated with their ecosystems.

| Cloud | Layer 4 | Layer 7 |
|---|---|---|
| **AWS** | Network Load Balancer (NLB) | Application Load Balancer (ALB) |
| **Azure** | Azure Load Balancer | Azure Application Gateway |
| **GCP** | Network Load Balancer | Global HTTP(S) Load Balancer |

| Feature | Cloud LB | Self-Managed LB |
|---|---|---|
| **Scaling** | Automatic | Manual (add instances) |
| **Availability** | Managed SLA (99.99%) | Must configure HA yourself |
| **Cost model** | Pay-per-use | Fixed infra cost |
| **Customization** | Limited to provider features | Full control |
| **SSL certificates** | Integrated (ACM, Key Vault) | Self-managed (Let's Encrypt, etc.) |

---

## SSL/TLS Termination

### SSL Offloading

The load balancer decrypts incoming HTTPS traffic and forwards plain HTTP to backend servers. This offloads the CPU-intensive TLS handshake from application servers.

```
  SSL Offloading
  ──────────────

  Client ──HTTPS──▶ ┌──────────┐ ──HTTP──▶ ┌──────────┐
                     │    LB    │           │ Server A │
                     │ (decrypt)│           │ (plain)  │
                     └──────────┘           └──────────┘

  TLS terminates at the LB. Backend traffic is unencrypted.
```

- ✅ Reduces CPU load on backend servers
- ✅ Centralizes certificate management
- ❌ Traffic between LB and backends is unencrypted — acceptable only in trusted networks

### End-to-End Encryption

The load balancer passes encrypted traffic through to backend servers without decrypting. The backend servers handle TLS termination themselves.

```
  End-to-End Encryption (TLS Passthrough)
  ────────────────────────────────────────

  Client ──HTTPS──▶ ┌──────────┐ ──HTTPS──▶ ┌──────────┐
                     │  L4 LB   │            │ Server A │
                     │ (no      │            │ (decrypt)│
                     │  decrypt)│            └──────────┘
                     └──────────┘

  LB cannot inspect HTTP content — limited to L4 routing.
```

- ✅ Maximum security — data is encrypted in transit everywhere
- ❌ Load balancer cannot inspect HTTP headers — no L7 routing
- ❌ Each backend server must manage its own certificates

### Re-Encryption

The load balancer decrypts incoming traffic, inspects it, and then re-encrypts it before forwarding to backends. This combines the benefits of L7 routing with encrypted backend traffic.

```
  Re-Encryption
  ─────────────

  Client ──HTTPS──▶ ┌──────────┐ ──HTTPS──▶ ┌──────────┐
                     │  L7 LB   │            │ Server A │
                     │ decrypt  │            │ decrypt  │
                     │ inspect  │            └──────────┘
                     │ re-encrypt│
                     └──────────┘

  Two TLS sessions: Client↔LB and LB↔Backend.
```

- ✅ L7 routing with encrypted backend traffic — best of both
- ❌ Double TLS overhead — higher CPU usage on the load balancer
- ❌ More complex certificate management (two sets of certificates)

| Pattern | LB Decrypts? | Backend Encrypted? | L7 Routing? | Use Case |
|---|---|---|---|---|
| **SSL Offloading** | Yes | No | Yes | Trusted internal network |
| **TLS Passthrough** | No | Yes | No | Maximum security, L4 only |
| **Re-Encryption** | Yes | Yes | Yes | Compliance, zero-trust network |

---

## High Availability for Load Balancers

The load balancer itself must not be a single point of failure. High availability configurations ensure that if one load balancer fails, another takes over.

### Active-Passive

One load balancer handles all traffic (active) while a standby (passive) monitors the active node. If the active node fails, the passive node takes over.

```
  Active-Passive
  ──────────────

  Clients ──traffic──▶ ┌──────────────┐
                        │  LB Active   │ ◀── handles all traffic
                        └──────┬───────┘
                               │ heartbeat
                        ┌──────▼───────┐
                        │  LB Passive  │ ◀── standby, no traffic
                        └──────────────┘

  On failure:

  Clients ──traffic──▶ ┌──────────────┐
                        │  LB Active   │ ✖ down
                        └──────────────┘
                        ┌──────────────┐
  Clients ──traffic──▶  │  LB Passive  │ ◀── promoted to active
                        └──────────────┘
```

- ✅ Simple — clear ownership of traffic
- ❌ Passive node is idle — wasted capacity
- ❌ Failover introduces a brief interruption (seconds)

### Active-Active

Multiple load balancers simultaneously handle traffic. Traffic is distributed across all nodes, typically via DNS round robin or an upstream network device.

```
  Active-Active
  ─────────────

                      ┌──────────────┐
  Clients ──traffic──▶│   LB Node 1  │──▶ Backend Pool
                      └──────────────┘
                      ┌──────────────┐
  Clients ──traffic──▶│   LB Node 2  │──▶ Backend Pool
                      └──────────────┘

  Both nodes serve traffic. If Node 1 fails,
  all traffic shifts to Node 2.
```

- ✅ No idle capacity — both nodes serve traffic
- ✅ Higher total throughput than active-passive
- ❌ More complex — must synchronize state (connection tables, sessions)
- ❌ Split-brain risk if nodes lose communication with each other

### Floating IPs

A **floating IP** (also called a virtual IP or VIP) is an IP address that can be reassigned from one load balancer to another in seconds. It is commonly used with active-passive setups to provide seamless failover.

```
  Floating IP Failover
  ────────────────────

  VIP: 203.0.113.50  ──▶ LB Active  (owns the VIP)
                          LB Passive (monitors)

  On failure:

  VIP: 203.0.113.50  ──▶ LB Passive (claims the VIP via ARP/GARP)

  Clients continue to connect to 203.0.113.50 — no DNS change needed.
```

- ✅ Instant failover — no DNS propagation delay
- ✅ Transparent to clients — same IP address
- ❌ Typically limited to the same Layer 2 network segment or cloud VPC

---

## Next Steps

Continue to the next topic to explore rate limiting, circuit breakers, and other resilience patterns for protecting services from overload and cascading failures.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial version — L4/L7 load balancing, algorithms, health checks, session persistence, GSLB, software/hardware/cloud LBs, SSL/TLS termination, high availability |
