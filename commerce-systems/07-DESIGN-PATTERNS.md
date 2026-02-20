# Design Patterns for Payment Systems

## Overview

This comprehensive guide covers design patterns, architectural approaches, and specialized techniques for building robust, scalable payment systems. Patterns are organized by category for easy reference.

> **Note:** This document consolidates content from the former Additional Patterns, Additional Architectural Patterns, and Specialized Patterns documents.

## Table of Contents

### Part 1: Architectural Styles
1. [Hexagonal Architecture (Ports & Adapters)](#hexagonal-architecture-ports-adapters)
2. [Onion Architecture](#onion-architecture)

### Part 2: Resilience Patterns
3. [Circuit Breaker Pattern](#circuit-breaker-pattern)
4. [Retry Pattern](#retry-pattern)
5. [Bulkhead Pattern](#bulkhead-pattern)
6. [Chaos Engineering for Payments](#chaos-engineering)

### Part 3: Integration Patterns
7. [Saga Pattern](#saga-pattern)
8. [Idempotency Pattern](#idempotency-pattern)
9. [Rate Limiting & Throttling Pattern](#rate-limiting-throttling-pattern)
10. [Reconciliation Pattern](#reconciliation-pattern)
11. [Notification Pattern](#notification-pattern)
12. [Priority Queue Pattern](#priority-queue-pattern)
13. [Claim Check Pattern](#claim-check-pattern)
14. [Pipes and Filters Pattern](#pipes-and-filters-pattern)
15. [Chain of Responsibility Pattern](#chain-of-responsibility-pattern)

### Part 4: Data Patterns
16. [Cache-Aside Pattern](#cache-aside-pattern)
17. [Materialized View Pattern](#materialized-view-pattern)
18. [Write-Behind Caching](#write-behind-caching)
19. [Multi-Currency Pattern](#multi-currency-pattern)
20. [Payment State Machine](#payment-state-machine)
21. [Specification Pattern](#specification-pattern)
22. [Command Query Separation (Not CQRS)](#command-query-separation-not-cqrs)
23. [Memento Pattern (State Recovery)](#memento-pattern-state-recovery)
24. [Time-Series Data Pattern](#time-series-data-pattern)

### Part 5: Infrastructure Patterns
25. [API Gateway Pattern](#api-gateway-pattern)
26. [Sidecar Pattern](#sidecar-pattern)
27. [Ambassador Pattern](#ambassador-pattern)
28. [Gatekeeper Pattern](#gatekeeper-pattern)
29. [Valet Key Pattern](#valet-key-pattern)

### Part 6: Deployment Patterns
30. [Blue-Green Deployment for Payments](#blue-green-deployment-for-payments)
31. [Canary Release Pattern](#canary-release-pattern)
32. [Feature Flags for Payments](#feature-flags-for-payments)

### Part 7: Distributed Systems
33. [Eventual Consistency Patterns](#eventual-consistency-patterns)
34. [Two-Phase Commit (2PC)](#two-phase-commit-2pc)
35. [Distributed Lock Pattern](#distributed-lock-pattern)
36. [Leader Election Pattern](#leader-election-pattern)
37. [Sharding Pattern](#sharding-pattern)

### Part 8: Anti-Patterns & Domain-Specific
38. [Anti-Pattern Avoidance](#anti-pattern-avoidance)
39. [Geofencing Pattern](#geofencing-pattern)

---

# Part 1: Architectural Styles
## Hexagonal Architecture (Ports & Adapters)

### Concept

Domain logic at the center, infrastructure at the edges. All external dependencies go through ports (interfaces).

```
        ┌─────────────────────────────────────┐
        │     Infrastructure Layer            │
        │  ┌─────────────────────────────┐   │
        │  │   Application Services      │   │
        │  │  ┌───────────────────────┐ │   │
        │  │  │   Domain Layer        │ │   │
        │  │  │  (Business Logic)     │ │   │
        │  │  └───────────────────────┘ │   │
        │  └─────────────────────────────┘   │
        └─────────────────────────────────────┘

Ports (Interfaces) connect to Adapters (Implementations)
```

### Implementation

```csharp
// ============ DOMAIN LAYER (Core - No dependencies) ============

namespace PaymentSystem.Domain
{
    // Pure business logic
    public class Payment
    {
        public Guid Id { get; private set; }
        public decimal Amount { get; private set; }
        public string Currency { get; private set; }
        public PaymentStatus Status { get; private set; }

        public static Payment Create(decimal amount, string currency)
        {
            if (amount <= 0)
                throw new DomainException("Amount must be positive");

            return new Payment
            {
                Id = Guid.NewGuid(),
                Amount = amount,
                Currency = currency,
                Status = PaymentStatus.Pending
            };
        }

        public void Process(string transactionId)
        {
            if (Status != PaymentStatus.Pending)
                throw new DomainException($"Cannot process payment in {Status} status");

            Status = PaymentStatus.Completed;
            // Domain logic only, no infrastructure concerns
        }
    }
}

// ============ PORTS (Interfaces - Domain defines what it needs) ============

namespace PaymentSystem.Application.Ports
{
    // Primary Ports (Driven by external actors)
    public interface IPaymentService
    {
        Task<PaymentResult> ProcessPaymentAsync(ProcessPaymentCommand command);
    }

    // Secondary Ports (Needed by domain)
    public interface IPaymentRepository
    {
        Task<Payment> GetByIdAsync(Guid id);
        Task SaveAsync(Payment payment);
    }

    public interface IPaymentGateway
    {
        Task<GatewayResult> ChargeAsync(decimal amount, string currency, string methodId);
    }

    public interface INotificationService
    {
        Task SendPaymentConfirmationAsync(string email, string transactionId);
    }
}

// ============ APPLICATION LAYER (Use Cases) ============

namespace PaymentSystem.Application
{
    public class PaymentService : IPaymentService
    {
        private readonly IPaymentRepository _repository;
        private readonly IPaymentGateway _gateway;
        private readonly INotificationService _notifications;

        // Dependencies injected as ports (interfaces)
        public PaymentService(
            IPaymentRepository repository,
            IPaymentGateway gateway,
            INotificationService notifications)
        {
            _repository = repository;
            _gateway = gateway;
            _notifications = notifications;
        }

        public async Task<PaymentResult> ProcessPaymentAsync(ProcessPaymentCommand command)
        {
            // Use case orchestration
            var payment = Payment.Create(command.Amount, command.Currency);
            await _repository.SaveAsync(payment);

            var result = await _gateway.ChargeAsync(
                payment.Amount,
                payment.Currency,
                command.PaymentMethodId);

            if (result.IsSuccess)
            {
                payment.Process(result.TransactionId);
                await _repository.SaveAsync(payment);
                await _notifications.SendPaymentConfirmationAsync(
                    command.CustomerEmail,
                    result.TransactionId);
            }

            return PaymentResult.FromDomain(payment);
        }
    }
}

// ============ ADAPTERS (Infrastructure implementations) ============

namespace PaymentSystem.Infrastructure.Adapters
{
    // Database Adapter
    public class SqlPaymentRepository : IPaymentRepository
    {
        private readonly ApplicationDbContext _context;

        public async Task<Payment> GetByIdAsync(Guid id)
        {
            var entity = await _context.Payments.FindAsync(id);
            return MapToDomain(entity); // Map from EF entity to domain
        }

        public async Task SaveAsync(Payment payment)
        {
            var entity = MapToEntity(payment); // Map from domain to EF entity
            _context.Payments.Update(entity);
            await _context.SaveChangesAsync();
        }
    }

    // Stripe Adapter
    public class StripePaymentGateway : IPaymentGateway
    {
        private readonly IStripeClient _stripe;

        public async Task<GatewayResult> ChargeAsync(
            decimal amount,
            string currency,
            string methodId)
        {
            var options = new PaymentIntentCreateOptions
            {
                Amount = (long)(amount * 100),
                Currency = currency,
                PaymentMethod = methodId,
                Confirm = true
            };

            var intent = await _stripe.PaymentIntents.CreateAsync(options);

            return new GatewayResult
            {
                IsSuccess = intent.Status == "succeeded",
                TransactionId = intent.Id
            };
        }
    }

    // Email Adapter
    public class SendGridNotificationService : INotificationService
    {
        private readonly ISendGridClient _sendGrid;

        public async Task SendPaymentConfirmationAsync(string email, string transactionId)
        {
            var message = new SendGridMessage
            {
                From = new EmailAddress("noreply@company.com"),
                Subject = "Payment Confirmed",
                PlainTextContent = $"Transaction {transactionId} completed"
            };
            message.AddTo(email);

            await _sendGrid.SendEmailAsync(message);
        }
    }
}

// ============ DEPENDENCY INJECTION (Wire it up) ============

public void ConfigureServices(IServiceCollection services)
{
    // Register adapters for ports
    services.AddScoped<IPaymentRepository, SqlPaymentRepository>();
    services.AddScoped<IPaymentGateway, StripePaymentGateway>();
    services.AddScoped<INotificationService, SendGridNotificationService>();

    // Register application services
    services.AddScoped<IPaymentService, PaymentService>();
}
```

**Benefits:**
- ✅ Domain logic completely independent of infrastructure
- ✅ Easy to test (mock any port)
- ✅ Easy to swap implementations (Stripe → PayPal)
- ✅ Clear separation of concerns

---


---

## Onion Architecture

### Concept

Similar to Hexagonal but with explicit layers. Dependencies flow inward.

```
┌──────────────────────────────────────┐
│  Infrastructure (DB, APIs, UI)       │ ← Depends on everything
│  ┌────────────────────────────────┐  │
│  │  Application Services          │  │ ← Depends on Domain
│  │  ┌──────────────────────────┐ │  │
│  │  │  Domain Services         │ │  │ ← Depends on Domain Model
│  │  │  ┌────────────────────┐  │ │  │
│  │  │  │  Domain Model      │  │ │  │ ← No dependencies
│  │  │  │  (Entities, VOs)   │  │ │  │
│  │  │  └────────────────────┘  │ │  │
│  │  └──────────────────────────┘ │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

### Implementation

```csharp
// ============ Core (innermost) ============

namespace PaymentSystem.Core.Domain
{
    // Value Objects
    public class Money
    {
        public decimal Amount { get; }
        public string Currency { get; }

        public Money(decimal amount, string currency)
        {
            if (amount < 0)
                throw new ArgumentException("Amount cannot be negative");

            Amount = amount;
            Currency = currency;
        }

        public Money Add(Money other)
        {
            if (Currency != other.Currency)
                throw new InvalidOperationException("Cannot add different currencies");

            return new Money(Amount + other.Amount, Currency);
        }
    }

    // Entities
    public class Payment
    {
        public Guid Id { get; private set; }
        public Money Amount { get; private set; }
        public PaymentStatus Status { get; private set; }

        // Rich domain model with behavior
        public void ApplyRefund(Money refundAmount)
        {
            if (Status != PaymentStatus.Completed)
                throw new InvalidOperationException("Can only refund completed payments");

            if (refundAmount.Amount > Amount.Amount)
                throw new InvalidOperationException("Refund exceeds payment amount");

            Status = PaymentStatus.Refunded;
        }
    }
}

// ============ Domain Services Layer ============

namespace PaymentSystem.Core.Services
{
    public interface IPaymentDomainService
    {
        bool CanRefund(Payment payment, Money amount);
        Money CalculateFees(Payment payment);
    }

    public class PaymentDomainService : IPaymentDomainService
    {
        public bool CanRefund(Payment payment, Money amount)
        {
            return payment.Status == PaymentStatus.Completed &&
                   amount.Amount <= payment.Amount.Amount;
        }

        public Money CalculateFees(Payment payment)
        {
            // Complex business logic for fee calculation
            var feePercentage = payment.Amount.Amount > 1000 ? 0.02m : 0.03m;
            var feeAmount = payment.Amount.Amount * feePercentage;
            return new Money(feeAmount, payment.Amount.Currency);
        }
    }
}

// ============ Application Services Layer ============

namespace PaymentSystem.Application
{
    public class ProcessPaymentUseCase
    {
        private readonly IPaymentRepository _repository;
        private readonly IPaymentGateway _gateway;
        private readonly IPaymentDomainService _domainService;

        public async Task<PaymentResult> ExecuteAsync(ProcessPaymentCommand command)
        {
            var money = new Money(command.Amount, command.Currency);
            var payment = Payment.Create(money);

            var fees = _domainService.CalculateFees(payment);
            var totalAmount = payment.Amount.Add(fees);

            var result = await _gateway.ChargeAsync(totalAmount);

            if (result.IsSuccess)
            {
                payment.MarkAsCompleted(result.TransactionId);
            }

            await _repository.SaveAsync(payment);

            return PaymentResult.FromPayment(payment);
        }
    }
}

// ============ Infrastructure Layer (outermost) ============

namespace PaymentSystem.Infrastructure
{
    // Implements interfaces defined in inner layers
    public class SqlPaymentRepository : IPaymentRepository
    {
        // Implementation
    }
}
```

---


---

# Part 2: Resilience Patterns

## Circuit Breaker Pattern

### Concept

Prevent cascading failures by stopping requests to a failing service temporarily.

### States

```
Closed → Normal operation, requests flow through
Open → Service failing, requests rejected immediately
Half-Open → Testing if service recovered
```

### Implementation with Polly

```csharp
public static class PaymentServiceExtensions
{
    public static IServiceCollection AddResilientPaymentGateway(
        this IServiceCollection services)
    {
        services.AddHttpClient<IPaymentGateway, StripeGateway>()
            .AddPolicyHandler(GetCircuitBreakerPolicy())
            .AddPolicyHandler(GetRetryPolicy());

        return services;
    }

    private static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
    {
        return HttpPolicyExtensions
            .HandleTransientHttpError()
            .CircuitBreakerAsync(
                handledEventsAllowedBeforeBreaking: 5, // Open after 5 failures
                durationOfBreak: TimeSpan.FromSeconds(30), // Stay open for 30s
                onBreak: (result, duration) =>
                {
                    // Log circuit breaker opened
                    Console.WriteLine($"Circuit breaker opened for {duration.TotalSeconds}s");
                },
                onReset: () =>
                {
                    // Log circuit breaker closed
                    Console.WriteLine("Circuit breaker closed");
                },
                onHalfOpen: () =>
                {
                    // Log circuit breaker testing
                    Console.WriteLine("Circuit breaker half-open, testing...");
                });
    }
}
```

### Manual Circuit Breaker

```csharp
public class CircuitBreaker
{
    private CircuitState _state = CircuitState.Closed;
    private int _failureCount = 0;
    private DateTime _lastFailureTime;

    private readonly int _failureThreshold = 5;
    private readonly TimeSpan _timeout = TimeSpan.FromSeconds(60);
    private readonly object _lock = new object();

    public async Task<T> ExecuteAsync<T>(Func<Task<T>> action)
    {
        lock (_lock)
        {
            if (_state == CircuitState.Open)
            {
                if (DateTime.UtcNow - _lastFailureTime > _timeout)
                {
                    _state = CircuitState.HalfOpen;
                }
                else
                {
                    throw new CircuitBreakerOpenException("Circuit breaker is open");
                }
            }
        }

        try
        {
            var result = await action();

            lock (_lock)
            {
                if (_state == CircuitState.HalfOpen)
                {
                    _state = CircuitState.Closed;
                    _failureCount = 0;
                }
            }

            return result;
        }
        catch (Exception)
        {
            lock (_lock)
            {
                _failureCount++;
                _lastFailureTime = DateTime.UtcNow;

                if (_failureCount >= _failureThreshold)
                {
                    _state = CircuitState.Open;
                }
            }

            throw;
        }
    }
}

public enum CircuitState
{
    Closed,
    Open,
    HalfOpen
}

public class CircuitBreakerOpenException : Exception
{
    public CircuitBreakerOpenException(string message) : base(message) { }
}
```

---


---

## Retry Pattern

### Exponential Backoff

```csharp
public class ExponentialRetryPolicy
{
    private readonly int _maxRetries;
    private readonly TimeSpan _initialDelay;

    public ExponentialRetryPolicy(int maxRetries = 3, TimeSpan? initialDelay = null)
    {
        _maxRetries = maxRetries;
        _initialDelay = initialDelay ?? TimeSpan.FromSeconds(1);
    }

    public async Task<T> ExecuteAsync<T>(Func<Task<T>> action)
    {
        for (int attempt = 0; attempt <= _maxRetries; attempt++)
        {
            try
            {
                return await action();
            }
            catch (Exception ex) when (attempt < _maxRetries && IsTransient(ex))
            {
                var delay = TimeSpan.FromMilliseconds(
                    _initialDelay.TotalMilliseconds * Math.Pow(2, attempt));

                await Task.Delay(delay);
            }
        }

        // Final attempt without catching
        return await action();
    }

    private bool IsTransient(Exception ex)
    {
        return ex is HttpRequestException ||
               ex is TimeoutException ||
               (ex is StripeException stripe && stripe.StripeError?.Type == "api_connection_error");
    }
}

// Usage
public class PaymentService
{
    private readonly ExponentialRetryPolicy _retryPolicy;

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        return await _retryPolicy.ExecuteAsync(async () =>
        {
            return await _gateway.ProcessPaymentAsync(request);
        });
    }
}
```

### Polly Retry with Jitter

```csharp
public static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
{
    var jitterer = new Random();

    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(
            retryCount: 3,
            sleepDurationProvider: retryAttempt =>
                TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)) +
                TimeSpan.FromMilliseconds(jitterer.Next(0, 1000)), // Add jitter
            onRetry: (outcome, timespan, retryAttempt, context) =>
            {
                Console.WriteLine($"Retry {retryAttempt} after {timespan.TotalSeconds}s");
            });
}
```

---


---

## Bulkhead Pattern

### Concept

Isolate resources to prevent cascading failures.

```
Payment Processing Pool (10 threads)
Webhook Processing Pool (5 threads)
Reporting Pool (3 threads)

If webhooks fail → doesn't affect payment processing
```

### Implementation

```csharp
public class BulkheadPaymentService
{
    private readonly SemaphoreSlim _paymentSemaphore;
    private readonly SemaphoreSlim _webhookSemaphore;
    private readonly SemaphoreSlim _reportingSemaphore;

    public BulkheadPaymentService()
    {
        _paymentSemaphore = new SemaphoreSlim(10, 10);      // 10 concurrent payments
        _webhookSemaphore = new SemaphoreSlim(5, 5);        // 5 concurrent webhooks
        _reportingSemaphore = new SemaphoreSlim(3, 3);      // 3 concurrent reports
    }

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        await _paymentSemaphore.WaitAsync();

        try
        {
            return await ProcessPaymentInternalAsync(request);
        }
        finally
        {
            _paymentSemaphore.Release();
        }
    }

    public async Task ProcessWebhookAsync(WebhookEvent webhook)
    {
        await _webhookSemaphore.WaitAsync();

        try
        {
            await ProcessWebhookInternalAsync(webhook);
        }
        finally
        {
            _webhookSemaphore.Release();
        }
    }

    public async Task<PaymentReport> GenerateReportAsync(ReportRequest request)
    {
        await _reportingSemaphore.WaitAsync();

        try
        {
            return await GenerateReportInternalAsync(request);
        }
        finally
        {
            _reportingSemaphore.Release();
        }
    }
}
```

### Thread Pool Isolation

```csharp
public class IsolatedThreadPoolService
{
    private readonly TaskScheduler _paymentScheduler;
    private readonly TaskScheduler _webhookScheduler;

    public IsolatedThreadPoolService()
    {
        // Create dedicated thread pools
        _paymentScheduler = new LimitedConcurrencyLevelTaskScheduler(10);
        _webhookScheduler = new LimitedConcurrencyLevelTaskScheduler(5);
    }

    public Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        return Task.Factory.StartNew(
            () => ProcessPaymentInternalAsync(request),
            CancellationToken.None,
            TaskCreationOptions.None,
            _paymentScheduler).Unwrap();
    }
}

public class LimitedConcurrencyLevelTaskScheduler : TaskScheduler
{
    private readonly int _maxDegreeOfParallelism;
    private readonly LinkedList<Task> _tasks = new();
    private int _delegatesQueuedOrRunning = 0;

    public LimitedConcurrencyLevelTaskScheduler(int maxDegreeOfParallelism)
    {
        _maxDegreeOfParallelism = maxDegreeOfParallelism;
    }

    protected override void QueueTask(Task task)
    {
        lock (_tasks)
        {
            _tasks.AddLast(task);

            if (_delegatesQueuedOrRunning < _maxDegreeOfParallelism)
            {
                ++_delegatesQueuedOrRunning;
                NotifyThreadPoolOfPendingWork();
            }
        }
    }

    private void NotifyThreadPoolOfPendingWork()
    {
        ThreadPool.UnsafeQueueUserWorkItem(_ =>
        {
            try
            {
                while (true)
                {
                    Task item;
                    lock (_tasks)
                    {
                        if (_tasks.Count == 0)
                        {
                            --_delegatesQueuedOrRunning;
                            break;
                        }

                        item = _tasks.First.Value;
                        _tasks.RemoveFirst();
                    }

                    TryExecuteTask(item);
                }
            }
            finally
            {
                // Implementation
            }
        }, null);
    }

    protected override bool TryExecuteTaskInline(Task task, bool taskWasPreviouslyQueued)
    {
        return false; // Don't allow inline execution
    }

    protected override IEnumerable<Task> GetScheduledTasks()
    {
        lock (_tasks)
        {
            return _tasks.ToArray();
        }
    }
}
```

---


---

## Chaos Engineering

### Concept

Intentionally inject failures to test system resilience.

```csharp
public class ChaosPaymentGateway : IPaymentGateway
{
    private readonly IPaymentGateway _actualGateway;
    private readonly IChaosConfiguration _chaosConfig;
    private readonly Random _random = new();

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        // Chaos: Random failures
        if (_chaosConfig.IsEnabled && ShouldInjectFailure())
        {
            await InjectChaosAsync();
        }

        return await _actualGateway.ProcessPaymentAsync(request);
    }

    private bool ShouldInjectFailure()
    {
        return _random.NextDouble() < _chaosConfig.FailureRate;
    }

    private async Task InjectChaosAsync()
    {
        var chaosType = _random.Next(4);

        switch (chaosType)
        {
            case 0: // Timeout
                await Task.Delay(TimeSpan.FromSeconds(30));
                throw new TimeoutException("Chaos: Simulated timeout");

            case 1: // Network error
                throw new HttpRequestException("Chaos: Simulated network error");

            case 2: // Random latency
                var latency = _random.Next(1000, 5000);
                await Task.Delay(latency);
                break;

            case 3: // Intermittent failure
                if (_random.NextDouble() < 0.5)
                {
                    throw new Exception("Chaos: Simulated intermittent failure");
                }
                break;
        }
    }
}

public interface IChaosConfiguration
{
    bool IsEnabled { get; }
    double FailureRate { get; }
}

// Enable only in test environments
services.AddSingleton<IPaymentGateway>(sp =>
{
    var actualGateway = new StripePaymentGateway();
    var env = sp.GetRequiredService<IHostEnvironment>();

    if (env.IsProduction())
    {
        return actualGateway;
    }

    return new ChaosPaymentGateway(actualGateway, new ChaosConfiguration
    {
        IsEnabled = true,
        FailureRate = 0.05 // 5% failure rate in test
    });
});
```

---


---

# Part 3: Integration Patterns

## Saga Pattern

### Concept

Manage distributed transactions across multiple services without using distributed transactions. Each step has a compensating action for rollback.

### When to Use

- Payment involves multiple services (inventory, shipping, loyalty points)
- Microservices architecture
- Long-running business processes

### Implementation

```csharp
public interface ISaga
{
    Task ExecuteAsync();
    Task CompensateAsync();
}

public class PaymentSaga : ISaga
{
    private readonly IPaymentService _paymentService;
    private readonly IInventoryService _inventoryService;
    private readonly IShippingService _shippingService;
    private readonly ILogger<PaymentSaga> _logger;

    private readonly List<Func<Task>> _compensations = new();

    public PaymentSaga(
        IPaymentService paymentService,
        IInventoryService inventoryService,
        IShippingService shippingService,
        ILogger<PaymentSaga> logger)
    {
        _paymentService = paymentService;
        _inventoryService = inventoryService;
        _shippingService = shippingService;
        _logger = logger;
    }

    public async Task ExecuteAsync()
    {
        try
        {
            // Step 1: Reserve Inventory
            await ReserveInventoryAsync();
            _compensations.Add(ReleaseInventoryAsync);

            // Step 2: Process Payment
            await ProcessPaymentAsync();
            _compensations.Add(RefundPaymentAsync);

            // Step 3: Create Shipment
            await CreateShipmentAsync();
            _compensations.Add(CancelShipmentAsync);

            _logger.LogInformation("Saga completed successfully");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Saga failed, compensating...");
            await CompensateAsync();
            throw;
        }
    }

    public async Task CompensateAsync()
    {
        // Execute compensations in reverse order
        _compensations.Reverse();

        foreach (var compensation in _compensations)
        {
            try
            {
                await compensation();
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Compensation failed");
                // Continue with other compensations
            }
        }
    }

    private async Task ReserveInventoryAsync()
    {
        await _inventoryService.ReserveAsync(/* items */);
        _logger.LogInformation("Inventory reserved");
    }

    private async Task ReleaseInventoryAsync()
    {
        await _inventoryService.ReleaseAsync(/* items */);
        _logger.LogInformation("Inventory released");
    }

    private async Task ProcessPaymentAsync()
    {
        await _paymentService.ProcessAsync(/* payment details */);
        _logger.LogInformation("Payment processed");
    }

    private async Task RefundPaymentAsync()
    {
        await _paymentService.RefundAsync(/* payment id */);
        _logger.LogInformation("Payment refunded");
    }

    private async Task CreateShipmentAsync()
    {
        await _shippingService.CreateAsync(/* shipment details */);
        _logger.LogInformation("Shipment created");
    }

    private async Task CancelShipmentAsync()
    {
        await _shippingService.CancelAsync(/* shipment id */);
        _logger.LogInformation("Shipment canceled");
    }
}
```

### Saga Orchestrator Pattern

```csharp
public class SagaOrchestrator
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<SagaOrchestrator> _logger;

    public async Task<SagaResult> ExecuteSagaAsync<TSaga>(TSaga saga)
        where TSaga : ISaga
    {
        var sagaState = new SagaState
        {
            SagaId = Guid.NewGuid(),
            StartedAt = DateTime.UtcNow,
            Status = SagaStatus.InProgress
        };

        try
        {
            await saga.ExecuteAsync();

            sagaState.Status = SagaStatus.Completed;
            sagaState.CompletedAt = DateTime.UtcNow;

            return SagaResult.Success(sagaState);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Saga {SagaId} failed", sagaState.SagaId);

            sagaState.Status = SagaStatus.Compensating;
            await saga.CompensateAsync();

            sagaState.Status = SagaStatus.Compensated;
            sagaState.CompletedAt = DateTime.UtcNow;

            return SagaResult.Failure(sagaState, ex.Message);
        }
    }
}

public enum SagaStatus
{
    InProgress,
    Completed,
    Compensating,
    Compensated,
    Failed
}
```

---


---

## Idempotency Pattern

### Concept

Ensure operations can be safely retried without side effects.

### Implementation

```csharp
public class IdempotentPaymentService : IPaymentService
{
    private readonly IPaymentService _inner;
    private readonly IIdempotencyStore _idempotencyStore;

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        var idempotencyKey = request.IdempotencyKey ??
            throw new ArgumentException("Idempotency key required");

        // Check if already processed
        var cached = await _idempotencyStore.GetAsync(idempotencyKey);
        if (cached != null)
        {
            return cached; // Return cached result
        }

        // Process payment
        var result = await _inner.ProcessPaymentAsync(request);

        // Store result
        await _idempotencyStore.SetAsync(
            idempotencyKey,
            result,
            TimeSpan.FromHours(24));

        return result;
    }
}

public interface IIdempotencyStore
{
    Task<PaymentResult> GetAsync(string key);
    Task SetAsync(string key, PaymentResult result, TimeSpan expiry);
}

public class RedisIdempotencyStore : IIdempotencyStore
{
    private readonly IDistributedCache _cache;

    public async Task<PaymentResult> GetAsync(string key)
    {
        var cached = await _cache.GetStringAsync($"idempotency:{key}");
        return cached != null
            ? JsonSerializer.Deserialize<PaymentResult>(cached)
            : null;
    }

    public async Task SetAsync(string key, PaymentResult result, TimeSpan expiry)
    {
        await _cache.SetStringAsync(
            $"idempotency:{key}",
            JsonSerializer.Serialize(result),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = expiry
            });
    }
}
```

### Idempotency Key Generation

```csharp
public static class IdempotencyKeyGenerator
{
    public static string GenerateForOrder(string orderId, string operation)
    {
        return $"{orderId}:{operation}:{DateTime.UtcNow:yyyyMMdd}";
    }

    public static string GenerateForPayment(Guid paymentId, string operation)
    {
        return $"payment:{paymentId}:{operation}";
    }

    public static string GenerateForRefund(Guid paymentId, decimal amount)
    {
        return $"refund:{paymentId}:{amount}:{DateTime.UtcNow.Ticks}";
    }
}

// Usage
var idempotencyKey = IdempotencyKeyGenerator.GenerateForOrder(
    order.Id,
    "process_payment");

var result = await _paymentService.ProcessPaymentAsync(new PaymentRequest
{
    OrderId = order.Id,
    Amount = order.Total,
    IdempotencyKey = idempotencyKey
});
```

---


---

## Rate Limiting & Throttling Pattern

### Token Bucket Algorithm

```csharp
public class TokenBucketRateLimiter
{
    private readonly int _capacity;
    private readonly int _refillRate; // tokens per second
    private int _tokens;
    private DateTime _lastRefill;
    private readonly object _lock = new object();

    public TokenBucketRateLimiter(int capacity, int refillRate)
    {
        _capacity = capacity;
        _refillRate = refillRate;
        _tokens = capacity;
        _lastRefill = DateTime.UtcNow;
    }

    public bool TryConsume(int tokens = 1)
    {
        lock (_lock)
        {
            Refill();

            if (_tokens >= tokens)
            {
                _tokens -= tokens;
                return true;
            }

            return false;
        }
    }

    private void Refill()
    {
        var now = DateTime.UtcNow;
        var elapsed = (now - _lastRefill).TotalSeconds;
        var tokensToAdd = (int)(elapsed * _refillRate);

        if (tokensToAdd > 0)
        {
            _tokens = Math.Min(_capacity, _tokens + tokensToAdd);
            _lastRefill = now;
        }
    }
}

// Usage
public class RateLimitedPaymentService : IPaymentService
{
    private readonly IPaymentService _inner;
    private readonly TokenBucketRateLimiter _rateLimiter;

    public RateLimitedPaymentService(IPaymentService inner)
    {
        _inner = inner;
        _rateLimiter = new TokenBucketRateLimiter(
            capacity: 100,      // 100 tokens
            refillRate: 10      // 10 per second
        );
    }

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        if (!_rateLimiter.TryConsume())
        {
            throw new RateLimitExceededException("Rate limit exceeded");
        }

        return await _inner.ProcessPaymentAsync(request);
    }
}
```

### Sliding Window Rate Limiter

```csharp
public class SlidingWindowRateLimiter
{
    private readonly IDistributedCache _cache;
    private readonly int _maxRequests;
    private readonly TimeSpan _window;

    public SlidingWindowRateLimiter(
        IDistributedCache cache,
        int maxRequests,
        TimeSpan window)
    {
        _cache = cache;
        _maxRequests = maxRequests;
        _window = window;
    }

    public async Task<bool> IsAllowedAsync(string key)
    {
        var cacheKey = $"ratelimit:{key}";
        var now = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
        var windowStart = now - (long)_window.TotalMilliseconds;

        // Get request timestamps from cache
        var requestsJson = await _cache.GetStringAsync(cacheKey);
        var requests = requestsJson != null
            ? JsonSerializer.Deserialize<List<long>>(requestsJson)
            : new List<long>();

        // Remove old requests outside window
        requests = requests.Where(r => r > windowStart).ToList();

        // Check if under limit
        if (requests.Count >= _maxRequests)
        {
            return false;
        }

        // Add current request
        requests.Add(now);

        // Save to cache
        await _cache.SetStringAsync(
            cacheKey,
            JsonSerializer.Serialize(requests),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = _window
            });

        return true;
    }
}
```

### ASP.NET Core Rate Limiting Middleware

```csharp
public class RateLimitingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly SlidingWindowRateLimiter _rateLimiter;

    public RateLimitingMiddleware(
        RequestDelegate next,
        SlidingWindowRateLimiter rateLimiter)
    {
        _next = next;
        _rateLimiter = rateLimiter;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Rate limit by IP or user
        var identifier = context.User.Identity?.Name ??
                        context.Connection.RemoteIpAddress?.ToString() ??
                        "anonymous";

        if (!await _rateLimiter.IsAllowedAsync(identifier))
        {
            context.Response.StatusCode = 429; // Too Many Requests
            context.Response.Headers.Add("Retry-After", "60");
            await context.Response.WriteAsync("Rate limit exceeded");
            return;
        }

        await _next(context);
    }
}

// Startup.cs
public void Configure(IApplicationBuilder app)
{
    app.UseMiddleware<RateLimitingMiddleware>();
    // ...
}
```

### Adaptive Throttling

```csharp
public class AdaptiveThrottler
{
    private readonly IMetricsCollector _metrics;
    private int _currentLimit = 100;
    private readonly int _minLimit = 10;
    private readonly int _maxLimit = 1000;

    public async Task<bool> ShouldThrottleAsync()
    {
        var metrics = await _metrics.GetCurrentMetricsAsync();

        // Adjust limits based on system health
        if (metrics.ErrorRate > 0.1) // 10% error rate
        {
            _currentLimit = Math.Max(_minLimit, _currentLimit - 10);
        }
        else if (metrics.ResponseTimeP95 > 1000) // P95 > 1 second
        {
            _currentLimit = Math.Max(_minLimit, _currentLimit - 5);
        }
        else if (metrics.ErrorRate < 0.01 && metrics.ResponseTimeP95 < 500)
        {
            _currentLimit = Math.Min(_maxLimit, _currentLimit + 5);
        }

        var currentLoad = await _metrics.GetCurrentLoadAsync();
        return currentLoad >= _currentLimit;
    }
}
```


### Priority-Based Throttling

```csharp
public class PriorityThrottler
{
    private readonly TokenBucketThrottler _highPriority;
    private readonly TokenBucketThrottler _normalPriority;
    private readonly TokenBucketThrottler _lowPriority;

    public PriorityThrottler()
    {
        _highPriority = new TokenBucketThrottler(100, 50);   // 50/sec
        _normalPriority = new TokenBucketThrottler(50, 25);  // 25/sec
        _lowPriority = new TokenBucketThrottler(20, 10);     // 10/sec
    }

    public bool TryConsume(PaymentPriority priority)
    {
        return priority switch
        {
            PaymentPriority.High => _highPriority.TryConsume(),
            PaymentPriority.Normal => _normalPriority.TryConsume(),
            PaymentPriority.Low => _lowPriority.TryConsume(),
            _ => false
        };
    }
}
```

---


---

## Reconciliation Pattern

### Concept

Compare internal records with payment provider records to detect discrepancies.

### Implementation

```csharp
public class PaymentReconciliationService
{
    private readonly IPaymentRepository _repository;
    private readonly IPaymentGateway _gateway;
    private readonly ILogger<PaymentReconciliationService> _logger;

    public async Task<ReconciliationReport> ReconcileAsync(DateTime date)
    {
        var report = new ReconciliationReport { Date = date };

        // Get our records
        var ourPayments = await _repository.GetPaymentsByDateAsync(date);

        // Get provider records
        var providerPayments = await _gateway.GetPaymentsByDateAsync(date);

        // Find mismatches
        foreach (var ourPayment in ourPayments)
        {
            var providerPayment = providerPayments
                .FirstOrDefault(p => p.Id == ourPayment.ProviderTransactionId);

            if (providerPayment == null)
            {
                report.MissingInProvider.Add(ourPayment);
                _logger.LogWarning(
                    "Payment {PaymentId} found in our system but not in provider",
                    ourPayment.Id);
            }
            else if (ourPayment.Amount != providerPayment.Amount)
            {
                report.AmountMismatches.Add(new AmountMismatch
                {
                    PaymentId = ourPayment.Id,
                    OurAmount = ourPayment.Amount,
                    ProviderAmount = providerPayment.Amount
                });
                _logger.LogWarning(
                    "Amount mismatch for payment {PaymentId}: Ours={OurAmount}, Provider={ProviderAmount}",
                    ourPayment.Id,
                    ourPayment.Amount,
                    providerPayment.Amount);
            }
            else if (ourPayment.Status != MapProviderStatus(providerPayment.Status))
            {
                report.StatusMismatches.Add(new StatusMismatch
                {
                    PaymentId = ourPayment.Id,
                    OurStatus = ourPayment.Status,
                    ProviderStatus = providerPayment.Status
                });
            }
        }

        // Find payments in provider but not in our system
        foreach (var providerPayment in providerPayments)
        {
            if (!ourPayments.Any(p => p.ProviderTransactionId == providerPayment.Id))
            {
                report.MissingInOurSystem.Add(providerPayment);
                _logger.LogWarning(
                    "Payment {TransactionId} found in provider but not in our system",
                    providerPayment.Id);
            }
        }

        report.TotalOurPayments = ourPayments.Count;
        report.TotalProviderPayments = providerPayments.Count;
        report.TotalDiscrepancies =
            report.MissingInProvider.Count +
            report.MissingInOurSystem.Count +
            report.AmountMismatches.Count +
            report.StatusMismatches.Count;

        return report;
    }
}

public class ReconciliationReport
{
    public DateTime Date { get; set; }
    public int TotalOurPayments { get; set; }
    public int TotalProviderPayments { get; set; }
    public int TotalDiscrepancies { get; set; }

    public List<Payment> MissingInProvider { get; set; } = new();
    public List<ProviderPayment> MissingInOurSystem { get; set; } = new();
    public List<AmountMismatch> AmountMismatches { get; set; } = new();
    public List<StatusMismatch> StatusMismatches { get; set; } = new();
}
```

### Automated Reconciliation

```csharp
public class ReconciliationBackgroundService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<ReconciliationBackgroundService> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                // Run daily at 2 AM
                var now = DateTime.UtcNow;
                var nextRun = now.Date.AddDays(1).AddHours(2);
                var delay = nextRun - now;

                await Task.Delay(delay, stoppingToken);

                using var scope = _serviceProvider.CreateScope();
                var reconciliationService = scope.ServiceProvider
                    .GetRequiredService<PaymentReconciliationService>();

                var yesterday = DateTime.UtcNow.Date.AddDays(-1);
                var report = await reconciliationService.ReconcileAsync(yesterday);

                if (report.TotalDiscrepancies > 0)
                {
                    _logger.LogWarning(
                        "Reconciliation found {Count} discrepancies for {Date}",
                        report.TotalDiscrepancies,
                        yesterday);

                    // Send alert
                    await SendAlertAsync(report);
                }
                else
                {
                    _logger.LogInformation(
                        "Reconciliation completed successfully for {Date}",
                        yesterday);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Reconciliation failed");
            }
        }
    }

    private async Task SendAlertAsync(ReconciliationReport report)
    {
        // Send email/Slack notification
    }
}
```

---


---

## Notification Pattern

### Observer Pattern for Payment Events

```csharp
public interface IPaymentObserver
{
    Task OnPaymentCompletedAsync(Payment payment);
    Task OnPaymentFailedAsync(Payment payment);
    Task OnRefundProcessedAsync(Payment payment, Refund refund);
}

public class EmailNotificationObserver : IPaymentObserver
{
    private readonly IEmailService _emailService;

    public async Task OnPaymentCompletedAsync(Payment payment)
    {
        await _emailService.SendAsync(new Email
        {
            To = payment.CustomerEmail,
            Subject = "Payment Confirmation",
            Body = $"Your payment of {payment.Amount} {payment.Currency} was successful."
        });
    }

    public async Task OnPaymentFailedAsync(Payment payment)
    {
        await _emailService.SendAsync(new Email
        {
            To = payment.CustomerEmail,
            Subject = "Payment Failed",
            Body = $"Your payment of {payment.Amount} {payment.Currency} failed."
        });
    }

    public async Task OnRefundProcessedAsync(Payment payment, Refund refund)
    {
        await _emailService.SendAsync(new Email
        {
            To = payment.CustomerEmail,
            Subject = "Refund Processed",
            Body = $"Your refund of {refund.Amount} {refund.Currency} has been processed."
        });
    }
}

public class SmsNotificationObserver : IPaymentObserver
{
    private readonly ISmsService _smsService;

    public async Task OnPaymentCompletedAsync(Payment payment)
    {
        if (!string.IsNullOrEmpty(payment.CustomerPhone))
        {
            await _smsService.SendAsync(
                payment.CustomerPhone,
                $"Payment confirmed: {payment.Amount} {payment.Currency}");
        }
    }

    // Other methods...
}

public class ObservablePaymentService : IPaymentService
{
    private readonly IPaymentService _inner;
    private readonly List<IPaymentObserver> _observers = new();

    public void Attach(IPaymentObserver observer)
    {
        _observers.Add(observer);
    }

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        var result = await _inner.ProcessPaymentAsync(request);

        if (result.IsSuccess)
        {
            await NotifyPaymentCompletedAsync(result.Payment);
        }
        else
        {
            await NotifyPaymentFailedAsync(result.Payment);
        }

        return result;
    }

    private async Task NotifyPaymentCompletedAsync(Payment payment)
    {
        foreach (var observer in _observers)
        {
            try
            {
                await observer.OnPaymentCompletedAsync(payment);
            }
            catch (Exception ex)
            {
                // Log but don't fail the payment
                Console.WriteLine($"Observer failed: {ex.Message}");
            }
        }
    }

    private async Task NotifyPaymentFailedAsync(Payment payment)
    {
        foreach (var observer in _observers)
        {
            try
            {
                await observer.OnPaymentFailedAsync(payment);
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Observer failed: {ex.Message}");
            }
        }
    }
}
```

---


---

## Priority Queue Pattern

### Concept

Process high-priority payments before low-priority ones.

```csharp
public class PriorityPaymentQueue
{
    private readonly PriorityQueue<PaymentRequest, int> _queue = new();
    private readonly SemaphoreSlim _signal = new(0);

    public void Enqueue(PaymentRequest payment, PaymentPriority priority)
    {
        var priorityValue = priority switch
        {
            PaymentPriority.Critical => 1,
            PaymentPriority.High => 2,
            PaymentPriority.Normal => 3,
            PaymentPriority.Low => 4,
            _ => 5
        };

        lock (_queue)
        {
            _queue.Enqueue(payment, priorityValue);
        }

        _signal.Release();
    }

    public async Task<PaymentRequest> DequeueAsync(CancellationToken cancellationToken)
    {
        await _signal.WaitAsync(cancellationToken);

        lock (_queue)
        {
            return _queue.Dequeue();
        }
    }
}

public class PriorityPaymentProcessor : BackgroundService
{
    private readonly PriorityPaymentQueue _queue;
    private readonly IPaymentService _paymentService;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var payment = await _queue.DequeueAsync(stoppingToken);

            try
            {
                await _paymentService.ProcessPaymentAsync(payment);
            }
            catch (Exception ex)
            {
                // Log and potentially re-queue with lower priority
            }
        }
    }
}

public enum PaymentPriority
{
    Critical,  // VIP customers, high-value transactions
    High,      // Regular customers, normal transactions
    Normal,    // Batch processing
    Low        // Background reconciliation
}
```

---


---

## Claim Check Pattern

### Concept

Store large payloads separately and pass references.

```csharp
public class ClaimCheckPaymentService
{
    private readonly IPaymentService _paymentService;
    private readonly IBlobStorage _blobStorage;
    private readonly IMessageQueue _messageQueue;

    public async Task ProcessLargePaymentAsync(LargePaymentRequest request)
    {
        // 1. Store large payload in blob storage
        var claimCheck = await _blobStorage.UploadAsync(
            $"payments/{request.PaymentId}/request.json",
            JsonSerializer.Serialize(request));

        // 2. Send small message with reference
        var message = new PaymentMessage
        {
            PaymentId = request.PaymentId,
            ClaimCheckUrl = claimCheck,
            Timestamp = DateTime.UtcNow
        };

        await _messageQueue.SendAsync(message);
    }

    public async Task ProcessPaymentMessageAsync(PaymentMessage message)
    {
        // 1. Retrieve large payload using claim check
        var payloadJson = await _blobStorage.DownloadAsync(message.ClaimCheckUrl);
        var request = JsonSerializer.Deserialize<LargePaymentRequest>(payloadJson);

        // 2. Process payment
        await _paymentService.ProcessPaymentAsync(request);

        // 3. Clean up blob (optional)
        await _blobStorage.DeleteAsync(message.ClaimCheckUrl);
    }
}

public class PaymentMessage
{
    public Guid PaymentId { get; set; }
    public string ClaimCheckUrl { get; set; }  // Reference to large payload
    public DateTime Timestamp { get; set; }
}

public class LargePaymentRequest
{
    public Guid PaymentId { get; set; }
    public decimal Amount { get; set; }

    // Large data
    public List<OrderItem> OrderItems { get; set; }  // Could be 1000s of items
    public byte[] Receipt { get; set; }               // Large PDF
    public List<AuditLog> AuditLogs { get; set; }    // Extensive history
}
```

---


---

## Pipes and Filters Pattern

### Concept

Process data through a series of independent processing steps.

```csharp
public interface IPaymentFilter
{
    Task<PaymentContext> ProcessAsync(PaymentContext context);
}

public class ValidationFilter : IPaymentFilter
{
    public async Task<PaymentContext> ProcessAsync(PaymentContext context)
    {
        if (context.Request.Amount <= 0)
        {
            context.AddError("Amount must be positive");
            context.IsValid = false;
        }

        if (string.IsNullOrEmpty(context.Request.Currency))
        {
            context.AddError("Currency is required");
            context.IsValid = false;
        }

        return context;
    }
}

public class FraudDetectionFilter : IPaymentFilter
{
    private readonly IFraudDetection _fraudDetection;

    public async Task<PaymentContext> ProcessAsync(PaymentContext context)
    {
        if (!context.IsValid) return context;

        var fraudScore = await _fraudDetection.CalculateRiskScoreAsync(context.Request);
        context.FraudScore = fraudScore;

        if (fraudScore > 0.9)
        {
            context.AddError("High fraud risk");
            context.IsValid = false;
        }
        else if (fraudScore > 0.7)
        {
            context.RequiresManualReview = true;
        }

        return context;
    }
}

public class EnrichmentFilter : IPaymentFilter
{
    private readonly ICustomerService _customerService;

    public async Task<PaymentContext> ProcessAsync(PaymentContext context)
    {
        if (!context.IsValid) return context;

        var customer = await _customerService.GetCustomerAsync(context.Request.CustomerId);
        context.CustomerName = customer.Name;
        context.CustomerEmail = customer.Email;
        context.CustomerTier = customer.Tier;

        return context;
    }
}

public class PaymentProcessingFilter : IPaymentFilter
{
    private readonly IPaymentGateway _gateway;

    public async Task<PaymentContext> ProcessAsync(PaymentContext context)
    {
        if (!context.IsValid || context.RequiresManualReview) return context;

        var result = await _gateway.ProcessPaymentAsync(context.Request);
        context.TransactionId = result.TransactionId;
        context.IsSuccess = result.IsSuccess;

        return context;
    }
}

public class PaymentPipeline
{
    private readonly List<IPaymentFilter> _filters;

    public PaymentPipeline()
    {
        _filters = new List<IPaymentFilter>
        {
            new ValidationFilter(),
            new FraudDetectionFilter(),
            new EnrichmentFilter(),
            new PaymentProcessingFilter()
        };
    }

    public async Task<PaymentContext> ExecuteAsync(PaymentRequest request)
    {
        var context = new PaymentContext { Request = request };

        foreach (var filter in _filters)
        {
            context = await filter.ProcessAsync(context);

            if (!context.IsValid)
            {
                break; // Stop processing on validation failure
            }
        }

        return context;
    }
}

public class PaymentContext
{
    public PaymentRequest Request { get; set; }
    public bool IsValid { get; set; } = true;
    public bool RequiresManualReview { get; set; }
    public bool IsSuccess { get; set; }
    public string TransactionId { get; set; }
    public double FraudScore { get; set; }
    public string CustomerName { get; set; }
    public string CustomerEmail { get; set; }
    public string CustomerTier { get; set; }
    public List<string> Errors { get; set; } = new();

    public void AddError(string error) => Errors.Add(error);
}
```

---


---

## Chain of Responsibility Pattern

### Concept

Pass request through a chain of handlers until one handles it.

```csharp
public abstract class PaymentHandler
{
    protected PaymentHandler _nextHandler;

    public PaymentHandler SetNext(PaymentHandler handler)
    {
        _nextHandler = handler;
        return handler;
    }

    public abstract Task<PaymentResult> HandleAsync(PaymentRequest request);
}

public class ValidationHandler : PaymentHandler
{
    public override async Task<PaymentResult> HandleAsync(PaymentRequest request)
    {
        if (request.Amount <= 0)
        {
            return PaymentResult.ValidationFailed("Amount must be positive");
        }

        if (string.IsNullOrEmpty(request.Currency))
        {
            return PaymentResult.ValidationFailed("Currency is required");
        }

        // Pass to next handler
        return await _nextHandler.HandleAsync(request);
    }
}

public class FraudCheckHandler : PaymentHandler
{
    private readonly IFraudDetection _fraudDetection;

    public override async Task<PaymentResult> HandleAsync(PaymentRequest request)
    {
        var fraudScore = await _fraudDetection.CalculateRiskScoreAsync(request);

        if (fraudScore > 0.9)
        {
            return PaymentResult.FraudDetected(fraudScore);
        }

        if (fraudScore > 0.7)
        {
            // Flag for manual review but continue
            request.RequiresManualReview = true;
        }

        return await _nextHandler.HandleAsync(request);
    }
}

public class LimitCheckHandler : PaymentHandler
{
    private readonly ICustomerService _customerService;

    public override async Task<PaymentResult> HandleAsync(PaymentRequest request)
    {
        var customer = await _customerService.GetCustomerAsync(request.CustomerId);

        if (request.Amount > customer.DailyLimit)
        {
            return PaymentResult.LimitExceeded("Daily limit exceeded");
        }

        return await _nextHandler.HandleAsync(request);
    }
}

public class PaymentProcessorHandler : PaymentHandler
{
    private readonly IPaymentGateway _gateway;

    public override async Task<PaymentResult> HandleAsync(PaymentRequest request)
    {
        if (request.RequiresManualReview)
        {
            return PaymentResult.PendingReview();
        }

        // Final handler - actually process payment
        var result = await _gateway.ProcessPaymentAsync(request);
        return PaymentResult.FromGateway(result);
    }
}

// Setup chain
public class PaymentService
{
    private readonly PaymentHandler _handlerChain;

    public PaymentService(
        IFraudDetection fraudDetection,
        ICustomerService customerService,
        IPaymentGateway gateway)
    {
        _handlerChain = new ValidationHandler();
        _handlerChain
            .SetNext(new FraudCheckHandler(fraudDetection))
            .SetNext(new LimitCheckHandler(customerService))
            .SetNext(new PaymentProcessorHandler(gateway));
    }

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        return await _handlerChain.HandleAsync(request);
    }
}
```

---


---

# Part 4: Data Patterns

## Cache-Aside Pattern

### Implementation

```csharp
public class CacheAsidePaymentRepository : IPaymentRepository
{
    private readonly IPaymentRepository _inner;
    private readonly IDistributedCache _cache;
    private readonly TimeSpan _cacheDuration = TimeSpan.FromMinutes(15);

    public async Task<Payment> GetByIdAsync(Guid id)
    {
        var cacheKey = $"payment:{id}";

        // 1. Try cache
        var cached = await _cache.GetStringAsync(cacheKey);
        if (cached != null)
        {
            return JsonSerializer.Deserialize<Payment>(cached);
        }

        // 2. Cache miss - get from database
        var payment = await _inner.GetByIdAsync(id);

        if (payment != null)
        {
            // 3. Store in cache
            await _cache.SetStringAsync(
                cacheKey,
                JsonSerializer.Serialize(payment),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = _cacheDuration
                });
        }

        return payment;
    }

    public async Task UpdateAsync(Payment payment)
    {
        // 1. Update database
        await _inner.UpdateAsync(payment);

        // 2. Invalidate cache
        await _cache.RemoveAsync($"payment:{payment.Id}");

        // Alternative: Update cache
        // await _cache.SetStringAsync($"payment:{payment.Id}", JsonSerializer.Serialize(payment));
    }
}
```

### Cache Stampede Prevention

```csharp
public class StampedePreventionCache
{
    private readonly IDistributedCache _cache;
    private readonly SemaphoreSlim _lock = new(1, 1);

    public async Task<T> GetOrSetAsync<T>(
        string key,
        Func<Task<T>> factory,
        TimeSpan expiration)
    {
        // Try cache first
        var cached = await _cache.GetStringAsync(key);
        if (cached != null)
        {
            return JsonSerializer.Deserialize<T>(cached);
        }

        // Prevent stampede with lock
        await _lock.WaitAsync();

        try
        {
            // Double-check after acquiring lock
            cached = await _cache.GetStringAsync(key);
            if (cached != null)
            {
                return JsonSerializer.Deserialize<T>(cached);
            }

            // Generate value
            var value = await factory();

            // Store in cache
            await _cache.SetStringAsync(
                key,
                JsonSerializer.Serialize(value),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = expiration
                });

            return value;
        }
        finally
        {
            _lock.Release();
        }
    }
}
```

---


---

## Materialized View Pattern

### Concept

Pre-compute and store complex query results for fast reads.

```csharp
// Complex query (slow)
var customerPayments = await _context.Payments
    .Include(p => p.Order)
    .Include(p => p.Customer)
    .Include(p => p.Refunds)
    .Where(p => p.CustomerId == customerId)
    .Select(p => new CustomerPaymentView
    {
        // Complex calculations
    })
    .ToListAsync();

// Materialized view (fast)
var customerPayments = await _context.CustomerPaymentViews
    .Where(v => v.CustomerId == customerId)
    .ToListAsync();
```

### Implementation

```csharp
public class CustomerPaymentView
{
    public Guid Id { get; set; }
    public string CustomerId { get; set; }
    public string CustomerName { get; set; }
    public string CustomerEmail { get; set; }

    public int TotalPayments { get; set; }
    public decimal TotalAmount { get; set; }
    public decimal TotalRefunded { get; set; }
    public decimal NetAmount { get; set; }

    public DateTime FirstPaymentDate { get; set; }
    public DateTime LastPaymentDate { get; set; }

    public string PreferredPaymentMethod { get; set; }
    public string RiskLevel { get; set; }

    public DateTime LastUpdated { get; set; }
}

public class MaterializedViewUpdater
{
    private readonly ApplicationDbContext _context;

    // Update view when payment is created
    public async Task OnPaymentCreatedAsync(PaymentCreatedEvent @event)
    {
        var view = await GetOrCreateViewAsync(@event.CustomerId);

        view.TotalPayments++;
        view.TotalAmount += @event.Amount;
        view.NetAmount = view.TotalAmount - view.TotalRefunded;
        view.LastPaymentDate = @event.Timestamp;
        view.LastUpdated = DateTime.UtcNow;

        await _context.SaveChangesAsync();
    }

    // Update view when refund is processed
    public async Task OnPaymentRefundedAsync(PaymentRefundedEvent @event)
    {
        var payment = await _context.Payments
            .Include(p => p.Customer)
            .FirstAsync(p => p.Id == @event.PaymentId);

        var view = await GetOrCreateViewAsync(payment.CustomerId);

        view.TotalRefunded += @event.RefundAmount;
        view.NetAmount = view.TotalAmount - view.TotalRefunded;
        view.LastUpdated = DateTime.UtcNow;

        await _context.SaveChangesAsync();
    }

    private async Task<CustomerPaymentView> GetOrCreateViewAsync(string customerId)
    {
        var view = await _context.CustomerPaymentViews
            .FirstOrDefaultAsync(v => v.CustomerId == customerId);

        if (view == null)
        {
            view = new CustomerPaymentView
            {
                Id = Guid.NewGuid(),
                CustomerId = customerId,
                FirstPaymentDate = DateTime.UtcNow
            };
            _context.CustomerPaymentViews.Add(view);
        }

        return view;
    }

    // Rebuild view from scratch
    public async Task RebuildViewAsync(string customerId)
    {
        var payments = await _context.Payments
            .Include(p => p.Refunds)
            .Where(p => p.CustomerId == customerId)
            .ToListAsync();

        var view = new CustomerPaymentView
        {
            CustomerId = customerId,
            TotalPayments = payments.Count,
            TotalAmount = payments.Sum(p => p.Amount),
            TotalRefunded = payments.SelectMany(p => p.Refunds).Sum(r => r.Amount),
            FirstPaymentDate = payments.Min(p => p.CreatedAt),
            LastPaymentDate = payments.Max(p => p.CreatedAt),
            LastUpdated = DateTime.UtcNow
        };

        view.NetAmount = view.TotalAmount - view.TotalRefunded;

        var existing = await _context.CustomerPaymentViews
            .FirstOrDefaultAsync(v => v.CustomerId == customerId);

        if (existing != null)
        {
            _context.CustomerPaymentViews.Remove(existing);
        }

        _context.CustomerPaymentViews.Add(view);
        await _context.SaveChangesAsync();
    }
}
```

---


---

## Write-Behind Caching

### Concept

Write to cache immediately, asynchronously persist to database.

```csharp
public class WriteBehindPaymentCache
{
    private readonly IMemoryCache _cache;
    private readonly IPaymentRepository _repository;
    private readonly Channel<Payment> _writeQueue;

    public WriteBehindPaymentCache(
        IMemoryCache cache,
        IPaymentRepository repository)
    {
        _cache = cache;
        _repository = repository;
        _writeQueue = Channel.CreateUnbounded<Payment>();

        // Start background writer
        _ = Task.Run(ProcessWriteQueueAsync);
    }

    public async Task<Payment> GetAsync(Guid id)
    {
        // Try cache first
        if (_cache.TryGetValue(id, out Payment payment))
        {
            return payment;
        }

        // Load from database
        payment = await _repository.GetByIdAsync(id);

        if (payment != null)
        {
            _cache.Set(id, payment, TimeSpan.FromMinutes(30));
        }

        return payment;
    }

    public async Task SaveAsync(Payment payment)
    {
        // Write to cache immediately
        _cache.Set(payment.Id, payment, TimeSpan.FromMinutes(30));

        // Queue for async database write
        await _writeQueue.Writer.WriteAsync(payment);
    }

    private async Task ProcessWriteQueueAsync()
    {
        await foreach (var payment in _writeQueue.Reader.ReadAllAsync())
        {
            try
            {
                await _repository.SaveAsync(payment);
            }
            catch (Exception ex)
            {
                // Log error, potentially retry
                Console.WriteLine($"Failed to persist payment {payment.Id}: {ex.Message}");

                // Could re-queue with backoff
            }
        }
    }
}
```

---


---

## Multi-Currency Pattern

### Currency Conversion Service

```csharp
public interface ICurrencyConverter
{
    Task<decimal> ConvertAsync(decimal amount, string fromCurrency, string toCurrency);
    Task<ExchangeRate> GetExchangeRateAsync(string fromCurrency, string toCurrency);
}

public class CurrencyConverter : ICurrencyConverter
{
    private readonly IExchangeRateProvider _rateProvider;
    private readonly IDistributedCache _cache;

    public async Task<decimal> ConvertAsync(
        decimal amount,
        string fromCurrency,
        string toCurrency)
    {
        if (fromCurrency == toCurrency)
            return amount;

        var rate = await GetExchangeRateAsync(fromCurrency, toCurrency);
        return amount * rate.Rate;
    }

    public async Task<ExchangeRate> GetExchangeRateAsync(
        string fromCurrency,
        string toCurrency)
    {
        var cacheKey = $"exchange_rate:{fromCurrency}:{toCurrency}";
        var cached = await _cache.GetStringAsync(cacheKey);

        if (cached != null)
        {
            return JsonSerializer.Deserialize<ExchangeRate>(cached);
        }

        var rate = await _rateProvider.GetRateAsync(fromCurrency, toCurrency);

        await _cache.SetStringAsync(
            cacheKey,
            JsonSerializer.Serialize(rate),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
            });

        return rate;
    }
}
```

### Multi-Currency Payment Processing

```csharp
public class MultiCurrencyPaymentService : IPaymentService
{
    private readonly IPaymentService _inner;
    private readonly ICurrencyConverter _currencyConverter;

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        var baseCurrency = "USD"; // Your base currency

        // Convert to base currency for processing
        var amountInBaseCurrency = await _currencyConverter.ConvertAsync(
            request.Amount,
            request.Currency,
            baseCurrency);

        // Store both amounts
        var payment = new Payment
        {
            Amount = request.Amount,
            Currency = request.Currency,
            BaseAmount = amountInBaseCurrency,
            BaseCurrency = baseCurrency,
            ExchangeRate = amountInBaseCurrency / request.Amount
        };

        // Process in base currency
        var processRequest = new PaymentRequest
        {
            Amount = amountInBaseCurrency,
            Currency = baseCurrency,
            // ... other fields
        };

        return await _inner.ProcessPaymentAsync(processRequest);
    }
}
```

---


---

## Payment State Machine

### State Pattern Implementation

```csharp
public abstract class PaymentState
{
    protected Payment Payment { get; set; }

    protected PaymentState(Payment payment)
    {
        Payment = payment;
    }

    public abstract Task ProcessAsync();
    public abstract Task CancelAsync();
    public abstract Task RefundAsync();

    protected void TransitionTo(PaymentState newState)
    {
        Payment.State = newState;
    }
}

public class PendingState : PaymentState
{
    public PendingState(Payment payment) : base(payment) { }

    public override async Task ProcessAsync()
    {
        // Process payment
        var result = await ProcessWithGatewayAsync();

        if (result.Success)
        {
            TransitionTo(new ProcessingState(Payment));
        }
        else
        {
            TransitionTo(new FailedState(Payment));
        }
    }

    public override async Task CancelAsync()
    {
        TransitionTo(new CanceledState(Payment));
    }

    public override Task RefundAsync()
    {
        throw new InvalidOperationException("Cannot refund pending payment");
    }
}

public class ProcessingState : PaymentState
{
    public ProcessingState(Payment payment) : base(payment) { }

    public override async Task ProcessAsync()
    {
        // Already processing
        throw new InvalidOperationException("Payment already processing");
    }

    public override async Task CancelAsync()
    {
        // Attempt to cancel
        var result = await CancelWithGatewayAsync();

        if (result.Success)
        {
            TransitionTo(new CanceledState(Payment));
        }
    }

    public override Task RefundAsync()
    {
        throw new InvalidOperationException("Cannot refund processing payment");
    }

    public async Task CompleteAsync()
    {
        TransitionTo(new CompletedState(Payment));
    }
}

public class CompletedState : PaymentState
{
    public CompletedState(Payment payment) : base(payment) { }

    public override Task ProcessAsync()
    {
        throw new InvalidOperationException("Payment already completed");
    }

    public override Task CancelAsync()
    {
        throw new InvalidOperationException("Cannot cancel completed payment");
    }

    public override async Task RefundAsync()
    {
        var result = await RefundWithGatewayAsync();

        if (result.Success)
        {
            TransitionTo(new RefundedState(Payment));
        }
    }
}

// Usage
public class Payment
{
    public PaymentState State { get; set; }

    public async Task ProcessAsync()
    {
        await State.ProcessAsync();
    }

    public async Task CancelAsync()
    {
        await State.CancelAsync();
    }

    public async Task RefundAsync()
    {
        await State.RefundAsync();
    }
}
```

### State Transition Validation

```csharp
public class PaymentStateMachine
{
    private static readonly Dictionary<PaymentStatus, List<PaymentStatus>>
        AllowedTransitions = new()
    {
        [PaymentStatus.Pending] = new()
        {
            PaymentStatus.Processing,
            PaymentStatus.Canceled
        },
        [PaymentStatus.Processing] = new()
        {
            PaymentStatus.Completed,
            PaymentStatus.Failed,
            PaymentStatus.Canceled
        },
        [PaymentStatus.Completed] = new()
        {
            PaymentStatus.Refunded,
            PaymentStatus.PartiallyRefunded
        },
        [PaymentStatus.Failed] = new List<PaymentStatus>(),
        [PaymentStatus.Canceled] = new List<PaymentStatus>(),
        [PaymentStatus.Refunded] = new List<PaymentStatus>()
    };

    public static bool CanTransition(
        PaymentStatus from,
        PaymentStatus to)
    {
        return AllowedTransitions.TryGetValue(from, out var allowed) &&
               allowed.Contains(to);
    }

    public static void ValidateTransition(
        PaymentStatus from,
        PaymentStatus to)
    {
        if (!CanTransition(from, to))
        {
            throw new InvalidOperationException(
                $"Cannot transition from {from} to {to}");
        }
    }
}

// Usage in Payment entity
public class Payment
{
    private PaymentStatus _status;

    public PaymentStatus Status
    {
        get => _status;
        private set
        {
            PaymentStateMachine.ValidateTransition(_status, value);
            _status = value;
        }
    }

    public void MarkAsCompleted()
    {
        Status = PaymentStatus.Completed; // Validates transition
    }
}
```

---


---

## Specification Pattern

### Concept

Encapsulate business rules in reusable specifications.

```csharp
public interface ISpecification<T>
{
    bool IsSatisfiedBy(T candidate);
}

public class AndSpecification<T> : ISpecification<T>
{
    private readonly ISpecification<T> _left;
    private readonly ISpecification<T> _right;

    public AndSpecification(ISpecification<T> left, ISpecification<T> right)
    {
        _left = left;
        _right = right;
    }

    public bool IsSatisfiedBy(T candidate)
    {
        return _left.IsSatisfiedBy(candidate) && _right.IsSatisfiedBy(candidate);
    }
}

// Payment specifications
public class CompletedPaymentSpecification : ISpecification<Payment>
{
    public bool IsSatisfiedBy(Payment payment)
    {
        return payment.Status == PaymentStatus.Completed;
    }
}

public class RefundablePaymentSpecification : ISpecification<Payment>
{
    public bool IsSatisfiedBy(Payment payment)
    {
        return payment.Status == PaymentStatus.Completed &&
               payment.CompletedAt > DateTime.UtcNow.AddDays(-30);
    }
}

public class HighValuePaymentSpecification : ISpecification<Payment>
{
    private readonly decimal _threshold;

    public HighValuePaymentSpecification(decimal threshold)
    {
        _threshold = threshold;
    }

    public bool IsSatisfiedBy(Payment payment)
    {
        return payment.Amount >= _threshold;
    }
}

// Usage
public class PaymentService
{
    public async Task<bool> CanRefundAsync(Payment payment)
    {
        var refundableSpec = new RefundablePaymentSpecification();
        return refundableSpec.IsSatisfiedBy(payment);
    }

    public async Task<List<Payment>> GetHighValueCompletedPaymentsAsync()
    {
        var completedSpec = new CompletedPaymentSpecification();
        var highValueSpec = new HighValuePaymentSpecification(10000);
        var combinedSpec = new AndSpecification<Payment>(completedSpec, highValueSpec);

        var allPayments = await _repository.GetAllAsync();
        return allPayments.Where(p => combinedSpec.IsSatisfiedBy(p)).ToList();
    }
}

// For EF Core queries
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();

    public bool IsSatisfiedBy(T entity)
    {
        var predicate = ToExpression().Compile();
        return predicate(entity);
    }
}

public class CompletedPaymentDbSpecification : Specification<Payment>
{
    public override Expression<Func<Payment, bool>> ToExpression()
    {
        return p => p.Status == PaymentStatus.Completed;
    }
}

// Usage with EF Core
var spec = new CompletedPaymentDbSpecification();
var payments = await _context.Payments
    .Where(spec.ToExpression())
    .ToListAsync();
```

---


---

## Command Query Separation (Not CQRS)

### Concept

Methods either change state (commands) or return data (queries), never both.

```csharp
// ❌ Bad: Method does both
public class PaymentService
{
    public Payment ProcessAndGetPayment(PaymentRequest request)
    {
        // Changes state AND returns data
        var payment = new Payment();
        _repository.Save(payment);
        return payment;
    }
}

// ✅ Good: Separate command and query
public class PaymentService
{
    // Command: Changes state, returns void or success indicator
    public async Task ProcessPaymentAsync(PaymentRequest request)
    {
        var payment = new Payment();
        await _repository.SaveAsync(payment);
    }

    // Query: Returns data, doesn't change state
    public async Task<Payment> GetPaymentAsync(Guid id)
    {
        return await _repository.GetByIdAsync(id);
    }
}
```

---


---

## Memento Pattern (State Recovery)

### Concept

Save and restore object state for undo/recovery.

```csharp
public class PaymentMemento
{
    public Guid PaymentId { get; set; }
    public decimal Amount { get; set; }
    public PaymentStatus Status { get; set; }
    public DateTime CapturedAt { get; set; }
    public string SerializedState { get; set; }
}

public class Payment
{
    public Guid Id { get; set; }
    public decimal Amount { get; set; }
    public PaymentStatus Status { get; set; }
    public string TransactionId { get; set; }

    // Create memento (snapshot)
    public PaymentMemento SaveState()
    {
        return new PaymentMemento
        {
            PaymentId = Id,
            Amount = Amount,
            Status = Status,
            CapturedAt = DateTime.UtcNow,
            SerializedState = JsonSerializer.Serialize(this)
        };
    }

    // Restore from memento
    public void RestoreState(PaymentMemento memento)
    {
        var payment = JsonSerializer.Deserialize<Payment>(memento.SerializedState);
        Id = payment.Id;
        Amount = payment.Amount;
        Status = payment.Status;
        TransactionId = payment.TransactionId;
    }
}

public class PaymentHistory
{
    private readonly Stack<PaymentMemento> _history = new();

    public void SaveCheckpoint(Payment payment)
    {
        _history.Push(payment.SaveState());
    }

    public void Undo(Payment payment)
    {
        if (_history.Count > 0)
        {
            var memento = _history.Pop();
            payment.RestoreState(memento);
        }
    }

    public bool CanUndo => _history.Count > 0;
}

// Usage
public class PaymentService
{
    private readonly PaymentHistory _history = new();

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        var payment = new Payment();

        // Save state before processing
        _history.SaveCheckpoint(payment);

        try
        {
            payment.Status = PaymentStatus.Processing;
            var result = await _gateway.ProcessAsync(payment);

            if (!result.IsSuccess)
            {
                // Restore previous state on failure
                _history.Undo(payment);
            }

            return result;
        }
        catch
        {
            // Restore on exception
            _history.Undo(payment);
            throw;
        }
    }
}
```

---


---

## Time-Series Data Pattern

### Concept

Efficiently store and query time-based payment metrics.

```csharp
public class PaymentMetrics
{
    public DateTime Timestamp { get; set; }
    public string MetricName { get; set; }
    public double Value { get; set; }
    public Dictionary<string, string> Tags { get; set; }
}

public class TimeSeriesPaymentMetricsStore
{
    private readonly InfluxDBClient _influxDb;

    public async Task RecordMetricAsync(
        string metric,
        double value,
        Dictionary<string, string> tags = null)
    {
        var point = PointData
            .Measurement("payment_metrics")
            .Tag("metric", metric)
            .Field("value", value)
            .Timestamp(DateTime.UtcNow, WritePrecision.Ns);

        if (tags != null)
        {
            foreach (var tag in tags)
            {
                point = point.Tag(tag.Key, tag.Value);
            }
        }

        await _influxDb.GetWriteApiAsync().WritePointAsync(point);
    }

    public async Task<List<PaymentMetrics>> QueryMetricsAsync(
        string metric,
        DateTime start,
        DateTime end)
    {
        var query = $@"
            from(bucket: ""payments"")
                |> range(start: {start:yyyy-MM-ddTHH:mm:ssZ}, stop: {end:yyyy-MM-ddTHH:mm:ssZ})
                |> filter(fn: (r) => r._measurement == ""payment_metrics"" and r.metric == ""{metric}"")
        ";

        var tables = await _influxDb.GetQueryApi().QueryAsync(query);

        var results = new List<PaymentMetrics>();
        foreach (var table in tables)
        {
            foreach (var record in table.Records)
            {
                results.Add(new PaymentMetrics
                {
                    Timestamp = record.GetTime().GetValueOrDefault().ToDateTimeUtc(),
                    MetricName = record.GetValueByKey("metric").ToString(),
                    Value = Convert.ToDouble(record.GetValue()),
                    Tags = record.Values
                        .Where(kv => kv.Key.StartsWith("tag_"))
                        .ToDictionary(kv => kv.Key, kv => kv.Value.ToString())
                });
            }
        }

        return results;
    }
}

// Usage
public class PaymentService
{
    private readonly TimeSeriesPaymentMetricsStore _metrics;

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        var sw = Stopwatch.StartNew();

        try
        {
            var result = await ProcessPaymentInternalAsync(request);

            sw.Stop();

            // Record metrics
            await _metrics.RecordMetricAsync("payment_duration", sw.ElapsedMilliseconds, new Dictionary<string, string>
            {
                ["currency"] = request.Currency,
                ["amount_range"] = GetAmountRange(request.Amount),
                ["success"] = result.IsSuccess.ToString()
            });

            await _metrics.RecordMetricAsync("payment_count", 1, new Dictionary<string, string>
            {
                ["status"] = result.IsSuccess ? "success" : "failure"
            });

            return result;
        }
        catch
        {
            sw.Stop();
            await _metrics.RecordMetricAsync("payment_error", 1);
            throw;
        }
    }
}
```

---


---

# Part 5: Infrastructure Patterns

## API Gateway Pattern

### Concept

Single entry point for multiple microservices, handling cross-cutting concerns.

```
Client → API Gateway → ┌─ Payment Service
                       ├─ Order Service
                       ├─ Customer Service
                       └─ Notification Service
```

### Implementation

```csharp
public class PaymentApiGateway
{
    private readonly IPaymentService _paymentService;
    private readonly IOrderService _orderService;
    private readonly ICustomerService _customerService;
    private readonly INotificationService _notificationService;
    private readonly ILogger<PaymentApiGateway> _logger;

    [HttpPost("api/payments/complete")]
    public async Task<CompletePaymentResponse> CompletePayment(
        CompletePaymentRequest request)
    {
        // 1. Validate customer
        var customer = await _customerService.GetCustomerAsync(request.CustomerId);
        if (customer == null)
            return CompletePaymentResponse.InvalidCustomer();

        // 2. Get order details
        var order = await _orderService.GetOrderAsync(request.OrderId);
        if (order == null)
            return CompletePaymentResponse.InvalidOrder();

        // 3. Process payment
        var paymentResult = await _paymentService.ProcessPaymentAsync(new PaymentRequest
        {
            OrderId = order.Id,
            Amount = order.TotalAmount,
            Currency = order.Currency,
            CustomerId = customer.Id,
            PaymentMethodId = request.PaymentMethodId
        });

        if (!paymentResult.IsSuccess)
            return CompletePaymentResponse.PaymentFailed(paymentResult.ErrorMessage);

        // 4. Update order status
        await _orderService.MarkAsPaidAsync(order.Id, paymentResult.TransactionId);

        // 5. Send notifications (fire and forget)
        _ = _notificationService.SendPaymentConfirmationAsync(
            customer.Email,
            paymentResult.TransactionId,
            order.TotalAmount);

        return CompletePaymentResponse.Success(paymentResult.TransactionId);
    }

    [HttpGet("api/payments/{id}/summary")]
    public async Task<PaymentSummary> GetPaymentSummary(Guid id)
    {
        // Aggregate data from multiple services
        var payment = await _paymentService.GetPaymentAsync(id);
        var order = await _orderService.GetOrderAsync(payment.OrderId);
        var customer = await _customerService.GetCustomerAsync(payment.CustomerId);

        return new PaymentSummary
        {
            PaymentId = payment.Id,
            Amount = payment.Amount,
            Status = payment.Status,
            OrderNumber = order.OrderNumber,
            CustomerName = customer.Name,
            CustomerEmail = customer.Email,
            OrderItems = order.Items
        };
    }
}
```

### Gateway Features

```csharp
public class ApiGatewayMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IAuthenticationService _auth;
    private readonly IRateLimiter _rateLimiter;
    private readonly ILogger<ApiGatewayMiddleware> _logger;

    public async Task InvokeAsync(HttpContext context)
    {
        var requestId = Guid.NewGuid().ToString();
        context.Items["RequestId"] = requestId;

        _logger.LogInformation("Gateway request {RequestId}: {Method} {Path}",
            requestId, context.Request.Method, context.Request.Path);

        // 1. Authentication
        if (!await _auth.AuthenticateAsync(context))
        {
            context.Response.StatusCode = 401;
            return;
        }

        // 2. Rate Limiting
        var identifier = context.User.Identity?.Name ?? context.Connection.RemoteIpAddress?.ToString();
        if (!await _rateLimiter.IsAllowedAsync(identifier))
        {
            context.Response.StatusCode = 429;
            context.Response.Headers.Add("Retry-After", "60");
            return;
        }

        // 3. Request transformation
        await TransformRequestAsync(context);

        // 4. Forward to backend
        var sw = Stopwatch.StartNew();
        await _next(context);
        sw.Stop();

        // 5. Response transformation
        await TransformResponseAsync(context);

        _logger.LogInformation("Gateway request {RequestId} completed in {Duration}ms",
            requestId, sw.ElapsedMilliseconds);
    }
}
```

---


---

## Sidecar Pattern

### Concept

Deploy helper components alongside main application.

```csharp
// Main Payment Service
public class PaymentService
{
    private readonly IPaymentGateway _gateway;

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        return await _gateway.ProcessPaymentAsync(request);
    }
}

// Sidecar: Logging
public class LoggingSidecar : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Collect logs from main service
        // Forward to centralized logging
    }
}

// Sidecar: Metrics
public class MetricsSidecar : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // Collect metrics
            // Send to monitoring system
            await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
        }
    }
}

// Kubernetes Sidecar Example
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  template:
    spec:
      containers:
      # Main container
      - name: payment-service
        image: payment-service:latest
        ports:
        - containerPort: 8080

      # Logging sidecar
      - name: logging-agent
        image: fluentd:latest
        volumeMounts:
        - name: logs
          mountPath: /var/log

      # Metrics sidecar
      - name: metrics-agent
        image: prometheus-exporter:latest
        ports:
        - containerPort: 9090
```

---


---

## Ambassador Pattern

### Concept

Proxy that handles cross-cutting concerns for outbound requests.

```csharp
public class PaymentGatewayAmbassador : IPaymentGateway
{
    private readonly IPaymentGateway _actualGateway;
    private readonly ICircuitBreaker _circuitBreaker;
    private readonly IRetryPolicy _retryPolicy;
    private readonly ILogger _logger;
    private readonly IMetrics _metrics;

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        var sw = Stopwatch.StartNew();

        try
        {
            // Circuit breaker
            if (_circuitBreaker.IsOpen)
            {
                _logger.LogWarning("Circuit breaker is open for payment gateway");
                return PaymentResult.CircuitBreakerOpen();
            }

            // Retry with exponential backoff
            var result = await _retryPolicy.ExecuteAsync(async () =>
            {
                return await _actualGateway.ProcessPaymentAsync(request);
            });

            sw.Stop();

            // Metrics
            _metrics.RecordPaymentDuration(sw.ElapsedMilliseconds);
            _metrics.RecordPaymentSuccess();

            // Circuit breaker success
            _circuitBreaker.RecordSuccess();

            return result;
        }
        catch (Exception ex)
        {
            sw.Stop();

            _logger.LogError(ex, "Payment processing failed");
            _metrics.RecordPaymentFailure();
            _circuitBreaker.RecordFailure();

            throw;
        }
    }
}
```

---


---

## Gatekeeper Pattern

### Concept

Additional security layer to validate and sanitize requests.

```csharp
public class PaymentGatekeeper
{
    private readonly IPaymentValidator _validator;
    private readonly IFraudDetection _fraudDetection;
    private readonly IRateLimiter _rateLimiter;
    private readonly ILogger<PaymentGatekeeper> _logger;

    public async Task<GatekeeperResult> ValidatePaymentRequestAsync(
        PaymentRequest request,
        HttpContext context)
    {
        var violations = new List<string>();

        // 1. Input validation
        if (!_validator.Validate(request, out var validationErrors))
        {
            violations.AddRange(validationErrors);
        }

        // 2. Rate limiting
        var clientId = context.User.Identity?.Name ?? context.Connection.RemoteIpAddress?.ToString();
        if (!await _rateLimiter.IsAllowedAsync(clientId))
        {
            violations.Add("Rate limit exceeded");
        }

        // 3. Fraud detection
        var fraudScore = await _fraudDetection.CalculateRiskScoreAsync(request, context);
        if (fraudScore > 0.8) // High risk
        {
            violations.Add("High fraud risk detected");
            _logger.LogWarning("Suspicious payment attempt: {PaymentId}, Score: {Score}",
                request.PaymentId, fraudScore);
        }

        // 4. IP validation
        var ipAddress = context.Connection.RemoteIpAddress?.ToString();
        if (await _fraudDetection.IsBlacklistedIpAsync(ipAddress))
        {
            violations.Add("IP address is blacklisted");
        }

        // 5. Amount validation
        if (request.Amount > 10000 && !request.IsVerified)
        {
            violations.Add("Large payment requires additional verification");
        }

        if (violations.Any())
        {
            return GatekeeperResult.Blocked(violations);
        }

        return GatekeeperResult.Allowed();
    }
}

public class GatekeeperMiddleware
{
    private readonly RequestDelegate _next;
    private readonly PaymentGatekeeper _gatekeeper;

    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.Path.StartsWithSegments("/api/payments"))
        {
            var request = await context.Request.ReadFromJsonAsync<PaymentRequest>();
            var result = await _gatekeeper.ValidatePaymentRequestAsync(request, context);

            if (!result.IsAllowed)
            {
                context.Response.StatusCode = 403;
                await context.Response.WriteAsJsonAsync(new
                {
                    error = "Request blocked by security gateway",
                    violations = result.Violations
                });
                return;
            }
        }

        await _next(context);
    }
}
```

---


---

## Valet Key Pattern

### Concept

Grant limited, time-bound access to resources.

```csharp
public class ValetKeyService
{
    private readonly IBlobStorage _blobStorage;

    // Generate temporary access URL for receipt
    public async Task<string> GenerateReceiptAccessUrlAsync(
        Guid paymentId,
        string userId)
    {
        // Verify user has access to this payment
        var payment = await _paymentRepository.GetByIdAsync(paymentId);
        if (payment.CustomerId != userId)
        {
            throw new UnauthorizedAccessException();
        }

        // Generate SAS token (Azure) or presigned URL (AWS S3)
        var blobPath = $"receipts/{paymentId}.pdf";
        var sasToken = _blobStorage.GenerateSasToken(
            blobPath,
            permissions: BlobSasPermissions.Read,
            expiresIn: TimeSpan.FromHours(1));

        return $"{_blobStorage.BaseUrl}/{blobPath}?{sasToken}";
    }

    // AWS S3 Presigned URL
    public string GenerateS3PresignedUrl(string key, TimeSpan expiration)
    {
        var request = new GetPreSignedUrlRequest
        {
            BucketName = "payment-receipts",
            Key = key,
            Expires = DateTime.UtcNow.Add(expiration),
            Verb = HttpVerb.GET
        };

        return _s3Client.GetPreSignedURL(request);
    }

    // Azure Blob SAS Token
    public string GenerateAzureSasToken(string blobName, TimeSpan expiration)
    {
        var blobClient = _blobContainerClient.GetBlobClient(blobName);

        var sasBuilder = new BlobSasBuilder
        {
            BlobContainerName = "receipts",
            BlobName = blobName,
            Resource = "b",
            StartsOn = DateTimeOffset.UtcNow,
            ExpiresOn = DateTimeOffset.UtcNow.Add(expiration)
        };

        sasBuilder.SetPermissions(BlobSasPermissions.Read);

        return blobClient.GenerateSasUri(sasBuilder).ToString();
    }
}
```

---


---

# Part 6: Deployment Patterns

## Blue-Green Deployment for Payments

### Concept

Run two identical production environments (Blue and Green). Switch traffic between them for zero-downtime deployments.

```csharp
public class BlueGreenDeploymentController
{
    private readonly IPaymentService _blueService;
    private readonly IPaymentService _greenService;
    private readonly IConfiguration _config;

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        var activeEnvironment = _config["ActiveEnvironment"]; // "blue" or "green"

        var service = activeEnvironment == "green" ? _greenService : _blueService;

        return await service.ProcessPaymentAsync(request);
    }

    public async Task SwitchEnvironmentAsync()
    {
        var current = _config["ActiveEnvironment"];
        var next = current == "blue" ? "green" : "blue";

        // Update configuration
        await UpdateConfigurationAsync("ActiveEnvironment", next);

        // Warm up the new environment
        await WarmUpEnvironmentAsync(next);

        // Switch traffic
        // (In practice, this would be done at load balancer level)
    }
}

// Kubernetes example
```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment
    version: blue  # Switch to 'green' to change
  ports:
    - port: 80

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment
      version: blue
  template:
    metadata:
      labels:
        app: payment
        version: blue
    spec:
      containers:
      - name: payment
        image: payment-service:v1.0.0

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment
      version: green
  template:
    metadata:
      labels:
        app: payment
        version: green
    spec:
      containers:
      - name: payment
        image: payment-service:v1.1.0  # New version
```

---


---

## Canary Release Pattern

### Concept

Gradually roll out new version to a small percentage of users.

```csharp
public class CanaryReleaseRouter
{
    private readonly IPaymentService _stableService;
    private readonly IPaymentService _canaryService;
    private readonly IMetricsCollector _metrics;

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        // Determine if this request goes to canary
        var useCanary = ShouldRouteToCanary(request);

        var service = useCanary ? _canaryService : _stableService;
        var version = useCanary ? "canary" : "stable";

        var sw = Stopwatch.StartNew();

        try
        {
            var result = await service.ProcessPaymentAsync(request);

            sw.Stop();

            // Collect metrics
            _metrics.RecordPayment(version, result.IsSuccess, sw.ElapsedMilliseconds);

            // Auto-rollback if canary has high error rate
            await CheckCanaryHealthAsync();

            return result;
        }
        catch (Exception ex)
        {
            sw.Stop();
            _metrics.RecordPayment(version, false, sw.ElapsedMilliseconds);
            throw;
        }
    }

    private bool ShouldRouteToCanary(PaymentRequest request)
    {
        // Strategy 1: Percentage-based
        var canaryPercentage = GetCanaryPercentage(); // 0-100
        var hash = Math.Abs(request.CustomerId.GetHashCode());
        return (hash % 100) < canaryPercentage;

        // Strategy 2: Whitelist
        // return _canaryUserList.Contains(request.CustomerId);

        // Strategy 3: Ring-based (internal users first)
        // return _internalUserList.Contains(request.CustomerId);
    }

    private async Task CheckCanaryHealthAsync()
    {
        var canaryMetrics = await _metrics.GetMetricsAsync("canary");
        var stableMetrics = await _metrics.GetMetricsAsync("stable");

        // If canary error rate > stable error rate * 2, rollback
        if (canaryMetrics.ErrorRate > stableMetrics.ErrorRate * 2)
        {
            await RollbackCanaryAsync();
        }
    }
}
```

---


---

## Feature Flags for Payments

### Concept

Toggle features on/off without deploying code.

```csharp
public class FeatureFlagService
{
    private readonly IDistributedCache _cache;
    private readonly IConfiguration _config;

    public async Task<bool> IsEnabledAsync(
        string featureName,
        string userId = null,
        Dictionary<string, string> context = null)
    {
        // Check cache
        var cacheKey = $"feature:{featureName}:{userId}";
        var cached = await _cache.GetStringAsync(cacheKey);
        if (cached != null)
        {
            return bool.Parse(cached);
        }

        // Evaluate feature flag
        var isEnabled = await EvaluateFeatureFlagAsync(featureName, userId, context);

        // Cache result
        await _cache.SetStringAsync(cacheKey, isEnabled.ToString(),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
            });

        return isEnabled;
    }

    private async Task<bool> EvaluateFeatureFlagAsync(
        string featureName,
        string userId,
        Dictionary<string, string> context)
    {
        var flag = _config.GetSection($"Features:{featureName}");

        // Simple on/off
        if (flag["Enabled"] != null)
        {
            return bool.Parse(flag["Enabled"]);
        }

        // Percentage rollout
        if (flag["Percentage"] != null)
        {
            var percentage = int.Parse(flag["Percentage"]);
            var hash = Math.Abs((userId ?? "").GetHashCode());
            return (hash % 100) < percentage;
        }

        // User whitelist
        if (flag["Users"] != null)
        {
            var allowedUsers = flag["Users"].Split(',');
            return allowedUsers.Contains(userId);
        }

        // Context-based rules
        if (flag["Rules"] != null && context != null)
        {
            // Evaluate complex rules
            return EvaluateRules(flag["Rules"], context);
        }

        return false;
    }
}

// Usage in payment service
public class PaymentService
{
    private readonly FeatureFlagService _featureFlags;

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        // Feature: New fraud detection algorithm
        if (await _featureFlags.IsEnabledAsync("enhanced-fraud-detection", request.CustomerId))
        {
            return await ProcessWithEnhancedFraudDetectionAsync(request);
        }
        else
        {
            return await ProcessWithStandardFlowAsync(request);
        }
    }
}

// Configuration
```json
{
  "Features": {
    "enhanced-fraud-detection": {
      "Percentage": 10,
      "Users": "user1,user2,user3"
    },
    "new-payment-provider": {
      "Enabled": false
    },
    "3d-secure-v2": {
      "Enabled": true,
      "Rules": "amount > 100 && country == 'EU'"
    }
  }
}
```

### Gradual Rollout

```csharp
public class GradualRolloutFeatureToggle : IFeatureToggleService
{
    private readonly IConfiguration _configuration;

    public bool IsEnabled(string featureName)
    {
        var rolloutPercentage = _configuration
            .GetValue<int>($"Features:{featureName}:RolloutPercentage");

        if (rolloutPercentage == 0) return false;
        if (rolloutPercentage >= 100) return true;

        // Use deterministic hash of user ID
        var userId = GetCurrentUserId();
        var hash = Math.Abs(userId.GetHashCode());
        var bucket = hash % 100;

        return bucket < rolloutPercentage;
    }
}

// appsettings.json
{
  "Features": {
    "NewPaymentProvider": {
      "RolloutPercentage": 10  // 10% of users
    }
  }
}
```

---


---

# Part 7: Distributed Systems

## Eventual Consistency Patterns

### Concept

Accept that data won't be immediately consistent across all services, but will eventually converge.

### Pattern 1: Read Your Own Writes

```csharp
public class EventualConsistencyPaymentService
{
    private readonly IPaymentRepository _writeRepo;
    private readonly IPaymentReadRepository _readRepo;
    private readonly IEventBus _eventBus;
    private readonly IMemoryCache _cache;

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        // Write to primary database
        var payment = new Payment { /* ... */ };
        await _writeRepo.SaveAsync(payment);

        // Store in local cache (read your own writes)
        _cache.Set($"payment:{payment.Id}", payment, TimeSpan.FromMinutes(5));

        // Publish event for eventual consistency
        await _eventBus.PublishAsync(new PaymentCreatedEvent
        {
            PaymentId = payment.Id,
            // ... data
        });

        return PaymentResult.Success(payment.Id);
    }

    public async Task<Payment> GetPaymentAsync(Guid id)
    {
        // Check local cache first (for recent writes)
        if (_cache.TryGetValue($"payment:{id}", out Payment cached))
        {
            return cached;
        }

        // Otherwise query read replica (may be slightly stale)
        return await _readRepo.GetByIdAsync(id);
    }
}
```

### Pattern 2: Version Vectors

```csharp
public class VersionVector
{
    private readonly Dictionary<string, long> _versions = new();

    public void Increment(string nodeId)
    {
        _versions.TryGetValue(nodeId, out var current);
        _versions[nodeId] = current + 1;
    }

    public bool IsConcurrent(VersionVector other)
    {
        var thisGreater = false;
        var otherGreater = false;

        foreach (var kvp in _versions)
        {
            var otherVersion = other._versions.GetValueOrDefault(kvp.Key);
            if (kvp.Value > otherVersion) thisGreater = true;
            if (kvp.Value < otherVersion) otherGreater = true;
        }

        return thisGreater && otherGreater; // Concurrent updates
    }
}

public class Payment
{
    public Guid Id { get; set; }
    public decimal Amount { get; set; }
    public VersionVector Version { get; set; } = new();

    public void UpdateAmount(decimal newAmount, string nodeId)
    {
        Amount = newAmount;
        Version.Increment(nodeId);
    }
}
```

### Pattern 3: Conflict Resolution

```csharp
public interface IConflictResolver<T>
{
    T Resolve(T version1, T version2);
}

public class LastWriteWinsResolver : IConflictResolver<Payment>
{
    public Payment Resolve(Payment v1, Payment v2)
    {
        return v1.UpdatedAt > v2.UpdatedAt ? v1 : v2;
    }
}

public class HighestAmountWinsResolver : IConflictResolver<Payment>
{
    public Payment Resolve(Payment v1, Payment v2)
    {
        return v1.Amount > v2.Amount ? v1 : v2;
    }
}

public class MergeResolver : IConflictResolver<Payment>
{
    public Payment Resolve(Payment v1, Payment v2)
    {
        // Custom merge logic
        return new Payment
        {
            Id = v1.Id,
            Amount = Math.Max(v1.Amount, v2.Amount),
            Status = v1.Status == PaymentStatus.Completed || v2.Status == PaymentStatus.Completed
                ? PaymentStatus.Completed
                : PaymentStatus.Pending
        };
    }
}
```

---


---

## Two-Phase Commit (2PC)

### Concept

Coordinate distributed transactions across multiple resources.

```
Phase 1: Prepare (Vote)
Coordinator → Participant A: Can you commit?
Coordinator → Participant B: Can you commit?

Phase 2: Commit (Decision)
Coordinator → Participant A: Commit!
Coordinator → Participant B: Commit!
```

### Implementation

```csharp
public class TwoPhaseCommitCoordinator
{
    private readonly List<ITransactionParticipant> _participants;

    public async Task<bool> ExecuteTransactionAsync(PaymentTransaction transaction)
    {
        var transactionId = Guid.NewGuid();
        var preparedParticipants = new List<ITransactionParticipant>();

        try
        {
            // PHASE 1: PREPARE
            foreach (var participant in _participants)
            {
                var prepared = await participant.PrepareAsync(transactionId, transaction);

                if (!prepared)
                {
                    // One participant can't commit - abort all
                    await RollbackAsync(transactionId, preparedParticipants);
                    return false;
                }

                preparedParticipants.Add(participant);
            }

            // PHASE 2: COMMIT
            foreach (var participant in preparedParticipants)
            {
                await participant.CommitAsync(transactionId);
            }

            return true;
        }
        catch (Exception)
        {
            // Error during commit - rollback
            await RollbackAsync(transactionId, preparedParticipants);
            return false;
        }
    }

    private async Task RollbackAsync(
        Guid transactionId,
        List<ITransactionParticipant> participants)
    {
        foreach (var participant in participants)
        {
            try
            {
                await participant.RollbackAsync(transactionId);
            }
            catch
            {
                // Log but continue rolling back others
            }
        }
    }
}

public interface ITransactionParticipant
{
    Task<bool> PrepareAsync(Guid transactionId, PaymentTransaction transaction);
    Task CommitAsync(Guid transactionId);
    Task RollbackAsync(Guid transactionId);
}

public class InventoryParticipant : ITransactionParticipant
{
    private readonly Dictionary<Guid, InventoryReservation> _reservations = new();

    public async Task<bool> PrepareAsync(Guid txId, PaymentTransaction transaction)
    {
        // Check if we can reserve inventory
        var available = await _inventoryService.CheckAvailabilityAsync(transaction.Items);

        if (available)
        {
            // Lock resources but don't commit
            var reservation = await _inventoryService.ReserveAsync(transaction.Items);
            _reservations[txId] = reservation;
            return true;
        }

        return false;
    }

    public async Task CommitAsync(Guid txId)
    {
        var reservation = _reservations[txId];
        await _inventoryService.CommitReservationAsync(reservation);
        _reservations.Remove(txId);
    }

    public async Task RollbackAsync(Guid txId)
    {
        if (_reservations.TryGetValue(txId, out var reservation))
        {
            await _inventoryService.ReleaseReservationAsync(reservation);
            _reservations.Remove(txId);
        }
    }
}

public class PaymentParticipant : ITransactionParticipant
{
    private readonly Dictionary<Guid, string> _authorizationIds = new();

    public async Task<bool> PrepareAsync(Guid txId, PaymentTransaction transaction)
    {
        // Authorize payment but don't capture
        var result = await _gateway.AuthorizeAsync(
            transaction.Amount,
            transaction.PaymentMethodId);

        if (result.IsSuccess)
        {
            _authorizationIds[txId] = result.AuthorizationId;
            return true;
        }

        return false;
    }

    public async Task CommitAsync(Guid txId)
    {
        var authId = _authorizationIds[txId];
        await _gateway.CaptureAsync(authId);
        _authorizationIds.Remove(txId);
    }

    public async Task RollbackAsync(Guid txId)
    {
        if (_authorizationIds.TryGetValue(txId, out var authId))
        {
            await _gateway.VoidAuthorizationAsync(authId);
            _authorizationIds.Remove(txId);
        }
    }
}
```

**Note:** 2PC is blocking and should be avoided when possible. Prefer Saga pattern for distributed transactions.

---


---

## Distributed Lock Pattern

### Concept

Ensure only one instance processes a payment at a time.

### Redis-based Implementation

```csharp
public class RedisDistributedLock
{
    private readonly IConnectionMultiplexer _redis;

    public async Task<bool> AcquireLockAsync(
        string resource,
        string owner,
        TimeSpan expiry)
    {
        var db = _redis.GetDatabase();
        var key = $"lock:{resource}";

        // SET key owner NX PX expiry_ms
        return await db.StringSetAsync(
            key,
            owner,
            expiry,
            When.NotExists);
    }

    public async Task<bool> ReleaseLockAsync(string resource, string owner)
    {
        var db = _redis.GetDatabase();
        var key = $"lock:{resource}";

        // Lua script for atomic check-and-delete
        var script = @"
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
        ";

        var result = await db.ScriptEvaluateAsync(
            script,
            new RedisKey[] { key },
            new RedisValue[] { owner });

        return (int)result == 1;
    }

    public async Task<T> ExecuteWithLockAsync<T>(
        string resource,
        Func<Task<T>> action,
        TimeSpan lockTimeout,
        TimeSpan waitTimeout)
    {
        var owner = Guid.NewGuid().ToString();
        var deadline = DateTime.UtcNow.Add(waitTimeout);

        // Try to acquire lock with retry
        while (DateTime.UtcNow < deadline)
        {
            if (await AcquireLockAsync(resource, owner, lockTimeout))
            {
                try
                {
                    return await action();
                }
                finally
                {
                    await ReleaseLockAsync(resource, owner);
                }
            }

            await Task.Delay(100); // Backoff
        }

        throw new TimeoutException($"Could not acquire lock on {resource}");
    }
}

// Usage
public class PaymentService
{
    private readonly RedisDistributedLock _distributedLock;

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        // Ensure only one instance processes this payment
        return await _distributedLock.ExecuteWithLockAsync(
            $"payment:{request.PaymentId}",
            async () =>
            {
                return await ProcessPaymentInternalAsync(request);
            },
            lockTimeout: TimeSpan.FromSeconds(30),
            waitTimeout: TimeSpan.FromSeconds(5));
    }
}
```

---


---

## Leader Election Pattern

### Concept

One instance becomes leader to coordinate work.

### Implementation with Redis

```csharp
public class RedisLeaderElection
{
    private readonly IConnectionMultiplexer _redis;
    private readonly string _instanceId;
    private bool _isLeader;
    private CancellationTokenSource _heartbeatCts;

