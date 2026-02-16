# Payment Systems Learning Path

A structured, self-paced training guide to mastering payment system development in .NET. Each phase builds on the previous one, progressing from fundamentals to production-grade expertise.

> **Time Estimate:** 8–12 weeks at ~5 hours/week. Adjust pace to your experience level.

---

## How to Use This Guide

1. **Follow the phases in order** — each one builds on prior knowledge
2. **Read the linked documents** — they contain the detailed content
3. **Complete the exercises** — hands-on practice solidifies understanding
4. **Check yourself** — use the knowledge checks before moving on
5. **Build the capstone project** — ties everything together

---

## Phase 1: Foundations (Week 1–2)

### Learning Objectives

- Understand the end-to-end payment lifecycle
- Learn core payment concepts (authorization, capture, settlement)
- Grasp the layered architecture approach for payment systems
- Set up a development environment with Stripe sandbox

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | Payment flow, tokenization, provider landscape |
| 2 | [01-ARCHITECTURE-PATTERNS](01-ARCHITECTURE-PATTERNS.md) | Clean Architecture, Adapter pattern, domain modeling |

### Hands-On Exercise

**Set up your workspace:**

```bash
dotnet new webapi -n PaymentSystem
cd PaymentSystem
dotnet add package Stripe.net
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package FluentValidation
```

**Create the project structure:**

```
PaymentSystem/
├── Domain/
│   ├── Entities/
│   ├── Enums/
│   └── Events/
├── Application/
│   ├── Interfaces/
│   ├── Services/
│   └── DTOs/
├── Infrastructure/
│   ├── Gateways/
│   ├── Repositories/
│   └── Data/
└── Api/
    ├── Controllers/
    └── Program.cs
```

**Build a `Payment` domain entity** with rich behavior:

```csharp
public class Payment
{
    public Guid Id { get; private set; }
    public string OrderId { get; private set; }
    public decimal Amount { get; private set; }
    public string Currency { get; private set; }
    public PaymentStatus Status { get; private set; }
    public string ProviderTransactionId { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? CompletedAt { get; private set; }

    public static Payment Create(string orderId, decimal amount, string currency)
    {
        // TODO: Add validation (orderId required, amount > 0, currency not empty)
        // TODO: Initialize with Pending status and set CreatedAt to UTC now
        throw new NotImplementedException();
    }

    public void MarkAsProcessing(string providerTransactionId)
    {
        // TODO: Enforce state transition — only Pending → Processing is valid
        throw new NotImplementedException();
    }

    public void MarkAsCompleted()
    {
        // TODO: Enforce state transition — only Processing → Completed is valid
        // TODO: Set CompletedAt timestamp
        throw new NotImplementedException();
    }

    public void MarkAsFailed(string errorMessage)
    {
        // TODO: Any status can transition to Failed
        throw new NotImplementedException();
    }
}
```

### Knowledge Check

- [ ] Can you explain the difference between authorization and capture?
- [ ] Why does the `Payment` entity use a private constructor and factory method?
- [ ] What is the Adapter pattern and why is it critical for multi-provider support?
- [ ] Draw the state diagram for a payment (Pending → Processing → Completed/Failed → Refunded)

---

## Phase 2: Core Patterns (Week 3–4)

### Learning Objectives

- Implement the Repository pattern for payment data access
- Use the Strategy pattern for provider routing and A/B testing
- Design robust error handling with typed exceptions and the Result pattern
- Model a payment-optimized database schema

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [02-REPOSITORY-PATTERN](02-REPOSITORY-PATTERN.md) | Repository, Unit of Work, Specification pattern |
| 2 | [03-STRATEGY-PATTERN](03-STRATEGY-PATTERN.md) | Provider routing, A/B testing, fallback chains |
| 3 | [05-ERROR-HANDLING](05-ERROR-HANDLING.md) | Exception hierarchy, Result pattern, retry logic |
| 4 | [06-DATABASE-DESIGN](06-DATABASE-DESIGN.md) | Schemas, indexing, EF Core configuration |

### Hands-On Exercises

