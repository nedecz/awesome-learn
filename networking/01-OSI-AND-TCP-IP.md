# OSI and TCP/IP

## Table of Contents

1. [Overview](#overview)
2. [Why Layering Exists](#why-layering-exists)
3. [OSI Model](#osi-model)
4. [TCP/IP Model](#tcpip-model)
5. [Encapsulation](#encapsulation)
6. [Practical Mapping](#practical-mapping)
7. [Next Steps](#next-steps)
8. [Version History](#version-history)

---

## Overview

Network layering separates responsibilities so that protocols can evolve independently. Application engineers rarely need to think about Ethernet framing, but understanding where each concern lives helps you troubleshoot quickly.

---

## Why Layering Exists

Layering provides:

- **Abstraction** — applications use sockets without reimplementing routing
- **Interoperability** — standards allow equipment and software from different vendors to work together
- **Troubleshooting boundaries** — you can ask whether the problem is at the application, transport, network, or link level

---

## OSI Model

| Layer | Purpose | Examples |
|---|---|---|
| 7. Application | User-facing application protocols | HTTP, DNS, SMTP |
| 6. Presentation | Data representation | TLS, compression, serialization |
| 5. Session | Session coordination | RPC session concepts, authentication state |
| 4. Transport | End-to-end delivery | TCP, UDP |
| 3. Network | Addressing and routing | IP, ICMP |
| 2. Data Link | Local delivery on a segment | Ethernet, Wi-Fi, VLAN tagging |
| 1. Physical | Electrical/optical/radio transport | Fiber, copper, radio |

The OSI model is primarily a teaching tool. Real-world systems often follow the simpler TCP/IP model.

---

## TCP/IP Model

| Layer | Purpose | Common Examples |
|---|---|---|
| Application | Protocols applications use directly | HTTP, DNS, SSH, TLS |
| Transport | End-to-end communication | TCP, UDP |
| Internet | Addressing and routing | IPv4, IPv6, ICMP |
| Link | Local network delivery | Ethernet, Wi-Fi, ARP |

---

## Encapsulation

Each layer adds its own header as data moves down the stack.

```text
Application Data
    ↓
[TCP Header][Application Data]
    ↓
[IP Header][TCP Header][Application Data]
    ↓
[Ethernet Header][IP Header][TCP Header][Application Data][FCS]
```

On the receiving side, the process is reversed. This is why packet captures expose different views depending on which layer you inspect.

---

## Practical Mapping

| Problem | Likely Layer |
|---|---|
| HTTP 503 from a gateway | Application / transport |
| TLS handshake failure | Presentation / application |
| Connection timeout | Transport / network |
| Host unreachable | Network |
| ARP resolution failure on LAN | Link |
| Cable unplugged or interface down | Physical |

A useful troubleshooting habit is to move from lower layers upward:

1. Is the interface up?
2. Does the host have an IP address?
3. Is there a route to the destination?
4. Can the destination port be reached?
5. Is the application healthy and returning correct responses?

---

## Next Steps

Move to [02-IP-ADDRESSING-AND-SUBNETTING.md](02-IP-ADDRESSING-AND-SUBNETTING.md) to learn the most common addressing concepts used in cloud and production systems.

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial version |
