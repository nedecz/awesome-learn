# Security and Signing

## Table of Contents

1. [Overview](#overview)
2. [GPG Commit Signing](#gpg-commit-signing)
   - [Generating a GPG Key](#generating-a-gpg-key)
   - [Configuring Git for GPG Signing](#configuring-git-for-gpg-signing)
   - [Signing Commits](#signing-commits)
   - [Verifying Signatures](#verifying-signatures)
   - [Exporting and Sharing Your Public Key](#exporting-and-sharing-your-public-key)
3. [SSH Commit Signing](#ssh-commit-signing)
   - [Requirements](#requirements)
   - [Configuring Git for SSH Signing](#configuring-git-for-ssh-signing)
   - [Allowed Signers File](#allowed-signers-file)
   - [Signing and Verifying with SSH](#signing-and-verifying-with-ssh)
4. [Tag Signing](#tag-signing)
   - [Creating Signed Tags](#creating-signed-tags)
   - [Verifying Signed Tags](#verifying-signed-tags)
   - [Signing Releases](#signing-releases)
5. [Vigilant Mode](#vigilant-mode)
   - [How Vigilant Mode Works](#how-vigilant-mode-works)
   - [Unsigned Commit Warnings](#unsigned-commit-warnings)
   - [Enabling Vigilant Mode](#enabling-vigilant-mode)
6. [Secrets Management](#secrets-management)
   - [Preventing Secrets in Repositories](#preventing-secrets-in-repositories)
   - [git-secrets](#git-secrets)
   - [gitleaks](#gitleaks)
   - [truffleHog](#trufflehog)
   - [Tool Comparison](#tool-comparison)
7. [Credential Management](#credential-management)
   - [Credential Helpers](#credential-helpers)
   - [SSH Keys](#ssh-keys)
   - [Personal Access Tokens](#personal-access-tokens)
   - [Credential Caching](#credential-caching)
8. [Repository Security](#repository-security)
   - [Branch Protection Rules](#branch-protection-rules)
   - [Required Reviews](#required-reviews)
   - [CODEOWNERS](#codeowners)
   - [Security Policies (SECURITY.md)](#security-policies-securitymd)
9. [Git Security Vulnerabilities](#git-security-vulnerabilities)
   - [Historical CVEs](#historical-cves)
   - [Keeping Git Updated](#keeping-git-updated)
   - [Safe Directory Settings](#safe-directory-settings)
10. [.gitignore Best Practices](#gitignore-best-practices)
    - [Common Patterns](#common-patterns)
    - [Global Gitignore](#global-gitignore)
    - [Preventing Sensitive File Commits](#preventing-sensitive-file-commits)
11. [Audit and Compliance](#audit-and-compliance)
    - [Git Log for Auditing](#git-log-for-auditing)
    - [Signed Commits for Compliance](#signed-commits-for-compliance)
    - [SLSA Framework](#slsa-framework)
12. [Best Practices Summary](#best-practices-summary)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

## Overview

Git security covers two complementary concerns: **integrity** (proving who authored each change and that it has not been tampered with) and **confidentiality** (keeping secrets, credentials, and sensitive data out of version history). Commit signing, credential management, secret scanning, and repository access controls work together to protect your codebase across its entire lifecycle.

### Target Audience

- Developers configuring commit signing for the first time
- Security engineers establishing Git security policies for an organization
- DevOps and platform engineers integrating secret scanning and credential management into CI/CD

### Scope

- GPG and SSH commit signing setup and verification
- Tag signing for release integrity
- GitHub vigilant mode and signature verification
- Secret detection and prevention tools (git-secrets, gitleaks, truffleHog)
- Credential helpers, SSH keys, and token management
- Branch protection, CODEOWNERS, and security policies
- Git CVE history and safe directory configuration
- .gitignore patterns for sensitive files
- Audit logging and SLSA supply chain compliance

## GPG Commit Signing

GPG (GNU Privacy Guard) signing attaches a cryptographic signature to each commit, proving that the commit was created by the holder of a specific private key and that the content has not been modified since signing.

```
Unsigned commit                    Signed commit

┌─────────────────────────┐       ┌─────────────────────────┐
│  commit abc1234          │       │  commit abc1234          │
│  Author: Alice           │       │  Author: Alice           │
│  Date: 2025-01-15        │       │  Date: 2025-01-15        │
│                           │       │  GPG Signature:          │
│  (no proof of identity)  │       │    Key ID: 3AA5C34371... │
│                           │       │    Status: ✅ Verified    │
└─────────────────────────┘       └─────────────────────────┘

Anyone can set Author to any       Signature cryptographically
name/email in git config           proves the committer's identity
```

### Generating a GPG Key

```bash
# Generate a new GPG key pair (use RSA 4096 or Ed25519)
gpg --full-generate-key

# Recommended selections:
#   Key type: (1) RSA and RSA   or   (9) ECC (sign and encrypt) with Curve 25519
#   Key size: 4096 (for RSA)
#   Expiration: 1y (set an expiration — you can extend it later)
#   Real name: Your Full Name
#   Email: your-email@example.com (must match your Git email)

# List your secret keys to find the key ID
gpg --list-secret-keys --keyid-format=long

# Output example:
# sec   rsa4096/3AA5C34371567BD2 2025-01-15 [SC] [expires: 2026-01-15]
#       ABCDEF1234567890ABCDEF1234567890ABCDEF12
# uid                 [ultimate] Alice <alice@example.com>
# ssb   rsa4096/1234567890ABCDEF 2025-01-15 [E] [expires: 2026-01-15]

# The key ID is the part after "rsa4096/" — in this case: 3AA5C34371567BD2
```

### Configuring Git for GPG Signing

```bash
# Tell Git which GPG key to use
git config --global user.signingkey 3AA5C34371567BD2

# Enable automatic signing for all commits
git config --global commit.gpgsign true

# Enable automatic signing for all tags
git config --global tag.gpgsign true

# If gpg2 is installed separately, point Git to it
git config --global gpg.program gpg2

# On macOS with GPG Suite, enable the agent for passphrase caching
# Add to ~/.gnupg/gpg-agent.conf:
#   default-cache-ttl 3600
#   max-cache-ttl 86400

# On macOS, you may also need to set the TTY for GPG passphrase prompts
export GPG_TTY=$(tty)
# Add this to your ~/.bashrc, ~/.zshrc, or shell profile
```

### Signing Commits

```bash
# Sign a single commit (if auto-signing is not enabled)
git commit -S -m "feat: add authentication module"

# Sign a commit with a specific key (overrides config)
git commit -S --gpg-sign=3AA5C34371567BD2 -m "fix: patch vulnerability"

# Amend the last commit and sign it
git commit --amend -S --no-edit

# Sign all commits during an interactive rebase
git rebase --exec 'git commit --amend --no-edit -S' -i HEAD~5
```

### Verifying Signatures

```bash
# Verify the signature on the most recent commit
git log --show-signature -1

# Show verification status for all commits
git log --pretty='format:%H %G? %GS %s' --abbrev-commit

# Signature status codes:
#   G = Good (valid signature)
#   B = Bad signature
#   U = Untrusted (valid signature, but key is not trusted)
#   X = Expired signature
#   Y = Expired key
#   R = Revoked key
#   E = Cannot check (missing key)
#   N = No signature

# Verify a specific commit
git verify-commit abc1234

# Verify a range of commits
git log --show-signature v1.0.0..HEAD

# Require signed commits during merge
git merge --verify-signatures feature-branch
```

### Exporting and Sharing Your Public Key

```bash
# Export your public key in ASCII armor format
gpg --armor --export 3AA5C34371567BD2

# Upload to GitHub: Settings > SSH and GPG keys > New GPG key
# Paste the output of the above command

# Upload to a public keyserver (optional)
gpg --keyserver hkps://keys.openpgp.org --send-keys 3AA5C34371567BD2

# Import a teammate's public key
gpg --import teammate-public-key.asc

# Trust a key after importing
gpg --edit-key teammate@example.com
# In the GPG prompt: trust > 4 (full trust) > quit
```

## SSH Commit Signing

Git 2.34+ supports using SSH keys for commit signing — the same keys you already use for authentication. This avoids the complexity of GPG key management while still providing cryptographic proof of authorship.

```
GPG Signing                          SSH Signing (Git 2.34+)

┌────────────────────────────┐      ┌────────────────────────────┐
│  Separate GPG keyring       │      │  Reuse existing SSH keys    │
│  GPG agent for passphrase   │      │  ssh-agent for passphrase   │
│  Key servers for sharing    │      │  GitHub keys already linked │
│  Complex trust model (WoT)  │      │  Simple allowed_signers     │
│  Widely supported           │      │  Requires Git 2.34+         │
└────────────────────────────┘      └────────────────────────────┘
```

### Requirements

- Git 2.34 or later
- OpenSSH 8.0 or later (for SSH signature support)
- An existing SSH key pair (Ed25519 recommended)

```bash
# Check your Git version
git --version

# Generate an SSH key if you do not have one
ssh-keygen -t ed25519 -C "your-email@example.com"
```

### Configuring Git for SSH Signing

```bash
# Set the signing format to SSH
git config --global gpg.format ssh

# Point Git to your SSH signing key
git config --global user.signingkey ~/.ssh/id_ed25519.pub

# Enable automatic signing for all commits
git config --global commit.gpgsign true

# Enable automatic signing for all tags
git config --global tag.gpgsign true
```

### Allowed Signers File

To verify SSH signatures, Git needs an `allowed_signers` file that maps email addresses to public keys — similar to `authorized_keys` for SSH authentication.

```bash
# Create an allowed_signers file
mkdir -p ~/.config/git

# Add entries (one per line): email namespaces="git" <public-key>
echo "alice@example.com namespaces=\"git\" $(cat ~/.ssh/id_ed25519.pub)" \
  >> ~/.config/git/allowed_signers

# Tell Git where to find the file
git config --global gpg.ssh.allowedSignersFile ~/.config/git/allowed_signers

# Example allowed_signers file:
# alice@example.com namespaces="git" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...
# bob@example.com namespaces="git" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...
```

### Signing and Verifying with SSH

```bash
# Sign a commit (same as GPG — Git uses the configured format)
git commit -S -m "feat: add SSH signing support"

# Verify a commit signature
git verify-commit HEAD

# Verify a tag signature
git verify-tag v1.0.0

# View signature details in the log
git log --show-signature -1
```

## Tag Signing

Signed tags provide tamper-proof release points. When you sign a tag, anyone with your public key can verify that the tag was created by you and that the tagged commit has not been altered.

### Creating Signed Tags

```bash
# Create a signed annotated tag (uses your configured signing key)
git tag -s v1.0.0 -m "Release v1.0.0"

# Create a signed tag with a specific GPG key
git tag -s v1.0.0 -u 3AA5C34371567BD2 -m "Release v1.0.0"

# Sign a tag using SSH (if gpg.format is set to ssh)
git tag -s v1.0.0 -m "Release v1.0.0"

# Push the signed tag to the remote
git push origin v1.0.0
```

### Verifying Signed Tags

```bash
# Verify a specific tag
git tag -v v1.0.0

# Verify a tag using git verify-tag
git verify-tag v1.0.0

# List tags with verification status
git tag -l --format='%(refname:short) %(if)%(objecttype)%(then)%(objecttype)%(end)' | \
  while read tag type; do
    echo -n "$tag: "
    git verify-tag "$tag" 2>&1 | head -1
  done

# Only allow merging from verified tags
git merge --verify-signatures v1.0.0
```

### Signing Releases

```
Release signing workflow:

1. Tag the release commit
   git tag -s v2.0.0 -m "Release v2.0.0: major feature update"

2. Push the tag
   git push origin v2.0.0

3. CI/CD verifies the tag signature before building
   git verify-tag v2.0.0

4. Build artifacts are produced only from verified tags
   ┌──────────────────────────────────────────────┐
   │  CI Pipeline                                   │
   │                                                │
   │  verify-tag ──▶ build ──▶ sign artifacts ──▶  │
   │                           (cosign/sigstore)    │
   │                                   │            │
   │                                   ▼            │
   │                            publish release     │
   └──────────────────────────────────────────────┘
```

## Vigilant Mode

GitHub's vigilant mode flags all unsigned commits with an "Unverified" badge, making it immediately visible when a commit lacks a cryptographic signature.

### How Vigilant Mode Works

```
Without Vigilant Mode              With Vigilant Mode

  abc1234  Add feature              abc1234  Add feature
           (no badge)                        ⚠️ Unverified

  def5678  Fix bug                  def5678  Fix bug
           Verified ✅                       Verified ✅

  ghi9012  Update docs             ghi9012  Update docs
           (no badge)                        ⚠️ Unverified

Unsigned commits are               ALL unsigned commits are
silently accepted                   flagged with a warning badge
```

### Unsigned Commit Warnings

| Badge | Meaning |
|---|---|
| **Verified** ✅ | Commit is signed with a key linked to the committer's GitHub account |
| **Partially verified** | Signature is valid but the key is not linked to the committer's GitHub account |
| **Unverified** ⚠️ | Commit is unsigned or the signature cannot be verified (shown only with vigilant mode) |

### Enabling Vigilant Mode

1. Go to **GitHub Settings** → **SSH and GPG keys**
2. Check **Flag unsigned commits as unverified**
3. Ensure all your signing keys are uploaded to your GitHub account

When vigilant mode is enabled, commits made through the GitHub web interface are automatically signed by GitHub's own key. Commits made locally must be signed with your own GPG or SSH key.

## Secrets Management

Secrets committed to Git history are extremely difficult to remove — even after deletion, they persist in the reflog and in any clone or fork. Prevention is far more effective than remediation.

### Preventing Secrets in Repositories

```
Defense-in-depth for secrets:

Layer 1: .gitignore
         Block known secret file patterns (.env, *.pem, *.key)

Layer 2: Pre-commit hooks
         Scan staged files before they enter history

Layer 3: CI/CD secret scanning
         Detect secrets in pull requests before merge

Layer 4: Continuous monitoring
         Scan repository history on a schedule

┌───────────────────────────────────────────────────────────┐
│  Developer workstation          CI/CD Pipeline             │
│                                                            │
│  .gitignore ──▶ pre-commit ──▶ PR scan ──▶ scheduled scan │
│  (block files)  (git-secrets)  (gitleaks)   (truffleHog)  │
│                                                            │
│  Each layer catches what the previous layer missed         │
└───────────────────────────────────────────────────────────┘
```

### git-secrets

AWS-maintained tool that prevents committing secrets by installing Git hooks.

```bash
# Install git-secrets
# macOS
brew install git-secrets

# From source
git clone https://github.com/awslabs/git-secrets.git
cd git-secrets && make install

# Initialize in a repository
cd your-repo
git secrets --install

# Register AWS secret patterns
git secrets --register-aws

# Add custom forbidden patterns
git secrets --add 'PRIVATE[_-]?KEY'
git secrets --add --allowed 'PRIVATE_KEY_EXAMPLE'

# Scan the entire repository history
git secrets --scan-history

# Scan staged changes (runs automatically via pre-commit hook)
git secrets --scan
```

### gitleaks

Fast, comprehensive secret scanner with support for custom rules and CI/CD integration.

```bash
# Install gitleaks
# macOS
brew install gitleaks

# Docker
docker run -v "$(pwd):/repo" zricethezav/gitleaks detect --source /repo

# Scan the current repository
gitleaks detect --source .

# Scan only staged changes (for pre-commit hooks)
gitleaks protect --staged

# Use a custom configuration
gitleaks detect --config .gitleaks.toml

# Generate a report
gitleaks detect --source . --report-format json --report-path gitleaks-report.json
```

Example `.gitleaks.toml` configuration:

```toml
[extend]
useDefault = true

[[rules]]
id = "custom-api-key"
description = "Custom API Key Pattern"
regex = '''(?i)my_service_api_key\s*[=:]\s*['"]?[\w-]{32,}['"]?'''
tags = ["custom", "api-key"]

[allowlist]
paths = [
  '''(.*?)test(.*?)''',
  '''(.*?)fixture(.*?)''',
]
```

### truffleHog

Deep history scanner that detects secrets across the entire Git history using both regex patterns and entropy analysis.

```bash
# Install truffleHog
pip install trufflehog

# Scan a local repository
trufflehog git file://. --only-verified

# Scan a remote repository
trufflehog git https://github.com/org/repo.git --only-verified

# Scan only recent changes (since a specific commit)
trufflehog git file://. --since-commit abc1234 --only-verified

# JSON output for CI integration
trufflehog git file://. --only-verified --json
```

### Tool Comparison

| Feature | git-secrets | gitleaks | truffleHog |
|---|---|---|---|
| **Pre-commit hook** | ✅ Built-in | ✅ Via `protect` | ❌ Separate setup |
| **Full history scan** | ✅ `--scan-history` | ✅ `detect` | ✅ Default behavior |
| **Entropy detection** | ❌ Regex only | ✅ Configurable | ✅ Built-in |
| **Custom rules** | ✅ Regex patterns | ✅ TOML config | ✅ Detectors |
| **CI/CD integration** | Basic | ✅ GitHub Action | ✅ GitHub Action |
| **Speed** | Fast | Very fast | Moderate |
| **Verified secrets** | ❌ | ❌ | ✅ Checks if secrets are live |

## Credential Management

Git needs credentials to authenticate with remote repositories. Choosing the right credential storage mechanism balances security with convenience.

### Credential Helpers

```bash
# List available credential helpers
git help -a | grep credential

# Store credentials in memory for a session (default: 15 minutes)
git config --global credential.helper cache

# Store credentials in memory for 8 hours
git config --global credential.helper 'cache --timeout=28800'

# Store credentials in the macOS Keychain
git config --global credential.helper osxkeychain

# Store credentials in Windows Credential Manager
git config --global credential.helper manager

# Store credentials in libsecret (Linux GNOME Keyring)
git config --global credential.helper /usr/lib/git-core/git-credential-libsecret

# ❌ Avoid: store credentials in plaintext on disk
# git config --global credential.helper store
# This saves credentials in ~/.git-credentials as plaintext
```

### SSH Keys

```bash
# Generate an Ed25519 SSH key (recommended)
ssh-keygen -t ed25519 -C "your-email@example.com"

# Start the SSH agent
eval "$(ssh-agent -s)"

# Add your key to the agent
ssh-add ~/.ssh/id_ed25519

# On macOS, add to Keychain for persistence
ssh-add --apple-use-keychain ~/.ssh/id_ed25519

# Configure SSH for GitHub in ~/.ssh/config
# Host github.com
#   HostName github.com
#   User git
#   IdentityFile ~/.ssh/id_ed25519
#   AddKeysToAgent yes

# Test your SSH connection
ssh -T git@github.com

# Switch an existing repo from HTTPS to SSH
git remote set-url origin git@github.com:org/repo.git
```

### Personal Access Tokens

```bash
# Use a personal access token (PAT) for HTTPS authentication
# Generate at: GitHub > Settings > Developer settings > Personal access tokens

# Scope recommendations:
#   repo           — full access to private repositories
#   read:org       — read organization membership
#   workflow       — update GitHub Actions workflows

# Use with credential helper (prompted on first push)
git push origin main
# Username: your-github-username
# Password: ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Or set directly in the remote URL (not recommended — stored in .git/config)
# git remote set-url origin https://ghp_TOKEN@github.com/org/repo.git

# Fine-grained tokens (recommended over classic PATs):
#   - Scoped to specific repositories
#   - Granular permissions per API endpoint
#   - Mandatory expiration
```

### Credential Caching

| Helper | Platform | Storage | Security Level |
|---|---|---|---|
| **osxkeychain** | macOS | Encrypted Keychain | High |
| **manager** | Windows | Windows Credential Manager | High |
| **libsecret** | Linux (GNOME) | GNOME Keyring (encrypted) | High |
| **cache** | All | Memory (temporary) | Medium |
| **store** | All | Plaintext file | ❌ Low |

## Repository Security

### Branch Protection Rules

Branch protection rules prevent unauthorized changes to critical branches.

```
Branch protection for main:

┌─────────────────────────────────────────────────────────┐
│  Rule                              │ Recommended Setting │
├─────────────────────────────────────────────────────────┤
│  Require pull request reviews      │ ✅ 2 approvals      │
│  Dismiss stale reviews on push     │ ✅ Enabled           │
│  Require status checks             │ ✅ CI must pass      │
│  Require signed commits            │ ✅ Enabled           │
│  Require linear history            │ ✅ Squash or rebase  │
│  Include administrators            │ ✅ No bypasses       │
│  Restrict who can push             │ ✅ Release team only │
│  Allow force pushes                │ ❌ Disabled          │
│  Allow deletions                   │ ❌ Disabled          │
└─────────────────────────────────────────────────────────┘
```

### Required Reviews

- ✅ Require at least two approving reviews before merge
- ✅ Dismiss stale reviews when new commits are pushed
- ✅ Require review from code owners for their owned paths
- ✅ Restrict dismissal of reviews to repository admins
- ❌ Do not allow self-approval of pull requests

### CODEOWNERS

The `CODEOWNERS` file automatically assigns reviewers based on file paths.

```
# .github/CODEOWNERS

# Default owners for everything in the repository
*                           @org/engineering-leads

# Security-sensitive files require security team review
SECURITY.md                 @org/security-team
.github/workflows/          @org/platform-team @org/security-team
*.pem                       @org/security-team
*.key                       @org/security-team

# Infrastructure code
terraform/                  @org/infrastructure-team
kubernetes/                 @org/platform-team

# Documentation
docs/                       @org/docs-team
```

### Security Policies (SECURITY.md)

Every repository should include a `SECURITY.md` file that describes how to report vulnerabilities.

```markdown
# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| 2.x     | ✅        |
| 1.x     | ❌        |

## Reporting a Vulnerability

If you discover a security vulnerability, please report it responsibly:

1. **Do not** open a public GitHub issue.
2. Email security@example.com with a detailed description.
3. Include steps to reproduce, impact assessment, and any suggested fixes.
4. We will acknowledge receipt within 48 hours.
5. We aim to release a fix within 7 days for critical vulnerabilities.
```

## Git Security Vulnerabilities

Git itself has had security vulnerabilities. Keeping Git updated and configuring safe defaults protects against known attack vectors.

### Historical CVEs

| CVE | Version Affected | Impact |
|---|---|---|
| **CVE-2022-24765** | < 2.35.2 | Arbitrary code execution via unsafe directory ownership on multi-user systems |
| **CVE-2022-39253** | < 2.38.1 | Local clone optimization could expose private data via symlinks |
| **CVE-2023-22490** | < 2.39.2 | Data exfiltration via crafted repositories using local clone |
| **CVE-2023-25652** | < 2.40.1 | `git apply --reject` could write outside the working tree |
| **CVE-2024-32002** | < 2.45.1 | Recursive clone with submodules could execute hooks on case-insensitive filesystems |

### Keeping Git Updated

```bash
# Check your current version
git --version

# Update on macOS (Homebrew)
brew upgrade git

# Update on Ubuntu/Debian (official PPA for latest stable)
sudo add-apt-repository ppa:git-core/ppa
sudo apt update && sudo apt install git

# Update on Windows
git update-git-for-windows

# Update on Fedora/RHEL
sudo dnf upgrade git
```

### Safe Directory Settings

Git 2.35.2+ introduced `safe.directory` to prevent privilege escalation on shared systems.

```bash
# If you see: "fatal: detected dubious ownership in repository"
# This means the repo is owned by a different user than the one running Git

# Mark a specific directory as safe
git config --global --add safe.directory /path/to/repo

# Mark all directories as safe (use with caution on single-user systems only)
git config --global --add safe.directory '*'

# In CI/CD environments (containers), this is commonly needed:
git config --global --add safe.directory /workspace

# Check current safe directories
git config --global --get-all safe.directory
```

## .gitignore Best Practices

### Common Patterns

```gitignore
# Environment and secrets
.env
.env.*
!.env.example
*.pem
*.key
*.p12
*.pfx
*.jks

# Credentials and tokens
credentials.json
service-account.json
**/secrets/
.htpasswd
.netrc

# IDE and editor files
.idea/
.vscode/
*.swp
*.swo
*~

# OS files
.DS_Store
Thumbs.db

# Build artifacts
node_modules/
dist/
build/
target/
*.pyc
__pycache__/

# Logs and databases
*.log
*.sqlite
*.db
```

### Global Gitignore

```bash
# Create a global gitignore for patterns that apply to all repositories
touch ~/.gitignore_global

# Configure Git to use it
git config --global core.excludesfile ~/.gitignore_global

# Common global patterns (OS/editor files — not project-specific):
# .DS_Store
# Thumbs.db
# .idea/
# .vscode/
# *.swp
```

### Preventing Sensitive File Commits

```bash
# Check if a file would be ignored
git check-ignore -v .env

# List all ignored files in the working tree
git status --ignored

# If a secret was already committed, remove it from tracking
# (the file stays on disk but is no longer tracked)
git rm --cached .env
echo ".env" >> .gitignore
git commit -m "chore: stop tracking .env"

# ⚠️ The secret is still in Git history — see "Removing secrets from history"

# Remove a file from the entire Git history (use with caution)
git filter-repo --invert-paths --path .env

# Or using BFG Repo-Cleaner
bfg --delete-files .env
git reflog expire --expire=now --all && git gc --prune=now --aggressive

# After rewriting history, force-push and notify all collaborators
git push origin --force --all
```

## Audit and Compliance

### Git Log for Auditing

```bash
# Show who changed what, and when
git log --pretty=format:'%h %ai %an <%ae> %s' --since='2025-01-01'

# Show commits that modified a specific file
git log --follow --all -- path/to/sensitive-file.conf

# Show all merge commits (code review decisions)
git log --merges --pretty=format:'%h %ai %an: %s'

# Show signature verification for all commits
git log --pretty=format:'%h %G? %GS %an %s'

# Export audit log as CSV
git log --pretty=format:'"%h","%ai","%an","%ae","%G?","%s"' > audit.csv

# Find who last modified each line in a file
git blame --line-porcelain path/to/file.py | \
  grep -E '^(author |author-mail |author-time )' | paste - - -
```

### Signed Commits for Compliance

Many compliance frameworks require proof of authorship and change integrity.

| Framework | Requirement | How Git Signing Helps |
|---|---|---|
| **SOC 2** | Access controls and audit trails | Signed commits prove identity; Git log provides change history |
| **PCI DSS** | Change management controls | Signed commits + branch protection enforce review and approval |
| **HIPAA** | Audit controls for ePHI access | Git log tracks all changes to code handling protected data |
| **FedRAMP** | Configuration management | Signed tags on releases prove artifact provenance |
| **ISO 27001** | Information security management | Signed commits + CODEOWNERS enforce segregation of duties |

### SLSA Framework

SLSA (Supply-chain Levels for Software Artifacts) defines levels of supply chain security. Git signing is a foundational component.

```
SLSA Levels and Git's Role:

Level 0: No guarantees
         └── No signing, no build provenance

Level 1: Documentation of the build process
         └── Build process is defined (e.g., CI/CD config in repo)

Level 2: Tamper resistance of the build service
         └── Builds run on a hosted, authenticated CI platform
         └── Signed commits verify source integrity

Level 3: Extra resistance to specific threats
         └── Source: verified source via signed commits + branch protection
         └── Build: isolated, ephemeral build environments
         └── Provenance: signed, non-falsifiable build provenance

Level 4: (Planned) Highest assurance
         └── Two-person review + hermetic builds + parameterless builds
```

## Best Practices Summary

- ✅ Sign all commits using GPG or SSH keys — enforce via `commit.gpgsign = true`
- ✅ Sign release tags to provide tamper-proof release points
- ✅ Enable GitHub vigilant mode to surface unsigned commits
- ✅ Use SSH signing (Git 2.34+) for simpler key management when GPG is not required
- ✅ Upload your signing keys to GitHub so commits show as "Verified"
- ✅ Use a credential helper backed by your OS keychain — never store credentials in plaintext
- ✅ Prefer SSH keys or fine-grained PATs over classic personal access tokens
- ✅ Install a pre-commit secret scanner (git-secrets or gitleaks) in every repository
- ✅ Run secret scanning in CI/CD to catch secrets in pull requests before merge
- ✅ Add a `.gitignore` that blocks `.env`, `*.pem`, `*.key`, and other sensitive patterns
- ✅ Configure a global gitignore for OS and editor files
- ✅ Set up branch protection rules requiring signed commits, status checks, and reviews
- ✅ Add a `CODEOWNERS` file to enforce review ownership for security-sensitive paths
- ✅ Include a `SECURITY.md` with responsible disclosure instructions
- ✅ Keep Git updated to patch known CVEs
- ✅ Use `safe.directory` on multi-user and CI/CD systems
- ✅ Use `git filter-repo` or BFG to remove accidentally committed secrets from history
- ❌ Do not commit secrets, credentials, or private keys to Git — even in private repositories
- ❌ Do not use `credential.helper store` — it saves passwords in plaintext
- ❌ Do not disable branch protection rules for administrators
- ❌ Do not rely on `.gitignore` alone to prevent secret commits — add pre-commit hooks as a second layer
- ❌ Do not skip Git updates — unpatched versions have known remote code execution vulnerabilities

## Next Steps

Continue to the [Git Overview](00-OVERVIEW.md) to review the complete learning path, or revisit [Hooks and Automation](06-HOOKS-AND-AUTOMATION.md) to set up pre-commit hooks that enforce signing and secret scanning in your workflow.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial security and signing documentation |
