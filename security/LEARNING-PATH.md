# Security Learning Path

A structured, self-paced training guide to mastering application security — from security fundamentals and secure coding through authentication, authorization, cryptography, supply chain security, infrastructure hardening, compliance, and incident response. Each phase builds on the previous one, progressing from core concepts to advanced production patterns.

> **Time Estimate:** 10–12 weeks at ~5 hours/week. Adjust pace to your experience level. Engineers with prior security experience may complete Phases 1–2 in half the time.

---

## How to Use This Guide

1. **Follow the phases in order** — each one builds directly on prior knowledge; jumping ahead creates gaps that slow you down later
2. **Read the linked documents** — they contain the detailed content, code examples, and reference tables
3. **Complete the exercises** — hands-on practice is how security concepts become intuition; do not skip them
4. **Check yourself** — use the knowledge check items before moving to the next phase; if you cannot answer them confidently, re-read the relevant sections
5. **Build the capstone project** — the final project ties together all phases and produces a portfolio artifact you can reference in security conversations

---

## Phase 1: Security Foundations (Week 1–2)

### Learning Objectives

- Understand the CIA triad and core security principles
- Learn threat modeling methodologies (STRIDE, DREAD)
- Know the OWASP Top 10 web application security risks
- Understand attack surfaces and how to minimize them
- Grasp the concept of defense in depth and zero trust

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | CIA triad, security principles, threat modeling, OWASP Top 10, attack surfaces, security layers |
| 2 | [06-SECURE-CODING](06-SECURE-CODING.md) | Input validation, output encoding, injection prevention, CSRF, secure defaults |

### Exercises

**1. Threat Model a Simple Application:**

Choose a web application you are familiar with (personal project, open-source app, or a work service with permission) and create a threat model:

- Draw a data flow diagram showing all processes, data stores, external entities, and trust boundaries
- Apply STRIDE to each element in the diagram
- Identify the top 5 threats by risk (likelihood × impact)
- Propose a mitigation for each threat
- Document which OWASP Top 10 categories your threats map to

**2. Secure Code Review Exercise:**

Review a code sample (your own code or an open-source project) for security issues:

- Check for SQL injection, XSS, and command injection vulnerabilities
- Verify input validation is present on all user-facing endpoints
- Check for hardcoded secrets or credentials
- Verify that error messages do not expose internal details
- Document each finding with the vulnerability type, location, and recommended fix

### Knowledge Check

- [ ] Can you explain the CIA triad and give an example of a control for each property?
- [ ] Can you apply STRIDE to a data flow diagram?
- [ ] Can you identify the difference between stored XSS and reflected XSS?
- [ ] Can you explain why parameterized queries prevent SQL injection?
- [ ] Can you list three examples of defense in depth?

---

## Phase 2: Authentication & Authorization (Week 3–4)

### Learning Objectives

- Understand password hashing algorithms and when to use each
- Implement multi-factor authentication (MFA)
- Design OAuth 2.0 and OpenID Connect flows
- Implement RBAC and ABAC access control models
- Understand session management and JWT security

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [01-AUTHENTICATION](01-AUTHENTICATION.md) | Password hashing, MFA, OAuth 2.0, OIDC, session management, JWT |
| 2 | [02-AUTHORIZATION](02-AUTHORIZATION.md) | RBAC, ABAC, ReBAC, policy engines, least privilege, IDOR prevention |

### Exercises

**1. Authentication Design Exercise:**

Design the authentication system for a multi-tenant SaaS application:

- Choose a password hashing algorithm and justify your choice
- Design the OAuth 2.0 + PKCE flow for the web application
- Design the session management strategy (token lifetime, refresh, revocation)
- Plan MFA implementation (which methods, enrollment flow, recovery)
- Document the security properties of each decision

**2. Authorization Model Exercise:**

Design the authorization model for a document management system with the following requirements:

- Users can create documents and share them with specific users or teams
- Team admins can manage team membership
- Organization admins can manage all teams and documents
- External users can be granted read-only access to specific documents

Choose between RBAC, ABAC, and ReBAC, justify your choice, and define:
- The roles/attributes/relationships
- The permission model
- How resource-level access is enforced
- How access is revoked

### Knowledge Check