    public RedisLeaderElection(IConnectionMultiplexer redis)
    {
        _redis = redis;
        _instanceId = $"{Environment.MachineName}_{Process.GetCurrentProcess().Id}";
    }

    public async Task<bool> TryBecomeLeaderAsync()
    {
        var db = _redis.GetDatabase();
        var leaderKey = "payment-service:leader";

        // Try to set ourselves as leader
        _isLeader = await db.StringSetAsync(
            leaderKey,
            _instanceId,
            TimeSpan.FromSeconds(30),
            When.NotExists);

        if (_isLeader)
        {
            // Start heartbeat to maintain leadership
            _heartbeatCts = new CancellationTokenSource();
            _ = Task.Run(() => MaintainLeadershipAsync(_heartbeatCts.Token));
        }

        return _isLeader;
    }

    private async Task MaintainLeadershipAsync(CancellationToken cancellationToken)
    {
        var db = _redis.GetDatabase();
        var leaderKey = "payment-service:leader";

        while (!cancellationToken.IsCancellationRequested)
        {
            try
            {
                // Extend our leadership
                var script = @"
                    if redis.call('get', KEYS[1]) == ARGV[1] then
                        return redis.call('expire', KEYS[1], ARGV[2])
                    else
                        return 0
                    end
                ";

                var result = await db.ScriptEvaluateAsync(
                    script,
                    new RedisKey[] { leaderKey },
                    new RedisValue[] { _instanceId, 30 });

                if ((int)result == 0)
                {
                    // Lost leadership
                    _isLeader = false;
                    break;
                }

                await Task.Delay(TimeSpan.FromSeconds(10), cancellationToken);
            }
            catch
            {
                _isLeader = false;
                break;
            }
        }
    }

