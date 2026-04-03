# E-Commerce & Subscription Glossary

A comprehensive reference of terms, abbreviations, and concepts used in e-commerce and subscription billing systems, with detailed explanations and .NET 10.0+ code examples.

## Table of Contents

1. [Dunning](#dunning)
2. [Subscription Lifecycle Terms](#subscription-lifecycle-terms)
3. [Billing & Pricing Terms](#billing--pricing-terms)
4. [Revenue Metrics (KPIs)](#revenue-metrics-kpis)
5. [Payment Processing Terms](#payment-processing-terms)
6. [Security & Compliance Terms](#security--compliance-terms)
7. [E-Commerce Flow Terms](#e-commerce-flow-terms)
8. [Quick Reference Abbreviations](#quick-reference-abbreviations)

---

## Dunning

**Dunning** is the process of systematically communicating with customers to collect overdue or failed payments. The term originates from 17th-century English (a "dun" was a creditor who persistently demanded payment). In modern SaaS and e-commerce, dunning refers to the automated workflow that retries failed charges, sends notification emails, and ultimately decides whether to suspend or cancel a subscription.

### Why Dunning Matters

Failed payments are a leading cause of **involuntary churn** — customers who didn't intend to cancel but lost access because their card expired, had insufficient funds, or their bank declined the transaction. A well-implemented dunning system can recover 20–40% of these failed payments before the subscription lapses.

### The Dunning Lifecycle

```
Payment fails
    │
    ▼
Enter dunning state (status: PastDue)
    │
    ├── Day 0:  Retry immediately (often succeeds for transient bank errors)
    │
    ├── Day 3:  Send "payment failed" email + retry
    │
    ├── Day 7:  Send "action required" email + retry
    │
    ├── Day 14: Send "final notice" email + retry
    │
    └── Day 21: Suspend / cancel subscription → involuntary churn
```

### Dunning Entities

```csharp
public enum DunningStatus
{
    Active,       // Currently in the dunning cycle
    Recovered,    // Payment succeeded during dunning
    Suspended,    // Access suspended, still retrying
    Cancelled     // Subscription cancelled after exhausting retries
}

public class DunningRecord
{
    public Guid Id { get; private set; }
    public Guid SubscriptionId { get; private set; }
    public int AttemptCount { get; private set; }
    public int MaxAttempts { get; private set; }
    public DateTime FirstFailedAt { get; private set; }
    public DateTime? LastAttemptAt { get; private set; }
    public DateTime? NextRetryAt { get; private set; }
    public DateTime? ResolvedAt { get; private set; }
    public DunningStatus Status { get; private set; }
    public string? LastFailureReason { get; private set; }

    public static DunningRecord Start(Guid subscriptionId, int maxAttempts = 4)
    {
        var record = new DunningRecord
        {
            Id = Guid.NewGuid(),
            SubscriptionId = subscriptionId,
            AttemptCount = 0,
            MaxAttempts = maxAttempts,
            FirstFailedAt = DateTime.UtcNow,
            Status = DunningStatus.Active
        };
        record.ScheduleNextRetry();
        return record;
    }

    public void RecordAttempt(bool succeeded, string? failureReason = null)
    {
        AttemptCount++;
        LastAttemptAt = DateTime.UtcNow;
        LastFailureReason = failureReason;

        if (succeeded)
        {
            Status = DunningStatus.Recovered;
            ResolvedAt = DateTime.UtcNow;
            NextRetryAt = null;
        }
        else if (AttemptCount >= MaxAttempts)
        {
            Status = DunningStatus.Cancelled;
            ResolvedAt = DateTime.UtcNow;
            NextRetryAt = null;
        }
        else
        {
            ScheduleNextRetry();
        }
    }

    private void ScheduleNextRetry()
    {
        // Exponential backoff: 3 days, 7 days, 14 days, 21 days
        var delayDays = AttemptCount switch
        {
            0 => 0,   // Immediate first retry
            1 => 3,
            2 => 7,
            3 => 14,
            _ => 21
        };
        NextRetryAt = DateTime.UtcNow.AddDays(delayDays);
    }
}
```

### Smart Retry Strategy

Not all payment failures are equal. A smart dunning engine classifies the decline code and chooses the best retry strategy:

```csharp
public enum DeclineCategory
{
    Transient,           // Try again soon (e.g., insufficient funds, bank timeout)
    CardExpired,         // Request card update; retry after customer action
    CardBlocked,         // Likely fraud; do not retry automatically
    InsufficientFunds,   // Retry at different time (e.g., after payday)
    DoNotHonor,          // Hard decline; do not retry
    Unknown
}

public static class DeclineClassifier
{
    // Stripe decline codes: https://stripe.com/docs/declines/codes
    public static DeclineCategory Classify(string declineCode) => declineCode switch
    {
        "insufficient_funds"           => DeclineCategory.InsufficientFunds,
        "do_not_honor"                 => DeclineCategory.DoNotHonor,
        "card_velocity_exceeded"       => DeclineCategory.CardBlocked,
        "lost_card" or "stolen_card"   => DeclineCategory.CardBlocked,
        "expired_card"                 => DeclineCategory.CardExpired,
        "card_not_supported"           => DeclineCategory.DoNotHonor,
        "processing_error"             => DeclineCategory.Transient,
        "try_again_later"              => DeclineCategory.Transient,
        _                              => DeclineCategory.Unknown
    };

    public static TimeSpan RecommendedDelay(DeclineCategory category) => category switch
    {
        DeclineCategory.Transient          => TimeSpan.FromHours(1),
        DeclineCategory.InsufficientFunds  => TimeSpan.FromDays(3),
        DeclineCategory.CardExpired        => TimeSpan.FromDays(7),  // Wait for update
        DeclineCategory.CardBlocked        => TimeSpan.MaxValue,      // Manual review only
        DeclineCategory.DoNotHonor         => TimeSpan.MaxValue,
        _                                  => TimeSpan.FromDays(3)
    };
}
```

### Dunning Service

```csharp
public interface IDunningService
{
    Task<DunningRecord> StartAsync(Guid subscriptionId, string declineCode);
    Task ProcessDueRetriesAsync(CancellationToken ct = default);
    Task<bool> AttemptRecoveryAsync(Guid dunningRecordId);
}

public class DunningService : IDunningService
{
    private readonly IDunningRepository _dunningRepo;
    private readonly ISubscriptionRepository _subscriptionRepo;
    private readonly IPaymentGateway _gateway;
    private readonly INotificationService _notifications;
    private readonly ILogger<DunningService> _logger;

    public DunningService(
        IDunningRepository dunningRepo,
        ISubscriptionRepository subscriptionRepo,
        IPaymentGateway gateway,
        INotificationService notifications,
        ILogger<DunningService> logger)
    {
        _dunningRepo = dunningRepo;
        _subscriptionRepo = subscriptionRepo;
        _gateway = gateway;
        _notifications = notifications;
        _logger = logger;
    }

    public async Task<DunningRecord> StartAsync(Guid subscriptionId, string declineCode)
    {
        var category = DeclineClassifier.Classify(declineCode);

        // Hard declines should never auto-retry
        if (category == DeclineCategory.CardBlocked || category == DeclineCategory.DoNotHonor)
        {
            _logger.LogWarning(
                "Hard decline for subscription {SubscriptionId}: {Code}",
                subscriptionId, declineCode);

            await _notifications.SendAsync(subscriptionId, NotificationType.HardDecline);
            throw new HardDeclineException(subscriptionId, declineCode);
        }

        var record = DunningRecord.Start(subscriptionId);
        await _dunningRepo.AddAsync(record);

        await _notifications.SendAsync(subscriptionId, NotificationType.PaymentFailed);

        _logger.LogInformation(
            "Started dunning for subscription {SubscriptionId}, next retry: {NextRetry}",
            subscriptionId, record.NextRetryAt);

        return record;
    }

    public async Task ProcessDueRetriesAsync(CancellationToken ct = default)
    {
        var dueRecords = await _dunningRepo.GetDueForRetryAsync(DateTime.UtcNow, ct);

        foreach (var record in dueRecords)
        {
            if (ct.IsCancellationRequested) break;
            await AttemptRecoveryAsync(record.Id);
        }
    }

    public async Task<bool> AttemptRecoveryAsync(Guid dunningRecordId)
    {
        var record = await _dunningRepo.GetByIdAsync(dunningRecordId)
            ?? throw new InvalidOperationException($"Dunning record {dunningRecordId} not found");

        var subscription = await _subscriptionRepo.GetByIdAsync(record.SubscriptionId)
            ?? throw new InvalidOperationException($"Subscription {record.SubscriptionId} not found");

        _logger.LogInformation(
            "Attempting dunning recovery for subscription {SubscriptionId} (attempt {Attempt}/{Max})",
            record.SubscriptionId, record.AttemptCount + 1, record.MaxAttempts);

        try
        {
            var result = await _gateway.ChargeAsync(new ChargeRequest
            {
                CustomerId = subscription.CustomerId,
                PaymentMethodId = subscription.DefaultPaymentMethodId,
                Amount = subscription.BillingAmount,
                Currency = subscription.Currency,
                IdempotencyKey = $"dunning-{record.Id}-attempt-{record.AttemptCount + 1}"
            });

            record.RecordAttempt(succeeded: result.IsSuccess);
            await _dunningRepo.UpdateAsync(record);

            if (result.IsSuccess)
            {
                await _subscriptionRepo.MarkAsActiveAsync(subscription.Id);
                await _notifications.SendAsync(subscription.Id, NotificationType.PaymentRecovered);
                _logger.LogInformation("Dunning recovery succeeded for {SubscriptionId}", subscription.Id);
                return true;
            }
        }
        catch (PaymentGatewayException ex)
        {
            var category = DeclineClassifier.Classify(ex.DeclineCode ?? "");
            record.RecordAttempt(succeeded: false, failureReason: ex.DeclineCode);
            await _dunningRepo.UpdateAsync(record);

            _logger.LogWarning(
                "Dunning retry failed for {SubscriptionId}: {Code}",
                subscription.Id, ex.DeclineCode);
        }

        if (record.Status == DunningStatus.Cancelled)
        {
            await _subscriptionRepo.CancelAsync(subscription.Id, CancellationReason.PaymentFailure);
            await _notifications.SendAsync(subscription.Id, NotificationType.SubscriptionCancelled);
        }
        else
        {
            await _notifications.SendAsync(subscription.Id,
                NotificationType.ForAttempt(record.AttemptCount));
        }

        return false;
    }
}
```

### Dunning Background Worker

```csharp
// Register as a hosted service in Program.cs:
// builder.Services.AddHostedService<DunningWorker>();

public class DunningWorker : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<DunningWorker> _logger;
    private static readonly TimeSpan Interval = TimeSpan.FromMinutes(30);

    public DunningWorker(IServiceScopeFactory scopeFactory, ILogger<DunningWorker> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await ProcessAsync(stoppingToken);
            await Task.Delay(Interval, stoppingToken);
        }
    }

    private async Task ProcessAsync(CancellationToken ct)
    {
        using var scope = _scopeFactory.CreateScope();
        var dunningService = scope.ServiceProvider.GetRequiredService<IDunningService>();

        try
        {
            await dunningService.ProcessDueRetriesAsync(ct);
        }
        catch (Exception ex) when (ex is not OperationCanceledException)
        {
            _logger.LogError(ex, "Error processing dunning retries");
        }
    }
}
```

### Notification Types

```csharp
public enum NotificationType
{
    PaymentFailed,        // Day 0 — first failure
    PaymentRetry1,        // Day 3 retry
    PaymentRetry2,        // Day 7 retry (with card-update link)
    FinalNotice,          // Day 14 retry (urgency)
    HardDecline,          // Card blocked/stolen — immediate human action needed
    PaymentRecovered,     // Dunning resolved
    SubscriptionCancelled // After all retries exhausted

}

public static class NotificationTypeExtensions
{
    public static NotificationType ForAttempt(int attemptCount) => attemptCount switch
    {
        1 => NotificationType.PaymentRetry1,
        2 => NotificationType.PaymentRetry2,
        3 => NotificationType.FinalNotice,
        _ => NotificationType.FinalNotice
    };
}
```

### Key Dunning Metrics

| Metric | Formula | Target |
|--------|---------|--------|
| Recovery Rate | Recovered / Total Entered Dunning | > 30% |
| Avg Recovery Attempt | Sum of attempts on recovered / recovered count | < 2.5 |
| Dunning Churn Rate | Cancelled in dunning / All churned | < 30% |
| Days to Recovery | Avg days from first failure to recovery | < 10 days |

---

## Subscription Lifecycle Terms

### **Trial** (Free Trial)

A time-limited period during which a customer can use a product for free (or at reduced cost) before committing to a paid subscription. Trials reduce purchase friction and allow customers to experience value before paying.

```csharp
public enum TrialType { None, Free, Paid }

public class Subscription
{
    public TrialType TrialType { get; private set; }
    public DateTime? TrialEndsAt { get; private set; }
    public SubscriptionStatus Status { get; private set; }

    public bool IsInTrial => TrialType != TrialType.None
        && TrialEndsAt.HasValue
        && DateTime.UtcNow < TrialEndsAt.Value;

    public static Subscription CreateWithTrial(
        Guid customerId, string planId, int trialDays)
    {
        return new Subscription
        {
            CustomerId = customerId,
            PlanId = planId,
            TrialType = TrialType.Free,
            TrialEndsAt = DateTime.UtcNow.AddDays(trialDays),
            Status = SubscriptionStatus.Trialing,
            CreatedAt = DateTime.UtcNow
        };
    }

    public Guid CustomerId { get; private set; }
    public string PlanId { get; private set; }
    public DateTime CreatedAt { get; private set; }
}
```

### **Freemium**

A hybrid model offering a permanently free tier with limited features alongside paid tiers with full functionality. Unlike a trial, freemium access does not expire. Revenue is generated by converting free users to paid tiers.

### **Grace Period**

A short window (typically 3–7 days) after a payment failure during which the customer retains full access while the dunning process begins. This prevents a bad customer experience from a temporary payment issue.

```csharp
public class SubscriptionAccessPolicy
{
    private readonly TimeSpan _gracePeriod = TimeSpan.FromDays(5);

    public bool HasAccess(Subscription subscription)
    {
        if (subscription.Status == SubscriptionStatus.Active) return true;

        if (subscription.Status == SubscriptionStatus.PastDue)
        {
            // Allow access within the grace period
            var gracePeriodEndsAt = subscription.LastPaymentFailedAt + _gracePeriod;
            return DateTime.UtcNow <= gracePeriodEndsAt;
        }

        return false;
    }
}
```

### **Subscription Status State Machine**

```
  Trialing ──────────────► Active ◄──────── Recovered (from PastDue)
      │                       │
      │ (trial ends,          │ (payment fails)
      │  no payment)          ▼
      │                   PastDue ──────────────► Cancelled
      │                       │                  (dunning exhausted)
      │                       │ (customer cancels)
      └──────────────────► Cancelled
                              ▲
                              │ (admin cancels)
                           Paused ◄──── Active (customer requested pause)
```

```csharp
public enum SubscriptionStatus
{
    Trialing,   // Customer in free/paid trial
    Active,     // Subscription current, paying
    PastDue,    // Payment failed, in dunning
    Paused,     // Voluntarily paused (billing suspended)
    Cancelled,  // Terminated — voluntary or involuntary
    Incomplete  // Awaiting initial payment confirmation (e.g., 3DS)
}
```

### **Proration**

The adjustment of charges or credits when a subscription changes mid-billing-cycle. For example, if a customer upgrades from a $10/month plan to a $20/month plan with 15 days remaining in their billing cycle, they are charged approximately $5 for the upgrade (half a month at the $10 difference).

```csharp
public class ProrationCalculator
{
    public ProrationResult Calculate(
        decimal oldPlanPrice,
        decimal newPlanPrice,
        DateTime cycleStartDate,
        DateTime cycleEndDate,
        DateTime changeDate)
    {
        var totalDays = (cycleEndDate - cycleStartDate).TotalDays;
        var remainingDays = (cycleEndDate - changeDate).TotalDays;
        var fraction = remainingDays / totalDays;

        var unusedOldPlan = oldPlanPrice * (decimal)fraction;
        var newPlanCost = newPlanPrice * (decimal)fraction;

        return new ProrationResult
        {
            CreditAmount = unusedOldPlan,
            ChargeAmount = newPlanCost,
            NetChargeAmount = Math.Max(0, newPlanCost - unusedOldPlan),
            NetCreditAmount = Math.Max(0, unusedOldPlan - newPlanCost)
        };
    }
}

public record ProrationResult(
    decimal CreditAmount,
    decimal ChargeAmount,
    decimal NetChargeAmount,
    decimal NetCreditAmount);
```

### **Billing Anchor**

The specific day of the month on which a subscription is billed. For example, a billing anchor of the 1st means all invoices are generated on the 1st of each month regardless of when the subscription started. This simplifies predictability for the customer.

```csharp
public class BillingAnchorService
{
    /// <summary>
    /// Returns the next billing date aligned to the anchor day.
    /// If today is the 15th and anchor is 1, next billing is the 1st of next month.
    /// </summary>
    public DateTime GetNextBillingDate(int anchorDay, DateTime from)
    {
        anchorDay = Math.Clamp(anchorDay, 1, 28); // Avoid month-length issues
        var candidate = new DateTime(from.Year, from.Month, anchorDay, 0, 0, 0, DateTimeKind.Utc);

        return candidate <= from
            ? candidate.AddMonths(1)
            : candidate;
    }
}
```

### **Churn**

The rate at which customers cancel or fail to renew subscriptions over a given period.

| Type | Description |
|------|-------------|
| **Voluntary Churn** | Customer actively cancels |
| **Involuntary Churn** | Subscription lost due to payment failure (mitigated by dunning) |
| **Revenue Churn** | Percentage of MRR lost (includes downgrades) |
| **Logo/Customer Churn** | Percentage of customers lost |

```csharp
public record ChurnMetrics(
    int TotalSubscriptionsAtStart,
    int VoluntaryCancellations,
    int InvoluntaryCancellations,
    decimal MrrAtStart,
    decimal MrrLost)
{
    public decimal CustomerChurnRate =>
        (VoluntaryCancellations + InvoluntaryCancellations) / (decimal)TotalSubscriptionsAtStart;

    public decimal RevenueChurnRate => MrrLost / MrrAtStart;
    public decimal InvoluntaryChurnRate => InvoluntaryCancellations / (decimal)TotalSubscriptionsAtStart;
}
```

### **Upgrade / Downgrade / Cross-sell**

| Term | Description |
|------|-------------|
| **Upsell** | Moving a customer to a higher-tier plan (more features, more revenue) |
| **Downsell** | Moving a customer to a lower-tier plan to prevent churn |
| **Cross-sell** | Selling an additional product/add-on alongside the existing subscription |
| **Expansion Revenue** | MRR gained from upsells and cross-sells from existing customers |

### **Cancellation Reason**

```csharp
public enum CancellationReason
{
    CustomerRequested,      // Voluntary — customer initiated
    PaymentFailure,         // Involuntary — dunning exhausted
    Fraud,                  // Admin action — fraudulent account
    TrialEnded,             // Trial expired with no payment method
    AdminInitiated,         // Manual admin action
    PlanDiscontinued        // Plan no longer offered
}
```

### **Pause**

A voluntary suspension of a subscription where billing stops for a defined period and the customer retains their account. Common for seasonal businesses or temporary financial hardship. Better for retention than cancellation.

```csharp
public class SubscriptionPauseService
{
    public async Task PauseAsync(Guid subscriptionId, DateTime resumeDate)
    {
        var sub = await _repository.GetByIdAsync(subscriptionId);

        if (sub.Status != SubscriptionStatus.Active)
            throw new InvalidOperationException("Only active subscriptions can be paused");

        if (resumeDate <= DateTime.UtcNow)
            throw new ArgumentException("Resume date must be in the future");

        sub.Pause(resumeDate);
        await _repository.UpdateAsync(sub);
        await _scheduler.ScheduleResumeAsync(subscriptionId, resumeDate);
    }
}
```

---

## Billing & Pricing Terms

### **Recurring Billing**

Automatic, periodic charges to a customer's payment method for ongoing access to a product or service. The billing interval is typically monthly or annual.

### **Invoice**

A formal document requesting payment for goods or services rendered. In subscription billing, invoices are generated automatically at the start of each billing cycle and may be paid immediately or sent to the customer with net payment terms.

```csharp
public class Invoice
{
    public Guid Id { get; init; }
    public Guid SubscriptionId { get; init; }
    public Guid CustomerId { get; init; }
    public InvoiceStatus Status { get; private set; }
    public decimal SubtotalAmount { get; init; }
    public decimal TaxAmount { get; init; }
    public decimal TotalAmount => SubtotalAmount + TaxAmount;
    public DateTime IssuedAt { get; init; }
    public DateTime DueAt { get; init; }
    public DateTime? PaidAt { get; private set; }
    public IReadOnlyList<InvoiceLineItem> LineItems { get; init; } = [];

    public void MarkPaid()
    {
        Status = InvoiceStatus.Paid;
        PaidAt = DateTime.UtcNow;
    }
}

public enum InvoiceStatus { Draft, Open, Paid, Void, Uncollectible }
```

### **Credit Note (Credit Memo)**

A document issued to reduce the amount a customer owes, typically issued for refunds, billing errors, or goodwill adjustments. A credit note offsets future invoices.

### **Deferred Revenue**

Revenue received but not yet earned — e.g., a customer who pays annually up front. Under ASC 606 / IFRS 15, revenue is recognized over the subscription period, not at payment time. The unearned portion appears as a liability on the balance sheet.

```csharp
// ASC 606 revenue recognition: recognize monthly share of annual prepayment
public class RevenueRecognitionService
{
    public decimal CalculateRecognizedRevenue(
        decimal totalAnnualPayment,
        DateTime subscriptionStart,
        DateTime periodStart,
        DateTime periodEnd)
    {
        var totalDays = (subscriptionStart.AddYears(1) - subscriptionStart).TotalDays;
        var periodDays = (periodEnd - periodStart).TotalDays;
        return totalAnnualPayment * (decimal)(periodDays / totalDays);
    }

    public decimal CalculateDeferredRevenue(
        decimal totalAnnualPayment,
        DateTime subscriptionStart,
        DateTime asOfDate)
    {
        var remainingDays = (subscriptionStart.AddYears(1) - asOfDate).TotalDays;
        var totalDays = (subscriptionStart.AddYears(1) - subscriptionStart).TotalDays;
        return totalAnnualPayment * (decimal)Math.Max(0, remainingDays / totalDays);
    }
}
```

### **Metered / Usage-Based Billing**

A billing model where charges are based on actual usage (e.g., API calls, GB stored, seats used) rather than a flat fee. Usage is aggregated over a billing period and charged at period end.

```csharp
public class UsageRecord
{
    public Guid SubscriptionId { get; init; }
    public string MetricName { get; init; } = "";    // e.g., "api_calls"
    public long Quantity { get; init; }
    public DateTime RecordedAt { get; init; }
}

public class MeteredBillingService
{
    public async Task<decimal> CalculateChargeAsync(
        Guid subscriptionId,
        string metricName,
        DateTime periodStart,
        DateTime periodEnd,
        decimal pricePerUnit)
    {
        var totalUsage = await _usageRepository.SumAsync(
            subscriptionId, metricName, periodStart, periodEnd);
        return totalUsage * pricePerUnit;
    }
}
```

### **Tiered Pricing**

Different unit prices apply at different volume thresholds.

| Model | Description | Example |
|-------|-------------|---------|
| **Volume** | All units at the price of the highest reached tier | 1–10 @ $1.00, 11–100 @ $0.80 → buy 50 = 50 × $0.80 |
| **Graduated** | Each tier applies only to units within that tier | 1–10 @ $1.00, 11–100 @ $0.80 → buy 50 = (10 × $1.00) + (40 × $0.80) |
| **Flat + Usage** | Fixed base fee plus usage charges | $49/month + $0.01 per API call |

```csharp
public static class TieredPricing
{
    public record PriceTier(int UpTo, decimal UnitPrice);

    // Graduated: each band is priced at its own rate
    public static decimal CalculateGraduated(long quantity, IEnumerable<PriceTier> tiers)
    {
        decimal total = 0;
        long remaining = quantity;
        long previousUpTo = 0;

        foreach (var tier in tiers.OrderBy(t => t.UpTo))
        {
            long bandSize = tier.UpTo - previousUpTo;
            long units = Math.Min(remaining, bandSize);
            total += units * tier.UnitPrice;
            remaining -= units;
            previousUpTo = tier.UpTo;
            if (remaining <= 0) break;
        }

        return total;
    }
}
```

### **Net Terms (Net 30 / Net 60)**

Payment terms specifying that a customer has 30 or 60 days from the invoice date to pay. Common in B2B contexts. The invoice is marked as "open" until paid or overdue.

---

## Revenue Metrics (KPIs)

### **MRR** — Monthly Recurring Revenue

The predictable, normalized monthly revenue from all active subscriptions. Annual subscriptions are divided by 12 to get their monthly contribution.

```
MRR = Σ (subscription_monthly_price for all Active subscriptions)
```

```csharp
public class MrrCalculator
{
    public decimal Calculate(IEnumerable<Subscription> activeSubscriptions)
    {
        return activeSubscriptions.Sum(s =>
            s.BillingInterval == BillingInterval.Annual
                ? s.Price / 12m
                : s.Price);
    }
}
```

### **ARR** — Annual Recurring Revenue

`ARR = MRR × 12`

The annualized version of MRR. Used for high-level financial planning and investor metrics.

### **MRR Movements**

| Movement | Description |
|----------|-------------|
| **New MRR** | MRR from newly acquired customers |
| **Expansion MRR** | MRR added from upgrades/cross-sells to existing customers |
| **Contraction MRR** | MRR lost from downgrades |
| **Churned MRR** | MRR lost from cancellations |
| **Net New MRR** | New + Expansion − Contraction − Churned |
| **Reactivation MRR** | MRR from previously churned customers who re-subscribed |

### **LTV** — Customer Lifetime Value (also **CLV**)

The total revenue expected from a customer over their entire relationship with the business.

```
LTV = ARPU / Churn Rate
```
- **ARPU** = Average Revenue Per User per month
- **Churn Rate** = monthly percentage of customers who cancel

```csharp
public record LtvMetrics(decimal Arpu, decimal MonthlyChurnRate)
{
    public decimal Ltv => MonthlyChurnRate > 0 ? Arpu / MonthlyChurnRate : 0;
    public decimal LtvCacRatio(decimal cac) => cac > 0 ? Ltv / cac : 0;
}
```

### **CAC** — Customer Acquisition Cost

The total cost to acquire a new customer: marketing spend + sales costs divided by the number of new customers acquired.

```
CAC = Total Sales & Marketing Spend / Number of New Customers Acquired
```

A healthy SaaS business targets **LTV:CAC > 3:1**.

### **NDR** — Net Dollar Retention (also **NRR** — Net Revenue Retention)

Measures how much revenue is retained from existing customers after accounting for churn and expansions. NDR > 100% means expansion revenue outpaces churn.

```
NDR = (Starting MRR + Expansion MRR − Contraction MRR − Churned MRR) / Starting MRR × 100%
```

### **GRR** — Gross Revenue Retention

Retention rate counting only losses (churn + contraction), ignoring expansion. Always ≤ 100%.

```
GRR = (Starting MRR − Contraction MRR − Churned MRR) / Starting MRR × 100%
```

### **ARPU** — Average Revenue Per User

`ARPU = MRR / Total Active Customers`

---

## Payment Processing Terms

### **Authorization**

A hold placed on a customer's funds to verify their payment method is valid and has sufficient funds. The funds are reserved but not yet transferred. Authorizations typically expire after 5–7 days if not captured.

### **Capture**

The act of actually collecting the authorized funds. In most card flows, authorization and capture happen together ("auth and capture"). In some flows (e.g., hotels, rentals), they are separate steps.

```csharp
// Two-step auth + capture with Stripe
public async Task<string> AuthorizeAsync(decimal amount, string paymentMethodId)
{
    var options = new PaymentIntentCreateOptions
    {
        Amount = (long)(amount * 100),
        Currency = "usd",
        PaymentMethod = paymentMethodId,
        CaptureMethod = "manual",   // Authorize only
        Confirm = true
    };
    var intent = await _paymentIntentService.CreateAsync(options);
    return intent.Id;
}

public async Task CaptureAsync(string paymentIntentId)
{
    await _paymentIntentService.CaptureAsync(paymentIntentId);
}
```

### **Settlement**

The transfer of funds from the customer's bank (issuing bank) to the merchant's bank account (acquiring bank). Settlement typically takes 1–3 business days after capture.

### **Refund**

The return of funds to a customer, either full or partial, after a completed payment.

```csharp
public async Task<RefundResult> RefundAsync(string chargeId, decimal? amount = null)
{
    var options = new RefundCreateOptions
    {
        Charge = chargeId,
        Amount = amount.HasValue ? (long)(amount.Value * 100) : null  // null = full refund
    };
    var refund = await _refundService.CreateAsync(options);
    return new RefundResult(refund.Id, refund.Status == "succeeded");
}
```

### **Chargeback**

A forced reversal of a payment initiated by the cardholder through their bank, bypassing the merchant. Chargebacks are costly (the merchant pays a ~$15–$100 dispute fee even if they win). High chargeback rates can result in account termination.

### **Dispute**

The formal process by which a merchant contests a chargeback. The merchant submits evidence (receipts, delivery confirmation, etc.) to the payment processor within a deadline (typically 7–21 days).

### **Idempotency Key**

A unique string attached to a payment API request that guarantees the operation is executed at most once, even if the request is retried. If the same idempotency key is submitted twice, the second call returns the result of the first without creating a new charge.

```csharp
public async Task<PaymentResult> ProcessWithIdempotencyAsync(
    string orderId,
    decimal amount,
    string paymentMethodId)
{
    // Key scoped to the order ensures at-most-once charge
    var idempotencyKey = $"order-{orderId}-payment";

    var options = new PaymentIntentCreateOptions
    {
        Amount = (long)(amount * 100),
        Currency = "usd",
        PaymentMethod = paymentMethodId,
        Confirm = true
    };

    var requestOptions = new RequestOptions { IdempotencyKey = idempotencyKey };
    var intent = await _paymentIntentService.CreateAsync(options, requestOptions);

    return new PaymentResult(intent.Id, intent.Status == "succeeded");
}
```

### **Webhook**

An HTTP callback sent by a payment provider to notify your system of asynchronous events (e.g., `payment_intent.succeeded`, `invoice.payment_failed`, `customer.subscription.deleted`). Webhooks must be verified with a signature to prevent spoofing.

### **Payment Intent**

Stripe's object representing a customer's intent to pay. It tracks the lifecycle of a payment and handles complexities like 3D Secure authentication. Other providers have equivalent concepts (PayPal Orders, Adyen PaymentSession).

### **Payment Method**

A customer's stored payment instrument: credit/debit card, bank account, digital wallet, etc. Stored as a token by the payment provider — your system never sees raw card data.

### **Customer (Provider Object)**

A record stored at the payment provider level that holds a customer's payment methods, billing details, and transaction history. Linking your customer ID to the provider's customer object enables saved payment methods, subscriptions, and future charges.

```csharp
public async Task<string> GetOrCreateProviderCustomerAsync(
    Guid appCustomerId,
    string email)
{
    var existing = await _customerMappingRepo.GetProviderIdAsync(appCustomerId);
    if (existing is not null) return existing;

    var providerCustomer = await _customerService.CreateAsync(new CustomerCreateOptions
    {
        Email = email,
        Metadata = new Dictionary<string, string>
        {
            ["app_customer_id"] = appCustomerId.ToString()
        }
    });

    await _customerMappingRepo.SaveAsync(appCustomerId, providerCustomer.Id);
    return providerCustomer.Id;
}
```

### **Gateway vs. Processor vs. Acquirer**

| Term | Role |
|------|------|
| **Payment Gateway** | Software that securely transmits payment data between merchant and processor (e.g., Stripe, PayPal) |
| **Payment Processor** | Handles the actual transaction between the merchant's bank and cardholder's bank |
| **Acquirer (Acquiring Bank)** | The merchant's bank that receives settled funds |
| **Issuer (Issuing Bank)** | The cardholder's bank that approves or declines transactions |
| **Interchange** | The fee paid by the acquirer to the issuer for each transaction (~1–3% of transaction value) |
| **MDR (Merchant Discount Rate)** | The total fee a merchant pays the payment provider, which includes interchange + markup |

---

## Security & Compliance Terms

### **PCI DSS** — Payment Card Industry Data Security Standard

A set of security standards required for any organization that processes, stores, or transmits cardholder data. Using a hosted payment form (Stripe.js, PayPal JS SDK) moves most PCI scope off your servers. Self-Assessment Questionnaire (SAQ) A is applicable to merchants who fully outsource card processing.

### **Tokenization**

The process of replacing sensitive card data (PAN, CVV) with a non-sensitive placeholder token. The token is meaningless outside the tokenization system and is safe to store in your database.

### **PAN** — Primary Account Number

The 16-digit card number embossed on a credit/debit card. You must never log, store, or transmit PANs on your servers.

### **CVV / CVC** — Card Verification Value / Code

The 3 or 4 digit security code on a card. Verifies physical possession of the card. Under PCI DSS, you may never store CVV after authorization.

### **BIN** — Bank Identification Number (also **IIN** — Issuer Identification Number)

The first 6–8 digits of a card number that identify the issuing bank and card type. Used for routing, risk assessment, and determining interchange fees.

### **SCA** — Strong Customer Authentication

An EU regulation (PSD2) requiring two-factor authentication for online payments. Requires two of: something you know (PIN), something you have (device), something you are (biometric). Implemented via 3D Secure.

### **3D Secure (3DS / 3DS2)**

A protocol adding an additional authentication step during online card transactions. The "3 domains" are: issuer domain, acquirer domain, and interoperability domain. 3DS2 uses behavioral data to make most transactions frictionless.

```csharp
// Handle 3DS challenge requirement from Stripe
if (paymentIntent.Status == "requires_action"
    && paymentIntent.NextAction?.Type == "use_stripe_sdk")
{
    // Return 3DS URL to client for browser-based authentication
    return new PaymentResult
    {
        RequiresAction = true,
        ClientSecret = paymentIntent.ClientSecret
    };
}
```

### **Fraud Score / Risk Score**

A numeric score computed by the payment provider or a fraud detection service that estimates the likelihood a transaction is fraudulent. Higher scores trigger additional verification or automatic decline.

---

## E-Commerce Flow Terms

### **Cart Abandonment**

When a customer adds items to their shopping cart but does not complete the checkout process. Average cart abandonment rates are ~70%. Recovery emails and one-click checkout reduce abandonment.

### **Checkout**

The multi-step process of completing a purchase: review cart → enter shipping → enter payment → confirm order. Hosted checkout pages (Stripe Checkout, PayPal Checkout) offload PCI complexity.

### **Order**

A record of a customer's purchase, including items, quantities, prices, and fulfillment status. Distinct from a payment — an order may have multiple payment attempts, partial fulfillments, or returns.

### **Fulfillment**

The process of delivering purchased goods or services to the customer. For physical goods: pick, pack, ship, deliver. For digital goods: provision access, send license key, activate subscription.

### **SKU** — Stock Keeping Unit

A unique identifier for a product variant (e.g., "SHIRT-RED-M" = red shirt, medium). SKUs enable inventory tracking per variant.

### **BNPL** — Buy Now, Pay Later

A short-term financing option where a customer pays for a purchase in installments (typically 4 interest-free payments over 6 weeks). Providers: Klarna, Afterpay, Affirm. The merchant receives full payment immediately; the BNPL provider collects installments.

### **SaaS** — Software as a Service

A software delivery model where the application is hosted in the cloud and sold via subscription rather than perpetual license. Examples: Slack, Salesforce, GitHub. Billing is typically monthly or annual MRR.

### **B2B / B2C / B2B2C**

| Term | Meaning |
|------|---------|
| **B2C** | Business to Consumer — selling directly to individuals |
| **B2B** | Business to Business — selling to organizations; often involves net terms, invoicing, purchase orders |
| **B2B2C** | Business to Business to Consumer — selling through an intermediary business to end consumers |

### **AOV** — Average Order Value

`AOV = Total Revenue / Number of Orders`

Increasing AOV (via upsells, bundles, free-shipping thresholds) increases revenue without acquiring more customers.

### **Conversion Rate**

The percentage of visitors (or trials) who complete a desired action (purchase, subscription). Improving conversion rate from checkout to paid is a high-leverage growth lever.

```
Checkout Conversion Rate = Completed Orders / Checkout Initiations × 100%
Trial-to-Paid Conversion Rate = Paid Conversions / Trial Starts × 100%
```

### **Reconciliation**

The process of matching payment provider records against your internal database to ensure every charge, refund, and payout is accounted for. Critical for accounting accuracy and fraud detection.

```csharp
public record ReconciliationResult(
    int MatchedCount,
    int MissingInDatabase,
    int MissingInProvider,
    IReadOnlyList<ReconciliationDiscrepancy> Discrepancies);

public class PaymentReconciliationService
{
    public async Task<ReconciliationResult> ReconcileAsync(
        DateOnly date, CancellationToken ct = default)
    {
        var dbPayments = await _paymentRepo.GetByDateAsync(date, ct);
        var providerPayments = await _gateway.ListTransactionsAsync(date, ct);

        var dbIds = dbPayments.Select(p => p.ProviderTransactionId).ToHashSet();
        var providerIds = providerPayments.Select(p => p.Id).ToHashSet();

        var missingInDb = providerIds.Except(dbIds).ToList();
        var missingInProvider = dbIds.Except(providerIds).ToList();
        var discrepancies = new List<ReconciliationDiscrepancy>();

        foreach (var id in dbIds.Intersect(providerIds))
        {
            var db = dbPayments.First(p => p.ProviderTransactionId == id);
            var prov = providerPayments.First(p => p.Id == id);

            if (db.Amount != prov.Amount || db.Status != prov.Status)
                discrepancies.Add(new ReconciliationDiscrepancy(id, db, prov));
        }

        return new ReconciliationResult(
            dbIds.Intersect(providerIds).Count() - discrepancies.Count,
            missingInDb.Count,
            missingInProvider.Count,
            discrepancies);
    }
}
```

---

## Quick Reference Abbreviations

| Abbreviation | Full Term | Context |
|---|---|---|
| **3DS / 3DS2** | 3-Domain Secure | Authentication protocol for online card payments |
| **ACH** | Automated Clearing House | US electronic bank transfer network |
| **AOV** | Average Order Value | E-commerce revenue metric |
| **ARR** | Annual Recurring Revenue | SaaS financial metric |
| **ARPU** | Average Revenue Per User | Unit economics metric |
| **BIN / IIN** | Bank / Issuer Identification Number | First digits of a card number |
| **BNPL** | Buy Now, Pay Later | Short-term installment financing |
| **B2B** | Business to Business | Sales channel descriptor |
| **B2C** | Business to Consumer | Sales channel descriptor |
| **CAC** | Customer Acquisition Cost | Unit economics metric |
| **CVV / CVC** | Card Verification Value / Code | Card security code |
| **EFT** | Electronic Funds Transfer | Umbrella term for digital money movement |
| **EMV** | Europay Mastercard Visa | Global chip card standard |
| **GRR** | Gross Revenue Retention | Subscription retention metric (losses only) |
| **LTV / CLV** | (Customer) Lifetime Value | Unit economics metric |
| **MDR** | Merchant Discount Rate | Total fee merchants pay for card acceptance |
| **MRR** | Monthly Recurring Revenue | Core SaaS billing metric |
| **NDR / NRR** | Net Dollar / Revenue Retention | Subscription retention metric (incl. expansion) |
| **PAN** | Primary Account Number | 16-digit card number |
| **PCI DSS** | Payment Card Industry Data Security Standard | Compliance standard for card data |
| **PSD2** | Payment Services Directive 2 | EU regulation requiring SCA |
| **SAQ** | Self-Assessment Questionnaire | PCI DSS compliance self-audit |
| **SCA** | Strong Customer Authentication | EU two-factor payment requirement |
| **SEPA** | Single Euro Payments Area | EU bank transfer network |
| **SKU** | Stock Keeping Unit | Product variant identifier |
| **SaaS** | Software as a Service | Cloud software delivery model |

---

## See Also

- [00-OVERVIEW](00-OVERVIEW.md) — Payment system fundamentals
- [18-COMMON-SCENARIOS](18-COMMON-SCENARIOS.md) — Subscription and checkout implementation examples
- [04-WEBHOOK-PATTERNS](04-WEBHOOK-PATTERNS.md) — Handling `invoice.payment_failed` and dunning webhooks
- [05-ERROR-HANDLING](05-ERROR-HANDLING.md) — Handling payment decline errors
- [16-STRIPE-INTEGRATION](16-STRIPE-INTEGRATION.md) — Stripe subscription API reference
