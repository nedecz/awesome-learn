# .NET and C# Learning Path

A structured 12-week curriculum for learning .NET and C# — from language fundamentals to production-ready application development.

## How to Use This Learning Path

```
Phase 1 (Weeks 1–3)        Phase 2 (Weeks 4–6)
C# Fundamentals             ASP.NET Core & Data Access
├── Language basics          ├── Web APIs
├── OOP in C#               ├── Entity Framework Core
├── Collections & LINQ       └── Middleware & routing
└── Async/await basics

Phase 3 (Weeks 7–9)        Phase 4 (Weeks 10–12)
Advanced Patterns            Production Readiness
├── DI deep dive             ├── Performance tuning
├── Testing strategies       ├── Security hardening
├── Configuration            ├── Deployment & containers
└── Background services      └── Observability
```

Each phase includes:
- **Learning objectives** — what you will be able to do
- **Readings** — references to documents in this repository and official docs
- **Hands-on exercises** — practical coding tasks
- **Knowledge check** — questions to validate understanding

---

## Phase 1: C# Fundamentals and .NET Basics (Weeks 1–3)

### Week 1: C# Language Fundamentals

**Learning Objectives:**
- Understand C# type system (value types, reference types, nullable)
- Write programs using control flow, methods, and error handling
- Use records, tuples, and pattern matching