    public async Task ResignLeadershipAsync()
    {
        _heartbeatCts?.Cancel();

        var db = _redis.GetDatabase();
        var leaderKey = "payment-service:leader";

        // Release leadership if we're still the leader
        var script = @"
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
        ";

        await db.ScriptEvaluateAsync(
            script,
            new RedisKey[] { leaderKey },
            new RedisValue[] { _instanceId });

        _isLeader = false;
    }
}

// Usage: Only leader processes reconciliation
public class ReconciliationService : BackgroundService
{
    private readonly RedisLeaderElection _leaderElection;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                // Try to become leader
                if (await _leaderElection.TryBecomeLeaderAsync())
                {
                    Console.WriteLine("I am the leader! Starting reconciliation...");

                    // Do leader work
                    await PerformReconciliationAsync(stoppingToken);
                }
                else
                {
                    Console.WriteLine("I am a follower. Waiting...");
                    await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
                }
            }
            catch
            {
                await _leaderElection.ResignLeadershipAsync();
                await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
            }
        }
    }
}
```

---


---

## Sharding Pattern

### Concept

Partition data across multiple databases based on a key.

### Implementation

```csharp
public class ShardedPaymentRepository : IPaymentRepository
{
    private readonly List<IPaymentRepository> _shards;
    private readonly IShardResolver _shardResolver;

