# RESTful API Design

## Table of Contents

1. [Overview](#overview)
2. [What is REST](#what-is-rest)
3. [The Six REST Constraints](#the-six-rest-constraints)
4. [Resource Naming Conventions](#resource-naming-conventions)
5. [HTTP Methods](#http-methods)
6. [HTTP Status Codes](#http-status-codes)
7. [Request and Response Design](#request-and-response-design)
8. [HATEOAS](#hateoas)
9. [Pagination](#pagination)
10. [Filtering, Sorting, and Searching](#filtering-sorting-and-searching)
11. [Error Response Format](#error-response-format)
12. [Idempotency and Safety](#idempotency-and-safety)
13. [Best Practices](#best-practices)
14. [Next Steps](#next-steps)
15. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to designing RESTful APIs. It covers foundational constraints, practical patterns for resource naming, HTTP method usage, status codes, content negotiation, pagination, error handling, and the properties of safety and idempotency.

### Target Audience

- **Backend Developers** building HTTP APIs for web and mobile clients
- **Frontend Developers** consuming REST APIs and needing consistent contracts
- **Architects** establishing API standards and reviewing designs
- **Tech Leads** defining team conventions for API development

### Scope

- REST architectural constraints and their implications
- Resource-oriented URI design
- Correct use of HTTP methods, headers, and status codes
- Hypermedia-driven APIs (HATEOAS)
- Pagination, filtering, sorting, and searching patterns
- Standardized error responses (RFC 7807)
- Idempotency and safety guarantees

---

## What is REST

REST (Representational State Transfer) is an **architectural style** for distributed hypermedia systems, defined by Roy Fielding in his 2000 doctoral dissertation. It is not a protocol or a standard — it is a set of constraints that, when applied to the design of a system, produce desirable properties such as scalability, simplicity, and loose coupling.

```
┌──────────┐    GET /api/orders/42     ┌──────────────┐
│  Client  │  ─────────────────────►   │    Server    │
│          │  Accept: application/json │  ┌────────┐  │
│          │  ◄─────────────────────   │  │Resource│  │
│          │  200 OK (JSON body)       │  │ Store  │  │
└──────────┘                           │  └────────┘  │
                                       └──────────────┘
```

A REST API models its domain as **resources** (nouns) that are identified by URIs and manipulated through a uniform interface (HTTP methods). Clients and servers exchange **representations** of resource state — typically JSON or XML documents.

---

## The Six REST Constraints

Fielding defines six constraints that characterize a RESTful architecture.

| # | Constraint | Required | Description |
|---|---|---|---|
| 1 | **Client-Server** | Yes | Separate user interface concerns from data storage concerns. Clients and servers evolve independently. |
| 2 | **Stateless** | Yes | Each request from client to server must contain all information needed to understand the request. No session state on the server. |
| 3 | **Cacheable** | Yes | Responses must define themselves as cacheable or non-cacheable to prevent stale data and improve efficiency. |
| 4 | **Uniform Interface** | Yes | A standardized way to communicate — resource identification via URIs, manipulation through representations, self-descriptive messages, and HATEOAS. |
| 5 | **Layered System** | Yes | The architecture can be composed of hierarchical layers (proxies, gateways, load balancers) without the client knowing. |
| 6 | **Code on Demand** | No | Servers can extend client functionality by transferring executable code (e.g., JavaScript). This is the only optional constraint. |

### Statelessness in Practice

```
❌ Stateful: POST /login → Server creates session → GET /orders uses session_id

✅ Stateless: GET /orders with Authorization: Bearer <token>
   → Token contains identity, permissions, expiry
   → Any server instance can handle the request
   → Horizontal scaling is straightforward
```

---

## Resource Naming Conventions

Resources are the fundamental abstraction in REST. Every resource has a URI that uniquely identifies it.

### Core Rules

1. **Use nouns, not verbs** — URIs identify resources, not actions
2. **Use plural nouns** for collections
3. **Use hierarchical paths** to express relationships
4. **Use lowercase** with hyphens for multi-word names
5. **Avoid file extensions** — use `Accept` headers instead

### Examples

```
✅ Good                              ❌ Bad
GET  /users                          GET  /getUsers
GET  /users/42                       GET  /user/42
GET  /users/42/orders                GET  /getUserOrders?userId=42
POST /users                          POST /createUser
PUT  /users/42                       POST /updateUser
DELETE /users/42                     POST /deleteUser?id=42
GET  /order-items                    GET  /orderItems
```

### Resource Hierarchy

Model parent-child relationships through nesting, but avoid going deeper than two levels:

```
/users                           → Collection of users
/users/{userId}                  → Single user
/users/{userId}/orders           → Orders belonging to a user
/users/{userId}/orders/{orderId} → A specific order for a user

❌ Too deep: /users/{userId}/orders/{orderId}/items/{itemId}/reviews/{reviewId}
✅ Prefer:   /reviews?orderId=42&itemId=7
```

### Actions on Resources

When an operation does not map cleanly to CRUD, model the action as a sub-resource noun:

```
POST /users/42/activation       → Activate user 42
POST /orders/99/cancellation    → Cancel order 99
POST /payments/100/refunds      → Refund payment 100
```

---

## HTTP Methods

The uniform interface constraint requires that a small, fixed set of well-defined methods is used to manipulate resources.

### Method Reference

| Method | Purpose | Request Body | Response Body | Safe | Idempotent |
|---|---|---|---|---|---|
| **GET** | Retrieve a resource or collection | No | Yes | ✅ | ✅ |
| **POST** | Create a new resource or trigger a process | Yes | Yes | ❌ | ❌ |
| **PUT** | Replace a resource entirely | Yes | Optional | ❌ | ✅ |
| **PATCH** | Partially update a resource | Yes | Yes | ❌ | ❌* |
| **DELETE** | Remove a resource | Optional | Optional | ❌ | ✅ |
| **HEAD** | Same as GET but without the response body | No | No | ✅ | ✅ |
| **OPTIONS** | Describe communication options for a resource | No | Yes | ✅ | ✅ |

*PATCH can be made idempotent depending on the patch format (e.g., JSON Merge Patch is idempotent, JSON Patch may not be).

### GET — Retrieve Resources

```http
GET /api/users/42 HTTP/1.1
Host: api.example.com
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{ "id": 42, "name": "Alice Johnson", "email": "alice@example.com", "createdAt": "2024-01-15T09:30:00Z" }
```

### POST — Create a Resource

```http
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json

{ "name": "Bob Smith", "email": "bob@example.com" }
```

```http
HTTP/1.1 201 Created
Location: /api/users/43
Content-Type: application/json

{ "id": 43, "name": "Bob Smith", "email": "bob@example.com", "createdAt": "2024-06-01T14:20:00Z" }
```

### PUT — Replace a Resource

```http
PUT /api/users/43 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{ "name": "Robert Smith", "email": "robert@example.com" }
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{ "id": 43, "name": "Robert Smith", "email": "robert@example.com", "createdAt": "2024-06-01T14:20:00Z" }
```

### PATCH — Partial Update (JSON Merge Patch, RFC 7396)

```http
PATCH /api/users/43 HTTP/1.1
Host: api.example.com
Content-Type: application/merge-patch+json

{ "email": "bob.smith@example.com" }
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{ "id": 43, "name": "Robert Smith", "email": "bob.smith@example.com", "createdAt": "2024-06-01T14:20:00Z" }
```

### DELETE — Remove a Resource

```http
DELETE /api/users/43 HTTP/1.1
Host: api.example.com
```

```http
HTTP/1.1 204 No Content
```

---

## HTTP Status Codes

Use the correct status code to communicate the result of an operation. Clients rely on the status code category to determine how to handle the response.

### 2xx — Success

| Code | Name | When to Use |
|---|---|---|
| **200** | OK | Successful GET, PUT, PATCH, or DELETE that returns a body |
| **201** | Created | Successful POST that creates a new resource. Include a `Location` header. |
| **202** | Accepted | Request accepted for asynchronous processing. The operation is not yet complete. |
| **204** | No Content | Successful request with no response body (common for DELETE). |

### 3xx — Redirection

| Code | Name | When to Use |
|---|---|---|
| **301** | Moved Permanently | Resource URI has changed permanently. Clients should update bookmarks. |
| **304** | Not Modified | Conditional request — cached version is still valid (`If-None-Match`). |
| **307** | Temporary Redirect | Resource temporarily at a different URI. Client preserves HTTP method. |

### 4xx — Client Errors

| Code | Name | When to Use |
|---|---|---|
| **400** | Bad Request | Malformed syntax, invalid request body, or failed validation. |
| **401** | Unauthorized | Missing or invalid authentication credentials. |
| **403** | Forbidden | Authenticated but not authorized to access the resource. |
| **404** | Not Found | Resource does not exist at the given URI. |
| **405** | Method Not Allowed | HTTP method not supported for this resource. Include `Allow` header. |
| **409** | Conflict | Request conflicts with current state (e.g., duplicate, edit conflict). |
| **422** | Unprocessable Entity | Syntactically valid but semantically incorrect (validation errors). |
| **429** | Too Many Requests | Rate limit exceeded. Include `Retry-After` header. |

### 5xx — Server Errors

| Code | Name | When to Use |
|---|---|---|
| **500** | Internal Server Error | Unexpected server failure. Never expose stack traces to clients. |
| **502** | Bad Gateway | Invalid response from an upstream service. |
| **503** | Service Unavailable | Temporarily unable to handle requests. Include `Retry-After`. |
| **504** | Gateway Timeout | Upstream service did not respond in time. |

---

## Request and Response Design

### Content Negotiation

Clients specify preferred response format using `Accept`. Servers declare actual format using `Content-Type`. If the server cannot produce the requested format, it returns `406 Not Acceptable`.

### Common Headers

| Header | Direction | Purpose |
|---|---|---|
| `Accept` | Request | Desired response media type |
| `Content-Type` | Both | Media type of the body |
| `Authorization` | Request | Authentication credentials (`Bearer <token>`) |
| `Location` | Response | URI of newly created resource (with 201) |
| `ETag` | Response | Entity tag for conditional requests and caching |
| `If-None-Match` | Request | Conditional GET — return 304 if ETag matches |
| `If-Match` | Request | Conditional PUT/PATCH — optimistic concurrency |
| `Retry-After` | Response | Wait duration before retrying (with 429, 503) |
| `Cache-Control` | Response | Caching directives (`no-store`, `max-age=3600`) |

### Response Envelope

A single resource returns the object directly. A collection returns an array with pagination metadata.

```json
{
  "data": [
    { "id": 1, "name": "Alice" },
    { "id": 2, "name": "Bob" }
  ],
  "pagination": { "page": 1, "pageSize": 20, "totalItems": 87, "totalPages": 5 }
}
```

---

## HATEOAS

HATEOAS (Hypermedia as the Engine of Application State) is the highest level of REST maturity. The server provides links in responses that tell the client what actions are available, removing the need to hardcode URIs.

### Example — Order with Links

```http
GET /api/orders/99 HTTP/1.1
```

```json
{
  "id": 99,
  "status": "pending",
  "total": 59.99,
  "currency": "USD",
  "items": [
    { "productId": 7, "name": "Wireless Mouse", "quantity": 1, "price": 29.99 },
    { "productId": 12, "name": "USB-C Hub", "quantity": 1, "price": 30.00 }
  ],
  "_links": {
    "self": { "href": "/api/orders/99" },
    "cancel": { "href": "/api/orders/99/cancellation", "method": "POST" },
    "payment": { "href": "/api/orders/99/payment", "method": "POST" },
    "customer": { "href": "/api/users/42" }
  }
}
```

### Example — Collection with Navigation

```json
{
  "data": [
    { "id": 98, "status": "shipped", "_links": { "self": { "href": "/api/orders/98" } } },
    { "id": 99, "status": "pending", "_links": { "self": { "href": "/api/orders/99" } } }
  ],
  "_links": {
    "self":  { "href": "/api/orders?page=2&pageSize=20" },
    "first": { "href": "/api/orders?page=1&pageSize=20" },
    "prev":  { "href": "/api/orders?page=1&pageSize=20" },
    "next":  { "href": "/api/orders?page=3&pageSize=20" },
    "last":  { "href": "/api/orders?page=5&pageSize=20" }
  }
}
```

When an order is shipped, the server omits the `cancel` link — the client discovers that cancellation is unavailable without knowing the business rules.

---

## Pagination

Never return unbounded collections. Every list endpoint should support pagination.

### Strategies

| Strategy | Mechanism | Pros | Cons |
|---|---|---|---|
| **Offset-based** | `?page=2&pageSize=20` or `?offset=20&limit=20` | Simple to implement. Supports jumping to any page. | Inconsistent results when data changes between pages. Slow on large offsets (`OFFSET 100000`). |
| **Cursor-based** | `?cursor=eyJpZCI6MTAwfQ&limit=20` | Stable results even when data changes. Performs well at any depth. | Cannot jump to arbitrary pages. Cursor is opaque to clients. |
| **Keyset-based** | `?createdAfter=2024-06-01T00:00:00Z&limit=20` | Uses indexed columns for fast queries. Transparent parameters. | Requires a unique, sequential sort key. Cannot jump to arbitrary pages. |

### Pagination Examples

```http
GET /api/products?page=3&pageSize=25 HTTP/1.1
→ { "data": [...], "pagination": { "page": 3, "pageSize": 25, "totalItems": 342, "totalPages": 14 } }

GET /api/events?limit=50&cursor=eyJpZCI6MjAwfQ HTTP/1.1
→ { "data": [...], "pagination": { "limit": 50, "hasMore": true, "nextCursor": "eyJpZCI6MjUwfQ" } }
```

### Defaults and Limits

Always define a default page size (e.g., 20) and enforce a maximum (e.g., 100) to prevent abusive queries. Return pagination metadata in every collection response.

---

## Filtering, Sorting, and Searching

### Filtering

Use query parameters named after resource fields:

```http
GET /api/products?category=electronics&inStock=true&minPrice=10&maxPrice=100 HTTP/1.1
```

For complex filtering, consider a structured syntax:

```http
GET /api/products?filter[category]=electronics&filter[price][gte]=10&filter[price][lte]=100 HTTP/1.1
```

### Sorting

Use a `sort` parameter with field names. Prefix with `-` for descending:

```http
GET /api/products?sort=-createdAt,name HTTP/1.1
```

### Searching

For full-text search, use a `q` or `search` parameter:

```http
GET /api/products?q=wireless+mouse HTTP/1.1
```

### Combined Example

```http
GET /api/products?category=electronics&q=keyboard&sort=-rating,price&page=1&pageSize=20 HTTP/1.1
```

```json
{
  "data": [
    { "id": 55, "name": "Mechanical Keyboard Pro", "category": "electronics", "price": 89.99, "rating": 4.8 },
    { "id": 71, "name": "Wireless Keyboard Slim", "category": "electronics", "price": 49.99, "rating": 4.5 }
  ],
  "pagination": { "page": 1, "pageSize": 20, "totalItems": 2, "totalPages": 1 }
}
```

---

## Error Response Format

Consistent error responses are critical for API usability. Adopt a standard structure for every error.

### Basic Error Response

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json

{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "The request body contains invalid fields.",
    "details": [
      { "field": "email", "message": "Must be a valid email address.", "rejectedValue": "not-an-email" },
      { "field": "name", "message": "Must not be blank." }
    ]
  }
}
```

### RFC 7807 — Problem Details for HTTP APIs

RFC 7807 defines a standardized error format (`application/problem+json`) widely adopted across the industry.

```http
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/insufficient-funds",
  "title": "Insufficient Funds",
  "status": 403,
  "detail": "Your account balance of $5.00 is not enough to cover the $29.99 charge.",
  "instance": "/api/payments/txn-abc-123",
  "balance": 5.00,
  "required": 29.99
}
```

| Field | Required | Description |
|---|---|---|
| `type` | Yes | URI reference identifying the problem type. Provides documentation link. |
| `title` | Yes | Short, human-readable summary of the problem type. |
| `status` | Yes | HTTP status code (repeated for convenience in message queues). |
| `detail` | No | Human-readable explanation specific to this occurrence. |
| `instance` | No | URI reference identifying the specific occurrence. |

Fields like `balance` and `required` above are **extension members** for machine-readable context.

---

## Idempotency and Safety

Understanding safety and idempotency is essential for building reliable APIs and enabling safe retries.

- **Safe**: The method does not modify server state.
- **Idempotent**: Multiple identical calls produce the same result as a single call.

### Method Properties

| Method | Safe | Idempotent | Notes |
|---|---|---|---|
| **GET** | ✅ | ✅ | Read-only. Always safe to retry. |
| **HEAD** | ✅ | ✅ | Same as GET without the body. |
| **OPTIONS** | ✅ | ✅ | Describes allowed methods. |
| **PUT** | ❌ | ✅ | Replaces entire resource. Same PUT twice → same state. |
| **DELETE** | ❌ | ✅ | First call deletes, subsequent calls return 404 or 204. End state is the same. |
| **POST** | ❌ | ❌ | Creates a new resource each time. Not safe to blindly retry. |
| **PATCH** | ❌ | ❌* | Depends on patch semantics. JSON Merge Patch is idempotent. |

### Making POST Idempotent with Idempotency Keys

Use an `Idempotency-Key` header to allow safe retries of POST requests:

```http
POST /api/payments HTTP/1.1
Content-Type: application/json
Idempotency-Key: 8a3e4b10-f5c9-4d2a-b8e1-1a2b3c4d5e6f

{ "amount": 29.99, "currency": "USD", "customerId": 42 }
```

```
First request  → 201 Created (payment processed)
Retry request  → 200 OK     (same key, returns cached result — no duplicate charge)
```

### Concurrency Control with ETags

Use `ETag` and `If-Match` headers to prevent lost updates:

```http
GET /api/users/42 HTTP/1.1
→ 200 OK, ETag: "a1b2c3d4"

PUT /api/users/42 HTTP/1.1
If-Match: "a1b2c3d4"
Content-Type: application/json

{ "name": "Alice Johnson", "email": "alice@example.com" }
```

If another client modified the resource, the ETag will not match:

```http
HTTP/1.1 412 Precondition Failed
Content-Type: application/problem+json

{ "type": "https://api.example.com/errors/precondition-failed", "title": "Precondition Failed", "status": 412, "detail": "The resource has been modified since your last read. Re-fetch and retry." }
```

---

## Best Practices

### URI Design

- ✅ Use plural nouns for collection URIs (`/users`, `/orders`)
- ✅ Use hyphens to separate words (`/order-items`, not `/orderItems`)
- ✅ Keep URIs lowercase and version the API (`/v1/users`)
- ❌ Do not put verbs in URIs (`/getUser`, `/createOrder`)
- ❌ Do not nest resources deeper than two levels

### Response Design

- ✅ Return the created or updated resource in POST, PUT, and PATCH responses
- ✅ Use `201 Created` with a `Location` header for resource creation
- ✅ Use `204 No Content` when there is no body to return
- ❌ Do not return `200 OK` with an error message in the body
- ❌ Do not expose internal implementation details (database IDs, stack traces)

### Reliability

- ✅ Make all endpoints idempotent where possible
- ✅ Use `Idempotency-Key` headers for non-idempotent operations
- ✅ Support `ETag` / `If-Match` for optimistic concurrency control
- ✅ Return `Retry-After` with 429 and 503 responses
- ❌ Do not rely on clients to enforce business rules — validate on the server

---

## Next Steps

Continue to [GraphQL](02-GRAPHQL.md) to explore an alternative API paradigm that gives clients fine-grained control over the data they receive, or jump to [API Versioning](04-API-VERSIONING.md) to learn strategies for evolving your REST API without breaking consumers.

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial RESTful API design documentation |
