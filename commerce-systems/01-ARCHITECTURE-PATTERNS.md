# Architecture Patterns for Payment Integration

## Overview

This document covers the architectural patterns and design principles for building robust payment systems in .NET applications.

## Table of Contents

1. [Layered Architecture](#layered-architecture)
2. [Service Layer Pattern](#service-layer-pattern)
3. [Adapter Pattern](#adapter-pattern)
4. [Strategy Pattern](#strategy-pattern)
5. [Repository Pattern](#repository-pattern)
6. [Event-Driven Architecture](#event-driven-architecture)

---

## Layered Architecture

### Concept

Separate payment logic into distinct layers to ensure maintainability and testability.

```
┌─────────────────────────────────────┐
│     Presentation Layer              │
│  (Controllers, APIs, UI)            │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│     Application Layer               │
│  (Use Cases, Commands, Handlers)    │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│     Domain Layer                    │
│  (Business Logic, Entities)         │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│     Infrastructure Layer            │
│  (Payment Gateways, Database)       │
└─────────────────────────────────────┘
```

### Implementation

**Domain Layer - Payment Domain Models:**

```csharp
namespace PaymentSystem.Domain.Entities
{
    public class Payment
    {
        public Guid Id { get; private set; }
        public string OrderId { get; private set; }
        public decimal Amount { get; private set; }
        public string Currency { get; private set; }
        public PaymentStatus Status { get; private set; }
        public string ProviderId { get; private set; }
        public string ProviderTransactionId { get; private set; }
        public DateTime CreatedAt { get; private set; }
        public DateTime? CompletedAt { get; private set; }
        
        private Payment() { } // EF Core
        
        public static Payment Create(string orderId, decimal amount, string currency)
        {
            if (amount <= 0)
                throw new ArgumentException("Amount must be positive", nameof(amount));
                
            return new Payment
            {
                Id = Guid.NewGuid(),
                OrderId = orderId,
                Amount = amount,
                Currency = currency.ToUpperInvariant(),
                Status = PaymentStatus.Pending,
                CreatedAt = DateTime.UtcNow
            };
        }
        
        public void MarkAsProcessing(string providerTransactionId)
        {
            if (Status != PaymentStatus.Pending)
                throw new InvalidOperationException($"Cannot process payment in {Status} status");
                
            Status = PaymentStatus.Processing;
            ProviderTransactionId = providerTransactionId;
        }
        
        public void MarkAsCompleted()
        {
            if (Status != PaymentStatus.Processing)
                throw new InvalidOperationException($"Cannot complete payment in {Status} status");
                
            Status = PaymentStatus.Completed;
            CompletedAt = DateTime.UtcNow;
        }
        
        public void MarkAsFailed(string reason)
        {
            Status = PaymentStatus.Failed;
            // Store reason in a separate field
        }
    }
    
    public enum PaymentStatus
    {
        Pending,
        Processing,
        Completed,
        Failed,
        Refunded
    }
}
```

**Application Layer - Use Cases:**

```csharp
namespace PaymentSystem.Application.UseCases
{
    public interface IProcessPaymentUseCase
    {
        Task<PaymentResult> ExecuteAsync(ProcessPaymentCommand command);
    }
    
    public class ProcessPaymentCommand
    {
        public string OrderId { get; set; }
        public decimal Amount { get; set; }
        public string Currency { get; set; }
        public string PaymentMethodId { get; set; }
        public string CustomerEmail { get; set; }
    }
    
    public class ProcessPaymentUseCase : IProcessPaymentUseCase
    {
        private readonly IPaymentGateway _paymentGateway;
        private readonly IPaymentRepository _paymentRepository;
        private readonly ILogger<ProcessPaymentUseCase> _logger;
        
        public ProcessPaymentUseCase(
            IPaymentGateway paymentGateway,
            IPaymentRepository paymentRepository,
            ILogger<ProcessPaymentUseCase> logger)
        {
            _paymentGateway = paymentGateway;
            _paymentRepository = paymentRepository;
            _logger = logger;
        }
        
        public async Task<PaymentResult> ExecuteAsync(ProcessPaymentCommand command)
        {
            // Create domain entity
            var payment = Payment.Create(
                command.OrderId, 
                command.Amount, 
                command.Currency
            );
            
            // Save initial state
            await _paymentRepository.AddAsync(payment);
            
            try
            {
                // Process through gateway
                var gatewayResult = await _paymentGateway.ProcessPaymentAsync(
                    new PaymentRequest
                    {
                        Amount = payment.Amount,
                        Currency = payment.Currency,
                        PaymentMethodId = command.PaymentMethodId,
                        Metadata = new Dictionary<string, string>
                        {
                            { "order_id", payment.OrderId },
                            { "payment_id", payment.Id.ToString() }
                        }
                    });
                
                if (gatewayResult.IsSuccess)
                {
                    payment.MarkAsProcessing(gatewayResult.TransactionId);
                    await _paymentRepository.UpdateAsync(payment);
                    
                    return PaymentResult.Success(payment.Id, gatewayResult.TransactionId);
                }
                else
                {
                    payment.MarkAsFailed(gatewayResult.ErrorMessage);
                    await _paymentRepository.UpdateAsync(payment);
                    
                    return PaymentResult.Failure(gatewayResult.ErrorMessage);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Payment processing failed for order {OrderId}", command.OrderId);
                payment.MarkAsFailed(ex.Message);
                await _paymentRepository.UpdateAsync(payment);
                
                return PaymentResult.Failure("Payment processing failed");
            }
        }
    }
}
```

**Infrastructure Layer - Payment Gateway:**

```csharp
namespace PaymentSystem.Infrastructure.Gateways
{
    public interface IPaymentGateway
    {
        Task<GatewayResult> ProcessPaymentAsync(PaymentRequest request);
        Task<GatewayResult> RefundPaymentAsync(string transactionId, decimal? amount = null);
    }
    
    public class StripePaymentGateway : IPaymentGateway
    {
        private readonly ILogger<StripePaymentGateway> _logger;
        
        public StripePaymentGateway(ILogger<StripePaymentGateway> logger)
        {
            _logger = logger;
        }
        
        public async Task<GatewayResult> ProcessPaymentAsync(PaymentRequest request)
        {
            try
            {
                var options = new PaymentIntentCreateOptions
                {
                    Amount = (long)(request.Amount * 100),
                    Currency = request.Currency,
                    PaymentMethod = request.PaymentMethodId,
                    Confirm = true,
                    Metadata = request.Metadata
                };
                
                var service = new PaymentIntentService();
                var paymentIntent = await service.CreateAsync(options);
                
                return new GatewayResult
                {
                    IsSuccess = paymentIntent.Status == "succeeded",
                    TransactionId = paymentIntent.Id,
                    Status = paymentIntent.Status
                };
            }
            catch (StripeException ex)
            {
                _logger.LogError(ex, "Stripe payment failed");
                return new GatewayResult
                {
                    IsSuccess = false,
                    ErrorMessage = ex.StripeError.Message
                };
            }
        }
        
        public async Task<GatewayResult> RefundPaymentAsync(string transactionId, decimal? amount = null)
        {
            // Implementation
            throw new NotImplementedException();
        }
    }
}
```

---

## Adapter Pattern

### Concept

Create adapters to normalize different payment provider APIs into a common interface.

### Why Use It?

- Switch payment providers without changing business logic
- Support multiple payment providers simultaneously
- Isolate provider-specific code

### Implementation

```csharp
namespace PaymentSystem.Infrastructure.Adapters
{
    // Common interface
    public interface IPaymentProviderAdapter
    {
        string ProviderName { get; }
        Task<PaymentResult> CreatePaymentAsync(PaymentRequest request);
        Task<RefundResult> RefundAsync(string transactionId, decimal? amount);
        Task<PaymentDetails> GetPaymentDetailsAsync(string transactionId);
    }
    
    // Stripe Adapter
    public class StripeAdapter : IPaymentProviderAdapter
    {
        public string ProviderName => "Stripe";
        
        public async Task<PaymentResult> CreatePaymentAsync(PaymentRequest request)
        {
            var options = new PaymentIntentCreateOptions
            {
                Amount = (long)(request.Amount * 100), // Stripe uses cents
                Currency = request.Currency.ToLowerInvariant(),
                Description = request.Description,
                Metadata = request.Metadata
            };
            
            var service = new PaymentIntentService();
            var intent = await service.CreateAsync(options);
            
            return MapToPaymentResult(intent);
        }
        
        public async Task<RefundResult> RefundAsync(string transactionId, decimal? amount)
        {
            var options = new RefundCreateOptions
            {
                PaymentIntent = transactionId
            };
            
            if (amount.HasValue)
            {
                options.Amount = (long)(amount.Value * 100);
            }
            
            var service = new RefundService();
            var refund = await service.CreateAsync(options);
            
            return MapToRefundResult(refund);
        }
        
        public async Task<PaymentDetails> GetPaymentDetailsAsync(string transactionId)
        {
            var service = new PaymentIntentService();
            var intent = await service.GetAsync(transactionId);
            
            return MapToPaymentDetails(intent);
        }
        
        private PaymentResult MapToPaymentResult(PaymentIntent intent)
        {
            return new PaymentResult
            {
                TransactionId = intent.Id,
                Status = MapStatus(intent.Status),
                Amount = intent.Amount / 100m,
                Currency = intent.Currency.ToUpperInvariant(),
                CreatedAt = intent.Created
            };
        }
        
        private PaymentStatus MapStatus(string stripeStatus)
        {
            return stripeStatus switch
            {
                "succeeded" => PaymentStatus.Completed,
                "processing" => PaymentStatus.Processing,
                "requires_payment_method" => PaymentStatus.Pending,
                "canceled" => PaymentStatus.Canceled,
                _ => PaymentStatus.Failed
            };
        }
    }
    
    // PayPal Adapter
    public class PayPalAdapter : IPaymentProviderAdapter
    {
        public string ProviderName => "PayPal";
        
        public async Task<PaymentResult> CreatePaymentAsync(PaymentRequest request)
        {
            // PayPal-specific implementation
            // Different API, different data structures
            // But returns the same PaymentResult
            
            var paypalOrder = new OrderRequest
            {
                CheckoutPaymentIntent = "CAPTURE",
                PurchaseUnits = new List<PurchaseUnitRequest>
                {
                    new PurchaseUnitRequest
                    {
                        AmountWithBreakdown = new AmountWithBreakdown
                        {
                            CurrencyCode = request.Currency,
                            Value = request.Amount.ToString("F2")
                        }
                    }
                }
            };
            
            var orderService = new OrdersCreateRequest();
            orderService.RequestBody(paypalOrder);
            
            var response = await _payPalClient.Execute(orderService);
            
            return MapPayPalToPaymentResult(response.Result<Order>());
        }
        
        // Other methods...
    }
}
```

---

## Strategy Pattern

### Concept

Dynamically select payment provider based on business rules (currency, amount, customer location, etc.).

### Implementation

```csharp
namespace PaymentSystem.Application.Strategies
{
    public interface IPaymentProviderStrategy
    {
        IPaymentProviderAdapter SelectProvider(PaymentContext context);
    }
    
    public class PaymentContext
    {
        public decimal Amount { get; set; }
        public string Currency { get; set; }
        public string CountryCode { get; set; }
        public bool IsRecurring { get; set; }
        public string CustomerTier { get; set; }
    }
    
    public class DefaultPaymentProviderStrategy : IPaymentProviderStrategy
    {
        private readonly IEnumerable<IPaymentProviderAdapter> _adapters;
        private readonly ILogger<DefaultPaymentProviderStrategy> _logger;
        
        public DefaultPaymentProviderStrategy(
            IEnumerable<IPaymentProviderAdapter> adapters,
            ILogger<DefaultPaymentProviderStrategy> logger)
        {
            _adapters = adapters;
            _logger = logger;
        }
        
        public IPaymentProviderAdapter SelectProvider(PaymentContext context)
        {
            // Rule 1: Use PayPal for specific countries
            if (ShouldUsePayPal(context))
            {
                _logger.LogInformation("Selected PayPal for country {Country}", context.CountryCode);
                return _adapters.First(a => a.ProviderName == "PayPal");
            }
            
            // Rule 2: Use Stripe for subscriptions
            if (context.IsRecurring)
            {
                _logger.LogInformation("Selected Stripe for recurring payment");
                return _adapters.First(a => a.ProviderName == "Stripe");
            }
            
            // Rule 3: Use lower-cost provider for small amounts
            if (context.Amount < 10)
            {
                return SelectLowestFeeProvider(context);
            }
            
            // Default to Stripe
            return _adapters.First(a => a.ProviderName == "Stripe");
        }
        
        private bool ShouldUsePayPal(PaymentContext context)
        {
            var paypalPreferredCountries = new[] { "DE", "FR", "IT", "ES" };
            return paypalPreferredCountries.Contains(context.CountryCode);
        }
        
        private IPaymentProviderAdapter SelectLowestFeeProvider(PaymentContext context)
        {
            // Logic to compare fees
            return _adapters.First();
        }
    }
}
```

### Usage in Application Layer

```csharp
public class SmartPaymentService
{
    private readonly IPaymentProviderStrategy _strategy;
    private readonly IPaymentRepository _repository;
    
    public SmartPaymentService(
        IPaymentProviderStrategy strategy,
        IPaymentRepository repository)
    {
        _strategy = strategy;
        _repository = repository;
    }
    
    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        var context = new PaymentContext
        {
            Amount = request.Amount,
            Currency = request.Currency,
            CountryCode = request.CountryCode,
            IsRecurring = request.IsRecurring
        };
        
        // Strategy selects the best provider
        var adapter = _strategy.SelectProvider(context);
        
        // Process with selected provider
        return await adapter.CreatePaymentAsync(request);
    }
}
```

---

## Event-Driven Architecture

### Concept

Use events to decouple payment processing from other business logic (order fulfillment, notifications, analytics).

### Implementation

**Domain Events:**

```csharp
namespace PaymentSystem.Domain.Events
{
    public abstract class DomainEvent
    {
        public DateTime OccurredOn { get; } = DateTime.UtcNow;
        public Guid EventId { get; } = Guid.NewGuid();
    }
    
    public class PaymentCompletedEvent : DomainEvent
    {
        public Guid PaymentId { get; }
        public string OrderId { get; }
        public decimal Amount { get; }
        public string Currency { get; }
        public string TransactionId { get; }
        
        public PaymentCompletedEvent(
            Guid paymentId, 
            string orderId, 
            decimal amount, 
            string currency,
            string transactionId)
        {
            PaymentId = paymentId;
            OrderId = orderId;
            Amount = amount;
            Currency = currency;
            TransactionId = transactionId;
        }
    }
    
    public class PaymentFailedEvent : DomainEvent
    {
        public Guid PaymentId { get; }
        public string OrderId { get; }
        public string Reason { get; }
        
        public PaymentFailedEvent(Guid paymentId, string orderId, string reason)
        {
            PaymentId = paymentId;
            OrderId = orderId;
            Reason = reason;
        }
    }
    
    public class RefundProcessedEvent : DomainEvent
    {
        public Guid PaymentId { get; }
        public decimal RefundAmount { get; }
        public string RefundId { get; }
        
        public RefundProcessedEvent(Guid paymentId, decimal refundAmount, string refundId)
        {
            PaymentId = paymentId;
            RefundAmount = refundAmount;
            RefundId = refundId;
        }
    }
}
```

**Event Publisher:**

```csharp
namespace PaymentSystem.Application.Events
{
    public interface IDomainEventPublisher
    {
        Task PublishAsync<TEvent>(TEvent domainEvent) where TEvent : DomainEvent;
    }
    
    public class DomainEventPublisher : IDomainEventPublisher
    {
        private readonly IMediator _mediator; // Using MediatR
        private readonly ILogger<DomainEventPublisher> _logger;
        
        public DomainEventPublisher(IMediator mediator, ILogger<DomainEventPublisher> logger)
        {
            _mediator = mediator;
            _logger = logger;
        }
        
        public async Task PublishAsync<TEvent>(TEvent domainEvent) where TEvent : DomainEvent
        {
            _logger.LogInformation(
                "Publishing event {EventType} with ID {EventId}", 
                typeof(TEvent).Name, 
                domainEvent.EventId);
                
            await _mediator.Publish(domainEvent);
        }
    }
}
```

**Event Handlers:**

```csharp
namespace PaymentSystem.Application.EventHandlers
{
    // Handler 1: Update order status
    public class PaymentCompletedOrderHandler : INotificationHandler<PaymentCompletedEvent>
    {
        private readonly IOrderService _orderService;
        private readonly ILogger<PaymentCompletedOrderHandler> _logger;
        
        public PaymentCompletedOrderHandler(
            IOrderService orderService,
            ILogger<PaymentCompletedOrderHandler> logger)
        {
            _orderService = orderService;
            _logger = logger;
        }
        
        public async Task Handle(PaymentCompletedEvent notification, CancellationToken cancellationToken)
        {
            _logger.LogInformation("Marking order {OrderId} as paid", notification.OrderId);
            await _orderService.MarkAsPaidAsync(notification.OrderId);
        }
    }
    
    // Handler 2: Send confirmation email
    public class PaymentCompletedEmailHandler : INotificationHandler<PaymentCompletedEvent>
    {
        private readonly IEmailService _emailService;
        private readonly ILogger<PaymentCompletedEmailHandler> _logger;
        
        public PaymentCompletedEmailHandler(
            IEmailService emailService,
            ILogger<PaymentCompletedEmailHandler> logger)
        {
            _emailService = emailService;
            _logger = logger;
        }
        
        public async Task Handle(PaymentCompletedEvent notification, CancellationToken cancellationToken)
        {
            _logger.LogInformation("Sending payment confirmation for order {OrderId}", notification.OrderId);
            
            await _emailService.SendPaymentConfirmationAsync(
                notification.OrderId,
                notification.Amount,
                notification.Currency,
                notification.TransactionId
            );
        }
    }
    
    // Handler 3: Track analytics
    public class PaymentCompletedAnalyticsHandler : INotificationHandler<PaymentCompletedEvent>
    {
        private readonly IAnalyticsService _analyticsService;
        
        public PaymentCompletedAnalyticsHandler(IAnalyticsService analyticsService)
        {
            _analyticsService = analyticsService;
        }
        
        public async Task Handle(PaymentCompletedEvent notification, CancellationToken cancellationToken)
        {
            await _analyticsService.TrackPaymentAsync(new PaymentMetrics
            {
                OrderId = notification.OrderId,
                Amount = notification.Amount,
                Currency = notification.Currency,
                Timestamp = notification.OccurredOn
            });
        }
    }
}
```

**Usage in Use Case:**

```csharp
public class ProcessPaymentUseCase : IProcessPaymentUseCase
{
    private readonly IPaymentGateway _gateway;
    private readonly IPaymentRepository _repository;
    private readonly IDomainEventPublisher _eventPublisher;
    
    public async Task<PaymentResult> ExecuteAsync(ProcessPaymentCommand command)
    {
        var payment = Payment.Create(command.OrderId, command.Amount, command.Currency);
        await _repository.AddAsync(payment);
        
        var result = await _gateway.ProcessPaymentAsync(/* ... */);
        
        if (result.IsSuccess)
        {
            payment.MarkAsCompleted();
            await _repository.UpdateAsync(payment);
            
            // Publish event - all handlers execute automatically
            await _eventPublisher.PublishAsync(new PaymentCompletedEvent(
                payment.Id,
                payment.OrderId,
                payment.Amount,
                payment.Currency,
                result.TransactionId
            ));
            
            return PaymentResult.Success(payment.Id, result.TransactionId);
        }
        else
        {
            payment.MarkAsFailed(result.ErrorMessage);
            await _repository.UpdateAsync(payment);
            
            // Publish failure event
            await _eventPublisher.PublishAsync(new PaymentFailedEvent(
                payment.Id,
                payment.OrderId,
                result.ErrorMessage
            ));
            
            return PaymentResult.Failure(result.ErrorMessage);
        }
    }
}
```

---

## Dependency Injection Configuration

**Program.cs / Startup.cs:**

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddPaymentServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // Configure Stripe
        StripeConfiguration.ApiKey = configuration["Stripe:SecretKey"];
        
        // Register adapters
        services.AddScoped<IPaymentProviderAdapter, StripeAdapter>();
        services.AddScoped<IPaymentProviderAdapter, PayPalAdapter>();
        
        // Register strategy
        services.AddScoped<IPaymentProviderStrategy, DefaultPaymentProviderStrategy>();
        
        // Register repositories
        services.AddScoped<IPaymentRepository, PaymentRepository>();
        
        // Register use cases
        services.AddScoped<IProcessPaymentUseCase, ProcessPaymentUseCase>();
        
        // Register event publisher
        services.AddScoped<IDomainEventPublisher, DomainEventPublisher>();
        
        // Add MediatR for event handling
        services.AddMediatR(cfg => 
            cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly()));
        
        return services;
    }
}
```

---

## Summary

| Pattern | Use When | Benefits |
|---------|----------|----------|
| Layered Architecture | Always | Clear separation, maintainability |
| Adapter | Multiple providers | Provider independence, flexibility |
| Strategy | Dynamic provider selection | Business-driven routing |
| Event-Driven | Decoupling concerns | Scalability, extensibility |
| Repository | Data access | Testability, abstraction |

## Next Steps

- [Webhook Patterns](04-WEBHOOK-PATTERNS.md)
- [Database Design](06-DATABASE-DESIGN.md)
- [Additional Patterns (Saga, Circuit Breaker, Retry)](07-DESIGN-PATTERNS.md)