    public ShardedPaymentRepository(List<IPaymentRepository> shards, IShardResolver shardResolver)
    {
        _shards = shards;
        _shardResolver = shardResolver;
    }

    public async Task<Payment> GetByIdAsync(Guid id)
    {
        var shard = _shardResolver.GetShard(id, _shards.Count);
        return await _shards[shard].GetByIdAsync(id);
    }

    public async Task SaveAsync(Payment payment)
    {
        var shard = _shardResolver.GetShard(payment.Id, _shards.Count);
        await _shards[shard].SaveAsync(payment);
    }

    public async Task<List<Payment>> GetByCustomerIdAsync(string customerId)
    {
        // Query all shards in parallel
        var tasks = _shards.Select(shard =>
            shard.GetByCustomerIdAsync(customerId));

        var results = await Task.WhenAll(tasks);

        return results.SelectMany(r => r).ToList();
    }
}

public interface IShardResolver
{
    int GetShard(Guid id, int shardCount);
    int GetShard(string customerId, int shardCount);
}

public class ConsistentHashShardResolver : IShardResolver
{
    public int GetShard(Guid id, int shardCount)
    {
        var hash = GetHash(id.ToString());
        return Math.Abs(hash % shardCount);
    }

    public int GetShard(string customerId, int shardCount)
    {
        var hash = GetHash(customerId);
        return Math.Abs(hash % shardCount);
    }

