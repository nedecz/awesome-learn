# Performance Optimization for Payment Systems

## Overview

Performance is critical for payment systems. This document covers optimization strategies from database queries to caching, covering the entire payment processing pipeline.

## Table of Contents

1. [Database Performance](#database-performance)
2. [Caching Strategies](#caching-strategies)
3. [Query Optimization](#query-optimization)
4. [Connection Pooling](#connection-pooling)
5. [Asynchronous Processing](#asynchronous-processing)
6. [API Performance](#api-performance)
7. [Memory Optimization](#memory-optimization)
8. [Network Optimization](#network-optimization)
9. [Load Testing](#load-testing)
10. [Monitoring & Profiling](#monitoring--profiling)

---

## Database Performance

### 1. Indexing Strategy

```sql
-- Composite indexes for common queries
CREATE INDEX IX_Payments_Status_CreatedAt 
ON Payments(Status, CreatedAt DESC)
INCLUDE (Amount, Currency, OrderId);

-- Filtered indexes for active records
CREATE INDEX IX_Payments_Active 
ON Payments(Status, CreatedAt)
WHERE Status IN ('Pending', 'Processing')
WITH (FILLFACTOR = 80); -- Leave room for updates

-- Covering indexes to avoid table lookups
CREATE INDEX IX_Payments_Customer_Covering
ON Payments(CustomerEmail)
INCLUDE (Amount, Currency, Status, CreatedAt);
```

### 2. Partitioning Large Tables

**Date-based Partitioning:**

```sql
-- SQL Server - Partition by month
CREATE PARTITION FUNCTION PF_PaymentsByMonth (DATETIME2)
AS RANGE RIGHT FOR VALUES (
    '2024-01-01', '2024-02-01', '2024-03-01', '2024-04-01',
    '2024-05-01', '2024-06-01', '2024-07-01', '2024-08-01',
    '2024-09-01', '2024-10-01', '2024-11-01', '2024-12-01'
);

CREATE PARTITION SCHEME PS_PaymentsByMonth
AS PARTITION PF_PaymentsByMonth
TO ([FG2024_01], [FG2024_02], [FG2024_03], [FG2024_04],
    [FG2024_05], [FG2024_06], [FG2024_07], [FG2024_08],
    [FG2024_09], [FG2024_10], [FG2024_11], [FG2024_12]);

-- Create partitioned table
CREATE TABLE Payments (
    Id UNIQUEIDENTIFIER NOT NULL,
    CreatedAt DATETIME2 NOT NULL,
    -- other columns...
    CONSTRAINT PK_Payments PRIMARY KEY (Id, CreatedAt)
) ON PS_PaymentsByMonth(CreatedAt);
```

**PostgreSQL Partitioning:**

```sql
-- Create parent table
CREATE TABLE payments (
    id UUID NOT NULL,
    created_at TIMESTAMP NOT NULL,
    amount NUMERIC(18,2) NOT NULL,
    status VARCHAR(50) NOT NULL
) PARTITION BY RANGE (created_at);

-- Create partitions
CREATE TABLE payments_2024_01 PARTITION OF payments
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE payments_2024_02 PARTITION OF payments
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Indexes on each partition
CREATE INDEX idx_payments_2024_01_status ON payments_2024_01(status);
CREATE INDEX idx_payments_2024_02_status ON payments_2024_02(status);
```

### 3. Archiving Old Data

```csharp
public class PaymentArchivalService
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<PaymentArchivalService> _logger;
    
    public async Task ArchiveOldPaymentsAsync(int monthsOld = 12)
    {
        var cutoffDate = DateTime.UtcNow.AddMonths(-monthsOld);
        
        using var transaction = await _context.Database.BeginTransactionAsync();
        
        try
        {
            // Copy to archive table
            await _context.Database.ExecuteSqlRawAsync(@"
                INSERT INTO PaymentsArchive (Id, OrderId, Amount, Currency, Status, CreatedAt, ArchivedAt)
                SELECT Id, OrderId, Amount, Currency, Status, CreatedAt, GETUTCDATE()
                FROM Payments
                WHERE CreatedAt < {0} 
                  AND Status IN ('Completed', 'Refunded', 'Failed', 'Canceled')
            ", cutoffDate);
            
            // Delete from main table
            var deleted = await _context.Database.ExecuteSqlRawAsync(@"
                DELETE FROM Payments
                WHERE CreatedAt < {0}
                  AND Status IN ('Completed', 'Refunded', 'Failed', 'Canceled')
            ", cutoffDate);
            
            await transaction.CommitAsync();
            
            _logger.LogInformation("Archived {Count} payments older than {Date}", 
                deleted, cutoffDate);
        }
        catch (Exception ex)
        {
            await transaction.RollbackAsync();
            _logger.LogError(ex, "Failed to archive payments");
            throw;
        }
    }
}
```

### 4. Read Replicas for Queries

```csharp
public class PaymentDbContextFactory
{
    private readonly string _primaryConnectionString;
    private readonly string _readReplicaConnectionString;
    
    public ApplicationDbContext CreateWriteContext()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlServer(_primaryConnectionString)
            .Options;
        
        return new ApplicationDbContext(options);
    }
    
    public ApplicationDbContext CreateReadContext()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlServer(_readReplicaConnectionString)
            .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking)
            .Options;
        
        return new ApplicationDbContext(options);
    }
}

// Usage
public class PaymentService
{
    private readonly PaymentDbContextFactory _dbFactory;
    
    // Write operations use primary
    public async Task<Payment> CreatePaymentAsync(PaymentRequest request)
    {
        using var context = _dbFactory.CreateWriteContext();
        var payment = new Payment(/* ... */);
        await context.Payments.AddAsync(payment);
        await context.SaveChangesAsync();
        return payment;
    }
    
    // Read operations use replica
    public async Task<Payment> GetPaymentAsync(Guid id)
    {
        using var context = _dbFactory.CreateReadContext();
        return await context.Payments
            .AsNoTracking()
            .FirstOrDefaultAsync(p => p.Id == id);
    }
}
```

---

## Caching Strategies

### 1. Multi-Level Caching

```csharp
public class MultiLevelCacheService
{
    private readonly IMemoryCache _l1Cache; // In-memory
    private readonly IDistributedCache _l2Cache; // Redis
    private readonly TimeSpan _l1Duration = TimeSpan.FromMinutes(5);
    private readonly TimeSpan _l2Duration = TimeSpan.FromMinutes(30);
    
    public async Task<T> GetOrSetAsync<T>(
        string key, 
        Func<Task<T>> factory)
    {
        // L1: Check memory cache
        if (_l1Cache.TryGetValue(key, out T value))
        {
            return value;
        }
        
        // L2: Check distributed cache
        var cached = await _l2Cache.GetStringAsync(key);
        if (cached != null)
        {
            value = JsonSerializer.Deserialize<T>(cached);
            _l1Cache.Set(key, value, _l1Duration);
            return value;
        }
        
        // Cache miss: Get from source
        value = await factory();
        
        // Store in both caches
        _l1Cache.Set(key, value, _l1Duration);
        await _l2Cache.SetStringAsync(
            key,
            JsonSerializer.Serialize(value),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = _l2Duration
            });
        
        return value;
    }
}

// Usage
public class PaymentRepository
{
    private readonly MultiLevelCacheService _cache;
    private readonly ApplicationDbContext _context;
    
    public async Task<Payment> GetByIdAsync(Guid id)
    {
        return await _cache.GetOrSetAsync(
            $"payment:{id}",
            async () => await _context.Payments.FindAsync(id)
        );
    }
}
```

### 2. Cache-Aside Pattern

For the full Cache-Aside pattern implementation with stampede prevention, see [Design Patterns — Cache-Aside Pattern](07-DESIGN-PATTERNS.md#cache-aside-pattern).

The pattern follows these steps:
1. Check cache before querying the database
2. On cache miss, load from database and store in cache
3. On write, update database first, then invalidate cache

### 3. Write-Through Cache

```csharp
public class WriteThroughCacheRepository : IPaymentRepository
{
    private readonly IPaymentRepository _inner;
    private readonly IDistributedCache _cache;
    
    public async Task AddAsync(Payment payment)
    {
        // Write to database
        await _inner.AddAsync(payment);
        
        // Write to cache immediately
        await _cache.SetStringAsync(
            $"payment:{payment.Id}",
            JsonSerializer.Serialize(payment),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
            });
    }
}
```

### 4. Cache Warming

```csharp
public class CacheWarmingService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<CacheWarmingService> _logger;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await WarmCacheAsync();
                await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Cache warming failed");
            }
        }
    }
    
    private async Task WarmCacheAsync()
    {
        using var scope = _serviceProvider.CreateScope();
        var cache = scope.ServiceProvider.GetRequiredService<IDistributedCache>();
        var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        
        // Cache frequently accessed data
        var recentPayments = await context.Payments
            .Where(p => p.CreatedAt > DateTime.UtcNow.AddHours(-24))
            .OrderByDescending(p => p.CreatedAt)
            .Take(1000)
            .ToListAsync();
        
        foreach (var payment in recentPayments)
        {
            await cache.SetStringAsync(
                $"payment:{payment.Id}",
                JsonSerializer.Serialize(payment),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
                });
        }
        
        _logger.LogInformation("Warmed cache with {Count} payments", recentPayments.Count);
    }
}
```

---

## Query Optimization

### 1. Compiled Queries

```csharp
public class OptimizedPaymentRepository
{
    // Compile frequently used queries
    private static readonly Func<ApplicationDbContext, Guid, Task<Payment>>
        GetByIdQuery = EF.CompileAsyncQuery(
            (ApplicationDbContext context, Guid id) =>
                context.Payments
                    .AsNoTracking()
                    .FirstOrDefault(p => p.Id == id));
    
    private static readonly Func<ApplicationDbContext, string, Task<Payment>>
        GetByOrderIdQuery = EF.CompileAsyncQuery(
            (ApplicationDbContext context, string orderId) =>
                context.Payments
                    .AsNoTracking()
                    .FirstOrDefault(p => p.OrderId == orderId));
    
    private static readonly Func<ApplicationDbContext, DateTime, DateTime, IAsyncEnumerable<Payment>>
        GetByDateRangeQuery = EF.CompileAsyncQuery(
            (ApplicationDbContext context, DateTime start, DateTime end) =>
                context.Payments
                    .AsNoTracking()
                    .Where(p => p.CreatedAt >= start && p.CreatedAt <= end));
    
    private readonly ApplicationDbContext _context;
    
    public async Task<Payment> GetByIdAsync(Guid id)
    {
        return await GetByIdQuery(_context, id);
    }
    
    public async Task<Payment> GetByOrderIdAsync(string orderId)
    {
        return await GetByOrderIdQuery(_context, orderId);
    }
    
    public async Task<List<Payment>> GetByDateRangeAsync(DateTime start, DateTime end)
    {
        return await GetByDateRangeQuery(_context, start, end).ToListAsync();
    }
}
```

### 2. Projection Instead of Full Entities

```csharp
// ❌ Bad: Loads entire entity
public async Task<List<Payment>> GetRecentPaymentsAsync()
{
    return await _context.Payments
        .Where(p => p.CreatedAt > DateTime.UtcNow.AddDays(-7))
        .ToListAsync(); // Loads all columns
}

// ✅ Good: Project only needed fields
public async Task<List<PaymentSummaryDto>> GetRecentPaymentSummariesAsync()
{
    return await _context.Payments
        .Where(p => p.CreatedAt > DateTime.UtcNow.AddDays(-7))
        .Select(p => new PaymentSummaryDto
        {
            Id = p.Id,
            Amount = p.Amount,
            Currency = p.Currency,
            Status = p.Status,
            CreatedAt = p.CreatedAt
        })
        .ToListAsync(); // Loads only selected columns
}
```

### 3. Batch Operations

```csharp
public class BatchPaymentRepository
{
    private readonly ApplicationDbContext _context;
    
    // ❌ Bad: Multiple round trips
    public async Task UpdatePaymentsAsync(List<Payment> payments)
    {
        foreach (var payment in payments)
        {
            _context.Payments.Update(payment);
            await _context.SaveChangesAsync(); // N queries
        }
    }
    
    // ✅ Good: Single batch
    public async Task UpdatePaymentsBatchAsync(List<Payment> payments)
    {
        _context.Payments.UpdateRange(payments);
        await _context.SaveChangesAsync(); // 1 query
    }
    
    // ✅ Better: Bulk update with raw SQL
    public async Task UpdatePaymentStatusBulkAsync(
        List<Guid> paymentIds, 
        PaymentStatus newStatus)
    {
        await _context.Database.ExecuteSqlRawAsync(@"
            UPDATE Payments 
            SET Status = {0}, UpdatedAt = GETUTCDATE()
            WHERE Id IN ({1})
        ", newStatus, string.Join(",", paymentIds.Select(id => $"'{id}'")));
    }
}
```

### 4. Avoid N+1 Queries

```csharp
// ❌ Bad: N+1 query problem
public async Task<List<PaymentWithRefundsDto>> GetPaymentsWithRefundsAsync()
{
    var payments = await _context.Payments.ToListAsync(); // 1 query
    
    var result = new List<PaymentWithRefundsDto>();
    foreach (var payment in payments)
    {
        var refunds = await _context.Refunds
            .Where(r => r.PaymentId == payment.Id)
            .ToListAsync(); // N queries!
        
        result.Add(new PaymentWithRefundsDto
        {
            Payment = payment,
            Refunds = refunds
        });
    }
    
    return result;
}

// ✅ Good: Single query with Include
public async Task<List<PaymentWithRefundsDto>> GetPaymentsWithRefundsOptimizedAsync()
{
    return await _context.Payments
        .Include(p => p.Refunds) // Join in single query
        .Select(p => new PaymentWithRefundsDto
        {
            Payment = p,
            Refunds = p.Refunds.ToList()
        })
        .ToListAsync(); // 1 query
}
```

### 5. Pagination

```csharp
public class PaginatedResult<T>
{
    public List<T> Items { get; set; }
    public int TotalCount { get; set; }
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
}

public async Task<PaginatedResult<Payment>> GetPaymentsPagedAsync(
    int page, 
    int pageSize = 50)
{
    var query = _context.Payments
        .AsNoTracking()
        .OrderByDescending(p => p.CreatedAt);
    
    var totalCount = await query.CountAsync();
    
    var items = await query
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync();
    
    return new PaginatedResult<Payment>
    {
        Items = items,
        TotalCount = totalCount,
        Page = page,
        PageSize = pageSize
    };
}
```

---

## Connection Pooling

### Configuration

```csharp
// appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=.;Database=PaymentDb;Integrated Security=true;Min Pool Size=10;Max Pool Size=100;Connection Lifetime=300;Pooling=true"
  }
}
```

### Custom Connection Pool Management

```csharp
public class ConnectionPoolManager
{
    private readonly string _connectionString;
    private readonly SemaphoreSlim _semaphore;
    
    public ConnectionPoolManager(string connectionString, int maxConnections = 100)
    {
        _connectionString = connectionString;
        _semaphore = new SemaphoreSlim(maxConnections, maxConnections);
    }
    
    public async Task<T> ExecuteAsync<T>(Func<SqlConnection, Task<T>> action)
    {
        await _semaphore.WaitAsync();
        
        try
        {
            using var connection = new SqlConnection(_connectionString);
            await connection.OpenAsync();
            return await action(connection);
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

---

## Asynchronous Processing

### 1. Background Job Processing

```csharp
public class PaymentProcessingService
{
    private readonly IBackgroundJobClient _backgroundJobs;
    private readonly IPaymentRepository _repository;
    
    // Synchronous API endpoint returns immediately
    public async Task<PaymentResponse> InitiatePaymentAsync(PaymentRequest request)
    {
        // Create payment record
        var payment = Payment.Create(request.OrderId, request.Amount, request.Currency);
        await _repository.AddAsync(payment);
        
        // Queue for background processing
        _backgroundJobs.Enqueue(() => ProcessPaymentAsync(payment.Id));
        
        // Return immediately
        return new PaymentResponse
        {
            PaymentId = payment.Id,
            Status = "Pending"
        };
    }
    
    // Processed asynchronously
    [Queue("payments")]
    [AutomaticRetry(Attempts = 3)]
    public async Task ProcessPaymentAsync(Guid paymentId)
    {
        var payment = await _repository.GetByIdAsync(paymentId);
        
        // Long-running payment processing
        var result = await _gateway.ProcessPaymentAsync(/* ... */);
        
        if (result.IsSuccess)
        {
            payment.MarkAsCompleted();
        }
        else
        {
            payment.MarkAsFailed(result.ErrorMessage);
        }
        
        await _repository.UpdateAsync(payment);
    }
}
```

### 2. Parallel Processing

```csharp
public class BulkPaymentProcessor
{
    private readonly IPaymentService _paymentService;
    private readonly int _maxDegreeOfParallelism = 10;
    
    public async Task<List<PaymentResult>> ProcessBulkPaymentsAsync(
        List<PaymentRequest> requests)
    {
        var results = new ConcurrentBag<PaymentResult>();
        
        await Parallel.ForEachAsync(
            requests,
            new ParallelOptions 
            { 
                MaxDegreeOfParallelism = _maxDegreeOfParallelism 
            },
            async (request, ct) =>
            {
                var result = await _paymentService.ProcessPaymentAsync(request);
                results.Add(result);
            });
        
        return results.ToList();
    }
}
```

### 3. Channel-based Processing

```csharp
public class ChannelBasedPaymentProcessor : BackgroundService
{
    private readonly Channel<PaymentRequest> _channel;
    private readonly IServiceProvider _serviceProvider;
    
    public ChannelBasedPaymentProcessor(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
        _channel = Channel.CreateUnbounded<PaymentRequest>(
            new UnboundedChannelOptions
            {
                SingleReader = false,
                SingleWriter = false
            });
    }
    
    public async Task EnqueuePaymentAsync(PaymentRequest request)
    {
        await _channel.Writer.WriteAsync(request);
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Process payments with multiple workers
        var workers = Enumerable.Range(0, 5)
            .Select(_ => ProcessPaymentsAsync(stoppingToken));
        
        await Task.WhenAll(workers);
    }
    
    private async Task ProcessPaymentsAsync(CancellationToken stoppingToken)
    {
        await foreach (var request in _channel.Reader.ReadAllAsync(stoppingToken))
        {
            using var scope = _serviceProvider.CreateScope();
            var paymentService = scope.ServiceProvider
                .GetRequiredService<IPaymentService>();
            
            try
            {
                await paymentService.ProcessPaymentAsync(request);
            }
            catch (Exception ex)
            {
                // Log error
            }
        }
    }
}
```

---

## API Performance

### 1. Response Compression

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddResponseCompression(options =>
    {
        options.EnableForHttps = true;
        options.Providers.Add<BrotliCompressionProvider>();
        options.Providers.Add<GzipCompressionProvider>();
    });
    
    services.Configure<BrotliCompressionProviderOptions>(options =>
    {
        options.Level = CompressionLevel.Fastest;
    });
}

public void Configure(IApplicationBuilder app)
{
    app.UseResponseCompression();
}
```

### 2. Output Caching

```csharp
[ApiController]
[Route("api/[controller]")]
public class PaymentsController : ControllerBase
{
    [HttpGet("{id}")]
    [ResponseCache(Duration = 60, VaryByQueryKeys = new[] { "id" })]
    public async Task<ActionResult<PaymentDto>> GetPayment(Guid id)
    {
        var payment = await _paymentService.GetByIdAsync(id);
        return Ok(payment);
    }
}
```

### 3. Minimal API Optimization

```csharp
// Use minimal APIs for better performance
app.MapGet("/api/payments/{id:guid}", async (
    Guid id,
    IPaymentRepository repository) =>
{
    var payment = await repository.GetByIdAsync(id);
    return payment is not null ? Results.Ok(payment) : Results.NotFound();
});

// Or with compiled query
app.MapGet("/api/payments/{id:guid}", 
    [EnableCors("AllowAll")] async (
        Guid id,
        ApplicationDbContext db) =>
{
    var payment = await GetByIdCompiledQuery(db, id);
    return payment is not null ? Results.Ok(payment) : Results.NotFound();
});

static readonly Func<ApplicationDbContext, Guid, Task<Payment>> 
    GetByIdCompiledQuery = EF.CompileAsyncQuery(
        (ApplicationDbContext context, Guid id) =>
            context.Payments.FirstOrDefault(p => p.Id == id));
```

---

## Memory Optimization

### 1. Object Pooling

```csharp
public class PaymentRequestPool
{
    private static readonly ObjectPool<PaymentRequest> Pool =
        ObjectPool.Create<PaymentRequest>();
    
    public static PaymentRequest Rent()
    {
        var request = Pool.Get();
        // Reset state
        request.Reset();
        return request;
    }
    
    public static void Return(PaymentRequest request)
    {
        Pool.Return(request);
    }
}

// Usage
var request = PaymentRequestPool.Rent();
try
{
    request.OrderId = orderId;
    request.Amount = amount;
    // Use request...
}
finally
{
    PaymentRequestPool.Return(request);
}
```

### 2. Span\<T\> for String Operations

```csharp
public class PaymentIdGenerator
{
    // ❌ Bad: Creates string allocations
    public string GenerateIdOld(string prefix, Guid guid)
    {
        return prefix + "-" + guid.ToString();
    }
    
    // ✅ Good: Uses spans, less allocation
    public string GenerateId(ReadOnlySpan<char> prefix, Guid guid)
    {
        Span<char> buffer = stackalloc char[50];
        
        prefix.CopyTo(buffer);
        buffer[prefix.Length] = '-';
        
        guid.TryFormat(buffer.Slice(prefix.Length + 1), out int written);
        
        return buffer.Slice(0, prefix.Length + 1 + written).ToString();
    }
}
```

### 3. Streaming Large Results

```csharp
[HttpGet("export")]
public async Task ExportPayments([FromQuery] DateTime startDate)
{
    Response.ContentType = "text/csv";
    Response.Headers.Add("Content-Disposition", "attachment; filename=payments.csv");
    
    await using var writer = new StreamWriter(Response.Body);
    await writer.WriteLineAsync("Id,Amount,Currency,Status,CreatedAt");
    
    // Stream results without loading all into memory
    await foreach (var payment in _context.Payments
        .Where(p => p.CreatedAt >= startDate)
        .AsAsyncEnumerable())
    {
        await writer.WriteLineAsync(
            $"{payment.Id},{payment.Amount},{payment.Currency},{payment.Status},{payment.CreatedAt}");
    }
}
```

---

## Network Optimization

### 1. HTTP/2

```csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.ConfigureKestrel(options =>
            {
                options.ConfigureEndpointDefaults(listenOptions =>
                {
                    listenOptions.Protocols = HttpProtocols.Http2;
                });
            });
        });
```

### 2. Connection Reuse

```csharp
public static class HttpClientConfiguration
{
    public static IServiceCollection AddPaymentHttpClients(
        this IServiceCollection services)
    {
        services.AddHttpClient<IPaymentGateway, StripeGateway>(client =>
        {
            client.BaseAddress = new Uri("https://api.stripe.com");
            client.Timeout = TimeSpan.FromSeconds(30);
        })
        .ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
        {
            PooledConnectionLifetime = TimeSpan.FromMinutes(2),
            PooledConnectionIdleTimeout = TimeSpan.FromMinutes(1),
            MaxConnectionsPerServer = 100,
            EnableMultipleHttp2Connections = true
        });
        
        return services;
    }
}
```

---

## Load Testing

### 1. k6 Load Test

```javascript
// payment-load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
    stages: [
        { duration: '2m', target: 100 },  // Ramp up
        { duration: '5m', target: 100 },  // Stay at 100
        { duration: '2m', target: 200 },  // Ramp to 200
        { duration: '5m', target: 200 },  // Stay at 200
        { duration: '2m', target: 0 },    // Ramp down
    ],
    thresholds: {
        http_req_duration: ['p(95)<500'],  // 95% under 500ms
        http_req_failed: ['rate<0.01'],    // Error rate < 1%
    },
};

export default function () {
    const payload = JSON.stringify({
        orderId: `order-${Date.now()}`,
        amount: 100.00,
        currency: 'USD',
        paymentMethodId: 'pm_test_valid'
    });
    
    const params = {
        headers: {
            'Content-Type': 'application/json',
        },
    };
    
    const res = http.post(
        'https://localhost:5001/api/payments',
        payload,
        params
    );
    
    check(res, {
        'status is 200': (r) => r.status === 200,
        'response time < 500ms': (r) => r.timings.duration < 500,
    });
    
    sleep(1);
}
```

Run: `k6 run payment-load-test.js`

### 2. NBomber Load Test (.NET)

```csharp
public class PaymentLoadTest
{
    [Fact]
    public void LoadTest_PaymentProcessing()
    {
        var httpClient = new HttpClient
        {
            BaseAddress = new Uri("https://localhost:5001")
        };
        
        var scenario = Scenario.Create("payment_processing", async context =>
        {
            var request = new PaymentRequest
            {
                OrderId = $"order-{Guid.NewGuid()}",
                Amount = 100.00m,
                Currency = "USD",
                PaymentMethodId = "pm_test_valid"
            };
            
            var response = await httpClient.PostAsJsonAsync("/api/payments", request);
            
            return response.IsSuccessStatusCode
                ? Response.Ok()
                : Response.Fail();
        })
        .WithWarmUpDuration(TimeSpan.FromSeconds(10))
        .WithLoadSimulations(
            Simulation.RampConstant(copies: 50, during: TimeSpan.FromMinutes(1)),
            Simulation.KeepConstant(copies: 100, during: TimeSpan.FromMinutes(3))
        );
        
        var stats = NBomberRunner
            .RegisterScenarios(scenario)
            .Run();
        
        var okCount = stats.ScenarioStats[0].Ok.Request.Count;
        var failCount = stats.ScenarioStats[0].Fail.Request.Count;
        
        Assert.True(okCount > 0);
        Assert.True(failCount < okCount * 0.01); // Less than 1% failure
    }
}
```

---

## Monitoring & Profiling

### 1. Performance Metrics

```csharp
public class PaymentMetrics
{
    private readonly ILogger<PaymentMetrics> _logger;
    private readonly DiagnosticListener _diagnosticListener;
    
    public async Task<T> TrackAsync<T>(
        string operationName,
        Func<Task<T>> operation)
    {
        var sw = Stopwatch.StartNew();
        
        try
        {
            var result = await operation();
            sw.Stop();
            
            _logger.LogInformation(
                "Operation {Operation} completed in {Duration}ms",
                operationName,
                sw.ElapsedMilliseconds);
            
            // Send to monitoring system
            _diagnosticListener.Write("OperationCompleted", new
            {
                Operation = operationName,
                Duration = sw.Elapsed,
                Success = true
            });
            
            return result;
        }
        catch (Exception ex)
        {
            sw.Stop();
            
            _logger.LogError(ex,
                "Operation {Operation} failed after {Duration}ms",
                operationName,
                sw.ElapsedMilliseconds);
            
            _diagnosticListener.Write("OperationFailed", new
            {
                Operation = operationName,
                Duration = sw.Elapsed,
                Success = false,
                Error = ex.Message
            });
            
            throw;
        }
    }
}

// Usage
public class PaymentService
{
    private readonly PaymentMetrics _metrics;
    
    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        return await _metrics.TrackAsync(
            "ProcessPayment",
            async () => await ProcessPaymentInternalAsync(request)
        );
    }
}
```

### 2. Application Insights

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddApplicationInsightsTelemetry();
    
    services.AddSingleton<ITelemetryInitializer, PaymentTelemetryInitializer>();
}

public class PaymentTelemetryInitializer : ITelemetryInitializer
{
    public void Initialize(ITelemetry telemetry)
    {
        if (telemetry is RequestTelemetry requestTelemetry)
        {
            // Add custom properties
            requestTelemetry.Properties["Service"] = "PaymentService";
            requestTelemetry.Properties["Version"] = "1.0.0";
        }
    }
}

// Track custom metrics
public class PaymentService
{
    private readonly TelemetryClient _telemetry;
    
    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        var sw = Stopwatch.StartNew();
        
        try
        {
            var result = await ProcessPaymentInternalAsync(request);
            sw.Stop();
            
            _telemetry.TrackMetric("PaymentProcessingTime", sw.ElapsedMilliseconds);
            _telemetry.TrackEvent("PaymentProcessed", new Dictionary<string, string>
            {
                ["Amount"] = request.Amount.ToString(),
                ["Currency"] = request.Currency,
                ["Success"] = result.IsSuccess.ToString()
            });
            
            return result;
        }
        catch (Exception ex)
        {
            _telemetry.TrackException(ex);
            throw;
        }
    }
}
```

### 3. Health Checks

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddHealthChecks()
        .AddDbContextCheck<ApplicationDbContext>()
        .AddRedis(Configuration["Redis:ConnectionString"])
        .AddUrlGroup(new Uri("https://api.stripe.com/healthcheck"), "Stripe API")
        .AddCheck<PaymentServiceHealthCheck>("payment-service");
}

public class PaymentServiceHealthCheck : IHealthCheck
{
    private readonly IPaymentRepository _repository;
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // Check if we can query the database
            var recentPayment = await _repository.GetRecentPaymentAsync();
            
            // Check if we have pending payments stuck
            var stuckPayments = await _repository.GetStuckPaymentsAsync();
            
            if (stuckPayments.Count > 100)
            {
                return HealthCheckResult.Degraded(
                    $"Found {stuckPayments.Count} stuck payments");
            }
            
            return HealthCheckResult.Healthy("Payment service is healthy");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                "Payment service is unhealthy",
                ex);
        }
    }
}

// Endpoint
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        var result = JsonSerializer.Serialize(new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                description = e.Value.Description,
                duration = e.Value.Duration.TotalMilliseconds
            })
        });
        await context.Response.WriteAsync(result);
    }
});
```

---

## Performance Checklist

### Database
- [ ] Proper indexes on all foreign keys and query columns
- [ ] Partitioning for large tables
- [ ] Archiving old data
- [ ] Read replicas for queries
- [ ] Connection pooling configured

### Caching
- [ ] Multi-level caching (memory + distributed)
- [ ] Cache-aside pattern implemented
- [ ] Cache invalidation strategy
- [ ] Cache warming for hot data

### Queries
- [ ] Compiled queries for frequent operations
- [ ] Projection instead of full entities
- [ ] Batch operations where possible
- [ ] No N+1 queries
- [ ] Pagination for large result sets

### API
- [ ] Response compression enabled
- [ ] Output caching where appropriate
- [ ] Minimal APIs for performance-critical endpoints
- [ ] HTTP/2 enabled

### Async Processing
- [ ] Background jobs for long operations
- [ ] Parallel processing for bulk operations
- [ ] Channel-based processing for queues

### Monitoring
- [ ] Performance metrics tracked
- [ ] Health checks implemented
- [ ] Load testing performed
- [ ] Profiling in production

## Summary

| Area | Impact | Difficulty | Priority |
|------|--------|------------|----------|
| Database Indexing | High | Low | 🔴 Critical |
| Caching | High | Medium | 🔴 Critical |
| Query Optimization | High | Medium | 🔴 Critical |
| Connection Pooling | Medium | Low | 🟡 Important |
| Async Processing | High | Medium | 🔴 Critical |
| Response Compression | Medium | Low | 🟡 Important |
| Load Testing | High | Medium | 🟡 Important |

## Next Steps

- [Database Design](06-DATABASE-DESIGN.md)
- [Additional Patterns](07-DESIGN-PATTERNS.md)
- [Cloud-Native AWS](09-CLOUD-NATIVE-AWS.md)
