# Microservices Security

## Table of Contents

1. [Overview](#overview)
2. [Zero Trust Architecture](#zero-trust-architecture)
3. [Authentication and Authorization](#authentication-and-authorization)
4. [Mutual TLS (mTLS)](#mutual-tls-mtls)
5. [API Security](#api-security)
6. [Secrets Management](#secrets-management)
7. [Network Security](#network-security)
8. [Supply Chain Security](#supply-chain-security)
9. [Security Checklist](#security-checklist)
10. [Next Steps](#next-steps)

## Overview

Microservices expand the attack surface compared to monoliths — there are more network calls, more endpoints, and more services to secure. Security must be built into every layer: network, transport, application, and data.

### Target Audience

- Security engineers designing microservices security architectures
- Developers implementing authentication and authorization
- Platform engineers configuring network policies and TLS

### Scope

- Zero trust networking principles
- OAuth2, OIDC, and JWT-based authentication
- Mutual TLS for service-to-service encryption
- API security patterns (rate limiting, input validation)
- Secrets management with Vault and cloud-native tools
- Container and supply chain security

## Zero Trust Architecture

Zero trust assumes that no service, user, or network is trusted by default — even if it is inside the perimeter. Every request must be authenticated, authorized, and encrypted.

### Principles

```
Traditional (perimeter-based)          Zero Trust

┌─────────────────────────────┐       ┌─────────────────────────────┐
│  Trusted Internal Network   │       │  No Implicit Trust          │
│                              │       │                              │
│  ┌────┐ ────▶ ┌────┐       │       │  ┌────┐ ─mTLS─▶ ┌────┐    │
│  │ A  │       │ B  │       │       │  │ A  │  +AuthZ  │ B  │    │
│  └────┘       └────┘       │       │  └────┘          └────┘    │
│                              │       │                              │
│  Once inside, everything    │       │  Every call is:             │
│  is trusted                  │       │  - Authenticated (who?)     │
└─────────────────────────────┘       │  - Authorized (allowed?)    │
                                       │  - Encrypted (mTLS)         │
                                       └─────────────────────────────┘
```

### Zero Trust Pillars

| Pillar | Description |
|---|---|
| **Verify explicitly** | Authenticate and authorize every request based on all available data |
| **Least privilege** | Grant minimum permissions needed for each service and user |
| **Assume breach** | Minimize blast radius; segment access; verify end-to-end encryption |

## Authentication and Authorization

### OAuth2 and OpenID Connect (OIDC)

OAuth2 handles authorization (what you can do), while OIDC adds authentication (who you are) on top of OAuth2.

```
                    ┌──────────────────┐
                    │  Identity Provider│
                    │  (Keycloak, Auth0,│
                    │   Okta, Azure AD) │
                    └────────┬─────────┘
                             │
            1. Authenticate  │  2. Issue tokens
               + Authorize   │     (access + ID token)
                             ▼
┌──────────┐         ┌──────────────┐         ┌──────────┐
│  Client  │──req──▶ │ API Gateway  │──req──▶ │ Service  │
│  (SPA,   │         │              │         │          │
│  mobile) │         │ Validate JWT │         │ Validate │
└──────────┘         │ + forward    │         │ claims   │
                     └──────────────┘         └──────────┘
```

### JWT Token Structure

```
Header.Payload.Signature

Payload (claims):
{
  "sub": "user-123",
  "email": "user@example.com",
  "roles": ["admin", "order-manager"],
  "scope": "read:orders write:orders",
  "iss": "https://auth.example.com",
  "aud": "order-service",
  "exp": 1705312800,
  "iat": 1705309200
}
```

### Service-to-Service Authentication

| Method | Description | Use Case |
|---|---|---|
| **mTLS** | Both client and server present certificates | Service mesh, high-security environments |
| **JWT with service identity** | Service obtains a JWT from the identity provider | OAuth2 client credentials flow |
| **API keys** | Static keys for service authentication | Simple internal services (less secure) |

## Mutual TLS (mTLS)

mTLS requires both the client and server to present TLS certificates. This ensures that both parties are who they claim to be.

```
Standard TLS (one-way):
  Client ──▶ Server presents certificate ──▶ Client verifies server

Mutual TLS (two-way):
  Client presents certificate ──▶ Server verifies client
  Server presents certificate ──▶ Client verifies server

  Both sides are authenticated.
```

### mTLS with Service Mesh

Service meshes like Istio and Linkerd automate mTLS between all services — no application code changes required.

```
┌──────────────────────────────────────┐
│  Istio Control Plane (istiod)        │
│  ┌────────────────────────────────┐  │
│  │  Certificate Authority (CA)    │  │
│  │  Issues and rotates certs      │  │
│  │  for every service proxy       │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
         │                    │
    cert │               cert │
         ▼                    ▼
  ┌─────────────┐     ┌─────────────┐
  │ ┌─────────┐ │     │ ┌─────────┐ │
  │ │ Envoy   │◀┼mTLS▶│ │ Envoy   │ │
  │ │ Proxy   │ │     │ │ Proxy   │ │
  │ ├─────────┤ │     │ ├─────────┤ │
  │ │ Order   │ │     │ │ Payment │ │
  │ │ Service │ │     │ │ Service │ │
  │ └─────────┘ │     │ └─────────┘ │
  └─────────────┘     └─────────────┘
```

## API Security

### Input Validation

Validate all input at the API boundary. Never trust data from other services or external clients.

- ✅ Validate request body schema (JSON Schema, Protobuf)
- ✅ Validate path and query parameters (type, length, range)
- ✅ Sanitize strings to prevent injection attacks
- ✅ Reject unknown fields in request bodies
- ❌ Do not rely solely on client-side validation

### Rate Limiting and Throttling

Protect services from abuse and denial-of-service attacks.

| Level | Scope | Implementation |
|---|---|---|
| **API Gateway** | Per-client, per-endpoint | Kong, AWS API Gateway rate limiting plugins |
| **Service level** | Per-service self-protection | Application-level rate limiter (token bucket) |
| **Infrastructure** | Global network-level | WAF, CDN rate limiting (Cloudflare, AWS WAF) |

### CORS and CSRF

- ✅ Configure strict CORS policies — allow only trusted origins
- ✅ Use CSRF tokens for state-changing operations from browsers
- ✅ Set secure cookie attributes (`HttpOnly`, `Secure`, `SameSite`)

## Secrets Management

### Where to Store Secrets

| Approach | Security Level | Use Case |
|---|---|---|
| **HashiCorp Vault** | High | Dynamic secrets, rotation, audit trail |
| **AWS Secrets Manager** | High | AWS-native, automatic rotation |
| **Azure Key Vault** | High | Azure-native, HSM-backed |
| **Kubernetes Secrets** | Medium | Base64-encoded; encrypt etcd at rest |
| **Environment variables** | Low | Simple but no rotation or audit |
| **Config files** | Very Low | ❌ Never store secrets in files checked into Git |

### Secret Injection Pattern

```
┌──────────────────┐     ┌──────────────────┐
│  Secret Store    │     │  Service Pod      │
│  (Vault)         │     │                   │
│                  │     │  ┌──────────────┐ │
│  DB_PASSWORD=... │────▶│  │ Vault Agent  │ │  (sidecar injects secrets)
│  API_KEY=...     │     │  │ (sidecar)    │ │
│                  │     │  └──────┬───────┘ │
└──────────────────┘     │         │          │
                          │  ┌──────▼───────┐ │
                          │  │ Application  │ │  (reads secrets from
                          │  │              │ │   file or env var)
                          │  └──────────────┘ │
                          └──────────────────┘
```

### Secret Rotation

- ✅ Rotate secrets automatically on a schedule (e.g., every 30 days)
- ✅ Use dynamic/short-lived credentials when possible
- ✅ Audit all secret access
- ❌ Never hardcode secrets in source code
- ❌ Never log secrets or include them in error messages

## Network Security

### Network Policies (Kubernetes)

Restrict traffic between services using Kubernetes NetworkPolicies.

```yaml
# Allow only frontend to reach the API service
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-service
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080
          protocol: TCP
```

### Defense in Depth

```
Layer 1: Network perimeter (WAF, DDoS protection)
Layer 2: API Gateway (authentication, rate limiting)
Layer 3: Service mesh (mTLS, authorization policies)
Layer 4: Application (input validation, business rules)
Layer 5: Data (encryption at rest, field-level encryption)
```

## Supply Chain Security

### Container Image Security

- ✅ Use minimal base images (distroless, Alpine)
- ✅ Scan images for vulnerabilities (Trivy, Grype, Snyk)
- ✅ Sign images with cosign or Notary
- ✅ Pin image digests instead of mutable tags
- ❌ Do not run containers as root
- ❌ Do not use `latest` tag in production

### Dependency Security

- ✅ Scan dependencies for known vulnerabilities (Dependabot, Snyk, OWASP)
- ✅ Pin dependency versions
- ✅ Use lock files (`package-lock.json`, `go.sum`, `Cargo.lock`)
- ✅ Review and audit third-party libraries before adoption

## Security Checklist

### Authentication & Authorization

- [ ] All external endpoints require authentication
- [ ] JWT tokens are validated (signature, expiry, audience, issuer)
- [ ] Service-to-service calls use mTLS or service-level JWTs
- [ ] Least-privilege RBAC is enforced per service

### Network

- [ ] All inter-service traffic is encrypted (mTLS or TLS)
- [ ] NetworkPolicies restrict traffic to only necessary paths
- [ ] External traffic goes through an API gateway with rate limiting
- [ ] No services are directly exposed to the internet without a proxy

### Secrets

- [ ] No secrets in source code or environment variable defaults
- [ ] Secrets are stored in a dedicated secrets manager (Vault, AWS SM)
- [ ] Secrets are rotated automatically
- [ ] Secret access is audited

### Containers

- [ ] Images are scanned for vulnerabilities before deployment
- [ ] Containers run as non-root users
- [ ] Read-only root filesystems where possible
- [ ] Image tags are pinned to digests

## Next Steps

Continue to [Observability](08-OBSERVABILITY.md) to learn about distributed tracing, centralized logging, metrics collection, and health check patterns for monitoring microservices in production.
