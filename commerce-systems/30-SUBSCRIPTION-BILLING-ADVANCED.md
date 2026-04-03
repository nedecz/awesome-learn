# Advanced Subscription Billing

## Overview

Comprehensive deep-dive into building a production-grade subscription billing system covering plan modeling, usage-based and seat-based pricing, proration, trial management, dunning and payment recovery, lifecycle events, invoice generation, revenue analytics, multi-currency support, and provider integration — all with .NET/C# implementation examples using domain-driven design.

> **Related documents:**
>
> - [16-STRIPE-INTEGRATION](16-STRIPE-INTEGRATION.md) — Stripe subscription APIs
> - [17-PAYPAL-INTEGRATION](17-PAYPAL-INTEGRATION.md) — PayPal subscription APIs
> - [25-ECOMMERCE-SUBSCRIPTION-GLOSSARY](25-ECOMMERCE-SUBSCRIPTION-GLOSSARY.md) — Subscription terminology
> - [18-COMMON-SCENARIOS](18-COMMON-SCENARIOS.md) — Subscription billing scenarios

## Table of Contents

1. [Subscription Data Model](#1-subscription-data-model)
2. [Usage-Based & Metered Billing](#2-usage-based--metered-billing)
3. [Seat-Based Pricing](#3-seat-based-pricing)
4. [Proration Engine](#4-proration-engine)
5. [Trial Management](#5-trial-management)
6. [Dunning & Payment Recovery](#6-dunning--payment-recovery)
7. [Subscription Lifecycle Events](#7-subscription-lifecycle-events)
8. [Invoice Generation](#8-invoice-generation)
9. [Revenue Metrics & Analytics](#9-revenue-metrics--analytics)
10. [Multi-Currency Subscriptions](#10-multi-currency-subscriptions)
11. [Provider Integration](#11-provider-integration)
12. [Implementation Checklist](#12-implementation-checklist)

---

## 1. Subscription Data Model

### Core Entities

A subscription billing system is built around a small number of interconnected entities: **Plan**, **PricingTier**, **Subscription**, **SubscriptionItem**, and **BillingCycle**. These form the aggregate root for all billing operations.

```csharp
public enum BillingInterval
{
    Daily,
    Weekly,
    Monthly,
    Quarterly,
    SemiAnnual,
    Annual
}

public enum PricingModel
{
    Flat,       // Single fixed price regardless of quantity
    Tiered,     // Price per unit decreases at volume breakpoints
    Volume,     // All units priced at the tier reached
    Graduated,  // Each tier range priced independently (Stripe-style)
    Staircase   // Price jumps at thresholds, flat within each step
}

public enum SubscriptionStatus
{
    Trialing,
    Active,
    PastDue,
    Paused,
    Cancelled,
    Expired,
    Incomplete
}
```

### Plan and Pricing Tier

```csharp
public class Plan
{
    public Guid Id { get; private set; }
    public string Name { get; private set; } = default!;
    public string ExternalId { get; private set; } = default!;
    public PricingModel PricingModel { get; private set; }
    public BillingInterval BillingInterval { get; private set; }
    public int IntervalCount { get; private set; } = 1;
    public string Currency { get; private set; } = "USD";
    public decimal BasePrice { get; private set; }
    public bool IsMetered { get; private set; }
    public int? TrialPeriodDays { get; private set; }
    public int? MinimumQuantity { get; private set; }
    public int? MaximumQuantity { get; private set; }
    public bool IsActive { get; private set; } = true;
    public DateTime CreatedAt { get; private set; }

    private readonly List<PricingTier> _tiers = new();
    public IReadOnlyCollection<PricingTier> Tiers => _tiers.AsReadOnly();

    public Plan(string name, PricingModel pricingModel,
                BillingInterval interval, decimal basePrice, string currency)
    {
        Id = Guid.NewGuid();
        Name = name;
        ExternalId = $"plan_{Guid.NewGuid():N}"[..20];
        PricingModel = pricingModel;
        BillingInterval = interval;
        BasePrice = basePrice;
        Currency = currency;
        CreatedAt = DateTime.UtcNow;
    }

    public void AddTier(int upTo, decimal unitPrice, decimal? flatFee = null)
    {
        _tiers.Add(new PricingTier
        {
            Id = Guid.NewGuid(),
            PlanId = Id,
            UpTo = upTo,
            UnitPrice = unitPrice,
            FlatFee = flatFee ?? 0m,
            Order = _tiers.Count + 1
        });
    }
}

public class PricingTier
{
    public Guid Id { get; set; }
    public Guid PlanId { get; set; }
    public int UpTo { get; set; }       // 0 = unlimited
    public decimal UnitPrice { get; set; }
    public decimal FlatFee { get; set; }
    public int Order { get; set; }
}
```

### Subscription and SubscriptionItem

```csharp
public class Subscription
{
    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    public Guid PlanId { get; private set; }
    public Plan Plan { get; private set; } = default!;
    public SubscriptionStatus Status { get; private set; }
    public DateTime StartDate { get; private set; }
    public DateTime CurrentPeriodStart { get; private set; }
    public DateTime CurrentPeriodEnd { get; private set; }
    public DateTime? TrialStart { get; private set; }
    public DateTime? TrialEnd { get; private set; }
    public DateTime? CancelledAt { get; private set; }
    public DateTime? CancelAt { get; private set; }  // scheduled cancellation
    public bool CancelAtPeriodEnd { get; private set; }
    public int BillingAnchorDay { get; private set; }
    public string Currency { get; private set; } = "USD";
    public Guid? DefaultPaymentMethodId { get; private set; }
    public DateTime CreatedAt { get; private set; }

    private readonly List<SubscriptionItem> _items = new();
    public IReadOnlyCollection<SubscriptionItem> Items => _items.AsReadOnly();

    private readonly List<BillingCycle> _billingCycles = new();
    public IReadOnlyCollection<BillingCycle> BillingCycles => _billingCycles.AsReadOnly();

    public static Subscription Create(Guid customerId, Plan plan,
        int? billingAnchorDay = null, int? trialDays = null)
    {
        var now = DateTime.UtcNow;
        var effectiveTrialDays = trialDays ?? plan.TrialPeriodDays;
        var sub = new Subscription
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            PlanId = plan.Id,
            Plan = plan,
            Currency = plan.Currency,
            BillingAnchorDay = billingAnchorDay ?? now.Day,
            StartDate = now,
            CreatedAt = now
        };

        if (effectiveTrialDays.HasValue && effectiveTrialDays > 0)
        {
            sub.Status = SubscriptionStatus.Trialing;
            sub.TrialStart = now;
            sub.TrialEnd = now.AddDays(effectiveTrialDays.Value);
            sub.CurrentPeriodStart = now;
            sub.CurrentPeriodEnd = sub.TrialEnd.Value;
        }
        else
        {
            sub.Status = SubscriptionStatus.Active;
            sub.CurrentPeriodStart = now;
            sub.CurrentPeriodEnd = CalculatePeriodEnd(
                now, plan.BillingInterval, plan.IntervalCount);
        }

        return sub;
    }

    public void AddItem(Guid planId, int quantity = 1, decimal? unitPrice = null)
    {
        _items.Add(new SubscriptionItem
        {
            Id = Guid.NewGuid(),
            SubscriptionId = Id,
            PlanId = planId,
            Quantity = quantity,
            UnitPriceOverride = unitPrice,
            CreatedAt = DateTime.UtcNow
        });
    }

    public void Cancel(bool atPeriodEnd = true)
    {
        CancelledAt = DateTime.UtcNow;
        if (atPeriodEnd)
        {
            CancelAtPeriodEnd = true;
            CancelAt = CurrentPeriodEnd;
        }
        else
        {
            Status = SubscriptionStatus.Cancelled;
            CancelAt = DateTime.UtcNow;
        }
    }

    public void Pause()
    {
        if (Status != SubscriptionStatus.Active)
            throw new InvalidOperationException(
                $"Cannot pause subscription in {Status} state.");
        Status = SubscriptionStatus.Paused;
    }

    public void Resume()
    {
        if (Status != SubscriptionStatus.Paused)
            throw new InvalidOperationException(
                $"Cannot resume subscription in {Status} state.");
        Status = SubscriptionStatus.Active;
    }

    private static DateTime CalculatePeriodEnd(
        DateTime start, BillingInterval interval, int count)
    {
        return interval switch
        {
            BillingInterval.Daily => start.AddDays(count),
            BillingInterval.Weekly => start.AddDays(7 * count),
            BillingInterval.Monthly => start.AddMonths(count),
            BillingInterval.Quarterly => start.AddMonths(3 * count),
            BillingInterval.SemiAnnual => start.AddMonths(6 * count),
            BillingInterval.Annual => start.AddYears(count),
            _ => throw new ArgumentOutOfRangeException(nameof(interval))
        };
    }
}

public class SubscriptionItem
{
    public Guid Id { get; set; }
    public Guid SubscriptionId { get; set; }
    public Guid PlanId { get; set; }
    public int Quantity { get; set; }
    public decimal? UnitPriceOverride { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

### Billing Cycle

```csharp
public class BillingCycle
{
    public Guid Id { get; set; }
    public Guid SubscriptionId { get; set; }
    public int CycleNumber { get; set; }
    public DateTime PeriodStart { get; set; }
    public DateTime PeriodEnd { get; set; }
    public decimal AmountDue { get; set; }
    public decimal AmountPaid { get; set; }
    public BillingCycleStatus Status { get; set; }
    public Guid? InvoiceId { get; set; }
    public DateTime CreatedAt { get; set; }
}

public enum BillingCycleStatus
{
    Pending,
    Invoiced,
    Paid,
    PastDue,
    Void
}
```

### EF Core Configuration

```csharp
public class SubscriptionDbContext : DbContext
{
    public DbSet<Plan> Plans => Set<Plan>();
    public DbSet<PricingTier> PricingTiers => Set<PricingTier>();
    public DbSet<Subscription> Subscriptions => Set<Subscription>();
    public DbSet<SubscriptionItem> SubscriptionItems => Set<SubscriptionItem>();
    public DbSet<BillingCycle> BillingCycles => Set<BillingCycle>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Plan>(b =>
        {
            b.ToTable("plans");
            b.HasKey(p => p.Id);
            b.Property(p => p.BasePrice).HasPrecision(18, 4);
            b.Property(p => p.Currency).HasMaxLength(3);
            b.HasMany(p => p.Tiers)
             .WithOne()
             .HasForeignKey(t => t.PlanId)
             .OnDelete(DeleteBehavior.Cascade);
            b.HasIndex(p => p.ExternalId).IsUnique();
        });

        modelBuilder.Entity<PricingTier>(b =>
        {
            b.ToTable("pricing_tiers");
            b.HasKey(t => t.Id);
            b.Property(t => t.UnitPrice).HasPrecision(18, 4);
            b.Property(t => t.FlatFee).HasPrecision(18, 4);
        });

        modelBuilder.Entity<Subscription>(b =>
        {
            b.ToTable("subscriptions");
            b.HasKey(s => s.Id);
            b.Property(s => s.Currency).HasMaxLength(3);
            b.HasOne(s => s.Plan).WithMany().HasForeignKey(s => s.PlanId);
            b.HasMany(s => s.Items)
             .WithOne()
             .HasForeignKey(i => i.SubscriptionId)
             .OnDelete(DeleteBehavior.Cascade);
            b.HasMany(s => s.BillingCycles)
             .WithOne()
             .HasForeignKey(c => c.SubscriptionId)
             .OnDelete(DeleteBehavior.Cascade);
            b.HasIndex(s => new { s.CustomerId, s.Status });
        });

        modelBuilder.Entity<SubscriptionItem>(b =>
        {
            b.ToTable("subscription_items");
            b.HasKey(i => i.Id);
            b.Property(i => i.UnitPriceOverride).HasPrecision(18, 4);
        });

        modelBuilder.Entity<BillingCycle>(b =>
        {
            b.ToTable("billing_cycles");
            b.HasKey(c => c.Id);
            b.Property(c => c.AmountDue).HasPrecision(18, 4);
            b.Property(c => c.AmountPaid).HasPrecision(18, 4);
            b.HasIndex(c => new { c.SubscriptionId, c.CycleNumber }).IsUnique();
        });
    }
}
```

### Pricing Calculation Engine

```csharp
public class PricingCalculator
{
    public decimal Calculate(Plan plan, int quantity)
    {
        return plan.PricingModel switch
        {
            PricingModel.Flat => plan.BasePrice,
            PricingModel.Tiered => CalculateTiered(plan, quantity),
            PricingModel.Volume => CalculateVolume(plan, quantity),
            PricingModel.Graduated => CalculateGraduated(plan, quantity),
            PricingModel.Staircase => CalculateStaircase(plan, quantity),
            _ => throw new ArgumentOutOfRangeException()
        };
    }

    /// <summary>
    /// Volume: all units priced at the tier the total quantity falls into.
    /// </summary>
    private decimal CalculateVolume(Plan plan, int quantity)
    {
        var tier = plan.Tiers
            .OrderBy(t => t.Order)
            .FirstOrDefault(t => t.UpTo == 0 || quantity <= t.UpTo);

        if (tier is null)
            tier = plan.Tiers.OrderByDescending(t => t.Order).First();

        return (quantity * tier.UnitPrice) + tier.FlatFee;
    }

    /// <summary>
    /// Graduated (Stripe-style tiered): each range of units is priced at
    /// the rate for that tier. E.g. first 10 at $10, next 40 at $8, rest at $5.
    /// </summary>
    private decimal CalculateGraduated(Plan plan, int quantity)
    {
        var total = 0m;
        var remaining = quantity;
        var previousUpperBound = 0;

        foreach (var tier in plan.Tiers.OrderBy(t => t.Order))
        {
            var tierSize = tier.UpTo == 0
                ? remaining
                : Math.Min(remaining, tier.UpTo - previousUpperBound);

            if (tierSize <= 0) break;

            total += (tierSize * tier.UnitPrice) + tier.FlatFee;
            remaining -= tierSize;
            previousUpperBound = tier.UpTo;
        }

        return total;
    }

    /// <summary>
    /// Tiered: price per unit decreases at breakpoints. Alias for Graduated.
    /// </summary>
    private decimal CalculateTiered(Plan plan, int quantity)
        => CalculateGraduated(plan, quantity);

    /// <summary>
    /// Staircase: fixed price at each step, flat within the step.
    /// E.g. 1–10 users = $100/mo, 11–25 users = $200/mo.
    /// </summary>
    private decimal CalculateStaircase(Plan plan, int quantity)
    {
        var tier = plan.Tiers
            .OrderBy(t => t.Order)
            .FirstOrDefault(t => t.UpTo == 0 || quantity <= t.UpTo);

        return tier?.FlatFee ?? plan.BasePrice;
    }
}
```

### Billing Anchor Dates

Billing anchor dates determine when each period starts. For example, if a customer subscribes on January 15 with an anchor day of 1, their first prorated period runs from January 15–31, then full cycles start on the 1st of each month.

```csharp
public static class BillingAnchor
{
    public static DateTime CalculateNextBillingDate(
        DateTime currentPeriodEnd, int anchorDay, BillingInterval interval)
    {
        if (interval != BillingInterval.Monthly)
            return currentPeriodEnd;

        var nextMonth = currentPeriodEnd.AddMonths(1);
        var daysInMonth = DateTime.DaysInMonth(nextMonth.Year, nextMonth.Month);
        var effectiveDay = Math.Min(anchorDay, daysInMonth);

        return new DateTime(nextMonth.Year, nextMonth.Month, effectiveDay,
            0, 0, 0, DateTimeKind.Utc);
    }

    public static (DateTime PeriodStart, DateTime PeriodEnd) GetAlignedPeriod(
        DateTime subscriptionStart, int anchorDay)
    {
        var daysInMonth = DateTime.DaysInMonth(
            subscriptionStart.Year, subscriptionStart.Month);
        var effectiveAnchor = Math.Min(anchorDay, daysInMonth);

        var periodStart = subscriptionStart;
        DateTime periodEnd;

        if (subscriptionStart.Day <= effectiveAnchor)
        {
            periodEnd = new DateTime(subscriptionStart.Year,
                subscriptionStart.Month, effectiveAnchor, 0, 0, 0, DateTimeKind.Utc);
        }
        else
        {
            var nextMonth = subscriptionStart.AddMonths(1);
            var nextDays = DateTime.DaysInMonth(nextMonth.Year, nextMonth.Month);
            periodEnd = new DateTime(nextMonth.Year, nextMonth.Month,
                Math.Min(anchorDay, nextDays), 0, 0, 0, DateTimeKind.Utc);
        }

        return (periodStart, periodEnd);
    }
}
```

---

## 2. Usage-Based & Metered Billing

### Usage Event Model

Usage-based billing charges customers based on consumption rather than a fixed fee. Events are ingested, aggregated, and rated at the end of each billing cycle.

```csharp
public class UsageEvent
{
    public Guid Id { get; set; }
    public Guid SubscriptionId { get; set; }
    public Guid SubscriptionItemId { get; set; }
    public string MetricName { get; set; } = default!; // e.g. "api_calls"
    public decimal Quantity { get; set; }
    public DateTime Timestamp { get; set; }
    public string? IdempotencyKey { get; set; }
    public Dictionary<string, string> Properties { get; set; } = new();
}

public class UsageRecord
{
    public Guid Id { get; set; }
    public Guid SubscriptionId { get; set; }
    public Guid SubscriptionItemId { get; set; }
    public string MetricName { get; set; } = default!;
    public DateTime PeriodStart { get; set; }
    public DateTime PeriodEnd { get; set; }
    public decimal AggregatedQuantity { get; set; }
    public decimal RatedAmount { get; set; }
    public AggregationStrategy Strategy { get; set; }
    public DateTime CalculatedAt { get; set; }
}

public enum AggregationStrategy
{
    Sum,
    Max,
    Last,
    UniqueCount
}
```

### Usage Event Ingestion

```csharp
public interface IUsageIngestionService
{
    Task RecordUsageAsync(UsageEvent usageEvent);
    Task<IReadOnlyList<UsageEvent>> GetEventsAsync(
        Guid subscriptionItemId, DateTime from, DateTime to);
}

public class UsageIngestionService : IUsageIngestionService
{
    private readonly SubscriptionDbContext _db;
    private readonly ILogger<UsageIngestionService> _logger;

    public UsageIngestionService(
        SubscriptionDbContext db, ILogger<UsageIngestionService> logger)
    {
        _db = db;
        _logger = logger;
    }

    public async Task RecordUsageAsync(UsageEvent usageEvent)
    {
        // Idempotency check
        if (!string.IsNullOrEmpty(usageEvent.IdempotencyKey))
        {
            var exists = await _db.Set<UsageEvent>()
                .AnyAsync(e => e.IdempotencyKey == usageEvent.IdempotencyKey);
            if (exists)
            {
                _logger.LogInformation(
                    "Duplicate usage event skipped: {Key}", usageEvent.IdempotencyKey);
                return;
            }
        }

        usageEvent.Id = Guid.NewGuid();
        usageEvent.Timestamp = usageEvent.Timestamp == default
            ? DateTime.UtcNow
            : usageEvent.Timestamp;

        _db.Set<UsageEvent>().Add(usageEvent);
        await _db.SaveChangesAsync();
    }

    public async Task<IReadOnlyList<UsageEvent>> GetEventsAsync(
        Guid subscriptionItemId, DateTime from, DateTime to)
    {
        return await _db.Set<UsageEvent>()
            .Where(e => e.SubscriptionItemId == subscriptionItemId
                     && e.Timestamp >= from
                     && e.Timestamp < to)
            .OrderBy(e => e.Timestamp)
            .ToListAsync();
    }
}
```

### Aggregation Engine

```csharp
public class UsageAggregator
{
    public decimal Aggregate(
        IReadOnlyList<UsageEvent> events, AggregationStrategy strategy)
    {
        if (!events.Any()) return 0m;

        return strategy switch
        {
            AggregationStrategy.Sum =>
                events.Sum(e => e.Quantity),

            AggregationStrategy.Max =>
                events.Max(e => e.Quantity),

            AggregationStrategy.Last =>
                events.OrderByDescending(e => e.Timestamp).First().Quantity,

            AggregationStrategy.UniqueCount =>
                events.Select(e => e.Properties.GetValueOrDefault("unique_id", ""))
                      .Where(id => !string.IsNullOrEmpty(id))
                      .Distinct()
                      .Count(),

            _ => throw new ArgumentOutOfRangeException(nameof(strategy))
        };
    }
}
```

### Rating Engine

The rating engine applies the plan's pricing tiers to aggregated usage to compute the billable amount.

```csharp
public class UsageRatingEngine
{
    private readonly PricingCalculator _calculator;
    private readonly UsageAggregator _aggregator;

    public UsageRatingEngine(PricingCalculator calculator, UsageAggregator aggregator)
    {
        _calculator = calculator;
        _aggregator = aggregator;
    }

    public UsageRecord Rate(
        IReadOnlyList<UsageEvent> events,
        Plan plan,
        Subscription subscription,
        SubscriptionItem item,
        AggregationStrategy strategy)
    {
        var quantity = _aggregator.Aggregate(events, strategy);
        var amount = _calculator.Calculate(plan, (int)quantity);

        // Handle included quantity / overage
        if (plan.MinimumQuantity.HasValue)
        {
            var includedQuantity = plan.MinimumQuantity.Value;
            if (quantity <= includedQuantity)
            {
                amount = plan.BasePrice; // base price covers included usage
            }
            else
            {
                var overage = (int)(quantity - includedQuantity);
                amount = plan.BasePrice + _calculator.Calculate(plan, overage);
            }
        }

        return new UsageRecord
        {
            Id = Guid.NewGuid(),
            SubscriptionId = subscription.Id,
            SubscriptionItemId = item.Id,
            MetricName = events.FirstOrDefault()?.MetricName ?? "unknown",
            PeriodStart = subscription.CurrentPeriodStart,
            PeriodEnd = subscription.CurrentPeriodEnd,
            AggregatedQuantity = quantity,
            RatedAmount = amount,
            Strategy = strategy,
            CalculatedAt = DateTime.UtcNow
        };
    }
}
```

### Real-Time vs. Batch Aggregation

```csharp
public interface IUsageAggregationPipeline
{
    Task<UsageRecord> AggregateAndRateAsync(Guid subscriptionItemId,
        DateTime periodStart, DateTime periodEnd);
}

/// <summary>
/// Batch aggregation: runs at end of billing period via a scheduled job.
/// </summary>
public class BatchUsageAggregationPipeline : IUsageAggregationPipeline
{
    private readonly IUsageIngestionService _ingestion;
    private readonly UsageRatingEngine _ratingEngine;
    private readonly SubscriptionDbContext _db;

    public BatchUsageAggregationPipeline(
        IUsageIngestionService ingestion,
        UsageRatingEngine ratingEngine,
        SubscriptionDbContext db)
    {
        _ingestion = ingestion;
        _ratingEngine = ratingEngine;
        _db = db;
    }

    public async Task<UsageRecord> AggregateAndRateAsync(
        Guid subscriptionItemId, DateTime periodStart, DateTime periodEnd)
    {
        var events = await _ingestion.GetEventsAsync(
            subscriptionItemId, periodStart, periodEnd);

        var item = await _db.SubscriptionItems
            .FirstAsync(i => i.Id == subscriptionItemId);
        var subscription = await _db.Subscriptions
            .Include(s => s.Plan).ThenInclude(p => p.Tiers)
            .FirstAsync(s => s.Id == item.SubscriptionId);

        var record = _ratingEngine.Rate(
            events, subscription.Plan, subscription, item,
            AggregationStrategy.Sum);

        _db.Set<UsageRecord>().Add(record);
        await _db.SaveChangesAsync();

        return record;
    }
}
```

---

## 3. Seat-Based Pricing

### Seat Model

Seat-based pricing charges per user or per unit. Common in SaaS products where the value scales with the number of team members.

```csharp
public class SeatAllocation
{
    public Guid Id { get; set; }
    public Guid SubscriptionId { get; set; }
    public Guid? UserId { get; set; }
    public string? Email { get; set; }
    public SeatStatus Status { get; set; }
    public DateTime AllocatedAt { get; set; }
    public DateTime? DeallocatedAt { get; set; }
}

public enum SeatStatus
{
    Active,
    Pending,
    Deactivated
}
```

### Seat Management Service

```csharp
public interface ISeatManagementService
{
    Task<SeatChangeResult> AddSeatsAsync(Guid subscriptionId, int count);
    Task<SeatChangeResult> RemoveSeatsAsync(Guid subscriptionId, int count);
    Task<int> GetActiveSeatCountAsync(Guid subscriptionId);
}

public class SeatManagementService : ISeatManagementService
{
    private readonly SubscriptionDbContext _db;
    private readonly IProrationEngine _prorationEngine;
    private readonly ILogger<SeatManagementService> _logger;

    public SeatManagementService(
        SubscriptionDbContext db,
        IProrationEngine prorationEngine,
        ILogger<SeatManagementService> logger)
    {
        _db = db;
        _prorationEngine = prorationEngine;
        _logger = logger;
    }

    public async Task<SeatChangeResult> AddSeatsAsync(
        Guid subscriptionId, int count)
    {
        var subscription = await _db.Subscriptions
            .Include(s => s.Plan)
            .Include(s => s.Items)
            .FirstAsync(s => s.Id == subscriptionId);

        var currentSeats = await GetActiveSeatCountAsync(subscriptionId);
        var newTotal = currentSeats + count;

        // Enforce maximum
        if (subscription.Plan.MaximumQuantity.HasValue
            && newTotal > subscription.Plan.MaximumQuantity.Value)
        {
            return SeatChangeResult.Failed(
                $"Cannot exceed {subscription.Plan.MaximumQuantity} seats.");
        }

        // Calculate proration for added seats
        var proration = _prorationEngine.CalculateProration(
            subscription, currentSeats, newTotal, ProrationStrategy.Immediate);

        // Update the subscription item quantity
        var seatItem = subscription.Items.First();
        await _db.Entry(seatItem).ReloadAsync();
        _db.Entry(seatItem).Property("Quantity").CurrentValue = newTotal;

        await _db.SaveChangesAsync();

        _logger.LogInformation(
            "Added {Count} seats to subscription {SubId}. Proration: {Amount}",
            count, subscriptionId, proration.Amount);

        return SeatChangeResult.Success(newTotal, proration);
    }

    public async Task<SeatChangeResult> RemoveSeatsAsync(
        Guid subscriptionId, int count)
    {
        var subscription = await _db.Subscriptions
            .Include(s => s.Plan)
            .Include(s => s.Items)
            .FirstAsync(s => s.Id == subscriptionId);

        var currentSeats = await GetActiveSeatCountAsync(subscriptionId);
        var newTotal = currentSeats - count;

        // Enforce minimum
        var minimum = subscription.Plan.MinimumQuantity ?? 1;
        if (newTotal < minimum)
        {
            return SeatChangeResult.Failed(
                $"Cannot go below {minimum} seats.");
        }

        var proration = _prorationEngine.CalculateProration(
            subscription, currentSeats, newTotal, ProrationStrategy.NextCycle);

        var seatItem = subscription.Items.First();
        await _db.Entry(seatItem).ReloadAsync();
        _db.Entry(seatItem).Property("Quantity").CurrentValue = newTotal;

        await _db.SaveChangesAsync();

        return SeatChangeResult.Success(newTotal, proration);
    }

    public async Task<int> GetActiveSeatCountAsync(Guid subscriptionId)
    {
        return await _db.Set<SeatAllocation>()
            .CountAsync(s => s.SubscriptionId == subscriptionId
                          && s.Status == SeatStatus.Active);
    }
}

public record SeatChangeResult
{
    public bool IsSuccess { get; init; }
    public int NewSeatCount { get; init; }
    public ProrationResult? Proration { get; init; }
    public string? Error { get; init; }

    public static SeatChangeResult Success(int seats, ProrationResult proration)
        => new() { IsSuccess = true, NewSeatCount = seats, Proration = proration };

    public static SeatChangeResult Failed(string error)
        => new() { IsSuccess = false, Error = error };
}
```

### True-Up Billing

True-up billing reconciles actual usage against committed quantities at the end of each cycle.

```csharp
public class TrueUpBillingService
{
    private readonly ISeatManagementService _seats;
    private readonly PricingCalculator _pricing;

    public TrueUpBillingService(
        ISeatManagementService seats, PricingCalculator pricing)
    {
        _seats = seats;
        _pricing = pricing;
    }

    public async Task<TrueUpResult> CalculateTrueUpAsync(
        Subscription subscription, int committedSeats)
    {
        var actualSeats = await _seats.GetActiveSeatCountAsync(subscription.Id);

        if (actualSeats <= committedSeats)
        {
            return new TrueUpResult
            {
                ActualSeats = actualSeats,
                CommittedSeats = committedSeats,
                OverageSeats = 0,
                OverageCharge = 0m
            };
        }

        var overageSeats = actualSeats - committedSeats;
        var overageCharge = _pricing.Calculate(subscription.Plan, overageSeats);

        return new TrueUpResult
        {
            ActualSeats = actualSeats,
            CommittedSeats = committedSeats,
            OverageSeats = overageSeats,
            OverageCharge = overageCharge
        };
    }
}

public record TrueUpResult
{
    public int ActualSeats { get; init; }
    public int CommittedSeats { get; init; }
    public int OverageSeats { get; init; }
    public decimal OverageCharge { get; init; }
}
```

---

## 4. Proration Engine

### Proration Strategies

When a customer upgrades, downgrades, or changes seat counts mid-cycle, the system must calculate fair charges or credits for the remaining time.

```csharp
public enum ProrationStrategy
{
    Immediate,   // charge/credit now for remaining days
    NextCycle,   // apply change at the next billing date
    Prorated     // create prorated invoice line items
}

public record ProrationResult
{
    public decimal Amount { get; init; }
    public decimal CreditAmount { get; init; }
    public decimal ChargeAmount { get; init; }
    public int DaysRemaining { get; init; }
    public int TotalDaysInPeriod { get; init; }
    public string Description { get; init; } = default!;
    public List<ProrationLineItem> LineItems { get; init; } = new();
}

public record ProrationLineItem
{
    public string Description { get; init; } = default!;
    public decimal Amount { get; init; }
    public DateTime PeriodStart { get; init; }
    public DateTime PeriodEnd { get; init; }
    public bool IsCredit { get; init; }
}
```

### Proration Engine Implementation

```csharp
public interface IProrationEngine
{
    ProrationResult CalculateProration(
        Subscription subscription, int oldQuantity, int newQuantity,
        ProrationStrategy strategy);

    ProrationResult CalculatePlanChange(
        Subscription subscription, Plan oldPlan, Plan newPlan,
        ProrationStrategy strategy);
}

public class ProrationEngine : IProrationEngine
{
    public ProrationResult CalculateProration(
        Subscription subscription, int oldQuantity, int newQuantity,
        ProrationStrategy strategy)
    {
        if (strategy == ProrationStrategy.NextCycle)
        {
            return new ProrationResult
            {
                Amount = 0m,
                Description = "Change applied at next billing cycle.",
                DaysRemaining = 0,
                TotalDaysInPeriod = 0
            };
        }

        var now = DateTime.UtcNow;
        var periodStart = subscription.CurrentPeriodStart;
        var periodEnd = subscription.CurrentPeriodEnd;
        var totalDays = (periodEnd - periodStart).TotalDays;
        var daysRemaining = Math.Max(0, (periodEnd - now).TotalDays);
        var fraction = daysRemaining / totalDays;

        var unitPrice = subscription.Plan.BasePrice;
        var oldCost = oldQuantity * unitPrice;
        var newCost = newQuantity * unitPrice;
        var difference = newCost - oldCost;
        var proratedAmount = Math.Round((decimal)fraction * difference, 2);

        var lineItems = new List<ProrationLineItem>();

        // Credit for unused time on old plan
        if (difference != 0)
        {
            var creditAmount = Math.Round(
                (decimal)fraction * oldCost, 2);
            lineItems.Add(new ProrationLineItem
            {
                Description = $"Credit for unused time ({oldQuantity} units)",
                Amount = -creditAmount,
                PeriodStart = now,
                PeriodEnd = periodEnd,
                IsCredit = true
            });

            // Charge for remaining time on new plan
            var chargeAmount = Math.Round(
                (decimal)fraction * newCost, 2);
            lineItems.Add(new ProrationLineItem
            {
                Description = $"Charge for remaining time ({newQuantity} units)",
                Amount = chargeAmount,
                PeriodStart = now,
                PeriodEnd = periodEnd,
                IsCredit = false
            });
        }

        return new ProrationResult
        {
            Amount = proratedAmount,
            CreditAmount = lineItems.Where(l => l.IsCredit).Sum(l => Math.Abs(l.Amount)),
            ChargeAmount = lineItems.Where(l => !l.IsCredit).Sum(l => l.Amount),
            DaysRemaining = (int)daysRemaining,
            TotalDaysInPeriod = (int)totalDays,
            Description = $"Prorated change from {oldQuantity} to {newQuantity} units "
                        + $"for {(int)daysRemaining} remaining days.",
            LineItems = lineItems
        };
    }

    public ProrationResult CalculatePlanChange(
        Subscription subscription, Plan oldPlan, Plan newPlan,
        ProrationStrategy strategy)
    {
        if (strategy == ProrationStrategy.NextCycle)
        {
            return new ProrationResult
            {
                Amount = 0m,
                Description = "Plan change applied at next billing cycle."
            };
        }

        var now = DateTime.UtcNow;
        var periodStart = subscription.CurrentPeriodStart;
        var periodEnd = subscription.CurrentPeriodEnd;
        var totalDays = (periodEnd - periodStart).TotalDays;
        var daysRemaining = Math.Max(0, (periodEnd - now).TotalDays);
        var fraction = daysRemaining / totalDays;

        var oldDailyRate = oldPlan.BasePrice / (decimal)totalDays;
        var newDailyRate = newPlan.BasePrice / (decimal)totalDays;

        var credit = Math.Round(oldDailyRate * (decimal)daysRemaining, 2);
        var charge = Math.Round(newDailyRate * (decimal)daysRemaining, 2);
        var netAmount = charge - credit;

        var lineItems = new List<ProrationLineItem>
        {
            new()
            {
                Description = $"Credit: unused time on {oldPlan.Name}",
                Amount = -credit,
                PeriodStart = now,
                PeriodEnd = periodEnd,
                IsCredit = true
            },
            new()
            {
                Description = $"Charge: remaining time on {newPlan.Name}",
                Amount = charge,
                PeriodStart = now,
                PeriodEnd = periodEnd,
                IsCredit = false
            }
        };

        return new ProrationResult
        {
            Amount = netAmount,
            CreditAmount = credit,
            ChargeAmount = charge,
            DaysRemaining = (int)daysRemaining,
            TotalDaysInPeriod = (int)totalDays,
            Description = netAmount >= 0
                ? $"Upgrade from {oldPlan.Name} to {newPlan.Name}: charge {netAmount:C}"
                : $"Downgrade from {oldPlan.Name} to {newPlan.Name}: credit {Math.Abs(netAmount):C}",
            LineItems = lineItems
        };
    }
}
```

---

## 5. Trial Management

### Trial Configuration

Free trials convert prospects into paying customers. Proper trial management includes abuse prevention, conversion tracking, and graceful expiration flows.

```csharp
public class TrialConfiguration
{
    public int DefaultTrialDays { get; set; } = 14;
    public bool RequirePaymentMethod { get; set; } = true;
    public bool AllowExtension { get; set; } = true;
    public int MaxExtensionDays { get; set; } = 14;
    public int MaxTrialsPerCustomer { get; set; } = 1;
    public bool EnableFingerprintCheck { get; set; } = true;
}

public class TrialRecord
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public Guid SubscriptionId { get; set; }
    public Guid PlanId { get; set; }
    public DateTime TrialStart { get; set; }
    public DateTime TrialEnd { get; set; }
    public TrialStatus Status { get; set; }
    public bool PaymentMethodOnFile { get; set; }
    public string? DeviceFingerprint { get; set; }
    public string? IpAddress { get; set; }
    public DateTime? ConvertedAt { get; set; }
    public DateTime? ExtendedAt { get; set; }
    public int ExtensionDays { get; set; }
}

public enum TrialStatus
{
    Active,
    Converted,
    Expired,
    Cancelled,
    Extended
}
```

### Trial Management Service

```csharp
public interface ITrialManagementService
{
    Task<TrialRecord> StartTrialAsync(StartTrialRequest request);
    Task<TrialRecord> ConvertTrialAsync(Guid subscriptionId);
    Task ExpireTrialAsync(Guid subscriptionId);
    Task<TrialRecord> ExtendTrialAsync(Guid subscriptionId, int additionalDays);
}

public record StartTrialRequest
{
    public Guid CustomerId { get; init; }
    public Guid PlanId { get; init; }
    public int? TrialDays { get; init; }
    public Guid? PaymentMethodId { get; init; }
    public string? DeviceFingerprint { get; init; }
    public string? IpAddress { get; init; }
}

public class TrialManagementService : ITrialManagementService
{
    private readonly SubscriptionDbContext _db;
    private readonly TrialConfiguration _config;
    private readonly ITrialAbuseDetector _abuseDetector;
    private readonly IEventPublisher _events;
    private readonly ILogger<TrialManagementService> _logger;

    public TrialManagementService(
        SubscriptionDbContext db,
        IOptions<TrialConfiguration> config,
        ITrialAbuseDetector abuseDetector,
        IEventPublisher events,
        ILogger<TrialManagementService> logger)
    {
        _db = db;
        _config = config.Value;
        _abuseDetector = abuseDetector;
        _events = events;
        _logger = logger;
    }

    public async Task<TrialRecord> StartTrialAsync(StartTrialRequest request)
    {
        // Check trial abuse
        var abuseCheck = await _abuseDetector.CheckAsync(
            request.CustomerId, request.DeviceFingerprint, request.IpAddress);
        if (abuseCheck.IsAbusive)
        {
            throw new TrialAbuseException(
                $"Trial denied: {abuseCheck.Reason}");
        }

        // Check payment method requirement
        if (_config.RequirePaymentMethod && request.PaymentMethodId is null)
        {
            throw new InvalidOperationException(
                "A payment method is required to start a trial.");
        }

        var plan = await _db.Plans.FirstAsync(p => p.Id == request.PlanId);
        var trialDays = request.TrialDays ?? _config.DefaultTrialDays;

        var subscription = Subscription.Create(
            request.CustomerId, plan, trialDays: trialDays);
        subscription.AddItem(plan.Id);

        if (request.PaymentMethodId.HasValue)
        {
            // set via reflection or internal setter in production
        }

        _db.Subscriptions.Add(subscription);

        var trial = new TrialRecord
        {
            Id = Guid.NewGuid(),
            CustomerId = request.CustomerId,
            SubscriptionId = subscription.Id,
            PlanId = plan.Id,
            TrialStart = DateTime.UtcNow,
            TrialEnd = DateTime.UtcNow.AddDays(trialDays),
            Status = TrialStatus.Active,
            PaymentMethodOnFile = request.PaymentMethodId.HasValue,
            DeviceFingerprint = request.DeviceFingerprint,
            IpAddress = request.IpAddress
        };

        _db.Set<TrialRecord>().Add(trial);
        await _db.SaveChangesAsync();

        await _events.PublishAsync(new TrialStartedEvent(
            subscription.Id, request.CustomerId, trial.TrialEnd));

        return trial;
    }

    public async Task<TrialRecord> ConvertTrialAsync(Guid subscriptionId)
    {
        var trial = await _db.Set<TrialRecord>()
            .FirstAsync(t => t.SubscriptionId == subscriptionId
                          && t.Status == TrialStatus.Active);

        trial.Status = TrialStatus.Converted;
        trial.ConvertedAt = DateTime.UtcNow;

        await _db.SaveChangesAsync();

        await _events.PublishAsync(new TrialConvertedEvent(
            subscriptionId, trial.CustomerId));

        return trial;
    }

    public async Task ExpireTrialAsync(Guid subscriptionId)
    {
        var trial = await _db.Set<TrialRecord>()
            .FirstAsync(t => t.SubscriptionId == subscriptionId
                          && t.Status == TrialStatus.Active);

        trial.Status = TrialStatus.Expired;
        await _db.SaveChangesAsync();

        await _events.PublishAsync(new TrialExpiredEvent(
            subscriptionId, trial.CustomerId));
    }

    public async Task<TrialRecord> ExtendTrialAsync(
        Guid subscriptionId, int additionalDays)
    {
        if (!_config.AllowExtension)
            throw new InvalidOperationException("Trial extensions are disabled.");

        var trial = await _db.Set<TrialRecord>()
            .FirstAsync(t => t.SubscriptionId == subscriptionId
                          && t.Status == TrialStatus.Active);

        if (trial.ExtensionDays + additionalDays > _config.MaxExtensionDays)
        {
            throw new InvalidOperationException(
                $"Maximum extension of {_config.MaxExtensionDays} days exceeded.");
        }

        trial.TrialEnd = trial.TrialEnd.AddDays(additionalDays);
        trial.ExtensionDays += additionalDays;
        trial.ExtendedAt = DateTime.UtcNow;
        trial.Status = TrialStatus.Extended;

        await _db.SaveChangesAsync();
        return trial;
    }
}
```

### Trial Abuse Prevention

```csharp
public interface ITrialAbuseDetector
{
    Task<AbuseCheckResult> CheckAsync(
        Guid customerId, string? fingerprint, string? ipAddress);
}

public class TrialAbuseDetector : ITrialAbuseDetector
{
    private readonly SubscriptionDbContext _db;
    private readonly TrialConfiguration _config;

    public TrialAbuseDetector(
        SubscriptionDbContext db, IOptions<TrialConfiguration> config)
    {
        _db = db;
        _config = config.Value;
    }

    public async Task<AbuseCheckResult> CheckAsync(
        Guid customerId, string? fingerprint, string? ipAddress)
    {
        // Check: max trials per customer
        var customerTrials = await _db.Set<TrialRecord>()
            .CountAsync(t => t.CustomerId == customerId);
        if (customerTrials >= _config.MaxTrialsPerCustomer)
        {
            return AbuseCheckResult.Blocked(
                "Maximum number of trials reached for this account.");
        }

        // Check: device fingerprint reuse
        if (_config.EnableFingerprintCheck && !string.IsNullOrEmpty(fingerprint))
        {
            var fingerprintTrials = await _db.Set<TrialRecord>()
                .CountAsync(t => t.DeviceFingerprint == fingerprint);
            if (fingerprintTrials > 0)
            {
                return AbuseCheckResult.Blocked(
                    "A trial has already been used from this device.");
            }
        }

        // Check: excessive trials from same IP in short window
        if (!string.IsNullOrEmpty(ipAddress))
        {
            var recentIpTrials = await _db.Set<TrialRecord>()
                .CountAsync(t => t.IpAddress == ipAddress
                              && t.TrialStart > DateTime.UtcNow.AddDays(-30));
            if (recentIpTrials >= 3)
            {
                return AbuseCheckResult.Blocked(
                    "Too many trial signups from this IP address.");
            }
        }

        return AbuseCheckResult.Allowed();
    }
}

public record AbuseCheckResult
{
    public bool IsAbusive { get; init; }
    public string? Reason { get; init; }

    public static AbuseCheckResult Allowed() => new() { IsAbusive = false };
    public static AbuseCheckResult Blocked(string reason)
        => new() { IsAbusive = true, Reason = reason };
}

public class TrialAbuseException : Exception
{
    public TrialAbuseException(string message) : base(message) { }
}
```

---

## 6. Dunning & Payment Recovery

### Dunning Configuration

Dunning is the process of recovering failed payments through automated retries, customer communication, and grace periods. A well-designed dunning flow significantly reduces involuntary churn.

```csharp
public class DunningConfiguration
{
    public int GracePeriodDays { get; set; } = 7;
    public int MaxRetryAttempts { get; set; } = 4;
    public List<RetryScheduleEntry> RetrySchedule { get; set; } = new()
    {
        new(1, TimeSpan.FromHours(4)),    // retry 1: 4 hours after failure
        new(2, TimeSpan.FromDays(1)),     // retry 2: 1 day later
        new(3, TimeSpan.FromDays(3)),     // retry 3: 3 days later
        new(4, TimeSpan.FromDays(5))      // retry 4: 5 days later (final)
    };
    public bool SendDunningEmails { get; set; } = true;
    public bool CancelOnFinalFailure { get; set; } = true;
}

public record RetryScheduleEntry(int AttemptNumber, TimeSpan DelayAfterPrevious);

public class PaymentRetryRecord
{
    public Guid Id { get; set; }
    public Guid SubscriptionId { get; set; }
    public Guid InvoiceId { get; set; }
    public int AttemptNumber { get; set; }
    public DateTime ScheduledAt { get; set; }
    public DateTime? AttemptedAt { get; set; }
    public PaymentRetryStatus Status { get; set; }
    public string? FailureReason { get; set; }
    public string? PaymentMethodId { get; set; }
}

public enum PaymentRetryStatus
{
    Scheduled,
    Succeeded,
    Failed,
    Skipped
}
```

### Dunning Service

```csharp
public interface IDunningService
{
    Task HandlePaymentFailureAsync(Guid subscriptionId, Guid invoiceId,
        string failureReason);
    Task ProcessScheduledRetriesAsync();
}

public class DunningService : IDunningService
{
    private readonly SubscriptionDbContext _db;
    private readonly DunningConfiguration _config;
    private readonly IPaymentGateway _paymentGateway;
    private readonly INotificationService _notifications;
    private readonly IEventPublisher _events;
    private readonly ILogger<DunningService> _logger;

    public DunningService(
        SubscriptionDbContext db,
        IOptions<DunningConfiguration> config,
        IPaymentGateway paymentGateway,
        INotificationService notifications,
        IEventPublisher events,
        ILogger<DunningService> logger)
    {
        _db = db;
        _config = config.Value;
        _paymentGateway = paymentGateway;
        _notifications = notifications;
        _events = events;
        _logger = logger;
    }

    public async Task HandlePaymentFailureAsync(
        Guid subscriptionId, Guid invoiceId, string failureReason)
    {
        var subscription = await _db.Subscriptions
            .FirstAsync(s => s.Id == subscriptionId);

        // Transition to PastDue
        if (subscription.Status == SubscriptionStatus.Active)
        {
            // Use internal setter or domain method
            _db.Entry(subscription).Property("Status")
               .CurrentValue = SubscriptionStatus.PastDue;
        }

        // Schedule first retry
        var firstRetry = _config.RetrySchedule.First();
        var retry = new PaymentRetryRecord
        {
            Id = Guid.NewGuid(),
            SubscriptionId = subscriptionId,
            InvoiceId = invoiceId,
            AttemptNumber = 1,
            ScheduledAt = DateTime.UtcNow.Add(firstRetry.DelayAfterPrevious),
            Status = PaymentRetryStatus.Scheduled,
            FailureReason = failureReason
        };

        _db.Set<PaymentRetryRecord>().Add(retry);
        await _db.SaveChangesAsync();

        // Send initial dunning notification
        if (_config.SendDunningEmails)
        {
            await _notifications.SendPaymentFailedNotificationAsync(
                subscriptionId, failureReason, retry.ScheduledAt);
        }

        await _events.PublishAsync(new PaymentFailedEvent(
            subscriptionId, invoiceId, failureReason));
    }

    public async Task ProcessScheduledRetriesAsync()
    {
        var dueRetries = await _db.Set<PaymentRetryRecord>()
            .Where(r => r.Status == PaymentRetryStatus.Scheduled
                     && r.ScheduledAt <= DateTime.UtcNow)
            .OrderBy(r => r.ScheduledAt)
            .ToListAsync();

        foreach (var retry in dueRetries)
        {
            retry.AttemptedAt = DateTime.UtcNow;

            try
            {
                var result = await _paymentGateway.RetryPaymentAsync(
                    retry.InvoiceId);

                if (result.Success)
                {
                    retry.Status = PaymentRetryStatus.Succeeded;
                    await ReactivateSubscriptionAsync(retry.SubscriptionId);
                    _logger.LogInformation(
                        "Payment retry succeeded for subscription {SubId}",
                        retry.SubscriptionId);
                }
                else
                {
                    retry.Status = PaymentRetryStatus.Failed;
                    retry.FailureReason = result.ErrorMessage;
                    await ScheduleNextRetryAsync(retry);
                }
            }
            catch (Exception ex)
            {
                retry.Status = PaymentRetryStatus.Failed;
                retry.FailureReason = ex.Message;
                await ScheduleNextRetryAsync(retry);
                _logger.LogError(ex,
                    "Payment retry error for subscription {SubId}",
                    retry.SubscriptionId);
            }

            await _db.SaveChangesAsync();
        }
    }

    private async Task ScheduleNextRetryAsync(PaymentRetryRecord currentRetry)
    {
        var nextAttempt = currentRetry.AttemptNumber + 1;
        var schedule = _config.RetrySchedule
            .FirstOrDefault(s => s.AttemptNumber == nextAttempt);

        if (schedule is null || nextAttempt > _config.MaxRetryAttempts)
        {
            // Final failure — cancel or expire
            if (_config.CancelOnFinalFailure)
            {
                var subscription = await _db.Subscriptions
                    .FirstAsync(s => s.Id == currentRetry.SubscriptionId);
                subscription.Cancel(atPeriodEnd: false);
                await _db.SaveChangesAsync();

                await _events.PublishAsync(new SubscriptionCancelledEvent(
                    currentRetry.SubscriptionId,
                    CancellationReason.InvoluntaryChurn));

                await _notifications.SendFinalPaymentFailureNotificationAsync(
                    currentRetry.SubscriptionId);
            }
            return;
        }

        var nextRetry = new PaymentRetryRecord
        {
            Id = Guid.NewGuid(),
            SubscriptionId = currentRetry.SubscriptionId,
            InvoiceId = currentRetry.InvoiceId,
            AttemptNumber = nextAttempt,
            ScheduledAt = DateTime.UtcNow.Add(schedule.DelayAfterPrevious),
            Status = PaymentRetryStatus.Scheduled
        };

        _db.Set<PaymentRetryRecord>().Add(nextRetry);

        if (_config.SendDunningEmails)
        {
            await _notifications.SendPaymentRetryNotificationAsync(
                currentRetry.SubscriptionId, nextAttempt,
                _config.MaxRetryAttempts, nextRetry.ScheduledAt);
        }
    }

    private async Task ReactivateSubscriptionAsync(Guid subscriptionId)
    {
        var subscription = await _db.Subscriptions
            .FirstAsync(s => s.Id == subscriptionId);

        if (subscription.Status == SubscriptionStatus.PastDue)
        {
            _db.Entry(subscription).Property("Status")
               .CurrentValue = SubscriptionStatus.Active;
            await _db.SaveChangesAsync();

            await _events.PublishAsync(
                new SubscriptionReactivatedEvent(subscriptionId));
        }
    }
}
```

### Smart Retry Timing

```csharp
public class SmartRetryScheduler
{
    /// <summary>
    /// Determines optimal retry time based on failure reason and
    /// historical success patterns. Avoids retrying during bank
    /// maintenance windows and prefers payroll deposit dates.
    /// </summary>
    public DateTime CalculateOptimalRetryTime(
        string failureCode, int attemptNumber, DateTime failedAt)
    {
        var baseDelay = attemptNumber switch
        {
            1 => TimeSpan.FromHours(4),
            2 => TimeSpan.FromDays(1),
            3 => TimeSpan.FromDays(3),
            _ => TimeSpan.FromDays(5)
        };

        var retryTime = failedAt.Add(baseDelay);

        // Avoid retrying between 1–5 AM local time (bank maintenance)
        if (retryTime.Hour is >= 1 and < 5)
            retryTime = retryTime.Date.AddHours(9);

        // For insufficient funds, try near common payroll dates (1st, 15th)
        if (failureCode == "insufficient_funds")
        {
            var nextFirst = new DateTime(
                retryTime.Year, retryTime.Month, 1).AddMonths(1);
            var nextFifteenth = retryTime.Day <= 15
                ? new DateTime(retryTime.Year, retryTime.Month, 15)
                : nextFirst;

            var nearestPayroll = nextFifteenth < nextFirst
                ? nextFifteenth : nextFirst;
            if ((nearestPayroll - retryTime).TotalDays <= 3)
                retryTime = nearestPayroll.AddHours(10);
        }

        return retryTime;
    }
}
```

---

## 7. Subscription Lifecycle Events

### Domain Events

A subscription progresses through a well-defined state machine. Domain events capture each transition for downstream processing (notifications, analytics, integrations).

```csharp
public interface ISubscriptionEvent
{
    Guid SubscriptionId { get; }
    DateTime OccurredAt { get; }
}

public record SubscriptionCreatedEvent(
    Guid SubscriptionId, Guid CustomerId, Guid PlanId)
    : ISubscriptionEvent
{
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}

public record SubscriptionRenewedEvent(
    Guid SubscriptionId, int CycleNumber, decimal Amount)
    : ISubscriptionEvent
{
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}

public record SubscriptionUpgradedEvent(
    Guid SubscriptionId, Guid OldPlanId, Guid NewPlanId,
    decimal ProrationAmount)
    : ISubscriptionEvent
{
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}

public record SubscriptionDowngradedEvent(
    Guid SubscriptionId, Guid OldPlanId, Guid NewPlanId,
    decimal CreditAmount)
    : ISubscriptionEvent
{
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}

public record SubscriptionPausedEvent(Guid SubscriptionId)
    : ISubscriptionEvent
{
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}

public record SubscriptionResumedEvent(Guid SubscriptionId)
    : ISubscriptionEvent
{
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}

public record SubscriptionCancelledEvent(
    Guid SubscriptionId, CancellationReason Reason)
    : ISubscriptionEvent
{
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}

public record SubscriptionExpiredEvent(Guid SubscriptionId)
    : ISubscriptionEvent
{
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}

public record SubscriptionReactivatedEvent(Guid SubscriptionId)
    : ISubscriptionEvent
{
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}

public record TrialStartedEvent(
    Guid SubscriptionId, Guid CustomerId, DateTime TrialEnd)
    : ISubscriptionEvent
{
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}

public record TrialConvertedEvent(
    Guid SubscriptionId, Guid CustomerId)
    : ISubscriptionEvent
{
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}

public record TrialExpiredEvent(
    Guid SubscriptionId, Guid CustomerId)
    : ISubscriptionEvent
{
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}

public record PaymentFailedEvent(
    Guid SubscriptionId, Guid InvoiceId, string FailureReason)
    : ISubscriptionEvent
{
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}

public enum CancellationReason
{
    CustomerRequest,
    InvoluntaryChurn,
    NonPayment,
    Fraud,
    TrialExpired,
    ContractEnd
}
```

### Event Publisher and Handlers

```csharp
public interface IEventPublisher
{
    Task PublishAsync<T>(T domainEvent) where T : ISubscriptionEvent;
}

public interface IEventHandler<in T> where T : ISubscriptionEvent
{
    Task HandleAsync(T domainEvent);
}

public class InMemoryEventPublisher : IEventPublisher
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<InMemoryEventPublisher> _logger;

    public InMemoryEventPublisher(
        IServiceProvider serviceProvider,
        ILogger<InMemoryEventPublisher> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    public async Task PublishAsync<T>(T domainEvent) where T : ISubscriptionEvent
    {
        _logger.LogInformation(
            "Publishing event {EventType} for subscription {SubId}",
            typeof(T).Name, domainEvent.SubscriptionId);

        var handlers = _serviceProvider.GetServices<IEventHandler<T>>();
        foreach (var handler in handlers)
        {
            try
            {
                await handler.HandleAsync(domainEvent);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex,
                    "Error handling event {EventType} in {Handler}",
                    typeof(T).Name, handler.GetType().Name);
            }
        }
    }
}
```

### Subscription Status State Machine

```csharp
/// <summary>
/// Enforces valid state transitions for a subscription lifecycle.
/// </summary>
public class SubscriptionStateMachine
{
    private static readonly Dictionary<SubscriptionStatus, HashSet<SubscriptionStatus>>
        ValidTransitions = new()
    {
        [SubscriptionStatus.Incomplete] = new()
        {
            SubscriptionStatus.Active,
            SubscriptionStatus.Trialing,
            SubscriptionStatus.Cancelled
        },
        [SubscriptionStatus.Trialing] = new()
        {
            SubscriptionStatus.Active,
            SubscriptionStatus.Cancelled,
            SubscriptionStatus.Expired
        },
        [SubscriptionStatus.Active] = new()
        {
            SubscriptionStatus.PastDue,
            SubscriptionStatus.Paused,
            SubscriptionStatus.Cancelled,
            SubscriptionStatus.Expired
        },
        [SubscriptionStatus.PastDue] = new()
        {
            SubscriptionStatus.Active,
            SubscriptionStatus.Cancelled,
            SubscriptionStatus.Expired
        },
        [SubscriptionStatus.Paused] = new()
        {
            SubscriptionStatus.Active,
            SubscriptionStatus.Cancelled
        },
        [SubscriptionStatus.Cancelled] = new()
        {
            SubscriptionStatus.Active // reactivation
        },
        [SubscriptionStatus.Expired] = new()
        {
            SubscriptionStatus.Active // renewal
        }
    };

    public bool CanTransition(SubscriptionStatus from, SubscriptionStatus to)
    {
        return ValidTransitions.TryGetValue(from, out var allowed)
            && allowed.Contains(to);
    }

    public void Transition(Subscription subscription, SubscriptionStatus newStatus)
    {
        if (!CanTransition(subscription.Status, newStatus))
        {
            throw new InvalidOperationException(
                $"Cannot transition from {subscription.Status} to {newStatus}.");
        }

        // In production, set via internal method or reflection
        // subscription.SetStatus(newStatus);
    }
}
```

---

## 8. Invoice Generation

### Invoice Entities

```csharp
public class Invoice
{
    public Guid Id { get; set; }
    public string InvoiceNumber { get; set; } = default!;
    public Guid CustomerId { get; set; }
    public Guid SubscriptionId { get; set; }
    public DateTime PeriodStart { get; set; }
    public DateTime PeriodEnd { get; set; }
    public InvoiceStatus Status { get; set; }
    public string Currency { get; set; } = "USD";
    public decimal Subtotal { get; set; }
    public decimal TaxAmount { get; set; }
    public decimal TotalAmount { get; set; }
    public decimal AmountPaid { get; set; }
    public decimal AmountDue { get; set; }
    public DateTime? PaidAt { get; set; }
    public DateTime? DueDate { get; set; }
    public DateTime CreatedAt { get; set; }
    public string? PdfUrl { get; set; }
    public string? CreditNoteId { get; set; }

    public List<InvoiceLineItem> LineItems { get; set; } = new();
}

public class InvoiceLineItem
{
    public Guid Id { get; set; }
    public Guid InvoiceId { get; set; }
    public string Description { get; set; } = default!;
    public decimal Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    public decimal Amount { get; set; }
    public DateTime PeriodStart { get; set; }
    public DateTime PeriodEnd { get; set; }
    public bool IsProration { get; set; }
    public bool IsCredit { get; set; }
}

public enum InvoiceStatus
{
    Draft,
    Open,
    Paid,
    Void,
    Uncollectible
}
```

### Invoice Generation Service

```csharp
public interface IInvoiceService
{
    Task<Invoice> GenerateInvoiceAsync(Guid subscriptionId);
    Task<Invoice> FinalizeInvoiceAsync(Guid invoiceId);
    Task<CreditNote> IssueCreditNoteAsync(Guid invoiceId, decimal amount,
        string reason);
}

public class InvoiceService : IInvoiceService
{
    private readonly SubscriptionDbContext _db;
    private readonly PricingCalculator _pricing;
    private readonly ITaxCalculator _tax;
    private readonly IInvoiceNumberGenerator _numberGenerator;
    private readonly ILogger<InvoiceService> _logger;

    public InvoiceService(
        SubscriptionDbContext db,
        PricingCalculator pricing,
        ITaxCalculator tax,
        IInvoiceNumberGenerator numberGenerator,
        ILogger<InvoiceService> logger)
    {
        _db = db;
        _pricing = pricing;
        _tax = tax;
        _numberGenerator = numberGenerator;
        _logger = logger;
    }

    public async Task<Invoice> GenerateInvoiceAsync(Guid subscriptionId)
    {
        var subscription = await _db.Subscriptions
            .Include(s => s.Plan).ThenInclude(p => p.Tiers)
            .Include(s => s.Items)
            .FirstAsync(s => s.Id == subscriptionId);

        var invoice = new Invoice
        {
            Id = Guid.NewGuid(),
            InvoiceNumber = await _numberGenerator.GenerateAsync(),
            CustomerId = subscription.CustomerId,
            SubscriptionId = subscription.Id,
            PeriodStart = subscription.CurrentPeriodStart,
            PeriodEnd = subscription.CurrentPeriodEnd,
            Currency = subscription.Currency,
            Status = InvoiceStatus.Draft,
            DueDate = subscription.CurrentPeriodEnd.AddDays(7),
            CreatedAt = DateTime.UtcNow
        };

        // Build line items from subscription items
        foreach (var item in subscription.Items)
        {
            var amount = _pricing.Calculate(subscription.Plan, item.Quantity);
            var unitPrice = item.UnitPriceOverride ?? subscription.Plan.BasePrice;

            invoice.LineItems.Add(new InvoiceLineItem
            {
                Id = Guid.NewGuid(),
                InvoiceId = invoice.Id,
                Description = $"{subscription.Plan.Name} × {item.Quantity}",
                Quantity = item.Quantity,
                UnitPrice = unitPrice,
                Amount = amount,
                PeriodStart = subscription.CurrentPeriodStart,
                PeriodEnd = subscription.CurrentPeriodEnd
            });
        }

        // Add usage-based line items
        var usageRecords = await _db.Set<UsageRecord>()
            .Where(u => u.SubscriptionId == subscriptionId
                     && u.PeriodStart == subscription.CurrentPeriodStart)
            .ToListAsync();

        foreach (var usage in usageRecords)
        {
            invoice.LineItems.Add(new InvoiceLineItem
            {
                Id = Guid.NewGuid(),
                InvoiceId = invoice.Id,
                Description = $"Usage: {usage.MetricName} "
                            + $"({usage.AggregatedQuantity:N0} units)",
                Quantity = usage.AggregatedQuantity,
                UnitPrice = usage.RatedAmount / Math.Max(1, usage.AggregatedQuantity),
                Amount = usage.RatedAmount,
                PeriodStart = usage.PeriodStart,
                PeriodEnd = usage.PeriodEnd
            });
        }

        invoice.Subtotal = invoice.LineItems.Sum(l => l.Amount);

        // Calculate tax
        var taxResult = await _tax.CalculateAsync(
            invoice.CustomerId, invoice.Subtotal, invoice.Currency);
        invoice.TaxAmount = taxResult.TaxAmount;
        invoice.TotalAmount = invoice.Subtotal + invoice.TaxAmount;
        invoice.AmountDue = invoice.TotalAmount;

        _db.Set<Invoice>().Add(invoice);
        await _db.SaveChangesAsync();

        _logger.LogInformation(
            "Invoice {Number} generated for subscription {SubId}: {Amount:C}",
            invoice.InvoiceNumber, subscriptionId, invoice.TotalAmount);

        return invoice;
    }

    public async Task<Invoice> FinalizeInvoiceAsync(Guid invoiceId)
    {
        var invoice = await _db.Set<Invoice>()
            .FirstAsync(i => i.Id == invoiceId);

        invoice.Status = InvoiceStatus.Open;
        await _db.SaveChangesAsync();
        return invoice;
    }

    public async Task<CreditNote> IssueCreditNoteAsync(
        Guid invoiceId, decimal amount, string reason)
    {
        var invoice = await _db.Set<Invoice>()
            .FirstAsync(i => i.Id == invoiceId);

        var creditNote = new CreditNote
        {
            Id = Guid.NewGuid(),
            InvoiceId = invoiceId,
            Amount = amount,
            Reason = reason,
            CreatedAt = DateTime.UtcNow
        };

        invoice.AmountDue = Math.Max(0, invoice.AmountDue - amount);
        invoice.CreditNoteId = creditNote.Id.ToString();

        _db.Set<CreditNote>().Add(creditNote);
        await _db.SaveChangesAsync();

        return creditNote;
    }
}

public class CreditNote
{
    public Guid Id { get; set; }
    public Guid InvoiceId { get; set; }
    public decimal Amount { get; set; }
    public string Reason { get; set; } = default!;
    public DateTime CreatedAt { get; set; }
}

public interface ITaxCalculator
{
    Task<TaxResult> CalculateAsync(
        Guid customerId, decimal subtotal, string currency);
}

public record TaxResult(decimal TaxAmount, decimal TaxRate, string TaxType);

public interface IInvoiceNumberGenerator
{
    Task<string> GenerateAsync();
}

public interface INotificationService
{
    Task SendPaymentFailedNotificationAsync(
        Guid subscriptionId, string reason, DateTime nextRetry);
    Task SendPaymentRetryNotificationAsync(
        Guid subscriptionId, int attempt, int maxAttempts, DateTime nextRetry);
    Task SendFinalPaymentFailureNotificationAsync(Guid subscriptionId);
}

public interface IPaymentGateway
{
    Task<PaymentResult> RetryPaymentAsync(Guid invoiceId);
}

public record PaymentResult(bool Success, string? ErrorMessage);
```

---

## 9. Revenue Metrics & Analytics

### MRR and ARR Calculation

Monthly Recurring Revenue (MRR) and Annual Recurring Revenue (ARR) are the core health metrics for any subscription business.

```csharp
public class RevenueMetricsService
{
    private readonly SubscriptionDbContext _db;

    public RevenueMetricsService(SubscriptionDbContext db)
    {
        _db = db;
    }

    /// <summary>
    /// Calculates MRR by summing the monthly-normalized value
    /// of all active subscriptions.
    /// </summary>
    public async Task<MrrBreakdown> CalculateMrrAsync()
    {
        var activeSubscriptions = await _db.Subscriptions
            .Include(s => s.Plan)
            .Include(s => s.Items)
            .Where(s => s.Status == SubscriptionStatus.Active
                     || s.Status == SubscriptionStatus.Trialing)
            .ToListAsync();

        var totalMrr = 0m;
        foreach (var sub in activeSubscriptions)
        {
            totalMrr += NormalizeToMonthly(sub);
        }

        return new MrrBreakdown
        {
            TotalMrr = totalMrr,
            Arr = totalMrr * 12,
            ActiveSubscriptionCount = activeSubscriptions.Count,
            CalculatedAt = DateTime.UtcNow
        };
    }

    /// <summary>
    /// Normalizes any billing interval to a monthly amount.
    /// </summary>
    private decimal NormalizeToMonthly(Subscription subscription)
    {
        var totalAmount = subscription.Items.Sum(item =>
            (item.UnitPriceOverride ?? subscription.Plan.BasePrice) * item.Quantity);

        return subscription.Plan.BillingInterval switch
        {
            BillingInterval.Daily => totalAmount * 30,
            BillingInterval.Weekly => totalAmount * 4.33m,
            BillingInterval.Monthly => totalAmount,
            BillingInterval.Quarterly => totalAmount / 3m,
            BillingInterval.SemiAnnual => totalAmount / 6m,
            BillingInterval.Annual => totalAmount / 12m,
            _ => totalAmount
        };
    }
}

public record MrrBreakdown
{
    public decimal TotalMrr { get; init; }
    public decimal Arr { get; init; }
    public int ActiveSubscriptionCount { get; init; }
    public DateTime CalculatedAt { get; init; }
}
```

### Churn Rate and Expansion/Contraction Revenue

```csharp
public class ChurnAnalyticsService
{
    private readonly SubscriptionDbContext _db;

    public ChurnAnalyticsService(SubscriptionDbContext db) => _db = db;

    public async Task<ChurnMetrics> CalculateChurnAsync(
        DateTime periodStart, DateTime periodEnd)
    {
        var cancelledSubs = await _db.Subscriptions
            .Where(s => s.CancelledAt >= periodStart
                     && s.CancelledAt < periodEnd
                     && s.Status == SubscriptionStatus.Cancelled)
            .ToListAsync();

        var activeAtStart = await _db.Subscriptions
            .CountAsync(s => s.StartDate < periodStart
                          && (s.CancelledAt == null || s.CancelledAt >= periodStart));

        var voluntaryChurn = cancelledSubs.Count(s =>
            !IsDunningCancellation(s));
        var involuntaryChurn = cancelledSubs.Count(s =>
            IsDunningCancellation(s));

        var totalChurn = cancelledSubs.Count;
        var churnRate = activeAtStart > 0
            ? (decimal)totalChurn / activeAtStart * 100
            : 0m;

        return new ChurnMetrics
        {
            PeriodStart = periodStart,
            PeriodEnd = periodEnd,
            TotalChurned = totalChurn,
            VoluntaryChurn = voluntaryChurn,
            InvoluntaryChurn = involuntaryChurn,
            ChurnRate = Math.Round(churnRate, 2),
            ActiveAtPeriodStart = activeAtStart
        };
    }

    public async Task<RevenueMovement> CalculateRevenueMovementAsync(
        DateTime periodStart, DateTime periodEnd)
    {
        // New MRR: subscriptions created in this period
        var newSubs = await _db.Subscriptions
            .Include(s => s.Plan)
            .Include(s => s.Items)
            .Where(s => s.StartDate >= periodStart && s.StartDate < periodEnd)
            .ToListAsync();

        // Expansion: upgrades/seat additions
        var expansionEvents = await _db.Set<SubscriptionUpgradedEvent>()
            .Where(e => e.OccurredAt >= periodStart && e.OccurredAt < periodEnd)
            .ToListAsync();

        // Contraction: downgrades/seat removals
        var contractionEvents = await _db.Set<SubscriptionDowngradedEvent>()
            .Where(e => e.OccurredAt >= periodStart && e.OccurredAt < periodEnd)
            .ToListAsync();

        return new RevenueMovement
        {
            NewMrr = newSubs.Sum(s =>
                s.Items.Sum(i => i.Quantity * (i.UnitPriceOverride ?? s.Plan.BasePrice))),
            ExpansionMrr = expansionEvents.Sum(e => e.ProrationAmount),
            ContractionMrr = contractionEvents.Sum(e => e.CreditAmount),
            PeriodStart = periodStart,
            PeriodEnd = periodEnd
        };
    }

    /// <summary>
    /// Calculates Customer Lifetime Value: Average MRR per customer / churn rate.
    /// </summary>
    public async Task<decimal> CalculateLtvAsync()
    {
        var activeCount = await _db.Subscriptions
            .CountAsync(s => s.Status == SubscriptionStatus.Active);
        if (activeCount == 0) return 0m;

        var totalMrr = await _db.Subscriptions
            .Include(s => s.Plan)
            .Include(s => s.Items)
            .Where(s => s.Status == SubscriptionStatus.Active)
            .SumAsync(s => s.Items.Sum(
                i => i.Quantity * (i.UnitPriceOverride ?? s.Plan.BasePrice)));

        var avgMrr = totalMrr / activeCount;

        var last12Months = DateTime.UtcNow.AddMonths(-12);
        var churn = await CalculateChurnAsync(last12Months, DateTime.UtcNow);
        var monthlyChurnRate = churn.ChurnRate / 12m;

        if (monthlyChurnRate <= 0) return avgMrr * 120; // cap at 10 years

        return Math.Round(avgMrr / (monthlyChurnRate / 100m), 2);
    }

    private static bool IsDunningCancellation(Subscription s)
    {
        // Heuristic: cancelled while in PastDue indicates involuntary
        return s.Status == SubscriptionStatus.Cancelled
            && s.CancelledAt.HasValue;
    }
}

public record ChurnMetrics
{
    public DateTime PeriodStart { get; init; }
    public DateTime PeriodEnd { get; init; }
    public int TotalChurned { get; init; }
    public int VoluntaryChurn { get; init; }
    public int InvoluntaryChurn { get; init; }
    public decimal ChurnRate { get; init; }
    public int ActiveAtPeriodStart { get; init; }
}

public record RevenueMovement
{
    public decimal NewMrr { get; init; }
    public decimal ExpansionMrr { get; init; }
    public decimal ContractionMrr { get; init; }
    public decimal NetNewMrr => NewMrr + ExpansionMrr - ContractionMrr;
    public DateTime PeriodStart { get; init; }
    public DateTime PeriodEnd { get; init; }
}
```

---

## 10. Multi-Currency Subscriptions

### Currency-Specific Pricing

Supporting multiple currencies requires maintaining separate price points per plan and locking exchange rates at subscription time to protect both the merchant and the customer from FX volatility.

```csharp
public class CurrencyPrice
{
    public Guid Id { get; set; }
    public Guid PlanId { get; set; }
    public string Currency { get; set; } = default!;
    public decimal Amount { get; set; }
    public DateTime EffectiveFrom { get; set; }
    public DateTime? EffectiveTo { get; set; }
}

public class FxRateLock
{
    public Guid Id { get; set; }
    public Guid SubscriptionId { get; set; }
    public string FromCurrency { get; set; } = default!;
    public string ToCurrency { get; set; } = default!;
    public decimal Rate { get; set; }
    public DateTime LockedAt { get; set; }
    public DateTime ExpiresAt { get; set; }
}
```

### Multi-Currency Service

```csharp
public interface IMultiCurrencyService
{
    Task<decimal> GetPriceInCurrencyAsync(Guid planId, string currency);
    Task<FxRateLock> LockExchangeRateAsync(
        Guid subscriptionId, string fromCurrency, string toCurrency);
    Task<decimal> ConvertAsync(
        decimal amount, string fromCurrency, string toCurrency);
}

public class MultiCurrencyService : IMultiCurrencyService
{
    private readonly SubscriptionDbContext _db;
    private readonly IFxRateProvider _fxRates;

    public MultiCurrencyService(
        SubscriptionDbContext db, IFxRateProvider fxRates)
    {
        _db = db;
        _fxRates = fxRates;
    }

    public async Task<decimal> GetPriceInCurrencyAsync(
        Guid planId, string currency)
    {
        var now = DateTime.UtcNow;

        // Look for explicit currency price
        var currencyPrice = await _db.Set<CurrencyPrice>()
            .Where(cp => cp.PlanId == planId
                      && cp.Currency == currency
                      && cp.EffectiveFrom <= now
                      && (cp.EffectiveTo == null || cp.EffectiveTo > now))
            .OrderByDescending(cp => cp.EffectiveFrom)
            .FirstOrDefaultAsync();

        if (currencyPrice is not null)
            return currencyPrice.Amount;

        // Fall back to FX conversion from base price
        var plan = await _db.Plans.FirstAsync(p => p.Id == planId);
        return await ConvertAsync(plan.BasePrice, plan.Currency, currency);
    }

    public async Task<FxRateLock> LockExchangeRateAsync(
        Guid subscriptionId, string fromCurrency, string toCurrency)
    {
        var rate = await _fxRates.GetRateAsync(fromCurrency, toCurrency);

        var rateLock = new FxRateLock
        {
            Id = Guid.NewGuid(),
            SubscriptionId = subscriptionId,
            FromCurrency = fromCurrency,
            ToCurrency = toCurrency,
            Rate = rate,
            LockedAt = DateTime.UtcNow,
            ExpiresAt = DateTime.UtcNow.AddDays(30) // lock for one billing period
        };

        _db.Set<FxRateLock>().Add(rateLock);
        await _db.SaveChangesAsync();

        return rateLock;
    }

    public async Task<decimal> ConvertAsync(
        decimal amount, string fromCurrency, string toCurrency)
    {
        if (fromCurrency == toCurrency) return amount;

        var rate = await _fxRates.GetRateAsync(fromCurrency, toCurrency);
        return Math.Round(amount * rate, 2);
    }
}

public interface IFxRateProvider
{
    Task<decimal> GetRateAsync(string fromCurrency, string toCurrency);
}
```

### Price Localization

```csharp
public class PriceLocalizationService
{
    private readonly IMultiCurrencyService _currencyService;

    // Map of country codes to preferred currencies
    private static readonly Dictionary<string, string> CountryCurrencyMap = new()
    {
        ["US"] = "USD", ["GB"] = "GBP", ["DE"] = "EUR", ["FR"] = "EUR",
        ["JP"] = "JPY", ["AU"] = "AUD", ["CA"] = "CAD", ["BR"] = "BRL",
        ["IN"] = "INR", ["MX"] = "MXN", ["SE"] = "SEK", ["NO"] = "NOK"
    };

    public PriceLocalizationService(IMultiCurrencyService currencyService)
    {
        _currencyService = currencyService;
    }

    public async Task<LocalizedPrice> GetLocalizedPriceAsync(
        Guid planId, string countryCode)
    {
        var currency = CountryCurrencyMap.GetValueOrDefault(
            countryCode.ToUpperInvariant(), "USD");

        var amount = await _currencyService
            .GetPriceInCurrencyAsync(planId, currency);

        return new LocalizedPrice
        {
            PlanId = planId,
            Currency = currency,
            Amount = amount,
            CountryCode = countryCode,
            FormattedPrice = FormatPrice(amount, currency)
        };
    }

    private static string FormatPrice(decimal amount, string currency)
    {
        return currency switch
        {
            "USD" or "CAD" or "AUD" => $"${amount:N2}",
            "GBP" => $"£{amount:N2}",
            "EUR" => $"€{amount:N2}",
            "JPY" => $"¥{amount:N0}",
            "BRL" => $"R${amount:N2}",
            "INR" => $"₹{amount:N2}",
            _ => $"{amount:N2} {currency}"
        };
    }
}

public record LocalizedPrice
{
    public Guid PlanId { get; init; }
    public string Currency { get; init; } = default!;
    public decimal Amount { get; init; }
    public string CountryCode { get; init; } = default!;
    public string FormattedPrice { get; init; } = default!;
}
```

---

## 11. Provider Integration

### Provider Abstraction

A provider abstraction lets you swap between Stripe Billing, PayPal subscriptions, or a custom billing engine without changing your domain logic.

```csharp
public interface ISubscriptionProvider
{
    string ProviderName { get; }
    Task<ProviderSubscription> CreateSubscriptionAsync(
        CreateProviderSubscriptionRequest request);
    Task<ProviderSubscription> CancelSubscriptionAsync(
        string providerSubscriptionId, bool atPeriodEnd);
    Task<ProviderSubscription> UpdateSubscriptionAsync(
        string providerSubscriptionId, UpdateProviderSubscriptionRequest request);
    Task<ProviderInvoice> GetUpcomingInvoiceAsync(
        string providerSubscriptionId);
    Task HandleWebhookAsync(string payload, string signature);
}

public record CreateProviderSubscriptionRequest
{
    public string CustomerId { get; init; } = default!;
    public string PriceId { get; init; } = default!;
    public int Quantity { get; init; } = 1;
    public int? TrialDays { get; init; }
    public string? PaymentMethodId { get; init; }
    public Dictionary<string, string> Metadata { get; init; } = new();
}

public record UpdateProviderSubscriptionRequest
{
    public string? NewPriceId { get; init; }
    public int? Quantity { get; init; }
    public string ProrationBehavior { get; init; } = "create_prorations";
}

public record ProviderSubscription
{
    public string ProviderId { get; init; } = default!;
    public string Status { get; init; } = default!;
    public DateTime CurrentPeriodStart { get; init; }
    public DateTime CurrentPeriodEnd { get; init; }
}

public record ProviderInvoice
{
    public string ProviderId { get; init; } = default!;
    public decimal AmountDue { get; init; }
    public string Currency { get; init; } = default!;
}
```

### Stripe Billing Integration

```csharp
public class StripeSubscriptionProvider : ISubscriptionProvider
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<StripeSubscriptionProvider> _logger;

    public string ProviderName => "Stripe";

    public StripeSubscriptionProvider(
        HttpClient httpClient,
        ILogger<StripeSubscriptionProvider> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }

    public async Task<ProviderSubscription> CreateSubscriptionAsync(
        CreateProviderSubscriptionRequest request)
    {
        var parameters = new Dictionary<string, string>
        {
            ["customer"] = request.CustomerId,
            ["items[0][price]"] = request.PriceId,
            ["items[0][quantity]"] = request.Quantity.ToString(),
            ["payment_behavior"] = "default_incomplete",
            ["expand[]"] = "latest_invoice.payment_intent"
        };

        if (request.TrialDays.HasValue)
            parameters["trial_period_days"] = request.TrialDays.Value.ToString();

        if (request.PaymentMethodId is not null)
            parameters["default_payment_method"] = request.PaymentMethodId;

        foreach (var (key, value) in request.Metadata)
            parameters[$"metadata[{key}]"] = value;

        var response = await _httpClient.PostAsync(
            "https://api.stripe.com/v1/subscriptions",
            new FormUrlEncodedContent(parameters));

        response.EnsureSuccessStatusCode();

        var json = await response.Content.ReadFromJsonAsync<JsonElement>();

        return new ProviderSubscription
        {
            ProviderId = json.GetProperty("id").GetString()!,
            Status = json.GetProperty("status").GetString()!,
            CurrentPeriodStart = DateTimeOffset.FromUnixTimeSeconds(
                json.GetProperty("current_period_start").GetInt64()).UtcDateTime,
            CurrentPeriodEnd = DateTimeOffset.FromUnixTimeSeconds(
                json.GetProperty("current_period_end").GetInt64()).UtcDateTime
        };
    }

    public async Task<ProviderSubscription> CancelSubscriptionAsync(
        string providerSubscriptionId, bool atPeriodEnd)
    {
        if (atPeriodEnd)
        {
            var response = await _httpClient.PostAsync(
                $"https://api.stripe.com/v1/subscriptions/{providerSubscriptionId}",
                new FormUrlEncodedContent(new Dictionary<string, string>
                {
                    ["cancel_at_period_end"] = "true"
                }));
            response.EnsureSuccessStatusCode();

            var json = await response.Content.ReadFromJsonAsync<JsonElement>();
            return MapToProviderSubscription(json);
        }
        else
        {
            var response = await _httpClient.DeleteAsync(
                $"https://api.stripe.com/v1/subscriptions/{providerSubscriptionId}");
            response.EnsureSuccessStatusCode();

            var json = await response.Content.ReadFromJsonAsync<JsonElement>();
            return MapToProviderSubscription(json);
        }
    }

    public async Task<ProviderSubscription> UpdateSubscriptionAsync(
        string providerSubscriptionId, UpdateProviderSubscriptionRequest request)
    {
        var parameters = new Dictionary<string, string>
        {
            ["proration_behavior"] = request.ProrationBehavior
        };

        if (request.NewPriceId is not null)
            parameters["items[0][price]"] = request.NewPriceId;

        if (request.Quantity.HasValue)
            parameters["items[0][quantity]"] = request.Quantity.Value.ToString();

        var response = await _httpClient.PostAsync(
            $"https://api.stripe.com/v1/subscriptions/{providerSubscriptionId}",
            new FormUrlEncodedContent(parameters));

        response.EnsureSuccessStatusCode();
        var json = await response.Content.ReadFromJsonAsync<JsonElement>();
        return MapToProviderSubscription(json);
    }

    public async Task<ProviderInvoice> GetUpcomingInvoiceAsync(
        string providerSubscriptionId)
    {
        var response = await _httpClient.GetAsync(
            $"https://api.stripe.com/v1/invoices/upcoming"
            + $"?subscription={providerSubscriptionId}");

        response.EnsureSuccessStatusCode();
        var json = await response.Content.ReadFromJsonAsync<JsonElement>();

        return new ProviderInvoice
        {
            ProviderId = json.GetProperty("id").GetString() ?? "upcoming",
            AmountDue = json.GetProperty("amount_due").GetInt64() / 100m,
            Currency = json.GetProperty("currency").GetString()!
        };
    }

    public async Task HandleWebhookAsync(string payload, string signature)
    {
        // Verify webhook signature and dispatch events
        _logger.LogInformation("Processing Stripe webhook");

        var json = JsonDocument.Parse(payload);
        var eventType = json.RootElement
            .GetProperty("type").GetString();

        _logger.LogInformation("Stripe event: {Type}", eventType);

        // Route to appropriate handler based on event type
        // See [16-STRIPE-INTEGRATION](16-STRIPE-INTEGRATION.md) for details
    }

    private static ProviderSubscription MapToProviderSubscription(JsonElement json)
    {
        return new ProviderSubscription
        {
            ProviderId = json.GetProperty("id").GetString()!,
            Status = json.GetProperty("status").GetString()!,
            CurrentPeriodStart = DateTimeOffset.FromUnixTimeSeconds(
                json.GetProperty("current_period_start").GetInt64()).UtcDateTime,
            CurrentPeriodEnd = DateTimeOffset.FromUnixTimeSeconds(
                json.GetProperty("current_period_end").GetInt64()).UtcDateTime
        };
    }
}
```

### Provider Comparison

| Feature | Stripe Billing | PayPal Subscriptions | Custom Engine |
|---------|---------------|---------------------|---------------|
| **Pricing models** | Flat, tiered, graduated, per-unit | Flat, tiered, volume | Any |
| **Usage-based billing** | Metered usage API | Limited | Full control |
| **Proration** | Automatic | Manual | Full control |
| **Trial management** | Built-in | Built-in | Full control |
| **Dunning** | Smart Retries (configurable) | Basic retry | Full control |
| **Multi-currency** | 135+ currencies | 25 currencies | Any |
| **Invoice generation** | Automatic | Basic | Full control |
| **Webhooks** | Comprehensive event set | Limited event set | Custom |
| **Setup complexity** | Low | Medium | High |
| **Revenue share** | 0.5–0.8% per invoice | PayPal transaction fees | None |

---

## 12. Implementation Checklist

### Phase 1: Data Model & Core Billing (Week 1–2)

- [ ] Define Plan, PricingTier, Subscription, SubscriptionItem entities
- [ ] Configure EF Core mappings with proper precision and indexes
- [ ] Implement PricingCalculator for flat, tiered, volume, graduated, staircase models
- [ ] Implement billing anchor date logic
- [ ] Build subscription creation and cancellation flows
- [ ] Add subscription status state machine with transition validation

### Phase 2: Usage & Seat-Based Billing (Week 3–4)

- [ ] Create UsageEvent ingestion pipeline with idempotency
- [ ] Implement aggregation strategies (sum, max, unique count)
- [ ] Build rating engine with overage handling
- [ ] Implement seat management service (add/remove mid-cycle)
- [ ] Add proration engine for quantity and plan changes
- [ ] Implement true-up billing reconciliation

### Phase 3: Trial & Lifecycle Management (Week 5)

- [ ] Build trial management service with configurable trial periods
- [ ] Implement trial abuse prevention (fingerprinting, IP checks)
- [ ] Add trial-to-paid conversion flow
- [ ] Define and publish subscription lifecycle domain events
- [ ] Implement event handlers for notifications and analytics
- [ ] Add trial expiration background job

### Phase 4: Dunning & Payment Recovery (Week 6)

- [ ] Configure smart retry scheduling with exponential backoff
- [ ] Implement dunning service with grace periods
- [ ] Build payment retry processor (scheduled background job)
- [ ] Add dunning email notification sequences
- [ ] Implement payment method update flow for failed payments
- [ ] Track voluntary vs. involuntary churn separately

### Phase 5: Invoicing & Revenue (Week 7–8)

- [ ] Build automated invoice generation service
- [ ] Integrate tax calculation for invoice line items
- [ ] Implement credit note issuance
- [ ] Add invoice number generation and PDF rendering
- [ ] Build MRR/ARR calculation service
- [ ] Implement churn rate, LTV, and expansion/contraction metrics

### Phase 6: Multi-Currency & Provider Integration (Week 9–10)

- [ ] Add currency-specific pricing per plan
- [ ] Implement FX rate locking at subscription creation
- [ ] Build price localization by country
- [ ] Integrate Stripe Billing (or primary provider) via abstraction layer
- [ ] Implement webhook handling for provider subscription events
- [ ] Add provider fallback and reconciliation logic

---

## Additional Resources

- **Stripe Billing Documentation** — https://stripe.com/docs/billing
- **PayPal Subscriptions API** — https://developer.paypal.com/docs/subscriptions/
- **SaaS Metrics 2.0** — https://www.forentrepreneurs.com/saas-metrics-2/
- **Revenue Recognition (ASC 606)** — https://asc.fasb.org/606

---

> **Next Steps:**
>
> - Review [16-STRIPE-INTEGRATION](16-STRIPE-INTEGRATION.md) for Stripe subscription API details.
> - Review [17-PAYPAL-INTEGRATION](17-PAYPAL-INTEGRATION.md) for PayPal subscription API details.
> - See [25-ECOMMERCE-SUBSCRIPTION-GLOSSARY](25-ECOMMERCE-SUBSCRIPTION-GLOSSARY.md) for subscription terminology reference.
> - See [18-COMMON-SCENARIOS](18-COMMON-SCENARIOS.md) for end-to-end subscription billing scenarios.
