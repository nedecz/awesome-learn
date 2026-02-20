# API Design Best Practices

## Table of Contents

1. [Overview](#overview)
2. [Naming Conventions](#naming-conventions)
3. [Pagination](#pagination)
4. [Error Handling](#error-handling)
5. [Filtering, Sorting, and Searching](#filtering-sorting-and-searching)
6. [Request and Response Design](#request-and-response-design)
7. [Idempotency](#idempotency)
8. [Caching](#caching)
9. [Security](#security)
10. [Consistency and Standards](#consistency-and-standards)
11. [Checklist](#checklist)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

This document collects the most impactful API design best practices — the patterns that, when applied consistently, result in APIs that are intuitive, reliable, and maintainable. Each practice is actionable and includes examples.

---

## Naming Conventions

### Resource Naming

| Rule | Good ✅ | Bad ❌ |
|---|---|---|
| Use nouns, not verbs | `/orders` | `/getOrders` |
| Use plural nouns | `/users` | `/user` |
| Use lowercase with hyphens | `/order-items` | `/orderItems`, `/order_items` |
| Use hierarchical URIs | `/users/{id}/orders` | `/getUserOrders` |
| Avoid file extensions | `/users/123` | `/users/123.json` |
| Avoid deep nesting (>3 levels) | `/users/{id}/orders` | `/users/{id}/orders/{id}/items/{id}/variants` |

### Field Naming

Use consistent casing in JSON response bodies. **snake_case** is most common in REST APIs:

```json
{
  "order_id": "ord_abc123",
  "created_at": "2025-01-15T14:30:00Z",
  "line_items": [],
  "total_amount": 49.99,
  "is_active": true
}
```

Pick one convention (snake_case or camelCase) and apply it everywhere.

---

## Pagination

### Cursor-Based Pagination (Recommended)

```json
{
  "data": [...],
  "pagination": {
    "has_more": true,
    "next_cursor": "eyJpZCI6MTAwfQ==",
    "previous_cursor": "eyJpZCI6ODF9"
  }
}
```

```http
GET /v1/orders?limit=20&cursor=eyJpZCI6MTAwfQ==
```

### Offset-Based Pagination

```json
{
  "data": [...],
  "pagination": {
    "page": 2,
    "per_page": 20,
    "total": 543
  }
}
```

### Comparison

| Approach | Consistency | Performance | Simplicity |
|---|---|---|---|
| **Cursor-based** | Stable under inserts/deletes | O(1) with indexed cursor | Moderate |
| **Offset-based** | Unstable (rows can shift) | O(n) for large offsets | Simple |
| **Keyset** | Stable | O(1) | Complex |

**Recommendation:** Use cursor-based pagination for production APIs. Use offset-based only for simple admin UIs where consistency matters less.

---

## Error Handling

### Standard Error Format (RFC 7807)

```json
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "The 'quantity' field must be a positive integer.",
  "instance": "/v1/orders",
  "errors": [
    {
      "field": "items[0].quantity",
      "code": "INVALID_VALUE",
      "message": "Must be greater than 0"
    }
  ]
}
```

### Error Response Rules

1. **Always return a JSON body** — even for errors
2. **Use appropriate HTTP status codes** — 400 for client errors, 500 for server errors
3. **Include a machine-readable error code** — `"code": "INSUFFICIENT_FUNDS"` not just a message
4. **Include a human-readable message** — helpful for debugging
5. **Include field-level errors** for validation failures
6. **Never expose stack traces** in production responses
7. **Log a correlation ID** and return it in the response for support debugging

### Common Status Codes

| Code | Meaning | When to Use |
|---|---|---|
| 400 | Bad Request | Malformed request syntax or invalid parameters |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Authenticated but insufficient permissions |
| 404 | Not Found | Resource does not exist |
| 409 | Conflict | State conflict (e.g., duplicate creation) |
| 422 | Unprocessable Entity | Valid syntax but semantically invalid |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Unexpected server failure |
| 503 | Service Unavailable | Temporary downtime or overload |

---

## Filtering, Sorting, and Searching

### Filtering

```http
GET /v1/orders?status=pending&created_after=2025-01-01
GET /v1/products?category=electronics&price_min=10&price_max=100
```

### Sorting

```http
GET /v1/orders?sort=created_at&order=desc
GET /v1/products?sort=-price,name
```

Use `-` prefix for descending (GitHub/Stripe convention) or explicit `order` parameter.

### Searching

```http
GET /v1/products?q=wireless+keyboard
GET /v1/users?search=john@example.com
```

---

## Request and Response Design

### Envelope Pattern

Wrap responses in a consistent envelope:

```json
{
  "data": { ... },
  "meta": {
    "request_id": "req_abc123",
    "timestamp": "2025-01-15T14:30:00Z"
  }
}
```

### Sparse Fieldsets

Allow clients to request only the fields they need:

```http
GET /v1/users/123?fields=id,name,email
```

### Timestamps

Always use ISO 8601 / RFC 3339 format in UTC:

```json
{
  "created_at": "2025-01-15T14:30:00Z",
  "updated_at": "2025-01-15T15:45:00Z"
}
```

### Null vs Absent Fields

Be explicit about your convention:
- **Include nulls** — `"middle_name": null` (Stripe style)
- **Omit absent fields** — field not present in response (smaller payload)

Document which convention you use and apply it consistently.

---

## Idempotency

Idempotent operations produce the same result whether called once or multiple times. This is critical for safe retries.

### Idempotency Keys

```http
POST /v1/orders
Idempotency-Key: ord_req_abc123
Content-Type: application/json

{
  "items": [{"product_id": "prod_1", "quantity": 1}]
}
```

If the same `Idempotency-Key` is sent again, return the same response without creating a duplicate order.

### HTTP Method Idempotency

| Method | Safe | Idempotent | Notes |
|---|---|---|---|
| GET | Yes | Yes | Never modifies state |
| HEAD | Yes | Yes | Like GET without body |
| OPTIONS | Yes | Yes | Returns allowed methods |
| PUT | No | Yes | Full replacement — same result every time |
| DELETE | No | Yes | Deleting twice yields same end state |
| POST | No | **No** | Requires idempotency key for safe retries |
| PATCH | No | **No** | Depends on implementation |

---

## Caching

### HTTP Caching Headers

```http
HTTP/1.1 200 OK
Cache-Control: public, max-age=300
ETag: "abc123"
Last-Modified: Wed, 15 Jan 2025 14:30:00 GMT
Vary: Accept, Authorization
```

### Conditional Requests

```http
# Client sends cached ETag
GET /v1/products/123
If-None-Match: "abc123"

# Server returns 304 if unchanged
HTTP/1.1 304 Not Modified
```

### When to Cache

| Resource Type | Cache Strategy |
|---|---|
| Static reference data | Long TTL (hours/days) |
| User-specific data | Private, short TTL or no-cache |
| Search results | Short TTL (seconds/minutes) |
| Real-time data | No cache |

---

## Security

1. **Always use HTTPS** — never expose APIs over plain HTTP
2. **Authenticate every request** — use OAuth 2.0 Bearer tokens or API keys
3. **Validate all input** — reject unexpected fields, enforce types and ranges
4. **Rate limit all endpoints** — protect against abuse and DDoS
5. **Use least privilege** — grant only the scopes/permissions needed
6. **Sanitize output** — prevent XSS in any HTML-rendering context
7. **Log security events** — authentication failures, permission denials, rate limit hits
8. **Use correlation IDs** — include a request ID in every response for auditing

---

## Consistency and Standards

1. **Use a style guide** — document and enforce naming, error format, and pagination conventions
2. **Validate against OpenAPI spec** — catch schema drift in CI
3. **Version from day one** — even if you only have v1
4. **Be consistent across endpoints** — if `/orders` uses cursor pagination, `/products` should too
5. **Follow existing conventions** — use patterns established by Stripe, GitHub, or Twilio as a baseline
6. **Review API design before implementation** — API design reviews prevent expensive changes later

---

## Checklist

Use this checklist before releasing a new API or endpoint:

- [ ] Resources use plural nouns (`/orders`, not `/order`)
- [ ] HTTP methods used correctly (GET reads, POST creates, PUT replaces, PATCH updates, DELETE removes)
- [ ] Appropriate HTTP status codes for all responses
- [ ] Consistent error format with machine-readable codes
- [ ] Pagination on all list endpoints
- [ ] Rate limiting configured and documented
- [ ] Authentication required on all non-public endpoints
- [ ] Input validation on all parameters and request bodies
- [ ] API version in URL or header
- [ ] OpenAPI spec updated and validated
- [ ] Documentation updated with examples
- [ ] Breaking change review completed

---

## Next Steps

| File | Topic |
|---|---|
| [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md) | Common API design mistakes |
| [01-REST.md](01-REST.md) | RESTful API design in depth |
| [07-DOCUMENTATION.md](07-DOCUMENTATION.md) | OpenAPI and documentation tools |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Structured learning guide |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial API design best practices |
