# Networking Fundamentals

## Table of Contents

1. [Overview](#overview)
2. [Why Networking Matters for Software Engineers](#why-networking-matters-for-software-engineers)
3. [Core Concepts](#core-concepts)
4. [Protocols, Ports, and Packets](#protocols-ports-and-packets)
5. [Latency, Throughput, and Reliability](#latency-throughput-and-reliability)
6. [Common Network Boundaries](#common-network-boundaries)
7. [A Practical Mental Model](#a-practical-mental-model)
8. [Prerequisites](#prerequisites)
9. [Next Steps](#next-steps)
10. [Version History](#version-history)

---

## Overview

Networking is the foundation beneath every distributed system, API call, database connection, and browser request. Even if you work primarily at the application layer, production incidents often come down to name resolution, routing, packet loss, timeouts, retransmissions, or misconfigured firewalls.

This guide introduces the concepts that help software engineers reason about network behavior without needing to become full-time network engineers.

### Target Audience

- **Backend engineers** building APIs, services, and data pipelines
- **Frontend engineers** debugging browser requests, CDNs, and TLS issues
- **Platform and DevOps engineers** designing cloud networks and troubleshooting connectivity
- **Architects** making decisions about load balancing, segmentation, and resilience

### Scope

- Core networking vocabulary and mental models
- Packets, ports, protocols, and connections
- Latency, bandwidth, packet loss, and reliability trade-offs
- DNS, routing, NAT, TLS, proxies, and load balancers
- Troubleshooting workflows that connect app symptoms to network causes

---

## Why Networking Matters for Software Engineers

A surprising number of application problems are actually networking problems in disguise:

- `TimeoutException` may really mean packet loss or a bad route
- `Connection refused` may mean the service is down, bound to the wrong interface, or blocked by a firewall
- `Name or service not known` usually points to DNS or service discovery issues
- Slow pages may be dominated by TLS handshakes, origin distance, or connection pool exhaustion
- Intermittent failures often come from retries amplifying a flaky dependency or overloaded load balancer

Networking literacy improves incident response because it helps you ask better questions:

- Can the client resolve the hostname?
- Can it reach the target IP and port?
- Is the route healthy and symmetric?
- Is the connection encrypted and trusted?
- Is the application failing, or is traffic never arriving?

---

## Core Concepts

| Concept | What it means | Why it matters |
|--------|----------------|----------------|
| Packet | Unit of data moved across a network | Most failures eventually affect packets: delay, drop, reorder, duplicate |
| Protocol | Agreed rules for communication | Different layers solve different problems |
| Port | Logical endpoint on a host | Lets many services share one IP address |
| Latency | Time taken for data to travel | Dominates user experience for chatty systems |
| Throughput | Data transferred per unit of time | Matters for bulk transfer and streaming |
| Jitter | Variation in latency | Critical for voice, video, and real-time systems |
| Packet loss | Dropped packets | Causes retransmits, poor performance, and timeouts |
| MTU | Maximum packet size on a link | Mismatches can create fragmentation or black holes |

---

## Protocols, Ports, and Packets

A typical web request moves through several protocols:

```text
Browser → DNS lookup → TCP handshake → TLS handshake → HTTP request → Response
```

At each stage, a different category of failure can occur:

1. **DNS** — hostname cannot be resolved
2. **TCP** — SYN packets never complete the handshake
3. **TLS** — certificate validation fails
4. **HTTP** — proxy, gateway, or application returns an error

Ports provide multiplexing on a host. Common examples:

- `53` — DNS
- `80` — HTTP
- `443` — HTTPS
- `22` — SSH
- `5432` — PostgreSQL
- `6379` — Redis

A listening service is not enough by itself; clients also need a reachable route, allowed firewall path, matching protocol, and enough capacity to complete the exchange.

---

## Latency, Throughput, and Reliability

### Latency Sources

Latency accumulates from multiple components:

- Physical distance and speed-of-light limits
- Queuing delay on overloaded devices or services
- DNS and TLS handshakes before the first byte of application data
- Retransmissions after packet loss
- Application retries and exponential backoff

### Throughput Limits

Throughput is constrained by the slowest part of the path:

- Link capacity
- Congestion control behavior
- Window sizes and flow control
- CPU limits on proxies, load balancers, or encrypted connections

### Reliability Trade-Offs

Reliable delivery typically costs more round trips and buffering. Faster, lower-overhead approaches often sacrifice guarantees.

| Protocol | Strength | Trade-off |
|----------|----------|-----------|
| TCP | Reliable, ordered delivery | More overhead and head-of-line blocking |
| UDP | Low overhead, low latency | App must handle ordering/reliability if needed |
| QUIC | Secure transport with improved multiplexing | Newer operational patterns and tooling |

---

## Common Network Boundaries

When traffic crosses boundaries, complexity increases:

- **Host boundary** — one process talks to another on the same machine
- **Subnet boundary** — a router or gateway becomes involved
- **Availability zone / region boundary** — latency and cost increase noticeably
- **Public internet boundary** — trust, filtering, and path variability become major factors
- **Corporate/VPN boundary** — split DNS, forced tunnels, and egress policies often apply

Knowing which boundary a request crosses helps narrow down the failure domain quickly.

---

## A Practical Mental Model

Use this quick checklist during incidents:

1. **Name** — Does the client resolve the intended hostname?
2. **Path** — Is there a valid route from source to destination?
3. **Reachability** — Is the target IP and port reachable?
4. **Trust** — Does TLS/certificate validation succeed?
5. **Capacity** — Are proxies, pools, or links overloaded?
6. **Application** — If the network path is healthy, what is the service doing?

This layer-by-layer approach prevents jumping straight into application logs when the issue is actually lower in the stack.

---

## Prerequisites

You do not need deep networking experience before continuing. Familiarity with basic client/server architecture and HTTP requests will help.

---

## Next Steps

- Continue with [01-OSI-AND-TCP-IP.md](01-OSI-AND-TCP-IP.md) for the main protocol-layer mental model
- Then learn [02-IP-ADDRESSING-AND-SUBNETTING.md](02-IP-ADDRESSING-AND-SUBNETTING.md) to understand how endpoints are identified

---

## Version History

| Version | Date | Changes |
|--------|------|---------|
| 1.0 | 2026 | Initial networking fundamentals overview |