    private int GetHash(string key)
    {
        using var md5 = MD5.Create();
        var hash = md5.ComputeHash(Encoding.UTF8.GetBytes(key));
        return BitConverter.ToInt32(hash, 0);
    }
}

// Configuration
services.AddSingleton<IPaymentRepository>(sp =>
{
    var shards = new List<IPaymentRepository>
    {
        new SqlPaymentRepository("Shard0ConnectionString"),
        new SqlPaymentRepository("Shard1ConnectionString"),
        new SqlPaymentRepository("Shard2ConnectionString"),
        new SqlPaymentRepository("Shard3ConnectionString")
    };

    var resolver = new ConsistentHashShardResolver();

    return new ShardedPaymentRepository(shards, resolver);
});
```

---


---

# Part 8: Anti-Patterns & Domain-Specific

## Anti-Pattern Avoidance

### God Object Anti-Pattern

**❌ Bad:**
```csharp
public class PaymentManager
{
    public void ProcessPayment() { }
    public void RefundPayment() { }
    public void ValidatePayment() { }
    public void SendEmail() { }
    public void LogToDatabase() { }
    public void CalculateTax() { }
    public void CheckInventory() { }
    // ... 50 more methods
}
```

**✅ Good:**
```csharp
public class PaymentProcessor { }
public class PaymentValidator { }
public class NotificationService { }
public class AuditLogger { }
public class TaxCalculator { }
public class InventoryChecker { }
```

### Database-as-IPC Anti-Pattern

**❌ Bad:**
```csharp
// Service A writes to shared table
await _db.ExecuteSqlAsync("INSERT INTO SharedQueue (Message) VALUES (...)");