- [ ] Can you explain why bcrypt is suitable for passwords but SHA-256 is not?
- [ ] Can you diagram the OAuth 2.0 Authorization Code + PKCE flow?
- [ ] Can you explain the difference between RBAC and ABAC and when to use each?
- [ ] Can you describe three JWT security risks and their mitigations?
- [ ] Can you explain how to prevent IDOR vulnerabilities?

---

## Phase 3: Cryptography & Data Protection (Week 5–6)

### Learning Objectives

- Understand symmetric and asymmetric encryption
- Implement encryption at rest and in transit
- Manage cryptographic keys securely
- Configure TLS correctly
- Use digital signatures and HMAC

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [03-CRYPTOGRAPHY](03-CRYPTOGRAPHY.md) | AES-GCM, TLS 1.3, key management, PKI, envelope encryption, common mistakes |

### Exercises

**1. Encryption Architecture Design:**

Design the data protection strategy for an application that stores sensitive customer data (PII, financial records):

- Choose encryption algorithms for data at rest and in transit
- Design the key management architecture (where are keys stored? how are they rotated?)
- Implement envelope encryption for application-level data protection
- Plan TLS configuration (cipher suites, certificate management, mTLS for internal services)
- Document the trust boundaries and what is encrypted at each boundary

**2. TLS Configuration Audit:**

Audit the TLS configuration of a web application (your own or a public website using tools like SSL Labs):

- Check the TLS version (1.2 minimum, 1.3 preferred)
- Review the cipher suites (are deprecated algorithms present?)
- Verify certificate validity and chain of trust
- Check for HSTS header
- Document findings and recommendations

### Knowledge Check

- [ ] Can you explain the difference between AES-GCM and AES-CBC?
- [ ] Can you describe envelope encryption and why it is used?
- [ ] Can you explain why nonce reuse in GCM is catastrophic?
- [ ] Can you describe the TLS 1.3 handshake?
- [ ] Can you list three common cryptographic mistakes and their fixes?

---

## Phase 4: Supply Chain & Infrastructure Security (Week 7–8)

### Learning Objectives

- Secure the software supply chain (dependencies, builds, artifacts)
- Implement SBOM generation and artifact signing
- Harden infrastructure and network security
- Manage secrets securely
- Apply zero trust architecture principles

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [04-SUPPLY-CHAIN-SECURITY](04-SUPPLY-CHAIN-SECURITY.md) | Dependency scanning, SBOM, SLSA, Cosign, container image security |
| 2 | [05-INFRASTRUCTURE-SECURITY](05-INFRASTRUCTURE-SECURITY.md) | Network policies, secrets management, zero trust, cloud security, IAM |

### Exercises

**1. Supply Chain Security Assessment:**

Assess the supply chain security of a project you work on:

- Run a dependency scan (npm audit, pip-audit, Trivy) and document the findings
- Check if lock files are committed and used in CI
- Verify that CI/CD actions/plugins are pinned to commit SHAs
- Generate an SBOM for the project
- Identify three improvements to the project's supply chain security posture

**2. Infrastructure Security Review:**

Review the infrastructure configuration of a project or environment:

- Check network segmentation (are databases accessible from the internet?)
- Audit IAM permissions (are there overly broad policies?)
- Review secrets management (how are secrets stored and rotated?)
- Check for security monitoring (are security events logged and alerted on?)
- Create an action plan to address the most critical findings

### Knowledge Check

- [ ] Can you explain the SLSA framework and its levels?
- [ ] Can you describe how Cosign keyless signing works?
- [ ] Can you explain the difference between static and dynamic secrets?
- [ ] Can you describe zero trust principles and give three implementation examples?
- [ ] Can you explain why Kubernetes Secrets are not encrypted by default and how to fix it?

---

## Phase 5: Compliance & Incident Response (Week 9–10)

### Learning Objectives

- Understand major compliance frameworks (SOC 2, PCI DSS, GDPR)
- Implement audit logging for compliance
- Build and test an incident response plan
- Conduct forensic investigation
- Perform post-incident reviews

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [07-COMPLIANCE](07-COMPLIANCE.md) | SOC 2, PCI DSS, GDPR, HIPAA, audit logging, evidence collection |
| 2 | [08-INCIDENT-RESPONSE](08-INCIDENT-RESPONSE.md) | IR lifecycle, preparation, detection, containment, recovery, forensics |

