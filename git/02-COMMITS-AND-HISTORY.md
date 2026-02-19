# Git Commits and History

## Table of Contents

1. [Overview](#overview)
2. [Anatomy of a Git Commit](#anatomy-of-a-git-commit)
3. [Writing Good Commit Messages](#writing-good-commit-messages)
4. [Conventional Commits](#conventional-commits)
   - [Why Conventional Commits Are Good](#why-conventional-commits-are-good)
5. [Atomic Commits](#atomic-commits)
6. [History Management](#history-management)
7. [Amending and Rewriting Commits](#amending-and-rewriting-commits)
8. [Squashing Commits](#squashing-commits)
9. [Git Blame and Annotate](#git-blame-and-annotate)
10. [Commit Signing](#commit-signing)
11. [Best Practices Summary](#best-practices-summary)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

## Overview

This document covers how Git commits work internally, how to write clear and meaningful commit messages, and how to navigate, manage, and rewrite project history. Understanding commits at a structural level — and adopting disciplined conventions around them — leads to repositories that are easier to review, debug, bisect, and maintain.

### Target Audience

- Developers writing commits daily and reviewing pull requests
- Tech leads establishing commit message conventions for their teams
- DevOps engineers building automation around commit metadata (changelogs, release notes)
- Open-source contributors following project commit standards

### Scope

- Git commit object internals — SHA-1 hash, tree, parent pointers, author/committer
- Commit message formatting rules and the imperative mood convention
- Conventional Commits specification and commit types
- Atomic commits and the one-logical-change-per-commit principle
- Navigating history with `git log`, `git shortlog`, and `git reflog`
- Amending, interactive rebase, squashing, and history cleanup
- Tracking changes with `git blame` and `git annotate`
- Commit signing overview (GPG/SSH)

## Anatomy of a Git Commit

A Git commit is an immutable snapshot of the entire project at a point in time. Internally, it is a small object stored in the `.git/objects` directory, addressed by a SHA-1 (or SHA-256) hash of its contents.

### Commit Object Structure

```
┌──────────────────────────────────────────────────────┐
│                   Commit Object                       │
│                                                       │
│  tree      a1b2c3d...   (pointer to root tree object)│
│  parent    f4e5d6c...   (pointer to parent commit)   │
│  parent    b7a8c9d...   (second parent, if merge)    │
│  author    Alice <alice@example.com> 1700000000 +0000│
│  committer Bob <bob@example.com>     1700000100 +0000│
│                                                       │
│  Add user authentication module                       │
│                                                       │
│  Implement JWT-based login and session management.    │
│  Includes unit tests and updated API documentation.   │
└──────────────────────────────────────────────────────┘
```

### How Objects Relate

Every commit points to a **tree object** that represents the top-level directory. Trees point to **blob objects** (file contents) and other trees (subdirectories). This forms a Merkle tree — any change to any file produces a new tree hash all the way up to the root.

```
  commit (abc123)
    │
    ▼
  tree (root)
    ├── blob  README.md
    ├── blob  main.py
    └── tree  src/
          ├── blob  app.py
          └── blob  utils.py
```

### SHA-1 Hash

The SHA-1 hash uniquely identifies a commit based on:

| Input | Description |
|---|---|
| **Tree hash** | The root tree snapshot |
| **Parent hash(es)** | Zero parents (initial commit), one (normal), or two+ (merge) |
| **Author** | Name, email, and timestamp of the person who wrote the change |
| **Committer** | Name, email, and timestamp of the person who applied the commit |
| **Message** | The full commit message (subject + body) |

Changing **any** of these inputs produces an entirely different SHA-1 hash. This is why rebasing or amending a commit creates a new commit object rather than modifying the existing one.

### Parent Pointers and the Commit DAG

Commits form a **Directed Acyclic Graph (DAG)** through their parent pointers. Each commit references its parent(s), creating a chain that represents the project's history.

```
Initial commit       Linear history          Merge commit

    A                A ◄── B ◄── C           A ◄── B ◄── D
    │                                               │     │
    (no parent)                               C ◄───┘     │
                                              │           │
                                              └───────────┘
                                              (D has two parents: B and C)
```

### Viewing Commit Details

```bash
# Show the full commit object
git cat-file -p HEAD

# Show commit details with diff
git show HEAD

# Show only the commit metadata (no diff)
git log -1 --format=fuller HEAD
```

The `--format=fuller` output distinguishes **author** (who wrote the change) from **committer** (who applied it) — these differ in workflows that use cherry-pick, rebase, or patches sent via email.

## Writing Good Commit Messages

A well-written commit message explains **what** changed and **why**. The diff shows *how* it changed — the message provides the context the diff cannot.

### The Seven Rules

| Rule | Guideline |
|---|---|
| 1 | Separate subject from body with a blank line |
| 2 | Limit the subject line to 50 characters |
| 3 | Capitalize the subject line |
| 4 | Do not end the subject line with a period |
| 5 | Use the imperative mood in the subject line |
| 6 | Wrap the body at 72 characters |
| 7 | Use the body to explain *what* and *why*, not *how* |

### Subject Line

The subject line is the most important part of the message. It appears in `git log --oneline`, pull request titles, and notification emails.

```
✅ Good subjects:
  Add rate limiting to authentication endpoint
  Fix null pointer exception in order processing
  Remove deprecated v1 API routes

❌ Bad subjects:
  added stuff                          (vague, past tense)
  fix.                                 (period, too short)
  Update the user service to fix the bug that caused issues when
  processing orders with special characters    (too long)
```

### Imperative Mood

Write the subject line as if completing the sentence: *"If applied, this commit will..."*

```
If applied, this commit will...
  ✅ Add rate limiting to authentication endpoint
  ✅ Fix null pointer exception in order processing
  ✅ Remove deprecated v1 API routes

  ❌ Added rate limiting...       (past tense)
  ❌ Adding rate limiting...      (gerund)
  ❌ Adds rate limiting...        (third person)
```

### Commit Message Body

Use the body for context that is not obvious from the diff:

```
Fix race condition in session cleanup

The session cleanup goroutine was reading the session map
without holding the read lock, causing intermittent panics
under high concurrency (> 500 concurrent users).

Switch from sync.Mutex to sync.RWMutex and acquire a read
lock before iterating the map. Write lock is still used only
for deletions.

Fixes #1234
```

### What to Include in the Body

- ✅ **Why** the change is necessary (motivation, problem description)
- ✅ **What** the change does at a high level
- ✅ Side effects or trade-offs the reviewer should know about
- ✅ References to issues, tickets, or related pull requests
- ❌ Do not describe *how* the code works line by line — that is what the diff is for

## Conventional Commits

The [Conventional Commits](https://www.conventionalcommits.org/) specification provides a lightweight convention on top of commit messages. It creates an explicit commit history that makes it easier to generate changelogs, trigger automated releases, and communicate the nature of changes.

### Format

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

### Commit Types

| Type | Purpose | Example |
|---|---|---|
| **feat** | A new feature | `feat(auth): add OAuth2 login flow` |
| **fix** | A bug fix | `fix(cart): resolve quantity update race condition` |
| **docs** | Documentation only changes | `docs(api): add rate limiting section to README` |
| **style** | Formatting, semicolons, whitespace (no logic change) | `style(lint): apply Prettier formatting rules` |
| **refactor** | Code change that neither fixes a bug nor adds a feature | `refactor(db): extract connection pooling into module` |
| **test** | Adding or updating tests | `test(auth): add integration tests for JWT refresh` |
| **chore** | Maintenance tasks (deps, tooling, config) | `chore(deps): upgrade express to 4.18.2` |
| **perf** | Performance improvement | `perf(query): add index on orders.customer_id` |
| **ci** | CI/CD configuration changes | `ci(github): add codeql analysis workflow` |
| **build** | Build system or external dependency changes | `build(docker): optimize multi-stage Dockerfile` |

### Breaking Changes

Indicate breaking changes with a `!` after the type/scope or with a `BREAKING CHANGE:` footer:

```
feat(api)!: change authentication endpoint response format

BREAKING CHANGE: The /auth/login endpoint now returns a JWT
token in the response body instead of setting a cookie. All
clients must update their authentication handling.
```

### Scopes

Scopes are optional and project-specific. Common patterns:

```
feat(auth):       # authentication module
fix(api):         # API layer
docs(readme):     # README documentation
refactor(db):     # database layer
test(e2e):        # end-to-end tests
ci(github):       # GitHub Actions workflows
```

### Why Conventional Commits Are Good

Conventional Commits transform commit history from an unstructured log into a **machine-readable, human-scannable record** of project evolution. The structured format unlocks automation, improves team communication, and makes large repositories maintainable.

#### 1. Automated Semantic Versioning

The commit type directly maps to [Semantic Versioning](https://semver.org/) bumps — no manual version decisions required:

```
Commit Type         → Version Bump     → Example

fix(auth): ...      → PATCH  (1.2.3 → 1.2.4)    Bug fix, no API change
feat(api): ...      → MINOR  (1.2.4 → 1.3.0)    New feature, backward compatible
feat(api)!: ...     → MAJOR  (1.3.0 → 2.0.0)    Breaking change
BREAKING CHANGE:    → MAJOR  (1.3.0 → 2.0.0)    Breaking change (footer)
docs, style, ci...  → (no release)               Non-functional changes
```

Tools like [semantic-release](https://github.com/semantic-release/semantic-release) and [release-please](https://github.com/googleapis/release-please) read your commit history and **automatically determine the next version, generate release notes, and publish the release** — with zero manual intervention.

```bash
# semantic-release reads commits since the last release and decides the version
npx semantic-release
# Output:
# Analyzing commits since v1.2.3...
#   feat(auth): add OAuth2 login flow  →  minor
#   fix(cart): resolve quantity bug     →  patch
# → Next release: v1.3.0
# → Publishing v1.3.0 to npm...
# → Creating GitHub release v1.3.0...
```

#### 2. Automated Changelog Generation

Structured commits produce **meaningful, categorized changelogs** without manual effort:

```markdown
# Changelog

## [1.3.0] - 2025-03-15

### Features
- **auth:** add OAuth2 login flow (#42)
- **api:** add rate limiting endpoint (#45)

### Bug Fixes
- **cart:** resolve quantity update race condition (#43)
- **payment:** fix decimal rounding in tax calculation (#44)

### Documentation
- **readme:** update installation instructions (#46)
```

Tools that generate changelogs from conventional commits:

| Tool | Language | Description |
|---|---|---|
| [conventional-changelog](https://github.com/conventional-changelog/conventional-changelog) | Node.js | Generate changelogs from conventional commit history |
| [git-cliff](https://github.com/orhun/git-cliff) | Rust | Highly customizable changelog generator |
| [auto-changelog](https://github.com/cookpete/auto-changelog) | Node.js | Simple changelog generation from commit history |
| [release-please](https://github.com/googleapis/release-please) | Node.js | Google's automated release and changelog tool |

#### 3. Scannable Git History

Conventional commits make `git log` output instantly parseable:

```bash
# Filter only features added this sprint
git log --oneline --grep="^feat" --since="2 weeks ago"

# Find all bug fixes in the auth module
git log --oneline --grep="^fix(auth)"

# Count changes by type for a retrospective
git log --oneline --since="2025-01-01" | grep -c "^.\{8\} feat"   # features
git log --oneline --since="2025-01-01" | grep -c "^.\{8\} fix"    # bug fixes
```

#### 4. CI/CD Pipeline Triggers

Commit types can drive CI/CD behavior — skip expensive builds for documentation changes, trigger deployment only for features and fixes:

```yaml
# GitHub Actions: skip CI for docs-only changes
on:
  push:
    branches: [main]

jobs:
  build:
    if: "!startsWith(github.event.head_commit.message, 'docs')"
    runs-on: ubuntu-latest
    steps:
      - run: npm run build && npm test
```

#### 5. Team Communication and Code Review

Conventional commits tell reviewers **what kind of change to expect** before they open the diff:

```
feat(auth): add OAuth2 login flow
  → "This adds a new feature — I should review for correctness and completeness"

fix(cart): resolve quantity update race condition
  → "This fixes a bug — I should verify the fix and check for regressions"

refactor(db): extract connection pooling into module
  → "This restructures code — behavior should be identical, focus on design"

chore(deps): upgrade express to 4.18.2
  → "This is a dependency update — check for breaking changes in the changelog"
```

#### 6. Tooling Ecosystem

Interactive tools help developers write conventional commits without memorizing the format:

| Tool | Description |
|---|---|
| [commitizen](https://github.com/commitizen/cz-cli) | Interactive CLI that prompts for type, scope, and description |
| [commitlint](https://commitlint.js.org/) | Lint commit messages against conventional commit rules |
| [cz-git](https://cz-git.qbb.sh/) | Lightweight, customizable commitizen adapter |
| [conventional-pre-commit](https://github.com/compilerla/conventional-pre-commit) | Pre-commit hook to validate conventional commit messages |

```bash
# commitizen: interactive conventional commit prompt
npx cz
# ? Select the type of change: feat
# ? What is the scope: auth
# ? Short description: add OAuth2 login flow
# ? Longer description: (optional)
# ? Breaking changes: (optional)
# → Creates commit: feat(auth): add OAuth2 login flow

# commitlint: validate commit messages in CI
echo "feat(auth): add OAuth2 login flow" | npx commitlint
# → ✅ Passed

echo "added stuff" | npx commitlint
# → ❌ subject may not be empty
# → ❌ type may not be empty
```

### Benefits Summary

| Benefit | Without Conventional Commits | With Conventional Commits |
|---|---|---|
| **Versioning** | Manual, error-prone | Automated via commit type |
| **Changelogs** | Written by hand before each release | Auto-generated from commit history |
| **Code review** | Must read the diff to understand intent | Commit type signals the nature of change |
| **History search** | Unstructured — `git log --grep` is unreliable | Structured — filter by type, scope, breaking |
| **CI/CD** | All commits trigger full pipeline | Pipeline adapts based on commit type |
| **Onboarding** | New developers guess at conventions | Clear, documented, enforced format |

## Atomic Commits

An atomic commit contains exactly **one logical change**. It is the smallest unit of work that makes sense on its own — the repository is in a valid, working state both before and after the commit.

### What Makes a Commit Atomic

```
✅ Atomic (one logical change):
  commit 1: Add User model and database migration
  commit 2: Add user registration endpoint
  commit 3: Add input validation for registration fields
  commit 4: Add unit tests for user registration

❌ Non-atomic (multiple unrelated changes):
  commit 1: Add user registration, fix cart bug, update README
```

### Benefits of Small, Atomic Commits

| Benefit | Explanation |
|---|---|
| **Easier code review** | Reviewers can understand each change in isolation |
| **Simpler debugging** | `git bisect` can pinpoint the exact commit that introduced a bug |
| **Clean reverts** | `git revert <commit>` removes only one logical change, with no collateral damage |
| **Better blame output** | `git blame` shows meaningful context for each line |
| **Flexible cherry-picking** | Individual features or fixes can be moved between branches independently |

### The Staging Area Enables Atomic Commits

Git's staging area (`git add -p`) allows you to split a messy working directory into multiple focused commits:

```bash
# Stage only specific hunks of changes interactively
git add -p

# Stage specific files
git add src/models/user.py src/migrations/001_create_users.py

# Commit the staged changes
git commit -m "feat(user): add User model and migration"

# Stage and commit the next logical change
git add src/routes/auth.py
git commit -m "feat(auth): add user registration endpoint"
```

## History Management

Git provides powerful tools for navigating and understanding project history.

### git log

The primary tool for viewing commit history. Its flexibility comes from formatting and filtering options.

```bash
# Default log (full details)
git log

# One-line summary
git log --oneline

# Graphical branch visualization
git log --oneline --graph --all

# Filter by author
git log --author="Alice"

# Filter by date range
git log --after="2024-01-01" --before="2024-06-30"

# Filter by file
git log -- src/auth/login.py

# Filter by commit message content
git log --grep="fix.*race condition"

# Show diff with each commit
git log -p

# Limit number of results
git log -10
```

### Graphical History Example

```bash
$ git log --oneline --graph --all
```

```
* e4f5a6b (HEAD -> main) Merge branch 'feature/auth'
|\
| * c3d4e5f (feature/auth) Add OAuth2 provider integration
| * a1b2c3d Add login endpoint
|/
* 9f8e7d6 Update dependencies
* 7a6b5c4 Initial project setup
```

This output corresponds to the following DAG:

```
    7a6b5c4 ◄── 9f8e7d6 ◄──┬── a1b2c3d ◄── c3d4e5f
                            │                   │
                            └───── e4f5a6b ◄────┘
                                  (merge)
```

### git shortlog

Summarizes commit history grouped by author — useful for release notes and contribution tracking.

```bash
# Group commits by author
git shortlog -sn

# Example output:
#    42  Alice <alice@example.com>
#    31  Bob <bob@example.com>
#    18  Charlie <charlie@example.com>

# Group by author with commit subjects
git shortlog --no-merges HEAD~20..HEAD
```

### git reflog

The **reflog** (reference log) records every change to `HEAD` and branch tips in your local repository — including commits, rebases, resets, and checkouts. It is your safety net for recovering lost work.

```bash
# Show the reflog for HEAD
git reflog

# Example output:
# e4f5a6b HEAD@{0}: commit: Add user authentication
# 9f8e7d6 HEAD@{1}: rebase -i (finish): returning to refs/heads/main
# 7a6b5c4 HEAD@{2}: rebase -i (squash): Update dependencies
# a1b2c3d HEAD@{3}: checkout: moving from feature/auth to main
```

```bash
# Recover a commit lost during rebase or reset
git checkout HEAD@{3}

# Create a new branch from a lost commit
git branch recovered-work HEAD@{3}

# Reflog entries expire after 90 days by default
git reflog expire --expire=90.days --all
```

> **Key point:** The reflog is local only — it is not pushed to remotes and is not shared with other developers.

## Amending and Rewriting Commits

Git allows you to modify recent commits and rewrite history before sharing it with others. These operations create new commit objects (new SHA-1 hashes) — they do not modify existing ones.

### git commit --amend

Replaces the most recent commit with a new one. Use it to fix typos in the message or add forgotten changes.

```bash
# Fix the commit message only
git commit --amend -m "fix(auth): correct token expiration logic"

# Add forgotten files to the last commit
git add forgotten-file.py
git commit --amend --no-edit

# Change the author of the last commit
git commit --amend --author="Alice <alice@example.com>"
```

```
Before amend:            After amend:

A ◄── B ◄── C           A ◄── B ◄── C'
             │                        │
           (HEAD)                   (HEAD)

C and C' have different SHA-1 hashes.
C still exists in the reflog but is no longer reachable from any branch.
```

> **Warning:** Do not amend commits that have already been pushed to a shared branch. Other developers who have based work on the original commit will encounter conflicts.

### Interactive Rebase for History Cleanup

Interactive rebase (`git rebase -i`) lets you edit, reorder, squash, drop, or reword multiple commits in a branch before merging.

```bash
# Rebase the last 4 commits interactively
git rebase -i HEAD~4
```

Git opens an editor with a list of commits:

```
pick a1b2c3d Add login endpoint
pick b2c3d4e Fix typo in login route
pick c3d4e5f Add OAuth2 provider integration
pick d4e5f6a Update OAuth2 tests

# Commands:
# p, pick   = use commit as-is
# r, reword = use commit, but edit the message
# e, edit   = use commit, but stop for amending
# s, squash = merge into previous commit (keep message)
# f, fixup  = merge into previous commit (discard message)
# d, drop   = remove commit entirely
```

### Common Interactive Rebase Operations

| Operation | Command | Use Case |
|---|---|---|
| **Reword** | `r` / `reword` | Fix a commit message without changing code |
| **Squash** | `s` / `squash` | Combine a commit with the one before it, keeping both messages |
| **Fixup** | `f` / `fixup` | Combine a commit with the one before it, discarding its message |
| **Edit** | `e` / `edit` | Pause rebase at a commit so you can amend it or split it |
| **Drop** | `d` / `drop` | Remove a commit entirely from history |
| **Reorder** | Move lines | Change the order in which commits are applied |

### Example: Clean Up Before Merging

```bash
# Original messy history on feature branch:
# d4e5f6a WIP
# c3d4e5f fix tests
# b2c3d4e Add OAuth2 provider
# a1b2c3d Add login endpoint

git rebase -i HEAD~4

# Edit the rebase plan:
pick a1b2c3d Add login endpoint
squash b2c3d4e Add OAuth2 provider
fixup c3d4e5f fix tests
fixup d4e5f6a WIP

# Result: one clean commit with a clear message
# "Add login endpoint with OAuth2 provider integration"
```

## Squashing Commits

Squashing combines multiple commits into a single commit. This is useful for cleaning up work-in-progress commits before merging a feature branch into main.

### When to Squash

- ✅ WIP, "fix typo", and "address review feedback" commits before merging
- ✅ A feature branch with many small incremental commits that tell a single story
- ✅ When the project requires a linear history on the main branch
- ❌ Do not squash across unrelated logical changes — keep separate atomic commits
- ❌ Do not squash commits from a shared branch that others have based work on

### Squash Methods

#### Method 1: Interactive Rebase

```bash
# Squash the last 3 commits into one
git rebase -i HEAD~3

# In the editor, mark commits to squash:
pick a1b2c3d Add login endpoint
squash b2c3d4e Add OAuth2 support
squash c3d4e5f Add tests for login
```

#### Method 2: Soft Reset

```bash
# Reset to the commit before the 3 you want to squash
git reset --soft HEAD~3

# All changes are now staged — create a single new commit
git commit -m "feat(auth): add login endpoint with OAuth2 and tests"
```

### Merge Commit vs. Squash Merge

When merging a pull request, platforms like GitHub offer different merge strategies:

```
Merge commit (--no-ff):           Squash merge:

* e4f5a6b Merge PR #42           * e4f5a6b feat(auth): add login
|\                                          (all changes in one commit)
| * c3d4e5f Add tests            |
| * b2c3d4e Add OAuth2           |
| * a1b2c3d Add endpoint         |
|/                                |
* 9f8e7d6 Previous commit        * 9f8e7d6 Previous commit
```

| Strategy | Preserves Branch History | Main Branch Linearity | When to Use |
|---|---|---|---|
| **Merge commit** | ✅ Yes — all individual commits visible | ❌ No — creates merge bubbles | When individual commit history matters |
| **Squash merge** | ❌ No — branch commits collapsed into one | ✅ Yes — clean linear history | When only the final result matters |
| **Rebase merge** | ✅ Yes — commits replayed onto main | ✅ Yes — linear, no merge commit | When individual commits are clean and atomic |

## Git Blame and Annotate

`git blame` and `git annotate` show who last modified each line of a file and in which commit. These tools are essential for understanding why code was written a certain way and who to ask about it.

### Basic Usage

```bash
# Show blame for an entire file
git blame src/auth/login.py

# Example output:
# a1b2c3d4 (Alice  2024-03-15 14:30 +0000  1) def login(username, password):
# a1b2c3d4 (Alice  2024-03-15 14:30 +0000  2)     user = find_user(username)
# f5e6d7c8 (Bob    2024-04-02 09:15 +0000  3)     if not user or not verify(password, user.hash):
# f5e6d7c8 (Bob    2024-04-02 09:15 +0000  4)         raise AuthError("Invalid credentials")
# c9d0e1f2 (Alice  2024-05-10 11:00 +0000  5)     return create_session(user)
```

### Advanced Blame Options

```bash
# Blame a specific line range (lines 10-20)
git blame -L 10,20 src/auth/login.py

# Ignore whitespace changes
git blame -w src/auth/login.py

# Detect lines moved or copied from other files
git blame -C src/auth/login.py

# Detect lines moved across commits (more aggressive)
git blame -C -C -C src/auth/login.py

# Show the commit that last changed each line before a specific commit
git blame <commit>^ -- src/auth/login.py
```

### Ignoring Formatting Commits

Large-scale formatting changes (e.g., applying a linter) pollute blame output. Git supports a `.git-blame-ignore-revs` file to skip specific commits:

```bash
# .git-blame-ignore-revs
# Apply Prettier formatting across entire codebase
a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0

# Apply Black formatter to Python files
b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1
```

```bash
# Tell git blame to use the ignore file
git blame --ignore-revs-file .git-blame-ignore-revs src/auth/login.py

# Configure git to always use the ignore file
git config blame.ignoreRevsFile .git-blame-ignore-revs
```

### git annotate

`git annotate` is functionally equivalent to `git blame` but outputs in a slightly different format. In practice, `git blame` is used far more widely.

```bash
git annotate src/auth/login.py
```

## Commit Signing

Signing commits proves that a commit was created by the person who claims to be the author. This is especially important for open-source projects and regulated environments.

### Why Sign Commits

| Reason | Explanation |
|---|---|
| **Identity verification** | Proves the commit author is who they claim to be |
| **Tamper detection** | Detects if a commit has been modified after signing |
| **Compliance** | Required by some organizations and regulatory frameworks |
| **Trust** | GitHub and GitLab display a "Verified" badge on signed commits |

### Signing Methods

Git supports two signing methods:

| Method | Key Type | Setup Complexity | Best For |
|---|---|---|---|
| **GPG signing** | GPG key pair | Moderate | Established teams, PGP web of trust |
| **SSH signing** | SSH key pair | Low | Developers already using SSH keys for Git access |

### Quick Overview

```bash
# GPG signing
git config --global commit.gpgsign true
git config --global user.signingkey <GPG-KEY-ID>
git commit -S -m "feat(auth): add signed login endpoint"

# SSH signing (Git 2.34+)
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git commit -S -m "feat(auth): add signed login endpoint"

# Verify a signed commit
git log --show-signature -1
```

> **For detailed setup instructions** — including key generation, GitHub/GitLab configuration, and verification workflows — see [Security and Signing](08-SECURITY-AND-SIGNING.md).

## Best Practices Summary

### Commit Messages

- ✅ Use the imperative mood in the subject line ("Add feature", not "Added feature")
- ✅ Keep the subject line under 50 characters
- ✅ Separate subject from body with a blank line
- ✅ Wrap the body at 72 characters
- ✅ Explain *what* and *why* in the body, not *how*
- ✅ Reference related issues and tickets
- ❌ Do not end the subject line with a period
- ❌ Do not write vague messages ("fix stuff", "WIP", "updates")

### Commit Hygiene

- ✅ Make atomic commits — one logical change per commit
- ✅ Ensure each commit leaves the repository in a valid, buildable state
- ✅ Use `git add -p` to stage specific changes when your working directory has multiple unrelated edits
- ✅ Clean up history with interactive rebase before merging feature branches
- ❌ Do not commit generated files, build artifacts, or secrets
- ❌ Do not amend or rebase commits that have been pushed to shared branches

### History Management

- ✅ Use `git log --oneline --graph` to visualize branch topology
- ✅ Use `git reflog` to recover lost commits after rebase or reset
- ✅ Use `.git-blame-ignore-revs` to exclude formatting commits from blame
- ✅ Sign commits in projects that require verified authorship
- ❌ Do not force push to main or other shared branches without coordination
- ❌ Do not rewrite history that other developers have based work on

## Next Steps

Continue to [Merging and Rebasing](03-MERGING-AND-REBASING.md) to learn about merge strategies (fast-forward, three-way, recursive), rebasing workflows, conflict resolution techniques, and how to choose the right integration strategy for your team.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial commits and history documentation |
