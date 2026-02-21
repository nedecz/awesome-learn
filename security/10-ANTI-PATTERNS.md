# Security Anti-Patterns

A catalogue of the most common security mistakes — what they look like, why they are harmful, and exactly how to fix them. Use this document as a code-review checklist, a pre-production gate, or a team learning resource.

---

## Table of Contents

- [Introduction](#introduction)
- [Anti-Patterns Summary Table](#anti-patterns-summary-table)
- [1. Storing Secrets in Source Code](#1-storing-secrets-in-source-code)
- [2. Using Plaintext or Weak Password Hashing](#2-using-plaintext-or-weak-password-hashing)
- [3. Missing Input Validation](#3-missing-input-validation)
- [4. Excessive Permissions](#4-excessive-permissions)
- [5. Security Through Obscurity](#5-security-through-obscurity)
- [6. Missing Authentication on Internal APIs](#6-missing-authentication-on-internal-apis)
- [7. Logging Sensitive Data](#7-logging-sensitive-data)
- [8. Using Outdated Dependencies](#8-using-outdated-dependencies)
- [9. Insufficient Error Handling](#9-insufficient-error-handling)
- [10. No Incident Response Plan](#10-no-incident-response-plan)
- [11. Rolling Your Own Cryptography](#11-rolling-your-own-cryptography)
- [12. Trusting Client-Side Validation](#12-trusting-client-side-validation)
- [Quick Reference Checklist](#quick-reference-checklist)
- [Next Steps](#next-steps)
- [Version History](#version-history)

---

## Introduction

### Why Anti-Patterns Matter

Security anti-patterns are recurring practices that seem reasonable at first but create significant vulnerabilities over time. They persist because they are often the easiest path — hard-coding a secret is faster than configuring a secrets manager, skipping input validation saves development time, and granting admin access avoids complex permission modeling.

The patterns documented here represent real security failures. Each one is:

- **Seductive** — it felt like the right approach when first implemented
- **Harmful** — it creates vulnerabilities that attackers actively exploit
- **Fixable** — there is a well-understood better approach

### How to Use This Document

1. **Code review** — Reference specific anti-patterns when reviewing pull requests
2. **Pre-production checklist** — Use the [Quick Reference Checklist](#quick-reference-checklist) before deploying to production
3. **Incident retros** — After a security incident, check which anti-patterns contributed
4. **Team onboarding** — Assign this document to new engineers before they ship to production
5. **Periodic audit** — Run through the checklist quarterly for existing systems

---

## Anti-Patterns Summary Table

| # | Anti-Pattern | Severity | Impact |
|---|-------------|----------|--------|
| 1 | Storing Secrets in Source Code | 🔴 Critical | Credential exposure, full system compromise |
| 2 | Using Plaintext or Weak Password Hashing | 🔴 Critical | Account takeover, credential stuffing |
| 3 | Missing Input Validation | 🔴 Critical | SQL injection, XSS, command injection |
| 4 | Excessive Permissions | 🟠 High | Larger blast radius on compromise |
| 5 | Security Through Obscurity | 🟠 High | False sense of security, eventual breach |
| 6 | Missing Authentication on Internal APIs | 🟠 High | Lateral movement, data exposure |
| 7 | Logging Sensitive Data | 🟠 High | Data exposure via log systems |
| 8 | Using Outdated Dependencies | 🟠 High | Known vulnerability exploitation |
| 9 | Insufficient Error Handling | 🟡 Medium | Information disclosure, system instability |
| 10 | No Incident Response Plan | 🟡 Medium | Delayed response, greater damage |
| 11 | Rolling Your Own Cryptography | 🔴 Critical | Broken encryption, data exposure |
| 12 | Trusting Client-Side Validation | 🔴 Critical | Bypassed security controls |

---

## 1. Storing Secrets in Source Code

### The Anti-Pattern

Embedding API keys, database passwords, encryption keys, or other secrets directly in source code, configuration files committed to version control, or container images.

### Why It Happens

- "It's a private repository" — perceived low risk
- Quick prototyping that becomes permanent
- Lack of awareness about secrets management tools
- Copying patterns from tutorials that hard-code credentials

### Why It Is Harmful

- Secrets in source code are accessible to everyone with repository access — current and former employees, contractors, and anyone who gains access to the repository
- Git history preserves secrets even after they are removed from the current codebase
- Leaked repositories (accidental public repos, compromised accounts) expose all secrets at once
- Automated scanners continuously search public repositories for secrets

### The Fix

- Use a dedicated secrets manager (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault)
- Inject secrets at runtime via environment variables or mounted files — never commit them
- Run secret scanning in CI/CD (GitLeaks, TruffleHog) and pre-commit hooks
- If a secret is committed, rotate it immediately — removing it from code is not sufficient
- Use git-secrets or similar pre-commit hooks to prevent accidental commits

---

## 2. Using Plaintext or Weak Password Hashing

### The Anti-Pattern

Storing passwords in plaintext, using reversible encryption, or using fast hash algorithms (MD5, SHA-256) without salt and work factor.

### Why It Happens

- Developers use general-purpose hashing (SHA-256) not knowing it is unsuitable for passwords
- Legacy systems that were built before modern password hashing standards
- "We encrypt passwords" — using reversible encryption instead of one-way hashing

### Why It Is Harmful

- Plaintext passwords are directly usable by anyone with database access
- Fast hashing algorithms (MD5, SHA-256) can be cracked at billions of hashes per second with GPUs
- Unsalted hashes are vulnerable to rainbow table attacks
- A database breach exposes all user credentials, enabling credential stuffing across other services

### The Fix

- Use **Argon2id** (preferred) or **bcrypt** for all password hashing
- Ensure unique per-password salts (most libraries handle this automatically)
- Configure an appropriate work factor (bcrypt cost 12+, Argon2id with recommended parameters)
- Migrate legacy hashed passwords on next login (hash the old hash with the new algorithm)
- Check passwords against breach databases (Have I Been Pwned) during registration and login

---

## 3. Missing Input Validation

### The Anti-Pattern

Processing user input without validating its type, length, format, or content. Trusting that input will be well-formed because "the UI restricts it."

### Why It Happens

- "The frontend validates it" — assuming client-side validation is sufficient
- Pressure to deliver features quickly — validation is seen as extra work
- Not understanding the attack surface of each input field

### Why It Is Harmful

- SQL injection — attacker manipulates database queries
- Cross-site scripting (XSS) — attacker injects scripts viewed by other users
- Command injection — attacker executes OS commands
- Path traversal — attacker accesses files outside intended directories
- Buffer overflows — attacker crashes or exploits the application

### The Fix

- Validate all input on the server side — type, length, range, format
- Use allowlists (define what IS valid) not denylists
- Use parameterized queries for all database access
- Use context-appropriate output encoding for all rendered content
- Apply framework-level input validation (FluentValidation, Zod, Joi, JSON Schema)
- Reject requests with unexpected fields

---

## 4. Excessive Permissions

### The Anti-Pattern

Granting broad permissions (admin, root, wildcard IAM policies) to users, services, or CI/CD pipelines because it is easier than figuring out the minimum required permissions.

### Why It Happens

- "It works with admin access; we'll tighten it later" — later never comes
- Complex IAM systems make least privilege difficult
- Lack of tooling to identify minimum required permissions
- Time pressure to get things working

### Why It Is Harmful

- A compromised service with admin access can affect the entire system
- A compromised CI/CD token with broad permissions can deploy malicious code to production
- Insider threats have maximum impact with excessive permissions
- Compliance frameworks (SOC 2, PCI DSS) require demonstrable least privilege

### The Fix

- Start with zero permissions and add only what is needed
- Use cloud IAM access analyzers to identify unused permissions (AWS IAM Access Analyzer, Azure Access Reviews)
- Scope CI/CD tokens to specific repositories, environments, and actions
- Use service-specific database accounts with appropriate read/write restrictions
- Implement just-in-time (JIT) access for elevated permissions
- Review and audit permissions quarterly

---

## 5. Security Through Obscurity

### The Anti-Pattern

Relying on secret URLs, hidden endpoints, obfuscated code, or non-standard ports as the primary security control instead of proper authentication and authorization.

### Why It Happens

- "Nobody will find this endpoint" — underestimating attackers
- Mistaking obscurity for a security control
- Quick fix to "secure" something without implementing proper auth

### Why It Is Harmful

- Attackers use automated scanners, brute-force, and reconnaissance to discover hidden resources
- Obfuscated code can be deobfuscated; non-standard ports can be port-scanned
- When the obscurity is discovered, there is no real security control behind it
- Creates a false sense of security that delays implementation of real controls

### The Fix

- Implement proper authentication and authorization on every endpoint
- Use obscurity as an additional layer (defense in depth), never the primary control
- Assume attackers know your architecture — design security accordingly
- Use network segmentation and access controls, not hidden URLs

---

## 6. Missing Authentication on Internal APIs

### The Anti-Pattern

Assuming that internal APIs (between microservices, between backend and frontend) do not need authentication because they are "inside the network."

### Why It Happens

- "It's behind the firewall" — trusting the network perimeter
- Perceived overhead of implementing service-to-service authentication
- Microservices that were originally internal but became exposed through misconfiguration

### Why It Is Harmful

- Network perimeters are not security boundaries — networks can be compromised
- Lateral movement is the primary technique attackers use after initial compromise
- Cloud environments have more fluid network boundaries than traditional data centers
- Misconfigured load balancers or ingress controllers can accidentally expose internal APIs

### The Fix

- Adopt zero trust — authenticate every request regardless of network location
- Implement mTLS between services (use a service mesh for automation)
- Use JWTs or service tokens for service-to-service authentication
- Enforce authorization at each service, not just at the edge

---

## 7. Logging Sensitive Data

### The Anti-Pattern

Including passwords, tokens, API keys, credit card numbers, Social Security numbers, or other sensitive data in application logs.

### Why It Happens

- Debugging with verbose logging left enabled in production
- Logging full request/response bodies without redacting sensitive fields
- Not having a clear policy about what should and should not be logged

### Why It Is Harmful

- Logs are often stored with less protection than databases (wider access, longer retention)
- Log aggregation systems (ELK, Splunk, CloudWatch) become unauthorized data stores
- Compliance violations (PCI DSS requires that full card numbers are never logged)
- Attackers who gain access to log systems gain access to all logged secrets

### The Fix

- Define and enforce a "never log" list (passwords, tokens, PII, card numbers)
- Implement structured logging with automatic field redaction
- Log identifiers (user ID, request ID) not values (password, token)
- Review log output regularly to catch accidental sensitive data inclusion
- Use log sanitization libraries or middleware

---

## 8. Using Outdated Dependencies

### The Anti-Pattern

Running applications with known vulnerable dependencies because updating is seen as risky or time-consuming.

### Why It Happens

- "If it works, don't touch it" — fear of breaking changes
- No automated dependency update process
- Dependency updates are deprioritized in favor of feature work
- Transitive dependencies are invisible and forgotten

### Why It Is Harmful

- Known vulnerabilities have public exploit code — attackers use them
- Log4Shell (CVE-2021-44228) demonstrated how a single dependency vulnerability can affect millions of applications
- Outdated dependencies accumulate technical debt, making eventual updates more difficult
- Compliance frameworks require timely patching of known vulnerabilities

### The Fix

- Enable automated dependency scanning (Dependabot, Snyk, Trivy) in CI/CD
- Configure automated PR creation for dependency updates (Dependabot, Renovate)
- Define SLAs for patching: critical within 72 hours, high within 2 weeks
- Commit lock files and use frozen installs in CI to ensure reproducibility
- Remove unused dependencies to reduce the attack surface

---

## 9. Insufficient Error Handling

### The Anti-Pattern

Returning detailed error messages (stack traces, SQL queries, internal file paths, server versions) to end users, or failing to handle errors securely (failing open).

### Why It Happens

- Debug mode enabled in production
- Default framework error pages not replaced with custom handlers
- Developers not considering error messages as an attack surface

### Why It Is Harmful

- Stack traces reveal framework versions, file paths, and internal architecture
- SQL error messages reveal table names, column names, and query structure
- Detailed errors help attackers map the application and craft targeted attacks
- Failing open (allowing access when an error occurs) bypasses security controls

### The Fix

- Return generic error messages to users ("An error occurred")
- Log detailed errors server-side for debugging
- Configure custom error handlers that return structured, safe responses
- Implement fail-closed behavior — if a security check fails, deny the request
- Test error handling paths in your security tests

---

## 10. No Incident Response Plan

### The Anti-Pattern

Having no documented, tested plan for responding to security incidents. Assuming the team will "figure it out" when something happens.

### Why It Happens

- "We haven't had an incident yet" — survivorship bias
- Incident response planning is seen as overhead rather than essential preparation
- No one is assigned ownership of incident response

### Why It Is Harmful

- Without a plan, response is slower, more chaotic, and makes more mistakes
- Critical actions (evidence preservation, credential rotation, notification) are missed or delayed
- Regulatory notification deadlines (GDPR 72-hour window) are missed
- Post-incident blame culture develops without a blameless review process

### The Fix

- Document an incident response plan with roles, procedures, and communication templates
- Assign an incident response team and train them
- Conduct tabletop exercises quarterly
- Test recovery procedures (can you actually restore from backups?)
- Review and update the plan after every incident

---

## 11. Rolling Your Own Cryptography

### The Anti-Pattern

Implementing custom encryption algorithms, creating homegrown authentication protocols, or inventing novel security mechanisms instead of using well-tested, peer-reviewed solutions.

### Why It Happens

- "Standard solutions don't fit our use case" — usually not true
- Misunderstanding of what constitutes cryptography (custom "encoding" treated as encryption)
- Not knowing that high-quality cryptographic libraries exist

### Why It Is Harmful

- Custom cryptography is almost certainly flawed — it takes decades of peer review to validate algorithms
- Subtle implementation errors (timing attacks, padding oracle attacks) are extremely difficult to detect
- No one else can audit or maintain custom cryptographic code
- Attackers target custom crypto because it is the weakest link

### The Fix

- Use well-tested libraries (libsodium, OpenSSL, platform-native APIs)
- Use standard algorithms (AES-256-GCM, RSA-2048+, ECDSA P-256, SHA-256)
- Use standard protocols (TLS 1.3, OAuth 2.0 + PKCE, OIDC)
- If you think you need custom crypto, you almost certainly do not — consult a security expert

---

## 12. Trusting Client-Side Validation

### The Anti-Pattern

Relying on client-side JavaScript validation, hidden form fields, or client-side state to enforce security controls. Assuming that because the UI restricts an action, the server does not need to check.

### Why It Happens

- Developers see the UI as the interface — if the button is hidden, the action is blocked
- Client-side validation gives immediate user feedback, creating a false sense of completeness
- Server-side validation is seen as redundant when the client validates

### Why It Is Harmful

- Attackers bypass the client entirely — they interact directly with APIs using curl, Postman, or custom scripts
- Hidden form fields can be modified before submission
- Client-side state (JWT decoded in the browser) can be manipulated
- Every major web application attack (SQLi, XSS, CSRF) works by bypassing client-side controls

### The Fix

- Validate all input on the server — always, no exceptions
- Authorize all actions on the server — check permissions before processing
- Treat client-side validation as a UX feature, not a security control
- Test your APIs with direct requests (bypassing the UI) to verify server-side enforcement
- Never store security-critical state on the client

---

## Quick Reference Checklist

Use this checklist for pre-production security reviews:

### Critical (Must Fix Before Production)

- [ ] No secrets in source code, configuration files, or container images
- [ ] Passwords hashed with Argon2id or bcrypt (not MD5, SHA-256, or plaintext)
- [ ] All user input validated server-side (type, length, format)
- [ ] Parameterized queries for all database access (no SQL concatenation)
- [ ] Output encoding for all user-supplied data rendered in HTML/JS/CSS
- [ ] All cryptography uses well-tested libraries and standard algorithms
- [ ] Authentication and authorization enforced on the server, not the client

### High (Fix Before Production or Immediately After)

- [ ] Least privilege applied to all IAM roles, service accounts, and database users
- [ ] All APIs (including internal) require authentication
- [ ] No sensitive data in logs (passwords, tokens, PII, card numbers)
- [ ] Dependencies scanned for known vulnerabilities; critical CVEs patched

### Medium (Address Within First Sprint)

- [ ] Error messages reveal no internal details to users
- [ ] Incident response plan documented and team trained
- [ ] Security logging enabled for auth, authz, and data access events
- [ ] Dependencies pinned with lock files; automated updates configured

---

## Next Steps

1. **Score yourself** — Use the [Quick Reference Checklist](#quick-reference-checklist) and score your current systems
2. **Fix critical severity first** — Address 🔴 Critical anti-patterns (#1–#3, #11, #12) before others
3. **Read the companion guide** — [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) describes the correct patterns
4. **Review authentication** — [01-AUTHENTICATION.md](01-AUTHENTICATION.md) covers secure auth implementation
5. **Review secure coding** — [06-SECURE-CODING.md](06-SECURE-CODING.md) covers input validation, injection prevention
6. **Follow the learning path** — [LEARNING-PATH.md](LEARNING-PATH.md) provides a structured curriculum

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Security Anti-Patterns documentation |
