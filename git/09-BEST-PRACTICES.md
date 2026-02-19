# Git Best Practices

## Table of Contents

1. [Overview](#overview)
2. [Repository Setup Best Practices](#repository-setup-best-practices)
3. [Commit Best Practices](#commit-best-practices)
4. [Branching Best Practices](#branching-best-practices)
5. [Code Review Best Practices](#code-review-best-practices)
6. [Merge Strategy Best Practices](#merge-strategy-best-practices)
7. [Team Workflow Conventions](#team-workflow-conventions)
8. [Git Configuration Best Practices](#git-configuration-best-practices)
9. [Performance Best Practices](#performance-best-practices)
10. [Security Best Practices](#security-best-practices)
11. [Production Readiness Checklist](#production-readiness-checklist)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

## Overview

This document compiles the essential best practices for using Git effectively in individual and team workflows. Use it as a reference for setting up repositories, writing commits, managing branches, and collaborating through pull requests — and as a checklist before taking a Git workflow to production.

### Target Audience

- Developers looking to improve their Git workflow
- Tech leads establishing team conventions
- Teams adopting Git for the first time or refining existing practices

### Scope

- Repository setup and configuration
- Commit, branching, and merge best practices
- Code review and team workflow conventions
- Performance, security, and production readiness

## Repository Setup Best Practices

A well-configured repository reduces friction, prevents mistakes, and sets the standard for every contributor.

### Essential Repository Files

| File | Purpose |
|---|---|
| **README.md** | Project overview, setup instructions, usage examples |
| **.gitignore** | Exclude build artifacts, dependencies, and secrets from tracking |
| **.gitattributes** | Normalize line endings, configure diff/merge for specific file types |
| **LICENSE** | Define how the code can be used and distributed |
| **CONTRIBUTING.md** | Explain contribution workflow, coding standards, and PR process |
| **CODEOWNERS** | Automatically assign reviewers based on file paths |

### .gitignore

- ✅ Start with a language-specific template from [github.com/github/gitignore](https://github.com/github/gitignore)
- ✅ Exclude build outputs, dependency directories, and IDE files
- ✅ Exclude environment files and secrets (`.env`, `*.pem`, `*.key`)
- ✅ Include an example environment file (`.env.example`) with placeholder values
- ✅ Set up a global gitignore for OS and editor files that apply to all repositories

```bash
# Set up a global gitignore
git config --global core.excludesfile ~/.gitignore_global

# Common global patterns (OS/editor files — not project-specific):
# .DS_Store
# Thumbs.db
# .idea/
# .vscode/
# *.swp
```

### .gitattributes

```gitattributes
# Normalize line endings across platforms
* text=auto

# Force specific file types to use LF
*.sh   text eol=lf
*.bash text eol=lf
*.yml  text eol=lf
*.yaml text eol=lf

# Force specific file types to use CRLF (Windows scripts)
*.bat  text eol=crlf
*.cmd  text eol=crlf

# Mark binary files to prevent diff/merge corruption
*.png  binary
*.jpg  binary
*.gif  binary
*.ico  binary
*.zip  binary
*.jar  binary
*.pdf  binary

# Use a custom diff driver for lock files
package-lock.json linguist-generated=true
yarn.lock         linguist-generated=true
```

### README

A good README answers three questions immediately: *what is this?*, *how do I set it up?*, and *how do I use it?*

- ✅ Start with a one-sentence project description
- ✅ Include prerequisites (language version, tools, dependencies)
- ✅ Provide step-by-step setup instructions that work on a fresh machine
- ✅ Show basic usage examples
- ✅ Link to detailed documentation, contributing guidelines, and license

### Branch Protection

Protect critical branches (at minimum, `main`) with rules that enforce quality gates.

```
Recommended branch protection for main:

┌──────────────────────────────────────────────────────────────┐
│  Rule                                │ Recommended Setting    │
├──────────────────────────────────────────────────────────────┤
│  Require pull request reviews        │ ✅ At least 1 approval │
│  Dismiss stale reviews on push       │ ✅ Enabled             │
│  Require status checks to pass       │ ✅ CI must pass        │
│  Require branches to be up to date   │ ✅ Enabled             │
│  Require linear history              │ Consider for teams     │
│  Require signed commits              │ Consider for teams     │
│  Allow force pushes                  │ ❌ Disabled            │
│  Allow deletions                     │ ❌ Disabled            │
└──────────────────────────────────────────────────────────────┘
```

## Commit Best Practices

Commits are the permanent record of your project's evolution. Well-crafted commits make debugging, reviewing, and understanding a codebase dramatically easier.

### Atomic Commits

Each commit should represent **one logical change** — a single unit of work that can be understood, reviewed, and reverted independently.

```
✅ Good — atomic commits:

  a1b2c3d  feat: add user registration endpoint
  e4f5g6h  feat: add email validation for registration
  i7j8k9l  test: add unit tests for registration validation

❌ Bad — monolithic commit:

  z9y8x7w  add registration with validation, tests, fix login bug,
           update readme, and refactor database module
```

### Do

- ✅ **Commit one logical change at a time** — if you can describe the commit with "and," it should probably be two commits
- ✅ **Commit working code** — every commit should leave the project in a buildable, testable state
- ✅ **Use `git add -p`** — stage specific hunks to create focused commits from a larger set of changes
- ✅ **Commit early and often** — smaller, frequent commits are easier to review and revert than large, infrequent ones

### Don't

- ❌ **Don't commit half-done features** — use `git stash` or a work-in-progress branch instead
- ❌ **Don't mix formatting changes with logic changes** — separate them into distinct commits
- ❌ **Don't include unrelated fixes** — create a separate commit or branch for unrelated work
- ❌ **Don't commit generated files** — build outputs, compiled assets, and dependency directories belong in `.gitignore`

### Meaningful Commit Messages

A good commit message explains *what* changed and *why* — the diff already shows *how*.

```
Structure:

  <type>: <subject>            ← concise summary (50 chars or less)
                                ← blank line
  <body>                       ← explain what and why (wrap at 72 chars)
                                ← blank line
  <footer>                     ← references, breaking changes
```

```
✅ Good commit messages:

  fix: prevent duplicate order submissions

  Users could submit the same order twice by double-clicking the
  submit button. Added a debounce guard that disables the button
  after the first click and re-enables it if the request fails.

  Closes #1234

---

  feat: add CSV export for transaction reports

  Finance team needs to download transaction data for reconciliation.
  Added a new /api/reports/transactions/export endpoint that streams
  CSV data for the requested date range.

  Refs #567

---

❌ Bad commit messages:

  fix stuff
  WIP
  asdf
  update code
  changes
  fix bug in the thing
```

### Conventional Commits

Conventional commits provide a structured format that enables automated changelog generation, semantic versioning, and consistent history.

```
Format:  <type>(<scope>): <description>

Types:
  feat:     A new feature
  fix:      A bug fix
  docs:     Documentation only changes
  style:    Formatting, missing semicolons (no code change)
  refactor: Code change that neither fixes a bug nor adds a feature
  perf:     Performance improvement
  test:     Adding or correcting tests
  build:    Changes to build system or dependencies
  ci:       Changes to CI configuration
  chore:    Other changes that don't modify src or test files

Examples:
  feat(auth): add OAuth2 login with Google
  fix(cart): correct tax calculation for international orders
  docs(api): update authentication endpoint examples
  refactor(db): extract connection pooling into shared module
  ci: add automated release workflow

Breaking changes:
  feat(api)!: change authentication from API keys to OAuth2

  BREAKING CHANGE: API key authentication is no longer supported.
  All clients must migrate to OAuth2 by 2025-06-01.
```

### Commit Frequency

```
Too infrequent                    Just right                     Too frequent
─────────────────────────────────────────────────────────────────────────────

One giant commit                  One commit per logical          Commit every
at end of day with                change — a feature, a fix,     file save or
1,000+ lines changed              or a meaningful refactor        minor typo fix

Hard to review                    Easy to review                  Noisy history
Hard to revert                    Easy to revert                  No meaningful
Hard to bisect                    Easy to bisect                  context per commit
```

## Branching Best Practices

### Short-Lived Branches

Branches should be short-lived — ideally merged within a few days. Long-lived branches accumulate merge conflicts, diverge from `main`, and are harder to review.

```
✅ Short-lived branch (merged in 2 days):

main ─────●─────●─────●───────●─────── main
           \                 /
            ●───●───●───●──●
            feature/add-search
            (5 focused commits, small diff)

❌ Long-lived branch (open for 3 weeks):

main ─────●─────●─────●─────●─────●─────●─────●─── main
           \                                     /
            ●───●───●───●───●───●───●───●───●──●
            feature/redesign-everything
            (20+ commits, massive diff, merge conflicts)
```

### Naming Conventions

Use a consistent naming convention so anyone can understand a branch's purpose at a glance.

```
Format:  <type>/<ticket-id>-<short-description>

Types:
  feature/    New functionality
  fix/        Bug fixes
  hotfix/     Urgent production fixes
  docs/       Documentation changes
  refactor/   Code restructuring
  test/       Test additions or corrections
  chore/      Maintenance tasks

Examples:
  feature/PROJ-123-user-registration
  fix/PROJ-456-cart-total-rounding
  hotfix/PROJ-789-payment-timeout
  docs/PROJ-101-api-authentication
  refactor/PROJ-202-extract-email-service

Rules:
  ✅ Use lowercase and hyphens
  ✅ Include ticket/issue number when available
  ✅ Keep descriptions short but meaningful
  ❌ Don't use your name (john/my-feature)
  ❌ Don't use vague names (fix/stuff, feature/update)
  ❌ Don't use special characters or spaces
```

### Cleanup Stale Branches

Stale branches clutter the repository and make it hard to find active work. Delete branches after they are merged.

```bash
# Delete a local branch after merge
git branch -d feature/PROJ-123-user-registration

# Delete a remote branch after merge
git push origin --delete feature/PROJ-123-user-registration

# Prune remote-tracking references to deleted remote branches
git fetch --prune

# Auto-prune on every fetch
git config --global fetch.prune true

# List merged branches that can be safely deleted
git branch --merged main | grep -v 'main'

# List branches older than 3 months with no recent commits
git for-each-ref --sort=committerdate --format='%(committerdate:short) %(refname:short)' refs/heads/
```

- ✅ Enable automatic branch deletion after merge (GitHub: Settings → General → Automatically delete head branches)
- ✅ Run periodic audits to identify and remove stale branches
- ✅ Set `fetch.prune = true` globally to clean up remote-tracking references automatically

## Code Review Best Practices

### Pull Request Size

Small pull requests get reviewed faster, receive better feedback, and are less likely to introduce bugs.

```
Lines Changed     Review Quality     Merge Confidence
─────────────────────────────────────────────────────
< 200 lines       ✅ Thorough         ✅ High
200–400 lines     ⚠️ Adequate         ⚠️ Medium
400–800 lines     ❌ Superficial       ❌ Low
800+ lines        ❌ Rubber-stamped    ❌ Very Low
```

- ✅ Aim for pull requests under 200 lines of meaningful change
- ✅ Break large features into a series of smaller, incremental PRs
- ✅ Separate refactoring, formatting, and dependency updates from feature work
- ✅ Use feature flags to merge incomplete features safely

### Pull Request Descriptions

Every PR should explain *what* it does, *why* it's needed, and *how* to verify it.

```markdown
## What

Added CSV export endpoint for transaction reports.

## Why

Finance team needs to download transaction data for monthly
reconciliation. Currently they have to query the database directly.
Closes #567.

## How to Test

1. Start the dev server: `make run`
2. Create some test transactions: `make seed`
3. Call the export endpoint:
   curl http://localhost:8080/api/reports/transactions/export?from=2025-01-01&to=2025-01-31
4. Verify the CSV output contains the expected columns and data.

## Screenshots

(if applicable)

## Checklist

- [x] Tests added
- [x] Documentation updated
- [ ] Migration required (describe in deployment notes)
```

### Review Turnaround

- ✅ Aim to review PRs within **4 working hours** of being requested
- ✅ If you cannot review promptly, communicate a timeline or suggest another reviewer
- ✅ Prioritize small PRs and bug fixes to keep the team unblocked
- ✅ Use "Request changes" only for issues that must be fixed before merge — not for style preferences

### Constructive Feedback

| Instead of | Try |
|---|---|
| "This is wrong." | "This may cause an issue because… Have you considered…?" |
| "Why did you do it this way?" | "What was the reasoning behind this approach? I wonder if X would…" |
| "Fix this." | "Nit: Could we rename this to `processOrder` for clarity?" |
| Silence (no review) | "LGTM — the approach is solid. One minor suggestion on line 42." |

- ✅ **Prefix comments with intent** — `nit:` for minor style issues, `question:` for clarification, `suggestion:` for optional improvements, `blocker:` for issues that must be fixed
- ✅ **Explain the "why"** — don't just say what to change; explain the reason
- ✅ **Offer alternatives** — suggest a specific solution, not just a problem
- ✅ **Acknowledge good work** — call out clever solutions, clean code, or thorough tests
- ❌ **Don't nitpick formatting** — automate style enforcement with linters and formatters
- ❌ **Don't gate PRs on personal preference** — reserve "Request changes" for correctness issues

## Merge Strategy Best Practices

### Choosing the Right Strategy

| Strategy | Best For | History | Result |
|---|---|---|---|
| **Merge commit** | Preserving full branch context | Branch history visible | `Merge branch 'feature/X' into main` |
| **Squash merge** | Clean, linear history from messy branches | Commits collapsed into one | Single commit on `main` |
| **Rebase and merge** | Clean, linear history with individual commits | Commits replayed on `main` | Individual commits, no merge commit |

### When to Use Each

```
Merge Commit (--no-ff):
  ✅ Team wants to see which commits belong to which feature
  ✅ Branch has well-crafted, atomic commits worth preserving
  ✅ You need to revert an entire feature as one unit
  ❌ Creates a non-linear history that can be harder to read

Squash Merge:
  ✅ Branch has messy work-in-progress commits
  ✅ Team prefers a clean, one-commit-per-feature history on main
  ✅ PR description provides sufficient context (it becomes the commit message)
  ❌ Loses individual commit history from the branch
  ❌ Can create very large commits that are hard to bisect

Rebase and Merge:
  ✅ Branch has clean, atomic commits that each make sense on their own
  ✅ Team prefers a linear history without merge commits
  ✅ You want individual commits visible on main
  ❌ Rewrites commit SHAs (breaks references to original commits)
  ❌ Requires a clean, well-organized branch history before merge
```

### Keeping History Clean

- ✅ **Rebase feature branches on `main` before merging** — resolve conflicts in the feature branch, not in `main`
- ✅ **Use interactive rebase to clean up before PR** — squash fixup commits, reorder for logical flow
- ✅ **Write meaningful merge/squash commit messages** — summarize the change, not just "Merge branch X"
- ❌ **Don't rebase shared branches** — only rebase branches that you alone are working on
- ❌ **Don't force-push to `main` or shared branches** — this rewrites history for everyone

```bash
# Rebase your feature branch on the latest main before merging
git checkout feature/PROJ-123-user-registration
git fetch origin
git rebase origin/main

# Interactive rebase to clean up commits before creating a PR
git rebase -i origin/main

# In the editor:
#   pick   a1b2c3d  feat: add registration endpoint
#   squash e4f5g6h  fix typo in registration
#   pick   i7j8k9l  test: add registration tests
```

## Team Workflow Conventions

Effective Git collaboration requires agreed-upon conventions that the whole team follows consistently.

### Agreed Branching Model

Choose one branching model and document it. The right choice depends on your release cadence and team size.

| Model | Best For | Complexity |
|---|---|---|
| **GitHub Flow** | Continuous deployment, small teams | Low |
| **Git Flow** | Scheduled releases, multiple versions in production | High |
| **Trunk-based development** | Continuous integration, experienced teams | Low |
| **GitLab Flow** | Environment-based deployments | Medium |

```
GitHub Flow (recommended for most teams):

  1. main is always deployable
  2. Create a branch from main for every change
  3. Open a pull request early for visibility
  4. Request reviews when ready
  5. Merge to main after approval and CI passes
  6. Deploy from main
```

### Commit Message Standards

Document your team's commit message format and enforce it with tooling.

```bash
# Install commitlint to enforce conventional commits
npm install --save-dev @commitlint/cli @commitlint/config-conventional

# commitlint.config.js
# module.exports = { extends: ['@commitlint/config-conventional'] };

# Add a commit-msg hook (via Husky or Git hooks)
# npx husky add .husky/commit-msg 'npx commitlint --edit $1'
```

### PR Templates

Create a `.github/pull_request_template.md` to standardize PR descriptions.

```markdown
## What does this PR do?

<!-- Describe the change in 1–3 sentences -->

## Why is this change needed?

<!-- Link to issue/ticket. Explain the motivation. -->

## How to test

<!-- Step-by-step instructions for reviewers -->

## Checklist

- [ ] Tests added/updated
- [ ] Documentation updated (if applicable)
- [ ] No breaking changes (or migration guide provided)
- [ ] Reviewed my own code before requesting review
```

### CODEOWNERS

Use a `CODEOWNERS` file to automatically assign the right reviewers based on what files are changed.

```
# .github/CODEOWNERS

# Default — engineering leads review everything
*                         @org/engineering-leads

# Frontend
src/components/           @org/frontend-team
src/styles/               @org/frontend-team

# Backend
src/api/                  @org/backend-team
src/services/             @org/backend-team

# Infrastructure
terraform/                @org/platform-team
.github/workflows/        @org/platform-team

# Documentation
docs/                     @org/docs-team
```

## Git Configuration Best Practices

### Useful Global Configuration

```bash
# Identity (required — set these first)
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"

# Default branch name
git config --global init.defaultBranch main

# Auto-prune deleted remote branches on fetch
git config --global fetch.prune true

# Always rebase on pull instead of merge (keeps history linear)
git config --global pull.rebase true

# Auto-setup remote tracking for new branches
git config --global push.autoSetupRemote true

# Only fast-forward on pull (prevents accidental merge commits)
git config --global pull.ff only

# Show diff in commit message editor
git config --global commit.verbose true

# Set a global gitignore for OS/editor files
git config --global core.excludesfile ~/.gitignore_global

# Use patience diff algorithm for better diff output
git config --global diff.algorithm patience

# Enable rerere (Reuse Recorded Resolution) for repeated conflict resolution
git config --global rerere.enabled true
```

### Aliases

Aliases reduce keystrokes and encode your team's conventions into short commands.

```bash
# Status and log
git config --global alias.st 'status --short --branch'
git config --global alias.lg "log --oneline --graph --decorate --all -20"
git config --global alias.last 'log -1 --stat'

# Branching
git config --global alias.co 'checkout'
git config --global alias.cb 'checkout -b'
git config --global alias.br 'branch -vv'

# Staging and committing
git config --global alias.cm 'commit -m'
git config --global alias.ca 'commit --amend --no-edit'
git config --global alias.unstage 'reset HEAD --'

# Working with remotes
git config --global alias.pushf 'push --force-with-lease'
git config --global alias.sync '!git fetch origin && git rebase origin/main'

# Cleanup
git config --global alias.cleanup '!git branch --merged main | grep -v main | xargs -r git branch -d'
```

### Editor Setup

```bash
# Set your preferred editor for commit messages
git config --global core.editor "vim"
git config --global core.editor "code --wait"      # VS Code
git config --global core.editor "nano"
git config --global core.editor "subl -n -w"        # Sublime Text
```

### Diff and Merge Tools

```bash
# Use a visual diff tool
git config --global diff.tool vscode
git config --global difftool.vscode.cmd 'code --wait --diff $LOCAL $REMOTE'

# Use a visual merge tool
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'

# Don't keep .orig backup files after merge
git config --global mergetool.keepBackup false

# Use delta or diff-so-fancy for better terminal diffs
# (install separately, then configure as pager)
# git config --global core.pager "delta"
# git config --global interactive.diffFilter "delta --color-only"
```

## Performance Best Practices

### .gitignore Optimization

A well-maintained `.gitignore` keeps the repository fast by preventing Git from tracking unnecessary files.

```gitignore
# Dependencies — often the largest directory in a project
node_modules/
vendor/
.venv/
__pycache__/

# Build outputs
dist/
build/
target/
*.o
*.pyc
*.class

# Large generated files
*.min.js
*.min.css
*.map

# Logs and temporary files
*.log
tmp/
.cache/
```

- ✅ Place the most commonly matched patterns near the top of `.gitignore`
- ✅ Use directory patterns (`node_modules/`) rather than recursive globs when possible
- ✅ Add `.gitignore` entries *before* committing files — removing tracked files from history is expensive

### Git LFS for Large Files

Git is not designed for large binary files. Use Git Large File Storage (LFS) for assets like images, videos, datasets, and compiled binaries.

```bash
# Install Git LFS
git lfs install

# Track large file types
git lfs track "*.psd"
git lfs track "*.zip"
git lfs track "*.mp4"
git lfs track "*.bin"
git lfs track "*.model"

# The tracking rules are stored in .gitattributes
cat .gitattributes
# *.psd filter=lfs diff=lfs merge=lfs -text
# *.zip filter=lfs diff=lfs merge=lfs -text

# Commit the .gitattributes file
git add .gitattributes
git commit -m "chore: configure Git LFS for large binary files"

# Verify what LFS is tracking
git lfs ls-files
```

```
When to use Git LFS:

  ✅ Binary files > 1 MB (images, videos, datasets, compiled artifacts)
  ✅ Files that change frequently and produce large diffs
  ✅ Files that don't benefit from text-based diff/merge

  ❌ Text files (source code, config, documentation)
  ❌ Small binary files (icons, favicons)
```

### Shallow Clones and Partial Clones

For large repositories, reduce clone time and disk usage by fetching only what you need.

```bash
# Shallow clone — fetch only the latest commit
git clone --depth 1 https://github.com/org/repo.git

# Shallow clone with limited history
git clone --depth 50 https://github.com/org/repo.git

# Fetch more history later if needed
git fetch --deepen=100

# Convert a shallow clone to a full clone
git fetch --unshallow

# Partial clone — fetch commit and tree objects, but download blobs on demand
git clone --filter=blob:none https://github.com/org/repo.git

# Treeless clone — fetch only commits, download trees and blobs on demand
git clone --filter=tree:0 https://github.com/org/repo.git

# Sparse checkout — check out only specific directories
git clone --filter=blob:none --sparse https://github.com/org/repo.git
cd repo
git sparse-checkout set src/backend docs
```

```
Clone Type           Speed       Disk Usage    Full History
──────────────────────────────────────────────────────────
Full clone           Slow        High          ✅ Yes
Shallow (depth=1)    Fast        Low           ❌ No
Partial (blob:none)  Medium      Medium*       ✅ Yes
Sparse checkout      Fast        Low           ✅ Yes (for selected paths)

* Blobs are downloaded on demand when files are checked out
```

## Security Best Practices

### Never Commit Secrets

Secrets committed to Git persist in history and propagate to every clone and fork. Prevention is the only reliable defense.

- ✅ **Use environment variables or a secrets manager** — never hardcode credentials in source files
- ✅ **Add secret file patterns to `.gitignore`** — `.env`, `*.pem`, `*.key`, `credentials.json`
- ✅ **Install a pre-commit secret scanner** — [git-secrets](https://github.com/awslabs/git-secrets), [gitleaks](https://github.com/gitleaks/gitleaks), or [truffleHog](https://github.com/trufflesecurity/trufflehog)
- ✅ **Run secret scanning in CI** — catch secrets in pull requests before they reach `main`
- ✅ **Enable GitHub push protection** — blocks pushes containing detected secrets

```
Defense-in-depth for secrets:

  Layer 1: .gitignore             Block known secret file patterns
  Layer 2: Pre-commit hooks       Scan staged files before commit
  Layer 3: CI/CD secret scanning  Detect secrets in PRs before merge
  Layer 4: GitHub push protection Block pushes with detected secrets
  Layer 5: Continuous monitoring   Scan history on a schedule
```

### If a Secret Is Committed

```bash
# 1. Immediately rotate the secret — assume it is compromised
# 2. Remove from history using git filter-repo (preferred)
git filter-repo --invert-paths --path .env

# Or using BFG Repo-Cleaner
bfg --delete-files .env
git reflog expire --expire=now --all && git gc --prune=now --aggressive

# 3. Force-push and notify all collaborators
git push origin --force --all

# 4. Invalidate all existing clones (collaborators must re-clone)
```

### Signed Commits

Sign commits to prove authorship and prevent impersonation. See [Security and Signing](08-SECURITY-AND-SIGNING.md) for detailed setup instructions.

```bash
# Enable commit signing globally
git config --global commit.gpgsign true

# Use SSH signing (simpler, Git 2.34+)
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub

# Or use GPG signing (traditional)
git config --global user.signingkey YOUR_GPG_KEY_ID
```

- ✅ Sign all commits on projects that require audit trails or compliance
- ✅ Upload your signing key to GitHub so commits display as "Verified"
- ✅ Enable GitHub vigilant mode to flag unsigned commits

### Credential Helpers

- ✅ Use your OS keychain for credential storage (`osxkeychain`, `manager`, `libsecret`)
- ✅ Prefer SSH keys or fine-grained personal access tokens over classic PATs
- ❌ Never use `credential.helper store` — it saves passwords as plaintext

### Audit Logs

```bash
# Review who changed what and when
git log --pretty=format:'%h %ai %an <%ae> %s' --since='2025-01-01'

# Show signature verification status
git log --pretty=format:'%h %G? %an %s'
#   G = Good signature
#   B = Bad signature
#   N = No signature

# Show all changes to a specific file
git log --follow --all -- path/to/sensitive-file.conf

# Export audit log
git log --pretty=format:'"%h","%ai","%an","%ae","%G?","%s"' > audit.csv
```

## Production Readiness Checklist

Use this checklist before adopting a Git workflow for a production project or team.

### Repository Setup

- [ ] **README.md** exists with setup instructions and project overview
- [ ] **.gitignore** excludes build artifacts, dependencies, secrets, and OS/editor files
- [ ] **.gitattributes** normalizes line endings and marks binary files
- [ ] **LICENSE** file is present and appropriate for the project
- [ ] **CONTRIBUTING.md** documents the contribution workflow and standards
- [ ] **.github/CODEOWNERS** assigns reviewers for critical paths

### Branch Protection

- [ ] **`main` branch is protected** — no direct pushes
- [ ] **Pull request reviews required** — at least one approval before merge
- [ ] **Status checks required** — CI must pass before merge
- [ ] **Stale reviews are dismissed** on new pushes
- [ ] **Force pushes are disabled** on protected branches
- [ ] **Branch deletion is disabled** on protected branches

### Commit and Branch Standards

- [ ] **Commit message format is documented** and enforced (conventional commits or team standard)
- [ ] **Branch naming convention is documented** and followed
- [ ] **Branches are short-lived** — merged within days, not weeks
- [ ] **Stale branches are cleaned up** regularly

### Code Review

- [ ] **PR template exists** (`.github/pull_request_template.md`)
- [ ] **PR size guidelines are documented** (target under 200 lines)
- [ ] **Review turnaround expectation is set** (e.g., within 4 hours)
- [ ] **Review feedback guidelines are documented**

### Merge Strategy

- [ ] **Team has agreed on a merge strategy** (merge, squash, or rebase)
- [ ] **Merge strategy is configured** in the repository settings
- [ ] **Feature branches are rebased** on `main` before merge

### Security

- [ ] **Pre-commit secret scanning is installed** (git-secrets or gitleaks)
- [ ] **CI runs secret scanning** on pull requests
- [ ] **GitHub push protection is enabled**
- [ ] **Commit signing is configured** (for projects requiring audit trails)
- [ ] **Credential storage uses OS keychain** — not plaintext

### Performance

- [ ] **Git LFS is configured** for large binary files
- [ ] **.gitignore is comprehensive** — no tracked files that should be ignored
- [ ] **CI uses shallow or partial clones** where appropriate

### Automation

- [ ] **CI/CD pipeline runs on every PR** (build, test, lint)
- [ ] **Branches are auto-deleted after merge**
- [ ] **Commit message linting is automated** (commitlint or similar)
- [ ] **Code formatting is automated** (pre-commit hooks or CI checks)

## Next Steps

Continue to [Anti-Patterns](10-ANTI-PATTERNS.md) to learn about the most common Git mistakes and how to avoid them, or revisit [Security and Signing](08-SECURITY-AND-SIGNING.md) for detailed instructions on commit signing and secrets management.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Git best practices documentation |
