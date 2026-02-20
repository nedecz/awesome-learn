# API Authentication and Authorization

## Table of Contents

1. [Overview](#overview)
2. [Authentication vs Authorization](#authentication-vs-authorization)
3. [API Keys](#api-keys)
4. [OAuth 2.0](#oauth-20)
5. [OpenID Connect](#openid-connect)
6. [JSON Web Tokens (JWT)](#json-web-tokens-jwt)
7. [Bearer Tokens and Token Management](#bearer-tokens-and-token-management)
8. [Mutual TLS (mTLS)](#mutual-tls-mtls)
9. [API Security Headers](#api-security-headers)
10. [Scopes and Permissions](#scopes-and-permissions)
11. [Comparison of Authentication Methods](#comparison-of-authentication-methods)
12. [Security Best Practices](#security-best-practices)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to authentication and authorization for APIs. It covers the mechanisms used to verify the identity of API consumers, control what resources they can access, and protect APIs against common security threats.

### Target Audience

- **Backend Developers** implementing authentication and authorization in API services
- **Frontend Developers** integrating with OAuth flows and managing tokens in clients
- **Architects** designing security models for APIs and microservices
- **Tech Leads** establishing security standards and reviewing API access control patterns

### Scope

- Distinguishing authentication from authorization
- API keys, OAuth 2.0, OpenID Connect, JWT, and mTLS
- Token lifecycle management: issuance, refresh, and revocation
- Security headers including CORS, CSP, and rate limiting
- Permission models: RBAC and ABAC
- Practical guidance on credential storage, rotation, and transport security

---

## Authentication vs Authorization

Authentication and authorization are distinct concerns that are often conflated. Understanding the difference is critical to designing a secure API.

| Aspect | Authentication (AuthN) | Authorization (AuthZ) |
|---|---|---|
| **Question answered** | *Who are you?* | *What are you allowed to do?* |
| **Purpose** | Verify identity | Enforce permissions |
| **When it happens** | Before authorization | After authentication |
| **Failure response** | `401 Unauthorized` | `403 Forbidden` |
| **Example mechanism** | Password, token, certificate | Roles, scopes, policies |

```
Request --> [ Authentication ] --> [ Authorization ] --> Resource
              Who is this?          Is this allowed?
              401 on failure        403 on failure
```

A `401 Unauthorized` response means the server cannot identify the caller. A `403 Forbidden` response means the caller's identity is known but they lack permission for the requested action.

---

## API Keys

An API key is a simple string token that a client includes with each request to identify itself. API keys are best suited for identifying calling applications rather than individual users.

### Passing API Keys

API keys can be sent via a custom header, a query parameter, or (less commonly) in the request body. The header approach is preferred because query parameters may be logged by proxies and servers.

```http
GET /api/weather?city=Seattle HTTP/1.1
Host: api.example.com
X-API-Key: ak_live_7f3a9b2c4d8e1f06
```

### Security Considerations

- ❌ **Do not embed API keys in client-side code** — they will be visible in source bundles and browser dev tools.
- ❌ **Do not pass API keys in URLs** — query strings appear in server logs, proxy logs, and browser history.
- ✅ **Transmit keys only over HTTPS** — plaintext transport exposes keys to network interception.
- ✅ **Scope keys to specific permissions** — a key for read-only access should not grant write access.
- ✅ **Bind keys to IP ranges or referrers** when the deployment model allows it.

### Key Rotation

Rotate keys periodically and immediately upon suspected compromise. Support overlapping validity windows so consumers can migrate without downtime — allow two active keys simultaneously, deploy the new key, then deactivate the old one.

---

## OAuth 2.0

OAuth 2.0 (RFC 6749) is the industry-standard framework for delegated authorization. It allows a user to grant a third-party application limited access to their resources without sharing their credentials.

### Roles

| Role | Description |
|---|---|
| **Resource Owner** | The user who owns the data (e.g., the end user) |
| **Client** | The application requesting access on behalf of the user |
| **Authorization Server** | Issues tokens after authenticating the resource owner |
| **Resource Server** | Hosts the protected resources and validates tokens |

### Authorization Code Grant

The authorization code grant is the most common flow for server-side web applications. Tokens are never exposed to the user's browser.

```
Client                                  Authorization Server
  |  (1) Auth request ------------------>  |
  |  (2) Authorization code <-----------  |
  |  (3) Exchange code + secret -------->  |
  |  (4) Access token + refresh <-------  |
  |                                        |
  |  (5) API request with access token     |
  v                                        |
Resource Server                            |
```

### Authorization Code with PKCE

PKCE (Proof Key for Code Exchange, RFC 7636) is required for public clients such as single-page applications and mobile apps that cannot securely store a client secret.

```
Client (SPA)                            Authorization Server
  |  (1) Generate code_verifier          |
  |      and code_challenge              |
  |  (2) Auth request + code_challenge ->|
  |  (3) Authorization code <----------- |
  |  (4) Exchange code + code_verifier ->|
  |  (5) Access token <----------------- |
```

The server compares `SHA256(code_verifier)` against the `code_challenge` sent in step 2, proving the same client that initiated the flow is exchanging the code.

### Client Credentials Grant

The client credentials grant is used for machine-to-machine communication where no user context is needed.

```http
POST /oauth/token HTTP/1.1
Host: auth.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id=service-inventory&client_secret=s3cr3t_v4lu3&scope=orders:read
```

```json
{"access_token":"eyJhbGciOiJSUzI1NiIs...","token_type":"Bearer","expires_in":3600,"scope":"orders:read"}
```

### When to Use Each Grant Type

| Grant Type | Use Case | Client Type |
|---|---|---|
| Authorization Code | Server-side web apps | Confidential |
| Authorization Code + PKCE | SPAs, mobile apps | Public |
| Client Credentials | Service-to-service | Confidential |
| Device Code | Smart TVs, CLI tools | Public (input-constrained) |

---

## OpenID Connect

OpenID Connect (OIDC) is an identity layer built on top of OAuth 2.0. While OAuth 2.0 handles authorization (what can I access?), OIDC handles authentication (who am I?).

### ID Tokens

OIDC introduces the **ID token**, a JWT that contains claims about the authenticated user. The ID token is intended for the client application and should not be sent to resource servers as an access credential.

```json
{
  "iss": "https://auth.example.com",
  "sub": "user-8842",
  "aud": "client-app-id",
  "exp": 1717027200,
  "iat": 1717023600,
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "email_verified": true
}
```

### UserInfo Endpoint

The UserInfo endpoint returns claims about the authenticated user when called with an access token.

```http
GET /userinfo HTTP/1.1
Host: auth.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

```json
{"sub":"user-8842","name":"Alice Johnson","email":"alice@example.com"}
```

### Discovery

OIDC providers publish a discovery document at a well-known URL, allowing clients to dynamically configure themselves without hardcoding endpoints.

```http
GET /.well-known/openid-configuration HTTP/1.1
Host: auth.example.com
```

```json
{
  "issuer": "https://auth.example.com",
  "authorization_endpoint": "https://auth.example.com/authorize",
  "token_endpoint": "https://auth.example.com/oauth/token",
  "userinfo_endpoint": "https://auth.example.com/userinfo",
  "jwks_uri": "https://auth.example.com/.well-known/jwks.json",
  "scopes_supported": ["openid", "profile", "email"],
  "id_token_signing_alg_values_supported": ["RS256", "ES256"]
}
```

---

## JSON Web Tokens (JWT)

A JSON Web Token (RFC 7519) is a compact, URL-safe token format used to represent claims between two parties. JWTs are widely used as access tokens and ID tokens in OAuth 2.0 and OIDC.

### Structure

A JWT consists of three Base64URL-encoded parts separated by dots: `header.payload.signature`.

**Header:**

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "key-2024-03"
}
```

**Payload (Claims):**

```json
{"iss":"https://auth.example.com","sub":"user-8842","aud":"https://api.example.com","exp":1717027200,"iat":1717023600,"jti":"unique-token-id-9f3a","scope":"read:orders write:orders","roles":["order-manager"]}
```

### Registered Claims

| Claim | Name | Description |
|---|---|---|
| `iss` | Issuer | Who issued the token |
| `sub` | Subject | Who the token is about |
| `aud` | Audience | Intended recipient(s) |
| `exp` | Expiration | When the token expires (Unix timestamp) |
| `nbf` | Not Before | Token is not valid before this time |
| `iat` | Issued At | When the token was issued |
| `jti` | JWT ID | Unique identifier for the token |

### Signing Algorithms

| Algorithm | Type | Key | Recommended Use |
|---|---|---|---|
| `HS256` | Symmetric | Shared secret | Single-service, internal use |
| `RS256` | Asymmetric | RSA key pair | Multi-service, public verification |
| `ES256` | Asymmetric | ECDSA key pair | Multi-service, smaller key size |
| `none` | — | — | ❌ Never use in production |

Asymmetric algorithms (RS256, ES256) are preferred because any service can verify tokens using the public key without holding the private signing key.

### Token Validation

Every API that receives a JWT must validate it before trusting its claims:

1. **Decode** the header and verify the `alg` claim matches an expected algorithm.
2. **Verify the signature** using the appropriate key (from the JWKS endpoint for asymmetric algorithms).
3. **Check `exp`** — reject expired tokens.
4. **Check `iss`** — confirm the issuer matches your authorization server.
5. **Check `aud`** — confirm the audience includes your API's identifier.
6. **Check scopes or roles** — ensure the token grants permission for the requested action.

### JWT Best Practices

- ✅ Keep tokens short-lived (5–15 minutes for access tokens)
- ✅ Use asymmetric signing (RS256 or ES256) for distributed systems
- ✅ Validate **all** registered claims, not just the signature
- ✅ Rotate signing keys regularly and publish them via a JWKS endpoint
- ❌ Do not store sensitive data in JWT payloads — they are Base64-encoded, not encrypted
- ❌ Do not accept the `alg` field without validation — prevents algorithm confusion attacks

---

## Bearer Tokens and Token Management

Bearer tokens are the most common token type in OAuth 2.0. Any party that possesses the token can use it — there is no proof-of-possession mechanism — so transport security is essential.

### Sending Bearer Tokens

```http
GET /api/orders/42 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

### Access Tokens vs Refresh Tokens

| Aspect | Access Token | Refresh Token |
|---|---|---|
| **Purpose** | Authorize API requests | Obtain new access tokens |
| **Lifetime** | Short (5-15 minutes) | Long (hours to days) |
| **Sent to** | Resource server | Authorization server only |
| **Storage** | Memory (preferred) | Secure HTTP-only cookie or encrypted storage |
| **Revocable** | Difficult (stateless JWT) | Yes (server-side check) |

### Token Refresh Flow

```http
POST /oauth/token HTTP/1.1
Host: auth.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&refresh_token=dGhpcyBpcyBhIHJlZnJlc2g...&client_id=my-app
```

```json
{"access_token":"eyJhbGci...(new)","token_type":"Bearer","expires_in":900,"refresh_token":"xRkN4a9s...(rotated)"}
```

Implement **refresh token rotation**: each time a refresh token is used, issue a new one and invalidate the old one. If a revoked refresh token is presented, revoke the entire token family to mitigate replay attacks.

### Token Revocation

RFC 7009 defines a standard token revocation endpoint. Clients call it when a user logs out or a token is no longer needed. The server responds with `200 OK` regardless of whether the token was valid, to prevent token enumeration.

```http
POST /oauth/revoke HTTP/1.1
Host: auth.example.com
Content-Type: application/x-www-form-urlencoded

token=eyJhbGciOiJSUzI1NiIs...&token_type_hint=access_token
```

---

## Mutual TLS (mTLS)

Mutual TLS extends the standard TLS handshake so that both the client and the server present certificates. It is the strongest authentication mechanism for service-to-service communication in zero-trust environments.

### How mTLS Works

```
Client                              Server
  |  (1) ClientHello ------------->  |
  |  (2) ServerHello + Server cert <-|
  |  (3) Client certificate ------>  |
  |  (4) Mutual verification        |
  |  <== Encrypted channel ==>      |
```

### When to Use mTLS

- ✅ Service-to-service communication within a service mesh (e.g., Istio, Linkerd)
- ✅ Zero-trust networks where network location is not sufficient proof of identity
- ✅ Regulated environments requiring strong mutual authentication (PCI-DSS, financial services)
- ❌ Not practical for browser-based or mobile clients

### Certificate-Bound Access Tokens

RFC 8705 defines certificate-bound access tokens, which tie an OAuth access token to the client's TLS certificate. The resource server verifies the certificate thumbprint in the token matches the one presented during TLS.

```json
{
  "sub": "service-inventory",
  "iss": "https://auth.example.com",
  "cnf": {
    "x5t#S256": "bwcK0esc3ACC3DB2Y5_lESsXE8o9ltc05O89jdN-dg2"
  }
}
```

---

## API Security Headers

HTTP headers play a critical role in securing API communication.

### CORS (Cross-Origin Resource Sharing)

CORS controls which origins can access your API from a browser.

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400
```

- ❌ **Never use `Access-Control-Allow-Origin: *` with credentials** — this is forbidden by the CORS specification.
- ✅ Whitelist specific origins rather than reflecting the `Origin` header blindly.

### Rate Limiting Headers

Rate limiting headers communicate quota information to clients:

```http
HTTP/1.1 200 OK
RateLimit-Limit: 1000
RateLimit-Remaining: 742
RateLimit-Reset: 1717027200
```

When a client exceeds the rate limit:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30

{"type":"https://api.example.com/errors/rate-limit","title":"Rate limit exceeded"}
```

### Additional Security Headers

| Header | Value | Purpose |
|---|---|---|
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains` | Force HTTPS |
| `X-Content-Type-Options` | `nosniff` | Prevent MIME type sniffing |
| `X-Frame-Options` | `DENY` | Prevent clickjacking |
| `Cache-Control` | `no-store` | Prevent caching of sensitive responses |
| `Content-Security-Policy` | `default-src 'none'` | Restrict content loading |

---

## Scopes and Permissions

Scopes and permissions define what actions an authenticated caller is allowed to perform. Two dominant models exist: Role-Based Access Control (RBAC) and Attribute-Based Access Control (ABAC).

### OAuth Scopes

Scopes are coarse-grained permissions that limit what an access token can do. They follow a `resource:action` convention (e.g., `orders:read`, `users:admin`, `billing:manage`). A token's effective permissions are the intersection of the scopes granted to the token and the permissions assigned to the user or service.

### Role-Based Access Control (RBAC)

RBAC assigns permissions to roles, and roles to users. It is simple and effective for systems with well-defined user categories.

```json
{
  "sub": "user-8842",
  "roles": ["order-manager"],
  "scope": "orders:read orders:write"
}
```

| Role | Permissions |
|---|---|
| `viewer` | `orders:read` |
| `order-manager` | `orders:read`, `orders:write` |
| `admin` | `orders:read`, `orders:write`, `users:admin` |

### Attribute-Based Access Control (ABAC)

ABAC evaluates access based on attributes of the subject, resource, action, and environment. It supports fine-grained policies that RBAC cannot express, such as "allow engineers to read internal documents during business hours."

### RBAC vs ABAC

| Aspect | RBAC | ABAC |
|---|---|---|
| **Complexity** | Low | High |
| **Granularity** | Coarse (role-level) | Fine (attribute-level) |
| **Scalability** | Role explosion in large systems | Scales with policy rules |
| **Auditability** | Easy — check role assignments | Harder — evaluate policy conditions |
| **Best for** | Well-defined user hierarchies | Dynamic, context-aware decisions |

---

## Comparison of Authentication Methods

| Method | Identity | Credential Type | Best For | Revocability | Complexity |
|---|---|---|---|---|---|
| API Key | Application | Static string | Simple integrations, public APIs | Immediate (server-side) | Low |
| OAuth 2.0 | User or Service | Bearer token | Delegated access, third-party apps | Token expiry + revocation | Medium–High |
| OIDC | User | ID token + access token | User authentication + API access | Token expiry + revocation | High |
| JWT (self-contained) | User or Service | Signed token | Stateless validation, microservices | Difficult (until expiry) | Medium |
| mTLS | Service | X.509 certificate | Service-to-service, zero trust | Certificate revocation (CRL/OCSP) | High |
| Basic Auth | User | Username + password | Internal tools, dev/test only | Immediate (server-side) | Low |

### Decision Guide

Use this flowchart to select the right authentication mechanism:

- **Service-to-service, no user context, zero-trust required** --> mTLS or mTLS + OAuth Client Credentials
- **Service-to-service, no user context, standard network** --> OAuth Client Credentials
- **User-facing, third-party application** --> OAuth 2.0 Authorization Code
- **User-facing, SPA or mobile app** --> Authorization Code + PKCE
- **User-facing, server-side app** --> Authorization Code (confidential client)
- **Application identification only** --> API Key

---

## Security Best Practices

### Token Storage

| Environment | Recommended Storage | Avoid |
|---|---|---|
| Server-side app | Encrypted at rest, in-memory when active | Plaintext files, logs |
| Single-page app | Memory (JavaScript variable) | `localStorage`, `sessionStorage` |
| Mobile app | OS secure storage (Keychain / Keystore) | Shared preferences |
| Browser (refresh) | `HttpOnly`, `Secure`, `SameSite=Strict` cookie | `localStorage` |

### HTTPS Everywhere

All API traffic must be encrypted with TLS 1.2 or higher. Enforce with HSTS headers and redirect HTTP to HTTPS at the infrastructure level.

### Credential Rotation

- Rotate API keys on a regular schedule (e.g., every 90 days) and immediately upon suspected compromise.
- Rotate JWT signing keys periodically; publish the new public key to JWKS before retiring the old one.
- Use the `kid` (Key ID) header in JWTs to support graceful key rollover.

### Defense in Depth

1. **Validate all inputs** — authentication does not replace input validation.
2. **Apply rate limiting** — protect against brute-force and credential-stuffing attacks.
3. **Log authentication events** — record successes, failures, and token issuance for audit trails.
4. **Use short-lived tokens** — minimize the window of exposure if a token is compromised.
5. **Enforce least privilege** — grant only the scopes and permissions required for the task.

### Common Mistakes

- ❌ Accepting JWTs without verifying the signature
- ❌ Hardcoding secrets in source code or version-controlled configuration
- ❌ Using symmetric JWT signing (HS256) across multiple services
- ❌ Logging tokens or credentials in application logs
- ❌ Returning detailed error messages that distinguish "user not found" from "wrong password"

---

## Next Steps

Return to the [API Design Overview](00-OVERVIEW.md) for the full topic map, or revisit [RESTful API Design](01-REST.md) to see how authentication integrates with REST conventions, or [API Versioning](04-API-VERSIONING.md) to understand how security contracts evolve across API versions.

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial API authentication and authorization documentation |
