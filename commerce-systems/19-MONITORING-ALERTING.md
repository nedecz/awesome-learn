# Monitoring and Alerting for Payment Systems

## Overview

Payment systems demand exceptional observability. A missed alert on rising error rates or latency can mean lost revenue and customer trust. This guide covers metrics, structured logging, distributed tracing, health checks, dashboards, and alert rules for .NET 10.0+ payment applications.

## Table of Contents

1. [Metrics with OpenTelemetry](#metrics-with-opentelemetry)
2. [Structured Logging with Serilog](#structured-logging-with-serilog)
3. [Distributed Tracing](#distributed-tracing)
4. [Health Checks](#health-checks)
5. [Application Insights Integration](#application-insights-integration)
6. [Prometheus + Grafana Stack](#prometheus--grafana-stack)
7. [Alert Rules](#alert-rules)
8. [Payment-Specific Dashboards](#payment-specific-dashboards)
9. [Incident Response Runbooks](#incident-response-runbooks)

---

## Metrics with OpenTelemetry

### Setup

```bash
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Exporter.Prometheus.AspNetCore
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
```

```csharp
// Program.cs
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics =>
    {
        metrics
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddMeter("PaymentSystem")
            .AddMeter("PaymentSystem.Errors")
            .AddPrometheusExporter();
    })
    .WithTracing(tracing =>
    {
        tracing
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddSource("PaymentSystem")
            .AddOtlpExporter(opts =>
            {
                opts.Endpoint = new Uri(builder.Configuration["Otlp:Endpoint"]);
            });
    });

app.MapPrometheusScrapingEndpoint();
```

### Custom Payment Metrics

```csharp
public class PaymentMetrics
{
    private readonly Counter<long> _paymentsProcessed;
    private readonly Counter<long> _paymentsFailed;
    private readonly Histogram<double> _paymentDuration;
    private readonly Histogram<double> _paymentAmount;
    private readonly UpDownCounter<long> _activePayments;
    private readonly Counter<long> _webhooksReceived;
    private readonly Counter<long> _refundsProcessed;

    public PaymentMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("PaymentSystem");

        _paymentsProcessed = meter.CreateCounter<long>(
            "payment.processed.total",
            description: "Total number of payments processed");

        _paymentsFailed = meter.CreateCounter<long>(
            "payment.failed.total",
            description: "Total number of failed payments");

        _paymentDuration = meter.CreateHistogram<double>(
            "payment.duration_ms",
            unit: "ms",
            description: "Payment processing duration in milliseconds");

        _paymentAmount = meter.CreateHistogram<double>(
            "payment.amount",
            description: "Payment amounts processed");

        _activePayments = meter.CreateUpDownCounter<long>(
            "payment.active",
            description: "Currently processing payments");

        _webhooksReceived = meter.CreateCounter<long>(
            "webhook.received.total",
            description: "Total webhooks received by provider and type");

        _refundsProcessed = meter.CreateCounter<long>(
            "refund.processed.total",
            description: "Total refunds processed");
    }

    public void RecordPaymentProcessed(string provider, string currency, string paymentMethod)
    {
        _paymentsProcessed.Add(1,
            new("provider", provider),
            new("currency", currency),
            new("payment_method", paymentMethod));
    }

    public void RecordPaymentFailed(string provider, string errorCode, bool isRetryable)
    {
        _paymentsFailed.Add(1,
            new("provider", provider),
            new("error_code", errorCode),
            new("retryable", isRetryable));
    }

    public void RecordPaymentDuration(double durationMs, string provider)
    {
        _paymentDuration.Record(durationMs, new("provider", provider));
    }

    public void RecordPaymentAmount(double amount, string currency)
    {
        _paymentAmount.Record(amount, new("currency", currency));
    }

    public IDisposable TrackActivePayment()
    {
        _activePayments.Add(1);
        return new ActivePaymentTracker(_activePayments);
    }

    public void RecordWebhook(string provider, string eventType, bool success)
    {
        _webhooksReceived.Add(1,
            new("provider", provider),
            new("event_type", eventType),
            new("success", success));
    }

    private class ActivePaymentTracker : IDisposable
    {
        private readonly UpDownCounter<long> _counter;
        public ActivePaymentTracker(UpDownCounter<long> counter) => _counter = counter;
        public void Dispose() => _counter.Add(-1);
    }
}
```

### Instrumented Payment Service

```csharp
public class InstrumentedPaymentService : IPaymentService
{
    private readonly IPaymentGateway _gateway;
    private readonly PaymentMetrics _metrics;
    private readonly ILogger<InstrumentedPaymentService> _logger;
    private static readonly ActivitySource _activitySource = new("PaymentSystem");

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        using var activity = _activitySource.StartActivity("ProcessPayment");
        activity?.SetTag("order.id", request.OrderId);
        activity?.SetTag("payment.currency", request.Currency);
        activity?.SetTag("payment.provider", _gateway.ProviderName);

        using var activeTracker = _metrics.TrackActivePayment();
        var stopwatch = Stopwatch.StartNew();

        try
        {
            var result = await _gateway.ProcessPaymentAsync(request);
            stopwatch.Stop();

            _metrics.RecordPaymentDuration(stopwatch.ElapsedMilliseconds, _gateway.ProviderName);
            _metrics.RecordPaymentAmount((double)request.Amount, request.Currency);

            if (result.IsSuccess)
            {
                _metrics.RecordPaymentProcessed(_gateway.ProviderName, request.Currency, request.PaymentMethod);
                activity?.SetTag("payment.status", "success");
            }
            else
            {
                _metrics.RecordPaymentFailed(_gateway.ProviderName, result.Error?.Code ?? "unknown", result.Error?.IsRetryable ?? false);
                activity?.SetTag("payment.status", "failed");
                activity?.SetTag("payment.error_code", result.Error?.Code);
            }

            return result;
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            _metrics.RecordPaymentDuration(stopwatch.ElapsedMilliseconds, _gateway.ProviderName);
            _metrics.RecordPaymentFailed(_gateway.ProviderName, "exception", true);
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            throw;
        }
    }
}
```

---

## Structured Logging with Serilog

### Setup

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.Seq
dotnet add package Serilog.Enrichers.Environment
dotnet add package Serilog.Enrichers.Thread
```

```csharp
// Program.cs
builder.Host.UseSerilog((context, config) =>
{
    config
        .ReadFrom.Configuration(context.Configuration)
        .Enrich.FromLogContext()
        .Enrich.WithMachineName()
        .Enrich.WithThreadId()
        .Enrich.WithProperty("Application", "PaymentSystem")
        .WriteTo.Console(outputTemplate:
            "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}")
        .WriteTo.Seq("http://localhost:5341");
});
```

### Payment-Specific Log Context

```csharp
public class PaymentLoggingMiddleware
{
    private readonly RequestDelegate _next;

    public async Task InvokeAsync(HttpContext context)
    {
        var correlationId = context.Request.Headers["X-Correlation-ID"].FirstOrDefault()
            ?? Guid.NewGuid().ToString();

        context.Response.Headers["X-Correlation-ID"] = correlationId;

        using (LogContext.PushProperty("CorrelationId", correlationId))
        using (LogContext.PushProperty("RequestPath", context.Request.Path))
        using (LogContext.PushProperty("ClientIP", context.Connection.RemoteIpAddress))
        {
            await _next(context);
        }
    }
}

// Logging best practices for payments
public class PaymentService
{
    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        using (LogContext.PushProperty("OrderId", request.OrderId))
        using (LogContext.PushProperty("PaymentProvider", _gateway.ProviderName))
        using (LogContext.PushProperty("Currency", request.Currency))
        // ⚠️ NEVER log Amount or card details to avoid PCI issues in some contexts
        {
            _logger.LogInformation("Payment processing started for order {OrderId}", request.OrderId);

            var result = await _gateway.ProcessPaymentAsync(request);

            if (result.IsSuccess)
            {
                _logger.LogInformation(
                    "Payment completed: {TransactionId} for order {OrderId}",
                    result.TransactionId, request.OrderId);
            }
            else
            {
                _logger.LogWarning(
                    "Payment declined for order {OrderId}: {ErrorCode} - {ErrorMessage}",
                    request.OrderId, result.Error?.Code, result.Error?.Message);
            }

            return result;
        }
    }
}
```

### What NOT to Log (PCI Compliance)

```csharp
// ❌ NEVER log these
_logger.LogInformation("Card: {CardNumber}", request.CardNumber);
_logger.LogInformation("CVV: {Cvv}", request.Cvv);
_logger.LogInformation("Full request: {@Request}", request); // May contain sensitive data

// ✅ OK to log
_logger.LogInformation("Card ending in {Last4}", request.Last4);
_logger.LogInformation("Payment {TransactionId} for {Amount} {Currency}", txnId, amount, currency);
```

---

## Distributed Tracing

```csharp
// Trace across payment service → gateway → webhook
public class PaymentTracingService
{
    private static readonly ActivitySource _source = new("PaymentSystem");

    public async Task<PaymentResult> ProcessWithTracingAsync(PaymentRequest request)
    {
        // Root span
        using var activity = _source.StartActivity("payment.process", ActivityKind.Server);
        activity?.SetTag("order.id", request.OrderId);

        // Child span: validate
        using (var validateActivity = _source.StartActivity("payment.validate"))
        {
            await ValidateRequestAsync(request);
            validateActivity?.SetTag("validation.passed", true);
        }

        // Child span: gateway call
        PaymentResult result;
        using (var gatewayActivity = _source.StartActivity("payment.gateway", ActivityKind.Client))
        {
            gatewayActivity?.SetTag("gateway.provider", _gateway.ProviderName);
            result = await _gateway.ProcessPaymentAsync(request);
            gatewayActivity?.SetTag("gateway.status", result.IsSuccess ? "success" : "failure");
        }

        // Child span: persist
        using (var persistActivity = _source.StartActivity("payment.persist"))
        {
            await SavePaymentAsync(request, result);
        }

        return result;
    }
}
```

---

## Health Checks

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddCheck<StripeHealthCheck>("stripe", tags: new[] { "payment-provider" })
    .AddCheck<PayPalHealthCheck>("paypal", tags: new[] { "payment-provider" })
    .AddCheck<DatabaseHealthCheck>("database", tags: new[] { "infrastructure" })
    .AddCheck<RedisHealthCheck>("redis", tags: new[] { "infrastructure" })
    .AddCheck<MessageBusHealthCheck>("messagebus", tags: new[] { "infrastructure" });

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("infrastructure")
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false // Just checks the app responds
});

// Stripe health check
public class StripeHealthCheck : IHealthCheck
{
    private readonly IStripeClient _client;

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, CancellationToken cancellationToken)
    {
        try
        {
            var service = new BalanceService(_client);
            var balance = await service.GetAsync(cancellationToken: cancellationToken);

            return HealthCheckResult.Healthy($"Stripe connected. Balance: {balance.Available?.Count ?? 0} currencies");
        }
        catch (StripeException ex)
        {
            return HealthCheckResult.Unhealthy($"Stripe unavailable: {ex.Message}");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy($"Stripe check failed: {ex.Message}");
        }
    }
}

// Database health check with query timeout
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly ApplicationDbContext _context;

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, CancellationToken cancellationToken)
    {
        try
        {
            using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
            cts.CancelAfter(TimeSpan.FromSeconds(5));

            await _context.Database.CanConnectAsync(cts.Token);

            var pendingPayments = await _context.Payments
                .Where(p => p.Status == PaymentStatus.Pending)
                .CountAsync(cts.Token);

            return HealthCheckResult.Healthy(
                $"Database connected. {pendingPayments} pending payments.");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy($"Database check failed: {ex.Message}");
        }
    }
}
```

---

## Application Insights Integration

```csharp
// Program.cs
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
});

builder.Services.AddSingleton<ITelemetryInitializer, PaymentTelemetryInitializer>();

// Custom telemetry initializer
public class PaymentTelemetryInitializer : ITelemetryInitializer
{
    public void Initialize(ITelemetry telemetry)
    {
        telemetry.Context.Cloud.RoleName = "PaymentService";
        telemetry.Context.GlobalProperties["Environment"] =
            Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "Unknown";
    }
}

// Track custom events
public class AppInsightsPaymentTracker
{
    private readonly TelemetryClient _telemetry;

    public void TrackPaymentCompleted(PaymentResult result, PaymentRequest request)
    {
        _telemetry.TrackEvent("PaymentCompleted", new Dictionary<string, string>
        {
            ["Provider"] = result.ProviderName,
            ["Currency"] = request.Currency,
            ["OrderId"] = request.OrderId
        }, new Dictionary<string, double>
        {
            ["Amount"] = (double)request.Amount,
            ["DurationMs"] = result.ProcessingTimeMs
        });
    }

    public void TrackPaymentFailed(string orderId, string errorCode, string provider)
    {
        _telemetry.TrackEvent("PaymentFailed", new Dictionary<string, string>
        {
            ["OrderId"] = orderId,
            ["ErrorCode"] = errorCode,
            ["Provider"] = provider
        });
    }
}
```

---

## Prometheus + Grafana Stack

### Prometheus Configuration

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'payment-service'
    scrape_interval: 15s
    static_configs:
      - targets: ['payment-service:8080']
    metrics_path: '/metrics'
```

### Key Prometheus Queries

```promql
# Payment success rate (last 5 minutes)
sum(rate(payment_processed_total[5m]))
/
(sum(rate(payment_processed_total[5m])) + sum(rate(payment_failed_total[5m])))

# Average payment processing time by provider
histogram_quantile(0.95, sum(rate(payment_duration_ms_bucket[5m])) by (le, provider))

# Payment volume per minute
sum(rate(payment_processed_total[1m])) by (provider)

# Error rate by provider
sum(rate(payment_failed_total[5m])) by (provider)

# Revenue per hour
sum(increase(payment_amount_sum[1h])) by (currency)
```

---

## Alert Rules

### Prometheus Alert Rules

```yaml
groups:
  - name: payment-alerts
    rules:
      # Payment failure rate > 10%
      - alert: HighPaymentFailureRate
        expr: |
          (sum(rate(payment_failed_total[5m])) /
           (sum(rate(payment_processed_total[5m])) + sum(rate(payment_failed_total[5m])))) > 0.10
        for: 2m
        labels:
          severity: critical
          team: payments
        annotations:
          summary: "Payment failure rate is {{ $value | humanizePercentage }}"
          runbook: "https://wiki.internal/runbooks/high-payment-failure-rate"

      # Payment processing time > 5s (p95)
      - alert: SlowPaymentProcessing
        expr: |
          histogram_quantile(0.95, sum(rate(payment_duration_ms_bucket[5m])) by (le, provider)) > 5000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Payment p95 latency is {{ $value }}ms for {{ $labels.provider }}"

      # No payments processed in 10 minutes (during business hours)
      - alert: NoPaymentsProcessed
        expr: |
          sum(increase(payment_processed_total[10m])) == 0
          and ON() hour() >= 8 and ON() hour() <= 22
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "No payments processed in the last 10 minutes"

      # Circuit breaker open
      - alert: CircuitBreakerOpen
        expr: payment_circuit_breaker_state{state="open"} == 1
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Circuit breaker OPEN for {{ $labels.provider }}"

      # Webhook processing failure rate
      - alert: WebhookProcessingFailures
        expr: |
          sum(rate(webhook_received_total{success="false"}[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Webhook processing failures: {{ $value }}/s"

      # Health check failing
      - alert: PaymentProviderUnhealthy
        expr: up{job="payment-service"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Payment service is down"

      # High pending payment count (possible stuck payments)
      - alert: HighPendingPayments
        expr: payment_active > 50
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "{{ $value }} payments currently being processed"

      # Refund spike
      - alert: RefundSpike
        expr: |
          sum(increase(refund_processed_total[1h])) > 
          3 * sum(increase(refund_processed_total[1h] offset 1d))
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Refund rate is 3x higher than yesterday"
```

---

## Payment-Specific Dashboards

### Key Dashboard Panels

| Panel | Metric | Visualization |
|-------|--------|--------------|
| Payment Success Rate | `payment_processed / (processed + failed)` | Gauge (target: 98%+) |
| Transaction Volume | `rate(payment_processed_total[5m])` | Time series |
| Revenue | `increase(payment_amount_sum[1h])` by currency | Stat panels |
| Processing Latency (p50, p95, p99) | `histogram_quantile(...)` | Time series |
| Errors by Type | `payment_failed_total` by error_code | Pie chart |
| Provider Health | Circuit breaker state | Status map |
| Active Payments | `payment_active` | Gauge |
| Webhook Delivery | `webhook_received_total` by type | Bar chart |
| Top Decline Codes | `payment_failed_total` by error_code | Table |

---

## Incident Response Runbooks

### Runbook: High Payment Failure Rate

```markdown
## Trigger: PaymentFailureRate > 10% for 2+ minutes

### Triage (1-2 minutes)
1. Check which provider is failing: Grafana → Provider Error Breakdown
2. Check if circuit breaker is open: Grafana → Provider Health panel
3. Check provider status pages:
   - Stripe: https://status.stripe.com
   - PayPal: https://www.paypal-status.com

### If Provider Outage
1. Confirm circuit breaker has activated (auto-failover to secondary)
2. If no failover, manually switch provider in config
3. Communicate to customers via status page

### If Not Provider Outage
1. Check error codes — are they declines or system errors?
2. If declines: check for fraud pattern, abnormal traffic source
3. If system errors:
   a. Check recent deployments: `kubectl rollout history deployment/payment-service`
   b. Check logs: `kubectl logs -l app=payment-service --since=10m | grep ERROR`
   c. Check database connectivity: `/health/ready` endpoint
   d. Roll back if recent deployment: `kubectl rollout undo deployment/payment-service`

### Resolution
1. Document root cause
2. Update runbook if needed
3. Create post-incident review ticket
```

### Runbook: No Payments Processed

```markdown
## Trigger: Zero payments for 10+ minutes during business hours

### Triage
1. Is the service up? Check `/health/live`
2. Is the database reachable? Check `/health/ready`
3. Are requests reaching the service? Check load balancer logs
4. Check for deployment in progress
5. Check for DNS or networking issues

### Common Causes
- Pod crash loop: `kubectl get pods -l app=payment-service`
- Database connection pool exhausted: check active connections
- Message bus down: check Redis/SQS health
- Certificate expired: check TLS cert expiry
```

---

## Monitoring Checklist

- [ ] OpenTelemetry metrics configured (counters, histograms)
- [ ] Structured logging with correlation IDs
- [ ] Distributed tracing across payment flow
- [ ] Health checks for all dependencies (DB, Redis, providers)
- [ ] Prometheus scraping endpoint exposed
- [ ] Grafana dashboard with payment KPIs
- [ ] Alert rules for failure rate, latency, volume anomalies
- [ ] PCI-compliant logging (no card data in logs)
- [ ] Incident response runbooks documented
- [ ] On-call rotation configured for critical alerts

## Next Steps

- [Performance Optimization](08-PERFORMANCE-OPTIMIZATION.md)
- [Error Handling](05-ERROR-HANDLING.md)
- [Cloud-Native AWS](09-CLOUD-NATIVE-AWS.md)
