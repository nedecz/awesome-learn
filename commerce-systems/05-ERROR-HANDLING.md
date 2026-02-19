# Error Handling Patterns for Payment Systems

## Overview

Robust error handling in payment systems is non-negotiable. A missed exception can result in double charges, lost money, or silent data corruption. This guide covers resilience patterns, structured error handling, and recovery strategies specific to payment processing.

## Table of Contents

1. [Payment Error Taxonomy](#payment-error-taxonomy)
2. [Structured Exception Hierarchy](#structured-exception-hierarchy)
3. [Result Pattern](#result-pattern)
4. [Circuit Breaker Pattern](#circuit-breaker-pattern)
5. [Retry with Polly](#retry-with-polly)
6. [Idempotency for Safe Retries](#idempotency-for-safe-retries)
7. [Dead Letter Queue](#dead-letter-queue)
8. [Global Error Handling Middleware](#global-error-handling-middleware)
9. [Monitoring and Alerting on Errors](#monitoring-and-alerting-on-errors)
10. [Testing Error Scenarios](#testing-error-scenarios)

---

## Payment Error Taxonomy

Understanding the type of error determines the correct response:

| Error Type | Retryable? | Example | Correct Response |
|-----------|-----------|---------|-----------------|
| **Network Timeout** | ✅ Yes | Gateway didn't respond | Retry with backoff |
| **Rate Limit** | ✅ Yes (with delay) | 429 Too Many Requests | Backoff and retry |
| **Server Error** | ✅ Yes (cautiously) | 500 from Stripe | Retry with circuit breaker |
| **Validation Error** | ❌ No | Invalid card number | Return error to user |
| **Decline** | ❌ No | Insufficient funds | Return decline reason |
| **Authentication Error** | ❌ No | Invalid API key | Alert ops, do not retry |
| **Duplicate** | ❌ No | Idempotency key conflict | Return original result |

---

## Structured Exception Hierarchy

```csharp
// Base exception for all payment errors
public abstract class PaymentException : Exception
{
    public string ErrorCode { get; }
    public string ProviderName { get; }
    public bool IsRetryable { get; }

    protected PaymentException(string message, string errorCode, string providerName, bool isRetryable, Exception inner = null)
        : base(message, inner)
    {
        ErrorCode = errorCode;
        ProviderName = providerName;
        IsRetryable = isRetryable;
    }
}

// Non-retryable: card declined, invalid input
public class PaymentDeclinedException : PaymentException
{
    public string DeclineCode { get; }

    public PaymentDeclinedException(string message, string declineCode, string providerName)
        : base(message, $"PAYMENT_DECLINED_{declineCode}", providerName, isRetryable: false)
    {
        DeclineCode = declineCode;
    }
}

// Retryable: timeouts, transient gateway failures
public class PaymentGatewayException : PaymentException
{
    public int? HttpStatusCode { get; }

    public PaymentGatewayException(string message, string providerName, int? statusCode = null, Exception inner = null)
        : base(message, "GATEWAY_ERROR", providerName, isRetryable: true, inner)
    {
        HttpStatusCode = statusCode;
    }
}

// Non-retryable: bad API key, account suspended
public class PaymentAuthenticationException : PaymentException
{
    public PaymentAuthenticationException(string message, string providerName)
        : base(message, "AUTH_ERROR", providerName, isRetryable: false) { }
}

// Non-retryable: provider doesn't support this operation
public class PaymentNotSupportedException : PaymentException
{
    public PaymentNotSupportedException(string message, string providerName)
        : base(message, "NOT_SUPPORTED", providerName, isRetryable: false) { }
}

// Gateway-specific exception translation
public static class StripeExceptionTranslator
{
    public static PaymentException Translate(StripeException ex)
    {
        return ex.StripeError.Type switch
        {
            "card_error" => new PaymentDeclinedException(
                ex.Message, ex.StripeError.DeclineCode, "Stripe"),

            "rate_limit_error" => new PaymentGatewayException(
                "Rate limit exceeded", "Stripe", 429, ex),

            "authentication_error" => new PaymentAuthenticationException(
                ex.Message, "Stripe"),

            "api_error" => new PaymentGatewayException(
                ex.Message, "Stripe", (int?)ex.HttpStatusCode, ex),

            _ => new PaymentGatewayException(
                ex.Message, "Stripe", (int?)ex.HttpStatusCode, ex)
        };
    }
}
```

---

## Result Pattern

Avoid exceptions for expected outcomes like declines. Use a Result type instead.

```csharp
public class PaymentResult
{
    public bool IsSuccess { get; init; }
    public string TransactionId { get; init; }
    public PaymentStatus Status { get; init; }
    public PaymentError Error { get; init; }

    public static PaymentResult Success(string transactionId) => new()
    {
        IsSuccess = true,
        TransactionId = transactionId,
        Status = PaymentStatus.Completed
    };

    public static PaymentResult Failure(PaymentError error) => new()
    {
        IsSuccess = false,
        Error = error,
        Status = PaymentStatus.Failed
    };

    public static PaymentResult Pending(string transactionId) => new()
    {
        IsSuccess = true,
        TransactionId = transactionId,
        Status = PaymentStatus.RequiresAction
    };
}

public class PaymentError
{
    public string Code { get; init; }
    public string Message { get; init; }
    public string DeclineCode { get; init; }
    public bool IsRetryable { get; init; }
    public string ProviderErrorCode { get; init; }

    // User-friendly decline messages
    public string UserFacingMessage => Code switch
    {
        "insufficient_funds" => "Your card has insufficient funds. Please try another payment method.",
        "card_declined" => "Your card was declined. Please contact your bank or try another card.",
        "expired_card" => "Your card has expired. Please update your payment method.",
        "incorrect_cvc" => "The security code is incorrect. Please check and try again.",
        "processing_error" => "A processing error occurred. Please try again in a moment.",
        _ => "Payment could not be processed. Please try again or use a different payment method."
    };
}
```

### Using the Result Pattern

```csharp
public class PaymentService
{
    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        try
        {
            var gateway = _strategySelector.SelectProvider(request);
            var result = await gateway.ProcessPaymentAsync(request);

            if (!result.IsSuccess)
            {
                _logger.LogWarning("Payment declined: {Code} for order {OrderId}",
                    result.Error.Code, request.OrderId);
            }

            return result; // No exception thrown for declines
        }
        catch (PaymentGatewayException ex) when (ex.IsRetryable)
        {
            _logger.LogError(ex, "Retryable gateway error for order {OrderId}", request.OrderId);
            throw; // Let Polly retry policy handle this
        }
        catch (PaymentException ex)
        {
            _logger.LogError(ex, "Non-retryable payment error for order {OrderId}", request.OrderId);
            return PaymentResult.Failure(new PaymentError
            {
                Code = ex.ErrorCode,
                Message = ex.Message,
                IsRetryable = false
            });
        }
    }
}
```

---

## Circuit Breaker Pattern

Prevent cascading failures when a payment provider is down.

```csharp
// Using Polly for Circuit Breaker
public static class PaymentResiliencePolicies
{
    public static IAsyncPolicy<PaymentResult> GetCircuitBreakerPolicy(
        string providerName, ILogger logger)
    {
        return Policy<PaymentResult>
            .Handle<PaymentGatewayException>(ex => ex.IsRetryable)
            .Or<HttpRequestException>()
            .Or<TimeoutException>()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 5,
                durationOfBreak: TimeSpan.FromSeconds(30),
                onBreak: (outcome, breakDelay) =>
                {
                    logger.LogWarning(
                        "Circuit OPEN for {Provider}. Breaking for {BreakDuration}s. Reason: {Reason}",
                        providerName, breakDelay.TotalSeconds, outcome.Exception?.Message);
                },
                onReset: () =>
                {
                    logger.LogInformation("Circuit CLOSED for {Provider}. Resuming normal operation.", providerName);
                },
                onHalfOpen: () =>
                {
                    logger.LogInformation("Circuit HALF-OPEN for {Provider}. Testing with next request.", providerName);
                });
    }
}

// Circuit Breaker Registry for multi-provider
public class CircuitBreakerRegistry
{
    private readonly ConcurrentDictionary<string, IAsyncPolicy<PaymentResult>> _policies = new();

    public IAsyncPolicy<PaymentResult> GetOrCreate(string providerName, ILogger logger)
    {
        return _policies.GetOrAdd(providerName,
            name => PaymentResiliencePolicies.GetCircuitBreakerPolicy(name, logger));
    }

    public bool IsOpen(string providerName)
    {
        if (_policies.TryGetValue(providerName, out var policy))
        {
            // Check if circuit breaker is in Open state
            return policy is CircuitBreakerPolicy<PaymentResult> cb
                && cb.CircuitState == CircuitState.Open;
        }
        return false;
    }
}
```

---

## Retry with Polly

```csharp
public static class PaymentResiliencePolicies
{
    // Retry with exponential backoff + jitter
    public static IAsyncPolicy<PaymentResult> GetRetryPolicy(string providerName, ILogger logger)
    {
        return Policy<PaymentResult>
            .Handle<PaymentGatewayException>(ex => ex.IsRetryable)
            .Or<HttpRequestException>()
            .Or<TimeoutException>()
            .WaitAndRetryAsync(
                retryCount: 3,
                sleepDurationProvider: attempt =>
                    TimeSpan.FromSeconds(Math.Pow(2, attempt))
                    + TimeSpan.FromMilliseconds(Random.Shared.Next(0, 1000)),
                onRetry: (outcome, delay, attempt, _) =>
                {
                    logger.LogWarning(
                        "Retry {Attempt} for {Provider} after {Delay}ms. Error: {Error}",
                        attempt, providerName, delay.TotalMilliseconds,
                        outcome.Exception?.Message ?? "Result failure");
                });
    }

    // Combined policy: Retry → Circuit Breaker → Timeout
    public static IAsyncPolicy<PaymentResult> GetCombinedPolicy(string providerName, ILogger logger)
    {
        var timeout = Policy.TimeoutAsync<PaymentResult>(TimeSpan.FromSeconds(10));
        var retry = GetRetryPolicy(providerName, logger);
        var circuitBreaker = GetCircuitBreakerPolicy(providerName, logger);

        // Order: outer → inner. Retry wraps Circuit Breaker wraps Timeout
        return Policy.WrapAsync(retry, circuitBreaker, timeout);
    }
}

// DI Registration with HttpClientFactory
services.AddHttpClient("Stripe")
    .AddPolicyHandler((provider, _) =>
    {
        var logger = provider.GetRequiredService<ILogger<StripeGateway>>();
        return PaymentResiliencePolicies.GetCombinedPolicy("Stripe", logger);
    });
```

---

## Idempotency for Safe Retries

Retries are only safe with idempotency guarantees.

```csharp
public class IdempotencyService
{
    private readonly IDistributedCache _cache;
    private readonly IPaymentRepository _repository;

    public async Task<PaymentResult> ExecuteIdempotently(
        string idempotencyKey,
        Func<Task<PaymentResult>> operation)
    {
        // 1. Check cache for existing result
        var cached = await _cache.GetStringAsync($"idempotency:{idempotencyKey}");
        if (cached != null)
        {
            return JsonSerializer.Deserialize<PaymentResult>(cached);
        }

        // 2. Check database for existing result
        var existing = await _repository.GetByIdempotencyKeyAsync(idempotencyKey);
        if (existing != null)
        {
            return existing.ToPaymentResult();
        }

        // 3. Execute and store result
        var result = await operation();

        await _cache.SetStringAsync(
            $"idempotency:{idempotencyKey}",
            JsonSerializer.Serialize(result),
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(24) });

        return result;
    }
}

// Idempotency key middleware
public class IdempotencyMiddleware
{
    private readonly RequestDelegate _next;

    public async Task InvokeAsync(HttpContext context, IdempotencyService idempotencyService)
    {
        if (context.Request.Method == "POST"
            && context.Request.Headers.TryGetValue("Idempotency-Key", out var key))
        {
            var result = await idempotencyService.GetCachedResultAsync(key);
            if (result != null)
            {
                context.Response.StatusCode = 200;
                context.Response.Headers["Idempotency-Replayed"] = "true";
                await context.Response.WriteAsJsonAsync(result);
                return;
            }
        }

        await _next(context);
    }
}
```

---

## Dead Letter Queue

Handle permanently failed payment operations.

```csharp
public class PaymentDeadLetterQueue
{
    private readonly IMessageBus _messageBus;
    private readonly IPaymentRepository _repository;
    private readonly ILogger<PaymentDeadLetterQueue> _logger;

    public async Task SendToDeadLetterAsync(PaymentRequest request, Exception exception, int attemptCount)
    {
        var deadLetter = new DeadLetterMessage
        {
            Id = Guid.NewGuid(),
            OriginalRequest = request,
            ErrorMessage = exception.Message,
            ExceptionType = exception.GetType().Name,
            StackTrace = exception.StackTrace,
            AttemptCount = attemptCount,
            CreatedAt = DateTime.UtcNow,
            Status = DeadLetterStatus.Pending
        };

        await _messageBus.PublishAsync("payment.dead-letter", deadLetter);

        _logger.LogError(exception,
            "Payment for order {OrderId} sent to dead letter queue after {Attempts} attempts",
            request.OrderId, attemptCount);
    }

    // Manual retry from DLQ
    public async Task<PaymentResult> RetryFromDeadLetterAsync(Guid deadLetterId)
    {
        var deadLetter = await _repository.GetDeadLetterAsync(deadLetterId);

        try
        {
            var result = await _paymentService.ProcessPaymentAsync(deadLetter.OriginalRequest);
            deadLetter.Status = DeadLetterStatus.Resolved;
            await _repository.UpdateDeadLetterAsync(deadLetter);
            return result;
        }
        catch (Exception ex)
        {
            deadLetter.RetryCount++;
            deadLetter.LastRetryAt = DateTime.UtcNow;
            await _repository.UpdateDeadLetterAsync(deadLetter);
            throw;
        }
    }
}
```

---

## Global Error Handling Middleware

```csharp
public class PaymentExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<PaymentExceptionMiddleware> _logger;

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (PaymentDeclinedException ex)
        {
            _logger.LogWarning("Payment declined: {Code} - {Message}", ex.ErrorCode, ex.Message);
            context.Response.StatusCode = 422; // Unprocessable Entity
            await context.Response.WriteAsJsonAsync(new ProblemDetails
            {
                Title = "Payment Declined",
                Detail = ex.Message,
                Status = 422,
                Extensions = { ["declineCode"] = ex.DeclineCode }
            });
        }
        catch (PaymentGatewayException ex)
        {
            _logger.LogError(ex, "Payment gateway error: {Provider}", ex.ProviderName);
            context.Response.StatusCode = 502; // Bad Gateway
            await context.Response.WriteAsJsonAsync(new ProblemDetails
            {
                Title = "Payment Processing Error",
                Detail = "The payment could not be processed at this time. Please try again.",
                Status = 502
            });
        }
        catch (PaymentAuthenticationException ex)
        {
            _logger.LogCritical(ex, "Payment auth failure for {Provider} — check API keys", ex.ProviderName);
            context.Response.StatusCode = 500;
            await context.Response.WriteAsJsonAsync(new ProblemDetails
            {
                Title = "Internal Server Error",
                Detail = "An internal error occurred.",
                Status = 500
            });
        }
        catch (Exception ex)
        {
            _logger.LogCritical(ex, "Unhandled exception in payment pipeline");
            context.Response.StatusCode = 500;
            await context.Response.WriteAsJsonAsync(new ProblemDetails
            {
                Title = "Internal Server Error",
                Detail = "An unexpected error occurred.",
                Status = 500
            });
        }
    }
}

// Registration
app.UseMiddleware<PaymentExceptionMiddleware>();
```

---

## Monitoring and Alerting on Errors

```csharp
public class PaymentErrorMetrics
{
    private readonly IMeterFactory _meterFactory;
    private readonly Counter<long> _errorCounter;
    private readonly Histogram<double> _errorLatency;

    public PaymentErrorMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("PaymentSystem.Errors");
        _errorCounter = meter.CreateCounter<long>("payment.errors.total", description: "Total payment errors");
        _errorLatency = meter.CreateHistogram<double>("payment.error.latency_ms", description: "Time to error");
    }

    public void RecordError(string provider, string errorType, bool isRetryable)
    {
        _errorCounter.Add(1,
            new KeyValuePair<string, object>("provider", provider),
            new KeyValuePair<string, object>("error_type", errorType),
            new KeyValuePair<string, object>("retryable", isRetryable));
    }
}

// Alert rules (Prometheus-style)
// ALERT PaymentErrorRateHigh
//   expr: rate(payment_errors_total[5m]) > 10
//   for: 2m
//   labels:
//     severity: critical
//   annotations:
//     summary: "High payment error rate: {{ $value }}/s"

// ALERT CircuitBreakerOpen
//   expr: payment_circuit_breaker_state == 1
//   for: 0m
//   labels:
//     severity: critical
//   annotations:
//     summary: "Circuit breaker open for {{ $labels.provider }}"
```

---

## Testing Error Scenarios

```csharp
[TestClass]
public class ErrorHandlingTests
{
    [TestMethod]
    public async Task ProcessPayment_Timeout_RetriesThreeTimes()
    {
        var callCount = 0;
        var mockGateway = new Mock<IPaymentGateway>();
        mockGateway
            .Setup(g => g.ProcessPaymentAsync(It.IsAny<PaymentRequest>()))
            .Callback(() => callCount++)
            .ThrowsAsync(new TimeoutException("Gateway timeout"));

        var policy = PaymentResiliencePolicies.GetRetryPolicy("Test", Mock.Of<ILogger>());

        await Assert.ThrowsExceptionAsync<TimeoutException>(async () =>
            await policy.ExecuteAsync(() => mockGateway.Object.ProcessPaymentAsync(new PaymentRequest())));

        Assert.AreEqual(4, callCount); // 1 initial + 3 retries
    }

    [TestMethod]
    public async Task ProcessPayment_Decline_NoRetry()
    {
        var callCount = 0;
        var mockGateway = new Mock<IPaymentGateway>();
        mockGateway
            .Setup(g => g.ProcessPaymentAsync(It.IsAny<PaymentRequest>()))
            .Callback(() => callCount++)
            .ThrowsAsync(new PaymentDeclinedException("Declined", "insufficient_funds", "Stripe"));

        var policy = PaymentResiliencePolicies.GetRetryPolicy("Test", Mock.Of<ILogger>());

        // PaymentDeclinedException is not retryable — should propagate immediately
        await Assert.ThrowsExceptionAsync<PaymentDeclinedException>(async () =>
            await policy.ExecuteAsync(() => mockGateway.Object.ProcessPaymentAsync(new PaymentRequest())));

        Assert.AreEqual(1, callCount); // No retries
    }

    [TestMethod]
    public async Task CircuitBreaker_OpensAfterConsecutiveFailures()
    {
        var mockGateway = new Mock<IPaymentGateway>();
        mockGateway
            .Setup(g => g.ProcessPaymentAsync(It.IsAny<PaymentRequest>()))
            .ThrowsAsync(new PaymentGatewayException("Server error", "Stripe", 500));

        var policy = PaymentResiliencePolicies.GetCircuitBreakerPolicy("Stripe", Mock.Of<ILogger>());

        // Trip the circuit breaker (5 failures)
        for (int i = 0; i < 5; i++)
        {
            await Assert.ThrowsExceptionAsync<PaymentGatewayException>(async () =>
                await policy.ExecuteAsync(() => mockGateway.Object.ProcessPaymentAsync(new PaymentRequest())));
        }

        // 6th call should throw BrokenCircuitException
        await Assert.ThrowsExceptionAsync<BrokenCircuitException<PaymentResult>>(async () =>
            await policy.ExecuteAsync(() => mockGateway.Object.ProcessPaymentAsync(new PaymentRequest())));
    }

    [TestMethod]
    public async Task Idempotency_ReturnsOriginalResult()
    {
        var service = new IdempotencyService(_cache, _repository);
        var callCount = 0;

        // First call
        var result1 = await service.ExecuteIdempotently("key-123", async () =>
        {
            callCount++;
            return PaymentResult.Success("txn_abc");
        });

        // Second call with same key
        var result2 = await service.ExecuteIdempotently("key-123", async () =>
        {
            callCount++;
            return PaymentResult.Success("txn_xyz");
        });

        Assert.AreEqual(1, callCount); // Only called once
        Assert.AreEqual(result1.TransactionId, result2.TransactionId);
    }
}
```

---

## Error Handling Checklist

- [ ] Every external call wrapped in try/catch with specific exception types
- [ ] Retries only for transient/retryable errors
- [ ] Circuit breaker on every provider gateway
- [ ] Idempotency keys on all payment mutations
- [ ] Dead letter queue for permanently failed operations
- [ ] User-facing messages sanitized (no stack traces or internal codes)
- [ ] Structured logging with correlation IDs on every error
- [ ] Alerts configured for error rate spikes and circuit breaker openings
- [ ] Decline codes mapped to user-friendly messages
- [ ] Timeout configured on every HTTP client call

## Next Steps

- [Webhook Patterns](04-WEBHOOK-PATTERNS.md) — error handling for async webhook delivery
- [Additional Patterns](07-DESIGN-PATTERNS.md) — Bulkhead, Rate Limiting
- [Security Patterns & Practices](15-SECURITY-PATTERNS-PRACTICES.md)