**Readings:**
- [00-OVERVIEW.md](00-OVERVIEW.md) — .NET ecosystem and C# language evolution
- [Microsoft: C# Fundamentals](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/)

**Hands-On Exercises:**

1. **Hello .NET** — Create a console app with `dotnet new console`, explore the project structure
2. **Type Explorer** — Write a program that demonstrates value types, reference types, and nullable types
3. **Pattern Matcher** — Implement a calculator using switch expressions and pattern matching

```csharp
// Exercise 3: Build this calculator
public static double Calculate(double a, double b, string op) => op switch
{
    "+" => a + b,
    "-" => a - b,
    "*" => a * b,
    "/" when b != 0 => a / b,
    "/" => throw new DivideByZeroException(),
    _ => throw new ArgumentException($"Unknown operator: {op}")
};
```

**Knowledge Check:**
- [ ] What is the difference between a `struct` and a `class` in C#?
- [ ] When should you use a `record` instead of a `class`?
- [ ] What does `string?` mean when nullable reference types are enabled?
- [ ] How do switch expressions differ from switch statements?

### Week 2: Collections, LINQ, and Generics

**Learning Objectives:**
- Use generic collections (`List<T>`, `Dictionary<TKey, TValue>`, `HashSet<T>`)
- Write LINQ queries using both query and method syntax
- Understand deferred execution and materialization

**Readings:**
- [Microsoft: LINQ](https://learn.microsoft.com/en-us/dotnet/csharp/linq/)
- [03-BEST-PRACTICES.md](03-BEST-PRACTICES.md) — LINQ Best Practices section

**Hands-On Exercises:**

1. **Collection Kata** — Implement a phonebook using `Dictionary<string, List<string>>`
2. **LINQ Queries** — Given a list of products, write queries to filter, group, sort, and aggregate
3. **Custom Collection** — Implement a generic `PagedList<T>` class with LINQ support

```csharp
// Exercise 2: Sample queries to implement
var products = GetProducts();

// Find all products over $50, sorted by price descending
var expensive = products
    .Where(p => p.Price > 50)
    .OrderByDescending(p => p.Price)
    .ToList();

// Group products by category, showing count and average price
var summary = products
    .GroupBy(p => p.Category)
    .Select(g => new
    {
        Category = g.Key,
        Count = g.Count(),
        AvgPrice = g.Average(p => p.Price)
    });
```

**Knowledge Check:**
- [ ] What is deferred execution in LINQ and why does it matter?
- [ ] When should you use `Any()` instead of `Count() > 0`?
- [ ] What is the difference between `IEnumerable<T>` and `IQueryable<T>`?
- [ ] How does `AsNoTracking()` improve EF Core query performance?

### Week 3: Asynchronous Programming

**Learning Objectives:**
- Understand `async`/`await` and `Task`-based patterns
- Use `CancellationToken` for cooperative cancellation
- Avoid common async pitfalls (sync-over-async, async void)

**Readings:**
- [03-BEST-PRACTICES.md](03-BEST-PRACTICES.md) — Async/Await Best Practices section
- [Microsoft: Async programming](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/)

**Hands-On Exercises:**

1. **Async File Processor** — Read, transform, and write files using async streams
2. **Parallel Fetcher** — Fetch data from multiple URLs concurrently with `Task.WhenAll`
3. **Cancellation Demo** — Implement a long-running operation that responds to cancellation

```csharp
// Exercise 2: Parallel HTTP fetcher
public async Task<Dictionary<string, string>> FetchAllAsync(
    IEnumerable<string> urls, CancellationToken ct = default)
{
    var tasks = urls.Select(async url =>
    {
        var content = await _httpClient.GetStringAsync(url, ct);
        return (url, content);
    });

    var results = await Task.WhenAll(tasks);
    return results.ToDictionary(r => r.url, r => r.content);
}
```

**Knowledge Check:**
- [ ] Why should you avoid `async void` methods?
- [ ] What happens if you call `.Result` on a `Task` in ASP.NET Core?
- [ ] When is `ValueTask<T>` preferred over `Task<T>`?
- [ ] What does `ConfigureAwait(false)` do and when should you use it?

---

## Phase 2: ASP.NET Core and Data Access (Weeks 4–6)

### Week 4: ASP.NET Core Web APIs

**Learning Objectives:**
- Build RESTful APIs using both minimal APIs and controllers
- Configure middleware pipeline and routing
- Implement request validation and error handling

**Readings:**
- [02-SERVICES.md](02-SERVICES.md) — ASP.NET Core Web API Services section
- [03-BEST-PRACTICES.md](03-BEST-PRACTICES.md) — API Design Guidelines section

**Hands-On Exercises:**

1. **Minimal API** — Build a TODO API using minimal APIs with CRUD operations
2. **Controller API** — Rebuild the same TODO API using controllers
3. **Middleware** — Write custom middleware for request logging and correlation IDs

**Knowledge Check:**
- [ ] What are the trade-offs between minimal APIs and controllers?
- [ ] In what order does middleware execute in the pipeline?
- [ ] How do you return a 201 Created response with a Location header?
- [ ] What is the difference between `[FromBody]` and `[FromQuery]`?

### Week 5: Entity Framework Core

**Learning Objectives:**
- Configure DbContext and entity mappings
- Write migrations and seed data
- Optimize queries with projections, includes, and tracking behavior

**Readings:**
- [Microsoft: EF Core](https://learn.microsoft.com/en-us/ef/core/)

**Hands-On Exercises:**

1. **Code First** — Model a blog system (Posts, Comments, Tags) with EF Core
2. **Migrations** — Create and apply migrations, seed initial data
3. **Query Optimization** — Identify and fix N+1 queries using EF Core logging

```csharp
// Exercise 3: Fix N+1 query
// ❌ N+1 problem
var posts = await _dbContext.Posts.ToListAsync();
foreach (var post in posts)
{
    var comments = await _dbContext.Comments
        .Where(c => c.PostId == post.Id).ToListAsync();
}

// ✅ Eager loading with Include
var posts = await _dbContext.Posts
    .Include(p => p.Comments)
    .ToListAsync();

// ✅ Projection (even better — only fetch what you need)
var summaries = await _dbContext.Posts
    .Select(p => new PostSummary(p.Id, p.Title, p.Comments.Count))
    .ToListAsync();
```

**Knowledge Check:**
- [ ] What is the difference between `Include` and `Select` projection?
- [ ] When should you use `AsNoTracking()`?
- [ ] How do you handle concurrent updates with EF Core?
- [ ] What is a migration and when should you create one?

### Week 6: Routing, Filters, and OpenAPI

**Learning Objectives:**
- Configure advanced routing with constraints and conventions
- Use action filters, result filters, and exception filters
- Generate OpenAPI documentation with Swagger/Swashbuckle

**Hands-On Exercises:**

1. **Versioned API** — Implement URL-based API versioning
2. **Filters** — Create an action filter for request validation and a result filter for response wrapping
3. **OpenAPI** — Configure Swagger UI with authentication, examples, and XML comments

**Knowledge Check:**
- [ ] What is the difference between route constraints and model binding?
- [ ] In what order do filters execute in ASP.NET Core?
- [ ] How do you add authentication support to Swagger UI?
- [ ] What is the purpose of `[ProducesResponseType]` attributes?

---

## Phase 3: Advanced Patterns (Weeks 7–9)

### Week 7: Dependency Injection Deep Dive

**Learning Objectives:**
- Understand service lifetimes and their implications
- Implement advanced DI patterns (decorator, composite, keyed services)
- Avoid common DI pitfalls (captive dependencies, service locator)

**Readings:**
- [02-SERVICES.md](02-SERVICES.md) — Dependency Injection section
- [03-BEST-PRACTICES.md](03-BEST-PRACTICES.md) — Production Readiness Checklist

**Hands-On Exercises:**

1. **Lifetime Explorer** — Build an app that demonstrates Singleton, Scoped, and Transient lifetimes
2. **Decorator Pattern** — Implement a caching decorator for a repository using DI
3. **Keyed Services** — Use .NET 8 keyed services for strategy pattern implementation

**Knowledge Check:**
- [ ] What is a captive dependency and how do you avoid it?
- [ ] When should you use `IServiceScopeFactory`?
- [ ] How do keyed services differ from named options?
- [ ] What happens if a Transient service implements `IDisposable`?

### Week 8: Testing Strategies

**Learning Objectives:**
- Write unit tests with xUnit, Moq, and FluentAssertions
- Write integration tests with WebApplicationFactory
- Implement test fixtures and shared context

**Readings:**
- [03-BEST-PRACTICES.md](03-BEST-PRACTICES.md) — Testing Best Practices section
- [Microsoft: Testing in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/test/)

**Hands-On Exercises:**

1. **Unit Tests** — Test a service class with mocked dependencies
2. **Integration Tests** — Test API endpoints with `WebApplicationFactory` and in-memory database
3. **Test Patterns** — Implement builder pattern for test data and custom assertions

**Knowledge Check:**
- [ ] What is the difference between `Fact` and `Theory` in xUnit?
- [ ] How does `WebApplicationFactory` help with integration testing?
- [ ] When should you use `Mock.Verify` vs testing the return value?
- [ ] How do you share setup between test classes with `IClassFixture`?

### Week 9: Configuration and Background Services

**Learning Objectives:**
- Master the Options pattern (`IOptions`, `IOptionsSnapshot`, `IOptionsMonitor`)
- Build background services with `BackgroundService` and `IHostedLifecycleService`
- Implement structured logging with correlation IDs

**Readings:**
- [02-SERVICES.md](02-SERVICES.md) — Configuration, Logging, and Worker Services sections

**Hands-On Exercises:**

1. **Options Pattern** — Configure a service with validated options and hot-reload support
2. **Background Worker** — Build a message queue consumer as a BackgroundService
3. **Structured Logging** — Add Serilog with structured logging and a correlation ID middleware

**Knowledge Check:**
- [ ] When should you use `IOptionsMonitor` vs `IOptionsSnapshot`?
- [ ] How do you validate options at startup with `ValidateOnStart`?
- [ ] What is the lifecycle of `IHostedLifecycleService`?
- [ ] Why is `IServiceScopeFactory` required in background services?

---

## Phase 4: Production Readiness (Weeks 10–12)

### Week 10: Performance and Memory

**Learning Objectives:**
- Profile memory usage and identify allocations
- Use `Span<T>`, `ArrayPool`, and object pooling
- Benchmark with BenchmarkDotNet

**Readings:**
- [03-BEST-PRACTICES.md](03-BEST-PRACTICES.md) — Memory Management and Performance sections

**Hands-On Exercises:**

1. **Benchmarking** — Compare string concatenation methods with BenchmarkDotNet
2. **Memory Optimization** — Refactor a CSV parser to use `Span<T>` for zero-allocation parsing
3. **Object Pooling** — Implement an object pool for expensive-to-create resources

**Knowledge Check:**
- [ ] When should you use `Span<T>` vs `Memory<T>`?
- [ ] What is the purpose of `ArrayPool<T>.Shared`?
- [ ] How does BenchmarkDotNet help avoid micro-benchmark pitfalls?
- [ ] What are the common causes of memory leaks in .NET applications?

### Week 11: Security and Hardening

**Learning Objectives:**
- Implement authentication with ASP.NET Core Identity and JWT
- Configure authorization policies and role-based access control
- Apply security headers and input validation

**Readings:**
- [03-BEST-PRACTICES.md](03-BEST-PRACTICES.md) — Security Best Practices section
- [Microsoft: ASP.NET Core Security](https://learn.microsoft.com/en-us/aspnet/core/security/)

**Hands-On Exercises:**

1. **JWT Authentication** — Implement token-based auth with refresh tokens
2. **Authorization Policies** — Build custom authorization requirements and handlers
3. **Security Audit** — Review an intentionally vulnerable app and fix OWASP Top 10 issues

**Knowledge Check:**
- [ ] What is the difference between authentication and authorization?
- [ ] How do you store secrets securely in .NET (not in appsettings.json)?
- [ ] What does the Data Protection API protect against?
- [ ] How do you prevent CSRF attacks in ASP.NET Core?

### Week 12: Deployment, Containers, and Observability

**Learning Objectives:**
- Containerize .NET apps with multi-stage Docker builds
- Configure health checks for Kubernetes liveness and readiness probes
- Set up distributed tracing with OpenTelemetry

**Readings:**
- [02-SERVICES.md](02-SERVICES.md) — Health Checks section
- [01-BREAKING-CHANGES-DOTNET-CORE-31.md](01-BREAKING-CHANGES-DOTNET-CORE-31.md) — Migration Checklist

**Hands-On Exercises:**

1. **Dockerfile** — Write a multi-stage Dockerfile for an ASP.NET Core app
2. **Health Checks** — Implement liveness and readiness probes with dependency checks
3. **OpenTelemetry** — Add distributed tracing and metrics to a multi-service app

```dockerfile
# Exercise 1: Multi-stage Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["MyApp.csproj", "."]
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .
USER $APP_UID
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

**Knowledge Check:**
- [ ] What is the difference between a liveness probe and a readiness probe?
- [ ] Why should you use multi-stage Docker builds for .NET apps?
- [ ] What telemetry data does OpenTelemetry collect (traces, metrics, logs)?
- [ ] How do you configure environment-specific settings in containers?

---

## Recommended Resources

### Official Documentation

| Resource | URL |
|----------|-----|
| C# Language Reference | https://learn.microsoft.com/en-us/dotnet/csharp/ |
| ASP.NET Core Documentation | https://learn.microsoft.com/en-us/aspnet/core/ |
| Entity Framework Core | https://learn.microsoft.com/en-us/ef/core/ |
| .NET API Browser | https://learn.microsoft.com/en-us/dotnet/api/ |

### Books

| Book | Author | Focus |
|------|--------|-------|
| *C# in Depth* | Jon Skeet | Deep language understanding |
| *Concurrency in C# Cookbook* | Stephen Cleary | Async/parallel patterns |
| *Dependency Injection Principles, Practices, Patterns* | Steven van Deursen & Mark Seemann | DI patterns |
| *ASP.NET Core in Action* | Andrew Lock | Web application development |

### Community

| Resource | Description |
|----------|-------------|
| [dotnet/runtime](https://github.com/dotnet/runtime) | .NET runtime source code |
| [dotnet/aspnetcore](https://github.com/dotnet/aspnetcore) | ASP.NET Core source code |
| [Stack Overflow — C#](https://stackoverflow.com/questions/tagged/c%23) | Q&A for C# developers |
| [r/dotnet](https://www.reddit.com/r/dotnet/) | .NET community discussions |

---

## Completion Checklist

After completing all four phases, you should be able to:

- [ ] Build RESTful APIs with ASP.NET Core (minimal APIs and controllers)
- [ ] Use Entity Framework Core for data access with optimized queries
- [ ] Implement dependency injection with proper lifetime management
- [ ] Write comprehensive tests (unit, integration, architecture)
- [ ] Apply async/await patterns correctly throughout the call stack
- [ ] Manage configuration with the Options pattern
- [ ] Build background services with graceful shutdown
- [ ] Optimize performance with Span<T>, pooling, and benchmarking
- [ ] Secure applications with authentication, authorization, and OWASP mitigations
- [ ] Deploy containerized .NET apps with health checks and observability
