# API Design Anti-Patterns

A catalogue of the most common API design mistakes — what they look like, why they are harmful, and exactly how to fix them. Use this document as a design review checklist or a team learning resource.

---

## Table of Contents

- [Introduction](#introduction)
- [Anti-Patterns Summary Table](#anti-patterns-summary-table)
- [1. Chatty APIs](#1-chatty-apis)
- [2. God Endpoint](#2-god-endpoint)
- [3. Ignoring HTTP Semantics](#3-ignoring-http-semantics)
- [4. Inconsistent Naming](#4-inconsistent-naming)
- [5. No Versioning](#5-no-versioning)
- [6. Breaking Changes Without Notice](#6-breaking-changes-without-notice)
- [7. Exposing Internal Implementation](#7-exposing-internal-implementation)
- [8. Missing Pagination](#8-missing-pagination)
- [9. Poor Error Responses](#9-poor-error-responses)
- [10. No Rate Limiting](#10-no-rate-limiting)
- [11. Ignoring Idempotency](#11-ignoring-idempotency)
- [12. Over-Fetching and Under-Fetching](#12-over-fetching-and-under-fetching)
- [Quick Reference Checklist](#quick-reference-checklist)
- [Next Steps](#next-steps)
- [Version History](#version-history)

---

## Introduction

Anti-patterns are recurring practices that seem reasonable initially but create significant problems over time. In API design, mistakes are expensive — once published and consumed, changing an API is far harder than changing internal code.

---

## Anti-Patterns Summary Table

| # | Anti-Pattern | Category | Severity |
|---|---|---|---|
| 1 | Chatty APIs | Performance | 🔴 Critical |
| 2 | God Endpoint | Design | 🔴 Critical |
| 3 | Ignoring HTTP Semantics | Design | 🟠 High |
| 4 | Inconsistent Naming | Design | 🟠 High |
| 5 | No Versioning | Compatibility | 🔴 Critical |
| 6 | Breaking Changes Without Notice | Compatibility | 🔴 Critical |
| 7 | Exposing Internal Implementation | Security / Design | 🟠 High |
| 8 | Missing Pagination | Performance | 🔴 Critical |
| 9 | Poor Error Responses | Developer Experience | 🟠 High |
| 10 | No Rate Limiting | Security | 🔴 Critical |
| 11 | Ignoring Idempotency | Reliability | 🟠 High |
| 12 | Over-Fetching and Under-Fetching | Performance | 🟡 Medium |

---

## 1. Chatty APIs

### Problem

The API requires multiple round trips to accomplish a single logical operation. A mobile client needs 5-10 calls to render a single screen.

### Example

```
# Rendering an order details page requires:
GET /users/123              → user info
GET /orders/456             → order info
GET /orders/456/items       → line items
GET /products/prod_1        → product details for item 1
GET /products/prod_2        → product details for item 2
GET /shipping/ship_789      → shipping info
```

### Fix

Provide composite endpoints or allow field expansion:

```http
GET /orders/456?include=user,items.product,shipping
```

Returns all related data in a single response. Alternatively, consider GraphQL for clients with diverse data needs.

---

## 2. God Endpoint

### Problem

A single endpoint that does everything — create, read, update, delete — based on request body parameters.

### Example

```http
POST /api
{
  "action": "getUser",
  "user_id": "123"
}

POST /api
{
  "action": "deleteOrder",
  "order_id": "456"
}
```

### Fix

Use proper RESTful resource endpoints with HTTP methods:

```http
GET    /users/123
DELETE /orders/456
```

---

## 3. Ignoring HTTP Semantics

### Problem

Using POST for everything, returning 200 for errors, using GET with request bodies.

### Example

```http
# Using POST to read data
POST /getUsers

# Returning 200 with an error in the body
HTTP/1.1 200 OK
{"success": false, "error": "User not found"}
```

### Fix

Use correct HTTP methods and status codes:

```http
GET /users         → 200 with data
POST /users        → 201 with created resource
GET /users/999     → 404 when not found
DELETE /users/123  → 204 on success
```

---

## 4. Inconsistent Naming

### Problem

Mixed naming conventions across endpoints — some use camelCase, others snake_case; some use singular nouns, others plural; some use verbs, others nouns.

### Example

```
GET /Users              ← PascalCase
GET /order-items        ← kebab-case
GET /getProductList     ← verb + camelCase
POST /user/create       ← singular + verb
```

### Fix

Adopt a single convention and enforce it:

```
GET  /users
GET  /order-items
GET  /products
POST /users
```

Document the convention in your API style guide and validate with linting tools.

---

## 5. No Versioning

### Problem

The API has no version identifier. Any change risks breaking existing consumers.

### Fix

Version from day one:

```http
GET /v1/orders
```

See [04-API-VERSIONING.md](04-API-VERSIONING.md) for strategies.

---

## 6. Breaking Changes Without Notice

### Problem

Fields are renamed, removed, or change type without warning. Response formats change. Consumers break unexpectedly.

### Fix

1. Use a deprecation process with timeline
2. Add `Sunset` and `Deprecation` headers
3. Detect breaking changes in CI
4. Communicate via changelog and developer portal

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 01 Mar 2026 00:00:00 GMT
Link: <https://api.example.com/v2/orders>; rel="successor-version"
```

---

## 7. Exposing Internal Implementation

### Problem

API responses leak database column names, internal IDs, stack traces, or implementation details.

### Example

```json
{
  "pk_id": 12345,
  "tbl_users_fk": 67890,
  "hibernate_version": 3,
  "stackTrace": "java.lang.NullPointerException at com.example..."
}
```

### Fix

Map internal models to API response models. Use opaque identifiers:

```json
{
  "id": "usr_abc123",
  "account_id": "acc_xyz789"
}
```

Never return stack traces in production.

---

## 8. Missing Pagination

### Problem

List endpoints return all records. Works fine with 10 items; crashes with 10 million.

### Fix

Always paginate list endpoints with a default and maximum page size:

```http
GET /v1/orders?limit=20&cursor=abc123
```

See the Pagination section in [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md).

---

## 9. Poor Error Responses

### Problem

Error responses provide no useful information — just a status code and a generic message.

### Example

```json
{"error": "Something went wrong"}
```

### Fix

Use structured error responses with machine-readable codes:

```json
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "The 'email' field is required.",
  "errors": [
    {"field": "email", "code": "REQUIRED", "message": "Email is required"}
  ]
}
```

---

## 10. No Rate Limiting

### Problem

No limits on request volume. A single client can monopolize resources, and there is no protection against abuse or DDoS.

### Fix

Implement rate limiting and return standard headers:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
```

See [06-RATE-LIMITING-AND-THROTTLING.md](06-RATE-LIMITING-AND-THROTTLING.md).

---

## 11. Ignoring Idempotency

### Problem

POST requests have no idempotency mechanism. Network retries create duplicate orders, payments, or records.

### Fix

Support idempotency keys on non-idempotent endpoints:

```http
POST /v1/orders
Idempotency-Key: client_req_abc123
```

Store the key and return the cached response on duplicate requests.

---

## 12. Over-Fetching and Under-Fetching

### Problem

- **Over-fetching:** Returning 50 fields when the client needs 3
- **Under-fetching:** Forcing multiple requests to assemble needed data

### Fix

- Support sparse fieldsets: `GET /users/123?fields=id,name,email`
- Support field expansion: `GET /orders/456?include=items,shipping`
- Consider GraphQL for clients with widely varying data needs

---

## Quick Reference Checklist

- [ ] No endpoint requires more than 2-3 calls for a common use case
- [ ] Each resource has its own endpoint (no god endpoint)
- [ ] HTTP methods and status codes used correctly
- [ ] Consistent naming convention across all endpoints
- [ ] API is versioned
- [ ] Breaking changes go through deprecation process
- [ ] No internal implementation details in responses
- [ ] All list endpoints are paginated
- [ ] Error responses include machine-readable codes
- [ ] Rate limiting is configured on all endpoints
- [ ] Non-idempotent operations support idempotency keys
- [ ] Clients can request only the data they need

---

## Next Steps

| File | Topic |
|---|---|
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Comprehensive best practices |
| [01-REST.md](01-REST.md) | RESTful API design patterns |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Structured learning guide |
| [00-OVERVIEW.md](00-OVERVIEW.md) | API design fundamentals |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial API design anti-patterns |
