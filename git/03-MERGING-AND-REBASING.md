# Git Merging and Rebasing

## Table of Contents

1. [Overview](#overview)
2. [Git Merge](#git-merge)
   - [Fast-Forward Merge](#fast-forward-merge)
   - [Three-Way Merge](#three-way-merge)
   - [Merge Commits](#merge-commits)
   - [Recursive Strategy](#recursive-strategy)
3. [Git Rebase](#git-rebase)
   - [Linear History with Rebase](#linear-history-with-rebase)
   - [How Rebase Works Internally](#how-rebase-works-internally)
   - [Interactive Rebase](#interactive-rebase)
4. [Merge vs. Rebase](#merge-vs-rebase)
   - [Comparison Table](#comparison-table)
   - [When to Use Each](#when-to-use-each)
   - [The Golden Rule of Rebasing](#the-golden-rule-of-rebasing)
5. [Merge Strategies](#merge-strategies)
   - [Ours](#ours)
   - [Theirs](#theirs)
   - [Octopus](#octopus)
   - [Subtree](#subtree)
6. [Conflict Resolution](#conflict-resolution)
   - [Identifying Conflicts](#identifying-conflicts)
   - [Resolution Workflow](#resolution-workflow)
   - [Conflict Resolution Tools](#conflict-resolution-tools)
   - [Rerere](#rerere)
7. [Merge Commit Strategies in Pull Requests](#merge-commit-strategies-in-pull-requests)
   - [Merge Commit](#merge-commit)
   - [Squash Merge](#squash-merge)
   - [Rebase and Merge](#rebase-and-merge)
8. [Cherry-Pick](#cherry-pick)
   - [Selecting Specific Commits](#selecting-specific-commits)
   - [Cherry-Pick Use Cases](#cherry-pick-use-cases)
9. [Best Practices for Clean History](#best-practices-for-clean-history)
10. [Next Steps](#next-steps)
11. [Version History](#version-history)

## Overview

This document covers how to integrate changes between Git branches using merge and rebase. Choosing the right integration strategy is critical for maintaining a clean, navigable project history — whether you are working on a solo project or collaborating across a large team.

### Target Audience

- Developers integrating feature branches into main or develop
- Tech leads choosing merge strategies for their team's pull request workflow
- DevOps engineers configuring branch protection rules and merge settings
- Open-source maintainers managing contributions from external collaborators

### Scope

- Merge mechanics — fast-forward, three-way, recursive
- Rebase mechanics — linear replay, interactive rebase for history cleanup
- Merge vs. rebase trade-offs and the golden rule of rebasing
- Merge strategies — ours, theirs, octopus, subtree
- Conflict identification, resolution workflow, and tooling
- Pull request merge strategies — merge commit, squash, rebase and merge
- Cherry-pick for selecting individual commits
- Best practices for maintaining clean project history

## Git Merge

`git merge` combines the work from two branches by creating a join point in the commit history. The result depends on the relationship between the branches — Git automatically selects the simplest strategy that preserves all changes.

### Fast-Forward Merge

A fast-forward merge occurs when the target branch has not diverged from the source branch. Git simply moves the branch pointer forward — no new commit is created.

```
Before: main has no new commits since feature branched off

  main
    │
    ▼
    A ◄── B ◄── C ◄── D
                       ▲
                       │
                    feature

After: git checkout main && git merge feature

                       main
                         │
                         ▼
    A ◄── B ◄── C ◄── D
                       ▲
                       │
                    feature

    main is simply moved forward to D.
    No merge commit is created.
```

```bash
# Fast-forward merge (default when possible)
git checkout main
git merge feature

# Force a merge commit even when fast-forward is possible
git merge --no-ff feature

# Only merge if fast-forward is possible (fail otherwise)
git merge --ff-only feature
```

#### When Fast-Forward Happens

| Condition | Fast-Forward? |
|---|---|
| Target branch has no new commits since the source branched off | ✅ Yes |
| Target branch has received new commits | ❌ No — three-way merge required |
| `--no-ff` flag is used | ❌ No — merge commit is forced |

### Three-Way Merge

A three-way merge is required when both branches have diverged — each has commits the other does not. Git finds the **merge base** (the most recent common ancestor) and combines the changes from both branches.

```
Before: main and feature have diverged

              main
                │
                ▼
    A ◄── B ◄── E
          │
          └──── C ◄── D
                       ▲
                       │
                    feature

    Merge base: B (most recent common ancestor)

After: git checkout main && git merge feature

              ┌────────────── main
              │                 │
              │                 ▼
    A ◄── B ◄── E ◄─────────── F  (merge commit)
          │                     │
          └──── C ◄── D ◄──────┘
                       ▲
                       │
                    feature

    F is a merge commit with two parents: E and D.
    Git combined the changes from B→E and B→D.
```

The three inputs to the merge algorithm:

| Input | Description |
|---|---|
| **Merge base** | The most recent common ancestor commit (B in the diagram) |
| **Ours** | The current branch tip (E — the branch you are on) |
| **Theirs** | The branch being merged in (D — the branch named in the merge command) |

### Merge Commits

A merge commit is a commit with **two or more parents**. It records the fact that two lines of development were combined at that point.

```bash
# Create a merge commit (even if fast-forward is possible)
git merge --no-ff feature/auth

# The resulting merge commit has two parent pointers:
git cat-file -p HEAD
# tree    a1b2c3d...
# parent  e4f5a6b...   (previous main tip)
# parent  c7d8e9f...   (feature/auth tip)
# author  Alice <alice@example.com> ...
#
# Merge branch 'feature/auth' into main
```

#### Merge Commit Pros and Cons

| Aspect | Benefit | Drawback |
|---|---|---|
| **Traceability** | Shows exactly when and where branches were integrated | Adds "merge bubbles" to history |
| **Reversibility** | `git revert -m 1 <merge>` cleanly undoes an entire feature | Reverting merges is more complex than reverting regular commits |
| **History** | Preserves the full branching topology | Can make `git log` output noisy |

### Recursive Strategy

The **recursive** strategy is Git's default for three-way merges. When there are multiple possible merge bases (due to criss-cross merges), the recursive strategy merges the merge bases first, creating a virtual ancestor commit, and then uses that as the base for the final merge.

```
Criss-cross merge scenario (multiple common ancestors):

    A ◄── B ◄── D ◄── F       (main)
          │     ▲     │
          │     │     │
          └──── C ◄── E       (feature)
                ▲     │
                └─────┘

    B and C are both common ancestors.
    Recursive strategy: merge B and C first to create a virtual base,
    then perform the three-way merge using that virtual base.
```

```bash
# Recursive is the default — these are equivalent:
git merge feature
git merge -s recursive feature

# Pass options to the recursive strategy:
git merge -s recursive -X patience feature     # Use patience diff algorithm
git merge -s recursive -X ignore-space-change feature
```

## Git Rebase

`git rebase` moves (replays) a sequence of commits onto a new base commit. Instead of creating a merge commit, rebase rewrites history so that your branch appears to have been started from a more recent point.

### Linear History with Rebase

Rebase produces a **linear history** — no merge commits, no branching topology. Every commit sits in a single straight line.

```
Before: feature has diverged from main

              main
                │
                ▼
    A ◄── B ◄── E
          │
          └──── C ◄── D
                       ▲
                       │
                    feature

After: git checkout feature && git rebase main

                       main
                         │
                         ▼
    A ◄── B ◄── E ◄── C' ◄── D'
                              ▲
                              │
                           feature

    C' and D' are NEW commits (different SHA-1 hashes).
    They contain the same changes as C and D but have E as their base.
    The original C and D are no longer reachable from any branch.
```

```bash
# Rebase feature branch onto main
git checkout feature
git rebase main

# After rebase, fast-forward main to include the feature
git checkout main
git merge feature    # This is now a fast-forward merge
```

### How Rebase Works Internally

Rebase performs the following steps under the hood:

```
Step-by-step: git checkout feature && git rebase main

1. Find the merge base between feature and main
   ─── merge base = B

2. Compute the diffs (patches) for each commit on feature since B
   ─── diff(B, C) = patch₁
   ─── diff(C, D) = patch₂

3. Reset feature to the tip of main (E)
   ─── feature now points to E

4. Apply each patch in order onto the new base
   ─── apply patch₁ onto E → creates C' (new SHA)
   ─── apply patch₂ onto C' → creates D' (new SHA)

5. Update the feature branch pointer to D'

Result: feature is now a linear extension of main.
```

> **Key insight:** Rebase creates entirely new commit objects. The original commits (C, D) still exist in the object store and can be recovered via `git reflog` — but they are no longer referenced by any branch.

### Interactive Rebase

Interactive rebase (`git rebase -i`) gives you fine-grained control over how commits are replayed. You can reorder, edit, squash, drop, or reword commits before they are applied.

```bash
# Interactively rebase the last 5 commits
git rebase -i HEAD~5

# Interactively rebase onto main
git rebase -i main
```

Git opens an editor with the commit list:

```
pick a1b2c3d Add user model
pick b2c3d4e Add user API endpoint
pick c3d4e5f WIP: debugging auth
pick d4e5f6a Fix auth token validation
pick e5f6a7b Add user tests

# Commands:
# p, pick   = use commit as-is
# r, reword = use commit, but edit the message
# e, edit   = use commit, but stop for amending
# s, squash = merge into previous commit (keep both messages)
# f, fixup  = merge into previous commit (discard this message)
# d, drop   = remove commit entirely
```

Example cleanup:

```
pick a1b2c3d Add user model
pick b2c3d4e Add user API endpoint
fixup c3d4e5f WIP: debugging auth
fixup d4e5f6a Fix auth token validation
pick e5f6a7b Add user tests

# Result: 3 clean commits instead of 5
# 1. Add user model
# 2. Add user API endpoint (with auth fixes folded in)
# 3. Add user tests
```

## Merge vs. Rebase

### Comparison Table

| Dimension | Merge | Rebase |
|---|---|---|
| **History shape** | Non-linear (preserves branches) | Linear (straight line) |
| **Merge commits** | Creates merge commits | No merge commits |
| **Original SHAs** | Preserved | Rewritten (new SHAs) |
| **Conflict resolution** | Resolve once during merge | Resolve per commit during replay |
| **Traceability** | Shows exactly when branches were integrated | Shows what the final sequence of changes was |
| **Complexity** | Simple — one merge operation | Can be complex if many commits conflict |
| **Safety** | Safe for shared branches | Dangerous on shared branches (rewrites history) |
| **Reversibility** | `git revert -m 1 <merge>` | Must reset to reflog entry |

### When to Use Each

| Scenario | Recommended Strategy | Reason |
|---|---|---|
| Integrating a feature branch into main | Merge (`--no-ff`) or squash merge | Preserves the integration point; safe for shared branches |
| Updating a feature branch with latest main | Rebase (`git rebase main`) | Keeps feature branch linear; avoids unnecessary merge commits |
| Cleaning up commits before opening a PR | Interactive rebase (`git rebase -i`) | Produces clean, atomic commits for review |
| Merging a long-lived release branch | Merge | Preserves the full release history |
| Collaborating on a shared feature branch | Merge | Rebase would rewrite commits others depend on |
| Solo work on a local branch not yet pushed | Rebase | No one else is affected; linear history is cleaner |

### The Golden Rule of Rebasing

> **Never rebase commits that have been pushed to a public/shared branch.**

Rebase rewrites commit history — it creates new commits with new SHA-1 hashes. If other developers have based their work on the original commits, rebasing creates divergent histories that are painful to reconcile.

```
The problem: Alice rebases a shared branch

  Before (both Alice and Bob have this):

      A ◄── B ◄── C     (shared-branch)

  Alice rebases onto main:

      A ◄── D ◄── B' ◄── C'    (Alice's rewritten history)

  Bob still has:

      A ◄── B ◄── C            (Bob's original history)

  Bob pulls and gets a mess:

      A ◄── D ◄── B' ◄── C' ◄── M   (merge of divergent histories)
            │                    │
            └── B ◄── C ◄───────┘

  Commits B and C now appear TWICE — once as originals, once as B' and C'.
```

**Safe to rebase:**
- ✅ Local branches that have not been pushed
- ✅ Personal feature branches that only you work on (even if pushed, with force-push)
- ✅ Commits that have not been shared with anyone

**Never rebase:**
- ❌ `main`, `develop`, or any shared integration branch
- ❌ Branches that other developers have checked out or based work on
- ❌ Published release tags

## Merge Strategies

Git supports several merge strategies selected with the `-s` flag. Each handles different scenarios.

### Ours

The **ours** strategy resolves the merge by keeping the current branch's version of every file, discarding all changes from the other branch. The merge commit still records both parents — but the content is entirely from "ours."

```bash
# Keep our version of everything, discard all changes from legacy-branch
git merge -s ours legacy-branch
```

| Use Case | Example |
|---|---|
| Marking a branch as merged without taking its changes | Closing a deprecated feature branch |
| Superseding an old branch with a rewrite | You rewrote the feature from scratch on the current branch |

> **Note:** `-s ours` (strategy) is different from `-X ours` (strategy option). The strategy discards all their changes. The option `-X ours` resolves only **conflicts** in our favor but still merges non-conflicting changes.

### Theirs

There is no built-in `-s theirs` strategy, but you can achieve the same effect using the `-X theirs` strategy option with the recursive strategy:

```bash
# Merge feature and resolve all conflicts in favor of feature's version
git merge -X theirs feature

# This still merges non-conflicting changes from both sides.
# Only conflicts are resolved using "theirs."
```

To truly replace the current branch content with theirs:

```bash
# Nuclear option: replace our content entirely with theirs
git merge -s ours --no-commit their-branch
git read-tree -m -u their-branch
git commit -m "Replace content with their-branch"
```

### Octopus

The **octopus** strategy merges more than two branches simultaneously. It is Git's default when you pass multiple branches to `git merge`. It cannot handle conflicts that require manual resolution.

```bash
# Merge three branches at once
git merge feature/auth feature/payments feature/notifications
```

```
Octopus merge:

                    main
                      │
                      ▼
    A ◄── B ◄─────── M    (octopus merge commit with 4 parents)
          │           │
          ├── C ◄─────┤    (feature/auth)
          │           │
          ├── D ◄─────┤    (feature/payments)
          │           │
          └── E ◄─────┘    (feature/notifications)
```

| Use Case | Limitation |
|---|---|
| Merging multiple independent feature branches simultaneously | Cannot resolve conflicts — fails if any branch conflicts with another |
| Integration branches that collect features before release | Only works when all branches apply cleanly |

### Subtree

The **subtree** strategy is used when one project is merged into a subdirectory of another. It adjusts the tree structure so that the merged project appears in the correct path.

```bash
# Add a library as a subtree in the libs/ directory
git remote add lib-utils https://github.com/org/lib-utils.git
git fetch lib-utils
git merge -s subtree --allow-unrelated-histories lib-utils/main
```

| Use Case | Alternative |
|---|---|
| Incorporating an external project into a subdirectory | `git subtree` (porcelain command) provides a friendlier interface |
| Vendoring dependencies directly into the repository | Submodules (if you prefer a reference rather than a copy) |

## Conflict Resolution

Conflicts occur when Git cannot automatically determine how to combine changes from two branches. This happens when both branches modify the **same lines** in the **same file**, or when one branch modifies a file that the other branch deletes.

### Identifying Conflicts

```bash
# Attempt a merge
git merge feature/auth

# Output when conflicts exist:
# Auto-merging src/auth/login.py
# CONFLICT (content): Merge conflict in src/auth/login.py
# Auto-merging src/config.py
# CONFLICT (modify/delete): src/old-module.py deleted in feature/auth
#   and modified in HEAD. Version HEAD of src/old-module.py left in tree.
# Automatic merge failed; fix conflicts and then commit the result.
```

```bash
# See which files are in conflict
git status

# Output:
# Unmerged paths:
#   both modified:   src/auth/login.py
#   deleted by them: src/old-module.py
```

### Conflict Markers

Git inserts conflict markers into files that it cannot auto-merge:

```python
def authenticate(username, password):
<<<<<<< HEAD
    # Current branch (ours): uses bcrypt
    if not bcrypt.verify(password, user.password_hash):
        raise AuthenticationError("Invalid credentials")
=======
    # Incoming branch (theirs): uses argon2
    if not argon2.verify(password, user.password_hash):
        raise AuthenticationError("Login failed")
>>>>>>> feature/auth
```

| Marker | Meaning |
|---|---|
| `<<<<<<< HEAD` | Start of the current branch's version (ours) |
| `=======` | Separator between the two versions |
| `>>>>>>> feature/auth` | End of the incoming branch's version (theirs) |

### Resolution Workflow

```
Conflict Resolution Steps:

1. Identify conflicted files
   └── git status

2. Open each conflicted file and resolve
   ├── Choose "ours" (current branch version)
   ├── Choose "theirs" (incoming branch version)
   ├── Combine both versions manually
   └── Write something entirely new

3. Remove all conflict markers (<<<, ===, >>>)

4. Stage the resolved files
   └── git add <resolved-file>

5. Complete the merge
   └── git commit     (or git merge --continue)

Bail out at any time:
   └── git merge --abort     (restores pre-merge state)
```

```bash
# Step-by-step conflict resolution
git merge feature/auth
# CONFLICT in src/auth/login.py

# 1. Edit the file and resolve the conflict
vim src/auth/login.py

# 2. Stage the resolved file
git add src/auth/login.py

# 3. Complete the merge
git commit -m "Merge feature/auth: resolve auth library conflict"

# Or abort the merge entirely
git merge --abort
```

### Conflict Resolution Tools

Visual merge tools provide a three-pane view: ours (left), theirs (right), and the merged result (center/bottom).

```bash
# Launch the configured merge tool
git mergetool

# Use a specific tool
git mergetool --tool=vimdiff
git mergetool --tool=vscode
git mergetool --tool=meld
```

#### Configuring Merge Tools

```bash
# Set VS Code as the default merge tool
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd \
  'code --wait --merge $REMOTE $LOCAL $BASE $MERGED'

# Set vimdiff as the default merge tool
git config --global merge.tool vimdiff

# Set IntelliJ as the default merge tool
git config --global merge.tool intellij
git config --global mergetool.intellij.cmd \
  'idea merge $LOCAL $REMOTE $BASE $MERGED'

# Disable .orig backup files after merge
git config --global mergetool.keepBackup false
```

| Tool | Interface | Strengths |
|---|---|---|
| **vimdiff** | Terminal (Vim) | Fast, available everywhere, scriptable |
| **VS Code** | GUI (editor) | Inline conflict buttons (Accept Current/Incoming/Both), familiar interface |
| **IntelliJ / JetBrains** | GUI (IDE) | Three-pane merge, automatic conflict detection, powerful diff |
| **meld** | GUI (standalone) | Clean three-pane view, cross-platform, open source |
| **KDiff3** | GUI (standalone) | Three-way merge with automatic resolution, line-by-line granularity |

### Rerere

**Rerere** (Reuse Recorded Resolution) tells Git to remember how you resolved a conflict so it can automatically apply the same resolution if it encounters the identical conflict again.

```bash
# Enable rerere globally
git config --global rerere.enabled true
```

```
How rerere works:

1. You resolve a conflict manually during a merge
   └── Git records the conflict and your resolution in .git/rr-cache/

2. Later, the same conflict appears (e.g., during rebase or re-merge)
   └── Git recognizes the conflict pattern
   └── Git applies your previous resolution automatically

3. You still need to verify and stage the result
   └── git add <file> && git commit
```

```bash
# See what rerere has recorded
git rerere status

# See the diff of what rerere would apply
git rerere diff

# Forget a specific recorded resolution
git rerere forget src/auth/login.py

# Rerere is especially useful when:
# - Rebasing long-lived branches that repeatedly conflict with main
# - Re-doing a merge after a git merge --abort
# - Cherry-picking commits that cause the same conflict
```

## Merge Commit Strategies in Pull Requests

Platforms like GitHub, GitLab, and Bitbucket offer three strategies when merging a pull request. Each produces a different history shape.

### Merge Commit

Creates a merge commit that joins the feature branch into the target branch. All individual commits from the feature branch are preserved.

```bash
# Equivalent Git command
git merge --no-ff feature/auth
```

```
Before:                         After merge commit:

main: A ◄── B                  main: A ◄── B ◄─────────── M
             │                               │              │
feature:     └── C ◄── D       feature:      └── C ◄── D ◄─┘

M is a merge commit with parents B and D.
Commits C and D remain in the history.
```

**When to use:** When you want to preserve the full branch history and see exactly which commits were part of each feature.

### Squash Merge

Combines all commits from the feature branch into a **single new commit** on the target branch. The feature branch's individual commits are not preserved in the main branch history.

```bash
# Equivalent Git commands
git merge --squash feature/auth
git commit -m "feat(auth): add authentication module (#42)"
```

```
Before:                         After squash merge:

main: A ◄── B                  main: A ◄── B ◄── S
             │
feature:     └── C ◄── D       S contains the combined changes of C and D.
                                Commits C and D do NOT appear in main's history.
```

**When to use:** When the individual commits on the feature branch are noisy (WIP, fixup, review feedback) and only the final result matters.

### Rebase and Merge

Replays each commit from the feature branch onto the tip of the target branch, producing a **linear history** with no merge commit. Each commit gets a new SHA.

```bash
# Equivalent Git commands
git checkout feature/auth
git rebase main
git checkout main
git merge --ff-only feature/auth
```

```
Before:                         After rebase and merge:

main: A ◄── B                  main: A ◄── B ◄── C' ◄── D'
             │
feature:     └── C ◄── D       C' and D' are replayed copies of C and D.
                                History is perfectly linear.
```

**When to use:** When individual commits are clean, atomic, and well-described — and the team values a linear history.

### Comparison

| Strategy | History Shape | Individual Commits | Merge Commit | Reversibility |
|---|---|---|---|---|
| **Merge commit** | Non-linear (branching) | ✅ Preserved | ✅ Yes | `git revert -m 1 <merge>` reverts entire feature |
| **Squash merge** | Linear | ❌ Collapsed into one | ❌ No | `git revert <squash>` reverts the single commit |
| **Rebase and merge** | Linear | ✅ Preserved (new SHAs) | ❌ No | Must revert each commit individually |

## Cherry-Pick

`git cherry-pick` applies the changes from a specific commit onto the current branch. It creates a **new commit** with the same changes but a different SHA.

### Selecting Specific Commits

```bash
# Cherry-pick a single commit
git cherry-pick a1b2c3d

# Cherry-pick multiple commits
git cherry-pick a1b2c3d b2c3d4e c3d4e5f

# Cherry-pick a range of commits (exclusive start, inclusive end)
git cherry-pick A..C    # Picks B and C (not A)

# Cherry-pick without committing (stage changes only)
git cherry-pick --no-commit a1b2c3d

# Cherry-pick and add a reference to the original commit
git cherry-pick -x a1b2c3d
# Appends "(cherry picked from commit a1b2c3d)" to the message
```

```
Cherry-pick operation:

Before:

  main:     A ◄── B ◄── E
                   │
  feature:         └── C ◄── D ◄── F

  Want to apply commit D onto main without merging the entire feature branch.

After: git checkout main && git cherry-pick D

  main:     A ◄── B ◄── E ◄── D'
                   │
  feature:         └── C ◄── D ◄── F

  D' has the same changes as D but a different SHA.
  Feature branch is unaffected.
```

### Cherry-Pick Use Cases

| Use Case | Example |
|---|---|
| **Hotfix backporting** | Apply a critical fix from `main` to a `release/1.x` branch |
| **Selective feature porting** | Bring one specific commit from a large feature branch |
| **Undoing a revert** | Cherry-pick the original commit to re-apply a reverted change |
| **Cross-branch fixes** | Apply a bug fix to multiple maintenance branches |

```bash
# Backport a hotfix to a release branch
git checkout release/1.2
git cherry-pick abc1234
# Commit message: "(cherry picked from commit abc1234)"

# Handle conflicts during cherry-pick
git cherry-pick a1b2c3d
# CONFLICT — resolve the conflict, then:
git add <resolved-file>
git cherry-pick --continue

# Abort a cherry-pick in progress
git cherry-pick --abort
```

> **Caution:** Cherry-picking creates duplicate changes in the history. If the source branch is later merged, Git may encounter the same changes twice. Use `cherry-pick -x` to record provenance and help reviewers understand where the commit originated.

## Best Practices for Clean History

### Integration Strategy

- ✅ Choose **one** integration strategy for your team and document it (merge, squash, or rebase)
- ✅ Use `--no-ff` merges on main/develop to preserve feature branch topology
- ✅ Rebase feature branches onto main before opening a PR to minimize merge conflicts
- ✅ Use squash merge for branches with messy WIP commits
- ✅ Use rebase-and-merge for branches with clean, atomic commits
- ❌ Do not mix merge strategies inconsistently — it creates confusing history
- ❌ Do not rebase shared branches that others have based work on

### Before Merging a Feature Branch

```
Pre-Merge Checklist:
├── 1. Rebase onto latest main (resolve conflicts early)
│       └── git fetch origin && git rebase origin/main
├── 2. Clean up commits with interactive rebase
│       └── git rebase -i origin/main
│           ├── Squash WIP and fixup commits
│           ├── Write clear commit messages
│           └── Ensure each commit is atomic and builds
├── 3. Run tests locally
│       └── Verify all commits pass CI
├── 4. Push and open/update the pull request
│       └── git push --force-with-lease origin feature/auth
└── 5. Merge using the team's agreed strategy
```

### Force Push Safety

- ✅ Use `git push --force-with-lease` instead of `git push --force`
- ✅ `--force-with-lease` fails if someone else has pushed to the branch since your last fetch — preventing accidental overwrites
- ❌ Never force push to `main`, `develop`, or other shared integration branches

```bash
# Safe force push after rebase (only if no one else has pushed)
git push --force-with-lease origin feature/auth

# Dangerous — overwrites remote unconditionally
git push --force origin feature/auth    # ❌ Avoid this
```

### History Readability

- ✅ Write descriptive merge commit messages that explain *what* was integrated
- ✅ Use `git log --oneline --graph` regularly to verify history shape
- ✅ Enable `rerere` to avoid resolving the same conflicts repeatedly
- ✅ Use `cherry-pick -x` to record the source commit when backporting
- ❌ Do not leave conflict markers (`<<<<<<<`) in committed code
- ❌ Do not merge main into a feature branch repeatedly — rebase instead

## Next Steps

Continue to [Remote Collaboration](04-REMOTE-COLLABORATION.md) to learn about working with remote repositories, pull request workflows, code review best practices, and forking strategies for open-source contributions.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial merging and rebasing documentation |
