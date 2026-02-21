# Security Learning Resources

A comprehensive guide to application security, DevSecOps, and security engineering — from foundational concepts and threat modeling to authentication, authorization, cryptography, supply chain security, secure coding, compliance, and incident response.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | Security fundamentals, threat modeling, OWASP Top 10 | **Start here** |
| [01-AUTHENTICATION](01-AUTHENTICATION.md) | Password hashing, MFA, SSO, session management | When implementing identity and login flows |
| [02-AUTHORIZATION](02-AUTHORIZATION.md) | RBAC, ABAC, policy engines, least privilege | When designing access control systems |
| [03-CRYPTOGRAPHY](03-CRYPTOGRAPHY.md) | Encryption at rest/transit, TLS, key management | When protecting data and communications |
| [04-SUPPLY-CHAIN-SECURITY](04-SUPPLY-CHAIN-SECURITY.md) | Dependency scanning, SBOM, signed artifacts | When securing your build and delivery pipeline |
| [05-INFRASTRUCTURE-SECURITY](05-INFRASTRUCTURE-SECURITY.md) | Network policies, secrets management, zero trust | When hardening infrastructure and environments |
| [06-SECURE-CODING](06-SECURE-CODING.md) | Input validation, injection prevention, secure defaults | **Essential — every developer** |
| [07-COMPLIANCE](07-COMPLIANCE.md) | SOC 2, PCI DSS, GDPR, audit logging | When meeting regulatory requirements |
| [08-INCIDENT-RESPONSE](08-INCIDENT-RESPONSE.md) | Security incident handling, forensics, post-mortems | **Essential — preparation before incidents** |
| [09-BEST-PRACTICES](09-BEST-PRACTICES.md) | Security checklist, shift-left security, automation | **Essential — production checklist** |
| [10-ANTI-PATTERNS](10-ANTI-PATTERNS.md) | Common security mistakes and how to avoid them | **Essential — what NOT to do** |
| [LEARNING-PATH](LEARNING-PATH.md) | Structured learning guide with exercises | **Start here** after the Overview |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Understand the CIA triad and core security principles
   - Learn how threat modeling identifies risks before they become vulnerabilities
   - Explore the OWASP Top 10 web application security risks

2. **Learn Secure Coding** ([06-SECURE-CODING](06-SECURE-CODING.md))
   - Validate and sanitize all input
   - Prevent injection attacks (SQL, XSS, command injection)
   - Apply secure defaults and defense in depth

3. **Understand Authentication** ([01-AUTHENTICATION](01-AUTHENTICATION.md))
   - Implement secure password hashing with bcrypt or Argon2
   - Add multi-factor authentication (MFA)
   - Manage sessions and tokens securely

4. **Follow the Learning Path** ([LEARNING-PATH](LEARNING-PATH.md))
   - Structured curriculum with hands-on exercises
   - Progressive skill building from basics to production

### For Experienced Engineers

1. **Review Best Practices** ([09-BEST-PRACTICES](09-BEST-PRACTICES.md))
   - Production-ready security checklists
   - Shift-left security integration
   - Automated security scanning and policy enforcement

2. **Avoid Anti-Patterns** ([10-ANTI-PATTERNS](10-ANTI-PATTERNS.md))
   - Common security mistakes in authentication, authorization, and cryptography
   - Infrastructure and deployment security pitfalls

3. **Secure Your Supply Chain** ([04-SUPPLY-CHAIN-SECURITY](04-SUPPLY-CHAIN-SECURITY.md))
   - Dependency scanning and vulnerability management
   - SBOM generation and artifact signing
   - SLSA framework and provenance

