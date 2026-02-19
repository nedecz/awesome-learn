# Git Anti-Patterns

## Table of Contents

1. [Overview](#overview)
2. [Commit Anti-Patterns](#commit-anti-patterns)
3. [Branching Anti-Patterns](#branching-anti-patterns)
4. [Merge Anti-Patterns](#merge-anti-patterns)
5. [History Anti-Patterns](#history-anti-patterns)
6. [Workflow Anti-Patterns](#workflow-anti-patterns)
7. [Repository Anti-Patterns](#repository-anti-patterns)
8. [Collaboration Anti-Patterns](#collaboration-anti-patterns)
9. [Security Anti-Patterns](#security-anti-patterns)
10. [How to Detect and Fix](#how-to-detect-and-fix)
11. [Anti-Pattern Summary Table](#anti-pattern-summary-table)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

## Overview

This document catalogs the most common Git anti-patterns — mistakes in commits, branching, merging, collaboration, and security that undermine productivity, introduce risk, and erode the value of version control. For each anti-pattern, we describe the problem, the symptoms, and the correct approach with concrete examples.

### Target Audience

- Developers looking to improve their Git habits
- Tech leads establishing team conventions and enforcing standards
- Teams troubleshooting slow, error-prone, or insecure Git workflows

### Scope

- Commit anti-patterns (huge commits, vague messages, committed secrets)
- Branching anti-patterns (long-lived branches, no cleanup, direct pushes to main)
- Merge anti-patterns (force push, unresolved conflicts, unnecessary merge commits)
- History anti-patterns (rewriting public history, losing commits, missing tags)
- Workflow anti-patterns (no code review, no CI, no branch protection)
- Repository anti-patterns (monorepo without tooling, large binaries, no .gitignore)
- Collaboration anti-patterns (not syncing, ignoring feedback, siloed knowledge)
- Security anti-patterns (secrets in repos, no signing, shared accounts)

## Commit Anti-Patterns

### Huge Commits

#### The Problem

A single commit contains hundreds of changed files or thousands of lines spanning multiple unrelated changes. This makes code review impossible, bisecting useless, and reverting dangerous.

```
❌ Huge Commit

  commit 4a2f8c1
  Author: dev@example.com
  Date:   Mon Jun 9 14:00:00 2025

    Refactor everything, fix bugs, add feature X, update deps

    72 files changed, 4,832 insertions(+), 2,109 deletions(-)

  → Cannot review meaningfully
  → Cannot revert without losing unrelated work
  → git bisect cannot isolate the bug
```

```
✅ Atomic Commits

  commit a1b2c3d — Refactor: extract PaymentProcessor class
    3 files changed, 84 insertions(+), 52 deletions(-)

  commit e4f5a6b — Fix: handle null shipping address in checkout
    2 files changed, 11 insertions(+), 3 deletions(-)

  commit c7d8e9f — Feat: add coupon code validation
    4 files changed, 127 insertions(+), 0 deletions(-)

  → Each commit is reviewable, revertable, and bisectable
```

#### How to Fix

- ✅ Commit one logical change at a time
- ✅ Use `git add -p` to stage specific hunks within a file
- ✅ If you have accumulated many changes, use interactive staging to split them into atomic commits
- ❌ Do not batch unrelated changes into a single commit

### Vague Commit Messages

#### The Problem

Commit messages like "fix stuff", "WIP", "updates", or "asdf" provide no context. They make the log useless for understanding why a change was made and hinder debugging.

```
❌ Vague Messages

  git log --oneline

  f3a1b2c WIP
  d4e5f6a fix stuff
  a7b8c9d updates
  e1f2a3b misc changes
  c4d5e6f asdf
  b7a8c9d .
```

```
✅ Descriptive Messages (Conventional Commits)

  git log --oneline

  f3a1b2c feat(auth): add OAuth2 PKCE flow for mobile clients
  d4e5f6a fix(cart): prevent negative quantities on line items
  a7b8c9d docs(api): add rate limiting section to API reference
  e1f2a3b refactor(db): replace raw SQL with query builder
  c4d5e6f ci: add integration test stage to GitHub Actions pipeline
  b7a8c9d chore(deps): upgrade express from 4.18 to 4.19
```

#### How to Fix

- ✅ Follow the [Conventional Commits](https://www.conventionalcommits.org/) format: `type(scope): description`
- ✅ Explain what changed and why — not just what files were touched
- ✅ Use the commit body for additional context when the subject line is not enough
- ✅ Enforce message format with commit-msg hooks or CI checks
- ❌ Do not leave "WIP" commits in shared branches — squash or amend before pushing

### Committing Generated Files

#### The Problem

Build artifacts, compiled binaries, dependency directories, and auto-generated code are committed to the repository, causing bloat, merge conflicts, and confusion about what is source and what is output.

```
❌ Generated Files in the Repository

  repo/
  ├── src/
  ├── dist/              ← build output
  │   ├── bundle.js
  │   └── bundle.js.map
  ├── node_modules/      ← dependencies (41,000 files)
  ├── coverage/          ← test coverage reports
  └── .env               ← environment secrets
```

```
✅ Proper .gitignore

  # .gitignore
  dist/
  node_modules/
  coverage/
  .env
  *.pyc
  __pycache__/
  *.class
  target/
  build/
```

#### How to Fix

- ✅ Add a comprehensive `.gitignore` before the first commit
- ✅ Start with a language-specific template from [github.com/github/gitignore](https://github.com/github/gitignore)
- ✅ Remove already-tracked generated files with `git rm --cached <path>`
- ❌ Do not commit anything that can be regenerated from source

### Committing Secrets

#### The Problem

API keys, passwords, tokens, private keys, and connection strings are committed to the repository. Even if removed in a later commit, they remain in Git history and are accessible to anyone who clones the repo.

```
❌ Secrets in Code

  # config.py
  DATABASE_URL = "postgresql://admin:s3cretP@ss!@db.prod.example.com:5432/app"
  AWS_SECRET_ACCESS_KEY = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
  STRIPE_API_KEY = "sk_live_4eC39HqLyjWDarjtT1zdp7dc"
```

```
✅ Secrets via Environment Variables

  # config.py
  import os

  DATABASE_URL = os.environ["DATABASE_URL"]
  AWS_SECRET_ACCESS_KEY = os.environ["AWS_SECRET_ACCESS_KEY"]
  STRIPE_API_KEY = os.environ["STRIPE_API_KEY"]

  # .env.example (committed — contains placeholder values only)
  DATABASE_URL=postgresql://user:password@localhost:5432/app
  AWS_SECRET_ACCESS_KEY=your-aws-secret-key
  STRIPE_API_KEY=sk_test_your-stripe-key

  # .env (NOT committed — listed in .gitignore)
  DATABASE_URL=postgresql://admin:real-password@db.prod.example.com:5432/app
```

#### How to Fix

- ✅ Use environment variables or a secrets manager (Vault, AWS Secrets Manager, Doppler)
- ✅ Add secret file patterns to `.gitignore` (`.env`, `*.pem`, `*.key`)
- ✅ Use pre-commit hooks or CI tools to detect secrets (gitleaks, trufflehog, git-secrets)
- ✅ If a secret is committed, rotate it immediately — removing the commit is not enough
- ❌ Do not rely on making a repo private as the only protection for secrets

## Branching Anti-Patterns

### Long-Lived Feature Branches

#### The Problem

Feature branches that live for weeks or months diverge significantly from main. The longer a branch lives, the harder it is to merge, the more conflicts accumulate, and the greater the risk of integration failures.

```
❌ Long-Lived Feature Branch

  main:     A───B───C───D───E───F───G───H───I───J
                 \
  feature:        B1──B2──B3──B4──B5──B6──B7──B8──B9──B10
                                                        \
                                          Merge after 3 weeks → 47 conflicts

✅ Short-Lived Branches with Frequent Integration

  main:     A───B───C───D───E───F───G───H───I───J
                 \       \       \       \
  feature:        B1──B2  D1──D2  F1──F2  H1──H2
                      |       |       |       |
                    merge   merge   merge   merge
                    to main to main to main to main

  → Each merge is small, conflicts are minimal
```

#### How to Fix

- ✅ Merge feature branches within 1–3 days
- ✅ Use feature flags to merge incomplete work behind a toggle
- ✅ Rebase or merge from main daily to stay current
- ✅ Break large features into small, independently mergeable increments
- ❌ Do not let branches diverge for more than a few days

### Not Deleting Merged Branches

#### The Problem

Merged branches are never cleaned up. Over time, the repository accumulates hundreds of stale branches, making it hard to find active work and cluttering tooling output.

```
❌ Stale Branch Proliferation

  $ git branch -r | wc -l
  347

  $ git branch -r
    origin/feature/add-login         ← merged 6 months ago
    origin/feature/fix-header        ← merged 4 months ago
    origin/feature/update-deps       ← merged 3 months ago
    origin/feature/redesign-v2       ← merged 2 months ago
    origin/hotfix/null-check         ← merged 1 year ago
    ... 342 more
```

```
✅ Clean Branch List

  $ git branch -r | wc -l
  8

  $ git branch -r
    origin/main
    origin/develop
    origin/feature/cart-redesign     ← active, PR open
    origin/feature/api-v3            ← active, PR open
    origin/release/2.4.0             ← in testing
```

#### How to Fix

- ✅ Delete branches immediately after merging (enable auto-delete in GitHub/GitLab)
- ✅ Run periodic cleanup: `git fetch --prune` to remove stale remote-tracking branches
- ✅ Use `git branch --merged main | grep -v main | xargs git branch -d` to delete local merged branches
- ❌ Do not keep branches "just in case" — merged commits are safely on main

### Direct Commits to Main

#### The Problem

Developers push commits directly to the main branch without going through a pull request or code review. This bypasses quality gates, introduces untested code, and makes it impossible to audit changes.

```
❌ Direct Push to Main

  $ git checkout main
  $ git commit -m "quick fix"
  $ git push origin main

  → No review, no CI, no audit trail
  → Broken code reaches production
```

```
✅ Branch + Pull Request + Review

  $ git checkout -b fix/null-pointer-checkout
  $ git commit -m "fix(checkout): handle null payment method"
  $ git push origin fix/null-pointer-checkout
  # → Open PR → CI runs → Reviewer approves → Merge

  → Every change is reviewed, tested, and traceable
```

#### How to Fix

- ✅ Enable branch protection rules on main (require PRs, require approvals, require CI to pass)
- ✅ Block force pushes to main and other protected branches
- ✅ Use CODEOWNERS to require reviews from domain experts for specific paths
- ❌ Do not grant direct push access to main for anyone, including administrators

### Branch Proliferation

#### The Problem

The team creates branches for every small task but also maintains permanent branches for environments (dev, staging, QA, UAT, prod). This creates a complex branching hierarchy that is hard to reason about and error-prone.

```
❌ Too Many Permanent Branches

  main
  develop
  staging
  qa
  uat
  release/2.3
  release/2.4
  hotfix/2.3.1
  hotfix/2.3.2
  feature/...  (dozens)

  → Cherry-pick bugs between branches
  → Unclear which branch has which code
  → Merges in wrong direction cause regressions
```

```
✅ Simplified Branch Model (GitHub Flow)

  main              ← always deployable
  feature/cart-v2   ← short-lived, one per task
  fix/api-timeout   ← short-lived, merged via PR

  Environments are managed by deployment tooling, not branches.
```

#### How to Fix

- ✅ Adopt a simple branching model (GitHub Flow or trunk-based development)
- ✅ Use deployment tooling — not branches — to manage environments
- ✅ Limit permanent branches to main (and optionally one release branch)
- ❌ Do not create branches for environments (dev, staging, QA)

## Merge Anti-Patterns

### Force Push to Shared Branches

#### The Problem

Using `git push --force` on shared branches (main, develop, release) rewrites history that other developers have already pulled. This causes lost commits, broken local repositories, and confusion.

```
❌ Force Push to Shared Branch

  Developer A:                     Developer B:
  main: A─B─C─D                   main: A─B─C─D  (pulled)
        ↓                                ↓
  Rewrites: A─B─X                 Pushes on top of D:
  git push --force                  → ERROR: diverged
                                    → B's commits C and D are lost
                                    → B must manually recover work
```

#### How to Fix

- ✅ Use `--force-with-lease` instead of `--force` if you must force push (it fails if someone else pushed)
- ✅ Only force push on your own personal feature branches
- ✅ Enable branch protection to block force pushes on main and shared branches
- ❌ Never use `git push --force` on main, develop, or release branches

### Merge Conflicts Left Unresolved

#### The Problem

Merge conflicts are "resolved" by accepting one side blindly, auto-merging incorrectly, or leaving conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) in the code. The result is broken code that compiles but behaves incorrectly — or does not compile at all.

```
❌ Conflict Markers Left in Code

  function calculateTotal(items) {
  <<<<<<< HEAD
    return items.reduce((sum, item) => sum + item.price, 0);
  =======
    return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  >>>>>>> feature/quantity-support
  }

  → File may still "compile" in some languages
  → Bug is hidden until runtime
```

```
✅ Properly Resolved Conflict

  function calculateTotal(items) {
    return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }

  → Developer understood both changes and merged them intentionally
  → Tests pass after resolution
```

#### How to Fix

- ✅ Always run tests after resolving conflicts
- ✅ Use a visual merge tool (VS Code, IntelliJ, Beyond Compare) to understand both sides
- ✅ Add a CI check that rejects files containing conflict markers
- ✅ If unsure, ask the author of the conflicting change to help resolve
- ❌ Do not blindly accept "ours" or "theirs" without understanding the change

### Unnecessary Merge Commits

#### The Problem

Every branch update pulls in a merge commit from main, creating a cluttered, non-linear history that is difficult to read and provides no useful information.

```
❌ Cluttered History (Merge Commits Everywhere)

  $ git log --oneline --graph

  *   e4f5a6b Merge branch 'main' into feature/cart
  |\
  | * d3c2b1a Merge branch 'main' into feature/cart
  | |\
  | | * a9b8c7d Merge branch 'main' into feature/cart
  | | |\
  ...
  Actual work is buried under merge noise
```

```
✅ Clean History (Rebase Before Merge)

  $ git log --oneline --graph

  * f3a1b2c feat(cart): add quantity selector
  * d4e5f6a feat(cart): display subtotal
  * a7b8c9d feat(cart): create cart page layout
  * e1f2a3b fix(auth): handle expired refresh tokens
  * c4d5e6f feat(api): add pagination to product list

  → Linear, readable, each commit tells a story
```

#### How to Fix

- ✅ Rebase feature branches onto main before merging: `git rebase main`
- ✅ Use squash merges for feature branches to collapse into a single commit
- ✅ Configure the repository default merge strategy (squash or rebase) in GitHub/GitLab
- ❌ Do not use `git pull` without `--rebase` on feature branches (it creates merge commits)

## History Anti-Patterns

### Rewriting Public History

#### The Problem

Using `git rebase`, `git commit --amend`, or `git filter-branch` on commits that have already been pushed to a shared branch. This rewrites commit SHAs that other developers are working on top of.

```
❌ Rewriting Public History

  Developer A pushes to main:     A─B─C─D
  Developer B pulls and builds on: A─B─C─D─E─F

  Developer A amends commit C:    A─B─C'─D'
  Developer A force pushes main:  A─B─C'─D'

  Developer B:
    → "fatal: refusing to merge unrelated histories"
    → Commits E and F are orphaned
    → Manual recovery required
```

#### How to Fix

- ✅ Only rewrite history on local, unpushed commits or personal feature branches
- ✅ Use `git rebase -i` before pushing to clean up local commits
- ✅ After pushing, use `git revert` to undo changes instead of rewriting
- ❌ Never amend, rebase, or squash commits that others have pulled

### Losing Commits

#### The Problem

Commits are lost through careless rebases, hard resets, or branch deletions without verifying that work was merged. Developers lose hours or days of work.

```
❌ Losing Work

  # Accidentally reset past important work
  $ git reset --hard HEAD~5      ← 5 commits gone

  # Deleted a branch without merging
  $ git branch -D feature/new-api  ← unmerged work lost

  # Rebased and dropped commits
  $ git rebase -i HEAD~10        ← accidentally deleted a "pick" line
```

```
✅ Recovering Lost Commits

  # The reflog saves you — find the lost commit
  $ git reflog
  a1b2c3d HEAD@{0}: reset: moving to HEAD~5
  f4e5d6c HEAD@{1}: commit: feat(api): add rate limiting middleware
  b7a8c9d HEAD@{2}: commit: feat(api): add pagination support

  # Recover the lost commits
  $ git checkout -b recovery-branch f4e5d6c

  # Or cherry-pick specific commits back
  $ git cherry-pick f4e5d6c b7a8c9d
```

#### How to Fix

- ✅ Use `git reflog` to find and recover lost commits (retained for 90 days by default)
- ✅ Prefer `git revert` over `git reset --hard` for undoing changes
- ✅ Push feature branches to the remote regularly as a backup
- ✅ Use `git stash` instead of discarding uncommitted work
- ❌ Do not use `git reset --hard` unless you are certain about what you are discarding

### Not Using Tags for Releases

#### The Problem

Releases are tracked by branch names, commit messages, or external documents instead of Git tags. There is no reliable way to check out the exact code that was deployed to production.

```
❌ No Release Tags

  $ git log --oneline
  f3a1b2c bump version to 2.4.1
  d4e5f6a deploy to prod
  a7b8c9d fix bug
  e1f2a3b version 2.4.0 maybe?

  → "Which commit is running in production?" → Nobody knows.
```

```
✅ Semantic Version Tags

  $ git tag -l
  v2.3.0
  v2.3.1
  v2.4.0
  v2.4.1

  $ git show v2.4.1
  tag v2.4.1
  Tagger: release-bot <ci@example.com>
  Date:   Mon Jun 9 14:00:00 2025

  Release 2.4.1 — hotfix for checkout null pointer

  # Check out exact production code
  $ git checkout v2.4.1
```

#### How to Fix

- ✅ Create annotated tags for every release: `git tag -a v2.4.1 -m "Release 2.4.1"`
- ✅ Use semantic versioning (vMAJOR.MINOR.PATCH)
- ✅ Automate tagging in CI/CD pipelines on successful deployments
- ✅ Use `git describe` to identify the nearest tag for any commit
- ❌ Do not track releases through commit messages or branch names alone

## Workflow Anti-Patterns

### No Code Review

#### The Problem

Code is merged without any peer review. Bugs, security vulnerabilities, and design flaws go undetected until they reach production.

### Symptoms

- Bugs are found in production that would have been caught in review
- Code quality degrades over time with no feedback loop
- Knowledge is siloed — only the author understands the code
- No audit trail for why changes were made

### How to Fix

- ✅ Require at least one approving review before merging
- ✅ Use CODEOWNERS to assign domain-specific reviewers automatically
- ✅ Keep PRs small (under 400 lines) so reviews are effective
- ❌ Do not merge your own PRs without another pair of eyes

### No CI Integration

#### The Problem

There is no continuous integration pipeline. Tests, linting, and builds are not run automatically on pull requests. Broken code is merged because nobody ran the tests.

```
❌ No CI

  Developer pushes → PR is opened → Reviewer skims code → Merged
  → Tests? "I ran them locally... I think."
  → Build breaks on main
```

```
✅ CI on Every PR

  Developer pushes → CI runs automatically:
    ✓ Lint passes
    ✓ Unit tests pass (247/247)
    ✓ Integration tests pass (38/38)
    ✓ Build succeeds
    ✓ Security scan clean
  → Reviewer reviews code + CI results → Merged
```

### How to Fix

- ✅ Set up CI to run on every push and pull request (GitHub Actions, GitLab CI, etc.)
- ✅ Require CI to pass before a PR can be merged (branch protection rules)
- ✅ Include linting, unit tests, integration tests, and security scans
- ❌ Do not rely on developers to "remember" to run tests locally

### No Branch Protection

#### The Problem

Main and release branches have no protection rules. Anyone can force push, push directly, or merge without reviews or passing CI.

### How to Fix

- ✅ Enable branch protection on main and release branches
- ✅ Require pull request reviews (minimum 1 approval)
- ✅ Require status checks to pass before merging
- ✅ Block force pushes and deletions on protected branches
- ✅ Require linear history (no merge commits) if desired
- ❌ Do not exempt administrators from branch protection rules

### Inconsistent Workflows

#### The Problem

Different team members use different branching models, commit conventions, and merge strategies. One developer uses rebase, another uses merge commits, a third uses squash. The history is chaotic and unpredictable.

### Symptoms

- History is a mix of merge commits, squash commits, and rebased commits
- Branch naming is inconsistent (`feature/x`, `feat-x`, `john/x`, `x`)
- Some PRs are reviewed, others are merged directly
- No documentation on the team's Git workflow

### How to Fix

- ✅ Document the team's Git workflow in CONTRIBUTING.md
- ✅ Standardize branch naming: `feature/`, `fix/`, `chore/`, `release/`
- ✅ Choose one merge strategy and enforce it (squash merge recommended for most teams)
- ✅ Use commit message linting (commitlint, Conventional Commits)
- ✅ Automate enforcement with hooks and CI checks

## Repository Anti-Patterns

### Monorepo Without Tooling

#### The Problem

Multiple projects live in a single repository, but there is no tooling to manage builds, tests, or deployments for individual projects. Every CI run builds and tests everything, even when only one project changed.

```
❌ Monorepo Without Tooling

  repo/
  ├── frontend/         ← React app
  ├── backend/          ← Node.js API
  ├── mobile/           ← React Native app
  └── infrastructure/   ← Terraform

  CI pipeline:
    1. Build frontend      (3 min)
    2. Build backend       (2 min)
    3. Build mobile        (5 min)
    4. Plan infrastructure (1 min)
    Total: 11 min — even if only one line changed in frontend
```

```
✅ Monorepo With Proper Tooling

  Same repo structure, but with:
  - Nx / Turborepo / Bazel for build orchestration
  - Affected-only builds: only build/test what changed
  - Caching: skip unchanged builds

  CI pipeline (frontend change only):
    1. Build frontend      (3 min, or 10s from cache)
    Total: 3 min (or 10 seconds with cache hit)
```

### How to Fix

- ✅ Use monorepo tooling (Nx, Turborepo, Bazel, Lerna) to manage builds
- ✅ Implement affected-only CI — only build and test what changed
- ✅ Use build caching to skip unchanged projects
- ✅ Define clear ownership boundaries with CODEOWNERS
- ❌ Do not put multiple projects in one repo without build orchestration

### Storing Large Binaries Without LFS

#### The Problem

Large binary files (images, videos, datasets, compiled binaries) are committed directly to the repository. Git stores every version of every file, so the repository size grows rapidly and clones become painfully slow.

```
❌ Large Binaries in Git

  $ git count-objects -vH
  size-pack: 2.4 GiB

  $ git clone repo.git
  Cloning into 'repo'...
  Receiving objects: 100% (84321/84321), 2.4 GiB | 5.2 MiB/s
  → 8 minutes to clone

  $ git log --all --diff-filter=A -- '*.psd' '*.mp4' '*.zip'
  → 200+ binary files across history
```

```
✅ Git LFS for Large Files

  # Track large file types with LFS
  $ git lfs track "*.psd" "*.mp4" "*.zip" "*.bin"

  $ cat .gitattributes
  *.psd filter=lfs diff=lfs merge=lfs -text
  *.mp4 filter=lfs diff=lfs merge=lfs -text
  *.zip filter=lfs diff=lfs merge=lfs -text

  $ git clone repo.git
  Cloning into 'repo'...
  Receiving objects: 100% (12043/12043), 48 MiB | 12 MiB/s
  → 4 seconds to clone (LFS files downloaded on demand)
```

### How to Fix

- ✅ Use Git LFS for files larger than 1 MB that are binary or non-diffable
- ✅ Add `.gitattributes` rules for large file types before they are committed
- ✅ Use `git lfs migrate` to retroactively move existing large files to LFS
- ❌ Do not commit datasets, media files, or compiled binaries directly to Git

### No .gitignore

#### The Problem

The repository has no `.gitignore` file, or the file is incomplete. Build artifacts, OS files, IDE configuration, dependencies, and secrets are tracked and committed.

### Symptoms

- `.DS_Store`, `Thumbs.db`, `*.swp` files litter the repository
- `node_modules/`, `__pycache__/`, `target/` directories are committed
- `.env` files with real credentials are in the repo
- Every contributor's IDE settings (`.idea/`, `.vscode/`) cause merge conflicts

### How to Fix

- ✅ Add a `.gitignore` before the first commit in every repository
- ✅ Use templates from [github.com/github/gitignore](https://github.com/github/gitignore)
- ✅ Set up a global gitignore for OS and editor files: `git config --global core.excludesfile ~/.gitignore_global`
- ✅ Remove already-tracked files: `git rm -r --cached node_modules/ && git commit -m "chore: remove tracked node_modules"`
- ❌ Do not rely on contributors to "remember" not to commit generated files

## Collaboration Anti-Patterns

### Not Syncing with Upstream

#### The Problem

Developers work on feature branches for days without pulling changes from main. When they finally merge, the branch has diverged significantly, leading to painful conflicts and integration failures.

```
❌ Not Syncing

  Day 1:  Branch from main (commit A)
  Day 2:  Work on feature (commits B1, B2)
  Day 3:  Work on feature (commits B3, B4)  ← main has moved 20 commits ahead
  Day 4:  Work on feature (commits B5, B6)  ← main has moved 35 commits ahead
  Day 5:  Try to merge → 23 conflicts across 14 files
```

```
✅ Daily Sync

  Day 1:  Branch from main, work (B1, B2)
  Day 2:  git pull --rebase origin main, work (B3, B4)  ← 0 conflicts
  Day 3:  git pull --rebase origin main, work (B5, B6)  ← 1 small conflict
  Day 4:  Open PR → clean merge, 0 conflicts
```

#### How to Fix

- ✅ Rebase or merge from main at least daily
- ✅ Use `git pull --rebase origin main` to stay current without merge commits
- ✅ Set up notifications for changes to files you are working on
- ❌ Do not go more than a day without syncing with the upstream branch

### Ignoring PR Feedback

#### The Problem

Reviewers leave comments on pull requests, but the author merges without addressing them. Feedback is ignored, dismissed without explanation, or resolved by clicking "Resolve" without making changes.

### Symptoms

- Reviewers stop leaving feedback because it is ignored
- Code quality degrades despite having a review process
- Resentment builds between team members
- The review process becomes performative — reviews happen but have no impact

### How to Fix

- ✅ Respond to every comment — either make the change or explain why not
- ✅ Use "Request changes" as a blocking review to prevent premature merges
- ✅ Require that all review threads are resolved before merging
- ✅ Treat code review as a learning opportunity, not a gatekeeping exercise
- ❌ Do not click "Resolve" on a comment without addressing the feedback

### Siloing Knowledge

#### The Problem

Only one person understands a part of the codebase. There is no documentation, no pair programming, no rotation of reviewers. When that person leaves or is unavailable, the team is stuck.

```
❌ Knowledge Silos

  ┌──────────────┐
  │ Backend API  │ ← Only Alice knows this
  ├──────────────┤
  │ Auth System  │ ← Only Bob knows this
  ├──────────────┤
  │ CI Pipeline  │ ← Only Carol knows this
  └──────────────┘

  Alice goes on vacation → API changes are blocked for 2 weeks
  Bob leaves the company → nobody can modify the auth system
```

```
✅ Shared Knowledge

  ┌──────────────┐
  │ Backend API  │ ← Alice (primary), Bob + Carol (reviewers)
  ├──────────────┤
  │ Auth System  │ ← Bob (primary), Alice + Dave (reviewers)
  ├──────────────┤
  │ CI Pipeline  │ ← Carol (primary), Dave + Alice (reviewers)
  └──────────────┘

  CODEOWNERS ensures at least 2 people review each area.
  Pair programming spreads knowledge organically.
```

#### How to Fix

- ✅ Rotate PR reviewers so multiple people learn each area
- ✅ Use CODEOWNERS to require reviews from at least two people per area
- ✅ Document architectural decisions in ADRs (Architecture Decision Records)
- ✅ Pair program on complex changes to spread knowledge
- ❌ Do not let one person be the sole expert on any critical system

## Security Anti-Patterns

### Secrets in Repositories

#### The Problem

Credentials, API keys, tokens, and private keys are committed to the repository. This is the most common and most dangerous Git security anti-pattern. Secrets in Git history are effectively public — even in private repos, they are accessible to every contributor and any tool with repo access.

| Secret Type | Risk if Exposed |
|---|---|
| **Database credentials** | Full data breach, data loss |
| **API keys (AWS, GCP, Stripe)** | Financial loss, resource abuse |
| **JWT signing keys** | Authentication bypass |
| **SSH private keys** | Server compromise |
| **OAuth client secrets** | Account takeover |

#### How to Fix

- ✅ Use a secrets manager (HashiCorp Vault, AWS Secrets Manager, Doppler)
- ✅ Run secret scanning in CI (gitleaks, trufflehog, GitHub secret scanning)
- ✅ Install pre-commit hooks that block secrets before they are committed
- ✅ Rotate any secret that was ever committed — removing the commit is not enough
- ❌ Do not store secrets in code, config files, or environment files committed to Git

### No Signed Commits in Regulated Environments

#### The Problem

In regulated industries (finance, healthcare, government), there is no way to verify that a commit was actually made by the person listed as the author. Git allows anyone to set any name and email in their config.

```
❌ Unverified Authorship

  # Anyone can impersonate any author
  $ git config user.name "CEO Name"
  $ git config user.email "ceo@company.com"
  $ git commit -m "Approve financial override"

  → Commit appears to be from the CEO
  → No cryptographic proof of authorship
  → Fails audit requirements
```

```
✅ Signed Commits with GPG/SSH

  $ git config commit.gpgsign true
  $ git config user.signingkey ~/.ssh/id_ed25519.pub
  $ git commit -m "Approve financial override"

  $ git log --show-signature
  commit f3a1b2c (HEAD -> main)
  Good "git" signature for ceo@company.com with ED25519 key SHA256:...
  Author: CEO Name <ceo@company.com>

  → Cryptographically verified authorship
  → GitHub shows "Verified" badge
  → Meets audit and compliance requirements
```

#### How to Fix

- ✅ Require signed commits in regulated environments (GPG or SSH signing)
- ✅ Configure vigilant mode on GitHub to flag unsigned commits
- ✅ Set `commit.gpgsign = true` in the global Git config
- ✅ Use SSH key signing (simpler setup than GPG): `git config gpg.format ssh`
- ❌ Do not rely on Git author metadata alone for audit trails

### Shared Accounts

#### The Problem

Multiple developers share a single Git account or SSH key. There is no way to attribute changes to a specific person, making auditing impossible and accountability nonexistent.

### Symptoms

- Multiple people commit as "deploy-bot" or "team-account"
- A single SSH key is shared across the team
- Git blame shows the same author for all changes
- Impossible to determine who introduced a bug or approved a change

### How to Fix

- ✅ Give every developer their own Git account and SSH key
- ✅ Use deploy keys or machine accounts for CI/CD (scoped to specific repos)
- ✅ Audit access regularly — remove accounts for departed team members
- ✅ Use short-lived tokens instead of long-lived credentials for automation
- ❌ Do not share SSH keys, personal access tokens, or Git credentials

## How to Detect and Fix

### Git Log Analysis

Use Git log commands to detect anti-patterns in your repository.

```bash
# Find huge commits (more than 20 files changed)
git log --oneline --shortstat | awk '/files? changed/ {
  split($0, a, ",")
  for (i in a) {
    if (a[i] ~ /file/) {
      gsub(/[^0-9]/, "", a[i])
      if (a[i]+0 > 20) print prev " → " $0
    }
  }
} { prev = $0 }'

# Find vague commit messages
git log --oneline --all | grep -iE "^[a-f0-9]+ (fix|wip|update|changes|stuff|misc|temp|test|asdf|\.+)$"

# Find commits with secrets (basic patterns)
git log -p --all -S 'AKIA' --oneline         # AWS access keys
git log -p --all -S 'sk_live_' --oneline      # Stripe live keys
git log -p --all -S 'BEGIN RSA PRIVATE' --oneline  # Private keys

# Find large files in history
git rev-list --objects --all | \
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | \
  awk '/^blob/ { if ($3 > 1048576) print $3/1048576 "MB", $4 }' | \
  sort -rn | head -20

# Find stale branches (merged but not deleted)
git branch -r --merged main | grep -v main

# Find long-lived unmerged branches
git for-each-ref --sort=-committerdate --format='%(committerdate:short) %(refname:short)' refs/remotes/ | head -20
```

### Pre-Commit Hooks

Prevent anti-patterns before they enter the repository.

```bash
# .pre-commit-config.yaml
repos:
  # Detect secrets
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks

  # Lint commit messages
  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: v3.1.0
    hooks:
      - id: conventional-pre-commit
        stages: [commit-msg]

  # Prevent large files
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: check-added-large-files
        args: ['--maxkb=500']
      - id: check-merge-conflict
      - id: detect-private-key
```

### CI Checks

Add automated checks to your CI pipeline to catch anti-patterns on every PR.

```yaml
# .github/workflows/git-hygiene.yml
name: Git Hygiene
on: [pull_request]

jobs:
  checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # Check for conflict markers
      - name: Check for conflict markers
        run: |
          if grep -rn '<<<<<<< \|======= \|>>>>>>> ' --include='*.js' --include='*.ts' --include='*.py' .; then
            echo "::error::Conflict markers found in source files"
            exit 1
          fi

      # Check for secrets
      - name: Run gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      # Lint commit messages
      - name: Lint commits
        uses: wagoid/commitlint-github-action@v5
```

### Useful Tools

| Tool | Purpose | Install |
|---|---|---|
| **gitleaks** | Detect secrets in Git repos | `brew install gitleaks` |
| **trufflehog** | Deep secret scanning across Git history | `brew install trufflehog` |
| **git-secrets** | AWS-focused secret prevention | `brew install git-secrets` |
| **commitlint** | Enforce Conventional Commits | `npm install -g @commitlint/cli` |
| **pre-commit** | Manage Git pre-commit hooks | `pip install pre-commit` |
| **git-sizer** | Analyze repository size and flag issues | `brew install git-sizer` |
| **BFG Repo-Cleaner** | Remove large files or secrets from history | [rtyley.github.io/bfg-repo-cleaner](https://rtyley.github.io/bfg-repo-cleaner/) |

## Anti-Pattern Summary Table

| Anti-Pattern | Problem | Impact | Solution |
|---|---|---|---|
| **Huge commits** | Hundreds of files in one commit | Unreviewable, unrevertable, breaks bisect | Atomic commits, `git add -p` |
| **Vague messages** | "fix stuff", "WIP", "updates" | Useless history, no audit trail | Conventional Commits, commit-msg hooks |
| **Generated files** | Build artifacts and deps committed | Repo bloat, false conflicts | `.gitignore` before first commit |
| **Committed secrets** | API keys and passwords in code | Data breaches, financial loss | Secrets manager, pre-commit scanning |
| **Long-lived branches** | Feature branches open for weeks | Painful merges, integration failures | Merge within 1–3 days, feature flags |
| **Stale branches** | Merged branches never deleted | Clutter, confusion about active work | Auto-delete on merge, `git fetch --prune` |
| **Direct commits to main** | No PR, no review, no CI | Untested code in production | Branch protection, require PRs |
| **Branch proliferation** | Branches for every environment | Complex merges, wrong-direction merges | GitHub Flow, deploy with tooling not branches |
| **Force push to shared** | `--force` on main or develop | Lost commits, broken local repos | `--force-with-lease`, branch protection |
| **Unresolved conflicts** | Conflict markers left in code | Runtime bugs, broken builds | Visual merge tools, post-merge tests |
| **Unnecessary merges** | Merge commits from every sync | Cluttered, unreadable history | Rebase before merge, squash merges |
| **Rewriting public history** | Amend/rebase pushed commits | Diverged histories, lost work | Only rewrite local commits, use revert |
| **Losing commits** | Careless reset or branch deletion | Lost work, hours of recovery | Use reflog, prefer revert over reset |
| **No release tags** | Releases tracked by commit messages | Cannot reproduce production builds | Annotated tags, semantic versioning |
| **No code review** | PRs merged without review | Bugs in production, no feedback loop | Require approvals, CODEOWNERS |
| **No CI** | Tests not run automatically | Broken main branch | CI on every PR, require passing checks |
| **No branch protection** | Anyone can push to main | Untested, unreviewed code in production | Enable branch protection rules |
| **Inconsistent workflows** | Mixed merge strategies, naming | Chaotic history, confusion | Document workflow, enforce with tooling |
| **Monorepo without tooling** | All projects build on every change | Slow CI, wasted resources | Nx, Turborepo, affected-only builds |
| **Large binaries without LFS** | Binary files committed directly | Slow clones, massive repo size | Git LFS, `.gitattributes` |
| **No .gitignore** | Generated files tracked | Bloat, false conflicts, leaked secrets | Add `.gitignore` from templates |
| **Not syncing upstream** | Branches diverge for days | Painful merge conflicts | Rebase daily, `git pull --rebase` |
| **Ignoring PR feedback** | Review comments dismissed | Eroded trust, declining code quality | Address every comment, require resolution |
| **Knowledge silos** | Only one person knows a system | Blocked work, single points of failure | Rotate reviewers, CODEOWNERS, pair programming |
| **Secrets in repos** | Credentials in Git history | Data breaches, compliance violations | Secret scanning, pre-commit hooks |
| **No signed commits** | Unverified authorship | Failed audits, impersonation risk | GPG/SSH signing, vigilant mode |
| **Shared accounts** | Multiple people, one account | No accountability, no audit trail | Individual accounts, scoped deploy keys |

## Next Steps

Review the [Best Practices](09-BEST-PRACTICES.md) guide for the positive counterpart to every anti-pattern covered here — conventions and workflows that prevent these mistakes from occurring.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025-06-10 | Initial release — commit, branching, merge, history, workflow, repository, collaboration, and security anti-patterns |
