# Design Patterns in C# and .NET

## Table of Contents

1. [Overview](#overview)
2. [Creational Patterns](#creational-patterns)
3. [Structural Patterns](#structural-patterns)
4. [Behavioral Patterns](#behavioral-patterns)
5. [CQRS Pattern](#cqrs-pattern)
6. [Repository and Unit of Work](#repository-and-unit-of-work)
7. [Options Pattern](#options-pattern)
8. [Result Pattern](#result-pattern)
9. [Specification Pattern](#specification-pattern)
10. [Pipeline Pattern](#pipeline-pattern)
11. [Best Practices](#best-practices)
12. [Next Steps](#next-steps)

## Overview

Design patterns are proven, reusable solutions to common software design problems. In modern .NET (targeting .NET 10 / C# 13), patterns work hand-in-hand with the built-in dependency injection container, middleware pipeline, and libraries like MediatR to produce maintainable, testable applications.

### Target Audience

- .NET developers building production applications with ASP.NET Core and EF Core
- Architects selecting patterns for service layers, CQRS, and domain logic
- Teams establishing consistent design conventions across C# codebases

### Scope

- Creational, structural, and behavioral Gang-of-Four patterns adapted for modern C#
- CQRS and Mediator patterns with MediatR
- Repository, Unit of Work, Options, Result, Specification, and Pipeline patterns
- Pattern selection guidance, anti-patterns, and production best practices

### Pattern Categories

```
┌─────────────────────────────────────────────────────────────────┐
│                      Design Patterns                            │
├───────────────────┬───────────────────┬─────────────────────────┤
│    Creational     │    Structural     │      Behavioral         │
│                   │                   │                         │
│  Factory Method   │  Adapter          │  Strategy               │
│  Abstract Factory │  Decorator        │  Observer               │
│  Builder          │  Facade           │  Command                │
│  Singleton        │  Proxy            │  Mediator               │
│                   │                   │  Chain of Responsibility│
├───────────────────┴───────────────────┴─────────────────────────┤
│                  .NET-Specific Patterns                          │
│                                                                 │
│  CQRS · Repository · Unit of Work · Options · Result            │
│  Specification · Pipeline (Middleware / MediatR Behaviors)       │
└─────────────────────────────────────────────────────────────────┘
```

## Creational Patterns

Creational patterns control how objects are created, promoting flexibility and decoupling from concrete types.

### Factory Method

Define an interface for creating an object and let subclasses decide which class to instantiate.

```csharp
public interface INotification
{
    Task SendAsync(string recipient, string message, CancellationToken ct = default);
}

public class EmailNotification : INotification
{
    public Task SendAsync(string recipient, string message, CancellationToken ct = default)
    {
        Console.WriteLine($"Email to {recipient}: {message}");
        return Task.CompletedTask;
    }
}

public class SmsNotification : INotification
{
    public Task SendAsync(string recipient, string message, CancellationToken ct = default)
    {
        Console.WriteLine($"SMS to {recipient}: {message}");
        return Task.CompletedTask;
    }
}

public interface INotificationFactory
{
    INotification Create(string channel);
}

public class NotificationFactory : INotificationFactory
{
    public INotification Create(string channel) => channel.ToLowerInvariant() switch
    {
        "email" => new EmailNotification(),
        "sms"   => new SmsNotification(),
        _       => throw new ArgumentException($"Unknown channel: {channel}")
    };
}

// DI registration
builder.Services.AddSingleton<INotificationFactory, NotificationFactory>();
```

### Abstract Factory

Produce families of related objects without specifying concrete classes.

```csharp
public interface IStorageFactory
{
    IBlobStore CreateBlobStore();
    IQueueClient CreateQueueClient();
}

public class AzureStorageFactory : IStorageFactory
{
    public IBlobStore CreateBlobStore() => new AzureBlobStore();
    public IQueueClient CreateQueueClient() => new AzureQueueClient();
}

public class AwsStorageFactory : IStorageFactory
{
    public IBlobStore CreateBlobStore() => new S3BlobStore();
    public IQueueClient CreateQueueClient() => new SqsQueueClient();
}

// Registration — swap cloud providers via configuration
builder.Services.AddSingleton<IStorageFactory>(sp =>
{
    var provider = builder.Configuration["Cloud:Provider"];
    return provider switch
    {
        "Azure" => new AzureStorageFactory(),
        "AWS"   => new AwsStorageFactory(),
        _       => throw new InvalidOperationException($"Unsupported provider: {provider}")
    };
});
```

### Builder

Separate the construction of a complex object from its representation.

```csharp
public class ReportBuilder
{
    private string _title = string.Empty;
    private readonly List<string> _sections = [];
    private bool _includeCharts;
    private DateOnly? _dateRange;

    public ReportBuilder WithTitle(string title)
    {
        _title = title;
        return this;
    }

    public ReportBuilder AddSection(string section)
    {
        _sections.Add(section);
        return this;
    }

    public ReportBuilder IncludeCharts(bool include = true)
    {
        _includeCharts = include;
        return this;
    }

    public ReportBuilder ForDate(DateOnly date)
    {
        _dateRange = date;
        return this;
    }

    public Report Build()
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(_title);
        return new Report(_title, [.. _sections], _includeCharts, _dateRange);
    }
}

public record Report(
    string Title,
    IReadOnlyList<string> Sections,
    bool IncludeCharts,
    DateOnly? DateRange);

// Usage
var report = new ReportBuilder()
    .WithTitle("Q4 Sales")
    .AddSection("Revenue")
    .AddSection("Expenses")
    .IncludeCharts()
    .ForDate(new DateOnly(2025, 12, 1))
    .Build();
```

### Singleton (with DI)

In modern .NET, avoid hand-rolled singletons — register services as `Singleton` in the DI container.

```csharp
// ❌ WRONG — Classic double-check lock singleton
public class LegacyCache
{
    private static LegacyCache? _instance;
    private static readonly Lock _lock = new();

    public static LegacyCache Instance
    {
        get
        {
            if (_instance is null)
            {
                lock (_lock)
                {
                    _instance ??= new LegacyCache();
                }
            }
            return _instance;
        }
    }
}

// ✅ CORRECT — DI-managed singleton
public interface ICacheService
{
    Task<T?> GetAsync<T>(string key, CancellationToken ct = default);
    Task SetAsync<T>(string key, T value, TimeSpan expiry, CancellationToken ct = default);
}

public class MemoryCacheService(IMemoryCache cache) : ICacheService
{
    public Task<T?> GetAsync<T>(string key, CancellationToken ct = default)
    {
        cache.TryGetValue(key, out T? value);
        return Task.FromResult(value);
    }

    public Task SetAsync<T>(string key, T value, TimeSpan expiry, CancellationToken ct = default)
    {
        cache.Set(key, value, expiry);
        return Task.CompletedTask;
    }
}

// Registration — container guarantees a single instance
builder.Services.AddMemoryCache();
builder.Services.AddSingleton<ICacheService, MemoryCacheService>();
```

## Structural Patterns

Structural patterns define how classes and objects are composed to form larger, flexible structures.

### Adapter

Convert the interface of an existing class into another interface clients expect.

```csharp
// Third-party legacy payment SDK (cannot modify)
public class LegacyPaymentGateway
{
    public int MakePayment(string account, decimal total)
    {
        // Returns transaction ID
        return Random.Shared.Next(10000, 99999);
    }
}

// Our application interface
public interface IPaymentProcessor
{
    Task<PaymentResult> ProcessAsync(PaymentRequest request, CancellationToken ct = default);
}

public record PaymentRequest(string AccountId, decimal Amount, string Currency);
public record PaymentResult(string TransactionId, bool Success);

// Adapter bridges the two
public class LegacyPaymentAdapter(LegacyPaymentGateway gateway) : IPaymentProcessor
{
    public Task<PaymentResult> ProcessAsync(PaymentRequest request, CancellationToken ct = default)
    {
        var transactionId = gateway.MakePayment(request.AccountId, request.Amount);
        return Task.FromResult(new PaymentResult(transactionId.ToString(), Success: true));
    }
}

// Registration
builder.Services.AddSingleton<LegacyPaymentGateway>();
builder.Services.AddSingleton<IPaymentProcessor, LegacyPaymentAdapter>();
```

### Decorator

Attach additional responsibilities to an object dynamically without modifying its code.

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(int id, CancellationToken ct = default);
}

public class SqlOrderRepository(AppDbContext db) : IOrderRepository
{
    public async Task<Order?> GetByIdAsync(int id, CancellationToken ct = default)
        => await db.Orders.FindAsync([id], ct);
}

// Decorator adds caching on top
public class CachedOrderRepository(
    IOrderRepository inner,
    IMemoryCache cache,
    ILogger<CachedOrderRepository> logger) : IOrderRepository
{
    public async Task<Order?> GetByIdAsync(int id, CancellationToken ct = default)
    {
        var key = $"order:{id}";
        if (cache.TryGetValue(key, out Order? cached))
        {
            logger.LogDebug("Cache hit for {Key}", key);
            return cached;
        }

        var order = await inner.GetByIdAsync(id, ct);
        if (order is not null)
        {
            cache.Set(key, order, TimeSpan.FromMinutes(5));
        }
        return order;
    }
}

// Registration — decorate manually or use Scrutor
builder.Services.AddScoped<SqlOrderRepository>();
builder.Services.AddScoped<IOrderRepository>(sp =>
    new CachedOrderRepository(
        sp.GetRequiredService<SqlOrderRepository>(),
        sp.GetRequiredService<IMemoryCache>(),
        sp.GetRequiredService<ILogger<CachedOrderRepository>>()));
```

### Facade

Provide a simplified interface to a complex subsystem.

```csharp
public class OrderFacade(
    IOrderRepository orders,
    IInventoryService inventory,
    IPaymentProcessor payment,
    INotificationFactory notifications)
{
    public async Task<OrderConfirmation> PlaceOrderAsync(
        PlaceOrderRequest request, CancellationToken ct = default)
    {
        // 1. Reserve inventory
        await inventory.ReserveAsync(request.Items, ct);

        // 2. Process payment
        var paymentResult = await payment.ProcessAsync(
            new PaymentRequest(request.AccountId, request.Total, "USD"), ct);

        if (!paymentResult.Success)
        {
            await inventory.ReleaseAsync(request.Items, ct);
            throw new PaymentFailedException(paymentResult.TransactionId);
        }

        // 3. Create order
        var order = new Order(request.CustomerId, request.Items, paymentResult.TransactionId);
        await orders.AddAsync(order, ct);

        // 4. Send confirmation
        var notifier = notifications.Create("email");
        await notifier.SendAsync(request.Email, $"Order {order.Id} confirmed", ct);

        return new OrderConfirmation(order.Id, paymentResult.TransactionId);
    }
}
```

### Proxy

Control access to an object through a surrogate that adds behavior such as authorization, logging, or lazy initialization.

```csharp
public interface IProductCatalog
{
    Task<IReadOnlyList<Product>> SearchAsync(string query, CancellationToken ct = default);
}

public class ProductCatalog(AppDbContext db) : IProductCatalog
{
    public async Task<IReadOnlyList<Product>> SearchAsync(string query, CancellationToken ct = default)
        => await db.Products
            .Where(p => p.Name.Contains(query))
            .ToListAsync(ct);
}

// Logging proxy
public class LoggingProductCatalogProxy(
    IProductCatalog inner,
    ILogger<LoggingProductCatalogProxy> logger) : IProductCatalog
{
    public async Task<IReadOnlyList<Product>> SearchAsync(string query, CancellationToken ct = default)
    {
        logger.LogInformation("Searching products: {Query}", query);
        var sw = Stopwatch.StartNew();

        var results = await inner.SearchAsync(query, ct);

        logger.LogInformation("Search returned {Count} results in {Elapsed}ms",
            results.Count, sw.ElapsedMilliseconds);
        return results;
    }
}
```

## Behavioral Patterns

Behavioral patterns define how objects communicate and distribute responsibilities.

### Strategy

Define a family of algorithms, encapsulate each one, and make them interchangeable.

```csharp
public interface IDiscountStrategy
{
    decimal Calculate(Order order);
}

public class NoDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => 0m;
}

public class PercentageDiscount(decimal percent) : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Subtotal * (percent / 100m);
}

public class LoyaltyDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Customer.LoyaltyYears switch
    {
        >= 5 => order.Subtotal * 0.15m,
        >= 2 => order.Subtotal * 0.10m,
        >= 1 => order.Subtotal * 0.05m,
        _    => 0m
    };
}

public class PricingService(IEnumerable<IDiscountStrategy> strategies)
{
    public decimal CalculateFinalPrice(Order order)
    {
        var bestDiscount = strategies.Max(s => s.Calculate(order));
        return order.Subtotal - bestDiscount;
    }
}

// Registration — inject all strategies
builder.Services.AddSingleton<IDiscountStrategy, NoDiscount>();
builder.Services.AddSingleton<IDiscountStrategy, PercentageDiscount>(
    _ => new PercentageDiscount(10));
builder.Services.AddSingleton<IDiscountStrategy, LoyaltyDiscount>();
builder.Services.AddScoped<PricingService>();
```

### Observer

Define a one-to-many dependency so that when one object changes state, all dependents are notified.

```csharp
public interface IDomainEvent { }

public record OrderPlacedEvent(int OrderId, int CustomerId, decimal Total) : IDomainEvent;

public interface IDomainEventHandler<in TEvent> where TEvent : IDomainEvent
{
    Task HandleAsync(TEvent domainEvent, CancellationToken ct = default);
}

public class SendOrderConfirmationHandler : IDomainEventHandler<OrderPlacedEvent>
{
    public Task HandleAsync(OrderPlacedEvent e, CancellationToken ct = default)
    {
        Console.WriteLine($"Sending confirmation for order {e.OrderId}");
        return Task.CompletedTask;
    }
}

public class UpdateInventoryHandler : IDomainEventHandler<OrderPlacedEvent>
{
    public Task HandleAsync(OrderPlacedEvent e, CancellationToken ct = default)
    {
        Console.WriteLine($"Updating inventory for order {e.OrderId}");
        return Task.CompletedTask;
    }
}

public class DomainEventDispatcher(IServiceProvider sp)
{
    public async Task DispatchAsync<TEvent>(TEvent domainEvent, CancellationToken ct = default)
        where TEvent : IDomainEvent
    {
        var handlers = sp.GetServices<IDomainEventHandler<TEvent>>();
        foreach (var handler in handlers)
        {
            await handler.HandleAsync(domainEvent, ct);
        }
    }
}

// Registration
builder.Services.AddScoped<IDomainEventHandler<OrderPlacedEvent>, SendOrderConfirmationHandler>();
builder.Services.AddScoped<IDomainEventHandler<OrderPlacedEvent>, UpdateInventoryHandler>();
builder.Services.AddScoped<DomainEventDispatcher>();
```

### Command

Encapsulate a request as an object, allowing parameterization, queuing, and undo.

```csharp
public interface ICommand<TResult>
{
    // Marker interface
}

public interface ICommandHandler<in TCommand, TResult> where TCommand : ICommand<TResult>
{
    Task<TResult> HandleAsync(TCommand command, CancellationToken ct = default);
}

public record CreateOrderCommand(int CustomerId, List<OrderItem> Items) : ICommand<int>;

public class CreateOrderHandler(
    AppDbContext db,
    ILogger<CreateOrderHandler> logger) : ICommandHandler<CreateOrderCommand, int>
{
    public async Task<int> HandleAsync(CreateOrderCommand command, CancellationToken ct = default)
    {
        var order = new Order
        {
            CustomerId = command.CustomerId,
            Items = command.Items,
            CreatedAt = DateTimeOffset.UtcNow
        };

        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);

        logger.LogInformation("Created order {OrderId}", order.Id);
        return order.Id;
    }
}
```

### Mediator (MediatR)

```
┌──────────────┐        ┌─────────────┐        ┌──────────────────┐
│   Controller │──req──▶│  IMediator  │──req──▶│   Handler        │
│   / Endpoint │        │  (MediatR)  │        │  (IRequestHandler)│
│              │◀─res───│             │◀─res───│                  │
└──────────────┘        └──────┬──────┘        └──────────────────┘
                               │
                    ┌──────────┼──────────┐
                    ▼          ▼          ▼
              ┌──────────┐ ┌────────┐ ┌──────────┐
              │Validation│ │Logging │ │ Caching  │
              │ Behavior │ │Behavior│ │ Behavior │
              └──────────┘ └────────┘ └──────────┘
              (IPipelineBehavior<,> pipeline behaviors)
```

```csharp
// Install: dotnet add package MediatR

// Request and response
public record GetOrderQuery(int OrderId) : IRequest<OrderDto?>;

public record OrderDto(int Id, int CustomerId, decimal Total, DateTimeOffset CreatedAt);

// Handler
public class GetOrderQueryHandler(
    AppDbContext db) : IRequestHandler<GetOrderQuery, OrderDto?>
{
    public async Task<OrderDto?> Handle(GetOrderQuery request, CancellationToken ct)
    {
        var order = await db.Orders.FindAsync([request.OrderId], ct);
        return order is null
            ? null
            : new OrderDto(order.Id, order.CustomerId, order.Total, order.CreatedAt);
    }
}

// Notification (publish to multiple handlers)
public record OrderPlacedNotification(int OrderId) : INotification;

public class OrderPlacedEmailHandler : INotificationHandler<OrderPlacedNotification>
{
    public Task Handle(OrderPlacedNotification notification, CancellationToken ct)
    {
        Console.WriteLine($"Email sent for order {notification.OrderId}");
        return Task.CompletedTask;
    }
}

// Registration
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssemblyContaining<Program>());

// Usage in minimal API
app.MapGet("/orders/{id:int}", async (int id, IMediator mediator) =>
{
    var order = await mediator.Send(new GetOrderQuery(id));
    return order is not null ? Results.Ok(order) : Results.NotFound();
});
```

### Chain of Responsibility

Pass a request along a chain of handlers until one handles it.

```csharp
public abstract class ApprovalHandler
{
    private ApprovalHandler? _next;

    public ApprovalHandler SetNext(ApprovalHandler next)
    {
        _next = next;
        return next;
    }

    public virtual Task<bool> HandleAsync(PurchaseRequest request, CancellationToken ct = default)
    {
        return _next is not null
            ? _next.HandleAsync(request, ct)
            : Task.FromResult(false);
    }
}

public record PurchaseRequest(string Requester, decimal Amount);

public class ManagerApproval : ApprovalHandler
{
    public override Task<bool> HandleAsync(PurchaseRequest request, CancellationToken ct = default)
    {
        if (request.Amount <= 1_000m)
        {
            Console.WriteLine($"Manager approved ${request.Amount} for {request.Requester}");
            return Task.FromResult(true);
        }
        return base.HandleAsync(request, ct);
    }
}

public class DirectorApproval : ApprovalHandler
{
    public override Task<bool> HandleAsync(PurchaseRequest request, CancellationToken ct = default)
    {
        if (request.Amount <= 10_000m)
        {
            Console.WriteLine($"Director approved ${request.Amount} for {request.Requester}");
            return Task.FromResult(true);
        }
        return base.HandleAsync(request, ct);
    }
}

public class VpApproval : ApprovalHandler
{
    public override Task<bool> HandleAsync(PurchaseRequest request, CancellationToken ct = default)
    {
        Console.WriteLine($"VP approved ${request.Amount} for {request.Requester}");
        return Task.FromResult(true);
    }
}

// Build the chain
var chain = new ManagerApproval();
chain.SetNext(new DirectorApproval())
     .SetNext(new VpApproval());

await chain.HandleAsync(new PurchaseRequest("Alice", 5_000m));
```

## CQRS Pattern

Command Query Responsibility Segregation separates read and write models for independently optimized data access.

```
┌─────────────────────────────────────────────────────────────────┐
│                         CQRS Architecture                       │
│                                                                 │
│  ┌──────────┐    Command    ┌──────────────┐   ┌─────────────┐ │
│  │          │──────────────▶│ Write Model  │──▶│  Write DB   │ │
│  │  Client  │               │ (Domain Obj) │   │ (Normalized)│ │
│  │          │    Query      ├──────────────┤   ├─────────────┤ │
│  │          │──────────────▶│ Read Model   │──▶│  Read DB    │ │
│  └──────────┘               │ (DTOs/Views) │   │(Denormalized│ │
│                             └──────────────┘   │  or Cache)  │ │
│                                                └─────────────┘ │
│                                                                 │
│  Commands: CreateOrder, UpdateCustomer   (return void / ID)     │
│  Queries:  GetOrderById, ListOrders      (return data)          │
└─────────────────────────────────────────────────────────────────┘
```

### CQRS with MediatR

```csharp
// ── Commands (write side) ──

public record CreateOrderCommand(
    int CustomerId,
    List<OrderItemDto> Items) : IRequest<int>;

public class CreateOrderCommandHandler(
    AppDbContext db) : IRequestHandler<CreateOrderCommand, int>
{
    public async Task<int> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        var order = new Order
        {
            CustomerId = cmd.CustomerId,
            Items = cmd.Items.Select(i => new OrderItem
            {
                ProductId = i.ProductId,
                Quantity = i.Quantity,
                UnitPrice = i.UnitPrice
            }).ToList(),
            Status = OrderStatus.Pending,
            CreatedAt = DateTimeOffset.UtcNow
        };

        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);
        return order.Id;
    }
}

// ── Queries (read side) ──

public record GetOrderByIdQuery(int OrderId) : IRequest<OrderDetailsDto?>;

public record OrderDetailsDto(
    int Id,
    string CustomerName,
    decimal Total,
    string Status,
    IReadOnlyList<OrderLineDto> Lines);

public record OrderLineDto(string ProductName, int Quantity, decimal LineTotal);

public class GetOrderByIdQueryHandler(
    IDbConnection connection) : IRequestHandler<GetOrderByIdQuery, OrderDetailsDto?>
{
    public async Task<OrderDetailsDto?> Handle(GetOrderByIdQuery query, CancellationToken ct)
    {
        // Read side uses Dapper for lightweight, optimized reads
        const string sql = """
            SELECT o.Id, c.Name AS CustomerName, o.Total, o.Status
            FROM Orders o
            JOIN Customers c ON c.Id = o.CustomerId
            WHERE o.Id = @OrderId
            """;

        return await connection.QuerySingleOrDefaultAsync<OrderDetailsDto>(
            sql, new { query.OrderId });
    }
}
```

## Repository and Unit of Work

### Repository Pattern

```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
    void Update(T entity);
    void Remove(T entity);
}

public class EfRepository<T>(AppDbContext db) : IRepository<T> where T : class
{
    protected readonly DbSet<T> DbSet = db.Set<T>();

    public async Task<T?> GetByIdAsync(int id, CancellationToken ct = default)
        => await DbSet.FindAsync([id], ct);

    public async Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default)
        => await DbSet.ToListAsync(ct);

    public async Task AddAsync(T entity, CancellationToken ct = default)
        => await DbSet.AddAsync(entity, ct);

    public void Update(T entity) => DbSet.Update(entity);

    public void Remove(T entity) => DbSet.Remove(entity);
}
```

### Specialized Repositories

```csharp
public interface IOrderRepository : IRepository<Order>
{
    Task<IReadOnlyList<Order>> GetByCustomerAsync(int customerId, CancellationToken ct = default);
    Task<Order?> GetWithItemsAsync(int orderId, CancellationToken ct = default);
}

public class OrderRepository(AppDbContext db) : EfRepository<Order>(db), IOrderRepository
{
    public async Task<IReadOnlyList<Order>> GetByCustomerAsync(
        int customerId, CancellationToken ct = default)
        => await DbSet
            .Where(o => o.CustomerId == customerId)
            .OrderByDescending(o => o.CreatedAt)
            .ToListAsync(ct);

    public async Task<Order?> GetWithItemsAsync(int orderId, CancellationToken ct = default)
        => await DbSet
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == orderId, ct);
}
```

### Unit of Work

```csharp
public interface IUnitOfWork : IAsyncDisposable
{
    IOrderRepository Orders { get; }
    IRepository<Customer> Customers { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}

public class UnitOfWork(AppDbContext db) : IUnitOfWork
{
    private IOrderRepository? _orders;
    private IRepository<Customer>? _customers;

    public IOrderRepository Orders => _orders ??= new OrderRepository(db);
    public IRepository<Customer> Customers => _customers ??= new EfRepository<Customer>(db);

    public Task<int> SaveChangesAsync(CancellationToken ct = default)
        => db.SaveChangesAsync(ct);

    public ValueTask DisposeAsync() => db.DisposeAsync();
}

// Registration
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
```

### EF Core as Built-in Unit of Work

EF Core's `DbContext` already implements Unit of Work and Repository patterns internally. In many applications, wrapping EF Core in a custom repository adds unnecessary abstraction.

```csharp
// DbContext IS the Unit of Work — SaveChangesAsync commits all tracked changes
public class OrderService(AppDbContext db)
{
    public async Task TransferItemAsync(int fromOrderId, int toOrderId, int itemId,
        CancellationToken ct = default)
    {
        var fromOrder = await db.Orders.Include(o => o.Items)
            .FirstAsync(o => o.Id == fromOrderId, ct);
        var toOrder = await db.Orders.Include(o => o.Items)
            .FirstAsync(o => o.Id == toOrderId, ct);

        var item = fromOrder.Items.First(i => i.Id == itemId);
        fromOrder.Items.Remove(item);
        toOrder.Items.Add(item);

        // Single SaveChanges commits both changes atomically
        await db.SaveChangesAsync(ct);
    }
}
```

| Approach | Use When |
|---|---|
| Direct `DbContext` injection | Simple CRUD apps, tight EF Core coupling is acceptable |
| Generic `IRepository<T>` | Need to abstract data access for testing or swappable stores |
| Specialized repositories | Domain-specific queries, complex aggregates |
| Full Unit of Work | Multiple repositories must participate in one transaction |

## Options Pattern

The Options pattern provides strongly-typed access to configuration sections with validation and change notification.

### IOptions Variants

| Interface | Lifetime | Reloads | Best For |
|---|---|---|---|
| `IOptions<T>` | Singleton | No | Static config read once at startup |
| `IOptionsSnapshot<T>` | Scoped | Per request | Config that may change between requests |
| `IOptionsMonitor<T>` | Singleton | Real-time | Long-running services needing live updates |

### Full Options Example

```csharp
public class SmtpOptions
{
    public const string SectionName = "Smtp";

    [Required, MinLength(1)]
    public string Host { get; set; } = string.Empty;

    [Range(1, 65535)]
    public int Port { get; set; } = 587;

    [Required, EmailAddress]
    public string FromAddress { get; set; } = string.Empty;

    public string? Username { get; set; }
    public string? Password { get; set; }
    public bool UseTls { get; set; } = true;
}

// Registration with validation
builder.Services.AddOptions<SmtpOptions>()
    .BindConfiguration(SmtpOptions.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

### Named Options

```csharp
// appsettings.json:
// { "Storage": { "Local": { "Path": "/data" }, "Cloud": { "Path": "s3://bucket" } } }

builder.Services.Configure<StorageOptions>("Local",
    builder.Configuration.GetSection("Storage:Local"));
builder.Services.Configure<StorageOptions>("Cloud",
    builder.Configuration.GetSection("Storage:Cloud"));

public class FileService(IOptionsSnapshot<StorageOptions> options)
{
    public string GetPath(string profile)
    {
        var opts = options.Get(profile);
        return opts.Path;
    }
}
```

### IOptionsMonitor for Live Reloads

```csharp
public class FeatureFlagService(IOptionsMonitor<FeatureFlags> monitor)
{
    public bool IsEnabled(string feature)
    {
        var flags = monitor.CurrentValue;
        return flags.EnabledFeatures.Contains(feature);
    }

    // React to changes
    public void StartWatching()
    {
        monitor.OnChange(flags =>
        {
            Console.WriteLine($"Features updated: {string.Join(", ", flags.EnabledFeatures)}");
        });
    }
}
```

## Result Pattern

Use the Result pattern to represent success or failure without throwing exceptions for expected outcomes.

### Custom Result Type

```csharp
public class Result<T>
{
    public T? Value { get; }
    public string? Error { get; }
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;

    private Result(T value) { Value = value; IsSuccess = true; }
    private Result(string error) { Error = error; IsSuccess = false; }

    public static Result<T> Success(T value) => new(value);
    public static Result<T> Failure(string error) => new(error);

    public TResult Match<TResult>(Func<T, TResult> onSuccess, Func<string, TResult> onFailure)
        => IsSuccess ? onSuccess(Value!) : onFailure(Error!);
}

// Usage in a service
public class UserService(AppDbContext db)
{
    public async Task<Result<User>> GetByEmailAsync(string email, CancellationToken ct = default)
    {
        if (string.IsNullOrWhiteSpace(email))
            return Result<User>.Failure("Email is required.");

        var user = await db.Users.FirstOrDefaultAsync(u => u.Email == email, ct);
        return user is not null
            ? Result<User>.Success(user)
            : Result<User>.Failure($"User with email '{email}' not found.");
    }
}

// Usage in a minimal API endpoint
app.MapGet("/users/{email}", async (string email, UserService users) =>
{
    var result = await users.GetByEmailAsync(email);
    return result.Match(
        onSuccess: user => Results.Ok(user),
        onFailure: error => Results.NotFound(new { error }));
});
```

### Result Pattern with OneOf

```csharp
// Install: dotnet add package OneOf

using OneOf;
using OneOf.Types;

public class OrderService(AppDbContext db)
{
    public async Task<OneOf<OrderDto, NotFound, ValidationError>> GetOrderAsync(
        int id, CancellationToken ct = default)
    {
        if (id <= 0)
            return new ValidationError("Order ID must be positive.");

        var order = await db.Orders.FindAsync([id], ct);
        if (order is null)
            return new NotFound();

        return new OrderDto(order.Id, order.CustomerId, order.Total, order.CreatedAt);
    }
}

public record ValidationError(string Message);

// Map to HTTP responses
app.MapGet("/orders/{id:int}", async (int id, OrderService orders) =>
{
    var result = await orders.GetOrderAsync(id);
    return result.Match<IResult>(
        order => Results.Ok(order),
        _     => Results.NotFound(),
        error => Results.BadRequest(new { error.Message }));
});
```

## Specification Pattern

The Specification pattern encapsulates query logic into reusable, composable objects for dynamic filtering.

```csharp
public abstract class Specification<T> where T : class
{
    public abstract Expression<Func<T, bool>> ToExpression();

    public Specification<T> And(Specification<T> other)
        => new AndSpecification<T>(this, other);

    public Specification<T> Or(Specification<T> other)
        => new OrSpecification<T>(this, other);

    public Specification<T> Not()
        => new NotSpecification<T>(this);
}

public class AndSpecification<T>(
    Specification<T> left,
    Specification<T> right) : Specification<T> where T : class
{
    public override Expression<Func<T, bool>> ToExpression()
    {
        var leftExpr = left.ToExpression();
        var rightExpr = right.ToExpression();
        var param = Expression.Parameter(typeof(T));
        var body = Expression.AndAlso(
            Expression.Invoke(leftExpr, param),
            Expression.Invoke(rightExpr, param));
        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}

public class OrSpecification<T>(
    Specification<T> left,
    Specification<T> right) : Specification<T> where T : class
{
    public override Expression<Func<T, bool>> ToExpression()
    {
        var leftExpr = left.ToExpression();
        var rightExpr = right.ToExpression();
        var param = Expression.Parameter(typeof(T));
        var body = Expression.OrElse(
            Expression.Invoke(leftExpr, param),
            Expression.Invoke(rightExpr, param));
        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}

public class NotSpecification<T>(
    Specification<T> spec) : Specification<T> where T : class
{
    public override Expression<Func<T, bool>> ToExpression()
    {
        var expr = spec.ToExpression();
        var param = Expression.Parameter(typeof(T));
        var body = Expression.Not(Expression.Invoke(expr, param));
        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}

// Concrete specifications
public class ActiveOrderSpec : Specification<Order>
{
    public override Expression<Func<Order, bool>> ToExpression()
        => order => order.Status != OrderStatus.Cancelled
                  && order.Status != OrderStatus.Completed;
}

public class CustomerOrderSpec(int customerId) : Specification<Order>
{
    public override Expression<Func<Order, bool>> ToExpression()
        => order => order.CustomerId == customerId;
}

public class MinTotalSpec(decimal minTotal) : Specification<Order>
{
    public override Expression<Func<Order, bool>> ToExpression()
        => order => order.Total >= minTotal;
}

// Extension for EF Core IQueryable
public static class SpecificationExtensions
{
    public static IQueryable<T> Where<T>(
        this IQueryable<T> query, Specification<T> spec) where T : class
        => query.Where(spec.ToExpression());
}

// Usage — compose specifications dynamically
public class OrderQueryService(AppDbContext db)
{
    public async Task<IReadOnlyList<Order>> SearchAsync(
        int? customerId, decimal? minTotal, bool activeOnly,
        CancellationToken ct = default)
    {
        IQueryable<Order> query = db.Orders;

        if (customerId.HasValue)
            query = query.Where(new CustomerOrderSpec(customerId.Value));

        if (minTotal.HasValue)
            query = query.Where(new MinTotalSpec(minTotal.Value));

        if (activeOnly)
            query = query.Where(new ActiveOrderSpec());

        return await query.ToListAsync(ct);
    }
}
```

## Pipeline Pattern

The Pipeline pattern processes a request through a series of ordered steps, commonly used as middleware or MediatR behaviors.

```
┌─────────────────────────────────────────────────────────────────┐
│                    MediatR Pipeline                             │
│                                                                 │
│  Request ──▶ ┌────────────┐ ┌────────────┐ ┌────────────┐      │
│              │ Validation  │▶│  Logging   │▶│  Caching   │      │
│              │  Behavior   │ │  Behavior  │ │  Behavior  │      │
│              └────────────┘ └────────────┘ └─────┬──────┘      │
│                                                   │             │
│                                                   ▼             │
│                                            ┌────────────┐       │
│                                            │  Handler   │       │
│                                            │ (Business  │       │
│                                            │  Logic)    │       │
│                                            └─────┬──────┘       │
│                                                   │             │
│  Response ◀─ ┌────────────┐ ┌────────────┐ ┌─────┘             │
│              │ Validation  │◀│  Logging   │◀│                   │
│              │  Behavior   │ │  Behavior  │                     │
│              └────────────┘ └────────────┘                      │
└─────────────────────────────────────────────────────────────────┘
```

### Validation Pipeline Behavior

```csharp
// Install: dotnet add package FluentValidation.DependencyInjectionExtensions

public class ValidationBehavior<TRequest, TResponse>(
    IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        if (!validators.Any())
            return await next(ct);

        var context = new ValidationContext<TRequest>(request);
        var failures = (await Task.WhenAll(
                validators.Select(v => v.ValidateAsync(context, ct))))
            .SelectMany(result => result.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count > 0)
            throw new ValidationException(failures);

        return await next(ct);
    }
}
```

### Logging Pipeline Behavior

```csharp
public class LoggingBehavior<TRequest, TResponse>(
    ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        var requestName = typeof(TRequest).Name;
        logger.LogInformation("Handling {RequestName}: {@Request}", requestName, request);

        var sw = Stopwatch.StartNew();
        var response = await next(ct);
        sw.Stop();

        logger.LogInformation("Handled {RequestName} in {Elapsed}ms",
            requestName, sw.ElapsedMilliseconds);

        return response;
    }
}
```

### Performance Pipeline Behavior

```csharp
public class PerformanceBehavior<TRequest, TResponse>(
    ILogger<PerformanceBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private const int SlowThresholdMs = 500;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        var sw = Stopwatch.StartNew();
        var response = await next(ct);
        sw.Stop();

        if (sw.ElapsedMilliseconds > SlowThresholdMs)
        {
            logger.LogWarning(
                "Slow request: {RequestName} took {Elapsed}ms ({@Request})",
                typeof(TRequest).Name, sw.ElapsedMilliseconds, request);
        }

        return response;
    }
}
```

### Pipeline Registration

```csharp
// Register behaviors — order matters (first registered = outermost)
builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssemblyContaining<Program>();
    cfg.AddOpenBehavior(typeof(LoggingBehavior<,>));
    cfg.AddOpenBehavior(typeof(ValidationBehavior<,>));
    cfg.AddOpenBehavior(typeof(PerformanceBehavior<,>));
});

// Register FluentValidation validators
builder.Services.AddValidatorsFromAssemblyContaining<Program>();
```

### Custom Middleware-Style Pipeline (without MediatR)

```csharp
public delegate Task<TResponse> PipelineDelegate<TResponse>(CancellationToken ct);

public interface IPipelineStep<TRequest, TResponse>
{
    Task<TResponse> ExecuteAsync(
        TRequest request,
        PipelineDelegate<TResponse> next,
        CancellationToken ct = default);
}

public class Pipeline<TRequest, TResponse>
{
    private readonly List<IPipelineStep<TRequest, TResponse>> _steps = [];

    public Pipeline<TRequest, TResponse> Use(IPipelineStep<TRequest, TResponse> step)
    {
        _steps.Add(step);
        return this;
    }

    public Task<TResponse> ExecuteAsync(
        TRequest request,
        Func<TRequest, CancellationToken, Task<TResponse>> handler,
        CancellationToken ct = default)
    {
        PipelineDelegate<TResponse> pipeline = (token) => handler(request, token);

        // Wrap from last to first so the first step executes outermost
        for (var i = _steps.Count - 1; i >= 0; i--)
        {
            var step = _steps[i];
            var next = pipeline;
            pipeline = (token) => step.ExecuteAsync(request, next, token);
        }

        return pipeline(ct);
    }
}
```

## Best Practices

### Pattern Selection Guide

| Pattern | Use When |
|---|---|
| Factory Method | Object creation logic varies by type or configuration |
| Abstract Factory | Families of related objects must be created together |
| Builder | Complex objects with many optional parameters |
| Singleton (DI) | Exactly one instance needed — always use DI registration |
| Adapter | Integrating third-party or legacy code with your interfaces |
| Decorator | Adding cross-cutting concerns (caching, logging, retry) |
| Facade | Simplifying a complex subsystem into a single entry point |
| Strategy | Swappable algorithms selected at runtime |
| Mediator | Decoupling request senders from handlers (CQRS, clean architecture) |
| Chain of Responsibility | Multiple handlers may process a request in sequence |
| CQRS | Read and write workloads have different performance needs |
| Repository | Abstracting data access for testability or store swapping |
| Options | Strongly-typed configuration with validation and reload |
| Result | Representing expected failures without exceptions |
| Specification | Composable, reusable query filters |
| Pipeline | Cross-cutting behavior (validation, logging) around handlers |

### Production Checklist

- ✅ Use DI-managed singletons — never hand-roll `static` singleton patterns
- ✅ Prefer composition over inheritance — use Decorator and Strategy
- ✅ Use MediatR behaviors for cross-cutting concerns instead of duplicating logic
- ✅ Validate commands and queries in pipeline behaviors, not in handlers
- ✅ Keep handlers focused — one handler per command or query
- ✅ Use the Result pattern for expected failures (validation, not-found)
- ✅ Use exceptions only for unexpected failures (network errors, bugs)
- ✅ Register open generics for pipeline behaviors and repositories
- ✅ Use the Specification pattern when query filters are dynamic or reusable
- ✅ Use `IOptionsMonitor<T>` in singletons, `IOptionsSnapshot<T>` in scoped services
- ✅ Validate options at startup with `ValidateOnStart()`
- ✅ Use the Facade pattern to simplify complex multi-service orchestrations

- ❌ Don't apply patterns just because they exist — every pattern adds complexity
- ❌ Don't wrap EF Core in a generic repository if you only have simple CRUD
- ❌ Don't use the Service Locator anti-pattern — always inject dependencies
- ❌ Don't throw exceptions for expected business outcomes (use Result instead)
- ❌ Don't put business logic in pipeline behaviors — keep them cross-cutting only
- ❌ Don't create God handlers that do everything — split by responsibility
- ❌ Don't use CQRS everywhere — it adds complexity; use it when read/write differ

### Common Mistakes

```csharp
// ❌ WRONG — Pattern overuse: unnecessary abstraction over simple operation
public interface IAdditionStrategy
{
    int Add(int a, int b);
}
public class StandardAddition : IAdditionStrategy
{
    public int Add(int a, int b) => a + b;
}

// ✅ CORRECT — Just write the code directly
var sum = a + b;

// ❌ WRONG — Leaking domain logic into pipeline behaviors
public class CreateOrderBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull
{
    public async Task<TResponse> Handle(TRequest request,
        RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        // Business logic in behavior — belongs in the handler
        if (request is CreateOrderCommand cmd && cmd.Items.Count == 0)
            throw new BusinessException("Order must have items");

        return await next(ct);
    }
}

// ✅ CORRECT — Validate shape in behavior, business rules in handler
public class CreateOrderValidator : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderValidator()
    {
        RuleFor(x => x.Items).NotEmpty()
            .WithMessage("Order must have at least one item.");
        RuleFor(x => x.CustomerId).GreaterThan(0);
    }
}
```

## Next Steps

Return to the [Learning Path](LEARNING-PATH.md) for the complete curriculum. Design patterns work best when combined with the dependency injection fundamentals in [Dependency Injection](06-DEPENDENCY-INJECTION.md) and the data access strategies covered in [Entity Framework Core](07-ENTITY-FRAMEWORK-CORE.md).

### Related Documents

1. **[Learning Path](LEARNING-PATH.md)** — Complete .NET learning curriculum
2. **[Dependency Injection](06-DEPENDENCY-INJECTION.md)** — DI container, lifetimes, registration patterns
3. **[Services](02-SERVICES.md)** — Web APIs, worker services, gRPC
4. **[Entity Framework Core](07-ENTITY-FRAMEWORK-CORE.md)** — Data access, migrations, performance
5. **[Middleware and HTTP Pipeline](10-MIDDLEWARE-AND-HTTP-PIPELINE.md)** — ASP.NET Core middleware patterns
6. **[Testing](08-TESTING.md)** — Unit testing, integration testing, mocking

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial design patterns documentation |