1. **Repository layer** — Implement `IPaymentRepository` with EF Core (CRUD + query by OrderId, by ProviderTransactionId)
2. **Provider abstraction** — Create `IPaymentGateway` interface; implement `StripeGateway` using Payment Intents API
3. **Error handling** — Build a `PaymentException` hierarchy: `PaymentDeclinedException`, `PaymentGatewayException`, `InsufficientFundsException`
4. **Database** — Write EF Core `OnModelCreating` config with proper precision for amounts (`HasPrecision(18, 2)`), indexes on OrderId and Status

### Knowledge Check

- [ ] When would you use Unit of Work vs. calling `SaveChangesAsync()` per repository method?
- [ ] How does the Strategy pattern enable switching providers without changing business logic?
- [ ] Why should you use `HasPrecision(18, 2)` for monetary columns instead of the default?
- [ ] What is the benefit of an idempotency key when calling Stripe?

---

## Phase 3: Integration & Webhooks (Week 5–6)

### Learning Objectives

- Integrate with Stripe and PayPal APIs in production-grade code
- Implement reliable webhook processing with signature verification
- Handle common payment scenarios (checkout, subscriptions, marketplace)

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [16-STRIPE-INTEGRATION](16-STRIPE-INTEGRATION.md) | Payment Intents, Connect, Checkout Sessions |
| 2 | [17-PAYPAL-INTEGRATION](17-PAYPAL-INTEGRATION.md) | Orders API, Subscriptions, Payouts |
| 3 | [04-WEBHOOK-PATTERNS](04-WEBHOOK-PATTERNS.md) | Signature verification, idempotency, async processing |
| 4 | [18-COMMON-SCENARIOS](18-COMMON-SCENARIOS.md) | Cart checkout, recurring billing, split payments |
| 5 | [24-PAYMENT-METHODS-AND-FLOWS](24-PAYMENT-METHODS-AND-FLOWS.md) | Cards, wallets, bank transfers, BNPL, regional methods |

### Hands-On Exercises

1. **Stripe integration** — Implement `StripeGateway.ProcessPaymentAsync()` using `PaymentIntentService`, with idempotency keys and proper error mapping
2. **Webhook endpoint** — Build `WebhooksController` that verifies Stripe signatures, deduplicates events, and queues processing:

```csharp
[HttpPost("stripe")]
public async Task<IActionResult> HandleStripeWebhook()
{
    var json = await new StreamReader(HttpContext.Request.Body).ReadToEndAsync();
    var signatureHeader = Request.Headers["Stripe-Signature"];

    // TODO: Verify signature using EventUtility.ConstructEvent()
    // TODO: Check for duplicate events (store EventId in WebhookEvents table)
    // TODO: Queue for background processing — don't process inline
    // TODO: Return 200 OK immediately (Stripe retries on non-2xx)
}
```

3. **Local webhook testing** — Set up Stripe CLI forwarding:

```bash
stripe listen --forward-to https://localhost:5001/api/webhooks/stripe
stripe trigger payment_intent.succeeded
```

4. **Multi-provider** — Add a second gateway implementation (`PayPalGateway`) and wire up the Strategy pattern to route between them

### Knowledge Check

- [ ] Why must you return 200 OK to webhooks before doing heavy processing?
- [ ] What happens if you don't verify the webhook signature?
- [ ] How do idempotency keys prevent duplicate charges?
- [ ] What is the difference between Stripe Payment Intents and Checkout Sessions?

---

## Phase 4: Security & Testing (Week 7–8)

### Learning Objectives

- Implement PCI DSS-compliant payment handling
- Secure API keys, secrets, and sensitive data
- Build comprehensive test suites (unit, integration, E2E)
- Use sandbox/test cards effectively

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [15-SECURITY-PATTERNS-PRACTICES](15-SECURITY-PATTERNS-PRACTICES.md) | PCI DSS, encryption, fraud prevention, input validation |
| 2 | [13-TEST-CARDS-SANDBOX](13-TEST-CARDS-SANDBOX.md) | Test card numbers, sandbox setup, provider registration |
| 3 | [14-E2E-INTEGRATION-TESTING](14-E2E-INTEGRATION-TESTING.md) | Unit tests, integration tests, E2E test strategies |

