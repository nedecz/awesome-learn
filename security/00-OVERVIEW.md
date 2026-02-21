# Security Fundamentals

## Table of Contents

1. [Overview](#overview)
2. [What is Application Security](#what-is-application-security)
3. [The CIA Triad](#the-cia-triad)
4. [Core Security Principles](#core-security-principles)
5. [Threat Modeling](#threat-modeling)
6. [OWASP Top 10](#owasp-top-10)
7. [Attack Surfaces](#attack-surfaces)
8. [Security Layers](#security-layers)
9. [Risk Assessment](#risk-assessment)
10. [Security in the Software Development Lifecycle](#security-in-the-software-development-lifecycle)
11. [Prerequisites](#prerequisites)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

This documentation provides a comprehensive introduction to application security fundamentals. It covers the core principles, threat modeling methodologies, common vulnerability categories, risk assessment frameworks, and the foundational knowledge needed to build, deploy, and operate secure software systems.

### Target Audience

- **Developers** writing application code and designing systems that must resist attack
- **Security Engineers** building security programs, tooling, and processes for organizations
- **DevOps Engineers** embedding security into CI/CD pipelines and infrastructure
- **Architects** designing systems with security as a first-class requirement
- **Engineering Managers** building security-aware teams and processes

### Scope

- What application security is and why it is a shared responsibility across engineering
- The CIA triad: Confidentiality, Integrity, and Availability
- Core security principles: least privilege, defense in depth, fail-safe defaults, zero trust
- Threat modeling: STRIDE, DREAD, attack trees, and data flow diagrams
- The OWASP Top 10 web application security risks
- Attack surfaces: understanding and minimizing your exposure
- Security layers: network, application, data, and operational security
- Risk assessment: likelihood, impact, and prioritization
- Integrating security into the software development lifecycle (SDLC)

---

## What is Application Security

**Application security** (AppSec) is the practice of finding, fixing, and preventing security vulnerabilities in software applications throughout their entire lifecycle — from design and development through deployment and operations. It encompasses the tools, techniques, processes, and mindset required to build software that resists attack and protects the data and users it serves.

### Why Security Matters

Security breaches are not abstract risks — they are measurable business events with quantifiable consequences:

- **Financial impact** — The average cost of a data breach in 2024 is $4.88 million (IBM Cost of a Data Breach Report). This includes detection, response, notification, lost business, and regulatory fines.
- **Reputation damage** — Customer trust takes years to build and moments to lose. Breached companies experience measurable drops in customer retention and revenue growth.
- **Regulatory penalties** — GDPR fines can reach 4% of annual global revenue. PCI DSS non-compliance can result in increased transaction fees and loss of card processing rights.
- **Operational disruption** — Ransomware attacks caused an average of 24 days of downtime per incident. Recovery costs often exceed the ransom itself.
- **Legal liability** — Organizations face lawsuits from affected customers, partners, and shareholders. Executives can be held personally liable for negligent security practices.

### The Shared Responsibility Model

Security is not the sole responsibility of a dedicated security team. In modern engineering organizations, security is a shared responsibility:

```
┌─────────────────────────────────────────────────────────────────┐
│                   Shared Responsibility Model                    │
│                                                                  │
│   Security Team        Platform/DevOps        Developers         │
│   ──────────────       ──────────────         ──────────────     │
│   - Threat modeling    - Infrastructure       - Secure coding    │
│   - Pen testing          hardening            - Input validation │
│   - Incident response  - Secret management    - Auth integration │
│   - Policy & standards - Network security     - Code review      │
│   - Compliance         - Monitoring &         - Dependency       │
│   - Security tooling     alerting               management      │
│   - Vulnerability      - Access control       - Unit testing     │
│     management         - Patch management       for security     │
│                                                                  │
│          Everyone: Report incidents │ Follow policies            │
└─────────────────────────────────────────────────────────────────┘
```

---

## The CIA Triad

The **CIA triad** is the foundational model for information security. Every security control, decision, and risk assessment ultimately maps back to one or more of these three properties.

### Confidentiality

**Confidentiality** ensures that information is accessible only to those authorized to access it. Breaches of confidentiality result in unauthorized disclosure of sensitive data.

**Controls that protect confidentiality:**

- **Encryption** — Encrypt data at rest (AES-256) and in transit (TLS 1.3) so that intercepted data is unreadable
- **Access controls** — Implement RBAC/ABAC to restrict who can read sensitive resources
- **Authentication** — Verify the identity of users and services before granting access
- **Data classification** — Label data by sensitivity level (public, internal, confidential, restricted) and apply appropriate controls
- **Data masking** — Mask or tokenize sensitive fields (PII, PAN) in logs, analytics, and non-production environments

```
Confidentiality Controls
────────────────────────
Encryption at rest     → AES-256-GCM for stored data
Encryption in transit  → TLS 1.3 for all network communication
Access control         → RBAC with least-privilege assignments
Authentication         → MFA for all human access, mTLS for services
Data classification    → Public → Internal → Confidential → Restricted
Data masking           → Mask PII in logs and non-production environments
```

### Integrity

**Integrity** ensures that information is accurate, complete, and has not been tampered with. Breaches of integrity result in unauthorized modification of data or systems.

**Controls that protect integrity:**

- **Hashing** — Use cryptographic hashes (SHA-256) to detect unauthorized changes to files, configurations, and artifacts
- **Digital signatures** — Sign code, artifacts, and communications to prove authenticity and detect tampering
- **Version control** — Track every change with full audit history (Git, immutable audit logs)
- **Input validation** — Reject malformed input that could corrupt data or exploit processing logic
- **Checksums** — Verify integrity of downloaded files, packages, and container images

### Availability

**Availability** ensures that information and systems are accessible to authorized users when needed. Breaches of availability result in denial of service.

**Controls that protect availability:**

- **Redundancy** — Deploy across multiple availability zones and regions
- **Load balancing** — Distribute traffic to prevent single points of failure
- **Rate limiting** — Protect services from abuse and denial-of-service attacks
- **Backups** — Maintain tested, encrypted backups with defined recovery time objectives (RTO)
- **DDoS protection** — Use CDNs, WAFs, and cloud-native DDoS mitigation services
- **Health monitoring** — Detect and respond to availability issues before users are affected

---

## Core Security Principles

### Least Privilege

Grant only the minimum permissions necessary for a user, service, or process to perform its intended function. Revoke access when it is no longer needed.

```
Least Privilege Examples
────────────────────────
Database access   → Application uses a read-only account; only the migration
                    service has write access to the schema
API keys          → Each service gets its own key with scoped permissions;
                    no shared "admin" keys
IAM roles         → Lambda functions have roles that permit only the specific
                    S3 bucket and DynamoDB table they need
SSH access        → Engineers use just-in-time access that expires after 8 hours;
                    no permanent SSH keys on production servers
```

### Defense in Depth

Apply multiple layers of security controls so that if one layer fails, the next layer provides protection. No single control is sufficient on its own.

```
Defense in Depth Layers
───────────────────────
┌─────────────────────────────────────────┐
│          Network Layer                   │
│  Firewalls, WAF, DDoS protection,       │
│  network segmentation, VPN              │
│  ┌─────────────────────────────────┐    │
│  │      Application Layer          │    │
│  │  Authentication, authorization, │    │
│  │  input validation, CSRF tokens  │    │
│  │  ┌─────────────────────────┐    │    │
│  │  │     Data Layer          │    │    │
│  │  │  Encryption, hashing,   │    │    │
│  │  │  data masking, backups  │    │    │
│  │  └─────────────────────────┘    │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

### Fail-Safe Defaults

Systems should deny access by default and require explicit grants. When a security control fails, it should fail closed (deny) rather than fail open (allow).

**Examples:**

- A firewall rule should block all traffic by default and explicitly allow only what is needed
- An authorization check should return "deny" if the policy engine is unavailable
- A new user account should have zero permissions until roles are explicitly assigned
- An API endpoint should require authentication unless explicitly marked as public

### Separation of Duties

No single person or system should control all aspects of a critical process. Require multiple approvals or actors for sensitive operations.

**Examples:**

- Code changes require a separate reviewer before merging to production
- Database schema changes require approval from both the development team and the DBA
- Production deployments require a separate approval from the person who authored the change
- Key management operations (generation, rotation, revocation) require multiple authorized parties

### Zero Trust

Never trust, always verify. Do not assume that any user, device, or network is inherently trustworthy — regardless of whether they are inside or outside the network perimeter.

**Core principles of zero trust:**

- **Verify explicitly** — Always authenticate and authorize based on all available data points (identity, location, device health, service, workload, data classification)
- **Use least-privilege access** — Limit access with just-in-time and just-enough-access (JIT/JEA), risk-based adaptive policies, and data protection
- **Assume breach** — Minimize blast radius with segmentation, detect threats in real time, and improve defenses continuously

```
Traditional Perimeter Security vs Zero Trust
─────────────────────────────────────────────
Traditional:
  "Inside the firewall = trusted"
  ┌──────────────────────┐
  │   Corporate Network  │     ← Everything inside is trusted
  │   ┌────┐ ┌────┐     │
  │   │ DB │ │ App│     │     ← No verification between services
  │   └────┘ └────┘     │
  └──────────────────────┘

Zero Trust:
  "Never trust, always verify"
  ┌──────────────────────┐
  │   ┌────┐   ┌────┐   │
  │   │ DB │◄─►│ App│   │     ← mTLS between every service
  │   └────┘   └────┘   │     ← Identity verified on every request
  │      ▲         ▲     │     ← Network location is irrelevant
  │      │         │     │
  │   Policy    Policy   │     ← Access decisions are continuous
  └──────────────────────┘
```

---

## Threat Modeling

**Threat modeling** is a structured process for identifying potential security threats to a system, determining their likelihood and impact, and defining mitigations. It should be performed during design and revisited as the system evolves.

### When to Perform Threat Modeling

- During initial system design or architecture review
- When adding a new feature that introduces a new data flow, integration, or trust boundary
- When a significant security incident reveals a previously unidentified threat
- Periodically (at least annually) for critical systems

### The STRIDE Model

**STRIDE** is a threat classification framework developed by Microsoft. It categorizes threats by the type of violation they represent:

| Category | Violation | Description | Example |
|----------|-----------|-------------|---------|
| **S**poofing | Authentication | Pretending to be another user or system | Stolen credentials used to access an API |
| **T**ampering | Integrity | Modifying data or code without authorization | SQL injection altering database records |
| **R**epudiation | Non-repudiation | Denying an action without proof | User denies placing an order, no audit log exists |
| **I**nformation Disclosure | Confidentiality | Exposing data to unauthorized parties | Error messages revealing stack traces and internal IPs |
| **D**enial of Service | Availability | Making a system unavailable | Unauthenticated endpoint exploited for resource exhaustion |
| **E**levation of Privilege | Authorization | Gaining access beyond what is authorized | Regular user exploits IDOR to access admin endpoints |

### Data Flow Diagrams

A data flow diagram (DFD) maps how data moves through your system, identifying trust boundaries, data stores, processes, and external entities. It is the foundation of a threat model.

```
Data Flow Diagram Elements
──────────────────────────
┌──────────┐     External entity (user, third-party service)
│  Entity  │
└──────────┘

╔══════════╗     Process (your application code, service)
║ Process  ║
╚══════════╝

═══════════      Data store (database, file system, cache)
│ DB Store │
═══════════

───────────►     Data flow (HTTP request, message, query)

┄┄┄┄┄┄┄┄┄┄┄     Trust boundary (network perimeter, auth boundary)
```

### Threat Modeling Process

1. **Decompose the system** — Create data flow diagrams showing processes, data stores, data flows, and trust boundaries
2. **Identify threats** — Apply STRIDE to each element in the DFD; for each data flow crossing a trust boundary, ask what could go wrong
3. **Assess risk** — Rate each threat by likelihood and impact using a risk matrix
4. **Define mitigations** — For each threat, identify the security control that reduces its likelihood or impact
5. **Validate** — Verify that mitigations are implemented and effective; revisit the model regularly

---

## OWASP Top 10

The **OWASP Top 10** is a standard awareness document for web application security. It represents a broad consensus about the most critical security risks to web applications, updated periodically based on data from hundreds of organizations.

### A01: Broken Access Control

Access control enforces that users cannot act outside their intended permissions. Failures lead to unauthorized information disclosure, modification, or destruction of data.

**Common vulnerabilities:**

- Bypassing access control checks by modifying the URL, application state, or HTML page
- Insecure direct object references (IDOR) — changing the `id` parameter to access another user's resources
- Missing access control for API endpoints (POST, PUT, DELETE)
- Elevation of privilege — acting as an admin when logged in as a standard user
- CORS misconfiguration allowing unauthorized API access

**Mitigations:**

- Deny by default — all requests require explicit access control checks
- Implement access control mechanisms once and reuse throughout the application
- Enforce record-level ownership — users can only access their own records unless explicitly authorized
- Disable directory listing and ensure metadata files (.git, .env) are not served
- Log access control failures and alert on repeated violations
- Rate-limit API access to minimize damage from automated attacks

### A02: Cryptographic Failures

Failures related to cryptography that lead to exposure of sensitive data. This includes using weak algorithms, improper key management, and failure to encrypt sensitive data.

**Common vulnerabilities:**

- Transmitting data in clear text (HTTP, SMTP without TLS)
- Using deprecated cryptographic algorithms (MD5, SHA-1, DES, RC4)
- Using hard-coded or default encryption keys
- Not enforcing encryption (missing HSTS headers, optional TLS)
- Insufficient key rotation and management

**Mitigations:**

- Encrypt all data in transit with TLS 1.2+ (prefer TLS 1.3)
- Encrypt sensitive data at rest with AES-256-GCM
- Use strong, purpose-specific cryptographic algorithms
- Manage keys through dedicated key management services (AWS KMS, Azure Key Vault, HashiCorp Vault)
- Rotate keys on a defined schedule and on compromise

### A03: Injection

Injection flaws occur when untrusted data is sent to an interpreter as part of a command or query. The attacker's hostile data can trick the interpreter into executing unintended commands or accessing data without authorization.

**Common vulnerability types:**

- **SQL injection** — Malicious SQL sent through user input to manipulate database queries
- **Cross-site scripting (XSS)** — Injecting client-side scripts into web pages viewed by other users
- **Command injection** — Injecting OS commands through application parameters
- **LDAP injection** — Manipulating LDAP queries through user input
- **Template injection** — Exploiting server-side template engines to execute arbitrary code

**Mitigations:**

- Use parameterized queries (prepared statements) for all database access
- Use an ORM with parameterized query support
- Apply context-aware output encoding for all user-supplied data rendered in HTML, JavaScript, CSS, or URLs
- Validate and sanitize all input — reject unexpected characters and formats
- Use Content Security Policy (CSP) headers to mitigate XSS

### A04: Insecure Design

Security flaws at the design level that cannot be fixed by a perfect implementation. Insecure design is about missing or ineffective security controls at the architecture level.

**Mitigations:**

- Perform threat modeling during design and architecture reviews
- Use secure design patterns and reference architectures
- Write abuse cases and misuse stories alongside user stories
- Integrate security review into the design process, not just the code review process

### A05: Security Misconfiguration

The application or its infrastructure is improperly configured, leaving unnecessary features enabled, default credentials in place, or overly permissive settings.

**Common vulnerabilities:**

- Default credentials not changed
- Unnecessary features enabled (debug endpoints, sample applications, admin consoles)
- Overly permissive cloud IAM policies
- Missing security headers (HSTS, CSP, X-Content-Type-Options)
- Verbose error messages exposing stack traces and internal information

**Mitigations:**

- Implement a hardening process for all environments
- Use infrastructure as code with security baselines
- Automate configuration scanning (Checkov, tfsec, ScoutSuite)
- Remove unnecessary features, components, and documentation from production
- Review cloud permissions regularly with automated tools

### A06: Vulnerable and Outdated Components

Using components (libraries, frameworks, OS packages) with known vulnerabilities. This includes both direct and transitive dependencies.

**Mitigations:**

- Maintain an inventory of all components and their versions (SBOM)
- Continuously monitor for vulnerabilities (Dependabot, Snyk, Trivy)
- Remove unused dependencies
- Pin dependency versions and review updates before applying
- Subscribe to security advisories for critical dependencies

### A07: Identification and Authentication Failures

Weaknesses in authentication mechanisms that allow attackers to compromise passwords, keys, or session tokens, or to exploit implementation flaws to assume other users' identities.

**Mitigations:**

- Implement multi-factor authentication (MFA)
- Use strong password policies with bcrypt or Argon2 hashing
- Implement account lockout and rate limiting for login attempts
- Use secure session management with proper timeout and invalidation
- Never expose session identifiers in URLs

### A08: Software and Data Integrity Failures

Failures related to code and infrastructure that do not protect against integrity violations — such as insecure deserialization, using plugins from untrusted sources, or CI/CD pipelines without integrity verification.

**Mitigations:**

- Verify integrity of downloaded dependencies and artifacts (checksums, signatures)
- Use a trusted CI/CD pipeline with proper access controls
- Sign releases and verify signatures before deployment
- Validate serialized data against a strict schema

### A09: Security Logging and Monitoring Failures

Insufficient logging, detection, escalation, or active response allows attackers to further attack systems, maintain persistence, or tamper with data.

**Mitigations:**

- Log all authentication events (successes and failures)
- Log access control failures
- Log input validation failures with sufficient context
- Ensure logs are sent to a centralized, append-only log system
- Implement alerting for suspicious patterns
- Establish an incident response plan and test it regularly

### A10: Server-Side Request Forgery (SSRF)

SSRF flaws occur when a web application fetches a remote resource without validating the user-supplied URL. This allows an attacker to coerce the application to send crafted requests to unexpected destinations.

**Mitigations:**

- Validate and sanitize all client-supplied URLs
- Use allowlists for permitted URL schemes, hosts, and ports
- Do not send raw HTTP responses to clients
- Disable HTTP redirects for server-side requests
- Use network segmentation to limit server-side request targets

---

## Attack Surfaces

An **attack surface** is the sum of all points where an unauthorized user can attempt to enter data to or extract data from a system. Minimizing the attack surface reduces the number of potential vulnerabilities an attacker can exploit.

### Types of Attack Surfaces

```
Attack Surface Categories
─────────────────────────
Network Attack Surface
  - Open ports and services
  - Public endpoints and APIs
  - DNS records and subdomains
  - Load balancers and proxies

Application Attack Surface
  - User input fields (forms, query params, headers, cookies)
  - File upload endpoints
  - Authentication and session endpoints
  - API endpoints (REST, GraphQL, gRPC)
  - WebSocket connections

Data Attack Surface
  - Databases and data stores
  - Backup and export files
  - Log files containing sensitive data
  - Configuration files with credentials

Human Attack Surface
  - Phishing and social engineering
  - Insider threats
  - Compromised credentials
  - Third-party vendor access
```

### Reducing Attack Surfaces

- Remove unused features, endpoints, and services
- Close unnecessary ports and disable unused protocols
- Minimize public-facing surface area (use private networking where possible)
- Apply the principle of least functionality — each component should provide only the minimum capabilities required
- Regularly audit and inventory your attack surface

---

## Security Layers

Security controls should be applied at every layer of the technology stack. Each layer provides independent protection that limits the blast radius of a breach at any other layer.

### Network Security

- Firewalls and security groups to control traffic flow
- Network segmentation to isolate sensitive systems
- VPNs and private networking for internal communication
- DDoS protection and rate limiting at the edge
- Web Application Firewalls (WAF) to filter malicious requests

### Application Security

- Secure coding practices (input validation, output encoding)
- Authentication and authorization at every entry point
- Session management with secure tokens
- Security headers (CSP, HSTS, X-Frame-Options)
- API security (rate limiting, input validation, authentication)

### Data Security

- Encryption at rest and in transit
- Data classification and handling policies
- Access controls on data stores
- Data masking in non-production environments
- Secure backup and recovery procedures

### Operational Security

- Monitoring and alerting for security events
- Incident response procedures
- Patch management and vulnerability scanning
- Access reviews and privilege audits
- Security training and awareness programs

---

## Risk Assessment

### Risk Matrix

Risk is the combination of the **likelihood** of a threat being exploited and the **impact** if it is. A risk matrix helps prioritize which threats to address first.

```
Risk Matrix
───────────

Impact ▲
       │
  High │  Medium    High      Critical
       │
Medium │  Low       Medium    High
       │
  Low  │  Low       Low       Medium
       │
       └──────────────────────────►
          Low      Medium     High
                                    Likelihood
```

### DREAD Risk Rating

**DREAD** is a risk assessment model that rates threats across five dimensions:

| Dimension | Question | Scale |
|-----------|----------|-------|
| **D**amage Potential | How much damage will the attack cause? | 1–10 |
| **R**eproducibility | How easy is it to reproduce the attack? | 1–10 |
| **E**xploitability | How much skill/effort is needed to exploit? | 1–10 |
| **A**ffected Users | How many users are impacted? | 1–10 |
| **D**iscoverability | How easy is it to discover the vulnerability? | 1–10 |

**Risk score** = (D + R + E + A + D) / 5

- **1–3:** Low risk — address in normal development cycle
- **4–6:** Medium risk — address in the next sprint
- **7–9:** High risk — address immediately
- **10:** Critical — stop other work and fix now

---

## Security in the Software Development Lifecycle

Security should be integrated into every phase of the SDLC, not bolted on at the end.

```
Secure SDLC
────────────
Phase              Security Activities
─────              ─────────────────────────────────────────────
Requirements       - Security requirements and compliance needs
                   - Abuse cases and misuse stories

Design             - Threat modeling (STRIDE)
                   - Security architecture review
                   - Secure design patterns

Development        - Secure coding practices
                   - SAST (static analysis)
                   - Code review with security focus
                   - Pre-commit hooks for secret detection

Testing            - DAST (dynamic analysis)
                   - Penetration testing
                   - Dependency scanning (SCA)
                   - Fuzz testing

Deployment         - Container image scanning
                   - Infrastructure as code scanning
                   - Secret scanning
                   - Artifact signing and verification

Operations         - Runtime monitoring and alerting
                   - Incident response
                   - Patch management
                   - Security audit logging
```

---

## Prerequisites

Before diving into the security topics, you should be familiar with:

- **Basic web development** — HTTP, REST APIs, client-server architecture
- **Programming fundamentals** — At least one backend language (Python, JavaScript, C#, Java, Go)
- **Command line basics** — Terminal, shell commands, file system navigation
- **Version control** — Git fundamentals (commits, branches, pull requests)
- **Basic networking** — TCP/IP, DNS, ports, TLS

---

## Next Steps

Continue your security learning journey:

| File | Topic | Description |
|---|---|---|
| [01-AUTHENTICATION.md](01-AUTHENTICATION.md) | Authentication | Password hashing, MFA, SSO, OAuth 2.0, session management |
| [02-AUTHORIZATION.md](02-AUTHORIZATION.md) | Authorization | RBAC, ABAC, policy engines, least privilege |
| [06-SECURE-CODING.md](06-SECURE-CODING.md) | Secure Coding | Input validation, injection prevention, secure defaults |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Learning Path | Structured curriculum with exercises |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Security Fundamentals documentation |
