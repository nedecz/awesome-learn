# Cost Optimization for Payment Systems

## Overview

Payment processing costs can be significant — both in infrastructure spend and provider transaction fees. This guide covers strategies to minimize both, from choosing the right provider per transaction to right-sizing infrastructure and leveraging reserved capacity.

## Table of Contents

1. [Transaction Fee Optimization](#transaction-fee-optimization)
2. [Provider Cost Comparison](#provider-cost-comparison)
3. [Smart Routing for Cost Savings](#smart-routing-for-cost-savings)
4. [Infrastructure Cost Optimization](#infrastructure-cost-optimization)
5. [Database Cost Optimization](#database-cost-optimization)
6. [Caching Strategies](#caching-strategies)
7. [Message Queue Cost Management](#message-queue-cost-management)
8. [Cloud Provider Cost Strategies](#cloud-provider-cost-strategies)
9. [Cost Monitoring and Budgets](#cost-monitoring-and-budgets)

---

## Transaction Fee Optimization

### Understanding Fee Structures

| Provider | Card-Present | Card-Not-Present | ACH/Bank | International |
|----------|-------------|-------------------|----------|---------------|
| **Stripe** | 2.7% + $0.05 | 2.9% + $0.30 | 0.8% ($5 cap) | +1.5% |
| **PayPal** | — | 3.49% + $0.49 | — | +1.5% |
| **Square** | 2.6% + $0.10 | 2.9% + $0.30 | — | +1.0% |
| **Adyen** | Interchange++ | Interchange++ | 0.25% | Varies |
| **Braintree** | — | 2.59% + $0.49 | — | +1.0% |

> **Interchange++** pricing (Adyen, Stripe custom) passes through the actual interchange rate + a fixed markup. This is typically cheaper at scale.

### Fee Reduction Strategies

```csharp
public class CostOptimizedPaymentRouter
{
    private readonly Dictionary<string, IPaymentGateway> _gateways;
    private readonly IFeeEstimator _feeEstimator;

    public IPaymentGateway SelectCheapestProvider(PaymentContext context)
    {
        var estimates = _gateways.Values
            .Where(g => g.SupportsCurrency(context.Currency)
                      && g.SupportsPaymentMethod(context.PaymentMethod))
            .Select(g => new
            {
                Gateway = g,
                EstimatedFee = _feeEstimator.Estimate(g.ProviderName, context)
            })
            .OrderBy(x => x.EstimatedFee)
            .ToList();

        return estimates.First().Gateway;
    }
}

public class FeeEstimator : IFeeEstimator
{
    public decimal Estimate(string provider, PaymentContext context)
    {
        var amount = context.Amount;
        var isInternational = context.CardCountry != context.MerchantCountry;
        var isDomestic = !isInternational;

        return provider switch
        {
            "Stripe" => isDomestic
                ? amount * 0.029m + 0.30m
                : amount * 0.044m + 0.30m, // 2.9% + 1.5% international

            "Square" => isDomestic
                ? amount * 0.029m + 0.30m
                : amount * 0.039m + 0.30m,

            "Adyen" => EstimateInterchangePlus(amount, context), // Interchange++ is usually cheaper for high volume

            _ => decimal.MaxValue
        };
    }

    private decimal EstimateInterchangePlus(decimal amount, PaymentContext context)
    {
        // Typical interchange rates by card type
        var interchangeRate = context.CardType switch
        {
            CardType.Debit => 0.0005m + 0.22m,       // Durbin regulated
            CardType.Credit => amount * 0.018m,        // ~1.8% average
            CardType.CorporateCard => amount * 0.025m, // Higher interchange
            _ => amount * 0.02m
        };

        var adyenMarkup = amount * 0.006m + 0.12m; // Adyen's margin
        return interchangeRate + adyenMarkup;
    }
}
```

### ACH/Bank Transfer for Large Payments

```csharp
public class PaymentMethodOptimizer
{
    /// <summary>
    /// For large payments, ACH is dramatically cheaper than cards.
    /// Stripe ACH: 0.8% capped at $5 vs. 2.9% + $0.30
    /// </summary>
    public PaymentMethodSuggestion SuggestPaymentMethod(decimal amount, string currency)
    {
        if (currency == "USD" && amount > 200m)
        {
            // ACH saves money above ~$200
            // Card fee: $200 * 2.9% + $0.30 = $6.10
            // ACH fee: $200 * 0.8% = $1.60
            return new PaymentMethodSuggestion
            {
                RecommendedMethod = PaymentMethodType.BankTransfer,
                EstimatedSavings = (amount * 0.029m + 0.30m) - Math.Min(amount * 0.008m, 5m),
                Reason = "Bank transfer saves significantly on larger payments"
            };
        }

        return new PaymentMethodSuggestion
        {
            RecommendedMethod = PaymentMethodType.CreditCard,
            EstimatedSavings = 0
        };
    }
}
```

---

## Provider Cost Comparison

### Monthly Cost Calculator

```csharp
public class MonthlyCostCalculator
{
    public ProviderCostReport CalculateMonthlyCost(
        string provider,
        int transactionCount,
        decimal averageTransactionAmount,
        decimal internationalPercentage)
    {
        var domesticCount = (int)(transactionCount * (1 - internationalPercentage));
        var internationalCount = transactionCount - domesticCount;

        var (domesticRate, domesticFixed, intlSurcharge) = GetProviderRates(provider);

        var domesticFees = domesticCount * (averageTransactionAmount * domesticRate + domesticFixed);
        var intlFees = internationalCount * (averageTransactionAmount * (domesticRate + intlSurcharge) + domesticFixed);
        var totalFees = domesticFees + intlFees;

        var totalRevenue = transactionCount * averageTransactionAmount;

        return new ProviderCostReport
        {
            Provider = provider,
            TotalFees = totalFees,
            EffectiveRate = totalFees / totalRevenue,
            DomesticFees = domesticFees,
            InternationalFees = intlFees,
            CostPerTransaction = totalFees / transactionCount
        };
    }

    private (decimal rate, decimal fixedFee, decimal intlSurcharge) GetProviderRates(string provider) =>
        provider switch
        {
            "Stripe" => (0.029m, 0.30m, 0.015m),
            "PayPal" => (0.0349m, 0.49m, 0.015m),
            "Square" => (0.029m, 0.30m, 0.010m),
            "Braintree" => (0.0259m, 0.49m, 0.010m),
            _ => (0.03m, 0.30m, 0.015m)
        };
}

// Example output
// Provider   | 10K txns × $50 avg | Effective Rate | Monthly Fees
// Stripe     | 10,000             | 3.50%          | $17,500
// PayPal     | 10,000             | 4.47%          | $22,350
// Square     | 10,000             | 3.50%          | $17,500
// Adyen (I++) | 10,000            | 2.70%          | $13,500  ← best at scale
```

### Volume Discount Negotiation

At high volumes (>$1M/month), most providers offer custom pricing:

| Volume (Monthly) | Stripe Negotiated | Adyen I++ | Savings vs Standard |
|-------------------|------------------|-----------|-------------------|
| < $100K | 2.9% + $0.30 | Not available | — |
| $100K–$500K | 2.7% + $0.25 | ~2.2% effective | 10–25% |
| $500K–$2M | 2.5% + $0.20 | ~1.9% effective | 20–35% |
| $2M+ | 2.2% + $0.15 | ~1.6% effective | 30–45% |

---

## Smart Routing for Cost Savings

```csharp
public class SmartCostRouter : IPaymentProviderStrategy
{
    private readonly Dictionary<string, IPaymentGateway> _gateways;
    private readonly IFeeEstimator _feeEstimator;
    private readonly ICircuitBreakerRegistry _circuitBreakers;
    private readonly ILogger<SmartCostRouter> _logger;

    public IPaymentGateway SelectProvider(PaymentContext context)
    {
        var candidates = _gateways.Values
            .Where(g => !_circuitBreakers.IsOpen(g.ProviderName))
            .Where(g => g.SupportsCurrency(context.Currency))
            .Where(g => g.SupportsPaymentMethod(context.PaymentMethod))
            .Select(g => new
            {
                Gateway = g,
                Fee = _feeEstimator.Estimate(g.ProviderName, context),
                SuccessRate = _metrics.GetSuccessRate(g.ProviderName, TimeSpan.FromHours(1))
            })
            .Where(c => c.SuccessRate > 0.95m) // Only consider providers with >95% success rate
            .OrderBy(c => c.Fee)
            .ToList();

        if (!candidates.Any())
            throw new NoEligibleProviderException("No healthy, cost-effective provider available");

        var selected = candidates.First();
        _logger.LogInformation(
            "Routed to {Provider} (fee: {Fee:C}, success rate: {SuccessRate:P})",
            selected.Gateway.ProviderName, selected.Fee, selected.SuccessRate);

        return selected.Gateway;
    }
}
```

---

## Infrastructure Cost Optimization

### Right-Sizing Kubernetes Resources

```csharp
// Monitor actual resource usage and adjust
// kubectl top pods -l app=payment-service -n payment-system

// Start conservative:
// requests: cpu: 250m, memory: 256Mi
// limits:   cpu: 1,    memory: 512Mi

// After observation, tighten based on actual usage:
// If avg CPU is 100m and peak is 300m:
// requests: cpu: 150m  (1.5x average)
// limits:   cpu: 500m  (buffer for spikes)
```

### Spot / Preemptible Nodes for Non-Critical Workloads

```yaml
# Use spot nodes for webhook processors (idempotent, retry-safe)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webhook-processor
spec:
  template:
    spec:
      nodeSelector:
        node-type: spot
      tolerations:
        - key: "spot"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
      # NEVER use spot for the payment API itself
```

### Serverless for Variable Workloads

```csharp
// Use AWS Lambda / Azure Functions for:
// - Webhook processing (bursty, event-driven)
// - Report generation (periodic, CPU-intensive)
// - Notification sending (variable volume)

// Keep on Kubernetes:
// - Payment API (low-latency, always-on)
// - Database connections (connection pooling)
```

---

## Database Cost Optimization

### Read Replicas for Reporting

```csharp
public class PaymentReportRepository
{
    private readonly ApplicationDbContext _readContext; // Points to read replica

    public async Task<RevenueReport> GetRevenueReportAsync(DateTime start, DateTime end)
    {
        // Heavy queries go to read replica — no impact on payment processing
        return await _readContext.Payments
            .Where(p => p.Status == PaymentStatus.Completed
                     && p.CompletedAt >= start
                     && p.CompletedAt <= end)
            .GroupBy(p => new { p.Currency, p.CompletedAt.Date })
            .Select(g => new DailyRevenue
            {
                Date = g.Key.Date,
                Currency = g.Key.Currency,
                Total = g.Sum(p => p.Amount),
                Count = g.Count()
            })
            .ToListAsync();
    }
}

// DI: separate DbContext for reads
services.AddDbContext<ReadOnlyDbContext>(options =>
    options.UseNpgsql(config["ConnectionStrings:ReadReplica"])
           .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));
```

### Data Archival

```csharp
public class PaymentArchivalService
{
    /// <summary>
    /// Move completed payments older than 90 days to cold storage.
    /// Reduces primary DB size → lower IOPS → lower cost.
    /// </summary>
    public async Task ArchiveOldPaymentsAsync()
    {
        var cutoff = DateTime.UtcNow.AddDays(-90);

        var oldPayments = await _context.Payments
            .Where(p => p.Status == PaymentStatus.Completed && p.CompletedAt < cutoff)
            .Take(1000) // Batch to avoid long transactions
            .ToListAsync();

        // Write to S3 / Blob storage (Parquet format for analytics)
        await _archiveStorage.WriteAsync(oldPayments);

        // Remove from primary DB
        _context.Payments.RemoveRange(oldPayments);
        await _context.SaveChangesAsync();

        _logger.LogInformation("Archived {Count} payments older than {Cutoff}", oldPayments.Count, cutoff);
    }
}
```

---

## Caching Strategies

```csharp
// Cache frequently accessed, rarely changing data
public class CachedConfigService
{
    private readonly IDistributedCache _cache;

    // Cache exchange rates (refresh every 15 minutes, not every request)
    public async Task<decimal> GetExchangeRateAsync(string from, string to)
    {
        var cacheKey = $"fx:{from}:{to}";
        var cached = await _cache.GetStringAsync(cacheKey);

        if (cached != null) return decimal.Parse(cached);

        var rate = await _exchangeRateApi.GetRateAsync(from, to);
        await _cache.SetStringAsync(cacheKey, rate.ToString(),
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15) });

        return rate;
    }

    // Cache provider status (avoid hammering status endpoints)
    public async Task<ProviderStatus> GetProviderStatusAsync(string provider)
    {
        var cacheKey = $"provider-status:{provider}";
        var cached = await _cache.GetAsync<ProviderStatus>(cacheKey);

        if (cached != null) return cached;

        var status = await _providerHealthChecker.CheckAsync(provider);
        await _cache.SetAsync(cacheKey, status,
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromSeconds(30) });

        return status;
    }
}
```

---

## Message Queue Cost Management

### SQS Cost Optimization

```csharp
// Use long polling (reduce empty receives = reduce cost)
var receiveRequest = new ReceiveMessageRequest
{
    QueueUrl = _queueUrl,
    MaxNumberOfMessages = 10,  // Batch up to 10 messages
    WaitTimeSeconds = 20       // Long polling (max 20s) — free, reduces API calls by ~90%
};

// Use SQS batch operations
var sendBatch = new SendMessageBatchRequest
{
    QueueUrl = _queueUrl,
    Entries = messages.Select((m, i) => new SendMessageBatchRequestEntry
    {
        Id = i.ToString(),
        MessageBody = m
    }).ToList()
};
```

### Dead Letter Queue Cost

```csharp
// DLQ messages still incur storage costs — set a retention policy
// Don't keep DLQ messages forever
// Process/archive → delete within 7 days
{
    "RedrivePolicy": {
        "deadLetterTargetArn": "arn:aws:sqs:...:payment-dlq",
        "maxReceiveCount": 5
    },
    "MessageRetentionPeriod": 604800  // 7 days (in seconds)
}
```

---

## Cloud Provider Cost Strategies

### AWS

| Strategy | Savings | Best For |
|----------|---------|----------|
| Reserved Instances | 30–60% | Always-on services |
| Savings Plans | 20–40% | Flexible compute |
| Spot Instances | 60–90% | Webhook processors, batch jobs |
| S3 Intelligent-Tiering | Variable | Payment archives |
| DynamoDB On-Demand | Variable | Variable webhook volume |
| Lambda | Pay-per-use | Event processing |

### Azure

| Strategy | Savings | Best For |
|----------|---------|----------|
| Reserved Instances | 30–55% | AKS node pools |
| Azure Hybrid Benefit | 40–80% | Windows/.NET workloads |
| Spot VMs | 60–90% | Batch processing |
| Cosmos DB serverless | Variable | Low/variable traffic |
| Azure Functions Consumption | Pay-per-use | Event handlers |

---

## Cost Monitoring and Budgets

### CloudWatch Budget Alerts (AWS)

```yaml
# Terraform example
resource "aws_budgets_budget" "payment_system" {
  name         = "payment-system-monthly"
  budget_type  = "COST"
  limit_amount = "5000"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator       = "GREATER_THAN"
    threshold                 = 80
    threshold_type            = "PERCENTAGE"
    notification_type         = "ACTUAL"
    subscriber_email_addresses = ["payments-team@example.com"]
  }

  notification {
    comparison_operator       = "GREATER_THAN"
    threshold                 = 100
    threshold_type            = "PERCENTAGE"
    notification_type         = "FORECASTED"
    subscriber_email_addresses = ["payments-team@example.com"]
  }

  cost_filter {
    name   = "TagKeyValue"
    values = ["user:project$PaymentSystem"]
  }
}
```

### Cost Dashboard Metrics

```csharp
public class CostTracker
{
    public async Task<CostReport> GenerateWeeklyReportAsync()
    {
        return new CostReport
        {
            // Transaction fees
            TransactionFees = await CalculateTransactionFeesAsync(),

            // Infrastructure
            ComputeCost = await _cloudCostApi.GetCostAsync("EC2", TimeSpan.FromDays(7)),
            DatabaseCost = await _cloudCostApi.GetCostAsync("RDS", TimeSpan.FromDays(7)),
            CacheCost = await _cloudCostApi.GetCostAsync("ElastiCache", TimeSpan.FromDays(7)),
            MessageQueueCost = await _cloudCostApi.GetCostAsync("SQS", TimeSpan.FromDays(7)),
            StorageCost = await _cloudCostApi.GetCostAsync("S3", TimeSpan.FromDays(7)),

            // Metrics
            TotalPaymentsProcessed = await _metrics.GetWeeklyTransactionCount(),
            AverageTransactionValue = await _metrics.GetAverageTransactionValue(),
            EffectiveFeeRate = await CalculateEffectiveFeeRateAsync(),

            // Recommendations
            Recommendations = await GenerateRecommendationsAsync()
        };
    }

    private async Task<List<string>> GenerateRecommendationsAsync()
    {
        var recommendations = new List<string>();

        var avgCpu = await _metrics.GetAverageCpuUsage(TimeSpan.FromDays(7));
        if (avgCpu < 20)
            recommendations.Add($"CPU utilization is {avgCpu:F0}%. Consider downsizing compute instances.");

        var achEligible = await _metrics.GetTransactionsOver200Usd();
        if (achEligible > 100)
            recommendations.Add($"{achEligible} transactions > $200 could save ~${achEligible * 3:F0}/month with ACH.");

        return recommendations;
    }
}
```

---

## Cost Optimization Checklist

### Transaction Fees
- [ ] Route to cheapest eligible provider per transaction
- [ ] Offer ACH/bank transfer for large payments
- [ ] Negotiate volume discounts (>$100K/month)
- [ ] Use Interchange++ pricing at scale (Adyen, Stripe custom)
- [ ] Reduce chargebacks (each costs $15–$25 in fees)

### Infrastructure
- [ ] Right-size K8s pod resources based on actual usage
- [ ] Use spot instances for idempotent workloads
- [ ] Use serverless for event-driven processing (webhooks, notifications)
- [ ] Enable auto-scaling with appropriate thresholds
- [ ] Use reserved instances for steady-state workloads

### Data
- [ ] Use read replicas for reporting queries
- [ ] Archive old payment records to cold storage
- [ ] Cache exchange rates and configuration
- [ ] Use SQS long polling and batch operations
- [ ] Set DLQ retention policies

### Monitoring
- [ ] Set cloud budget alerts (80% and 100% thresholds)
- [ ] Track effective fee rate per provider
- [ ] Generate weekly cost reports with optimization recommendations
- [ ] Tag all resources for cost allocation

## Next Steps

- [Cloud-Native AWS](09-CLOUD-NATIVE-AWS.md)
- [Multi-Cloud Alternatives](10-MULTI-CLOUD-ALTERNATIVES.md)
- [Performance Optimization](08-PERFORMANCE-OPTIMIZATION.md)
- [Kubernetes Deployment](20-KUBERNETES-DEPLOYMENT.md)