### Hands-On Exercises

1. **Security audit** — Review your code against this checklist:
   - [ ] API keys in environment variables or Key Vault (never in `appsettings.json`)
   - [ ] No raw card data touches your server (use Stripe.js / PayPal JS SDK)
   - [ ] HTTPS enforced; HSTS headers set
   - [ ] Webhook signatures verified
   - [ ] Input validation on all endpoints (amount > 0, currency ISO 4217, etc.)
   - [ ] Structured logging without PII in log messages
   - [ ] Rate limiting on payment endpoints

2. **Unit tests** — Test `PaymentService` with mocked `IPaymentGateway` and `IPaymentRepository`:

```csharp
[Fact]
public async Task ProcessPayment_WhenGatewaySucceeds_ReturnsSuccessAndUpdatesStatus()
{
    // Arrange: mock gateway to return success
    // Act: call ProcessPaymentAsync
    // Assert: payment status is Completed, repository was updated
}

[Fact]
public async Task ProcessPayment_WhenGatewayFails_ReturnsFailureAndMarksPaymentFailed()
{
    // Arrange: mock gateway to return failure
    // Act: call ProcessPaymentAsync
    // Assert: payment status is Failed, error message is set
}
```

3. **Integration tests** — Test against Stripe sandbox with test cards:

```csharp
"4242424242424242" // Visa — succeeds
"4000000000000002" // Visa — declined
"4000000000009995" // Visa — insufficient funds
"4000002500003155" // Visa — requires 3D Secure
```

4. **Webhook tests** — Write integration tests that simulate incoming webhook payloads and verify event processing

### Knowledge Check

- [ ] What PCI DSS SAQ level applies when you use Stripe.js (no card data on your server)?
- [ ] Why should you never log full card numbers, even in development?
- [ ] What's the difference between a unit test with mocked dependencies and an integration test against the sandbox?
- [ ] How do you test 3D Secure flows in a sandbox environment?

---

## Phase 5: Advanced Architecture (Week 9–10)

### Learning Objectives

- Apply CQRS and Event Sourcing to payment systems
- Design event-driven architectures with domain events and message brokers
- Understand microservices decomposition for payment domains
- Master advanced design patterns (Saga, Circuit Breaker, Outbox)

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [11-CQRS-EVENT-SOURCING](11-CQRS-EVENT-SOURCING.md) | Command/Query separation, event stores, Outbox pattern |
| 2 | [22-EVENT-DRIVEN-ARCHITECTURE](22-EVENT-DRIVEN-ARCHITECTURE.md) | Domain events, Saga orchestration, message brokers |
| 3 | [23-MICROSERVICES-PATTERNS](23-MICROSERVICES-PATTERNS.md) | Service boundaries, API gateway, contract testing |
| 4 | [07-DESIGN-PATTERNS](07-DESIGN-PATTERNS.md) | Browse the 39-pattern catalog — focus on Saga, Circuit Breaker, Outbox, Sharding |

### Hands-On Exercises

1. **Domain events** — Publish `PaymentCompletedEvent` from your `PaymentService` using MediatR:

```csharp
public record PaymentCompletedEvent(
    Guid PaymentId,
    string OrderId,
    decimal Amount,
    string Currency,
    DateTime CompletedAt) : INotification;
```

2. **Event handlers** — Create handlers for order fulfillment, email notification, and analytics — each reacting independently to the same event
3. **Circuit Breaker** — Add Polly circuit breaker around your gateway calls:

```csharp
builder.Services.AddHttpClient<StripeGateway>()
    .AddTransientHttpErrorPolicy(p =>
        p.CircuitBreakerAsync(5, TimeSpan.FromSeconds(30)));
```

4. **Outbox pattern** — Implement transactional outbox to guarantee event delivery alongside database commits

### Knowledge Check

