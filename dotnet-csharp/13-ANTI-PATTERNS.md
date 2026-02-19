# .NET and C# Anti-Patterns

## Table of Contents

1. [Overview](#overview)
2. [Async Anti-Patterns](#async-anti-patterns)
3. [DI Anti-Patterns](#di-anti-patterns)
4. [EF Core Anti-Patterns](#ef-core-anti-patterns)
5. [Exception Anti-Patterns](#exception-anti-patterns)
6. [Memory Anti-Patterns](#memory-anti-patterns)
7. [API Design Anti-Patterns](#api-design-anti-patterns)
8. [Architecture Anti-Patterns](#architecture-anti-patterns)
9. [Configuration Anti-Patterns](#configuration-anti-patterns)
10. [Testing Anti-Patterns](#testing-anti-patterns)
11. [Detection and Prevention](#detection-and-prevention)
12. [Next Steps](#next-steps)

## Overview

Anti-patterns are recurring solutions that appear reasonable on the surface but introduce hidden costs — deadlocks, memory leaks, performance degradation, or unmaintainable code. Recognizing and eliminating them early prevents production incidents and reduces long-term technical debt.

### Target Audience

- .NET developers building production applications with ASP.NET Core, EF Core, and background services
- Tech leads and architects performing code reviews and establishing team standards
- Teams migrating legacy .NET Framework codebases to modern .NET 10

### Scope

- Async/await misuse, deadlocks, and thread-pool starvation
- Dependency injection lifetime bugs and Service Locator abuse
- EF Core query performance traps (N+1, full table loads, singleton DbContext)
- Exception handling mistakes that hide bugs or degrade performance
- Memory leaks, allocations, and IDisposable mismanagement
- API design smells in ASP.NET Core controllers and domain models
- Distributed architecture pitfalls (shared databases, chatty services)
- Configuration and secrets management mistakes
- Testing anti-patterns that produce fragile, slow, or misleading test suites
- Static analysis tools and code review checklists for prevention

### Anti-Pattern Identification Workflow

```
┌──────────────────────────────────────────────────────────────────┐
│                  Smell → Diagnosis → Fix Workflow                │
│                                                                  │
│  ┌─────────┐      ┌──────────────┐      ┌───────────────────┐   │
│  │  SMELL  │─────▶│  DIAGNOSIS   │─────▶│       FIX         │   │
│  │         │      │              │      │                   │   │
│  │ Symptom │      │ Root Cause   │      │ Refactored Code   │   │
│  │ noticed │      │ identified   │      │ + Regression Test │   │
│  └─────────┘      └──────────────┘      └───────────────────┘   │
│       │                  │                       │               │
│       ▼                  ▼                       ▼               │
│  ┌─────────┐      ┌──────────────┐      ┌───────────────────┐   │
│  │Examples │      │  Analyzers   │      │  Verify with      │   │
│  │- Timeout│      │- Roslyn      │      │  - Unit tests     │   │
│  │- OOM    │      │- SonarQube   │      │  - Load tests     │   │
│  │- Deadlck│      │- Code review │      │  - Monitoring     │   │
│  │- Slow   │      │- Profilers   │      │  - Profilers      │   │
│  └─────────┘      └──────────────┘      └───────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### Severity Classification

| Severity | Impact | Examples |
|---|---|---|
| 🔴 Critical | Deadlocks, data loss, security vulnerabilities | sync-over-async, secrets in source |
| 🟠 High | Memory leaks, thread-pool starvation, N+1 queries | event handler leaks, singleton DbContext |
| 🟡 Medium | Performance degradation, maintainability issues | string concat in loops, God controllers |
| 🔵 Low | Code smell, minor inefficiency | unnecessary async, over-mocking |

## Async Anti-Patterns

Async/await is the default concurrency model in .NET. Misusing it causes deadlocks, thread-pool starvation, and silent failures.

### Async Void (🔴 Critical)

❌ **Anti-pattern** — `async void` methods swallow exceptions and cannot be awaited:

```csharp
// ❌ WRONG — async void swallows exceptions and crashes the process
public async void SendNotificationAsync(string userId, string message)
{
    var user = await _userRepository.GetByIdAsync(userId);
    await _emailService.SendAsync(user.Email, message);
    // If this throws, the exception is unobserved and may crash the app
}
```

✅ **The fix** — always return `Task` (or `ValueTask`) from async methods:

```csharp
// ✅ CORRECT — returns Task so callers can await and handle exceptions
public async Task SendNotificationAsync(string userId, string message)
{
    var user = await _userRepository.GetByIdAsync(userId);
    await _emailService.SendAsync(user.Email, message);
}
```

**Why it matters** — `async void` methods cannot be awaited, so exceptions propagate to the `SynchronizationContext` or `ThreadPool`, crashing the process. The only valid use of `async void` is event handlers in UI frameworks.

### Sync-Over-Async — .Result / .Wait() (🔴 Critical)

❌ **Anti-pattern** — blocking on async code causes deadlocks and thread-pool starvation:

```csharp
// ❌ WRONG — blocks a thread-pool thread waiting for async work
public OrderDto GetOrder(int orderId)
{
    // Deadlock risk in ASP.NET Core under load; guaranteed starvation at scale
    var order = _orderRepository.GetByIdAsync(orderId).Result;
    return MapToDto(order);
}
```

✅ **The fix** — go async all the way up the call stack:

```csharp
// ✅ CORRECT — async all the way
public async Task<OrderDto> GetOrderAsync(int orderId)
{
    var order = await _orderRepository.GetByIdAsync(orderId);
    return MapToDto(order);
}
```

**Why it matters** — `.Result` and `.Wait()` block the calling thread. Under load, this exhausts the thread pool and causes cascading timeouts. In legacy contexts where sync is required, use `Task.Run(() => AsyncMethod()).GetAwaiter().GetResult()` as a last resort.

### Fire-and-Forget Without Error Handling (🟠 High)

❌ **Anti-pattern** — discarding a task loses exceptions silently:

```csharp
// ❌ WRONG — exception is silently lost
public void ProcessOrder(Order order)
{
    _ = SendConfirmationEmailAsync(order); // fire-and-forget, no error handling
}
```

✅ **The fix** — use a background service or at minimum log errors:

```csharp
// ✅ CORRECT — use a channel or background queue for reliable fire-and-forget
public class EmailBackgroundService(
    Channel<EmailRequest> channel,
    IEmailService emailService,
    ILogger<EmailBackgroundService> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var request in channel.Reader.ReadAllAsync(stoppingToken))
        {
            try
            {
                await emailService.SendAsync(request);
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Failed to send email to {To}", request.To);
            }
        }
    }
}
```

**Why it matters** — unobserved task exceptions are swallowed in .NET. Lost errors make debugging impossible. A background queue provides retries, logging, and graceful shutdown.

### Missing ConfigureAwait in Library Code (🟡 Medium)

❌ **Anti-pattern** — library code capturing the synchronization context unnecessarily:

```csharp
// ❌ WRONG — in library code, capturing context adds overhead
public async Task<byte[]> CompressAsync(Stream input)
{
    using var output = new MemoryStream();
    await input.CopyToAsync(output); // captures SynchronizationContext
    return output.ToArray();
}
```

✅ **The fix** — use `ConfigureAwait(false)` in reusable library code:

```csharp
// ✅ CORRECT — avoids unnecessary context capture in library code
public async Task<byte[]> CompressAsync(Stream input)
{
    using var output = new MemoryStream();
    await input.CopyToAsync(output).ConfigureAwait(false);
    return output.ToArray();
}
```

**Why it matters** — in library code, capturing the synchronization context is unnecessary and can contribute to deadlocks when callers use `.Result`. Application-level ASP.NET Core code does not need `ConfigureAwait(false)` because there is no `SynchronizationContext`.

### Unnecessary Async (🔵 Low)

❌ **Anti-pattern** — wrapping a single await in async/await adds a state machine with no benefit:

```csharp
// ❌ WRONG — unnecessary state machine allocation
public async Task<User> GetUserAsync(int id)
{
    return await _repository.GetByIdAsync(id);
}
```

✅ **The fix** — return the task directly when there is no additional logic:

```csharp
// ✅ CORRECT — pass-through avoids the state machine overhead
public Task<User> GetUserAsync(int id)
{
    return _repository.GetByIdAsync(id);
}
```

**Why it matters** — each `async` method allocates a state machine. For simple pass-through methods, returning the `Task` directly avoids that allocation. However, keep `async`/`await` if you have `using` statements, `try`/`catch`, or multiple awaits — correctness outweighs the micro-optimization.

## DI Anti-Patterns

Dependency injection misuse creates hidden runtime bugs, memory leaks, and tightly coupled code.

### Service Locator (🔴 Critical)

❌ **Anti-pattern** — resolving services manually hides dependencies:

```csharp
// ❌ WRONG — Service Locator hides dependencies and defeats DI
public class OrderService
{
    private readonly IServiceProvider _provider;

    public OrderService(IServiceProvider provider)
    {
        _provider = provider;
    }

    public void ProcessOrder(Order order)
    {
        var repo = _provider.GetRequiredService<IOrderRepository>();
        var email = _provider.GetRequiredService<IEmailService>();
        repo.Save(order);
        email.SendConfirmation(order);
    }
}
```

✅ **The fix** — use constructor injection to declare dependencies explicitly:

```csharp
// ✅ CORRECT — dependencies are visible in the constructor
public class OrderService(
    IOrderRepository repository,
    IEmailService emailService)
{
    public void ProcessOrder(Order order)
    {
        repository.Save(order);
        emailService.SendConfirmation(order);
    }
}
```

**Why it matters** — Service Locator hides what a class depends on, making it impossible to reason about at compile time. Constructor injection makes dependencies explicit, enables `ValidateOnBuild`, and simplifies testing.

### Captive Dependencies — Singleton Holding Scoped (🔴 Critical)

❌ **Anti-pattern** — a singleton captures a scoped service, keeping it alive forever:

```csharp
// ❌ WRONG — singleton captures scoped DbContext, causing stale data and leaks
public class CacheWarmer : IHostedService
{
    private readonly AppDbContext _db; // scoped service held by singleton!

    public CacheWarmer(AppDbContext db) => _db = db;

    public async Task StartAsync(CancellationToken ct)
    {
        var products = await _db.Products.ToListAsync(ct); // stale context
    }

    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

✅ **The fix** — inject `IServiceScopeFactory` and create a scope:

```csharp
// ✅ CORRECT — create a scope to resolve scoped dependencies safely
public class CacheWarmer(IServiceScopeFactory scopeFactory) : IHostedService
{
    public async Task StartAsync(CancellationToken ct)
    {
        await using var scope = scopeFactory.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var products = await db.Products.ToListAsync(ct);
    }

    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

**Why it matters** — a scoped service captured by a singleton never gets disposed, leading to stale data, connection leaks, and memory growth. Enable `ValidateScopes` in development to catch this automatically.

### God Service / Over-Injection (🟠 High)

❌ **Anti-pattern** — a class with too many dependencies does too much:

```csharp
// ❌ WRONG — 8+ dependencies = this class has too many responsibilities
public class OrderProcessor(
    IOrderRepository orderRepo,
    IProductRepository productRepo,
    IInventoryService inventory,
    IPricingService pricing,
    IDiscountService discounts,
    ITaxService tax,
    IEmailService email,
    ISmsService sms,
    IAuditService audit,
    ILogger<OrderProcessor> logger)
{
    // 500+ lines of business logic covering ordering, pricing, comms, audit...
}
```

✅ **The fix** — decompose into focused services following the Single Responsibility Principle:

```csharp
// ✅ CORRECT — each service has a clear, bounded responsibility
public class OrderProcessor(
    IOrderPricingService pricing,
    IOrderFulfillmentService fulfillment,
    IOrderNotificationService notifications,
    ILogger<OrderProcessor> logger)
{
    public async Task ProcessAsync(Order order)
    {
        var pricedOrder = await pricing.CalculateAsync(order);
        await fulfillment.FulfillAsync(pricedOrder);
        await notifications.SendConfirmationsAsync(pricedOrder);
    }
}
```

**Why it matters** — a class with more than 4-5 constructor dependencies is typically violating SRP. It becomes hard to test, hard to reason about, and any change risks breaking unrelated logic. Decompose into smaller, focused services.

### Ambient Context (🟡 Medium)

❌ **Anti-pattern** — static mutable state acting as an invisible dependency:

```csharp
// ❌ WRONG — ambient context via static property
public static class CurrentUser
{
    public static string? UserId { get; set; }
    public static string? TenantId { get; set; }
}

public class AuditService
{
    public void LogAction(string action)
    {
        var userId = CurrentUser.UserId; // hidden dependency on static state
        _repository.Save(new AuditEntry(userId!, action));
    }
}
```

✅ **The fix** — inject context explicitly via a scoped service:

```csharp
// ✅ CORRECT — inject user context through DI
public interface IUserContext
{
    string UserId { get; }
    string TenantId { get; }
}

public class HttpUserContext(IHttpContextAccessor accessor) : IUserContext
{
    public string UserId => accessor.HttpContext!.User.FindFirstValue(ClaimTypes.NameIdentifier)!;
    public string TenantId => accessor.HttpContext!.User.FindFirstValue("tenant_id")!;
}

public class AuditService(IUserContext userContext, IAuditRepository repository)
{
    public void LogAction(string action)
    {
        repository.Save(new AuditEntry(userContext.UserId, action));
    }
}
```

**Why it matters** — ambient context creates hidden coupling, makes testing difficult (you must set/reset static state), and is not thread-safe. Scoped DI services provide the same convenience with explicit, testable contracts.

## EF Core Anti-Patterns

EF Core is a powerful ORM, but common misuse patterns cause severe performance degradation.

### N+1 Queries (🔴 Critical)

❌ **Anti-pattern** — lazy loading or looping with individual queries:

```csharp
// ❌ WRONG — 1 query for orders + N queries for order items
var orders = await _db.Orders.ToListAsync();
foreach (var order in orders)
{
    // Each iteration triggers a separate SQL query
    var items = await _db.OrderItems
        .Where(i => i.OrderId == order.Id)
        .ToListAsync();
    order.Items = items;
}
```

✅ **The fix** — use `Include` for eager loading or a single projection:

```csharp
// ✅ CORRECT — single query with eager loading
var orders = await _db.Orders
    .Include(o => o.Items)
    .ToListAsync();

// ✅ ALSO CORRECT — projection eliminates Include entirely
var orderDtos = await _db.Orders
    .Select(o => new OrderDto
    {
        Id = o.Id,
        Total = o.Total,
        Items = o.Items.Select(i => new OrderItemDto
        {
            ProductName = i.ProductName,
            Quantity = i.Quantity
        }).ToList()
    })
    .ToListAsync();
```

**Why it matters** — N+1 queries scale linearly with data volume. An order list of 1,000 rows generates 1,001 queries. Eager loading or projections collapse this to 1-2 queries.

### Loading Entire Tables (🔴 Critical)

❌ **Anti-pattern** — calling `ToList()` before filtering:

```csharp
// ❌ WRONG — loads every product into memory, then filters in C#
var expensiveProducts = _db.Products
    .ToList()  // SELECT * FROM Products — entire table!
    .Where(p => p.Price > 100)
    .ToList();
```

✅ **The fix** — filter in the query before materializing:

```csharp
// ✅ CORRECT — filter runs in SQL, only matching rows are returned
var expensiveProducts = await _db.Products
    .Where(p => p.Price > 100)
    .ToListAsync();
```

**Why it matters** — loading full tables consumes memory, saturates network bandwidth, and ignores database indexes. For a million-row table, this can cause out-of-memory crashes.

### DbContext as Singleton (🔴 Critical)

❌ **Anti-pattern** — registering DbContext as singleton:

```csharp
// ❌ WRONG — DbContext is NOT thread-safe; singleton causes data corruption
builder.Services.AddSingleton<AppDbContext>();
// or
builder.Services.AddDbContext<AppDbContext>(o =>
    o.UseSqlServer(connectionString), ServiceLifetime.Singleton);
```

✅ **The fix** — use the default scoped lifetime:

```csharp
// ✅ CORRECT — scoped lifetime ensures one DbContext per request
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

// For DbContextFactory in singletons / background services:
builder.Services.AddDbContextFactory<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
```

**Why it matters** — `DbContext` uses an internal identity map and change tracker that are not thread-safe. A singleton `DbContext` causes race conditions, stale data, and `InvalidOperationException` under concurrency.

### No-Tracking Misuse (🟡 Medium)

❌ **Anti-pattern** — using tracking queries for read-only data:

```csharp
// ❌ WRONG — change tracker overhead for read-only display
var products = await _db.Products
    .Include(p => p.Category)
    .ToListAsync(); // tracked by default
```

✅ **The fix** — use `AsNoTracking()` for read-only queries:

```csharp
// ✅ CORRECT — no-tracking skips the change tracker, reducing memory and CPU
var products = await _db.Products
    .AsNoTracking()
    .Include(p => p.Category)
    .ToListAsync();

// ✅ ALSO — set globally for read-heavy apps
protected override void OnConfiguring(DbContextOptionsBuilder options)
{
    options.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
}
```

**Why it matters** — the change tracker adds significant overhead for read-only scenarios. `AsNoTracking()` reduces memory usage and improves query performance, especially for large result sets.

### Missing Indexes (🟠 High)

❌ **Anti-pattern** — querying frequently filtered columns without indexes:

```csharp
// ❌ WRONG — no index on Email causes full table scan
var user = await _db.Users
    .FirstOrDefaultAsync(u => u.Email == email); // WHERE Email = @p0 — table scan
```

✅ **The fix** — add indexes via Fluent API or data annotations:

```csharp
// ✅ CORRECT — index on frequently queried columns
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.HasIndex(u => u.Email).IsUnique();
        builder.HasIndex(u => u.TenantId);
        builder.HasIndex(u => new { u.TenantId, u.CreatedAt }); // composite
    }
}
```

**Why it matters** — missing indexes cause full table scans that degrade exponentially with data growth. Always analyze query plans and add indexes for `WHERE`, `JOIN`, and `ORDER BY` columns.

### Lazy Loading Pitfalls (🟠 High)

❌ **Anti-pattern** — enabling lazy loading without awareness of query count:

```csharp
// ❌ WRONG — lazy loading fires queries inside serialization or loops
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseLazyLoadingProxies()
           .UseSqlServer(connectionString));

// In a controller:
var orders = await _db.Orders.ToListAsync();
return Ok(orders); // Serialization touches .Items, .Customer, etc. — N+1!
```

✅ **The fix** — use explicit eager loading or projections:

```csharp
// ✅ CORRECT — explicit loading makes queries predictable
var orders = await _db.Orders
    .Include(o => o.Items)
    .Include(o => o.Customer)
    .AsSplitQuery()
    .ToListAsync();
```

**Why it matters** — lazy loading hides the number of queries executed. In serialization scenarios (JSON, gRPC), every navigation property access fires a separate query, causing massive N+1 problems that are invisible in unit tests.

## Exception Anti-Patterns

Exception handling mistakes hide bugs, degrade performance, and make debugging painful.

### Catching System.Exception (🟠 High)

❌ **Anti-pattern** — catching the base `Exception` type hides specific errors:

```csharp
// ❌ WRONG — catches everything including OutOfMemoryException, StackOverflowException
try
{
    await ProcessOrderAsync(order);
}
catch (Exception ex)
{
    _logger.LogError(ex, "Something went wrong");
    return StatusCode(500);
}
```

✅ **The fix** — catch specific exception types and use global exception handling:

```csharp
// ✅ CORRECT — catch specific exceptions, let unexpected ones propagate
try
{
    await ProcessOrderAsync(order);
}
catch (OrderValidationException ex)
{
    _logger.LogWarning(ex, "Order validation failed for {OrderId}", order.Id);
    return BadRequest(ex.Errors);
}
catch (InventoryUnavailableException ex)
{
    _logger.LogWarning(ex, "Inventory unavailable for {OrderId}", order.Id);
    return Conflict(new { message = ex.Message });
}
// Unexpected exceptions propagate to global exception handler (middleware)
```

**Why it matters** — catching `System.Exception` swallows critical runtime exceptions and prevents callers from distinguishing recoverable errors from fatal ones. Use ASP.NET Core's `IExceptionHandler` (.NET 8+) for global fallback handling.

### Swallowing Exceptions (🔴 Critical)

❌ **Anti-pattern** — catching and ignoring exceptions silently:

```csharp
// ❌ WRONG — exception is completely swallowed
try
{
    await _paymentService.ChargeAsync(order);
}
catch (Exception)
{
    // Silently swallowed — payment failure goes unnoticed
}
```

✅ **The fix** — always log, rethrow, or convert to a meaningful result:

```csharp
// ✅ CORRECT — log and propagate the failure
try
{
    await _paymentService.ChargeAsync(order);
}
catch (PaymentGatewayException ex)
{
    _logger.LogError(ex, "Payment failed for order {OrderId}", order.Id);
    throw; // re-throw preserving stack trace
}
```

**Why it matters** — swallowed exceptions cause silent data corruption, lost transactions, and impossible-to-diagnose production issues. Every caught exception should be logged, rethrown, or converted to an explicit error result.

### Exceptions for Flow Control (🟡 Medium)

❌ **Anti-pattern** — using exceptions for expected conditions:

```csharp
// ❌ WRONG — exceptions are expensive; this is a normal control flow
public decimal GetDiscount(string promoCode)
{
    try
    {
        return _promoRepository.GetByCode(promoCode).DiscountPercent;
    }
    catch (NotFoundException)
    {
        return 0m; // promo not found is an expected case, not exceptional
    }
}
```

✅ **The fix** — use conditional logic or TryGet patterns:

```csharp
// ✅ CORRECT — check for null instead of catching exceptions
public decimal GetDiscount(string promoCode)
{
    var promo = _promoRepository.FindByCode(promoCode);
    return promo?.DiscountPercent ?? 0m;
}

// ✅ ALSO — TryGet pattern for parse/lookup operations
public bool TryGetDiscount(string promoCode, out decimal discount)
{
    var promo = _promoRepository.FindByCode(promoCode);
    discount = promo?.DiscountPercent ?? 0m;
    return promo is not null;
}
```

**Why it matters** — throwing and catching exceptions is orders of magnitude slower than conditional checks. In hot paths, exception-based flow control creates measurable performance degradation and clutters logs.

### throw ex — Losing the Stack Trace (🟠 High)

❌ **Anti-pattern** — `throw ex` resets the stack trace:

```csharp
// ❌ WRONG — throw ex resets the stack trace to this catch block
try
{
    await _repository.SaveAsync(entity);
}
catch (DbUpdateException ex)
{
    _logger.LogError(ex, "Database error");
    throw ex; // stack trace starts HERE, not at the original failure
}
```

✅ **The fix** — use `throw` to preserve the original stack trace:

```csharp
// ✅ CORRECT — throw preserves the full stack trace
try
{
    await _repository.SaveAsync(entity);
}
catch (DbUpdateException ex)
{
    _logger.LogError(ex, "Database error saving {Entity}", entity.GetType().Name);
    throw; // original stack trace preserved
}

// ✅ ALSO — wrap in a domain exception with InnerException
catch (DbUpdateException ex)
{
    throw new RepositoryException("Failed to save entity", ex); // ex is InnerException
}
```

**Why it matters** — `throw ex` discards the original stack trace, making it impossible to find where the error actually originated. Always use bare `throw;` or wrap in a new exception with the original as `InnerException`.

## Memory Anti-Patterns

Memory mismanagement in .NET causes GC pressure, large object heap (LOH) fragmentation, and leaks in long-running services.

### String Concatenation in Loops (🟠 High)

❌ **Anti-pattern** — concatenating strings in a loop creates many intermediate allocations:

```csharp
// ❌ WRONG — each += allocates a new string; O(n²) allocation pattern
string csv = "";
foreach (var item in items)
{
    csv += item.Name + "," + item.Price + "\n"; // new string every iteration
}
```

✅ **The fix** — use `StringBuilder` or `string.Join`:

```csharp
// ✅ CORRECT — StringBuilder amortizes allocations
var sb = new StringBuilder(items.Count * 50); // estimate capacity
foreach (var item in items)
{
    sb.Append(item.Name).Append(',').Append(item.Price).AppendLine();
}
string csv = sb.ToString();

// ✅ ALSO — string.Join for simple cases
string csv = string.Join('\n', items.Select(i => $"{i.Name},{i.Price}"));
```

**Why it matters** — string concatenation in loops creates O(n) temporary strings, each copied to a new allocation. For 10,000 items, this means ~50 million characters copied. `StringBuilder` reduces this to amortized O(n).

### Large Object Heap Fragmentation (🟠 High)

❌ **Anti-pattern** — repeatedly allocating and discarding large arrays (≥85KB):

```csharp
// ❌ WRONG — allocates a new 1MB buffer on the LOH for each request
public async Task<byte[]> ProcessFileAsync(Stream input)
{
    var buffer = new byte[1_048_576]; // 1MB — goes on LOH
    int bytesRead = await input.ReadAsync(buffer);
    return buffer[..bytesRead];
}
```

✅ **The fix** — use `ArrayPool<T>` to rent and return buffers:

```csharp
// ✅ CORRECT — pool large arrays to avoid LOH fragmentation
public async Task<byte[]> ProcessFileAsync(Stream input)
{
    var buffer = ArrayPool<byte>.Shared.Rent(1_048_576);
    try
    {
        int bytesRead = await input.ReadAsync(buffer);
        return buffer[..bytesRead]; // allocates small result array
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

**Why it matters** — the LOH is not compacted by default (only during full GC with compaction enabled). Frequent large allocations fragment the LOH, increasing memory usage and GC pause times. `ArrayPool<T>` eliminates repeated allocations.

### Event Handler Leaks (🟠 High)

❌ **Anti-pattern** — subscribing to events without unsubscribing:

```csharp
// ❌ WRONG — subscriber is never unsubscribed; publisher holds a reference forever
public class DashboardViewModel
{
    public DashboardViewModel(OrderService orderService)
    {
        orderService.OrderPlaced += OnOrderPlaced; // leaked reference
    }

    private void OnOrderPlaced(object? sender, OrderEventArgs e)
    {
        RefreshDashboard();
    }
}
```

✅ **The fix** — unsubscribe in `Dispose` or use weak event patterns:

```csharp
// ✅ CORRECT — unsubscribe when done
public class DashboardViewModel : IDisposable
{
    private readonly OrderService _orderService;

    public DashboardViewModel(OrderService orderService)
    {
        _orderService = orderService;
        _orderService.OrderPlaced += OnOrderPlaced;
    }

    private void OnOrderPlaced(object? sender, OrderEventArgs e)
    {
        RefreshDashboard();
    }

    public void Dispose()
    {
        _orderService.OrderPlaced -= OnOrderPlaced;
    }
}
```

**Why it matters** — event handlers create strong references from publisher to subscriber. If the publisher outlives the subscriber, the subscriber is never garbage collected. In long-running services, this leaks memory proportional to the number of subscriptions.

### Improper IDisposable (🟠 High)

❌ **Anti-pattern** — not disposing resources or not implementing `IDisposable` correctly:

```csharp
// ❌ WRONG — HttpClient is never disposed; socket exhaustion under load
public class ApiClient
{
    public async Task<string> FetchDataAsync(string url)
    {
        var client = new HttpClient(); // new instance per call!
        var response = await client.GetStringAsync(url);
        return response; // client never disposed — socket leak
    }
}
```

✅ **The fix** — use `IHttpClientFactory` or manage lifetime properly:

```csharp
// ✅ CORRECT — IHttpClientFactory manages HttpClient lifetime and pooling
public class ApiClient(IHttpClientFactory httpClientFactory)
{
    public async Task<string> FetchDataAsync(string url)
    {
        using var client = httpClientFactory.CreateClient();
        return await client.GetStringAsync(url);
    }
}

// Registration
builder.Services.AddHttpClient();
```

**Why it matters** — `HttpClient` holds TCP connections. Creating and discarding instances without disposal causes socket exhaustion (`SocketException: Address already in use`). `IHttpClientFactory` pools `HttpMessageHandler` instances and rotates DNS.

### Closure Allocations in Hot Paths (🟡 Medium)

❌ **Anti-pattern** — lambda closures allocating on every invocation:

```csharp
// ❌ WRONG — closure captures 'minPrice', allocating a delegate + closure object each call
public List<Product> FilterProducts(List<Product> products, decimal minPrice)
{
    return products.Where(p => p.Price > minPrice).ToList(); // closure allocation
}
```

✅ **The fix** — use static lambdas or pass state explicitly in hot paths:

```csharp
// ✅ CORRECT — for hot paths, avoid closures with local functions or explicit state
public List<Product> FilterProducts(List<Product> products, decimal minPrice)
{
    var result = new List<Product>(products.Count);
    foreach (var product in products)
    {
        if (product.Price > minPrice)
            result.Add(product);
    }
    return result;
}

// ✅ ALSO — in .NET 9+, LINQ has better allocation patterns,
// but for extreme hot paths, manual iteration is still faster.
```

**Why it matters** — each lambda capturing a local variable allocates a closure object and a delegate on the heap. In tight loops processing millions of items, this creates significant GC pressure. Profile before optimizing — most code paths do not need this level of attention.

## API Design Anti-Patterns

ASP.NET Core API design mistakes produce bloated, inconsistent, and insecure endpoints.

### God Controllers (🟠 High)

❌ **Anti-pattern** — a single controller handling dozens of unrelated actions:

```csharp
// ❌ WRONG — 30+ actions covering orders, products, users, reports
[ApiController]
[Route("api/[controller]")]
public class AdminController : ControllerBase
{
    [HttpGet("orders")] public Task<IActionResult> GetOrders() { ... }
    [HttpPost("orders")] public Task<IActionResult> CreateOrder() { ... }
    [HttpGet("products")] public Task<IActionResult> GetProducts() { ... }
    [HttpPost("users")] public Task<IActionResult> CreateUser() { ... }
    [HttpGet("reports/sales")] public Task<IActionResult> SalesReport() { ... }
    [HttpGet("reports/inventory")] public Task<IActionResult> InventoryReport() { ... }
    // ... 25 more actions
}
```

✅ **The fix** — split into focused controllers by resource:

```csharp
// ✅ CORRECT — one controller per resource/aggregate
[ApiController]
[Route("api/orders")]
public class OrdersController(IOrderService orderService) : ControllerBase
{
    [HttpGet] public Task<IActionResult> GetAll() => ...;
    [HttpGet("{id:int}")] public Task<IActionResult> GetById(int id) => ...;
    [HttpPost] public Task<IActionResult> Create(CreateOrderRequest request) => ...;
}

[ApiController]
[Route("api/reports")]
public class ReportsController(IReportService reportService) : ControllerBase
{
    [HttpGet("sales")] public Task<IActionResult> Sales() => ...;
    [HttpGet("inventory")] public Task<IActionResult> Inventory() => ...;
}
```

**Why it matters** — God controllers violate SRP, make navigation difficult, and create merge conflicts in team environments. They also accumulate excessive constructor dependencies.

### Anemic Domain Models (🟡 Medium)

❌ **Anti-pattern** — domain entities are just data bags with no behavior:

```csharp
// ❌ WRONG — entity is a dumb data container; all logic in services
public class Order
{
    public int Id { get; set; }
    public List<OrderItem> Items { get; set; } = [];
    public decimal Total { get; set; }
    public OrderStatus Status { get; set; }
}

// All logic lives in a service — the domain model is "anemic"
public class OrderService
{
    public void AddItem(Order order, Product product, int qty)
    {
        order.Items.Add(new OrderItem { ProductId = product.Id, Quantity = qty });
        order.Total = order.Items.Sum(i => i.Quantity * i.UnitPrice);
    }
}
```

✅ **The fix** — encapsulate behavior inside the domain entity:

```csharp
// ✅ CORRECT — entity owns its invariants and behavior
public class Order
{
    public int Id { get; private set; }
    private readonly List<OrderItem> _items = [];
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
    public decimal Total { get; private set; }
    public OrderStatus Status { get; private set; }

    public void AddItem(Product product, int quantity)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(quantity);
        _items.Add(new OrderItem(product.Id, product.Price, quantity));
        Total = _items.Sum(i => i.Quantity * i.UnitPrice);
    }
}
```

**Why it matters** — anemic models scatter business rules across services, leading to duplication and inconsistency. Rich domain models enforce invariants at the entity level, making illegal states unrepresentable.

### Over-Posting (🔴 Critical)

❌ **Anti-pattern** — binding directly to domain entities allows mass assignment:

```csharp
// ❌ WRONG — client can set IsAdmin, Id, or any other property
[HttpPost]
public async Task<IActionResult> CreateUser([FromBody] User user)
{
    // Client sends: { "name": "Alice", "isAdmin": true } — privilege escalation!
    await _db.Users.AddAsync(user);
    await _db.SaveChangesAsync();
    return Ok(user);
}
```

✅ **The fix** — use dedicated request DTOs:

```csharp
// ✅ CORRECT — DTO limits what the client can set
public record CreateUserRequest(string Name, string Email);

[HttpPost]
public async Task<IActionResult> CreateUser([FromBody] CreateUserRequest request)
{
    var user = new User
    {
        Name = request.Name,
        Email = request.Email,
        IsAdmin = false, // server-controlled
        CreatedAt = DateTime.UtcNow
    };
    await _db.Users.AddAsync(user);
    await _db.SaveChangesAsync();
    return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
}
```

**Why it matters** — over-posting allows attackers to set fields like `IsAdmin`, `Price`, or `Balance` by including them in the JSON body. Request DTOs act as a whitelist, exposing only the fields the client should provide.

### Inconsistent Error Responses (🟡 Medium)

❌ **Anti-pattern** — each endpoint returns errors in a different shape:

```csharp
// ❌ WRONG — every action invents its own error format
[HttpGet("{id}")]
public IActionResult Get(int id)
{
    var order = _repo.Find(id);
    if (order is null)
        return NotFound(new { error = "not found" }); // shape 1
    return Ok(order);
}

[HttpPost]
public IActionResult Create(OrderRequest req)
{
    if (!ModelState.IsValid)
        return BadRequest(ModelState); // shape 2 — different structure
    // ...
}
```

✅ **The fix** — use RFC 9457 Problem Details consistently:

```csharp
// ✅ CORRECT — standardized problem details across all endpoints
builder.Services.AddProblemDetails();

// In Program.cs
app.UseExceptionHandler();
app.UseStatusCodePages();

// In controllers, use TypedResults or Problem():
[HttpGet("{id}")]
public async Task<Results<Ok<OrderDto>, NotFound<ProblemDetails>>> Get(int id)
{
    var order = await _repo.FindAsync(id);
    if (order is null)
        return TypedResults.NotFound(new ProblemDetails
        {
            Title = "Order not found",
            Detail = $"No order with ID {id} exists",
            Status = 404
        });
    return TypedResults.Ok(MapToDto(order));
}
```

**Why it matters** — inconsistent error formats force clients to handle multiple shapes, increasing coupling and bugs. RFC 9457 (Problem Details) provides a standard structure that tools and clients can parse uniformly.

### Missing Validation (🟠 High)

❌ **Anti-pattern** — no input validation, trusting client data:

```csharp
// ❌ WRONG — no validation; negative quantities, empty names accepted
[HttpPost]
public async Task<IActionResult> CreateProduct([FromBody] CreateProductRequest request)
{
    var product = new Product { Name = request.Name, Price = request.Price };
    await _db.Products.AddAsync(product);
    await _db.SaveChangesAsync();
    return Ok(product);
}
```

✅ **The fix** — validate with data annotations or FluentValidation:

```csharp
// ✅ CORRECT — validate input before processing
public record CreateProductRequest
{
    [Required, StringLength(200, MinimumLength = 1)]
    public required string Name { get; init; }

    [Range(0.01, 1_000_000)]
    public required decimal Price { get; init; }
}

// In Program.cs (.NET 10+)
builder.Services.AddValidation();

[HttpPost]
public async Task<IActionResult> CreateProduct(
    [FromBody] CreateProductRequest request)
{
    // ModelState is automatically validated by the framework
    var product = new Product { Name = request.Name, Price = request.Price };
    await _db.Products.AddAsync(product);
    await _db.SaveChangesAsync();
    return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
}
```

**Why it matters** — unvalidated input leads to data corruption, injection attacks, and cryptic downstream exceptions. Validation at the API boundary is the first line of defense.

## Architecture Anti-Patterns

Architectural anti-patterns manifest at the system level, causing coupling, latency, and deployment bottlenecks.

### Distributed Monolith (🔴 Critical)

```
┌────────────────────────────────────────────────────────────────┐
│                    Distributed Monolith                         │
│                                                                │
│  ┌─────────┐  sync  ┌─────────┐  sync  ┌─────────┐           │
│  │Order Svc│───────▶│Stock Svc│───────▶│Price Svc│           │
│  └────┬────┘        └────┬────┘        └────┬────┘           │
│       │                  │                  │                 │
│       ▼                  ▼                  ▼                 │
│  All services must be deployed together                       │
│  All services must be running for any request to succeed      │
│  Synchronous chain: latency = sum of all service latencies    │
│                                                                │
│  Same coupling as a monolith + network unreliability           │
└────────────────────────────────────────────────────────────────┘
```

❌ **Anti-pattern** — microservices that are tightly coupled via synchronous calls:

```csharp
// ❌ WRONG — synchronous chain; if any service is down, everything fails
public class OrderService(HttpClient stockClient, HttpClient priceClient)
{
    public async Task<OrderResult> PlaceOrderAsync(OrderRequest request)
    {
        var stock = await stockClient.GetFromJsonAsync<StockResponse>(
            $"/api/stock/{request.ProductId}");
        var price = await priceClient.GetFromJsonAsync<PriceResponse>(
            $"/api/price/{request.ProductId}");
        // ...
    }
}
```

✅ **The fix** — use async messaging for non-critical paths and data duplication for reads:

```csharp
// ✅ CORRECT — publish events; downstream services react asynchronously
public class OrderService(
    IOrderRepository repository,
    IMessagePublisher publisher)
{
    public async Task<OrderResult> PlaceOrderAsync(OrderRequest request)
    {
        // Use local data (cached/replicated) for price and stock checks
        var order = Order.Create(request);
        await repository.SaveAsync(order);

        // Publish event — downstream services handle fulfillment asynchronously
        await publisher.PublishAsync(new OrderPlacedEvent(order.Id, order.Items));
        return new OrderResult(order.Id, OrderStatus.Accepted);
    }
}
```

**Why it matters** — a distributed monolith has all the deployment complexity of microservices with none of the independence. Use async events, local caches, and data duplication to decouple services.

### Shared Database Between Services (🔴 Critical)

❌ **Anti-pattern** — multiple services reading/writing the same database tables:

```csharp
// ❌ WRONG — Order service directly queries Product tables owned by Product service
public class OrderService(AppDbContext db)
{
    public async Task<OrderDto> PlaceOrder(OrderRequest request)
    {
        // Directly accessing another service's table
        var product = await db.Products.FindAsync(request.ProductId);
        var order = new Order { ProductName = product!.Name, Price = product.Price };
        db.Orders.Add(order);
        await db.SaveChangesAsync();
        return MapToDto(order);
    }
}
```

✅ **The fix** — each service owns its data; communicate via APIs or events:

```csharp
// ✅ CORRECT — Order service stores its own product snapshot
public class OrderService(
    IOrderRepository orderRepo,
    IProductApiClient productClient)
{
    public async Task<OrderDto> PlaceOrder(OrderRequest request)
    {
        var product = await productClient.GetProductAsync(request.ProductId);
        var order = Order.Create(product.Name, product.Price, request.Quantity);
        await orderRepo.SaveAsync(order);
        return MapToDto(order);
    }
}
```

**Why it matters** — shared databases create hidden coupling. Schema changes in one service break another. Services cannot be deployed, scaled, or migrated independently. Each service should own its data store.

### Tight Coupling Through Shared Libraries (🟡 Medium)

❌ **Anti-pattern** — a shared NuGet package containing domain models used by all services:

```
❌ WRONG — single shared library couples all services
SharedModels.csproj
├── Order.cs           ← used by Order, Shipping, Billing services
├── Product.cs         ← used by Catalog, Order, Inventory services
├── Customer.cs        ← used by Auth, Order, CRM services
└── ...50 more models
```

✅ **The fix** — share contracts (interfaces, DTOs, events) only; each service owns its domain:

```
✅ CORRECT — share only integration contracts
Contracts.Orders.csproj         ← OrderPlacedEvent, OrderDto (integration schemas)
Contracts.Products.csproj       ← ProductCreatedEvent, ProductDto

Order.Service/
├── Domain/Order.cs             ← service-specific domain model
Catalog.Service/
├── Domain/Product.cs           ← service-specific domain model
```

**Why it matters** — a monolithic shared library forces coordinated releases. Changing a single model requires redeploying every consuming service. Share only integration events and DTOs; keep domain models private to each service.

### Chatty Services (🟠 High)

❌ **Anti-pattern** — making many fine-grained HTTP calls for a single operation:

```csharp
// ❌ WRONG — 4 HTTP calls to render a single order page
var order = await _orderClient.GetOrderAsync(orderId);
var customer = await _customerClient.GetCustomerAsync(order.CustomerId);
var items = await _catalogClient.GetProductsAsync(order.ProductIds);
var shipping = await _shippingClient.GetTrackingAsync(orderId);
```

✅ **The fix** — use the BFF (Backend-for-Frontend) pattern or batch APIs:

```csharp
// ✅ CORRECT — BFF aggregates data in a single endpoint
[HttpGet("orders/{id}/summary")]
public async Task<OrderSummaryDto> GetOrderSummary(int id)
{
    // Parallel calls within the BFF, single response to client
    var orderTask = _orderClient.GetOrderAsync(id);
    var trackingTask = _shippingClient.GetTrackingAsync(id);
    await Task.WhenAll(orderTask, trackingTask);

    return new OrderSummaryDto
    {
        Order = orderTask.Result,
        Tracking = trackingTask.Result
    };
}
```

**Why it matters** — chatty services increase latency (sum of all round trips), amplify failure probability, and waste bandwidth. Aggregate related data server-side and return composite responses.

## Configuration Anti-Patterns

Configuration mistakes cause security incidents, deployment failures, and environment-specific bugs.

### Hardcoded Values (🟡 Medium)

❌ **Anti-pattern** — magic strings and numbers embedded in code:

```csharp
// ❌ WRONG — hardcoded connection string and timeouts
public class OrderRepository
{
    private readonly string _connectionString =
        "Server=sql-prod;Database=Orders;User=sa;Password=P@ssw0rd!";

    public async Task<Order?> GetByIdAsync(int id)
    {
        using var conn = new SqlConnection(_connectionString);
        conn.Open();
        // timeout hardcoded to 30 — can't change without redeployment
        using var cmd = new SqlCommand("SELECT ...", conn) { CommandTimeout = 30 };
        // ...
    }
}
```

✅ **The fix** — use the Options pattern with configuration binding:

```csharp
// ✅ CORRECT — externalized configuration via Options pattern
public class OrderDatabaseOptions
{
    public required string ConnectionString { get; init; }
    public int CommandTimeoutSeconds { get; init; } = 30;
}

builder.Services.AddOptions<OrderDatabaseOptions>()
    .BindConfiguration("OrderDatabase")
    .ValidateDataAnnotations()
    .ValidateOnStart();

public class OrderRepository(IOptions<OrderDatabaseOptions> options)
{
    public async Task<Order?> GetByIdAsync(int id)
    {
        using var conn = new SqlConnection(options.Value.ConnectionString);
        // ...
    }
}
```

**Why it matters** — hardcoded values require recompilation to change, risk exposing credentials in source, and prevent per-environment tuning. The Options pattern supports validation, hot reload, and environment-specific overrides.

### Secrets in Source Code (🔴 Critical)

❌ **Anti-pattern** — API keys and passwords in `appsettings.json` or source code:

```jsonc
// ❌ WRONG — secrets committed to source control
// appsettings.json
{
    "Stripe": {
        "SecretKey": "sk_live_abc123xyz789..."
    },
    "Database": {
        "Password": "SuperSecret123!"
    }
}
```

✅ **The fix** — use Secret Manager (dev), environment variables, or Azure Key Vault (prod):

```csharp
// ✅ CORRECT — secrets loaded from secure providers, never in source
// Development: User Secrets
// dotnet user-secrets set "Stripe:SecretKey" "sk_live_abc123xyz789..."

// Production: Azure Key Vault or environment variables
builder.Configuration.AddAzureKeyVault(
    new Uri("https://myapp-vault.vault.azure.net/"),
    new DefaultAzureCredential());

// Access via Options pattern — same code, different source per environment
builder.Services.AddOptions<StripeOptions>()
    .BindConfiguration("Stripe")
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

**Why it matters** — secrets in source code are the #1 cause of credential leaks. Once committed, they persist in git history even after deletion. Use secure vaults and rotate credentials regularly.

### Missing Configuration Validation (🟠 High)

❌ **Anti-pattern** — configuration read at runtime without validation:

```csharp
// ❌ WRONG — missing or malformed config causes runtime NullReferenceException
public class EmailService(IConfiguration config)
{
    public async Task SendAsync(string to, string body)
    {
        var smtpHost = config["Email:SmtpHost"]; // null if missing!
        var port = int.Parse(config["Email:Port"]!); // FormatException if invalid
        // ...
    }
}
```

✅ **The fix** — validate options eagerly at startup:

```csharp
// ✅ CORRECT — fail fast at startup if config is invalid
public class EmailOptions
{
    [Required]
    public required string SmtpHost { get; init; }

    [Range(1, 65535)]
    public int Port { get; init; } = 587;

    [Required, EmailAddress]
    public required string FromAddress { get; init; }
}

builder.Services.AddOptions<EmailOptions>()
    .BindConfiguration("Email")
    .ValidateDataAnnotations()
    .ValidateOnStart(); // throws at startup if config is invalid

public class EmailService(IOptions<EmailOptions> options)
{
    public async Task SendAsync(string to, string body)
    {
        var host = options.Value.SmtpHost; // guaranteed non-null
        // ...
    }
}
```

**Why it matters** — without validation, configuration errors surface as runtime exceptions minutes or hours after deployment. `ValidateOnStart()` catches missing or invalid values immediately during startup, before serving traffic.

### Environment-Specific Code (🟡 Medium)

❌ **Anti-pattern** — branching on environment names throughout the codebase:

```csharp
// ❌ WRONG — environment checks scattered across the application
public class PaymentService(IWebHostEnvironment env)
{
    public async Task<PaymentResult> ChargeAsync(decimal amount)
    {
        if (env.IsDevelopment())
        {
            return new PaymentResult { Success = true }; // fake payment
        }
        else if (env.EnvironmentName == "Staging")
        {
            // use sandbox API
            return await _sandboxGateway.ChargeAsync(amount);
        }
        else
        {
            return await _liveGateway.ChargeAsync(amount);
        }
    }
}
```

✅ **The fix** — use DI to swap implementations per environment:

```csharp
// ✅ CORRECT — register environment-specific implementations via DI
public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(decimal amount);
}

// Program.cs
if (builder.Environment.IsDevelopment())
    builder.Services.AddSingleton<IPaymentGateway, FakePaymentGateway>();
else
    builder.Services.AddSingleton<IPaymentGateway, StripePaymentGateway>();

// Service code is environment-agnostic
public class PaymentService(IPaymentGateway gateway)
{
    public Task<PaymentResult> ChargeAsync(decimal amount)
        => gateway.ChargeAsync(amount);
}
```

**Why it matters** — environment-specific branching scatters deployment logic, makes testing unreliable (tests run in a different environment), and risks accidental execution of dev code in production. Use DI or configuration to swap behavior.

## Testing Anti-Patterns

Testing anti-patterns produce suites that are slow, brittle, misleading, or provide false confidence.

### Testing Implementation Details (🟠 High)

❌ **Anti-pattern** — testing internal method calls instead of observable behavior:

```csharp
// ❌ WRONG — tests break when refactoring internals, even if behavior is correct
[Fact]
public async Task PlaceOrder_CallsRepositorySaveThenPublishesEvent()
{
    var mockRepo = new Mock<IOrderRepository>();
    var mockPublisher = new Mock<IEventPublisher>();
    var service = new OrderService(mockRepo.Object, mockPublisher.Object);

    await service.PlaceOrderAsync(new OrderRequest("SKU-1", 2));

    // Verifies call ORDER — brittle; breaks if internals are reordered
    var sequence = new MockSequence();
    mockRepo.InSequence(sequence).Verify(r => r.SaveAsync(It.IsAny<Order>()), Times.Once);
    mockPublisher.InSequence(sequence).Verify(p => p.PublishAsync(It.IsAny<OrderPlacedEvent>()), Times.Once);
}
```

✅ **The fix** — test observable outcomes:

```csharp
// ✅ CORRECT — test the result and side effects, not the internal call order
[Fact]
public async Task PlaceOrder_ReturnsAcceptedStatus_And_PersistsOrder()
{
    var repo = new InMemoryOrderRepository();
    var publisher = new InMemoryEventPublisher();
    var service = new OrderService(repo, publisher);

    var result = await service.PlaceOrderAsync(new OrderRequest("SKU-1", 2));

    Assert.Equal(OrderStatus.Accepted, result.Status);
    Assert.Single(repo.SavedOrders);
    Assert.Single(publisher.PublishedEvents.OfType<OrderPlacedEvent>());
}
```

**Why it matters** — tests coupled to implementation break on every refactor, discouraging improvement. Tests should verify what a component does, not how it does it.

### Brittle Tests (🟠 High)

❌ **Anti-pattern** — tests that fail due to unrelated changes:

```csharp
// ❌ WRONG — test depends on exact JSON string, date format, and ordering
[Fact]
public void Serialize_Order_Returns_ExactJson()
{
    var order = new Order { Id = 1, Total = 99.99m, CreatedAt = new DateTime(2025, 1, 1) };

    var json = JsonSerializer.Serialize(order);

    Assert.Equal(
        "{\"Id\":1,\"Total\":99.99,\"CreatedAt\":\"2025-01-01T00:00:00\"}",
        json); // breaks if a property is added or serialization options change
}
```

✅ **The fix** — assert on individual properties or use semantic comparison:

```csharp
// ✅ CORRECT — assert on meaningful properties, not string representation
[Fact]
public void Serialize_And_Deserialize_Order_RoundTrips()
{
    var order = new Order { Id = 1, Total = 99.99m, CreatedAt = new DateTime(2025, 1, 1) };

    var json = JsonSerializer.Serialize(order);
    var deserialized = JsonSerializer.Deserialize<Order>(json);

    Assert.NotNull(deserialized);
    Assert.Equal(order.Id, deserialized.Id);
    Assert.Equal(order.Total, deserialized.Total);
    Assert.Equal(order.CreatedAt, deserialized.CreatedAt);
}
```

**Why it matters** — brittle tests fail for the wrong reasons, erode team confidence in the test suite, and slow down development as developers constantly fix unrelated test failures.

### No Integration Tests (🟠 High)

❌ **Anti-pattern** — only unit tests with mocked dependencies:

```csharp
// ❌ WRONG — 100% mock-based; proves nothing about real database/API behavior
[Fact]
public async Task GetUser_ReturnsUser()
{
    var mock = new Mock<IUserRepository>();
    mock.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(new User { Id = 1, Name = "Alice" });

    var service = new UserService(mock.Object);
    var result = await service.GetByIdAsync(1);

    Assert.Equal("Alice", result.Name);
    // But does the real SQL query work? Does EF mapping work? We don't know.
}
```

✅ **The fix** — add integration tests with `WebApplicationFactory`:

```csharp
// ✅ CORRECT — integration test exercises real middleware, DI, and database
public class UsersApiTests(WebApplicationFactory<Program> factory)
    : IClassFixture<WebApplicationFactory<Program>>
{
    [Fact]
    public async Task GetUser_ReturnsOk_WithUserData()
    {
        var client = factory.CreateClient();

        var response = await client.GetAsync("/api/users/1");

        response.EnsureSuccessStatusCode();
        var user = await response.Content.ReadFromJsonAsync<UserDto>();
        Assert.NotNull(user);
        Assert.Equal(1, user.Id);
    }
}
```

**Why it matters** — unit tests with mocks only verify that your code calls mocks correctly. Integration tests catch real wiring issues: misconfigured DI, incorrect EF mappings, middleware ordering, and serialization bugs.

### Excessive Mocking (🟡 Medium)

❌ **Anti-pattern** — mocking everything including the class under test's own collaborators:

```csharp
// ❌ WRONG — so many mocks that the test tests nothing real
[Fact]
public async Task ProcessOrder_WithMocksForEverything()
{
    var mockRepo = new Mock<IOrderRepository>();
    var mockPricing = new Mock<IPricingService>();
    var mockInventory = new Mock<IInventoryService>();
    var mockEmail = new Mock<IEmailService>();
    var mockLogger = new Mock<ILogger<OrderProcessor>>();
    var mockMetrics = new Mock<IMetricsService>();

    mockPricing.Setup(p => p.CalculateAsync(It.IsAny<Order>()))
        .ReturnsAsync(new PricedOrder { Total = 100 });
    mockInventory.Setup(i => i.ReserveAsync(It.IsAny<int>(), It.IsAny<int>()))
        .ReturnsAsync(true);
    // ... 10 more Setup calls

    var processor = new OrderProcessor(
        mockRepo.Object, mockPricing.Object, mockInventory.Object,
        mockEmail.Object, mockLogger.Object, mockMetrics.Object);

    await processor.ProcessAsync(new Order());

    // This "test" just verifies mock wiring, not business logic
}
```

✅ **The fix** — use fakes for complex collaborators and real implementations where practical:

```csharp
// ✅ CORRECT — use simple fakes and test real behavior
[Fact]
public async Task ProcessOrder_CalculatesTotalAndReservesInventory()
{
    var repo = new InMemoryOrderRepository();
    var pricing = new InMemoryPricingService(pricePerUnit: 25m);
    var inventory = new InMemoryInventoryService(availableStock: 100);
    var notifications = new FakeNotificationService();

    var processor = new OrderProcessor(
        repo, pricing, inventory, notifications, NullLogger<OrderProcessor>.Instance);

    var order = new Order { ProductId = 1, Quantity = 4 };
    await processor.ProcessAsync(order);

    Assert.Equal(100m, repo.SavedOrders.Single().Total); // 4 × 25
    Assert.Equal(96, inventory.GetStock(1)); // 100 - 4
}
```

**Why it matters** — excessive mocking creates tests that pass regardless of actual behavior. Fakes (in-memory implementations) exercise real logic while remaining fast and deterministic.

### Test Interdependency (🟠 High)

❌ **Anti-pattern** — tests that depend on execution order or shared mutable state:

```csharp
// ❌ WRONG — tests share a static database; order-dependent
public class UserTests
{
    private static readonly AppDbContext _db = CreateTestDb();

    [Fact]
    public async Task CreateUser_InsertsUser()
    {
        _db.Users.Add(new User { Id = 1, Name = "Alice" });
        await _db.SaveChangesAsync();
        Assert.Single(_db.Users);
    }

    [Fact]
    public async Task GetUsers_ReturnsAll()
    {
        // Depends on CreateUser_InsertsUser running first!
        var users = await _db.Users.ToListAsync();
        Assert.Single(users); // fails if run in isolation or different order
    }
}
```

✅ **The fix** — isolate each test with its own data:

```csharp
// ✅ CORRECT — each test has independent state
public class UserTests : IAsyncLifetime
{
    private AppDbContext _db = null!;

    public async Task InitializeAsync()
    {
        _db = await CreateFreshTestDbAsync(); // fresh database per test
    }

    public async Task DisposeAsync() => await _db.DisposeAsync();

    [Fact]
    public async Task CreateUser_InsertsUser()
    {
        _db.Users.Add(new User { Id = 1, Name = "Alice" });
        await _db.SaveChangesAsync();
        Assert.Single(_db.Users);
    }

    [Fact]
    public async Task GetUsers_ReturnsEmpty_WhenNoUsersExist()
    {
        var users = await _db.Users.ToListAsync();
        Assert.Empty(users); // independent — always passes
    }
}
```

**Why it matters** — interdependent tests fail unpredictably based on execution order, create false passes in CI, and are impossible to run in parallel. Each test must be fully self-contained.

## Detection and Prevention

Proactive tooling catches anti-patterns before they reach production.

### Static Analyzers

| Tool | Focus | Integration |
|---|---|---|
| **Roslynator** | 500+ C# analyzers and refactorings | NuGet, IDE, CI |
| **SonarQube / SonarCloud** | Code smells, bugs, security vulnerabilities | CI pipeline |
| **Microsoft.CodeAnalysis.NetAnalyzers** | .NET API usage, security, performance | Built into .NET SDK |
| **AsyncFixer** | Async/await anti-patterns | NuGet analyzer |
| **EF Core Power Tools** | Query analysis, model validation | Visual Studio extension |

```csharp
// Enable analyzers in your .csproj
<PropertyGroup>
    <AnalysisLevel>latest-all</AnalysisLevel>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
</PropertyGroup>

<ItemGroup>
    <PackageReference Include="Roslynator.Analyzers" Version="4.*" PrivateAssets="all" />
    <PackageReference Include="AsyncFixer" Version="1.*" PrivateAssets="all" />
    <PackageReference Include="SonarAnalyzer.CSharp" Version="10.*" PrivateAssets="all" />
</ItemGroup>
```

### Code Review Checklist

Use this checklist during pull request reviews:

- ✅ No `async void` methods (except UI event handlers)
- ✅ No `.Result` or `.Wait()` on tasks
- ✅ No `catch (Exception)` without rethrow or specific handling
- ✅ No `throw ex;` — use `throw;` or wrap with `InnerException`
- ✅ DbContext is scoped (never singleton)
- ✅ All `IDisposable` resources are disposed (via `using` or DI)
- ✅ No N+1 queries — verified via SQL logging or EF profiler
- ✅ Request DTOs used instead of binding directly to entities
- ✅ Secrets loaded from vault/env, not in `appsettings.json`
- ✅ Configuration validated with `ValidateOnStart()`
- ✅ Tests are independent and assert behavior, not implementation

### Architectural Fitness Functions

Automate architectural rules with tests:

```csharp
// ✅ Enforce layering rules with architecture tests (e.g., ArchUnitNET)
[Fact]
public void DomainLayer_ShouldNotReference_InfrastructureLayer()
{
    var domainTypes = Types().That()
        .ResideInNamespace("MyApp.Domain").GetTypes();

    domainTypes.Should()
        .NotDependOnAny(
            Types().That().ResideInNamespace("MyApp.Infrastructure").GetTypes());
}

[Fact]
public void Controllers_ShouldNotDependOn_DbContext()
{
    var controllers = Types().That()
        .ImplementInterface(typeof(ControllerBase)).GetTypes();

    controllers.Should()
        .NotDependOnAny(
            Types().That().AreAssignableTo(typeof(DbContext)).GetTypes());
}
```

### CI Pipeline Integration

```
┌───────────────────────────────────────────────────────────────┐
│                    Anti-Pattern Prevention Pipeline            │
│                                                               │
│  ┌──────────┐   ┌───────────┐   ┌──────────┐   ┌─────────┐  │
│  │  Build   │──▶│ Analyzers │──▶│  Tests   │──▶│  Gate   │  │
│  │          │   │           │   │          │   │         │  │
│  │ dotnet   │   │ Roslyn    │   │ Unit +   │   │ Quality │  │
│  │ build    │   │ Sonar     │   │ Integr.  │   │ Gate    │  │
│  │ --warnas │   │ AsyncFixr │   │ Arch.    │   │ Pass/   │  │
│  │ error    │   │           │   │ Fitness  │   │ Fail    │  │
│  └──────────┘   └───────────┘   └──────────┘   └─────────┘  │
│                                                               │
│  Fail the build if:                                           │
│  - Any analyzer error is introduced                           │
│  - Code coverage drops below threshold                        │
│  - Architecture tests fail                                    │
│  - SonarQube quality gate fails                               │
└───────────────────────────────────────────────────────────────┘
```

## Next Steps

Return to the [Learning Path](LEARNING-PATH.md) for the complete curriculum. Combine this guide with the [Best Practices](03-BEST-PRACTICES.md) and [Design Patterns](11-DESIGN-PATTERNS.md) documents to build a comprehensive understanding of what to do and what to avoid in .NET applications.

### Related Documents

1. **[Learning Path](LEARNING-PATH.md)** — Complete .NET learning curriculum
2. **[Best Practices](03-BEST-PRACTICES.md)** — Async/await, memory, security, testing
3. **[Concurrency](05-CONCURRENCY-ASYNC-PARALLELISM.md)** — Async patterns, channels, parallelism
4. **[Dependency Injection](06-DEPENDENCY-INJECTION.md)** — DI lifetimes, registration, anti-patterns
5. **[Entity Framework Core](07-ENTITY-FRAMEWORK-CORE.md)** — EF Core patterns and performance
6. **[Testing](08-TESTING.md)** — Unit, integration, and architecture testing
7. **[Design Patterns](11-DESIGN-PATTERNS.md)** — GOF and .NET-specific patterns
8. **[Configuration and Deployment](12-CONFIGURATION-AND-DEPLOYMENT.md)** — Options pattern, secrets, CI/CD

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial .NET and C# anti-patterns documentation |
