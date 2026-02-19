# Git Remote Collaboration

## Table of Contents

1. [Overview](#overview)
2. [Git Remotes](#git-remotes)
   - [Adding Remotes](#adding-remotes)
   - [Removing and Renaming Remotes](#removing-and-renaming-remotes)
   - [The Origin Convention](#the-origin-convention)
   - [Multiple Remotes](#multiple-remotes)
3. [Fetch, Pull, and Push](#fetch-pull-and-push)
   - [Git Fetch](#git-fetch)
   - [Git Pull](#git-pull)
   - [Fetch vs. Pull](#fetch-vs-pull)
   - [Git Push](#git-push)
   - [Tracking Branches and Upstream Configuration](#tracking-branches-and-upstream-configuration)
4. [Pull Requests / Merge Requests](#pull-requests--merge-requests)
   - [PR Workflow](#pr-workflow)
   - [Creating Effective PRs](#creating-effective-prs)
   - [PR Description Templates](#pr-description-templates)
5. [Code Review Best Practices](#code-review-best-practices)
   - [What to Look For](#what-to-look-for)
   - [Constructive Feedback](#constructive-feedback)
   - [Review Checklists](#review-checklists)
   - [Approval Workflows](#approval-workflows)
6. [Forking Workflow](#forking-workflow)
   - [Fork and Clone](#fork-and-clone)
   - [Keeping Your Fork Synced](#keeping-your-fork-synced)
   - [Contributing to Open Source](#contributing-to-open-source)
7. [Collaboration Models](#collaboration-models)
   - [Shared Repository Model](#shared-repository-model)
   - [Fork-and-Pull Model](#fork-and-pull-model)
   - [Model Comparison](#model-comparison)
8. [Handling Remote Conflicts](#handling-remote-conflicts)
   - [Push Rejection](#push-rejection)
   - [Force Push Dangers](#force-push-dangers)
   - [Safe Force Push with --force-with-lease](#safe-force-push-with---force-with-lease)
9. [Git Fetch and Prune](#git-fetch-and-prune)
   - [Stale Remote-Tracking Branches](#stale-remote-tracking-branches)
   - [Pruning Workflows](#pruning-workflows)
10. [Working with Multiple Remotes](#working-with-multiple-remotes)
    - [Upstream Tracking](#upstream-tracking)
    - [Triangle Workflows](#triangle-workflows)
11. [Best Practices](#best-practices)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

## Overview

This document covers how to collaborate with others using Git remote repositories. Remotes are the mechanism that transforms Git from a local version control tool into a powerful distributed collaboration platform — enabling teams to share work, review code, and contribute to projects across organizational boundaries.

### Target Audience

- Developers collaborating on shared repositories with teammates
- Open-source contributors forking and submitting pull requests
- Tech leads defining collaboration workflows and code review policies
- DevOps engineers configuring remote repository access and branch protections

### Scope

- Managing remote repositories (add, remove, rename, inspect)
- Fetch, pull, and push workflows with tracking branches
- Pull request and merge request workflows
- Code review practices and approval gates
- Forking workflows for open-source contribution
- Handling push rejections and force push safety
- Working with multiple remotes and triangle workflows

## Git Remotes

A **remote** is a named reference to another copy of the repository, typically hosted on a server like GitHub, GitLab, or Bitbucket. Remotes allow you to fetch changes from and push changes to other repositories.

```
Local Repository                              Remote Repositories
┌──────────────────────┐
│  .git/               │       origin (primary)
│  ├── objects/         │      ┌──────────────────────────┐
│  ├── refs/            │      │  github.com/you/repo.git │
│  │   ├── heads/      │◄────▶│                          │
│  │   │   ├── main    │      │  main, feature/auth      │
│  │   │   └── feature │      └──────────────────────────┘
│  │   └── remotes/    │
│  │       ├── origin/ │       upstream (original project)
│  │       │   └── main│      ┌──────────────────────────┐
│  │       └── upstream│◄────▶│  github.com/org/repo.git │
│  │           └── main│      │                          │
│  └── config          │      │  main, develop           │
└──────────────────────┘      └──────────────────────────┘
```

### Adding Remotes

```bash
# View existing remotes
git remote -v

# Output:
# origin    https://github.com/you/project.git (fetch)
# origin    https://github.com/you/project.git (push)

# Add a new remote
git remote add upstream https://github.com/original-org/project.git

# Add a remote using SSH
git remote add origin git@github.com:you/project.git

# Verify remotes after adding
git remote -v

# Output:
# origin    https://github.com/you/project.git (fetch)
# origin    https://github.com/you/project.git (push)
# upstream  https://github.com/original-org/project.git (fetch)
# upstream  https://github.com/original-org/project.git (push)

# Inspect details of a specific remote
git remote show origin

# Output:
# * remote origin
#   Fetch URL: https://github.com/you/project.git
#   Push  URL: https://github.com/you/project.git
#   HEAD branch: main
#   Remote branches:
#     main        tracked
#     feature/auth tracked
#   Local branches configured for 'git pull':
#     main merges with remote main
#   Local refs configured for 'git push':
#     main pushes to main (up to date)
```

### Removing and Renaming Remotes

```bash
# Rename a remote
git remote rename origin old-origin

# Remove a remote (also removes all remote-tracking branches for it)
git remote remove old-origin

# Change a remote's URL (e.g., switch from HTTPS to SSH)
git remote set-url origin git@github.com:you/project.git

# Set different URLs for fetch and push
git remote set-url --push origin git@github.com:you/project.git
```

### The Origin Convention

When you clone a repository, Git automatically creates a remote called `origin` pointing to the URL you cloned from. This is a **convention**, not a requirement — you can rename it or add others.

```bash
# Cloning automatically sets up origin
git clone https://github.com/org/project.git
cd project

git remote -v
# origin    https://github.com/org/project.git (fetch)
# origin    https://github.com/org/project.git (push)
```

| Convention | Remote Name | Typical URL |
|---|---|---|
| **Your copy** (clone source) | `origin` | `github.com/you/project.git` |
| **Original project** (upstream source) | `upstream` | `github.com/org/project.git` |
| **Colleague's fork** | `alice` or `teammate` | `github.com/alice/project.git` |

### Multiple Remotes

Working with multiple remotes is common in open-source workflows where you maintain a fork alongside the original repository.

```
Multiple Remotes Setup:

  upstream (original project)          origin (your fork)
  ┌─────────────────────┐             ┌─────────────────────┐
  │ org/project.git     │             │ you/project.git     │
  │                     │    fork     │                     │
  │ main ───────────────│────────────▶│ main                │
  │ develop             │             │ feature/my-change   │
  └─────────┬───────────┘             └──────────┬──────────┘
            │                                    │
            │              fetch                 │   push / fetch
            │                                    │
            └──────────┐    ┌────────────────────┘
                       │    │
                       ▼    ▼
              ┌──────────────────────┐
              │  Local Repository    │
              │                      │
              │  remotes/upstream/*  │
              │  remotes/origin/*    │
              │  main, feature/*    │
              └──────────────────────┘
```

```bash
# Typical multi-remote setup for open-source contribution
git clone https://github.com/you/project.git
cd project
git remote add upstream https://github.com/org/project.git

# Fetch from both remotes
git fetch origin
git fetch upstream

# Or fetch from all remotes at once
git fetch --all
```

## Fetch, Pull, and Push

### Git Fetch

`git fetch` downloads commits, files, and refs from a remote repository into your local repo. It updates your **remote-tracking branches** (e.g., `origin/main`) but does **not** modify your working directory or local branches.

```bash
# Fetch from the default remote (origin)
git fetch

# Fetch from a specific remote
git fetch upstream

# Fetch a specific branch
git fetch origin main

# Fetch all remotes
git fetch --all

# Fetch and show what changed
git fetch --verbose
```

```
Before git fetch:

  Local                              Remote (origin)
  ┌──────────┐                       ┌──────────┐
  │ main: C  │                       │ main: E  │
  │          │                       │          │
  │ origin/  │                       │ A◄B◄C◄D◄E│
  │ main: C  │                       └──────────┘
  │          │
  │ A ◄ B ◄ C│
  └──────────┘

After git fetch:

  Local                              Remote (origin)
  ┌──────────────┐                   ┌──────────┐
  │ main: C      │  (unchanged!)     │ main: E  │
  │              │                   │          │
  │ origin/      │                   │ A◄B◄C◄D◄E│
  │ main: E      │  (updated)       └──────────┘
  │              │
  │ A ◄ B ◄ C   │
  │         │   │
  │         D◄E │  (new objects downloaded)
  └──────────────┘

  Your local 'main' still points to C.
  Only origin/main moved to E.
```

### Git Pull

`git pull` is a convenience command that performs `git fetch` followed by `git merge` (or `git rebase` if configured). It updates your local branch with changes from the remote.

```bash
# Pull with merge (default)
git pull origin main
# Equivalent to:
#   git fetch origin main
#   git merge origin/main

# Pull with rebase instead of merge
git pull --rebase origin main
# Equivalent to:
#   git fetch origin main
#   git rebase origin/main

# Configure pull to always rebase (recommended)
git config --global pull.rebase true

# Pull from the tracked upstream branch
git pull
```

```
git pull --rebase workflow:

  Before:                            After git pull --rebase:

  origin/main: A ◄ B ◄ D ◄ E        origin/main: A ◄ B ◄ D ◄ E

  local main:  A ◄ B ◄ C            local main:  A ◄ B ◄ D ◄ E ◄ C'

  Your commit C is replayed on top of E as C'.
  Result is a clean, linear history.


git pull (merge) workflow:

  Before:                            After git pull (merge):

  origin/main: A ◄ B ◄ D ◄ E        origin/main: A ◄ B ◄ D ◄ E

  local main:  A ◄ B ◄ C            local main:  A ◄ B ◄ C ◄─── M
                                                       │         │
                                                       D ◄ E ◄───┘

  A merge commit M is created to combine C and E.
```

### Fetch vs. Pull

| Aspect | `git fetch` | `git pull` |
|---|---|---|
| **Downloads new data** | ✅ Yes | ✅ Yes (calls fetch internally) |
| **Updates remote-tracking branches** | ✅ Yes | ✅ Yes |
| **Modifies local branches** | ❌ No | ✅ Yes (merge or rebase) |
| **Modifies working directory** | ❌ No | ✅ Yes |
| **Risk of conflicts** | None | Can produce merge conflicts |
| **Recommended for** | Inspecting changes before integrating | Quick sync when you know changes are safe |

> **Tip:** Prefer `git fetch` + `git rebase` (or `git merge`) over `git pull` when you want to inspect remote changes before integrating them.

### Git Push

`git push` uploads your local commits to a remote repository, updating the remote branch to match your local branch.

```bash
# Push to the default upstream branch
git push

# Push a specific branch to a specific remote
git push origin feature/auth

# Push and set the upstream tracking reference
git push -u origin feature/auth
# After this, 'git push' and 'git pull' work without specifying remote/branch

# Push all local branches
git push --all origin

# Push tags
git push origin --tags

# Push a single tag
git push origin v1.0.0

# Delete a remote branch
git push origin --delete feature/old-branch
```

```
Push workflow:

  Local                              Remote (origin)
  ┌──────────────┐    git push       ┌──────────────┐
  │ main: E      │ ─────────────▶    │ main: E      │
  │              │                   │              │
  │ A ◄ B ◄ C   │                   │ A ◄ B ◄ C   │
  │         │   │                   │         │   │
  │         D◄E │                   │         D◄E │
  └──────────────┘                   └──────────────┘

  Local commits D and E are uploaded to origin.
  Remote main now matches local main.
```

### Tracking Branches and Upstream Configuration

A **tracking branch** (or upstream branch) is a local branch that has a direct relationship with a remote branch. This relationship allows `git pull` and `git push` to work without specifying the remote and branch name each time.

```bash
# Set upstream when pushing a new branch
git push -u origin feature/auth
# -u is shorthand for --set-upstream

# Set upstream for an existing branch
git branch --set-upstream-to=origin/main main

# View tracking configuration for all branches
git branch -vv

# Output:
#   feature/auth a1b2c3d [origin/feature/auth] Add login endpoint
# * main         d4e5f6a [origin/main] Merge pull request #42
#   develop      b7c8d9e [origin/develop: behind 3] Update dependencies

# Remove upstream tracking
git branch --unset-upstream feature/auth
```

| Status | Meaning |
|---|---|
| `[origin/main]` | Up to date with remote |
| `[origin/main: ahead 2]` | 2 local commits not yet pushed |
| `[origin/main: behind 3]` | 3 remote commits not yet pulled |
| `[origin/main: ahead 2, behind 3]` | Diverged — local and remote both have new commits |

## Pull Requests / Merge Requests

A **pull request** (GitHub, Bitbucket) or **merge request** (GitLab) is a request to merge changes from one branch into another. PRs are the primary mechanism for code review and collaboration in modern Git workflows.

### PR Workflow

```
Pull Request Lifecycle:

  1. Create Branch          2. Make Changes           3. Push Branch
  ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
  │ git checkout -b  │      │ Edit, commit    │      │ git push -u     │
  │ feature/auth     │─────▶│ locally         │─────▶│ origin          │
  └─────────────────┘      └─────────────────┘      │ feature/auth    │
                                                     └────────┬────────┘
                                                              │
  6. Merge PR               5. Approve & CI          4. Open PR
  ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
  │ Merge into main │◀─────│ Reviewers approve│◀─────│ Create PR on    │
  │ Delete branch   │      │ CI checks pass  │      │ GitHub/GitLab   │
  └─────────────────┘      └─────────────────┘      └─────────────────┘
```

```bash
# 1. Create a feature branch
git checkout -b feature/auth

# 2. Make changes and commit
git add src/auth/login.py
git commit -m "feat(auth): add JWT login endpoint"

# 3. Push to remote with upstream tracking
git push -u origin feature/auth

# 4. Open a PR via the web UI or CLI
gh pr create --title "feat(auth): add JWT login" \
  --body "Adds JWT-based authentication" \
  --base main

# 5. Address review feedback (push additional commits)
git add src/auth/login.py
git commit -m "fix: address review feedback on error handling"
git push

# 6. After approval, merge via the web UI or CLI
gh pr merge --squash
```

### Creating Effective PRs

A well-crafted PR makes reviewers' lives easier and speeds up the review cycle.

| Principle | Description |
|---|---|
| **Keep PRs small** | 200–400 lines of diff is ideal. Large PRs are harder to review and more likely to contain bugs |
| **Single responsibility** | Each PR should address one concern — a feature, a bug fix, or a refactor |
| **Descriptive title** | Use a conventional commit prefix: `feat:`, `fix:`, `refactor:`, `docs:` |
| **Explain the "why"** | The diff shows *what* changed — the description should explain *why* |
| **Link related issues** | Reference issues with `Closes #123` or `Relates to #456` |
| **Include screenshots** | For UI changes, include before/after screenshots |
| **Self-review first** | Review your own diff before requesting reviews from others |

### PR Description Templates

Most platforms support PR templates that standardize descriptions across the team.

```markdown
## Description

Brief description of what this PR does and why.

## Type of Change

- [ ] Bug fix (non-breaking change that fixes an issue)
- [ ] New feature (non-breaking change that adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to change)
- [ ] Refactoring (no functional changes)
- [ ] Documentation update

## How Has This Been Tested?

Describe the tests you ran to verify your changes.

- [ ] Unit tests
- [ ] Integration tests
- [ ] Manual testing

## Checklist

- [ ] My code follows the project's style guidelines
- [ ] I have performed a self-review of my code
- [ ] I have added tests that prove my fix/feature works
- [ ] New and existing unit tests pass locally
- [ ] I have updated the documentation accordingly

## Related Issues

Closes #(issue number)
```

> **Tip:** Place this template in `.github/PULL_REQUEST_TEMPLATE.md` (GitHub) or `.gitlab/merge_request_templates/Default.md` (GitLab) to auto-populate every new PR.

## Code Review Best Practices

Code review is one of the most effective quality assurance practices in software development. It catches bugs, enforces standards, shares knowledge, and improves code quality across the team.

### What to Look For

```
Code Review Focus Areas:

  ┌──────────────────────────────────────────────────────────┐
  │                     Code Review                          │
  │                                                          │
  │  ┌────────────┐  ┌────────────┐  ┌────────────────────┐ │
  │  │ Correctness│  │ Design     │  │ Readability        │ │
  │  │            │  │            │  │                    │ │
  │  │ • Logic    │  │ • SOLID    │  │ • Naming           │ │
  │  │ • Edge     │  │ • DRY     │  │ • Comments         │ │
  │  │   cases    │  │ • Patterns │  │ • Code structure   │ │
  │  │ • Error    │  │ • API      │  │ • Complexity       │ │
  │  │   handling │  │   design   │  │                    │ │
  │  └────────────┘  └────────────┘  └────────────────────┘ │
  │                                                          │
  │  ┌────────────┐  ┌────────────┐  ┌────────────────────┐ │
  │  │ Security   │  │ Testing    │  │ Performance        │ │
  │  │            │  │            │  │                    │ │
  │  │ • Input    │  │ • Coverage │  │ • Algorithmic      │ │
  │  │   validation│  │ • Edge    │  │   complexity       │ │
  │  │ • Auth     │  │   cases   │  │ • N+1 queries      │ │
  │  │ • Secrets  │  │ • Mocking │  │ • Memory usage     │ │
  │  └────────────┘  └────────────┘  └────────────────────┘ │
  └──────────────────────────────────────────────────────────┘
```

| Category | What to Check |
|---|---|
| **Correctness** | Does the code do what it claims? Are edge cases handled? |
| **Design** | Is the code well-structured? Does it follow project patterns? |
| **Readability** | Can you understand the code without the author explaining it? |
| **Security** | Are inputs validated? Are secrets kept out of code? |
| **Testing** | Are there tests? Do they cover the important cases? |
| **Performance** | Are there obvious inefficiencies? N+1 queries? Unbounded loops? |

### Constructive Feedback

Effective code review feedback is specific, actionable, and respectful.

| ❌ Unhelpful | ✅ Constructive |
|---|---|
| "This is wrong." | "This will throw a NullPointerException if `user` is null. Consider adding a null check on line 42." |
| "Why did you do it this way?" | "I think using a HashMap here instead of a linear scan would improve lookup performance from O(n) to O(1). What do you think?" |
| "This code is messy." | "This method is doing three things: validation, transformation, and persistence. Splitting it into three methods would improve readability." |
| "Fix this." | "Nit: this variable name `x` could be more descriptive — maybe `retryCount`?" |

**Guidelines for reviewers:**

- ✅ Praise good code — not just problems ("Nice use of the builder pattern here!")
- ✅ Ask questions instead of making demands ("Have you considered...?")
- ✅ Distinguish between blocking issues and nits (prefix with `nit:` for style suggestions)
- ✅ Suggest alternatives with code examples when possible
- ❌ Do not nitpick formatting issues that a linter should catch
- ❌ Do not gatekeep based on personal style preferences
- ❌ Do not approve without actually reading the code

### Review Checklists

```
Reviewer Checklist:

├── Functionality
│   ├── Does the code work as described in the PR description?
│   ├── Are error cases handled gracefully?
│   └── Are there any unintended side effects?
│
├── Code Quality
│   ├── Is the code DRY (no unnecessary duplication)?
│   ├── Are functions/methods focused on a single responsibility?
│   └── Are variable and function names descriptive?
│
├── Testing
│   ├── Are new features covered by tests?
│   ├── Are edge cases tested?
│   └── Do existing tests still pass?
│
├── Security
│   ├── Is user input validated and sanitized?
│   ├── Are there any hardcoded secrets or credentials?
│   └── Are authorization checks in place?
│
├── Documentation
│   ├── Are public APIs documented?
│   ├── Is the README updated if needed?
│   └── Are complex algorithms explained with comments?
│
└── Operations
    ├── Are database migrations backward-compatible?
    ├── Are new environment variables documented?
    └── Will this change require a deployment change?
```

### Approval Workflows

Different teams require different levels of review rigor based on the criticality of the codebase.

| Workflow | Required Approvals | Use Case |
|---|---|---|
| **Single approval** | 1 reviewer | Small teams, fast iteration |
| **Two approvals** | 2 reviewers | Standard for most production code |
| **CODEOWNERS** | Approval from code owner of each changed path | Large repos with domain-specific expertise |
| **Lead approval** | At least 1 senior/lead engineer | Critical paths (auth, payments, infrastructure) |
| **No approval needed** | 0 (CI only) | Documentation, typo fixes, config changes |

```bash
# Example GitHub branch protection rule (via gh CLI)
# Require 2 approvals, dismiss stale reviews, require CI to pass
gh api repos/{owner}/{repo}/branches/main/protection --method PUT \
  --field required_pull_request_reviews='{"required_approving_review_count":2,"dismiss_stale_reviews":true}' \
  --field required_status_checks='{"strict":true,"contexts":["ci/build","ci/test"]}'
```

## Forking Workflow

The forking workflow is the standard model for contributing to open-source projects. Instead of pushing branches to the original repository, each contributor works on their own copy (fork) and submits pull requests.

### Fork and Clone

```
Forking Workflow:

  Step 1: Fork on GitHub
  ┌─────────────────────┐       Fork        ┌─────────────────────┐
  │ org/project         │ ─────────────────▶ │ you/project         │
  │ (upstream)          │                    │ (origin)            │
  │                     │                    │                     │
  │ main ────────       │                    │ main ────────       │
  └─────────────────────┘                    └──────────┬──────────┘
                                                        │
  Step 2: Clone your fork locally                       │ clone
                                                        │
                                             ┌──────────▼──────────┐
                                             │ Local Repository    │
                                             │                     │
                                             │ origin → you/project│
                                             │ main ────────       │
                                             └─────────────────────┘

  Step 3: Add upstream remote
                                             ┌─────────────────────┐
                                             │ Local Repository    │
                                             │                     │
                                             │ origin   → you/     │
                                             │ upstream → org/     │
                                             └─────────────────────┘
```

```bash
# 1. Fork the repository on GitHub (via web UI or CLI)
gh repo fork org/project --clone

# Or manually:
# Fork via GitHub web UI, then:
git clone https://github.com/you/project.git
cd project

# 2. Add the original repository as upstream
git remote add upstream https://github.com/org/project.git

# 3. Verify remotes
git remote -v
# origin    https://github.com/you/project.git (fetch)
# origin    https://github.com/you/project.git (push)
# upstream  https://github.com/org/project.git (fetch)
# upstream  https://github.com/org/project.git (push)
```

### Keeping Your Fork Synced

Your fork does not automatically receive updates from the upstream repository. You must manually sync it.

```bash
# Fetch latest changes from upstream
git fetch upstream

# Switch to your main branch
git checkout main

# Merge upstream changes into your local main
git merge upstream/main

# Or rebase (for a cleaner history)
git rebase upstream/main

# Push the updated main to your fork
git push origin main
```

```
Syncing your fork:

  upstream/main:  A ◄ B ◄ C ◄ D ◄ E     (upstream has new commits D, E)

  origin/main:    A ◄ B ◄ C              (your fork is behind)

  local main:     A ◄ B ◄ C              (your local is behind)

  After git fetch upstream && git merge upstream/main && git push origin main:

  upstream/main:  A ◄ B ◄ C ◄ D ◄ E
  origin/main:    A ◄ B ◄ C ◄ D ◄ E     (fork is now synced)
  local main:     A ◄ B ◄ C ◄ D ◄ E     (local is now synced)
```

### Contributing to Open Source

```
Open-Source Contribution Flow:

  1. Fork & clone          2. Branch & code         3. Push to fork
  ┌───────────────┐       ┌───────────────┐       ┌───────────────┐
  │ Fork repo     │       │ Create branch │       │ Push branch   │
  │ Clone locally │──────▶│ Make changes  │──────▶│ to origin     │
  │ Add upstream  │       │ Commit        │       │               │
  └───────────────┘       └───────────────┘       └──────┬────────┘
                                                         │
  6. Merge                 5. Iterate                4. Open PR
  ┌───────────────┐       ┌───────────────┐       ┌──────▼────────┐
  │ Maintainer    │       │ Address review │       │ PR from       │
  │ merges PR     │◀──────│ feedback       │◀──────│ you/project   │
  │ into upstream │       │ Push updates   │       │ → org/project │
  └───────────────┘       └───────────────┘       └───────────────┘
```

```bash
# Complete open-source contribution workflow

# 1. Fork and clone
gh repo fork org/project --clone
cd project
git remote add upstream https://github.com/org/project.git

# 2. Sync with upstream before starting work
git fetch upstream
git checkout main
git rebase upstream/main

# 3. Create a feature branch
git checkout -b fix/typo-in-readme

# 4. Make changes and commit
git add README.md
git commit -m "docs: fix typo in installation instructions"

# 5. Push to your fork
git push -u origin fix/typo-in-readme

# 6. Open a pull request against the upstream repository
gh pr create --repo org/project \
  --title "docs: fix typo in installation instructions" \
  --body "Fixed a typo in the README installation section."

# 7. After the PR is merged, clean up
git checkout main
git fetch upstream
git rebase upstream/main
git push origin main
git branch -d fix/typo-in-readme
git push origin --delete fix/typo-in-readme
```

## Collaboration Models

### Shared Repository Model

In the shared repository model, all collaborators have push access to a single repository. Developers create branches directly in the shared repo and open PRs to merge into the main branch.

```
Shared Repository Model:

  ┌─────────────────────────────────────────────┐
  │           org/project (shared)              │
  │                                             │
  │  main ◄──────────── merge ◄── PR            │
  │                                │             │
  │  feature/auth ─────────────────┘  (Alice)   │
  │  feature/search ──────────────── PR (Bob)   │
  │  fix/login-bug ───────────────── PR (Carol) │
  └─────────────────────────────────────────────┘

  All developers push branches to the same repository.
  PRs are opened within the same repo (branch → main).
```

**Characteristics:**

- ✅ Simple setup — everyone clones the same repo
- ✅ Easy to see all active branches
- ✅ No fork syncing required
- ❌ Requires granting push access to all collaborators
- ❌ Branch namespace can get cluttered

### Fork-and-Pull Model

In the fork-and-pull model, each contributor forks the repository and pushes changes to their own copy. PRs are opened from the fork to the upstream repository.

```
Fork-and-Pull Model:

  ┌─────────────────────┐
  │  org/project        │◀──── PR ── alice/project (Alice's fork)
  │  (upstream)         │◀──── PR ── bob/project   (Bob's fork)
  │                     │◀──── PR ── carol/project (Carol's fork)
  │  main               │
  └─────────────────────┘

  Each contributor has their own fork.
  PRs cross repository boundaries (fork → upstream).
```

**Characteristics:**

- ✅ No push access needed to the upstream repository
- ✅ Each contributor has a personal sandbox
- ✅ Standard model for open-source projects
- ❌ Requires keeping forks synced with upstream
- ❌ More complex remote setup (origin + upstream)

### Model Comparison

| Factor | Shared Repository | Fork-and-Pull |
|---|---|---|
| **Push access required** | Yes — all collaborators | No — only maintainers |
| **Typical use** | Private team projects, companies | Open-source projects, external contributions |
| **Setup complexity** | Low — single remote | Medium — fork + upstream remote |
| **Branch visibility** | All branches in one repo | Branches are in individual forks |
| **PR source** | Same repo (branch → branch) | Cross-repo (fork → upstream) |
| **Keeping in sync** | `git pull` | `git fetch upstream && git rebase` |
| **Access control** | Branch protection rules | Fork permissions + PR reviews |
| **Scalability** | Dozens of contributors | Thousands of contributors |

## Handling Remote Conflicts

### Push Rejection

A push is rejected when the remote branch has commits that your local branch does not have. This means someone else pushed to the same branch since your last fetch.

```
Push rejection scenario:

  You:     A ◄ B ◄ C ◄ D     (local main, you created D)
  Remote:  A ◄ B ◄ C ◄ E     (origin/main, someone else pushed E)

  git push origin main
  # ! [rejected]        main -> main (non-fast-forward)
  # error: failed to push some refs to 'origin'
  # hint: Updates were rejected because the tip of your current branch
  # hint: is behind its remote counterpart.
```

**Resolution:**

```bash
# Option 1: Fetch and rebase (preferred — linear history)
git fetch origin
git rebase origin/main
# Resolve any conflicts, then:
git push origin main

# Option 2: Fetch and merge
git fetch origin
git merge origin/main
# Resolve any conflicts, then:
git push origin main

# Option 3: Pull with rebase (shorthand for fetch + rebase)
git pull --rebase origin main
git push origin main
```

```
After fetch and rebase:

  Before rebase:
    local:   A ◄ B ◄ C ◄ D
    remote:  A ◄ B ◄ C ◄ E

  After git rebase origin/main:
    local:   A ◄ B ◄ C ◄ E ◄ D'     (D replayed on top of E)

  git push succeeds — fast-forward to D'.
```

### Force Push Dangers

`git push --force` overwrites the remote branch unconditionally, discarding any commits on the remote that are not in your local branch. **This can permanently destroy other people's work.**

```
Force push disaster:

  Alice pushes commit E:
    remote:  A ◄ B ◄ C ◄ E

  Bob force-pushes, not knowing about E:
    Bob's local:  A ◄ B ◄ C ◄ D
    git push --force origin main

  Remote after Bob's force push:
    remote:  A ◄ B ◄ C ◄ D       ← Alice's commit E is GONE

  Alice pulls and her commit has vanished.
  If no one has a reflog entry for E, it is lost.
```

- ❌ **Never** use `git push --force` on shared branches (`main`, `develop`, release branches)
- ❌ **Never** force push without communicating with your team
- ✅ Force push is acceptable on personal feature branches that only you work on

### Safe Force Push with --force-with-lease

`--force-with-lease` is a safer alternative that fails if the remote branch has been updated since your last fetch. It prevents you from accidentally overwriting someone else's work.

```bash
# Safe force push — fails if remote has unexpected commits
git push --force-with-lease origin feature/auth

# How it works:
# 1. Git checks: "Is origin/feature/auth still where I think it is?"
# 2. If yes → force push succeeds
# 3. If no  → push is rejected (someone else pushed in the meantime)

# Even safer: specify the expected remote ref
git push --force-with-lease=feature/auth:abc1234 origin feature/auth
```

```
--force-with-lease protection:

  Scenario: You rebased your feature branch and need to force push.

  Your expectation:    origin/feature/auth → C
  Actual remote:       origin/feature/auth → C     ← matches!
  Result: Force push succeeds ✅

  Your expectation:    origin/feature/auth → C
  Actual remote:       origin/feature/auth → F     ← someone pushed F!
  Result: Push rejected ❌ (safe — you didn't lose F)
```

| Command | Safety | Use Case |
|---|---|---|
| `git push` | ✅ Safe | Normal push (fast-forward only) |
| `git push --force-with-lease` | ⚠️ Mostly safe | After rebase on personal branches |
| `git push --force` | ❌ Dangerous | Almost never — only in recovery scenarios |

## Git Fetch and Prune

### Stale Remote-Tracking Branches

When branches are deleted on the remote (e.g., after a PR is merged), your local remote-tracking references (`origin/feature/old-branch`) still exist. These become **stale** references pointing to branches that no longer exist on the remote.

```bash
# List all remote-tracking branches (including stale ones)
git branch -r

# Output may include branches already deleted on the remote:
#   origin/main
#   origin/feature/auth          ← still exists on remote
#   origin/feature/old-search    ← deleted on remote (stale!)
#   origin/fix/login-bug         ← deleted on remote (stale!)
```

### Pruning Workflows

```bash
# Prune stale remote-tracking branches for a specific remote
git fetch --prune origin

# Or use the shorthand
git fetch -p

# Prune without fetching
git remote prune origin

# See what would be pruned (dry run)
git remote prune origin --dry-run

# Output:
# * [would prune] origin/feature/old-search
# * [would prune] origin/fix/login-bug

# Configure Git to always prune on fetch
git config --global fetch.prune true
```

```bash
# Clean up local branches that tracked now-deleted remote branches
# List merged branches (safe to delete)
git branch --merged main

# Delete local branches whose remote counterpart is gone
git branch -vv | grep ': gone]' | awk '{print $1}' | xargs git branch -d
```

```
Before prune:

  Remote (origin)                 Local remote-tracking refs
  ┌───────────────┐              ┌──────────────────────────┐
  │ main          │              │ origin/main              │
  │ feature/auth  │              │ origin/feature/auth      │
  │               │              │ origin/feature/old-search│ ← stale
  │               │              │ origin/fix/login-bug     │ ← stale
  └───────────────┘              └──────────────────────────┘

After git fetch --prune:

  Remote (origin)                 Local remote-tracking refs
  ┌───────────────┐              ┌──────────────────────────┐
  │ main          │              │ origin/main              │
  │ feature/auth  │              │ origin/feature/auth      │
  └───────────────┘              └──────────────────────────┘

  Stale references are removed.
```

## Working with Multiple Remotes

### Upstream Tracking

When working with forks, you typically have two remotes: `origin` (your fork) and `upstream` (the original repository). Keeping track of which remote each branch tracks is essential.

```bash
# View all remote-tracking relationships
git branch -vv

# Output:
# * main          a1b2c3d [origin/main] Latest commit message
#   feature/auth  d4e5f6a [origin/feature/auth] Add auth endpoint

# Set a local branch to track a different remote branch
git branch --set-upstream-to=upstream/main main

# Now 'git pull' on main fetches from upstream instead of origin
```

```bash
# Fetch from a specific remote
git fetch upstream

# Compare your branch with upstream
git log --oneline main..upstream/main
# Shows commits in upstream/main that are not in your local main

git log --oneline upstream/main..main
# Shows commits in your local main that are not in upstream/main
```

### Triangle Workflows

A triangle workflow involves three repositories: upstream (source of truth), your fork (origin), and your local clone. You fetch from upstream and push to your fork.

```
Triangle Workflow:

                    upstream (org/project)
                   ┌─────────────────────┐
          fetch    │                     │    PR (merge)
        ┌─────────▶│  main               │◀───────────┐
        │          │                     │             │
        │          └─────────────────────┘             │
        │                                              │
  ┌─────┴──────────────┐                  ┌────────────┴────────┐
  │ Local Repository   │      push        │ origin (you/project)│
  │                    │─────────────────▶│                     │
  │ main               │                  │ main                │
  │ feature/my-change  │                  │ feature/my-change   │
  └────────────────────┘                  └─────────────────────┘

  Fetch from: upstream (to get latest changes)
  Push to:    origin   (your fork)
  PR from:    origin → upstream
```

```bash
# Configure triangle workflow
git remote add upstream https://github.com/org/project.git

# Configure push to go to origin by default
git config remote.pushdefault origin

# Configure pull to come from upstream for the main branch
git branch main --set-upstream-to upstream/main

# Now the triangle works:
git pull          # fetches from upstream/main
git push          # pushes to origin/main

# Or configure per-remote push URLs for a single remote
git remote set-url --push upstream https://github.com/you/project.git
# Now 'git push upstream' pushes to your fork, but 'git fetch upstream' fetches from org
```

```bash
# Daily triangle workflow
# 1. Start the day by syncing with upstream
git checkout main
git fetch upstream
git rebase upstream/main
git push origin main

# 2. Create a feature branch
git checkout -b feature/improve-search

# 3. Work and commit
git add .
git commit -m "feat(search): add fuzzy matching"

# 4. Before opening a PR, rebase onto latest upstream
git fetch upstream
git rebase upstream/main

# 5. Push to your fork
git push -u origin feature/improve-search

# 6. Open a PR from origin to upstream
gh pr create --repo org/project
```

## Best Practices

### Remote Management

- ✅ Use `origin` for your primary remote (or your fork in open-source workflows)
- ✅ Use `upstream` for the original repository when working with forks
- ✅ Configure `fetch.prune = true` globally to automatically clean stale references
- ✅ Use SSH keys or credential helpers to avoid repeated authentication
- ❌ Do not add remotes with hardcoded credentials in the URL
- ❌ Do not leave stale remote-tracking branches cluttering your local repo

### Push and Pull

- ✅ Pull with rebase (`git pull --rebase`) to maintain a linear history
- ✅ Always use `--force-with-lease` instead of `--force` when force pushing
- ✅ Set upstream tracking with `git push -u` on the first push of a new branch
- ✅ Fetch before starting new work to ensure you have the latest changes
- ❌ Do not force push to shared branches (`main`, `develop`, release branches)
- ❌ Do not push directly to main — use pull requests

### Pull Requests

- ✅ Keep PRs small and focused on a single change
- ✅ Write clear PR descriptions explaining the "what" and "why"
- ✅ Use PR templates to standardize descriptions across the team
- ✅ Self-review your code before requesting reviews from others
- ✅ Respond to review feedback promptly and push updates
- ❌ Do not merge your own PR without at least one review (on team projects)
- ❌ Do not leave PRs open for weeks — either finish them or close them

### Code Review

- ✅ Review the code, not the person — keep feedback professional and kind
- ✅ Distinguish between blocking issues and nits
- ✅ Provide suggestions with code examples when possible
- ✅ Approve promptly once issues are addressed — do not block progress
- ❌ Do not rubber-stamp reviews — actually read the code
- ❌ Do not use reviews to enforce personal style preferences that are not in the style guide

### Fork Maintenance

- ✅ Sync your fork's main branch with upstream regularly
- ✅ Create feature branches from the latest upstream main, not from a stale fork
- ✅ Delete feature branches after their PRs are merged
- ✅ Keep your fork clean — do not accumulate unmerged branches
- ❌ Do not commit directly to your fork's main branch — keep it a mirror of upstream

## Next Steps

Continue to [Git Hooks and Automation](05-GIT-HOOKS-AND-AUTOMATION.md) to learn about client-side and server-side Git hooks, pre-commit frameworks, CI/CD integration, and automating code quality checks in your Git workflow.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial remote collaboration documentation |