- [ ] When is Event Sourcing beneficial vs. overkill for a payment system?
- [ ] What problem does the Outbox pattern solve that simple event publishing doesn't?
- [ ] How does a Saga differ from a distributed transaction?
- [ ] What are sensible Circuit Breaker thresholds for payment gateway calls?

---

## Phase 6: Production Operations (Week 11–12)

### Learning Objectives

- Deploy payment services to cloud infrastructure (AWS, Azure, GCP)
- Implement observability with metrics, logging, and tracing
- Optimize performance and costs
- Deploy and operate on Kubernetes

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [09-CLOUD-NATIVE-AWS](09-CLOUD-NATIVE-AWS.md) | Lambda, API Gateway, DynamoDB, Secrets Manager |
| 2 | [10-MULTI-CLOUD-ALTERNATIVES](10-MULTI-CLOUD-ALTERNATIVES.md) | Azure, GCP, multi-cloud strategies |
| 3 | [19-MONITORING-ALERTING](19-MONITORING-ALERTING.md) | OpenTelemetry, dashboards, incident response |
| 4 | [20-KUBERNETES-DEPLOYMENT](20-KUBERNETES-DEPLOYMENT.md) | Manifests, Helm charts, network policies |
| 5 | [08-PERFORMANCE-OPTIMIZATION](08-PERFORMANCE-OPTIMIZATION.md) | Caching, connection pooling, async processing |
| 6 | [21-COST-OPTIMIZATION](21-COST-OPTIMIZATION.md) | Fee optimization, smart routing, infrastructure costs |

### Hands-On Exercises

1. **Structured logging** — Add OpenTelemetry traces to your payment flow:

```csharp
using var activity = ActivitySource.StartActivity("ProcessPayment");
activity?.SetTag("payment.order_id", request.OrderId);
activity?.SetTag("payment.amount", request.Amount);
activity?.SetTag("payment.currency", request.Currency);
```

2. **Health checks** — Add health check endpoints for your database and Stripe connectivity
3. **Kubernetes manifests** — Write deployment, service, and secrets manifests for your payment API
4. **Performance** — Add response caching for `GET /api/payments/{id}` and connection pooling for your database

### Knowledge Check

- [ ] What are the critical metrics to monitor for a payment system? (hint: success rate, latency p99, error rate by type)
- [ ] Why should you store Stripe API keys in Kubernetes Secrets or a cloud secrets manager rather than ConfigMaps?
- [ ] How does connection pooling improve database performance under load?
- [ ] What is the cost difference between processing $10,000/month through Stripe vs. routing partially through a cheaper provider?

---

## Phase 7: Specialized Topics (Ongoing)

### Reading

These documents cover niche topics. Read them as your project requires:

| Document | When to Read |
|----------|-------------|
| [12-TAX-AND-MULTI-PROVIDER](12-TAX-AND-MULTI-PROVIDER.md) | When you need tax calculation or multi-gateway routing |
| [07-DESIGN-PATTERNS](07-DESIGN-PATTERNS.md) | Full 39-pattern reference — revisit as you encounter specific problems |
| [24-PAYMENT-METHODS-AND-FLOWS](24-PAYMENT-METHODS-AND-FLOWS.md) | When adding new payment methods or expanding to new regions |

---

## Capstone Project

Build a complete **multi-provider payment API** that demonstrates everything you've learned:

### Requirements

1. **Two payment providers** (Stripe + PayPal) with Strategy-based routing
2. **Domain-driven design** with rich `Payment` entity and state machine
3. **Repository pattern** with EF Core and proper database schema
4. **Webhook endpoints** for both providers with signature verification and idempotent processing
5. **Error handling** with typed exceptions, Result pattern, and Polly retry/circuit breaker
6. **Security** — no raw card data, secrets in Key Vault, input validation, rate limiting
7. **Tests** — unit tests for business logic, integration tests against sandbox
8. **Observability** — structured logging, health checks, OpenTelemetry traces
9. **Refund flow** — full and partial refunds through both providers

### Evaluation Criteria

