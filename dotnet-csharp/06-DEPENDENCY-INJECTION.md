# Dependency Injection in .NET

## Table of Contents

1. [Overview](#overview)
2. [Service Lifetimes Deep Dive](#service-lifetimes-deep-dive)
3. [Registration Patterns](#registration-patterns)
4. [Constructor Injection](#constructor-injection)
5. [Advanced Patterns](#advanced-patterns)
6. [Keyed Services (.NET 8+)](#keyed-services-net-8)
7. [Scope Validation and Diagnostics](#scope-validation-and-diagnostics)
8. [Third-Party DI Containers](#third-party-di-containers)
9. [Testing with DI](#testing-with-di)
10. [Anti-Patterns](#anti-patterns)
11. [Best Practices](#best-practices)
12. [Next Steps](#next-steps)

## Overview

Dependency Injection (DI) is a first-class citizen in .NET. The built-in `Microsoft.Extensions.DependencyInjection` container supports constructor injection, multiple service lifetimes, keyed services, open generics, and factory-based registration — covering the vast majority of production scenarios without third-party libraries.

### Target Audience

- .NET developers building applications with ASP.NET Core, worker services, or console apps
- Architects designing modular, testable service layers
- Teams establishing DI conventions and avoiding common anti-patterns

### Scope

- Service lifetimes (Singleton, Scoped, Transient) and captive dependency pitfalls
- Registration patterns including keyed services, open generics, and decorators
- Constructor injection with primary constructors (C# 12+)
- Options pattern, factory pattern, and composite/strategy patterns
- Scope validation, diagnostics, and troubleshooting
- Testing strategies with DI
- Anti-patterns and production best practices

### Why Dependency Injection?

DI implements the **Inversion of Control (IoC)** principle — classes declare their dependencies rather than creating them. This enables loose coupling, testability, and flexible composition.

```
┌─────────────────────────────────────────────────────────────┐
│                  Without DI (Tight Coupling)                │
│                                                             │
│  class OrderService                                         │
│  {                                                          │
│      private readonly SqlOrderRepo _repo = new();  ← HARD  │
│      private readonly SmtpEmailSender _email = new(); CODED │
│  }                                                          │
├─────────────────────────────────────────────────────────────┤
│                  With DI (Loose Coupling)                    │
│                                                             │
│  class OrderService(IOrderRepo repo, IEmailSender email)    │
│  {                                                          │
│      // Dependencies injected — easy to test and swap  ✅   │
│  }                                                          │
└─────────────────────────────────────────────────────────────┘
```

## Service Lifetimes Deep Dive

### Lifetime Behavior

```
┌────────────────────────────────────────────────────────────────────┐
│                        Service Lifetimes                           │
│                                                                    │
│  SINGLETON (one instance for the entire application)               │
│  ┌────────────────────────────────────────────────────────┐        │
│  │                    Application                         │        │
│  │   Request 1 ──► instance A                             │        │
│  │   Request 2 ──► instance A  (same instance)            │        │
│  │   Request 3 ──► instance A  (same instance)            │        │
│  └────────────────────────────────────────────────────────┘        │
│                                                                    │
│  SCOPED (one instance per scope / HTTP request)                    │
│  ┌────────────────────────────────────────────────────────┐        │
│  │   Request 1 ──► instance A ─┐                          │        │
│  │                              ├─ same within request     │        │
│  │   Middleware  ──► instance A ─┘                         │        │
│  │                                                        │        │
│  │   Request 2 ──► instance B ─┐                          │        │
│  │                              ├─ different request       │        │
│  │   Middleware  ──► instance B ─┘                         │        │
│  └────────────────────────────────────────────────────────┘        │
│                                                                    │
│  TRANSIENT (new instance every time)                               │
│  ┌────────────────────────────────────────────────────────┐        │
│  │   Injection 1 ──► instance A                           │        │
│  │   Injection 2 ──► instance B                           │        │
│  │   Injection 3 ──► instance C                           │        │
│  │   (every GetService / constructor call = new instance)  │        │
│  └────────────────────────────────────────────────────────┘        │
└────────────────────────────────────────────────────────────────────┘
```

### Lifetime Comparison

| Lifetime | Instance Count | Disposed | Typical Use |
|----------|---------------|----------|-------------|
| **Singleton** | 1 per app | At app shutdown | Caches, HttpClient factories, configuration |
| **Scoped** | 1 per scope | At scope disposal | DbContext, unit of work, per-request state |
| **Transient** | 1 per injection | At scope disposal | Lightweight stateless services, validators |

### The Captive Dependency Anti-Pattern

A **captive dependency** occurs when a longer-lived service captures a shorter-lived dependency, effectively extending its lifetime.

```
 ❌  Singleton ──depends on──► Scoped service
     (lives forever)            (should die per request)
     Result: the scoped service becomes a de-facto singleton

 ✅  Singleton ──depends on──► Singleton
 ✅  Scoped    ──depends on──► Scoped or Transient
 ✅  Transient ──depends on──► any lifetime
```

**Safe dependency direction:**

| Consumer Lifetime | Can Depend On |
|-------------------|---------------|
| **Singleton** | Singleton only |
| **Scoped** | Singleton, Scoped |
| **Transient** | Singleton, Scoped, Transient |

```csharp
// ❌ WRONG — singleton captures scoped DbContext
builder.Services.AddSingleton<ICacheService, CacheService>();
// CacheService constructor takes AppDbContext (scoped) — CAPTIVE!

// ✅ CORRECT — use IServiceScopeFactory to create scopes manually
public class CacheService : ICacheService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public CacheService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public async Task RefreshAsync(CancellationToken ct)
    {
        await using var scope = _scopeFactory.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // Use db within this scope — properly scoped lifetime
    }
}
```

## Registration Patterns

### Basic Registration

```csharp
var builder = WebApplication.CreateBuilder(args);

// Interface → implementation
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();

// Self-registration
builder.Services.AddScoped<OrderService>();

// Factory delegate
builder.Services.AddScoped<IEmailSender>(sp =>
{
    var config = sp.GetRequiredService<IOptions<SmtpSettings>>().Value;
    return new SmtpEmailSender(config.Host, config.Port);
});

// Instance registration (singleton only)
builder.Services.AddSingleton(TimeProvider.System);
```

### TryAdd Variants

`TryAdd` only registers if no service of the same type exists — useful for library defaults that consumers can override.

```csharp
using Microsoft.Extensions.DependencyInjection.Extensions;

// Only registers if ICache is not already registered
builder.Services.TryAddSingleton<ICache, InMemoryCache>();

// Library provides defaults
public static class MyLibraryExtensions
{
    public static IServiceCollection AddMyLibrary(this IServiceCollection services)
    {
        services.TryAddScoped<ISerializer, JsonSerializer>();
        services.TryAddScoped<IValidator, DefaultValidator>();
        return services;
    }
}
```

### Open Generic Registration

Register generic interfaces once, resolved for any closed type.

```csharp
// Register open generic
builder.Services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>));

// Resolves automatically:
// IRepository<Order>   → EfRepository<Order>
// IRepository<Product> → EfRepository<Product>
// IRepository<User>    → EfRepository<User>
```

### Multiple Implementations (IEnumerable&lt;T&gt;)

```csharp
builder.Services.AddScoped<INotificationSender, EmailSender>();
builder.Services.AddScoped<INotificationSender, SmsSender>();
builder.Services.AddScoped<INotificationSender, PushSender>();

// Inject all implementations
public class NotificationService(IEnumerable<INotificationSender> senders)
{
    public async Task NotifyAllAsync(string message, CancellationToken ct)
    {
        foreach (var sender in senders)
        {
            await sender.SendAsync(message, ct);
        }
    }
}
```

### Decorator Pattern Registration

The built-in container does not natively support decorators, but you can use factory delegates.

```csharp
// Manual decorator registration
builder.Services.AddScoped<SqlOrderRepository>();
builder.Services.AddScoped<IOrderRepository>(sp =>
{
    var inner = sp.GetRequiredService<SqlOrderRepository>();
    var logger = sp.GetRequiredService<ILogger<LoggingOrderRepository>>();
    return new LoggingOrderRepository(inner, logger);
});

// Decorator implementation
public class LoggingOrderRepository : IOrderRepository
{
    private readonly IOrderRepository _inner;
    private readonly ILogger _logger;

    public LoggingOrderRepository(
        IOrderRepository inner, ILogger<LoggingOrderRepository> logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public async Task<Order?> GetByIdAsync(int id, CancellationToken ct)
    {
        _logger.LogInformation("Getting order {OrderId}", id);
        var result = await _inner.GetByIdAsync(id, ct);
        _logger.LogInformation("Order {OrderId} found: {Found}", id, result is not null);
        return result;
    }
}
```

## Constructor Injection

### Primary Constructors (C# 12+)

C# 12 primary constructors reduce boilerplate for DI. The parameters are available throughout the class body.

```csharp
// C# 12+ primary constructor — concise DI
public class OrderService(
    IOrderRepository repository,
    IEmailSender emailSender,
    ILogger<OrderService> logger) : IOrderService
{
    public async Task<Order> CreateOrderAsync(
        CreateOrderRequest request, CancellationToken ct)
    {
        logger.LogInformation("Creating order for {CustomerId}", request.CustomerId);

        var order = Order.Create(request);
        await repository.AddAsync(order, ct);
        await emailSender.SendOrderConfirmationAsync(order, ct);

        return order;
    }
}

// Equivalent traditional constructor
public class OrderServiceTraditional : IOrderService
{
    private readonly IOrderRepository _repository;
    private readonly IEmailSender _emailSender;
    private readonly ILogger<OrderServiceTraditional> _logger;

    public OrderServiceTraditional(
        IOrderRepository repository,
        IEmailSender emailSender,
        ILogger<OrderServiceTraditional> logger)
    {
        _repository = repository;
        _emailSender = emailSender;
        _logger = logger;
    }
}
```

### Validation Patterns

```csharp
// Guard clauses for required dependencies
public class PaymentService : IPaymentService
{
    private readonly IPaymentGateway _gateway;
    private readonly ILogger<PaymentService> _logger;

    public PaymentService(
        IPaymentGateway gateway,
        ILogger<PaymentService> logger)
    {
        _gateway = gateway ?? throw new ArgumentNullException(nameof(gateway));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }
}
```

### Optional Dependencies

```csharp
// Use default values or null for optional dependencies
public class ReportService(
    IReportGenerator generator,
    ILogger<ReportService> logger,
    IReportCache? cache = null)
{
    public async Task<Report> GenerateAsync(ReportRequest request, CancellationToken ct)
    {
        if (cache is not null)
        {
            var cached = await cache.GetAsync(request.Key, ct);
            if (cached is not null) return cached;
        }

        var report = await generator.GenerateAsync(request, ct);
        if (cache is not null)
        {
            await cache.SetAsync(request.Key, report, ct);
        }

        return report;
    }
}
```

## Advanced Patterns

### Options Pattern

The Options pattern provides strongly typed access to configuration sections.

```csharp
// Configuration class
public sealed class SmtpSettings
{
    public const string SectionName = "Smtp";

    public required string Host { get; init; }
    public required int Port { get; init; }
    public required string Username { get; init; }
    public required string Password { get; init; }
    public bool UseSsl { get; init; } = true;
}

// Registration
builder.Services
    .AddOptions<SmtpSettings>()
    .BindConfiguration(SmtpSettings.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

#### IOptions vs IOptionsSnapshot vs IOptionsMonitor

| Interface | Lifetime | Reloads | Use When |
|-----------|----------|---------|----------|
| `IOptions<T>` | Singleton | ❌ Never | Static configuration, startup-only |
| `IOptionsSnapshot<T>` | Scoped | ✅ Per request | Per-request config in web apps |
| `IOptionsMonitor<T>` | Singleton | ✅ On change | Long-lived services needing live updates |

```csharp
// IOptions — read once, never changes
public class EmailSender(IOptions<SmtpSettings> options)
{
    private readonly SmtpSettings _settings = options.Value;
}

// IOptionsSnapshot — fresh value per scope/request
public class EmailSender(IOptionsSnapshot<SmtpSettings> options)
{
    public Task SendAsync()
    {
        var settings = options.Value; // fresh per request
        // ...
    }
}

// IOptionsMonitor — reacts to changes in real-time
public class EmailSender(IOptionsMonitor<SmtpSettings> monitor)
{
    public Task SendAsync()
    {
        var settings = monitor.CurrentValue; // always current
        // ...
    }
}
```

### Named Options

```csharp
// Register multiple named configurations
builder.Services.Configure<StorageOptions>("azure", 
    builder.Configuration.GetSection("Storage:Azure"));
builder.Services.Configure<StorageOptions>("aws", 
    builder.Configuration.GetSection("Storage:Aws"));

// Resolve by name
public class StorageFactory(IOptionsSnapshot<StorageOptions> options)
{
    public IStorageProvider Create(string provider)
    {
        var config = options.Get(provider);
        return provider switch
        {
            "azure" => new AzureBlobStorage(config),
            "aws" => new S3Storage(config),
            _ => throw new ArgumentException($"Unknown provider: {provider}")
        };
    }
}
```

### Factory Pattern with DI

```csharp
public interface INotificationChannelFactory
{
    INotificationChannel Create(string channelType);
}

public class NotificationChannelFactory(IServiceProvider sp)
    : INotificationChannelFactory
{
    public INotificationChannel Create(string channelType)
    {
        return channelType switch
        {
            "email" => sp.GetRequiredService<EmailChannel>(),
            "sms" => sp.GetRequiredService<SmsChannel>(),
            "push" => sp.GetRequiredService<PushChannel>(),
            _ => throw new ArgumentException(
                $"Unknown channel: {channelType}", nameof(channelType))
        };
    }
}

builder.Services.AddScoped<INotificationChannelFactory, NotificationChannelFactory>();
builder.Services.AddScoped<EmailChannel>();
builder.Services.AddScoped<SmsChannel>();
builder.Services.AddScoped<PushChannel>();
```

### Strategy Pattern with DI

```csharp
public interface IDiscountStrategy
{
    string Name { get; }
    decimal CalculateDiscount(Order order);
}

public class VolumeDiscount : IDiscountStrategy
{
    public string Name => "volume";
    public decimal CalculateDiscount(Order order) =>
        order.Items.Count > 10 ? 0.15m : 0m;
}

public class LoyaltyDiscount : IDiscountStrategy
{
    public string Name => "loyalty";
    public decimal CalculateDiscount(Order order) =>
        order.Customer.YearsActive > 2 ? 0.10m : 0m;
}

// Register all strategies
builder.Services.AddScoped<IDiscountStrategy, VolumeDiscount>();
builder.Services.AddScoped<IDiscountStrategy, LoyaltyDiscount>();

// Resolve and apply
public class DiscountService(IEnumerable<IDiscountStrategy> strategies)
{
    public decimal GetBestDiscount(Order order)
    {
        return strategies.Max(s => s.CalculateDiscount(order));
    }

    public decimal GetDiscountByName(Order order, string strategyName)
    {
        var strategy = strategies.FirstOrDefault(s => s.Name == strategyName)
            ?? throw new InvalidOperationException(
                $"No discount strategy '{strategyName}' registered");
        return strategy.CalculateDiscount(order);
    }
}
```

## Keyed Services (.NET 8+)

Keyed services allow registering and resolving multiple implementations of the same interface by a key. Available since .NET 8.

### Registration

```csharp
// Register keyed services
builder.Services.AddKeyedSingleton<ICache, RedisCache>("distributed");
builder.Services.AddKeyedSingleton<ICache, MemoryCache>("local");
builder.Services.AddKeyedScoped<IStorage, AzureBlobStorage>("azure");
builder.Services.AddKeyedScoped<IStorage, S3Storage>("aws");
```

### Resolution with [FromKeyedServices]

```csharp
public class ProductService(
    [FromKeyedServices("distributed")] ICache distributedCache,
    [FromKeyedServices("local")] ICache localCache,
    ILogger<ProductService> logger)
{
    public async Task<Product?> GetProductAsync(int id, CancellationToken ct)
    {
        // Check local cache first, then distributed
        var key = $"product:{id}";

        if (localCache.TryGet(key, out Product? product))
        {
            logger.LogDebug("Local cache hit for {Key}", key);
            return product;
        }

        product = await distributedCache.GetAsync<Product>(key, ct);
        if (product is not null)
        {
            logger.LogDebug("Distributed cache hit for {Key}", key);
            localCache.Set(key, product, TimeSpan.FromMinutes(5));
            return product;
        }

        return null;
    }
}
```

### Keyed Services in Minimal APIs

```csharp
app.MapGet("/files/{name}", async (
    string name,
    [FromKeyedServices("azure")] IStorage storage,
    CancellationToken ct) =>
{
    var file = await storage.GetAsync(name, ct);
    return file is not null ? Results.File(file.Stream, file.ContentType) : Results.NotFound();
});
```

## Scope Validation and Diagnostics

### ValidateScopes

Detects captive dependencies at runtime by throwing when a scoped service is resolved from the root provider.

```csharp
var builder = WebApplication.CreateBuilder(args);

// Enabled by default in Development environment
builder.Host.UseDefaultServiceProvider(options =>
{
    options.ValidateScopes = builder.Environment.IsDevelopment();
    options.ValidateOnBuild = builder.Environment.IsDevelopment();
});
```

### ValidateOnBuild

Validates all service registrations at startup — catches missing registrations before the first request.

```csharp
// Catches errors like:
// "Unable to resolve service for type 'IOrderRepository'
//  while attempting to activate 'OrderService'"
```

### Common DI Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `InvalidOperationException: Unable to resolve service` | Missing registration | Register the service in `Program.cs` |
| `InvalidOperationException: Cannot resolve scoped service from root provider` | Resolving scoped from singleton | Use `IServiceScopeFactory` or fix lifetime |
| `ObjectDisposedException` | Using a disposed scoped service | Don't store scoped services in singleton fields |
| `InvalidOperationException: Multiple constructors` | Ambiguous constructor | Use `[ActivatorUtilitiesConstructor]` attribute |
| Unexpected singleton behavior | Captive dependency | Check lifetime hierarchy — enable `ValidateScopes` |

## Third-Party DI Containers

### Comparison

| Feature | Built-in (MS.DI) | Autofac | Lamar |
|---------|------------------|---------|-------|
| **Constructor injection** | ✅ | ✅ | ✅ |
| **Property injection** | ❌ | ✅ | ✅ |
| **Keyed services** | ✅ (.NET 8+) | ✅ (named/keyed) | ✅ |
| **Open generics** | ✅ | ✅ | ✅ |
| **Decorators** | Manual (factory) | ✅ Built-in | ✅ Built-in |
| **Child containers** | ❌ | ✅ | ✅ |
| **Assembly scanning** | ❌ | ✅ | ✅ |
| **Interception (AOP)** | ❌ | ✅ | ❌ |
| **Performance** | Fastest | Fast | Fast |
| **Complexity** | Low | Medium | Medium |

### When to Use a Third-Party Container

- **Property injection** — injecting into properties instead of constructors
- **Decorator support** — automatic decorator chain registration
- **Child/nested containers** — tenant isolation in multi-tenant apps
- **Assembly scanning** — auto-register all implementations by convention
- **Interception/AOP** — cross-cutting concerns without manual decorators

For most applications, the built-in container is sufficient. Evaluate third-party containers only when you need features not available in `Microsoft.Extensions.DependencyInjection`.

## Testing with DI

### Mocking Dependencies

```csharp
public class OrderServiceTests
{
    [Fact]
    public async Task CreateOrder_SendsConfirmationEmail()
    {
        // Arrange
        var mockRepo = new Mock<IOrderRepository>();
        var mockEmail = new Mock<IEmailSender>();
        var logger = NullLogger<OrderService>.Instance;

        mockRepo
            .Setup(r => r.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()))
            .Returns(Task.CompletedTask);

        var service = new OrderService(mockRepo.Object, mockEmail.Object, logger);

        // Act
        var result = await service.CreateOrderAsync(
            new CreateOrderRequest { CustomerId = 1 },
            CancellationToken.None);

        // Assert
        mockEmail.Verify(
            e => e.SendOrderConfirmationAsync(
                It.Is<Order>(o => o.CustomerId == 1),
                It.IsAny<CancellationToken>()),
            Times.Once);
    }
}
```

### Using ServiceCollection in Tests

```csharp
public class IntegrationTests
{
    [Fact]
    public async Task OrderPipeline_ProcessesSuccessfully()
    {
        // Build a real DI container for integration testing
        var services = new ServiceCollection();
        services.AddScoped<IOrderRepository, InMemoryOrderRepository>();
        services.AddScoped<IEmailSender, FakeEmailSender>();
        services.AddScoped<IOrderService, OrderService>();
        services.AddLogging();

        await using var provider = services.BuildServiceProvider();
        using var scope = provider.CreateScope();

        var service = scope.ServiceProvider.GetRequiredService<IOrderService>();
        var result = await service.CreateOrderAsync(
            new CreateOrderRequest { CustomerId = 42 },
            CancellationToken.None);

        Assert.NotNull(result);
        Assert.Equal(42, result.CustomerId);
    }
}
```

### WebApplicationFactory

```csharp
public class OrderApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public OrderApiTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Replace real services with fakes
                services.RemoveAll<IEmailSender>();
                services.AddScoped<IEmailSender, FakeEmailSender>();

                services.RemoveAll<IOrderRepository>();
                services.AddScoped<IOrderRepository, InMemoryOrderRepository>();
            });
        });
    }

    [Fact]
    public async Task PostOrder_ReturnsCreated()
    {
        var client = _factory.CreateClient();

        var response = await client.PostAsJsonAsync("/api/orders", new
        {
            CustomerId = 1,
            Items = new[] { new { ProductId = 10, Quantity = 2 } }
        });

        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    }
}
```

## Anti-Patterns

### Service Locator

```csharp
// ❌ WRONG — Service Locator anti-pattern
public class OrderService
{
    private readonly IServiceProvider _sp;

    public OrderService(IServiceProvider sp) => _sp = sp;

    public async Task ProcessAsync()
    {
        // Hidden dependency — not visible in constructor
        var repo = _sp.GetRequiredService<IOrderRepository>();
        await repo.SaveAsync();
    }
}

// ✅ CORRECT — Explicit constructor injection
public class OrderService(
    IOrderRepository repository) : IOrderService
{
    public async Task ProcessAsync()
    {
        await repository.SaveAsync();
    }
}
```

### Captive Dependencies

```csharp
// ❌ WRONG — Singleton captures Scoped DbContext
builder.Services.AddSingleton<IReportService, ReportService>();
// ReportService(AppDbContext db) — scoped service captured!

// ✅ CORRECT — Use IServiceScopeFactory
builder.Services.AddSingleton<IReportService, ReportService>();

public class ReportService(IServiceScopeFactory scopeFactory) : IReportService
{
    public async Task GenerateAsync(CancellationToken ct)
    {
        await using var scope = scopeFactory.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // db is properly scoped
    }
}
```

### Over-Injection

```csharp
// ❌ WRONG — Too many dependencies (> 4 suggests SRP violation)
public class MegaService(
    IOrderRepo orders,
    IUserRepo users,
    IProductRepo products,
    IEmailSender email,
    ISmsSender sms,
    IPushSender push,
    ILogger<MegaService> logger,
    ICache cache,
    IMetrics metrics)
{ }

// ✅ CORRECT — Split into focused services
public class OrderNotificationService(
    IEmailSender email,
    ISmsSender sms,
    ILogger<OrderNotificationService> logger)
{ }

public class OrderProcessingService(
    IOrderRepo orders,
    OrderNotificationService notifications,
    ILogger<OrderProcessingService> logger)
{ }
```

### Static Service Access

```csharp
// ❌ WRONG — Static access to service provider
public static class ServiceLocator
{
    public static IServiceProvider Provider { get; set; } = null!;
}

// Used anywhere:
var repo = ServiceLocator.Provider.GetRequiredService<IOrderRepository>();

// ✅ CORRECT — Always use constructor injection
// If you need DI in a place that doesn't support it (e.g., an attribute),
// refactor to use a pattern that does (e.g., IFilterFactory in ASP.NET Core).
```

## Best Practices

### Production Checklist

- ✅ Register all services explicitly — avoid relying on implicit resolution
- ✅ Use `ValidateScopes` and `ValidateOnBuild` in Development
- ✅ Prefer interfaces for services that will be mocked in tests
- ✅ Use primary constructors (C# 12+) for concise DI
- ✅ Use the Options pattern for configuration — not raw `IConfiguration` injection
- ✅ Use `TryAdd` in library registration methods to allow consumer overrides
- ✅ Use keyed services (.NET 8+) instead of named-resolution workarounds
- ✅ Use `IServiceScopeFactory` in singletons that need scoped services
- ✅ Keep constructor dependency count ≤ 4 — split services if exceeding
- ✅ Register open generics for repository and handler patterns
- ✅ Dispose scoped services by letting the container manage their lifetime
- ✅ Use `WebApplicationFactory` for integration testing with DI overrides

- ❌ Don't use the Service Locator pattern — inject dependencies explicitly
- ❌ Don't inject `IServiceProvider` unless building a factory or framework code
- ❌ Don't let singletons capture scoped services (captive dependency)
- ❌ Don't resolve services in static classes or static methods
- ❌ Don't register `DbContext` as singleton — it is not thread-safe
- ❌ Don't use `new` to create services that have dependencies — let DI handle it
- ❌ Don't ignore `ValidateOnBuild` errors — they indicate real wiring issues

## Next Steps

Return to the [Learning Path](LEARNING-PATH.md) for the complete curriculum and additional topics. Also review [Services](02-SERVICES.md) for patterns on building ASP.NET Core APIs, background workers, and service-to-service communication.

### Related Documents

1. **[Learning Path](LEARNING-PATH.md)** — Complete .NET learning curriculum
2. **[Services](02-SERVICES.md)** — Web APIs, worker services, gRPC
3. **[Best Practices](03-BEST-PRACTICES.md)** — Async/await, memory, security, testing
4. **[Concurrency](05-CONCURRENCY-ASYNC-PARALLELISM.md)** — Async patterns, channels, parallelism

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial dependency injection documentation |
