# Git Branching Strategies

## Table of Contents

1. [Overview](#overview)
2. [Git Flow](#git-flow)
3. [GitHub Flow](#github-flow)
4. [GitLab Flow](#gitlab-flow)
5. [Trunk-Based Development](#trunk-based-development)
6. [Release Branching Strategies](#release-branching-strategies)
7. [Comparison of Strategies](#comparison-of-strategies)
8. [Choosing the Right Strategy](#choosing-the-right-strategy)
9. [Branch Naming Conventions](#branch-naming-conventions)
10. [Branch Protection Rules](#branch-protection-rules)
11. [Best Practices](#best-practices)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

## Overview

This document covers the most widely used Git branching strategies — from the structured, multi-branch Git Flow model to the minimalist trunk-based development approach. Choosing the right strategy affects release cadence, code quality, team velocity, and CI/CD complexity.

### Target Audience

- Developers adopting a branching model for a new project
- Tech leads and engineering managers standardizing team workflows
- DevOps engineers designing CI/CD pipelines around branch conventions
- Open-source maintainers managing contributions from external collaborators

### Scope

- Git Flow — feature, develop, release, hotfix, and main branches
- GitHub Flow — simplified main + feature branch model
- GitLab Flow — environment branches and release branches
- Trunk-Based Development — short-lived branches and feature flags
- Release branching strategies — release branches, tagging, and versioning
- Comparison matrix across all strategies
- Decision framework for choosing a strategy
- Branch naming conventions and protection rules

## Git Flow

Git Flow, introduced by Vincent Driessen in 2010, is a structured branching model designed for projects with scheduled releases. It uses long-lived branches to separate development, release preparation, and production-ready code.

### Branch Roles

| Branch | Lifetime | Purpose |
|---|---|---|
| **main** | Permanent | Always reflects production-ready state; every commit is a release |
| **develop** | Permanent | Integration branch for features; reflects the latest delivered development changes |
| **feature/*** | Temporary | New features branched from and merged back into develop |
| **release/*** | Temporary | Release preparation; branched from develop, merged into main and develop |
| **hotfix/*** | Temporary | Urgent production fixes; branched from main, merged into main and develop |

### Branch Diagram

```
main        ●─────────────────────●───────────────────────●──────────●
            │                     ▲                       ▲          ▲
            │                     │ merge                 │          │
            │               release/1.0                   │     hotfix/1.0.1
            │              ╱             ╲                 │     ╱          ╲
develop     ●────●────●───●───●───────────●───●────●─────●────●────────────●───●
                 │         ▲              ▲        │          ▲
                 │         │ merge        │        │          │ merge
            feature/login  │         feature/cart  │     feature/search
                 ╲         │              ╲        │          ╲
                  ●───●───●                ●──●───●            ●───●───●
```

### Workflow Details

```
Feature Development:

  develop          feature/login
     │                  │
     ├── git checkout ──▶ Create branch from develop
     │                  │
     │                  ├── Implement feature
     │                  ├── Write tests
     │                  ├── Commit changes
     │                  │
     ◀── merge ─────────┤  Merge back into develop (via PR)
     │                  │
     │              Delete branch
```

```
Release Preparation:

  develop       release/1.0           main
     │               │                  │
     ├── checkout ──▶│                  │
     │               ├── Bump version   │
     │               ├── Fix bugs       │
     │               ├── Update docs    │
     │               │                  │
     ◀── merge ──────┤── merge ────────▶│── tag v1.0
     │               │                  │
     │           Delete branch          │
```

```
Hotfix:

  main         hotfix/1.0.1         develop
     │               │                  │
     ├── checkout ──▶│                  │
     │               ├── Fix bug        │
     │               ├── Bump patch     │
     │               │                  │
     ◀── merge ──────┤── merge ────────▶│
     │               │                  │
  tag v1.0.1     Delete branch          │
```

### When to Use Git Flow

- ✅ Projects with scheduled, versioned releases (e.g., quarterly releases)
- ✅ Software that must maintain multiple release versions simultaneously
- ✅ Teams that need formal release preparation and stabilization phases
- ✅ Packaged software, mobile apps, or firmware with long QA cycles
- ❌ Not ideal for continuous deployment or fast-moving web applications
- ❌ Overhead is too high for small teams or solo developers

## GitHub Flow

GitHub Flow is a lightweight branching model built around a single long-lived branch — `main`. All work happens on short-lived feature branches that are merged into `main` via pull requests. It is designed for continuous delivery.

### Branch Roles

| Branch | Lifetime | Purpose |
|---|---|---|
| **main** | Permanent | Always deployable; represents the latest production-ready code |
| **feature/*** | Temporary | Short-lived branches for features, fixes, or experiments |

### Branch Diagram

```
main     ●────────●─────────────●────────────●──────────●────────●
              ╲         ▲            ╲           ▲           ▲
               ╲        │ PR          ╲          │ PR        │ PR
          feature/login  │       feature/cart     │     fix/typo
                ╲        │              ╲         │          ╲
                 ●──●───●               ●───●───●            ●
```

### Workflow

```
GitHub Flow — Step by Step:

  1. Create branch     2. Add commits      3. Open PR
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │ git checkout │    │ Make changes │    │ Open Pull    │
  │   -b feature │───▶│ git commit   │───▶│ Request on   │
  │              │    │ git push     │    │ GitHub       │
  └──────────────┘    └──────────────┘    └──────┬───────┘
                                                 │
  6. Delete branch    5. Merge & Deploy    4. Review
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │ Delete the   │    │ Merge PR     │    │ Team reviews │
  │ feature      │◀───│ Deploy to    │◀───│ code, CI     │
  │ branch       │    │ production   │    │ checks pass  │
  └──────────────┘    └──────────────┘    └──────────────┘
```

### Key Principles

1. **Anything in `main` is deployable** — `main` is always in a releasable state
2. **Branch off `main`** — create a descriptively named branch for every change
3. **Commit early and often** — push to the remote branch regularly for visibility
4. **Open a pull request early** — use PRs for discussion, even before the code is ready
5. **Merge only after review** — require at least one approval and passing CI checks
6. **Deploy immediately after merge** — continuous deployment keeps `main` honest

### When to Use GitHub Flow

- ✅ Web applications and SaaS products with continuous deployment
- ✅ Small to mid-sized teams that deploy frequently (daily or more)
- ✅ Open-source projects using fork-and-pull contribution models
- ✅ Teams that want simplicity and fast iteration
- ❌ Not ideal when you need to maintain multiple release versions
- ❌ Insufficient if you need a formal release stabilization phase

## GitLab Flow

GitLab Flow is a middle ground between the simplicity of GitHub Flow and the structure of Git Flow. It introduces environment branches or release branches to support deployment pipelines and versioned releases.

### Environment Branches Model

In the environment branches model, each environment (staging, pre-production, production) has a dedicated long-lived branch. Code flows downstream from `main` through each environment branch.

```
main             ●────●────●────●────●────●────●────●
                      │         │              │
                      ▼         ▼              ▼
pre-production   ─────●─────────●──────────────●────
                      │         │              │
                      ▼         ▼              ▼
production       ─────●─────────●──────────────●────

Code flows downstream:  main → pre-production → production
Fixes flow upstream:    Fix on main, then cherry-pick or merge downstream
```

### Release Branches Model

In the release branches model, each major or minor version gets its own long-lived branch. Bug fixes are applied to `main` first, then cherry-picked into the relevant release branch.

```
main              ●────●────●────●────●────●────●────●
                       │              │
                       ▼              ▼
release/1.0       ─────●────●         │
                       │    │         │
                  v1.0.0  v1.0.1      │
                                      ▼
release/2.0                      ─────●────●────●
                                      │    │    │
                                 v2.0.0 v2.0.1 v2.0.2
```

### Branch Roles

| Branch | Model | Lifetime | Purpose |
|---|---|---|---|
| **main** | Both | Permanent | Integration branch; always contains the latest code |
| **feature/*** | Both | Temporary | Short-lived feature and fix branches |
| **pre-production** | Environment | Permanent | Mirrors what is deployed to pre-production |
| **production** | Environment | Permanent | Mirrors what is deployed to production |
| **release/X.Y** | Release | Long-lived | Receives bug fixes for a specific version |

### Key Principles

1. **Use `main` as the starting point** — all feature branches originate from `main`
2. **Merge downstream, cherry-pick upstream** — code flows from `main` to environment or release branches
3. **Fix bugs on `main` first** — then cherry-pick or merge the fix into release or environment branches
4. **Every environment has a branch** — deploy by merging into the environment branch
5. **Use merge requests for everything** — all changes go through code review

### When to Use GitLab Flow

- ✅ Teams that need environment-specific branches for staged deployments
- ✅ Projects that maintain multiple supported release versions
- ✅ Organizations that want more structure than GitHub Flow without Git Flow's complexity
- ✅ CI/CD pipelines that trigger deployments based on branch targets
- ❌ Overkill for small projects that deploy directly from `main`
- ❌ Environment branches add overhead if you have a single deployment target

## Trunk-Based Development

Trunk-Based Development (TBD) is a branching strategy where developers commit directly to a single shared branch — the trunk (usually `main`). Feature branches, if used at all, are very short-lived (hours, not days). This approach relies on feature flags, continuous integration, and small incremental changes.

### Branch Diagram

```
main (trunk)  ●──●──●──●──●──●──●──●──●──●──●──●──●──●──●──●
                  ╲  ▲     ╲ ▲        ╲ ▲
                   ╲ │      ╲│         ╲│
                    ●        ●          ●
              (short-lived feature branches — merged within hours)
```

### How It Works

```
Trunk-Based Development Workflow:

  Developer A                    Developer B
       │                              │
       ├── Pull latest main           ├── Pull latest main
       ├── Create short branch        ├── Commit directly to main
       ├── Small, focused change      │   (if team allows)
       ├── Push and open PR           │
       ├── Review + merge (< 1 day)   │
       │                              │
       ▼                              ▼
  ┌──────────────────────────────────────────────────┐
  │                 main (trunk)                      │
  │  Always building ✅  Always tested ✅              │
  │  Always deployable ✅  Feature flags for WIP ✅    │
  └──────────────────────────────────────────────────┘
       │
       ▼
  Continuous Integration / Continuous Deployment
```

### Feature Flags

Feature flags decouple deployment from release. Code is deployed to production but hidden behind a flag until it is ready.

```
Feature Flag Lifecycle:

  1. Code behind flag     2. Deploy to prod     3. Enable gradually    4. Remove flag
  ┌──────────────┐       ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
  │ if (flag ON) │       │ Code is live │      │ 5% → 25%    │      │ Remove flag  │
  │   newFeature │──────▶│ Flag is OFF  │─────▶│ → 50% → 100%│─────▶│ Clean up old │
  │ else         │       │ Users see    │      │ Monitor      │      │ code path    │
  │   oldFeature │       │ old behavior │      │ metrics      │      │              │
  └──────────────┘       └──────────────┘      └──────────────┘      └──────────────┘
```

### Key Principles

1. **Keep the trunk green** — the trunk must always build and pass all tests
2. **Small, frequent commits** — merge changes at least once per day
3. **Feature flags over feature branches** — hide incomplete work behind flags, not branches
4. **No long-lived branches** — if a branch exists, it lives for hours, not days
5. **Continuous integration is mandatory** — every commit triggers the full test suite

### When to Use Trunk-Based Development

- ✅ Teams practicing continuous deployment with fast CI pipelines
- ✅ Mature engineering teams with strong testing and code review culture
- ✅ High-velocity web applications where speed of delivery is critical
- ✅ Teams using feature flag infrastructure (LaunchDarkly, Unleash, Flagsmith)
- ❌ Difficult for teams without comprehensive automated test coverage
- ❌ Not ideal for open-source projects with many external contributors
- ❌ Risky without a robust feature flag system for incomplete work

## Release Branching Strategies

Regardless of the branching model you use, you need a strategy for managing releases. This section covers the most common approaches to release branches, tagging, and versioning.

### Release Branches

A release branch is created when a set of features is ready for release. It allows final stabilization — bug fixes, documentation updates, version bumps — without blocking new feature development on the main integration branch.

```
Release Branch Lifecycle:

  develop / main
       │
       ├── Feature A merged
       ├── Feature B merged
       ├── Feature C merged
       │
       ├───────────▶ release/2.0     (branch off)
       │                  │
       │                  ├── Fix bug found in QA
       │                  ├── Update changelog
       │                  ├── Bump version to 2.0.0
       │                  │
       ├◀── merge ────────┤           (merge fixes back)
       │                  │
       │             tag v2.0.0       (tag the release)
       │                  │
       │             merge to main    (if using Git Flow)
       │                  │
       │            Delete branch     (or keep for patch releases)
```

### Tagging Releases

Tags mark specific commits as release points. Use annotated tags for releases — they store the tagger, date, and a message.

```bash
# Create an annotated tag
git tag -a v2.0.0 -m "Release version 2.0.0"

# Push tags to remote
git push origin v2.0.0

# List tags matching a pattern
git tag -l "v2.*"
```

### Semantic Versioning

Follow [Semantic Versioning (SemVer)](https://semver.org/) for release tags:

```
  MAJOR.MINOR.PATCH
    │      │     │
    │      │     └── Bug fixes, no API changes          (v2.0.1)
    │      └──────── New features, backward compatible   (v2.1.0)
    └─────────────── Breaking changes                    (v3.0.0)

  Pre-release:    v2.0.0-beta.1, v2.0.0-rc.1
  Build metadata: v2.0.0+build.123
```

### Patch Releases from Release Branches

```
main / develop
     │
     ├── Fix critical bug on main first
     │
     ├── Cherry-pick fix ──▶ release/2.0
     │                           │
     │                      tag v2.0.1
     │
     ├── Cherry-pick fix ──▶ release/1.5   (if still supported)
     │                           │
     │                      tag v1.5.3
```

## Comparison of Strategies

### Strategy Matrix

| Dimension | Git Flow | GitHub Flow | GitLab Flow | Trunk-Based |
|---|---|---|---|---|
| **Long-lived branches** | main, develop | main | main + environment or release | main (trunk) |
| **Feature branches** | Yes (from develop) | Yes (from main) | Yes (from main) | Optional, very short-lived |
| **Release branches** | Yes | No | Optional | No |
| **Hotfix branches** | Yes (from main) | No (fix on feature branch) | No (fix on main, cherry-pick) | No (fix on trunk) |
| **Merge target** | develop → release → main | feature → main | feature → main → environments | feature → main |
| **Best for team size** | 5–20+ developers | 1–15 developers | 5–20+ developers | 2–15+ developers (mature teams) |
| **Release cadence** | Scheduled (weekly, monthly) | Continuous (daily or more) | Continuous or scheduled | Continuous (multiple per day) |
| **Complexity** | High | Low | Medium | Low (but requires discipline) |
| **CI/CD requirement** | Recommended | Required | Required | Mandatory |
| **Multiple versions** | Yes | No | Yes (release branches model) | No |
| **Learning curve** | Steep | Gentle | Moderate | Gentle (process is strict) |

### Deployment Model Fit

| Deployment Model | Recommended Strategy |
|---|---|
| **Continuous deployment** (SaaS, web apps) | GitHub Flow or Trunk-Based Development |
| **Scheduled releases** (mobile apps, packaged software) | Git Flow or GitLab Flow (release branches) |
| **Environment promotion** (staging → production) | GitLab Flow (environment branches) |
| **Multiple supported versions** (libraries, frameworks) | Git Flow or GitLab Flow (release branches) |
| **Open-source with external contributors** | GitHub Flow (fork-and-pull) |
| **Monorepo with many teams** | Trunk-Based Development with feature flags |

## Choosing the Right Strategy

### Decision Tree

```
Start Here
│
├── Do you deploy continuously (multiple times per day)?
│   ├── YES → Is your team mature with strong CI/CD and test coverage?
│   │         ├── YES → Trunk-Based Development
│   │         └── NO  → GitHub Flow
│   │
│   └── NO  → Do you need to maintain multiple release versions?
│             ├── YES → Do you need formal release stabilization?
│             │         ├── YES → Git Flow
│             │         └── NO  → GitLab Flow (release branches)
│             │
│             └── NO  → Do you promote code through environments?
│                       ├── YES → GitLab Flow (environment branches)
│                       └── NO  → GitHub Flow
```

### Decision Matrix

Answer these questions to narrow your choice:

| Question | Git Flow | GitHub Flow | GitLab Flow | Trunk-Based |
|---|---|---|---|---|
| Do you release on a fixed schedule? | ✅ | ❌ | ✅ | ❌ |
| Do you deploy continuously? | ❌ | ✅ | ✅ | ✅ |
| Do you maintain multiple versions? | ✅ | ❌ | ✅ | ❌ |
| Is your team < 5 people? | ❌ | ✅ | ❌ | ✅ |
| Do you have comprehensive test coverage? | — | — | — | ✅ Required |
| Do external contributors submit PRs? | ✅ | ✅ | ✅ | ❌ |
| Do you want minimal process overhead? | ❌ | ✅ | ❌ | ✅ |
| Do you use feature flags? | Optional | Optional | Optional | ✅ Required |

### Common Migration Paths

```
Starting Point          Trigger                     Target Strategy
─────────────────────────────────────────────────────────────────────
No formal strategy  →   Team growing, need process  →  GitHub Flow
GitHub Flow         →   Need staged deployments     →  GitLab Flow
GitHub Flow         →   Need release versions       →  Git Flow
Git Flow            →   Moving to continuous deploy  →  GitHub Flow
Git Flow            →   Want less overhead           →  Trunk-Based
Any strategy        →   High-velocity team + flags   →  Trunk-Based
```

## Branch Naming Conventions

Consistent branch names improve readability, enable CI/CD automation, and make repository navigation easier.

### Recommended Prefixes

| Prefix | Purpose | Example |
|---|---|---|
| **feature/** | New functionality | `feature/user-authentication` |
| **bugfix/** | Non-urgent bug fixes | `bugfix/cart-total-rounding` |
| **hotfix/** | Urgent production fixes | `hotfix/payment-timeout` |
| **release/** | Release preparation | `release/2.1.0` |
| **chore/** | Maintenance tasks (deps, CI config) | `chore/upgrade-node-18` |
| **docs/** | Documentation changes | `docs/api-reference-update` |
| **test/** | Adding or updating tests | `test/payment-edge-cases` |
| **refactor/** | Code restructuring without behavior change | `refactor/extract-auth-module` |

### Naming Rules

```
Branch Naming Format:

  <prefix>/<short-description>
  <prefix>/<ticket-id>-<short-description>

  ✅ Good                              ❌ Bad
  ─────────────────────────────────    ─────────────────────────────────
  feature/user-authentication          new-feature
  feature/PROJ-123-user-auth           Feature_User_Auth
  bugfix/cart-total-rounding           fix
  hotfix/payment-timeout               john/stuff
  release/2.1.0                        temp
  chore/upgrade-node-18                test123
```

### Rules

- ✅ Use lowercase letters, numbers, and hyphens
- ✅ Use a prefix that describes the type of work
- ✅ Include a ticket or issue ID when available
- ✅ Keep names short but descriptive (3–5 words after the prefix)
- ❌ Do not use spaces, underscores, or uppercase letters
- ❌ Do not use personal names as branch names (e.g., `john/feature`)
- ❌ Do not use generic names like `fix`, `test`, or `temp`

## Branch Protection Rules

Branch protection rules prevent accidental or unauthorized changes to critical branches. Configure these on your hosting platform (GitHub, GitLab, Bitbucket).

### Recommended Rules for `main`

| Rule | Purpose | Recommended Setting |
|---|---|---|
| **Require pull request** | No direct pushes to main | ✅ Enabled |
| **Required reviewers** | Minimum number of approvals before merge | 1–2 reviewers |
| **Dismiss stale reviews** | Re-require approval after new commits | ✅ Enabled |
| **Require status checks** | CI must pass before merge | ✅ Enabled (build + test) |
| **Require branch up-to-date** | Branch must be current with main | ✅ Enabled |
| **Require signed commits** | All commits must be GPG-signed | Optional (recommended for security-critical repos) |
| **Restrict force pushes** | Prevent history rewriting on main | ✅ Enabled |
| **Restrict deletions** | Prevent accidental branch deletion | ✅ Enabled |
| **Require linear history** | Enforce rebase or squash merges | Optional (depends on team preference) |

### Protection by Branch Type

```
Branch Protection Levels:

  main / production         (highest protection)
  ┌──────────────────────────────────────────────┐
  │  ✅ Require PR with 2 reviewers              │
  │  ✅ Require CI status checks to pass         │
  │  ✅ Restrict force push and deletion          │
  │  ✅ Dismiss stale reviews on new commits      │
  │  ✅ Require branch to be up-to-date           │
  └──────────────────────────────────────────────┘

  develop / staging          (medium protection)
  ┌──────────────────────────────────────────────┐
  │  ✅ Require PR with 1 reviewer               │
  │  ✅ Require CI status checks to pass         │
  │  ✅ Restrict force push                       │
  └──────────────────────────────────────────────┘

  release/*                  (medium protection)
  ┌──────────────────────────────────────────────┐
  │  ✅ Require PR with 1–2 reviewers            │
  │  ✅ Require CI status checks to pass         │
  │  ✅ Restrict force push and deletion          │
  └──────────────────────────────────────────────┘

  feature/* / bugfix/*       (minimal protection)
  ┌──────────────────────────────────────────────┐
  │  ✅ Require CI status checks to pass         │
  │  ⚠️  Force push allowed (for rebase workflow) │
  └──────────────────────────────────────────────┘
```

### GitHub Branch Protection Example

```json
{
  "required_pull_request_reviews": {
    "required_approving_review_count": 2,
    "dismiss_stale_reviews": true,
    "require_code_owner_reviews": true
  },
  "required_status_checks": {
    "strict": true,
    "contexts": ["ci/build", "ci/test", "ci/lint"]
  },
  "enforce_admins": true,
  "restrictions": null,
  "allow_force_pushes": false,
  "allow_deletions": false
}
```

## Best Practices

### Strategy Selection

- ✅ Choose the simplest strategy that meets your deployment and release needs
- ✅ Document your branching strategy in the repository (e.g., `CONTRIBUTING.md`)
- ✅ Align your CI/CD pipeline with your branching model
- ✅ Revisit your strategy as team size, release cadence, or product needs evolve
- ❌ Do not adopt Git Flow just because it is well-known — evaluate fit first
- ❌ Do not mix strategies without a clear reason (e.g., Git Flow for one repo, TBD for another in the same team)

### Branch Hygiene

- ✅ Delete feature branches after they are merged
- ✅ Keep the number of active branches small (< 10 per developer)
- ✅ Rebase or update feature branches regularly to reduce merge conflicts
- ✅ Use automated branch cleanup (GitHub auto-delete on merge)
- ❌ Do not leave stale branches open for weeks — they accumulate merge debt
- ❌ Do not reuse branch names after deletion — create a new branch

### Merge Practices

- ✅ Prefer squash merges for feature branches to keep main history clean
- ✅ Use merge commits (no-ff) for release and hotfix branches to preserve context
- ✅ Require CI checks to pass before merging any branch
- ✅ Require at least one code review approval before merging
- ❌ Do not merge broken branches — fix CI failures before merging
- ❌ Do not force push to shared branches (main, develop, release)

## Next Steps

Continue to [Commits and History](02-COMMITS-AND-HISTORY.md) to learn about writing clear commit messages, following the Conventional Commits specification, navigating project history, and using interactive rebase to clean up commits before merging.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial branching strategies documentation |
