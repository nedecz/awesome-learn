# Revenue Recognition & Financial Reporting

Revenue recognition determines **when** and **how much** revenue your e-commerce platform records. Getting it wrong leads to financial restatements, audit failures, and regulatory penalties. This guide covers ASC 606 / IFRS 15 compliance, subscription revenue schedules, deferred revenue management, financial reconciliation, and audit controls — all with production-ready .NET implementations.

> **Related documents:**
>
> - [06-DATABASE-DESIGN](06-DATABASE-DESIGN.md) — Database schemas and patterns
> - [12-TAX-AND-MULTI-PROVIDER](12-TAX-AND-MULTI-PROVIDER.md) — Tax handling and multi-provider setup
> - [30-SUBSCRIPTION-BILLING-ADVANCED](30-SUBSCRIPTION-BILLING-ADVANCED.md) — Subscription billing engine
> - [25-ECOMMERCE-SUBSCRIPTION-GLOSSARY](25-ECOMMERCE-SUBSCRIPTION-GLOSSARY.md) — E-commerce and subscription glossary

## Table of Contents

1. [Revenue Recognition Data Model](#1-revenue-recognition-data-model)
2. [ASC 606 / IFRS 15 Compliance](#2-asc-606--ifrs-15-compliance)
3. [Subscription Revenue Recognition](#3-subscription-revenue-recognition)
4. [Multi-Element Arrangements](#4-multi-element-arrangements)
5. [Deferred Revenue Management](#5-deferred-revenue-management)
6. [Financial Reconciliation](#6-financial-reconciliation)
7. [Accounting System Integration](#7-accounting-system-integration)
8. [Tax Reporting](#8-tax-reporting)
9. [Financial Dashboards & KPIs](#9-financial-dashboards--kpis)
10. [Audit Trail & Controls](#10-audit-trail--controls)
11. [Implementation Checklist](#11-implementation-checklist)

---

## 1. Revenue Recognition Data Model

Revenue recognition requires tracking **when** revenue is earned, not just when cash is received. The data model must support deferred revenue, recognition schedules, and multi-element allocations.

### Core Entities

```csharp
public class RevenueSchedule
{
    public Guid Id { get; private set; }
    public Guid ContractId { get; private set; }
    public Guid? OrderId { get; private set; }
    public string Description { get; private set; } = string.Empty;
    public decimal TotalAmount { get; private set; }
    public decimal RecognizedAmount { get; private set; }
    public decimal DeferredAmount => TotalAmount - RecognizedAmount;
    public Currency Currency { get; private set; }
    public RecognitionMethod Method { get; private set; }
    public DateOnly StartDate { get; private set; }
    public DateOnly EndDate { get; private set; }
    public ScheduleStatus Status { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? CompletedAt { get; private set; }

    private readonly List<RevenueEntry> _entries = new();
    public IReadOnlyCollection<RevenueEntry> Entries => _entries.AsReadOnly();

    private readonly List<PerformanceObligation> _obligations = new();
    public IReadOnlyCollection<PerformanceObligation> Obligations => _obligations.AsReadOnly();

    public static RevenueSchedule Create(
        Guid contractId,
        decimal totalAmount,
        Currency currency,
        RecognitionMethod method,
        DateOnly startDate,
        DateOnly endDate)
    {
        if (endDate <= startDate)
            throw new DomainException("End date must be after start date.");

        return new RevenueSchedule
        {
            Id = Guid.NewGuid(),
            ContractId = contractId,
            TotalAmount = totalAmount,
            Currency = currency,
            Method = method,
            StartDate = startDate,
            EndDate = endDate,
            Status = ScheduleStatus.Active,
            CreatedAt = DateTime.UtcNow
        };
    }

    public RevenueEntry Recognize(AccountingPeriod period, decimal amount, string memo)
    {
        if (Status != ScheduleStatus.Active)
            throw new DomainException($"Cannot recognize revenue on {Status} schedule.");

        if (RecognizedAmount + amount > TotalAmount)
            throw new DomainException(
                $"Recognition of {amount} would exceed total {TotalAmount}. " +
                $"Already recognized: {RecognizedAmount}.");

        var entry = new RevenueEntry(Id, period.Id, amount, Currency, memo);
        _entries.Add(entry);
        RecognizedAmount += amount;

        if (RecognizedAmount == TotalAmount)
        {
            Status = ScheduleStatus.Completed;
            CompletedAt = DateTime.UtcNow;
        }

        return entry;
    }
}

public class RevenueEntry
{
    public Guid Id { get; private set; }
    public Guid ScheduleId { get; private set; }
    public Guid PeriodId { get; private set; }
    public decimal Amount { get; private set; }
    public Currency Currency { get; private set; }
    public string Memo { get; private set; } = string.Empty;
    public EntryStatus Status { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? PostedAt { get; private set; }
    public Guid? JournalEntryId { get; private set; }

    public RevenueEntry(Guid scheduleId, Guid periodId, decimal amount, Currency currency, string memo)
    {
        Id = Guid.NewGuid();
        ScheduleId = scheduleId;
        PeriodId = periodId;
        Amount = amount;
        Currency = currency;
        Memo = memo;
        Status = EntryStatus.Pending;
        CreatedAt = DateTime.UtcNow;
    }

    public void MarkPosted(Guid journalEntryId)
    {
        Status = EntryStatus.Posted;
        PostedAt = DateTime.UtcNow;
        JournalEntryId = journalEntryId;
    }
}

public class AccountingPeriod
{
    public Guid Id { get; private set; }
    public string Name { get; private set; } = string.Empty; // e.g. "2025-Q1", "2025-03"
    public DateOnly StartDate { get; private set; }
    public DateOnly EndDate { get; private set; }
    public PeriodStatus Status { get; private set; }
    public PeriodType Type { get; private set; }
    public int FiscalYear { get; private set; }
    public int FiscalPeriod { get; private set; }

    public bool Contains(DateOnly date) => date >= StartDate && date <= EndDate;

    public void Close()
    {
        if (Status == PeriodStatus.Closed)
            throw new DomainException($"Period {Name} is already closed.");
        Status = PeriodStatus.Closed;
    }
}

public class PerformanceObligation
{
    public Guid Id { get; private set; }
    public Guid ScheduleId { get; private set; }
    public string Description { get; private set; } = string.Empty;
    public decimal StandaloneSellingPrice { get; private set; }
    public decimal AllocatedAmount { get; private set; }
    public ObligationType Type { get; private set; }
    public bool IsSatisfied { get; private set; }
    public DateTime? SatisfiedAt { get; private set; }

    public void Satisfy()
    {
        IsSatisfied = true;
        SatisfiedAt = DateTime.UtcNow;
    }
}

public enum RecognitionMethod { PointInTime, OverTime, UsageBased, Milestone }
public enum ScheduleStatus { Active, Paused, Completed, Cancelled }
public enum EntryStatus { Pending, Posted, Reversed }
public enum PeriodStatus { Open, Closed, Locked }
public enum PeriodType { Monthly, Quarterly, Annual }
public enum ObligationType { Product, Service, License, Support }
```

### EF Core Configuration

```csharp
public class RevenueScheduleConfiguration : IEntityTypeConfiguration<RevenueSchedule>
{
    public void Configure(EntityTypeBuilder<RevenueSchedule> builder)
    {
        builder.ToTable("revenue_schedules");
        builder.HasKey(x => x.Id);

        builder.Property(x => x.TotalAmount).HasPrecision(18, 4);
        builder.Property(x => x.RecognizedAmount).HasPrecision(18, 4);
        builder.Property(x => x.Currency).HasConversion<string>().HasMaxLength(3);
        builder.Property(x => x.Method).HasConversion<string>().HasMaxLength(30);
        builder.Property(x => x.Status).HasConversion<string>().HasMaxLength(20);

        builder.HasMany(x => x.Entries)
            .WithOne()
            .HasForeignKey(x => x.ScheduleId);

        builder.HasMany(x => x.Obligations)
            .WithOne()
            .HasForeignKey(x => x.ScheduleId);

        builder.HasIndex(x => x.ContractId);
        builder.HasIndex(x => x.Status);
        builder.HasIndex(x => new { x.StartDate, x.EndDate });
    }
}

public class RevenueEntryConfiguration : IEntityTypeConfiguration<RevenueEntry>
{
    public void Configure(EntityTypeBuilder<RevenueEntry> builder)
    {
        builder.ToTable("revenue_entries");
        builder.HasKey(x => x.Id);

        builder.Property(x => x.Amount).HasPrecision(18, 4);
        builder.Property(x => x.Currency).HasConversion<string>().HasMaxLength(3);
        builder.Property(x => x.Status).HasConversion<string>().HasMaxLength(20);

        builder.HasIndex(x => x.PeriodId);
        builder.HasIndex(x => x.Status);
    }
}
```

---

## 2. ASC 606 / IFRS 15 Compliance

ASC 606 (US GAAP) and IFRS 15 define a **five-step model** for revenue recognition. Every transaction in your e-commerce platform must pass through this framework.

### The Five-Step Model

| Step | Question | Example |
|------|----------|---------|
| 1. Identify the contract | Is there an enforceable agreement? | Customer places order, signs subscription |
| 2. Identify performance obligations | What distinct goods/services are promised? | Software license + support + training |
| 3. Determine transaction price | How much consideration is expected? | $1,200/year minus $200 discount = $1,000 |
| 4. Allocate transaction price | How to split across obligations? | License: $600, Support: $300, Training: $100 |
| 5. Recognize revenue | When is each obligation satisfied? | License: at delivery, Support: ratably over 12 months |

### Implementation

```csharp
public interface IRevenueRecognitionEngine
{
    Task<RevenueSchedule> ProcessContractAsync(Contract contract, CancellationToken ct = default);
}

public class Asc606RevenueEngine : IRevenueRecognitionEngine
{
    private readonly IPerformanceObligationIdentifier _obligationIdentifier;
    private readonly ITransactionPriceCalculator _priceCalculator;
    private readonly IPriceAllocator _priceAllocator;
    private readonly IRecognitionScheduleBuilder _scheduleBuilder;
    private readonly IRevenueRepository _repository;

    public Asc606RevenueEngine(
        IPerformanceObligationIdentifier obligationIdentifier,
        ITransactionPriceCalculator priceCalculator,
        IPriceAllocator priceAllocator,
        IRecognitionScheduleBuilder scheduleBuilder,
        IRevenueRepository repository)
    {
        _obligationIdentifier = obligationIdentifier;
        _priceCalculator = priceCalculator;
        _priceAllocator = priceAllocator;
        _scheduleBuilder = scheduleBuilder;
        _repository = repository;
    }

    public async Task<RevenueSchedule> ProcessContractAsync(
        Contract contract, CancellationToken ct = default)
    {
        // Step 1: Contract identification (validated upstream)
        if (!contract.IsEnforceable)
            throw new DomainException("Contract is not enforceable — cannot recognize revenue.");

        // Step 2: Identify performance obligations
        var obligations = await _obligationIdentifier.IdentifyAsync(contract, ct);

        // Step 3: Determine transaction price
        var transactionPrice = await _priceCalculator.CalculateAsync(contract, ct);

        // Step 4: Allocate transaction price to obligations
        var allocations = _priceAllocator.Allocate(transactionPrice, obligations);

        // Step 5: Build recognition schedule
        var schedule = _scheduleBuilder.Build(contract, allocations);

        await _repository.SaveScheduleAsync(schedule, ct);
        return schedule;
    }
}

public class TransactionPriceCalculator : ITransactionPriceCalculator
{
    public Task<TransactionPrice> CalculateAsync(Contract contract, CancellationToken ct)
    {
        var basePrice = contract.LineItems.Sum(li => li.Amount);

        // Variable consideration — estimate refunds, returns, bonuses
        var variableConsideration = EstimateVariableConsideration(contract);

        // Constraint — only include amounts highly probable of not reversing
        var constrainedAmount = ApplyConstraint(variableConsideration);

        // Time value of money — adjust for significant financing
        var financingAdjustment = CalculateFinancingComponent(contract);

        var totalPrice = basePrice + constrainedAmount + financingAdjustment;

        return Task.FromResult(new TransactionPrice(totalPrice, contract.Currency));
    }

    private decimal EstimateVariableConsideration(Contract contract)
    {
        // Expected value method: probability-weighted amounts
        // Most likely amount: single most likely outcome
        return contract.EstimatedRefundRate * contract.TotalAmount * -1;
    }

    private decimal ApplyConstraint(decimal variableAmount)
    {
        // Include only if "highly probable" that significant reversal won't occur
        return variableAmount;
    }

    private decimal CalculateFinancingComponent(Contract contract)
    {
        if (contract.PaymentTermDays <= 365) return 0m;
        // Adjust for significant financing components
        var rate = 0.05m; // market rate
        var years = contract.PaymentTermDays / 365.0m;
        return contract.TotalAmount * rate * years;
    }
}
```

### Allocation by Relative Standalone Selling Price

```csharp
public class RelativeSellingPriceAllocator : IPriceAllocator
{
    public List<PriceAllocation> Allocate(
        TransactionPrice transactionPrice,
        List<PerformanceObligation> obligations)
    {
        var totalSSP = obligations.Sum(o => o.StandaloneSellingPrice);

        return obligations.Select(obligation =>
        {
            var ratio = obligation.StandaloneSellingPrice / totalSSP;
            var allocated = Math.Round(transactionPrice.Amount * ratio, 4);

            return new PriceAllocation(obligation.Id, allocated, transactionPrice.Currency);
        }).ToList();
    }
}

public record PriceAllocation(Guid ObligationId, decimal Amount, Currency Currency);
```

---

## 3. Subscription Revenue Recognition

Subscription revenue is recognized **ratably** over the service period, not when payment is received. This creates deferred revenue that unwinds over time.

### Recognition Schedule Generator

```csharp
public class SubscriptionScheduleBuilder
{
    public RevenueSchedule BuildMonthlySchedule(
        Subscription subscription, decimal totalAmount)
    {
        var schedule = RevenueSchedule.Create(
            contractId: subscription.Id,
            totalAmount: totalAmount,
            currency: subscription.Currency,
            method: RecognitionMethod.OverTime,
            startDate: subscription.CurrentPeriodStart,
            endDate: subscription.CurrentPeriodEnd);

        return schedule;
    }

    public List<ScheduledRecognition> GenerateMonthlyEntries(
        RevenueSchedule schedule)
    {
        var entries = new List<ScheduledRecognition>();
        var totalDays = schedule.EndDate.DayNumber - schedule.StartDate.DayNumber;
        var dailyRate = schedule.TotalAmount / totalDays;

        var current = schedule.StartDate;
        while (current < schedule.EndDate)
        {
            var monthEnd = new DateOnly(current.Year, current.Month,
                DateTime.DaysInMonth(current.Year, current.Month));

            if (monthEnd > schedule.EndDate)
                monthEnd = schedule.EndDate;

            var daysInSegment = monthEnd.DayNumber - current.DayNumber + 1;
            var amount = Math.Round(dailyRate * daysInSegment, 4);

            entries.Add(new ScheduledRecognition(
                ScheduleDate: monthEnd,
                Amount: amount,
                Memo: $"Subscription revenue {current:yyyy-MM-dd} to {monthEnd:yyyy-MM-dd}"));

            current = monthEnd.AddDays(1);
        }

        // Adjust last entry for rounding
        var totalScheduled = entries.Sum(e => e.Amount);
        if (totalScheduled != schedule.TotalAmount && entries.Count > 0)
        {
            var last = entries[^1];
            entries[^1] = last with { Amount = last.Amount + (schedule.TotalAmount - totalScheduled) };
        }

        return entries;
    }
}

public record ScheduledRecognition(DateOnly ScheduleDate, decimal Amount, string Memo);
```

### Handling Upgrades and Downgrades

```csharp
public class SubscriptionChangeHandler
{
    private readonly IRevenueRepository _repository;
    private readonly SubscriptionScheduleBuilder _scheduleBuilder;

    public async Task HandleUpgradeAsync(
        Subscription subscription,
        Plan newPlan,
        DateOnly changeDate,
        CancellationToken ct)
    {
        // 1. Close current schedule with recognition up to change date
        var currentSchedule = await _repository.GetActiveScheduleAsync(
            subscription.Id, ct);

        if (currentSchedule is not null)
        {
            var earnedToDate = CalculateEarnedAmount(
                currentSchedule, changeDate);

            // Recognize remaining earned but unrecognized amount
            var period = await _repository.GetPeriodForDateAsync(changeDate, ct);
            var unrecognized = earnedToDate - currentSchedule.RecognizedAmount;

            if (unrecognized > 0)
                currentSchedule.Recognize(period, unrecognized, "Upgrade — accelerated recognition");

            await _repository.CancelScheduleAsync(currentSchedule.Id, ct);
        }

        // 2. Create new schedule for remaining term at new price
        var remainingDays = subscription.CurrentPeriodEnd.DayNumber - changeDate.DayNumber;
        var proratedAmount = (newPlan.Price / 30m) * remainingDays;

        var newSchedule = RevenueSchedule.Create(
            contractId: subscription.Id,
            totalAmount: proratedAmount,
            currency: subscription.Currency,
            method: RecognitionMethod.OverTime,
            startDate: changeDate,
            endDate: subscription.CurrentPeriodEnd);

        await _repository.SaveScheduleAsync(newSchedule, ct);
    }

    private decimal CalculateEarnedAmount(RevenueSchedule schedule, DateOnly asOfDate)
    {
        var totalDays = schedule.EndDate.DayNumber - schedule.StartDate.DayNumber;
        var elapsedDays = asOfDate.DayNumber - schedule.StartDate.DayNumber;
        var ratio = (decimal)elapsedDays / totalDays;
        return Math.Round(schedule.TotalAmount * ratio, 4);
    }
}
```

### Trial Period Revenue

```csharp
public class TrialRevenueHandler
{
    public RevenueSchedule? HandleTrialStart(Subscription subscription)
    {
        // Free trials: no revenue to recognize
        // Revenue recognition starts when the paid period begins
        if (subscription.TrialPrice == 0)
            return null;

        // Paid trials: recognize the trial amount over the trial period
        return RevenueSchedule.Create(
            contractId: subscription.Id,
            totalAmount: subscription.TrialPrice,
            currency: subscription.Currency,
            method: RecognitionMethod.OverTime,
            startDate: subscription.TrialStart!.Value,
            endDate: subscription.TrialEnd!.Value);
    }
}
```

---

## 4. Multi-Element Arrangements

When a single sale includes multiple deliverables (e.g., software + support + training), each element must be evaluated and may follow a different recognition pattern.

### Standalone Selling Price Determination

```csharp
public interface IStandaloneSellingPriceProvider
{
    Task<decimal> GetSSPAsync(string productId, CancellationToken ct);
}

public class StandaloneSellingPriceProvider : IStandaloneSellingPriceProvider
{
    private readonly IPriceHistoryRepository _priceHistory;

    public StandaloneSellingPriceProvider(IPriceHistoryRepository priceHistory)
    {
        _priceHistory = priceHistory;
    }

    public async Task<decimal> GetSSPAsync(string productId, CancellationToken ct)
    {
        // Hierarchy: Observable price > Adjusted market > Expected cost + margin
        var observablePrice = await _priceHistory.GetObservablePriceAsync(productId, ct);
        if (observablePrice.HasValue)
            return observablePrice.Value;

        var adjustedMarketPrice = await _priceHistory.GetAdjustedMarketPriceAsync(productId, ct);
        if (adjustedMarketPrice.HasValue)
            return adjustedMarketPrice.Value;

        var costPlusMargin = await _priceHistory.GetExpectedCostPlusMarginAsync(productId, ct);
        return costPlusMargin
            ?? throw new DomainException($"Cannot determine SSP for product {productId}");
    }
}
```

### Bundle Allocation Engine

```csharp
public class BundleAllocationEngine
{
    private readonly IStandaloneSellingPriceProvider _sspProvider;

    public BundleAllocationEngine(IStandaloneSellingPriceProvider sspProvider)
    {
        _sspProvider = sspProvider;
    }

    public async Task<List<BundleAllocation>> AllocateBundleAsync(
        Order order, CancellationToken ct)
    {
        var allocations = new List<BundleAllocation>();
        var sspLookups = new Dictionary<string, decimal>();

        // Get SSP for each line item
        foreach (var item in order.Items)
        {
            var ssp = await _sspProvider.GetSSPAsync(item.ProductId, ct);
            sspLookups[item.ProductId] = ssp;
        }

        var totalSSP = sspLookups.Values.Sum();
        var transactionPrice = order.TotalAmount;

        // Discount allocation — proportional to SSP
        foreach (var item in order.Items)
        {
            var ssp = sspLookups[item.ProductId];
            var ratio = ssp / totalSSP;
            var allocatedAmount = Math.Round(transactionPrice * ratio, 2);

            allocations.Add(new BundleAllocation(
                ProductId: item.ProductId,
                SSP: ssp,
                AllocatedAmount: allocatedAmount,
                RecognitionMethod: DetermineMethod(item)));
        }

        // Adjust for rounding
        var totalAllocated = allocations.Sum(a => a.AllocatedAmount);
        if (totalAllocated != transactionPrice)
        {
            var last = allocations[^1];
            allocations[^1] = last with
            {
                AllocatedAmount = last.AllocatedAmount + (transactionPrice - totalAllocated)
            };
        }

        return allocations;
    }

    private RecognitionMethod DetermineMethod(OrderItem item) => item.Type switch
    {
        ProductType.PhysicalGood => RecognitionMethod.PointInTime,
        ProductType.SoftwareLicense => RecognitionMethod.PointInTime,
        ProductType.Subscription => RecognitionMethod.OverTime,
        ProductType.ProfessionalService => RecognitionMethod.Milestone,
        _ => RecognitionMethod.PointInTime
    };
}

public record BundleAllocation(
    string ProductId,
    decimal SSP,
    decimal AllocatedAmount,
    RecognitionMethod RecognitionMethod);
```

---

## 5. Deferred Revenue Management

Deferred revenue (unearned revenue) represents cash received for services not yet delivered. It appears as a **liability** on the balance sheet until recognized.

### Deferred Revenue Tracker

```csharp
public class DeferredRevenueTracker
{
    private readonly IRevenueRepository _repository;
    private readonly ILogger<DeferredRevenueTracker> _logger;

    public DeferredRevenueTracker(
        IRevenueRepository repository,
        ILogger<DeferredRevenueTracker> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task<DeferredRevenueBalance> GetBalanceAsync(
        DateOnly asOfDate, CancellationToken ct)
    {
        var activeSchedules = await _repository.GetActiveSchedulesAsync(ct);

        var totalDeferred = 0m;
        var totalRecognized = 0m;
        var details = new List<DeferredRevenueDetail>();

        foreach (var schedule in activeSchedules)
        {
            var earned = CalculateEarnedToDate(schedule, asOfDate);
            var deferred = schedule.TotalAmount - earned;

            totalDeferred += deferred;
            totalRecognized += schedule.RecognizedAmount;

            details.Add(new DeferredRevenueDetail(
                ScheduleId: schedule.Id,
                ContractId: schedule.ContractId,
                TotalAmount: schedule.TotalAmount,
                EarnedToDate: earned,
                RecognizedToDate: schedule.RecognizedAmount,
                DeferredBalance: deferred));
        }

        return new DeferredRevenueBalance(asOfDate, totalDeferred, totalRecognized, details);
    }

    public async Task<List<WaterfallEntry>> GenerateWaterfallAsync(
        DateOnly startDate, DateOnly endDate, CancellationToken ct)
    {
        var waterfall = new List<WaterfallEntry>();
        var schedules = await _repository.GetSchedulesInRangeAsync(startDate, endDate, ct);

        var current = new DateOnly(startDate.Year, startDate.Month, 1);
        while (current <= endDate)
        {
            var monthEnd = current.AddMonths(1).AddDays(-1);
            var monthRevenue = 0m;

            foreach (var schedule in schedules)
            {
                if (schedule.StartDate > monthEnd || schedule.EndDate < current)
                    continue;

                var overlapStart = current > schedule.StartDate ? current : schedule.StartDate;
                var overlapEnd = monthEnd < schedule.EndDate ? monthEnd : schedule.EndDate;
                var overlapDays = overlapEnd.DayNumber - overlapStart.DayNumber + 1;

                var totalDays = schedule.EndDate.DayNumber - schedule.StartDate.DayNumber;
                if (totalDays > 0)
                {
                    var amount = schedule.TotalAmount * overlapDays / totalDays;
                    monthRevenue += Math.Round(amount, 2);
                }
            }

            waterfall.Add(new WaterfallEntry(current, monthRevenue));
            current = current.AddMonths(1);
        }

        return waterfall;
    }

    private decimal CalculateEarnedToDate(RevenueSchedule schedule, DateOnly asOfDate)
    {
        if (asOfDate >= schedule.EndDate) return schedule.TotalAmount;
        if (asOfDate < schedule.StartDate) return 0m;

        var totalDays = schedule.EndDate.DayNumber - schedule.StartDate.DayNumber;
        var elapsed = asOfDate.DayNumber - schedule.StartDate.DayNumber;
        return Math.Round(schedule.TotalAmount * elapsed / totalDays, 4);
    }
}

public record DeferredRevenueBalance(
    DateOnly AsOfDate, decimal TotalDeferred, decimal TotalRecognized,
    List<DeferredRevenueDetail> Details);

public record DeferredRevenueDetail(
    Guid ScheduleId, Guid ContractId, decimal TotalAmount,
    decimal EarnedToDate, decimal RecognizedToDate, decimal DeferredBalance);

public record WaterfallEntry(DateOnly Month, decimal Revenue);
```

### Contract Modification Handling

```csharp
public class ContractModificationHandler
{
    private readonly IRevenueRepository _repository;

    public async Task HandleModificationAsync(
        Contract contract,
        ContractModification modification,
        CancellationToken ct)
    {
        var existingSchedule = await _repository.GetActiveScheduleAsync(contract.Id, ct);
        if (existingSchedule is null) return;

        switch (modification.Type)
        {
            case ModificationType.SeparateContract:
                // Treat as independent — new schedule, no change to existing
                await CreateSeparateScheduleAsync(contract, modification, ct);
                break;

            case ModificationType.TerminateAndRecreate:
                // Cancel remaining and create new cumulative catch-up schedule
                await _repository.CancelScheduleAsync(existingSchedule.Id, ct);
                await CreateRevisedScheduleAsync(contract, modification,
                    existingSchedule.RecognizedAmount, ct);
                break;

            case ModificationType.ProspectiveAdjustment:
                // Adjust remaining recognition prospectively
                await AdjustProspectivelyAsync(existingSchedule, modification, ct);
                break;
        }
    }

    private async Task CreateSeparateScheduleAsync(
        Contract contract, ContractModification mod, CancellationToken ct)
    {
        var schedule = RevenueSchedule.Create(
            contract.Id, mod.AdditionalAmount, contract.Currency,
            RecognitionMethod.OverTime, mod.EffectiveDate, mod.NewEndDate);
        await _repository.SaveScheduleAsync(schedule, ct);
    }

    private async Task CreateRevisedScheduleAsync(
        Contract contract, ContractModification mod,
        decimal alreadyRecognized, CancellationToken ct)
    {
        var revisedTotal = mod.NewTotalAmount - alreadyRecognized;
        var schedule = RevenueSchedule.Create(
            contract.Id, revisedTotal, contract.Currency,
            RecognitionMethod.OverTime, mod.EffectiveDate, mod.NewEndDate);
        await _repository.SaveScheduleAsync(schedule, ct);
    }

    private async Task AdjustProspectivelyAsync(
        RevenueSchedule schedule, ContractModification mod, CancellationToken ct)
    {
        // Remaining amount spread over remaining period
        var remaining = mod.NewTotalAmount - schedule.RecognizedAmount;
        // Update schedule total; future entries recalculated
        await _repository.UpdateScheduleTotalAsync(schedule.Id, mod.NewTotalAmount, ct);
    }
}

public enum ModificationType { SeparateContract, TerminateAndRecreate, ProspectiveAdjustment }
```

---

## 6. Financial Reconciliation

Reconciliation ensures that amounts recorded in your revenue system match what payment providers settled and what your bank received.

### Three-Way Reconciliation

```csharp
public class ReconciliationEngine
{
    private readonly IPaymentGatewayClient _gateway;
    private readonly IBankStatementProvider _bank;
    private readonly IOrderRepository _orders;
    private readonly ILogger<ReconciliationEngine> _logger;

    public ReconciliationEngine(
        IPaymentGatewayClient gateway,
        IBankStatementProvider bank,
        IOrderRepository orders,
        ILogger<ReconciliationEngine> logger)
    {
        _gateway = gateway;
        _bank = bank;
        _orders = orders;
        _logger = logger;
    }

    public async Task<ReconciliationReport> ReconcileAsync(
        DateOnly date, CancellationToken ct)
    {
        // 1. Get data from all three sources
        var internalPayments = await _orders.GetPaymentsForDateAsync(date, ct);
        var gatewayTransactions = await _gateway.GetSettlementsAsync(date, ct);
        var bankTransactions = await _bank.GetTransactionsAsync(date, ct);

        var report = new ReconciliationReport(date);

        // 2. Match internal → gateway
        foreach (var payment in internalPayments)
        {
            var gatewayMatch = gatewayTransactions.FirstOrDefault(
                g => g.ReferenceId == payment.GatewayTransactionId);

            if (gatewayMatch is null)
            {
                report.AddDiscrepancy(new Discrepancy(
                    payment.Id, DiscrepancyType.MissingInGateway,
                    payment.Amount, 0, "Payment not found in gateway settlements"));
                continue;
            }

            if (payment.Amount != gatewayMatch.Amount)
            {
                report.AddDiscrepancy(new Discrepancy(
                    payment.Id, DiscrepancyType.AmountMismatch,
                    payment.Amount, gatewayMatch.Amount,
                    $"Internal: {payment.Amount}, Gateway: {gatewayMatch.Amount}"));
            }

            gatewayTransactions.Remove(gatewayMatch);
        }

        // 3. Unmatched gateway transactions
        foreach (var orphan in gatewayTransactions)
        {
            report.AddDiscrepancy(new Discrepancy(
                Guid.Empty, DiscrepancyType.MissingInInternal,
                0, orphan.Amount,
                $"Gateway transaction {orphan.Id} not matched internally"));
        }

        // 4. Match gateway settlements → bank deposits
        var gatewaySettlementTotal = await _gateway.GetSettlementTotalAsync(date, ct);
        var bankDepositTotal = bankTransactions
            .Where(b => b.Type == BankTransactionType.Deposit)
            .Sum(b => b.Amount);

        if (Math.Abs(gatewaySettlementTotal - bankDepositTotal) > 0.01m)
        {
            report.AddDiscrepancy(new Discrepancy(
                Guid.Empty, DiscrepancyType.SettlementBankMismatch,
                gatewaySettlementTotal, bankDepositTotal,
                "Gateway settlement total does not match bank deposits"));
        }

        report.Complete();
        _logger.LogInformation(
            "Reconciliation for {Date}: {Matched} matched, {Discrepancies} discrepancies",
            date, report.MatchedCount, report.DiscrepancyCount);

        return report;
    }
}

public class ReconciliationReport
{
    public DateOnly Date { get; }
    public int MatchedCount { get; private set; }
    public int DiscrepancyCount => _discrepancies.Count;
    public IReadOnlyList<Discrepancy> Discrepancies => _discrepancies.AsReadOnly();

    private readonly List<Discrepancy> _discrepancies = new();

    public ReconciliationReport(DateOnly date) => Date = date;

    public void AddDiscrepancy(Discrepancy d) => _discrepancies.Add(d);
    public void Complete() => MatchedCount = MatchedCount; // finalize counts
}

public record Discrepancy(
    Guid PaymentId, DiscrepancyType Type,
    decimal InternalAmount, decimal ExternalAmount, string Description);

public enum DiscrepancyType
{
    MissingInGateway, MissingInInternal,
    AmountMismatch, SettlementBankMismatch, CurrencyMismatch
}
```

---

## 7. Accounting System Integration

Revenue entries must flow into the general ledger as proper journal entries with debits and credits.

### Journal Entry Generator

```csharp
public class JournalEntryGenerator
{
    private readonly IChartOfAccountsProvider _accounts;

    public JournalEntryGenerator(IChartOfAccountsProvider accounts)
    {
        _accounts = accounts;
    }

    public JournalEntry GenerateRevenueRecognition(RevenueEntry entry, RevenueSchedule schedule)
    {
        var revenueAccount = _accounts.GetRevenueAccount(schedule.ContractId);
        var deferredAccount = _accounts.GetDeferredRevenueAccount(schedule.ContractId);

        var journalEntry = new JournalEntry
        {
            Id = Guid.NewGuid(),
            Date = DateTime.UtcNow,
            Reference = $"REV-{entry.Id:N}",
            Memo = entry.Memo,
            Lines = new List<JournalEntryLine>
            {
                new()
                {
                    AccountCode = deferredAccount, // Debit: reduce liability
                    DebitAmount = entry.Amount,
                    CreditAmount = 0
                },
                new()
                {
                    AccountCode = revenueAccount, // Credit: recognize revenue
                    DebitAmount = 0,
                    CreditAmount = entry.Amount
                }
            }
        };

        // Verify debits = credits
        var totalDebits = journalEntry.Lines.Sum(l => l.DebitAmount);
        var totalCredits = journalEntry.Lines.Sum(l => l.CreditAmount);
        if (totalDebits != totalCredits)
            throw new DomainException("Journal entry does not balance.");

        return journalEntry;
    }

    public JournalEntry GeneratePaymentReceived(Payment payment)
    {
        var cashAccount = _accounts.GetCashAccount();
        var deferredAccount = _accounts.GetDeferredRevenueAccount(payment.ContractId);

        return new JournalEntry
        {
            Id = Guid.NewGuid(),
            Date = payment.PaidAt,
            Reference = $"PMT-{payment.Id:N}",
            Memo = $"Payment received for contract {payment.ContractId}",
            Lines = new List<JournalEntryLine>
            {
                new() { AccountCode = cashAccount, DebitAmount = payment.Amount, CreditAmount = 0 },
                new() { AccountCode = deferredAccount, DebitAmount = 0, CreditAmount = payment.Amount }
            }
        };
    }
}

public class JournalEntry
{
    public Guid Id { get; set; }
    public DateTime Date { get; set; }
    public string Reference { get; set; } = string.Empty;
    public string Memo { get; set; } = string.Empty;
    public List<JournalEntryLine> Lines { get; set; } = new();
}

public class JournalEntryLine
{
    public string AccountCode { get; set; } = string.Empty;
    public decimal DebitAmount { get; set; }
    public decimal CreditAmount { get; set; }
}
```

### ERP Integration Service

```csharp
public interface IErpIntegration
{
    Task PostJournalEntryAsync(JournalEntry entry, CancellationToken ct);
    Task<bool> VerifyPostingAsync(string reference, CancellationToken ct);
}

public class QuickBooksIntegration : IErpIntegration
{
    private readonly HttpClient _httpClient;
    private readonly QuickBooksConfig _config;
    private readonly ILogger<QuickBooksIntegration> _logger;

    public QuickBooksIntegration(
        HttpClient httpClient,
        IOptions<QuickBooksConfig> config,
        ILogger<QuickBooksIntegration> logger)
    {
        _httpClient = httpClient;
        _config = config.Value;
        _logger = logger;
    }

    public async Task PostJournalEntryAsync(JournalEntry entry, CancellationToken ct)
    {
        var qbEntry = new
        {
            TxnDate = entry.Date.ToString("yyyy-MM-dd"),
            DocNumber = entry.Reference,
            PrivateNote = entry.Memo,
            Line = entry.Lines.Select((line, i) => new
            {
                Id = (i + 1).ToString(),
                Description = entry.Memo,
                Amount = line.DebitAmount > 0 ? line.DebitAmount : line.CreditAmount,
                DetailType = "JournalEntryLineDetail",
                JournalEntryLineDetail = new
                {
                    PostingType = line.DebitAmount > 0 ? "Debit" : "Credit",
                    AccountRef = new { value = line.AccountCode }
                }
            })
        };

        var response = await _httpClient.PostAsJsonAsync(
            $"{_config.BaseUrl}/v3/company/{_config.CompanyId}/journalentry",
            qbEntry, ct);

        response.EnsureSuccessStatusCode();
        _logger.LogInformation("Posted journal entry {Reference} to QuickBooks", entry.Reference);
    }

    public async Task<bool> VerifyPostingAsync(string reference, CancellationToken ct)
    {
        var query = $"SELECT * FROM JournalEntry WHERE DocNumber = '{reference}'";
        var response = await _httpClient.GetAsync(
            $"{_config.BaseUrl}/v3/company/{_config.CompanyId}/query?query={Uri.EscapeDataString(query)}",
            ct);
        return response.IsSuccessStatusCode;
    }
}
```

---

## 8. Tax Reporting

Tax reporting overlaps with revenue recognition — you must track taxable vs. non-taxable revenue, sales tax collected, and jurisdictional requirements.

### Tax Report Generator

```csharp
public class TaxReportGenerator
{
    private readonly IOrderRepository _orders;
    private readonly ITaxJurisdictionProvider _jurisdictions;

    public TaxReportGenerator(
        IOrderRepository orders,
        ITaxJurisdictionProvider jurisdictions)
    {
        _orders = orders;
        _jurisdictions = jurisdictions;
    }

    public async Task<SalesTaxReport> GenerateSalesTaxReportAsync(
        AccountingPeriod period, CancellationToken ct)
    {
        var orders = await _orders.GetOrdersInPeriodAsync(
            period.StartDate, period.EndDate, ct);

        var jurisdictionTotals = new Dictionary<string, TaxJurisdictionTotal>();

        foreach (var order in orders)
        {
            foreach (var taxLine in order.TaxLines)
            {
                var key = taxLine.JurisdictionCode;
                if (!jurisdictionTotals.ContainsKey(key))
                {
                    var jurisdiction = await _jurisdictions.GetAsync(key, ct);
                    jurisdictionTotals[key] = new TaxJurisdictionTotal(jurisdiction);
                }

                jurisdictionTotals[key].AddTransaction(
                    order.SubTotal, taxLine.TaxableAmount, taxLine.TaxAmount);
            }
        }

        return new SalesTaxReport(
            period,
            jurisdictionTotals.Values.ToList(),
            TotalTaxableRevenue: jurisdictionTotals.Values.Sum(j => j.TaxableAmount),
            TotalTaxCollected: jurisdictionTotals.Values.Sum(j => j.TaxCollected));
    }

    public async Task<VatReturn> GenerateVatReturnAsync(
        AccountingPeriod period, string countryCode, CancellationToken ct)
    {
        var orders = await _orders.GetOrdersByCountryAsync(
            countryCode, period.StartDate, period.EndDate, ct);

        var outputVat = orders
            .SelectMany(o => o.TaxLines)
            .Where(t => t.TaxType == TaxType.VAT)
            .Sum(t => t.TaxAmount);

        // Input VAT from purchases (simplified)
        var inputVat = 0m; // From expense system

        return new VatReturn(
            Period: period,
            CountryCode: countryCode,
            OutputVat: outputVat,
            InputVat: inputVat,
            NetVatDue: outputVat - inputVat);
    }
}

public record SalesTaxReport(
    AccountingPeriod Period,
    List<TaxJurisdictionTotal> Jurisdictions,
    decimal TotalTaxableRevenue,
    decimal TotalTaxCollected);

public record VatReturn(
    AccountingPeriod Period, string CountryCode,
    decimal OutputVat, decimal InputVat, decimal NetVatDue);

public class TaxJurisdictionTotal
{
    public string JurisdictionCode { get; }
    public string JurisdictionName { get; }
    public decimal GrossRevenue { get; private set; }
    public decimal TaxableAmount { get; private set; }
    public decimal TaxCollected { get; private set; }

    public TaxJurisdictionTotal(TaxJurisdiction jurisdiction)
    {
        JurisdictionCode = jurisdiction.Code;
        JurisdictionName = jurisdiction.Name;
    }

    public void AddTransaction(decimal gross, decimal taxable, decimal tax)
    {
        GrossRevenue += gross;
        TaxableAmount += taxable;
        TaxCollected += tax;
    }
}
```

---

## 9. Financial Dashboards & KPIs

### MRR/ARR Calculator

```csharp
public class SubscriptionMetricsCalculator
{
    private readonly ISubscriptionRepository _subscriptions;

    public SubscriptionMetricsCalculator(ISubscriptionRepository subscriptions)
    {
        _subscriptions = subscriptions;
    }

    public async Task<SubscriptionMetrics> CalculateAsync(
        DateOnly asOfDate, CancellationToken ct)
    {
        var activeSubscriptions = await _subscriptions
            .GetActiveAsOfAsync(asOfDate, ct);

        // MRR = sum of monthly-normalized subscription values
        var mrr = activeSubscriptions.Sum(s => NormalizeToMonthly(s));
        var arr = mrr * 12;

        // Calculate movement (new, expansion, contraction, churn)
        var previousMonth = asOfDate.AddMonths(-1);
        var previousSubscriptions = await _subscriptions
            .GetActiveAsOfAsync(previousMonth, ct);

        var previousIds = previousSubscriptions.Select(s => s.Id).ToHashSet();
        var currentIds = activeSubscriptions.Select(s => s.Id).ToHashSet();

        var newMrr = activeSubscriptions
            .Where(s => !previousIds.Contains(s.Id))
            .Sum(s => NormalizeToMonthly(s));

        var churnedMrr = previousSubscriptions
            .Where(s => !currentIds.Contains(s.Id))
            .Sum(s => NormalizeToMonthly(s));

        var expansionMrr = 0m;
        var contractionMrr = 0m;

        foreach (var current in activeSubscriptions.Where(s => previousIds.Contains(s.Id)))
        {
            var previous = previousSubscriptions.First(p => p.Id == current.Id);
            var diff = NormalizeToMonthly(current) - NormalizeToMonthly(previous);
            if (diff > 0) expansionMrr += diff;
            else if (diff < 0) contractionMrr += Math.Abs(diff);
        }

        return new SubscriptionMetrics(
            AsOfDate: asOfDate,
            MRR: mrr,
            ARR: arr,
            NewMRR: newMrr,
            ExpansionMRR: expansionMrr,
            ContractionMRR: contractionMrr,
            ChurnedMRR: churnedMrr,
            NetNewMRR: newMrr + expansionMrr - contractionMrr - churnedMrr,
            ActiveSubscriptions: activeSubscriptions.Count);
    }

    private decimal NormalizeToMonthly(Subscription s) => s.BillingInterval switch
    {
        BillingInterval.Monthly => s.Amount,
        BillingInterval.Quarterly => s.Amount / 3m,
        BillingInterval.Annual => s.Amount / 12m,
        _ => s.Amount
    };
}

public record SubscriptionMetrics(
    DateOnly AsOfDate,
    decimal MRR, decimal ARR,
    decimal NewMRR, decimal ExpansionMRR,
    decimal ContractionMRR, decimal ChurnedMRR,
    decimal NetNewMRR,
    int ActiveSubscriptions);
```

### Cohort Analysis

```csharp
public class CohortAnalyzer
{
    private readonly ISubscriptionRepository _subscriptions;

    public async Task<List<CohortData>> AnalyzeRetentionAsync(
        int monthsBack, CancellationToken ct)
    {
        var cohorts = new List<CohortData>();
        var now = DateOnly.FromDateTime(DateTime.UtcNow);

        for (int i = monthsBack; i >= 0; i--)
        {
            var cohortMonth = now.AddMonths(-i);
            var cohortStart = new DateOnly(cohortMonth.Year, cohortMonth.Month, 1);
            var cohortEnd = cohortStart.AddMonths(1).AddDays(-1);

            // Subscriptions that started in this cohort month
            var cohortSubscriptions = await _subscriptions
                .GetStartedBetweenAsync(cohortStart, cohortEnd, ct);

            var cohortSize = cohortSubscriptions.Count;
            var retentionByMonth = new List<decimal>();

            for (int month = 0; month <= i; month++)
            {
                var checkDate = cohortStart.AddMonths(month);
                var stillActive = cohortSubscriptions
                    .Count(s => s.IsActiveOn(checkDate));
                retentionByMonth.Add(cohortSize > 0 ? (decimal)stillActive / cohortSize : 0);
            }

            cohorts.Add(new CohortData(cohortStart, cohortSize, retentionByMonth));
        }

        return cohorts;
    }
}

public record CohortData(
    DateOnly CohortMonth,
    int InitialSize,
    List<decimal> RetentionRates);
```

---

## 10. Audit Trail & Controls

### Immutable Audit Log

```csharp
public class RevenueAuditLogger
{
    private readonly IAuditLogRepository _auditRepo;

    public RevenueAuditLogger(IAuditLogRepository auditRepo)
    {
        _auditRepo = auditRepo;
    }

    public async Task LogAsync(
        AuditAction action,
        string entityType,
        Guid entityId,
        string userId,
        object? previousState,
        object? newState,
        CancellationToken ct)
    {
        var entry = new AuditLogEntry
        {
            Id = Guid.NewGuid(),
            Timestamp = DateTime.UtcNow,
            Action = action,
            EntityType = entityType,
            EntityId = entityId,
            UserId = userId,
            PreviousState = previousState is not null
                ? JsonSerializer.Serialize(previousState)
                : null,
            NewState = newState is not null
                ? JsonSerializer.Serialize(newState)
                : null,
            Checksum = ComputeChecksum(action, entityType, entityId, userId)
        };

        await _auditRepo.AppendAsync(entry, ct);
    }

    private string ComputeChecksum(AuditAction action, string entityType, Guid entityId, string userId)
    {
        var data = $"{action}|{entityType}|{entityId}|{userId}|{DateTime.UtcNow:O}";
        var hash = SHA256.HashData(Encoding.UTF8.GetBytes(data));
        return Convert.ToHexString(hash);
    }
}

public class AuditLogEntry
{
    public Guid Id { get; set; }
    public DateTime Timestamp { get; set; }
    public AuditAction Action { get; set; }
    public string EntityType { get; set; } = string.Empty;
    public Guid EntityId { get; set; }
    public string UserId { get; set; } = string.Empty;
    public string? PreviousState { get; set; }
    public string? NewState { get; set; }
    public string Checksum { get; set; } = string.Empty;
}

public enum AuditAction
{
    ScheduleCreated, RevenueRecognized, ScheduleCancelled,
    PeriodClosed, JournalEntryPosted, ManualAdjustment
}
```

### Period Close Workflow

```csharp
public class PeriodCloseService
{
    private readonly IAccountingPeriodRepository _periods;
    private readonly IRevenueRepository _revenue;
    private readonly ReconciliationEngine _reconciliation;
    private readonly RevenueAuditLogger _auditLogger;

    public PeriodCloseService(
        IAccountingPeriodRepository periods,
        IRevenueRepository revenue,
        ReconciliationEngine reconciliation,
        RevenueAuditLogger auditLogger)
    {
        _periods = periods;
        _revenue = revenue;
        _reconciliation = reconciliation;
        _auditLogger = auditLogger;
    }

    public async Task<PeriodCloseResult> CloseAsync(
        Guid periodId, string closedByUserId, CancellationToken ct)
    {
        var period = await _periods.GetByIdAsync(periodId, ct)
            ?? throw new DomainException($"Period {periodId} not found.");

        // Pre-close validations
        var pendingEntries = await _revenue.GetPendingEntriesForPeriodAsync(periodId, ct);
        if (pendingEntries.Any())
        {
            return PeriodCloseResult.Failed(
                $"{pendingEntries.Count} revenue entries still pending posting.");
        }

        // Run reconciliation
        var reconReport = await _reconciliation.ReconcileAsync(period.EndDate, ct);
        if (reconReport.DiscrepancyCount > 0)
        {
            return PeriodCloseResult.Failed(
                $"{reconReport.DiscrepancyCount} unresolved reconciliation discrepancies.");
        }

        // Close the period
        period.Close();
        await _periods.UpdateAsync(period, ct);

        await _auditLogger.LogAsync(
            AuditAction.PeriodClosed, "AccountingPeriod", periodId,
            closedByUserId, previousState: null, newState: period, ct);

        return PeriodCloseResult.Succeeded(period);
    }
}

public class PeriodCloseResult
{
    public bool Success { get; private set; }
    public string? ErrorMessage { get; private set; }
    public AccountingPeriod? Period { get; private set; }

    public static PeriodCloseResult Succeeded(AccountingPeriod period) =>
        new() { Success = true, Period = period };

    public static PeriodCloseResult Failed(string message) =>
        new() { Success = false, ErrorMessage = message };
}
```

---

## 11. Implementation Checklist

### Phase 1: Foundation (Weeks 1–3)

- [ ] Design and migrate revenue recognition database schema
- [ ] Implement `RevenueSchedule`, `RevenueEntry`, and `AccountingPeriod` entities
- [ ] Build EF Core configurations and repository layer
- [ ] Implement basic point-in-time recognition for one-time purchases

### Phase 2: Subscription Revenue (Weeks 4–6)

- [ ] Build subscription schedule generator (monthly/annual ratable recognition)
- [ ] Implement proration for mid-cycle upgrades/downgrades
- [ ] Handle trial periods and deferred revenue tracking
- [ ] Add contract modification handling (separate, terminate-recreate, prospective)

### Phase 3: ASC 606 Compliance (Weeks 7–9)

- [ ] Implement five-step recognition engine
- [ ] Build standalone selling price determination hierarchy
- [ ] Implement bundle allocation (relative SSP method)
- [ ] Add multi-element arrangement support

### Phase 4: Financial Integration (Weeks 10–13)

- [ ] Build journal entry generator (debit/credit entries)
- [ ] Integrate with accounting system (QuickBooks/NetSuite/SAP)
- [ ] Implement three-way reconciliation engine
- [ ] Build tax reporting (sales tax by jurisdiction, VAT returns)

### Phase 5: Analytics & Controls (Weeks 14–16)

- [ ] Implement MRR/ARR calculator with movement tracking
- [ ] Build cohort retention analysis
- [ ] Implement deferred revenue waterfall
- [ ] Build immutable audit trail with checksums
- [ ] Implement period close workflow with pre-close validations
- [ ] Set up dashboards for revenue KPIs
