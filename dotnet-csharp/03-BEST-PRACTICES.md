# .NET and C# Best Practices

## Table of Contents

1. [Overview](#overview)
2. [Production Readiness Checklist](#production-readiness-checklist)
3. [Async/Await Best Practices](#asyncawait-best-practices)
4. [Memory Management](#memory-management)
5. [Exception Handling](#exception-handling)
6. [Null Safety](#null-safety)
7. [LINQ Best Practices](#linq-best-practices)
8. [Performance Best Practices](#performance-best-practices)
9. [Security Best Practices](#security-best-practices)
10. [API Design Guidelines](#api-design-guidelines)
11. [Testing Best Practices](#testing-best-practices)
12. [Next Steps](#next-steps)

## Overview

This document covers essential best practices for building production-grade .NET applications — from async programming and memory management to security, API design, and testing.

### Target Audience

- .NET developers building production applications
- Tech leads establishing team coding standards
- Architects reviewing .NET codebases for production readiness

### Scope

- Async/await patterns and common pitfalls
- Memory management with Span<T>, IDisposable, and pooling
- Exception handling, null safety, and LINQ usage
- Performance optimization and benchmarking
- Security hardening and OWASP considerations
- RESTful API design and testing strategies

## Production Readiness Checklist

### Application

- [ ] Structured logging with correlation IDs
- [ ] Health check endpoints (`/health/live`, `/health/ready`)
- [ ] Graceful shutdown handling (hosted service lifecycle)
- [ ] Request validation and error handling middleware
- [ ] Rate limiting configured for public endpoints
- [ ] Response compression enabled
- [ ] HTTPS enforced with HSTS headers

### Performance

- [ ] Connection pooling for databases and HTTP clients
- [ ] Output caching or response caching for read-heavy endpoints
- [ ] Async/await used throughout the call chain (no sync-over-async)
- [ ] Object pooling for frequently allocated objects
- [ ] Memory profiling completed (no significant leaks)
- [ ] Load testing performed under expected production traffic

### Security

- [ ] Authentication and authorization configured
- [ ] Input validation on all public endpoints
- [ ] CORS policy configured (not wildcard in production)
- [ ] Secrets stored in vault (not in appsettings.json)
- [ ] Data protection API configured for sensitive data
- [ ] Security headers set (CSP, X-Content-Type-Options, etc.)
- [ ] Dependency vulnerability scanning enabled

### Observability

- [ ] Distributed tracing configured (OpenTelemetry)
- [ ] Metrics collection for key business operations
- [ ] Alerting configured for error rates and latency
- [ ] Request/response logging (with PII redaction)
- [ ] Dashboard for service health and performance

### Deployment

- [ ] Container image optimized (multi-stage build, non-root user)
- [ ] CI/CD pipeline with automated tests
- [ ] Database migrations automated and reversible
- [ ] Feature flags for risky deployments
- [ ] Rollback strategy documented and tested
- [ ] Environment-specific configuration validated

## Async/Await Best Practices

### Do's and Don'ts

✅ **Do: Use async all the way**

```csharp
// ✅ Async throughout the call chain
public async Task<Order> GetOrderAsync(int id, CancellationToken ct)
{
    var order = await _repository.GetByIdAsync(id, ct);
    var enriched = await _enrichmentService.EnrichAsync(order, ct);
    return enriched;
}
```

❌ **Don't: Block on async code (sync-over-async)**

```csharp
// ❌ Blocks the thread — risks deadlocks
public Order GetOrder(int id)
{
    return _repository.GetByIdAsync(id).Result; // Deadlock risk!
}

// ❌ Also blocks
public Order GetOrder(int id)
{
    return _repository.GetByIdAsync(id).GetAwaiter().GetResult();
}
```

✅ **Do: Accept CancellationToken parameters**

```csharp
// ✅ Propagate cancellation tokens
public async Task<IReadOnlyList<Product>> SearchAsync(
    string query, CancellationToken cancellationToken = default)
{
    return await _dbContext.Products
        .Where(p => p.Name.Contains(query))
        .ToListAsync(cancellationToken);
}
```

❌ **Don't: Use async void (except for event handlers)**

```csharp
// ❌ Exceptions cannot be caught by the caller
public async void ProcessOrder(Order order)
{
    await _service.ProcessAsync(order); // Unobserved exception!
}

// ✅ Use async Task instead
public async Task ProcessOrderAsync(Order order)
{
    await _service.ProcessAsync(order);
}
```

✅ **Do: Use ValueTask for hot paths that often complete synchronously**

```csharp
// ✅ ValueTask avoids Task allocation when result is cached
public ValueTask<Product?> GetCachedProductAsync(int id)
{
    if (_cache.TryGetValue(id, out Product? product))
    {
        return ValueTask.FromResult(product); // No allocation
    }

    return new ValueTask<Product?>(LoadFromDatabaseAsync(id));
}
```

✅ **Do: Use ConfigureAwait(false) in library code**

```csharp
// ✅ In library code — avoids capturing synchronization context
public async Task<string> FetchDataAsync(string url)
{
    var response = await _httpClient.GetAsync(url)
        .ConfigureAwait(false);
    return await response.Content.ReadAsStringAsync()
        .ConfigureAwait(false);
}
```

### Async Patterns Summary

| Pattern | When to Use | Avoid When |
|---------|-------------|------------|
| `Task` | Return type for async methods without a result | N/A |
| `Task<T>` | Return type for async methods with a result | N/A |
| `ValueTask<T>` | Hot paths that often return synchronously | Method awaited multiple times |
| `IAsyncEnumerable<T>` | Streaming async data (e.g., database cursors) | Small, bounded datasets |
| `ConfigureAwait(false)` | Library code, not dependent on sync context | ASP.NET Core (no sync context) |
| `CancellationToken` | Always accept in public async APIs | N/A |

## Memory Management

### IDisposable and IAsyncDisposable

```csharp
// ✅ IAsyncDisposable for async cleanup
public class DatabaseConnection : IAsyncDisposable
{
    private readonly SqlConnection _connection;

    public DatabaseConnection(string connectionString)
    {
        _connection = new SqlConnection(connectionString);
    }

    public async ValueTask DisposeAsync()
    {
        await _connection.DisposeAsync();
        GC.SuppressFinalize(this);
    }
}

// ✅ Using declaration (C# 8+)
await using var connection = new DatabaseConnection(connectionString);
await connection.ExecuteAsync(query);
// Disposed automatically at end of scope
```

### Span&lt;T&gt; and Memory&lt;T&gt;

```csharp
// ✅ Span<T> for stack-based, zero-allocation slicing
public static int CountOccurrences(ReadOnlySpan<char> text, char target)
{
    int count = 0;
    foreach (var c in text)
    {
        if (c == target) count++;
    }
    return count;
}

// ✅ Parse without allocating substrings
public static (int Year, int Month, int Day) ParseDate(
    ReadOnlySpan<char> dateString)
{
    // "2025-01-15"
    var year = int.Parse(dateString[..4]);
    var month = int.Parse(dateString[5..7]);
    var day = int.Parse(dateString[8..10]);
    return (year, month, day);
}

// ✅ Memory<T> when you need to store a reference (heap-safe)
public async Task ProcessDataAsync(Memory<byte> buffer)
{
    // Memory<T> can be used in async methods (Span<T> cannot)
    await _stream.ReadAsync(buffer);
}
```

### ArrayPool and Object Pooling

```csharp
// ✅ ArrayPool — rent and return arrays to avoid GC pressure
public static string ProcessLargeData(Stream stream)
{
    byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);
    try
    {
        int bytesRead = stream.Read(buffer, 0, buffer.Length);
        return Encoding.UTF8.GetString(buffer, 0, bytesRead);
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer, clearArray: true);
    }
}

// ✅ ObjectPool for expensive-to-create objects
builder.Services.AddSingleton<ObjectPool<StringBuilder>>(
    new DefaultObjectPoolProvider().CreateStringBuilderPool());

public class ReportGenerator
{
    private readonly ObjectPool<StringBuilder> _pool;

    public ReportGenerator(ObjectPool<StringBuilder> pool)
    {
        _pool = pool;
    }

    public string GenerateReport(IEnumerable<ReportLine> lines)
    {
        var sb = _pool.Get();
        try
        {
            foreach (var line in lines)
            {
                sb.AppendLine($"{line.Label}: {line.Value}");
            }
            return sb.ToString();
        }
        finally
        {
            _pool.Return(sb);
        }
    }
}
```

## Exception Handling

### Patterns

✅ **Do: Use specific exception types**

```csharp
// ✅ Catch specific exceptions
try
{
    var data = await _httpClient.GetFromJsonAsync<Data>(url, ct);
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    _logger.LogWarning("Resource not found at {Url}", url);
    return null;
}
catch (TaskCanceledException) when (ct.IsCancellationRequested)
{
    _logger.LogInformation("Request cancelled");
    throw;
}
catch (TaskCanceledException ex)
{
    _logger.LogError(ex, "Request to {Url} timed out", url);
    throw new TimeoutException($"Request to {url} timed out", ex);
}
```

❌ **Don't: Catch and swallow exceptions**

```csharp
// ❌ Swallows exceptions — hides bugs
try
{
    await ProcessAsync();
}
catch (Exception)
{
    // Silently swallowed
}

// ❌ Resets stack trace
catch (Exception ex)
{
    throw ex; // Stack trace lost!
}

// ✅ Preserves stack trace
catch (Exception ex)
{
    _logger.LogError(ex, "Processing failed");
    throw; // Re-throw preserves stack trace
}
```

### Result Pattern (Alternative to Exceptions)

```csharp
public record Result<T>
{
    public T? Value { get; }
    public string? Error { get; }
    public bool IsSuccess => Error is null;

    private Result(T value) => Value = value;
    private Result(string error) => Error = error;

    public static Result<T> Success(T value) => new(value);
    public static Result<T> Failure(string error) => new(error);
}

// Usage
public async Task<Result<Order>> CreateOrderAsync(CreateOrderRequest request)
{
    if (request.Items.Count == 0)
        return Result<Order>.Failure("Order must have at least one item");

    var order = new Order(request);
    await _repository.AddAsync(order);
    return Result<Order>.Success(order);
}
```

## Null Safety

### Nullable Reference Types

```csharp
// Enable in .csproj
// <Nullable>enable</Nullable>

// ✅ Explicit nullability
public class UserService
{
    // Non-nullable — must always have a value
    public string GetDisplayName(User user)
    {
        return user.DisplayName; // Guaranteed non-null by type system
    }

    // Nullable — may return null
    public User? FindByEmail(string email)
    {
        return _users.FirstOrDefault(u => u.Email == email);
    }
}
```

### Null Handling Patterns

```csharp
// ✅ Null-conditional operator
var length = customer?.Address?.City?.Length;

// ✅ Null-coalescing operator
var name = customer?.Name ?? "Unknown";

// ✅ Null-coalescing assignment
customer.Name ??= "Default";

// ✅ Pattern matching for null checks
if (order is { Status: OrderStatus.Completed, Total: > 0 })
{
    ProcessCompletedOrder(order);
}

// ✅ Guard clause with throw expression
public void Process(Order order)
{
    ArgumentNullException.ThrowIfNull(order);
    ArgumentException.ThrowIfNullOrWhiteSpace(order.CustomerId);
    // Process...
}

// ✅ Null-forgiving operator (use sparingly, only when you know better)
var user = _cache.Get(key)!; // Suppress warning — known non-null
```

## LINQ Best Practices

✅ **Do: Use method syntax for complex queries**

```csharp
// ✅ Readable method chain
var activeUsers = users
    .Where(u => u.IsActive)
    .OrderBy(u => u.LastName)
    .ThenBy(u => u.FirstName)
    .Select(u => new UserDto(u.Id, u.FullName, u.Email))
    .ToList();
```

❌ **Don't: Enumerate multiple times**

```csharp
// ❌ Enumerates the query twice
var users = GetUsers(); // Returns IEnumerable<User>
var count = users.Count();     // First enumeration
var first = users.First();     // Second enumeration

// ✅ Materialize once
var userList = GetUsers().ToList();
var count = userList.Count;
var first = userList[0];
```

✅ **Do: Use Any() instead of Count() for existence checks**

```csharp
// ❌ Counts all items just to check if any exist
if (orders.Count() > 0) { }

// ✅ Short-circuits on first match
if (orders.Any()) { }

// ✅ With predicate
if (orders.Any(o => o.Status == OrderStatus.Pending)) { }
```

✅ **Do: Be aware of deferred execution**

```csharp
// ⚠️ Query is not executed here — it's deferred
var query = _dbContext.Products.Where(p => p.Price > 100);

// Query executes here (ToListAsync materializes the results)
var products = await query.ToListAsync(ct);

// ✅ Use AsNoTracking for read-only queries (EF Core)
var products = await _dbContext.Products
    .AsNoTracking()
    .Where(p => p.IsActive)
    .ToListAsync(ct);
```

## Performance Best Practices

### Benchmarking with BenchmarkDotNet

```csharp
[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net80)]
public class StringBenchmarks
{
    private readonly string[] _items = Enumerable
        .Range(0, 1000)
        .Select(i => $"item-{i}")
        .ToArray();

    [Benchmark(Baseline = true)]
    public string StringConcat()
    {
        var result = "";
        foreach (var item in _items)
            result += item + ",";
        return result;
    }

    [Benchmark]
    public string StringBuilder()
    {
        var sb = new StringBuilder();
        foreach (var item in _items)
            sb.Append(item).Append(',');
        return sb.ToString();
    }

    [Benchmark]
    public string StringJoin()
    {
        return string.Join(",", _items);
    }
}

// Run: dotnet run -c Release
```

### Performance Do's and Don'ts

| Area | ✅ Do | ❌ Don't |
|------|-------|---------|
| Strings | Use `StringBuilder` for concatenation in loops | Concatenate with `+` in loops |
| Collections | Pre-size collections when count is known | Use `List<T>` without capacity hint |
| LINQ | Use `Any()` for existence checks | Use `Count() > 0` |
| Async | Use `ValueTask` for hot paths | Allocate `Task` when result is cached |
| Serialization | Use `System.Text.Json` source generators | Use reflection-based serialization in hot paths |
| Caching | Cache expensive computations | Recompute on every request |
| Allocation | Use `Span<T>` / `stackalloc` for small buffers | Allocate arrays for temporary data |
| HTTP | Use `IHttpClientFactory` | Create `new HttpClient()` per request |

### Reducing Allocations

```csharp
// ❌ Allocates a new string for each substring
string input = "Hello, World!";
string sub = input.Substring(0, 5);

// ✅ Zero-allocation slice with Span
ReadOnlySpan<char> span = input.AsSpan(0, 5);

// ❌ Allocates array for small temporary data
byte[] buffer = new byte[128];

// ✅ Stack allocation for small buffers
Span<byte> buffer = stackalloc byte[128];
```

## Security Best Practices

### Input Validation

```csharp
// ✅ Use Data Annotations for model validation
public class CreateUserRequest
{
    [Required]
    [StringLength(100, MinimumLength = 2)]
    public string Name { get; init; } = string.Empty;

    [Required]
    [EmailAddress]
    public string Email { get; init; } = string.Empty;

    [Required]
    [RegularExpression(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d).{8,}$")]
    public string Password { get; init; } = string.Empty;
}

// ✅ Use FluentValidation for complex rules
public class CreateOrderValidator : AbstractValidator<CreateOrderRequest>
{
    public CreateOrderValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Items).NotEmpty()
            .WithMessage("Order must have at least one item");
        RuleForEach(x => x.Items).ChildRules(item =>
        {
            item.RuleFor(i => i.Quantity).GreaterThan(0);
            item.RuleFor(i => i.Price).GreaterThan(0);
        });
    }
}
```

### OWASP Top 10 Mitigations

| Threat | Mitigation in .NET |
|--------|-------------------|
| **Injection** | Parameterized queries (EF Core), input validation |
| **Broken Authentication** | ASP.NET Core Identity, OAuth2/OIDC |
| **Sensitive Data Exposure** | Data Protection API, HTTPS enforcement |
| **XML External Entities** | `XmlReaderSettings.DtdProcessing = DtdProcessing.Prohibit` |
| **Broken Access Control** | `[Authorize]` attribute, policy-based authorization |
| **Security Misconfiguration** | Security headers middleware, CORS configuration |
| **XSS** | Razor auto-encoding, Content Security Policy |
| **Insecure Deserialization** | `System.Text.Json` (safe by default), `TypeNameHandling.None` |
| **Known Vulnerabilities** | `dotnet list package --vulnerable`, Dependabot |
| **Insufficient Logging** | Structured logging, audit trails, no PII in logs |

### Data Protection API

```csharp
// Registration
builder.Services.AddDataProtection()
    .PersistKeysToFileSystem(new DirectoryInfo("/keys"))
    .SetApplicationName("MyApp")
    .SetDefaultKeyLifetime(TimeSpan.FromDays(90));

// Usage
public class SensitiveDataService
{
    private readonly IDataProtector _protector;

    public SensitiveDataService(IDataProtectionProvider provider)
    {
        _protector = provider.CreateProtector("SensitiveData.v1");
    }

    public string Protect(string plainText) =>
        _protector.Protect(plainText);

    public string Unprotect(string protectedText) =>
        _protector.Unprotect(protectedText);
}
```

### Security Headers

```csharp
app.Use(async (context, next) =>
{
    context.Response.Headers.Append("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Append("X-Frame-Options", "DENY");
    context.Response.Headers.Append("X-XSS-Protection", "1; mode=block");
    context.Response.Headers.Append("Referrer-Policy",
        "strict-origin-when-cross-origin");
    context.Response.Headers.Append("Content-Security-Policy",
        "default-src 'self'");
    await next();
});
```

## API Design Guidelines

### RESTful Conventions

| HTTP Method | Route | Action | Status Code |
|-------------|-------|--------|-------------|
| `GET` | `/api/products` | List all products | 200 OK |
| `GET` | `/api/products/{id}` | Get single product | 200 OK / 404 Not Found |
| `POST` | `/api/products` | Create product | 201 Created |
| `PUT` | `/api/products/{id}` | Replace product | 200 OK / 404 Not Found |
| `PATCH` | `/api/products/{id}` | Partial update | 200 OK / 404 Not Found |
| `DELETE` | `/api/products/{id}` | Delete product | 204 No Content |

### API Versioning

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("X-Api-Version"));
});

// URL-based versioning
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("1.0")]
public class ProductsController : ControllerBase { }

[ApiVersion("2.0")]
public class ProductsV2Controller : ControllerBase { }
```

### Error Response Format

```csharp
// Standard Problem Details (RFC 7807)
app.UseExceptionHandler(appError =>
{
    appError.Run(async context =>
    {
        context.Response.ContentType = "application/problem+json";
        var exception = context.Features
            .Get<IExceptionHandlerFeature>()?.Error;

        var problem = new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Title = "An unexpected error occurred",
            Instance = context.Request.Path
        };

        context.Response.StatusCode = problem.Status.Value;
        await context.Response.WriteAsJsonAsync(problem);
    });
});

// Validation error response example:
// {
//   "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
//   "title": "One or more validation errors occurred.",
//   "status": 400,
//   "errors": {
//     "Name": ["The Name field is required."],
//     "Email": ["The Email field is not a valid e-mail address."]
//   }
// }
```

## Testing Best Practices

### Test Project Structure

```
tests/
├── MyApp.UnitTests/
│   ├── Services/
│   │   └── OrderServiceTests.cs
│   └── Validators/
│       └── CreateOrderValidatorTests.cs
├── MyApp.IntegrationTests/
│   ├── Api/
│   │   └── ProductsEndpointTests.cs
│   └── Fixtures/
│       └── WebApplicationFixture.cs
└── MyApp.ArchitectureTests/
    └── DependencyTests.cs
```

### Unit Testing with xUnit and Moq

```csharp
public class OrderServiceTests
{
    private readonly Mock<IOrderRepository> _repositoryMock;
    private readonly Mock<ILogger<OrderService>> _loggerMock;
    private readonly OrderService _sut;

    public OrderServiceTests()
    {
        _repositoryMock = new Mock<IOrderRepository>();
        _loggerMock = new Mock<ILogger<OrderService>>();
        _sut = new OrderService(_repositoryMock.Object, _loggerMock.Object);
    }

    [Fact]
    public async Task CreateOrderAsync_WithValidRequest_ReturnsOrder()
    {
        // Arrange
        var request = new CreateOrderRequest
        {
            CustomerId = "cust-123",
            Items = [new OrderItem("prod-1", 2, 29.99m)]
        };

        _repositoryMock
            .Setup(r => r.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync((Order o, CancellationToken _) => o);

        // Act
        var result = await _sut.CreateOrderAsync(request);

        // Assert
        result.Should().NotBeNull();
        result.CustomerId.Should().Be("cust-123");
        result.Items.Should().HaveCount(1);
        _repositoryMock.Verify(
            r => r.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()),
            Times.Once);
    }

    [Fact]
    public async Task CreateOrderAsync_WithEmptyItems_ThrowsValidationException()
    {
        // Arrange
        var request = new CreateOrderRequest
        {
            CustomerId = "cust-123",
            Items = []
        };

        // Act & Assert
        await _sut.Invoking(s => s.CreateOrderAsync(request))
            .Should().ThrowAsync<ValidationException>()
            .WithMessage("*at least one item*");
    }
}
```

### Integration Testing with WebApplicationFactory

```csharp
public class ProductsEndpointTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public ProductsEndpointTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Replace real database with in-memory
                services.RemoveAll<DbContextOptions<AppDbContext>>();
                services.AddDbContext<AppDbContext>(options =>
                    options.UseInMemoryDatabase("TestDb"));
            });
        }).CreateClient();
    }

    [Fact]
    public async Task GetProducts_ReturnsOkWithProducts()
    {
        // Act
        var response = await _client.GetAsync("/api/products");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var products = await response.Content
            .ReadFromJsonAsync<List<Product>>();
        products.Should().NotBeNull();
    }

    [Fact]
    public async Task CreateProduct_WithValidData_Returns201()
    {
        // Arrange
        var request = new { Name = "Widget", Price = 9.99 };

        // Act
        var response = await _client.PostAsJsonAsync("/api/products", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();
    }
}
```

### Testing Do's and Don'ts

| ✅ Do | ❌ Don't |
|-------|---------|
| Follow Arrange-Act-Assert pattern | Mix arrangement and assertions |
| Use descriptive test method names | Name tests `Test1`, `Test2` |
| Test one behavior per test | Assert multiple unrelated behaviors |
| Use FluentAssertions for readability | Use only `Assert.Equal` |
| Mock external dependencies | Call real databases in unit tests |
| Use `WebApplicationFactory` for integration tests | Start real server manually |
| Test edge cases and error paths | Only test the happy path |
| Use `CancellationToken` in async tests | Ignore cancellation in tests |

## Next Steps

Continue to [Style Guide](04-STYLE-GUIDE.md) to learn about C# naming conventions, code formatting, EditorConfig setup, and Roslyn analyzer recommendations.

### Related Documents

1. **[Services](02-SERVICES.md)** — DI, Web APIs, worker services, gRPC, configuration
2. **[Style Guide](04-STYLE-GUIDE.md)** — Naming conventions, formatting, analyzers
3. **[Learning Path](LEARNING-PATH.md)** — Structured 12-week curriculum

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial .NET and C# best practices documentation |