4. **Prepare for Incidents** ([08-INCIDENT-RESPONSE](08-INCIDENT-RESPONSE.md))
   - Incident response plans and runbooks
   - Forensics and evidence collection
   - Post-incident reviews and lessons learned

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      Security Domains                            │
│                                                                  │
│   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│   │  Authentication  │  │  Authorization   │  │ Cryptography │  │
│   │  (Who are you?)  │  │  (What can you   │  │ (Protect     │  │
│   │                  │  │   access?)       │  │  data)       │  │
│   │  - Passwords     │  │  - RBAC / ABAC   │  │  - TLS       │  │
│   │  - MFA / SSO     │  │  - Policy engines│  │  - AES / RSA │  │
│   │  - OAuth / OIDC  │  │  - Least privilege│ │  - Key mgmt  │  │
│   └──────────────────┘  └──────────────────┘  └──────────────┘  │
│                                                                  │
│   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│   │  Secure Coding   │  │  Supply Chain    │  │ Infrastructure│ │
│   │  (Build secure   │  │  (Trust your     │  │ (Harden your │  │
│   │   software)      │  │   dependencies)  │  │  platform)   │  │
│   │                  │  │                  │  │              │  │
│   │  - Input valid.  │  │  - Dep scanning  │  │  - Zero trust│  │
│   │  - Injection prev│  │  - SBOM / SLSA   │  │  - Secrets   │  │
│   │  - Secure defaults│ │  - Signing       │  │  - Network   │  │
│   └──────────────────┘  └──────────────────┘  └──────────────┘  │
│                                                                  │
│   ┌──────────────────┐  ┌──────────────────┐                    │
│   │  Compliance      │  │ Incident Response│                    │
│   │  (Meet standards)│  │ (When things go  │                    │
│   │                  │  │  wrong)          │                    │
│   │  - SOC 2 / PCI   │  │  - Detection     │                    │
│   │  - GDPR / HIPAA  │  │  - Containment   │                    │
│   │  - Audit logging │  │  - Recovery      │                    │
│   └──────────────────┘  └──────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
```

## 🔑 Key Concepts

```
Security Fundamentals
─────────────────────
CIA Triad            → Confidentiality, Integrity, Availability
Defense in Depth     → Multiple layers of security controls
Least Privilege      → Grant only the minimum access required
Zero Trust           → Never trust, always verify — regardless of network location
Threat Modeling      → Systematically identify and prioritize security risks

Authentication & Authorization
──────────────────────────────
Authentication  → Verify identity (who are you?)
Authorization   → Verify permissions (what can you do?)
OAuth 2.0       → Delegated authorization framework
OpenID Connect  → Identity layer on top of OAuth 2.0
MFA             → Multi-factor authentication (something you know + have + are)

Cryptography
────────────
Encryption at Rest   → Protect stored data (AES-256, disk encryption)
Encryption in Transit → Protect data in motion (TLS 1.3)
Hashing              → One-way transformation for passwords (bcrypt, Argon2)
Key Management       → Secure generation, storage, rotation of cryptographic keys

Supply Chain Security
─────────────────────
SBOM        → Software Bill of Materials — inventory of all components
SLSA        → Supply chain Levels for Software Artifacts — provenance framework
SCA         → Software Composition Analysis — scan dependencies for vulnerabilities
Signing     → Cryptographically sign artifacts to prove authenticity
```

## 📋 Topics Covered

- **Foundations** — CIA triad, defense in depth, threat modeling, OWASP Top 10, security principles
- **Authentication** — Password hashing, MFA, SSO, OAuth 2.0, OpenID Connect, session management
- **Authorization** — RBAC, ABAC, policy engines, least privilege, zero trust access control
- **Cryptography** — Encryption at rest and in transit, TLS, hashing, key management, PKI
- **Supply Chain Security** — Dependency scanning, SBOM, SLSA, artifact signing, provenance
- **Infrastructure Security** — Network policies, secrets management, zero trust architecture
- **Secure Coding** — Input validation, injection prevention, output encoding, secure defaults
- **Compliance** — SOC 2, PCI DSS, GDPR, HIPAA, audit logging, evidence collection
- **Incident Response** — Detection, containment, eradication, recovery, post-mortems
- **Best Practices** — Security checklists, shift-left security, automation, continuous improvement
- **Anti-Patterns** — Common security mistakes in code, infrastructure, and processes

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to security?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md) then follow [LEARNING-PATH.md](LEARNING-PATH.md)

**Already familiar with security?** → Jump to [04-SUPPLY-CHAIN-SECURITY.md](04-SUPPLY-CHAIN-SECURITY.md) or [08-INCIDENT-RESPONSE.md](08-INCIDENT-RESPONSE.md)

**Going to production?** → Review [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) and [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md)

**Want a structured path?** → Follow the [LEARNING-PATH.md](LEARNING-PATH.md) — progressive exercises from basics to production