| Area | What to Verify |
|------|---------------|
| Architecture | Clean separation of Domain, Application, Infrastructure, API layers |
| Correctness | Payment state transitions enforced; idempotent operations |
| Security | PCI-compliant; no secrets in code; webhook signatures verified |
| Resilience | Circuit breaker on gateway calls; retry with exponential backoff |
| Testability | >80% coverage on business logic; integration tests pass against sandbox |
| Observability | Structured logs with correlation IDs; health check endpoints |

---

## Recommended Tools

| Tool | Purpose |
|------|---------|
| [Stripe CLI](https://stripe.com/docs/stripe-cli) | Webhook forwarding, event triggering |
| [ngrok](https://ngrok.com/) | Expose local endpoints for webhook testing |
| [Polly](https://github.com/App-vNext/Polly) | Resilience policies (retry, circuit breaker) |
| [MediatR](https://github.com/jbogard/MediatR) | In-process domain event publishing |
| [FluentValidation](https://docs.fluentvalidation.net/) | Request validation |
| [Seq](https://datalust.co/seq) or [Jaeger](https://www.jaegertracing.io/) | Local log/trace viewing |

---

## Quick Reference: Document Map

| # | Document | Phase |
|---|----------|-------|
| 00 | [OVERVIEW](00-OVERVIEW.md) | 1 |
| 01 | [ARCHITECTURE-PATTERNS](01-ARCHITECTURE-PATTERNS.md) | 1 |
| 02 | [REPOSITORY-PATTERN](02-REPOSITORY-PATTERN.md) | 2 |
| 03 | [STRATEGY-PATTERN](03-STRATEGY-PATTERN.md) | 2 |
| 04 | [WEBHOOK-PATTERNS](04-WEBHOOK-PATTERNS.md) | 3 |
| 05 | [ERROR-HANDLING](05-ERROR-HANDLING.md) | 2 |
| 06 | [DATABASE-DESIGN](06-DATABASE-DESIGN.md) | 2 |
| 07 | [DESIGN-PATTERNS](07-DESIGN-PATTERNS.md) | 5, 7 |
| 08 | [PERFORMANCE-OPTIMIZATION](08-PERFORMANCE-OPTIMIZATION.md) | 6 |
| 09 | [CLOUD-NATIVE-AWS](09-CLOUD-NATIVE-AWS.md) | 6 |
| 10 | [MULTI-CLOUD-ALTERNATIVES](10-MULTI-CLOUD-ALTERNATIVES.md) | 6 |
| 11 | [CQRS-EVENT-SOURCING](11-CQRS-EVENT-SOURCING.md) | 5 |
| 12 | [TAX-AND-MULTI-PROVIDER](12-TAX-AND-MULTI-PROVIDER.md) | 7 |
| 13 | [TEST-CARDS-SANDBOX](13-TEST-CARDS-SANDBOX.md) | 4 |
| 14 | [E2E-INTEGRATION-TESTING](14-E2E-INTEGRATION-TESTING.md) | 4 |
| 15 | [SECURITY-PATTERNS-PRACTICES](15-SECURITY-PATTERNS-PRACTICES.md) | 4 |
| 16 | [STRIPE-INTEGRATION](16-STRIPE-INTEGRATION.md) | 3 |
| 17 | [PAYPAL-INTEGRATION](17-PAYPAL-INTEGRATION.md) | 3 |
| 18 | [COMMON-SCENARIOS](18-COMMON-SCENARIOS.md) | 3 |
| 19 | [MONITORING-ALERTING](19-MONITORING-ALERTING.md) | 6 |
| 20 | [KUBERNETES-DEPLOYMENT](20-KUBERNETES-DEPLOYMENT.md) | 6 |
| 21 | [COST-OPTIMIZATION](21-COST-OPTIMIZATION.md) | 6 |
| 22 | [EVENT-DRIVEN-ARCHITECTURE](22-EVENT-DRIVEN-ARCHITECTURE.md) | 5 |
| 23 | [MICROSERVICES-PATTERNS](23-MICROSERVICES-PATTERNS.md) | 5 |
| 24 | [PAYMENT-METHODS-AND-FLOWS](24-PAYMENT-METHODS-AND-FLOWS.md) | 3, 7 |
