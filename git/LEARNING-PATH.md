# Git Learning Path

A structured, self-paced training guide to mastering Git — from basic version control fundamentals to advanced workflows, automation, and production-scale strategies. Each phase builds on the previous one, progressing from core commands to expert-level techniques.

> **Time Estimate:** 13–14 weeks at ~5 hours/week. Adjust pace to your experience level.

---

## Table of Contents

- [How to Use This Guide](#how-to-use-this-guide)
- [Phase 1: Git Fundamentals (Week 1–2)](#phase-1-git-fundamentals-week-12)
- [Phase 2: Branching and Merging (Week 3–4)](#phase-2-branching-and-merging-week-34)
- [Phase 3: Remote Collaboration (Week 5–6)](#phase-3-remote-collaboration-week-56)
- [Phase 4: History and Commits (Week 7–8)](#phase-4-history-and-commits-week-78)
- [Phase 5: Advanced Techniques (Week 9–10)](#phase-5-advanced-techniques-week-910)
- [Phase 6: Automation and Security (Week 11–12)](#phase-6-automation-and-security-week-1112)
- [Phase 7: Scaling and Production (Week 13–14)](#phase-7-scaling-and-production-week-1314)
- [Quick Reference: Document Map](#quick-reference-document-map)
- [Recommended Resources](#recommended-resources)
- [Certification and Assessment](#certification-and-assessment)
- [Version History](#version-history)

---

## How to Use This Guide

1. **Follow the phases in order** — each one builds on prior knowledge
2. **Read the linked documents** — they contain the detailed content
3. **Complete the exercises** — hands-on practice solidifies understanding
4. **Check yourself** — use the knowledge checks before moving on
5. **Practice in a real repository** — apply concepts to your own projects

---

## Phase 1: Git Fundamentals (Week 1–2)

### Learning Objectives

- Install and configure Git on your platform
- Understand Git's object model: blobs, trees, commits, and refs
- Create repositories and manage the working directory, staging area, and commit history
- Use basic commands: `init`, `add`, `commit`, `status`, `diff`, `log`
- Configure user identity, editor, and global settings

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | Core concepts, architecture, object model, configuration |

### Exercises

**1. Installation and Configuration:**

Set up Git on your machine and configure your environment:

- Install Git and verify with `git --version`
- Configure your identity: `git config --global user.name` and `user.email`
- Set your preferred editor: `git config --global core.editor`
- Create a global `.gitignore` for OS and editor files
- Run `git config --list` and review all settings

**2. Repository Creation and Basic Workflow:**

Create a new project and practice the core workflow:

- Initialize a repository with `git init`
- Create three files and add them in separate commits
- Use `git status` between each step to observe state changes
- Use `git diff` and `git diff --staged` to inspect changes
- Run `git log --oneline --graph` to view commit history

**3. Exploring the Object Model:**

Inspect Git internals to understand how data is stored:

- Use `git cat-file -t <hash>` to inspect object types
- Use `git cat-file -p <hash>` to view object contents
- Explore the `.git` directory: `objects/`, `refs/`, `HEAD`
- Create a file, stage it, and trace the blob → tree → commit chain

### Knowledge Check

- [ ] What are the three areas in Git (working directory, staging area, repository)?
- [ ] What is the difference between `git add` and `git commit`?
- [ ] What are the four Git object types and how do they relate to each other?
- [ ] What does `HEAD` point to and why is it important?
- [ ] How does `git diff` differ from `git diff --staged`?

---

## Phase 2: Branching and Merging (Week 3–4)

### Learning Objectives

- Create, switch, and delete branches
- Understand fast-forward vs. three-way merges
- Perform rebasing and understand when to use it vs. merging
- Resolve merge conflicts confidently
- Learn branching strategies: Git Flow, GitHub Flow, trunk-based development

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [01-BRANCHING-STRATEGIES](01-BRANCHING-STRATEGIES.md) | Branch models, Git Flow, GitHub Flow, trunk-based development |
| 2 | [03-MERGING-AND-REBASING](03-MERGING-AND-REBASING.md) | Fast-forward, three-way merge, rebase, conflict resolution |

### Exercises

**1. Branch Management:**

Practice the full branch lifecycle:

- Create a feature branch: `git switch -c feature/add-login`
- Make several commits on the feature branch
- Switch back to `main` and make a diverging commit
- Merge the feature branch into `main` using `--no-ff`
- Delete the merged branch: `git branch -d feature/add-login`
- Use `git log --oneline --graph --all` to visualize the result

**2. Conflict Resolution:**

Deliberately create and resolve a merge conflict:

- Create two branches from the same commit
- Edit the same line of the same file on both branches
- Attempt to merge and observe the conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`)
- Resolve the conflict manually, then complete the merge
- Repeat the exercise using `git rebase` instead of merge

**3. Branching Strategy Comparison:**

For a team of five developers working on a SaaS product:

- Diagram the Git Flow workflow for a release cycle
- Diagram the GitHub Flow workflow for the same feature
- Diagram a trunk-based development workflow
- List the pros and cons of each for your team size
- Recommend one strategy and justify your choice

### Knowledge Check

- [ ] What is the difference between a fast-forward merge and a three-way merge?
- [ ] When should you use `rebase` instead of `merge`?
- [ ] What is the golden rule of rebasing?
- [ ] How does Git Flow differ from trunk-based development?
- [ ] What are merge conflict markers and how do you resolve them?

---

## Phase 3: Remote Collaboration (Week 5–6)

### Learning Objectives

- Configure and manage remote repositories
- Understand `push`, `pull`, `fetch`, and their differences
- Create and review pull requests effectively
- Work with forks and upstream repositories
- Collaborate using code review best practices

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [04-REMOTE-COLLABORATION](04-REMOTE-COLLABORATION.md) | Remotes, push/pull/fetch, PRs, forking, code review |

### Exercises

**1. Remote Workflow:**

Set up and work with remote repositories:

- Create a repository on GitHub and clone it locally
- Add a second remote (`upstream`) pointing to another repository
- Push a feature branch: `git push -u origin feature/my-feature`
- Use `git fetch --all` and inspect remote tracking branches
- Use `git pull --rebase` to sync changes cleanly
- Practice `git remote -v`, `git branch -r`, and `git branch -vv`

**2. Pull Request Workflow:**

Complete a full pull request cycle:

- Fork an open-source repository
- Create a feature branch and make a small improvement
- Push the branch to your fork
- Open a pull request against the upstream repository
- Respond to review feedback by pushing follow-up commits
- Rebase and squash before the final merge

**3. Code Review Practice:**

Review a teammate's (or your own) pull request and check for:

- [ ] Clear, descriptive PR title and description
- [ ] Each commit is atomic and has a meaningful message
- [ ] No unrelated changes included
- [ ] Tests are included for new functionality
- [ ] No secrets, credentials, or debug code committed

### Knowledge Check

- [ ] What is the difference between `git fetch` and `git pull`?
- [ ] What does `git push -u origin feature` do?
- [ ] How does a fork differ from a branch?
- [ ] What are remote tracking branches and how do you inspect them?
- [ ] What makes a good pull request description?

---

## Phase 4: History and Commits (Week 7–8)

### Learning Objectives

- Write clear, conventional commit messages
- Navigate and search commit history effectively
- Use `blame`, `log`, `reflog`, and `show` for investigation
- Understand commit amendments and history rewriting implications
- Maintain a clean, readable project history

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [02-COMMITS-AND-HISTORY](02-COMMITS-AND-HISTORY.md) | Conventional commits, history management, blame, log, reflog |

### Exercises

**1. Conventional Commits:**

Practice writing well-structured commit messages:

- Write commits using the format: `type(scope): description`
- Use at least five different types: `feat`, `fix`, `docs`, `refactor`, `test`
- Write a commit with a body and footer (including `BREAKING CHANGE:`)
- Review your last 10 commits and rewrite any unclear messages using `git rebase -i`

**2. History Investigation:**

Use history tools to answer questions about a repository:

- Use `git log --oneline --since="2 weeks ago"` to find recent changes
- Use `git log --author="name"` to filter by contributor
- Use `git log -S "function_name"` to find when a function was introduced
- Use `git blame <file>` to identify who last modified each line
- Use `git show <commit>` to inspect a specific commit's changes

**3. Reflog Recovery:**

Practice recovering from mistakes using the reflog:

- Make a commit, then reset it with `git reset --hard HEAD~1`
- Use `git reflog` to find the lost commit hash
- Recover the commit with `git cherry-pick <hash>` or `git reset --hard <hash>`
- Repeat the exercise with a deleted branch: create, delete, then recover

### Knowledge Check

- [ ] What are the parts of a conventional commit message?
- [ ] What does `git log -S "search_term"` do (pickaxe search)?
- [ ] How does `git reflog` differ from `git log`?
- [ ] When is it safe to amend a commit vs. when is it dangerous?
- [ ] What information does `git blame` show and when is it useful?

---

## Phase 5: Advanced Techniques (Week 9–10)

### Learning Objectives

- Use interactive rebase to clean up commit history
- Cherry-pick individual commits across branches
- Use `git bisect` to find the commit that introduced a bug
- Manage work-in-progress with `stash` and `worktree`
- Understand `reset`, `revert`, and their appropriate use cases

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [05-ADVANCED-TECHNIQUES](05-ADVANCED-TECHNIQUES.md) | Interactive rebase, cherry-pick, bisect, stash, worktree, reset vs. revert |

### Exercises

**1. Interactive Rebase:**

Clean up a messy branch history:

- Create a branch with five commits, including typo fixes and "WIP" messages
- Use `git rebase -i HEAD~5` to:
  - Squash the typo fix into the original commit
  - Reword unclear commit messages
  - Reorder commits for logical grouping
  - Drop an unnecessary commit
- Verify the result with `git log --oneline`

**2. Git Bisect:**

Find the commit that introduced a bug:

- Create a repository with 10+ commits where one introduces a "bug" (e.g., a specific string in a file)
- Start bisect: `git bisect start`, `git bisect bad`, `git bisect good <hash>`
- Walk through the bisect process, marking commits as `good` or `bad`
- Automate the process: `git bisect run <test-script>`
- Finish with `git bisect reset`

**3. Stash and Worktree:**

Manage multiple tasks without losing work:

- Make changes to several files without committing
- Stash them: `git stash push -m "work in progress: feature X"`
- Switch branches, do other work, switch back
- Apply the stash: `git stash pop`
- Create a worktree: `git worktree add ../hotfix hotfix-branch`
- Work in both directories simultaneously
- Clean up: `git worktree remove ../hotfix`

### Knowledge Check

- [ ] What operations can you perform during an interactive rebase?
- [ ] When should you use `cherry-pick` vs. `merge`?
- [ ] How does `git bisect` determine the number of steps needed?
- [ ] What is the difference between `git stash pop` and `git stash apply`?
- [ ] When would you use `git worktree` instead of switching branches?

---

## Phase 6: Automation and Security (Week 11–12)

### Learning Objectives

- Write and configure Git hooks for workflow automation
- Use pre-commit frameworks for code quality enforcement
- Integrate Git with CI/CD pipelines
- Sign commits and tags with GPG or SSH keys
- Manage secrets and prevent accidental credential leaks

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [06-HOOKS-AND-AUTOMATION](06-HOOKS-AND-AUTOMATION.md) | Git hooks, pre-commit framework, CI/CD integration |
| 2 | [08-SECURITY-AND-SIGNING](08-SECURITY-AND-SIGNING.md) | GPG/SSH signing, secrets management, credential protection |

### Exercises

**1. Git Hooks Setup:**

Configure hooks for a project:

- Write a `pre-commit` hook that runs a linter on staged files
- Write a `commit-msg` hook that enforces conventional commit format
- Write a `pre-push` hook that runs tests before pushing
- Install the hooks by placing them in `.git/hooks/` and making them executable
- Test each hook by triggering the corresponding Git action

**2. Pre-commit Framework:**

Set up the pre-commit framework for team-wide enforcement:

- Install pre-commit: `pip install pre-commit`
- Create a `.pre-commit-config.yaml` with hooks for:
  - Trailing whitespace removal
  - End-of-file fixer
  - YAML/JSON validation
  - Secret detection (e.g., `detect-secrets`)
- Run `pre-commit install` and `pre-commit run --all-files`
- Add a CI step that runs `pre-commit run --all-files`

**3. Commit Signing:**

Configure and verify signed commits:

- Generate a GPG key: `gpg --full-generate-key`
- Configure Git to sign commits: `git config --global commit.gpgsign true`
- Make a signed commit and verify with `git log --show-signature`
- Upload your public key to GitHub and verify the "Verified" badge
- Alternatively, set up SSH signing: `git config --global gpg.format ssh`

### Knowledge Check

- [ ] What is the difference between client-side and server-side hooks?
- [ ] Which hook runs before a commit is created and can prevent it?
- [ ] How does the pre-commit framework differ from raw Git hooks?
- [ ] Why should commits be signed and what does signing verify?
- [ ] What tools can detect secrets accidentally staged for commit?

---

## Phase 7: Scaling and Production (Week 13–14)

### Learning Objectives

- Work with monorepos and understand their trade-offs
- Use Git submodules and subtrees for multi-repository management
- Configure Git LFS for large file handling
- Apply best practices for production Git workflows
- Identify and avoid common Git anti-patterns

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [07-MONOREPOS-AND-SUBMODULES](07-MONOREPOS-AND-SUBMODULES.md) | Monorepos, submodules, subtrees, Git LFS |
| 2 | [09-BEST-PRACTICES](09-BEST-PRACTICES.md) | Production readiness, workflow standards, team conventions |
| 3 | [10-ANTI-PATTERNS](10-ANTI-PATTERNS.md) | Common mistakes, bad habits, and how to fix them |

### Exercises

**1. Submodule Management:**

Practice adding and updating submodules:

- Create a parent repository and a library repository
- Add the library as a submodule: `git submodule add <url> libs/library`
- Clone the parent repository and initialize submodules: `git clone --recurse-submodules`
- Update the submodule to a new version: `cd libs/library && git pull && cd ../.. && git add libs/library && git commit`
- Remove a submodule and clean up all references

**2. Git LFS Setup:**

Configure LFS for large binary files:

- Install Git LFS: `git lfs install`
- Track large file types: `git lfs track "*.psd" "*.zip" "*.bin"`
- Verify `.gitattributes` is updated and committed
- Add a large file, commit, and push — verify it is stored via LFS
- Use `git lfs ls-files` to list tracked files

**3. Anti-Pattern Audit:**

Review the following workflow description and identify the anti-patterns:

> "Our team of 15 developers all work directly on the `main` branch. We commit large files (build artifacts, database dumps) directly to the repository. Commit messages are usually 'fix' or 'update'. We force-push to shared branches regularly, and we store API keys in configuration files. Nobody reviews pull requests because we don't use them."

<details>
<summary>Anti-patterns identified (click to reveal)</summary>

1. **No branching strategy** — all developers committing directly to `main`
2. **Large files in repository** — build artifacts and database dumps bloating history
3. **Poor commit messages** — non-descriptive messages like "fix" and "update"
4. **Force-pushing shared branches** — rewriting history others depend on
5. **Secrets in source code** — API keys stored in configuration files
6. **No code review process** — no pull requests or review gates

</details>

### Knowledge Check

- [ ] When should you use a monorepo vs. multiple repositories?
- [ ] What is the difference between `git submodule` and `git subtree`?
- [ ] What types of files should be tracked with Git LFS?
- [ ] Why should you never force-push to a shared branch?
- [ ] What are the signs that a repository has grown too large?

---

## Quick Reference: Document Map

| # | Document | Phase |
|---|----------|-------|
| 00 | [OVERVIEW](00-OVERVIEW.md) | 1 |
| 01 | [BRANCHING-STRATEGIES](01-BRANCHING-STRATEGIES.md) | 2 |
| 02 | [COMMITS-AND-HISTORY](02-COMMITS-AND-HISTORY.md) | 4 |
| 03 | [MERGING-AND-REBASING](03-MERGING-AND-REBASING.md) | 2 |
| 04 | [REMOTE-COLLABORATION](04-REMOTE-COLLABORATION.md) | 3 |
| 05 | [ADVANCED-TECHNIQUES](05-ADVANCED-TECHNIQUES.md) | 5 |
| 06 | [HOOKS-AND-AUTOMATION](06-HOOKS-AND-AUTOMATION.md) | 6 |
| 07 | [MONOREPOS-AND-SUBMODULES](07-MONOREPOS-AND-SUBMODULES.md) | 7 |
| 08 | [SECURITY-AND-SIGNING](08-SECURITY-AND-SIGNING.md) | 6 |
| 09 | [BEST-PRACTICES](09-BEST-PRACTICES.md) | 7 |
| 10 | [ANTI-PATTERNS](10-ANTI-PATTERNS.md) | 7 |

---

## Recommended Resources

### Books

| Book | Author | Focus |
|------|--------|-------|
| *Pro Git* | Scott Chacon & Ben Straub | Comprehensive Git reference (free online) |
| *Git for Teams* | Emma Jane Hogbin Westby | Team workflows and collaboration |
| *Head First Git* | Raju Gandhi | Visual, beginner-friendly introduction |
| *Version Control with Git* | Prem Kumar Ponuthorai & Jon Loeliger | In-depth Git internals |
| *Git Pocket Guide* | Richard Silverman | Quick reference for daily use |

### Online Resources

- **git-scm.com/doc** — Official Git documentation and Pro Git book
- **learngitbranching.js.org** — Interactive visual tutorial for branching and merging
- **gitimmersion.com** — Guided, hands-on Git walkthrough
- **ohshitgit.com** — Quick fixes for common Git mistakes
- **conventionalcommits.org** — Conventional commit message specification

### Courses

| Course | Platform | Level |
|--------|----------|-------|
| Git & GitHub for Beginners | freeCodeCamp (YouTube) | Beginner |
| Version Control with Git | Atlassian (Coursera) | Beginner–Intermediate |
| Git Advanced Techniques | LinkedIn Learning | Intermediate–Advanced |
| How Git Works | Pluralsight | Intermediate |

### Tools

| Tool | Purpose |
|------|---------|
| Git | Distributed version control |
| GitHub / GitLab / Bitbucket | Repository hosting and collaboration |
| pre-commit | Git hook management framework |
| git-lfs | Large file storage extension |
| lazygit | Terminal-based Git UI |
| tig | Text-mode interface for Git |
| delta | Enhanced diff viewer |
| BFG Repo-Cleaner | Repository history cleanup |
| detect-secrets | Secret detection in commits |
| GPG / SSH | Commit and tag signing |

---

## Certification and Assessment

### Skills Checklist

Use this checklist to self-assess your Git proficiency. Check each item only when you can perform it confidently and explain why it works.

#### Fundamentals

- [ ] Initialize a repository and configure user identity
- [ ] Stage, commit, and diff changes
- [ ] Explain the Git object model (blob, tree, commit, tag)
- [ ] Navigate the `.git` directory and inspect objects
- [ ] Write a `.gitignore` file with appropriate patterns

#### Branching and Merging

- [ ] Create, switch, and delete branches
- [ ] Perform fast-forward and three-way merges
- [ ] Rebase a feature branch onto an updated main branch
- [ ] Resolve merge conflicts manually
- [ ] Choose an appropriate branching strategy for a given team

#### Remote Collaboration

- [ ] Clone, fetch, pull, and push to remote repositories
- [ ] Work with multiple remotes (origin, upstream)
- [ ] Create and review pull requests
- [ ] Fork a repository and contribute back via PR
- [ ] Use remote tracking branches effectively

#### History and Commits

- [ ] Write conventional commit messages with type, scope, and body
- [ ] Search history with `log`, `blame`, `show`, and pickaxe (`-S`)
- [ ] Use `reflog` to recover lost commits and branches
- [ ] Amend commits safely (local vs. shared branches)
- [ ] Maintain a clean, linear project history

#### Advanced Techniques

- [ ] Perform interactive rebase (squash, reword, reorder, drop)
- [ ] Cherry-pick commits across branches
- [ ] Use `bisect` to find bug-introducing commits
- [ ] Stash and restore work-in-progress changes
- [ ] Use `worktree` to work on multiple branches simultaneously

#### Automation and Security

- [ ] Write custom Git hooks (pre-commit, commit-msg, pre-push)
- [ ] Configure the pre-commit framework with shared hook definitions
- [ ] Sign commits with GPG or SSH keys
- [ ] Detect and prevent secrets from entering the repository
- [ ] Integrate Git hooks with CI/CD pipelines

#### Scaling and Production

- [ ] Evaluate monorepo vs. multi-repo trade-offs
- [ ] Add, update, and remove Git submodules
- [ ] Configure Git LFS for large binary files
- [ ] Apply production best practices to a team workflow
- [ ] Identify and remediate common Git anti-patterns

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-07-14 | Initial learning path covering all seven phases |
