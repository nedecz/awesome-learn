# Microservices Patterns for Payment Systems

## Overview

This guide covers patterns for decomposing a payment system into independently deployable microservices while maintaining data consistency, security, and operational reliability. It builds on the [Architecture Patterns](01-ARCHITECTURE-PATTERNS.md) and [Design Patterns](07-DESIGN-PATTERNS.md) guides with payment-specific decomposition strategies.

## Table of Contents

1. [Service Decomposition](#service-decomposition)
2. [API Gateway Pattern](#api-gateway-pattern)
3. [Inter-Service Communication](#inter-service-communication)
4. [Data Ownership & Database per Service](#data-ownership)
5. [Service Discovery](#service-discovery)
6. [Distributed Transactions](#distributed-transactions)
7. [Resilience Patterns](#resilience-patterns)
8. [Shared Nothing Architecture](#shared-nothing-architecture)
9. [API Versioning](#api-versioning)
10. [Deployment Patterns](#deployment-patterns)
11. [Testing Microservices](#testing-microservices)

---

## Service Decomposition

### Bounded Contexts for Payment Systems

```
┌─────────────────────────────────────────────────────────────┐
│                     API Gateway / BFF                        │
├──────────┬──────────┬──────────┬──────────┬─────────────────┤
│ Payment  │ Customer │ Billing  │ Fraud    │ Notification    │
│ Service  │ Service  │ Service  │ Service  │ Service         │
│          │          │          │          │                 │
│ • Intents│ • CRUD   │ • Invoices│• Rules  │ • Email         │
│ • Capture│ • Vault  │ • Subs   │ • ML    │ • SMS           │
│ • Refund │ • Prefs  │ • Dunning│ • Score │ • Push          │
│ • Status │ • History│ • Tax    │ • Alert │ • Webhooks      │
├──────────┼──────────┼──────────┼──────────┼─────────────────┤
│ PaymentDB│ CustDB   │ BillingDB│ FraudDB │ NotifDB/Queue   │
└──────────┴──────────┴──────────┴──────────┴─────────────────┘
```

### Service Contracts

```csharp
// Payment Service — public contract (shared via NuGet package or OpenAPI)
namespace PaymentService.Contracts;

public record ProcessPaymentCommand(
    string OrderId,
    decimal Amount,
    string Currency,
    string PaymentMethodToken,
    string CustomerId,
    string IdempotencyKey);

public record PaymentResponse(
    string PaymentId,
    string Status,
    string TransactionId,
    DateTime ProcessedAt);

// Events published by Payment Service
public record PaymentProcessedEvent(
    string PaymentId,
    string OrderId,
    decimal Amount,
    string Currency,
    string Status,
    DateTime OccurredAt);
```

---

## API Gateway Pattern

### Backend for Frontend (BFF)

```csharp
// API Gateway using YARP reverse proxy
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("payment-api", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
    });
});

var app = builder.Build();
app.UseRateLimiter();
app.MapReverseProxy();
app.Run();
```

```json
// appsettings.json — YARP route configuration
{
  "ReverseProxy": {
    "Routes": {
      "payment-route": {
        "ClusterId": "payment-cluster",
        "Match": { "Path": "/api/payments/{**catch-all}" },
        "Transforms": [
          { "PathRemovePrefix": "/api/payments" }
        ]
      },
      "billing-route": {
        "ClusterId": "billing-cluster",
        "Match": { "Path": "/api/billing/{**catch-all}" }
      }
    },
    "Clusters": {
      "payment-cluster": {
        "Destinations": {
          "payment-svc": { "Address": "http://payment-service:8080/" }
        },
        "HealthCheck": {
          "Active": { "Enabled": true, "Interval": "00:00:10", "Path": "/health" }
        }
      }
    }
  }
}
```

### API Composition (Aggregation)

```csharp
// When a single client request needs data from multiple services
[ApiController]
[Route("api/checkout")]
public class CheckoutAggregatorController : ControllerBase
{
    private readonly IPaymentServiceClient _paymentClient;
    private readonly ICustomerServiceClient _customerClient;
    private readonly IBillingServiceClient _billingClient;

    [HttpGet("summary/{orderId}")]
    public async Task<IActionResult> GetCheckoutSummary(string orderId)
    {
        // Fan out to multiple services in parallel
        var paymentTask = _paymentClient.GetPaymentStatusAsync(orderId);
        var customerTask = _customerClient.GetCustomerAsync(HttpContext.User.GetCustomerId());
        var invoiceTask = _billingClient.GetInvoiceAsync(orderId);

        await Task.WhenAll(paymentTask, customerTask, invoiceTask);

        return Ok(new CheckoutSummary
        {
            PaymentStatus = paymentTask.Result.Status,
            CustomerName = customerTask.Result.Name,
            InvoiceUrl = invoiceTask.Result.DownloadUrl,
            Total = paymentTask.Result.Amount
        });
    }
}
```

---

## Inter-Service Communication

### Synchronous (HTTP + gRPC)

```csharp
// Typed HTTP client with Polly resilience
builder.Services.AddHttpClient<IPaymentServiceClient, PaymentServiceClient>(client =>
{
    client.BaseAddress = new Uri("http://payment-service:8080");
    client.Timeout = TimeSpan.FromSeconds(10);
})
.AddStandardResilienceHandler();

public class PaymentServiceClient : IPaymentServiceClient
{
    private readonly HttpClient _httpClient;

    public async Task<PaymentResponse> ProcessPaymentAsync(ProcessPaymentCommand command)
    {
        var response = await _httpClient.PostAsJsonAsync("/payments", command);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<PaymentResponse>();
    }
}
```

```csharp
// gRPC for internal high-performance calls
// payment.proto
syntax = "proto3";

service PaymentGrpcService {
  rpc ProcessPayment (PaymentRequest) returns (PaymentReply);
  rpc GetPaymentStatus (PaymentStatusRequest) returns (PaymentStatusReply);
}

message PaymentRequest {
  string order_id = 1;
  int64 amount_cents = 2;
  string currency = 3;
  string payment_method_token = 4;
}

message PaymentReply {
  string payment_id = 1;
  string status = 2;
  string transaction_id = 3;
}
```

### Asynchronous (Message-Based)

```csharp
// MassTransit with RabbitMQ (or SQS, Azure Service Bus)
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<PaymentProcessedConsumer>();

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("rabbitmq://localhost");

        cfg.ReceiveEndpoint("billing-payment-processed", e =>
        {
            e.ConfigureConsumer<PaymentProcessedConsumer>(context);
            e.UseMessageRetry(r => r.Intervals(1000, 5000, 15000));
        });
    });
});

public class PaymentProcessedConsumer : IConsumer<PaymentProcessedEvent>
{
    private readonly IBillingService _billingService;

    public async Task Consume(ConsumeContext<PaymentProcessedEvent> context)
    {
        await _billingService.CreateInvoiceAsync(
            context.Message.OrderId,
            context.Message.Amount,
            context.Message.Currency);
    }
}
```

### When to Use Which

| Pattern | Use When | Latency | Coupling |
|---------|----------|---------|----------|
| HTTP (REST) | CRUD, simple request/response | Medium | Medium |
| gRPC | Internal, high-throughput, streaming | Low | Medium |
| Message queue | Fire-and-forget, event notification | High | Low |
| Event streaming (Kafka) | Audit log, replay, analytics | High | Very low |

---

## Data Ownership

### Database per Service

```
Payment Service → payment_db
  ├── payments
  ├── payment_intents
  └── refunds

Customer Service → customer_db
  ├── customers
  ├── payment_methods (tokenized)
  └── addresses

Billing Service → billing_db
  ├── invoices
  ├── subscriptions
  └── tax_records

Fraud Service → fraud_db
  ├── fraud_rules
  ├── risk_scores
  └── blocked_entities
```

### Cross-Service Queries (CQRS Materialized View)

```csharp
// When you need a joined view across services, build a read model
public class PaymentDashboardProjection
{
    // Consumes events from multiple services
    public async Task HandlePaymentProcessed(PaymentProcessedEvent evt)
    {
        await _readDb.UpsertAsync(new PaymentDashboardRow
        {
            PaymentId = evt.PaymentId,
            OrderId = evt.OrderId,
            Amount = evt.Amount,
            Status = evt.Status,
            ProcessedAt = evt.OccurredAt
        });
    }

    public async Task HandleCustomerUpdated(CustomerUpdatedEvent evt)
    {
        await _readDb.UpdateFieldsAsync(
            filter: r => r.CustomerId == evt.CustomerId,
            update: r => r.CustomerName = evt.Name);
    }
}
```

---

## Service Discovery

### Kubernetes-Native (DNS)

```yaml
# Services are discoverable via DNS: <service-name>.<namespace>.svc.cluster.local
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: commerce
spec:
  selector:
    app: payment-service
  ports:
    - name: http
      port: 8080
    - name: grpc
      port: 5001
```

```csharp
// Use Kubernetes DNS in configuration
{
  "Services": {
    "PaymentService": "http://payment-service.commerce.svc.cluster.local:8080",
    "BillingService": "http://billing-service.commerce.svc.cluster.local:8080"
  }
}
```

### Consul (Non-Kubernetes)

```csharp
builder.Services.AddSingleton<IConsulClient>(new ConsulClient(config =>
{
    config.Address = new Uri("http://consul:8500");
}));

// Register service on startup
public class ConsulRegistration : IHostedService
{
    public async Task StartAsync(CancellationToken ct)
    {
        await _consulClient.Agent.ServiceRegister(new AgentServiceRegistration
        {
            ID = $"payment-{Environment.MachineName}",
            Name = "payment-service",
            Port = 8080,
            Check = new AgentServiceCheck
            {
                HTTP = "http://localhost:8080/health",
                Interval = TimeSpan.FromSeconds(10)
            }
        });
    }
}
```

---

## Distributed Transactions

For comprehensive Saga pattern implementations (orchestration, choreography, and compensating transactions), see [Design Patterns — Saga Pattern](07-DESIGN-PATTERNS.md#saga-pattern). For event-driven saga patterns with state machines, see [Event-Driven Architecture](22-EVENT-DRIVEN-ARCHITECTURE.md).

Key considerations for microservices sagas:
- Use **orchestration** when a central service needs to coordinate the transaction flow
- Use **choreography** when services should remain fully decoupled via events
- Always persist saga state to survive process restarts
- Implement compensating transactions for every forward action

---

## Resilience Patterns

### Circuit Breaker per Downstream Service

```csharp
builder.Services.AddHttpClient<IPaymentServiceClient, PaymentServiceClient>()
    .AddResilienceHandler("payment-pipeline", builder =>
    {
        builder
            .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
            {
                FailureRatio = 0.5,
                SamplingDuration = TimeSpan.FromSeconds(30),
                MinimumThroughput = 10,
                BreakDuration = TimeSpan.FromSeconds(30)
            })
            .AddTimeout(TimeSpan.FromSeconds(5))
            .AddRetry(new HttpRetryStrategyOptions
            {
                MaxRetryAttempts = 2,
                Delay = TimeSpan.FromMilliseconds(500),
                BackoffType = DelayBackoffType.Exponential
            });
    });
```

### Bulkhead Isolation

```csharp
// Isolate resources per downstream service to prevent cascade failures
builder.Services.AddResiliencePipeline("stripe-bulkhead", pipeline =>
{
    pipeline.AddConcurrencyLimiter(new ConcurrencyLimiterOptions
    {
        PermitLimit = 50,
        QueueLimit = 20
    });
});

builder.Services.AddResiliencePipeline("paypal-bulkhead", pipeline =>
{
    pipeline.AddConcurrencyLimiter(new ConcurrencyLimiterOptions
    {
        PermitLimit = 30,
        QueueLimit = 10
    });
});
```

---

## Shared Nothing Architecture

### Token-Based Data Sharing (No Shared DB)

```csharp
// BAD: Services share a database
// Payment Service queries customer_db directly ❌

// GOOD: Services communicate through APIs and events ✅
public class PaymentService
{
    public async Task ProcessPayment(ProcessPaymentCommand command)
    {
        // Get customer info via API (not direct DB access)
        var customer = await _customerServiceClient.GetCustomerAsync(command.CustomerId);

        // Get payment method token (vault owns this data)
        var paymentMethod = await _customerServiceClient.GetPaymentMethodAsync(
            command.CustomerId, command.PaymentMethodToken);

        // Process with provider
        var result = await _stripeClient.ProcessAsync(paymentMethod.ProviderToken, command.Amount);
    }
}
```

### Shared Libraries (Contract Packages)

```xml
<!-- PaymentService.Contracts NuGet package -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <PackageId>Commerce.PaymentService.Contracts</PackageId>
    <Version>2.1.0</Version>
  </PropertyGroup>
</Project>
```

```
✅ Share: Event contracts, DTOs, client interfaces
❌ Never share: Entities, DbContext, business logic, configuration
```

---

## API Versioning

```csharp
// NuGet: Asp.Versioning.Http
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("X-Api-Version"));
});

[ApiController]
[ApiVersion(1)]
[Route("api/v{version:apiVersion}/payments")]
public class PaymentsV1Controller : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> ProcessPayment([FromBody] PaymentRequestV1 request) { ... }
}

[ApiController]
[ApiVersion(2)]
[Route("api/v{version:apiVersion}/payments")]
public class PaymentsV2Controller : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> ProcessPayment([FromBody] PaymentRequestV2 request) { ... }
}
```

---

## Deployment Patterns

### Independent Deployability

```yaml
# Each service has its own CI/CD pipeline
# .github/workflows/payment-service.yml
name: Payment Service CI/CD

on:
  push:
    paths:
      - 'services/payment-service/**'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: dotnet build services/payment-service/

      - name: Test
        run: dotnet test services/payment-service/

      - name: Build Docker Image
        run: |
          docker build -t payment-service:${{ github.sha }} \
            services/payment-service/

      - name: Deploy to Kubernetes
        run: |
          helm upgrade payment-service charts/payment-service \
            --set image.tag=${{ github.sha }} \
            --namespace commerce
```

### Blue-Green Deployment

```yaml
# Two identical environments, switch traffic after validation
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment-service
    version: green  # Switch to blue after validating new version
  ports:
    - port: 8080
```

### Canary Deployment with Istio

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
    - payment-service
  http:
    - route:
        - destination:
            host: payment-service
            subset: stable
          weight: 90
        - destination:
            host: payment-service
            subset: canary
          weight: 10
```

---

## Testing Microservices

### Contract Testing (Pact)

```csharp
// Consumer test: Billing Service expects this from Payment Service
[TestClass]
public class PaymentServiceContractTests
{
    private readonly IPactBuilderV4 _pactBuilder;

    [TestMethod]
    public async Task GetPaymentStatus_ReturnsExpectedFormat()
    {
        _pactBuilder
            .UponReceiving("a request for payment status")
            .WithRequest(HttpMethod.Get, "/payments/pay-123/status")
            .WillRespond()
            .WithStatus(HttpStatusCode.OK)
            .WithJsonBody(new
            {
                paymentId = "pay-123",
                status = Match.Regex("completed|pending|failed", "completed"),
                amount = Match.Decimal(99.99m),
                currency = Match.Regex("[A-Z]{3}", "USD")
            });

        await _pactBuilder.VerifyAsync(async ctx =>
        {
            var client = new PaymentServiceClient(new HttpClient { BaseAddress = ctx.MockServerUri });
            var result = await client.GetPaymentStatusAsync("pay-123");

            Assert.AreEqual("completed", result.Status);
        });
    }
}
```

### Integration Testing with Testcontainers

```csharp
[TestClass]
public class PaymentServiceIntegrationTests : IAsyncLifetime
{
    private PostgreSqlContainer _postgres;
    private RabbitMqContainer _rabbitmq;
    private WebApplicationFactory<Program> _factory;

    public async Task InitializeAsync()
    {
        _postgres = new PostgreSqlBuilder()
            .WithDatabase("payment_test")
            .Build();
        await _postgres.StartAsync();

        _rabbitmq = new RabbitMqBuilder().Build();
        await _rabbitmq.StartAsync();

        _factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    services.AddDbContext<PaymentDbContext>(opt =>
                        opt.UseNpgsql(_postgres.GetConnectionString()));
                });
            });
    }

    [TestMethod]
    public async Task ProcessPayment_PublishesEvent()
    {
        var client = _factory.CreateClient();

        var response = await client.PostAsJsonAsync("/payments", new
        {
            orderId = "order-1",
            amount = 49.99,
            currency = "USD",
            paymentMethodToken = "tok_visa"
        });

        Assert.AreEqual(HttpStatusCode.Created, response.StatusCode);
        // Verify event was published to RabbitMQ...
    }

    public async Task DisposeAsync()
    {
        await _postgres.DisposeAsync();
        await _rabbitmq.DisposeAsync();
    }
}
```

---

## Microservices Checklist

- [ ] Each service owns its data (no shared databases)
- [ ] Services communicate via APIs or events (not direct DB queries)
- [ ] Contract packages for shared DTOs and event definitions
- [ ] API versioning for backward-compatible changes
- [ ] Circuit breakers and bulkheads per downstream dependency
- [ ] Independent CI/CD pipelines per service
- [ ] Health checks and readiness probes
- [ ] Distributed tracing with correlation IDs
- [ ] Contract tests between service boundaries
- [ ] Saga pattern for distributed transactions
- [ ] Rate limiting at the API gateway

## Next Steps

- [Event-Driven Architecture](22-EVENT-DRIVEN-ARCHITECTURE.md)
- [Architecture Patterns](01-ARCHITECTURE-PATTERNS.md)
- [Design Patterns](07-DESIGN-PATTERNS.md)
- [Kubernetes Deployment](20-KUBERNETES-DEPLOYMENT.md)