// Service B polls shared table
while (true)
{
    var messages = await _db.ExecuteSqlAsync("SELECT * FROM SharedQueue");
    // Process messages
}
```

**✅ Good:**
```csharp
// Use proper message queue
await _messageQueue.SendAsync(message);
```

### Distributed Monolith Anti-Pattern

**❌ Bad:**
```
Service A → HTTP → Service B → HTTP → Service C → HTTP → Service D
(All services tightly coupled, must deploy together)
```

**✅ Good:**
```
Service A → Event Bus
Service B → Subscribes to events (independent deployment)
Service C → Subscribes to events (independent deployment)
```

---


---

## Geofencing Pattern

### Concept

Restrict or route payments based on geographic location.

```csharp
public class GeofencingService
{
    private readonly IHttpClientFactory _httpClientFactory;

    public async Task<GeoLocation> GetLocationAsync(string ipAddress)
    {
        var client = _httpClientFactory.CreateClient();
        var response = await client.GetAsync($"https://ipapi.co/{ipAddress}/json/");
        var data = await response.Content.ReadFromJsonAsync<IpApiResponse>();

        return new GeoLocation
        {
            Country = data.CountryCode,
            Region = data.Region,
            City = data.City,
            Latitude = data.Latitude,
            Longitude = data.Longitude
        };
    }

