# Networking Learning Resources

A practical guide to computer networking — from packets, IP addressing, routing, and DNS to HTTP/TLS, load balancing, network security, and production troubleshooting.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | Networking fundamentals, packet flow, core protocols, and why networking matters | **Start here** |
| [01-OSI-AND-TCP-IP](01-OSI-AND-TCP-IP.md) | OSI model, TCP/IP model, encapsulation, and protocol layering | When building networking fundamentals |
| [02-IP-ADDRESSING-AND-SUBNETTING](02-IP-ADDRESSING-AND-SUBNETTING.md) | IPv4/IPv6, CIDR, subnetting, NAT, and address planning | When working with hosts, VPCs, or Kubernetes networking |
| [03-ROUTING-AND-SWITCHING](03-ROUTING-AND-SWITCHING.md) | Switching, VLANs, routing tables, gateways, and common routing protocols | When designing or operating networks |
| [04-DNS-AND-NAME-RESOLUTION](04-DNS-AND-NAME-RESOLUTION.md) | DNS hierarchy, records, caching, and service discovery basics | When debugging connectivity or service resolution |
| [05-HTTP-TLS-AND-LOAD-BALANCING](05-HTTP-TLS-AND-LOAD-BALANCING.md) | HTTP, TLS handshakes, reverse proxies, and load balancing patterns | When operating web systems and APIs |
| [06-NETWORK-SECURITY](06-NETWORK-SECURITY.md) | Firewalls, segmentation, VPNs, zero trust, and secure network design | When hardening infrastructure |
| [07-TROUBLESHOOTING-AND-OBSERVABILITY](07-TROUBLESHOOTING-AND-OBSERVABILITY.md) | Ping, traceroute, packet capture, flow logs, and practical diagnostics | **Essential — production operations** |
| [08-BEST-PRACTICES](08-BEST-PRACTICES.md) | Design, resilience, and operations checklists for reliable networking | **Essential — production checklist** |
| [09-ANTI-PATTERNS](09-ANTI-PATTERNS.md) | Common networking mistakes and how to avoid them | **Essential — what NOT to do** |
| [LEARNING-PATH](LEARNING-PATH.md) | Structured learning plan with phases, exercises, and milestones | **Start here** after the Overview |
| [README](README.md) | This index file | Start here for navigation |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Learn how packets move through networks
   - Understand the role of IP, TCP, UDP, DNS, and HTTP
   - Build intuition for latency, bandwidth, and reliability

2. **Study Layering** ([01-OSI-AND-TCP-IP](01-OSI-AND-TCP-IP.md))
   - Understand encapsulation and how protocols stack together
   - Learn where Ethernet, IP, TCP, TLS, and HTTP fit

3. **Practice Addressing** ([02-IP-ADDRESSING-AND-SUBNETTING](02-IP-ADDRESSING-AND-SUBNETTING.md))
   - Learn CIDR notation and subnet masks
   - Understand private ranges, NAT, and IPv6 basics

4. **Follow the Learning Path** ([LEARNING-PATH](LEARNING-PATH.md))
   - Structured progression from theory to troubleshooting
   - Exercises for local, cloud, and production-style scenarios

### For Experienced Engineers

1. **Review Troubleshooting** ([07-TROUBLESHOOTING-AND-OBSERVABILITY](07-TROUBLESHOOTING-AND-OBSERVABILITY.md))
   - Build a systematic workflow for incidents
   - Use packet captures, flow logs, and latency metrics effectively

2. **Review Best Practices** ([08-BEST-PRACTICES](08-BEST-PRACTICES.md))
   - Resilience, segmentation, capacity planning, and documentation standards

3. **Audit Against Anti-Patterns** ([09-ANTI-PATTERNS](09-ANTI-PATTERNS.md))
   - Spot common causes of outages, timeouts, and fragile network designs

4. **Harden the Network** ([06-NETWORK-SECURITY](06-NETWORK-SECURITY.md))
   - Apply layered controls with least privilege and secure defaults

## 🏗️ Architecture Overview

```text
Client Device
    |
    | DNS lookup → finds service IP / load balancer
    v
┌────────────────────┐
│ Edge / Load Balancer│  TLS termination, routing, health checks
└─────────┬──────────┘
          |
          v
┌────────────────────┐
│ Reverse Proxy / API│  HTTP routing, retries, rate limits
│ Gateway            │
└─────────┬──────────┘
          |
          v
┌────────────────────┐
│ Application Service│  Business logic
└─────────┬──────────┘
          |
          v
┌────────────────────┐
│ Data / Downstream  │  Databases, queues, caches, third-party APIs
└────────────────────┘

Cross-cutting concerns:
- IP addressing and routing decide how packets reach each hop
- TLS protects traffic in transit
- Firewalls and network policies restrict who can talk to whom
- Observability tools help you diagnose latency, drops, and misroutes
```

## 🔑 Key Concepts

```text
Packet            → Unit of data forwarded across a network
IP Address        → Logical address for identifying a host or interface
Subnet            → Group of IP addresses defined by a CIDR prefix
Routing           → Process of choosing where packets go next
Switching         → Forwarding frames inside a local network segment
DNS               → Maps names like api.example.com to IP addresses
TCP               → Reliable, connection-oriented transport
UDP               → Lightweight, connectionless transport
TLS               → Encryption and identity for traffic in transit
Load Balancer     → Distributes traffic across multiple healthy backends
```

## 📋 Topics Covered

- **Foundations** — packets, latency, bandwidth, ports, sockets, and core protocols
- **Layering** — OSI vs TCP/IP, encapsulation, and where protocols operate
- **Addressing** — IPv4, IPv6, private ranges, CIDR, subnetting, NAT
- **Routing & Switching** — forwarding decisions, VLANs, default gateways, route types
- **DNS** — record types, recursive resolution, TTL, caching, split-horizon patterns
- **Web Networking** — HTTP, TLS, proxies, load balancers, and connection reuse
- **Security** — segmentation, firewalls, VPNs, bastions, zero trust, and east-west controls
- **Operations** — ping, traceroute, tcpdump, logs, metrics, and troubleshooting workflows
- **Best Practices** — documentation, resilience, observability, and capacity planning
- **Anti-Patterns** — flat networks, hard-coded IPs, oversized trust zones, and guess-driven debugging

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to networking?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md) then follow [LEARNING-PATH.md](LEARNING-PATH.md)

**Working in cloud or Kubernetes?** → Focus on [02-IP-ADDRESSING-AND-SUBNETTING.md](02-IP-ADDRESSING-AND-SUBNETTING.md) and [03-ROUTING-AND-SWITCHING.md](03-ROUTING-AND-SWITCHING.md)

**Troubleshooting production issues?** → Review [07-TROUBLESHOOTING-AND-OBSERVABILITY.md](07-TROUBLESHOOTING-AND-OBSERVABILITY.md), [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md), and [09-ANTI-PATTERNS.md](09-ANTI-PATTERNS.md)
