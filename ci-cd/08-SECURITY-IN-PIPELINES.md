# Security in Pipelines

A comprehensive guide to security in CI/CD pipelines — covering DevSecOps principles, static and dynamic application security testing (SAST/DAST), dependency scanning, container image scanning, infrastructure as code scanning, secret detection, software bill of materials (SBOM), supply chain security, compliance as code, and vulnerability management workflows.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [DevSecOps and Shift-Left Security](#devsecops-and-shift-left-security)
   - [Integrating Security into CI/CD](#integrating-security-into-cicd)
   - [Security as Code](#security-as-code)
   - [The Security Pipeline Architecture](#the-security-pipeline-architecture)
3. [Static Application Security Testing (SAST)](#static-application-security-testing-sast)
   - [How SAST Works](#how-sast-works)
   - [SAST Tools Comparison](#sast-tools-comparison)
   - [SonarQube Integration](#sonarqube-integration)
   - [Semgrep Integration](#semgrep-integration)
   - [CodeQL Integration](#codeql-integration)
4. [Dynamic Application Security Testing (DAST)](#dynamic-application-security-testing-dast)
   - [How DAST Works](#how-dast-works)
   - [OWASP ZAP in Pipelines](#owasp-zap-in-pipelines)
   - [Burp Suite in Pipelines](#burp-suite-in-pipelines)
   - [DAST vs SAST Tradeoffs](#dast-vs-sast-tradeoffs)
5. [Software Composition Analysis (SCA)](#software-composition-analysis-sca)
   - [Dependency Scanning](#dependency-scanning)
   - [Dependabot](#dependabot)
   - [Snyk](#snyk)
   - [Trivy for SCA](#trivy-for-sca)
6. [Container Image Scanning](#container-image-scanning)
   - [Trivy](#trivy)
   - [Grype](#grype)
   - [Docker Scout](#docker-scout)
   - [Scanning in CI](#scanning-in-ci)
7. [Infrastructure as Code Scanning](#infrastructure-as-code-scanning)
   - [Checkov](#checkov)
   - [tfsec](#tfsec)
   - [KICS](#kics)
   - [Policy Enforcement](#policy-enforcement)
8. [Secret Detection](#secret-detection)
   - [GitLeaks](#gitleaks)
   - [TruffleHog](#trufflehog)
   - [Pre-Commit Hooks](#pre-commit-hooks)
   - [GitHub Secret Scanning](#github-secret-scanning)
9. [Software Bill of Materials (SBOM)](#software-bill-of-materials-sbom)
   - [CycloneDX](#cyclonedx)
   - [SPDX](#spdx)
   - [SBOM Generation and Attestation](#sbom-generation-and-attestation)
10. [Supply Chain Security](#supply-chain-security)
    - [SLSA Framework](#slsa-framework)
    - [Sigstore and Cosign](#sigstore-and-cosign)
    - [Artifact Signing](#artifact-signing)
    - [Provenance](#provenance)
11. [Compliance as Code](#compliance-as-code)
    - [Open Policy Agent](#open-policy-agent)
    - [Policy Gates in Pipelines](#policy-gates-in-pipelines)
12. [Vulnerability Management Workflow](#vulnerability-management-workflow)
    - [Triage](#triage)
    - [SLA and Exceptions](#sla-and-exceptions)
    - [Reporting](#reporting)
13. [Security Gate Implementation](#security-gate-implementation)
    - [Blocking vs Advisory](#blocking-vs-advisory)
    - [Risk Scoring](#risk-scoring)
    - [Break-the-Build Thresholds](#break-the-build-thresholds)
14. [Next Steps](#next-steps)
15. [Version History](#version-history)

---

## Overview

Security cannot be an afterthought bolted onto the end of a release cycle. When security scanning only happens in production or during quarterly audits, vulnerabilities live in deployed code for weeks or months. The cost of fixing a vulnerability grows exponentially the later it is discovered — a flaw caught during code review costs minutes, while the same flaw found in production costs incident response, emergency patches, and potentially breached customer data.

This document covers the full spectrum of automated security tooling in CI/CD pipelines: how to scan source code, dependencies, container images, and infrastructure definitions; how to detect secrets, generate SBOMs, sign artifacts, and enforce compliance policies — all integrated into every commit and pull request.

### Target Audience

- **DevOps Engineers** embedding security scanning into CI/CD pipelines
- **Developers** writing secure code and responding to automated security findings
- **Security Engineers** designing AppSec programs and vulnerability management workflows
- **Platform Engineers** building secure-by-default pipeline templates and policy gates

### Scope

- DevSecOps principles and shift-left security integration
- SAST, DAST, and SCA tools and their pipeline integration patterns
- Container image and Infrastructure as Code scanning
- Secret detection and prevention
- SBOM generation, supply chain security, and artifact signing
- Compliance as code and policy enforcement
- Vulnerability management workflows and security gate implementation

---

## DevSecOps and Shift-Left Security

### Integrating Security into CI/CD

Shift-left security means moving security checks earlier in the software development lifecycle — from production and staging all the way into the developer's pull request. Every commit should trigger security analysis, and every pull request should include security feedback alongside test results.

```
Traditional Security vs Shift-Left Security
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Traditional: Security at the end
  ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐   ┌──────────────┐
  │ Code │──▶│Build │──▶│ Test │──▶│Deploy│──▶│Security Audit│
  └──────┘   └──────┘   └──────┘   └──────┘   └──────────────┘
                                                  ▲
                                                  │ Expensive
                                                  │ Late discovery
                                                  │ Slow remediation

  Shift-Left: Security everywhere
  ┌──────────────────────────────────────────────────────────┐
  │                    Security Scanning                      │
  │  ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐  │
  │  │ Code │──▶│Build │──▶│ Test │──▶│Stage │──▶│Deploy│  │
  │  └──┬───┘   └──┬───┘   └──┬───┘   └──┬───┘   └──┬───┘  │
  │     │          │          │          │          │        │
  │   SAST       Image     DAST      Compliance  Runtime   │
  │   Secrets    SCA       Pentest   Policy      Monitoring│
  │   Linting    SBOM                OPA                    │
  └──────────────────────────────────────────────────────────┘
```

### Security as Code

Security policies, scanning configurations, and compliance rules should be version-controlled alongside application code. This approach enables:

- **Reproducibility** — the same security checks run identically across all environments
- **Auditability** — every policy change is tracked in Git history
- **Collaboration** — developers and security teams review the same pull requests
- **Automation** — policies are enforced automatically, not manually

```yaml
# .security/policy.yaml — version-controlled security policy
security:
  sast:
    enabled: true
    tools: [semgrep, codeql]
    fail_on: [critical, high]
    ignore_paths:
      - "test/**"
      - "vendor/**"

  sca:
    enabled: true
    tools: [trivy]
    fail_on_cvss: 7.0
    ignore_unfixed: true

  secrets:
    enabled: true
    tools: [gitleaks]
    pre_commit: true

  container:
    enabled: true
    tools: [trivy]
    fail_on: [critical]
    ignore_unfixed: true

  iac:
    enabled: true
    tools: [checkov]
    framework: [terraform, kubernetes]
    fail_on: [high, critical]
```

### The Security Pipeline Architecture

A comprehensive security pipeline integrates multiple scanning stages without significantly increasing build time. Parallel execution is the key — most security scans can run simultaneously.

```
Security Pipeline Architecture
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌───────────┐
  │  PR / Push│
  └─────┬─────┘
        │
        ▼
  ┌───────────────────────────────────────────────────┐
  │              Parallel Security Scans               │
  │                                                    │
  │  ┌─────────┐  ┌─────────┐  ┌─────────┐           │
  │  │  SAST   │  │ Secrets │  │   IaC   │           │
  │  │ Semgrep │  │GitLeaks │  │ Checkov │           │
  │  │ CodeQL  │  │         │  │         │           │
  │  └────┬────┘  └────┬────┘  └────┬────┘           │
  │       │            │            │                  │
  │  ┌────┴────┐  ┌────┴────┐  ┌────┴────┐           │
  │  │   SCA   │  │Container│  │  SBOM   │           │
  │  │  Trivy  │  │  Scan   │  │CycloneDX│           │
  │  │  Snyk   │  │  Grype  │  │         │           │
  │  └────┬────┘  └────┬────┘  └────┬────┘           │
  │       │            │            │                  │
  └───────┼────────────┼────────────┼──────────────────┘
          │            │            │
          ▼            ▼            ▼
  ┌───────────────────────────────────────────────────┐
  │              Security Gate (Policy)                │
  │                                                    │
  │   Aggregate findings → Apply thresholds → Pass/Fail│
  └─────────────────────────┬─────────────────────────┘
                            │
                     ┌──────┴──────┐
                     │  Pass/Fail  │
                     └──────┬──────┘
                            │
                 ┌──────────┴──────────┐
                 ▼                     ▼
          ┌────────────┐       ┌────────────┐
          │   Deploy   │       │   Block    │
          │  (if pass) │       │ (if fail)  │
          └────────────┘       └────────────┘
```

```yaml
# GitHub Actions — parallel security scanning pipeline
name: Security Pipeline

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/default
            p/owasp-top-ten
            p/cwe-top-25

  secrets-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: fs
          severity: CRITICAL,HIGH
          exit-code: 1

  iac-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Checkov
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: ./terraform
          soft_fail: false

  security-gate:
    needs: [sast, secrets-scan, dependency-scan, iac-scan]
    runs-on: ubuntu-latest
    steps:
      - name: All security checks passed
        run: echo "All security scans passed — deployment approved"
```

---

## Static Application Security Testing (SAST)

### How SAST Works

SAST tools analyze source code, bytecode, or binaries without executing the application. They parse the code into an abstract syntax tree (AST) or control flow graph (CFG) and apply rules to detect patterns that match known vulnerability classes — SQL injection, cross-site scripting, path traversal, buffer overflows, and more.

```
SAST Analysis Flow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Source Code
      │
      ▼
  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐
  │   Parser     │───▶│  AST / CFG   │───▶│ Rule Engine      │
  │ (per-lang)   │    │ Construction │    │ (pattern match)  │
  └──────────────┘    └──────────────┘    └────────┬─────────┘
                                                   │
                                          ┌────────▼─────────┐
                                          │   Findings        │
                                          │ - Severity        │
                                          │ - CWE ID          │
                                          │ - File + Line     │
                                          │ - Remediation     │
                                          └──────────────────┘

  Key Characteristics:
  ✅ No running application needed
  ✅ Finds bugs early (in source)
  ✅ Language-specific precision
  ⚠️  False positives common
  ⚠️  Cannot detect runtime issues
  ⚠️  Configuration issues often missed
```

### SAST Tools Comparison

| Tool | Languages | License | CI Integration | False Positive Rate | Analysis Depth |
|---|---|---|---|---|---|
| **SonarQube** | 30+ | Community (free) / Enterprise | GitHub, GitLab, Azure DevOps | Medium | AST + dataflow |
| **Semgrep** | 30+ | LGPL / Team (paid) | Native GitHub Action | Low | Pattern matching |
| **CodeQL** | 12+ | Free for OSS / GHAS | GitHub-native | Low | Full dataflow + taint |
| **Checkmarx** | 25+ | Commercial | All major CI | Medium-High | Deep dataflow |
| **Snyk Code** | 10+ | Free tier / Paid | GitHub, GitLab, Bitbucket | Low-Medium | AI-assisted |
| **Bandit** | Python only | Apache 2.0 | Any CI | Medium | AST pattern matching |

### SonarQube Integration

SonarQube provides a comprehensive code quality and security platform. The scanner runs in your pipeline and sends results to a central SonarQube server for analysis and tracking.

```yaml
# GitHub Actions — SonarQube SAST scan
name: SonarQube Analysis

on:
  pull_request:
    branches: [main]

jobs:
  sonarqube:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: SonarQube Scan
        uses: SonarSource/sonarqube-scan-action@v5
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
        with:
          args: >
            -Dsonar.projectKey=my-project
            -Dsonar.sources=src/
            -Dsonar.tests=tests/
            -Dsonar.test.inclusions=**/*test*/**
            -Dsonar.coverage.exclusions=**/*test*/**

      - name: SonarQube Quality Gate
        uses: SonarSource/sonarqube-quality-gate-action@v1
        timeout-minutes: 5
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

### Semgrep Integration

Semgrep is a fast, lightweight static analysis tool that uses pattern-matching rules. Its rules are written in YAML and are easy to customize, making it popular for custom security policies.

```yaml
# GitHub Actions — Semgrep SAST scan
name: Semgrep

on:
  pull_request:
    branches: [main]

jobs:
  semgrep:
    runs-on: ubuntu-latest
    container:
      image: semgrep/semgrep
    steps:
      - uses: actions/checkout@v4

      - name: Run Semgrep
        run: semgrep scan --config auto --sarif --output semgrep.sarif
        env:
          SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}

      - name: Upload SARIF
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: semgrep.sarif
```

```yaml
# .semgrep/custom-rules.yaml — writing custom Semgrep rules
rules:
  - id: no-exec-user-input
    patterns:
      - pattern: subprocess.run($CMD, ...)
      - pattern-not: subprocess.run(["...", ...], ...)
      - metavariable-pattern:
          metavariable: $CMD
          pattern: |
            request.$METHOD(...)
    message: >
      User input passed to subprocess.run() without sanitization.
      This can lead to command injection (CWE-78).
    severity: ERROR
    languages: [python]
    metadata:
      cwe: "CWE-78"
      owasp: "A03:2021 Injection"

  - id: no-hardcoded-secrets
    pattern-either:
      - pattern: |
          $VAR = "AKIA..."
      - pattern: |
          password = "..."
    message: Hardcoded credential detected
    severity: WARNING
    languages: [python, javascript, go]
```

### CodeQL Integration

CodeQL is GitHub's semantic code analysis engine. It treats code as data — building a relational database from the codebase and running queries against it. This enables deep dataflow and taint-tracking analysis.

```yaml
# GitHub Actions — CodeQL analysis
name: CodeQL

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
  schedule:
    - cron: "0 6 * * 1"

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      contents: read
    strategy:
      matrix:
        language: [javascript, python]
    steps:
      - uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
          queries: +security-extended,security-and-quality

      - name: Autobuild
        uses: github/codeql-action/autobuild@v3

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: "/language:${{ matrix.language }}"
```

---

## Dynamic Application Security Testing (DAST)

### How DAST Works

DAST tools test a running application by sending crafted HTTP requests and analyzing the responses. They simulate an attacker's perspective — probing for XSS, SQL injection, authentication flaws, and misconfigurations — without access to the source code.

```
DAST Analysis Flow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌─────────────┐     ┌──────────────┐     ┌──────────────┐
  │ DAST Scanner│────▶│   Running    │────▶│   Analyze    │
  │  (ZAP/Burp) │◀────│  Application │◀────│  Responses   │
  └─────────────┘     └──────────────┘     └──────┬───────┘
        │                                          │
        │  1. Crawl / Spider                       │
        │  2. Inject payloads                      │
        │  3. Observe behavior                     │
        │                                          ▼
        │                                  ┌──────────────┐
        │                                  │   Report     │
        └──────────────────────────────────│  Findings    │
                                           └──────────────┘

  Key Characteristics:
  ✅ Tests real runtime behavior
  ✅ Finds configuration issues
  ✅ Language-agnostic
  ⚠️  Requires running application
  ⚠️  Slow (minutes to hours)
  ⚠️  Limited code-level detail
```

### OWASP ZAP in Pipelines

OWASP ZAP (Zed Attack Proxy) is the most widely used open-source DAST tool. It supports automated scanning in CI via Docker containers and a comprehensive API.

```yaml
# GitHub Actions — OWASP ZAP baseline scan
name: DAST

on:
  pull_request:
    branches: [main]

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    outputs:
      url: ${{ steps.deploy.outputs.url }}
    steps:
      - uses: actions/checkout@v4
      - name: Deploy preview environment
        id: deploy
        run: |
          # Deploy to a preview environment and capture URL
          echo "url=https://preview-${{ github.event.pull_request.number }}.example.com" >> "$GITHUB_OUTPUT"

  zap-scan:
    needs: deploy-preview
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.14.0
        with:
          target: ${{ needs.deploy-preview.outputs.url }}
          rules_file_name: .zap/rules.tsv
          cmd_options: "-a"

      - name: Upload ZAP Report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: zap-report
          path: report_html.html
```

```yaml
# GitHub Actions — ZAP full scan with custom policy
name: ZAP Full Scan

on:
  schedule:
    - cron: "0 2 * * 1"

jobs:
  zap-full:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: ZAP Full Scan
        uses: zaproxy/action-full-scan@v0.12.0
        with:
          target: https://staging.example.com
          rules_file_name: .zap/rules.tsv
          cmd_options: >
            -z "-config scanner.strength=INSANE
                 -config scanner.threshold=LOW"
```

### Burp Suite in Pipelines

Burp Suite Enterprise enables automated DAST scanning through its REST API. While it is a commercial tool, it offers deep scanning capabilities and integrates with CI via its API.

```yaml
# GitHub Actions — Burp Suite Enterprise scan
name: Burp Suite DAST

on:
  workflow_dispatch:
    inputs:
      target_url:
        description: "Target URL to scan"
        required: true

jobs:
  burp-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger Burp Suite scan
        id: trigger
        run: |
          SCAN_ID=$(curl -s -X POST \
            "${{ secrets.BURP_API_URL }}/api/scans" \
            -H "Authorization: Bearer ${{ secrets.BURP_API_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{
              "scan_configurations": [{"type": "NamedConfiguration", "name": "Audit checks - light"}],
              "urls": ["${{ inputs.target_url }}"]
            }' | jq -r '.scan_id')
          echo "scan_id=$SCAN_ID" >> "$GITHUB_OUTPUT"

      - name: Wait for scan completion
        run: |
          while true; do
            STATUS=$(curl -s \
              "${{ secrets.BURP_API_URL }}/api/scans/${{ steps.trigger.outputs.scan_id }}" \
              -H "Authorization: Bearer ${{ secrets.BURP_API_TOKEN }}" | jq -r '.status')
            if [ "$STATUS" = "succeeded" ] || [ "$STATUS" = "failed" ]; then
              break
            fi
            sleep 30
          done
```

### DAST vs SAST Tradeoffs

| Aspect | SAST | DAST |
|---|---|---|
| **When it runs** | On source code (pre-build) | On running application |
| **Speed** | Fast (seconds to minutes) | Slow (minutes to hours) |
| **Coverage** | All code paths (static) | Only reachable endpoints |
| **False positives** | Higher | Lower |
| **Language dependency** | Language-specific | Language-agnostic |
| **Finds** | Code-level bugs, injection, XSS patterns | Runtime config, auth flaws, headers |
| **Misses** | Runtime config, business logic | Dead code, unreachable paths |
| **Pipeline stage** | PR / build | Post-deployment / staging |
| **Remediation detail** | Exact file + line number | URL + request/response |
| **Best for** | Developer feedback loop | Pre-production validation |

> **Best practice**: Use both SAST and DAST. SAST catches code-level flaws early. DAST validates runtime security before production. They are complementary, not competing.

---

## Software Composition Analysis (SCA)

### Dependency Scanning

Modern applications are 70–90% open-source code by volume. SCA tools scan your dependency manifests (package.json, requirements.txt, go.sum, pom.xml) and lock files against vulnerability databases like the National Vulnerability Database (NVD) and GitHub Advisory Database.

```
SCA Scanning Flow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌──────────────┐     ┌────────────────┐     ┌─────────────┐
  │  Dependency   │     │  Vulnerability  │     │  Findings   │
  │  Manifests    │────▶│   Databases     │────▶│  Report     │
  │               │     │                │     │             │
  │ package.json  │     │ NVD            │     │ CVE-2024-XX │
  │ go.sum        │     │ GitHub Advisory│     │ CVSS: 9.8   │
  │ requirements  │     │ OSV            │     │ Fix: 2.3.1  │
  │ Gemfile.lock  │     │ Snyk DB        │     │             │
  └──────────────┘     └────────────────┘     └─────────────┘

  Dependency Tree Depth Matters:
  ┌─────────┐
  │ Your App│
  └────┬────┘
       ├──▶ express@4.18.2      ← Direct dependency
       │    ├──▶ qs@6.11.0      ← Transitive (depth 1)
       │    │   └──▶ side-channel@1.0.4
       │    └──▶ body-parser@1.20.1
       │        └──▶ qs@6.11.0  ← Same transitive dep
       └──▶ lodash@4.17.21
            └──▶ (no deps)

  Most vulnerabilities are in transitive dependencies.
```

### Dependabot

Dependabot is GitHub's built-in dependency update and vulnerability alerting tool. It monitors dependency manifests and automatically creates pull requests to update vulnerable or outdated packages.

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
      day: monday
    open-pull-requests-limit: 10
    reviewers:
      - security-team
    labels:
      - dependencies
      - security
    groups:
      production-deps:
        dependency-type: production
      dev-deps:
        dependency-type: development
        update-types:
          - minor
          - patch
    ignore:
      - dependency-name: "aws-sdk"
        update-types: ["version-update:semver-major"]

  - package-ecosystem: docker
    directory: /
    schedule:
      interval: weekly

  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

### Snyk

Snyk provides SCA, SAST, container scanning, and IaC scanning in a unified platform. Its SCA engine is known for high-quality vulnerability data and actionable fix recommendations.

```yaml
# GitHub Actions — Snyk dependency scan
name: Snyk Security

on:
  pull_request:
    branches: [main]

jobs:
  snyk:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/node@master
        continue-on-error: true
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: >
            --severity-threshold=high
            --fail-on=upgradable

      - name: Upload Snyk SARIF
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: snyk.sarif
```

### Trivy for SCA

Trivy is an all-in-one open-source security scanner that covers dependencies, container images, IaC, and more. For SCA, it scans file system lock files and produces results in multiple formats.

```yaml
# GitHub Actions — Trivy filesystem scan (SCA)
name: Trivy SCA

on:
  pull_request:
    branches: [main]

jobs:
  trivy-sca:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy filesystem scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: fs
          scan-ref: .
          format: table
          severity: CRITICAL,HIGH
          exit-code: 1

      - name: Run Trivy (SARIF output)
        if: always()
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: fs
          scan-ref: .
          format: sarif
          output: trivy-sca.sarif

      - name: Upload Trivy SARIF
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-sca.sarif
```

| SCA Tool | License | Languages | Fix Suggestions | CI Integration | Vulnerability DB |
|---|---|---|---|---|---|
| **Dependabot** | Free (GitHub) | 15+ ecosystems | Auto-PRs | GitHub-native | GitHub Advisory |
| **Snyk** | Free tier / Paid | 10+ ecosystems | Yes (with PR) | GitHub, GitLab, etc. | Snyk Intel DB |
| **Trivy** | Apache 2.0 | 8+ ecosystems | No | Any CI | NVD, GitHub, etc. |
| **OWASP Dep-Check** | Apache 2.0 | Java, .NET, Node | No | Any CI | NVD |
| **Renovate** | AGPL-3.0 | 50+ ecosystems | Auto-PRs | Any Git platform | Multiple DBs |

---

## Container Image Scanning

### Trivy

Trivy scans container images for OS package vulnerabilities, language-specific dependency vulnerabilities, misconfigurations, and secrets — all from a single command.

```yaml
# GitHub Actions — Trivy container image scan
name: Container Security

on:
  push:
    branches: [main]

jobs:
  build-and-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Trivy image scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: table
          severity: CRITICAL,HIGH
          exit-code: 1
          ignore-unfixed: true

      - name: Trivy image scan (SARIF)
        if: always()
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: sarif
          output: trivy-image.sarif

      - name: Upload Trivy SARIF
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-image.sarif
```

### Grype

Grype is Anchore's open-source vulnerability scanner. It pairs with Syft (an SBOM generator) and provides fast, accurate image scanning.

```yaml
# GitHub Actions — Grype container image scan
name: Grype Scan

on:
  push:
    branches: [main]

jobs:
  grype:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Scan with Grype
        uses: anchore/scan-action@v6
        id: scan
        with:
          image: myapp:${{ github.sha }}
          fail-build: true
          severity-cutoff: high

      - name: Upload SARIF
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: ${{ steps.scan.outputs.sarif }}
```

### Docker Scout

Docker Scout is Docker's native vulnerability analysis tool, integrated into Docker Desktop and Docker Hub. It provides policy evaluation and real-time CVE monitoring.

```yaml
# GitHub Actions — Docker Scout scan
name: Docker Scout

on:
  push:
    branches: [main]

jobs:
  scout:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: myorg/myapp:${{ github.sha }}

      - name: Docker Scout CVEs
        uses: docker/scout-action@v1
        with:
          command: cves
          image: myorg/myapp:${{ github.sha }}
          only-severities: critical,high
          exit-code: true

      - name: Docker Scout Policy
        uses: docker/scout-action@v1
        with:
          command: policy
          image: myorg/myapp:${{ github.sha }}
```

### Scanning in CI

| Scanner | Speed | DB Source | Formats | SBOM Support | License |
|---|---|---|---|---|---|
| **Trivy** | Fast | NVD, GitHub, etc. | Table, JSON, SARIF | Yes (CycloneDX, SPDX) | Apache 2.0 |
| **Grype** | Fast | Grype DB (NVD-based) | Table, JSON, SARIF | Yes (via Syft) | Apache 2.0 |
| **Docker Scout** | Medium | Docker Scout DB | CLI, SARIF | Yes | Proprietary |
| **Snyk Container** | Medium | Snyk Intel DB | JSON, SARIF | Yes | Free tier / Paid |
| **Anchore Enterprise** | Slow | Anchore feeds | JSON, SARIF | Yes (Syft) | Commercial |

---

## Infrastructure as Code Scanning

### Checkov

Checkov is a static analysis tool for Infrastructure as Code. It scans Terraform, CloudFormation, Kubernetes, Helm, ARM templates, and Dockerfiles against hundreds of built-in policies.

```yaml
# GitHub Actions — Checkov IaC scan
name: IaC Security

on:
  pull_request:
    paths:
      - "terraform/**"
      - "kubernetes/**"

jobs:
  checkov:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Checkov
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: ./terraform
          framework: terraform
          output_format: cli,sarif
          output_file_path: console,checkov.sarif
          soft_fail: false
          skip_check: CKV_AWS_18,CKV_AWS_21

      - name: Upload SARIF
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: checkov.sarif
```

### tfsec

tfsec (now part of Trivy) is a Terraform-specific static analysis tool. It understands Terraform's HCL syntax and can resolve module references and variable values.

```yaml
# GitHub Actions — tfsec scan
name: tfsec

on:
  pull_request:
    paths:
      - "terraform/**"

jobs:
  tfsec:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.3
        with:
          working_directory: ./terraform
          soft_fail: false
          additional_args: --format sarif --out tfsec.sarif

      - name: Upload SARIF
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: tfsec.sarif
```

### KICS

KICS (Keeping Infrastructure as Code Secure) by Checkmarx scans Terraform, CloudFormation, Ansible, Docker, Kubernetes, and more. It has one of the broadest IaC platform coverage sets.

```yaml
# GitHub Actions — KICS scan
name: KICS

on:
  pull_request:
    paths:
      - "infrastructure/**"

jobs:
  kics:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run KICS
        uses: checkmarx/kics-github-action@v2.1.3
        with:
          path: ./infrastructure
          output_path: kics-results/
          output_formats: json,sarif
          fail_on: high,critical
          enable_comments: true

      - name: Upload SARIF
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: kics-results/results.sarif
```

### Policy Enforcement

| IaC Scanner | Frameworks | Custom Policies | Speed | License |
|---|---|---|---|---|
| **Checkov** | Terraform, K8s, CloudFormation, ARM, Helm | Python-based | Fast | Apache 2.0 |
| **tfsec** | Terraform (now merged into Trivy) | Rego, JSON | Fast | MIT |
| **KICS** | Terraform, K8s, CloudFormation, Ansible, Docker | Rego | Medium | Apache 2.0 |
| **Terrascan** | Terraform, K8s, CloudFormation, Helm | Rego (OPA) | Medium | Apache 2.0 |
| **Snyk IaC** | Terraform, K8s, CloudFormation, ARM | Snyk rules | Medium | Free tier / Paid |

---

## Secret Detection

### GitLeaks

GitLeaks scans Git repositories for hardcoded secrets — API keys, passwords, tokens, and private keys — across the entire commit history or just new commits.

```yaml
# GitHub Actions — GitLeaks secret detection
name: Secret Detection

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  gitleaks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run GitLeaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
```

```toml
# .gitleaks.toml — custom GitLeaks configuration
title = "Custom GitLeaks Config"

[extend]
useDefault = true

[[rules]]
id = "custom-api-key"
description = "Custom API key pattern"
regex = '''(?i)mycompany[_-]?api[_-]?key\s*[:=]\s*['"]?([a-zA-Z0-9]{32,})['"]?'''
secretGroup = 1

[allowlist]
paths = [
  '''test/fixtures/.*''',
  '''docs/examples/.*''',
]
regexes = [
  '''EXAMPLE_KEY''',
  '''PLACEHOLDER''',
]
```

### TruffleHog

TruffleHog detects secrets using both regular expressions and entropy analysis. It can also verify detected secrets against live APIs to determine if they are still active.

```yaml
# GitHub Actions — TruffleHog secret detection
name: TruffleHog

on:
  pull_request:
    branches: [main]

jobs:
  trufflehog:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: TruffleHog scan
        uses: trufflesecurity/trufflehog@main
        with:
          extra_args: --only-verified --results=verified
```

### Pre-Commit Hooks

Pre-commit hooks catch secrets before they enter the repository. This is the earliest possible detection point — before the commit is even created.

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.2
    hooks:
      - id: gitleaks

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: detect-private-key
      - id: check-added-large-files
        args: ["--maxkb=500"]
```

```
Secret Detection Layers
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Defense in Depth for Secrets:

  Layer 1: IDE Plugin          ← Developer sees warning in editor
       │
       ▼
  Layer 2: Pre-Commit Hook     ← Blocks commit locally
       │
       ▼
  Layer 3: CI Pipeline Scan    ← Blocks PR merge
       │
       ▼
  Layer 4: GitHub Secret Scan  ← Alerts on push to remote
       │
       ▼
  Layer 5: Runtime Detection   ← Monitors deployed secrets

  Each layer catches what previous layers missed.
  No single layer is sufficient on its own.
```

### GitHub Secret Scanning

GitHub's built-in secret scanning automatically detects secrets pushed to repositories. Partner programs enable automatic revocation of leaked tokens for supported providers (AWS, Azure, GCP, Slack, Stripe, and many more).

```yaml
# .github/secret_scanning.yml — custom patterns
paths-ignore:
  - "test/**"
  - "docs/examples/**"
```

```
GitHub Secret Scanning Flow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Developer pushes code
       │
       ▼
  ┌──────────────────────┐
  │  GitHub scans push   │
  │  against 200+ token  │
  │  patterns            │
  └──────────┬───────────┘
             │
      ┌──────┴──────┐
      │  Match found │
      └──────┬──────┘
             │
    ┌────────┴────────┐
    ▼                 ▼
  Partner Token    Non-partner Token
    │                 │
    ▼                 ▼
  Notify partner   Alert repo admin
  (auto-revoke)    (manual action)
```

---

## Software Bill of Materials (SBOM)

### CycloneDX

CycloneDX is an OWASP standard for SBOMs. It supports multiple formats (JSON, XML) and captures components, dependencies, vulnerabilities, and licensing information.

```yaml
# GitHub Actions — generate CycloneDX SBOM
name: SBOM Generation

on:
  push:
    branches: [main]

jobs:
  sbom:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Generate SBOM with Syft
        uses: anchore/sbom-action@v0
        with:
          image: myapp:${{ github.sha }}
          format: cyclonedx-json
          output-file: sbom.cyclonedx.json
          upload-artifact: true

      - name: Attest SBOM
        uses: actions/attest-sbom@v2
        with:
          subject-path: sbom.cyclonedx.json
          sbom-format: cyclonedx+json
```

### SPDX

SPDX (Software Package Data Exchange) is a Linux Foundation standard for communicating software bill of materials information. It is widely used for license compliance and supply chain transparency.

```bash
# Generate SPDX SBOM with Syft
syft packages dir:. -o spdx-json=sbom.spdx.json

# Generate SPDX SBOM with Trivy
trivy fs . --format spdx-json --output sbom.spdx.json

# Validate SPDX document
pyspdxtools -i sbom.spdx.json
```

### SBOM Generation and Attestation

```
SBOM Lifecycle
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌──────────┐    ┌────────────┐    ┌────────────┐
  │  Build   │───▶│  Generate  │───▶│   Sign     │
  │ Artifact │    │   SBOM     │    │  (cosign)  │
  └──────────┘    └────────────┘    └─────┬──────┘
                                          │
                  ┌────────────┐    ┌─────▼──────┐
                  │  Consume   │◀───│   Attach   │
                  │ (audit,    │    │ to artifact│
                  │  comply)   │    │ (registry) │
                  └────────────┘    └────────────┘

  Standards Comparison:

  ┌──────────────┬───────────────┬───────────────┐
  │              │  CycloneDX    │    SPDX       │
  ├──────────────┼───────────────┼───────────────┤
  │ Origin       │ OWASP         │ Linux Found.  │
  │ Focus        │ Security      │ Licensing     │
  │ Formats      │ JSON, XML     │ JSON, RDF,    │
  │              │               │ tag-value     │
  │ VEX Support  │ Native        │ External      │
  │ Adoption     │ Security-led  │ Compliance-led│
  │ ISO Standard │ No            │ ISO/IEC 5962  │
  └──────────────┴───────────────┴───────────────┘
```

```yaml
# GitHub Actions — full SBOM pipeline
name: SBOM Pipeline

on:
  push:
    branches: [main]

jobs:
  build-sbom-sign:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
    steps:
      - uses: actions/checkout@v4

      - name: Build and push image
        uses: docker/build-push-action@v6
        id: build
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          image: ghcr.io/${{ github.repository }}:${{ github.sha }}
          format: cyclonedx-json
          output-file: sbom.json

      - name: Attest SBOM
        uses: actions/attest-sbom@v2
        with:
          subject-name: ghcr.io/${{ github.repository }}
          subject-digest: ${{ steps.build.outputs.digest }}
          sbom-format: cyclonedx+json
          sbom-path: sbom.json
```

---

## Supply Chain Security

### SLSA Framework

SLSA (Supply-chain Levels for Software Artifacts) is a framework for ensuring the integrity of software artifacts throughout the supply chain. It defines four levels of increasing assurance.

```
SLSA Levels
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Level 0: No guarantees
  ┌────────────────────────────────────────────────────────┐
  │  No provenance. No verification. Most projects today.  │
  └────────────────────────────────────────────────────────┘

  Level 1: Provenance exists
  ┌────────────────────────────────────────────────────────┐
  │  Build process generates provenance metadata.          │
  │  Documents how artifact was built.                     │
  └────────────────────────────────────────────────────────┘

  Level 2: Hosted build + signed provenance
  ┌────────────────────────────────────────────────────────┐
  │  Build runs on hosted CI (not developer laptop).       │
  │  Provenance is signed by the build service.            │
  └────────────────────────────────────────────────────────┘

  Level 3: Hardened build platform
  ┌────────────────────────────────────────────────────────┐
  │  Build platform prevents tampering.                    │
  │  Isolated, ephemeral build environments.               │
  │  Non-falsifiable provenance.                           │
  └────────────────────────────────────────────────────────┘

  Adoption path:
  Most teams ──▶ Level 1 (add provenance) ──▶ Level 2 (hosted CI)
  ──▶ Level 3 (hardened builds — GitHub Actions, Google Cloud Build)
```

```yaml
# GitHub Actions — SLSA Level 3 provenance with official generator
name: SLSA Provenance

on:
  push:
    tags:
      - "v*"

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.hash.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - name: Build artifact
        run: |
          go build -o myapp .
          sha256sum myapp > digest.txt
      - name: Output digest
        id: hash
        run: echo "digest=$(sha256sum myapp | cut -d ' ' -f1)" >> "$GITHUB_OUTPUT"
      - uses: actions/upload-artifact@v4
        with:
          name: myapp
          path: myapp

  provenance:
    needs: build
    permissions:
      actions: read
      id-token: write
      contents: write
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v2.1.0
    with:
      base64-subjects: ${{ needs.build.outputs.digest }}
```

### Sigstore and Cosign

Sigstore provides free, open-source code signing for software artifacts. Cosign is the tool for signing and verifying container images and other OCI artifacts using keyless signing with OIDC identity.

```yaml
# GitHub Actions — sign container image with cosign (keyless)
name: Sign Image

on:
  push:
    branches: [main]

jobs:
  build-sign:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write
    steps:
      - uses: actions/checkout@v4

      - name: Install cosign
        uses: sigstore/cosign-installer@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v6
        id: build
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Sign image with cosign
        run: cosign sign --yes ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}
        env:
          COSIGN_EXPERIMENTAL: "true"
```

### Artifact Signing

Artifact signing ensures that software artifacts have not been tampered with after being built. It binds an identity (the builder) to an artifact (the binary, image, or package).

```bash
# Sign a container image (keyless — uses OIDC)
cosign sign --yes ghcr.io/myorg/myapp@sha256:abc123...

# Verify a container image signature
cosign verify \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  --certificate-identity-regexp="https://github.com/myorg/myapp/.*" \
  ghcr.io/myorg/myapp@sha256:abc123...

# Sign a binary blob
cosign sign-blob --yes --output-signature=myapp.sig myapp

# Verify a binary blob
cosign verify-blob \
  --signature=myapp.sig \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  --certificate-identity-regexp="https://github.com/myorg/myapp/.*" \
  myapp
```

### Provenance

Provenance records document how, where, and by whom a software artifact was built. They are critical for supply chain integrity and are required by frameworks like SLSA.

```
Supply Chain Security Architecture
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Source           Build            Sign             Deploy
  ┌──────┐       ┌──────┐       ┌──────┐       ┌──────────┐
  │ Git  │──────▶│  CI  │──────▶│Cosign│──────▶│ Registry │
  │ Repo │       │Build │       │ Sign │       │  (GHCR)  │
  └──┬───┘       └──┬───┘       └──┬───┘       └────┬─────┘
     │              │              │                 │
     ▼              ▼              ▼                 ▼
  Commit         Provenance     Signature       Verification
  Signing        Attestation    + SBOM          before deploy
  (GPG/SSH)      (SLSA L3)     (Sigstore)      (cosign verify)

  Verification Chain:
  ┌─────────────────────────────────────────────────────┐
  │  1. Verify commit signature (who authored it?)      │
  │  2. Verify provenance (where was it built?)         │
  │  3. Verify image signature (has it been tampered?)  │
  │  4. Verify SBOM (what's inside?)                    │
  │  5. Verify policy (does it meet our standards?)     │
  └─────────────────────────────────────────────────────┘
```

---

## Compliance as Code

### Open Policy Agent

Open Policy Agent (OPA) is a general-purpose policy engine that uses the Rego language to define and enforce policies. In CI/CD pipelines, OPA evaluates artifacts against organizational policies before deployment.

```rego
# policy/pipeline.rego — OPA policy for CI/CD gates
package pipeline.security

default allow = false

# Allow deployment only if all security checks pass
allow if {
    no_critical_vulnerabilities
    no_leaked_secrets
    image_is_signed
    sbom_exists
}

no_critical_vulnerabilities if {
    input.vulnerability_scan.critical_count == 0
    input.vulnerability_scan.high_count < 5
}

no_leaked_secrets if {
    input.secret_scan.findings_count == 0
}

image_is_signed if {
    input.image.signature != ""
    input.image.signature_verified == true
}

sbom_exists if {
    input.sbom.format != ""
    input.sbom.components_count > 0
}

# Generate human-readable denial reasons
reasons contains msg if {
    not no_critical_vulnerabilities
    msg := sprintf("Found %d critical and %d high vulnerabilities",
        [input.vulnerability_scan.critical_count, input.vulnerability_scan.high_count])
}

reasons contains msg if {
    not no_leaked_secrets
    msg := sprintf("Found %d leaked secrets", [input.secret_scan.findings_count])
}

reasons contains msg if {
    not image_is_signed
    msg := "Container image is not signed or signature verification failed"
}
```

### Policy Gates in Pipelines

```yaml
# GitHub Actions — OPA policy gate
name: Policy Gate

on:
  workflow_call:
    inputs:
      scan_results:
        required: true
        type: string

jobs:
  policy-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install OPA
        run: |
          curl -L -o opa https://openpolicyagent.org/downloads/v1.4.2/opa_linux_amd64_static
          chmod 755 opa
          sudo mv opa /usr/local/bin/

      - name: Evaluate policy
        run: |
          echo '${{ inputs.scan_results }}' > input.json
          RESULT=$(opa eval \
            --data policy/ \
            --input input.json \
            --format raw \
            "data.pipeline.security.allow")

          if [ "$RESULT" != "true" ]; then
            echo "❌ Policy gate FAILED"
            opa eval \
              --data policy/ \
              --input input.json \
              --format pretty \
              "data.pipeline.security.reasons"
            exit 1
          fi

          echo "✅ Policy gate PASSED"
```

```
Compliance as Code Architecture
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌──────────────────────────────────────────────────────────┐
  │                   Policy Repository                       │
  │                                                          │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
  │  │ Security │  │ License  │  │ Deploy   │               │
  │  │ Policies │  │ Policies │  │ Policies │               │
  │  │ (.rego)  │  │ (.rego)  │  │ (.rego)  │               │
  │  └────┬─────┘  └────┬─────┘  └────┬─────┘               │
  │       └──────────────┼─────────────┘                     │
  │                      ▼                                   │
  │                 ┌─────────┐                              │
  │                 │   OPA   │                              │
  │                 │ Engine  │                              │
  │                 └────┬────┘                              │
  └──────────────────────┼───────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ CI Gate  │    │ Admission│    │ Runtime  │
  │ (pre-    │    │ Control  │    │ Audit    │
  │  deploy) │    │ (K8s)    │    │ (cron)   │
  └──────────┘    └──────────┘    └──────────┘
```

---

## Vulnerability Management Workflow

### Triage

Not every vulnerability finding warrants immediate action. A triage process evaluates findings based on exploitability, context, and business impact to prioritize remediation.

```
Vulnerability Triage Workflow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Finding detected
       │
       ▼
  ┌──────────────┐
  │ Auto-classify│──── CVSS Score + Exploitability
  │ by severity  │     + Asset criticality
  └──────┬───────┘
         │
    ┌────┴────┐
    ▼         ▼
  True      False
  Positive  Positive
    │         │
    │         ▼
    │    ┌──────────┐
    │    │ Suppress  │──── Document reason
    │    │ with note │     Track suppression
    │    └──────────┘
    ▼
  ┌──────────────┐
  │ Assign owner │──── Based on code ownership (CODEOWNERS)
  └──────┬───────┘
         │
    ┌────┴─────────┐
    ▼              ▼
  Fixable       No fix
  (patch)       available
    │              │
    ▼              ▼
  ┌─────────┐  ┌────────────┐
  │ Patch + │  │ Mitigate / │──── WAF rule, network policy,
  │ verify  │  │ accept risk│     compensating control
  └─────────┘  └────────────┘
```

### SLA and Exceptions

Remediation SLAs define the maximum time allowed to fix vulnerabilities based on their severity. Exceptions must be formally documented and approved.

| Severity | CVSS Range | Remediation SLA | Escalation |
|---|---|---|---|
| **Critical** | 9.0–10.0 | 24–72 hours | VP Engineering + CISO |
| **High** | 7.0–8.9 | 7 days | Engineering Manager |
| **Medium** | 4.0–6.9 | 30 days | Team Lead |
| **Low** | 0.1–3.9 | 90 days | Developer |
| **Informational** | 0.0 | Best effort | None |

```yaml
# vulnerability-policy.yaml — SLA and exception configuration
remediation:
  sla:
    critical:
      max_days: 3
      auto_escalate: true
      escalation_contacts: ["vp-eng@company.com", "ciso@company.com"]
    high:
      max_days: 7
      auto_escalate: true
      escalation_contacts: ["eng-manager@company.com"]
    medium:
      max_days: 30
      auto_escalate: false
    low:
      max_days: 90
      auto_escalate: false

  exceptions:
    require_approval: true
    approvers: ["security-team"]
    max_exception_days: 90
    require_justification: true
    require_compensating_control: true
```

### Reporting

Security dashboards aggregate findings across all scanning tools and provide visibility into vulnerability trends, SLA compliance, and risk posture.

```yaml
# GitHub Actions — aggregate security report
name: Security Report

on:
  schedule:
    - cron: "0 8 * * 1"

jobs:
  report:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Collect scan results
        run: |
          # Pull latest results from all security tools
          mkdir -p reports

          # Trivy summary
          trivy fs . --format json --output reports/trivy.json

          # GitLeaks summary
          gitleaks detect --source . --report-format json \
            --report-path reports/gitleaks.json || true

      - name: Generate summary report
        run: |
          python3 scripts/aggregate-security-report.py \
            --input reports/ \
            --output security-summary.md \
            --format markdown

      - name: Post to Slack
        if: always()
        uses: slackapi/slack-github-action@v2.0.0
        with:
          webhook: ${{ secrets.SLACK_SECURITY_WEBHOOK }}
          webhook-type: incoming-webhook
          payload: |
            {
              "text": "Weekly Security Report generated. See workflow run for details."
            }
```

---

## Security Gate Implementation

### Blocking vs Advisory

Security gates can operate in two modes: **blocking** (fail the pipeline) or **advisory** (warn but allow). The choice depends on the maturity of the security program and the tolerance for false positives.

```
Blocking vs Advisory Modes
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Blocking Mode (strict):
  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │  Scan    │────▶│ Critical │────▶│ ❌ FAIL  │
  │  Result  │     │ found?   │     │  Pipeline │
  └──────────┘     └─────┬────┘     └──────────┘
                         │ No
                         ▼
                   ┌──────────┐
                   │ ✅ PASS  │
                   │ Pipeline │
                   └──────────┘

  Advisory Mode (soft):
  ┌──────────┐     ┌──────────┐     ┌──────────────┐
  │  Scan    │────▶│ Critical │────▶│ ⚠️ WARNING   │
  │  Result  │     │ found?   │     │  Continue     │
  └──────────┘     └─────┬────┘     │  + PR comment │
                         │ No       └──────────────┘
                         ▼
                   ┌──────────┐
                   │ ✅ PASS  │
                   └──────────┘

  Recommended Rollout:
  Week 1-4:   Advisory mode (all tools) — build confidence
  Week 5-8:   Block on secrets + critical CVEs only
  Week 9-12:  Block on SAST critical + high
  Week 13+:   Full blocking with exception workflow
```

### Risk Scoring

Aggregate risk scoring combines findings from multiple tools into a single score that determines whether the pipeline passes or fails.

```yaml
# security-gate.yaml — risk scoring configuration
scoring:
  weights:
    sast:
      critical: 10
      high: 5
      medium: 2
      low: 1
    sca:
      critical: 10
      high: 5
      medium: 2
      low: 1
    container:
      critical: 10
      high: 5
      medium: 2
      low: 1
    secrets:
      any: 100
    iac:
      critical: 8
      high: 4
      medium: 2
      low: 1

  thresholds:
    block: 50
    warn: 20
    pass: 0

  # Example: 1 critical SAST (10) + 3 high SCA (15) = 25 → warn
  # Example: 1 secret (100) → block immediately
```

### Break-the-Build Thresholds

```yaml
# GitHub Actions — configurable security gate with risk scoring
name: Security Gate

on:
  workflow_call:
    inputs:
      sast-sarif:
        required: true
        type: string
      sca-json:
        required: true
        type: string
      container-json:
        required: true
        type: string
      secrets-json:
        required: true
        type: string

jobs:
  gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Calculate risk score
        id: score
        run: |
          # Parse findings from all tools
          SAST_CRITICAL=$(jq '[.runs[].results[] | select(.level=="error")] | length' "${{ inputs.sast-sarif }}")
          SAST_HIGH=$(jq '[.runs[].results[] | select(.level=="warning")] | length' "${{ inputs.sast-sarif }}")
          SCA_CRITICAL=$(jq '[.Results[].Vulnerabilities[] | select(.Severity=="CRITICAL")] | length' "${{ inputs.sca-json }}")
          SECRET_COUNT=$(jq '.findings | length' "${{ inputs.secrets-json }}")

          # Calculate weighted score
          SCORE=$(( SAST_CRITICAL * 10 + SAST_HIGH * 5 + SCA_CRITICAL * 10 + SECRET_COUNT * 100 ))

          echo "risk_score=$SCORE" >> "$GITHUB_OUTPUT"
          echo "## Security Gate Results" >> "$GITHUB_STEP_SUMMARY"
          echo "| Category | Critical | High | Score |" >> "$GITHUB_STEP_SUMMARY"
          echo "|---|---|---|---|" >> "$GITHUB_STEP_SUMMARY"
          echo "| SAST | $SAST_CRITICAL | $SAST_HIGH | $(( SAST_CRITICAL * 10 + SAST_HIGH * 5 )) |" >> "$GITHUB_STEP_SUMMARY"
          echo "| SCA | $SCA_CRITICAL | - | $(( SCA_CRITICAL * 10 )) |" >> "$GITHUB_STEP_SUMMARY"
          echo "| Secrets | $SECRET_COUNT | - | $(( SECRET_COUNT * 100 )) |" >> "$GITHUB_STEP_SUMMARY"
          echo "| **Total** | | | **$SCORE** |" >> "$GITHUB_STEP_SUMMARY"

      - name: Enforce threshold
        run: |
          SCORE=${{ steps.score.outputs.risk_score }}
          BLOCK_THRESHOLD=50
          WARN_THRESHOLD=20

          if [ "$SCORE" -ge "$BLOCK_THRESHOLD" ]; then
            echo "❌ Risk score $SCORE exceeds block threshold ($BLOCK_THRESHOLD)"
            exit 1
          elif [ "$SCORE" -ge "$WARN_THRESHOLD" ]; then
            echo "⚠️ Risk score $SCORE exceeds warning threshold ($WARN_THRESHOLD)"
            echo "Pipeline continues but findings should be addressed"
          else
            echo "✅ Risk score $SCORE is within acceptable range"
          fi
```

| Gate Type | Blocks Pipeline | Use Case | Maturity Level |
|---|---|---|---|
| **Hard gate** | Yes — no exceptions | Leaked secrets, critical+exploitable CVEs | High maturity |
| **Soft gate** | No — warning only | New tool rollout, medium-severity findings | Early adoption |
| **Time-delayed gate** | After grace period | Known CVEs with no immediate fix | Medium maturity |
| **Exception gate** | Yes, unless exempted | Business-critical overrides with approval | High maturity |
| **Score-based gate** | Above threshold | Aggregate risk across all tools | High maturity |

---

## Next Steps

Continue your CI/CD learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | CI/CD Overview | Foundational CI/CD concepts, principles, and pipeline design |
| [01-CONTINUOUS-INTEGRATION.md](01-CONTINUOUS-INTEGRATION.md) | Continuous Integration | Build, test, and integration automation |
| [02-CONTINUOUS-DELIVERY.md](02-CONTINUOUS-DELIVERY.md) | Continuous Delivery | Deployment strategies, release management, and delivery pipelines |
| [03-GITHUB-ACTIONS.md](03-GITHUB-ACTIONS.md) | GitHub Actions | Workflow authoring, actions marketplace, and CI/CD with GitHub |
| [04-AZURE-DEVOPS.md](04-AZURE-DEVOPS.md) | Azure DevOps | Azure Pipelines, Boards, Repos, and Artifacts |
| [05-JENKINS.md](05-JENKINS.md) | Jenkins | Jenkins pipelines, distributed builds, and plugin ecosystem |
| [06-GITOPS.md](06-GITOPS.md) | GitOps | ArgoCD, Flux, and Git-based deployment workflows |
| [07-TESTING-IN-PIPELINES.md](07-TESTING-IN-PIPELINES.md) | Testing in Pipelines | Test strategies, parallelization, quality gates, and reporting |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Security in Pipelines documentation |
