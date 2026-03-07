# Routing and Switching

## Table of Contents

1. [Overview](#overview)
2. [Switching Basics](#switching-basics)
3. [Routing Basics](#routing-basics)
4. [Default Gateways and Route Tables](#default-gateways-and-route-tables)
5. [VLANs and Segmentation](#vlans-and-segmentation)
6. [Dynamic Routing Concepts](#dynamic-routing-concepts)
7. [Next Steps](#next-steps)
8. [Version History](#version-history)

---

## Overview

Switches move frames within a local network. Routers move packets between networks. In cloud environments these functions are often virtualised, but the underlying ideas are the same.

---

## Switching Basics

A switch learns which MAC addresses live behind which ports and forwards Ethernet frames efficiently inside the same Layer 2 domain.

Important ideas:

- Broadcast traffic stays inside the broadcast domain unless routed
- Loops at Layer 2 are dangerous and are controlled with protocols like STP in traditional environments
- Large flat Layer 2 networks increase blast radius and operational complexity

---

## Routing Basics

Routers forward packets based on destination IP addresses and route tables.

```text
Host A → default gateway → router → next hop → destination subnet → Host B
```

Routing decisions consider:

- Destination prefix length (longest prefix match wins)
- Route source (connected, static, or dynamic)
- Administrative preference / cost

---

## Default Gateways and Route Tables

Every host typically needs:

- A local IP address and subnet mask
- A default gateway for non-local destinations
- DNS configuration for name resolution

If a destination is not inside the local subnet, the host sends traffic to its default gateway.

| Route Type | Example | Use |
|---|---|---|
| Connected | `10.40.10.0/24 dev eth0` | Reach directly attached subnet |
| Default | `0.0.0.0/0 via 10.40.10.1` | Send unknown destinations to gateway |
| Static | `10.50.0.0/16 via 10.40.10.254` | Predictable, explicit paths |
| Dynamic | Learned via BGP/OSPF | Larger or changing networks |

---

## VLANs and Segmentation

VLANs logically separate traffic on shared switching infrastructure.

Benefits:

- Reduce broadcast scope
- Separate workloads by environment or trust level
- Support policy enforcement between segments

In cloud networking, subnets and security groups often play the role that VLANs and ACLs play on-premise.

---

## Dynamic Routing Concepts

You do not need to master every routing protocol to be effective, but these ideas matter:

- **OSPF / IS-IS** — common internal routing protocols
- **BGP** — policy-driven routing used for internet exchange, cloud edge, and data center fabrics
- **ECMP** — equal-cost multi-path routing for distributing traffic across parallel paths
- **Failover** — route convergence determines how quickly traffic switches after a path failure

---

## Next Steps

Read [04-DNS-AND-NAME-RESOLUTION.md](04-DNS-AND-NAME-RESOLUTION.md) to understand why services may be reachable by IP but still fail for users and applications.

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial version |
