# Git Overview

## Table of Contents

1. [Overview](#overview)
2. [What Is Git](#what-is-git)
3. [Centralized vs. Distributed VCS](#centralized-vs-distributed-vcs)
4. [Core Concepts](#core-concepts)
5. [Git's Object Model](#gits-object-model)
6. [When to Use Git](#when-to-use-git)
7. [Key Components of a Git Workflow](#key-components-of-a-git-workflow)
8. [Prerequisites](#prerequisites)
9. [Next Steps](#next-steps)

## Overview

This documentation provides a comprehensive introduction to Git — a free, open-source distributed version control system designed to handle everything from small to very large projects with speed and efficiency. Created by Linus Torvalds in 2005 for Linux kernel development, Git has become the most widely adopted version control system in the world.

### Target Audience

- Developers learning version control for the first time
- Engineers migrating from centralized VCS (SVN, CVS, Perforce) to Git
- DevOps and platform engineers managing repositories and CI/CD pipelines
- Team leads establishing Git workflows and conventions

### Scope

- What Git is and how distributed version control works
- Centralized vs. distributed VCS trade-offs
- Core concepts — working directory, staging area, repository, HEAD, refs, objects
- Git's internal object model and content-addressable storage
- When Git is the right tool (and when alternatives might be better)
- Key components of a typical Git workflow

## What Is Git

Git is a distributed version control system (DVCS) that tracks changes to files over time, allowing multiple developers to collaborate on a project without overwriting each other's work. Unlike centralized systems that store only diffs (differences between file versions), Git records full snapshots of the project state at each commit.

### Key Capabilities

| Capability | Description |
|---|---|
| **Distributed architecture** | Every clone is a full repository with complete history |
| **Snapshot-based storage** | Records full project snapshots, not file-by-file diffs |
| **Branching and merging** | Lightweight branches enable parallel development |
| **Data integrity** | Every object is checksummed with SHA-1 hashes |
| **Speed** | Nearly all operations are local — no network latency |
| **Staging area** | Fine-grained control over which changes to include in a commit |

### Snapshots vs. Diffs

```
Delta-based VCS (SVN, CVS)           Snapshot-based VCS (Git)

File A:  v1 ──Δ1──▶ v2 ──Δ2──▶ v3   File A:  [snap1] [snap2] [snap3]
File B:  v1 ──Δ1──▶ v2               File B:  [snap1] [snap2] [snap2]*
File C:  v1 ──Δ1──▶ v2 ──Δ2──▶ v3   File C:  [snap1] [snap2] [snap3]

Stores changes (deltas) to each       Stores snapshots of all tracked files.
file over time. Reconstructing a       Unchanged files are stored as links
version means replaying all deltas.    to the previous identical snapshot (*).
```

```
How Git Thinks About Data:

Commit 1            Commit 2            Commit 3
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Snapshot     │    │ Snapshot     │    │ Snapshot     │
│              │    │              │    │              │
│ FileA  v1   │    │ FileA  v2   │    │ FileA  v2*  │
│ FileB  v1   │    │ FileB  v1*  │    │ FileB  v2   │
│ FileC  v1   │    │ FileC  v2   │    │ FileC  v2*  │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       └───────────────────┴───────────────────┘
                    Linked list of snapshots
                    (* = unchanged, stored as pointer)
```

## Centralized vs. Distributed VCS

### Architecture Comparison

```
Centralized VCS (SVN, CVS)             Distributed VCS (Git, Mercurial)

         ┌──────────┐                  ┌──────────┐    ┌──────────┐
         │ Central  │                  │ Remote   │    │ Remote   │
         │ Server   │                  │ Repo A   │    │ Repo B   │
         │          │                  │ (GitHub) │    │ (GitLab) │
         │ History  │                  └────┬─────┘    └────┬─────┘
         │ Branches │                       │               │
         │ Tags     │                  push/fetch       push/fetch
         └────┬─────┘                       │               │
              │                        ┌────▼─────┐    ┌────▼─────┐
     ┌────────┼────────┐               │ Dev A    │    │ Dev B    │
     │        │        │               │ Full     │    │ Full     │
┌────▼──┐ ┌──▼───┐ ┌──▼───┐           │ History  │    │ History  │
│Dev A  │ │Dev B │ │Dev C │           │ Branches │    │ Branches │
│       │ │      │ │      │           │ Tags     │    │ Tags     │
│Working│ │Work- │ │Work- │           └──────────┘    └──────────┘
│ Copy  │ │ ing  │ │ ing  │
│ Only  │ │ Copy │ │ Copy │           Each developer has a COMPLETE
└───────┘ └──────┘ └──────┘           copy of the entire repository.
                                      Work continues even when offline.
Developers only have working
copies. All history lives on
the central server.
```

### Trade-Off Analysis

| Dimension | Centralized (SVN/CVS) | Distributed (Git/Mercurial) |
|---|---|---|
| **Repository model** | Single source of truth on server | Every clone is a full repository |
| **Offline work** | Requires server connection for most operations | Nearly all operations work offline |
| **Speed** | Network-dependent for commits, logs, diffs | Local operations are near-instant |
| **Branching** | Heavyweight; branches are directory copies | Lightweight; branches are pointer references |
| **Single point of failure** | Server down = team blocked | Any clone can restore the repository |
| **Learning curve** | Simpler mental model | Steeper learning curve |
| **Large binary files** | Better native support (SVN) | Requires Git LFS for large binaries |
| **Access control** | Path-level permissions built in | Repository-level; path-level requires extra tools |
| **History rewriting** | Not supported | Supported (rebase, amend, filter-branch) |
| **Merging** | Basic merge support | Advanced merge strategies and algorithms |

## Core Concepts

### 1. The Three Areas

Git manages files across three main areas: the working directory, the staging area (index), and the repository (object database).

```
┌─────────────────────────────────────────────────────────────────┐
│                        Git's Three Areas                        │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │   Working    │    │   Staging    │    │   Repository     │  │
│  │   Directory  │    │   Area       │    │   (.git)         │  │
│  │              │    │   (Index)    │    │                  │  │
│  │  Your files  │    │  Next commit │    │  Commit history  │  │
│  │  as you see  │    │  snapshot    │    │  and all objects │  │
│  │  and edit    │    │  preview     │    │                  │  │
│  └──────┬───────┘    └──────┬───────┘    └──────────────────┘  │
│         │                   │                     ▲             │
│         │   git add         │   git commit        │             │
│         └──────────────────▶└─────────────────────┘             │
│                                                                 │
│         ◀──────────────────── git restore ──────────────────    │
│         ◀──────────────────────────── git reset ────────────    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2. HEAD, Branches, and Refs

**HEAD** is a pointer to the current commit (or branch) you are working on. Branches are simply movable pointers to commits. Tags are fixed pointers to specific commits.

```
Refs (References)
├── HEAD                 → Points to current branch (e.g., refs/heads/main)
├── refs/heads/          → Branch pointers
│   ├── main             → Points to latest commit on main
│   ├── feature/login    → Points to latest commit on feature/login
│   └── develop          → Points to latest commit on develop
├── refs/tags/           → Tag pointers
│   ├── v1.0.0           → Points to a specific commit (lightweight)
│   └── v2.0.0           → Points to a tag object (annotated)
└── refs/remotes/        → Remote-tracking branches
    └── origin/
        ├── main         → Last known state of origin/main
        └── develop      → Last known state of origin/develop
```

```
Branch Pointer Movement:

Before commit:                After commit:

  main                          main
    │                             │
    ▼                             ▼
┌───────┐    ┌───────┐      ┌───────┐    ┌───────┐    ┌───────┐
│  A    │───▶│  B    │      │  A    │───▶│  B    │───▶│  C    │
└───────┘    └───────┘      └───────┘    └───────┘    └───────┘
                  ▲                                        ▲
                 HEAD                                     HEAD
```

### 3. Git Objects

Git's data model is built on four types of objects, each identified by a SHA-1 hash.

| Object | Purpose | Contains |
|---|---|---|
| **Blob** | Stores file content | Raw file data (no filename or metadata) |
| **Tree** | Represents a directory | List of blobs and other trees with names and modes |
| **Commit** | Records a snapshot | Pointer to root tree, parent commit(s), author, message |
| **Tag** | Names a specific commit | Pointer to a commit, tag name, tagger, message (annotated) |

```
Commit Object Anatomy:

┌─────────────────────────────────────────┐
│              Commit  (abc123)           │
│                                         │
│  tree      → def456 (root tree)        │
│  parent    → 789abc (previous commit)  │
│  author    → Alice <alice@dev.io>      │
│  committer → Alice <alice@dev.io>      │
│  message   → "Add login feature"       │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│              Tree  (def456)             │
│                                         │
│  blob  → aaa111  README.md             │
│  blob  → bbb222  index.js             │
│  tree  → ccc333  src/                  │
└────────┬───────────────────┬────────────┘
         │                   │
         ▼                   ▼
┌──────────────┐    ┌──────────────────┐
│ Blob (aaa111)│    │ Tree (ccc333)    │
│              │    │                  │
│ # README     │    │ blob → ddd444   │
│ Welcome...   │    │   app.js        │
└──────────────┘    │ blob → eee555   │
                    │   util.js       │
                    └──────────────────┘
```

## Git's Object Model

Git is fundamentally a content-addressable filesystem — a key-value store where data is stored and retrieved based on the SHA-1 hash of its contents.

### How It Works

1. **Write content** — Git compresses the content, prepends a header (type + size), and computes the SHA-1 hash
2. **Store the object** — The object is stored in `.git/objects/` using the hash as the filename
3. **Reference by hash** — Any object can be retrieved by its 40-character hexadecimal hash

```
Content-Addressable Storage:

  "Hello, World!\n"
         │
         ▼
  ┌──────────────────────────────────────┐
  │  header = "blob 14\0"               │
  │  content = "Hello, World!\n"        │
  │  sha1(header + content)             │
  │        = 8ab686eafeb1f44702738c8b0f │
  │          24f2567c36da6d               │
  └──────────────────────────────────────┘
         │
         ▼
  .git/objects/8a/b686eafeb1f44702738c8b0f24f2567c36da6d

  Same content ALWAYS produces the same hash.
  Two identical files share a single blob object.
```

### Object Relationships

```
┌──────────────────────────────────────────────────────────────┐
│                    Git Object Graph                           │
│                                                              │
│  Tag ──▶ Commit ──▶ Tree ──▶ Blob                           │
│  (v1.0)   (abc1)    (root)   (file content)                 │
│                       │                                      │
│                       ├──▶ Blob  (README.md)                │
│                       ├──▶ Blob  (Makefile)                 │
│                       └──▶ Tree  (src/)                     │
│                              ├──▶ Blob  (main.c)            │
│                              └──▶ Blob  (util.c)            │
│                                                              │
│  Commits form a directed acyclic graph (DAG):               │
│                                                              │
│  ┌──┐    ┌──┐    ┌──┐    ┌──┐                               │
│  │C1│───▶│C2│───▶│C3│───▶│C4│  (linear history)             │
│  └──┘    └──┘    └──┘    └──┘                               │
│                                                              │
│  ┌──┐    ┌──┐    ┌──┐                                       │
│  │C1│───▶│C2│───▶│C4│  (merge commit — two parents)         │
│  └──┘    └──┘  ╱ └──┘                                       │
│          ┌──┐╱                                               │
│          │C3│                                                │
│          └──┘                                                │
└──────────────────────────────────────────────────────────────┘
```

### Integrity Guarantees

- **Tamper-proof** — Any change to content produces a different hash
- **Deduplication** — Identical content is stored only once
- **Verifiable** — Any object can be verified by recomputing its hash
- **Connected** — Parent references create an unbreakable chain of history

## When to Use Git

```
Git Is the Right Choice When:
├── You need to track changes to text-based files (source code, config, docs)
├── Multiple developers collaborate on the same codebase
├── You want lightweight, fast branching and merging
├── Offline work and local history are important
├── You need a rich ecosystem (GitHub, GitLab, CI/CD integrations)
└── Data integrity and auditability of changes matter
```

### Good Candidates

| Scenario | Why Git Works Well |
|---|---|
| **Software development** | Core use case — track source code with full history and branching |
| **Infrastructure as Code** | Version Terraform, Ansible, Kubernetes manifests alongside app code |
| **Documentation projects** | Markdown/AsciiDoc repos with pull-request-based review workflows |
| **Configuration management** | Track and audit changes to config files across environments |
| **Open-source collaboration** | Fork-and-pull model enables contributions from anyone |
| **Data science notebooks** | Track Jupyter notebooks and analysis scripts (with caveats for output cells) |

### When to Consider Alternatives

```
Git May Not Be Ideal When:
├── Managing large binary files (videos, datasets) → Consider Git LFS or DVC
├── Storing secrets or credentials → Use a secrets manager (Vault, AWS Secrets Manager)
├── Real-time collaborative editing → Use Google Docs, Notion, or CRDTs
├── Extremely large monorepos (millions of files) → Consider VFS for Git or Sapling
└── Strict path-level access control is needed → Consider SVN or Perforce
```

## Key Components of a Git Workflow

### Workflow Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    Complete Git Workflow                          │
│                                                                  │
│  ┌──────────────┐                      ┌──────────────────────┐  │
│  │   Working    │    git add           │   Staging Area       │  │
│  │   Tree       │ ──────────────────▶  │   (Index)            │  │
│  │              │                      │                      │  │
│  │  Edit files  │    git restore       │  Preview of next     │  │
│  │  Create new  │ ◀──────────────────  │  commit              │  │
│  │  Delete old  │                      │                      │  │
│  └──────────────┘                      └──────────┬───────────┘  │
│                                                   │              │
│                                          git commit              │
│                                                   │              │
│                                                   ▼              │
│                                        ┌──────────────────────┐  │
│                                        │   Local Repository   │  │
│                                        │   (.git directory)   │  │
│                                        │                      │  │
│                                        │  Commits, branches,  │  │
│                                        │  tags, full history  │  │
│                                        └──────────┬───────────┘  │
│                                                   │              │
│                                           git push│  ▲git fetch  │
│                                                   │  │git pull   │
│                                                   ▼  │           │
│                                        ┌──────────────────────┐  │
│                                        │  Remote Repository   │  │
│                                        │  (GitHub, GitLab,    │  │
│                                        │   Bitbucket)         │  │
│                                        │                      │  │
│                                        │  Shared source of    │  │
│                                        │  truth for the team  │  │
│                                        └──────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### Typical Developer Workflow

```
 1. Clone or Pull       2. Create Branch     3. Make Changes
 ┌──────────────┐      ┌──────────────┐     ┌──────────────┐
 │ git clone    │      │ git checkout │     │ Edit files   │
 │   <url>      │─────▶│   -b feature │────▶│ git add .    │
 │ git pull     │      │              │     │ git commit   │
 └──────────────┘      └──────────────┘     └──────┬───────┘
                                                   │
 6. Merge & Clean Up    5. Code Review       4. Push Branch
 ┌──────────────┐      ┌──────────────┐     ┌──────────────┐
 │ Merge PR     │      │ Open Pull    │     │ git push     │
 │ Delete branch│◀─────│ Request      │◀────│   origin     │
 │ git pull     │      │ Review &     │     │   feature    │
 └──────────────┘      │ approve      │     └──────────────┘
                       └──────────────┘
```

### Essential Git Commands by Workflow Stage

| Stage | Commands | Purpose |
|---|---|---|
| **Setup** | `git init`, `git clone` | Create or copy a repository |
| **Stage changes** | `git add`, `git restore --staged` | Select changes for the next commit |
| **Commit** | `git commit`, `git commit --amend` | Save a snapshot to the local repository |
| **Branch** | `git branch`, `git checkout`, `git switch` | Create and navigate branches |
| **Merge** | `git merge`, `git rebase` | Integrate changes from one branch into another |
| **Sync** | `git push`, `git fetch`, `git pull` | Exchange commits with a remote repository |
| **Inspect** | `git status`, `git log`, `git diff` | View the current state and history |
| **Undo** | `git reset`, `git revert`, `git restore` | Roll back or undo changes |

## Prerequisites

### Required Knowledge

- **Command-line basics** — Navigating directories, creating files, running commands in a terminal
- **Text editor proficiency** — Comfortable editing files in VS Code, Vim, or any code editor
- **Basic file system concepts** — Understanding of files, directories, and paths
- **Software development fundamentals** — Familiarity with writing and saving code in any language

### Recommended Tools

| Tool | Purpose |
|---|---|
| Git | Version control CLI (install from git-scm.com) |
| GitHub / GitLab | Remote repository hosting and collaboration |
| VS Code | Editor with excellent built-in Git integration |
| GitHub CLI (`gh`) | Command-line tool for GitHub pull requests and issues |
| Git GUI clients | GitKraken, Sourcetree, or Fork for visual Git workflows |
| Delta / diff-so-fancy | Enhanced diff viewers for the terminal |

## Next Steps

Continue to [Branching Strategies](01-BRANCHING-STRATEGIES.md) to learn about Git Flow, GitHub Flow, trunk-based development, and how to choose the right branching model for your team.

### Suggested Learning Path

1. **[Branching Strategies](01-BRANCHING-STRATEGIES.md)** — Git Flow, GitHub Flow, trunk-based development
2. **[Commits and History](02-COMMITS-AND-HISTORY.md)** — Commit best practices, conventional commits
3. **[Merging and Rebasing](03-MERGING-AND-REBASING.md)** — Merge strategies, rebasing, conflict resolution
4. **[Remote Collaboration](04-REMOTE-COLLABORATION.md)** — Remotes, pull requests, code review
5. **[Advanced Techniques](05-ADVANCED-TECHNIQUES.md)** — Interactive rebase, cherry-pick, bisect, reflog
6. **[Hooks and Automation](06-HOOKS-AND-AUTOMATION.md)** — Git hooks, CI/CD integration
7. **[Monorepos and Submodules](07-MONOREPOS-AND-SUBMODULES.md)** — Monorepo strategies, submodules, subtrees
8. **[Security and Signing](08-SECURITY-AND-SIGNING.md)** — GPG signing, secrets management
9. **[Best Practices](09-BEST-PRACTICES.md)** — Workflow best practices, team conventions
10. **[Anti-Patterns](10-ANTI-PATTERNS.md)** — Common mistakes and how to avoid them

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Git overview documentation |
