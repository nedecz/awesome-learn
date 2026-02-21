# Security Best Practices

A comprehensive security checklist and best practices guide — covering shift-left security, secure development lifecycle, production security, cloud security, API security, and continuous security automation for modern software organizations.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Shift-Left Security](#shift-left-security)
   - [Security in Design](#security-in-design)
   - [Security in Development](#security-in-development)
   - [Security in CI/CD](#security-in-cicd)
3. [Authentication Best Practices](#authentication-best-practices)
4. [Authorization Best Practices](#authorization-best-practices)
5. [Data Protection Best Practices](#data-protection-best-practices)
6. [API Security Best Practices](#api-security-best-practices)
7. [Cloud Security Best Practices](#cloud-security-best-practices)
8. [Container Security Best Practices](#container-security-best-practices)
9. [Secrets Management Best Practices](#secrets-management-best-practices)
10. [Logging and Monitoring Best Practices](#logging-and-monitoring-best-practices)
11. [Dependency Management Best Practices](#dependency-management-best-practices)
12. [Security Automation](#security-automation)
    - [Automated Security Scanning](#automated-security-scanning)
    - [Security Policy Enforcement](#security-policy-enforcement)
    - [Automated Remediation](#automated-remediation)
13. [Production Security Checklist](#production-security-checklist)
14. [Next Steps](#next-steps)
15. [Version History](#version-history)

---

## Overview

This document consolidates the most critical security best practices into an actionable guide. It is designed to be used as a reference during design reviews, code reviews, deployment planning, and periodic security assessments. Each practice is stated as a clear directive with rationale and context.

### Target Audience

- **Developers** reviewing their code and architecture against security best practices
- **DevOps Engineers** hardening infrastructure and CI/CD pipelines
- **Security Engineers** establishing baseline security standards for the organization
- **Tech Leads** conducting security-focused design and code reviews
- **Engineering Managers** ensuring their teams follow security best practices

### Scope

- Shift-left security integration throughout the SDLC
- Best practices for authentication, authorization, data protection, API security
- Cloud, container, and secrets management best practices
- Logging, monitoring, and dependency management
- Security automation and continuous compliance
- Production security checklist

---

## Shift-Left Security

### Security in Design

- **Perform threat modeling** for every new system and significant feature change using STRIDE or similar frameworks
- **Write abuse cases** alongside user stories — ask "how could this be misused?"
- **Review architecture** for security concerns before starting implementation
- **Identify data classification** early — know what data is sensitive and apply appropriate controls from the start
- **Choose secure defaults** in architecture decisions — encryption on, authentication required, access denied by default

### Security in Development

- **Validate all input** on the server side — type, length, range, format
- **Use parameterized queries** for all database access — never concatenate user input into queries
- **Encode all output** using context-appropriate encoding (HTML, JavaScript, URL)
- **Use secure libraries** for authentication, encryption, and session management — never roll your own
- **Review code for security** during every pull request — check for injection, access control, and data exposure
- **Run SAST tools** in the IDE and pre-commit hooks for immediate feedback

### Security in CI/CD

- **Run SAST** (static application security testing) on every pull request
- **Run SCA** (software composition analysis) to detect vulnerable dependencies
- **Scan container images** for vulnerabilities before pushing to a registry
- **Scan IaC templates** (Terraform, CloudFormation) for misconfigurations
- **Detect secrets** in code with automated secret scanning (GitLeaks, TruffleHog)
- **Sign artifacts** and verify signatures before deployment
- **Generate SBOM** for every release

---

## Authentication Best Practices

| Practice | Details |
|----------|---------|
| **Hash passwords with Argon2id or bcrypt** | Never store plaintext or reversibly encrypted passwords |
| **Enforce MFA for all users** | Prefer hardware keys or passkeys; TOTP as fallback |
| **Use OAuth 2.0 + PKCE for all flows** | Authorization Code + PKCE for web, mobile, and SPA |
| **Set short access token lifetimes** | 5–15 minutes for access tokens; use refresh tokens for renewal |
| **Implement secure session management** | HttpOnly, Secure, SameSite cookies; regenerate session ID on login |
| **Enforce account lockout** | Lock after 5–10 failed attempts; implement rate limiting |
| **Check passwords against breach databases** | Use Have I Been Pwned API to reject compromised passwords |
| **Use secure password reset flows** | Time-limited, single-use tokens; hash tokens before storing |
| **Invalidate sessions on password change** | Force re-authentication across all devices |
| **Log all authentication events** | Success, failure, lockout, MFA challenge, password reset |

---

## Authorization Best Practices

| Practice | Details |
|----------|---------|
| **Deny by default** | Require explicit permission grants; new endpoints require auth |
| **Enforce authorization on every request** | Do not cache authorization decisions across requests |
| **Check resource-level access** | Verify the user can access this specific resource, not just resources of this type |
| **Use a consistent authorization framework** | Apply authorization middleware to all routes by default |
| **Implement least privilege** | Users, services, and CI/CD tokens get only the permissions they need |
| **Review access quarterly** | Audit who has access to what; remove stale permissions |
| **Use time-bound access (JIT)** | Grant elevated access temporarily and revoke automatically |
| **Separate duties for privilege management** | Admins cannot elevate their own privileges without approval |
| **Test authorization in automated tests** | Write tests that verify non-authorized users are rejected |
| **Log all authorization failures** | Alert on repeated failures; they may indicate an attack |

---

## Data Protection Best Practices

| Practice | Details |
|----------|---------|
| **Encrypt all data in transit** | TLS 1.2+ for all connections; prefer TLS 1.3 |
| **Encrypt sensitive data at rest** | AES-256-GCM for application-level; enable volume/disk encryption |
| **Use envelope encryption** | Encrypt data with DEK; encrypt DEK with KMS-managed KEK |
| **Classify data by sensitivity** | Public, Internal, Confidential, Restricted — apply controls accordingly |
| **Mask PII in non-production environments** | Never use production data in dev/staging without masking |
| **Implement data retention policies** | Automatically delete data after the retention period |
| **Never log sensitive data** | Exclude passwords, tokens, PII, card numbers from logs |
| **Use secure deletion** | Cryptographic erasure for encrypted data; secure wipe for unencrypted |
| **Protect backups** | Encrypt backups; test restore procedures; control access to backup storage |
| **Implement data loss prevention (DLP)** | Monitor and block unauthorized data transfers |

---

## API Security Best Practices

| Practice | Details |
|----------|---------|
| **Authenticate all API endpoints** | Require authentication by default; explicitly mark public endpoints |
| **Validate all input** | Use schema validation (JSON Schema, Zod, Joi); reject unknown fields |
| **Rate limit all endpoints** | Protect against abuse; stricter limits on auth endpoints |
| **Use HTTPS only** | Redirect HTTP to HTTPS; set HSTS header |
| **Return appropriate status codes** | 401 for unauthenticated, 403 for unauthorized, 400 for bad input |
| **Do not expose internal details** | No stack traces, SQL errors, or server versions in API responses |
| **Implement request size limits** | Reject oversized payloads at the gateway level |
| **Version your APIs** | Enable security fixes without breaking backward compatibility |
| **Use OAuth 2.0 scopes** | Scope access tokens to the minimum required permissions |
| **Log all API access** | Include method, path, status, user, source IP |

---

## Cloud Security Best Practices

| Practice | Details |
|----------|---------|
| **Use IAM roles, not long-lived keys** | Prefer workload identity; rotate any remaining keys every 90 days |
| **Enable MFA for all console access** | Require MFA for all human accounts; enforce via IAM policies |
| **Enable cloud audit logging** | CloudTrail, Azure Activity Logs, GCP Audit Logs — all regions, all accounts |
| **Encrypt all storage** | Enable default encryption for S3, EBS, RDS, Azure Blob, GCS |
| **Block public access by default** | S3 Block Public Access, Azure storage firewalls, GCS uniform bucket-level |
| **Use private networking** | VPCs, private subnets, VPC endpoints, Private Link |
| **Scan IaC before deployment** | Checkov, tfsec, or KICS on every Terraform/CloudFormation change |
| **Enable security services** | GuardDuty, Defender for Cloud, Security Command Center |
| **Tag all resources** | Enforce tagging for cost, ownership, and compliance |
| **Implement network segmentation** | Separate workloads by VPC or subnet; restrict traffic between segments |

---

## Container Security Best Practices

| Practice | Details |
|----------|---------|
| **Use minimal base images** | Distroless or scratch images for production |
| **Run as non-root** | Set `USER` in Dockerfile; enforce via PodSecurityPolicy or PSA |
| **Scan images in CI/CD** | Trivy, Grype, or Snyk before pushing to registry |
| **Sign container images** | Use Cosign/Sigstore; verify signatures before deployment |
| **Use read-only file systems** | Set `readOnlyRootFilesystem: true` in Kubernetes security context |
| **Drop all capabilities** | Drop all Linux capabilities; add only what is needed |
| **Pin image digests** | Use `image@sha256:...` in deployment manifests, not `:latest` |
| **Enable network policies** | Default-deny between pods; allow only required traffic |
| **Limit resource usage** | Set CPU and memory limits to prevent resource exhaustion |
| **Use Pod Security Standards** | Enforce restricted profile in Kubernetes namespaces |

---

## Secrets Management Best Practices

| Practice | Details |
|----------|---------|
| **Never store secrets in code** | No hard-coded credentials, API keys, or tokens in source files |
| **Use a secrets manager** | HashiCorp Vault, AWS Secrets Manager, Azure Key Vault |
| **Rotate secrets regularly** | 90 days or less; immediately on compromise |
| **Use dynamic secrets** | Vault dynamic secrets for database credentials — short-lived, auto-expiring |
| **Audit secret access** | Log every secret read; alert on unusual access patterns |
| **Mount secrets as files, not env vars** | Environment variables appear in process inspection; files are more isolated |
| **Use workload identity** | Eliminate static credentials by using cloud-native identity federation |
| **Scan for leaked secrets** | Run GitLeaks/TruffleHog in CI/CD and pre-commit hooks |
| **Encrypt secrets at rest** | Use KMS-backed encryption for all secrets in the secrets manager |
| **Separate secrets by environment** | Production secrets should be inaccessible from non-production systems |

---

## Logging and Monitoring Best Practices

| Practice | Details |
|----------|---------|
| **Log all security events** | Authentication, authorization, data access, config changes |
| **Centralize logs** | Send all logs to a centralized, append-only log system |
| **Protect log integrity** | Use append-only storage; separate write and read access |
| **Set up security alerts** | Alert on repeated auth failures, privilege changes, config modifications |
| **Monitor for anomalies** | Detect unusual access patterns, traffic spikes, data exports |
| **Retain logs per compliance** | 1 year minimum (SOC 2, PCI DSS); 6 years (HIPAA) |
| **Never log sensitive data** | Exclude passwords, tokens, PII, card numbers |
| **Include correlation IDs** | Trace requests across services for incident investigation |
| **Test your alerting** | Regularly trigger alerts to verify they work end-to-end |
| **Integrate with incident response** | Security alerts should automatically create incident tickets |

---

## Dependency Management Best Practices

| Practice | Details |
|----------|---------|
| **Scan dependencies in CI/CD** | Dependabot, Snyk, or Trivy on every pull request |
| **Pin dependency versions** | Commit lock files; use `--frozen-lockfile` in CI |
| **Review dependency updates** | Do not auto-merge dependency PRs without CI validation |
| **Remove unused dependencies** | Reduce attack surface; audit dependencies quarterly |
| **Generate SBOM for releases** | Use Syft or cdxgen; attach to release artifacts |
| **Monitor for new CVEs** | Subscribe to advisories for critical dependencies |
| **Prefer well-maintained libraries** | Check maintenance activity, security history, community size |
| **Pin GitHub Actions to SHAs** | Prevent supply chain attacks via mutable tags |
| **Audit transitive dependencies** | You are responsible for your entire dependency tree |
| **Set up automated updates** | Dependabot or Renovate with appropriate review and CI checks |

---

## Security Automation

### Automated Security Scanning

Integrate security scanning at every stage of the development lifecycle:

```
Security Scanning Pipeline
──────────────────────────
IDE         → SAST (Semgrep, SonarLint), secret detection
Pre-commit  → Secret scanning (GitLeaks), linting
Pull Request → SAST + SCA + license check + IaC scan
Build       → Container image scan + SBOM generation
Pre-deploy  → Artifact signature verification + policy gate
Production  → DAST + CSPM + runtime monitoring
```

### Security Policy Enforcement

Use policy-as-code to automatically enforce security requirements:

- **Kubernetes admission control** — Kyverno or OPA Gatekeeper to enforce pod security, image signing
- **IaC policy** — Checkov or Sentinel to enforce encryption, access control in Terraform
- **CI/CD policy** — Required security scans as pipeline gates (block merge on critical findings)
- **Cloud policy** — AWS SCPs, Azure Policy, GCP Organization Policies for account-level guardrails

### Automated Remediation

Where possible, automate the remediation of security findings:

- Auto-create PRs for dependency updates (Dependabot, Renovate)
- Auto-fix common SAST findings (code formatting, import ordering)
- Auto-rotate secrets on a schedule (Vault, AWS Secrets Manager rotation)
- Auto-block deployments that fail security gates

---

## Production Security Checklist

Use this checklist before deploying to production:

```
Production Security Checklist
─────────────────────────────

Authentication & Authorization
  [ ] All endpoints require authentication (except explicitly public ones)
  [ ] MFA is enabled for all users
  [ ] Authorization checks on every request (server-side)
  [ ] Least privilege applied to all service accounts and IAM roles
  [ ] Session management uses secure cookies (HttpOnly, Secure, SameSite)

Data Protection
  [ ] All data in transit encrypted with TLS 1.2+ (prefer 1.3)
  [ ] Sensitive data at rest encrypted with AES-256 via KMS
  [ ] PII is masked in logs and non-production environments
  [ ] Data retention policies are implemented and automated
  [ ] Backups are encrypted, tested, and access-controlled

Application Security
  [ ] All user input is validated server-side
  [ ] Parameterized queries for all database access
  [ ] Output encoding for all user-supplied data
  [ ] Security headers configured (HSTS, CSP, X-Content-Type-Options)
  [ ] Error messages reveal no internal details
  [ ] Rate limiting on all endpoints

Infrastructure
  [ ] No secrets in source code or environment variables (use secrets manager)
  [ ] Network segmentation between tiers
  [ ] Default-deny firewall rules
  [ ] Container images scanned and signed
  [ ] Containers run as non-root with minimal capabilities

CI/CD & Supply Chain
  [ ] SAST, SCA, and secret scanning in CI/CD pipeline
  [ ] Dependencies pinned with lock files
  [ ] Container images tagged by digest (not :latest)
  [ ] GitHub Actions pinned to commit SHAs
  [ ] SBOM generated for every release

Monitoring & Incident Response
  [ ] Security events logged to centralized, append-only store
  [ ] Alerts configured for authentication failures and authorization violations
  [ ] Incident response plan documented and tested
  [ ] On-call rotation for security alerts
  [ ] Post-incident review process in place
```

---

## Next Steps

Continue your security learning journey:

| File | Topic | Description |
|---|---|---|
| [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md) | Anti-Patterns | Common security mistakes and how to avoid them |
| [08-INCIDENT-RESPONSE.md](08-INCIDENT-RESPONSE.md) | Incident Response | Security incident handling, forensics, post-mortems |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Learning Path | Structured learning guide with exercises |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Security Best Practices documentation |
