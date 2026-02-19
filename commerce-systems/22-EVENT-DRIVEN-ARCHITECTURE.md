# Event-Driven Architecture for Payment Systems

## Overview

Event-Driven Architecture (EDA) decouples payment processing from downstream actions (notifications, analytics, fulfillment) by publishing domain events. This guide covers event design, messaging infrastructure, eventual consistency, and the Saga pattern for distributed payment transactions.

## Table of Contents

1. [Why Event-Driven for Payments](#why-event-driven-for-payments)
2. [Domain Events](#domain-events)
3. [Event Bus Implementation](#event-bus-implementation)
4. [Outbox Pattern](#outbox-pattern)
5. [Event Handlers](#event-handlers)
6. [Saga Pattern for Distributed Transactions](#saga-pattern-for-distributed-transactions)
7. [Eventual Consistency](#eventual-consistency)
8. [Message Broker Options](#message-broker-options)
9. [Testing Event-Driven Systems](#testing-event-driven-systems)

---

## Why Event-Driven for Payments

| Synchronous (Without EDA) | Event-Driven |
|---------------------------|-------------|
| Payment → Email → Inventory → Analytics (sequential, slow) | Payment → publishes event → handlers process independently |
| One failure blocks entire flow | Partial failure doesn't block payment |
| Tight coupling between services | Loose coupling |
| Hard to add new post-payment actions | Add a new handler — no changes to payment service |
| Difficult to scale independently | Each handler scales independently |

---

## Domain Events

### Event Definitions

```csharp
// Base event
public abstract class DomainEvent
{
    public Guid EventId { get; init; } = Guid.NewGuid();
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
    public string EventType => GetType().Name;
    public int Version { get; init; } = 1;
}

// Payment events
public class PaymentInitiated : DomainEvent
{
    public Guid PaymentId { get; init; }
    public string OrderId { get; init; }
    public decimal Amount { get; init; }
    public string Currency { get; init; }
    public string Provider { get; init; }
    public string CustomerId { get; init; }
}

public class PaymentCompleted : DomainEvent
{
    public Guid PaymentId { get; init; }
    public string OrderId { get; init; }
    public string TransactionId { get; init; }
    public decimal Amount { get; init; }
    public string Currency { get; init; }
    public string Provider { get; init; }
    public DateTime CompletedAt { get; init; }
}

public class PaymentFailed : DomainEvent
{
    public Guid PaymentId { get; init; }
    public string OrderId { get; init; }
    public string ErrorCode { get; init; }
    public string ErrorMessage { get; init; }
    public bool IsRetryable { get; init; }
}

public class PaymentRefunded : DomainEvent
{
    public Guid PaymentId { get; init; }
    public string RefundId { get; init; }
    public decimal RefundAmount { get; init; }
    public string Reason { get; init; }
    public bool IsPartialRefund { get; init; }
}

public class PaymentDisputeOpened : DomainEvent
{
    public Guid PaymentId { get; init; }
    public string DisputeId { get; init; }
    public decimal DisputeAmount { get; init; }
    public string Reason { get; init; }
    public DateTime ResponseDeadline { get; init; }
}
```

### Event Envelope

```csharp
public class EventEnvelope<T> where T : DomainEvent
{
    public string MessageId { get; init; }
    public string CorrelationId { get; init; }
    public string CausationId { get; init; }
    public string Source { get; init; } = "PaymentService";
    public T Data { get; init; }
    public Dictionary<string, string> Metadata { get; init; } = new();

    public static EventEnvelope<T> Wrap(T domainEvent, string correlationId = null)
    {
        return new EventEnvelope<T>
        {
            MessageId = Guid.NewGuid().ToString(),
            CorrelationId = correlationId ?? Guid.NewGuid().ToString(),
            Data = domainEvent
        };
    }
}
```

---

## Event Bus Implementation

### In-Process with MediatR

```csharp
// NuGet: MediatR
public interface IDomainEventPublisher
{
    Task PublishAsync<T>(T domainEvent, CancellationToken ct = default) where T : DomainEvent;
}

public class MediatREventPublisher : IDomainEventPublisher
{
    private readonly IMediator _mediator;

    public async Task PublishAsync<T>(T domainEvent, CancellationToken ct = default) where T : DomainEvent
    {
        await _mediator.Publish(new DomainEventNotification<T>(domainEvent), ct);
    }
}

public class DomainEventNotification<T> : INotification where T : DomainEvent
{
    public T DomainEvent { get; }
    public DomainEventNotification(T domainEvent) => DomainEvent = domainEvent;
}

// Handler
public class SendPaymentReceiptHandler : INotificationHandler<DomainEventNotification<PaymentCompleted>>
{
    private readonly IEmailService _emailService;

    public async Task Handle(DomainEventNotification<PaymentCompleted> notification, CancellationToken ct)
    {
        var evt = notification.DomainEvent;
        await _emailService.SendReceiptAsync(evt.OrderId, evt.Amount, evt.Currency, evt.TransactionId);
    }
}
```

### Distributed with AWS SQS/SNS

```csharp
public class SnsEventPublisher : IDomainEventPublisher
{
    private readonly IAmazonSimpleNotificationService _sns;
    private readonly string _topicArn;

    public async Task PublishAsync<T>(T domainEvent, CancellationToken ct = default) where T : DomainEvent
    {
        var envelope = EventEnvelope<T>.Wrap(domainEvent);

        var message = new PublishRequest
        {
            TopicArn = _topicArn,
            Message = JsonSerializer.Serialize(envelope),
            MessageAttributes = new Dictionary<string, MessageAttributeValue>
            {
                ["EventType"] = new() { DataType = "String", StringValue = domainEvent.EventType },
                ["Version"] = new() { DataType = "Number", StringValue = domainEvent.Version.ToString() }
            }
        };

        await _sns.PublishAsync(message, ct);
    }
}

// SQS Consumer (background service)
public class SqsEventConsumer : BackgroundService
{
    private readonly IAmazonSQS _sqs;
    private readonly IServiceProvider _serviceProvider;
    private readonly string _queueUrl;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var response = await _sqs.ReceiveMessageAsync(new ReceiveMessageRequest
            {
                QueueUrl = _queueUrl,
                MaxNumberOfMessages = 10,
                WaitTimeSeconds = 20,
                MessageAttributeNames = new List<string> { "All" }
            }, stoppingToken);

            foreach (var message in response.Messages)
            {
                try
                {
                    await ProcessMessageAsync(message);
                    await _sqs.DeleteMessageAsync(_queueUrl, message.ReceiptHandle, stoppingToken);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Failed to process message {MessageId}", message.MessageId);
                    // Message will become visible again after visibility timeout → automatic retry
                }
            }
        }
    }

    private async Task ProcessMessageAsync(Message message)
    {
        var eventType = message.MessageAttributes["EventType"].StringValue;

        using var scope = _serviceProvider.CreateScope();

        switch (eventType)
        {
            case nameof(PaymentCompleted):
                var completed = JsonSerializer.Deserialize<EventEnvelope<PaymentCompleted>>(message.Body);
                await HandlePaymentCompleted(scope, completed.Data);
                break;

            case nameof(PaymentFailed):
                var failed = JsonSerializer.Deserialize<EventEnvelope<PaymentFailed>>(message.Body);
                await HandlePaymentFailed(scope, failed.Data);
                break;
        }
    }
}
```

---

## Outbox Pattern

Ensure events are published reliably even if the message broker is temporarily unavailable.

```csharp
// 1. Save event to outbox table in the same DB transaction as the payment
public class PaymentServiceWithOutbox
{
    private readonly ApplicationDbContext _context;

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        await using var transaction = await _context.Database.BeginTransactionAsync();

        try
        {
            // Process payment
            var payment = Payment.Create(request);
            await _context.Payments.AddAsync(payment);

            var result = await _gateway.ProcessPaymentAsync(request);
            payment.Complete(result.TransactionId);

            // Save event to outbox (same transaction!)
            var outboxMessage = new OutboxMessage
            {
                Id = Guid.NewGuid(),
                EventType = nameof(PaymentCompleted),
                Payload = JsonSerializer.Serialize(new PaymentCompleted
                {
                    PaymentId = payment.Id,
                    OrderId = request.OrderId,
                    TransactionId = result.TransactionId,
                    Amount = request.Amount,
                    Currency = request.Currency
                }),
                CreatedAt = DateTime.UtcNow,
                ProcessedAt = null
            };
            await _context.OutboxMessages.AddAsync(outboxMessage);

            await _context.SaveChangesAsync();
            await transaction.CommitAsync();

            return result;
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}

// 2. Background worker publishes outbox messages
public class OutboxProcessor : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var messages = await _context.OutboxMessages
                .Where(m => m.ProcessedAt == null)
                .OrderBy(m => m.CreatedAt)
                .Take(100)
                .ToListAsync(stoppingToken);

            foreach (var message in messages)
            {
                try
                {
                    await _eventPublisher.PublishRawAsync(message.EventType, message.Payload);
                    message.ProcessedAt = DateTime.UtcNow;
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Failed to publish outbox message {Id}", message.Id);
                    message.RetryCount++;
                }
            }

            await _context.SaveChangesAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}

// 3. Outbox table
public class OutboxMessage
{
    public Guid Id { get; set; }
    public string EventType { get; set; }
    public string Payload { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
    public int RetryCount { get; set; }
}
```

---

## Event Handlers

### Handler Registry

```csharp
// Multiple handlers can react to the same event independently
public class PaymentCompletedHandlers
{
    // Handler 1: Send receipt email
    public class SendReceipt : INotificationHandler<DomainEventNotification<PaymentCompleted>>
    {
        public async Task Handle(DomainEventNotification<PaymentCompleted> n, CancellationToken ct)
        {
            await _emailService.SendReceiptAsync(n.DomainEvent.OrderId, n.DomainEvent.Amount);
        }
    }

    // Handler 2: Update inventory
    public class UpdateInventory : INotificationHandler<DomainEventNotification<PaymentCompleted>>
    {
        public async Task Handle(DomainEventNotification<PaymentCompleted> n, CancellationToken ct)
        {
            await _inventoryService.ConfirmReservationAsync(n.DomainEvent.OrderId);
        }
    }

    // Handler 3: Update analytics
    public class RecordAnalytics : INotificationHandler<DomainEventNotification<PaymentCompleted>>
    {
        public async Task Handle(DomainEventNotification<PaymentCompleted> n, CancellationToken ct)
        {
            await _analyticsService.RecordRevenueAsync(n.DomainEvent.Amount, n.DomainEvent.Currency);
        }
    }

    // Handler 4: Trigger fulfillment
    public class StartFulfillment : INotificationHandler<DomainEventNotification<PaymentCompleted>>
    {
        public async Task Handle(DomainEventNotification<PaymentCompleted> n, CancellationToken ct)
        {
            await _fulfillmentService.StartShipmentAsync(n.DomainEvent.OrderId);
        }
    }
}
```

---

## Saga Pattern for Distributed Transactions

Coordinate multi-step processes with compensating actions on failure.

### Order Processing Saga

```csharp
public class OrderProcessingSaga
{
    private readonly IPaymentGateway _paymentGateway;
    private readonly IInventoryService _inventoryService;
    private readonly IShippingService _shippingService;
    private readonly INotificationService _notifications;
    private readonly ILogger<OrderProcessingSaga> _logger;

    public async Task<SagaResult> ExecuteAsync(OrderRequest request)
    {
        var compensations = new Stack<Func<Task>>();

        try
        {
            // Step 1: Reserve inventory
            _logger.LogInformation("Saga step 1: Reserving inventory for order {OrderId}", request.OrderId);
            var reservation = await _inventoryService.ReserveAsync(request.Items);
            compensations.Push(() => _inventoryService.ReleaseReservationAsync(reservation.Id));

            // Step 2: Process payment
            _logger.LogInformation("Saga step 2: Processing payment for order {OrderId}", request.OrderId);
            var payment = await _paymentGateway.ProcessPaymentAsync(new PaymentRequest
            {
                OrderId = request.OrderId,
                Amount = request.TotalAmount,
                Currency = request.Currency,
                PaymentMethodId = request.PaymentMethodId
            });

            if (!payment.IsSuccess)
            {
                throw new SagaStepFailedException("Payment", payment.Error?.Message);
            }
            compensations.Push(() => _paymentGateway.ProcessRefundAsync(new RefundRequest
            {
                PaymentIntentId = payment.TransactionId,
                Amount = request.TotalAmount,
                Reason = "Order saga rollback"
            }));

            // Step 3: Create shipment
            _logger.LogInformation("Saga step 3: Creating shipment for order {OrderId}", request.OrderId);
            var shipment = await _shippingService.CreateShipmentAsync(request.ShippingAddress, request.Items);
            compensations.Push(() => _shippingService.CancelShipmentAsync(shipment.Id));

            // Step 4: Send confirmation
            await _notifications.SendOrderConfirmationAsync(request.CustomerEmail, request.OrderId);

            _logger.LogInformation("Saga completed successfully for order {OrderId}", request.OrderId);

            return SagaResult.Success(payment.TransactionId, shipment.TrackingNumber);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Saga failed for order {OrderId}. Running compensations...", request.OrderId);

            // Execute compensations in reverse order
            while (compensations.Count > 0)
            {
                var compensation = compensations.Pop();
                try
                {
                    await compensation();
                }
                catch (Exception compEx)
                {
                    _logger.LogCritical(compEx,
                        "Compensation failed for order {OrderId}. Manual intervention required!",
                        request.OrderId);
                    // Alert on-call immediately
                }
            }

            return SagaResult.Failed(ex.Message);
        }
    }
}
```

### State Machine Saga (for Long-Running Processes)

```csharp
public class SubscriptionSaga
{
    public enum SagaState
    {
        Created,
        CustomerCreated,
        PaymentMethodAttached,
        SubscriptionStarted,
        WelcomeEmailSent,
        Completed,
        Failed,
        CompensatingCustomerCreation,
        CompensatingPaymentMethod
    }

    private SagaState _state = SagaState.Created;

    public async Task<SagaResult> ExecuteAsync(SubscriptionRequest request)
    {
        while (_state != SagaState.Completed && _state != SagaState.Failed)
        {
            switch (_state)
            {
                case SagaState.Created:
                    await CreateCustomerAsync(request);
                    break;
                case SagaState.CustomerCreated:
                    await AttachPaymentMethodAsync(request);
                    break;
                case SagaState.PaymentMethodAttached:
                    await StartSubscriptionAsync(request);
                    break;
                case SagaState.SubscriptionStarted:
                    await SendWelcomeEmailAsync(request);
                    break;
                case SagaState.WelcomeEmailSent:
                    _state = SagaState.Completed;
                    break;

                // Compensation states
                case SagaState.CompensatingPaymentMethod:
                    await DetachPaymentMethodAsync();
                    _state = SagaState.CompensatingCustomerCreation;
                    break;
                case SagaState.CompensatingCustomerCreation:
                    await DeleteCustomerAsync();
                    _state = SagaState.Failed;
                    break;
            }

            // Persist state for recovery
            await _sagaStore.SaveStateAsync(request.SagaId, _state);
        }

        return _state == SagaState.Completed ? SagaResult.Success() : SagaResult.Failed("Saga rolled back");
    }
}
```

---

## Eventual Consistency

### Handling Eventual Consistency in the UI

```csharp
[ApiController]
public class OrderController : ControllerBase
{
    // Optimistic UI pattern: return immediately, process async
    [HttpPost("orders")]
    public async Task<IActionResult> CreateOrder([FromBody] OrderRequest request)
    {
        var orderId = await _orderService.CreateOrderAsync(request);

        // Return 202 Accepted — order is being processed
        return Accepted(new
        {
            orderId,
            status = "processing",
            statusUrl = $"/api/orders/{orderId}/status"
        });
    }

    // Polling endpoint
    [HttpGet("orders/{orderId}/status")]
    public async Task<IActionResult> GetOrderStatus(Guid orderId)
    {
        var order = await _orderRepo.GetByIdAsync(orderId);

        return Ok(new
        {
            orderId = order.Id,
            status = order.Status.ToString(),
            paymentStatus = order.PaymentStatus?.ToString(),
            updatedAt = order.UpdatedAt
        });
    }
}
```

### Idempotent Event Handling

```csharp
public class IdempotentEventHandler<T> : INotificationHandler<DomainEventNotification<T>>
    where T : DomainEvent
{
    private readonly IProcessedEventStore _processedEvents;
    private readonly INotificationHandler<DomainEventNotification<T>> _inner;

    public async Task Handle(DomainEventNotification<T> notification, CancellationToken ct)
    {
        var eventId = notification.DomainEvent.EventId;

        if (await _processedEvents.HasBeenProcessedAsync(eventId))
        {
            _logger.LogInformation("Event {EventId} already processed, skipping", eventId);
            return;
        }

        await _inner.Handle(notification, ct);
        await _processedEvents.MarkAsProcessedAsync(eventId);
    }
}
```

---

## Message Broker Options

| Broker | Best For | Key Features |
|--------|----------|-------------|
| **AWS SQS/SNS** | AWS-native, simple pub/sub | Managed, auto-scaling, DLQ, FIFO queues |
| **Azure Service Bus** | Azure-native, enterprise | Sessions, transactions, topics |
| **RabbitMQ** | Self-hosted, flexible routing | Exchanges, routing keys, plugins |
| **Apache Kafka** | Event log, high throughput | Replay, partitions, compaction |
| **AWS EventBridge** | Event routing with rules | Schema registry, filtering |

### Decision Guide

```
Need event replay / audit log?
  → Kafka or EventBridge
  
Managed + simple?
  → SQS/SNS (AWS) or Service Bus (Azure)

Self-hosted + fine-grained routing?
  → RabbitMQ

Need < 10ms latency?
  → In-process MediatR + async outbox
```

---

## Testing Event-Driven Systems

```csharp
[TestClass]
public class PaymentSagaTests
{
    [TestMethod]
    public async Task Saga_PaymentFails_CompensatesInventoryReservation()
    {
        // Arrange
        var inventoryService = new Mock<IInventoryService>();
        inventoryService.Setup(i => i.ReserveAsync(It.IsAny<List<OrderItem>>()))
            .ReturnsAsync(new Reservation { Id = "res-1" });

        var paymentGateway = new Mock<IPaymentGateway>();
        paymentGateway.Setup(p => p.ProcessPaymentAsync(It.IsAny<PaymentRequest>()))
            .ReturnsAsync(PaymentResult.Failure(new PaymentError { Code = "declined" }));

        var saga = new OrderProcessingSaga(paymentGateway.Object, inventoryService.Object, ...);

        // Act
        var result = await saga.ExecuteAsync(new OrderRequest { OrderId = "order-1" });

        // Assert
        Assert.IsFalse(result.IsSuccess);
        inventoryService.Verify(i => i.ReleaseReservationAsync("res-1"), Times.Once);
    }

    [TestMethod]
    public async Task OutboxProcessor_PublishesUnprocessedMessages()
    {
        // Arrange
        var context = CreateInMemoryDbContext();
        var outboxMessage = new OutboxMessage
        {
            Id = Guid.NewGuid(),
            EventType = nameof(PaymentCompleted),
            Payload = "{ ... }",
            CreatedAt = DateTime.UtcNow,
            ProcessedAt = null
        };
        context.OutboxMessages.Add(outboxMessage);
        await context.SaveChangesAsync();

        var publisher = new Mock<IDomainEventPublisher>();
        var processor = new OutboxProcessor(context, publisher.Object);

        // Act
        await processor.ProcessBatchAsync();

        // Assert
        publisher.Verify(p => p.PublishRawAsync(nameof(PaymentCompleted), It.IsAny<string>()), Times.Once);
        Assert.IsNotNull(outboxMessage.ProcessedAt);
    }
}
```

---

## EDA Checklist

- [ ] Domain events defined for all payment lifecycle changes
- [ ] Event envelope includes correlationId, messageId, timestamp
- [ ] Outbox pattern for reliable event publishing
- [ ] Idempotent event handlers (deduplication by eventId)
- [ ] Dead letter queue for unprocessable events
- [ ] Saga pattern for multi-step transactions with compensations
- [ ] Event versioning strategy (backward compatibility)
- [ ] Monitoring on event processing lag and DLQ depth
- [ ] Integration tests covering saga compensation flows

## Next Steps

- [CQRS & Event Sourcing](11-CQRS-EVENT-SOURCING.md)
- [Microservices Patterns](23-MICROSERVICES-PATTERNS.md)
- [Architecture Patterns](01-ARCHITECTURE-PATTERNS.md)
