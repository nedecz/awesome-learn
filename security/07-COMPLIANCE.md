# Compliance

A comprehensive guide to security compliance — covering SOC 2, PCI DSS, GDPR, HIPAA, audit logging, evidence collection, compliance automation, and building a compliance program for modern software organizations.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Compliance Fundamentals](#compliance-fundamentals)
   - [Why Compliance Matters](#why-compliance-matters)
   - [Compliance vs Security](#compliance-vs-security)
   - [Common Frameworks](#common-frameworks)
3. [SOC 2](#soc-2)
   - [Trust Service Criteria](#trust-service-criteria)
   - [SOC 2 Type I vs Type II](#soc-2-type-i-vs-type-ii)
   - [SOC 2 Controls](#soc-2-controls)
   - [Preparing for a SOC 2 Audit](#preparing-for-a-soc-2-audit)
4. [PCI DSS](#pci-dss)
   - [PCI DSS Requirements](#pci-dss-requirements)
   - [Cardholder Data Environment](#cardholder-data-environment)
   - [PCI DSS Compliance Levels](#pci-dss-compliance-levels)
   - [Reducing PCI Scope](#reducing-pci-scope)
5. [GDPR](#gdpr)
   - [GDPR Principles](#gdpr-principles)
   - [Data Subject Rights](#data-subject-rights)
   - [GDPR for Engineers](#gdpr-for-engineers)
   - [Data Protection Impact Assessment](#data-protection-impact-assessment)
6. [HIPAA](#hipaa)
   - [HIPAA Security Rule](#hipaa-security-rule)
   - [Protected Health Information](#protected-health-information)
7. [Audit Logging](#audit-logging)
   - [What to Log](#what-to-log)
   - [Log Architecture](#log-architecture)
   - [Log Retention](#log-retention)
   - [Log Integrity](#log-integrity)
8. [Evidence Collection](#evidence-collection)
   - [Automated Evidence Collection](#automated-evidence-collection)
   - [Evidence Types](#evidence-types)
9. [Compliance Automation](#compliance-automation)
   - [Policy as Code](#policy-as-code)
   - [Continuous Compliance Monitoring](#continuous-compliance-monitoring)
   - [Compliance in CI/CD](#compliance-in-cicd)
10. [Next Steps](#next-steps)
11. [Version History](#version-history)

---

## Overview

Compliance is the practice of meeting regulatory, legal, and contractual requirements for how data is handled, systems are protected, and operations are conducted. For software organizations, compliance requirements come from industry regulations (PCI DSS for payment data, HIPAA for health data), privacy laws (GDPR, CCPA), and customer contracts (SOC 2 reports).

This document covers the major compliance frameworks relevant to software engineering teams, practical implementation of audit logging and evidence collection, and strategies for automating compliance in modern development workflows.

### Target Audience

- **Developers** building features that handle regulated data (payments, health, personal data)
- **DevOps Engineers** implementing audit logging, access controls, and infrastructure compliance
- **Security Engineers** designing and managing compliance programs
- **Engineering Managers** navigating compliance requirements for their teams

### Scope

- Compliance fundamentals: why it matters, compliance vs security, common frameworks
- SOC 2: trust service criteria, controls, audit preparation
- PCI DSS: requirements, cardholder data environment, scope reduction
- GDPR: principles, data subject rights, engineering requirements
- HIPAA: security rule, protected health information
- Audit logging: what to log, architecture, retention, integrity
- Evidence collection: automated collection, evidence types
- Compliance automation: policy as code, continuous monitoring, CI/CD integration

---

## Compliance Fundamentals

### Why Compliance Matters

- **Legal obligation** — Failure to comply with regulations (GDPR, HIPAA) carries significant financial penalties
- **Customer trust** — SOC 2 reports and compliance certifications are often prerequisites for enterprise sales
- **Risk management** — Compliance frameworks provide structured approaches to security that reduce the risk of breaches
- **Contractual requirements** — Enterprise customers require vendors to demonstrate compliance through audits and reports

### Compliance vs Security

| Aspect | Compliance | Security |
|--------|-----------|----------|
| Focus | Meeting specific requirements | Protecting against real threats |
| Scope | Defined by regulations and standards | Comprehensive, risk-based |
| Measurement | Pass/fail against requirements | Continuous, risk score |
| Frequency | Periodic (annual audit) | Continuous |
| Goal | Demonstrate due diligence | Prevent breaches |

**Important:** Compliance is a minimum bar, not the goal. An organization can be fully compliant and still be insecure if the compliance requirements do not cover all relevant threats.

### Common Frameworks

| Framework | Focus Area | Applies To | Mandatory? |
|-----------|-----------|-----------|-----------|
| **SOC 2** | Service organization controls | SaaS, cloud services | Contractual (customer-driven) |
| **PCI DSS** | Payment card data | Any entity handling card data | Yes (if handling card data) |
| **GDPR** | Personal data of EU residents | Any entity processing EU data | Yes (legal) |
| **HIPAA** | Health information | US healthcare and associates | Yes (legal) |
| **ISO 27001** | Information security management | Any organization | Voluntary (but often contractual) |
| **NIST 800-53** | Security controls | US federal systems | Yes (for federal) |
| **CCPA/CPRA** | Personal data of CA residents | Businesses meeting thresholds | Yes (legal) |

---

## SOC 2

### Trust Service Criteria

SOC 2 is based on five Trust Service Criteria (TSC) defined by the AICPA:

| Criteria | Description | Required? |
|----------|-------------|----------|
| **Security** (Common Criteria) | Protection against unauthorized access | ✅ Always required |
| **Availability** | System is available for operation and use | Optional |
| **Processing Integrity** | Processing is complete, valid, accurate, timely | Optional |
| **Confidentiality** | Confidential information is protected | Optional |
| **Privacy** | Personal information is handled per privacy notice | Optional |

### SOC 2 Type I vs Type II

| Aspect | Type I | Type II |
|--------|--------|---------|
| **Scope** | Control design at a point in time | Control effectiveness over a period (6–12 months) |
| **Duration** | Snapshot | 6–12 month observation period |
| **Value** | "Controls exist" | "Controls work consistently" |
| **Customer preference** | Acceptable for early-stage companies | Required by most enterprise customers |

### SOC 2 Controls

Common SOC 2 controls for software organizations:

```
SOC 2 Control Areas for Engineering
────────────────────────────────────
Access Control
  - MFA for all production access
  - Role-based access with least privilege
  - Quarterly access reviews
  - Automated provisioning/deprovisioning

Change Management
  - All changes go through version control
  - Code review required before merge
  - Automated testing in CI/CD
  - Separate environments (dev, staging, production)

Incident Response
  - Documented incident response plan
  - Regular incident response drills
  - Post-incident reviews
  - Customer notification procedures

Monitoring & Logging
  - Centralized logging for all systems
  - Security event monitoring and alerting
  - Audit logs for access and changes
  - Log retention (1 year minimum)

Encryption
  - Encryption at rest (AES-256)
  - Encryption in transit (TLS 1.2+)
  - Key management via KMS
  - Certificate management

Vendor Management
  - Third-party risk assessments
  - Vendor SOC 2 reports reviewed annually
  - Data processing agreements
```

### Preparing for a SOC 2 Audit

1. **Define scope** — Which systems, services, and trust criteria are in scope
2. **Gap assessment** — Compare current controls against SOC 2 requirements
3. **Implement controls** — Close gaps with technical and process controls
4. **Document policies** — Write security policies that map to SOC 2 criteria
5. **Collect evidence** — Automate evidence collection where possible
6. **Engage an auditor** — Select an AICPA-registered CPA firm
7. **Complete the audit** — Provide evidence, answer questions, remediate findings
8. **Maintain continuously** — SOC 2 is ongoing, not a one-time event

---

## PCI DSS

### PCI DSS Requirements

PCI DSS (Payment Card Industry Data Security Standard) has 12 requirement categories:

| # | Requirement | Engineering Impact |
|---|------------|-------------------|
| 1 | Install and maintain network security controls | Network segmentation, firewall rules |
| 2 | Apply secure configurations to all system components | System hardening, remove defaults |
| 3 | Protect stored account data | Encrypt cardholder data at rest |
| 4 | Protect cardholder data with strong cryptography during transmission | TLS for all card data in transit |
| 5 | Protect all systems against malware | Antivirus, endpoint protection |
| 6 | Develop and maintain secure systems and software | Secure SDLC, vulnerability management |
| 7 | Restrict access to system components and cardholder data by business need-to-know | RBAC, least privilege |
| 8 | Identify users and authenticate access to system components | Strong authentication, MFA |
| 9 | Restrict physical access to cardholder data | Physical security (cloud: provider responsibility) |
| 10 | Log and monitor all access to system components and cardholder data | Audit logging, SIEM |
| 11 | Test security of systems and networks regularly | Vulnerability scans, penetration tests |
| 12 | Support information security with organizational policies and programs | Policies, training, incident response |

### Cardholder Data Environment

The **Cardholder Data Environment (CDE)** is the scope of PCI DSS — every system, network segment, and process that stores, processes, or transmits cardholder data.

### PCI DSS Compliance Levels

| Level | Transactions/Year | Requirements |
|-------|------------------|-------------|
| 1 | >6 million | Annual on-site audit by QSA, quarterly network scans |
| 2 | 1–6 million | Annual Self-Assessment Questionnaire (SAQ), quarterly scans |
| 3 | 20,000–1 million (e-commerce) | Annual SAQ, quarterly scans |
| 4 | <20,000 (e-commerce) | Annual SAQ recommended, quarterly scans if applicable |

### Reducing PCI Scope

The most effective strategy is to minimize your PCI scope:

- **Tokenization** — Replace cardholder data with tokens; the token provider handles PCI compliance
- **Hosted payment pages** — Use Stripe Elements, Braintree Drop-in, or PayPal Checkout so card data never touches your servers
- **Point-to-point encryption (P2PE)** — Encrypt card data at the point of interaction

---

## GDPR

### GDPR Principles

The General Data Protection Regulation (GDPR) is based on seven principles:

| Principle | Description |
|-----------|-------------|
| **Lawfulness, fairness, transparency** | Process personal data lawfully, fairly, and transparently |
| **Purpose limitation** | Collect data for specified, explicit, and legitimate purposes |
| **Data minimization** | Collect only the data that is necessary for the stated purpose |
| **Accuracy** | Keep personal data accurate and up to date |
| **Storage limitation** | Retain personal data only as long as necessary |
| **Integrity and confidentiality** | Protect personal data with appropriate security measures |
| **Accountability** | Demonstrate compliance with GDPR principles |

### Data Subject Rights

| Right | Description | Engineering Impact |
|-------|-------------|-------------------|
| **Right to access** | Users can request a copy of their personal data | Build data export functionality |
| **Right to rectification** | Users can correct inaccurate data | Build data editing functionality |
| **Right to erasure** | Users can request deletion of their data | Build data deletion (including backups, logs) |
| **Right to portability** | Users can receive their data in a machine-readable format | Export data in JSON/CSV format |
| **Right to restrict processing** | Users can limit how their data is used | Build processing restriction flags |
| **Right to object** | Users can object to certain data processing | Build opt-out mechanisms |

### GDPR for Engineers

Practical engineering requirements for GDPR compliance:

- **Data inventory** — Know exactly what personal data you collect, where it is stored, and how it flows
- **Consent management** — Implement explicit consent collection with granular opt-in/opt-out
- **Data minimization** — Only collect data that is strictly necessary; do not collect "just in case"
- **Encryption** — Encrypt personal data at rest and in transit
- **Access controls** — Restrict access to personal data to authorized personnel only
- **Data retention** — Automatically delete data after the retention period expires
- **Breach notification** — Report breaches to the supervisory authority within 72 hours
- **Privacy by design** — Consider privacy implications during system design, not as an afterthought

### Data Protection Impact Assessment

A DPIA is required when data processing is likely to result in a high risk to individuals:

- Large-scale processing of sensitive data (health, biometric, criminal records)
- Systematic monitoring of publicly accessible areas
- Automated decision-making with legal or significant effects (credit scoring, profiling)

---

## HIPAA

### HIPAA Security Rule

The HIPAA Security Rule establishes standards for protecting electronic protected health information (ePHI):

| Safeguard | Description | Examples |
|-----------|-------------|---------|
| **Administrative** | Policies and procedures for ePHI management | Security officer, risk assessments, training, incident response |
| **Physical** | Physical access controls for facilities and devices | Facility access controls, workstation security, device disposal |
| **Technical** | Technology-based protections for ePHI | Access controls, audit controls, integrity controls, encryption |

### Protected Health Information

PHI includes any health-related information that can identify an individual:

- Patient names, addresses, dates of birth, Social Security numbers
- Medical record numbers, health plan numbers
- Diagnoses, treatment information, lab results
- Any other information that relates to health and identifies an individual

---

## Audit Logging

### What to Log

| Category | Events to Log |
|----------|--------------|
| **Authentication** | Login success, failure, lockout, MFA challenge, password reset |
| **Authorization** | Access granted, access denied, privilege escalation |
| **Data access** | Read/write of sensitive data, bulk exports, data deletion |
| **Configuration** | System configuration changes, security setting modifications |
| **User management** | User creation, role changes, deactivation |
| **Infrastructure** | Server start/stop, deployment events, certificate rotation |
| **Security events** | Vulnerability detections, intrusion attempts, policy violations |

### Log Architecture

```
Audit Log Architecture
──────────────────────
Applications ──┐
               │
Databases ─────┼──► Log Aggregator ──► Central Log Store ──► SIEM/Analytics
               │    (Fluentd,         (Elasticsearch,       (Splunk,
Infrastructure ┤     Vector,           S3, CloudWatch        Sentinel,
               │     Logstash)         Logs)                 Chronicle)
Identity ──────┘
```

### Log Retention

| Regulation | Minimum Retention | Notes |
|-----------|-------------------|-------|
| SOC 2 | 1 year | Typical minimum; some auditors expect longer |
| PCI DSS | 1 year (3 months readily available) | Audit trail must be available on demand |
| GDPR | As long as necessary for the purpose | Balance retention with data minimization |
| HIPAA | 6 years | From date of creation or last effective date |
| General best practice | 1–3 years | Hot storage (searchable) + cold storage (archive) |

### Log Integrity

Ensure that audit logs cannot be tampered with:

- Write logs to an append-only store (immutable storage)
- Use separate credentials for log writing (application) and log reading (security team)
- Hash log entries or use cryptographic chaining to detect tampering
- Send logs to a centralized system in near-real-time to prevent local deletion
- Restrict access to log management systems (least privilege)

---

## Evidence Collection

### Automated Evidence Collection

Automate evidence collection to reduce the burden of audits and ensure completeness:

- **Access reviews** — Automatically export user-role mappings from IAM systems
- **Configuration snapshots** — Capture infrastructure configuration via IaC state files
- **Scan results** — Export vulnerability scan results, dependency audit reports
- **Deployment logs** — Export CI/CD pipeline execution history
- **Code review evidence** — Export pull request reviews and approval records
- **Training records** — Track security training completion in your LMS

### Evidence Types

| Evidence Type | Source | Automation |
|--------------|--------|-----------|
| Access control lists | IAM, RBAC configuration | API export, scheduled snapshots |
| Change management records | Git history, PR reviews | Git log export, PR API |
| Vulnerability scan results | Trivy, Snyk, Dependabot | CI/CD pipeline artifacts |
| Encryption configuration | Cloud KMS, TLS configuration | Configuration scanner output |
| Incident response records | Incident management system | Ticket export |
| Security training records | LMS, HR system | Training completion reports |
| Audit logs | SIEM, log aggregator | Automated log exports |

---

## Compliance Automation

### Policy as Code

Express compliance requirements as machine-readable policies that can be automatically enforced:

- **Open Policy Agent (OPA)** — General-purpose policy engine for Kubernetes, CI/CD, APIs
- **Checkov** — Scan IaC templates against compliance benchmarks (CIS, SOC 2)
- **AWS Config Rules** — Automatically evaluate AWS resource configuration against compliance rules
- **Azure Policy** — Enforce organizational standards across Azure resources

### Continuous Compliance Monitoring

Instead of periodic point-in-time audits, monitor compliance continuously:

- Deploy CSPM tools to continuously scan cloud configuration
- Set up alerts for compliance drift (e.g., encryption disabled, public S3 bucket)
- Automate evidence collection on a schedule (daily/weekly)
- Generate compliance dashboards for real-time visibility

### Compliance in CI/CD

Integrate compliance checks into your CI/CD pipeline:

```
Compliance Gates in CI/CD
─────────────────────────
1. Code commit
   └─ Secret scanning (prevent secrets in code)

2. Pull request
   └─ SAST scan (code security)
   └─ Dependency scan (vulnerable components)
   └─ License compliance check

3. Build
   └─ Container image scan (vulnerabilities)
   └─ IaC scan (infrastructure compliance)
   └─ SBOM generation

4. Pre-deployment
   └─ Policy gate (OPA, admission controller)
   └─ Artifact signing verification

5. Post-deployment
   └─ Runtime compliance monitoring (CSPM)
   └─ Audit logging verification
```

---

## Next Steps

Continue your security learning journey:

| File | Topic | Description |
|---|---|---|
| [08-INCIDENT-RESPONSE.md](08-INCIDENT-RESPONSE.md) | Incident Response | Security incident handling, forensics, post-mortems |
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Best Practices | Security checklists, shift-left security, automation |
| [05-INFRASTRUCTURE-SECURITY.md](05-INFRASTRUCTURE-SECURITY.md) | Infrastructure Security | Secrets management, network policies, zero trust |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Compliance documentation |
