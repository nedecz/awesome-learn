# OSI and TCP/IP

## Table of Contents

1. [Overview](#overview)
2. [Why Layering Helps](#why-layering-helps)
3. [OSI Model](#osi-model)
4. [TCP/IP Model](#tcpip-model)
5. [Mapping Common Technologies to Layers](#mapping-common-technologies-to-layers)
6. [Encapsulation](#encapsulation)
7. [Troubleshooting by Layer](#troubleshooting-by-layer)
8. [Next Steps](#next-steps)
9. [Version History](#version-history)

---

## Overview

The OSI and TCP/IP models are not perfect descriptions of reality, but they are extremely useful thinking tools. They help engineers separate concerns, communicate clearly during incidents, and isolate failures faster.

---

## Why Layering Helps

Layering works because each layer solves a specific problem while depending on the services provided by lower layers.

- Physical and link layers move frames on a medium
- Network layers move packets between networks
- Transport layers manage end-to-end delivery semantics
- Application layers define what the data means to users and services

If you know where a component lives, you know which questions to ask first.

---

## OSI Model

| Layer | Purpose | Examples |
|------|---------|----------|
| 7. Application | User-facing protocols and semantics | HTTP, DNS, SMTP, gRPC |
| 6. Presentation | Format and encoding concerns | TLS, compression, serialization |
| 5. Session | Session management | RPC sessions, connection state |
| 4. Transport | End-to-end communication | TCP, UDP, QUIC |
| 3. Network | Routing between networks | IP, ICMP |
| 2. Data Link | Local network delivery | Ethernet, Wi-Fi, ARP |
| 1. Physical | Signals on the medium | Copper, fiber, radio |

The OSI model is often taught for conceptual clarity even though most engineers work more naturally with the simplified TCP/IP model.

---

## TCP/IP Model

| TCP/IP Layer | Comparable OSI Layers | Examples |
|-------------|------------------------|----------|
| Application | OSI 5–7 | HTTP, DNS, TLS, SSH |
| Transport | OSI 4 | TCP, UDP, QUIC |
| Internet | OSI 3 | IPv4, IPv6, ICMP |
| Link | OSI 1–2 | Ethernet, Wi-Fi, ARP |

This model is closer to how real Internet protocols evolved and how most practitioners talk about networks today.

---

## Mapping Common Technologies to Layers

| Technology | Layer | Typical Issue |
|-----------|-------|---------------|
| DNS | Application | Wrong record, stale cache, bad TTL |
| TLS | Application/Presentation | Expired certificate, hostname mismatch |
| HTTP | Application | 4xx/5xx errors, proxy headers, timeouts |
| TCP | Transport | SYN backlog, retransmits, reset packets |
| IP | Network | Missing route, asymmetric path, fragmentation |
| Ethernet / Wi-Fi | Link | Duplex mismatch, signal degradation |

A browser showing `ERR_CERT_COMMON_NAME_INVALID` is not a layer 3 routing problem. A missing default gateway is not an HTTP problem. Layering keeps investigations focused.

---

## Encapsulation

Data is wrapped as it moves down the stack:

```text
Application data
  ↓
Transport segment (TCP/UDP headers added)
  ↓
IP packet (source/destination IP added)
  ↓
Link frame (MAC addresses and frame checks added)
```

The reverse happens at the receiver. This explains why packet captures can reveal headers from multiple layers and why one broken layer can prevent all higher-level communication.

---

## Troubleshooting by Layer

A useful order of operations:

1. **Application layer** — Is the hostname correct? Are responses valid?
2. **Transport layer** — Does the TCP or QUIC handshake succeed?
3. **Network layer** — Is the destination IP reachable? Is routing correct?
4. **Link/physical layer** — Is there local connectivity or interface trouble?

Example workflow:

```text
App error → check DNS → check TCP connection → check route → inspect firewall → inspect packet capture
```

This prevents wasting time on application code when packets never reach the server.

---

## Next Steps

- Learn how hosts and ranges are addressed in [02-IP-ADDRESSING-AND-SUBNETTING.md](02-IP-ADDRESSING-AND-SUBNETTING.md)
- Then study path selection in [03-ROUTING-AND-SWITCHING.md](03-ROUTING-AND-SWITCHING.md)

---

## Version History

| Version | Date | Changes |
|--------|------|---------|
| 1.0 | 2026 | Initial OSI and TCP/IP guide |
