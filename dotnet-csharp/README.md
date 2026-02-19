# .NET and C# Learning Resources

A comprehensive guide to .NET and C# development — from language fundamentals and framework architecture to production-grade services, best practices, and migration strategies.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | .NET ecosystem, C# language evolution, architecture components | **Start here** |
| [01-BREAKING-CHANGES-DOTNET-CORE-31](01-BREAKING-CHANGES-DOTNET-CORE-31.md) | Breaking changes from .NET Core 3.1 through .NET 10, migration guides | When migrating or upgrading |
| [02-SERVICES](02-SERVICES.md) | Web APIs, worker services, gRPC, configuration, logging | When building .NET services |
| [03-BEST-PRACTICES](03-BEST-PRACTICES.md) | Async/await, memory management, security, testing, API design | **Essential — production checklist** |
| [04-STYLE-GUIDE](04-STYLE-GUIDE.md) | C# naming conventions, formatting, analyzers, EditorConfig | When writing or reviewing code |
| [05-CONCURRENCY-ASYNC-PARALLELISM](05-CONCURRENCY-ASYNC-PARALLELISM.md) | Async deep dive, TPL, PLINQ, channels, job queuing | When building concurrent systems |
| [06-DEPENDENCY-INJECTION](06-DEPENDENCY-INJECTION.md) | DI patterns, lifetimes, keyed services, testing, anti-patterns | When designing service architectures |
| [07-ENTITY-FRAMEWORK-CORE](07-ENTITY-FRAMEWORK-CORE.md) | EF Core, DbContext, migrations, LINQ queries, Dapper, performance | When building data access layers |
| [08-TESTING](08-TESTING.md) | Unit testing, integration testing, xUnit, mocking, test architecture | When writing or improving tests |
| [09-SECURITY-AND-AUTHENTICATION](09-SECURITY-AND-AUTHENTICATION.md) | Identity, OAuth2/OIDC, JWT, CORS, data protection, secrets | When securing applications |
| [10-MIDDLEWARE-AND-HTTP-PIPELINE](10-MIDDLEWARE-AND-HTTP-PIPELINE.md) | Middleware, request pipeline, filters, rate limiting, output caching | When customizing the HTTP pipeline |
| [11-DESIGN-PATTERNS](11-DESIGN-PATTERNS.md) | Factory, Strategy, Mediator, CQRS, Repository, Result pattern | When designing application architecture |
| [12-CONFIGURATION-AND-DEPLOYMENT](12-CONFIGURATION-AND-DEPLOYMENT.md) | Docker, CI/CD, health checks, feature flags, publishing | When deploying to production |
| [13-ANTI-PATTERNS](13-ANTI-PATTERNS.md) | Common .NET anti-patterns and fixes across all areas | **Essential — what NOT to do** |
| [LEARNING-PATH](LEARNING-PATH.md) | Structured 12-week training guide | **Start here** after the Overview |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Understand the .NET ecosystem and its components
   - Learn the history from .NET Framework to modern .NET
   - Explore C# language highlights

2. **Learn About Services** ([02-SERVICES](02-SERVICES.md))
   - Dependency injection fundamentals
   - Building your first Web API
   - Configuration and logging patterns

3. **Follow the Style Guide** ([04-STYLE-GUIDE](04-STYLE-GUIDE.md))
   - C# naming conventions
   - Code formatting standards
   - Analyzer and EditorConfig setup

4. **Follow the Learning Path** ([LEARNING-PATH](LEARNING-PATH.md))
   - Structured, phased curriculum
   - Hands-on exercises and knowledge checks

### For Experienced Users

1. **Review Breaking Changes** ([01-BREAKING-CHANGES-DOTNET-CORE-31](01-BREAKING-CHANGES-DOTNET-CORE-31.md))
   - Migration path from .NET Core 3.1 to .NET 10
   - API removals and behavioral changes
   - Before/after code migration patterns

2. **Review Best Practices** ([03-BEST-PRACTICES](03-BEST-PRACTICES.md))
   - Production readiness checklist
   - Performance and memory optimization
   - Security hardening

3. **Deep Dive into Concurrency** ([05-CONCURRENCY-ASYNC-PARALLELISM](05-CONCURRENCY-ASYNC-PARALLELISM.md))
   - Async/await internals and best practices
   - Task Parallel Library and PLINQ
   - Channels and job queuing patterns

4. **Master Dependency Injection** ([06-DEPENDENCY-INJECTION](06-DEPENDENCY-INJECTION.md))
   - Advanced DI patterns and keyed services
   - Scope validation and diagnostics
   - Testing with DI

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Your Application                         │
│            (Console, Web API, Blazor, MAUI, Worker)             │
└──────────────────────────┬──────────────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   ┌─────────────┐  ┌───────────┐  ┌──────────────┐
   │ ASP.NET     │  │ EF Core   │  │ Worker       │
   │ Core        │  │ (ORM)     │  │ Services     │
   │             │  │           │  │              │
   │ • Web API   │  │ • LINQ    │  │ • Background │
   │ • Blazor    │  │ • Migrate │  │   tasks      │
   │ • gRPC      │  │ • DbCtx   │  │ • Hosted     │
   │ • SignalR   │  │           │  │   services   │
   └──────┬──────┘  └─────┬─────┘  └──────┬───────┘
          │                │               │
          └────────┬───────┴───────┬───────┘
                   ▼               ▼
          ┌──────────────┐  ┌──────────────┐
          │  .NET SDK    │  │  .NET CLI    │
          │  (compiler,  │  │  (dotnet     │
          │   analyzers, │  │   build,     │
          │   NuGet)     │  │   run, test) │
          └──────┬───────┘  └──────┬───────┘
                 │                 │
                 └────────┬────────┘
                          ▼
              ┌───────────────────────┐
              │   .NET Runtime (CLR)  │
              │                       │
              │  • Garbage Collector  │
              │  • JIT Compiler       │
              │  • Type System        │
              │  • Thread Pool        │
              └───────────┬───────────┘
                          ▼
              ┌───────────────────────┐
              │  Base Class Library   │
              │  (BCL)               │
              │                       │
              │  • Collections        │
              │  • IO / Networking    │
              │  • LINQ               │
              │  • System.Text.Json   │
              │  • Cryptography       │
              └───────────────────────┘
