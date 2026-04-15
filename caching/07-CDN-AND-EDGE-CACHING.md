# CDN and Edge Caching

## Table of Contents

- [Overview](#overview)
  - [What Is CDN Caching](#what-is-cdn-caching)
  - [Edge Computing vs Origin](#edge-computing-vs-origin)
  - [Why CDN Matters for Global Applications](#why-cdn-matters-for-global-applications)
- [How CDNs Work](#how-cdns-work)
  - [Request Flow](#request-flow)
  - [Cache Hit and Miss at the Edge](#cache-hit-and-miss-at-the-edge)
  - [Origin Shield](#origin-shield)
- [HTTP Caching Headers](#http-caching-headers)
  - [Cache-Control Directives](#cache-control-directives)
  - [ETag and Conditional Requests](#etag-and-conditional-requests)
  - [Last-Modified and If-Modified-Since](#last-modified-and-if-modified-since)
  - [Vary Header](#vary-header)
  - [Age and Expires Headers](#age-and-expires-headers)
  - [Header Scenarios Summary](#header-scenarios-summary)
- [CDN Cache Invalidation](#cdn-cache-invalidation)
  - [Purge by URL](#purge-by-url)
  - [Purge by Tag or Surrogate Key](#purge-by-tag-or-surrogate-key)
  - [Wildcard Purge](#wildcard-purge)
  - [Soft Purge](#soft-purge)
  - [API-Based Invalidation](#api-based-invalidation)
- [Edge Computing](#edge-computing)
  - [Running Logic at the Edge](#running-logic-at-the-edge)
  - [Cloudflare Workers](#cloudflare-workers)
  - [AWS CloudFront Functions and Lambda at Edge](#aws-cloudfront-functions-and-lambda-at-edge)
  - [Vercel Edge Functions](#vercel-edge-functions)
  - [Fastly Compute](#fastly-compute)
  - [Edge Computing Comparison Table](#edge-computing-comparison-table)
- [Reverse Proxy Caching](#reverse-proxy-caching)
  - [Varnish Cache](#varnish-cache)
  - [Nginx Caching](#nginx-caching)
  - [HAProxy Caching](#haproxy-caching)
- [CDN Providers](#cdn-providers)
  - [Provider Comparison Table](#provider-comparison-table)
  - [Choosing a CDN](#choosing-a-cdn)
- [Cache Key Design](#cache-key-design)
  - [What Makes a Good Cache Key](#what-makes-a-good-cache-key)
  - [Varying by Header, Cookie, and Query String](#varying-by-header-cookie-and-query-string)
  - [Cache Key Normalization](#cache-key-normalization)
  - [Avoiding Low Hit Ratios](#avoiding-low-hit-ratios)
- [Static vs Dynamic Content Caching](#static-vs-dynamic-content-caching)
  - [What to Cache at the Edge](#what-to-cache-at-the-edge)
  - [Caching API Responses](#caching-api-responses)
  - [Personalization Challenges](#personalization-challenges)
  - [Edge Side Includes](#edge-side-includes)
- [Security at the Edge](#security-at-the-edge)
  - [DDoS Protection](#ddos-protection)
  - [WAF at Edge](#waf-at-edge)
  - [TLS Termination](#tls-termination)
  - [Signed URLs and Cookies for Private Content](#signed-urls-and-cookies-for-private-content)
- [Monitoring CDN Performance](#monitoring-cdn-performance)
  - [Hit Ratio and Origin Offload](#hit-ratio-and-origin-offload)
  - [Latency Percentiles](#latency-percentiles)
  - [Cache Status Headers](#cache-status-headers)
  - [Real User Monitoring](#real-user-monitoring)
- [Common Patterns](#common-patterns)
  - [Full-Page Caching](#full-page-caching)
  - [API Response Caching](#api-response-caching)
  - [Asset Fingerprinting and Cache Busting](#asset-fingerprinting-and-cache-busting)
  - [Multi-Tier Caching](#multi-tier-caching)
  - [A/B Testing at Edge](#ab-testing-at-edge)
- [Version History](#version-history)

---

## Overview

### What Is CDN Caching

A **Content Delivery Network (CDN)** is a geographically distributed group of servers
that work together to deliver internet content quickly. CDN caching stores copies of
content at **edge locations** (Points of Presence, or POPs) close to end users, reducing
the round-trip time to the origin server.

```
Without CDN:                          With CDN:

User (Tokyo) ──────────────────►      User (Tokyo) ──► Edge POP (Tokyo)
         5000 km                                        Cache HIT ✓
         Origin (US-East)                               ~5 ms response

         Latency: ~180 ms              Latency: ~5 ms
```

Key benefits of CDN caching:

- **Reduced latency** — Serve content from the nearest edge location
- **Lower origin load** — Offload traffic from origin servers
- **Improved availability** — Edge nodes can serve stale content if origin is down
- **Bandwidth savings** — Reduce data transfer from origin
- **Better user experience** — Faster page loads globally

### Edge Computing vs Origin

| Aspect            | Edge                              | Origin                            |
|-------------------|-----------------------------------|-----------------------------------|
| Location          | Distributed globally at POPs      | Centralized data center(s)        |
| Latency           | Low (near user)                   | Higher (farther from user)        |
| Compute power     | Limited (lightweight functions)   | Full server capabilities          |
| Storage           | Cache only (ephemeral)            | Persistent (databases, files)     |
| Use case          | Caching, routing, auth checks     | Business logic, data persistence  |
| Scalability       | Automatic at network level        | Requires explicit scaling         |
| Cost model        | Per-request / bandwidth           | Per-instance / infrastructure     |

### Why CDN Matters for Global Applications

Modern applications serve users worldwide. Without a CDN, every request travels to a
centralized origin, suffering from:

1. **Physical distance** — Speed of light imposes ~1 ms per 200 km of fiber
2. **Network congestion** — More hops means more potential bottlenecks
3. **Origin overload** — All traffic hits one region
4. **Single point of failure** — Origin outage means global outage

A CDN addresses all four problems by distributing content and compute to the edge.

```
Global CDN Topology:

                    ┌──────────────┐
                    │   Origin     │
                    │  (US-East)   │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────┴─────┐ ┌───┴────┐ ┌────┴─────┐
        │ Edge POP  │ │Edge POP│ │ Edge POP │
        │ (Europe)  │ │ (Asia) │ │(S.America│
        └─────┬─────┘ └───┬────┘ └────┬─────┘
              │            │            │
         ┌────┴────┐  ┌───┴───┐  ┌────┴────┐
         │  Users  │  │ Users │  │  Users  │
         │ London  │  │ Tokyo │  │São Paulo│
         └─────────┘  └───────┘  └─────────┘
```

---

## How CDNs Work

### Request Flow

When a user requests a resource, the CDN intercepts the request and determines whether
it can serve the content from its edge cache or must fetch it from the origin.

```
CDN Request Flow:

  ┌──────┐        ┌──────────────┐        ┌──────────────┐        ┌────────┐
  │ User │──DNS──►│ CDN DNS/     │──Route─►│  Edge POP    │        │ Origin │
  │      │        │ Anycast      │        │  (Nearest)   │        │ Server │
  └──────┘        └──────────────┘        └──────┬───────┘        └────┬───┘
                                                  │                     │
                                          ┌───────▼────────┐           │
                                          │  Cache Lookup   │           │
                                          └───────┬────────┘           │
                                                  │                     │
                                    ┌─────────────┼─────────────┐      │
                                    │ HIT         │ MISS        │      │
                                    ▼             ▼             │      │
                              ┌──────────┐  ┌──────────┐       │      │
                              │  Return  │  │  Fetch   │───────┼─────►│
                              │  Cached  │  │  from    │       │      │
                              │  Content │  │  Origin  │◄──────┼──────│
                              └──────────┘  └────┬─────┘       │      │
                                                 │              │      │
                                          ┌──────▼───────┐     │      │
                                          │ Cache + Serve │     │      │
                                          │ to User       │     │      │
                                          └───────────────┘     │      │
```

Step-by-step:

1. **DNS Resolution** — User's DNS query resolves to the CDN's anycast IP or nearest POP
2. **Edge Routing** — Request is routed to the closest available edge server
3. **Cache Lookup** — Edge checks its local cache for the requested resource
4. **Cache HIT** — Content is found and returned immediately (fastest path)
5. **Cache MISS** — Edge forwards the request to origin (or origin shield)
6. **Origin Response** — Origin processes the request and returns the content
7. **Cache Store** — Edge caches the response according to caching headers
8. **Serve to User** — Edge returns the content to the user

### Cache Hit and Miss at the Edge

Cache status at the edge is critical for understanding CDN performance:

| Status      | Meaning                                                          |
|-------------|------------------------------------------------------------------|
| **HIT**     | Content served from edge cache                                   |
| **MISS**    | Content not in cache; fetched from origin and then cached        |
| **EXPIRED** | Content was in cache but TTL expired; revalidated with origin    |
| **STALE**   | Serving expired content while revalidating in background         |
| **BYPASS**  | Cache intentionally skipped (e.g., POST requests, Set-Cookie)   |
| **DYNAMIC** | Content marked as uncacheable by origin                          |
| **ERROR**   | Origin returned an error; edge may serve stale if configured     |

### Origin Shield

An **origin shield** is an intermediate caching layer between edge POPs and the origin
server. It consolidates cache misses from multiple edge nodes, reducing origin load.

```
Without Origin Shield:              With Origin Shield:

Edge POP 1 ──MISS──► Origin        Edge POP 1 ──MISS──► Shield ──MISS──► Origin
Edge POP 2 ──MISS──► Origin        Edge POP 2 ──MISS──► Shield (HIT!)
Edge POP 3 ──MISS──► Origin        Edge POP 3 ──MISS──► Shield (HIT!)

Origin receives 3 requests          Origin receives 1 request
```

Benefits of origin shield:

- **Reduced origin traffic** — Shield absorbs repeated cache misses
- **Higher hit ratio** — Consolidated cache has more content
- **Protection during spikes** — Shield throttles requests to origin
- **Consistent caching** — Single cache layer reduces fragmentation

---

## HTTP Caching Headers

HTTP caching headers are the primary mechanism for controlling how CDNs (and browsers)
cache content. Understanding these headers is essential for effective CDN configuration.

### Cache-Control Directives

The `Cache-Control` header is the most important caching header. It supports multiple
directives that can be combined.

**`max-age`** — Time in seconds the response is considered fresh by browsers.

```http
Cache-Control: max-age=3600
# Browser can cache for 1 hour
```

**`s-maxage`** — Overrides `max-age` for shared caches (CDNs, proxies). Browsers ignore it.

```http
Cache-Control: s-maxage=86400, max-age=60
# CDN caches for 24 hours, browser caches for 60 seconds
```

**`public`** — Response can be cached by any cache (CDN, proxy, browser).

```http
Cache-Control: public, max-age=31536000
# Cacheable by everyone for 1 year (immutable assets)
```

**`private`** — Response is for a single user only; CDN must not cache.

```http
Cache-Control: private, max-age=600
# Browser can cache for 10 minutes, CDN must not cache
```

**`no-cache`** — Cache may store the response, but must revalidate with origin
before each use.

```http
Cache-Control: no-cache
# Always check with origin, but can use cached copy if 304
```

**`no-store`** — No cache (CDN or browser) should store the response at all.

```http
Cache-Control: no-store
# Sensitive data — never cache anywhere
```

**`must-revalidate`** — Once the response becomes stale, the cache must revalidate
before using it. Cannot serve stale content.

```http
Cache-Control: max-age=3600, must-revalidate
# Cache for 1 hour, then must revalidate (no stale serving)
```

**`stale-while-revalidate`** — Serve stale content while asynchronously revalidating.

```http
Cache-Control: max-age=300, stale-while-revalidate=60
# Fresh for 5 min, serve stale for 60 more seconds while revalidating
```

**`stale-if-error`** — Serve stale content if the origin returns an error.

```http
Cache-Control: max-age=300, stale-if-error=86400
# Fresh for 5 min, serve stale for 24 hours if origin errors
```

### ETag and Conditional Requests

An **ETag** (Entity Tag) is a unique identifier for a specific version of a resource.

```
First Request:
  Client ──GET /style.css──────────────────► Origin
  Client ◄──200 OK, ETag: "abc123"────────── Origin

Subsequent Request (Conditional):
  Client ──GET /style.css──────────────────► Origin
           If-None-Match: "abc123"
  Client ◄──304 Not Modified───────────────── Origin  (no body!)
```

Example headers:

```http
# Origin response with ETag
HTTP/1.1 200 OK
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
Content-Type: text/css
Cache-Control: no-cache

# Client conditional request
GET /style.css HTTP/1.1
If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"
```

ETags can be:
- **Strong** — `"abc123"` — Byte-for-byte identical
- **Weak** — `W/"abc123"` — Semantically equivalent

### Last-Modified and If-Modified-Since

An alternative to ETags using timestamps:

```http
# Origin response
HTTP/1.1 200 OK
Last-Modified: Wed, 15 Jan 2025 08:00:00 GMT
Content-Type: image/png

# Client conditional request
GET /logo.png HTTP/1.1
If-Modified-Since: Wed, 15 Jan 2025 08:00:00 GMT

# Origin response if unchanged
HTTP/1.1 304 Not Modified
```

ETag vs Last-Modified:

| Feature          | ETag                     | Last-Modified            |
|------------------|--------------------------|--------------------------|
| Precision        | Byte-level accuracy      | 1-second granularity     |
| Overhead         | Needs hash computation   | Filesystem timestamp     |
| Proxy-friendly   | Yes                      | Yes                      |
| Preferred for    | Dynamic content          | Static files             |

### Vary Header

The `Vary` header tells caches to store separate versions of a response based on
specific request headers.

```http
# Cache separate versions per Accept-Encoding
Vary: Accept-Encoding

# Cache separate versions per encoding AND language
Vary: Accept-Encoding, Accept-Language

# Never cache (effectively) — every cookie value creates a new cache entry
Vary: Cookie   # Dangerous! Avoid this on CDNs.
```

Common `Vary` values for CDN caching:

| Vary Value         | Use Case                                   | Impact on Hit Ratio  |
|--------------------|--------------------------------------------|----------------------|
| `Accept-Encoding`  | Serve gzip/brotli/identity versions        | Low (few variants)   |
| `Accept-Language`  | Serve localized content                    | Medium               |
| `Accept`           | Content negotiation (JSON/HTML)            | Medium               |
| `Cookie`           | Per-user caching                           | Very High (avoid!)   |
| `User-Agent`       | Device-specific responses                  | Very High (avoid!)   |

### Age and Expires Headers

**`Age`** — Tells the client how long (in seconds) the response has been in the cache.

```http
HTTP/1.1 200 OK
Cache-Control: max-age=3600
Age: 1200
# Response has been cached for 20 minutes; 40 minutes of freshness remain
```

**`Expires`** — Specifies an absolute date/time after which the response is stale.
Superseded by `Cache-Control: max-age` but still used for backward compatibility.

```http
HTTP/1.1 200 OK
Expires: Thu, 15 Apr 2025 20:00:00 GMT
# Stale after this date/time
```

> **Note:** If both `Cache-Control: max-age` and `Expires` are present,
> `max-age` takes precedence.

### Header Scenarios Summary

| Scenario                        | Recommended Headers                                              |
|---------------------------------|------------------------------------------------------------------|
| Immutable assets (JS/CSS hash)  | `Cache-Control: public, max-age=31536000, immutable`             |
| HTML pages                      | `Cache-Control: public, s-maxage=300, max-age=0, must-revalidate`|
| API response (public)           | `Cache-Control: public, s-maxage=60, stale-while-revalidate=30` |
| API response (private)          | `Cache-Control: private, max-age=60`                             |
| User dashboard                  | `Cache-Control: private, no-cache`                               |
| Login / payment page            | `Cache-Control: no-store`                                        |
| Images (mutable URL)            | `Cache-Control: public, max-age=86400` + `ETag`                 |
| Fonts                           | `Cache-Control: public, max-age=31536000, immutable`             |

---

## CDN Cache Invalidation

When content changes at the origin, stale cached copies at the edge must be removed or
refreshed. CDNs provide several invalidation mechanisms.

### Purge by URL

The simplest form of invalidation — purge a specific URL from all edge caches.

```bash
# CloudFront — Create invalidation
aws cloudfront create-invalidation \
  --distribution-id E1A2B3C4D5 \
  --paths "/images/logo.png"

# Cloudflare — Purge by URL
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  --data '{"files":["https://example.com/images/logo.png"]}'

# Fastly — Purge by URL
curl -X PURGE "https://example.com/images/logo.png" \
  -H "Fastly-Key: {api_token}"
```

Limitations:
- Must know the exact URL(s) to purge
- Does not scale when many URLs change at once
- Propagation time varies by provider (instant to minutes)

### Purge by Tag or Surrogate Key

Tag-based purging allows you to associate cached objects with tags and purge all objects
sharing a tag at once.

```
Origin adds surrogate key headers to responses:

HTTP/1.1 200 OK
Surrogate-Key: product-123 category-electronics homepage
Cache-Control: s-maxage=86400

When product-123 changes, purge all content tagged "product-123":
  - /products/123
  - /categories/electronics  (also tagged product-123)
  - /homepage                (also tagged product-123)
```

```bash
# Fastly — Purge by surrogate key
curl -X POST "https://api.fastly.com/service/{service_id}/purge/product-123" \
  -H "Fastly-Key: {api_token}"

# Cloudflare — Purge by cache tag
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  --data '{"tags":["product-123"]}'
```

### Wildcard Purge

Purge all URLs matching a wildcard pattern.

```bash
# CloudFront — Wildcard invalidation
aws cloudfront create-invalidation \
  --distribution-id E1A2B3C4D5 \
  --paths "/images/*"

# Purge everything
aws cloudfront create-invalidation \
  --distribution-id E1A2B3C4D5 \
  --paths "/*"
```

> **Warning:** Wildcard purges can be expensive. CloudFront charges per invalidation
> path, with the first 1,000 free per month.

### Soft Purge

A **soft purge** marks content as stale rather than removing it. The CDN serves stale
content while asynchronously fetching a fresh copy from origin.

```bash
# Fastly — Soft purge by URL
curl -X PURGE "https://example.com/api/products" \
  -H "Fastly-Key: {api_token}" \
  -H "Fastly-Soft-Purge: 1"
```

Benefits of soft purge:
- No cache miss storm — Users always get a response
- Origin is not overwhelmed by simultaneous refetches
- Graceful content transition

### API-Based Invalidation

Modern CDNs provide REST APIs for programmatic cache invalidation, enabling integration
with CI/CD pipelines and content management systems.

```python
# Example: Invalidate CDN after deployment (Python)
import requests

def invalidate_cloudfront(distribution_id, paths):
    """Invalidate CloudFront cache after deployment."""
    import boto3
    import time

    client = boto3.client("cloudfront")
    response = client.create_invalidation(
        DistributionId=distribution_id,
        InvalidationBatch={
            "Paths": {
                "Quantity": len(paths),
                "Items": paths,
            },
            "CallerReference": str(int(time.time())),
        },
    )
    return response["Invalidation"]["Id"]

def invalidate_cloudflare(zone_id, token, files):
    """Purge specific files from Cloudflare cache."""
    url = f"https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache"
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
    }
    response = requests.post(url, headers=headers, json={"files": files})
    return response.json()["success"]
```

Invalidation comparison:

| Provider   | Purge by URL | Purge by Tag | Wildcard | Soft Purge | Propagation     |
|------------|:------------:|:------------:|:--------:|:----------:|-----------------|
| CloudFront | ✅           | ❌            | ✅       | ❌         | 1–15 minutes    |
| Cloudflare | ✅           | ✅ (Enterprise)| ✅     | ❌         | ~30 seconds     |
| Fastly     | ✅           | ✅            | ❌       | ✅         | ~150 ms         |
| Akamai     | ✅           | ✅            | ✅       | ✅         | ~5 seconds      |

---

## Edge Computing

### Running Logic at the Edge

Edge computing extends the CDN beyond simple caching by allowing developers to run code
at edge locations. This enables request routing, A/B testing, authentication, and
content transformation without round-trips to the origin.

```
Traditional Architecture:              Edge Computing Architecture:

User ──► CDN (cache only) ──► Origin   User ──► CDN (cache + compute) ──► Origin
                                                  │                         │
                                                  ├─ Auth check             │
                                                  ├─ A/B routing            │
                                                  ├─ Header manipulation    │
                                                  ├─ URL rewriting          │
                                                  └─ Response transform     │
                                                                            │
                                              (only if edge cannot handle)──┘
```

### Cloudflare Workers

Cloudflare Workers run JavaScript/TypeScript at the edge using the V8 isolate model.

```javascript
// Cloudflare Worker — A/B testing at the edge
export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    // Determine variant from cookie or assign randomly
    const cookie = request.headers.get("Cookie") || "";
    let variant = cookie.match(/ab-variant=(A|B)/)?.[1];

    if (!variant) {
      variant = Math.random() < 0.5 ? "A" : "B";
    }

    // Route to appropriate origin
    if (variant === "B") {
      url.pathname = `/experimental${url.pathname}`;
    }

    const response = await fetch(url.toString(), request);
    const newResponse = new Response(response.body, response);

    // Set cookie for consistent experience
    newResponse.headers.append(
      "Set-Cookie",
      `ab-variant=${variant}; Path=/; Max-Age=86400`
    );

    return newResponse;
  },
};
```

### AWS CloudFront Functions and Lambda at Edge

AWS offers two edge compute options with different tradeoffs:

```javascript
// CloudFront Function — Lightweight (viewer request/response only)
// Max 10 KB, < 1 ms execution, no network access
function handler(event) {
  var request = event.request;
  var headers = request.headers;

  // Add security headers
  var response = event.response;
  response.headers["strict-transport-security"] = {
    value: "max-age=63072000; includeSubDomains; preload",
  };
  response.headers["x-content-type-options"] = {
    value: "nosniff",
  };
  response.headers["x-frame-options"] = {
    value: "DENY",
  };

  return response;
}
```

```javascript
// Lambda@Edge — Full Lambda runtime at edge (viewer + origin events)
// Up to 5 seconds (viewer) / 30 seconds (origin), network access allowed
exports.handler = async (event) => {
  const request = event.Records[0].cf.request;

  // Redirect based on geolocation
  const country = request.headers["cloudfront-viewer-country"]?.[0]?.value;

  if (country === "DE") {
    return {
      status: "302",
      headers: {
        location: [{ value: "https://de.example.com" + request.uri }],
      },
    };
  }

  return request;
};
```

### Vercel Edge Functions

Vercel Edge Functions use the Web API standard and run on Cloudflare's network.

```typescript
// Vercel Edge Function — Geolocation-based content
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  const country = request.geo?.country || "US";
  const city = request.geo?.city || "Unknown";

  // Add geolocation headers for the application
  const response = NextResponse.next();
  response.headers.set("X-User-Country", country);
  response.headers.set("X-User-City", city);

  // Block traffic from sanctioned countries
  const blockedCountries = ["KP", "IR"];
  if (blockedCountries.includes(country)) {
    return new NextResponse("Access Denied", { status: 403 });
  }

  return response;
}

export const config = {
  matcher: "/api/:path*",
};
```

### Fastly Compute

Fastly Compute (formerly Compute@Edge) runs WebAssembly at the edge, supporting
Rust, JavaScript, Go, and other languages compiled to Wasm.

```rust
// Fastly Compute — Rust example: image optimization routing
use fastly::http::{header, Method, StatusCode};
use fastly::{Error, Request, Response};

#[fastly::main]
fn main(req: Request) -> Result<Response, Error> {
    // Only handle GET requests
    if req.get_method() != &Method::GET {
        return Ok(Response::from_status(StatusCode::METHOD_NOT_ALLOWED));
    }

    let path = req.get_path().to_string();

    // Route image requests to optimization backend
    if path.ends_with(".jpg") || path.ends_with(".png") || path.ends_with(".webp") {
        let mut bereq = req.clone_without_body();
        bereq.set_header("X-Image-Optimize", "true");
        return Ok(bereq.send("image-optimizer")?);
    }

    // All other requests to default backend
    Ok(req.send("default-backend")?)
}
```

### Edge Computing Comparison Table

| Feature             | CF Workers        | CF Functions     | Lambda@Edge       | Vercel Edge      | Fastly Compute   |
|---------------------|-------------------|------------------|-------------------|------------------|------------------|
| **Runtime**         | V8 Isolates       | CloudFront JS    | Node.js Lambda    | V8 Isolates      | WebAssembly      |
| **Languages**       | JS, TS, Wasm      | JS only          | Node.js, Python   | JS, TS           | Rust, JS, Go     |
| **Max exec time**   | 30 s (paid)       | < 1 ms           | 5–30 s            | 25 s             | 120 s            |
| **Max memory**      | 128 MB            | 2 MB             | 128–3008 MB       | 128 MB           | 128 MB           |
| **Network access**  | ✅                | ❌               | ✅                | ✅               | ✅               |
| **KV storage**      | Workers KV        | ❌               | DynamoDB/S3       | Edge Config      | KV Store         |
| **Deploy time**     | Seconds           | Seconds          | Minutes           | Seconds          | Seconds          |
| **Edge locations**  | 300+              | 400+ (CF POPs)   | 13 regions        | 300+             | 90+              |
| **Free tier**       | 100K req/day      | 2M invocations   | 1M requests       | 100K req/month   | Limited          |
| **Cold start**      | ~0 ms             | ~0 ms            | 100–500 ms        | ~0 ms            | < 50 µs          |
| **Best for**        | Full-stack edge   | Header manip     | Complex logic     | Next.js apps     | High-perf edge   |

---

## Reverse Proxy Caching

Reverse proxies sit between clients and origin servers, caching responses to reduce
origin load. They are often used alongside or behind CDNs for multi-tier caching.

```
Multi-Tier Caching Architecture:

  User ──► CDN Edge ──► Reverse Proxy (Varnish/Nginx) ──► Application Server
           (Tier 1)        (Tier 2)                          (Origin)
```

### Varnish Cache

Varnish is a high-performance HTTP reverse proxy and cache, widely used for web
acceleration. It uses VCL (Varnish Configuration Language) for flexible cache policies.

```vcl
# /etc/varnish/default.vcl — Basic Varnish configuration
vcl 4.1;

backend default {
    .host = "127.0.0.1";
    .port = "8080";
    .connect_timeout = 5s;
    .first_byte_timeout = 30s;
    .between_bytes_timeout = 10s;
}

sub vcl_recv {
    # Strip tracking query parameters for better cache hit ratio
    if (req.url ~ "(\?|&)(utm_source|utm_medium|utm_campaign|fbclid)=") {
        set req.url = regsuball(req.url, "(utm_source|utm_medium|utm_campaign|fbclid)=[^&]+&?", "");
        set req.url = regsub(req.url, "[?&]+$", "");
    }

    # Do not cache POST requests
    if (req.method == "POST") {
        return (pass);
    }

    # Do not cache requests with Authorization header
    if (req.http.Authorization) {
        return (pass);
    }

    # Remove cookies for static assets
    if (req.url ~ "\.(css|js|png|jpg|jpeg|gif|ico|woff2|svg)$") {
        unset req.http.Cookie;
        return (hash);
    }

    return (hash);
}

sub vcl_backend_response {
    # Cache static assets for 1 day
    if (bereq.url ~ "\.(css|js|png|jpg|jpeg|gif|ico|woff2|svg)$") {
        set beresp.ttl = 1d;
        unset beresp.http.Set-Cookie;
    }

    # Enable grace mode (serve stale while revalidating)
    set beresp.grace = 1h;
}

sub vcl_deliver {
    # Add cache status header for debugging
    if (obj.hits > 0) {
        set resp.http.X-Cache = "HIT";
        set resp.http.X-Cache-Hits = obj.hits;
    } else {
        set resp.http.X-Cache = "MISS";
    }
}
```

### Nginx Caching

Nginx can act as a reverse proxy cache with built-in caching directives.

```nginx
# /etc/nginx/nginx.conf — Nginx reverse proxy caching

# Define the cache zone
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m
                 max_size=10g inactive=60m use_temp_path=off;

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_cache my_cache;
        proxy_pass http://backend:8080;

        # Cache successful responses for 10 minutes
        proxy_cache_valid 200 302 10m;
        proxy_cache_valid 404     1m;

        # Use stale content while revalidating or on error
        proxy_cache_use_stale error timeout updating
                              http_500 http_502 http_503 http_504;
        proxy_cache_background_update on;

        # Cache key definition
        proxy_cache_key "$scheme$request_method$host$request_uri";

        # Add cache status header
        add_header X-Cache-Status $upstream_cache_status;

        # Bypass cache for specific conditions
        proxy_cache_bypass $cookie_nocache $arg_nocache;
        proxy_no_cache $cookie_nocache $arg_nocache;

        # Lock to prevent cache stampede
        proxy_cache_lock on;
        proxy_cache_lock_timeout 5s;
    }

    # Static assets with long cache TTL
    location ~* \.(css|js|png|jpg|jpeg|gif|ico|woff2)$ {
        proxy_cache my_cache;
        proxy_pass http://backend:8080;
        proxy_cache_valid 200 30d;
        add_header X-Cache-Status $upstream_cache_status;
    }
}
```

### HAProxy Caching

HAProxy added caching support in version 1.8+.

```haproxy
# /etc/haproxy/haproxy.cfg — HAProxy caching configuration

global
    log stdout format raw local0

defaults
    mode http
    timeout connect 5s
    timeout client  30s
    timeout server  30s

cache my_cache
    total-max-size 256      # 256 MB cache
    max-object-size 10000   # 10 KB max per object
    max-age 3600            # Default TTL: 1 hour

frontend http_front
    bind *:80
    default_backend servers

    # Use cache for GET requests
    http-request cache-use my_cache if { method GET }
    http-response cache-store my_cache

backend servers
    balance roundrobin
    server app1 127.0.0.1:8080 check
    server app2 127.0.0.1:8081 check
```

---

## CDN Providers

### Provider Comparison Table

| Feature             | CloudFront        | Cloudflare        | Fastly            | Akamai            | Azure CDN         | Google Cloud CDN  |
|---------------------|-------------------|-------------------|-------------------|-------------------|-------------------|-------------------|
| **Edge locations**  | 600+              | 300+              | 90+               | 4,100+            | 190+              | 190+              |
| **Pricing model**   | Pay-per-use       | Free + paid plans | Pay-per-use       | Contract-based    | Pay-per-use       | Pay-per-use       |
| **Free tier**       | 1 TB/month (1yr)  | Unlimited (basic) | $50 credit        | None              | None              | None              |
| **Purge speed**     | 1–15 min          | ~30 sec           | ~150 ms           | ~5 sec            | ~2 min            | ~10 sec           |
| **Edge compute**    | CF Functions,     | Workers           | Compute           | EdgeWorkers       | ❌                | ❌                |
|                     | Lambda@Edge       |                   |                   |                   |                   |                   |
| **Tag-based purge** | ❌                | ✅ (Enterprise)   | ✅                | ✅                | ❌                | ❌                |
| **HTTP/3 + QUIC**   | ✅                | ✅                | ✅                | ✅                | ✅                | ✅                |
| **WebSocket**       | ✅                | ✅                | ✅                | ✅                | ❌                | ❌                |
| **Origin shield**   | ✅                | ✅ (Tiered Cache) | ✅ (Shielding)    | ✅ (SureRoute)    | ❌                | ✅                |
| **DDoS protection** | AWS Shield        | Built-in          | Built-in          | Prolexic          | Azure DDoS        | Cloud Armor       |
| **WAF**             | AWS WAF           | Built-in          | WAF (paid)        | Kona WAF          | Azure WAF         | Cloud Armor       |
| **Best for**        | AWS ecosystem     | All-in-one easy   | Real-time purge   | Enterprise scale  | Azure ecosystem   | GCP ecosystem     |

### Choosing a CDN

Factors to consider when selecting a CDN provider:

1. **Geographic coverage** — Where are your users? Ensure the CDN has POPs nearby
2. **Purge speed** — How quickly do you need cache invalidations to take effect?
3. **Edge compute** — Do you need to run logic at the edge?
4. **Integration** — Does it integrate with your existing cloud provider?
5. **Cost** — Pay-per-use vs. commit-based pricing
6. **Support** — Enterprise support, SLAs, and documentation
7. **Security** — Built-in DDoS, WAF, and bot management

---

## Cache Key Design

### What Makes a Good Cache Key

The **cache key** uniquely identifies a cached resource. A well-designed cache key
maximizes hit ratio while correctly serving different content variants.

```
Default Cache Key:
  scheme + host + path + query string
  Example: https://example.com/api/products?page=1&sort=price

Good Cache Key Properties:
  ✅ Unique per distinct content variant
  ✅ Normalized (consistent ordering, case)
  ✅ Minimal (excludes irrelevant parameters)
  ✅ Deterministic (same input = same key)
```

### Varying by Header, Cookie, and Query String

```
Cache Key Components:

┌───────────────────────────────────────────┐
│              Cache Key                     │
│                                            │
│  URL Path    /api/products                 │
│  Query       ?category=electronics         │
│  Headers     Accept-Encoding: gzip         │
│  Cookies     (specific cookies only!)      │
│  Device      mobile / desktop              │
│  Country     US / DE / JP                  │
└───────────────────────────────────────────┘
```

Examples of cache key customization:

```
# CloudFront — Cache policy with custom key
{
  "CachePolicyConfig": {
    "Name": "ProductPagePolicy",
    "DefaultTTL": 300,
    "MaxTTL": 86400,
    "MinTTL": 0,
    "ParametersInCacheKeyAndForwardedToOrigin": {
      "EnableAcceptEncodingGzip": true,
      "EnableAcceptEncodingBrotli": true,
      "HeadersConfig": {
        "HeaderBehavior": "whitelist",
        "Headers": { "Items": ["Accept-Language"] }
      },
      "CookiesConfig": {
        "CookieBehavior": "whitelist",
        "Cookies": { "Items": ["currency", "region"] }
      },
      "QueryStringsConfig": {
        "QueryStringBehavior": "whitelist",
        "QueryStrings": { "Items": ["page", "sort", "category"] }
      }
    }
  }
}
```

### Cache Key Normalization

Normalization ensures that equivalent requests map to the same cache key.

```
Before Normalization:                 After Normalization:
  /products?sort=price&page=1         /products?page=1&sort=price
  /products?page=1&sort=price         /products?page=1&sort=price
  /Products?page=1&sort=price         /products?page=1&sort=price
  /products?page=1&sort=price&utm=x   /products?page=1&sort=price

All four URLs should resolve to the SAME cache key.
```

Normalization strategies:

1. **Sort query parameters** alphabetically
2. **Lowercase** the path and host
3. **Remove tracking parameters** (utm_*, fbclid, gclid)
4. **Remove default values** (?page=1 when 1 is default)
5. **Normalize encoding** (%20 vs +)

### Avoiding Low Hit Ratios

Common causes of low cache hit ratios and their fixes:

| Problem                         | Cause                                    | Fix                                    |
|---------------------------------|------------------------------------------|----------------------------------------|
| Unique query strings            | Analytics params in URL                  | Strip tracking params                  |
| `Vary: Cookie`                  | Every user gets a unique cache entry     | Remove cookies from cache key          |
| `Vary: User-Agent`              | Thousands of unique UAs                  | Normalize to device class              |
| Random request IDs in URL       | Cache key includes request ID            | Exclude from cache key                 |
| Cache-Control: private          | Origin marks responses as private        | Use s-maxage for CDN layer             |
| Short TTLs                      | Content expires before reuse             | Increase TTL + use stale-while-revalidate |
| Low traffic pages               | Not enough requests to warm cache        | Use origin shield for consolidation    |

---

## Static vs Dynamic Content Caching

### What to Cache at the Edge

```
Cacheability Spectrum:

  Always Cache          Sometimes Cache           Rarely Cache
  ◄──────────────────────────────────────────────────────────►

  ┌─────────────┐     ┌──────────────────┐     ┌──────────────┐
  │ Static Files│     │ Public API Resp. │     │ User-Specific│
  │ - JS/CSS    │     │ - Product catalog│     │ - Dashboard  │
  │ - Images    │     │ - Blog posts     │     │ - Cart       │
  │ - Fonts     │     │ - Search results │     │ - Account    │
  │ - Videos    │     │ - Public feeds   │     │ - Checkout   │
  └─────────────┘     └──────────────────┘     └──────────────┘
  TTL: 1 year          TTL: 1 min–1 hour        TTL: 0 (private)
```

### Caching API Responses

API responses can be cached at the edge when they are public and change infrequently.

```http
# Cacheable API response
GET /api/v1/products/123 HTTP/1.1
Host: api.example.com

HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: public, s-maxage=300, stale-while-revalidate=60
ETag: "v1-product-123-abc"
Vary: Accept-Encoding

{
  "id": 123,
  "name": "Widget",
  "price": 29.99,
  "updated_at": "2025-04-14T12:00:00Z"
}
```

```http
# Non-cacheable API response (user-specific)
GET /api/v1/users/me/orders HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJ...

HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: private, no-cache
```

### Personalization Challenges

Personalized content is the biggest challenge for CDN caching. Strategies to handle it:

```
Strategy 1: Cache the Shell, Fetch Personalization Client-Side

  CDN serves:     <html>... <div id="user-greeting"></div> ...</html>
  Browser fetches: GET /api/me/greeting → "Hello, Alice!"
  JavaScript:     document.getElementById("user-greeting").textContent = data;

Strategy 2: Edge Side Includes (ESI)
  CDN assembles page from cached fragments + personalized fragments

Strategy 3: Edge Compute
  Worker at edge reads user cookie, modifies cached response
```

### Edge Side Includes

**ESI (Edge Side Includes)** allows CDNs to assemble pages from multiple fragments,
each with its own caching policy.

```html
<!-- Page template with ESI tags -->
<html>
<head>
  <title>Product Page</title>
</head>
<body>
  <!-- Cached for 1 hour (shared) -->
  <esi:include src="/fragments/header" />

  <!-- Cached for 5 minutes -->
  <esi:include src="/fragments/product/123" />

  <!-- Not cached (per-user) -->
  <esi:include src="/fragments/user-recommendations" />

  <!-- Cached for 1 day (shared) -->
  <esi:include src="/fragments/footer" />
</body>
</html>
```

```
ESI Assembly at the Edge:

  ┌──────────────────────────────────┐
  │         Edge POP                  │
  │                                   │
  │  Template: product-page.html      │
  │    ├─ /fragments/header    (HIT)  │  ◄── Cached 1 hour
  │    ├─ /fragments/product   (HIT)  │  ◄── Cached 5 min
  │    ├─ /fragments/user-recs (MISS) │  ──► Origin fetch
  │    └─ /fragments/footer    (HIT)  │  ◄── Cached 1 day
  │                                   │
  │  Assembled page ──► User          │
  └──────────────────────────────────┘
```

ESI support by CDN:

| CDN         | ESI Support   | Notes                            |
|-------------|:-------------:|----------------------------------|
| Fastly      | ✅            | Full ESI via VCL                 |
| Akamai      | ✅            | Industry-leading ESI support     |
| Varnish     | ✅            | Built-in ESI processing          |
| Cloudflare  | ❌            | Use Workers for fragment assembly|
| CloudFront  | ❌            | Use Lambda@Edge for composition  |

---

## Security at the Edge

### DDoS Protection

CDNs inherently provide DDoS protection by distributing traffic across their global
network, absorbing attacks at the edge before they reach the origin.

```
DDoS Attack Mitigation:

  Attacker ──► ┌───────────────────────┐
  Botnet       │     CDN Edge Layer     │
  Traffic ──►  │                        │
               │  ┌──────────────────┐  │
  Legitimate   │  │ Rate Limiting    │  │ ──► Blocked traffic
  Users ───►   │  │ IP Reputation    │  │     (absorbed at edge)
               │  │ Geo-blocking     │  │
               │  │ Challenge pages  │  │
               │  └────────┬─────────┘  │
               │           │            │
               └───────────┼────────────┘
                           │
                    Clean traffic only
                           │
                           ▼
                    ┌──────────────┐
                    │    Origin    │
                    └──────────────┘
```

DDoS protection layers:

| Layer     | Protection                                        | Example                   |
|-----------|---------------------------------------------------|---------------------------|
| Network   | Volumetric attack absorption (L3/L4)              | SYN flood, UDP flood      |
| Transport | Connection-level filtering                        | Slowloris, TCP state      |
| App       | HTTP request analysis, rate limiting               | HTTP flood, bot attacks   |
| DNS       | DNS amplification protection                      | DNS flood                 |

### WAF at Edge

A **Web Application Firewall (WAF)** at the edge inspects HTTP requests and blocks
malicious traffic before it reaches the origin.

```
WAF Rule Examples:

  Rule: Block SQL injection
  Pattern: req.uri matches "(\b(SELECT|INSERT|UPDATE|DELETE|DROP)\b)"
  Action: BLOCK

  Rule: Rate limit login attempts
  Pattern: req.uri == "/api/login" && req.method == "POST"
  Action: RATE_LIMIT (10 req/min per IP)

  Rule: Block known bad bots
  Pattern: req.headers["User-Agent"] matches known_bot_signatures
  Action: CHALLENGE
```

### TLS Termination

CDNs terminate TLS at the edge, reducing the TLS handshake latency for end users.

```
TLS Termination at Edge:

  User ──TLS 1.3──► Edge POP ──HTTP/TLS──► Origin
         (fast)       │          (internal)
                      │
              TLS terminated here
              Certificate managed by CDN
              OCSP stapling
              HTTP/2 + HTTP/3 support
```

Benefits:
- **Reduced latency** — TLS handshake with nearest POP instead of distant origin
- **Certificate management** — CDN manages certificate renewal (e.g., Let's Encrypt)
- **Protocol optimization** — HTTP/2, HTTP/3 at edge even if origin only supports HTTP/1.1
- **Origin simplification** — Origin can run plain HTTP internally

### Signed URLs and Cookies for Private Content

Signed URLs and cookies restrict access to cached private content (e.g., paid videos,
premium downloads) without disabling CDN caching entirely.

```
Signed URL Flow:

  1. User authenticates with application
  2. Application generates signed URL with expiry:
     https://cdn.example.com/video.mp4?Policy=...&Signature=...&Key-Pair-Id=...
  3. User requests signed URL from CDN
  4. CDN validates signature and expiry
  5. If valid → serve cached content
  6. If invalid or expired → 403 Forbidden
```

```python
# Generate a signed CloudFront URL (Python)
import datetime
from botocore.signers import CloudFrontSigner
import rsa

def rsa_signer(message):
    with open("private_key.pem", "rb") as key_file:
        private_key = rsa.PrivateKey.load_pkcs1(key_file.read())
    return rsa.sign(message, private_key, "SHA-1")

def generate_signed_url(url, key_pair_id, expire_minutes=60):
    cf_signer = CloudFrontSigner(key_pair_id, rsa_signer)
    expire_date = datetime.datetime.utcnow() + datetime.timedelta(
        minutes=expire_minutes
    )
    signed_url = cf_signer.generate_presigned_url(
        url, date_less_than=expire_date
    )
    return signed_url

# Usage
url = generate_signed_url(
    "https://cdn.example.com/premium/video.mp4",
    "K2JCJMDEHXQW7F",
    expire_minutes=120,
)
```

---

## Monitoring CDN Performance

### Hit Ratio and Origin Offload

The **cache hit ratio** is the percentage of requests served from cache vs. origin.

```
Hit Ratio Calculation:

  Hit Ratio = Cache Hits / (Cache Hits + Cache Misses) × 100%

  Example:
    Cache Hits:   950,000
    Cache Misses:  50,000
    Total:      1,000,000

    Hit Ratio = 950,000 / 1,000,000 × 100% = 95%
```

Target hit ratios by content type:

| Content Type     | Target Hit Ratio | Notes                             |
|------------------|:----------------:|-----------------------------------|
| Static assets    | > 98%            | Should almost always be cached    |
| Images           | > 95%            | Few variants, long TTL            |
| HTML pages       | > 80%            | Varies by site dynamism           |
| API responses    | > 60%            | Depends on personalization level  |
| Overall          | > 90%            | Good baseline for most sites      |

**Origin offload** is the percentage of bandwidth saved at the origin:

```
Origin Offload = (1 - Origin Bandwidth / Total Bandwidth) × 100%
```

### Latency Percentiles

Monitor latency at multiple percentiles to understand user experience:

```
Latency Percentiles Dashboard:

  P50 (Median):  12 ms   ████████████░░░░░░░░░░░░░░░░░░
  P75:           28 ms   ████████████████████████████░░░
  P90:           65 ms   █████████████████████████████░░
  P95:          142 ms   ██████████████████████████████░
  P99:          380 ms   ██████████████████████████████▒

  ───────────────────────────────────────────────
  Target:  P50 < 20 ms, P95 < 200 ms, P99 < 500 ms
```

Key latency metrics:

| Metric           | Description                              | Target        |
|------------------|------------------------------------------|---------------|
| TTFB (edge)      | Time to first byte from edge             | < 20 ms       |
| TTFB (origin)    | Time to first byte from origin           | < 200 ms      |
| Cache HIT TTFB   | TTFB for cached responses                | < 10 ms       |
| Cache MISS TTFB  | TTFB for origin-fetched responses        | < 300 ms      |
| DNS resolution   | Time to resolve CDN hostname             | < 20 ms       |
| TLS handshake    | Time for TLS negotiation at edge         | < 30 ms       |

### Cache Status Headers

CDNs return custom headers indicating cache status. These are essential for debugging.

```http
# CloudFront
X-Cache: Hit from cloudfront
X-Amz-Cf-Pop: IAD89-P3          # POP location identifier
X-Amz-Cf-Id: abc123...          # Request trace ID
Age: 45                          # Seconds since cached

# Cloudflare
CF-Cache-Status: HIT             # HIT, MISS, EXPIRED, DYNAMIC, BYPASS
CF-Ray: 7f2a...                  # Unique request ID
Age: 120

# Fastly
X-Cache: HIT                     # HIT, MISS, PASS, SYNTH
X-Cache-Hits: 5                  # Number of times served from cache
X-Served-By: cache-iad-kiad...  # POP identifier
X-Timer: S1234.000,VS0,VE1      # Timing breakdown

# Akamai
X-Cache: TCP_HIT from a23-45... # Cache status with server ID
X-Akamai-Request-ID: abc123
X-Check-Cacheable: YES           # Whether response was cacheable
```

Cache status values reference:

| Header Value       | Provider   | Meaning                                            |
|--------------------|------------|----------------------------------------------------|
| `HIT`              | All        | Served from edge cache                             |
| `MISS`             | All        | Fetched from origin, now cached                    |
| `EXPIRED`          | Cloudflare | Was cached but TTL expired, re-fetched             |
| `DYNAMIC`          | Cloudflare | Determined to be uncacheable                       |
| `BYPASS`           | Multiple   | Cache intentionally skipped                        |
| `STALE`            | Fastly     | Served stale while revalidating                    |
| `PASS`             | Fastly     | Request passed to origin without caching           |
| `REVALIDATED`      | Cloudflare | Cache was revalidated via conditional request      |
| `TCP_HIT`          | Akamai     | Served from Akamai edge cache                      |
| `TCP_MISS`         | Akamai     | Fetched from origin through Akamai                 |

### Real User Monitoring

Real User Monitoring (RUM) collects performance data from actual users to understand
CDN effectiveness in the real world.

```javascript
// Collect CDN performance metrics via Navigation Timing API
function collectCDNMetrics() {
  const entries = performance.getEntriesByType("navigation");
  if (entries.length === 0) return;

  const nav = entries[0];

  const metrics = {
    dnsLookup: nav.domainLookupEnd - nav.domainLookupStart,
    tlsHandshake: nav.connectEnd - nav.secureConnectionStart,
    ttfb: nav.responseStart - nav.requestStart,
    download: nav.responseEnd - nav.responseStart,
    totalLoad: nav.loadEventEnd - nav.fetchStart,
  };

  // Send to analytics endpoint
  navigator.sendBeacon("/analytics/cdn-perf", JSON.stringify(metrics));
}

// Collect per-resource CDN metrics
function collectResourceMetrics() {
  const resources = performance.getEntriesByType("resource");

  resources.forEach((res) => {
    if (res.transferSize === 0 && res.decodedBodySize > 0) {
      // Served from browser cache (not CDN)
      console.log(`Browser cache: ${res.name}`);
    }
    // CDN cache status must be checked via response headers
  });
}

window.addEventListener("load", () => {
  setTimeout(collectCDNMetrics, 0);
  setTimeout(collectResourceMetrics, 0);
});
```

---

## Common Patterns

### Full-Page Caching

Cache entire HTML pages at the edge for maximum performance.

```
Full-Page Caching:

  Request ──► Edge POP
               │
               ├─ Has cached HTML? ──YES──► Serve cached page (< 10 ms)
               │
               └─ No ──► Fetch from origin ──► Cache + Serve
                          │
                          Origin renders full page:
                          - Server-side rendering
                          - Template compilation
                          - Database queries
                          (200–500 ms)
```

```http
# Origin response headers for full-page caching
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Cache-Control: public, s-maxage=300, stale-while-revalidate=60, stale-if-error=3600
Surrogate-Key: page-homepage product-123 category-electronics
ETag: "page-v42"
Vary: Accept-Encoding
```

### API Response Caching

Cache API responses at the edge with appropriate headers.

```http
# Cacheable list endpoint
GET /api/v1/products?category=electronics&page=1 HTTP/1.1

HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: public, s-maxage=120, stale-while-revalidate=30
Surrogate-Key: products category-electronics
Vary: Accept-Encoding, Accept
```

```
API Caching Decision Tree:

  Is the response user-specific?
    ├─ YES ──► Cache-Control: private, no-cache
    │           (or don't cache at CDN)
    └─ NO
        │
        Is the data frequently updated?
          ├─ YES ──► s-maxage=60, stale-while-revalidate=30
          └─ NO
              │
              Is it truly static?
                ├─ YES ──► s-maxage=86400
                └─ NO  ──► s-maxage=300, stale-while-revalidate=60
```

### Asset Fingerprinting and Cache Busting

Fingerprinting embeds a hash of the file content in the filename, enabling infinite
cache TTLs while guaranteeing fresh content on changes.

```
Asset Fingerprinting:

  Build Process:
    style.css ──hash──► style.a1b2c3d4.css
    app.js    ──hash──► app.e5f6g7h8.js

  HTML references updated automatically:
    <link rel="stylesheet" href="/style.a1b2c3d4.css">
    <script src="/app.e5f6g7h8.js"></script>

  Cache Headers:
    Cache-Control: public, max-age=31536000, immutable

  When file changes:
    style.css ──hash──► style.x9y0z1w2.css  (new hash = new URL)
    Old version: still cached (no purge needed!)
    New version: fetched as new resource
```

```nginx
# Nginx — Serve fingerprinted assets with long cache TTL
location ~* \.[0-9a-f]{8}\.(css|js|png|jpg|woff2)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
    add_header X-Cache-Status $upstream_cache_status;
}
```

### Multi-Tier Caching

Combine CDN, reverse proxy, and application caches for maximum offload and resilience.

```
Multi-Tier Caching Architecture:

  ┌──────┐     ┌──────────┐     ┌───────────┐     ┌──────────┐     ┌──────────┐
  │ User │────►│ CDN Edge │────►│  Reverse  │────►│   App    │────►│ Database │
  │      │     │  (Tier 1)│     │   Proxy   │     │  Server  │     │          │
  └──────┘     └──────────┘     │  (Tier 2) │     │ (Tier 3) │     └──────────┘
                                └───────────┘     └──────────┘

  Tier 1: CDN Edge Cache
    - Global distribution
    - Static assets, public pages
    - TTL: minutes to years
    - Hit ratio: 85-98%

  Tier 2: Reverse Proxy (Varnish/Nginx)
    - Regional (per data center)
    - Dynamic content, API responses
    - TTL: seconds to minutes
    - Hit ratio: 60-90%

  Tier 3: Application Cache (Redis/Memcached)
    - Per-server or shared cluster
    - Database query results, sessions
    - TTL: seconds to hours
    - Hit ratio: 70-95%
```

Request flow with hit ratios:

```
1,000,000 requests from users
    │
    ▼
CDN Edge (95% hit ratio)
    │ 950,000 served from CDN
    ▼
50,000 requests to Reverse Proxy (80% hit ratio)
    │ 40,000 served from proxy cache
    ▼
10,000 requests to Application (70% hit ratio)
    │ 7,000 served from app cache
    ▼
3,000 requests to Database

Overall: 99.7% of requests never hit the database!
```

### A/B Testing at Edge

Run A/B tests at the edge without impacting origin performance or requiring client-side
JavaScript.

```
A/B Testing at Edge:

  ┌──────┐     ┌──────────────────────────────┐
  │ User │────►│         Edge POP              │
  └──────┘     │                                │
               │  1. Check for variant cookie   │
               │  2. If no cookie: assign       │
               │     randomly (50/50)           │
               │  3. Fetch cached variant page  │
               │  4. Set variant cookie         │
               │  5. Return response            │
               │                                │
               │  Cache Key includes variant:   │
               │    /page + variant=A           │
               │    /page + variant=B           │
               └──────────────────────────────┘
```

```javascript
// Cloudflare Worker — A/B testing with cached variants
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const cookies = request.headers.get("Cookie") || "";

    // Read or assign variant
    let variant = cookies.match(/ab_test=(control|experiment)/)?.[1];
    if (!variant) {
      variant = Math.random() < 0.5 ? "control" : "experiment";
    }

    // Create cache key that includes variant
    const cacheKey = new Request(url.toString(), {
      headers: { "X-AB-Variant": variant },
    });

    const cache = caches.default;
    let response = await cache.match(cacheKey);

    if (!response) {
      // Fetch from origin with variant header
      const originReq = new Request(url.toString(), request);
      originReq.headers.set("X-AB-Variant", variant);

      response = await fetch(originReq);
      response = new Response(response.body, response);
      response.headers.set("Cache-Control", "public, s-maxage=300");

      // Store in cache with variant key
      await cache.put(cacheKey, response.clone());
    }

    // Set variant cookie for consistency
    response = new Response(response.body, response);
    response.headers.append(
      "Set-Cookie",
      `ab_test=${variant}; Path=/; Max-Age=2592000; SameSite=Lax`
    );

    return response;
  },
};
```

Key considerations for edge A/B testing:

| Concern               | Solution                                                |
|-----------------------|---------------------------------------------------------|
| Consistent assignment | Use cookies to persist variant across requests          |
| Cache separation      | Include variant in cache key                            |
| Analytics tracking    | Pass variant header to origin for server-side logging   |
| Gradual rollout       | Adjust random threshold (e.g., 10% experiment)         |
| Multiple tests        | Combine test IDs in cookie, keep cache key manageable   |
| SEO impact            | Serve same content to bots, or use canonical URLs       |

---

## Version History

| Date       | Change          | Author |
|------------|-----------------|--------|
| 2025-04-15 | Initial version | Team   |
