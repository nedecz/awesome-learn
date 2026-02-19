# Microservices Resilience Patterns

## Table of Contents

1. [Overview](#overview)
2. [Circuit Breaker](#circuit-breaker)
3. [Retry Pattern](#retry-pattern)
4. [Timeout Pattern](#timeout-pattern)
5. [Bulkhead Pattern](#bulkhead-pattern)
6. [Fallback Pattern](#fallback-pattern)
7. [Rate Limiting](#rate-limiting)
8. [Health Checks and Self-Healing](#health-checks-and-self-healing)
9. [Combining Patterns](#combining-patterns)
10. [Best Practices](#best-practices)
11. [Next Steps](#next-steps)

## Overview

In a microservices architecture, failures are inevitable. Services go down, networks partition, and latency spikes occur. Resilience patterns help services handle failures gracefully without cascading failures across the entire system.

### Target Audience

- Developers building fault-tolerant microservices
- SREs designing reliability strategies
- Architects selecting resilience patterns for distributed systems

### Scope

- Circuit breaker pattern and state management
- Retry strategies with exponential backoff
- Timeout configuration and propagation
- Bulkhead isolation for fault containment
- Fallback strategies and graceful degradation
- Rate limiting for self-protection

## Circuit Breaker

The circuit breaker prevents a service from repeatedly calling a failing dependency. It monitors failures and "trips" (opens) when the failure rate exceeds a threshold, stopping further calls until the dependency recovers.

### State Machine

```
                    ┌────────────────────────┐
                    │                        │
         Success   │   Failure threshold    │
         counter   │   exceeded             │
         reset     │                        ▼
                   │                  ┌───────────┐
┌──────────┐      │                  │   OPEN    │
│  CLOSED  │──────┘                  │           │
│          │                         │  All calls│
│ Calls    │                         │  fail fast│
│ pass     │                         │  (no call │
│ through  │                         │  to dep)  │
└──────────┘                         └─────┬─────┘
      ▲                                    │
      │                              Timer expires
      │ Success                            │
      │ (trial call)                       ▼
      │                          ┌──────────────┐
      └──────────────────────────│  HALF-OPEN   │
                                 │              │
                                 │ Allow limited│
                                 │ trial calls  │
                                 └──────┬───────┘
                                        │
                                   Failure
                                   (back to OPEN)
```

### Configuration Parameters

| Parameter | Description | Typical Value |
|---|---|---|
| **Failure threshold** | Number of failures before opening | 5 failures |
| **Success threshold** | Successes in half-open before closing | 3 successes |
| **Timeout** | How long the circuit stays open | 30 seconds |
| **Monitoring window** | Time window for counting failures | 60 seconds |

### Implementation Libraries

| Language | Library | Features |
|---|---|---|
| **Java** | Resilience4j | Circuit breaker, retry, rate limiter, bulkhead |
| **.NET** | Polly | Circuit breaker, retry, timeout, fallback |
| **Go** | sony/gobreaker | Circuit breaker with customizable settings |
| **Node.js** | opossum | Circuit breaker with event emitters |
| **Python** | pybreaker | Circuit breaker with Redis-backed state |

## Retry Pattern

Retries automatically re-attempt failed operations. They are effective for transient failures like network blips or temporary service unavailability.

### Retry Strategies

```
Fixed Interval:
  Attempt 1 ──X── wait 1s ── Attempt 2 ──X── wait 1s ── Attempt 3 ──✓

Exponential Backoff:
  Attempt 1 ──X── wait 1s ── Attempt 2 ──X── wait 2s ── Attempt 3 ──X── wait 4s ── Attempt 4 ──✓

Exponential Backoff with Jitter:
  Attempt 1 ──X── wait 1.2s ── Attempt 2 ──X── wait 2.7s ── Attempt 3 ──✓
  (random jitter prevents thundering herd)
```

### When to Retry

| Retry ✅ | Do NOT Retry ❌ |
|---|---|
| Network timeout (connection reset) | Authentication failure (401, 403) |
| HTTP 503 (Service Unavailable) | Validation error (400) |
| HTTP 429 (Too Many Requests) | Not found (404) |
| Database connection timeout | Business logic error |
| Transient DNS failure | HTTP 409 (Conflict — requires user action) |

### Configuration

| Parameter | Description | Typical Value |
|---|---|---|
| **Max attempts** | Maximum number of retry attempts | 3–5 |
| **Initial delay** | Wait time before first retry | 100ms–1s |
| **Max delay** | Upper bound for backoff | 30s–60s |
| **Backoff multiplier** | Factor for exponential backoff | 2 |
| **Jitter** | Random offset to prevent thundering herd | 0–100% of delay |

## Timeout Pattern

Timeouts prevent a service from waiting indefinitely for a response from a dependency. Every external call should have a timeout.

### Timeout Types

```
┌──────────────┐                      ┌──────────────┐
│ Order        │                      │ Payment      │
│ Service      │                      │ Service      │
│              │                      │              │
│ Connection   │   ┌──────────────┐   │              │
│ Timeout ─────│──▶│  Establish   │──▶│              │
│ (e.g., 5s)   │   │  TCP conn    │   │              │
│              │   └──────────────┘   │              │
│              │                      │              │
│ Read         │   ┌──────────────┐   │              │
│ Timeout ─────│──▶│  Wait for    │──▶│  Processing  │
│ (e.g., 10s)  │   │  response    │   │              │
│              │   └──────────────┘   │              │
└──────────────┘                      └──────────────┘
```

### Timeout Propagation (Deadline Propagation)

When Service A calls Service B, and B calls C, the timeout should propagate so the total time does not exceed the client's expectation.

```
Client timeout: 10s

  Client ──▶ Service A (budget: 10s)
                │
                └──▶ Service B (budget: 8s — minus A's processing time)
                        │
                        └──▶ Service C (budget: 5s — minus B's processing time)
```

### Guidelines

- ✅ Set connection timeouts shorter than read timeouts
- ✅ Propagate timeout budgets downstream
- ✅ Return a meaningful error when a timeout occurs (HTTP 504 Gateway Timeout)
- ❌ Do not set timeouts too high — this ties up resources and connections
- ❌ Do not use infinite timeouts — always set an upper bound

## Bulkhead Pattern

The bulkhead pattern isolates failures by partitioning resources. If one part of the system fails, it does not consume all available resources and crash the entire service.

### Types of Bulkheads

```
Thread Pool Bulkhead:

  ┌───────────────────────────────────────────┐
  │             Order Service                  │
  │                                            │
  │  ┌─────────────────┐ ┌──────────────────┐ │
  │  │ Payment Pool    │ │ Inventory Pool   │ │
  │  │ (10 threads)    │ │ (10 threads)     │ │
  │  │                 │ │                  │ │
  │  │ If payment svc  │ │ Inventory calls  │ │
  │  │ is slow, only   │ │ are unaffected   │ │
  │  │ these 10 threads│ │                  │ │
  │  │ are blocked     │ │                  │ │
  │  └─────────────────┘ └──────────────────┘ │
  └───────────────────────────────────────────┘

Semaphore Bulkhead:

  Max concurrent calls to Payment Service: 10
  Max concurrent calls to Inventory Service: 10

  If Payment reaches 10 concurrent calls → reject new calls immediately
  Inventory continues to accept calls normally
```

### When to Use

- ✅ A service calls multiple downstream dependencies
- ✅ One slow dependency could exhaust all available connections/threads
- ✅ Different dependencies have different reliability profiles
- ❌ A service calls only one downstream dependency (no isolation needed)

## Fallback Pattern

When a service call fails, the fallback pattern provides an alternative response instead of propagating the error.

### Fallback Strategies

| Strategy | Description | Example |
|---|---|---|
| **Cached response** | Return the last known good response | Product catalog returns cached prices |
| **Default value** | Return a safe default | Recommendation service returns popular items |
| **Degraded response** | Return partial data | User profile without the avatar URL |
| **Alternative service** | Call a backup or secondary provider | Switch from primary to backup payment gateway |
| **Queue for retry** | Accept the request and process it later | Order accepted, payment retried asynchronously |

```
┌──────────────┐     ┌──────────────┐
│ Order        │────▶│ Payment      │  ← Fails (circuit open)
│ Service      │     │ Service      │
└──────┬───────┘     └──────────────┘
       │
       │ Fallback
       ▼
┌──────────────┐
│ Queue order  │     Process payment when Payment Service recovers
│ for retry    │
└──────────────┘
```

## Rate Limiting

Rate limiting controls the number of requests a service accepts within a given time window. It protects services from being overwhelmed by excessive traffic.

### Algorithms

| Algorithm | Description | Use Case |
|---|---|---|
| **Fixed window** | Count requests in fixed time intervals (e.g., 100/min) | Simple rate limiting |
| **Sliding window** | Smooths the fixed window boundary problem | More accurate rate limiting |
| **Token bucket** | Tokens are added at a fixed rate; each request consumes a token | Allows bursts up to bucket size |
| **Leaky bucket** | Requests are processed at a fixed rate; excess requests are queued | Smooth output rate |

### Response Headers

```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1705312800
```

## Health Checks and Self-Healing

### Health Check Types

| Type | Question | Failure Response |
|---|---|---|
| **Liveness** | Is the process alive? | Restart the container |
| **Readiness** | Can it handle requests? | Remove from load balancer |
| **Startup** | Has initialization completed? | Wait (don't restart prematurely) |

### Self-Healing Mechanisms

- **Container orchestration** — Kubernetes restarts failed containers automatically
- **Auto-scaling** — Scale up when healthy instances are under pressure
- **Automatic failover** — Switch to a standby instance or replica
- **Queue draining** — Unhealthy consumers stop accepting messages; healthy ones take over

## Combining Patterns

In practice, resilience patterns are layered together for comprehensive fault tolerance.

```
Request Flow with Layered Resilience:

  Client Request
       │
       ▼
  ┌─────────────┐
  │ Rate Limiter│ ── Rejects excess traffic (429)
  └──────┬──────┘
         ▼
  ┌─────────────┐
  │  Timeout    │ ── Limits wait time
  └──────┬──────┘
         ▼
  ┌─────────────┐
  │  Circuit    │ ── Fails fast if dependency is down
  │  Breaker    │
  └──────┬──────┘
         ▼
  ┌─────────────┐
  │  Bulkhead   │ ── Isolates resources per dependency
  └──────┬──────┘
         ▼
  ┌─────────────┐
  │  Retry      │ ── Retries transient failures
  └──────┬──────┘
         ▼
  ┌─────────────┐
  │  Fallback   │ ── Returns alternative if all retries fail
  └─────────────┘
```

## Best Practices

- ✅ Always set timeouts on every external call (HTTP, database, message broker)
- ✅ Use circuit breakers for all synchronous inter-service calls
- ✅ Combine retries with exponential backoff and jitter
- ✅ Use bulkheads to isolate calls to different downstream services
- ✅ Implement fallbacks for non-critical functionality
- ✅ Monitor circuit breaker state transitions and retry metrics
- ❌ Do not retry non-idempotent operations without careful consideration
- ❌ Do not set retry limits too high — this can amplify failures (retry storm)
- ❌ Do not ignore timeout propagation in multi-hop call chains

## Next Steps

Continue to [Security](07-SECURITY.md) to learn about zero-trust networking, OAuth2/OIDC authentication, mutual TLS, API security, and secrets management in microservices.
