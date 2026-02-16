# Webhook Patterns for Payment Integration

## Overview

Webhooks are critical for payment systems. They provide asynchronous notifications about payment events, allowing your system to respond to changes in payment status without polling.

## Table of Contents

1. [Webhook Fundamentals](#webhook-fundamentals)
2. [Security Patterns](#security-patterns)
3. [Processing Patterns](#processing-patterns)
4. [Error Handling](#error-handling)
5. [Testing Webhooks](#testing-webhooks)

---

## Webhook Fundamentals

### What Are Webhooks?

Payment providers send HTTP POST requests to your server when events occur:
- Payment succeeded
- Payment failed
- Refund processed
- Subscription renewed
- Dispute created

### Why Webhooks Matter

```
Without Webhooks (Polling):
Your Server → Check status? → Payment Provider
                ↓ (repeat every few seconds)
            Inefficient, delayed

With Webhooks:
Payment Provider → Event occurred! → Your Server
                                      ↓
                                   Immediate action
```

### Key Principles

1. **Idempotency**: Process each webhook once, even if received multiple times
2. **Security**: Always verify webhook signatures
3. **Async Processing**: Return 200 quickly, process in background
4. **Retry Logic**: Handle provider retries gracefully

---

## Security Patterns

### Pattern 1: Signature Verification

**Why**: Prevents malicious actors from sending fake webhooks to your server.

**Stripe Implementation:**

```csharp
namespace PaymentSystem.Api.Controllers
{
    [ApiController]
    [Route("api/webhooks")]
    public class WebhookController : ControllerBase
    {
        private readonly IConfiguration _configuration;
        private readonly IWebhookProcessor _webhookProcessor;
        private readonly ILogger<WebhookController> _logger;
        
        public WebhookController(
            IConfiguration configuration,
            IWebhookProcessor webhookProcessor,
            ILogger<WebhookController> logger)
        {
            _configuration = configuration;
            _webhookProcessor = webhookProcessor;
            _logger = logger;
        }
        
        [HttpPost("stripe")]
        public async Task<IActionResult> HandleStripeWebhook()
        {
            var json = await new StreamReader(HttpContext.Request.Body).ReadToEndAsync();
            
            try
            {
                // Verify signature
                var stripeEvent = ConstructVerifiedEvent(json);
                
                // Process asynchronously
                await _webhookProcessor.ProcessAsync(stripeEvent);
                
                return Ok();
            }
            catch (StripeException ex)
            {
                _logger.LogError(ex, "Webhook signature verification failed");
                return BadRequest("Invalid signature");
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Webhook processing failed");
                // Still return 200 to prevent retries for permanent failures
                return Ok();
            }
        }
        
        private Event ConstructVerifiedEvent(string json)
        {
            var signatureHeader = Request.Headers["Stripe-Signature"].ToString();
            var webhookSecret = _configuration["Stripe:WebhookSecret"];
            
            // This will throw StripeException if signature is invalid
            return EventUtility.ConstructEvent(
                json,
                signatureHeader,
                webhookSecret,
                throwOnApiVersionMismatch: false
            );
        }
    }
}
```

**PayPal Implementation:**

```csharp
[HttpPost("paypal")]
public async Task<IActionResult> HandlePayPalWebhook()
{
    var json = await new StreamReader(HttpContext.Request.Body).ReadToEndAsync();
    var headers = Request.Headers;
    
    // PayPal verification
    var verificationRequest = new VerifyWebhookSignatureRequest()
        .WebhookId(_configuration["PayPal:WebhookId"])
        .WebhookEvent(JsonConvert.DeserializeObject<WebhookEvent>(json));
        
    // Add headers for verification
    verificationRequest.TransmissionId(headers["PAYPAL-TRANSMISSION-ID"]);
    verificationRequest.TransmissionTime(headers["PAYPAL-TRANSMISSION-TIME"]);
    verificationRequest.TransmissionSig(headers["PAYPAL-TRANSMISSION-SIG"]);
    verificationRequest.CertUrl(headers["PAYPAL-CERT-URL"]);
    verificationRequest.AuthAlgo(headers["PAYPAL-AUTH-ALGO"]);
    
    var verifyResponse = await _payPalClient.Execute(verificationRequest);
    
    if (verifyResponse.Result.VerificationStatus != "SUCCESS")
    {
        _logger.LogWarning("PayPal webhook verification failed");
        return BadRequest("Invalid signature");
    }
    
    await _webhookProcessor.ProcessPayPalWebhookAsync(json);
    return Ok();
}
```

### Pattern 2: IP Allowlisting

Add an additional security layer by checking the source IP.

```csharp
public class WebhookIpFilterAttribute : ActionFilterAttribute
{
    private static readonly HashSet<string> AllowedIpRanges = new()
    {
        // Stripe IPs (example - check Stripe documentation for current ranges)
        "3.18.12.0/23",
        "3.130.192.0/23",
        "13.52.14.0/23",
        // Add other provider IPs
    };
    
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        var remoteIp = context.HttpContext.Connection.RemoteIpAddress;
        
        if (!IsAllowedIp(remoteIp))
        {
            context.Result = new StatusCodeResult(403);
            return;
        }
        
        base.OnActionExecuting(context);
    }
    
    private bool IsAllowedIp(IPAddress remoteIp)
    {
        // Implementation to check if IP is in allowed ranges
        return true; // Simplified
    }
}

// Usage
[WebhookIpFilter]
[HttpPost("stripe")]
public async Task<IActionResult> HandleStripeWebhook()
{
    // ...
}
```

---

## Processing Patterns

### Pattern 1: Immediate Response with Background Processing

**Problem**: Webhook processing might take time, but providers expect quick response.

**Solution**: Return 200 immediately, process in background.

```csharp
public interface IWebhookProcessor
{
    Task ProcessAsync(Event stripeEvent);
}

public class WebhookProcessor : IWebhookProcessor
{
    private readonly IBackgroundJobClient _backgroundJobs;
    private readonly ILogger<WebhookProcessor> _logger;
    
    public WebhookProcessor(
        IBackgroundJobClient backgroundJobs,
        ILogger<WebhookProcessor> logger)
    {
        _backgroundJobs = backgroundJobs;
        _logger = logger;
    }
    
    public Task ProcessAsync(Event stripeEvent)
    {
        _logger.LogInformation(
            "Queuing webhook {EventId} of type {EventType}",
            stripeEvent.Id,
            stripeEvent.Type);
        
        // Queue for background processing (using Hangfire, for example)
        _backgroundJobs.Enqueue<IWebhookHandler>(handler => 
            handler.HandleAsync(stripeEvent.Id, stripeEvent.Type, stripeEvent.Data.RawObject));
        
        return Task.CompletedTask;
    }
}

public interface IWebhookHandler
{
    Task HandleAsync(string eventId, string eventType, object eventData);
}

public class WebhookHandler : IWebhookHandler
{
    private readonly IPaymentRepository _paymentRepository;
    private readonly IDomainEventPublisher _eventPublisher;
    private readonly ILogger<WebhookHandler> _logger;
    
    public WebhookHandler(
        IPaymentRepository paymentRepository,
        IDomainEventPublisher eventPublisher,
        ILogger<WebhookHandler> logger)
    {
        _paymentRepository = paymentRepository;
        _eventPublisher = eventPublisher;
        _logger = logger;
    }
    
    public async Task HandleAsync(string eventId, string eventType, object eventData)
    {
        _logger.LogInformation("Processing webhook {EventId} of type {EventType}", eventId, eventType);
        
        try
        {
            switch (eventType)
            {
                case Events.PaymentIntentSucceeded:
                    await HandlePaymentSucceededAsync(eventData as PaymentIntent);
                    break;
                    
                case Events.PaymentIntentPaymentFailed:
                    await HandlePaymentFailedAsync(eventData as PaymentIntent);
                    break;
                    
                case Events.ChargeRefunded:
                    await HandleRefundAsync(eventData as Charge);
                    break;
                    
                case Events.CustomerSubscriptionCreated:
                    await HandleSubscriptionCreatedAsync(eventData as Subscription);
                    break;
                    
                default:
                    _logger.LogInformation("Unhandled event type: {EventType}", eventType);
                    break;
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing webhook {EventId}", eventId);
            throw; // Let background job retry
        }
    }
    
    private async Task HandlePaymentSucceededAsync(PaymentIntent paymentIntent)
    {
        var paymentId = paymentIntent.Metadata.TryGetValue("payment_id", out var id) 
            ? Guid.Parse(id) 
            : Guid.Empty;
            
        if (paymentId == Guid.Empty)
        {
            _logger.LogWarning("Payment ID not found in metadata for {PaymentIntentId}", paymentIntent.Id);
            return;
        }
        
        var payment = await _paymentRepository.GetByIdAsync(paymentId);
        
        if (payment == null)
        {
            _logger.LogWarning("Payment {PaymentId} not found", paymentId);
            return;
        }
        
        // Update payment status
        payment.MarkAsCompleted();
        await _paymentRepository.UpdateAsync(payment);
        
        // Publish domain event
        await _eventPublisher.PublishAsync(new PaymentCompletedEvent(
            payment.Id,
            payment.OrderId,
            payment.Amount,
            payment.Currency,
            paymentIntent.Id
        ));
        
        _logger.LogInformation("Payment {PaymentId} marked as completed", paymentId);
    }
    
    private async Task HandlePaymentFailedAsync(PaymentIntent paymentIntent)
    {
        var paymentId = Guid.Parse(paymentIntent.Metadata["payment_id"]);
        var payment = await _paymentRepository.GetByIdAsync(paymentId);
        
        if (payment != null)
        {
            payment.MarkAsFailed(paymentIntent.LastPaymentError?.Message ?? "Unknown error");
            await _paymentRepository.UpdateAsync(payment);
            
            await _eventPublisher.PublishAsync(new PaymentFailedEvent(
                payment.Id,
                payment.OrderId,
                paymentIntent.LastPaymentError?.Message ?? "Payment failed"
            ));
        }
    }
    
    private async Task HandleRefundAsync(Charge charge)
    {
        // Implementation for refund handling
        _logger.LogInformation("Processing refund for charge {ChargeId}", charge.Id);
    }
    
    private async Task HandleSubscriptionCreatedAsync(Subscription subscription)
    {
        // Implementation for subscription handling
        _logger.LogInformation("Processing new subscription {SubscriptionId}", subscription.Id);
    }
}
```

### Pattern 2: Idempotent Processing

**Problem**: Providers may send the same webhook multiple times.

**Solution**: Track processed webhooks and skip duplicates.

```csharp
public interface IWebhookEventRepository
{
    Task<bool> HasBeenProcessedAsync(string eventId);
    Task MarkAsProcessedAsync(string eventId);
}

public class WebhookEventRepository : IWebhookEventRepository
{
    private readonly ApplicationDbContext _context;
    
    public WebhookEventRepository(ApplicationDbContext context)
    {
        _context = context;
    }
    
    public async Task<bool> HasBeenProcessedAsync(string eventId)
    {
        return await _context.ProcessedWebhookEvents
            .AnyAsync(e => e.EventId == eventId);
    }
    
    public async Task MarkAsProcessedAsync(string eventId)
    {
        _context.ProcessedWebhookEvents.Add(new ProcessedWebhookEvent
        {
            EventId = eventId,
            ProcessedAt = DateTime.UtcNow
        });
        
        await _context.SaveChangesAsync();
    }
}

// Entity
public class ProcessedWebhookEvent
{
    public int Id { get; set; }
    public string EventId { get; set; }
    public DateTime ProcessedAt { get; set; }
    
    // Optional: Add cleanup logic for old events
}

// Updated handler with idempotency
public class IdempotentWebhookHandler : IWebhookHandler
{
    private readonly IWebhookEventRepository _webhookEventRepository;
    private readonly IWebhookHandler _innerHandler;
    private readonly ILogger<IdempotentWebhookHandler> _logger;
    
    public IdempotentWebhookHandler(
        IWebhookEventRepository webhookEventRepository,
        IWebhookHandler innerHandler,
        ILogger<IdempotentWebhookHandler> logger)
    {
        _webhookEventRepository = webhookEventRepository;
        _innerHandler = innerHandler;
        _logger = logger;
    }
    
    public async Task HandleAsync(string eventId, string eventType, object eventData)
    {
        // Check if already processed
        if (await _webhookEventRepository.HasBeenProcessedAsync(eventId))
        {
            _logger.LogInformation("Webhook {EventId} already processed, skipping", eventId);
            return;
        }
        
        // Process the event
        await _innerHandler.HandleAsync(eventId, eventType, eventData);
        
        // Mark as processed
        await _webhookEventRepository.MarkAsProcessedAsync(eventId);
        
        _logger.LogInformation("Webhook {EventId} processed and marked as complete", eventId);
    }
}
```

### Pattern 3: Event Sourcing for Webhooks

Store all webhook events for audit and replay.

```csharp
public class WebhookEvent
{
    public int Id { get; set; }
    public string EventId { get; set; }
    public string Provider { get; set; }
    public string EventType { get; set; }
    public string RawPayload { get; set; }
    public DateTime ReceivedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
    public string ProcessingStatus { get; set; } // Pending, Success, Failed
    public string ErrorMessage { get; set; }
    public int RetryCount { get; set; }
}

public interface IWebhookEventStore
{
    Task<int> StoreAsync(WebhookEvent webhookEvent);
    Task UpdateProcessingStatusAsync(int id, string status, string errorMessage = null);
    Task<IEnumerable<WebhookEvent>> GetFailedEventsAsync();
}

public class WebhookEventStore : IWebhookEventStore
{
    private readonly ApplicationDbContext _context;
    
    public async Task<int> StoreAsync(WebhookEvent webhookEvent)
    {
        _context.WebhookEvents.Add(webhookEvent);
        await _context.SaveChangesAsync();
        return webhookEvent.Id;
    }
    
    public async Task UpdateProcessingStatusAsync(int id, string status, string errorMessage = null)
    {
        var webhookEvent = await _context.WebhookEvents.FindAsync(id);
        if (webhookEvent != null)
        {
            webhookEvent.ProcessingStatus = status;
            webhookEvent.ProcessedAt = DateTime.UtcNow;
            webhookEvent.ErrorMessage = errorMessage;
            await _context.SaveChangesAsync();
        }
    }
    
    public async Task<IEnumerable<WebhookEvent>> GetFailedEventsAsync()
    {
        return await _context.WebhookEvents
            .Where(e => e.ProcessingStatus == "Failed" && e.RetryCount < 3)
            .ToListAsync();
    }
}

// Enhanced webhook controller
[HttpPost("stripe")]
public async Task<IActionResult> HandleStripeWebhook()
{
    var json = await new StreamReader(HttpContext.Request.Body).ReadToEndAsync();
    
    // Store webhook immediately
    var webhookEvent = new WebhookEvent
    {
        EventId = "temp", // Will be updated after parsing
        Provider = "Stripe",
        RawPayload = json,
        ReceivedAt = DateTime.UtcNow,
        ProcessingStatus = "Pending"
    };
    
    var webhookEventId = await _webhookEventStore.StoreAsync(webhookEvent);
    
    try
    {
        var stripeEvent = ConstructVerifiedEvent(json);
        
        // Update with actual event ID
        webhookEvent.EventId = stripeEvent.Id;
        webhookEvent.EventType = stripeEvent.Type;
        await _webhookEventStore.UpdateProcessingStatusAsync(webhookEventId, "Pending");
        
        // Queue for processing
        _backgroundJobs.Enqueue<IWebhookHandler>(handler => 
            handler.HandleStoredEventAsync(webhookEventId));
        
        return Ok();
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Webhook validation failed");
        await _webhookEventStore.UpdateProcessingStatusAsync(
            webhookEventId, 
            "Failed", 
            ex.Message);
        return BadRequest();
    }
}
```

---

## Error Handling

### Pattern 1: Graceful Degradation

```csharp
public class ResilientWebhookHandler : IWebhookHandler
{
    private readonly IWebhookHandler _innerHandler;
    private readonly ILogger<ResilientWebhookHandler> _logger;
    
    public async Task HandleAsync(string eventId, string eventType, object eventData)
    {
        try
        {
            await _innerHandler.HandleAsync(eventId, eventType, eventData);
        }
        catch (DbUpdateException ex)
        {
            _logger.LogError(ex, "Database error processing webhook {EventId}", eventId);
            // Don't throw - log and potentially queue for manual review
            await LogFailedWebhookAsync(eventId, eventType, ex.Message);
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "External service error processing webhook {EventId}", eventId);
            // Throw to trigger retry
            throw;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unexpected error processing webhook {EventId}", eventId);
            await LogFailedWebhookAsync(eventId, eventType, ex.Message);
        }
    }
    
    private async Task LogFailedWebhookAsync(string eventId, string eventType, string error)
    {
        // Log to monitoring system, create alert, etc.
    }
}
```

### Pattern 2: Retry with Exponential Backoff

Using Hangfire for automatic retries:

```csharp
public class WebhookBackgroundJob
{
    [AutomaticRetry(Attempts = 5, OnAttemptsExceeded = AttemptsExceededAction.Delete)]
    public async Task ProcessWebhookAsync(int webhookEventId)
    {
        var webhookEvent = await _webhookEventStore.GetByIdAsync(webhookEventId);
        
        try
        {
            await _handler.HandleAsync(
                webhookEvent.EventId,
                webhookEvent.EventType,
                JsonConvert.DeserializeObject(webhookEvent.RawPayload)
            );
            
            await _webhookEventStore.UpdateProcessingStatusAsync(
                webhookEventId, 
                "Success");
        }
        catch (Exception ex)
        {
            webhookEvent.RetryCount++;
            await _webhookEventStore.UpdateProcessingStatusAsync(
                webhookEventId, 
                "Failed", 
                ex.Message);
            throw; // Hangfire will retry
        }
    }
}
```

---

## Testing Webhooks

### Local Testing with Stripe CLI

```bash
# Install Stripe CLI
brew install stripe/stripe-cli/stripe

# Login
stripe login

# Forward webhooks to local endpoint
stripe listen --forward-to localhost:5000/api/webhooks/stripe

# Trigger test events
stripe trigger payment_intent.succeeded
stripe trigger payment_intent.payment_failed
stripe trigger charge.refunded
```

### Integration Tests

```csharp
[TestClass]
public class WebhookControllerTests
{
    private WebApplication _app;
    private HttpClient _client;
    
    [TestInitialize]
    public async Task Setup()
    {
        _app = CreateTestApplication();
        await _app.StartAsync();
        _client = _app.GetTestClient();
    }
    
    [TestMethod]
    public async Task HandleStripeWebhook_ValidSignature_ReturnsOk()
    {
        // Arrange
        var payload = GetTestWebhookPayload();
        var signature = GenerateStripeSignature(payload);
        
        var request = new HttpRequestMessage(HttpMethod.Post, "/api/webhooks/stripe");
        request.Content = new StringContent(payload, Encoding.UTF8, "application/json");
        request.Headers.Add("Stripe-Signature", signature);
        
        // Act
        var response = await _client.SendAsync(request);
        
        // Assert
        Assert.AreEqual(HttpStatusCode.OK, response.StatusCode);
    }
    
    [TestMethod]
    public async Task HandleStripeWebhook_InvalidSignature_ReturnsBadRequest()
    {
        // Arrange
        var payload = GetTestWebhookPayload();
        
        var request = new HttpRequestMessage(HttpMethod.Post, "/api/webhooks/stripe");
        request.Content = new StringContent(payload, Encoding.UTF8, "application/json");
        request.Headers.Add("Stripe-Signature", "invalid_signature");
        
        // Act
        var response = await _client.SendAsync(request);
        
        // Assert
        Assert.AreEqual(HttpStatusCode.BadRequest, response.StatusCode);
    }
    
    [TestMethod]
    public async Task HandleStripeWebhook_DuplicateEvent_ProcessesOnlyOnce()
    {
        // Arrange
        var payload = GetTestWebhookPayload();
        var signature = GenerateStripeSignature(payload);
        
        var request = new HttpRequestMessage(HttpMethod.Post, "/api/webhooks/stripe");
        request.Content = new StringContent(payload, Encoding.UTF8, "application/json");
        request.Headers.Add("Stripe-Signature", signature);
        
        // Act - Send same webhook twice
        await _client.SendAsync(request);
        request = new HttpRequestMessage(HttpMethod.Post, "/api/webhooks/stripe");
        request.Content = new StringContent(payload, Encoding.UTF8, "application/json");
        request.Headers.Add("Stripe-Signature", signature);
        var response = await _client.SendAsync(request);
        
        // Assert
        Assert.AreEqual(HttpStatusCode.OK, response.StatusCode);
        // Verify in database that event was processed only once
        var processedCount = await VerifyEventProcessedOnceAsync("evt_test_123");
        Assert.AreEqual(1, processedCount);
    }
}
```

### Manual Testing with Postman

Create a collection with test webhooks:

```json
{
  "id": "evt_test_webhook",
  "object": "event",
  "type": "payment_intent.succeeded",
  "data": {
    "object": {
      "id": "pi_test_123",
      "amount": 2000,
      "currency": "usd",
      "status": "succeeded",
      "metadata": {
        "order_id": "order_123",
        "payment_id": "550e8400-e29b-41d4-a716-446655440000"
      }
    }
  }
}
```

---

## Monitoring and Alerting

### Key Metrics to Track

```csharp
public class WebhookMetrics
{
    private readonly ILogger<WebhookMetrics> _logger;
    
    public void TrackWebhookReceived(string eventType)
    {
        // Increment counter
        _logger.LogInformation("Webhook received: {EventType}", eventType);
    }
    
    public void TrackWebhookProcessed(string eventType, long durationMs)
    {
        // Track processing time
        _logger.LogInformation(
            "Webhook processed: {EventType} in {Duration}ms", 
            eventType, 
            durationMs);
    }
    
    public void TrackWebhookFailed(string eventType, string error)
    {
        // Alert on failures
        _logger.LogError(
            "Webhook failed: {EventType} - {Error}", 
            eventType, 
            error);
    }
}
```

### Health Check Endpoint

```csharp
[HttpGet("health/webhooks")]
public async Task<IActionResult> WebhookHealth()
{
    var failedCount = await _webhookEventStore.GetFailedCountAsync();
    var oldestUnprocessed = await _webhookEventStore.GetOldestUnprocessedAsync();
    
    if (failedCount > 10 || (oldestUnprocessed.HasValue && 
        DateTime.UtcNow - oldestUnprocessed.Value > TimeSpan.FromMinutes(30)))
    {
        return StatusCode(503, new { 
            status = "unhealthy", 
            failedCount,
            oldestUnprocessed 
        });
    }
    
    return Ok(new { status = "healthy" });
}
```

---

## Best Practices Summary

✅ **Always**:
- Verify webhook signatures
- Return 200 quickly, process asynchronously
- Implement idempotency
- Store raw webhook payloads
- Log all webhook events
- Test with provider CLI tools

❌ **Never**:
- Process webhooks synchronously
- Trust webhook data without signature verification
- Assume webhooks arrive in order
- Ignore failed webhooks
- Skip error handling

## Next Steps

- [Security Patterns & Practices](15-SECURITY-PATTERNS-PRACTICES.md)
- [Additional Patterns (Retry, Circuit Breaker)](07-DESIGN-PATTERNS.md)
- [Testing Guide](14-E2E-INTEGRATION-TESTING.md)
