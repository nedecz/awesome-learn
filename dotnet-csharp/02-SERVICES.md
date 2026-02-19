# .NET Services Patterns and Implementation

## Table of Contents

1. [Overview](#overview)
2. [Dependency Injection](#dependency-injection)
3. [ASP.NET Core Web API Services](#aspnet-core-web-api-services)
4. [Worker Services and BackgroundService](#worker-services-and-backgroundservice)
5. [gRPC Services in .NET](#grpc-services-in-net)
6. [Health Checks](#health-checks)
7. [Configuration and Options Pattern](#configuration-and-options-pattern)
8. [Logging](#logging)
9. [Service-to-Service Communication](#service-to-service-communication)
10. [Next Steps](#next-steps)

## Overview

This document covers the patterns and building blocks for constructing .NET services — from dependency injection and ASP.NET Core Web APIs to background workers, gRPC, health checks, configuration, logging, and service-to-service communication.

### Target Audience

- Developers building web services and APIs with ASP.NET Core
- Developers implementing background processing and worker services
- Architects designing service communication patterns in .NET

### Scope

- Built-in dependency injection container and service lifetimes
- Minimal APIs vs controller-based APIs
- Worker services, hosted services, and BackgroundService
- gRPC service implementation
- Health checks, configuration, logging, and HTTP communication

## Dependency Injection

.NET includes a built-in IoC (Inversion of Control) container that supports constructor injection, service lifetimes, and keyed services (.NET 8+).

### DI Container Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   IServiceCollection                     │
│                  (Registration Phase)                    │
│                                                         │
│  AddSingleton<ICache, RedisCache>()                     │
│  AddScoped<IUserService, UserService>()                 │
│  AddTransient<IValidator, RequestValidator>()            │
│  AddKeyedSingleton<ICache, MemCache>("memory")          │
└──────────────────────┬──────────────────────────────────┘
                       │  BuildServiceProvider()
                       ▼
┌─────────────────────────────────────────────────────────┐
│                   IServiceProvider                       │
│                  (Resolution Phase)                      │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │ Singleton Scope (application lifetime)            │  │
│  │  ┌──────────┐                                     │  │
│  │  │RedisCache│  ← one instance, shared by all     │  │
│  │  └──────────┘                                     │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │ Scoped Scope (per request / per scope)            │  │
│  │  ┌────────────┐  ┌────────────┐                   │  │
│  │  │UserService │  │UserService │  ← one per scope  │  │
│  │  │ (Request1) │  │ (Request2) │                   │  │
│  │  └────────────┘  └────────────┘                   │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │ Transient (new instance every time)               │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐          │  │
│  │  │Validator │ │Validator │ │Validator │           │  │
│  │  └──────────┘ └──────────┘ └──────────┘           │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### Service Lifetimes

| Lifetime | Method | Behavior | Use When |
|----------|--------|----------|----------|
| **Singleton** | `AddSingleton<T>()` | One instance for entire app lifetime | Stateless services, caches, configuration |
| **Scoped** | `AddScoped<T>()` | One instance per request/scope | DbContext, unit of work, per-request state |
| **Transient** | `AddTransient<T>()` | New instance every time | Lightweight, stateless services |

### Registration Patterns

```csharp
var builder = WebApplication.CreateBuilder(args);

// Interface to implementation
builder.Services.AddScoped<IUserRepository, SqlUserRepository>();

// Self-registration
builder.Services.AddScoped<UserService>();

// Factory registration
builder.Services.AddScoped<IEmailSender>(sp =>
{
    var config = sp.GetRequiredService<IConfiguration>();
    var apiKey = config["SendGrid:ApiKey"]!;
    return new SendGridEmailSender(apiKey);
});

// Multiple implementations
builder.Services.AddScoped<INotifier, EmailNotifier>();
builder.Services.AddScoped<INotifier, SmsNotifier>();
// Inject IEnumerable<INotifier> to get all implementations

// Conditional registration
builder.Services.TryAddScoped<IUserRepository, SqlUserRepository>();
// Only registers if IUserRepository is not already registered

// Decorator pattern
builder.Services.AddScoped<IUserRepository, SqlUserRepository>();
builder.Services.Decorate<IUserRepository, CachedUserRepository>();

// Open generics
builder.Services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
```

### Keyed Services (.NET 8+)

```csharp
// Registration with keys
builder.Services.AddKeyedSingleton<ICache, RedisCache>("distributed");
builder.Services.AddKeyedSingleton<ICache, MemoryCache>("local");

// Constructor injection
public class ProductService
{
    private readonly ICache _cache;

    public ProductService([FromKeyedServices("distributed")] ICache cache)
    {
        _cache = cache;
    }
}

// Minimal API parameter injection
app.MapGet("/products", ([FromKeyedServices("local")] ICache cache) =>
{
    return cache.GetOrCreate("products", () => GetProducts());
});
```

### Common DI Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Captive dependency | Singleton holds reference to Scoped service | Use `IServiceScopeFactory` in singletons |
| Service locator | Using `IServiceProvider` directly | Prefer constructor injection |
| Disposable transients | Transient `IDisposable` services leak memory | Use Scoped lifetime instead |
| Circular dependencies | Service A depends on B, B depends on A | Refactor or use `Lazy<T>` |

```csharp
// ❌ Captive dependency — Scoped service captured by Singleton
public class SingletonService
{
    private readonly IScopedService _scoped; // Dangerous!

    public SingletonService(IScopedService scoped)
    {
        _scoped = scoped;
    }
}

// ✅ Correct — create scope in Singleton
public class SingletonService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public SingletonService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public async Task DoWorkAsync()
    {
        using var scope = _scopeFactory.CreateScope();
        var scoped = scope.ServiceProvider.GetRequiredService<IScopedService>();
        await scoped.ProcessAsync();
    }
}
```

## ASP.NET Core Web API Services

### Minimal APIs vs Controllers

```
┌────────────────────────────────────────────────────────────┐
│                    ASP.NET Core Web API                     │
├──────────────────────────┬─────────────────────────────────┤
│      Minimal APIs        │       Controller-based          │
├──────────────────────────┼─────────────────────────────────┤
│ • Concise, functional    │ • Class-based, structured       │
│ • Single-file APIs       │ • Conventions (routing, model   │
│ • Great for small APIs   │   binding, filters)             │
│ • AOT-friendly           │ • Better for large APIs         │
│ • .NET 6+                │ • Full feature set              │
└──────────────────────────┴─────────────────────────────────┘
```

**Minimal API example:**

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IProductService, ProductService>();

var app = builder.Build();

var products = app.MapGroup("/api/products");

products.MapGet("/", async (IProductService service) =>
    TypedResults.Ok(await service.GetAllAsync()));

products.MapGet("/{id:int}", async (int id, IProductService service) =>
{
    var product = await service.GetByIdAsync(id);
    return product is not null
        ? TypedResults.Ok(product)
        : TypedResults.NotFound();
});

products.MapPost("/", async (CreateProductRequest request,
    IProductService service) =>
{
    var product = await service.CreateAsync(request);
    return TypedResults.Created($"/api/products/{product.Id}", product);
});

app.Run();
```

**Controller-based example:**

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _service;

    public ProductsController(IProductService service)
    {
        _service = service;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<Product>>> GetAll()
    {
        return Ok(await _service.GetAllAsync());
    }

    [HttpGet("{id:int}")]
    public async Task<ActionResult<Product>> GetById(int id)
    {
        var product = await _service.GetByIdAsync(id);
        return product is not null ? Ok(product) : NotFound();
    }

    [HttpPost]
    public async Task<ActionResult<Product>> Create(CreateProductRequest request)
    {
        var product = await _service.CreateAsync(request);
        return CreatedAtAction(nameof(GetById),
            new { id = product.Id }, product);
    }
}
```

### Middleware Pipeline

```
                         Request
                            │
                            ▼
┌───────────────────────────────────────────────────┐
│  ExceptionHandler Middleware                       │
│  ┌─────────────────────────────────────────────┐  │
│  │  HTTPS Redirection                          │  │
│  │  ┌───────────────────────────────────────┐  │  │
│  │  │  Static Files                         │  │  │
│  │  │  ┌─────────────────────────────────┐  │  │  │
│  │  │  │  Authentication                 │  │  │  │
│  │  │  │  ┌───────────────────────────┐  │  │  │  │
│  │  │  │  │  Authorization            │  │  │  │  │
│  │  │  │  │  ┌─────────────────────┐  │  │  │  │  │
│  │  │  │  │  │  Rate Limiting      │  │  │  │  │  │
│  │  │  │  │  │  ┌───────────────┐  │  │  │  │  │  │
│  │  │  │  │  │  │  Endpoint     │  │  │  │  │  │  │
│  │  │  │  │  │  │  (Your API)   │  │  │  │  │  │  │
│  │  │  │  │  │  └───────────────┘  │  │  │  │  │  │
│  │  │  │  │  └─────────────────────┘  │  │  │  │  │
│  │  │  │  └───────────────────────────┘  │  │  │  │
│  │  │  └─────────────────────────────────┘  │  │  │
│  │  └───────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────┘
                            │
                            ▼
                         Response
```

**Pipeline configuration:**

```csharp
var app = builder.Build();

// Order matters — middleware executes top to bottom on request,
// bottom to top on response
app.UseExceptionHandler("/error");
app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();
app.MapControllers();

app.Run();
```

### Custom Middleware

```csharp
public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTimingMiddleware> _logger;

    public RequestTimingMiddleware(RequestDelegate next,
        ILogger<RequestTimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();

        context.Response.OnStarting(() =>
        {
            stopwatch.Stop();
            context.Response.Headers["X-Response-Time"] =
                $"{stopwatch.ElapsedMilliseconds}ms";
            return Task.CompletedTask;
        });

        await _next(context);

        _logger.LogInformation(
            "{Method} {Path} responded {StatusCode} in {Elapsed}ms",
            context.Request.Method,
            context.Request.Path,
            context.Response.StatusCode,
            stopwatch.ElapsedMilliseconds);
    }
}

// Registration
app.UseMiddleware<RequestTimingMiddleware>();
```

## Worker Services and BackgroundService

### Hosted Service Lifecycle

```
Application Start
       │
       ▼
┌──────────────────────────────┐
│  IHostedLifecycleService     │
│                              │
│  1. StartingAsync()          │  ← Pre-start (warm up)
│  2. StartAsync()             │  ← Main start logic
│  3. StartedAsync()           │  ← Post-start (ready)
│                              │
│  ─── Application Running ─── │
│                              │
│  4. StoppingAsync()          │  ← Pre-stop (drain)
│  5. StopAsync()              │  ← Main stop logic
│  6. StoppedAsync()           │  ← Post-stop (cleanup)
└──────────────────────────────┘
       │
       ▼
Application Exit
```

### BackgroundService Pattern

```csharp
public class OrderProcessingWorker : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<OrderProcessingWorker> _logger;

    public OrderProcessingWorker(
        IServiceScopeFactory scopeFactory,
        ILogger<OrderProcessingWorker> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Order processing worker started");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = _scopeFactory.CreateScope();
                var processor = scope.ServiceProvider
                    .GetRequiredService<IOrderProcessor>();

                await processor.ProcessPendingOrdersAsync(stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                _logger.LogError(ex, "Error processing orders");
            }

            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
        }
    }
}

// Registration
builder.Services.AddHostedService<OrderProcessingWorker>();
```

### Timed Hosted Service with PeriodicTimer

```csharp
public class MetricsCollector : BackgroundService
{
    private readonly TimeProvider _timeProvider;
    private readonly ILogger<MetricsCollector> _logger;

    public MetricsCollector(TimeProvider timeProvider,
        ILogger<MetricsCollector> logger)
    {
        _timeProvider = timeProvider;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(
            TimeSpan.FromMinutes(1), _timeProvider);

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            _logger.LogInformation("Collecting metrics at {Time}",
                _timeProvider.GetUtcNow());
            await CollectMetricsAsync(stoppingToken);
        }
    }

    private Task CollectMetricsAsync(CancellationToken ct) =>
        Task.CompletedTask;
}
```

## gRPC Services in .NET

### gRPC Service Implementation

```protobuf
// Protos/product.proto
syntax = "proto3";

option csharp_namespace = "MyApp.Grpc";

service ProductService {
  rpc GetProduct (GetProductRequest) returns (ProductResponse);
  rpc ListProducts (ListProductsRequest) returns (stream ProductResponse);
}

message GetProductRequest {
  int32 id = 1;
}

message ListProductsRequest {
  int32 page_size = 1;
  string page_token = 2;
}

message ProductResponse {
  int32 id = 1;
  string name = 2;
  double price = 3;
}
```

```csharp
// Services/ProductGrpcService.cs
public class ProductGrpcService : ProductService.ProductServiceBase
{
    private readonly IProductRepository _repository;

    public ProductGrpcService(IProductRepository repository)
    {
        _repository = repository;
    }

    public override async Task<ProductResponse> GetProduct(
        GetProductRequest request, ServerCallContext context)
    {
        var product = await _repository.GetByIdAsync(request.Id);

        if (product is null)
        {
            throw new RpcException(new Status(
                StatusCode.NotFound, $"Product {request.Id} not found"));
        }

        return new ProductResponse
        {
            Id = product.Id,
            Name = product.Name,
            Price = (double)product.Price
        };
    }

    public override async Task ListProducts(
        ListProductsRequest request,
        IServerStreamWriter<ProductResponse> responseStream,
        ServerCallContext context)
    {
        await foreach (var product in _repository.StreamAllAsync())
        {
            await responseStream.WriteAsync(new ProductResponse
            {
                Id = product.Id,
                Name = product.Name,
                Price = (double)product.Price
            });
        }
    }
}

// Program.cs
builder.Services.AddGrpc();
app.MapGrpcService<ProductGrpcService>();
```

## Health Checks

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"])
    .AddSqlServer(
        builder.Configuration.GetConnectionString("Default")!,
        name: "sqlserver",
        tags: ["ready"])
    .AddRedis(
        builder.Configuration.GetConnectionString("Redis")!,
        name: "redis",
        tags: ["ready"])
    .AddCheck<ExternalApiHealthCheck>("external-api", tags: ["ready"]);

var app = builder.Build();

// Liveness probe — is the process alive?
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live")
});

// Readiness probe — is the service ready to accept requests?
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = WriteHealthCheckResponse
});

app.Run();

static Task WriteHealthCheckResponse(HttpContext context,
    HealthReport report)
{
    context.Response.ContentType = "application/json";
    var result = new
    {
        status = report.Status.ToString(),
        checks = report.Entries.Select(e => new
        {
            name = e.Key,
            status = e.Value.Status.ToString(),
            duration = e.Value.Duration.TotalMilliseconds
        })
    };
    return context.Response.WriteAsJsonAsync(result);
}
```

### Custom Health Check

```csharp
public class ExternalApiHealthCheck : IHealthCheck
{
    private readonly HttpClient _httpClient;

    public ExternalApiHealthCheck(IHttpClientFactory httpClientFactory)
    {
        _httpClient = httpClientFactory.CreateClient("ExternalApi");
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var response = await _httpClient.GetAsync("/health",
                cancellationToken);

            return response.IsSuccessStatusCode
                ? HealthCheckResult.Healthy("External API is reachable")
                : HealthCheckResult.Degraded(
                    $"External API returned {response.StatusCode}");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                "External API is unreachable", ex);
        }
    }
}
```

## Configuration and Options Pattern

### Configuration Sources (Priority Order)

```
Highest Priority
       │
       ▼
┌──────────────────────────────┐
│  Command-line arguments      │  dotnet run --Urls=http://+:5000
├──────────────────────────────┤
│  Environment variables       │  ASPNETCORE_ENVIRONMENT=Production
├──────────────────────────────┤
│  User secrets (dev only)     │  dotnet user-secrets set "Key" "Value"
├──────────────────────────────┤
│  appsettings.{Env}.json     │  appsettings.Production.json
├──────────────────────────────┤
│  appsettings.json            │  Base configuration
└──────────────────────────────┘
       │
Lowest Priority
```

### Options Pattern

```csharp
// Configuration classes
public class SmtpOptions
{
    public const string SectionName = "Smtp";

    public string Host { get; set; } = string.Empty;
    public int Port { get; set; } = 587;
    public string Username { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
    public bool UseSsl { get; set; } = true;
}

// appsettings.json
// {
//   "Smtp": {
//     "Host": "smtp.example.com",
//     "Port": 587,
//     "Username": "user@example.com",
//     "Password": "secret",
//     "UseSsl": true
//   }
// }

// Registration with validation
builder.Services.AddOptions<SmtpOptions>()
    .BindConfiguration(SmtpOptions.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

### IOptions Variants

| Interface | Lifetime | Reloads on Change | Use When |
|-----------|----------|-------------------|----------|
| `IOptions<T>` | Singleton | No | Static configuration that never changes |
| `IOptionsSnapshot<T>` | Scoped | Yes (per request) | Config that may change between requests |
| `IOptionsMonitor<T>` | Singleton | Yes (real-time) | Long-running services needing live updates |

```csharp
// IOptions — reads once, cached forever
public class EmailService
{
    private readonly SmtpOptions _options;

    public EmailService(IOptions<SmtpOptions> options)
    {
        _options = options.Value;
    }
}

// IOptionsSnapshot — reads per scope/request
public class EmailService
{
    private readonly SmtpOptions _options;

    public EmailService(IOptionsSnapshot<SmtpOptions> options)
    {
        _options = options.Value; // Fresh per request
    }
}

// IOptionsMonitor — reads in real-time, notifies on change
public class EmailService
{
    private readonly IOptionsMonitor<SmtpOptions> _monitor;

    public EmailService(IOptionsMonitor<SmtpOptions> monitor)
    {
        _monitor = monitor;
        _monitor.OnChange(opts =>
        {
            // React to configuration changes
        });
    }

    public void Send()
    {
        var current = _monitor.CurrentValue; // Always current
    }
}
```

## Logging

### Built-in ILogger

```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public OrderService(ILogger<OrderService> logger)
    {
        _logger = logger;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        _logger.LogInformation("Creating order for customer {CustomerId}",
            request.CustomerId);

        try
        {
            var order = await ProcessOrder(request);

            _logger.LogInformation(
                "Order {OrderId} created successfully for {CustomerId}",
                order.Id, request.CustomerId);

            return order;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "Failed to create order for customer {CustomerId}",
                request.CustomerId);
            throw;
        }
    }
}
```

### High-Performance Logging with Source Generators

```csharp
public static partial class LogMessages
{
    [LoggerMessage(Level = LogLevel.Information,
        Message = "Order {OrderId} created for customer {CustomerId}")]
    public static partial void OrderCreated(
        this ILogger logger, int orderId, int customerId);

    [LoggerMessage(Level = LogLevel.Error,
        Message = "Failed to process order {OrderId}")]
    public static partial void OrderProcessingFailed(
        this ILogger logger, int orderId, Exception exception);

    [LoggerMessage(Level = LogLevel.Warning,
        Message = "Order {OrderId} retry attempt {Attempt}")]
    public static partial void OrderRetry(
        this ILogger logger, int orderId, int attempt);
}

// Usage
_logger.OrderCreated(order.Id, request.CustomerId);
_logger.OrderProcessingFailed(orderId, ex);
```

### Structured Logging with Serilog

```csharp
// Program.cs
builder.Host.UseSerilog((context, config) =>
{
    config
        .ReadFrom.Configuration(context.Configuration)
        .Enrich.FromLogContext()
        .Enrich.WithMachineName()
        .Enrich.WithEnvironmentName()
        .WriteTo.Console(outputTemplate:
            "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj}" +
            " {Properties:j}{NewLine}{Exception}")
        .WriteTo.Seq("http://localhost:5341");
});

// appsettings.json
// {
//   "Serilog": {
//     "MinimumLevel": {
//       "Default": "Information",
//       "Override": {
//         "Microsoft.AspNetCore": "Warning",
//         "Microsoft.EntityFrameworkCore": "Warning"
//       }
//     }
//   }
// }
```

## Service-to-Service Communication

### HttpClientFactory

```csharp
// Named client registration
builder.Services.AddHttpClient("OrderApi", client =>
{
    client.BaseAddress = new Uri("https://orders.internal:5001");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
    client.Timeout = TimeSpan.FromSeconds(30);
})
.AddStandardResilienceHandler(); // .NET 8 resilience

// Typed client
public class OrderApiClient
{
    private readonly HttpClient _httpClient;

    public OrderApiClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<Order?> GetOrderAsync(int id,
        CancellationToken ct = default)
    {
        return await _httpClient.GetFromJsonAsync<Order>(
            $"/api/orders/{id}", ct);
    }

    public async Task<Order> CreateOrderAsync(
        CreateOrderRequest request, CancellationToken ct = default)
    {
        var response = await _httpClient.PostAsJsonAsync(
            "/api/orders", request, ct);
        response.EnsureSuccessStatusCode();
        return (await response.Content.ReadFromJsonAsync<Order>(ct))!;
    }
}

// Registration
builder.Services.AddHttpClient<OrderApiClient>(client =>
{
    client.BaseAddress = new Uri("https://orders.internal:5001");
});
```

### Refit (Declarative HTTP Client)

```csharp
// Define interface
public interface IOrderApi
{
    [Get("/api/orders/{id}")]
    Task<Order> GetOrderAsync(int id, CancellationToken ct = default);

    [Post("/api/orders")]
    Task<Order> CreateOrderAsync([Body] CreateOrderRequest request,
        CancellationToken ct = default);

    [Get("/api/orders")]
    Task<IReadOnlyList<Order>> ListOrdersAsync(
        [Query] int page = 1,
        [Query] int pageSize = 20,
        CancellationToken ct = default);
}

// Registration
builder.Services.AddRefitClient<IOrderApi>()
    .ConfigureHttpClient(client =>
    {
        client.BaseAddress = new Uri("https://orders.internal:5001");
    })
    .AddStandardResilienceHandler();
```

### gRPC Client

```csharp
// Registration
builder.Services.AddGrpcClient<ProductService.ProductServiceClient>(options =>
{
    options.Address = new Uri("https://products.internal:5002");
})
.ConfigureChannel(options =>
{
    options.MaxRetryAttempts = 3;
});

// Usage
public class CatalogService
{
    private readonly ProductService.ProductServiceClient _client;

    public CatalogService(ProductService.ProductServiceClient client)
    {
        _client = client;
    }

    public async Task<ProductResponse> GetProductAsync(int id)
    {
        return await _client.GetProductAsync(
            new GetProductRequest { Id = id });
    }
}
```

## Next Steps

Continue to [Best Practices](03-BEST-PRACTICES.md) to learn about async/await patterns, memory management, security, testing, and production readiness for .NET applications.

### Related Documents

1. **[Breaking Changes](01-BREAKING-CHANGES-DOTNET-CORE-31.md)** — Migration from .NET Core 3.1 through .NET 8
2. **[Best Practices](03-BEST-PRACTICES.md)** — Async/await, memory, security, testing
3. **[Style Guide](04-STYLE-GUIDE.md)** — Naming conventions, formatting, analyzers

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial .NET services patterns documentation |
