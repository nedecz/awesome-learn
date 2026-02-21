# Incident Response

A comprehensive guide to security incident response — covering incident response plans, detection and classification, containment strategies, eradication and recovery, forensic investigation, post-incident reviews, and building an incident response capability for modern software organizations.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Incident Response Fundamentals](#incident-response-fundamentals)
   - [What is a Security Incident](#what-is-a-security-incident)
   - [Incident Response Lifecycle](#incident-response-lifecycle)
   - [Incident Classification](#incident-classification)
3. [Preparation](#preparation)
   - [Incident Response Plan](#incident-response-plan)
   - [Roles and Responsibilities](#roles-and-responsibilities)
   - [Communication Templates](#communication-templates)
   - [Runbooks](#runbooks)
   - [Incident Response Tools](#incident-response-tools)
4. [Detection and Analysis](#detection-and-analysis)
   - [Detection Sources](#detection-sources)
   - [Indicators of Compromise (IOC)](#indicators-of-compromise-ioc)
   - [Triage Process](#triage-process)
   - [Initial Assessment](#initial-assessment)
5. [Containment](#containment)
   - [Short-Term Containment](#short-term-containment)
   - [Long-Term Containment](#long-term-containment)
   - [Evidence Preservation](#evidence-preservation)
6. [Eradication and Recovery](#eradication-and-recovery)
   - [Root Cause Elimination](#root-cause-elimination)
   - [System Recovery](#system-recovery)
   - [Verification](#verification)
7. [Post-Incident Activity](#post-incident-activity)
   - [Post-Incident Review](#post-incident-review)
   - [Lessons Learned](#lessons-learned)
   - [Metrics and Reporting](#metrics-and-reporting)
8. [Forensic Investigation](#forensic-investigation)
   - [Evidence Collection](#evidence-collection)
   - [Chain of Custody](#chain-of-custody)
   - [Log Analysis](#log-analysis)
9. [Communication](#communication)
   - [Internal Communication](#internal-communication)
   - [Customer Notification](#customer-notification)
   - [Regulatory Notification](#regulatory-notification)
10. [Incident Response Drills](#incident-response-drills)
    - [Tabletop Exercises](#tabletop-exercises)
    - [Simulation Drills](#simulation-drills)
11. [Next Steps](#next-steps)
12. [Version History](#version-history)

---

## Overview

Incident response is the organized approach to addressing and managing the aftermath of a security breach or cyberattack. The goal is to handle the incident in a way that limits damage, reduces recovery time and cost, and prevents recurrence. Every organization that operates software systems will experience security incidents — the difference between a managed incident and a crisis is the quality of your preparation and response.

This document covers the complete incident response lifecycle: preparation, detection, containment, eradication, recovery, and post-incident activity, including forensic investigation, communication, and response drills.

### Target Audience

- **Developers** who may be first responders to security alerts in their services
- **DevOps Engineers** responsible for system recovery and infrastructure containment
- **Security Engineers** leading incident investigation and response
- **Engineering Managers** coordinating team response and stakeholder communication
- **On-Call Engineers** handling security alerts during off-hours

### Scope

- Incident response fundamentals: definition, lifecycle, classification
- Preparation: plans, roles, communication templates, runbooks, tools
- Detection and analysis: sources, indicators of compromise, triage
- Containment: short-term and long-term strategies, evidence preservation
- Eradication and recovery: root cause elimination, system recovery, verification
- Post-incident activity: reviews, lessons learned, metrics
- Forensic investigation: evidence collection, chain of custody, log analysis
- Communication: internal, customer, regulatory notification
- Incident response drills: tabletop exercises, simulations

---

## Incident Response Fundamentals

### What is a Security Incident

A **security incident** is any event that compromises the confidentiality, integrity, or availability of information or information systems. Examples include:

- Unauthorized access to systems or data
- Data breaches (exfiltration of sensitive data)
- Malware or ransomware infections
- Denial-of-service attacks
- Compromised credentials or API keys
- Insider threats (malicious or accidental)
- Supply chain compromises
- Configuration errors exposing sensitive data

### Incident Response Lifecycle

The incident response lifecycle follows the NIST SP 800-61 framework:

```
Incident Response Lifecycle (NIST)
──────────────────────────────────
┌─────────────┐     ┌──────────────────┐     ┌──────────────┐
│ Preparation │────►│ Detection &      │────►│ Containment, │
│             │     │ Analysis         │     │ Eradication, │
│ Plans,      │     │                  │     │ & Recovery   │
│ tools,      │     │ Alerts, triage,  │     │              │
│ training    │     │ classification   │     │ Stop, fix,   │
│             │     │                  │     │ restore      │
└─────────────┘     └──────────────────┘     └──────┬───────┘
       ▲                                            │
       │            ┌──────────────────┐            │
       └────────────│ Post-Incident    │◄───────────┘
                    │ Activity         │
                    │                  │
                    │ Review, lessons, │
                    │ improvements     │
                    └──────────────────┘
```

### Incident Classification

| Severity | Description | Response Time | Examples |
|----------|-------------|--------------|---------|
| **Critical (P1)** | Active data breach or system-wide compromise | Immediate (within 15 min) | Data exfiltration, ransomware, production database exposed |
| **High (P2)** | Confirmed compromise with limited scope | Within 1 hour | Compromised service account, targeted attack in progress |
| **Medium (P3)** | Potential compromise requiring investigation | Within 4 hours | Suspicious activity, vulnerability being actively exploited |
| **Low (P4)** | Security event requiring follow-up | Within 24 hours | Failed brute-force attempt, policy violation, phishing attempt |

---

## Preparation

### Incident Response Plan

An incident response plan is a documented, tested set of procedures for handling security incidents:

```
Incident Response Plan Components
──────────────────────────────────
1. Purpose and scope — What incidents does this plan cover?
2. Roles and responsibilities — Who does what during an incident?
3. Classification criteria — How do we categorize incident severity?
4. Communication plan — Who do we notify, when, and how?
5. Escalation procedures — When and how to escalate to leadership and legal
6. Containment strategies — Pre-approved actions for common incident types
7. Recovery procedures — How to restore services safely
8. Evidence preservation — How to collect and protect forensic evidence
9. Post-incident review process — How we learn from every incident
10. Regulatory notification requirements — GDPR (72h), PCI DSS, state breach laws
```

### Roles and Responsibilities

| Role | Responsibilities |
|------|-----------------|
| **Incident Commander (IC)** | Coordinates response, makes decisions, owns communication |
| **Technical Lead** | Directs technical investigation and containment |
| **Communications Lead** | Manages internal and external communications |
| **Operations Lead** | Handles system recovery and service restoration |
| **Scribe** | Documents timeline, actions, and decisions in real time |
| **Subject Matter Experts** | Provide expertise on affected systems (database, network, application) |

### Communication Templates

Prepare templates in advance for common communications:

- **Internal alert** — Notify the incident response team of a new incident
- **Status update** — Regular updates during an active incident (every 30–60 min for P1/P2)
- **Customer notification** — Inform affected customers of a security incident
- **Regulatory notification** — Notify regulators as required (GDPR: 72 hours)
- **All-clear** — Communicate that the incident is resolved and services are restored
- **Post-incident summary** — Summary of what happened, impact, and actions taken

### Runbooks

Runbooks are step-by-step procedures for responding to specific incident types:

| Incident Type | Key Runbook Steps |
|--------------|-------------------|
| **Compromised credentials** | Reset credentials, revoke sessions, audit access logs, notify affected users |
| **Data breach** | Contain the leak, assess scope, preserve evidence, notify legal and compliance |
| **Ransomware** | Isolate affected systems, do NOT pay ransom, restore from clean backups |
| **DDoS attack** | Enable DDoS protection, scale infrastructure, identify and block attack vectors |
| **Compromised dependency** | Identify affected systems, update/remove dependency, scan for exploitation |
| **Leaked secrets** | Rotate all exposed secrets, audit usage, scan for unauthorized access |

### Incident Response Tools

| Category | Tools |
|----------|-------|
| **Communication** | Slack/Teams incident channel, PagerDuty, Opsgenie |
| **Investigation** | SIEM (Splunk, Sentinel), log aggregator (ELK, Loki) |
| **Forensics** | Velociraptor, GRR Rapid Response, cloud-native forensics |
| **Documentation** | Incident management platforms (PagerDuty, Jira, Rootly) |
| **Containment** | Network isolation tools, IAM emergency policies, WAF rules |

---

## Detection and Analysis

### Detection Sources

| Source | What It Detects |
|--------|----------------|
| **SIEM alerts** | Correlated security events across multiple systems |
| **IDS/IPS** | Network-based attack patterns (Suricata, Snyk) |
| **Application logs** | Authentication failures, authorization violations, error spikes |
| **Cloud security tools** | GuardDuty, Defender for Cloud, Security Command Center |
| **User reports** | Phishing emails, suspicious activity, unauthorized access |
| **Vulnerability scanners** | Newly discovered vulnerabilities in production |
| **External reports** | Bug bounty submissions, security researcher disclosures |
| **Threat intelligence** | Indicators of compromise matching your environment |

### Indicators of Compromise (IOC)

| IOC Type | Examples |
|----------|---------|
| **Network** | Unexpected outbound connections, unusual DNS queries, traffic to known C2 servers |
| **Host** | New processes, unexpected file changes, disabled security tools |
| **Account** | Failed login spikes, logins from unusual locations, new admin accounts |
| **Application** | Unusual API call patterns, data exports, error rate spikes |
| **Data** | Unexpected database queries, bulk data access, files in unexpected locations |

### Triage Process

1. **Acknowledge** — Confirm receipt of the alert within SLA
2. **Assess** — Is this a real incident or a false positive?
3. **Classify** — Assign severity (P1–P4) based on impact and scope
4. **Assign** — Assign incident commander and assemble the response team
5. **Document** — Create an incident record and start the timeline

### Initial Assessment

Answer these questions during initial assessment:

- What systems are affected?
- What data is at risk?
- Is the attack still in progress?
- How did the attacker gain access?
- What is the blast radius (scope of potential damage)?
- Are there regulatory notification requirements?

---

## Containment

### Short-Term Containment

Immediate actions to stop the bleeding (minutes to hours):

- Isolate affected systems from the network
- Block attacker IP addresses and domains
- Revoke compromised credentials and tokens
- Disable compromised accounts
- Apply emergency firewall rules or WAF blocks
- Take affected services offline if necessary

### Long-Term Containment

Sustainable containment while the root cause is addressed:

- Deploy temporary monitoring on affected systems
- Implement additional access controls
- Set up enhanced logging for the affected area
- Coordinate with cloud providers for account-level containment
- Maintain service availability with clean, patched systems

### Evidence Preservation

Preserve evidence before making changes that could destroy it:

- Take snapshots of affected virtual machines, volumes, and containers
- Export relevant logs before they rotate or are overwritten
- Capture memory dumps of running processes if malware is suspected
- Document the state of affected systems (screenshots, configuration exports)
- Follow chain of custody procedures for evidence that may be used in legal proceedings

---

## Eradication and Recovery

### Root Cause Elimination

Completely remove the attacker's presence and fix the underlying vulnerability:

- Patch the vulnerability that was exploited
- Remove malware, backdoors, and unauthorized accounts
- Reset all credentials that may have been compromised
- Verify that no persistence mechanisms remain (cron jobs, startup scripts, SSH keys)
- Update firewall rules and security configurations

### System Recovery

Restore affected systems to normal operation:

- Restore from known-good backups (verify backup integrity before restoring)
- Rebuild affected systems from clean images (prefer rebuild over clean-in-place)
- Verify that restored systems are free of compromise
- Gradually restore network connectivity and user access
- Monitor closely for signs of re-compromise during recovery

### Verification

Before declaring recovery complete:

- Run vulnerability scans on all affected systems
- Verify that all compromised credentials have been rotated
- Confirm that audit logging is functioning correctly
- Test security controls to ensure they are working
- Validate that the attacker's original access path is closed
- Monitor for indicators of compromise for an extended period (30+ days)

---

## Post-Incident Activity

### Post-Incident Review

Conduct a blameless post-incident review within 3–5 business days of incident resolution:

```
Post-Incident Review Agenda
────────────────────────────
1. Timeline — What happened, when, and in what order?
2. Detection — How was the incident detected? How long was the gap between compromise and detection?
3. Response — What actions were taken? What worked? What didn't?
4. Root cause — What was the underlying vulnerability or failure?
5. Impact — What was the scope? What data/systems were affected?
6. Contributing factors — What conditions enabled this incident?
7. Action items — What specific changes will prevent recurrence?
8. Process improvements — What should we change in our response process?
```

### Lessons Learned

Convert post-incident findings into concrete improvements:

- Update runbooks based on what worked and what was missing
- Improve detection rules to catch similar incidents faster
- Add automated responses for common containment actions
- Update the incident response plan with process improvements
- Share anonymized lessons with the broader team
- Track action items to completion

### Metrics and Reporting

| Metric | Description | Target |
|--------|-------------|--------|
| **MTTD** (Mean Time to Detect) | Time from compromise to detection | Minimize |
| **MTTR** (Mean Time to Respond) | Time from detection to containment | < SLA for severity level |
| **MTTC** (Mean Time to Contain) | Time from detection to full containment | Hours (P1), days (P2) |
| **Incidents per quarter** | Number of security incidents | Trending down |
| **Action item completion rate** | % of post-incident actions completed on time | > 90% |

---

## Forensic Investigation

### Evidence Collection

Collect evidence systematically and comprehensively:

- **Volatile evidence first** — Memory dumps, network connections, running processes (disappear on reboot)
- **Disk images** — Full disk images of affected systems
- **Log exports** — Application logs, system logs, cloud audit logs, network logs
- **Network captures** — Packet captures from affected network segments
- **Configuration snapshots** — Current state of system configurations
- **Cloud API logs** — CloudTrail, Azure Activity Logs, GCP Audit Logs

### Chain of Custody

If evidence may be used in legal proceedings, maintain chain of custody:

- Document who collected each piece of evidence, when, and how
- Store evidence in a secure, access-controlled location
- Use cryptographic hashes to prove evidence has not been tampered with
- Track every access to the evidence
- Use tamper-evident storage (locked containers, access-logged digital storage)

### Log Analysis

Focus your log analysis on answering key questions:

- **When** did the compromise begin? (look for the earliest indicator)
- **How** did the attacker gain initial access? (authentication logs, application logs)
- **What** did the attacker do? (access logs, data access patterns, command history)
- **What** data was accessed or exfiltrated? (database query logs, file access logs)
- **Where** did the attacker go? (network logs, lateral movement indicators)
- **Are they still present?** (check for persistence mechanisms)

---

## Communication

### Internal Communication

- Create a dedicated incident channel (Slack/Teams) for real-time coordination
- Post regular status updates at defined intervals (every 30 min for P1, every 2 hours for P2)
- Brief executive leadership for P1/P2 incidents
- Share final post-incident report with the engineering organization

### Customer Notification

If customer data is affected:

- Notify affected customers promptly and transparently
- Describe what happened, what data was affected, and what you are doing about it
- Provide actionable guidance (change passwords, monitor accounts)
- Provide a point of contact for questions
- Follow up with resolution and prevention measures

### Regulatory Notification

| Regulation | Notification Deadline | Who to Notify |
|-----------|----------------------|---------------|
| GDPR | 72 hours from awareness | Supervisory authority + affected individuals |
| HIPAA | 60 days from discovery | HHS, affected individuals, media (500+ individuals) |
| PCI DSS | Immediately | Payment brands, acquiring bank |
| State breach laws | Varies (30–90 days) | State AG, affected individuals |

---

## Incident Response Drills

### Tabletop Exercises

A **tabletop exercise** is a discussion-based drill where the incident response team walks through a hypothetical scenario:

- Schedule quarterly tabletop exercises
- Use realistic scenarios based on your threat model
- Include all IR team roles (IC, technical lead, communications, legal)
- Focus on decision-making, communication, and process gaps
- Document findings and update the IR plan accordingly

### Simulation Drills

Go beyond tabletop exercises with hands-on simulations:

- **Red team exercises** — An internal or external team attempts to breach your systems
- **Purple team exercises** — Red and blue teams work together to improve detection and response
- **Chaos engineering** — Inject failures to test incident response and recovery
- **Phishing simulations** — Test employee awareness and reporting behavior
- **Recovery drills** — Practice restoring systems from backups after a simulated compromise

---

## Next Steps

Continue your security learning journey:

| File | Topic | Description |
|---|---|---|
| [07-COMPLIANCE.md](07-COMPLIANCE.md) | Compliance | SOC 2, PCI DSS, GDPR, audit logging |
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Best Practices | Security checklists, shift-left security |
| [05-INFRASTRUCTURE-SECURITY.md](05-INFRASTRUCTURE-SECURITY.md) | Infrastructure Security | Network policies, secrets management |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Incident Response documentation |
