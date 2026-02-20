# Rate Limiting and Throttling

## Table of Contents

1. [Overview](#overview)
2. [Why Rate Limiting Matters](#why-rate-limiting-matters)
3. [Common Algorithms](#common-algorithms)
4. [Implementation Patterns](#implementation-patterns)
5. [HTTP Headers](#http-headers)
6. [Backpressure](#backpressure)
7. [Best Practices](#best-practices)
8. [Next Steps](#next-steps)
9. [Version History](#version-history)

---

## Overview

Rate limiting controls how many requests a client can make to an API within a given time window. Throttling is the mechanism that enforces those limits — typically by rejecting, queuing, or slowing down excess requests. Together, they protect APIs from abuse, ensure fair usage, and maintain service stability.

### Why Rate Limiting Matters

| Concern | How Rate Limiting Helps |
|---|---|
| **Availability** | Prevents a single client from overwhelming shared infrastructure |
| **Fair usage** | Ensures all consumers get equitable access to resources |
| **Cost control** | Limits compute and bandwidth spend caused by runaway clients |
| **Security** | Mitigates brute-force attacks, credential stuffing, and scraping |
| **Compliance** | Enforces contractual usage tiers (free, pro, enterprise) |

---

## Common Algorithms

### 1. Fixed Window

Counts requests in fixed time intervals (e.g., 100 requests per minute). Simple but suffers from burst problems at window boundaries.

```
Window 1 (00:00–00:59)     Window 2 (01:00–01:59)
├─────────────────────┤    ├─────────────────────┤
│ 100 requests allowed│    │ 100 requests allowed│
└─────────────────────┘    └─────────────────────┘

Problem: 100 requests at 00:59 + 100 at 01:00 = 200 in 2 seconds
```

### 2. Sliding Window Log

Tracks timestamps of all requests. Counts requests within a rolling window from the current time. Accurate but memory-intensive.

### 3. Sliding Window Counter

Hybrid of fixed window and sliding window. Weights the previous window's count by overlap percentage. Good balance of accuracy and efficiency.

```
Previous window count: 80 (40% overlap with current time)
Current window count:  30

Weighted count = (80 × 0.4) + 30 = 62
```

### 4. Token Bucket

A bucket holds tokens up to a maximum capacity. Each request consumes one token. Tokens are added at a fixed rate. Allows short bursts while enforcing average rate.

```
Bucket capacity: 10 tokens
Refill rate: 2 tokens/second

Time 0s:  ●●●●●●●●●● (10 tokens — full)
Burst:    ●●●●●●●●●● → 10 requests served instantly
Time 1s:  ●●         (2 tokens refilled)
Time 2s:  ●●●●       (2 more refilled)
```

### 5. Leaky Bucket

Requests enter a queue (bucket) and are processed at a fixed rate. Excess requests overflow and are rejected. Smooths traffic to a constant output rate.

### Algorithm Comparison

| Algorithm | Burst Handling | Memory | Accuracy | Complexity |
|---|---|---|---|---|
| Fixed Window | Poor (boundary burst) | Low | Low | Simple |
| Sliding Window Log | Good | High | High | Medium |
| Sliding Window Counter | Good | Low | Medium | Medium |
| Token Bucket | Good (controlled burst) | Low | High | Medium |
| Leaky Bucket | None (smoothed) | Low | High | Simple |

---

## Implementation Patterns

### API Gateway Rate Limiting

Apply rate limits at the API gateway layer before requests reach backend services.

```yaml
# Example: Kong rate-limiting plugin
plugins:
  - name: rate-limiting
    config:
      minute: 100
      hour: 5000
      policy: redis
      redis_host: redis.internal
      redis_port: 6379
```

### Application-Level Rate Limiting

Implement in application code using middleware for fine-grained control.

```python
# Python example with a simple token bucket
import time
from threading import Lock

class TokenBucket:
    def __init__(self, rate: float, capacity: int):
        self.rate = rate          # tokens per second
        self.capacity = capacity  # max burst size
        self.tokens = capacity
        self.last_refill = time.monotonic()
        self.lock = Lock()

    def allow(self) -> bool:
        with self.lock:
            now = time.monotonic()
            elapsed = now - self.last_refill
            self.tokens = min(self.capacity, self.tokens + elapsed * self.rate)
            self.last_refill = now
            if self.tokens >= 1:
                self.tokens -= 1
                return True
            return False
```

### Distributed Rate Limiting

For multi-instance deployments, use a shared store (Redis, Memcached) to track counts across all instances.

```
Client → Load Balancer → Instance A ─┐
                         Instance B ──┼──→ Redis (shared counter)
                         Instance C ─┘
```

### Per-Client vs Global Limits

| Scope | Use Case | Key |
|---|---|---|
| **Per API key** | Fair usage across consumers | `rate:{api_key}` |
| **Per user** | Authenticated user quotas | `rate:{user_id}` |
| **Per IP** | Anonymous/unauthenticated protection | `rate:{client_ip}` |
| **Per endpoint** | Protect expensive operations | `rate:{api_key}:{endpoint}` |
| **Global** | Total system protection | `rate:global` |

---

## HTTP Headers

Standard headers for communicating rate limit status to clients:

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1687200060
Retry-After: 30
```

| Header | Description |
|---|---|
| `X-RateLimit-Limit` | Maximum requests allowed in the window |
| `X-RateLimit-Remaining` | Requests remaining in the current window |
| `X-RateLimit-Reset` | Unix timestamp when the window resets |
| `Retry-After` | Seconds to wait before retrying (on 429 response) |

### 429 Too Many Requests

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1687200060

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "You have exceeded the rate limit of 100 requests per minute.",
    "retry_after": 30
  }
}
```

### IETF RateLimit Header Fields (Draft Standard)

The IETF is standardizing rate limit headers as `RateLimit-Limit`, `RateLimit-Remaining`, and `RateLimit-Reset` (without the `X-` prefix). Adopt these when clients support them.

---

## Backpressure

Backpressure is a flow-control mechanism where a service signals upstream that it is overloaded, causing callers to slow down rather than continue sending at full speed.

### Patterns

| Pattern | Description |
|---|---|
| **HTTP 429 + Retry-After** | Standard API backpressure signal |
| **HTTP 503 + Retry-After** | Service temporarily unavailable |
| **gRPC RESOURCE_EXHAUSTED** | gRPC equivalent of 429 |
| **Queue depth limits** | Reject new messages when queue is full |
| **Load shedding** | Drop low-priority requests under load |
| **Circuit breaker** | Stop calling an overwhelmed downstream service |

### Client-Side Handling

Clients should implement exponential backoff with jitter when receiving 429 or 503:

```python
import random
import time

def request_with_backoff(make_request, max_retries=5):
    for attempt in range(max_retries):
        response = make_request()
        if response.status_code != 429:
            return response

        # Exponential backoff with jitter
        base_delay = min(2 ** attempt, 60)
        jitter = random.uniform(0, base_delay * 0.5)
        delay = base_delay + jitter
        time.sleep(delay)

    raise Exception("Max retries exceeded")
```

---

## Best Practices

1. **Always return rate limit headers** — clients need visibility into their remaining quota
2. **Use 429 status code** — not 403 or 503 — for rate limit responses
3. **Include Retry-After** — tell clients exactly when to retry
4. **Apply limits at the gateway** — catch abuse before it reaches application logic
5. **Use distributed counters** — Redis or similar for multi-instance consistency
6. **Differentiate by tier** — give premium users higher limits
7. **Rate limit by API key, not just IP** — IPs can be shared (NAT, proxies)
8. **Document limits clearly** — publish limits in API documentation and developer portal
9. **Monitor rate limit hits** — track 429 responses to identify clients that need higher limits or are misbehaving
10. **Implement graceful degradation** — return cached or partial data instead of hard rejecting

---

## Next Steps

| File | Topic |
|---|---|
| [07-DOCUMENTATION.md](07-DOCUMENTATION.md) | OpenAPI/Swagger, API documentation tools |
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Comprehensive API design best practices |
| [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md) | Common API design mistakes |
| [05-AUTHENTICATION-AND-AUTHORIZATION.md](05-AUTHENTICATION-AND-AUTHORIZATION.md) | OAuth 2.0, JWT, API keys |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial rate limiting and throttling documentation |
