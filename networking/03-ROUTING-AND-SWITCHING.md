# Routing and Switching

## Table of Contents

1. [Overview](#overview)
2. [Switching](#switching)
3. [Routing](#routing)
4. [Routing Tables and Next Hops](#routing-tables-and-next-hops)
5. [Static vs Dynamic Routing](#static-vs-dynamic-routing)
6. [Common Cloud Equivalents](#common-cloud-equivalents)
7. [Failure Modes](#failure-modes)
8. [Next Steps](#next-steps)
9. [Version History](#version-history)

---

## Overview

Traffic within a local network is usually switched. Traffic between networks is routed. Engineers do not need to configure every router by hand to benefit from understanding this distinction; they just need to know which device is responsible for forwarding each hop.

---

## Switching

Switches forward frames inside a local network using MAC addresses.

Key ideas:
- Operate primarily at layer 2
- Learn which MAC addresses live on which interfaces
- Keep local traffic local when possible
- Often participate in VLAN segmentation

In modern cloud systems, you rarely manage physical switches directly, but the same concepts appear in virtual networks and overlay systems.

---

## Routing

Routers move packets between different IP networks.

Responsibilities include:
- Choosing a next hop using a routing table
- Enforcing boundary policies
- Connecting subnets, VPCs, sites, and internet edges
- Supporting redundancy and path selection

Without routing, isolated subnets remain isolated forever.

---

## Routing Tables and Next Hops

A routing table answers: *Where should this packet go next?*

Example:

| Destination | Next Hop | Meaning |
|------------|----------|---------|
| `10.0.1.0/24` | local | directly connected subnet |
| `10.0.2.0/24` | `10.0.1.1` | send to gateway for that subnet |
| `0.0.0.0/0` | `10.0.1.254` | default route for everything else |

The most specific route usually wins. A `/24` route is preferred over a `/16` route when both match.

---

## Static vs Dynamic Routing

### Static Routing

- Manually configured
- Predictable and simple for small environments
- Hard to scale across many paths or changing topologies

### Dynamic Routing

Protocols such as OSPF or BGP exchange path information automatically.

Advantages:
- Better scaling
- Faster adaptation to change
- More resilient multi-path designs

Trade-offs:
- Greater operational complexity
- Requires stronger understanding of policy, convergence, and failure behavior

---

## Common Cloud Equivalents

Cloud environments express routing using managed constructs:

- **VPC/VNet route tables** decide where subnet traffic goes
- **Internet gateways** provide public internet connectivity
- **NAT gateways** provide egress for private subnets
- **Transit gateways / virtual WANs** connect many networks together
- **Load balancers** and **service meshes** affect application traffic flow at higher layers

Cloud networking hides hardware, not the need for routing literacy.

---

## Failure Modes

- Missing route to a remote CIDR
- Wrong next hop causing black holes
- Asymmetric routing that breaks stateful firewalls
- Route overlap that sends traffic somewhere unexpected
- Default route changes that unintentionally bypass security inspection

A common debugging pattern is to compare the expected path with the actual path from routing tables, traceroute output, and firewall logs.

---

## Next Steps

- Continue with [04-DNS-AND-NAME-RESOLUTION.md](04-DNS-AND-NAME-RESOLUTION.md)
- Review [06-LOAD-BALANCING-AND-TRAFFIC-MANAGEMENT.md](06-LOAD-BALANCING-AND-TRAFFIC-MANAGEMENT.md) to see how traffic is intentionally distributed once routes are working

---

## Version History

| Version | Date | Changes |
|--------|------|---------|
| 1.0 | 2026 | Initial routing and switching guide |