```

## 🔑 Key Concepts

### Type System

```
C# Type System
├── Value Types           (struct, enum, int, bool, DateTime)
├── Reference Types       (class, interface, delegate, string, array)
├── Nullable Types        (int?, string?, Nullable<T>)
├── Generic Types         (List<T>, Dictionary<TKey, TValue>)
├── Record Types          (record, record struct — immutable data)
└── Span / Memory         (Span<T>, Memory<T> — stack-safe slices)
```

### Async/Await

```
Asynchronous Programming
├── Task / Task<T>        (represents an asynchronous operation)
├── async / await         (compiler-generated state machines)
├── ValueTask<T>          (reduces allocations for sync paths)
├── IAsyncEnumerable<T>   (async streams)
└── ConfigureAwait        (context capture control)
```

### LINQ

```
Language Integrated Query
├── Query Syntax           (from x in collection where ... select ...)
├── Method Syntax          (collection.Where(...).Select(...))
├── Deferred Execution     (query runs when iterated)
├── IQueryable<T>          (translates to SQL via EF Core)
└── LINQ to Objects        (in-memory collection operations)
```

### Dependency Injection

```
Built-in DI Container
├── Service Lifetimes
│   ├── Singleton          (one instance for application lifetime)
│   ├── Scoped             (one instance per request/scope)
│   └── Transient          (new instance every time)
├── Registration
│   ├── AddSingleton<T>()
│   ├── AddScoped<T>()
│   └── AddTransient<T>()
└── Resolution
    ├── Constructor Injection  (preferred)
    ├── IServiceProvider       (service locator — avoid)
    └── Keyed Services         (.NET 8+)
```

## 📋 Topics Covered

- **Fundamentals** — .NET ecosystem, C# language evolution, architecture components
- **Breaking Changes** — Migration from .NET Core 3.1 through .NET 10, API removals, behavioral changes
- **Services** — Web APIs, worker services, gRPC, configuration, logging
- **Best Practices** — Async/await, memory management, security, testing, API design
- **Style Guide** — Naming conventions, formatting, analyzers, EditorConfig
- **Concurrency & Parallelism** — Async deep dive, TPL, PLINQ, channels, job queuing
- **Dependency Injection** — DI patterns, lifetimes, keyed services, testing, anti-patterns
- **Entity Framework Core** — DbContext, migrations, LINQ queries, Dapper, performance optimization
- **Testing** — Unit testing, integration testing, xUnit, mocking, test architecture
- **Security & Authentication** — Identity, OAuth2/OIDC, JWT, CORS, data protection, secrets
- **Middleware & HTTP Pipeline** — Middleware, filters, rate limiting, output caching
- **Design Patterns** — Factory, Strategy, Mediator, CQRS, Repository, Result pattern
- **Configuration & Deployment** — Docker, CI/CD, health checks, feature flags, publishing
- **Anti-Patterns** — Common .NET anti-patterns across async, DI, EF Core, memory, API design
- **Learning Path** — Structured 12-week curriculum with hands-on exercises

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to .NET?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md)

**Migrating from .NET Core 3.1?** → Read [01-BREAKING-CHANGES-DOTNET-CORE-31.md](01-BREAKING-CHANGES-DOTNET-CORE-31.md)

**Building services?** → Review [02-SERVICES.md](02-SERVICES.md) and [03-BEST-PRACTICES.md](03-BEST-PRACTICES.md)

**Working with databases?** → Read [07-ENTITY-FRAMEWORK-CORE.md](07-ENTITY-FRAMEWORK-CORE.md)

**Need concurrency or job queuing?** → Read [05-CONCURRENCY-ASYNC-PARALLELISM.md](05-CONCURRENCY-ASYNC-PARALLELISM.md)

**Writing tests?** → Read [08-TESTING.md](08-TESTING.md)

**Securing your app?** → Read [09-SECURITY-AND-AUTHENTICATION.md](09-SECURITY-AND-AUTHENTICATION.md)

**Designing DI architecture?** → Read [06-DEPENDENCY-INJECTION.md](06-DEPENDENCY-INJECTION.md)

**Deploying to production?** → Read [12-CONFIGURATION-AND-DEPLOYMENT.md](12-CONFIGURATION-AND-DEPLOYMENT.md)

**Avoiding pitfalls?** → Read [13-ANTI-PATTERNS.md](13-ANTI-PATTERNS.md)

**Want a structured path?** → Follow the [Learning Path](LEARNING-PATH.md)
