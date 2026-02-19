# .NET and C# Overview

## Table of Contents

1. [Overview](#overview)
2. [History of .NET](#history-of-net)
3. [.NET Architecture Components](#net-architecture-components)
4. [C# Language Evolution](#c-language-evolution)
5. [.NET Framework vs .NET Core vs .NET 5+](#net-framework-vs-net-core-vs-net-5)
6. [When to Use .NET](#when-to-use-net)
7. [When NOT to Use .NET](#when-not-to-use-net)
8. [.NET Project Types](#net-project-types)
9. [Prerequisites](#prerequisites)
10. [Next Steps](#next-steps)

## Overview

This documentation provides a comprehensive introduction to the .NET platform and C# programming language — a cross-platform, open-source framework for building modern applications including web services, desktop apps, cloud-native microservices, and mobile applications.

### Target Audience

- Developers new to .NET and C# looking for a structured introduction
- Developers migrating from .NET Framework to modern .NET
- Software architects evaluating .NET as a technology choice
- Experienced developers from other ecosystems (Java, Node.js, Python) exploring .NET

### Scope

- History and evolution of the .NET platform
- Core architecture components (CLR, BCL, SDK, runtime)
- C# language evolution from C# 7 through C# 12
- Comparison of .NET Framework, .NET Core, and .NET 5+
- When .NET is (and is not) the right choice
- Available project types and templates
- Prerequisites and recommended tooling

## History of .NET

The .NET platform has undergone significant transformation since its initial release in 2002.

### Timeline

```
2002    .NET Framework 1.0    — Windows-only, first release
  │
2004    .NET Framework 2.0    — Generics, nullable types
  │
2007    .NET Framework 3.5    — LINQ, lambda expressions
  │
2012    .NET Framework 4.5    — async/await, portable class libraries
  │
2014    .NET Core announced   — Open-source, cross-platform rewrite
  │
2016    .NET Core 1.0         — First cross-platform release
  │
2017    .NET Core 2.0         — .NET Standard 2.0, broader API surface
  │
2019    .NET Core 3.1 (LTS)   — Windows desktop support, Blazor Server
  │
2020    .NET 5                — Unification begins, single .NET going forward
  │
2021    .NET 6 (LTS)          — Minimal APIs, hot reload, MAUI preview
  │
2022    .NET 7                — Rate limiting, output caching, native AOT
  │
2023    .NET 8 (LTS)          — Native AOT for ASP.NET, keyed DI, Blazor United
  │
2024    .NET 9                — Hybrid cache, OpenAPI built-in, AI extensions
  │
2025    .NET 10 (LTS)         — C# 13, extended AOT, improved DI, Blazor enhancements
```

### Key Milestones

| Year | Milestone | Significance |
|------|-----------|-------------|
| 2002 | .NET Framework 1.0 | Established managed runtime for Windows |
| 2014 | .NET Foundation | Open-source governance established |
| 2016 | .NET Core 1.0 | Cross-platform .NET became reality |
| 2019 | .NET Core 3.1 | Last release branded as ".NET Core" |
| 2020 | .NET 5 | Unification of .NET Framework and .NET Core |
| 2021 | .NET 6 | First unified LTS release |
| 2023 | .NET 8 | Full native AOT support for web workloads |
| 2025 | .NET 10 | Latest LTS — C# 13, extended AOT, DI improvements |

## .NET Architecture Components

```
┌─────────────────────────────────────────────────────────────────┐
│                     Developer Tools                             │
│   Visual Studio │ VS Code │ Rider │ .NET CLI (dotnet)          │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                        .NET SDK                                 │
│                                                                 │
│   Compiler (Roslyn)  │  NuGet  │  MSBuild  │  Project System   │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                    Application Frameworks                        │
│                                                                 │
│  ASP.NET Core  │  EF Core  │  MAUI  │  Blazor  │  Orleans      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│              Base Class Library (BCL)                            │
│                                                                 │
│  System.Collections  │  System.IO  │  System.Net               │
│  System.Linq         │  System.Text.Json  │  System.Threading  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│              Common Language Runtime (CLR)                       │
│                                                                 │
│  JIT Compiler  │  Garbage Collector  │  Type System             │
│  Exception Handling  │  Thread Pool  │  Interop (P/Invoke)     │
└─────────────────────────────────────────────────────────────────┘
```

### Component Descriptions

| Component | Role | Key Responsibilities |
|-----------|------|---------------------|
| **CLR** | Runtime engine | JIT compilation, garbage collection, type safety, exception handling |
| **BCL** | Standard libraries | Collections, IO, networking, serialization, cryptography, LINQ |
| **SDK** | Development toolkit | Compiler (Roslyn), NuGet package manager, MSBuild, project templates |
| **Roslyn** | Compiler platform | Compiles C#/VB.NET, powers code analysis, supports source generators |
| **NuGet** | Package manager | Dependency resolution, package publishing, versioning |
| **MSBuild** | Build engine | Project compilation, output generation, publish profiles |

### Runtime Execution Model

```
   C# Source Code (.cs)
          │
          ▼
   ┌──────────────┐
   │   Roslyn      │    Compile time
   │   Compiler    │
   └──────┬───────┘
          │
          ▼
   Intermediate Language (IL)
   + Metadata (.dll / .exe)
          │
          ▼
   ┌──────────────┐
   │   CLR         │    Runtime
   │   JIT         │
   │   Compiler    │
   └──────┬───────┘
          │
          ▼
   Native Machine Code
          │
          ▼
   ┌──────────────┐
   │   Execution   │
   │   (managed    │
   │    by CLR)    │
   └──────────────┘
```

## C# Language Evolution

C# has evolved rapidly with annual releases. Below are the key features introduced in each version.

### C# 7.x (2017–2019)

| Feature | Example |
|---------|---------|
| Tuples and deconstruction | `var (x, y) = GetPoint();` |
| Pattern matching (basic) | `if (obj is int i)` |
| Local functions | `int Add(int a, int b) => a + b;` inside a method |
| `ref` returns and locals | `ref int Find(int[] arr, int val)` |
| `async Main` | `static async Task Main(string[] args)` |
| Default interface methods | Interface methods with implementations |

### C# 8.0 (2019)

| Feature | Example |
|---------|---------|
| Nullable reference types | `string? name = null;` |
| Switch expressions | `var result = x switch { 1 => "one", _ => "other" };` |
| `using` declarations | `using var stream = File.OpenRead(path);` |
| Async streams | `await foreach (var item in GetItemsAsync())` |
| Index and range | `array[^1]`, `array[1..3]` |

### C# 9.0 (2020)

| Feature | Example |
|---------|---------|
| Records | `public record Person(string Name, int Age);` |
| `init`-only setters | `public string Name { get; init; }` |
| Top-level statements | No `Main` method boilerplate |
| Target-typed `new` | `List<int> list = new();` |
| Pattern matching (relational) | `x is > 0 and < 100` |

### C# 10.0 (2021)

| Feature | Example |
|---------|---------|
| Global usings | `global using System.Linq;` |
| File-scoped namespaces | `namespace MyApp;` (one line) |
| Record structs | `public record struct Point(int X, int Y);` |
| Constant interpolated strings | `const string s = $"Hello {name}";` (if name is const) |
| Extended property patterns | `{ Address.City: "Seattle" }` |

### C# 11.0 (2022)

| Feature | Example |
|---------|---------|
| Raw string literals | `"""{ "key": "value" }"""` |
| Required members | `public required string Name { get; init; }` |
| Generic math | `T Add<T>(T a, T b) where T : INumber<T>` |
| List patterns | `if (list is [1, 2, ..])` |
| UTF-8 string literals | `ReadOnlySpan<byte> utf8 = "Hello"u8;` |

### C# 12.0 (2023)

| Feature | Example |
|---------|---------|
| Primary constructors | `public class Service(ILogger logger)` |
| Collection expressions | `int[] arr = [1, 2, 3];` |
| `ref readonly` parameters | `void Process(ref readonly LargeStruct data)` |
| Default lambda parameters | `var add = (int a, int b = 0) => a + b;` |
| Alias any type | `using Point = (int X, int Y);` |
| Inline arrays | `[InlineArray(10)] struct Buffer { int _element; }` |

## .NET Framework vs .NET Core vs .NET 5+

| Aspect | .NET Framework | .NET Core (1.0–3.1) | .NET 5+ (Modern .NET) |
|--------|---------------|---------------------|----------------------|
| **Platform** | Windows only | Cross-platform | Cross-platform |
| **Open source** | Partially | Fully | Fully |
| **Release model** | Infrequent | Regular | Annual (Nov) |
| **Support** | Maintenance mode | 3.1 EOL Dec 2022 | LTS (even) / STS (odd) |
| **Performance** | Baseline | Faster | Fastest |
| **Deployment** | GAC, machine-wide | Self-contained / FDD | Self-contained / AOT |
| **App models** | WinForms, WPF, ASP.NET, WCF | ASP.NET Core, Console | All modern workloads |
| **Container support** | Limited | Good | Excellent |
| **Minimum APIs** | No | No | Yes (.NET 6+) |
| **Native AOT** | No | No | Yes (.NET 7+ / full in .NET 8) |
| **Latest LTS** | N/A | N/A | .NET 10 (Nov 2025) |
| **Status** | ⚠️ Legacy | ⚠️ EOL | ✅ Active development |

### Migration Decision Tree

```
┌──────────────────────────────────────┐
│ Are you starting a new project?      │
└──────────┬───────────────────────────┘
           │
     ┌─────▼─────┐
     │    Yes     │──────────────► Use .NET 10 (latest LTS)
     └───────────┘
           │
     ┌─────▼──────┐
     │     No      │
     └─────┬──────┘
           │
┌──────────▼───────────────────────────┐
│ Are you on .NET Framework?           │
└──────────┬───────────────────────────┘
           │
     ┌─────▼─────┐
     │    Yes     │──────────────► Plan migration to .NET 10
     └───────────┘                 See 01-BREAKING-CHANGES
           │
     ┌─────▼──────┐
     │     No      │
     │ (.NET Core) │──────────────► Upgrade to .NET 10 (latest LTS)
     └────────────┘                 See 01-BREAKING-CHANGES
```

## When to Use .NET

### Good Candidates

| Use Case | Why .NET |
|----------|----------|
| **Enterprise web APIs** | ASP.NET Core is fast, mature, well-supported |
| **Cloud-native microservices** | Excellent container support, low memory footprint |
| **Real-time applications** | SignalR provides built-in WebSocket support |
| **Background processing** | Worker services and hosted service patterns |
| **Cross-platform desktop** | MAUI for iOS, Android, macOS, Windows |
| **High-performance systems** | Span<T>, AOT, and low-level control |
| **gRPC services** | First-class gRPC support in ASP.NET Core |
| **Enterprise integration** | Extensive library ecosystem via NuGet |

### .NET Strengths

- **Performance** — Consistently ranks among the fastest web frameworks (TechEmpower benchmarks)
- **Type safety** — Strong static typing catches errors at compile time
- **Tooling** — Visual Studio, Rider, and VS Code provide excellent development experiences
- **Ecosystem** — Over 350,000 packages on NuGet
- **Long-term support** — LTS releases supported for 3 years by Microsoft

## When NOT to Use .NET

| Scenario | Why Not | Better Alternative |
|----------|---------|-------------------|
| Tiny serverless functions | Cold start overhead (without AOT) | Node.js, Python, Go |
| ML/AI research prototyping | Smaller ML ecosystem | Python (PyTorch, TensorFlow) |
| Systems programming | No manual memory control | Rust, C++ |
| Quick scripting/automation | Heavier toolchain | Python, Bash |
| Existing Java/Spring shop | Team retraining cost | Stay with Java ecosystem |
| Embedded/IoT (bare metal) | Requires managed runtime | C, Rust |

## .NET Project Types

| Template | CLI Command | Description |
|----------|-------------|-------------|
| Console App | `dotnet new console` | Command-line application |
| Web API | `dotnet new webapi` | RESTful HTTP API with controllers or minimal APIs |
| Blazor Server | `dotnet new blazorserver` | Server-rendered interactive web UI |
| Blazor WebAssembly | `dotnet new blazorwasm` | Client-side web UI running in browser |
| Worker Service | `dotnet new worker` | Long-running background service |
| gRPC Service | `dotnet new grpc` | High-performance RPC service |
| MAUI App | `dotnet new maui` | Cross-platform native UI (mobile + desktop) |
| Class Library | `dotnet new classlib` | Reusable .NET library |
| xUnit Test | `dotnet new xunit` | Unit test project using xUnit |
| NUnit Test | `dotnet new nunit` | Unit test project using NUnit |

### Project Structure Example

```
MyWebApi/
├── MyWebApi.sln                    # Solution file
├── src/
│   └── MyWebApi/
│       ├── MyWebApi.csproj         # Project file
│       ├── Program.cs              # Application entry point
│       ├── appsettings.json        # Configuration
│       ├── appsettings.Development.json
│       ├── Controllers/
│       │   └── WeatherController.cs
│       ├── Models/
│       │   └── WeatherForecast.cs
│       └── Services/
│           └── WeatherService.cs
└── tests/
    └── MyWebApi.Tests/
        ├── MyWebApi.Tests.csproj
        └── WeatherControllerTests.cs
```

## Prerequisites

### Required Knowledge

- **Programming fundamentals** — Variables, control flow, OOP concepts
- **Command line basics** — Terminal navigation, running commands
- **Version control** — Git fundamentals (clone, commit, push, pull)

### Recommended Knowledge

- **HTTP fundamentals** — Status codes, methods (GET, POST, PUT, DELETE), headers
- **JSON** — Data serialization format used extensively in .NET APIs
- **SQL basics** — Helpful for Entity Framework Core and data access
- **Docker basics** — Container packaging and deployment

### Recommended Tools

| Tool | Purpose |
|------|---------|
| .NET SDK (latest LTS) | Runtime, compiler, CLI tools |
| Visual Studio 2022 | Full-featured IDE (Windows/Mac) |
| Visual Studio Code | Lightweight editor with C# Dev Kit extension |
| JetBrains Rider | Cross-platform .NET IDE |
| Docker Desktop | Container development and testing |
| SQL Server / PostgreSQL | Database for EF Core development |
| Postman / Bruno | API testing and exploration |
| Git | Version control |

## Next Steps

Continue to [Breaking Changes from .NET Core 3.1](01-BREAKING-CHANGES-DOTNET-CORE-31.md) to understand the migration path from .NET Core 3.1 through .NET 10, including removed APIs, behavioral changes, and code migration patterns.

### Suggested Learning Path

1. **[Breaking Changes](01-BREAKING-CHANGES-DOTNET-CORE-31.md)** — Migration from .NET Core 3.1 through .NET 10
2. **[Services](02-SERVICES.md)** — Web APIs, worker services, gRPC, configuration
3. **[Best Practices](03-BEST-PRACTICES.md)** — Async/await, memory, security, testing
4. **[Style Guide](04-STYLE-GUIDE.md)** — Naming conventions, formatting, analyzers
5. **[Concurrency & Parallelism](05-CONCURRENCY-ASYNC-PARALLELISM.md)** — Async deep dive, TPL, channels, job queuing
6. **[Dependency Injection](06-DEPENDENCY-INJECTION.md)** — DI patterns, lifetimes, keyed services, testing
7. **[Learning Path](LEARNING-PATH.md)** — Structured 12-week curriculum

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial .NET and C# overview documentation |
