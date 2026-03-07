# HTTP, HTTPS, and TLS

## Table of Contents

1. [Overview](#overview)
2. [HTTP Request Lifecycle](#http-request-lifecycle)
3. [TCP, TLS, and Application Data](#tcp-tls-and-application-data)
4. [Why HTTPS Matters](#why-https-matters)
5. [Connection Reuse and Performance](#connection-reuse-and-performance)
6. [Reverse Proxies and Gateways](#reverse-proxies-and-gateways)
7. [Common Failure Modes](#common-failure-modes)
8. [Next Steps](#next-steps)
9. [Version History](#version-history)

---

## Overview

Modern application networking is often experienced as HTTP requests moving over encrypted transport. Understanding how DNS, TCP, TLS, and HTTP fit together makes API and browser behavior far easier to reason about.

---

## HTTP Request Lifecycle

A typical HTTPS request includes:

1. Resolve the hostname with DNS
2. Open a TCP connection to the destination
3. Negotiate TLS and validate the certificate
4. Send the HTTP request headers and body
5. Receive the response
6. Reuse or close the connection depending on protocol behavior and pool settings

Even a simple `GET /health` depends on several layers working correctly.

---

## TCP, TLS, and Application Data

### TCP

TCP provides ordered, reliable delivery. It is a strong default for APIs and browser traffic because applications rarely want to handle packet ordering or retransmission themselves.

### TLS

TLS secures traffic in transit and verifies that the client is speaking to the intended endpoint.

TLS provides:
- Encryption
- Integrity protection
- Server authentication
- Optional client authentication

### HTTP

HTTP defines application-level semantics such as methods, headers, status codes, content negotiation, and caching.

---

## Why HTTPS Matters

HTTPS is no longer optional for serious systems.

Without TLS:
- Credentials and tokens can be intercepted
- Responses can be modified in transit
- Users cannot trust the identity of the service
- Modern browsers and platforms may block or warn on insecure behavior

Use HTTPS everywhere, including internal systems when feasible.

---

## Connection Reuse and Performance

Repeated handshakes are expensive. Connection reuse improves latency and resource efficiency.

Important concepts:
- **HTTP keep-alive** reduces repeated TCP handshakes
- **HTTP/2 multiplexing** reduces the need for many parallel connections
- **Connection pools** must be sized carefully to avoid exhaustion or stampedes
- **Timeouts** should reflect expected network behavior and retry policy

Application performance problems are often really poor connection-management problems.

---

## Reverse Proxies and Gateways

Common roles:

- TLS termination
- Header normalization and forwarding
- Authentication and authorization enforcement
- Rate limiting and request shaping
- Routing traffic to the appropriate backend service

Examples include NGINX, Envoy, HAProxy, cloud application gateways, and Kubernetes ingress controllers.

---

## Common Failure Modes

- Expired or misissued certificates
- Hostname mismatch between DNS record and certificate SAN
- Proxies dropping large headers or bodies
- Idle timeout mismatches between client, proxy, and backend
- Retries multiplying load when a dependency is already struggling
- Connection pool starvation causing requests to queue before they ever leave the host

---

## Next Steps

- Continue with [06-LOAD-BALANCING-AND-TRAFFIC-MANAGEMENT.md](06-LOAD-BALANCING-AND-TRAFFIC-MANAGEMENT.md)
- Review [07-NETWORK-SECURITY.md](07-NETWORK-SECURITY.md) for policy controls that protect these paths

---

## Version History

| Version | Date | Changes |
|--------|------|---------|
| 1.0 | 2026 | Initial HTTP, HTTPS, and TLS guide |
