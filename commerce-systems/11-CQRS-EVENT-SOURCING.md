# Advanced Architectural Patterns for Payment Systems

## Overview

This document covers advanced architectural patterns including CQRS (Command Query Responsibility Segregation), Event Sourcing, Outbox Pattern, and other enterprise patterns for building robust payment systems.

## Table of Contents

1. [CQRS Pattern](#cqrs-pattern)
2. [Event Sourcing](#event-sourcing)
3. [CQRS + Event Sourcing](#cqrs--event-sourcing)
4. [Outbox Pattern](#outbox-pattern)
5. [Saga Pattern (Orchestration vs Choreography)](#saga-pattern)
6. [Inbox Pattern](#inbox-pattern)
7. [Backend for Frontend (BFF)](#backend-for-frontend-bff)
8. [Strangler Fig Pattern](#strangler-fig-pattern)
9. [Anti-Corruption Layer](#anti-corruption-layer)
10. [Competing Consumers Pattern](#competing-consumers-pattern)

---

## CQRS Pattern

### Concept

**CQRS** separates read and write operations into different models:
- **Commands**: Change state (writes)
- **Queries**: Read state (reads)

### Why Use CQRS for Payments?

- ✅ Different optimization strategies for reads vs writes
- ✅ Scale reads and writes independently
- ✅ Complex queries don't impact write performance
- ✅ Better security (separate read/write permissions)
- ✅ Event sourcing integration

### Basic CQRS Implementation

```csharp
// Command Model (Write)
public interface ICommand { }

public class ProcessPaymentCommand : ICommand
{
    public Guid PaymentId { get; set; }
    public string OrderId { get; set; }
    public decimal Amount { get; set; }
    public string Currency { get; set; }
    public string PaymentMethodId { get; set; }
}

public class RefundPaymentCommand : ICommand
{
    public Guid PaymentId { get; set; }
    public decimal Amount { get; set; }
    public string Reason { get; set; }
}

// Query Model (Read)
public interface IQuery<TResult> { }

public class GetPaymentQuery : IQuery<PaymentDto>
{
    public Guid PaymentId { get; set; }
}

public class GetPaymentsByOrderQuery : IQuery<List<PaymentDto>>
{
    public string OrderId { get; set; }
}

public class GetPaymentHistoryQuery : IQuery<List<PaymentDto>>
{
    public string CustomerId { get; set; }
    public DateTime StartDate { get; set; }
    public DateTime EndDate { get; set; }
}
```

### Command Handlers

```csharp
public interface ICommandHandler<TCommand> where TCommand : ICommand
{
    Task HandleAsync(TCommand command);
}

public class ProcessPaymentCommandHandler : ICommandHandler<ProcessPaymentCommand>
{
    private readonly IPaymentRepository _repository;
    private readonly IPaymentGateway _gateway;
    private readonly IEventPublisher _eventPublisher;
    private readonly ILogger<ProcessPaymentCommandHandler> _logger;
    
    public async Task HandleAsync(ProcessPaymentCommand command)
    {
        _logger.LogInformation("Processing payment command for {OrderId}", command.OrderId);
        
        // Create payment entity
        var payment = Payment.Create(
            command.PaymentId,
            command.OrderId,
            command.Amount,
            command.Currency);
        
        // Save to write database
        await _repository.AddAsync(payment);
        
        // Process with gateway
        var result = await _gateway.ProcessPaymentAsync(new PaymentRequest
        {
            Amount = command.Amount,
            Currency = command.Currency,
            PaymentMethodId = command.PaymentMethodId
        });
        
        // Update state
        if (result.IsSuccess)
        {
            payment.MarkAsCompleted(result.TransactionId);
        }
        else
        {
            payment.MarkAsFailed(result.ErrorMessage);
        }
        
        await _repository.UpdateAsync(payment);
        
        // Publish events for read model sync
        await _eventPublisher.PublishAsync(new PaymentProcessedEvent
        {
            PaymentId = payment.Id,
            OrderId = payment.OrderId,
            Amount = payment.Amount,
            Status = payment.Status,
            Timestamp = DateTime.UtcNow
        });
    }
}

public class RefundPaymentCommandHandler : ICommandHandler<RefundPaymentCommand>
{
    private readonly IPaymentRepository _repository;
    private readonly IPaymentGateway _gateway;
    private readonly IEventPublisher _eventPublisher;
    
    public async Task HandleAsync(RefundPaymentCommand command)
    {
        var payment = await _repository.GetByIdAsync(command.PaymentId);
        
        if (payment == null)
            throw new PaymentNotFoundException(command.PaymentId);
        
        if (!payment.CanBeRefunded())
            throw new InvalidOperationException("Payment cannot be refunded");
        
        // Process refund
        var result = await _gateway.RefundAsync(
            payment.ProviderTransactionId,
            command.Amount);
        
        if (result.IsSuccess)
        {
            payment.MarkAsRefunded(command.Amount, command.Reason);
            await _repository.UpdateAsync(payment);
            
            await _eventPublisher.PublishAsync(new PaymentRefundedEvent
            {
                PaymentId = payment.Id,
                RefundAmount = command.Amount,
                Reason = command.Reason,
                Timestamp = DateTime.UtcNow
            });
        }
    }
}
```

### Query Handlers

```csharp
public interface IQueryHandler<TQuery, TResult> where TQuery : IQuery<TResult>
{
    Task<TResult> HandleAsync(TQuery query);
}

public class GetPaymentQueryHandler : IQueryHandler<GetPaymentQuery, PaymentDto>
{
    private readonly IPaymentReadRepository _readRepository;
    private readonly IMemoryCache _cache;
    
    public async Task<PaymentDto> HandleAsync(GetPaymentQuery query)
    {
        // Check cache first
        var cacheKey = $"payment:{query.PaymentId}";
        if (_cache.TryGetValue(cacheKey, out PaymentDto cached))
        {
            return cached;
        }
        
        // Query read model (optimized for reads)
        var payment = await _readRepository.GetPaymentAsync(query.PaymentId);
        
        if (payment != null)
        {
            _cache.Set(cacheKey, payment, TimeSpan.FromMinutes(5));
        }
        
        return payment;
    }
}

public class GetPaymentHistoryQueryHandler : IQueryHandler<GetPaymentHistoryQuery, List<PaymentDto>>
{
    private readonly IPaymentReadRepository _readRepository;
    
    public async Task<List<PaymentDto>> HandleAsync(GetPaymentHistoryQuery query)
    {
        // Complex query optimized for reading
        return await _readRepository.GetPaymentHistoryAsync(
            query.CustomerId,
            query.StartDate,
            query.EndDate);
    }
}
```

### Read Model (Denormalized)

```csharp
// Write Model (Normalized)
public class Payment
{
    public Guid Id { get; private set; }
    public string OrderId { get; private set; }
    public decimal Amount { get; private set; }
    public string Currency { get; private set; }
    public PaymentStatus Status { get; private set; }
    public DateTime CreatedAt { get; private set; }
    // ... domain logic
}

// Read Model (Denormalized for queries)
public class PaymentDto
{
    public Guid PaymentId { get; set; }
    public string OrderId { get; set; }
    public decimal Amount { get; set; }
    public string Currency { get; set; }
    public string Status { get; set; }
    public DateTime CreatedAt { get; set; }
    
    // Denormalized customer data
    public string CustomerName { get; set; }
    public string CustomerEmail { get; set; }
    
    // Denormalized order data
    public string OrderDescription { get; set; }
    public List<OrderItemDto> OrderItems { get; set; }
    
    // Computed fields
    public decimal RefundedAmount { get; set; }
    public decimal NetAmount { get; set; }
    public string DisplayStatus { get; set; }
}

public interface IPaymentReadRepository
{
    Task<PaymentDto> GetPaymentAsync(Guid paymentId);
    Task<List<PaymentDto>> GetPaymentsByOrderAsync(string orderId);
    Task<List<PaymentDto>> GetPaymentHistoryAsync(string customerId, DateTime start, DateTime end);
    Task UpdateReadModelAsync(PaymentProcessedEvent @event);
}

public class PaymentReadRepository : IPaymentReadRepository
{
    private readonly ApplicationDbContext _context;
    
    // Optimized read queries
    public async Task<PaymentDto> GetPaymentAsync(Guid paymentId)
    {
        return await _context.PaymentReadModels
            .Where(p => p.PaymentId == paymentId)
            .Select(p => new PaymentDto
            {
                PaymentId = p.PaymentId,
                OrderId = p.OrderId,
                Amount = p.Amount,
                Currency = p.Currency,
                Status = p.Status,
                CustomerName = p.CustomerName,
                CustomerEmail = p.CustomerEmail,
                DisplayStatus = FormatStatus(p.Status)
            })
            .FirstOrDefaultAsync();
    }
    
    // Update read model when events occur
    public async Task UpdateReadModelAsync(PaymentProcessedEvent @event)
    {
        var readModel = await _context.PaymentReadModels
            .FirstOrDefaultAsync(p => p.PaymentId == @event.PaymentId);
        
        if (readModel == null)
        {
            readModel = new PaymentReadModel
            {
                PaymentId = @event.PaymentId,
                OrderId = @event.OrderId,
                Amount = @event.Amount,
                Currency = @event.Currency,
                CreatedAt = @event.Timestamp
            };
            _context.PaymentReadModels.Add(readModel);
        }
        
        readModel.Status = @event.Status;
        readModel.UpdatedAt = @event.Timestamp;
        
        await _context.SaveChangesAsync();
    }
}
```

### Separate Databases for Read/Write

```csharp
public class CqrsDbContextFactory
{
    private readonly string _writeConnectionString;
    private readonly string _readConnectionString;
    
    // Write database (transactional, normalized)
    public ApplicationDbContext CreateWriteContext()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlServer(_writeConnectionString)
            .Options;
        
        return new ApplicationDbContext(options);
    }
    
    // Read database (replicated, denormalized)
    public ApplicationDbContext CreateReadContext()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlServer(_readConnectionString)
            .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking)
            .Options;
        
        return new ApplicationDbContext(options);
    }
}
```

---

## Event Sourcing

### Concept

Instead of storing current state, store all events that led to that state.

**Traditional:**
```
Payment: { Id: 123, Status: "Completed", Amount: 100 }
```

**Event Sourcing:**
```
Events:
1. PaymentCreated { Id: 123, Amount: 100 }
2. PaymentProcessed { Id: 123 }
3. PaymentCompleted { Id: 123 }
```

### Why Use Event Sourcing?

- ✅ Complete audit trail (required for payments)
- ✅ Time travel (reconstruct state at any point)
- ✅ Event replay for debugging
- ✅ Analytics and reporting
- ✅ Compliance (immutable history)

### Event Store Implementation

```csharp
public interface IEventStore
{
    Task SaveEventsAsync(Guid aggregateId, IEnumerable<DomainEvent> events, int expectedVersion);
    Task<List<DomainEvent>> GetEventsAsync(Guid aggregateId);
    Task<List<DomainEvent>> GetEventsAsync(Guid aggregateId, int fromVersion);
}

public class SqlEventStore : IEventStore
{
    private readonly ApplicationDbContext _context;
    
    public async Task SaveEventsAsync(
        Guid aggregateId, 
        IEnumerable<DomainEvent> events, 
        int expectedVersion)
    {
        var currentVersion = await GetCurrentVersionAsync(aggregateId);
        
        // Optimistic concurrency check
        if (currentVersion != expectedVersion)
        {
            throw new ConcurrencyException(
                $"Expected version {expectedVersion} but found {currentVersion}");
        }
        
        var version = expectedVersion;
        foreach (var @event in events)
        {
            version++;
            
            var eventData = new EventData
            {
                EventId = Guid.NewGuid(),
                AggregateId = aggregateId,
                EventType = @event.GetType().Name,
                EventData = JsonSerializer.Serialize(@event),
                Version = version,
                Timestamp = DateTime.UtcNow
            };
            
            _context.Events.Add(eventData);
        }
        
        await _context.SaveChangesAsync();
    }
    
    public async Task<List<DomainEvent>> GetEventsAsync(Guid aggregateId)
    {
        var eventData = await _context.Events
            .Where(e => e.AggregateId == aggregateId)
            .OrderBy(e => e.Version)
            .ToListAsync();
        
        return eventData.Select(DeserializeEvent).ToList();
    }
    
    private DomainEvent DeserializeEvent(EventData eventData)
    {
        var type = Type.GetType(eventData.EventType);
        return (DomainEvent)JsonSerializer.Deserialize(eventData.EventData, type);
    }
}

public class EventData
{
    public Guid EventId { get; set; }
    public Guid AggregateId { get; set; }
    public string EventType { get; set; }
    public string EventData { get; set; }
    public int Version { get; set; }
    public DateTime Timestamp { get; set; }
}
```

### Domain Events

```csharp
public abstract class DomainEvent
{
    public Guid EventId { get; set; } = Guid.NewGuid();
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
}

public class PaymentCreatedEvent : DomainEvent
{
    public Guid PaymentId { get; set; }
    public string OrderId { get; set; }
    public decimal Amount { get; set; }
    public string Currency { get; set; }
}

public class PaymentProcessingStartedEvent : DomainEvent
{
    public Guid PaymentId { get; set; }
    public string PaymentMethodId { get; set; }
}

public class PaymentCompletedEvent : DomainEvent
{
    public Guid PaymentId { get; set; }
    public string TransactionId { get; set; }
    public DateTime CompletedAt { get; set; }
}

public class PaymentFailedEvent : DomainEvent
{
    public Guid PaymentId { get; set; }
    public string ErrorMessage { get; set; }
    public string ErrorCode { get; set; }
}

public class PaymentRefundedEvent : DomainEvent
{
    public Guid PaymentId { get; set; }
    public decimal RefundAmount { get; set; }
    public string Reason { get; set; }
}
```

### Event-Sourced Aggregate

```csharp
public abstract class AggregateRoot
{
    private readonly List<DomainEvent> _uncommittedEvents = new();
    
    public Guid Id { get; protected set; }
    public int Version { get; protected set; } = -1;
    
    public IReadOnlyList<DomainEvent> GetUncommittedEvents() => _uncommittedEvents;
    
    public void MarkEventsAsCommitted() => _uncommittedEvents.Clear();
    
    protected void ApplyEvent(DomainEvent @event)
    {
        ApplyEventToAggregate(@event);
        _uncommittedEvents.Add(@event);
    }
    
    public void LoadFromHistory(IEnumerable<DomainEvent> events)
    {
        foreach (var @event in events)
        {
            ApplyEventToAggregate(@event);
            Version++;
        }
    }
    
    protected abstract void ApplyEventToAggregate(DomainEvent @event);
}

public class Payment : AggregateRoot
{
    // Current state (rebuilt from events)
    public string OrderId { get; private set; }
    public decimal Amount { get; private set; }
    public string Currency { get; private set; }
    public PaymentStatus Status { get; private set; }
    public string TransactionId { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? CompletedAt { get; private set; }
    
    private Payment() { } // For reconstruction
    
    // Factory method (creates event)
    public static Payment Create(Guid id, string orderId, decimal amount, string currency)
    {
        var payment = new Payment();
        payment.ApplyEvent(new PaymentCreatedEvent
        {
            PaymentId = id,
            OrderId = orderId,
            Amount = amount,
            Currency = currency
        });
        return payment;
    }
    
    // Business logic (creates events)
    public void StartProcessing(string paymentMethodId)
    {
        if (Status != PaymentStatus.Pending)
            throw new InvalidOperationException($"Cannot process payment in {Status} status");
        
        ApplyEvent(new PaymentProcessingStartedEvent
        {
            PaymentId = Id,
            PaymentMethodId = paymentMethodId
        });
    }
    
    public void MarkAsCompleted(string transactionId)
    {
        if (Status != PaymentStatus.Processing)
            throw new InvalidOperationException($"Cannot complete payment in {Status} status");
        
        ApplyEvent(new PaymentCompletedEvent
        {
            PaymentId = Id,
            TransactionId = transactionId,
            CompletedAt = DateTime.UtcNow
        });
    }
    
    public void MarkAsFailed(string errorMessage, string errorCode)
    {
        ApplyEvent(new PaymentFailedEvent
        {
            PaymentId = Id,
            ErrorMessage = errorMessage,
            ErrorCode = errorCode
        });
    }
    
    public void Refund(decimal amount, string reason)
    {
        if (Status != PaymentStatus.Completed)
            throw new InvalidOperationException("Can only refund completed payments");
        
        if (amount > Amount)
            throw new InvalidOperationException("Refund amount exceeds payment amount");
        
        ApplyEvent(new PaymentRefundedEvent
        {
            PaymentId = Id,
            RefundAmount = amount,
            Reason = reason
        });
    }
    
    // Apply events to rebuild state
    protected override void ApplyEventToAggregate(DomainEvent @event)
    {
        switch (@event)
        {
            case PaymentCreatedEvent e:
                Id = e.PaymentId;
                OrderId = e.OrderId;
                Amount = e.Amount;
                Currency = e.Currency;
                Status = PaymentStatus.Pending;
                CreatedAt = e.Timestamp;
                break;
                
            case PaymentProcessingStartedEvent e:
                Status = PaymentStatus.Processing;
                break;
                
            case PaymentCompletedEvent e:
                Status = PaymentStatus.Completed;
                TransactionId = e.TransactionId;
                CompletedAt = e.CompletedAt;
                break;
                
            case PaymentFailedEvent e:
                Status = PaymentStatus.Failed;
                break;
                
            case PaymentRefundedEvent e:
                Status = PaymentStatus.Refunded;
                break;
        }
    }
}
```

### Repository for Event-Sourced Aggregates

```csharp
public class EventSourcedPaymentRepository
{
    private readonly IEventStore _eventStore;
    
    public async Task SaveAsync(Payment payment)
    {
        var events = payment.GetUncommittedEvents();
        await _eventStore.SaveEventsAsync(payment.Id, events, payment.Version);
        payment.MarkEventsAsCommitted();
    }
    
    public async Task<Payment> GetByIdAsync(Guid id)
    {
        var events = await _eventStore.GetEventsAsync(id);
        
        if (!events.Any())
            return null;
        
        var payment = new Payment();
        payment.LoadFromHistory(events);
        return payment;
    }
}
```

---

## CQRS + Event Sourcing

### Combined Architecture

```
Command → Command Handler → Aggregate (Event Sourced)
                              ↓
                         Event Store
                              ↓
                         Event Bus
                              ↓
                    ┌─────────┴─────────┐
                    ▼                   ▼
            Read Model Projector    Other Services
                    ↓
            Read Database (Queries)
```

### Complete Implementation

```csharp
public class PaymentCommandHandler : ICommandHandler<ProcessPaymentCommand>
{
    private readonly EventSourcedPaymentRepository _repository;
    private readonly IPaymentGateway _gateway;
    private readonly IEventBus _eventBus;
    
    public async Task HandleAsync(ProcessPaymentCommand command)
    {
        // Get or create aggregate
        var payment = await _repository.GetByIdAsync(command.PaymentId) 
            ?? Payment.Create(
                command.PaymentId,
                command.OrderId,
                command.Amount,
                command.Currency);
        
        // Business logic (generates events)
        payment.StartProcessing(command.PaymentMethodId);
        
        // Process with gateway
        var result = await _gateway.ProcessPaymentAsync(new PaymentRequest
        {
            Amount = payment.Amount,
            Currency = payment.Currency,
            PaymentMethodId = command.PaymentMethodId
        });
        
        if (result.IsSuccess)
        {
            payment.MarkAsCompleted(result.TransactionId);
        }
        else
        {
            payment.MarkAsFailed(result.ErrorMessage, result.ErrorCode);
        }
        
        // Save events
        await _repository.SaveAsync(payment);
        
        // Publish events to bus
        foreach (var @event in payment.GetUncommittedEvents())
        {
            await _eventBus.PublishAsync(@event);
        }
    }
}

// Projector to update read models
public class PaymentReadModelProjector
{
    private readonly IPaymentReadRepository _readRepository;
    
    [EventHandler]
    public async Task Handle(PaymentCreatedEvent @event)
    {
        await _readRepository.InsertAsync(new PaymentReadModel
        {
            PaymentId = @event.PaymentId,
            OrderId = @event.OrderId,
            Amount = @event.Amount,
            Currency = @event.Currency,
            Status = "Pending",
            CreatedAt = @event.Timestamp
        });
    }
    
    [EventHandler]
    public async Task Handle(PaymentCompletedEvent @event)
    {
        await _readRepository.UpdateStatusAsync(
            @event.PaymentId,
            "Completed",
            @event.TransactionId);
    }
    
    [EventHandler]
    public async Task Handle(PaymentRefundedEvent @event)
    {
        await _readRepository.UpdateStatusAsync(
            @event.PaymentId,
            "Refunded");
        
        await _readRepository.UpdateRefundAmountAsync(
            @event.PaymentId,
            @event.RefundAmount);
    }
}
```

---

## Outbox Pattern

### Problem

When saving to database and publishing events, we need both to succeed or both to fail (atomicity).

**Without Outbox:**
```
1. Save to database ✅
2. Publish event ❌ (network failure)
Result: Inconsistent state!
```

### Solution: Outbox Pattern

```csharp
public class OutboxMessage
{
    public Guid Id { get; set; }
    public string EventType { get; set; }
    public string Payload { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
    public int RetryCount { get; set; }
}

public class PaymentCommandHandler : ICommandHandler<ProcessPaymentCommand>
{
    private readonly ApplicationDbContext _context;
    
    public async Task HandleAsync(ProcessPaymentCommand command)
    {
        using var transaction = await _context.Database.BeginTransactionAsync();
        
        try
        {
            // 1. Save business data
            var payment = new Payment { /* ... */ };
            _context.Payments.Add(payment);
            
            // 2. Save event to outbox (same transaction)
            var outboxMessage = new OutboxMessage
            {
                Id = Guid.NewGuid(),
                EventType = nameof(PaymentCreatedEvent),
                Payload = JsonSerializer.Serialize(new PaymentCreatedEvent
                {
                    PaymentId = payment.Id,
                    OrderId = payment.OrderId,
                    Amount = payment.Amount
                }),
                CreatedAt = DateTime.UtcNow
            };
            
            _context.OutboxMessages.Add(outboxMessage);
            
            // 3. Commit both together (atomic)
            await _context.SaveChangesAsync();
            await transaction.CommitAsync();
            
            // Now both are guaranteed to be saved
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}

// Background service to process outbox
public class OutboxProcessor : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly IEventBus _eventBus;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await ProcessOutboxMessagesAsync();
                await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
            }
            catch (Exception ex)
            {
                // Log error
            }
        }
    }
    
    private async Task ProcessOutboxMessagesAsync()
    {
        using var scope = _serviceProvider.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        
        // Get unprocessed messages
        var messages = await context.OutboxMessages
            .Where(m => m.ProcessedAt == null && m.RetryCount < 5)
            .OrderBy(m => m.CreatedAt)
            .Take(100)
            .ToListAsync();
        
        foreach (var message in messages)
        {
            try
            {
                // Deserialize and publish event
                var eventType = Type.GetType(message.EventType);
                var @event = JsonSerializer.Deserialize(message.Payload, eventType);
                
                await _eventBus.PublishAsync(@event);
                
                // Mark as processed
                message.ProcessedAt = DateTime.UtcNow;
                await context.SaveChangesAsync();
            }
            catch (Exception ex)
            {
                // Increment retry count
                message.RetryCount++;
                await context.SaveChangesAsync();
            }
        }
    }
}
```

---

## Inbox Pattern

### Problem

Prevent duplicate processing of messages/events (idempotency).

### Solution: Inbox Pattern

```csharp
public class InboxMessage
{
    public Guid MessageId { get; set; }
    public string MessageType { get; set; }
    public string Payload { get; set; }
    public DateTime ReceivedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
    public string Status { get; set; } // Pending, Processing, Processed, Failed
}

public class WebhookHandler
{
    private readonly ApplicationDbContext _context;
    
    public async Task HandleWebhookAsync(string webhookId, string payload)
    {
        // Check if already processed
        var existing = await _context.InboxMessages
            .FirstOrDefaultAsync(m => m.MessageId == Guid.Parse(webhookId));
        
        if (existing != null && existing.ProcessedAt != null)
        {
            // Already processed, return success
            return;
        }
        
        using var transaction = await _context.Database.BeginTransactionAsync();
        
        try
        {
            // Store in inbox
            if (existing == null)
            {
                existing = new InboxMessage
                {
                    MessageId = Guid.Parse(webhookId),
                    MessageType = "StripeWebhook",
                    Payload = payload,
                    ReceivedAt = DateTime.UtcNow,
                    Status = "Processing"
                };
                _context.InboxMessages.Add(existing);
                await _context.SaveChangesAsync();
            }
            
            // Process the webhook
            await ProcessWebhookAsync(payload);
            
            // Mark as processed
            existing.ProcessedAt = DateTime.UtcNow;
            existing.Status = "Processed";
            
            await _context.SaveChangesAsync();
            await transaction.CommitAsync();
        }
        catch
        {
            existing.Status = "Failed";
            await _context.SaveChangesAsync();
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

---

## Saga Pattern

For comprehensive Saga pattern implementations including orchestration, choreography, and the `SagaOrchestrator` class, see [Design Patterns — Saga Pattern](07-DESIGN-PATTERNS.md#saga-pattern).

The Saga pattern pairs well with CQRS and Event Sourcing because sagas react to domain events (produced by Event Sourcing) and issue commands (handled through CQRS command handlers). Use orchestration when you need central coordination and choreography when services should remain fully decoupled.

---

## Backend for Frontend (BFF)

### Concept

Create separate backends optimized for different frontends (web, mobile, admin).

```csharp
// Web BFF
public class WebPaymentController : ControllerBase
{
    [HttpGet("payments/{id}")]
    public async Task<PaymentWebViewModel> GetPayment(Guid id)
    {
        var payment = await _queryHandler.HandleAsync(new GetPaymentQuery { PaymentId = id });
        
        // Transform for web (detailed)
        return new PaymentWebViewModel
        {
            PaymentId = payment.PaymentId,
            OrderId = payment.OrderId,
            Amount = payment.Amount,
            Currency = payment.Currency,
            Status = payment.Status,
            CustomerName = payment.CustomerName,
            CustomerEmail = payment.CustomerEmail,
            OrderItems = payment.OrderItems,
            Timeline = payment.Timeline,
            RefundHistory = payment.RefundHistory
        };
    }
}

// Mobile BFF
public class MobilePaymentController : ControllerBase
{
    [HttpGet("payments/{id}")]
    public async Task<PaymentMobileViewModel> GetPayment(Guid id)
    {
        var payment = await _queryHandler.HandleAsync(new GetPaymentQuery { PaymentId = id });
        
        // Transform for mobile (minimal)
        return new PaymentMobileViewModel
        {
            PaymentId = payment.PaymentId,
            Amount = $"{payment.Amount:C}",
            Status = payment.Status,
            StatusIcon = GetStatusIcon(payment.Status),
            CanRefund = payment.Status == "Completed"
        };
    }
}
```

---

## Summary

| Pattern | Use Case | Complexity | Benefits |
|---------|----------|------------|----------|
| **CQRS** | Different read/write needs | Medium | Scalability, optimization |
| **Event Sourcing** | Audit trail required | High | Complete history, compliance |
| **CQRS + ES** | Complex domains | Very High | Best of both |
| **Outbox** | Reliable event publishing | Medium | Consistency |
| **Inbox** | Idempotent message processing | Low | Duplicate prevention |
| **Saga (Orchestration)** | Centralized control | Medium | Easy to reason about |
| **Saga (Choreography)** | Decentralized microservices | High | Loose coupling |
| **BFF** | Multiple clients | Low | Optimized per client |

## Next Steps

- [Design Patterns](07-DESIGN-PATTERNS.md)
- [Testing Guide](14-E2E-INTEGRATION-TESTING.md)
