# Networking Fundamentals

## Table of Contents

1. [Overview](#overview)
2. [Why Networking Matters](#why-networking-matters)
3. [Core Building Blocks](#core-building-blocks)
4. [How a Packet Moves](#how-a-packet-moves)
5. [Latency, Bandwidth, and Reliability](#latency-bandwidth-and-reliability)
6. [Common Protocols](#common-protocols)
7. [Prerequisites](#prerequisites)
8. [Next Steps](#next-steps)
9. [Version History](#version-history)

---

## Overview

Networking is the foundation of every distributed system. Whether you are calling an API, opening a database connection, browsing a website, or debugging a Kubernetes service, you are relying on a network path that moves packets between endpoints.

This guide introduces the concepts that appear repeatedly across on-premise infrastructure, cloud networking, and application delivery:

- Hosts, interfaces, and IP addresses
- Ports, sockets, and transport protocols
- Routing, switching, and name resolution
- TLS, proxies, and load balancing
- Network security and troubleshooting

---

## Why Networking Matters

Modern systems are composed of many services that collaborate over a network. When networking is poorly understood, teams often misdiagnose application failures that are actually caused by DNS issues, firewall rules, subnet mistakes, or overloaded load balancers.

| Engineering Area | Networking Question |
|---|---|
| Web APIs | Can clients resolve the service name and reach the correct port? |
| Databases | Are connections allowed through security groups and routes? |
| Kubernetes | How do pods, services, and ingress objects communicate? |
| Cloud | How are VPCs, subnets, NAT gateways, and VPNs connected? |
| Security | Which systems are allowed to talk, and over which protocols? |
| Operations | Where is latency introduced, and where are packets dropped? |

---

## Core Building Blocks

### Endpoints

An endpoint is a host, device, VM, container, or service that can send or receive network traffic.

### Addresses

- **MAC address** — identifies a network interface on a local network
- **IP address** — identifies a host or interface across networks
- **Port** — identifies a process or service on a host
- **Socket** — an IP + port combination used for communication

### Protocols

Protocols define the rules for communication. Examples include Ethernet, IP, TCP, UDP, DNS, HTTP, and TLS.

---

## How a Packet Moves

```text
Application creates HTTP request
        |
        v
TCP segments the data and manages reliability
        |
        v
IP adds source and destination addresses
        |
        v
Link layer places the packet on the local network
        |
        v
Routers forward the packet hop by hop
        |
        v
Destination host reassembles the packet and delivers it to the application
```

A single user request typically depends on many hidden networking steps:

1. Resolve a hostname with DNS
2. Select a route to the destination
3. Establish a TCP connection or send a UDP datagram
4. Negotiate TLS if encryption is required
5. Send application data
6. Receive a response or retry on timeout/failure

---

## Latency, Bandwidth, and Reliability

These three ideas explain much of network behavior:

- **Latency** — how long it takes data to travel from source to destination
- **Bandwidth** — how much data can be transferred per unit of time
- **Reliability** — whether traffic arrives consistently and in order

Low latency does not guarantee high bandwidth, and high bandwidth does not guarantee low latency. Many production problems come from assuming these are interchangeable.

| Symptom | Likely Networking Concern |
|---|---|
| Requests are slow but succeed | High latency, queueing, or TLS/DNS overhead |
| Large file transfer is slow | Limited bandwidth or congestion |
| Random timeouts | Packet loss, flaky routes, firewall state, overloaded backends |
| Some users fail, others succeed | Regional routing, DNS, MTU, or load balancer issues |

---

## Common Protocols

| Protocol | Purpose | Typical Use |
|---|---|---|
| ARP / ND | Resolve local link addresses | Host discovers next-hop MAC address |
| IP | Addressing and forwarding | Basic delivery between networks |
| ICMP | Diagnostics and control | ping, TTL exceeded, unreachable messages |
| TCP | Reliable transport | HTTP, databases, SSH |
| UDP | Lightweight transport | DNS, media streaming, QUIC foundation |
| DNS | Name resolution | Convert names to IP addresses |
| HTTP | Application protocol | APIs, websites, service-to-service calls |
| TLS | Encryption in transit | HTTPS, secure service connections |

---

## Prerequisites

You do not need deep hardware knowledge to start, but these concepts help:

- Basic command line usage
- Familiarity with web applications or cloud services
- Curiosity about how requests travel between systems

---

## Next Steps

- Continue with [01-OSI-AND-TCP-IP.md](01-OSI-AND-TCP-IP.md) to understand protocol layering
- Practice CIDR and subnetting in [02-IP-ADDRESSING-AND-SUBNETTING.md](02-IP-ADDRESSING-AND-SUBNETTING.md)
- Build operational intuition with [07-TROUBLESHOOTING-AND-OBSERVABILITY.md](07-TROUBLESHOOTING-AND-OBSERVABILITY.md)

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial version |