    public bool IsWithinGeofence(GeoLocation location, Geofence fence)
    {
        var distance = CalculateDistance(
            location.Latitude, location.Longitude,
            fence.CenterLatitude, fence.CenterLongitude);

        return distance <= fence.RadiusKm;
    }

    private double CalculateDistance(double lat1, double lon1, double lat2, double lon2)
    {
        const double R = 6371; // Earth's radius in km

        var dLat = (lat2 - lat1) * Math.PI / 180;
        var dLon = (lon2 - lon1) * Math.PI / 180;

        var a = Math.Sin(dLat / 2) * Math.Sin(dLat / 2) +
                Math.Cos(lat1 * Math.PI / 180) * Math.Cos(lat2 * Math.PI / 180) *
                Math.Sin(dLon / 2) * Math.Sin(dLon / 2);

        var c = 2 * Math.Atan2(Math.Sqrt(a), Math.Sqrt(1 - a));

        return R * c;
    }
}

public class GeofencedPaymentService
{
    private readonly GeofencingService _geofencing;
    private readonly IPaymentService _paymentService;

    public async Task<PaymentResult> ProcessPaymentAsync(
        PaymentRequest request,
        string ipAddress)
    {
        var location = await _geofencing.GetLocationAsync(ipAddress);

        // Check if in restricted country
        if (IsRestrictedCountry(location.Country))
        {
            return PaymentResult.Blocked("Payments not available in your region");
        }

        // Check if in high-risk region (require additional verification)
        if (IsHighRiskRegion(location))
        {
            request.RequiresAdditional3DSecure = true;
        }

        // Route to regional payment provider
        var provider = SelectProviderByRegion(location);
        request.PreferredProvider = provider;

        return await _paymentService.ProcessPaymentAsync(request);
    }

