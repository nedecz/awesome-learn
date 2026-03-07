# Networking Learning Resources

A practical guide to networking fundamentals for software engineers — from OSI and TCP/IP basics to IP addressing, routing, DNS, HTTP/TLS, load balancing, network security, and production troubleshooting.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | Core networking concepts, packets, latency, and protocol layers | **Start here** |
| [01-OSI-AND-TCP-IP](01-OSI-AND-TCP-IP.md) | How protocol layers fit together and where common tools operate | When you need a mental model for debugging |
| [02-IP-ADDRESSING-AND-SUBNETTING](02-IP-ADDRESSING-AND-SUBNETTING.md) | IPv4/IPv6, CIDR, subnets, NAT, and address planning | When designing or troubleshooting networks |
| [03-ROUTING-AND-SWITCHING](03-ROUTING-AND-SWITCHING.md) | How traffic moves inside and between networks | When learning how packets find their destination |
| [04-DNS-AND-NAME-RESOLUTION](04-DNS-AND-NAME-RESOLUTION.md) | DNS records, resolution flow, caching, and failure modes | When services cannot find each other |
| [05-HTTP-HTTPS-AND-TLS](05-HTTP-HTTPS-AND-TLS.md) | Application-layer networking, encryption in transit, and connection behavior | When building APIs and web applications |
| [06-LOAD-BALANCING-AND-TRAFFIC-MANAGEMENT](06-LOAD-BALANCING-AND-TRAFFIC-MANAGEMENT.md) | Reverse proxies, load balancers, CDNs, and traffic shaping | When operating distributed systems |
| [07-NETWORK-SECURITY](07-NETWORK-SECURITY.md) | Firewalls, segmentation, zero trust, and defensive controls | **Essential — every production system** |
| [08-OBSERVABILITY-AND-TROUBLESHOOTING](08-OBSERVABILITY-AND-TROUBLESHOOTING.md) | Metrics, packet captures, traceroute, and practical debugging workflows | When incidents involve latency or reachability |
| [09-BEST-PRACTICES](09-BEST-PRACTICES.md) | Design principles and production-ready networking habits | **Essential — production checklist** |
| [10-ANTI-PATTERNS](10-ANTI-PATTERNS.md) | Common networking mistakes and how to avoid them | **Essential — what NOT to do** |
| [LEARNING-PATH](LEARNING-PATH.md) | Structured guide with phased study and exercises | **Use after the overview** |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Understand packets, latency, bandwidth, and reliability
   - Learn why software engineers need networking literacy
   - Build a vocabulary for debugging connectivity issues

2. **Learn the Protocol Layers** ([01-OSI-AND-TCP-IP](01-OSI-AND-TCP-IP.md))
   - Map switches, routers, DNS, TCP, and HTTP to their layers
   - Understand encapsulation and why layer boundaries matter

3. **Study Addressing and Routing** ([02-IP-ADDRESSING-AND-SUBNETTING](02-IP-ADDRESSING-AND-SUBNETTING.md), [03-ROUTING-AND-SWITCHING](03-ROUTING-AND-SWITCHING.md))
   - Learn CIDR, subnets, gateways, routing tables, and NAT
   - See how traffic crosses network boundaries

4. **Follow the Learning Path** ([LEARNING-PATH](LEARNING-PATH.md))
   - Progress from fundamentals to production troubleshooting
   - Reinforce concepts with hands-on exercises

### For Experienced Engineers

1. **Review HTTP/TLS** ([05-HTTP-HTTPS-AND-TLS](05-HTTP-HTTPS-AND-TLS.md))
   - Refresh connection reuse, handshakes, and certificate validation
   - Close common knowledge gaps between app and network behavior

2. **Strengthen Traffic Management Skills** ([06-LOAD-BALANCING-AND-TRAFFIC-MANAGEMENT](06-LOAD-BALANCING-AND-TRAFFIC-MANAGEMENT.md))
   - Compare L4 vs L7 balancing, proxies, and ingress patterns
   - Understand how retry and timeout policies interact with the network

3. **Audit Security and Operations** ([07-NETWORK-SECURITY](07-NETWORK-SECURITY.md), [08-OBSERVABILITY-AND-TROUBLESHOOTING](08-OBSERVABILITY-AND-TROUBLESHOOTING.md))
   - Apply segmentation and least-privilege access
   - Use packet- and path-level tools to isolate failures quickly

## 🏗️ Networking Overview

```text
┌────────────────────────────────────────────────────────────────────┐
│                          Modern Networking                        │
│                                                                    │
│  Applications → HTTP/gRPC → TLS → TCP/UDP → IP → Ethernet/Wi-Fi  │
│                                                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │
│  │ Name Service │  │ Routing      │  │ Traffic Management       │ │
│  │ DNS, caches  │  │ Gateways,    │  │ Reverse proxies, LB, CDN │ │
│  │ service disc │  │ BGP, NAT     │  │ retries, rate limiting   │ │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘ │
│                                                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │
│  │ Security     │  │ Observability│  │ Operations               │ │
│  │ Firewalls,   │  │ Logs, traces,│  │ Capacity, resilience,    │ │
│  │ ACLs, zero   │  │ packet capture│ │ failover, troubleshooting│ │
│  │ trust        │  │ traceroute   │  │                          │ │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘ │
└────────────────────────────────────────────────────────────────────┘
```

## 🔑 Key Concepts

- **Packet** — A unit of data transmitted over a network.
- **Latency** — How long it takes data to travel from source to destination.
- **Bandwidth** — Maximum amount of data that can be transferred over time.
- **CIDR** — Classless Inter-Domain Routing notation used to describe IP ranges.
- **DNS** — System that maps names like `api.example.com` to IP addresses.
- **TCP** — Reliable, ordered transport protocol used by most web traffic.
- **UDP** — Low-overhead transport protocol used where latency matters more than guaranteed delivery.
- **TLS** — Cryptographic protocol that secures traffic in transit.
- **Load Balancer** — Component that distributes traffic across multiple healthy backends.
- **Firewall** — Policy enforcement point that allows or denies traffic flows.

## 📋 Topics Covered

- **Fundamentals** — Packets, protocols, ports, latency, throughput, and reliability
- **Protocol Layers** — OSI model, TCP/IP, encapsulation, and troubleshooting by layer
- **Addressing** — IPv4, IPv6, subnetting, CIDR, gateways, and NAT
- **Traffic Flow** — Switching, routing, VLANs, routing tables, and path selection
- **Name Resolution** — DNS records, recursive resolvers, caching, and TTL trade-offs
- **Application Networking** — HTTP, HTTPS, TLS, keep-alive, proxies, and connection pooling
- **Traffic Management** — Load balancers, CDNs, retries, rate limiting, and failover
- **Security** — Segmentation, ACLs, zero trust, bastions, and secure remote access
- **Operations** — Packet capture, traceroute, latency analysis, and incident playbooks

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to networking?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md) and then [01-OSI-AND-TCP-IP.md](01-OSI-AND-TCP-IP.md)

**Working on production systems?** → Review [07-NETWORK-SECURITY.md](07-NETWORK-SECURITY.md), [08-OBSERVABILITY-AND-TROUBLESHOOTING.md](08-OBSERVABILITY-AND-TROUBLESHOOTING.md), and [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md)

**Want a structured plan?** → Follow [LEARNING-PATH.md](LEARNING-PATH.md)