### Exercises

**1. Audit Logging Design:**

Design the audit logging system for an application that handles sensitive data:

- Define the events to log (authentication, authorization, data access, configuration changes)
- Design the log format (structured JSON with timestamp, actor, action, resource, result)
- Plan the log architecture (application → aggregator → central store → SIEM)
- Define retention policies based on applicable regulations
- Document how log integrity is protected (append-only, access controls)

**2. Incident Response Tabletop Exercise:**

Conduct a tabletop exercise for the following scenario:

> A security researcher contacts you saying they found a public S3 bucket containing customer data. They provide a sample of the data as proof.

Walk through:
- Initial triage: Is this real? What is the scope?
- Containment: What are the immediate actions?
- Investigation: What logs do you check? What questions do you answer?
- Communication: Who do you notify (internal, customers, regulators)?
- Recovery: How do you remediate?
- Post-incident: What are the lessons learned and action items?

### Knowledge Check

- [ ] Can you explain the difference between SOC 2 Type I and Type II?
- [ ] Can you list the GDPR data subject rights and their engineering impact?
- [ ] Can you describe the NIST incident response lifecycle?
- [ ] Can you explain the difference between MTTD and MTTR?
- [ ] Can you describe how to preserve evidence during an incident?

---

## Phase 6: Best Practices & Production Readiness (Week 11–12)

### Learning Objectives

- Apply the production security checklist
- Identify and remediate security anti-patterns
- Automate security scanning in CI/CD
- Build a continuous security improvement process

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [09-BEST-PRACTICES](09-BEST-PRACTICES.md) | Shift-left security, production checklist, security automation |
| 2 | [10-ANTI-PATTERNS](10-ANTI-PATTERNS.md) | Common security mistakes, code review checklist |

### Exercises

**1. Security Audit of a Production System:**

Perform a comprehensive security review of a system using the production security checklist from [09-BEST-PRACTICES](09-BEST-PRACTICES.md):

- Walk through every item in the checklist
- Score the system: compliant, partially compliant, or non-compliant for each item
- Prioritize findings by severity (critical, high, medium)
- Create an action plan with specific tasks, owners, and deadlines
- Present your findings and recommendations to the team

**2. Capstone Project — Secure Application Design:**

Design a complete security architecture for a new application. Choose one of:

- A healthcare patient portal (HIPAA compliance)
- A payment processing service (PCI DSS compliance)
- A multi-tenant SaaS platform (SOC 2 compliance)

Your design document should include:

- Threat model with data flow diagram and STRIDE analysis
- Authentication design (MFA, OAuth 2.0, session management)
- Authorization model (RBAC/ABAC, resource-level access)
- Data protection (encryption at rest and in transit, key management)
- Supply chain security (dependency scanning, SBOM, image signing)
- Infrastructure security (network segmentation, secrets management, zero trust)
- Compliance controls (audit logging, evidence collection, data subject rights)
- Incident response plan (detection, containment, recovery, communication)
- Security automation (CI/CD scanning, policy enforcement)
- Anti-pattern review (verify none of the 12 anti-patterns are present)

### Knowledge Check

- [ ] Can you walk through the full production security checklist from memory?
- [ ] Can you identify the 12 security anti-patterns and their fixes?
- [ ] Can you design a security scanning pipeline for CI/CD?
- [ ] Can you explain how shift-left security works in practice?
- [ ] Can you present a complete security architecture for a production application?

---

## Completion Criteria

You have completed this learning path when you can:

1. **Perform threat modeling** for any new system or feature
2. **Implement secure authentication** with MFA, OAuth 2.0, and proper session management
3. **Design authorization models** that enforce least privilege at the resource level
4. **Apply encryption** correctly for data at rest and in transit
5. **Secure the supply chain** with dependency scanning, SBOM, and artifact signing
6. **Harden infrastructure** with network segmentation, secrets management, and zero trust
7. **Meet compliance requirements** with audit logging, evidence collection, and data subject rights
8. **Respond to incidents** with a tested plan and effective communication
9. **Automate security** in CI/CD with scanning, policy enforcement, and continuous monitoring
10. **Avoid common anti-patterns** and advocate for security best practices on your team
