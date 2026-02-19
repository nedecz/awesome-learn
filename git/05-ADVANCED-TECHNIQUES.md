# Git Advanced Techniques

## Table of Contents

1. [Overview](#overview)
2. [Interactive Rebase](#interactive-rebase)
   - [Reordering Commits](#reordering-commits)
   - [Editing Commits](#editing-commits)
   - [Squashing Commits](#squashing-commits)
   - [Fixup Commits](#fixup-commits)
   - [Dropping Commits](#dropping-commits)
3. [Cherry-Pick](#cherry-pick)
   - [Basic Cherry-Pick](#basic-cherry-pick)
   - [Cherry-Picking a Range](#cherry-picking-a-range)
   - [Conflict Handling](#conflict-handling)
4. [Git Bisect](#git-bisect)
   - [Binary Search for Bugs](#binary-search-for-bugs)
   - [Automated Bisect with Scripts](#automated-bisect-with-scripts)
5. [Git Reflog](#git-reflog)
   - [Recovering Lost Commits](#recovering-lost-commits)
   - [Understanding Reflog Entries](#understanding-reflog-entries)
   - [Reflog Expiration](#reflog-expiration)
6. [Git Stash](#git-stash)
   - [Stash Save, Pop, Apply, and Drop](#stash-save-pop-apply-and-drop)
   - [Stash with Message](#stash-with-message)
   - [Partial Stashing](#partial-stashing)
7. [Git Worktree](#git-worktree)
   - [Multiple Working Trees](#multiple-working-trees)
   - [Use Cases](#use-cases)
   - [Managing Worktrees](#managing-worktrees)
8. [Git Filter-Branch and Filter-Repo](#git-filter-branch-and-filter-repo)
   - [Rewriting History](#rewriting-history)
   - [Removing Sensitive Data](#removing-sensitive-data)
   - [BFG Repo-Cleaner](#bfg-repo-cleaner)
9. [Git Subtree](#git-subtree)
   - [Alternative to Submodules](#alternative-to-submodules)
   - [Adding, Pulling, and Pushing Subtrees](#adding-pulling-and-pushing-subtrees)
10. [Sparse Checkout](#sparse-checkout)
    - [Working with Large Repos](#working-with-large-repos)
    - [Cone Mode](#cone-mode)
11. [Git Blame and Log Advanced](#git-blame-and-log-advanced)
    - [Pickaxe Search](#pickaxe-search)
    - [Advanced Log Formatting](#advanced-log-formatting)
    - [Diff-Filter](#diff-filter)
12. [Best Practices](#best-practices)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

## Overview

Advanced Git techniques go beyond the everyday `add`, `commit`, and `push` workflow. They give you fine-grained control over project history, help you recover from mistakes, and enable efficient workflows in large or complex repositories. Mastering these tools is what separates a competent Git user from a truly effective one.

### Target Audience

- Developers who are comfortable with basic Git operations and want deeper control
- Tech leads responsible for maintaining clean project history
- DevOps engineers managing large repositories and CI/CD pipelines

### Scope

- Interactive rebase for rewriting local history
- Cherry-pick for selectively applying commits
- Bisect for tracking down bugs efficiently
- Reflog for recovering lost work
- Stash, worktree, filter-repo, subtree, sparse checkout
- Advanced blame and log techniques

## Interactive Rebase

Interactive rebase (`git rebase -i`) lets you rewrite commit history by reordering, editing, squashing, fixing up, or dropping commits. It is the single most powerful tool for keeping a clean, readable history before sharing your work.

```
Before interactive rebase              After interactive rebase

  * d4e5f6 Fix typo in readme            * a1b2c3' Add user auth feature
  * c3d4e5 WIP save                         (squashed, reordered, clean)
  * b2c3d4 Add user auth feature
  * a1b2c3 Initial commit                * a1b2c3 Initial commit
```

Launch interactive rebase for the last N commits:

```bash
# Rebase the last 4 commits
git rebase -i HEAD~4

# Rebase onto a specific base
git rebase -i main
```

Git opens your editor with a list of commits (oldest first):

```
pick a1b2c3d Add login endpoint
pick b2c3d4e Add session management
pick c3d4e5f Fix typo in login
pick d4e5f6a Add logout endpoint
```

### Reordering Commits

Rearrange lines in the editor to change the order commits are applied. Git replays them in the new sequence.

```
pick b2c3d4e Add session management      ← moved up
pick a1b2c3d Add login endpoint          ← moved down
pick d4e5f6a Add logout endpoint
pick c3d4e5f Fix typo in login
```

> ⚠️ Reordering can cause conflicts if later commits depend on earlier ones. Resolve conflicts as prompted and run `git rebase --continue`.

### Editing Commits

Use `edit` to pause the rebase at a specific commit so you can amend it — change the code, the message, or both.

```
pick a1b2c3d Add login endpoint
edit b2c3d4e Add session management      ← rebase pauses here
pick c3d4e5f Fix typo in login
pick d4e5f6a Add logout endpoint
```

When the rebase pauses:

```bash
# Make your changes to files
vim src/session.js

# Amend the commit
git add src/session.js
git commit --amend

# Continue the rebase
git rebase --continue
```

### Squashing Commits

Use `squash` (or `s`) to merge a commit into the one above it. Git combines the commit messages and lets you edit the result.

```
pick a1b2c3d Add login endpoint
squash c3d4e5f Fix typo in login         ← merges into a1b2c3d
pick b2c3d4e Add session management
pick d4e5f6a Add logout endpoint
```

The combined commit message editor opens:

```
# This is a combination of 2 commits.
# This is the 1st commit message:

Add login endpoint

# This is the commit message #2:

Fix typo in login
```

### Fixup Commits

Use `fixup` (or `f`) to merge a commit into the one above it but **discard** the fixup commit's message. This is ideal for small corrections.

```
pick a1b2c3d Add login endpoint
fixup c3d4e5f Fix typo in login          ← merged, message discarded
pick b2c3d4e Add session management
pick d4e5f6a Add logout endpoint
```

Git also supports `--autosquash` to automatically reorder fixup commits created with `git commit --fixup`:

```bash
# Create a fixup commit targeting a1b2c3d
git commit --fixup a1b2c3d

# Rebase with autosquash — fixup commits are placed automatically
git rebase -i --autosquash main
```

### Dropping Commits

Use `drop` (or `d`) to remove a commit entirely, or simply delete the line from the editor.

```
pick a1b2c3d Add login endpoint
drop c3d4e5f Fix typo in login           ← this commit is removed
pick b2c3d4e Add session management
pick d4e5f6a Add logout endpoint
```

### Interactive Rebase Command Summary

| Command | Short | Effect |
|---|---|---|
| `pick` | `p` | Keep the commit as-is |
| `reword` | `r` | Keep the commit but edit its message |
| `edit` | `e` | Pause rebase to amend the commit |
| `squash` | `s` | Merge into previous commit, combine messages |
| `fixup` | `f` | Merge into previous commit, discard this message |
| `drop` | `d` | Remove the commit entirely |
| `exec` | `x` | Run a shell command after the commit |
| `break` | `b` | Pause rebase at this point |

## Cherry-Pick

Cherry-pick applies the changes from a specific commit onto your current branch. It creates a **new** commit with the same diff but a different SHA.

```
main:       A ── B ── C ── D
                       \
feature:                E ── F ── G

git cherry-pick F

main:       A ── B ── C ── D ── F'
                       \
feature:                E ── F ── G
```

### Basic Cherry-Pick

```bash
# Apply a single commit by SHA
git cherry-pick abc1234

# Apply without committing (stage changes only)
git cherry-pick --no-commit abc1234

# Cherry-pick and edit the commit message
git cherry-pick --edit abc1234
```

### Cherry-Picking a Range

```bash
# Cherry-pick a range of commits (exclusive of start, inclusive of end)
git cherry-pick A..C          # applies B and C

# Inclusive range (includes A)
git cherry-pick A^..C         # applies A, B, and C

# Cherry-pick multiple individual commits
git cherry-pick abc1234 def5678 ghi9012
```

```
Before                                After cherry-pick A^..C

main:  M1 ── M2                       main:  M1 ── M2 ── A' ── B' ── C'
                \                                    \
feature:         A ── B ── C           feature:       A ── B ── C
```

### Conflict Handling

When a cherry-pick encounters conflicts:

```bash
# 1. Git pauses and reports the conflict
git cherry-pick abc1234
# CONFLICT (content): Merge conflict in src/app.js

# 2. Resolve the conflict in your editor
vim src/app.js

# 3. Stage the resolved file
git add src/app.js

# 4. Continue the cherry-pick
git cherry-pick --continue

# Or abort the cherry-pick entirely
git cherry-pick --abort

# Or skip the current commit and continue with the rest
git cherry-pick --skip
```

| Option | Effect |
|---|---|
| `--continue` | Resume after resolving conflicts |
| `--abort` | Cancel and restore the original branch state |
| `--skip` | Skip the conflicting commit and proceed |
| `--no-commit` | Apply changes without creating a commit |
| `-x` | Append "cherry picked from commit ..." to the message |

## Git Bisect

Git bisect performs a binary search through your commit history to find the exact commit that introduced a bug. Instead of checking every commit linearly, bisect cuts the search space in half each step.

```
                       bisect narrows down
     ◀──────────────────────────────────────▶

     A ── B ── C ── D ── E ── F ── G ── H
     ✅                                  ❌
     good                                bad

     Step 1: test D (midpoint)
     A ── B ── C ── D ── E ── F ── G ── H
                     ✅

     Step 2: test F (midpoint of D..H)
     A ── B ── C ── D ── E ── F ── G ── H
                                    ✅

     Step 3: test G
     A ── B ── C ── D ── E ── F ── G ── H
                                    ❌
     Result: G introduced the bug
```

### Binary Search for Bugs

```bash
# Start bisect session
git bisect start

# Mark the current commit as bad (has the bug)
git bisect bad

# Mark a known good commit (before the bug existed)
git bisect good v2.0.0

# Git checks out a midpoint commit — test it
# Then mark it as good or bad
git bisect good     # if this commit works
git bisect bad      # if this commit has the bug

# Repeat until Git identifies the first bad commit
# Bisecting: 0 revisions left to test after this (roughly 0 steps)
# abc1234 is the first bad commit

# End the bisect session and return to your branch
git bisect reset
```

For N commits, bisect requires at most **log₂(N)** steps. For 1000 commits, that is roughly 10 tests.

### Automated Bisect with Scripts

You can automate bisect by providing a test script. Git runs the script at each step and uses the exit code to determine good (exit 0) or bad (exit 1–124, 127, or 128+).

```bash
# Automated bisect with a test script
git bisect start HEAD v2.0.0
git bisect run ./test-for-bug.sh

# Or use a one-liner test
git bisect run bash -c 'make test 2>&1 | grep -q "PASS"'

# Using a test suite directly
git bisect run npm test
```

Example test script (`test-for-bug.sh`):

```bash
#!/bin/bash
# Exit 0 = good commit, Exit 1 = bad commit
# Exit 125 = skip (cannot test this commit)

# Build the project
make build || exit 125

# Run the specific test that fails
./run-test --case=login-redirect
exit $?
```

| Exit Code | Meaning |
|---|---|
| `0` | Good commit (bug not present) |
| `1–124, 128+` | Bad commit (bug is present) |
| `125` | Skip this commit (cannot be tested, e.g., build failure) |

## Git Reflog

The reflog (reference log) records every change to the tip of branches and HEAD in your local repository. It is your safety net — even after a hard reset, an accidental branch deletion, or a botched rebase, the reflog remembers where you were.

```
HEAD@{0}: reset: moving to HEAD~3
HEAD@{1}: commit: Add payment processing
HEAD@{2}: commit: Add order validation
HEAD@{3}: commit: Add cart functionality
HEAD@{4}: checkout: moving from feature to main
```

### Recovering Lost Commits

```bash
# View the reflog
git reflog

# View reflog for a specific branch
git reflog show feature/auth

# Recover a commit after an accidental hard reset
git reset --hard HEAD~3          # oops, lost 3 commits

git reflog                       # find the lost commit SHA
# abc1234 HEAD@{1}: commit: Add payment processing

git reset --hard abc1234         # restore to that commit

# Or create a new branch from the lost commit
git branch recovered-work abc1234
```

```
Accidental reset                     Recovery via reflog

main: A ── B ── C ── D              main: A ── B ── C ── D
                ▲                                          ▲
             HEAD (after reset)                         HEAD (restored)

Commits C and D are still in          git reset --hard abc1234
the object store — reflog             restores HEAD to D
knows about them
```

### Understanding Reflog Entries

Each reflog entry records:

| Field | Description | Example |
|---|---|---|
| **SHA** | The commit hash HEAD pointed to | `abc1234` |
| **Selector** | Position in the reflog | `HEAD@{0}`, `HEAD@{5}` |
| **Action** | What caused the change | `commit`, `reset`, `rebase`, `checkout` |
| **Details** | Additional context | `moving to main`, `moving to HEAD~3` |

```bash
# View reflog with timestamps
git reflog --date=iso

# View reflog with relative times
git reflog --date=relative
# abc1234 HEAD@{2 hours ago}: commit: Add payment processing

# Filter reflog to a specific time window
git reflog --since="2 days ago" --until="1 day ago"
```

### Reflog Expiration

Reflog entries are not permanent. Git garbage-collects them based on expiration rules.

| Type | Default Expiration | Description |
|---|---|---|
| **Reachable entries** | 90 days | Entries reachable from current branch tips |
| **Unreachable entries** | 30 days | Entries for commits no longer reachable from any ref |

```bash
# View current expiration settings
git config gc.reflogExpire             # default: 90 days
git config gc.reflogExpireUnreachable  # default: 30 days

# Change expiration (use with caution)
git config gc.reflogExpire "180 days"
git config gc.reflogExpireUnreachable "60 days"

# Manually expire old reflog entries
git reflog expire --expire=now --all

# Run garbage collection (also prunes expired reflogs)
git gc
```

> ⚠️ Setting `gc.reflogExpireUnreachable` too low reduces your safety net for recovering lost work. The default of 30 days is a reasonable balance.

## Git Stash

Git stash saves uncommitted changes (both staged and unstaged) to a temporary stack so you can switch context — change branches, pull updates, or work on something else — and then restore them later.

```
Working directory (dirty)       After git stash         After git stash pop

  modified: app.js                (clean)               modified: app.js
  modified: styles.css                                  modified: styles.css
  staged:   utils.js                                    staged:   utils.js
            │                                                     ▲
            ▼                                                     │
       ┌──────────┐                                          ┌──────────┐
       │  Stash   │                                          │  Stash   │
       │  Stack   │                                          │  (empty) │
       └──────────┘                                          └──────────┘
```

### Stash Save, Pop, Apply, and Drop

```bash
# Save all uncommitted changes to the stash
git stash

# Save and include untracked files
git stash --include-untracked    # or git stash -u

# Save everything including ignored files
git stash --all

# Restore the most recent stash and remove it from the stack
git stash pop

# Restore the most recent stash but keep it on the stack
git stash apply

# Apply a specific stash entry
git stash apply stash@{2}

# Drop a specific stash entry
git stash drop stash@{0}

# Clear the entire stash stack
git stash clear

# List all stash entries
git stash list
# stash@{0}: WIP on feature: abc1234 Add login
# stash@{1}: WIP on main: def5678 Update readme
```

### Stash with Message

Give your stashes descriptive names to remember what they contain:

```bash
# Stash with a custom message
git stash push -m "WIP: payment form validation"

# The message shows in the stash list
git stash list
# stash@{0}: On feature: WIP: payment form validation
# stash@{1}: WIP on main: def5678 Update readme
```

### Partial Stashing

Stash only specific files or interactively select hunks to stash:

```bash
# Stash specific files only
git stash push -m "stash styles" -- src/styles.css src/theme.css

# Interactive stash — choose which hunks to stash
git stash push --patch
# Git prompts you for each hunk:
#   Stash this hunk [y,n,q,a,d,/,s,e,?]?

# Stash only staged changes (keep unstaged changes in working directory)
git stash push --staged
```

| Command | Effect |
|---|---|
| `git stash` | Stash all tracked changes (staged + unstaged) |
| `git stash -u` | Also stash untracked files |
| `git stash --all` | Also stash ignored files |
| `git stash pop` | Apply top stash and remove it |
| `git stash apply` | Apply top stash but keep it |
| `git stash drop` | Remove top stash without applying |
| `git stash clear` | Remove all stash entries |
| `git stash push --patch` | Interactively select hunks to stash |
| `git stash push --staged` | Stash only staged changes |
| `git stash branch <name>` | Create a branch from a stash entry |

## Git Worktree

Git worktree lets you check out multiple branches simultaneously in separate directories, all backed by a single `.git` repository. This eliminates the need to stash, commit, or clone to switch context.

```
Single repository, multiple working trees

  .git/                              ← shared object store
    │
    ├── main-worktree/               ← main branch checked out
    │     ├── src/
    │     └── package.json
    │
    ├── ../feature-worktree/         ← feature/auth branch checked out
    │     ├── src/
    │     └── package.json
    │
    └── ../hotfix-worktree/          ← hotfix/login branch checked out
          ├── src/
          └── package.json
```

### Multiple Working Trees

```bash
# Add a new worktree for an existing branch
git worktree add ../feature-auth feature/auth

# Add a new worktree and create a new branch
git worktree add -b hotfix/login ../hotfix-login main

# List all worktrees
git worktree list
# /home/user/project           abc1234 [main]
# /home/user/feature-auth      def5678 [feature/auth]
# /home/user/hotfix-login      ghi9012 [hotfix/login]
```

### Use Cases

| Scenario | Benefit |
|---|---|
| Reviewing a PR while working on a feature | No stashing or committing WIP required |
| Running long tests on one branch while developing on another | Parallel workflows in separate directories |
| Comparing behavior between branches side by side | Both branches are checked out and runnable |
| Building a release while fixing a hotfix | Isolated environments, shared object store |

### Managing Worktrees

```bash
# Remove a worktree (after you are done with it)
git worktree remove ../feature-auth

# Prune stale worktree references (e.g., manually deleted directories)
git worktree prune

# Move a worktree to a new location
git worktree move ../feature-auth ../auth-feature

# Lock a worktree to prevent pruning (useful for network drives)
git worktree lock ../feature-auth
git worktree unlock ../feature-auth
```

> ⚠️ A branch can only be checked out in one worktree at a time. If `feature/auth` is checked out in a worktree, you cannot check it out elsewhere until that worktree is removed.

## Git Filter-Branch and Filter-Repo

Sometimes you need to rewrite repository history at a large scale — removing sensitive data, changing author information, or restructuring directories. These tools rewrite every affected commit in the history.

### Rewriting History

`git filter-branch` is the legacy tool. `git filter-repo` is its modern, faster, and safer replacement.

```bash
# Install git-filter-repo (recommended over filter-branch)
pip install git-filter-repo

# Change author email across all commits
git filter-repo --email-callback '
    return email.replace(b"old@example.com", b"new@example.com")
'

# Rename a directory across all history
git filter-repo --path-rename src/old-name/:src/new-name/

# Remove a directory from all history
git filter-repo --invert-paths --path secrets/
```

### Removing Sensitive Data

If a password, API key, or other secret was accidentally committed:

```bash
# Remove a specific file from all history
git filter-repo --invert-paths --path config/secrets.yml

# Replace specific text strings across all history
git filter-repo --replace-text expressions.txt
```

Where `expressions.txt` contains:

```
PASSWORD123==>***REMOVED***
api_key_abc==>***REMOVED***
```

```
Before                                After filter-repo

  commit A: Add config                  commit A': Add config
    config/secrets.yml  ← secret          (file removed from history)
    config/app.yml                        config/app.yml

  commit B: Update config              commit B': Update config
    config/secrets.yml  ← secret          (file removed from history)
    config/app.yml                        config/app.yml
```

> ⚠️ After rewriting history, all commit SHAs change. Every collaborator must re-clone or `git pull --rebase` against the rewritten history. Coordinate with your team before doing this.

### BFG Repo-Cleaner

BFG Repo-Cleaner is a simpler, faster alternative to `filter-branch` specifically designed for removing large files or sensitive data.

```bash
# Remove all files over 100MB from history
java -jar bfg.jar --strip-blobs-bigger-than 100M

# Remove specific files by name
java -jar bfg.jar --delete-files "*.pem"

# Replace sensitive text
java -jar bfg.jar --replace-text passwords.txt

# After BFG, clean up
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

| Tool | Speed | Ease of Use | Flexibility |
|---|---|---|---|
| `git filter-branch` | Slow | Complex | High (arbitrary scripts) |
| `git filter-repo` | Fast | Moderate | High (callbacks, path filters) |
| **BFG Repo-Cleaner** | Very fast | Simple | Focused (files, text replacement) |

## Git Subtree

Git subtree embeds an external repository inside a subdirectory of your project. Unlike submodules, the subtree's files are part of your repository — no special commands are needed to clone or update for other contributors.

### Alternative to Submodules

| Feature | Submodules | Subtree |
|---|---|---|
| **Files in repo** | No (pointer only) | Yes (full copy) |
| **Clone behavior** | Requires `--recurse-submodules` | Normal clone works |
| **Update workflow** | `git submodule update` | `git subtree pull` |
| **Contributor friction** | High (extra commands) | Low (transparent) |
| **Push back upstream** | Work in submodule directory | `git subtree push` |
| **History** | Separate | Merged into project history |

### Adding, Pulling, and Pushing Subtrees

```bash
# Add a subtree from a remote repository
git subtree add --prefix=lib/shared \
    https://github.com/org/shared-lib.git main --squash

# Pull updates from the upstream repository
git subtree pull --prefix=lib/shared \
    https://github.com/org/shared-lib.git main --squash

# Push local changes back to the upstream repository
git subtree push --prefix=lib/shared \
    https://github.com/org/shared-lib.git main
```

```
Your repository after subtree add

  my-project/
  ├── src/
  ├── lib/
  │   └── shared/            ← subtree (full copy of shared-lib)
  │       ├── index.js
  │       └── utils.js
  ├── package.json
  └── .git/
```

The `--squash` option merges all upstream history into a single commit, keeping your project history clean. Without it, every upstream commit appears in your log.

## Sparse Checkout

Sparse checkout lets you check out only a subset of files from a repository. This is essential for working with large monorepos where you only need a fraction of the codebase.

### Working with Large Repos

```bash
# Clone with no files checked out
git clone --no-checkout https://github.com/org/monorepo.git
cd monorepo

# Initialize sparse checkout
git sparse-checkout init

# Specify the directories you need
git sparse-checkout set src/frontend docs/frontend

# Check out files (only the specified directories appear)
git checkout main
```

```
Full repository                     After sparse checkout

  monorepo/                          monorepo/
  ├── src/                           ├── src/
  │   ├── frontend/   ✅             │   └── frontend/    ✅
  │   ├── backend/    ╌╌             ├── docs/
  │   ├── mobile/     ╌╌             │   └── frontend/    ✅
  │   └── infra/      ╌╌             └── .git/
  ├── docs/
  │   ├── frontend/   ✅
  │   ├── backend/    ╌╌
  │   └── api/        ╌╌
  ├── tools/          ╌╌
  └── .git/

  ╌╌ = not checked out (saves disk space and time)
```

### Cone Mode

Cone mode (the default since Git 2.37) is faster and more predictable. It restricts patterns to directory-level matching rather than arbitrary gitignore patterns.

```bash
# Enable cone mode explicitly (default in modern Git)
git sparse-checkout init --cone

# Add directories to checkout
git sparse-checkout set src/frontend src/shared

# Add more directories without replacing existing ones
git sparse-checkout add docs/frontend

# List current sparse-checkout directories
git sparse-checkout list

# Disable sparse checkout and restore all files
git sparse-checkout disable
```

| Mode | Pattern Style | Performance | Use Case |
|---|---|---|---|
| **Cone mode** | Directory-based | Fast | Most monorepo workflows |
| **Non-cone mode** | Gitignore-style | Slower | Fine-grained file-level control |

## Git Blame and Log Advanced

Beyond basic `git log` and `git blame`, Git provides powerful options for tracing code changes, searching for specific modifications, and understanding how a codebase evolved.

### Pickaxe Search

The pickaxe (`-S` and `-G`) finds commits that introduced or removed a specific string or pattern.

```bash
# Find commits that added or removed the string "calculateTotal"
git log -S "calculateTotal"

# Find commits where the number of occurrences of "calculateTotal" changed
git log -S "calculateTotal" --diff-filter=M -- "*.js"

# Find commits matching a regex pattern
git log -G "def (calculate|compute)_total"

# Show the actual diffs for pickaxe matches
git log -S "calculateTotal" -p
```

| Flag | Behavior |
|---|---|
| `-S "string"` | Find commits that change the **count** of `string` (added or removed) |
| `-G "regex"` | Find commits where the **diff** matches the regex |

### Advanced Log Formatting

```bash
# Visualize the full branch and merge history
git log --all --graph --oneline --decorate
# * abc1234 (HEAD -> main) Merge feature/auth
# |\
# | * def5678 (feature/auth) Add OAuth support
# | * ghi9012 Add login page
# |/
# * jkl3456 Initial commit

# Log commits that touched a specific function (requires language support)
git log -L :calculateTotal:src/billing.js

# Log commits between two tags
git log v1.0.0..v2.0.0 --oneline

# Log commits by author with stats
git log --author="Alice" --stat --since="2 weeks ago"

# Show only merge commits
git log --merges --oneline

# Show only non-merge commits
git log --no-merges --oneline

# Custom format
git log --pretty=format:"%h %ad | %s%d [%an]" --date=short
# abc1234 2025-01-15 | Add auth feature (HEAD -> main) [Alice]
```

### Diff-Filter

Filter `git log` and `git diff` output to show only commits that added, modified, deleted, or renamed files.

```bash
# Show only commits that added new files
git log --diff-filter=A --name-only --oneline
# abc1234 Add user module
# src/user.js

# Show only commits that deleted files
git log --diff-filter=D --name-only --oneline

# Show only commits that modified files (not added or deleted)
git log --diff-filter=M --name-only -- "*.js"

# Show renamed files
git diff --diff-filter=R --name-status HEAD~5..HEAD
```

| Filter | Meaning |
|---|---|
| `A` | Added |
| `C` | Copied |
| `D` | Deleted |
| `M` | Modified |
| `R` | Renamed |
| `T` | Type changed |

### Advanced Blame

```bash
# Blame with line range
git blame -L 10,20 src/app.js

# Ignore whitespace changes in blame
git blame -w src/app.js

# Detect lines moved within the same file
git blame -M src/app.js

# Detect lines moved across files
git blame -C src/app.js

# Show the commit that last moved/copied each line
git blame -C -C -C src/app.js

# Ignore specific revisions (e.g., bulk formatting commits)
echo "abc1234" >> .git-blame-ignore-revs
git config blame.ignoreRevsFile .git-blame-ignore-revs
git blame src/app.js
```

## Best Practices

- ✅ Use interactive rebase to clean up local history **before** pushing to a shared branch
- ✅ Use `--autosquash` with fixup commits to streamline rebase workflows
- ✅ Always use `git cherry-pick -x` to annotate where a commit was picked from
- ✅ Use `git bisect run` with an automated test script for repeatable bug hunts
- ✅ Check the reflog before panicking — your work is almost certainly recoverable
- ✅ Give stashes descriptive messages so you remember their purpose
- ✅ Use worktrees instead of stashing or cloning when switching context frequently
- ✅ Prefer `git filter-repo` over `git filter-branch` for history rewriting
- ✅ Use sparse checkout cone mode for large monorepo workflows
- ✅ Maintain a `.git-blame-ignore-revs` file for bulk formatting commits
- ❌ Do not rebase or rewrite commits that have been pushed to a shared branch
- ❌ Do not rely on reflog for long-term recovery — entries expire after 30–90 days
- ❌ Do not use `git stash` as a long-term storage mechanism — commit or branch instead
- ❌ Do not run `filter-branch` or `filter-repo` without coordinating with your team
- ❌ Do not force-push rewritten history to shared branches without explicit team agreement

## Next Steps

Continue to [Hooks and Automation](06-HOOKS-AND-AUTOMATION.md) to learn about Git hooks, pre-commit checks, CI/CD integration, and automating your Git workflows for consistency and quality.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial advanced techniques documentation |
