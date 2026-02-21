# Infrastructure Security

A comprehensive guide to infrastructure security — covering network policies, secrets management, zero trust architecture, cloud security posture, identity and access management, infrastructure hardening, and security monitoring for production environments.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Network Security](#network-security)
   - [Network Segmentation](#network-segmentation)
   - [Firewalls and Security Groups](#firewalls-and-security-groups)
   - [Network Policies in Kubernetes](#network-policies-in-kubernetes)
   - [Web Application Firewalls (WAF)](#web-application-firewalls-waf)
   - [DDoS Protection](#ddos-protection)
3. [Secrets Management](#secrets-management)
   - [What Are Secrets](#what-are-secrets)
   - [Secrets Management Tools](#secrets-management-tools)
   - [HashiCorp Vault](#hashicorp-vault)
   - [Cloud-Native Secrets Managers](#cloud-native-secrets-managers)
   - [Kubernetes Secrets](#kubernetes-secrets)
   - [Secret Rotation](#secret-rotation)
4. [Zero Trust Architecture](#zero-trust-architecture)
   - [Zero Trust Principles](#zero-trust-principles)
   - [Service Mesh](#service-mesh)
   - [Identity-Aware Proxy](#identity-aware-proxy)
   - [Micro-Segmentation](#micro-segmentation)
5. [Cloud Security Posture](#cloud-security-posture)
   - [Cloud Security Benchmarks](#cloud-security-benchmarks)
   - [Cloud Security Posture Management (CSPM)](#cloud-security-posture-management-cspm)
   - [Infrastructure as Code Security](#infrastructure-as-code-security)
6. [Identity and Access Management (IAM)](#identity-and-access-management-iam)
   - [IAM Best Practices](#iam-best-practices)
   - [Service Accounts](#service-accounts)
   - [Workload Identity](#workload-identity)
7. [Infrastructure Hardening](#infrastructure-hardening)
   - [OS Hardening](#os-hardening)
   - [Container Runtime Security](#container-runtime-security)
   - [Patch Management](#patch-management)
8. [Security Monitoring](#security-monitoring)
   - [Security Information and Event Management (SIEM)](#security-information-and-event-management-siem)
   - [Intrusion Detection](#intrusion-detection)
   - [Cloud Security Monitoring](#cloud-security-monitoring)
9. [Next Steps](#next-steps)
10. [Version History](#version-history)

---

## Overview

Infrastructure security protects the platforms, networks, and environments where your applications run. Even the most secure application code is vulnerable if the underlying infrastructure is compromised — network misconfiguration, leaked secrets, excessive IAM permissions, or unpatched servers can all lead to data breaches and service outages.

This document covers the practical controls needed to secure production infrastructure: network segmentation, secrets management, zero trust architecture, cloud security posture, IAM, hardening, and security monitoring.

### Target Audience

- **DevOps Engineers** configuring and securing production infrastructure
- **Platform Engineers** building secure-by-default infrastructure platforms
- **Security Engineers** auditing infrastructure configurations and implementing security controls
- **Cloud Architects** designing secure cloud architectures

### Scope

- Network security: segmentation, firewalls, security groups, WAF, DDoS protection
- Secrets management: tools, patterns, rotation, Kubernetes secrets
- Zero trust architecture: principles, service mesh, identity-aware proxy
- Cloud security posture: benchmarks, CSPM, IaC security scanning
- IAM: least privilege, service accounts, workload identity
- Infrastructure hardening: OS, container runtime, patch management
- Security monitoring: SIEM, intrusion detection, cloud security monitoring

---

## Network Security

### Network Segmentation

Divide your network into isolated segments to limit the blast radius of a compromise. If an attacker gains access to one segment, they should not be able to reach other segments without additional authentication and authorization.

```
Network Segmentation Model
──────────────────────────
┌──────────────────────────────────────────────────┐
│                   Public Zone                     │
│   Load Balancer, CDN, WAF                         │
│   (internet-facing, minimal attack surface)       │
└───────────────────────┬──────────────────────────┘
                        │ Only HTTPS (443)
┌───────────────────────▼──────────────────────────┐
│                   DMZ / Application Zone          │
│   API Servers, Web Servers                        │
│   (can receive traffic from public zone only)     │
└───────────────────────┬──────────────────────────┘
                        │ Only app-specific ports
┌───────────────────────▼──────────────────────────┐
│                   Data Zone                       │
│   Databases, Caches, Message Queues               │
│   (no direct internet access)                     │
└───────────────────────┬──────────────────────────┘
                        │ Restricted
┌───────────────────────▼──────────────────────────┐
│                   Management Zone                 │
│   Bastion Hosts, CI/CD, Monitoring                │
│   (highly restricted access)                      │
└──────────────────────────────────────────────────┘
```

### Firewalls and Security Groups

- **Default deny** — Block all traffic by default; explicitly allow only what is needed
- **Stateful rules** — Use stateful firewalls that track connections (allow return traffic automatically)
- **Principle of least privilege** — Allow only the minimum ports, protocols, and IP ranges
- **Egress filtering** — Restrict outbound traffic (not just inbound) to prevent data exfiltration
- **Regular review** — Audit firewall rules quarterly; remove stale rules

### Network Policies in Kubernetes

Kubernetes Network Policies control traffic flow between pods:

```yaml
# Example: Allow traffic to backend pods only from frontend pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

**Best practices:**

- Start with a default-deny policy for all namespaces
- Explicitly allow only required traffic between services
- Use namespace isolation for different environments and teams
- Use a network policy-aware CNI (Calico, Cilium)

### Web Application Firewalls (WAF)

A WAF inspects HTTP/HTTPS traffic and blocks requests that match attack patterns:

- **Managed rulesets** — Use OWASP Core Rule Set (CRS) as a baseline
- **Custom rules** — Add rules specific to your application (rate limits, geo-blocking)
- **Cloud WAFs** — AWS WAF, Azure Front Door WAF, Cloudflare WAF
- **Self-hosted** — ModSecurity (open source)
- **Tuning** — Monitor false positives and tune rules; never deploy in blocking mode without testing

### DDoS Protection

- Use cloud-native DDoS protection (AWS Shield, Azure DDoS Protection, Cloudflare)
- Place a CDN in front of your application to absorb volumetric attacks
- Implement rate limiting at the API gateway and application level
- Use auto-scaling to absorb traffic spikes
- Have a DDoS response playbook and test it regularly

---

## Secrets Management

### What Are Secrets

Secrets are any credential, key, token, or certificate that must be kept confidential:

- Database passwords and connection strings
- API keys and tokens
- TLS certificates and private keys
- Encryption keys
- SSH keys
- OAuth client secrets
- Service account credentials

**The cardinal rule:** Secrets must never be stored in source code, environment variables (in Dockerfiles), container images, or unencrypted configuration files.

### Secrets Management Tools

| Tool | Type | Features |
|------|------|----------|
| **HashiCorp Vault** | Self-hosted / SaaS | Dynamic secrets, transit encryption, PKI, audit logging |
| **AWS Secrets Manager** | Cloud-native | Automatic rotation, RDS integration, cross-account access |
| **Azure Key Vault** | Cloud-native | Keys, secrets, certificates; HSM-backed |
| **Google Secret Manager** | Cloud-native | Versioning, IAM-based access, audit logging |
| **Doppler** | SaaS | Multi-environment, team sharing, CLI integration |
| **1Password Secrets** | SaaS | Developer-friendly, CI/CD integration |

### HashiCorp Vault

Vault is the most widely used general-purpose secrets manager. Key capabilities:

- **Static secrets** — Key/value store with versioning and access policies
- **Dynamic secrets** — Generate short-lived credentials on demand (database, AWS, etc.)
- **Transit encryption** — Encrypt data without exposing encryption keys to the application
- **PKI** — Issue and manage TLS certificates
- **Audit logging** — Every secret access is logged with who, what, when

### Cloud-Native Secrets Managers

Use your cloud provider's secrets manager for cloud-native workloads:

```
AWS Secrets Manager
───────────────────
- Automatic rotation for RDS, Redshift, DocumentDB
- Cross-account access via resource policies
- Integrates with ECS, EKS, Lambda via IAM roles

Azure Key Vault
───────────────
- Store keys, secrets, and certificates in one service
- HSM-backed for FIPS 140-2 Level 2/3 compliance
- Integrates with App Service, AKS, Functions via managed identity

Google Secret Manager
─────────────────────
- Versioned secrets with IAM-based access control
- Integrates with Cloud Run, GKE, Cloud Functions
- Audit logging via Cloud Audit Logs
```

### Kubernetes Secrets

Kubernetes Secrets are base64-encoded (not encrypted) by default. They require additional controls:

- **Enable encryption at rest** — Configure the Kubernetes API server to encrypt secrets with a KMS provider
- **Use external secrets operators** — Sync secrets from Vault, AWS Secrets Manager, or Azure Key Vault into Kubernetes Secrets (External Secrets Operator)
- **Limit access** — Use RBAC to restrict which pods and service accounts can access which secrets
- **Avoid environment variables** — Mount secrets as volumes rather than environment variables (environment variables appear in container inspection)

### Secret Rotation

- Rotate all secrets on a defined schedule (90 days or less for most secrets)
- Rotate immediately on compromise
- Use dynamic secrets (Vault) to eliminate long-lived credentials
- Automate rotation with your secrets manager's built-in rotation capabilities
- Test rotation in non-production environments before enabling in production
- Ensure applications can handle secret rotation without downtime (graceful reload)

---

## Zero Trust Architecture

### Zero Trust Principles

1. **Never trust, always verify** — Authenticate and authorize every request regardless of source network
2. **Least privilege access** — Grant the minimum access needed for the specific task and time
3. **Assume breach** — Design as if any component could be compromised; minimize blast radius
4. **Verify explicitly** — Use all available signals (identity, device, location, behavior) for access decisions
5. **Encrypt everywhere** — Encrypt all traffic, even between internal services

### Service Mesh

A service mesh provides zero trust networking for microservices:

| Feature | Istio | Linkerd | Cilium |
|---------|-------|---------|--------|
| **mTLS** | Automatic | Automatic | Automatic |
| **Authorization policies** | Yes (Rego, custom) | Yes (Server/HTTPRoute) | Yes (network + L7) |
| **Observability** | Yes | Yes | Yes |
| **Traffic management** | Yes | Yes | Yes |
| **Performance overhead** | Higher | Lower | Lower (eBPF-based) |

### Identity-Aware Proxy

An **identity-aware proxy (IAP)** verifies user identity and device security before granting access to applications — replacing traditional VPN-based access.

- **Google IAP** — Integrated with Google Cloud; verifies identity via Google accounts
- **Cloudflare Access** — Zero trust access for any application; integrates with multiple IdPs
- **Pomerium** — Open-source identity-aware proxy
- Replace VPN access with identity-aware access for internal applications

### Micro-Segmentation

Go beyond network segmentation to control traffic at the workload level:

- Define policies based on service identity, not IP addresses
- Use service mesh authorization policies to control which services can communicate
- Implement egress policies to control outbound traffic from each service
- Log and monitor all inter-service communication

---

## Cloud Security Posture

### Cloud Security Benchmarks

| Benchmark | Focus | Provider |
|-----------|-------|----------|
| **CIS Benchmarks** | Configuration hardening for AWS, Azure, GCP, K8s | Center for Internet Security |
| **AWS Well-Architected (Security Pillar)** | AWS-specific security best practices | AWS |
| **Azure Security Benchmark** | Azure-specific security controls | Microsoft |
| **NIST 800-53** | Comprehensive security controls | NIST |

### Cloud Security Posture Management (CSPM)

CSPM tools continuously monitor your cloud configuration for security issues:

| Tool | Type | Features |
|------|------|----------|
| **AWS Security Hub** | Cloud-native | Aggregates findings from GuardDuty, Inspector, Macie |
| **Microsoft Defender for Cloud** | Cloud-native | Multi-cloud, compliance scores, recommendations |
| **Prowler** | Open source | AWS, Azure, GCP; CIS benchmark scanning |
| **ScoutSuite** | Open source | Multi-cloud security auditing |
| **Wiz** | SaaS | Agentless, graph-based, cross-cloud |

### Infrastructure as Code Security

Scan IaC templates before deployment to catch misconfigurations:

| Tool | Supported IaC | Features |
|------|--------------|----------|
| **Checkov** | Terraform, CloudFormation, K8s, Dockerfile | Open source, custom policies, CI/CD integration |
| **tfsec** | Terraform | Fast, focused on Terraform, custom rules |
| **KICS** | Terraform, CloudFormation, K8s, Ansible, Docker | Open source, multi-framework |
| **Trivy** | Terraform, CloudFormation, K8s, Dockerfile | Multi-purpose scanner including IaC |

---

## Identity and Access Management (IAM)

### IAM Best Practices

- **Least privilege** — Grant only the permissions needed for the specific task
- **No long-lived credentials** — Prefer roles and temporary credentials over static API keys
- **MFA everywhere** — Require MFA for all human access, especially console and admin access
- **Service accounts** — Use dedicated service accounts with scoped permissions for automation
- **Regular audits** — Review IAM policies and permissions quarterly
- **Condition-based policies** — Restrict access by time, IP, resource tags, or MFA status

### Service Accounts

- Create dedicated service accounts for each application or service
- Apply the principle of least privilege — only the permissions needed for that specific service
- Do not share service accounts between applications
- Rotate service account credentials regularly (or use short-lived tokens)
- Monitor service account activity for anomalous behavior

### Workload Identity

Modern cloud platforms support workload identity — assigning cloud IAM identities to workloads without managing static credentials:

- **AWS:** IAM Roles for Service Accounts (IRSA) on EKS, IAM roles for EC2 instances
- **Azure:** Managed Identities for AKS pods, VMs, App Service
- **GCP:** Workload Identity for GKE pods, service account impersonation

Benefits:

- No static credentials to rotate or leak
- Fine-grained, workload-level access control
- Audit trail of which workload accessed which resources

---

## Infrastructure Hardening

### OS Hardening

- Remove unnecessary packages, services, and user accounts
- Disable root login; use sudo with individual accounts
- Enable audit logging (auditd on Linux)
- Configure automatic security updates
- Use immutable infrastructure — replace servers instead of patching in place
- Follow CIS benchmarks for your operating system

### Container Runtime Security

- Run containers as non-root users
- Use read-only file systems where possible
- Drop all Linux capabilities and add only what is needed
- Use seccomp and AppArmor profiles to restrict system calls
- Do not run containers with `--privileged`
- Use runtime security tools (Falco, Sysdig) to detect anomalous container behavior

### Patch Management

- Automate patching for OS packages, language runtimes, and dependencies
- Prioritize patches by severity: critical patches within 24–72 hours
- Test patches in non-production environments before applying to production
- Use immutable infrastructure to ensure patches are applied consistently
- Track patch compliance across all systems

---

## Security Monitoring

### Security Information and Event Management (SIEM)

A SIEM collects, correlates, and analyzes security events from across your infrastructure:

- **Cloud-native:** AWS Security Lake, Microsoft Sentinel, Google Chronicle
- **Self-hosted:** Elastic SIEM, Wazuh, Splunk
- Centralize logs from all sources: applications, infrastructure, identity providers, cloud APIs
- Define detection rules for known attack patterns
- Implement alerting for critical security events

### Intrusion Detection

- **Network-based IDS (NIDS)** — Monitor network traffic for suspicious patterns (Suricata, Zeek)
- **Host-based IDS (HIDS)** — Monitor file changes, process activity, and system calls (OSSEC, Wazuh)
- **Cloud-native** — AWS GuardDuty, Azure Defender, GCP Security Command Center
- Complement detection with response automation (SOAR) where possible

### Cloud Security Monitoring

- Enable cloud audit logging for all accounts and regions (CloudTrail, Azure Activity Logs, GCP Audit Logs)
- Monitor for unusual API activity (new regions, privilege escalation, resource creation)
- Alert on IAM changes (new admin users, policy modifications)
- Use anomaly detection for resource usage (unexpected compute, network, storage)

---

## Next Steps

Continue your security learning journey:

| File | Topic | Description |
|---|---|---|
| [06-SECURE-CODING.md](06-SECURE-CODING.md) | Secure Coding | Input validation, injection prevention, secure defaults |
| [04-SUPPLY-CHAIN-SECURITY.md](04-SUPPLY-CHAIN-SECURITY.md) | Supply Chain Security | Dependency scanning, SBOM, artifact signing |
| [08-INCIDENT-RESPONSE.md](08-INCIDENT-RESPONSE.md) | Incident Response | Security incident handling and forensics |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Infrastructure Security documentation |
