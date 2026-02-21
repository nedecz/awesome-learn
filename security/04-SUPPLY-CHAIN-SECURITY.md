# Supply Chain Security

A comprehensive guide to software supply chain security — covering dependency scanning, software bill of materials (SBOM), SLSA framework, artifact signing, provenance, container image security, and supply chain attack prevention for modern software delivery.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Supply Chain Fundamentals](#supply-chain-fundamentals)
   - [What is Software Supply Chain Security](#what-is-software-supply-chain-security)
   - [Notable Supply Chain Attacks](#notable-supply-chain-attacks)
   - [The Software Supply Chain](#the-software-supply-chain)
3. [Dependency Management](#dependency-management)
   - [Dependency Scanning](#dependency-scanning)
   - [Software Composition Analysis (SCA)](#software-composition-analysis-sca)
   - [Lock Files and Pinning](#lock-files-and-pinning)
   - [Automated Dependency Updates](#automated-dependency-updates)
4. [Software Bill of Materials (SBOM)](#software-bill-of-materials-sbom)
   - [SBOM Formats](#sbom-formats)
   - [SBOM Generation](#sbom-generation)
   - [SBOM in CI/CD](#sbom-in-cicd)
5. [SLSA Framework](#slsa-framework)
   - [SLSA Levels](#slsa-levels)
   - [Provenance](#provenance)
   - [Achieving SLSA Compliance](#achieving-slsa-compliance)
6. [Artifact Signing](#artifact-signing)
   - [Sigstore and Cosign](#sigstore-and-cosign)
   - [Container Image Signing](#container-image-signing)
   - [Verification in Deployment](#verification-in-deployment)
7. [Container Image Security](#container-image-security)
   - [Base Image Selection](#base-image-selection)
   - [Image Scanning](#image-scanning)
   - [Minimal Images](#minimal-images)
   - [Image Provenance](#image-provenance)
8. [Registry Security](#registry-security)
   - [Private Registry Configuration](#private-registry-configuration)
   - [Access Controls](#access-controls)
   - [Content Trust](#content-trust)
9. [CI/CD Pipeline Security](#cicd-pipeline-security)
   - [Pipeline Integrity](#pipeline-integrity)
   - [Runner Security](#runner-security)
   - [Third-Party Actions and Plugins](#third-party-actions-and-plugins)
10. [Next Steps](#next-steps)
11. [Version History](#version-history)

---

## Overview

Software supply chain security protects the integrity and trustworthiness of software from source code through build, packaging, distribution, and deployment. A supply chain attack targets any point in this chain — compromised dependencies, tampered build artifacts, malicious CI/CD plugins, or poisoned container images can introduce vulnerabilities into your production systems without touching your application code.

This document covers practical supply chain security controls: dependency scanning, SBOM generation, the SLSA framework, artifact signing, container image security, and CI/CD pipeline hardening.

### Target Audience

- **Developers** managing dependencies and authoring build configurations
- **DevOps Engineers** building and securing CI/CD pipelines
- **Security Engineers** implementing supply chain security programs
- **Platform Engineers** providing secure-by-default build and deployment infrastructure

### Scope

- Software supply chain fundamentals and notable attacks
- Dependency management: scanning, SCA, lock files, automated updates
- SBOM: formats, generation, and integration into CI/CD
- SLSA framework: levels, provenance, and compliance
- Artifact signing: Sigstore, Cosign, container image signing
- Container image security: base images, scanning, minimal images
- Registry security: private registries, access controls, content trust
- CI/CD pipeline integrity and runner security

---

## Supply Chain Fundamentals

### What is Software Supply Chain Security

The **software supply chain** encompasses everything that goes into your software — from the source code you write to the third-party libraries you depend on, the tools that build and test your code, the registries that host your artifacts, and the infrastructure that deploys them.

```
Software Supply Chain
─────────────────────
Source Code → Dependencies → Build System → Artifacts → Registry → Deployment
     │            │              │             │           │           │
     ▼            ▼              ▼             ▼           ▼           ▼
  Compromised  Malicious     Tampered      Modified   Poisoned    Unauthorized
  code review  dependency   build process  artifact   registry     deployment
```

### Notable Supply Chain Attacks

| Attack | Year | Vector | Impact |
|--------|------|--------|--------|
| **SolarWinds (Sunburst)** | 2020 | Compromised build system injected backdoor into Orion updates | 18,000+ organizations including US government agencies |
| **Codecov** | 2021 | Modified bash uploader script exfiltrated CI/CD environment variables | Secrets from hundreds of private repositories exposed |
| **ua-parser-js** | 2021 | Compromised npm package maintainer account; malicious code in 3 versions | Cryptominer and password stealer distributed to millions |
| **Log4Shell** | 2021 | Critical vulnerability in widely-used Log4j library (CVE-2021-44228) | Remote code execution in millions of applications |
| **xz Utils** | 2024 | Multi-year social engineering to inject backdoor into compression library | Potential SSH server compromise in Linux distributions |
| **tj-actions/changed-files** | 2025 | Compromised GitHub Action that exfiltrated CI/CD secrets via logs | Secrets from thousands of repositories exposed |

### The Software Supply Chain

```
Supply Chain Trust Boundaries
─────────────────────────────
┌──────────────────────────────────────────────────────────┐
│  Your Code (highest trust)                                │
│  - Code you write and review                             │
│  - Internal libraries your team maintains                │
└───────────────────────┬──────────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────────┐
│  Direct Dependencies (medium trust)                       │
│  - Libraries you explicitly choose and evaluate          │
│  - Frameworks you depend on (React, Spring, .NET)        │
└───────────────────────┬──────────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────────┐
│  Transitive Dependencies (lower trust)                    │
│  - Dependencies of your dependencies                     │
│  - You often don't know they exist until a CVE appears   │
└───────────────────────┬──────────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────────┐
│  Build Tools & Infrastructure (trust required)            │
│  - CI/CD platforms, runners, build tools                 │
│  - Container base images, package registries             │
└──────────────────────────────────────────────────────────┘
```

---

## Dependency Management

### Dependency Scanning

Continuously scan your dependencies for known vulnerabilities:

| Tool | Ecosystems | Integration | Notes |
|------|-----------|-------------|-------|
| **Dependabot** | npm, pip, Maven, NuGet, Go, Rust, etc. | GitHub native | Automated PRs for updates |
| **Snyk** | Most ecosystems | CI/CD, IDE, CLI | Fix PRs, license compliance |
| **Trivy** | OS packages, language deps, IaC | CLI, CI/CD | Open source, fast |
| **Grype** | OS packages, language deps | CLI, CI/CD | Open source, pairs with Syft for SBOM |
| **npm audit** | npm | CLI | Built into npm |
| **pip-audit** | Python | CLI | Audits pip dependencies |

### Software Composition Analysis (SCA)

SCA tools analyze your codebase to identify all open-source components, their versions, known vulnerabilities, and license obligations.

```
SCA Workflow in CI/CD
─────────────────────
1. Developer commits code with dependencies
2. CI pipeline runs SCA scan
3. SCA tool inventories all direct and transitive dependencies
4. Tool checks each dependency against vulnerability databases (NVD, GitHub Advisory, OSV)
5. Results are reported: critical/high/medium/low findings
6. Pipeline fails if critical or high vulnerabilities are found (configurable threshold)
7. Developer remediates by updating, patching, or replacing the vulnerable dependency
```

### Lock Files and Pinning

Lock files ensure reproducible builds by recording the exact versions of all dependencies (including transitive):

| Ecosystem | Lock File | Command |
|-----------|-----------|---------|
| npm | `package-lock.json` | `npm ci` (install from lock file) |
| Yarn | `yarn.lock` | `yarn install --frozen-lockfile` |
| Python (pip) | `requirements.txt` (pinned) | `pip install -r requirements.txt` |
| Python (Poetry) | `poetry.lock` | `poetry install` |
| Go | `go.sum` | `go mod verify` |
| .NET | `packages.lock.json` | `dotnet restore --locked-mode` |
| Rust | `Cargo.lock` | `cargo install --locked` |

**Best practices:**

- Always commit lock files to version control
- Use `--frozen-lockfile` or `--locked` in CI to prevent silent dependency changes
- Review lock file changes in pull requests — they represent real changes to your software

### Automated Dependency Updates

Automate dependency updates to reduce the window of exposure to known vulnerabilities:

- **Dependabot** — Automatically opens PRs for outdated or vulnerable dependencies
- **Renovate** — Highly configurable dependency update bot with grouping and scheduling
- Configure update frequency based on risk: daily for security updates, weekly for minor updates
- Require CI checks to pass before merging dependency update PRs
- Group related updates (e.g., all AWS SDK packages) into a single PR

---

## Software Bill of Materials (SBOM)

### SBOM Formats

| Format | Maintainer | Format | Use Case |
|--------|-----------|--------|----------|
| **CycloneDX** | OWASP | JSON, XML | Application security, vulnerability management |
| **SPDX** | Linux Foundation | JSON, XML, RDF, tag-value | License compliance, open-source governance |
| **SWID** | ISO/IEC 19770-2 | XML | Software asset management |

### SBOM Generation

Generate SBOMs as part of your CI/CD pipeline:

| Tool | Output Formats | Ecosystems |
|------|---------------|------------|
| **Syft** (Anchore) | CycloneDX, SPDX | Containers, file systems, language deps |
| **Trivy** | CycloneDX, SPDX | Containers, file systems, Git repos |
| **cdxgen** | CycloneDX | Java, Node.js, Python, Go, .NET, Rust |
| **Microsoft SBOM Tool** | SPDX | .NET, npm, Python, Go |

### SBOM in CI/CD

```
SBOM Pipeline Integration
─────────────────────────
1. Build the application and container image
2. Generate SBOM from the container image or build artifacts
3. Sign the SBOM with Cosign or a similar tool
4. Attach the SBOM to the container image as an attestation
5. Store the SBOM in an artifact registry alongside the image
6. Use the SBOM for vulnerability scanning and compliance reporting
7. Require SBOM presence as a deployment gate
```

---

## SLSA Framework

### SLSA Levels

**SLSA (Supply chain Levels for Software Artifacts)** is a framework for ensuring the integrity of software artifacts throughout the supply chain.

| Level | Requirements | Protection Against |
|-------|-------------|-------------------|
| **SLSA 1** | Build process documented; provenance generated | Mistakes, incomplete records |
| **SLSA 2** | Provenance generated by a hosted build service; authenticated | Tampering after build |
| **SLSA 3** | Hardened build platform; non-falsifiable provenance; isolated builds | Compromised build environment |

### Provenance

**Provenance** is a record of how an artifact was built — the source code, build instructions, build environment, and inputs used. SLSA provenance proves:

- **What** was built (artifact digest)
- **From what source** (Git commit, repository)
- **By what build system** (GitHub Actions, Cloud Build)
- **With what configuration** (workflow file, build script)

### Achieving SLSA Compliance

```
SLSA Compliance Checklist
─────────────────────────
SLSA 1:
  ✓ Build process is scripted (not manual)
  ✓ Provenance is generated (even if self-attested)
  ✓ Provenance includes source and build references

SLSA 2:
  ✓ Build runs on a hosted service (GitHub Actions, Cloud Build)
  ✓ Provenance is authenticated (signed by the build service)
  ✓ Provenance is generated by the build service, not the user

SLSA 3:
  ✓ Build runs in an ephemeral, isolated environment
  ✓ Build configuration is defined in source control
  ✓ Provenance cannot be falsified by build service admins
```

---

## Artifact Signing

### Sigstore and Cosign

**Sigstore** is an open-source project that provides free, easy-to-use tools for signing, verifying, and protecting software artifacts. **Cosign** is the tool for signing container images.

```
Cosign Signing Flow (Keyless)
─────────────────────────────
1. Developer or CI authenticates with an OIDC provider (GitHub, Google)
2. Cosign requests a short-lived signing certificate from Fulcio (Sigstore CA)
3. Cosign signs the artifact (container image digest) with the ephemeral key
4. Signature and certificate are stored in Rekor (transparency log)
5. Ephemeral key is discarded — no long-lived keys to manage
```

### Container Image Signing

Sign container images in your CI/CD pipeline:

```bash
# Sign an image (keyless mode — uses OIDC identity)
cosign sign ghcr.io/myorg/myapp@sha256:abc123...

# Verify an image
cosign verify --certificate-identity=ci@myorg.com \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  ghcr.io/myorg/myapp@sha256:abc123...
```

### Verification in Deployment

Enforce signature verification at deployment time:

- **Kubernetes:** Use admission controllers (Kyverno, Sigstore Policy Controller) to reject unsigned images
- **Cloud platforms:** Enable container image signing policies (GKE Binary Authorization, AWS Signer)
- **Registries:** Configure registries to flag or reject unsigned artifacts

---

## Container Image Security

### Base Image Selection

Choose base images carefully — they form the foundation of your container's security posture:

| Image | Size | Packages | Security Surface |
|-------|------|----------|-----------------|
| **Distroless** | Minimal | Runtime only (no shell, no package manager) | Smallest attack surface |
| **Alpine** | ~5 MB | musl libc, apk | Small, but musl can cause compatibility issues |
| **Debian slim** | ~80 MB | glibc, apt (minimal packages) | Good compatibility, moderate surface |
| **Ubuntu** | ~80 MB | glibc, apt | Broad compatibility, larger surface |
| **Scratch** | 0 MB | Nothing | Statically compiled binaries only |

### Image Scanning

Scan container images for vulnerabilities in CI/CD and in registries:

| Tool | Integration | Features |
|------|-------------|----------|
| **Trivy** | CLI, CI/CD, registries | OS + language deps, IaC, secrets, SBOM |
| **Grype** | CLI, CI/CD | Fast scanning, pairs with Syft |
| **Docker Scout** | Docker CLI, Docker Hub | Real-time CVE monitoring, recommendations |
| **Snyk Container** | CLI, CI/CD, registries | Fix advice, base image recommendations |

### Minimal Images

Reduce the attack surface by minimizing what is in your container images:

- Use multi-stage builds — build in a full image, run in a minimal image
- Remove build tools, compilers, and package managers from the final image
- Do not install debugging tools in production images
- Use distroless or scratch images for production

### Image Provenance

Track the provenance of every image deployed to production:

- Tag images with the Git commit SHA, not `latest`
- Sign images with Cosign and attach provenance attestations
- Store SBOM alongside the image
- Use image digests (not tags) in deployment manifests for immutability

---

## Registry Security

### Private Registry Configuration

- Use a private container registry (AWS ECR, Azure ACR, Google GAR, GitHub GHCR)
- Enable vulnerability scanning on push
- Configure retention policies to remove old, unpatched images
- Enable audit logging for all registry operations

### Access Controls

- Use IAM roles/policies to control who can push and pull images
- Implement least privilege: CI/CD pipelines push; deployment systems pull
- Use repository-level permissions (not registry-wide)
- Require MFA for human access to registry management consoles

### Content Trust

- Enable Docker Content Trust (DCT) or Notary to ensure only signed images are used
- Configure registries to reject unsigned pushes
- Use admission controllers in Kubernetes to enforce image signing policies

---

## CI/CD Pipeline Security

### Pipeline Integrity

- Store pipeline definitions in version control (pipeline as code)
- Require code review for pipeline changes
- Use reusable, centrally-managed workflow templates for security-critical steps
- Pin action versions to commit SHAs, not mutable tags

### Runner Security

- Use ephemeral runners (destroyed after each job) to prevent cross-job contamination
- Isolate runners from production networks
- Minimize the credentials available to runners (least privilege)
- Monitor runner activity for anomalous behavior

### Third-Party Actions and Plugins

- Pin third-party actions to a specific commit SHA (not a tag)
- Review the source code of third-party actions before adopting them
- Prefer actions from verified publishers or official repositories
- Monitor for compromised actions (subscribe to security advisories)

```
# Bad — mutable tag, can be changed by the action owner
- uses: some-action/action@v1

# Good — pinned to a specific, immutable commit SHA
- uses: some-action/action@a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2
```

---

## Next Steps

Continue your security learning journey:

| File | Topic | Description |
|---|---|---|
| [05-INFRASTRUCTURE-SECURITY.md](05-INFRASTRUCTURE-SECURITY.md) | Infrastructure Security | Network policies, secrets management, zero trust |
| [03-CRYPTOGRAPHY.md](03-CRYPTOGRAPHY.md) | Cryptography | Encryption, key management, signing foundations |
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Best Practices | Security checklists and automation |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Supply Chain Security documentation |
