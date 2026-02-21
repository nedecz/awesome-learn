# Authorization

A comprehensive guide to authorization — covering access control models (RBAC, ABAC, ReBAC), policy engines, least privilege, permission design, API authorization, and zero trust access control patterns for modern applications and microservices.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Authorization Fundamentals](#authorization-fundamentals)
   - [What is Authorization](#what-is-authorization)
   - [Authorization vs Authentication](#authorization-vs-authentication)
   - [Key Terminology](#key-terminology)
3. [Access Control Models](#access-control-models)
   - [Role-Based Access Control (RBAC)](#role-based-access-control-rbac)
   - [Attribute-Based Access Control (ABAC)](#attribute-based-access-control-abac)
   - [Relationship-Based Access Control (ReBAC)](#relationship-based-access-control-rebac)
   - [Model Comparison](#model-comparison)
4. [Permission Design](#permission-design)
   - [Permission Granularity](#permission-granularity)
   - [Resource-Level Permissions](#resource-level-permissions)
   - [Hierarchical Permissions](#hierarchical-permissions)
   - [Deny-by-Default](#deny-by-default)
5. [Policy Engines](#policy-engines)
   - [Open Policy Agent (OPA)](#open-policy-agent-opa)
   - [Cedar](#cedar)
   - [Casbin](#casbin)
   - [Choosing a Policy Engine](#choosing-a-policy-engine)
6. [API Authorization](#api-authorization)
   - [OAuth 2.0 Scopes](#oauth-20-scopes)
   - [JWT Claims-Based Authorization](#jwt-claims-based-authorization)
   - [API Gateway Authorization](#api-gateway-authorization)
7. [Microservices Authorization](#microservices-authorization)
   - [Centralized vs Distributed](#centralized-vs-distributed)
   - [Token Propagation](#token-propagation)
   - [Service-to-Service Authorization](#service-to-service-authorization)
8. [Least Privilege](#least-privilege)
   - [Principle of Least Privilege](#principle-of-least-privilege)
   - [Just-in-Time Access](#just-in-time-access)
   - [Privilege Escalation Prevention](#privilege-escalation-prevention)
9. [Access Reviews and Auditing](#access-reviews-and-auditing)
   - [Periodic Access Reviews](#periodic-access-reviews)
   - [Audit Logging](#audit-logging)
   - [Entitlement Management](#entitlement-management)
10. [Common Authorization Vulnerabilities](#common-authorization-vulnerabilities)
    - [Insecure Direct Object References (IDOR)](#insecure-direct-object-references-idor)
    - [Broken Function-Level Authorization](#broken-function-level-authorization)
    - [Mass Assignment](#mass-assignment)
    - [Privilege Escalation](#privilege-escalation)
11. [Next Steps](#next-steps)
12. [Version History](#version-history)

---

## Overview

Authorization determines **what an authenticated entity is allowed to do**. While authentication verifies identity ("who are you?"), authorization enforces permissions ("what can you access?"). Authorization failures are among the most critical and common security vulnerabilities — Broken Access Control is the #1 risk in the OWASP Top 10.

This document covers access control models (RBAC, ABAC, ReBAC), policy engines, least privilege, API authorization, microservices authorization patterns, and common authorization vulnerabilities.

### Target Audience

- **Developers** implementing access control checks in application code
- **Architects** designing authorization models for multi-service systems
- **Security Engineers** auditing authorization implementations and policies
- **Platform Engineers** building shared authorization services and policy infrastructure

### Scope

- Authorization fundamentals and terminology
- Access control models: RBAC, ABAC, ReBAC
- Permission design: granularity, hierarchies, deny-by-default
- Policy engines: OPA, Cedar, Casbin
- API authorization: OAuth scopes, JWT claims, API gateways
- Microservices authorization patterns
- Least privilege, just-in-time access, and privilege escalation prevention
- Access reviews, audit logging, and entitlement management
- Common authorization vulnerabilities and mitigations

---

## Authorization Fundamentals

### What is Authorization

Authorization is the process of determining whether an authenticated entity (user, service, or system) has permission to perform a specific action on a specific resource. It is evaluated on every request, not just at login.

```
Authorization Decision Flow
────────────────────────────
Request → Who (subject) wants to do What (action) on Which (resource)?

┌─────────┐     ┌──────────────┐     ┌──────────┐
│ Subject  │────►│  Policy      │────►│ Decision │
│ (user,   │     │  Engine      │     │ Allow /  │
│  service)│     │              │     │ Deny     │
└─────────┘     └──────────────┘     └──────────┘
                      ▲
                      │
               ┌──────┴───────┐
               │   Policy     │
               │   (rules,    │
               │    roles,    │
               │    attributes)│
               └──────────────┘
```

### Authorization vs Authentication

| Aspect | Authentication | Authorization |
|--------|---------------|---------------|
| Question answered | Who are you? | What can you do? |
| When it happens | First (identity verification) | After authentication |
| HTTP status on failure | 401 Unauthorized | 403 Forbidden |
| Mechanism | Credentials, tokens, certificates | Roles, permissions, policies |
| Frequency | Once per session (or token lifetime) | Every request |

### Key Terminology

| Term | Definition |
|------|-----------|
| **Subject** | The entity requesting access (user, service, API client) |
| **Resource** | The object being accessed (document, API endpoint, database record) |
| **Action** | The operation being performed (read, write, delete, execute) |
| **Policy** | A rule or set of rules that defines who can do what on which resource |
| **Permission** | A specific allowed action on a resource (e.g., `documents:read`) |
| **Role** | A named collection of permissions (e.g., `editor` = read + write) |
| **Scope** | An OAuth 2.0 concept — a permission string associated with an access token |
| **Principal** | The authenticated identity making the request |
| **Entitlement** | A granted access right, often used in enterprise IAM contexts |

---

## Access Control Models

### Role-Based Access Control (RBAC)

**RBAC** assigns permissions to roles, and roles to users. Users inherit permissions through their role assignments. RBAC is the most widely used access control model due to its simplicity.

```
RBAC Model
──────────
Users ──► Roles ──► Permissions

Example:
  User: alice@example.com
    └── Role: editor
          ├── Permission: documents:read
          ├── Permission: documents:write
          └── Permission: documents:publish

  User: bob@example.com
    └── Role: viewer
          └── Permission: documents:read
```

**Advantages:**

- Simple to understand and implement
- Easy to audit (which roles have which permissions)
- Well-supported by frameworks and identity providers

**Limitations:**

- Role explosion — as requirements grow, the number of roles multiplies
- Coarse-grained — RBAC alone cannot express "Alice can edit only her own documents"
- Static — roles do not consider context (time, location, device, resource attributes)

### Attribute-Based Access Control (ABAC)

**ABAC** makes access decisions based on attributes of the subject, resource, action, and environment. It is more flexible than RBAC and can express fine-grained, context-aware policies.

```
ABAC Policy Structure
─────────────────────
Subject Attributes    → user.role, user.department, user.clearance_level
Resource Attributes   → document.classification, document.owner, document.department
Action Attributes     → action.type (read, write, delete)
Environment Attributes → time.current_hour, network.is_internal, request.ip_country

Policy Example:
  ALLOW if:
    subject.department == resource.department
    AND subject.clearance_level >= resource.classification
    AND action.type IN ["read", "write"]
    AND environment.time BETWEEN 09:00 AND 18:00
```

**Advantages:**

- Fine-grained, context-aware access decisions
- No role explosion — attributes combine dynamically
- Can express complex policies (department-based, time-based, location-based)

**Limitations:**

- More complex to implement and audit
- Requires well-defined attribute schemas
- Policy debugging can be challenging

### Relationship-Based Access Control (ReBAC)

**ReBAC** makes access decisions based on the relationships between subjects and resources. It is particularly suited to collaborative applications (Google Docs, social networks) where access depends on who owns, shares, or is connected to a resource.

```
ReBAC Model
───────────
Access is derived from a relationship graph:

  alice ──owner──► document-1      → alice can edit document-1
  alice ──member──► team-eng       → alice inherits team-eng permissions
  team-eng ──viewer──► project-x   → team-eng members can view project-x
  bob ──viewer──► document-1       → bob was explicitly shared read access
```

**Advantages:**

- Natural fit for collaborative and social applications
- Handles nested and inherited access (org → team → project → document)
- Scales to millions of relationships with systems like Google Zanzibar

**Limitations:**

- More complex infrastructure (requires a relationship graph store)
- Not necessary for simple, role-based scenarios
- Query performance depends on graph traversal depth

### Model Comparison

| Feature | RBAC | ABAC | ReBAC |
|---------|------|------|-------|
| Granularity | Coarse (role-level) | Fine (attribute-level) | Fine (relationship-level) |
| Context-aware | No | Yes | Yes |
| Complexity | Low | High | Medium-High |
| Best for | Admin panels, simple apps | Enterprise, compliance | Collaborative apps, sharing |
| Audit clarity | High | Medium | Medium |
| Scalability | High | Medium | High (with Zanzibar-style) |

---

## Permission Design

### Permission Granularity

Design permissions at the right level of granularity. Too coarse leads to over-privileged users; too fine leads to unmanageable policy complexity.

```
Permission Naming Convention
────────────────────────────
Format: resource:action

Examples:
  documents:read          → Read any document
  documents:write         → Create or update documents
  documents:delete        → Delete documents
  documents:publish       → Publish a draft document

  users:read              → View user profiles
  users:admin             → Manage user accounts

  billing:read            → View invoices
  billing:manage          → Create, update, cancel subscriptions
```

### Resource-Level Permissions

In many applications, you need to control access at the individual resource level — not just "can the user read documents" but "can the user read this specific document."

Implement resource-level checks at the data access layer:

- Check ownership: does the requesting user own this resource?
- Check explicit grants: has the resource been shared with this user?
- Check role-based access: does the user's role permit this action on this resource type?
- Never rely solely on client-side enforcement — always validate on the server

### Hierarchical Permissions

Organize permissions in a hierarchy where higher-level permissions imply lower-level ones:

```
Permission Hierarchy Example
─────────────────────────────
admin
  └── manage
        └── write
              └── read

If a user has "manage", they implicitly have "write" and "read".
If a user has "read", they only have "read".
```

### Deny-by-Default

Always start with zero permissions and explicitly grant what is needed:

- New users should have no permissions until roles are assigned
- New API endpoints should require authentication and authorization by default
- New resources should be private until explicitly shared
- When in doubt, deny — it is safer to grant access later than to revoke it after a breach

---

## Policy Engines

### Open Policy Agent (OPA)

**OPA** is an open-source, general-purpose policy engine that decouples policy decision-making from application code. Policies are written in **Rego**, a declarative language.

```
# Example Rego policy: allow editors to read and write documents
package documents

default allow := false

allow if {
    input.user.role == "editor"
    input.action in ["read", "write"]
}

allow if {
    input.user.role == "admin"
}
```

**Use cases:**

- API authorization (sidecar or middleware)
- Kubernetes admission control
- Infrastructure policy (Terraform plans, CI/CD gates)
- Data filtering (row-level security)

### Cedar

**Cedar** is a policy language developed by AWS, used by Amazon Verified Permissions. It is designed for fine-grained authorization with a focus on readability and formal verification.

```
// Cedar policy: allow editors to update documents they own
permit(
  principal in Role::"editor",
  action == Action::"update",
  resource
) when {
  resource.owner == principal
};
```

### Casbin

**Casbin** is a lightweight, open-source authorization library that supports multiple access control models (RBAC, ABAC, ACL) through a configurable model and policy definition.

```
# Casbin model definition (RBAC)
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[role_definition]
g = _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act
```

### Choosing a Policy Engine

| Criteria | OPA | Cedar | Casbin |
|----------|-----|-------|--------|
| Language | Rego (custom) | Cedar (custom) | Config-based |
| Deployment | Sidecar, library, service | AWS service, library | Library (embedded) |
| Complexity | Medium | Medium | Low |
| Best for | K8s, API gateways, infra | AWS workloads, fine-grained | Application-embedded RBAC |
| Formal verification | No | Yes | No |

---

## API Authorization

### OAuth 2.0 Scopes

OAuth scopes define the permissions associated with an access token. Use scopes to limit what an API client can do, independent of the user's permissions.

```
OAuth Scope Examples
────────────────────
read:documents        → Read access to documents API
write:documents       → Write access to documents API
admin:users           → Administrative access to users API
openid profile email  → Standard OIDC scopes for identity
```

### JWT Claims-Based Authorization

Use custom claims in JWTs to carry authorization information:

```json
{
  "sub": "user-123",
  "roles": ["editor"],
  "permissions": ["documents:read", "documents:write"],
  "org_id": "org-456",
  "exp": 1700000000
}
```

**Authorization check (server-side):**

1. Validate the JWT signature and expiry
2. Extract roles/permissions/scopes from claims
3. Check if the required permission is present for the requested action
4. For resource-level checks, verify the user has access to the specific resource

### API Gateway Authorization

API gateways can handle coarse-grained authorization before requests reach your services:

- Validate JWT tokens and reject expired or invalid tokens
- Check OAuth scopes against endpoint requirements
- Enforce rate limits per client or user
- Route requests to authorization services for fine-grained decisions

---

## Microservices Authorization

### Centralized vs Distributed

| Approach | Description | Pros | Cons |
|----------|-------------|------|------|
| **Centralized** | Single authorization service for all decisions | Consistent policies, single audit point | Single point of failure, latency |
| **Distributed** | Each service makes its own decisions | Low latency, independent | Policy inconsistency, harder to audit |
| **Hybrid** | Gateway handles coarse-grained; services handle fine-grained | Balance of consistency and performance | More complex architecture |

### Token Propagation

In a microservices architecture, propagate the user's identity through the call chain so that each service can make its own authorization decision:

```
Token Propagation Pattern
─────────────────────────
Client → API Gateway → Service A → Service B → Service C
         │              │           │           │
         └─ Validate    └─ Check    └─ Check    └─ Check
            JWT            perms       perms       perms
```

- Forward the original access token (or a derived internal token) to downstream services
- Each service validates the token and checks permissions for its own resources
- Never create a new, elevated token for internal calls unless explicitly required by the architecture

### Service-to-Service Authorization

For machine-to-machine calls (no user context), use:

- **Client credentials grant** — Each service authenticates with its own client ID/secret
- **mTLS** — Services authenticate via client certificates
- **Service mesh** — Istio, Linkerd provide identity and authorization at the network level

---

## Least Privilege

### Principle of Least Privilege

Every user, process, and service should have only the minimum permissions necessary to perform its function. Excessive permissions increase the blast radius of a compromise.

```
Least Privilege Checklist
─────────────────────────
✓ Default role has zero permissions
✓ Permissions are granted per resource, not globally
✓ Service accounts have scoped access (not admin)
✓ CI/CD tokens are scoped to the specific repository and actions needed
✓ Database connections use read-only accounts where writes are not needed
✓ Cloud IAM roles use condition-based policies (resource tags, time, IP)
✓ Access is time-bound where possible (JIT access)
```

### Just-in-Time Access

**Just-in-time (JIT) access** grants elevated permissions temporarily, for a specific purpose, and automatically revokes them after a defined period.

**Examples:**

- Production database access granted for 2 hours during an incident, then automatically revoked
- Admin role granted for a specific deployment, then revoked after the deployment completes
- SSH access to a production server granted with a 1-hour TTL

### Privilege Escalation Prevention

- Validate that users cannot modify their own roles or permissions
- Require separate authorization for role assignment (an admin cannot assign themselves higher privileges)
- Implement approval workflows for privilege changes
- Log and alert on all privilege escalation events

---

## Access Reviews and Auditing

### Periodic Access Reviews

Regularly review who has access to what:

- **Quarterly:** Review all access to production systems
- **Monthly:** Review access for high-privilege roles (admin, root)
- **On event:** Review access when employees change teams or leave the organization
- **Automated:** Use tooling to flag unused permissions and stale accounts

### Audit Logging

Log all authorization decisions for security monitoring and compliance:

- Log the subject, action, resource, and decision (allow/deny)
- Log the policy or rule that produced the decision
- Include timestamp, request ID, and source IP
- Send logs to a centralized, append-only log system
- Retain logs according to compliance requirements (typically 1–7 years)

### Entitlement Management

Track and manage all access rights across the organization:

- Maintain an inventory of all roles, permissions, and their assignments
- Automate provisioning and deprovisioning of access based on HR events (hire, transfer, termination)
- Implement a self-service access request workflow with approval
- Generate reports for compliance audits

---

## Common Authorization Vulnerabilities

### Insecure Direct Object References (IDOR)

An **IDOR** vulnerability occurs when an application exposes internal object references (database IDs, file paths) and does not verify that the requesting user is authorized to access the referenced object.

**Example:** Changing `/api/invoices/123` to `/api/invoices/124` to access another user's invoice.

**Prevention:**

- Always check that the requesting user is authorized to access the specific resource
- Use indirect references (UUIDs) instead of sequential IDs (reduces guessability, but does not replace authorization checks)
- Implement resource-level authorization at the data access layer

### Broken Function-Level Authorization

Missing authorization checks on specific functions or API endpoints — typically admin functions that are not exposed in the UI but are accessible via direct API calls.

**Prevention:**

- Enforce authorization on every API endpoint, not just the UI
- Use a consistent authorization middleware that applies to all routes by default
- Test authorization by calling admin endpoints with non-admin credentials

### Mass Assignment

An attacker modifies request parameters to set fields that should not be user-controllable (e.g., `role`, `is_admin`, `price`).

**Prevention:**

- Use allowlists (not denylists) for writable fields
- Define explicit DTOs/view models for each endpoint
- Never bind request data directly to database models

### Privilege Escalation

An attacker gains higher privileges than intended — either by exploiting a vulnerability or by manipulating authorization data.

**Prevention:**

- Validate roles and permissions server-side on every request
- Do not store authorization data in client-accessible storage (cookies, localStorage)
- Use signed tokens (JWT) for claims — never accept unsigned or tampered claims
- Implement separation of duties for privilege management

---

## Next Steps

Continue your security learning journey:

| File | Topic | Description |
|---|---|---|
| [01-AUTHENTICATION.md](01-AUTHENTICATION.md) | Authentication | Password hashing, MFA, SSO, session management |
| [03-CRYPTOGRAPHY.md](03-CRYPTOGRAPHY.md) | Cryptography | Encryption, TLS, key management |
| [06-SECURE-CODING.md](06-SECURE-CODING.md) | Secure Coding | Input validation, injection prevention |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Authorization documentation |
