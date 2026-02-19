# Monorepos and Submodules

## Table of Contents

1. [Overview](#overview)
2. [Monorepo vs Polyrepo](#monorepo-vs-polyrepo)
   - [Architecture Comparison](#architecture-comparison)
   - [Trade-Offs](#trade-offs)
   - [When to Use Each](#when-to-use-each)
3. [Git Submodules](#git-submodules)
   - [Adding a Submodule](#adding-a-submodule)
   - [Cloning with Submodules](#cloning-with-submodules)
   - [Updating Submodules](#updating-submodules)
   - [Pinning Versions](#pinning-versions)
   - [Removing a Submodule](#removing-a-submodule)
4. [Git Subtree](#git-subtree)
   - [Adding a Subtree](#adding-a-subtree)
   - [Pulling and Pushing Changes](#pulling-and-pushing-changes)
   - [Submodules vs Subtrees](#submodules-vs-subtrees)
5. [Monorepo Tooling](#monorepo-tooling)
   - [Nx](#nx)
   - [Turborepo](#turborepo)
   - [Bazel](#bazel)
   - [Lerna](#lerna)
   - [Rush](#rush)
   - [Tool Comparison](#tool-comparison)
6. [Sparse Checkout](#sparse-checkout)
   - [Cone Mode](#cone-mode)
   - [Partial Clone](#partial-clone)
   - [Combining Sparse Checkout and Partial Clone](#combining-sparse-checkout-and-partial-clone)
7. [Git LFS (Large File Storage)](#git-lfs-large-file-storage)
   - [When to Use Git LFS](#when-to-use-git-lfs)
   - [Setup and Configuration](#setup-and-configuration)
   - [Tracking Files](#tracking-files)
   - [Migrating Existing Files](#migrating-existing-files)
8. [Scaling Git for Large Repos](#scaling-git-for-large-repos)
   - [Shallow Clone](#shallow-clone)
   - [Partial Clone](#partial-clone-1)
   - [Commit Graph](#commit-graph)
   - [Filesystem Monitor](#filesystem-monitor)
9. [Monorepo CI/CD Strategies](#monorepo-cicd-strategies)
   - [Affected/Changed Detection](#affectedchanged-detection)
   - [Path-Based Triggers](#path-based-triggers)
   - [Build Caching](#build-caching)
10. [Best Practices](#best-practices)
11. [Next Steps](#next-steps)
12. [Version History](#version-history)

## Overview

Monorepos consolidate multiple projects, libraries, and services into a single Git repository. This approach simplifies dependency management and cross-project refactoring — but it requires specialized tooling to maintain performance and developer experience at scale. Submodules and subtrees offer alternative strategies for composing repositories without merging all code into one.

### Target Audience

- Engineers evaluating monorepo vs polyrepo strategies for their organization
- Developers working with Git submodules or subtrees in multi-project setups
- Platform and DevOps engineers scaling Git and CI/CD for large repositories

### Scope

- Monorepo and polyrepo architecture trade-offs
- Git submodules and subtrees for repository composition
- Monorepo build tools (Nx, Turborepo, Bazel, Lerna, Rush)
- Sparse checkout and partial clone for large repo performance
- Git LFS for managing large binary assets
- CI/CD strategies for monorepo change detection and build caching

## Monorepo vs Polyrepo

### Architecture Comparison

```
Monorepo                                 Polyrepo

┌──────────────────────────────────┐    ┌──────────────┐  ┌──────────────┐
│  single-repo/                    │    │  frontend/   │  │  backend/    │
│  ├── apps/                       │    │  ├── src/    │  │  ├── src/    │
│  │   ├── frontend/               │    │  ├── test/   │  │  ├── test/   │
│  │   ├── backend/                │    │  └── ci.yml  │  │  └── ci.yml  │
│  │   └── mobile/                 │    └──────────────┘  └──────────────┘
│  ├── libs/                       │
│  │   ├── shared-ui/              │    ┌──────────────┐  ┌──────────────┐
│  │   ├── auth/                   │    │  mobile/     │  │  shared-ui/  │
│  │   └── utils/                  │    │  ├── src/    │  │  ├── src/    │
│  ├── package.json                │    │  ├── test/   │  │  ├── test/   │
│  └── nx.json / turbo.json        │    │  └── ci.yml  │  │  └── ci.yml  │
└──────────────────────────────────┘    └──────────────┘  └──────────────┘

One repo, one history, shared tooling   Separate repos, independent histories
```

### Trade-Offs

| Aspect | Monorepo | Polyrepo |
|---|---|---|
| **Code sharing** | Direct imports across projects — no versioning friction | Requires publishing packages to a registry |
| **Atomic changes** | Single commit can update API + clients + docs | Coordinated PRs across multiple repos |
| **Dependency management** | Single lockfile, unified versions | Each repo manages its own dependencies |
| **CI/CD complexity** | Needs affected-project detection and build caching | Simple per-repo pipelines |
| **Repository size** | Grows large over time — needs sparse checkout, LFS | Each repo stays small and focused |
| **Access control** | Coarser — everyone can see all code (CODEOWNERS helps) | Fine-grained per-repo permissions |
| **Onboarding** | One clone, one setup — but more to navigate | Clone only what you need — but more repos to discover |
| **Tooling required** | Nx, Turborepo, Bazel, or similar | Standard Git workflows |
| **Refactoring** | Cross-project renames in a single PR | Requires synchronized changes across repos |
| **Release cadence** | Can release everything together or independently | Each repo has its own release cycle |

### When to Use Each

```
Choose MONOREPO when:                    Choose POLYREPO when:

✅ Teams share code frequently            ✅ Teams are autonomous with clear boundaries
✅ APIs and clients evolve together        ✅ Projects have very different tech stacks
✅ You want atomic cross-project changes   ✅ Strict access control per project is required
✅ Unified tooling and CI is a priority    ✅ Repositories would be extremely large (>50 GB)
✅ You have tooling expertise (Nx, Bazel)  ✅ Open-source projects needing separate histories
```

```
Hybrid Approach — Grouped Polyrepo:

┌─────────────────────────────┐     ┌─────────────────────────────┐
│  platform-monorepo/         │     │  data-monorepo/             │
│  ├── apps/web/              │     │  ├── pipelines/etl/         │
│  ├── apps/api/              │     │  ├── pipelines/ml/          │
│  ├── libs/shared-ui/        │     │  ├── libs/connectors/       │
│  └── libs/auth/             │     │  └── libs/schemas/          │
└─────────────────────────────┘     └─────────────────────────────┘

Teams that collaborate closely share a monorepo.
Separate domains get separate repositories.
```

## Git Submodules

Git submodules embed one Git repository inside another at a specific commit. The parent repo stores a pointer (SHA) to the exact commit of the child repo — not the child's files.

```
Parent Repository
├── .gitmodules              ← Tracks submodule URLs and paths
├── src/
├── libs/
│   └── shared-utils/        ← Submodule (pointer to commit abc123)
│       ├── src/
│       └── package.json
└── README.md

The parent stores:
  - .gitmodules        → URL and path mapping
  - libs/shared-utils  → Git tree entry pointing to commit abc123
```

### Adding a Submodule

```bash
# Add a submodule at a specific path
git submodule add https://github.com/org/shared-utils.git libs/shared-utils

# Add a submodule pinned to a specific branch
git submodule add -b main https://github.com/org/shared-utils.git libs/shared-utils

# This creates/updates two things:
#   1. .gitmodules file (tracked — commit this)
#   2. A tree entry in the index pointing to the submodule's current HEAD
```

```ini
# .gitmodules — auto-generated by git submodule add
[submodule "libs/shared-utils"]
    path = libs/shared-utils
    url = https://github.com/org/shared-utils.git
    branch = main
```

```bash
# Commit the submodule addition
git add .gitmodules libs/shared-utils
git commit -m "chore: add shared-utils submodule"
```

### Cloning with Submodules

```bash
# Clone and initialize all submodules in one command
git clone --recurse-submodules https://github.com/org/parent-repo.git

# If you already cloned without --recurse-submodules
git submodule init
git submodule update

# Or combine init + update in one step
git submodule update --init

# For nested submodules (submodules within submodules)
git submodule update --init --recursive
```

```
Clone Flow:

  git clone --recurse-submodules <url>
       │
       ├──▶ Clone parent repo
       │
       ├──▶ Read .gitmodules
       │
       ├──▶ Clone each submodule repo
       │       │
       │       └──▶ Checkout pinned commit (detached HEAD)
       │
       └──▶ Working tree is ready
```

### Updating Submodules

```bash
# Update all submodules to the latest commit on their tracked branch
git submodule update --remote

# Update a specific submodule
git submodule update --remote libs/shared-utils

# Update and merge (instead of detached HEAD checkout)
git submodule update --remote --merge

# Update and rebase local changes onto the new upstream commit
git submodule update --remote --rebase

# After updating, commit the new submodule pointer in the parent
git add libs/shared-utils
git commit -m "chore: update shared-utils to latest"
```

```bash
# Check submodule status — shows current commit and diff from index
git submodule status

# Show a summary of changes in submodules
git submodule summary

# Run a command in every submodule
git submodule foreach 'git fetch origin && git rebase origin/main'
```

### Pinning Versions

```bash
# Enter the submodule directory and checkout a specific tag or commit
cd libs/shared-utils
git fetch --tags
git checkout v2.1.0

# Return to parent and commit the pinned version
cd ../..
git add libs/shared-utils
git commit -m "chore: pin shared-utils to v2.1.0"
```

```
Pinning Flow:

  Parent repo index:
    libs/shared-utils → commit a1b2c3d (v2.1.0)

  When someone clones or runs git submodule update:
    ─▶ Checks out exactly commit a1b2c3d in libs/shared-utils
    ─▶ Detached HEAD at that commit
    ─▶ Reproducible build guaranteed
```

### Removing a Submodule

Removing a submodule requires several steps — there is no single `git submodule remove` command in older Git versions.

```bash
# Step 1: Deinit the submodule (removes from .git/config)
git submodule deinit -f libs/shared-utils

# Step 2: Remove from the index and working tree
git rm -f libs/shared-utils

# Step 3: Remove the submodule's .git metadata
rm -rf .git/modules/libs/shared-utils

# Step 4: Commit the removal
git commit -m "chore: remove shared-utils submodule"
```

```bash
# Git 2.35+ — simplified removal
git rm libs/shared-utils
rm -rf .git/modules/libs/shared-utils
git commit -m "chore: remove shared-utils submodule"
```

## Git Subtree

Git subtree merges the contents of another repository directly into a subdirectory of your repository. Unlike submodules, the files are part of your repo's history — no special commands needed to clone or checkout.

```
Submodule vs Subtree:

  Submodule                              Subtree
  ┌────────────────────────────┐        ┌────────────────────────────┐
  │  parent-repo/              │        │  parent-repo/              │
  │  ├── src/                  │        │  ├── src/                  │
  │  ├── libs/utils/ ──pointer─┼──▶ ☁   │  ├── libs/utils/           │
  │  │   (not actual files)    │        │  │   ├── src/              │
  │  └── .gitmodules           │        │  │   └── package.json     │
  └────────────────────────────┘        │  │   (actual files here)   │
                                        │  └── ...                   │
  Files live in separate repo           └────────────────────────────┘
  Pointer stored in parent
                                        Files merged into parent history
                                        No pointer — full copy
```

### Adding a Subtree

```bash
# Add a remote for the external repo (optional but recommended)
git remote add shared-utils https://github.com/org/shared-utils.git

# Add the subtree — squash merges the entire history into one commit
git subtree add --prefix=libs/shared-utils shared-utils main --squash

# Without squash — preserves full history of the external repo
git subtree add --prefix=libs/shared-utils shared-utils main
```

### Pulling and Pushing Changes

```bash
# Pull updates from the external repo into the subtree
git subtree pull --prefix=libs/shared-utils shared-utils main --squash

# Push changes made in the subtree back to the external repo
git subtree push --prefix=libs/shared-utils shared-utils main
```

```
Subtree Workflow:

  External Repo (shared-utils)
       │
       │  git subtree add / pull
       ▼
  ┌────────────────────────────────┐
  │  Parent Repo                    │
  │  libs/shared-utils/             │
  │    ├── (files merged in)        │
  │    └── (part of parent history) │
  └────────────────────────────────┘
       │
       │  git subtree push
       ▼
  External Repo (shared-utils)
  (changes pushed back upstream)
```

### Submodules vs Subtrees

| Aspect | Submodules | Subtrees |
|---|---|---|
| **How it works** | Pointer to external repo commit | Files merged into parent history |
| **Clone behavior** | Requires `--recurse-submodules` or `submodule update` | Normal `git clone` — no extra steps |
| **History** | Separate — child has its own commit log | Merged — child commits are in parent history |
| **Updating** | `git submodule update --remote` | `git subtree pull` |
| **Contributing back** | Work in submodule dir, push from there | `git subtree push` to extract and push changes |
| **Pinning versions** | Checkout specific SHA/tag in submodule | Pull specific branch/tag with subtree pull |
| **Repo size** | Small — only pointers stored | Larger — full file contents duplicated |
| **Complexity** | More moving parts (`.gitmodules`, detached HEAD) | Simpler for consumers, harder to split back out |
| **Best for** | Libraries with independent release cycles | Vendoring code or infrequently updated deps |

## Monorepo Tooling

### Nx

[Nx](https://nx.dev/) is a build system with first-class monorepo support. It provides dependency graph analysis, computation caching, and distributed task execution.

```bash
# Create a new Nx workspace
npx create-nx-workspace@latest my-monorepo

# Add a project to the workspace
nx generate @nx/react:application my-app
nx generate @nx/node:library shared-utils

# Run tasks with affected detection
nx affected --target=build
nx affected --target=test
nx affected --target=lint

# Visualize the dependency graph
nx graph
```

```
Nx Dependency Graph:

  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │  my-app  │────▶│ shared-  │────▶│  utils   │
  │ (React)  │     │ ui       │     │  (lib)   │
  └──────────┘     └──────────┘     └──────────┘
       │                                  ▲
       │           ┌──────────┐           │
       └──────────▶│  auth    │───────────┘
                   │  (lib)   │
                   └──────────┘

  nx affected --target=build
  Only rebuilds projects affected by the current changes.
```

### Turborepo

[Turborepo](https://turbo.build/) is a high-performance build system for JavaScript/TypeScript monorepos. It focuses on caching and parallel task execution.

```json
// turbo.json — Pipeline configuration
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "inputs": ["src/**", "test/**"]
    },
    "lint": {},
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

```bash
# Run build for all packages (with caching)
turbo run build

# Run tests only for affected packages
turbo run test --filter=...[HEAD~1]

# Run dev servers in parallel
turbo run dev --parallel
```

### Bazel

[Bazel](https://bazel.build/) is a language-agnostic build system from Google, designed for massive monorepos. It uses hermetic builds and aggressive caching.

```python
# BUILD file — Bazel build target
load("@rules_go//go:def.bzl", "go_library", "go_test")

go_library(
    name = "auth",
    srcs = ["auth.go"],
    importpath = "github.com/org/monorepo/libs/auth",
    deps = [
        "//libs/utils:utils",
        "@com_github_golang_jwt_jwt//:jwt",
    ],
    visibility = ["//visibility:public"],
)

go_test(
    name = "auth_test",
    srcs = ["auth_test.go"],
    embed = [":auth"],
)
```

```bash
# Build a specific target
bazel build //apps/api:api

# Test everything affected by changes
bazel test //... --test_tag_filters=-integration

# Query the dependency graph
bazel query 'deps(//apps/api:api)' --output graph
```

### Lerna

[Lerna](https://lerna.js.org/) manages JavaScript monorepos with a focus on versioning and publishing npm packages. Lerna is now powered by Nx under the hood.

```json
// lerna.json
{
  "$schema": "node_modules/lerna/schemas/lerna-schema.json",
  "version": "independent",
  "npmClient": "npm",
  "command": {
    "publish": {
      "conventionalCommits": true,
      "message": "chore(release): publish"
    }
  }
}
```

```bash
# Bootstrap — install deps and link local packages
npx lerna bootstrap

# Run a script in all packages
npx lerna run build

# Run only in changed packages since main
npx lerna run test --since=main

# Publish changed packages
npx lerna publish
```

### Rush

[Rush](https://rushjs.io/) is a scalable monorepo manager from Microsoft. It enforces consistent dependency versions and supports phantom dependency prevention.

```bash
# Initialize a Rush monorepo
rush init

# Install dependencies (uses a single shrinkwrap file)
rush update

# Build only affected projects
rush build --changed-projects-only

# Add a dependency to a project
cd apps/frontend
rush add --package react --dev
```

### Tool Comparison

| Feature | Nx | Turborepo | Bazel | Lerna | Rush |
|---|---|---|---|---|---|
| **Language support** | JS/TS, Go, Rust, Java, .NET | JS/TS | Any language | JS/TS | JS/TS |
| **Affected detection** | ✅ Built-in | ✅ `--filter` | ✅ Query system | ✅ `--since` | ✅ `--changed-projects-only` |
| **Local caching** | ✅ | ✅ | ✅ | ✅ (via Nx) | ✅ (via Cobuild) |
| **Remote caching** | ✅ Nx Cloud | ✅ Vercel Remote Cache | ✅ Remote execution | ✅ (via Nx) | ✅ (via Cobuild) |
| **Distributed execution** | ✅ Nx Agents | ❌ | ✅ Remote execution | ❌ | ❌ |
| **Dependency graph** | ✅ Visual + CLI | ✅ CLI | ✅ Query language | ✅ Basic | ✅ Basic |
| **Code generation** | ✅ Generators + plugins | ❌ | ❌ | ❌ | ❌ |
| **Learning curve** | Medium | Low | High | Low | Medium |
| **Best for** | Full-featured monorepo platform | Fast JS/TS build caching | Massive multi-language repos | npm package publishing | Enterprise JS/TS monorepos |

## Sparse Checkout

Sparse checkout lets you check out only a subset of files from a repository. This is essential for large monorepos where developers only need one or two projects.

### Cone Mode

Cone mode (recommended) restricts checkout to specific directories and is optimized for performance.

```bash
# Enable sparse checkout in cone mode
git clone --no-checkout https://github.com/org/monorepo.git
cd monorepo
git sparse-checkout init --cone

# Check out only specific directories
git sparse-checkout set apps/frontend libs/shared-ui

# Add more directories without replacing existing ones
git sparse-checkout add libs/auth

# View current sparse checkout patterns
git sparse-checkout list

# Disable sparse checkout (restore full working tree)
git sparse-checkout disable
```

```
Full Repo                              Sparse Checkout (cone mode)

monorepo/                              monorepo/
├── apps/                              ├── apps/
│   ├── frontend/    ◀── checked out   │   └── frontend/    ◀── only this
│   ├── backend/     ◀── checked out   ├── libs/
│   └── mobile/      ◀── checked out   │   ├── shared-ui/   ◀── and this
├── libs/                              │   └── auth/         ◀── and this
│   ├── shared-ui/   ◀── checked out   └── (root files)
│   ├── auth/        ◀── checked out
│   └── analytics/   ◀── checked out   Everything else is excluded from
├── tools/           ◀── checked out   the working tree but still in the
└── docs/            ◀── checked out   Git index (fast switch possible).
```

### Partial Clone

Partial clone downloads only the Git objects you actually need — skipping blobs (file contents) for files outside your working set.

```bash
# Blobless clone — download commits and trees, fetch blobs on demand
git clone --filter=blob:none https://github.com/org/monorepo.git

# Treeless clone — download only commits, fetch trees and blobs on demand
git clone --filter=tree:0 https://github.com/org/monorepo.git

# Combine with sparse checkout for maximum efficiency
git clone --filter=blob:none --sparse https://github.com/org/monorepo.git
cd monorepo
git sparse-checkout set apps/frontend libs/shared-ui
```

```
Clone Types — What Gets Downloaded:

  Full Clone            Blobless Clone        Treeless Clone
  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │ ✅ Commits    │     │ ✅ Commits    │     │ ✅ Commits    │
  │ ✅ Trees      │     │ ✅ Trees      │     │ ❌ Trees      │
  │ ✅ Blobs      │     │ ❌ Blobs      │     │ ❌ Blobs      │
  └──────────────┘     └──────────────┘     └──────────────┘
  Everything           Blobs fetched        Trees + blobs
  up front             on checkout          fetched on demand

  Best for:            Best for:            Best for:
  Small repos          CI/CD, dev work      One-off scripts,
                       in monorepos         shallow automation
```

### Combining Sparse Checkout and Partial Clone

```bash
# Optimal setup for monorepo developers
git clone \
  --filter=blob:none \
  --sparse \
  https://github.com/org/monorepo.git

cd monorepo

# Check out only your team's projects
git sparse-checkout set \
  apps/frontend \
  libs/shared-ui \
  libs/auth

# Git fetches blobs only for files in these directories
# Result: fast clone + small working tree
```

## Git LFS (Large File Storage)

Git LFS replaces large files in your repository with lightweight pointer files. The actual file content is stored on a separate LFS server, keeping your Git history small and clones fast.

```
Without LFS                              With LFS

.git/objects/                            .git/objects/
├── ab/c123... (50 MB PSD file)          ├── ab/c123... (130-byte pointer)
├── de/f456... (50 MB PSD file v2)       ├── de/f456... (130-byte pointer)
└── ...                                  └── ...

Total .git size: hundreds of MB          Total .git size: small
                                         LFS server stores actual files
```

### When to Use Git LFS

- ✅ Binary assets (images, videos, audio, fonts, PSD/Sketch files)
- ✅ Compiled artifacts (`.dll`, `.so`, `.wasm`)
- ✅ Large datasets or ML models checked into the repo
- ✅ Any file type that does not benefit from Git's delta compression
- ❌ Source code files (Git handles these efficiently)
- ❌ Files that change on every commit (LFS stores each version separately)

### Setup and Configuration

```bash
# Install Git LFS (one-time global setup)
git lfs install

# Verify installation
git lfs version

# Check LFS status in a repository
git lfs status
git lfs ls-files          # List all LFS-tracked files
git lfs env               # Show LFS configuration
```

### Tracking Files

```bash
# Track file patterns with LFS
git lfs track "*.psd"
git lfs track "*.png"
git lfs track "*.mp4"
git lfs track "*.zip"
git lfs track "assets/models/**"

# View currently tracked patterns
git lfs track

# Always commit .gitattributes after tracking changes
git add .gitattributes
git commit -m "chore: track large files with Git LFS"
```

```gitattributes
# .gitattributes — auto-generated by git lfs track
*.psd filter=lfs diff=lfs merge=lfs -text
*.png filter=lfs diff=lfs merge=lfs -text
*.mp4 filter=lfs diff=lfs merge=lfs -text
*.zip filter=lfs diff=lfs merge=lfs -text
assets/models/** filter=lfs diff=lfs merge=lfs -text
```

```
LFS Pointer File (what Git stores):

  version https://git-lfs.github.com/spec/v1
  oid sha256:4d7a214614ab2935c943f9e0ff69d22eadbb8f32...
  size 52428800

  ─▶ 130 bytes instead of 50 MB
  ─▶ Actual content fetched from LFS server on checkout
```

### Migrating Existing Files

```bash
# Migrate existing files in history to LFS (rewrites history)
git lfs migrate import --include="*.psd,*.png,*.mp4" --everything

# Migrate only in the current branch
git lfs migrate import --include="*.psd" --include-ref=refs/heads/main

# Preview what would be migrated (dry run)
git lfs migrate info --include="*.psd,*.png,*.mp4" --everything

# After migration, force-push the rewritten history
git push --force-with-lease --all
git push --force-with-lease --tags

# Have all team members re-clone after a history rewrite
```

## Scaling Git for Large Repos

### Shallow Clone

Shallow clones download only recent history, significantly reducing clone time and disk usage.

```bash
# Clone with only the last N commits
git clone --depth=1 https://github.com/org/monorepo.git

# Deepen a shallow clone later
git fetch --deepen=50

# Convert a shallow clone to a full clone
git fetch --unshallow
```

```
Full Clone                  Shallow Clone (depth=1)

  ○ HEAD                     ○ HEAD
  │                          (no parent history)
  ○ commit N-1
  │
  ○ commit N-2              Useful for:
  │                         - CI/CD build agents
  ○ commit N-3              - Quick local builds
  │                         - Scripted automation
  ○ ...
  │                         Limitations:
  ○ initial commit          - No git log / git blame history
                            - Cannot push in some setups
```

### Partial Clone

Partial clones (introduced in Git 2.22+) allow the server to omit certain objects during clone, fetching them lazily on demand.

```bash
# Blobless clone — best for development
# Downloads all commits and trees but fetches file blobs on demand
git clone --filter=blob:none https://github.com/org/monorepo.git

# Treeless clone — best for CI or scripted access
# Downloads only commits, fetches trees and blobs on demand
git clone --filter=tree:0 https://github.com/org/monorepo.git

# Size-limited clone — exclude blobs larger than a threshold
git clone --filter=blob:limit=1m https://github.com/org/monorepo.git
```

### Commit Graph

The commit-graph file accelerates commit history operations (`git log`, `git merge-base`, reachability queries) by precomputing commit metadata.

```bash
# Write the commit-graph file
git commit-graph write --reachable

# Enable automatic commit-graph updates
git config --global fetch.writeCommitGraph true
git config --global gc.writeCommitGraph true

# Verify the commit-graph
git commit-graph verify
```

```
Without Commit Graph                With Commit Graph

git log --oneline                   git log --oneline
  Traverses every commit object       Reads precomputed data from
  from pack files                      .git/objects/info/commit-graph

  O(n) object lookups                 O(1) generation number lookups
  Slow for repos with 100K+ commits   Fast regardless of history size
```

### Filesystem Monitor

The filesystem monitor (fsmonitor) integrates Git with the OS file watcher to skip scanning unchanged files during `git status` and `git add`.

```bash
# Enable the built-in filesystem monitor (Git 2.37+)
git config core.fsmonitor true
git config core.untrackedcache true

# Verify fsmonitor is running
git status   # First run initializes the daemon
```

```
Without fsmonitor                    With fsmonitor

git status                           git status
  Scans EVERY file in working tree     Queries OS file watcher for changes
  Stats each file for modification     Only checks files that actually changed
  Slow for repos with 100K+ files      Fast — O(changed files) not O(all files)
```

## Monorepo CI/CD Strategies

### Affected/Changed Detection

Only build and test projects affected by the current changeset — avoid rebuilding the entire monorepo on every commit.

```
Change Detection Flow:

  ┌─────────────────────┐
  │  New Commit          │
  │  Changed: libs/auth/ │
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────────────────┐
  │  Dependency Graph Analysis       │
  │                                  │
  │  libs/auth ◀── apps/frontend    │
  │  libs/auth ◀── apps/backend     │
  │  libs/auth     apps/mobile      │
  │  (changed)     (not affected)   │
  └──────────┬──────────────────────┘
             │
             ▼
  ┌─────────────────────────────────┐
  │  Build + Test:                   │
  │  ✅ libs/auth                    │
  │  ✅ apps/frontend                │
  │  ✅ apps/backend                 │
  │  ⏭ apps/mobile (skipped)        │
  └─────────────────────────────────┘
```

```bash
# Nx — affected commands
nx affected --target=build --base=origin/main --head=HEAD
nx affected --target=test --base=origin/main --head=HEAD

# Turborepo — filter by changes
turbo run build --filter=...[origin/main]
turbo run test --filter=...[origin/main]

# Manual detection with git diff
CHANGED_DIRS=$(git diff --name-only origin/main...HEAD | cut -d'/' -f1-2 | sort -u)
```

### Path-Based Triggers

Configure CI/CD pipelines to run only when relevant paths change.

```yaml
# GitHub Actions — path-based triggers
name: Frontend CI

on:
  push:
    branches: [main]
    paths:
      - 'apps/frontend/**'
      - 'libs/shared-ui/**'
      - 'libs/auth/**'
  pull_request:
    paths:
      - 'apps/frontend/**'
      - 'libs/shared-ui/**'
      - 'libs/auth/**'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: nx build frontend
```

```yaml
# GitLab CI — rules with changes
build-frontend:
  stage: build
  script:
    - nx build frontend
  rules:
    - changes:
        - apps/frontend/**/*
        - libs/shared-ui/**/*
        - libs/auth/**/*
```

```yaml
# Azure Pipelines — path triggers
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - apps/frontend
      - libs/shared-ui
    exclude:
      - docs
```

### Build Caching

Cache build outputs to avoid redundant work across CI runs and developer machines.

```
Build Cache Flow:

  ┌──────────────────────────────────────────────┐
  │  Task: build libs/auth                        │
  │  Inputs hash: abc123 (source + deps + config) │
  └──────────┬───────────────────────────────────┘
             │
             ▼
  ┌────────────────────┐     Cache HIT
  │  Check cache        │ ─────────────────▶ Return cached output
  │  (local or remote)  │                    (skip build entirely)
  └──────────┬──────────┘
             │ Cache MISS
             ▼
  ┌────────────────────┐
  │  Run build task     │
  │  Store output in    │
  │  cache with hash    │
  └────────────────────┘
```

```bash
# Nx — remote caching with Nx Cloud
# nx.json
# { "nxCloudAccessToken": "..." }
nx build auth   # Cache miss → builds and stores result
nx build auth   # Cache hit → instant (reads from cache)

# Turborepo — remote caching with Vercel
turbo run build --remote-only
```

```yaml
# GitHub Actions — cache node_modules and build outputs
- uses: actions/cache@v4
  with:
    path: |
      node_modules
      .nx/cache
    key: ${{ runner.os }}-build-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-build-
```

## Best Practices

- ✅ Start with a polyrepo and migrate to a monorepo only when cross-project coordination becomes a bottleneck
- ✅ Use a monorepo build tool (Nx, Turborepo, Bazel) — do not try to scale a monorepo with plain `npm` scripts
- ✅ Enable sparse checkout and partial clone for developers who only work on part of the monorepo
- ✅ Set up `CODEOWNERS` to enforce per-project review ownership in a monorepo
- ✅ Use path-based CI triggers to avoid rebuilding unaffected projects
- ✅ Enable build caching (local and remote) to speed up CI and developer builds
- ✅ Pin submodule versions to tags or known-good commits — not to branch tips
- ✅ Prefer subtrees over submodules when consumers should not need extra Git commands to clone
- ✅ Use Git LFS for binary assets and large files — never commit them directly
- ✅ Enable `core.fsmonitor` and `commit-graph` for repos with tens of thousands of files
- ✅ Document monorepo setup steps (sparse checkout, LFS, tooling) in the repository README
- ❌ Do not create a monorepo without investing in CI/CD affected-detection tooling
- ❌ Do not use submodules if most contributors are unfamiliar with the workflow — the learning curve causes frequent mistakes
- ❌ Do not force-push a repository after an LFS migration without coordinating with all contributors
- ❌ Do not track auto-generated or frequently changing large files in LFS — each version is stored separately
- ❌ Do not skip `--recurse-submodules` in CI/CD clone steps — builds will fail with missing dependencies

## Next Steps

Continue to [Security and Signing](08-SECURITY-AND-SIGNING.md) to learn about GPG signing, SSH commit signing, secrets management, credential storage, and securing your Git workflow.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial monorepos and submodules documentation |
