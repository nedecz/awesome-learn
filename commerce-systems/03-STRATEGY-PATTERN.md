# Strategy Pattern for Payment Provider Selection

## Overview

The Strategy Pattern enables runtime selection of payment processing algorithms without modifying client code. In payment systems, this allows dynamically choosing between Stripe, PayPal, Square, Adyen, and other providers based on configurable rules.

## Table of Contents

1. [Why Strategy Pattern for Payments](#why-strategy-pattern-for-payments)
2. [Basic Strategy Implementation](#basic-strategy-implementation)
3. [Configuration-Driven Strategies](#configuration-driven-strategies)
4. [Routing Strategies](#routing-strategies)
5. [Fallback Strategy](#fallback-strategy)
6. [A/B Testing with Strategies](#ab-testing-with-strategies)
7. [Strategy Factory](#strategy-factory)
8. [Testing Strategies](#testing-strategies)

---

## Why Strategy Pattern for Payments

| Without Strategy | With Strategy |
|-----------------|---------------|
| Giant `if/else` or `switch` to pick provider | Clean, extensible provider selection |
| Adding a provider means changing existing code | Add a new class, register it — done |
| Hard to test individual providers in isolation | Each strategy is independently testable |
| Business rules coupled with gateway logic | Rules and gateways are decoupled |

---

## Basic Strategy Implementation

### Payment Gateway Interface (Strategy)

```csharp
public interface IPaymentGateway
{
    string ProviderName { get; }
    Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request);
    Task<RefundResult> ProcessRefundAsync(RefundRequest request);
    Task<PaymentStatusResult> GetPaymentStatusAsync(string transactionId);
    bool SupportsPaymentMethod(PaymentMethodType method);
    bool SupportsCurrency(string currencyCode);
}
```

### Concrete Strategies

```csharp
public class StripeGateway : IPaymentGateway
{
    private readonly StripeClient _client;
    public string ProviderName => "Stripe";

    public bool SupportsPaymentMethod(PaymentMethodType method) =>
        method is PaymentMethodType.CreditCard
             or PaymentMethodType.DebitCard
             or PaymentMethodType.BankTransfer
             or PaymentMethodType.Wallet;

    public bool SupportsCurrency(string currencyCode) =>
        _supportedCurrencies.Contains(currencyCode.ToUpperInvariant());

    private static readonly HashSet<string> _supportedCurrencies = new()
    {
        "USD", "EUR", "GBP", "CAD", "AUD", "JPY", "CHF", "SEK", "NOK", "DKK"
    };

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        var options = new PaymentIntentCreateOptions
        {
            Amount = (long)(request.Amount * 100),
            Currency = request.Currency.ToLower(),
            PaymentMethod = request.PaymentMethodId,
            Confirm = true,
            ReturnUrl = request.ReturnUrl
        };

        var service = new PaymentIntentService(_client);
        var intent = await service.CreateAsync(options);

        return new PaymentResult
        {
            IsSuccess = intent.Status == "succeeded",
            TransactionId = intent.Id,
            ProviderName = ProviderName,
            Status = MapStatus(intent.Status)
        };
    }

    // Additional methods...
}

public class PayPalGateway : IPaymentGateway
{
    public string ProviderName => "PayPal";

    public bool SupportsPaymentMethod(PaymentMethodType method) =>
        method is PaymentMethodType.PayPal or PaymentMethodType.CreditCard;

    public bool SupportsCurrency(string currencyCode) => true;

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        // PayPal Checkout SDK v2 implementation
        var orderRequest = new OrdersCreateRequest();
        orderRequest.Prefer("return=representation");
        orderRequest.RequestBody(new OrderRequest
        {
            CheckoutPaymentIntent = "CAPTURE",
            PurchaseUnits = new List<PurchaseUnitRequest>
            {
                new() { AmountWithBreakdown = new AmountWithBreakdown { Value = request.Amount.ToString("F2"), CurrencyCode = request.Currency } }
            }
        });

        var response = await _client.Execute(orderRequest);
        // Map response to PaymentResult...
    }
}

public class SquareGateway : IPaymentGateway
{
    public string ProviderName => "Square";

    public bool SupportsPaymentMethod(PaymentMethodType method) =>
        method is PaymentMethodType.CreditCard or PaymentMethodType.DebitCard;

    public bool SupportsCurrency(string currencyCode) =>
        currencyCode is "USD" or "CAD" or "GBP" or "AUD" or "JPY";

    // Implementation...
}
```

---

## Configuration-Driven Strategies

### Provider Configuration

```json
{
  "PaymentProviders": {
    "DefaultProvider": "Stripe",
    "Providers": {
      "Stripe": {
        "Enabled": true,
        "Priority": 1,
        "MaxTransactionAmount": 999999.99,
        "SupportedCurrencies": ["USD", "EUR", "GBP"],
        "SupportedRegions": ["US", "EU", "UK"]
      },
      "PayPal": {
        "Enabled": true,
        "Priority": 2,
        "MaxTransactionAmount": 10000.00,
        "SupportedCurrencies": ["USD", "EUR", "GBP", "BRL", "MXN"],
        "SupportedRegions": ["US", "EU", "LATAM"]
      },
      "Square": {
        "Enabled": true,
        "Priority": 3,
        "MaxTransactionAmount": 50000.00,
        "SupportedCurrencies": ["USD", "CAD"],
        "SupportedRegions": ["US", "CA"]
      }
    }
  }
}
```

### Configuration-Based Strategy Selector

```csharp
public class ConfigurableProviderStrategy : IPaymentProviderStrategy
{
    private readonly Dictionary<string, IPaymentGateway> _gateways;
    private readonly PaymentProviderOptions _options;

    public ConfigurableProviderStrategy(
        IEnumerable<IPaymentGateway> gateways,
        IOptions<PaymentProviderOptions> options)
    {
        _gateways = gateways.ToDictionary(g => g.ProviderName, StringComparer.OrdinalIgnoreCase);
        _options = options.Value;
    }

    public IPaymentGateway SelectProvider(PaymentContext context)
    {
        var candidates = _options.Providers
            .Where(p => p.Value.Enabled)
            .Where(p => p.Value.SupportedCurrencies.Contains(context.Currency))
            .Where(p => p.Value.SupportedRegions.Contains(context.Region))
            .Where(p => context.Amount <= p.Value.MaxTransactionAmount)
            .OrderBy(p => p.Value.Priority)
            .ToList();

        foreach (var candidate in candidates)
        {
            if (_gateways.TryGetValue(candidate.Key, out var gateway)
                && gateway.SupportsPaymentMethod(context.PaymentMethod))
            {
                return gateway;
            }
        }

        throw new NoEligibleProviderException(
            $"No provider available for {context.Currency} / {context.Region} / {context.PaymentMethod}");
    }
}
```

---

## Routing Strategies

### Cost-Optimized Routing

```csharp
public class CostOptimizedStrategy : IPaymentProviderStrategy
{
    private readonly IProviderFeeCalculator _feeCalculator;
    private readonly Dictionary<string, IPaymentGateway> _gateways;

    public IPaymentGateway SelectProvider(PaymentContext context)
    {
        var eligible = _gateways.Values
            .Where(g => g.SupportsCurrency(context.Currency)
                      && g.SupportsPaymentMethod(context.PaymentMethod))
            .ToList();

        return eligible
            .Select(g => new
            {
                Gateway = g,
                Fee = _feeCalculator.CalculateFee(g.ProviderName, context.Amount, context.Currency)
            })
            .OrderBy(x => x.Fee)
            .First()
            .Gateway;
    }
}

public class ProviderFeeCalculator : IProviderFeeCalculator
{
    public decimal CalculateFee(string provider, decimal amount, string currency) => provider switch
    {
        "Stripe" => amount * 0.029m + 0.30m,
        "PayPal" => amount * 0.0349m + 0.49m,
        "Square" => amount * 0.026m + 0.10m,
        _ => decimal.MaxValue
    };
}
```

### Region-Based Routing

```csharp
public class RegionBasedStrategy : IPaymentProviderStrategy
{
    private readonly Dictionary<string, string> _regionProviderMap = new()
    {
        ["US"]    = "Stripe",
        ["EU"]    = "Adyen",
        ["UK"]    = "Stripe",
        ["LATAM"] = "PayPal",
        ["APAC"]  = "Adyen"
    };

    private readonly Dictionary<string, IPaymentGateway> _gateways;

    public IPaymentGateway SelectProvider(PaymentContext context)
    {
        if (_regionProviderMap.TryGetValue(context.Region, out var providerName)
            && _gateways.TryGetValue(providerName, out var gateway))
        {
            return gateway;
        }

        // Fallback to default
        return _gateways[_options.DefaultProvider];
    }
}
```

---

## Fallback Strategy

Handle primary provider outages gracefully.

```csharp
public class FallbackProviderStrategy : IPaymentProviderStrategy
{
    private readonly IPaymentProviderStrategy _primary;
    private readonly IEnumerable<IPaymentGateway> _gateways;
    private readonly ICircuitBreakerRegistry _circuitBreakers;
    private readonly ILogger<FallbackProviderStrategy> _logger;

    public async Task<PaymentResult> ProcessWithFallbackAsync(PaymentContext context)
    {
        var prioritizedGateways = GetPrioritizedGateways(context);

        foreach (var gateway in prioritizedGateways)
        {
            if (_circuitBreakers.IsOpen(gateway.ProviderName))
            {
                _logger.LogWarning("Circuit breaker open for {Provider}, skipping", gateway.ProviderName);
                continue;
            }

            try
            {
                var result = await gateway.ProcessPaymentAsync(context.ToRequest());

                if (result.IsSuccess) return result;

                _logger.LogWarning("Provider {Provider} declined, trying next", gateway.ProviderName);
            }
            catch (Exception ex) when (ex is TimeoutException or HttpRequestException)
            {
                _circuitBreakers.RecordFailure(gateway.ProviderName);
                _logger.LogError(ex, "Provider {Provider} failed, trying next", gateway.ProviderName);
            }
        }

        throw new AllProvidersFailedException("All payment providers failed or are unavailable");
    }

    private IEnumerable<IPaymentGateway> GetPrioritizedGateways(PaymentContext context)
    {
        return _gateways
            .Where(g => g.SupportsCurrency(context.Currency)
                      && g.SupportsPaymentMethod(context.PaymentMethod))
            .OrderBy(g => _options.Providers[g.ProviderName].Priority);
    }
}
```

---

## A/B Testing with Strategies

Route a percentage of traffic to different providers for comparison.

```csharp
public class ABTestProviderStrategy : IPaymentProviderStrategy
{
    private readonly Dictionary<string, IPaymentGateway> _gateways;
    private readonly ICohortAssigner _cohortAssigner;

    public IPaymentGateway SelectProvider(PaymentContext context)
    {
        var cohort = _cohortAssigner.GetCohort(context.CustomerId);

        return cohort switch
        {
            "control" => _gateways["Stripe"],   // 80% of traffic
            "variant" => _gateways["Adyen"],     // 20% of traffic
            _ => _gateways["Stripe"]
        };
    }
}

public class CohortAssigner : ICohortAssigner
{
    public string GetCohort(string customerId)
    {
        // Deterministic: same customer always gets same cohort
        var hash = Math.Abs(customerId.GetHashCode()) % 100;
        return hash < 80 ? "control" : "variant";
    }
}
```

---

## Strategy Factory

Use a factory to resolve the right strategy based on context.

```csharp
public interface IPaymentStrategyFactory
{
    IPaymentProviderStrategy Create(StrategyType type);
}

public class PaymentStrategyFactory : IPaymentStrategyFactory
{
    private readonly IServiceProvider _serviceProvider;

    public PaymentStrategyFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public IPaymentProviderStrategy Create(StrategyType type) => type switch
    {
        StrategyType.CostOptimized   => _serviceProvider.GetRequiredService<CostOptimizedStrategy>(),
        StrategyType.RegionBased     => _serviceProvider.GetRequiredService<RegionBasedStrategy>(),
        StrategyType.Fallback        => _serviceProvider.GetRequiredService<FallbackProviderStrategy>(),
        StrategyType.ABTest          => _serviceProvider.GetRequiredService<ABTestProviderStrategy>(),
        StrategyType.Configuration   => _serviceProvider.GetRequiredService<ConfigurableProviderStrategy>(),
        _ => throw new ArgumentException($"Unknown strategy type: {type}")
    };
}

// DI Registration
services.AddScoped<CostOptimizedStrategy>();
services.AddScoped<RegionBasedStrategy>();
services.AddScoped<FallbackProviderStrategy>();
services.AddScoped<ABTestProviderStrategy>();
services.AddScoped<ConfigurableProviderStrategy>();
services.AddScoped<IPaymentStrategyFactory, PaymentStrategyFactory>();
```

---

## Testing Strategies

```csharp
[TestClass]
public class CostOptimizedStrategyTests
{
    [TestMethod]
    public void SelectProvider_ChoosesCheapestOption()
    {
        // Arrange
        var stripe = new Mock<IPaymentGateway>();
        stripe.Setup(g => g.ProviderName).Returns("Stripe");
        stripe.Setup(g => g.SupportsCurrency("USD")).Returns(true);
        stripe.Setup(g => g.SupportsPaymentMethod(PaymentMethodType.CreditCard)).Returns(true);

        var square = new Mock<IPaymentGateway>();
        square.Setup(g => g.ProviderName).Returns("Square");
        square.Setup(g => g.SupportsCurrency("USD")).Returns(true);
        square.Setup(g => g.SupportsPaymentMethod(PaymentMethodType.CreditCard)).Returns(true);

        var feeCalc = new Mock<IProviderFeeCalculator>();
        feeCalc.Setup(f => f.CalculateFee("Stripe", 100m, "USD")).Returns(3.20m);
        feeCalc.Setup(f => f.CalculateFee("Square", 100m, "USD")).Returns(2.70m);

        var strategy = new CostOptimizedStrategy(
            feeCalc.Object,
            new[] { stripe.Object, square.Object });

        var context = new PaymentContext { Amount = 100m, Currency = "USD", PaymentMethod = PaymentMethodType.CreditCard };

        // Act
        var result = strategy.SelectProvider(context);

        // Assert
        Assert.AreEqual("Square", result.ProviderName);
    }

    [TestMethod]
    public void FallbackStrategy_SkipsOpenCircuitBreaker()
    {
        // Arrange
        var primary = new Mock<IPaymentGateway>();
        primary.Setup(g => g.ProviderName).Returns("Stripe");

        var secondary = new Mock<IPaymentGateway>();
        secondary.Setup(g => g.ProviderName).Returns("PayPal");
        secondary.Setup(g => g.ProcessPaymentAsync(It.IsAny<PaymentRequest>()))
            .ReturnsAsync(new PaymentResult { IsSuccess = true });

        var circuitBreakers = new Mock<ICircuitBreakerRegistry>();
        circuitBreakers.Setup(c => c.IsOpen("Stripe")).Returns(true);
        circuitBreakers.Setup(c => c.IsOpen("PayPal")).Returns(false);

        // Act & Assert — should skip Stripe and use PayPal
    }
}
```

---

## Summary

| Strategy | Use Case | Key Benefit |
|----------|----------|-------------|
| Configuration-Driven | Standard multi-provider setups | Easy to change via config |
| Cost-Optimized | Minimize processing fees | Lowest transaction cost |
| Region-Based | Multi-region deployments | Local provider compliance |
| Fallback | High availability | Automatic failover |
| A/B Test | Provider evaluation | Data-driven decisions |

## Next Steps

- [Repository Pattern](02-REPOSITORY-PATTERN.md)
- [Architecture Patterns](01-ARCHITECTURE-PATTERNS.md)
- [Error Handling Patterns](05-ERROR-HANDLING.md)
