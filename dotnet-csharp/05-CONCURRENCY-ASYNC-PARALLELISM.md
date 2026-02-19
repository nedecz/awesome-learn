# Concurrency, Async, and Parallelism in .NET

## Table of Contents

1. [Overview](#overview)
2. [Async/Await Deep Dive](#asyncawait-deep-dive)
3. [Task Parallel Library (TPL)](#task-parallel-library-tpl)
4. [PLINQ (Parallel LINQ)](#plinq-parallel-linq)
5. [System.Threading.Channels](#systemthreadingchannels)
6. [Thread Safety and Synchronization](#thread-safety-and-synchronization)
7. [Job Queuing Patterns](#job-queuing-patterns)
8. [Best Practices](#best-practices)
9. [Next Steps](#next-steps)

## Overview

This document covers the concurrency, asynchronous, and parallel programming models in .NET — from async/await fundamentals and the Task Parallel Library to channels, synchronization primitives, and production job queuing patterns.

### Target Audience

- .NET developers writing async services and APIs
- Developers implementing background processing and parallel workloads
- Architects designing high-throughput, concurrent .NET systems

### Scope

- Async/await internals, ValueTask, IAsyncEnumerable, cancellation
- Task Parallel Library and PLINQ for CPU-bound parallelism
- System.Threading.Channels for producer-consumer pipelines
- Synchronization primitives and thread-safe collections
- Job queuing with BackgroundService, Channels, and third-party schedulers

### Concurrency vs Parallelism vs Asynchrony

```
┌─────────────────────────────────────────────────────────────────┐
│                      Concurrency                                │
│   "Dealing with multiple things at once" (structure)            │
│                                                                 │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│   │   Task A      │  │   Task B      │  │   Task C      │       │
│   │  ─────►       │  │  ─────►       │  │  ─────►       │       │
│   │    ◄─────     │  │    ◄─────     │  │    ◄─────     │       │
│   └──────────────┘  └──────────────┘  └──────────────┘         │
│         Interleaved on one or more threads                      │
├─────────────────────────────────────────────────────────────────┤
│                      Parallelism                                │
│   "Doing multiple things at the same time" (execution)          │
│                                                                 │
│   Core 1: ═══════════════► Task A                               │
│   Core 2: ═══════════════► Task B                               │
│   Core 3: ═══════════════► Task C                               │
│         Simultaneous execution on multiple cores                │
├─────────────────────────────────────────────────────────────────┤
│                      Asynchrony                                 │
│   "Non-blocking waiting" (mechanism)                            │
│                                                                 │
│   Thread 1: ──► Start I/O ──► [released] ──► Resume on callback │
│                     │                            ▲               │
│                     └──── OS / hardware ─────────┘               │
│         Thread not blocked during I/O wait                      │
└─────────────────────────────────────────────────────────────────┘
```

| Concept | Best For | .NET Mechanism | Blocks Thread? |
|---------|----------|----------------|----------------|
| **Concurrency** | Managing multiple in-flight operations | `Task`, `async/await`, Channels | No |
| **Parallelism** | CPU-bound work across cores | `Parallel.ForEachAsync`, PLINQ | Yes (worker threads) |
| **Asynchrony** | I/O-bound waiting (HTTP, DB, file) | `async/await`, `ValueTask` | No |

## Async/Await Deep Dive

### Task and Task&lt;T&gt;

`Task` represents a unit of asynchronous work. `Task<T>` adds a result value. The runtime schedules continuations on the thread pool when the awaited operation completes.

```csharp
// I/O-bound — no thread is consumed while waiting
public async Task<Order> GetOrderAsync(int id, CancellationToken ct)
{
    await using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync(ct);

    return await connection.QuerySingleOrDefaultAsync<Order>(
        "SELECT * FROM Orders WHERE Id = @Id",
        new { Id = id });
}

// CPU-bound — offload to thread pool only when called from UI thread
public async Task<byte[]> CompressAsync(byte[] data, CancellationToken ct)
{
    return await Task.Run(() =>
    {
        using var output = new MemoryStream();
        using (var gzip = new GZipStream(output, CompressionLevel.Optimal))
        {
            gzip.Write(data);
        }
        return output.ToArray();
    }, ct);
}
```

### ValueTask&lt;T&gt;

`ValueTask<T>` avoids the heap allocation of `Task<T>` when the result is already available synchronously. Use it on hot paths that frequently complete synchronously.

```csharp
// Cache hit returns synchronously — ValueTask avoids allocation
public ValueTask<Product?> GetProductAsync(int id)
{
    if (_cache.TryGetValue(id, out var product))
    {
        return ValueTask.FromResult<Product?>(product);
    }

    return new ValueTask<Product?>(LoadProductFromDbAsync(id));
}

private async Task<Product?> LoadProductFromDbAsync(int id)
{
    var product = await _dbContext.Products.FindAsync(id);
    if (product is not null)
    {
        _cache[id] = product;
    }
    return product;
}
```

| Type | Allocation | Await Multiple Times | Use When |
|------|------------|---------------------|----------|
| `Task<T>` | Heap (once) | ✅ Yes | General-purpose async |
| `ValueTask<T>` | Stack (sync) / Heap (async) | ❌ No — await exactly once | Hot paths, frequent sync completion |

### Async State Machine

When the compiler encounters `async`, it generates a state machine struct (boxed to a class if the first `await` is not already completed). Each `await` is a suspension point.

```
┌──────────────────────────────────────────────────────────────┐
│                  Async State Machine Flow                     │
│                                                              │
│  Caller                State Machine              Awaiter    │
│    │                       │                         │       │
│    │──── CallAsync() ─────►│                         │       │
│    │                       │── state = 0             │       │
│    │                       │── invoke awaiter ──────►│       │
│    │                       │                         │       │
│    │                       │   IsCompleted?           │       │
│    │                       │   ┌─── Yes ──► GetResult + continue   │
│    │                       │   │                     │       │
│    │                       │   └─── No ──► AwaitOnCompleted        │
│    │◄── returns Task ──────│        (suspend)        │       │
│    │                       │                         │       │
│    │                       │◄── MoveNext() callback ─│       │
│    │                       │── state = 1             │       │
│    │                       │── next await or return  │       │
│    │                       │                         │       │
│    │◄── Task completes ────│                         │       │
└──────────────────────────────────────────────────────────────┘
```

### ConfigureAwait

`ConfigureAwait(false)` tells the runtime not to marshal the continuation back to the original `SynchronizationContext`. Use it in library code to avoid deadlocks and improve performance.

```csharp
// Library code — no need to resume on the original context
public async Task<string> FetchDataAsync(string url)
{
    using var client = new HttpClient();
    var response = await client.GetAsync(url).ConfigureAwait(false);
    var content = await response.Content.ReadAsStringAsync().ConfigureAwait(false);
    return content;
}
```

| Context | ConfigureAwait(false) | Reason |
|---------|----------------------|--------|
| ASP.NET Core | Optional (no SynchronizationContext) | No-op, but harmless and signals intent |
| Class libraries | ✅ Always use | Prevents deadlocks when consumed by UI apps |
| UI apps (WPF, MAUI) | ❌ Not on UI-updating code | Need to return to UI thread |

### IAsyncEnumerable&lt;T&gt;

Stream results one at a time without buffering the entire collection.

```csharp
// Streaming rows from the database
public async IAsyncEnumerable<Product> GetProductsAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync(ct);

    await using var command = new SqlCommand("SELECT * FROM Products", connection);
    await using var reader = await command.ExecuteReaderAsync(ct);

    while (await reader.ReadAsync(ct))
    {
        yield return new Product
        {
            Id = reader.GetInt32(0),
            Name = reader.GetString(1),
            Price = reader.GetDecimal(2)
        };
    }
}

// Consuming the stream
await foreach (var product in GetProductsAsync(cancellationToken))
{
    Console.WriteLine($"{product.Name}: {product.Price:C}");
}
```

### Cancellation Tokens

Always accept and forward `CancellationToken` to support cooperative cancellation.

```csharp
public async Task ProcessOrdersAsync(CancellationToken ct)
{
    var orders = await _repository.GetPendingOrdersAsync(ct);

    foreach (var order in orders)
    {
        ct.ThrowIfCancellationRequested();

        await _paymentService.ChargeAsync(order, ct);
        await _notificationService.SendConfirmationAsync(order, ct);
    }
}

// Creating a linked token with timeout
using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
    ct, timeoutCts.Token);

await ProcessOrdersAsync(linkedCts.Token);
```

### SynchronizationContext

ASP.NET Core does **not** have a `SynchronizationContext`. Desktop frameworks (WPF, WinForms, MAUI) do. The context determines which thread the continuation runs on after `await`.

```
┌────────────────────────────────────────────┐
│            SynchronizationContext           │
│                                            │
│  ASP.NET Core:  null (thread pool)         │
│  WPF / MAUI:    DispatcherSynchronization  │
│  WinForms:      WindowsFormsSynchronization│
│  Console:       null (thread pool)         │
└────────────────────────────────────────────┘
```

## Task Parallel Library (TPL)

### Parallel.ForEachAsync

The preferred way to process collections concurrently with async I/O (introduced in .NET 6).

```csharp
var urls = new[] { "https://api.a.com", "https://api.b.com", "https://api.c.com" };

await Parallel.ForEachAsync(urls, new ParallelOptions
{
    MaxDegreeOfParallelism = 4,
    CancellationToken = ct
}, async (url, token) =>
{
    using var client = new HttpClient();
    var result = await client.GetStringAsync(url, token);
    Console.WriteLine($"{url}: {result.Length} chars");
});
```

### Parallel.ForEach and Parallel.For

For synchronous CPU-bound parallelism.

```csharp
// CPU-bound image processing
Parallel.ForEach(images, new ParallelOptions
{
    MaxDegreeOfParallelism = Environment.ProcessorCount
}, image =>
{
    var thumbnail = GenerateThumbnail(image);
    SaveThumbnail(thumbnail);
});

// Parallel.For with thread-local state
Parallel.For(0, items.Length,
    () => 0L,  // localInit
    (i, state, localSum) =>
    {
        localSum += ComputeExpensiveValue(items[i]);
        return localSum;
    },
    localSum => Interlocked.Add(ref totalSum, localSum)  // localFinally
);
```

### Task.WhenAll and Task.WhenAny

```csharp
// Fan-out: execute multiple async operations concurrently
var userTask = _userService.GetUserAsync(userId, ct);
var ordersTask = _orderService.GetOrdersAsync(userId, ct);
var prefsTask = _prefsService.GetPreferencesAsync(userId, ct);

await Task.WhenAll(userTask, ordersTask, prefsTask);

var user = await userTask;
var orders = await ordersTask;
var prefs = await prefsTask;

// First-completed: race multiple sources
var primaryTask = _primaryDb.GetAsync(key, ct);
var replicaTask = _replicaDb.GetAsync(key, ct);

var completedTask = await Task.WhenAny(primaryTask, replicaTask);
var result = await completedTask;
```

### TaskCompletionSource

Bridge callback-based APIs into the Task world.

```csharp
public Task<string> ReadLineAsync(StreamReader reader)
{
    var tcs = new TaskCompletionSource<string>();

    reader.BaseStream.BeginRead(buffer, 0, buffer.Length, ar =>
    {
        try
        {
            var bytesRead = reader.BaseStream.EndRead(ar);
            tcs.SetResult(Encoding.UTF8.GetString(buffer, 0, bytesRead));
        }
        catch (Exception ex)
        {
            tcs.SetException(ex);
        }
    }, null);

    return tcs.Task;
}
```

### TPL Comparison Table

| API | Sync / Async | Best For | Thread Usage |
|-----|-------------|----------|--------------|
| `Parallel.For` | Sync | CPU-bound loops with index | Thread pool |
| `Parallel.ForEach` | Sync | CPU-bound collection processing | Thread pool |
| `Parallel.ForEachAsync` | Async | I/O-bound concurrent processing | Thread pool + async |
| `Task.WhenAll` | Async | Fan-out multiple independent tasks | Depends on tasks |
| `Task.WhenAny` | Async | First-wins / timeout racing | Depends on tasks |
| `TaskCompletionSource` | Async | Wrapping callback APIs as Tasks | Caller thread |

## PLINQ (Parallel LINQ)

### AsParallel

PLINQ partitions a source sequence and executes the query in parallel across multiple threads.

```csharp
// CPU-bound: hash computation in parallel
var hashes = documents
    .AsParallel()
    .WithDegreeOfParallelism(Environment.ProcessorCount)
    .Select(doc => new
    {
        doc.Id,
        Hash = ComputeSha256(doc.Content)
    })
    .ToList();
```

### WithDegreeOfParallelism and ForAll

```csharp
// Limit parallelism
var results = source
    .AsParallel()
    .WithDegreeOfParallelism(4)
    .Where(x => x.IsValid)
    .Select(Transform)
    .ToList();

// ForAll — process in parallel without merging (side-effect only)
source
    .AsParallel()
    .Where(x => x.NeedsUpdate)
    .ForAll(x => UpdateRecord(x));
```

### When PLINQ Helps vs Hurts

| Scenario | PLINQ? | Reason |
|----------|--------|--------|
| CPU-bound transform on large collections | ✅ Yes | Scales across cores |
| Small collections (< 1,000 items) | ❌ No | Partitioning overhead exceeds benefit |
| I/O-bound operations | ❌ No | Use `Parallel.ForEachAsync` instead |
| Order-dependent processing | ❌ No | PLINQ does not preserve order by default |
| Operations with heavy locking | ❌ No | Contention negates parallelism gains |
| Independent pure functions | ✅ Yes | No shared state, ideal for parallelism |

## System.Threading.Channels

Channels provide a high-performance, thread-safe, async-ready producer-consumer data structure. They are the recommended replacement for `BlockingCollection<T>` in async code.

### Channel Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                     Channel<T> Pipeline                          │
│                                                                  │
│  Producer(s)              Channel               Consumer(s)      │
│  ┌──────────┐     ┌────────────────────┐     ┌──────────────┐   │
│  │ Write    │────►│  ChannelWriter<T>  │     │ ChannelReader │   │
│  │ items    │     │                    │     │   <T>         │   │
│  │          │     │  ┌──┬──┬──┬──┬──┐  │     │              │   │
│  └──────────┘     │  │T │T │T │T │T │  │────►│  Read items  │   │
│                   │  └──┴──┴──┴──┴──┘  │     │              │   │
│  ┌──────────┐     │    (buffer)        │     └──────────────┘   │
│  │ Write    │────►│                    │     ┌──────────────┐   │
│  │ items    │     │  Bounded: capacity │     │ ChannelReader │   │
│  └──────────┘     │  Unbounded: ∞      │────►│   <T>         │   │
│                   └────────────────────┘     └──────────────┘   │
│                                                                  │
│  Bounded:   blocks/drops when full    SingleReader optimization  │
│  Unbounded: grows without limit       SingleWriter optimization  │
└──────────────────────────────────────────────────────────────────┘
```

### Bounded vs Unbounded Channels

| Feature | Bounded | Unbounded |
|---------|---------|-----------|
| Creation | `Channel.CreateBounded<T>(capacity)` | `Channel.CreateUnbounded<T>()` |
| Backpressure | ✅ Writer waits when full | ❌ No backpressure |
| Memory safety | ✅ Capped memory usage | ❌ Can grow unbounded |
| Use case | Rate-limited pipelines | Low-latency, trusted producers |

### Full Producer-Consumer Example

```csharp
public sealed class OrderProcessingPipeline
{
    private readonly Channel<Order> _channel;
    private readonly ILogger<OrderProcessingPipeline> _logger;

    public OrderProcessingPipeline(ILogger<OrderProcessingPipeline> logger)
    {
        _logger = logger;
        _channel = Channel.CreateBounded<Order>(new BoundedChannelOptions(100)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = false,
            SingleWriter = false
        });
    }

    public ChannelWriter<Order> Writer => _channel.Writer;
    public ChannelReader<Order> Reader => _channel.Reader;

    // Producer
    public async Task EnqueueOrderAsync(Order order, CancellationToken ct)
    {
        await _channel.Writer.WriteAsync(order, ct);
        _logger.LogInformation("Enqueued order {OrderId}", order.Id);
    }

    // Consumer
    public async Task ProcessOrdersAsync(CancellationToken ct)
    {
        await foreach (var order in _channel.Reader.ReadAllAsync(ct))
        {
            try
            {
                _logger.LogInformation("Processing order {OrderId}", order.Id);
                await ProcessOrderAsync(order, ct);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to process order {OrderId}", order.Id);
            }
        }
    }

    private async Task ProcessOrderAsync(Order order, CancellationToken ct)
    {
        // Simulate processing
        await Task.Delay(100, ct);
        order.Status = OrderStatus.Completed;
    }
}
```

## Thread Safety and Synchronization

### lock Statement

The simplest synchronization primitive. Uses `Monitor` internally. In C# 13 / .NET 10, prefer `System.Threading.Lock` for new code.

```csharp
// .NET 10 / C# 13 — new Lock type
private readonly Lock _lock = new();

public void AddItem(string item)
{
    lock (_lock)
    {
        _items.Add(item);
    }
}

// Pre-.NET 9 — object-based lock
private readonly object _syncRoot = new();

public void AddItemLegacy(string item)
{
    lock (_syncRoot)
    {
        _items.Add(item);
    }
}
```

### SemaphoreSlim

Async-compatible throttle for limiting concurrent access.

```csharp
private readonly SemaphoreSlim _semaphore = new(maxConcurrency: 5);

public async Task<T> ThrottledCallAsync<T>(Func<Task<T>> action, CancellationToken ct)
{
    await _semaphore.WaitAsync(ct);
    try
    {
        return await action();
    }
    finally
    {
        _semaphore.Release();
    }
}
```

### Concurrent Collections

```csharp
// ConcurrentDictionary — thread-safe dictionary
var cache = new ConcurrentDictionary<string, byte[]>();

cache.GetOrAdd("key", key => ComputeExpensiveValue(key));

cache.AddOrUpdate("key",
    addValueFactory: key => ComputeNew(key),
    updateValueFactory: (key, existing) => ComputeUpdated(key, existing));

// ConcurrentQueue — thread-safe FIFO queue
var queue = new ConcurrentQueue<WorkItem>();
queue.Enqueue(new WorkItem("task-1"));

if (queue.TryDequeue(out var item))
{
    Process(item);
}
```

### Interlocked

Lock-free atomic operations for simple counters and flags.

```csharp
private long _requestCount;
private int _isRunning; // 0 = false, 1 = true

public void IncrementRequests()
{
    Interlocked.Increment(ref _requestCount);
}

public bool TryStart()
{
    return Interlocked.CompareExchange(ref _isRunning, 1, 0) == 0;
}

public long GetCount() => Interlocked.Read(ref _requestCount);
```

### ReaderWriterLockSlim

Optimized for scenarios with many reads and few writes.

```csharp
private readonly ReaderWriterLockSlim _rwLock = new();
private readonly Dictionary<string, string> _data = new();

public string? Read(string key)
{
    _rwLock.EnterReadLock();
    try
    {
        return _data.GetValueOrDefault(key);
    }
    finally
    {
        _rwLock.ExitReadLock();
    }
}

public void Write(string key, string value)
{
    _rwLock.EnterWriteLock();
    try
    {
        _data[key] = value;
    }
    finally
    {
        _rwLock.ExitWriteLock();
    }
}
```

### Synchronization Primitives Comparison

| Primitive | Async Support | Use Case | Overhead |
|-----------|--------------|----------|----------|
| `lock` / `Lock` | ❌ No | Short critical sections | Very low |
| `SemaphoreSlim` | ✅ Yes | Throttling, async mutual exclusion | Low |
| `ReaderWriterLockSlim` | ❌ No | Many readers, few writers | Medium |
| `Interlocked` | N/A (lock-free) | Counters, flags, CAS operations | Lowest |
| `ConcurrentDictionary` | N/A (built-in) | Thread-safe key-value store | Low |
| `ConcurrentQueue` | N/A (built-in) | Thread-safe FIFO queue | Low |
| `Channel<T>` | ✅ Yes | Async producer-consumer | Low |

## Job Queuing Patterns

### BackgroundService and IHostedService

.NET provides `BackgroundService` (which implements `IHostedService`) as the foundation for long-running background work in hosted applications.

```csharp
public sealed class OrderProcessingWorker : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly Channel<OrderJob> _jobChannel;
    private readonly ILogger<OrderProcessingWorker> _logger;

    public OrderProcessingWorker(
        IServiceScopeFactory scopeFactory,
        Channel<OrderJob> jobChannel,
        ILogger<OrderProcessingWorker> logger)
    {
        _scopeFactory = scopeFactory;
        _jobChannel = jobChannel;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Order processing worker started");

        await foreach (var job in _jobChannel.Reader.ReadAllAsync(stoppingToken))
        {
            try
            {
                await using var scope = _scopeFactory.CreateAsyncScope();
                var processor = scope.ServiceProvider
                    .GetRequiredService<IOrderProcessor>();

                await processor.ProcessAsync(job, stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to process job {JobId}", job.Id);
            }
        }
    }
}
```

### Timer-Based Background Jobs

```csharp
public sealed class MetricsCollectorService : BackgroundService
{
    private readonly ILogger<MetricsCollectorService> _logger;
    private readonly TimeSpan _interval = TimeSpan.FromMinutes(1);

    public MetricsCollectorService(ILogger<MetricsCollectorService> logger)
    {
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(_interval);

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                _logger.LogDebug("Collecting metrics...");
                await CollectMetricsAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Metrics collection failed");
            }
        }
    }

    private Task CollectMetricsAsync(CancellationToken ct)
    {
        // Collect and push metrics
        return Task.CompletedTask;
    }
}
```

### Complete Producer-Consumer Job Queue

```csharp
// Job definition
public sealed record BackgroundJob(
    string Id,
    string Type,
    string Payload,
    DateTimeOffset CreatedAt);

// Job queue service (singleton)
public sealed class JobQueue
{
    private readonly Channel<BackgroundJob> _channel;

    public JobQueue(int capacity = 500)
    {
        _channel = Channel.CreateBounded<BackgroundJob>(
            new BoundedChannelOptions(capacity)
            {
                FullMode = BoundedChannelFullMode.Wait,
                SingleReader = false,
                SingleWriter = false
            });
    }

    public async ValueTask EnqueueAsync(
        BackgroundJob job, CancellationToken ct = default)
    {
        await _channel.Writer.WriteAsync(job, ct);
    }

    public IAsyncEnumerable<BackgroundJob> DequeueAllAsync(
        CancellationToken ct = default)
    {
        return _channel.Reader.ReadAllAsync(ct);
    }

    public void Complete() => _channel.Writer.Complete();
}

// Job processor worker (hosted service, multiple instances for concurrency)
public sealed class JobProcessorWorker : BackgroundService
{
    private readonly JobQueue _queue;
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<JobProcessorWorker> _logger;

    public JobProcessorWorker(
        JobQueue queue,
        IServiceScopeFactory scopeFactory,
        ILogger<JobProcessorWorker> logger)
    {
        _queue = queue;
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var job in _queue.DequeueAllAsync(stoppingToken))
        {
            using var scope = _scopeFactory.CreateScope();
            var handler = ResolveHandler(scope.ServiceProvider, job.Type);

            try
            {
                _logger.LogInformation(
                    "Processing job {JobId} of type {Type}", job.Id, job.Type);
                await handler.HandleAsync(job, stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex,
                    "Job {JobId} failed after processing", job.Id);
            }
        }
    }

    private static IJobHandler ResolveHandler(
        IServiceProvider sp, string jobType)
    {
        return jobType switch
        {
            "email" => sp.GetRequiredService<EmailJobHandler>(),
            "report" => sp.GetRequiredService<ReportJobHandler>(),
            _ => sp.GetRequiredService<DefaultJobHandler>()
        };
    }
}

// Registration
builder.Services.AddSingleton<JobQueue>();
builder.Services.AddScoped<EmailJobHandler>();
builder.Services.AddScoped<ReportJobHandler>();
builder.Services.AddScoped<DefaultJobHandler>();
builder.Services.AddHostedService<JobProcessorWorker>();

// Enqueue from a controller
app.MapPost("/api/jobs", async (JobRequest request, JobQueue queue) =>
{
    var job = new BackgroundJob(
        Id: Guid.NewGuid().ToString(),
        Type: request.Type,
        Payload: request.Payload,
        CreatedAt: DateTimeOffset.UtcNow);

    await queue.EnqueueAsync(job);
    return Results.Accepted($"/api/jobs/{job.Id}", job);
});
```

### Third-Party Job Scheduling Libraries

| Feature | Channel + BackgroundService | Hangfire | Quartz.NET | Coravel |
|---------|----------------------------|----------|------------|---------|
| **Persistence** | ❌ In-memory only | ✅ SQL, Redis | ✅ SQL, RAM | ❌ In-memory |
| **Dashboard** | ❌ None | ✅ Built-in web UI | ❌ None (third-party) | ❌ None |
| **Retries** | Manual | ✅ Automatic | ✅ Configurable | ✅ Basic |
| **Cron scheduling** | Manual (PeriodicTimer) | ✅ Built-in | ✅ Full cron support | ✅ Built-in |
| **Distributed** | ❌ Single process | ✅ Yes | ✅ Clustered | ❌ Single process |
| **Complexity** | Low | Medium | High | Low |
| **Best for** | Simple in-process queues | Web app background jobs | Enterprise scheduling | Simple scheduled tasks |
| **License** | MIT (.NET) | LGPL (free) / Commercial | Apache 2.0 | MIT |

## Best Practices

### Do ✅

- ✅ Use `async`/`await` all the way down — never block on async code with `.Result` or `.Wait()`
- ✅ Accept and forward `CancellationToken` in every async method signature
- ✅ Use `ValueTask<T>` on hot paths that frequently complete synchronously
- ✅ Use `ConfigureAwait(false)` in library code
- ✅ Use `Parallel.ForEachAsync` for concurrent I/O-bound collection processing
- ✅ Use `Channel<T>` for async producer-consumer instead of `BlockingCollection<T>`
- ✅ Prefer `SemaphoreSlim` over `lock` when you need async-compatible throttling
- ✅ Use `ConcurrentDictionary` and `ConcurrentQueue` for thread-safe collections
- ✅ Use `Interlocked` for simple atomic counters and flags
- ✅ Scope DI services correctly in `BackgroundService` using `IServiceScopeFactory`
- ✅ Use `System.Threading.Lock` in .NET 10 / C# 13 for new lock declarations
- ✅ Use `PeriodicTimer` instead of `Task.Delay` loops for timer-based background work
- ✅ Limit `MaxDegreeOfParallelism` to avoid thread pool starvation

### Don't ❌

- ❌ Don't use `async void` — only exception is event handlers
- ❌ Don't call `.Result` or `.Wait()` on tasks — causes deadlocks in UI/legacy ASP.NET
- ❌ Don't use `Task.Run` in ASP.NET Core request handling — the request is already on the thread pool
- ❌ Don't await a `ValueTask<T>` more than once — undefined behavior
- ❌ Don't use `Thread.Sleep` in async code — use `await Task.Delay` instead
- ❌ Don't capture variables in `Parallel.ForEach` without synchronization
- ❌ Don't use `BlockingCollection<T>` in new async code — use `Channel<T>`
- ❌ Don't forget to dispose `CancellationTokenSource` instances
- ❌ Don't create unbounded channels without understanding memory implications
- ❌ Don't use PLINQ for I/O-bound work — it blocks thread pool threads
- ❌ Don't hold locks across `await` boundaries — use `SemaphoreSlim` instead

## Next Steps

Continue to [Dependency Injection](06-DEPENDENCY-INJECTION.md) for registration patterns, service lifetimes, keyed services, and the Options pattern. Also review [Best Practices](03-BEST-PRACTICES.md) for broader .NET production readiness guidance.

### Related Documents

1. **[Best Practices](03-BEST-PRACTICES.md)** — Async/await patterns, memory management, security
2. **[Dependency Injection](06-DEPENDENCY-INJECTION.md)** — DI container, lifetimes, keyed services
3. **[Services](02-SERVICES.md)** — BackgroundService, hosted services, worker patterns
4. **[Style Guide](04-STYLE-GUIDE.md)** — Naming conventions, formatting, analyzers

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial concurrency, async, and parallelism documentation |
