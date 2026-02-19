# Microservices Testing Strategies

## Table of Contents

1. [Overview](#overview)
2. [Testing Pyramid for Microservices](#testing-pyramid-for-microservices)
3. [Unit Testing](#unit-testing)
4. [Integration Testing](#integration-testing)
5. [Contract Testing](#contract-testing)
6. [End-to-End Testing](#end-to-end-testing)
7. [Chaos Engineering](#chaos-engineering)
8. [Performance Testing](#performance-testing)
9. [Testing in Production](#testing-in-production)
10. [Best Practices](#best-practices)
11. [Next Steps](#next-steps)

## Overview

Testing microservices is more complex than testing monoliths because of the distributed nature of the system. This document covers the testing strategies, tools, and patterns that enable confidence in microservices deployments.

### Target Audience

- Developers writing tests for microservices
- QA engineers designing test strategies
- SREs building reliability testing frameworks

### Scope

- Testing pyramid adapted for microservices
- Unit, integration, contract, and end-to-end testing
- Contract testing with Pact and Spring Cloud Contract
- Chaos engineering for resilience validation
- Performance and load testing patterns

## Testing Pyramid for Microservices

The testing pyramid for microservices differs from monolithic applications. Contract tests replace many of the integration tests that would otherwise be needed.

```
                    ╱╲
                   ╱  ╲
                  ╱ E2E╲           Few — expensive, slow, brittle
                 ╱──────╲
                ╱        ╲
               ╱ Contract ╲        Medium — validate service agreements
              ╱────────────╲
             ╱              ╲
            ╱  Integration   ╲     Medium — test with real dependencies
           ╱──────────────────╲
          ╱                    ╲
         ╱     Unit Tests       ╲  Many — fast, isolated, cheap
        ╱────────────────────────╲
```

| Level | Scope | Speed | Confidence | Volume |
|---|---|---|---|---|
| **Unit** | Single function/class | Very fast | Low (isolated) | Many |
| **Integration** | Service + its dependencies (DB, cache) | Medium | Medium | Medium |
| **Contract** | API agreements between services | Fast | High for interfaces | Medium |
| **E2E** | Full system flow | Slow | High (but brittle) | Few |

## Unit Testing

Unit tests verify individual functions, methods, or classes in isolation. External dependencies are mocked or stubbed.

### What to Unit Test

- Business logic and domain rules
- Data transformation and validation
- Error handling paths
- Edge cases and boundary conditions

### Example

```go
// order_service_test.go
func TestCalculateOrderTotal(t *testing.T) {
    items := []OrderItem{
        {ProductID: "p1", Quantity: 2, Price: 29.99},
        {ProductID: "p2", Quantity: 1, Price: 49.99},
    }

    total := CalculateOrderTotal(items)

    assert.Equal(t, 109.97, total)
}

func TestCalculateOrderTotal_EmptyOrder(t *testing.T) {
    items := []OrderItem{}

    total := CalculateOrderTotal(items)

    assert.Equal(t, 0.0, total)
}

func TestCreateOrder_InvalidCustomerID(t *testing.T) {
    svc := NewOrderService(mockRepo, mockPayment)

    _, err := svc.CreateOrder("", items)

    assert.ErrorIs(t, err, ErrInvalidCustomerID)
}
```

## Integration Testing

Integration tests verify that a service works correctly with its real dependencies (database, message broker, cache) using test containers or in-memory instances.

### Test Containers

```go
// order_repository_integration_test.go
func TestOrderRepository_CreateAndFind(t *testing.T) {
    // Start a PostgreSQL test container
    ctx := context.Background()
    pgContainer, _ := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:16-alpine"),
        postgres.WithDatabase("testdb"),
    )
    defer pgContainer.Terminate(ctx)

    connStr, _ := pgContainer.ConnectionString(ctx)
    repo := NewOrderRepository(connStr)

    // Test create
    order := Order{CustomerID: "cust-1", Total: 99.99}
    created, err := repo.Create(ctx, order)
    assert.NoError(t, err)

    // Test find
    found, err := repo.FindByID(ctx, created.ID)
    assert.NoError(t, err)
    assert.Equal(t, order.CustomerID, found.CustomerID)
    assert.Equal(t, order.Total, found.Total)
}
```

### What to Integration Test

- Database queries and migrations
- Message broker produce/consume
- Cache read/write operations
- External API client behavior (with test doubles or sandboxes)

## Contract Testing

Contract testing verifies that the API between a consumer and a provider conforms to an agreed-upon contract. This replaces many expensive end-to-end tests.

### How Contract Testing Works

```
1. Consumer writes a contract test
   (what it expects from the provider)

  ┌────────────────┐                     ┌────────────────┐
  │ Order Service  │     Contract        │ Payment Service│
  │ (Consumer)     │◀───────────────────▶│ (Provider)     │
  │                │     Agreement       │                │
  │ "When I POST   │                     │ "I return a    │
  │  /payments     │                     │  PaymentResult │
  │  with {...}    │                     │  with status,  │
  │  I expect a    │                     │  id, amount"   │
  │  201 response  │                     │                │
  │  with {...}"   │                     │                │
  └────────────────┘                     └────────────────┘

2. Contract is shared (via a broker or artifact)

3. Provider verifies it can fulfill the contract

4. Both sides run contract tests in their own CI pipelines
```

### Contract Testing Tools

| Tool | Approach | Languages |
|---|---|---|
| **Pact** | Consumer-driven contracts | Java, .NET, Go, JS, Python, Ruby |
| **Spring Cloud Contract** | Producer-driven contracts | Java/Spring |
| **Specmatic** | OpenAPI-based contracts | Language-agnostic |

### Consumer-Driven Contract Example (Pact)

```javascript
// Consumer test (Order Service)
describe("Payment Service Contract", () => {
  it("creates a payment successfully", async () => {
    await provider
      .given("a valid payment request")
      .uponReceiving("a POST request to /payments")
      .withRequest({
        method: "POST",
        path: "/payments",
        body: {
          orderId: "order-123",
          amount: 99.99,
          currency: "USD"
        }
      })
      .willRespondWith({
        status: 201,
        body: {
          id: like("pay-abc"),
          status: "COMPLETED",
          amount: 99.99
        }
      });

    // Execute the consumer code against the mock provider
    const result = await paymentClient.createPayment({
      orderId: "order-123",
      amount: 99.99,
      currency: "USD"
    });

    expect(result.status).toBe("COMPLETED");
  });
});
```

## End-to-End Testing

End-to-end tests validate complete user flows across multiple services. They are the most expensive and brittle tests, so keep them minimal.

### When to Use E2E Tests

- ✅ Critical business flows (checkout, payment, registration)
- ✅ Smoke tests after deployment to verify the system is functional
- ❌ Do not use E2E tests for edge cases — use unit and contract tests instead
- ❌ Do not write hundreds of E2E tests — they are slow and fragile

### E2E Test Strategy

```
Keep E2E tests focused on critical paths:

1. User registration → login → browse catalog → add to cart → checkout → payment
2. Order placement → payment processing → inventory update → shipping notification
3. User login → profile update → password change → logout

Total E2E tests: 5–10 critical user journeys (not 500 variations)
```

## Chaos Engineering

Chaos engineering deliberately introduces failures into a system to verify that it handles them gracefully. It validates resilience patterns (circuit breakers, retries, fallbacks) under real conditions.

### Principles of Chaos Engineering

1. **Define steady state** — What does "normal" look like? (latency, error rate, throughput)
2. **Hypothesize** — "If Service X goes down, the system will degrade gracefully"
3. **Introduce failure** — Kill a service, add latency, saturate CPU
4. **Observe** — Does the system behave as expected?
5. **Fix** — Address any unexpected failures

### Chaos Experiments

| Experiment | What to Inject | What to Observe |
|---|---|---|
| **Service failure** | Kill a service instance | Does the circuit breaker open? Do retries work? |
| **Network latency** | Add 500ms delay between services | Do timeouts trigger correctly? |
| **Database failure** | Block database connections | Does the fallback return cached data? |
| **CPU stress** | Saturate CPU on a node | Does auto-scaling trigger? |
| **Disk full** | Fill disk on a node | Does logging and alerting continue? |
| **DNS failure** | Block DNS resolution | Does service discovery fail gracefully? |

### Chaos Engineering Tools

| Tool | Features |
|---|---|
| **Chaos Monkey** (Netflix) | Randomly terminates instances in production |
| **Litmus** (CNCF) | Kubernetes-native chaos experiments |
| **Gremlin** | SaaS platform for chaos engineering |
| **Chaos Mesh** | Kubernetes-native, CRD-based experiments |
| **Toxiproxy** (Shopify) | TCP proxy for simulating network conditions |

## Performance Testing

### Types of Performance Tests

| Type | Purpose | Duration |
|---|---|---|
| **Load test** | Verify performance under expected load | Minutes–hours |
| **Stress test** | Find breaking points under extreme load | Minutes |
| **Soak test** | Detect memory leaks and degradation over time | Hours–days |
| **Spike test** | Verify behavior during sudden traffic surges | Minutes |

### Performance Testing Tools

| Tool | Type | Best For |
|---|---|---|
| **k6** | Load testing | Developer-friendly, scriptable in JS |
| **Gatling** | Load testing | Scala-based, detailed reports |
| **Locust** | Load testing | Python-based, distributed testing |
| **Apache JMeter** | Load testing | GUI-based, protocol support |

## Testing in Production

### Techniques

| Technique | Description |
|---|---|
| **Canary testing** | Route a small % of production traffic to the new version |
| **Shadow testing** | Duplicate production traffic to a test instance; compare responses |
| **Synthetic monitoring** | Automated scripts that simulate user actions in production |
| **Feature flags** | Enable features for specific users or groups in production |
| **Observability-driven** | Deploy and monitor; roll back if SLOs are violated |

## Best Practices

### Testing Strategy

- ✅ Invest heavily in unit tests (fast feedback, low cost)
- ✅ Use contract tests to validate inter-service APIs
- ✅ Run integration tests with test containers (real dependencies, not mocks)
- ✅ Keep E2E tests minimal — 5–10 critical paths, not hundreds
- ✅ Practice chaos engineering in staging before production
- ❌ Do not rely solely on E2E tests — they are too slow and brittle
- ❌ Do not mock everything — integration tests with real databases are valuable

### CI/CD Integration

- ✅ Unit + contract tests run on every commit (fast feedback)
- ✅ Integration tests run on every PR merge (medium feedback)
- ✅ E2E tests run as post-deployment smoke tests
- ✅ Performance tests run on a schedule (weekly or before major releases)
- ✅ Chaos experiments run in staging regularly

## Next Steps

Continue to [Best Practices](11-BEST-PRACTICES.md) for a comprehensive production readiness checklist, team organization patterns, and documentation standards for microservices.
