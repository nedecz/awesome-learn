# Git Hooks and Automation

## Table of Contents

1. [Overview](#overview)
2. [Git Hooks](#git-hooks)
   - [Client-Side vs Server-Side Hooks](#client-side-vs-server-side-hooks)
   - [Hook Types](#hook-types)
3. [Client-Side Hooks](#client-side-hooks)
   - [Pre-Commit Hook](#pre-commit-hook)
   - [Commit-Msg Hook](#commit-msg-hook)
   - [Pre-Push Hook](#pre-push-hook)
   - [Other Client-Side Hooks](#other-client-side-hooks)
4. [Server-Side Hooks](#server-side-hooks)
   - [Pre-Receive Hook](#pre-receive-hook)
   - [Update Hook](#update-hook)
   - [Post-Receive Hook](#post-receive-hook)
   - [Enforcing Policies](#enforcing-policies)
5. [Security Commit Checks](#security-commit-checks)
   - [Secret Detection Hooks](#secret-detection-hooks)
   - [Commit Signature Verification](#commit-signature-verification)
   - [Dependency Vulnerability Scanning](#dependency-vulnerability-scanning)
   - [Complete Security Hook Pipeline](#complete-security-hook-pipeline)
6. [Pre-Commit Framework](#pre-commit-framework)
   - [Installation](#installation)
   - [Configuration](#configuration)
   - [Popular Hooks](#popular-hooks)
7. [Husky (JavaScript)](#husky-javascript)
   - [Setup](#setup)
   - [Lint-Staged Integration](#lint-staged-integration)
8. [Git Hooks for CI/CD](#git-hooks-for-cicd)
   - [Triggering Pipelines](#triggering-pipelines)
   - [GitHub Actions Integration](#github-actions-integration)
   - [Webhook Events](#webhook-events)
9. [Automating with Git Aliases](#automating-with-git-aliases)
   - [Useful Aliases for Productivity](#useful-aliases-for-productivity)
   - [Global Gitconfig](#global-gitconfig)
10. [Git Attributes](#git-attributes)
    - [Line Endings](#line-endings)
    - [Merge Drivers](#merge-drivers)
    - [Diff Drivers](#diff-drivers)
    - [Git LFS](#git-lfs)
11. [Git Templates](#git-templates)
    - [Repository Templates](#repository-templates)
    - [Default Hooks](#default-hooks)
12. [Best Practices](#best-practices)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

## Overview

Git hooks and automation tools transform Git from a simple version control system into a powerful workflow engine. Hooks enforce code quality at commit time, automation reduces repetitive tasks, and integration with CI/CD pipelines ensures that every change meets your team's standards before it reaches production.

### Target Audience

- Developers looking to automate code quality checks
- Team leads enforcing commit and coding standards
- DevOps engineers integrating Git with CI/CD pipelines

### Scope

- Client-side and server-side Git hooks
- Pre-commit framework and Husky for managing hooks
- CI/CD pipeline integration with Git events
- Git aliases, attributes, and templates for workflow automation

## Git Hooks

Git hooks are scripts that Git executes automatically before or after specific events such as committing, pushing, or merging. They live in the `.git/hooks/` directory of every repository.

```
.git/hooks/
├── pre-commit          ← Runs before a commit is created
├── prepare-commit-msg  ← Runs before the commit message editor opens
├── commit-msg          ← Runs after the commit message is entered
├── post-commit         ← Runs after a commit is created
├── pre-push            ← Runs before a push is executed
├── pre-rebase          ← Runs before a rebase starts
├── post-merge          ← Runs after a merge completes
├── post-checkout        ← Runs after a checkout or switch
├── pre-receive         ← (server) Runs before refs are updated
├── update              ← (server) Runs once per ref being updated
└── post-receive        ← (server) Runs after refs are updated
```

### Client-Side vs Server-Side Hooks

```
┌──────────────────────────────────────────────────────────┐
│                  Developer Machine                         │
│                                                           │
│  edit → stage → pre-commit → commit-msg → post-commit    │
│                                                           │
│  push → pre-push ─────────────────────────────────┐      │
└───────────────────────────────────────────────────┼──────┘
                                                    │
                                                    ▼
┌──────────────────────────────────────────────────────────┐
│                  Remote Server                            │
│                                                           │
│  pre-receive → update (per ref) → post-receive           │
│                                                           │
│  Enforce policies, trigger CI/CD, send notifications     │
└──────────────────────────────────────────────────────────┘
```

| Aspect | Client-Side Hooks | Server-Side Hooks |
|---|---|---|
| **Location** | Developer's local `.git/hooks/` | Remote server repository |
| **Enforcement** | Advisory — developers can bypass with `--no-verify` | Mandatory — cannot be bypassed |
| **Use case** | Linting, formatting, commit message validation | Policy enforcement, CI triggers |
| **Distribution** | Must be shared via tooling (Husky, pre-commit) | Configured once on the server |
| **Performance** | Runs on developer's machine | Runs on server resources |

### Hook Types

| Hook | Trigger | Side | Can Block? | Typical Use |
|---|---|---|---|---|
| `pre-commit` | Before commit is created | Client | ✅ Yes | Lint, format, run quick tests |
| `prepare-commit-msg` | Before commit message editor opens | Client | ✅ Yes | Auto-populate commit template |
| `commit-msg` | After commit message is entered | Client | ✅ Yes | Validate commit message format |
| `post-commit` | After commit is created | Client | ❌ No | Notifications, logging |
| `pre-rebase` | Before rebase starts | Client | ✅ Yes | Prevent rebasing published commits |
| `post-checkout` | After `git checkout` or `git switch` | Client | ❌ No | Update dependencies, clear caches |
| `post-merge` | After a merge completes | Client | ❌ No | Reinstall dependencies |
| `pre-push` | Before push is executed | Client | ✅ Yes | Run tests, check branch policies |
| `pre-receive` | Before any refs are updated | Server | ✅ Yes | Enforce branch protection policies |
| `update` | Once per ref being pushed | Server | ✅ Yes | Per-branch policy enforcement |
| `post-receive` | After all refs are updated | Server | ❌ No | Trigger CI/CD, send notifications |

## Client-Side Hooks

### Pre-Commit Hook

The `pre-commit` hook runs before a commit is created. It is the most commonly used hook for enforcing code quality.

```bash
#!/usr/bin/env bash
# .git/hooks/pre-commit — Lint and format staged files

set -euo pipefail

echo "Running pre-commit checks..."

# Get list of staged files
STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACM)

# Run linter on staged Python files
PYTHON_FILES=$(echo "$STAGED_FILES" | grep '\.py$' || true)
if [ -n "$PYTHON_FILES" ]; then
    echo "Linting Python files..."
    echo "$PYTHON_FILES" | xargs ruff check --fix
    echo "$PYTHON_FILES" | xargs ruff format --check
fi

# Run linter on staged JavaScript/TypeScript files
JS_FILES=$(echo "$STAGED_FILES" | grep -E '\.(js|ts|jsx|tsx)$' || true)
if [ -n "$JS_FILES" ]; then
    echo "Linting JS/TS files..."
    echo "$JS_FILES" | xargs npx eslint
fi

# Prevent committing large files (> 5MB)
for file in $STAGED_FILES; do
    if [ -f "$file" ]; then
        FILE_SIZE=$(wc -c < "$file")
        if [ "$FILE_SIZE" -gt 5242880 ]; then
            echo "ERROR: $file is larger than 5MB. Use Git LFS for large files."
            exit 1
        fi
    fi
done

# Check for debug statements
if echo "$STAGED_FILES" | xargs grep -l 'debugger\|console\.log\|binding\.pry\|import pdb' 2>/dev/null; then
    echo "WARNING: Debug statements found in staged files. Remove them before committing."
    exit 1
fi

echo "Pre-commit checks passed ✓"
```

### Commit-Msg Hook

The `commit-msg` hook validates the commit message format. This example enforces the [Conventional Commits](https://www.conventionalcommits.org/) specification.

```bash
#!/usr/bin/env bash
# .git/hooks/commit-msg — Enforce Conventional Commits

set -euo pipefail

COMMIT_MSG_FILE="$1"
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

# Conventional Commits pattern:
#   type(optional-scope): description
#   Examples: feat(auth): add login page
#             fix: resolve null pointer exception
#             docs: update README
PATTERN="^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\(.+\))?: .{1,72}$"

# Check the first line (subject) of the commit message
FIRST_LINE=$(echo "$COMMIT_MSG" | head -1)

if ! echo "$FIRST_LINE" | grep -Eq "$PATTERN"; then
    echo "ERROR: Commit message does not follow Conventional Commits format."
    echo ""
    echo "Expected: <type>(<scope>): <description>"
    echo ""
    echo "Types: feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert"
    echo ""
    echo "Examples:"
    echo "  feat(auth): add OAuth2 login flow"
    echo "  fix: resolve race condition in worker pool"
    echo "  docs: update API documentation for v2"
    echo ""
    echo "Your message: $FIRST_LINE"
    exit 1
fi

echo "Commit message format OK ✓"
```

### Pre-Push Hook

The `pre-push` hook runs before a push is executed. Use it to run tests and prevent pushing broken code.

```bash
#!/usr/bin/env bash
# .git/hooks/pre-push — Run tests before pushing

set -euo pipefail

REMOTE="$1"
URL="$2"

echo "Running tests before push to $REMOTE..."

# Read the refs being pushed
while read LOCAL_REF LOCAL_SHA REMOTE_REF REMOTE_SHA; do
    # Skip delete operations
    if [ "$LOCAL_SHA" = "0000000000000000000000000000000000000000" ]; then
        continue
    fi

    # Prevent pushing directly to main/master
    if echo "$REMOTE_REF" | grep -qE 'refs/heads/(main|master)$'; then
        echo "ERROR: Direct push to main/master is not allowed."
        echo "Create a pull request instead."
        exit 1
    fi
done

# Run the test suite
if [ -f "package.json" ]; then
    npm test
elif [ -f "Makefile" ]; then
    make test
elif [ -f "pytest.ini" ] || [ -f "pyproject.toml" ]; then
    python -m pytest --tb=short -q
fi

echo "All tests passed — push allowed ✓"
```

### Other Client-Side Hooks

```bash
#!/usr/bin/env bash
# .git/hooks/post-merge — Reinstall dependencies after merge

set -euo pipefail

CHANGED_FILES=$(git diff-tree -r --name-only --no-commit-id ORIG_HEAD HEAD)

# Reinstall Node.js dependencies if package-lock.json changed
if echo "$CHANGED_FILES" | grep -q "package-lock.json"; then
    echo "package-lock.json changed — running npm install..."
    npm install
fi

# Reinstall Python dependencies if requirements.txt changed
if echo "$CHANGED_FILES" | grep -q "requirements.txt"; then
    echo "requirements.txt changed — running pip install..."
    pip install -r requirements.txt
fi
```

```bash
#!/usr/bin/env bash
# .git/hooks/post-checkout — Notify about dependency changes after branch switch

set -euo pipefail

PREV_HEAD="$1"
NEW_HEAD="$2"
BRANCH_CHECKOUT="$3"  # 1 = branch checkout, 0 = file checkout

if [ "$BRANCH_CHECKOUT" = "1" ]; then
    CHANGED_FILES=$(git diff --name-only "$PREV_HEAD" "$NEW_HEAD")

    if echo "$CHANGED_FILES" | grep -q "package-lock.json"; then
        echo "⚠ package-lock.json changed — run 'npm install' to update dependencies."
    fi

    if echo "$CHANGED_FILES" | grep -q "migrations/"; then
        echo "⚠ Database migrations changed — run migrations before testing."
    fi
fi
```

```bash
#!/usr/bin/env bash
# .git/hooks/prepare-commit-msg — Auto-add branch name as scope

set -euo pipefail

COMMIT_MSG_FILE="$1"
COMMIT_SOURCE="$2"

# Only modify if this is a regular commit (not merge, squash, etc.)
if [ -z "${COMMIT_SOURCE:-}" ]; then
    BRANCH_NAME=$(git symbolic-ref --short HEAD 2>/dev/null)

    # Extract ticket number from branch name (e.g., feature/JIRA-123-description)
    TICKET=$(echo "$BRANCH_NAME" | grep -oE '[A-Z]+-[0-9]+' || true)

    if [ -n "$TICKET" ]; then
        # Prepend ticket number to commit message
        CURRENT_MSG=$(cat "$COMMIT_MSG_FILE")
        echo "[$TICKET] $CURRENT_MSG" > "$COMMIT_MSG_FILE"
    fi
fi
```

## Server-Side Hooks

Server-side hooks run on the remote repository and **cannot be bypassed** by developers. They are the last line of defense for enforcing repository policies.

### Pre-Receive Hook

The `pre-receive` hook runs once before any refs are updated. It receives all refs being pushed on stdin and can reject the entire push.

```bash
#!/usr/bin/env bash
# hooks/pre-receive — Enforce repository-wide policies

set -euo pipefail

ZERO="0000000000000000000000000000000000000000"

while read OLD_REV NEW_REV REF_NAME; do
    # Skip delete operations
    if [ "$NEW_REV" = "$ZERO" ]; then
        continue
    fi

    # Prevent force-push to protected branches
    if echo "$REF_NAME" | grep -qE 'refs/heads/(main|master|release/.*)$'; then
        if [ "$OLD_REV" != "$ZERO" ]; then
            MERGE_BASE=$(git merge-base "$OLD_REV" "$NEW_REV" 2>/dev/null || true)
            if [ "$MERGE_BASE" != "$OLD_REV" ]; then
                echo "ERROR: Force-push to $REF_NAME is not allowed."
                exit 1
            fi
        fi
    fi

    # Reject commits with files larger than 10MB
    if [ "$OLD_REV" = "$ZERO" ]; then
        DIFF_RANGE="$NEW_REV"
    else
        DIFF_RANGE="$OLD_REV..$NEW_REV"
    fi

    LARGE_FILES=$(git diff --stat "$DIFF_RANGE" 2>/dev/null | \
        awk '$3 > 10485760 {print $1}' || true)
    if [ -n "$LARGE_FILES" ]; then
        echo "ERROR: The following files exceed the 10MB size limit:"
        echo "$LARGE_FILES"
        echo "Use Git LFS for large files."
        exit 1
    fi
done

echo "Pre-receive checks passed ✓"
```

### Update Hook

The `update` hook runs once per ref being updated. It receives the ref name, old SHA, and new SHA as arguments — useful for per-branch policies.

```bash
#!/usr/bin/env bash
# hooks/update — Per-branch policy enforcement

set -euo pipefail

REF_NAME="$1"
OLD_REV="$2"
NEW_REV="$3"

# Only allow tags matching semver pattern
if echo "$REF_NAME" | grep -q 'refs/tags/'; then
    TAG_NAME=$(echo "$REF_NAME" | sed 's|refs/tags/||')
    if ! echo "$TAG_NAME" | grep -Eq '^v[0-9]+\.[0-9]+\.[0-9]+(-[a-zA-Z0-9.]+)?$'; then
        echo "ERROR: Tag '$TAG_NAME' does not follow semver (e.g., v1.2.3)."
        exit 1
    fi
fi

# Require signed commits on main
if [ "$REF_NAME" = "refs/heads/main" ]; then
    UNSIGNED=$(git log --format='%H %G?' "$OLD_REV..$NEW_REV" | grep ' N$' || true)
    if [ -n "$UNSIGNED" ]; then
        echo "ERROR: All commits to main must be GPG-signed."
        echo "Unsigned commits found:"
        echo "$UNSIGNED" | awk '{print $1}'
        exit 1
    fi
fi
```

### Post-Receive Hook

The `post-receive` hook runs after all refs are updated. It cannot reject the push but is ideal for triggering side effects.

```bash
#!/usr/bin/env bash
# hooks/post-receive — Trigger deployment and notifications

set -euo pipefail

while read OLD_REV NEW_REV REF_NAME; do
    BRANCH=$(echo "$REF_NAME" | sed 's|refs/heads/||')

    # Trigger deployment when main is updated
    if [ "$BRANCH" = "main" ]; then
        echo "Triggering production deployment..."
        curl -s -X POST "https://deploy.example.com/api/deploy" \
            -H "Authorization: Bearer $DEPLOY_TOKEN" \
            -H "Content-Type: application/json" \
            -d "{\"branch\": \"$BRANCH\", \"sha\": \"$NEW_REV\"}"
    fi

    # Send Slack notification
    AUTHOR=$(git log -1 --format='%an' "$NEW_REV")
    COMMIT_MSG=$(git log -1 --format='%s' "$NEW_REV")
    curl -s -X POST "$SLACK_WEBHOOK_URL" \
        -H "Content-Type: application/json" \
        -d "{\"text\": \"🚀 $AUTHOR pushed to $BRANCH: $COMMIT_MSG\"}"
done
```

### Enforcing Policies

```
Policy Enforcement Layers:

  ┌──────────────────────────────────────────────────────────┐
  │  Client-Side (Advisory)                                   │
  │                                                          │
  │  pre-commit ──▶ Lint, format, check for secrets          │
  │  commit-msg ──▶ Validate Conventional Commits            │
  │  pre-push   ──▶ Run tests, check branch naming           │
  │                                                          │
  │  ⚠ Can be bypassed with --no-verify                      │
  └──────────────────────────────────────────────────────────┘
                          │
                          ▼
  ┌──────────────────────────────────────────────────────────┐
  │  Server-Side (Mandatory)                                  │
  │                                                          │
  │  pre-receive ──▶ Block force-push, reject large files    │
  │  update      ──▶ Enforce signed commits, tag format      │
  │  post-receive ──▶ Trigger CI/CD, send notifications      │
  │                                                          │
  │  ✅ Cannot be bypassed by developers                      │
  └──────────────────────────────────────────────────────────┘
                          │
                          ▼
  ┌──────────────────────────────────────────────────────────┐
  │  Platform-Level (GitHub/GitLab)                           │
  │                                                          │
  │  Branch protection rules, required status checks,        │
  │  required reviews, CODEOWNERS                            │
  │                                                          │
  │  ✅ Enforced by the hosting platform                      │
  └──────────────────────────────────────────────────────────┘
```

## Security Commit Checks

Security-focused Git hooks prevent secrets, unsigned commits, and vulnerable dependencies from entering your repository. By catching security issues **at commit time**, you avoid costly remediation after code is pushed or deployed.

```
Security Hook Pipeline:

  ┌───────────────────────────────────────────────────────────────┐
  │                    Developer Workflow                          │
  │                                                               │
  │  git add . ──▶ git commit ──▶ git push                       │
  │                  │      │              │                       │
  │            ┌─────┘      └─────┐        └──────┐               │
  │            ▼                  ▼               ▼               │
  │     ┌─────────────┐  ┌──────────────┐  ┌───────────┐        │
  │     │ pre-commit   │  │ commit-msg   │  │ pre-push  │        │
  │     │              │  │              │  │           │        │
  │     │ • Secrets    │  │ • Signed?    │  │ • Dep     │        │
  │     │   detection  │  │ • Conven-    │  │   audit   │        │
  │     │ • Private    │  │   tional     │  │ • Full    │        │
  │     │   key check  │  │   format?    │  │   secret  │        │
  │     │ • Sensitive  │  │              │  │   scan    │        │
  │     │   file block │  │              │  │           │        │
  │     └─────────────┘  └──────────────┘  └───────────┘        │
  │           │                │                 │                │
  │           ▼                ▼                 ▼                │
  │     Block commit     Block commit      Block push            │
  │     if secrets       if unsigned or    if vulnerable          │
  │     found            malformed msg     deps found             │
  └───────────────────────────────────────────────────────────────┘
```

### Secret Detection Hooks

The most critical security hook prevents secrets (API keys, passwords, private keys, tokens) from being committed to the repository. Once a secret is in Git history, it is **extremely difficult to fully remove** — even after deletion, it remains in the reflog and can be found by anyone who clones the repo.

#### Pre-Commit Hook: Secret Scanner

```bash
#!/usr/bin/env bash
# .git/hooks/pre-commit — Block commits containing secrets

set -euo pipefail

STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACM)
[ -z "$STAGED_FILES" ] && exit 0

echo "🔍 Scanning staged files for secrets..."

# --- Pattern-based detection ---
# Common secret patterns (AWS keys, generic passwords, private keys, tokens)
SECRETS_PATTERNS=(
    'AKIA[0-9A-Z]{16}'                               # AWS Access Key ID
    'ASIA[0-9A-Z]{16}'                               # AWS Temporary Access Key
    '["\x27]?password["\x27]?\s*[:=]\s*["\x27].+'    # Hardcoded passwords
    '-----BEGIN (RSA |EC |DSA |OPENSSH )?PRIVATE KEY' # Private keys
    'ghp_[0-9a-zA-Z]{36}'                            # GitHub Personal Access Token
    'github_pat_[0-9a-zA-Z]{22}_[0-9a-zA-Z]{59}'    # GitHub Fine-grained PAT
    'sk-[0-9a-zA-Z]{48}'                             # OpenAI API Key
    'xox[bporas]-[0-9a-zA-Z-]+'                      # Slack Tokens
    'SG\.[0-9A-Za-z_-]{22}\.[0-9A-Za-z_-]{43}'      # SendGrid API Key
    'eyJ[A-Za-z0-9_-]*\.eyJ[A-Za-z0-9_-]*\.'        # JWT Token
)

COMBINED_PATTERN=$(IFS='|'; echo "${SECRETS_PATTERNS[*]}")
FOUND_SECRETS=0

for file in $STAGED_FILES; do
    if [ -f "$file" ]; then
        MATCHES=$(git diff --cached -- "$file" | grep -PE "$COMBINED_PATTERN" || true)
        if [ -n "$MATCHES" ]; then
            echo "❌ SECRET DETECTED in $file:"
            echo "$MATCHES" | head -3
            FOUND_SECRETS=1
        fi
    fi
done

# --- File-based detection ---
# Block files that commonly contain secrets
BLOCKED_FILES_PATTERN='\.(pem|key|p12|pfx|jks|keystore|env\.local|env\.production)$'
BLOCKED=$(echo "$STAGED_FILES" | grep -E "$BLOCKED_FILES_PATTERN" || true)
if [ -n "$BLOCKED" ]; then
    echo "❌ BLOCKED FILES that commonly contain secrets:"
    echo "$BLOCKED"
    echo "   Add these to .gitignore or use a secret manager."
    FOUND_SECRETS=1
fi

if [ "$FOUND_SECRETS" -eq 1 ]; then
    echo ""
    echo "🛑 Commit blocked. Remove secrets before committing."
    echo "   If this is a false positive, use: git commit --no-verify"
    exit 1
fi

echo "✅ No secrets detected."
```

#### Using gitleaks as a Pre-Commit Hook

[gitleaks](https://github.com/gitleaks/gitleaks) is the industry-standard tool for secret detection. It supports custom rules, baseline files (to ignore known false positives), and integration with CI pipelines.

```bash
#!/usr/bin/env bash
# .git/hooks/pre-commit — Use gitleaks for secret detection

set -euo pipefail

# Scan only staged changes (fast, focused)
if command -v gitleaks &>/dev/null; then
    echo "🔍 Running gitleaks on staged changes..."
    gitleaks protect --staged --verbose
else
    echo "⚠ gitleaks not installed. Install: brew install gitleaks"
    echo "  Falling back to pattern-based detection..."
    # Add pattern-based fallback here
fi
```

### Commit Signature Verification

A server-side hook (or a CI check) can **reject unsigned commits**, ensuring every change in the repository has a verified author identity.

#### Server-Side Pre-Receive Hook

```bash
#!/usr/bin/env bash
# hooks/pre-receive — Reject unsigned commits (server-side)
# Deploy on Git server or use as GitHub Actions check

while read OLD_REV NEW_REV REF_NAME; do
    # Skip branch deletions
    if [ "$NEW_REV" = "0000000000000000000000000000000000000000" ]; then
        continue
    fi

    # Only enforce on main and release branches
    case "$REF_NAME" in
        refs/heads/main|refs/heads/release/*)
            ;;
        *)
            continue
            ;;
    esac

    # Check each new commit for a valid signature
    COMMITS=$(git rev-list "$OLD_REV".."$NEW_REV" 2>/dev/null || echo "$NEW_REV")
    for COMMIT in $COMMITS; do
        SIGNATURE=$(git verify-commit "$COMMIT" 2>&1 || true)
        if echo "$SIGNATURE" | grep -q "Good signature"; then
            continue
        fi
        echo "❌ UNSIGNED COMMIT: $COMMIT"
        echo "   All commits to $REF_NAME must be GPG or SSH signed."
        echo "   See: git commit -S -m 'your message'"
        exit 1
    done
done

echo "✅ All commits are signed."
```

#### GitHub Actions: Verify Commit Signatures

```yaml
# .github/workflows/verify-signatures.yml
name: Verify Commit Signatures
on:
  pull_request:
    branches: [main, release/*]

jobs:
  verify-signatures:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Verify all commits are signed
        run: |
          UNSIGNED=0
          for COMMIT in $(git rev-list origin/${{ github.base_ref }}..${{ github.sha }}); do
            if ! git verify-commit "$COMMIT" 2>/dev/null; then
              echo "❌ Unsigned commit: $COMMIT ($(git log -1 --format='%s' $COMMIT))"
              UNSIGNED=1
            fi
          done
          if [ "$UNSIGNED" -eq 1 ]; then
            echo "🛑 All commits must be signed. See: https://docs.github.com/en/authentication/managing-commit-signature-verification"
            exit 1
          fi
          echo "✅ All commits are signed."
```

### Dependency Vulnerability Scanning

A `pre-push` hook can audit dependencies for known vulnerabilities before code reaches the remote repository:

```bash
#!/usr/bin/env bash
# .git/hooks/pre-push — Audit dependencies before pushing

set -euo pipefail

echo "🔍 Scanning dependencies for vulnerabilities..."

# Node.js projects
if [ -f "package-lock.json" ]; then
    echo "Auditing npm dependencies..."
    AUDIT_OUTPUT=$(npm audit --production 2>&1 || true)
    if echo "$AUDIT_OUTPUT" | grep -q "high\|critical"; then
        echo "❌ Critical/high vulnerability found in npm dependencies:"
        npm audit --production 2>&1 | grep -A2 "high\|critical" | head -20
        echo ""
        echo "Run 'npm audit fix' to resolve, or 'git push --no-verify' to bypass."
        exit 1
    fi
fi

# Python projects
if [ -f "requirements.txt" ]; then
    if command -v pip-audit &>/dev/null; then
        echo "Auditing Python dependencies..."
        if ! pip-audit -r requirements.txt --desc 2>/dev/null; then
            echo "❌ Vulnerabilities found in Python dependencies."
            exit 1
        fi
    fi
fi

# Go projects
if [ -f "go.sum" ]; then
    echo "Auditing Go dependencies..."
    if command -v govulncheck &>/dev/null; then
        if ! govulncheck ./... 2>/dev/null; then
            echo "❌ Vulnerabilities found in Go dependencies."
            exit 1
        fi
    fi
fi

echo "✅ No known vulnerabilities found."
```

### Complete Security Hook Pipeline

Here is a complete `.pre-commit-config.yaml` that combines all security checks into a single, manageable configuration:

```yaml
# .pre-commit-config.yaml — Security-focused hook pipeline

repos:
  # --- Secret Detection ---
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks
        name: "🔒 Detect secrets (gitleaks)"

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: detect-private-key
        name: "🔑 Detect private keys"
      - id: check-added-large-files
        name: "📦 Block large files (may contain secrets)"
        args: ['--maxkb=500']

  # --- Commit Message Validation ---
  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: v3.3.0
    hooks:
      - id: conventional-pre-commit
        name: "📝 Validate conventional commit message"
        stages: [commit-msg]
        args: [feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert]

  # --- File Security ---
  - repo: local
    hooks:
      - id: block-sensitive-files
        name: "🚫 Block sensitive file patterns"
        entry: bash -c 'echo "$@" | grep -E "\.(pem|key|p12|pfx|env\.local|env\.production)$" && echo "❌ Sensitive file detected" && exit 1 || exit 0'
        language: system
        stages: [pre-commit]
```

#### Security Check Summary

| Hook Stage | Check | Tool | Bypassable? |
|---|---|---|---|
| `pre-commit` | Secret detection | gitleaks, git-secrets | ⚠ `--no-verify` |
| `pre-commit` | Private key detection | pre-commit-hooks | ⚠ `--no-verify` |
| `pre-commit` | Sensitive file blocking | custom script | ⚠ `--no-verify` |
| `commit-msg` | Conventional commit format | conventional-pre-commit, commitlint | ⚠ `--no-verify` |
| `pre-push` | Dependency vulnerability audit | npm audit, pip-audit, govulncheck | ⚠ `--no-verify` |
| `pre-receive` | Commit signature verification | git verify-commit (server-side) | ✅ Cannot bypass |
| Platform | Branch protection, required checks | GitHub/GitLab settings | ✅ Cannot bypass |

> **Important:** Client-side hooks can be bypassed with `--no-verify`. For mandatory enforcement, use **server-side hooks** or **platform-level branch protection rules** (required status checks, required signed commits). Client-side hooks are a first line of defense — not the only one.

## Pre-Commit Framework

The [pre-commit](https://pre-commit.com/) framework is a language-agnostic tool for managing and running Git hooks. It uses a `.pre-commit-config.yaml` file to define which hooks to run.

### Installation

```bash
# Install with pip
pip install pre-commit

# Install with Homebrew (macOS)
brew install pre-commit

# Install hooks into .git/hooks/
pre-commit install

# Install for additional hook stages
pre-commit install --hook-type commit-msg
pre-commit install --hook-type pre-push

# Run all hooks manually against all files
pre-commit run --all-files
```

### Configuration

```yaml
# .pre-commit-config.yaml

# Minimum pre-commit version required
minimum_pre_commit_version: "3.0.0"

# Default settings for all hooks
default_language_version:
  python: python3.12
  node: "20.11.0"

default_stages: [pre-commit]

repos:
  # --- General ---
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace       # Remove trailing whitespace
      - id: end-of-file-fixer         # Ensure files end with a newline
      - id: check-yaml                # Validate YAML syntax
      - id: check-json                # Validate JSON syntax
      - id: check-toml                # Validate TOML syntax
      - id: check-added-large-files   # Prevent large files from being committed
        args: ['--maxkb=500']
      - id: check-merge-conflict      # Check for merge conflict markers
      - id: detect-private-key        # Detect private keys

  # --- Python ---
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.5.0
    hooks:
      - id: ruff                       # Python linter (replaces flake8, isort, etc.)
        args: [--fix]
      - id: ruff-format               # Python formatter (replaces black)

  # --- JavaScript / TypeScript ---
  - repo: https://github.com/pre-commit/mirrors-eslint
    rev: v9.6.0
    hooks:
      - id: eslint
        files: \.(js|jsx|ts|tsx)$
        additional_dependencies:
          - eslint@9.6.0
          - typescript@5.5.0
          - '@typescript-eslint/parser@7.0.0'

  - repo: https://github.com/pre-commit/mirrors-prettier
    rev: v4.0.0-alpha.8
    hooks:
      - id: prettier
        files: \.(js|jsx|ts|tsx|json|css|md|yaml|yml)$

  # --- Shell ---
  - repo: https://github.com/shellcheck-py/shellcheck-py
    rev: v0.10.0.1
    hooks:
      - id: shellcheck                 # Shell script linter

  # --- Secrets Detection ---
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks                   # Detect secrets in code

  # --- Commit Message ---
  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: v3.3.0
    hooks:
      - id: conventional-pre-commit
        stages: [commit-msg]
        args: [feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert]
```

### Popular Hooks

| Hook | Language | Purpose |
|---|---|---|
| `ruff` | Python | Fast linter and formatter (replaces flake8, isort, black) |
| `eslint` | JavaScript/TypeScript | Lint JS/TS files |
| `prettier` | Multi-language | Opinionated code formatter |
| `shellcheck` | Shell | Static analysis for shell scripts |
| `gitleaks` | Any | Detect hardcoded secrets |
| `mypy` | Python | Static type checking |
| `hadolint` | Docker | Lint Dockerfiles |
| `markdownlint` | Markdown | Lint Markdown files |
| `terraform_fmt` | Terraform | Format Terraform files |
| `check-yaml` | YAML | Validate YAML syntax |

## Husky (JavaScript)

[Husky](https://typicode.github.io/husky/) is the standard tool for managing Git hooks in JavaScript/TypeScript projects. It integrates with `lint-staged` to run linters only on staged files.

### Setup

```bash
# Install Husky
npm install --save-dev husky

# Initialize Husky (creates .husky/ directory)
npx husky init

# This creates .husky/pre-commit with a default script
```

```
Project Structure:

  my-project/
  ├── .husky/
  │   ├── pre-commit       ← Hook scripts live here
  │   └── commit-msg
  ├── package.json
  └── ...
```

```bash
# .husky/pre-commit
npx lint-staged
```

```bash
# .husky/commit-msg
npx --no -- commitlint --edit "$1"
```

### Lint-Staged Integration

`lint-staged` runs linters only against files that are staged for commit, making hooks fast even in large repositories.

```bash
# Install lint-staged
npm install --save-dev lint-staged
```

```json
// package.json
{
  "scripts": {
    "prepare": "husky"
  },
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md,yaml,yml}": [
      "prettier --write"
    ],
    "*.css": [
      "stylelint --fix",
      "prettier --write"
    ],
    "*.py": [
      "ruff check --fix",
      "ruff format"
    ]
  },
  "devDependencies": {
    "husky": "^9.0.0",
    "lint-staged": "^15.0.0",
    "eslint": "^9.0.0",
    "prettier": "^3.0.0",
    "@commitlint/cli": "^19.0.0",
    "@commitlint/config-conventional": "^19.0.0"
  }
}
```

```javascript
// commitlint.config.js
export default {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      ['feat', 'fix', 'docs', 'style', 'refactor', 'perf', 'test', 'build', 'ci', 'chore', 'revert'],
    ],
    'subject-max-length': [2, 'always', 72],
    'body-max-line-length': [2, 'always', 100],
  },
};
```

## Git Hooks for CI/CD

### Triggering Pipelines

Git hooks and platform events work together to trigger CI/CD pipelines automatically.

```
Push Event Flow:

  Developer
      │
      │  git push
      ▼
  ┌──────────────┐    webhook     ┌──────────────┐
  │ Git Server   │ ──────────────▶│  CI/CD       │
  │ (GitHub)     │                │  (Actions)   │
  │              │                │              │
  │ post-receive │                │  Build       │
  │ hook fires   │                │  Test        │
  └──────────────┘                │  Deploy      │
                                  └──────────────┘
```

### GitHub Actions Integration

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

permissions:
  contents: read
  pull-requests: write

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npx eslint .
      - run: npx prettier --check .

  test:
    runs-on: ubuntu-latest
    needs: lint
    strategy:
      matrix:
        node-version: [18, 20, 22]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: npm test

  deploy:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    steps:
      - uses: actions/checkout@v4
      - run: echo "Deploying to production..."
```

```yaml
# .github/workflows/pr-checks.yml — Enforce commit message format in PRs
name: PR Checks

on:
  pull_request:
    types: [opened, synchronize, reopened, edited]

jobs:
  commitlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm install @commitlint/cli @commitlint/config-conventional
      - run: npx commitlint --from ${{ github.event.pull_request.base.sha }} --to ${{ github.event.pull_request.head.sha }}
```

### Webhook Events

| Event | Trigger | Common Use |
|---|---|---|
| `push` | Code pushed to a branch | Run CI pipeline |
| `pull_request` | PR opened, updated, or merged | Run checks, request reviews |
| `release` | Release created or published | Build and publish artifacts |
| `tag` | Tag created | Build versioned release |
| `issue_comment` | Comment on an issue or PR | Bot commands (`/deploy`, `/approve`) |
| `workflow_dispatch` | Manual trigger via UI or API | On-demand deployments |
| `schedule` | Cron-based schedule | Nightly builds, dependency updates |

## Automating with Git Aliases

### Useful Aliases for Productivity

```bash
# Add aliases via command line
git config --global alias.st "status --short --branch"
git config --global alias.co "checkout"
git config --global alias.br "branch"
git config --global alias.cm "commit -m"
git config --global alias.amend "commit --amend --no-edit"

# Log aliases
git config --global alias.lg "log --oneline --graph --decorate --all -20"
git config --global alias.last "log -1 --stat"
git config --global alias.today "log --oneline --since='midnight' --author='$(git config user.name)'"

# Workflow aliases
git config --global alias.undo "reset --soft HEAD~1"
git config --global alias.unstage "restore --staged"
git config --global alias.wip "!git add -A && git commit -m 'chore: WIP [skip ci]'"
git config --global alias.unwip "!git log -1 --format='%s' | grep -q 'WIP' && git reset HEAD~1"

# Cleanup aliases
git config --global alias.prune-branches "!git fetch -p && git branch -vv | grep ': gone]' | awk '{print \$1}' | xargs -r git branch -d"
git config --global alias.prune-merged "!git branch --merged main | grep -v 'main' | xargs -r git branch -d"
```

### Global Gitconfig

```ini
# ~/.gitconfig

[user]
    name = Your Name
    email = your.email@example.com
    signingkey = ABCDEF1234567890

[core]
    editor = vim
    autocrlf = input          # Convert CRLF to LF on commit (macOS/Linux)
    whitespace = fix           # Fix whitespace issues automatically
    pager = delta              # Use delta for better diffs

[init]
    defaultBranch = main

[pull]
    rebase = true              # Rebase instead of merge on pull

[push]
    autoSetupRemote = true     # Auto-set upstream on first push
    default = current

[fetch]
    prune = true               # Remove stale remote-tracking branches

[merge]
    conflictstyle = zdiff3     # Show base version in conflict markers

[diff]
    algorithm = histogram      # Better diff algorithm
    colorMoved = default

[commit]
    gpgsign = true             # Sign all commits with GPG

[rerere]
    enabled = true             # Remember resolved merge conflicts

[alias]
    st = status --short --branch
    co = checkout
    br = branch
    cm = commit -m
    amend = commit --amend --no-edit
    lg = log --oneline --graph --decorate --all -20
    last = log -1 --stat
    undo = reset --soft HEAD~1
    unstage = restore --staged
    wip = "!git add -A && git commit -m 'chore: WIP [skip ci]'"
    prune-branches = "!git fetch -p && git branch -vv | grep ': gone]' | awk '{print $1}' | xargs -r git branch -d"
```

## Git Attributes

The `.gitattributes` file controls how Git handles files — line endings, diff rendering, merge strategies, and LFS tracking.

### Line Endings

```gitattributes
# .gitattributes — Normalize line endings

# Default: auto-detect and normalize to LF in the repo
* text=auto eol=lf

# Force LF for source code
*.js    text eol=lf
*.ts    text eol=lf
*.py    text eol=lf
*.rb    text eol=lf
*.go    text eol=lf
*.java  text eol=lf
*.sh    text eol=lf

# Force CRLF for Windows-specific files
*.bat   text eol=crlf
*.cmd   text eol=crlf
*.ps1   text eol=crlf

# Binary files — do not normalize
*.png   binary
*.jpg   binary
*.gif   binary
*.ico   binary
*.zip   binary
*.gz    binary
*.pdf   binary
*.woff  binary
*.woff2 binary
```

### Merge Drivers

```gitattributes
# .gitattributes — Custom merge strategies

# Always keep our version of the lock file during merge conflicts
package-lock.json merge=ours
yarn.lock         merge=ours

# Use union merge for changelogs (combine both sides)
CHANGELOG.md merge=union
```

```bash
# Configure the "ours" merge driver in .gitconfig
git config --global merge.ours.driver true
```

### Diff Drivers

```gitattributes
# .gitattributes — Custom diff drivers

# Use word-level diff for Markdown
*.md diff=markdown

# Custom diff for minified files (show no diff)
*.min.js -diff
*.min.css -diff

# Use custom diff for CSV files
*.csv diff=csv
```

```ini
# ~/.gitconfig — Define custom diff drivers

[diff "csv"]
    textconv = column -t -s ','
```

### Git LFS

[Git LFS](https://git-lfs.com/) (Large File Storage) replaces large files with lightweight pointers in the repository while storing the actual file content on a separate server.

```bash
# Install Git LFS
git lfs install

# Track file types with LFS
git lfs track "*.psd"
git lfs track "*.zip"
git lfs track "*.tar.gz"
git lfs track "*.mp4"
git lfs track "assets/models/**"

# View tracked patterns
git lfs track

# Check LFS status
git lfs status
```

```gitattributes
# .gitattributes — Git LFS tracking (auto-generated by git lfs track)

*.psd filter=lfs diff=lfs merge=lfs -text
*.zip filter=lfs diff=lfs merge=lfs -text
*.tar.gz filter=lfs diff=lfs merge=lfs -text
*.mp4 filter=lfs diff=lfs merge=lfs -text
assets/models/** filter=lfs diff=lfs merge=lfs -text
```

## Git Templates

Git templates provide default files for every new repository created with `git init` or `git clone`. They are useful for distributing default hooks, `.gitignore` patterns, and commit message templates across a team.

### Repository Templates

```bash
# Create a template directory
mkdir -p ~/.git-templates/hooks
mkdir -p ~/.git-templates/info

# Set the global template directory
git config --global init.templateDir '~/.git-templates'

# Every new git init / git clone will copy from this template
```

```
~/.git-templates/
├── hooks/
│   ├── pre-commit         ← Default pre-commit hook for all repos
│   ├── commit-msg         ← Default commit message validation
│   └── pre-push           ← Default pre-push checks
├── info/
│   └── exclude            ← Global ignore patterns (like .gitignore)
└── config                 ← Default config settings
```

### Default Hooks

```bash
#!/usr/bin/env bash
# ~/.git-templates/hooks/pre-commit — Default hook for all repositories

set -euo pipefail

# Check for common mistakes
STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACM)

# Detect secrets (basic patterns)
SECRETS_PATTERN='(AKIA[0-9A-Z]{16}|password\s*=\s*["\x27].+["\x27]|-----BEGIN (RSA |EC )?PRIVATE KEY-----)'
if echo "$STAGED_FILES" | xargs grep -PlE "$SECRETS_PATTERN" 2>/dev/null; then
    echo "ERROR: Potential secrets detected in staged files."
    echo "Remove them before committing, or use a .env file."
    exit 1
fi

# Prevent TODO/FIXME from being committed (optional — uncomment to enable)
# if echo "$STAGED_FILES" | xargs grep -l 'TODO\|FIXME' 2>/dev/null; then
#     echo "WARNING: TODO/FIXME found in staged files."
#     exit 1
# fi
```

```bash
# Apply templates to an existing repository
cd existing-repo
git init   # Re-running git init copies template files without overwriting existing hooks
```

## Best Practices

- ✅ Use a hook management tool (pre-commit, Husky) instead of manually copying scripts
- ✅ Keep hooks fast — slow hooks frustrate developers and get bypassed
- ✅ Run linters and formatters only on staged files, not the entire codebase
- ✅ Use server-side hooks for mandatory policy enforcement
- ✅ Commit your hook configuration (`.pre-commit-config.yaml`, `.husky/`) to the repository
- ✅ Normalize line endings with `.gitattributes` in every repository
- ✅ Use Git LFS for binary and large files instead of committing them directly
- ✅ Set up Git templates for consistent defaults across all repositories
- ✅ Document required hooks and setup steps in your project README
- ✅ Use `git config --global rerere.enabled true` to remember conflict resolutions
- ❌ Do not put long-running tests in `pre-commit` — use `pre-push` or CI instead
- ❌ Do not rely solely on client-side hooks for enforcement — they can be bypassed
- ❌ Do not hardcode paths or environment-specific values in hook scripts
- ❌ Do not skip hooks in CI — run `pre-commit run --all-files` in your pipeline
- ❌ Do not commit `.git/hooks/` — use `.husky/` or `.pre-commit-config.yaml` instead

## Next Steps

Continue to [Monorepos and Submodules](07-MONOREPOS-AND-SUBMODULES.md) to learn about monorepo strategies, Git submodules, subtrees, sparse checkout, and managing multi-project repositories.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial hooks and automation documentation |