    private bool IsRestrictedCountry(string countryCode)
    {
        var restricted = new[] { "KP", "IR", "SY" }; // Sanctioned countries
        return restricted.Contains(countryCode);
    }

    private bool IsHighRiskRegion(GeoLocation location)
    {
        // Implement risk assessment
        return _riskDatabase.GetRiskScore(location.Country) > 0.7;
    }

    private string SelectProviderByRegion(GeoLocation location)
    {
        return location.Country switch
        {
            "US" or "CA" or "MX" => "Stripe",
            "GB" or "FR" or "DE" => "Adyen",
            "BR" => "PagSeguro",
            "IN" => "Razorpay",
            _ => "PayPal"
        };
    }
}
```

---


---

## Summary Table

| # | Pattern | Category | Use Case | Complexity |
|---|---------|----------|----------|------------|
| 1 | **Hexagonal Architecture** | Architectural | Clean architecture with ports & adapters | High |
| 2 | **Onion Architecture** | Architectural | Layered design with inward dependencies | High |
| 3 | **Circuit Breaker** | Resilience | Prevent cascading failures | Medium |
| 4 | **Retry** | Resilience | Handle transient failures | Low |
| 5 | **Bulkhead** | Resilience | Isolate resource pools | Low |
| 6 | **Chaos Engineering** | Resilience | Test system resilience | Medium |
| 7 | **Saga** | Integration | Distributed transactions | High |
| 8 | **Idempotency** | Integration | Safe retries | Medium |
| 9 | **Rate Limiting & Throttling** | Integration | Prevent abuse and overload | Medium |
| 10 | **Reconciliation** | Integration | Detect discrepancies | Medium |
| 11 | **Notification** | Integration | Decouple event broadcasting | Low |
| 12 | **Priority Queue** | Integration | Process by importance | Medium |
| 13 | **Claim Check** | Integration | Handle large payloads | Low |
| 14 | **Pipes and Filters** | Integration | Sequential processing | Medium |
| 15 | **Chain of Responsibility** | Integration | Pipeline processing | Low |
| 16 | **Cache-Aside** | Data | Improve read performance | Low |
| 17 | **Materialized View** | Data | Complex query optimization | Medium |
| 18 | **Write-Behind Cache** | Data | Async persistence | Medium |
| 19 | **Multi-Currency** | Data | Currency handling | Medium |
| 20 | **Payment State Machine** | Data | Enforce valid state transitions | Medium |
| 21 | **Specification** | Data | Reusable business rules | Medium |
| 22 | **Command Query Separation** | Data | Clean code | Low |
| 23 | **Memento** | Data | State recovery/undo | Low |
| 24 | **Time-Series Data** | Data | Metrics storage | Medium |
| 25 | **API Gateway** | Infrastructure | Single entry point | Medium |
| 26 | **Sidecar** | Infrastructure | Cross-cutting concerns | Low |
| 27 | **Ambassador** | Infrastructure | Outbound proxy | Medium |
| 28 | **Gatekeeper** | Infrastructure | Security validation | Medium |
| 29 | **Valet Key** | Infrastructure | Temporary resource access | Low |
| 30 | **Blue-Green Deployment** | Deployment | Zero-downtime releases | Low |
| 31 | **Canary Release** | Deployment | Safe rollouts | Medium |
| 32 | **Feature Flags** | Deployment | Toggle features | Low |
| 33 | **Eventual Consistency** | Distributed | Accept delayed consistency | High |
| 34 | **Two-Phase Commit** | Distributed | Coordinated transactions | Very High |
| 35 | **Distributed Lock** | Distributed | Prevent duplicate processing | Medium |
| 36 | **Leader Election** | Distributed | Coordinate work | Medium |
| 37 | **Sharding** | Distributed | Scale data horizontally | High |
| 38 | **Anti-Pattern Avoidance** | Best Practices | Avoid common mistakes | Low |
| 39 | **Geofencing** | Domain-Specific | Location-based routing | Low |

## Next Steps

- [Architecture Patterns](01-ARCHITECTURE-PATTERNS.md)
- [Performance Optimization](08-PERFORMANCE-OPTIMIZATION.md)
- [CQRS and Event Sourcing](11-CQRS-EVENT-SOURCING.md)
- [Cloud-Native Architecture](09-CLOUD-NATIVE-AWS.md)
