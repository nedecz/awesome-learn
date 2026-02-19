# Comprehensive Testing Guide for Payment Systems

## Overview

Complete guide to testing payment systems including unit tests, integration tests, end-to-end tests, UI automation, API testing, and production testing strategies. Essential for ensuring payment reliability and preventing costly bugs.

> **Note**: This document consolidates all testing guidance into a single comprehensive reference.

## Table of Contents

1. [Testing Pyramid for Payments](#testing-pyramid-for-payments)
2. [Unit Testing](#unit-testing)
3. [Integration Testing](#integration-testing)
4. [End-to-End Testing](#end-to-end-testing)
5. [UI Automation Testing](#ui-automation-testing)
6. [API Testing](#api-testing)
7. [Contract Testing](#contract-testing)
8. [Performance Testing](#performance-testing)
9. [Security Testing](#security-testing)
10. [Production Testing](#production-testing)
11. [Testing Infrastructure](#testing-infrastructure)

---

## Testing Pyramid for Payments

### Payment System Testing Strategy

```
                    ┌─────────────────┐
                    │   E2E Tests     │ ← 10%
                    │  (Slow, Brittle) │
                    └─────────────────┘
                  ┌───────────────────────┐
                  │  Integration Tests    │ ← 30%
                  │  (Medium Speed)       │
                  └───────────────────────┘
              ┌─────────────────────────────────┐
              │      Unit Tests                 │ ← 60%
              │   (Fast, Reliable)              │
              └─────────────────────────────────┘
```

### Test Distribution

| Test Type | Percentage | Purpose | Speed |
|-----------|-----------|---------|-------|
| **Unit Tests** | 60% | Individual components | Fast (ms) |
| **Integration Tests** | 30% | Service interactions | Medium (seconds) |
| **E2E Tests** | 10% | Full user workflows | Slow (minutes) |

---

## Unit Testing

### Testing Payment Services

```csharp
using Microsoft.VisualStudio.TestTools.UnitTesting;
using Moq;

[TestClass]
public class PaymentServiceTests
{
    private Mock<IPaymentGateway> _mockGateway;
    private Mock<IPaymentRepository> _mockRepository;
    private Mock<ILogger<PaymentService>> _mockLogger;
    private PaymentService _sut;

    [TestInitialize]
    public void Setup()
    {
        _mockGateway = new Mock<IPaymentGateway>();
        _mockRepository = new Mock<IPaymentRepository>();
        _mockLogger = new Mock<ILogger<PaymentService>>();
        _sut = new PaymentService(_mockGateway.Object, _mockRepository.Object, _mockLogger.Object);
    }

    [TestMethod]
    public async Task ProcessPayment_ValidRequest_ReturnsSuccess()
    {
        // Arrange
        var request = new PaymentRequest
        {
            OrderId = "order-123",
            Amount = 100.00m,
            Currency = "USD",
            PaymentMethodId = "pm_test_123"
        };

        _mockGateway
            .Setup(g => g.ProcessPaymentAsync(It.IsAny<PaymentRequest>()))
            .ReturnsAsync(new GatewayResult { IsSuccess = true, TransactionId = "txn_123" });

        // Act
        var result = await _sut.ProcessPaymentAsync(request);

        // Assert
        Assert.IsTrue(result.IsSuccess);
        Assert.AreEqual("txn_123", result.TransactionId);
        _mockRepository.Verify(
            r => r.AddAsync(It.Is<Payment>(p => p.OrderId == "order-123")), Times.Once);
    }

    [TestMethod]
    public async Task ProcessPayment_GatewayFailure_ReturnsFail()
    {
        var request = new PaymentRequest { OrderId = "order-123", Amount = 100.00m, Currency = "USD" };

        _mockGateway
            .Setup(g => g.ProcessPaymentAsync(It.IsAny<PaymentRequest>()))
            .ReturnsAsync(new GatewayResult { IsSuccess = false, ErrorMessage = "Card declined" });

        var result = await _sut.ProcessPaymentAsync(request);

        Assert.IsFalse(result.IsSuccess);
        Assert.AreEqual("Card declined", result.ErrorMessage);
    }

    [TestMethod]
    [ExpectedException(typeof(ArgumentException))]
    public async Task ProcessPayment_NegativeAmount_ThrowsException()
    {
        var request = new PaymentRequest { OrderId = "order-123", Amount = -100.00m, Currency = "USD" };
        await _sut.ProcessPaymentAsync(request);
    }
}
```

### Testing Domain Logic

```csharp
[TestClass]
public class PaymentEntityTests
{
    [TestMethod]
    public void Create_ValidParameters_CreatesPayment()
    {
        var payment = Payment.Create("order-123", 100.00m, "USD");

        Assert.AreEqual("order-123", payment.OrderId);
        Assert.AreEqual(100.00m, payment.Amount);
        Assert.AreEqual(PaymentStatus.Pending, payment.Status);
    }

    [TestMethod]
    [ExpectedException(typeof(ArgumentException))]
    public void Create_NegativeAmount_ThrowsException()
    {
        Payment.Create("order-123", -100.00m, "USD");
    }

    [TestMethod]
    public void MarkAsCompleted_FromProcessing_Succeeds()
    {
        var payment = Payment.Create("order-123", 100.00m, "USD");
        payment.MarkAsProcessing("txn_123");
        payment.MarkAsCompleted();

        Assert.AreEqual(PaymentStatus.Completed, payment.Status);
        Assert.IsNotNull(payment.CompletedAt);
    }

    [TestMethod]
    [ExpectedException(typeof(InvalidOperationException))]
    public void MarkAsCompleted_FromPending_ThrowsException()
    {
        var payment = Payment.Create("order-123", 100.00m, "USD");
        payment.MarkAsCompleted(); // Can't complete from Pending
    }
}
```

### Testing Adapters

```csharp
[TestClass]
public class StripeAdapterTests
{
    [TestMethod]
    public async Task CreatePayment_ValidRequest_MapsAmountToCents()
    {
        var mockService = new Mock<PaymentIntentService>();
        var adapter = new StripeAdapter(mockService.Object);

        var request = new PaymentRequest { Amount = 100.00m, Currency = "USD" };

        mockService
            .Setup(s => s.CreateAsync(
                It.IsAny<PaymentIntentCreateOptions>(),
                It.IsAny<RequestOptions>(),
                It.IsAny<CancellationToken>()))
            .ReturnsAsync(new PaymentIntent
            {
                Id = "pi_123", Amount = 10000, Currency = "usd", Status = "succeeded"
            });

        var result = await adapter.CreatePaymentAsync(request);

        Assert.AreEqual("pi_123", result.TransactionId);
        Assert.AreEqual(100.00m, result.Amount);

        // Verify Stripe was called with amount in cents
        mockService.Verify(s => s.CreateAsync(
            It.Is<PaymentIntentCreateOptions>(o => o.Amount == 10000),
            It.IsAny<RequestOptions>(),
            It.IsAny<CancellationToken>()), Times.Once);
    }
}
```

---

## Integration Testing

### Database Integration Tests

```csharp
using Xunit;
using Microsoft.EntityFrameworkCore;
using Testcontainers.PostgreSql;

public class PaymentRepositoryIntegrationTests : IAsyncLifetime
{
    private PostgreSqlContainer _postgres;
    private ApplicationDbContext _context;
    private PaymentRepository _repository;
    
    public async Task InitializeAsync()
    {
        // Start PostgreSQL container
        _postgres = new PostgreSqlBuilder()
            .WithImage("postgres:15")
            .WithDatabase("payment_test")
            .WithUsername("test")
            .WithPassword("test")
            .Build();
        
        await _postgres.StartAsync();
        
        // Create DbContext
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseNpgsql(_postgres.GetConnectionString())
            .Options;
        
        _context = new ApplicationDbContext(options);
        await _context.Database.MigrateAsync();
        
        _repository = new PaymentRepository(_context);
    }
    
    [Fact]
    public async Task CreatePayment_ShouldPersistToDatabase()
    {
        // Arrange
        var payment = new Payment
        {
            Id = Guid.NewGuid(),
            OrderId = "ORDER-123",
            Amount = 100.00m,
            Currency = "USD",
            Status = PaymentStatus.Pending,
            CreatedAt = DateTime.UtcNow
        };
        
        // Act
        await _repository.CreateAsync(payment);
        
        // Assert
        var saved = await _repository.GetByIdAsync(payment.Id);
        Assert.NotNull(saved);
        Assert.Equal(payment.OrderId, saved.OrderId);
        Assert.Equal(payment.Amount, saved.Amount);
    }
    
    [Fact]
    public async Task UpdatePaymentStatus_ShouldUpdateInDatabase()
    {
        // Arrange
        var payment = await CreateTestPaymentAsync();
        
        // Act
        payment.Status = PaymentStatus.Completed;
        payment.CompletedAt = DateTime.UtcNow;
        await _repository.UpdateAsync(payment);
        
        // Assert
        var updated = await _repository.GetByIdAsync(payment.Id);
        Assert.Equal(PaymentStatus.Completed, updated.Status);
        Assert.NotNull(updated.CompletedAt);
    }
    
    [Fact]
    public async Task GetPaymentsByOrderId_ShouldReturnAllPayments()
    {
        // Arrange
        var orderId = "ORDER-456";
        await CreateTestPaymentAsync(orderId);
        await CreateTestPaymentAsync(orderId);
        await CreateTestPaymentAsync("OTHER-ORDER");
        
        // Act
        var payments = await _repository.GetByOrderIdAsync(orderId);
        
        // Assert
        Assert.Equal(2, payments.Count);
        Assert.All(payments, p => Assert.Equal(orderId, p.OrderId));
    }
    
    [Fact]
    public async Task Transaction_ShouldRollbackOnError()
    {
        // Arrange
        var payment = new Payment
        {
            Id = Guid.NewGuid(),
            OrderId = "ORDER-789",
            Amount = 100.00m,
            Currency = "USD"
        };
        
        // Act & Assert
        await Assert.ThrowsAsync<DbUpdateException>(async () =>
        {
            using var transaction = await _context.Database.BeginTransactionAsync();
            
            await _repository.CreateAsync(payment);
            
            // Simulate error by creating duplicate
            await _repository.CreateAsync(payment);
            
            await transaction.CommitAsync();
        });
        
        // Verify nothing was saved
        var saved = await _repository.GetByIdAsync(payment.Id);
        Assert.Null(saved);
    }
    
    public async Task DisposeAsync()
    {
        await _context.DisposeAsync();
        await _postgres.DisposeAsync();
    }
    
    private async Task<Payment> CreateTestPaymentAsync(string orderId = null)
    {
        var payment = new Payment
        {
            Id = Guid.NewGuid(),
            OrderId = orderId ?? $"ORDER-{Guid.NewGuid()}",
            Amount = 100.00m,
            Currency = "USD",
            Status = PaymentStatus.Pending
        };
        
        await _repository.CreateAsync(payment);
        return payment;
    }
}
```

### Payment Gateway Integration Tests

```csharp
public class StripeGatewayIntegrationTests
{
    private readonly IConfiguration _configuration;
    private readonly StripePaymentGateway _gateway;
    
    public StripeGatewayIntegrationTests()
    {
        _configuration = new ConfigurationBuilder()
            .AddUserSecrets<StripeGatewayIntegrationTests>()
            .Build();
        
        // Use test API key
        _gateway = new StripePaymentGateway(_configuration["Stripe:TestSecretKey"]);
    }
    
    [Fact]
    public async Task ProcessPayment_WithValidCard_ShouldSucceed()
    {
        // Arrange
        var request = new PaymentRequest
        {
            Amount = 10.00m,
            Currency = "usd",
            CardNumber = "4242424242424242",
            ExpMonth = 12,
            ExpYear = DateTime.Now.Year + 2,
            Cvc = "123",
            Description = "Integration test payment"
        };
        
        // Act
        var result = await _gateway.ProcessPaymentAsync(request);
        
        // Assert
        Assert.True(result.IsSuccess);
        Assert.NotNull(result.TransactionId);
        Assert.StartsWith("pi_", result.TransactionId); // Stripe payment intent ID
    }
    
    [Fact]
    public async Task ProcessPayment_WithDeclinedCard_ShouldFail()
    {
        // Arrange
        var request = new PaymentRequest
        {
            Amount = 10.00m,
            Currency = "usd",
            CardNumber = "4000000000000002", // Decline card
            ExpMonth = 12,
            ExpYear = DateTime.Now.Year + 2,
            Cvc = "123"
        };
        
        // Act
        var result = await _gateway.ProcessPaymentAsync(request);
        
        // Assert
        Assert.False(result.IsSuccess);
        Assert.NotNull(result.ErrorMessage);
        Assert.Contains("declined", result.ErrorMessage.ToLower());
    }
    
    [Fact]
    public async Task RefundPayment_ShouldSucceed()
    {
        // Arrange - Create successful payment first
        var paymentResult = await CreateSuccessfulPaymentAsync();
        
        // Act
        var refundResult = await _gateway.RefundPaymentAsync(
            paymentResult.TransactionId,
            10.00m,
            "usd");
        
        // Assert
        Assert.True(refundResult.IsSuccess);
        Assert.NotNull(refundResult.RefundId);
    }
    
    [Fact]
    public async Task ProcessPayment_WithInvalidAmount_ShouldFail()
    {
        // Arrange
        var request = new PaymentRequest
        {
            Amount = -10.00m, // Invalid negative amount
            Currency = "usd",
            CardNumber = "4242424242424242",
            ExpMonth = 12,
            ExpYear = DateTime.Now.Year + 2,
            Cvc = "123"
        };
        
        // Act & Assert
        await Assert.ThrowsAsync<ArgumentException>(
            () => _gateway.ProcessPaymentAsync(request));
    }
    
    private async Task<PaymentResult> CreateSuccessfulPaymentAsync()
    {
        var request = new PaymentRequest
        {
            Amount = 10.00m,
            Currency = "usd",
            CardNumber = "4242424242424242",
            ExpMonth = 12,
            ExpYear = DateTime.Now.Year + 2,
            Cvc = "123"
        };
        
        return await _gateway.ProcessPaymentAsync(request);
    }
}
```

### Message Queue Integration Tests

```csharp
using RabbitMQ.Client;
using Testcontainers.RabbitMq;

public class PaymentQueueIntegrationTests : IAsyncLifetime
{
    private RabbitMqContainer _rabbitMq;
    private IConnection _connection;
    private IModel _channel;
    private PaymentQueueService _queueService;
    
    public async Task InitializeAsync()
    {
        // Start RabbitMQ container
        _rabbitMq = new RabbitMqBuilder()
            .WithImage("rabbitmq:3-management")
            .Build();
        
        await _rabbitMq.StartAsync();
        
        // Create connection
        var factory = new ConnectionFactory
        {
            Uri = new Uri(_rabbitMq.GetConnectionString())
        };
        
        _connection = factory.CreateConnection();
        _channel = _connection.CreateModel();
        
        _queueService = new PaymentQueueService(_connection);
    }
    
    [Fact]
    public async Task PublishPayment_ShouldBeReceivable()
    {
        // Arrange
        var payment = new Payment
        {
            Id = Guid.NewGuid(),
            Amount = 100.00m,
            Currency = "USD"
        };
        
        var received = new TaskCompletionSource<Payment>();
        
        // Setup consumer
        _queueService.StartConsuming(p =>
        {
            received.SetResult(p);
            return Task.CompletedTask;
        });
        
        // Act
        await _queueService.PublishAsync(payment);
        
        // Wait for message (with timeout)
        var result = await Task.WhenAny(
            received.Task,
            Task.Delay(TimeSpan.FromSeconds(5)));
        
        // Assert
        Assert.Equal(received.Task, result);
        var receivedPayment = await received.Task;
        Assert.Equal(payment.Id, receivedPayment.Id);
        Assert.Equal(payment.Amount, receivedPayment.Amount);
    }
    
    [Fact]
    public async Task PublishMultiplePayments_ShouldReceiveInOrder()
    {
        // Arrange
        var payments = new List<Payment>();
        for (int i = 0; i < 5; i++)
        {
            payments.Add(new Payment
            {
                Id = Guid.NewGuid(),
                Amount = 100.00m * i,
                Currency = "USD"
            });
        }
        
        var received = new List<Payment>();
        var semaphore = new SemaphoreSlim(0);
        
        _queueService.StartConsuming(async p =>
        {
            received.Add(p);
            semaphore.Release();
            await Task.CompletedTask;
        });
        
        // Act
        foreach (var payment in payments)
        {
            await _queueService.PublishAsync(payment);
        }
        
        // Wait for all messages
        for (int i = 0; i < payments.Count; i++)
        {
            await semaphore.WaitAsync(TimeSpan.FromSeconds(10));
        }
        
        // Assert
        Assert.Equal(payments.Count, received.Count);
        for (int i = 0; i < payments.Count; i++)
        {
            Assert.Equal(payments[i].Id, received[i].Id);
        }
    }
    
    public async Task DisposeAsync()
    {
        _channel?.Dispose();
        _connection?.Dispose();
        await _rabbitMq.DisposeAsync();
    }
}
```

---

## End-to-End Testing

### Complete Payment Flow E2E Test

```csharp
using Xunit;
using Microsoft.AspNetCore.Mvc.Testing;
using System.Net.Http.Json;

public class PaymentFlowE2ETests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;
    
    public PaymentFlowE2ETests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Use test database
                services.RemoveAll<DbContextOptions<ApplicationDbContext>>();
                services.AddDbContext<ApplicationDbContext>(options =>
                {
                    options.UseInMemoryDatabase("TestDb");
                });
                
                // Use test payment gateway
                services.RemoveAll<IPaymentGateway>();
                services.AddSingleton<IPaymentGateway, TestPaymentGateway>();
            });
        });
        
        _client = _factory.CreateClient();
    }
    
    [Fact]
    public async Task CompletePaymentFlow_Success()
    {
        // Step 1: Create customer
        var createCustomerRequest = new
        {
            Email = "test@example.com",
            Name = "Test Customer"
        };
        
        var customerResponse = await _client.PostAsJsonAsync(
            "/api/customers",
            createCustomerRequest);
        
        customerResponse.EnsureSuccessStatusCode();
        var customer = await customerResponse.Content.ReadFromJsonAsync<CustomerDto>();
        
        // Step 2: Create order
        var createOrderRequest = new
        {
            CustomerId = customer.Id,
            Items = new[]
            {
                new { ProductId = "PROD-1", Quantity = 2, Price = 25.00m },
                new { ProductId = "PROD-2", Quantity = 1, Price = 50.00m }
            }
        };
        
        var orderResponse = await _client.PostAsJsonAsync(
            "/api/orders",
            createOrderRequest);
        
        orderResponse.EnsureSuccessStatusCode();
        var order = await orderResponse.Content.ReadFromJsonAsync<OrderDto>();
        
        Assert.Equal(100.00m, order.TotalAmount);
        
        // Step 3: Process payment
        var processPaymentRequest = new
        {
            OrderId = order.Id,
            Amount = order.TotalAmount,
            Currency = "USD",
            PaymentMethod = new
            {
                CardNumber = "4242424242424242",
                ExpMonth = 12,
                ExpYear = 2025,
                Cvc = "123"
            }
        };
        
        var paymentResponse = await _client.PostAsJsonAsync(
            "/api/payments",
            processPaymentRequest);
        
        paymentResponse.EnsureSuccessStatusCode();
        var payment = await paymentResponse.Content.ReadFromJsonAsync<PaymentDto>();
        
        Assert.Equal("Completed", payment.Status);
        Assert.NotNull(payment.TransactionId);
        
        // Step 4: Verify order status updated
        var updatedOrderResponse = await _client.GetAsync($"/api/orders/{order.Id}");
        updatedOrderResponse.EnsureSuccessStatusCode();
        var updatedOrder = await updatedOrderResponse.Content.ReadFromJsonAsync<OrderDto>();
        
        Assert.Equal("Paid", updatedOrder.Status);
        Assert.Equal(payment.Id, updatedOrder.PaymentId);
        
        // Step 5: Verify payment in database
        var retrievedPaymentResponse = await _client.GetAsync($"/api/payments/{payment.Id}");
        retrievedPaymentResponse.EnsureSuccessStatusCode();
        var retrievedPayment = await retrievedPaymentResponse.Content.ReadFromJsonAsync<PaymentDto>();
        
        Assert.Equal(payment.Id, retrievedPayment.Id);
        Assert.Equal(payment.Amount, retrievedPayment.Amount);
    }
    
    [Fact]
    public async Task PaymentFlow_WithDeclinedCard_ShouldFailGracefully()
    {
        // Arrange
        var customer = await CreateTestCustomerAsync();
        var order = await CreateTestOrderAsync(customer.Id);
        
        // Act
        var processPaymentRequest = new
        {
            OrderId = order.Id,
            Amount = order.TotalAmount,
            Currency = "USD",
            PaymentMethod = new
            {
                CardNumber = "4000000000000002", // Declined card
                ExpMonth = 12,
                ExpYear = 2025,
                Cvc = "123"
            }
        };
        
        var paymentResponse = await _client.PostAsJsonAsync(
            "/api/payments",
            processPaymentRequest);
        
        // Assert
        Assert.Equal(HttpStatusCode.PaymentRequired, paymentResponse.StatusCode);
        
        var error = await paymentResponse.Content.ReadFromJsonAsync<ErrorDto>();
        Assert.Contains("declined", error.Message.ToLower());
        
        // Verify order status unchanged
        var orderResponse = await _client.GetAsync($"/api/orders/{order.Id}");
        var updatedOrder = await orderResponse.Content.ReadFromJsonAsync<OrderDto>();
        Assert.Equal("Pending", updatedOrder.Status);
    }
    
    [Fact]
    public async Task RefundFlow_ShouldCompleteSuccessfully()
    {
        // Arrange - Create successful payment
        var payment = await CreateSuccessfulPaymentAsync();
        
        // Act - Request refund
        var refundRequest = new
        {
            PaymentId = payment.Id,
            Amount = payment.Amount,
            Reason = "Customer request"
        };
        
        var refundResponse = await _client.PostAsJsonAsync(
            "/api/refunds",
            refundRequest);
        
        refundResponse.EnsureSuccessStatusCode();
        var refund = await refundResponse.Content.ReadFromJsonAsync<RefundDto>();
        
        // Assert
        Assert.Equal(payment.Id, refund.PaymentId);
        Assert.Equal(payment.Amount, refund.Amount);
        Assert.Equal("Completed", refund.Status);
        
        // Verify payment status updated
        var updatedPaymentResponse = await _client.GetAsync($"/api/payments/{payment.Id}");
        var updatedPayment = await updatedPaymentResponse.Content.ReadFromJsonAsync<PaymentDto>();
        Assert.Equal("Refunded", updatedPayment.Status);
    }
    
    private async Task<CustomerDto> CreateTestCustomerAsync()
    {
        var response = await _client.PostAsJsonAsync("/api/customers", new
        {
            Email = $"test{Guid.NewGuid()}@example.com",
            Name = "Test Customer"
        });
        
        return await response.Content.ReadFromJsonAsync<CustomerDto>();
    }
    
    private async Task<OrderDto> CreateTestOrderAsync(Guid customerId)
    {
        var response = await _client.PostAsJsonAsync("/api/orders", new
        {
            CustomerId = customerId,
            Items = new[]
            {
                new { ProductId = "PROD-1", Quantity = 1, Price = 100.00m }
            }
        });
        
        return await response.Content.ReadFromJsonAsync<OrderDto>();
    }
    
    private async Task<PaymentDto> CreateSuccessfulPaymentAsync()
    {
        var customer = await CreateTestCustomerAsync();
        var order = await CreateTestOrderAsync(customer.Id);
        
        var response = await _client.PostAsJsonAsync("/api/payments", new
        {
            OrderId = order.Id,
            Amount = order.TotalAmount,
            Currency = "USD",
            PaymentMethod = new
            {
                CardNumber = "4242424242424242",
                ExpMonth = 12,
                ExpYear = 2025,
                Cvc = "123"
            }
        });
        
        return await response.Content.ReadFromJsonAsync<PaymentDto>();
    }
}
```

### Webhook E2E Test

```csharp
public class WebhookE2ETests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    private readonly ITestOutputHelper _output;
    
    public WebhookE2ETests(
        WebApplicationFactory<Program> factory,
        ITestOutputHelper output)
    {
        _output = output;
        _client = factory.CreateClient();
    }
    
    [Fact]
    public async Task StripeWebhook_PaymentSucceeded_ShouldUpdatePayment()
    {
        // Arrange
        var payment = await CreatePendingPaymentAsync();
        
        var webhookPayload = $$"""
        {
            "id": "evt_test_{{Guid.NewGuid()}}",
            "type": "payment_intent.succeeded",
            "data": {
                "object": {
                    "id": "{{payment.ProviderTransactionId}}",
                    "amount": 10000,
                    "currency": "usd",
                    "status": "succeeded",
                    "metadata": {
                        "payment_id": "{{payment.Id}}"
                    }
                }
            }
        }
        """;
        
        var signature = GenerateStripeSignature(webhookPayload);
        
        // Act
        var request = new HttpRequestMessage(HttpMethod.Post, "/api/webhooks/stripe")
        {
            Content = new StringContent(webhookPayload, Encoding.UTF8, "application/json")
        };
        request.Headers.Add("Stripe-Signature", signature);
        
        var response = await _client.SendAsync(request);
        
        // Assert
        response.EnsureSuccessStatusCode();
        
        // Verify payment was updated
        await Task.Delay(1000); // Give webhook handler time to process
        
        var paymentResponse = await _client.GetAsync($"/api/payments/{payment.Id}");
        var updatedPayment = await paymentResponse.Content.ReadFromJsonAsync<PaymentDto>();
        
        Assert.Equal("Completed", updatedPayment.Status);
    }
    
    [Fact]
    public async Task PayPalWebhook_PaymentCaptured_ShouldProcessCorrectly()
    {
        // Arrange
        var payment = await CreatePendingPaymentAsync();
        
        var webhookPayload = $$"""
        {
            "event_type": "PAYMENT.CAPTURE.COMPLETED",
            "resource": {
                "id": "{{payment.ProviderTransactionId}}",
                "status": "COMPLETED",
                "amount": {
                    "value": "100.00",
                    "currency_code": "USD"
                }
            }
        }
        """;
        
        // Act
        var response = await _client.PostAsync(
            "/api/webhooks/paypal",
            new StringContent(webhookPayload, Encoding.UTF8, "application/json"));
        
        // Assert
        response.EnsureSuccessStatusCode();
        
        await Task.Delay(1000);
        
        var paymentResponse = await _client.GetAsync($"/api/payments/{payment.Id}");
        var updatedPayment = await paymentResponse.Content.ReadFromJsonAsync<PaymentDto>();
        
        Assert.Equal("Completed", updatedPayment.Status);
    }
    
    private async Task<PaymentDto> CreatePendingPaymentAsync()
    {
        var response = await _client.PostAsJsonAsync("/api/payments/create-pending", new
        {
            Amount = 100.00m,
            Currency = "USD",
            OrderId = $"ORDER-{Guid.NewGuid()}"
        });
        
        return await response.Content.ReadFromJsonAsync<PaymentDto>();
    }
    
    private string GenerateStripeSignature(string payload)
    {
        // Implementation of Stripe signature generation
        // This is a simplified version
        var secret = "whsec_test_secret";
        var timestamp = DateTimeOffset.UtcNow.ToUnixTimeSeconds();
        var signedPayload = $"{timestamp}.{payload}";
        
        using var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(secret));
        var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(signedPayload));
        var signature = Convert.ToHexString(hash).ToLower();
        
        return $"t={timestamp},v1={signature}";
    }
}
```

---

## UI Automation Testing

### Selenium/Playwright Tests

```csharp
using Microsoft.Playwright;
using Xunit;

public class PaymentUITests : IAsyncLifetime
{
    private IPlaywright _playwright;
    private IBrowser _browser;
    private IPage _page;
    
    public async Task InitializeAsync()
    {
        _playwright = await Playwright.CreateAsync();
        _browser = await _playwright.Chromium.LaunchAsync(new()
        {
            Headless = true, // Set false to see browser
            SlowMo = 100 // Slow down for debugging
        });
        _page = await _browser.NewPageAsync();
    }
    
    [Fact]
    public async Task CheckoutFlow_WithValidCard_ShouldComplete()
    {
        // Navigate to checkout
        await _page.GotoAsync("https://localhost:5001/checkout");
        
        // Fill in customer information
        await _page.FillAsync("#email", "test@example.com");
        await _page.FillAsync("#name", "Test Customer");
        
        // Fill in card details
        await _page.FillAsync("#card-number", "4242424242424242");
        await _page.FillAsync("#exp-month", "12");
        await _page.FillAsync("#exp-year", "2025");
        await _page.FillAsync("#cvc", "123");
        
        // Fill in billing address
        await _page.FillAsync("#address", "123 Test St");
        await _page.FillAsync("#city", "Test City");
        await _page.FillAsync("#zip", "12345");
        
        // Submit payment
        await _page.ClickAsync("#submit-payment");
        
        // Wait for success page
        await _page.WaitForURLAsync("**/payment/success");
        
        // Verify success message
        var successMessage = await _page.TextContentAsync(".success-message");
        Assert.Contains("Payment successful", successMessage);
        
        // Verify order number displayed
        var orderNumber = await _page.TextContentAsync(".order-number");
        Assert.NotNull(orderNumber);
        Assert.Matches(@"ORDER-\d+", orderNumber);
    }
    
    [Fact]
    public async Task CheckoutFlow_WithDeclinedCard_ShouldShowError()
    {
        // Navigate to checkout
        await _page.GotoAsync("https://localhost:5001/checkout");
        
        // Fill in details with declined card
        await _page.FillAsync("#email", "test@example.com");
        await _page.FillAsync("#name", "Test Customer");
        await _page.FillAsync("#card-number", "4000000000000002"); // Declined
        await _page.FillAsync("#exp-month", "12");
        await _page.FillAsync("#exp-year", "2025");
        await _page.FillAsync("#cvc", "123");
        
        // Submit payment
        await _page.ClickAsync("#submit-payment");
        
        // Wait for error message
        var errorMessage = await _page.WaitForSelectorAsync(".error-message");
        var errorText = await errorMessage.TextContentAsync();
        
        Assert.Contains("declined", errorText.ToLower());
        
        // Verify still on checkout page
        Assert.Contains("/checkout", _page.Url);
    }
    
    [Fact]
    public async Task CheckoutFlow_With3DSecure_ShouldRedirectAndComplete()
    {
        // Navigate to checkout
        await _page.GotoAsync("https://localhost:5001/checkout");
        
        // Fill in details with 3DS card
        await _page.FillAsync("#email", "test@example.com");
        await _page.FillAsync("#name", "Test Customer");
        await _page.FillAsync("#card-number", "4000002500003155"); // Requires 3DS
        await _page.FillAsync("#exp-month", "12");
        await _page.FillAsync("#exp-year", "2025");
        await _page.FillAsync("#cvc", "123");
        
        // Submit payment
        await _page.ClickAsync("#submit-payment");
        
        // Wait for 3DS redirect
        await _page.WaitForURLAsync("**/3d-secure/**");
        
        // Complete 3DS authentication (test flow)
        await _page.ClickAsync("#test-success-button");
        
        // Wait for return to success page
        await _page.WaitForURLAsync("**/payment/success");
        
        // Verify success
        var successMessage = await _page.TextContentAsync(".success-message");
        Assert.Contains("Payment successful", successMessage);
    }
    
    [Fact]
    public async Task PaymentForm_ShouldValidateInputs()
    {
        await _page.GotoAsync("https://localhost:5001/checkout");
        
        // Try to submit without filling anything
        await _page.ClickAsync("#submit-payment");
        
        // Check for validation errors
        var emailError = await _page.TextContentAsync("#email-error");
        Assert.Contains("required", emailError.ToLower());
        
        var cardError = await _page.TextContentAsync("#card-error");
        Assert.Contains("required", cardError.ToLower());
        
        // Fill invalid email
        await _page.FillAsync("#email", "invalid-email");
        await _page.ClickAsync("#submit-payment");
        
        emailError = await _page.TextContentAsync("#email-error");
        Assert.Contains("valid email", emailError.ToLower());
        
        // Fill invalid card number
        await _page.FillAsync("#card-number", "1234"); // Too short
        await _page.ClickAsync("#submit-payment");
        
        cardError = await _page.TextContentAsync("#card-error");
        Assert.Contains("invalid", cardError.ToLower());
    }
    
    [Fact]
    public async Task PaymentForm_ShouldShowLoadingState()
    {
        await _page.GotoAsync("https://localhost:5001/checkout");
        
        // Fill in valid details
        await _page.FillAsync("#email", "test@example.com");
        await _page.FillAsync("#name", "Test Customer");
        await _page.FillAsync("#card-number", "4242424242424242");
        await _page.FillAsync("#exp-month", "12");
        await _page.FillAsync("#exp-year", "2025");
        await _page.FillAsync("#cvc", "123");
        
        // Submit and check for loading state
        await _page.ClickAsync("#submit-payment");
        
        // Verify button shows loading
        var button = await _page.QuerySelectorAsync("#submit-payment");
        var buttonText = await button.TextContentAsync();
        Assert.Contains("Processing", buttonText);
        
        // Verify spinner is visible
        var spinner = await _page.QuerySelectorAsync(".spinner");
        Assert.True(await spinner.IsVisibleAsync());
        
        // Verify button is disabled
        Assert.True(await button.IsDisabledAsync());
    }
    
    public async Task DisposeAsync()
    {
        await _page.CloseAsync();
        await _browser.CloseAsync();
        _playwright.Dispose();
    }
}
```

### Screenshot Testing

```csharp
public class VisualRegressionTests : IAsyncLifetime
{
    private IPlaywright _playwright;
    private IBrowser _browser;
    private IPage _page;
    
    public async Task InitializeAsync()
    {
        _playwright = await Playwright.CreateAsync();
        _browser = await _playwright.Chromium.LaunchAsync();
        _page = await _browser.NewPageAsync(new()
        {
            ViewportSize = new ViewportSize { Width = 1920, Height = 1080 }
        });
    }
    
    [Fact]
    public async Task CheckoutPage_ShouldMatchBaseline()
    {
        await _page.GotoAsync("https://localhost:5001/checkout");
        
        // Take screenshot
        var screenshot = await _page.ScreenshotAsync(new()
        {
            Path = "screenshots/checkout-page.png",
            FullPage = true
        });
        
        // Compare with baseline (using external library like ImageSharp)
        var baseline = await File.ReadAllBytesAsync("baselines/checkout-page.png");
        
        var difference = CompareImages(baseline, screenshot);
        
        Assert.True(difference < 0.01, $"Visual difference: {difference:P2}");
    }
    
    [Fact]
    public async Task PaymentSuccess_ShouldMatchBaseline()
    {
        // Complete successful payment first
        await CompleteSuccessfulPaymentAsync();
        
        // Take screenshot of success page
        await _page.ScreenshotAsync(new()
        {
            Path = "screenshots/payment-success.png"
        });
        
        // Compare with baseline
        // ... comparison logic
    }
    
    private double CompareImages(byte[] image1, byte[] image2)
    {
        // Implementation using ImageSharp or similar
        // Returns difference percentage (0.0 = identical, 1.0 = completely different)
        return 0.0; // Placeholder
    }
    
    private async Task CompleteSuccessfulPaymentAsync()
    {
        await _page.GotoAsync("https://localhost:5001/checkout");
        await _page.FillAsync("#email", "test@example.com");
        await _page.FillAsync("#card-number", "4242424242424242");
        // ... fill other fields
        await _page.ClickAsync("#submit-payment");
        await _page.WaitForURLAsync("**/payment/success");
    }
    
    public async Task DisposeAsync()
    {
        await _page.CloseAsync();
        await _browser.CloseAsync();
        _playwright.Dispose();
    }
}
```

---

## API Testing

### REST API Tests

```csharp
using RestSharp;
using Xunit;

public class PaymentAPITests
{
    private readonly RestClient _client;
    private readonly string _baseUrl = "https://localhost:5001";
    
    public PaymentAPITests()
    {
        _client = new RestClient(_baseUrl);
    }
    
    [Fact]
    public async Task POST_CreatePayment_ReturnsCreated()
    {
        // Arrange
        var request = new RestRequest("/api/payments", Method.Post);
        request.AddJsonBody(new
        {
            OrderId = "ORDER-123",
            Amount = 100.00,
            Currency = "USD",
            PaymentMethod = new
            {
                Type = "card",
                CardNumber = "4242424242424242",
                ExpMonth = 12,
                ExpYear = 2025,
                Cvc = "123"
            }
        });
        
        // Act
        var response = await _client.ExecuteAsync<PaymentResponse>(request);
        
        // Assert
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        Assert.NotNull(response.Data);
        Assert.NotNull(response.Data.Id);
        Assert.Equal(100.00m, response.Data.Amount);
        Assert.Equal("Pending", response.Data.Status);
    }
    
    [Fact]
    public async Task GET_Payment_ReturnsPaymentDetails()
    {
        // Arrange - Create payment first
        var paymentId = await CreateTestPaymentAsync();
        
        var request = new RestRequest($"/api/payments/{paymentId}", Method.Get);
        
        // Act
        var response = await _client.ExecuteAsync<PaymentResponse>(request);
        
        // Assert
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
        Assert.NotNull(response.Data);
        Assert.Equal(paymentId, response.Data.Id);
    }
    
    [Fact]
    public async Task POST_RefundPayment_ReturnsRefundDetails()
    {
        // Arrange
        var paymentId = await CreateAndCompletePaymentAsync();
        
        var request = new RestRequest("/api/refunds", Method.Post);
        request.AddJsonBody(new
        {
            PaymentId = paymentId,
            Amount = 50.00,
            Reason = "Customer request"
        });
        
        // Act
        var response = await _client.ExecuteAsync<RefundResponse>(request);
        
        // Assert
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        Assert.NotNull(response.Data);
        Assert.Equal(paymentId, response.Data.PaymentId);
        Assert.Equal(50.00m, response.Data.Amount);
    }
    
    [Theory]
    [InlineData(-10.00, "Amount must be positive")]
    [InlineData(0, "Amount must be greater than zero")]
    [InlineData(1000000, "Amount exceeds maximum")]
    public async Task POST_CreatePayment_WithInvalidAmount_ReturnsBadRequest(
        decimal amount,
        string expectedError)
    {
        // Arrange
        var request = new RestRequest("/api/payments", Method.Post);
        request.AddJsonBody(new
        {
            OrderId = "ORDER-123",
            Amount = amount,
            Currency = "USD"
        });
        
        // Act
        var response = await _client.ExecuteAsync<ErrorResponse>(request);
        
        // Assert
        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
        Assert.Contains(expectedError, response.Data.Message);
    }
    
    [Fact]
    public async Task API_RateLimiting_ShouldReturn429()
    {
        // Arrange - Make many requests quickly
        var tasks = new List<Task<RestResponse>>();
        
        for (int i = 0; i < 100; i++)
        {
            var request = new RestRequest("/api/payments", Method.Get);
            tasks.Add(_client.ExecuteAsync(request));
        }
        
        // Act
        var responses = await Task.WhenAll(tasks);
        
        // Assert - At least some should be rate limited
        Assert.Contains(responses, r => r.StatusCode == HttpStatusCode.TooManyRequests);
        
        var rateLimitedResponse = responses.First(r => 
            r.StatusCode == HttpStatusCode.TooManyRequests);
        
        Assert.True(rateLimitedResponse.Headers.Any(h => 
            h.Name == "Retry-After"));
    }
    
    private async Task<Guid> CreateTestPaymentAsync()
    {
        var request = new RestRequest("/api/payments", Method.Post);
        request.AddJsonBody(new
        {
            OrderId = $"ORDER-{Guid.NewGuid()}",
            Amount = 100.00,
            Currency = "USD",
            PaymentMethod = new
            {
                CardNumber = "4242424242424242",
                ExpMonth = 12,
                ExpYear = 2025,
                Cvc = "123"
            }
        });
        
        var response = await _client.ExecuteAsync<PaymentResponse>(request);
        return response.Data.Id;
    }
    
    private async Task<Guid> CreateAndCompletePaymentAsync()
    {
        var paymentId = await CreateTestPaymentAsync();
        
        // Simulate webhook to complete payment
        // ... implementation
        
        return paymentId;
    }
}
```

---

**(Continuing in next message due to length...)**

Let me continue with the remaining sections:

---

## Contract Testing

### Pact Consumer Tests

```csharp
using PactNet;
using PactNet.Matchers;

public class PaymentAPIConsumerTests : IClassFixture<PaymentAPIConsumerTests.ConsumerPactFixture>
{
    private readonly PactBuilder _pact;
    private readonly int _mockServerPort = 9222;
    
    public PaymentAPIConsumerTests(ConsumerPactFixture fixture)
    {
        _pact = Pact.V3("PaymentConsumer", "PaymentAPI", new PactConfig
        {
            PactDir = "../../../pacts/",
            LogLevel = PactLogLevel.Debug
        });
    }
    
    [Fact]
    public async Task CreatePayment_WithValidRequest_ReturnsCreated()
    {
        // Arrange
        _pact
            .UponReceiving("a request to create a payment")
                .Given("a valid payment request")
                .WithRequest(HttpMethod.Post, "/api/payments")
                .WithHeader("Content-Type", "application/json")
                .WithJsonBody(new
                {
                    orderId = "ORDER-123",
                    amount = Match.Type(100.00m),
                    currency = Match.Regex("USD", "^[A-Z]{3}$")
                })
            .WillRespond()
                .WithStatus(HttpStatusCode.Created)
                .WithHeader("Content-Type", "application/json")
                .WithJsonBody(new
                {
                    id = Match.Type(Guid.NewGuid()),
                    orderId = "ORDER-123",
                    amount = 100.00m,
                    currency = "USD",
                    status = "Pending",
                    createdAt = Match.Type(DateTime.UtcNow)
                });
        
        await _pact.VerifyAsync(async ctx =>
        {
            // Act
            var client = new HttpClient
            {
                BaseAddress = new Uri($"http://localhost:{_mockServerPort}")
            };
            
            var response = await client.PostAsJsonAsync("/api/payments", new
            {
                orderId = "ORDER-123",
                amount = 100.00m,
                currency = "USD"
            });
            
            // Assert
            Assert.Equal(HttpStatusCode.Created, response.StatusCode);
            var payment = await response.Content.ReadFromJsonAsync<PaymentDto>();
            Assert.NotNull(payment);
            Assert.NotEqual(Guid.Empty, payment.Id);
        });
    }
    
    public class ConsumerPactFixture : IDisposable
    {
        public void Dispose()
        {
            // Cleanup
        }
    }
}
```

---

## Performance Testing

### Load Testing with k6

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export let options = {
    stages: [
        { duration: '2m', target: 100 },  // Ramp up to 100 users
        { duration: '5m', target: 100 },  // Stay at 100 users
        { duration: '2m', target: 200 },  // Ramp to 200 users
        { duration: '5m', target: 200 },  // Stay at 200 users
        { duration: '2m', target: 0 },    // Ramp down
    ],
    thresholds: {
        http_req_duration: ['p(95)<500'],  // 95% of requests must complete below 500ms
        errors: ['rate<0.01'],              // Error rate must be below 1%
    },
};

const BASE_URL = 'https://localhost:5001';

export default function () {
    // Create payment
    const payload = JSON.stringify({
        orderId: `ORDER-${Date.now()}-${Math.random()}`,
        amount: 100.00,
        currency: 'USD',
        paymentMethod: {
            cardNumber: '4242424242424242',
            expMonth: 12,
            expYear: 2025,
            cvc: '123'
        }
    });
    
    const params = {
        headers: {
            'Content-Type': 'application/json',
        },
    };
    
    const response = http.post(`${BASE_URL}/api/payments`, payload, params);
    
    const success = check(response, {
        'status is 201': (r) => r.status === 201,
        'response has payment id': (r) => JSON.parse(r.body).id !== undefined,
        'response time < 500ms': (r) => r.timings.duration < 500,
    });
    
    errorRate.add(!success);
    
    sleep(1);
}

export function handleSummary(data) {
    return {
        'summary.json': JSON.stringify(data),
        stdout: textSummary(data, { indent: ' ', enableColors: true }),
    };
}
```

Run with:
```bash
k6 run load-test.js
```

---

## Security Testing

### OWASP ZAP Integration

```csharp
using System.Diagnostics;

public class SecurityTests
{
    [Fact(Skip = "Run manually")]
    public async Task RunOWASPZAPScan()
    {
        // Start ZAP
        var zapProcess = Process.Start(new ProcessStartInfo
        {
            FileName = "zap.sh",
            Arguments = "-daemon -port 8080 -config api.disablekey=true",
            UseShellExecute = false
        });
        
        // Wait for ZAP to start
        await Task.Delay(TimeSpan.FromSeconds(30));
        
        // Configure target
        var zapClient = new HttpClient
        {
            BaseAddress = new Uri("http://localhost:8080")
        };
        
        var target = "https://localhost:5001";
        
        // Spider the application
        await zapClient.GetAsync($"/JSON/spider/action/scan/?url={target}");
        
        // Wait for spider to complete
        await WaitForSpiderCompletion(zapClient);
        
        // Active scan
        await zapClient.GetAsync($"/JSON/ascan/action/scan/?url={target}");
        
        // Wait for scan to complete
        await WaitForScanCompletion(zapClient);
        
        // Get results
        var response = await zapClient.GetAsync("/JSON/core/view/alerts/");
        var content = await response.Content.ReadAsStringAsync();
        
        // Parse and assert
        var alerts = JsonSerializer.Deserialize<ZapAlerts>(content);
        
        var highRiskAlerts = alerts.Alerts.Where(a => a.Risk == "High").ToList();
        
        Assert.Empty(highRiskAlerts);
        
        // Cleanup
        zapProcess.Kill();
    }
    
    private async Task WaitForSpiderCompletion(HttpClient client)
    {
        while (true)
        {
            var response = await client.GetAsync("/JSON/spider/view/status/");
            var content = await response.Content.ReadAsStringAsync();
            
            if (content.Contains("100"))
                break;
            
            await Task.Delay(TimeSpan.FromSeconds(5));
        }
    }
    
    private async Task WaitForScanCompletion(HttpClient client)
    {
        while (true)
        {
            var response = await client.GetAsync("/JSON/ascan/view/status/");
            var content = await response.Content.ReadAsStringAsync();
            
            if (content.Contains("100"))
                break;
            
            await Task.Delay(TimeSpan.FromSeconds(5));
        }
    }
}
```

---

## Production Testing

### Canary Testing

```csharp
public class CanaryTests
{
    private readonly HttpClient _canaryClient;
    private readonly HttpClient _stableClient;
    private readonly IMetricsCollector _metrics;
    
    [Fact]
    public async Task CompareCanaryVsStable_ErrorRates()
    {
        // Arrange
        var testDuration = TimeSpan.FromMinutes(10);
        var requestsPerSecond = 10;
        var endTime = DateTime.UtcNow.Add(testDuration);
        
        var canaryErrors = 0;
        var canaryTotal = 0;
        var stableErrors = 0;
        var stableTotal = 0;
        
        // Act - Send traffic to both versions
        while (DateTime.UtcNow < endTime)
        {
            var canaryTask = TestPaymentEndpoint(_canaryClient);
            var stableTask = TestPaymentEndpoint(_stableClient);
            
            await Task.WhenAll(canaryTask, stableTask);
            
            canaryTotal++;
            if (!canaryTask.Result) canaryErrors++;
            
            stableTotal++;
            if (!stableTask.Result) stableErrors++;
            
            await Task.Delay(1000 / requestsPerSecond);
        }
        
        // Assert - Canary error rate should not be significantly higher
        var canaryErrorRate = (double)canaryErrors / canaryTotal;
        var stableErrorRate = (double)stableErrors / stableTotal;
        
        Assert.True(canaryErrorRate <= stableErrorRate * 1.5,
            $"Canary error rate ({canaryErrorRate:P2}) is significantly higher than stable ({stableErrorRate:P2})");
    }
    
    [Fact]
    public async Task CompareCanaryVsStable_Latency()
    {
        // Arrange
        var iterations = 100;
        var canaryLatencies = new List<double>();
        var stableLatencies = new List<double>();
        
        // Act
        for (int i = 0; i < iterations; i++)
        {
            var canaryLatency = await MeasureLatency(_canaryClient);
            var stableLatency = await MeasureLatency(_stableClient);
            
            canaryLatencies.Add(canaryLatency);
            stableLatencies.Add(stableLatency);
            
            await Task.Delay(100);
        }
        
        // Assert - P95 latency should be similar
        var canaryP95 = canaryLatencies.OrderBy(x => x).ElementAt((int)(iterations * 0.95));
        var stableP95 = stableLatencies.OrderBy(x => x).ElementAt((int)(iterations * 0.95));
        
        Assert.True(canaryP95 <= stableP95 * 1.2,
            $"Canary P95 ({canaryP95}ms) is significantly higher than stable ({stableP95}ms)");
    }
    
    private async Task<bool> TestPaymentEndpoint(HttpClient client)
    {
        try
        {
            var response = await client.PostAsJsonAsync("/api/payments/test", new
            {
                amount = 10.00m,
                currency = "USD"
            });
            
            return response.IsSuccessStatusCode;
        }
        catch
        {
            return false;
        }
    }
    
    private async Task<double> MeasureLatency(HttpClient client)
    {
        var sw = Stopwatch.StartNew();
        
        await client.GetAsync("/api/health");
        
        sw.Stop();
        return sw.Elapsed.TotalMilliseconds;
    }
}
```

### Synthetic Monitoring

```csharp
public class SyntheticMonitoring : BackgroundService
{
    private readonly IHttpClientFactory _clientFactory;
    private readonly IMetricsCollector _metrics;
    private readonly IAlertService _alerts;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await RunSyntheticTestsAsync();
                await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
            }
            catch (Exception ex)
            {
                await _alerts.SendAlertAsync($"Synthetic monitoring failed: {ex.Message}");
            }
        }
    }
    
    private async Task RunSyntheticTestsAsync()
    {
        var client = _clientFactory.CreateClient("production");
        
        // Test 1: Health check
        await TestHealthEndpoint(client);
        
        // Test 2: Create payment (with test card)
        await TestCreatePayment(client);
        
        // Test 3: Retrieve payment
        await TestRetrievePayment(client);
        
        // Test 4: Webhook endpoint
        await TestWebhookEndpoint(client);
    }
    
    private async Task TestHealthEndpoint(HttpClient client)
    {
        var sw = Stopwatch.StartNew();
        var response = await client.GetAsync("/health");
        sw.Stop();
        
        _metrics.RecordSyntheticTest("health_check", 
            response.IsSuccessStatusCode, 
            sw.ElapsedMilliseconds);
        
        if (!response.IsSuccessStatusCode)
        {
            await _alerts.SendAlertAsync("Health check failed in production");
        }
    }
    
    private async Task TestCreatePayment(HttpClient client)
    {
        var sw = Stopwatch.StartNew();
        
        var response = await client.PostAsJsonAsync("/api/payments", new
        {
            orderId = $"SYNTHETIC-{Guid.NewGuid()}",
            amount = 1.00m, // Small amount for test
            currency = "USD",
            paymentMethod = new
            {
                cardNumber = "4242424242424242",
                expMonth = 12,
                expYear = 2025,
                cvc = "123"
            },
            metadata = new
            {
                synthetic_test = true
            }
        });
        
        sw.Stop();
        
        _metrics.RecordSyntheticTest("create_payment",
            response.IsSuccessStatusCode,
            sw.ElapsedMilliseconds);
        
        if (response.IsSuccessStatusCode)
        {
            var payment = await response.Content.ReadFromJsonAsync<PaymentDto>();
            
            // Immediately refund synthetic payment
            await client.PostAsJsonAsync("/api/refunds", new
            {
                paymentId = payment.Id,
                amount = 1.00m,
                reason = "Synthetic test cleanup"
            });
        }
    }
}
```

---

## Testing Infrastructure

### Docker Compose for Testing

```yaml
# docker-compose.test.yml
version: '3.8'

services:
  postgres-test:
    image: postgres:15
    environment:
      POSTGRES_DB: payment_test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    ports:
      - "5432:5432"
    volumes:
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql
  
  redis-test:
    image: redis:7-alpine
    ports:
      - "6379:6379"
  
  rabbitmq-test:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"
  
  stripe-mock:
    image: stripemock/stripe-mock:latest
    ports:
      - "12111:12111"
      - "12112:12112"
  
  payment-api:
    build:
      context: .
      dockerfile: Dockerfile.test
    environment:
      - ASPNETCORE_ENVIRONMENT=Test
      - ConnectionStrings__DefaultConnection=Host=postgres-test;Database=payment_test;Username=test;Password=test
      - Redis__ConnectionString=redis-test:6379
      - RabbitMQ__HostName=rabbitmq-test
      - Stripe__ApiBase=http://stripe-mock:12111
    depends_on:
      - postgres-test
      - redis-test
      - rabbitmq-test
      - stripe-mock
    ports:
      - "5001:80"
```

### CI/CD Pipeline Configuration

```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '10.0.x'
      
      - name: Restore dependencies
        run: dotnet restore
      
      - name: Run unit tests
        run: dotnet test --filter "Category=Unit" --logger "trx;LogFileName=unit-tests.trx"
      
      - name: Upload test results
        uses: actions/upload-artifact@v3
        with:
          name: unit-test-results
          path: '**/unit-tests.trx'

  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: payment_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '10.0.x'
      
      - name: Run integration tests
        run: dotnet test --filter "Category=Integration" --logger "trx;LogFileName=integration-tests.trx"
        env:
          ConnectionStrings__DefaultConnection: Host=localhost;Database=payment_test;Username=test;Password=test
          Redis__ConnectionString: localhost:6379
      
      - name: Upload test results
        uses: actions/upload-artifact@v3
        with:
          name: integration-test-results
          path: '**/integration-tests.trx'

  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Start services
        run: docker-compose -f docker-compose.test.yml up -d
      
      - name: Wait for services
        run: |
          timeout 60 bash -c 'until docker-compose -f docker-compose.test.yml ps | grep healthy; do sleep 5; done'
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '10.0.x'
      
      - name: Install Playwright
        run: |
          dotnet tool install --global Microsoft.Playwright.CLI
          playwright install
      
      - name: Run E2E tests
        run: dotnet test --filter "Category=E2E" --logger "trx;LogFileName=e2e-tests.trx"
      
      - name: Upload screenshots on failure
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: e2e-screenshots
          path: '**/screenshots/'
      
      - name: Stop services
        if: always()
        run: docker-compose -f docker-compose.test.yml down

  performance-tests:
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      
      - name: Start application
        run: docker-compose -f docker-compose.test.yml up -d
      
      - name: Setup k6
        run: |
          sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
          echo "deb https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
          sudo apt-get update
          sudo apt-get install k6
      
      - name: Run performance tests
        run: k6 run tests/performance/load-test.js
      
      - name: Upload performance results
        uses: actions/upload-artifact@v3
        with:
          name: performance-results
          path: summary.json
```

---

## Summary

### Test Coverage Goals

| Test Type | Coverage | Purpose |
|-----------|----------|---------|
| **Unit** | 80%+ | Individual components |
| **Integration** | 60%+ | Service interactions |
| **E2E** | Critical paths | Full workflows |
| **Performance** | Baseline + regression | Load handling |
| **Security** | OWASP Top 10 | Vulnerability detection |

### Testing Best Practices

✅ **Automate everything** - All tests in CI/CD  
✅ **Fast feedback** - Unit tests complete in seconds  
✅ **Isolated tests** - No dependencies between tests  
✅ **Deterministic** - Same input = same output  
✅ **Test pyramid** - More unit tests, fewer E2E  
✅ **Real services** - Use Testcontainers for integration tests  
✅ **Monitor production** - Synthetic tests in prod  
✅ **Load test regularly** - Catch performance regressions  

## Next Steps

- [Test Cards & Sandbox](13-TEST-CARDS-SANDBOX.md)
- [Security Patterns & Practices](15-SECURITY-PATTERNS-PRACTICES.md)
- [Performance Optimization](08-PERFORMANCE-OPTIMIZATION.md)
