# Git Learning Resources

A comprehensive guide to Git version control — from fundamental concepts and branching strategies to advanced techniques, automation, and team collaboration best practices.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | Git fundamentals, distributed version control | **Start here** |
| [01-BRANCHING-STRATEGIES](01-BRANCHING-STRATEGIES.md) | Git Flow, GitHub Flow, trunk-based development | When choosing a workflow |
| [02-COMMITS-AND-HISTORY](02-COMMITS-AND-HISTORY.md) | Commit best practices, conventional commits | When writing commits |
| [03-MERGING-AND-REBASING](03-MERGING-AND-REBASING.md) | Merge strategies, rebasing, conflict resolution | When integrating changes |
| [04-REMOTE-COLLABORATION](04-REMOTE-COLLABORATION.md) | Remotes, pull requests, code review | When collaborating with others |
| [05-ADVANCED-TECHNIQUES](05-ADVANCED-TECHNIQUES.md) | Interactive rebase, cherry-pick, bisect, reflog | When you need advanced operations |
| [06-HOOKS-AND-AUTOMATION](06-HOOKS-AND-AUTOMATION.md) | Git hooks, CI/CD integration | When automating workflows |
| [07-MONOREPOS-AND-SUBMODULES](07-MONOREPOS-AND-SUBMODULES.md) | Monorepo strategies, submodules, subtrees | When managing multi-project repos |
| [08-SECURITY-AND-SIGNING](08-SECURITY-AND-SIGNING.md) | GPG signing, secrets management | When securing your repository |
| [09-BEST-PRACTICES](09-BEST-PRACTICES.md) | Workflow best practices, team conventions | **Essential — team guidelines** |
| [10-ANTI-PATTERNS](10-ANTI-PATTERNS.md) | Common mistakes and pitfalls | **Essential — what NOT to do** |
| [LEARNING-PATH](LEARNING-PATH.md) | Structured learning guide | **Start here** after the Overview |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Understand what Git is and why it matters
   - Learn distributed vs. centralized version control
   - Explore the basic Git object model

2. **Learn Branching Strategies** ([01-BRANCHING-STRATEGIES](01-BRANCHING-STRATEGIES.md))
   - Understand feature branches and release branches
   - Compare Git Flow, GitHub Flow, and trunk-based development
   - Choose the right workflow for your team

3. **Master Commits and History** ([02-COMMITS-AND-HISTORY](02-COMMITS-AND-HISTORY.md))
   - Write clear, meaningful commit messages
   - Follow conventional commits specification
   - Navigate and understand project history

4. **Follow the Learning Path** ([LEARNING-PATH](LEARNING-PATH.md))
   - Structured, phased curriculum
   - Hands-on exercises and knowledge checks

### For Experienced Users

1. **Review Best Practices** ([09-BEST-PRACTICES](09-BEST-PRACTICES.md))
   - Workflow best practices for teams
   - Branch naming and commit conventions
   - Repository hygiene and maintenance

2. **Avoid Anti-Patterns** ([10-ANTI-PATTERNS](10-ANTI-PATTERNS.md))
   - Force-push pitfalls
   - Monolithic commit traps
   - Common history-rewriting mistakes

3. **Deep Dive into Advanced Techniques** ([05-ADVANCED-TECHNIQUES](05-ADVANCED-TECHNIQUES.md))
   - Interactive rebase for clean history
   - Cherry-pick and bisect workflows
   - Recovering lost work with reflog

## 🔑 Key Concepts

### Git Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Remote Repository                       │
│                  (GitHub, GitLab, Bitbucket)                 │
└──────────────────────┬──────────────────────────────────────┘
                       │
                 push  │  ▲  fetch/pull
                       ▼  │
┌─────────────────────────────────────────────────────────────┐
│                     Local Repository                         │
│                  (.git directory)                            │
│                                                             │
│   ┌─────────────┐  commit  ┌──────────────┐                │
│   │  Staging     │ ───────▶ │   Commit     │                │
│   │  Area        │          │   History    │                │
│   │  (Index)     │ ◀─────── │   (Objects)  │                │
│   └──────┬───────┘  reset   └──────────────┘                │
│          │  ▲                                                │
│    add   │  │  restore                                      │
│          ▼  │                                                │
│   ┌──────────────┐                                          │
│   │  Working      │                                          │
│   │  Directory    │                                          │
│   │  (Your Files) │                                          │
│   └──────────────┘                                          │
└─────────────────────────────────────────────────────────────┘
```

### Core Workflow

```
Git Workflow
├── Working Directory          (edit files)
├── Staging Area               (select changes to commit)
├── Local Repository           (save snapshots of your project)
└── Remote Repository          (share and collaborate)
```

### Branching Models

```
Git Flow                             GitHub Flow
├── main (production)                ├── main (production)
├── develop (integration)            └── feature/* (short-lived)
├── feature/* (new features)
├── release/* (release prep)         Trunk-Based Development
└── hotfix/* (urgent fixes)          ├── main (trunk)
                                     └── short-lived branches (< 1 day)
```

## 📋 Topics Covered

- **Fundamentals** — Core concepts, distributed version control, Git object model
- **Branching** — Git Flow, GitHub Flow, trunk-based development, branch management
- **Commits** — Commit best practices, conventional commits, meaningful messages
- **Merging** — Merge strategies, rebasing, conflict resolution, fast-forward vs. no-ff
- **Collaboration** — Remotes, pull requests, code review, forking workflows
- **Advanced** — Interactive rebase, cherry-pick, bisect, reflog, stash
- **Automation** — Git hooks, pre-commit checks, CI/CD integration
- **Monorepos** — Monorepo strategies, submodules, subtrees, sparse checkout
- **Security** — GPG signing, secrets management, credential storage
- **Best Practices** — Workflow conventions, team guidelines, repository maintenance
- **Anti-Patterns** — Force-push mistakes, large binary files, history rewriting pitfalls

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to Git?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md)

**Choosing a workflow?** → Read [01-BRANCHING-STRATEGIES.md](01-BRANCHING-STRATEGIES.md) and [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md)

**Going to production?** → Review [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) and [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md)

**Want a structured path?** → Follow the [Learning Path](LEARNING-PATH.md)
