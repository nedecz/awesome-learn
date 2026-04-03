# Discount and Promotion Engine

## Overview

Comprehensive guide to building a discount and promotion engine for e-commerce and payment systems. Covers promotion data modeling, discount calculation pipelines, coupon management, rules engines, dynamic pricing, flash sales, loyalty programs, cart-level vs. item-level discount application, promotion analytics, and fraud prevention for promotions — all with .NET implementation examples.

> **Related documents:**
>
> - [18-COMMON-SCENARIOS](18-COMMON-SCENARIOS.md) — cart checkout scenarios
> - [06-DATABASE-DESIGN](06-DATABASE-DESIGN.md) — database schemas
> - [08-PERFORMANCE-OPTIMIZATION](08-PERFORMANCE-OPTIMIZATION.md) — caching for promotion rules
> - [28-ADDRESS-VERIFICATION-FRAUD-PREVENTION](28-ADDRESS-VERIFICATION-FRAUD-PREVENTION.md) — fraud detection

## Table of Contents

1. [Promotion Data Model](#1-promotion-data-model)
2. [Discount Types and Calculation](#2-discount-types-and-calculation)
3. [Coupon Management](#3-coupon-management)
4. [Promotion Rules Engine](#4-promotion-rules-engine)
5. [Dynamic Pricing](#5-dynamic-pricing)
6. [Flash Sales and Limited-Time Offers](#6-flash-sales-and-limited-time-offers)
7. [Loyalty and Points Programs](#7-loyalty-and-points-programs)
8. [Cart-Level vs. Item-Level Discounts](#8-cart-level-vs-item-level-discounts)
9. [Promotion Analytics](#9-promotion-analytics)
10. [Fraud Prevention for Promotions](#10-fraud-prevention-for-promotions)
11. [Implementation Checklist](#11-implementation-checklist)

---

## 1. Promotion Data Model

### Core Entities

A promotion engine requires a well-structured data model that captures promotion definitions, rules, coupons, and discount applications. The core entities form the backbone of the system and must support complex business scenarios such as stackable discounts, time-bound offers, and customer-segment targeting.

| Entity | Purpose |
|--------|---------|
| **Promotion** | Top-level container for a promotional offer |
| **PromotionRule** | Conditions that must be met for the promotion to apply |
| **Coupon** | A redeemable code linked to a promotion |
| **DiscountApplication** | Records how a discount was applied to an order |

### Entity Definitions

```csharp
public enum DiscountType
{
    Percentage,
    FixedAmount,
    BuyOneGetOne,
    TieredPricing,
    VolumeDiscount,
    BundleDiscount,
    FreeShipping
}

public enum PromotionStatus
{
    Draft,
    Scheduled,
    Active,
    Paused,
    Expired,
    Archived
}

public class Promotion
{
    public Guid Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public DiscountType DiscountType { get; set; }
    public PromotionStatus Status { get; set; }
    public decimal DiscountValue { get; set; }
    public decimal? MaxDiscountAmount { get; set; }
    public decimal? MinOrderAmount { get; set; }
    public DateTime ValidFrom { get; set; }
    public DateTime ValidTo { get; set; }
    public int? MaxRedemptions { get; set; }
    public int CurrentRedemptions { get; set; }
    public int Priority { get; set; }
    public bool IsCombinable { get; set; }
    public bool IsAutoApplied { get; set; }
    public string? CustomerSegment { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }

    public ICollection<PromotionRule> Rules { get; set; } = new List<PromotionRule>();
    public ICollection<Coupon> Coupons { get; set; } = new List<Coupon>();
    public ICollection<DiscountApplication> Applications { get; set; } = new List<DiscountApplication>();
}

public enum RuleType
{
    MinCartTotal,
    MinItemCount,
    SpecificProduct,
    ProductCategory,
    CustomerSegment,
    FirstPurchase,
    GeographicRegion,
    PaymentMethod,
    DayOfWeek,
    TimePeriod
}

public class PromotionRule
{
    public Guid Id { get; set; }
    public Guid PromotionId { get; set; }
    public RuleType RuleType { get; set; }
    public string Operator { get; set; } = string.Empty;  // ">=", "==", "in", "contains"
    public string Value { get; set; } = string.Empty;
    public string? AdditionalData { get; set; }
    public bool IsRequired { get; set; } = true;

    public Promotion Promotion { get; set; } = null!;
}

public enum CouponStatus
{
    Active,
    Redeemed,
    Expired,
    Disabled
}

public class Coupon
{
    public Guid Id { get; set; }
    public Guid PromotionId { get; set; }
    public string Code { get; set; } = string.Empty;
    public CouponStatus Status { get; set; }
    public bool IsSingleUse { get; set; }
    public int? MaxUses { get; set; }
    public int TimesUsed { get; set; }
    public Guid? AssignedToCustomerId { get; set; }
    public string? CouponPoolId { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? RedeemedAt { get; set; }

    public Promotion Promotion { get; set; } = null!;
}

public class DiscountApplication
{
    public Guid Id { get; set; }
    public Guid PromotionId { get; set; }
    public Guid OrderId { get; set; }
    public Guid? CouponId { get; set; }
    public Guid CustomerId { get; set; }
    public decimal DiscountAmount { get; set; }
    public decimal OrderTotalBefore { get; set; }
    public decimal OrderTotalAfter { get; set; }
    public string AppliedToItems { get; set; } = string.Empty;  // JSON array of item IDs
    public DateTime AppliedAt { get; set; }

    public Promotion Promotion { get; set; } = null!;
}
```

### EF Core Configuration

```csharp
public class PromotionDbContext : DbContext
{
    public DbSet<Promotion> Promotions => Set<Promotion>();
    public DbSet<PromotionRule> PromotionRules => Set<PromotionRule>();
    public DbSet<Coupon> Coupons => Set<Coupon>();
    public DbSet<DiscountApplication> DiscountApplications => Set<DiscountApplication>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Promotion>(entity =>
        {
            entity.HasKey(p => p.Id);
            entity.Property(p => p.Name).HasMaxLength(200).IsRequired();
            entity.Property(p => p.DiscountValue).HasPrecision(18, 4);
            entity.Property(p => p.MaxDiscountAmount).HasPrecision(18, 4);
            entity.Property(p => p.MinOrderAmount).HasPrecision(18, 4);
            entity.HasIndex(p => p.Status);
            entity.HasIndex(p => new { p.ValidFrom, p.ValidTo });
            entity.HasIndex(p => p.IsAutoApplied);
        });

        modelBuilder.Entity<PromotionRule>(entity =>
        {
            entity.HasKey(r => r.Id);
            entity.Property(r => r.Operator).HasMaxLength(20);
            entity.Property(r => r.Value).HasMaxLength(500);
            entity.HasOne(r => r.Promotion)
                  .WithMany(p => p.Rules)
                  .HasForeignKey(r => r.PromotionId)
                  .OnDelete(DeleteBehavior.Cascade);
        });

        modelBuilder.Entity<Coupon>(entity =>
        {
            entity.HasKey(c => c.Id);
            entity.Property(c => c.Code).HasMaxLength(50).IsRequired();
            entity.HasIndex(c => c.Code).IsUnique();
            entity.HasIndex(c => c.CouponPoolId);
            entity.HasOne(c => c.Promotion)
                  .WithMany(p => p.Coupons)
                  .HasForeignKey(c => c.PromotionId)
                  .OnDelete(DeleteBehavior.Cascade);
        });

        modelBuilder.Entity<DiscountApplication>(entity =>
        {
            entity.HasKey(d => d.Id);
            entity.Property(d => d.DiscountAmount).HasPrecision(18, 4);
            entity.Property(d => d.OrderTotalBefore).HasPrecision(18, 4);
            entity.Property(d => d.OrderTotalAfter).HasPrecision(18, 4);
            entity.HasIndex(d => d.OrderId);
            entity.HasIndex(d => d.CustomerId);
            entity.HasOne(d => d.Promotion)
                  .WithMany(p => p.Applications)
                  .HasForeignKey(d => d.PromotionId)
                  .OnDelete(DeleteBehavior.Restrict);
        });
    }
}
```

### Validity Period Handling

```csharp
public class PromotionValidityService
{
    private readonly PromotionDbContext _db;
    private readonly TimeProvider _timeProvider;

    public PromotionValidityService(PromotionDbContext db, TimeProvider timeProvider)
    {
        _db = db;
        _timeProvider = timeProvider;
    }

    public async Task<IReadOnlyList<Promotion>> GetActivePromotionsAsync(
        CancellationToken ct = default)
    {
        var now = _timeProvider.GetUtcNow().UtcDateTime;

        return await _db.Promotions
            .Include(p => p.Rules)
            .Where(p => p.Status == PromotionStatus.Active
                     && p.ValidFrom <= now
                     && p.ValidTo > now
                     && (p.MaxRedemptions == null
                         || p.CurrentRedemptions < p.MaxRedemptions))
            .OrderBy(p => p.Priority)
            .ToListAsync(ct);
    }

    /// <summary>
    /// Background job to transition promotions between statuses based on dates.
    /// </summary>
    public async Task UpdatePromotionStatusesAsync(CancellationToken ct = default)
    {
        var now = _timeProvider.GetUtcNow().UtcDateTime;

        // Activate scheduled promotions
        await _db.Promotions
            .Where(p => p.Status == PromotionStatus.Scheduled && p.ValidFrom <= now)
            .ExecuteUpdateAsync(s =>
                s.SetProperty(p => p.Status, PromotionStatus.Active)
                 .SetProperty(p => p.UpdatedAt, now), ct);

        // Expire active promotions
        await _db.Promotions
            .Where(p => p.Status == PromotionStatus.Active && p.ValidTo <= now)
            .ExecuteUpdateAsync(s =>
                s.SetProperty(p => p.Status, PromotionStatus.Expired)
                 .SetProperty(p => p.UpdatedAt, now), ct);
    }
}
```

---

## 2. Discount Types and Calculation

### Discount Type Overview

| Type | Description | Example |
|------|-------------|---------|
| **Percentage** | Reduce price by a percentage | 20% off all items |
| **Fixed Amount** | Subtract a flat amount | $10 off your order |
| **BOGO** | Buy one, get one free/discounted | Buy 2 shirts, get 1 free |
| **Tiered Pricing** | Price decreases at quantity thresholds | 1–9 units: $10, 10–49: $8, 50+: $6 |
| **Volume Discount** | Discount based on total quantity | 5% off when buying 10+ items |
| **Bundle Discount** | Discount for buying specific items together | Buy laptop + case for $50 off |
| **Free Shipping** | Waive shipping costs | Free shipping on orders over $75 |

### Discount Calculator Interface and Implementations

```csharp
public record DiscountContext
{
    public required Cart Cart { get; init; }
    public required Customer Customer { get; init; }
    public required Promotion Promotion { get; init; }
    public IReadOnlyList<CartItem> EligibleItems { get; init; } = Array.Empty<CartItem>();
}

public record DiscountResult
{
    public decimal Amount { get; init; }
    public string Description { get; init; } = string.Empty;
    public IReadOnlyList<Guid> AffectedItemIds { get; init; } = Array.Empty<Guid>();
    public bool Applied { get; init; }
}

public interface IDiscountCalculator
{
    DiscountType DiscountType { get; }
    DiscountResult Calculate(DiscountContext context);
}

public class PercentageDiscountCalculator : IDiscountCalculator
{
    public DiscountType DiscountType => DiscountType.Percentage;

    public DiscountResult Calculate(DiscountContext context)
    {
        var subtotal = context.EligibleItems.Sum(i => i.Price * i.Quantity);
        var discount = subtotal * (context.Promotion.DiscountValue / 100m);

        if (context.Promotion.MaxDiscountAmount.HasValue)
            discount = Math.Min(discount, context.Promotion.MaxDiscountAmount.Value);

        return new DiscountResult
        {
            Amount = Math.Round(discount, 2),
            Description = $"{context.Promotion.DiscountValue}% off",
            AffectedItemIds = context.EligibleItems.Select(i => i.Id).ToList(),
            Applied = discount > 0
        };
    }
}

public class FixedAmountDiscountCalculator : IDiscountCalculator
{
    public DiscountType DiscountType => DiscountType.FixedAmount;

    public DiscountResult Calculate(DiscountContext context)
    {
        var subtotal = context.EligibleItems.Sum(i => i.Price * i.Quantity);
        var discount = Math.Min(context.Promotion.DiscountValue, subtotal);

        return new DiscountResult
        {
            Amount = Math.Round(discount, 2),
            Description = $"${context.Promotion.DiscountValue} off",
            AffectedItemIds = context.EligibleItems.Select(i => i.Id).ToList(),
            Applied = discount > 0
        };
    }
}

public class BuyOneGetOneCalculator : IDiscountCalculator
{
    public DiscountType DiscountType => DiscountType.BuyOneGetOne;

    public DiscountResult Calculate(DiscountContext context)
    {
        // Sort eligible items by price ascending — cheapest items are free
        var items = context.EligibleItems
            .OrderBy(i => i.Price)
            .ToList();

        int totalQuantity = items.Sum(i => i.Quantity);
        int freeItems = totalQuantity / 3;  // Buy 2, get 1 free

        decimal discount = 0m;
        int freeRemaining = freeItems;
        var affectedIds = new List<Guid>();

        foreach (var item in items)
        {
            if (freeRemaining <= 0) break;

            int freeFromThis = Math.Min(item.Quantity, freeRemaining);
            discount += item.Price * freeFromThis;
            freeRemaining -= freeFromThis;
            affectedIds.Add(item.Id);
        }

        return new DiscountResult
        {
            Amount = Math.Round(discount, 2),
            Description = $"Buy 2, get 1 free ({freeItems} free item(s))",
            AffectedItemIds = affectedIds,
            Applied = discount > 0
        };
    }
}
```

### Tiered and Volume Discount Calculators

```csharp
public class TieredPricingCalculator : IDiscountCalculator
{
    public DiscountType DiscountType => DiscountType.TieredPricing;

    public DiscountResult Calculate(DiscountContext context)
    {
        // Tiers stored as JSON in Promotion.Description or a related table
        var tiers = JsonSerializer.Deserialize<List<PriceTier>>(
            context.Promotion.Description) ?? new List<PriceTier>();

        decimal totalDiscount = 0m;
        var affectedIds = new List<Guid>();

        foreach (var item in context.EligibleItems)
        {
            var tier = tiers
                .Where(t => item.Quantity >= t.MinQuantity)
                .OrderByDescending(t => t.MinQuantity)
                .FirstOrDefault();

            if (tier is not null && tier.TierPrice < item.Price)
            {
                totalDiscount += (item.Price - tier.TierPrice) * item.Quantity;
                affectedIds.Add(item.Id);
            }
        }

        return new DiscountResult
        {
            Amount = Math.Round(totalDiscount, 2),
            Description = "Tiered pricing applied",
            AffectedItemIds = affectedIds,
            Applied = totalDiscount > 0
        };
    }
}

public record PriceTier
{
    public int MinQuantity { get; init; }
    public decimal TierPrice { get; init; }
}

public class BundleDiscountCalculator : IDiscountCalculator
{
    public DiscountType DiscountType => DiscountType.BundleDiscount;

    public DiscountResult Calculate(DiscountContext context)
    {
        // Bundle requires all specified products to be in the cart
        var requiredProductIds = JsonSerializer.Deserialize<List<Guid>>(
            context.Promotion.Description) ?? new List<Guid>();

        var cartProductIds = context.EligibleItems.Select(i => i.ProductId).ToHashSet();
        bool allPresent = requiredProductIds.All(id => cartProductIds.Contains(id));

        if (!allPresent)
        {
            return new DiscountResult { Applied = false };
        }

        var discount = context.Promotion.DiscountValue;

        return new DiscountResult
        {
            Amount = Math.Round(discount, 2),
            Description = "Bundle discount applied",
            AffectedItemIds = context.EligibleItems
                .Where(i => requiredProductIds.Contains(i.ProductId))
                .Select(i => i.Id).ToList(),
            Applied = true
        };
    }
}
```

### Discount Calculation Pipeline

```csharp
public class DiscountPipeline
{
    private readonly IEnumerable<IDiscountCalculator> _calculators;
    private readonly ILogger<DiscountPipeline> _logger;

    public DiscountPipeline(
        IEnumerable<IDiscountCalculator> calculators,
        ILogger<DiscountPipeline> logger)
    {
        _calculators = calculators;
        _logger = logger;
    }

    public IReadOnlyList<DiscountResult> CalculateDiscounts(
        Cart cart,
        Customer customer,
        IReadOnlyList<Promotion> promotions)
    {
        var results = new List<DiscountResult>();
        var appliedNonCombinable = false;

        // Sort by priority — lowest number = highest priority
        var sorted = promotions.OrderBy(p => p.Priority).ToList();

        foreach (var promotion in sorted)
        {
            if (appliedNonCombinable && !promotion.IsCombinable)
                continue;

            var calculator = _calculators.FirstOrDefault(
                c => c.DiscountType == promotion.DiscountType);

            if (calculator is null)
            {
                _logger.LogWarning(
                    "No calculator registered for {DiscountType}", promotion.DiscountType);
                continue;
            }

            var context = new DiscountContext
            {
                Cart = cart,
                Customer = customer,
                Promotion = promotion,
                EligibleItems = cart.Items.ToList()
            };

            var result = calculator.Calculate(context);

            if (result.Applied)
            {
                results.Add(result);
                if (!promotion.IsCombinable)
                    appliedNonCombinable = true;

                _logger.LogInformation(
                    "Applied promotion {PromotionId}: {Description} = -{Amount:C}",
                    promotion.Id, result.Description, result.Amount);
            }
        }

        return results;
    }
}
```

---

## 3. Coupon Management

### Single-Use vs. Multi-Use Codes

Coupons fall into two primary categories based on redemption limits:

| Type | Use Case | Example |
|------|----------|---------|
| **Single-use** | Personalized offers, referral rewards | `WELCOME-A1B2C3` — one customer, one time |
| **Multi-use** | Broad campaigns, social media promotions | `SUMMER25` — anyone, unlimited uses |
| **Limited-use** | Controlled campaigns | `FLASH50` — first 500 customers only |

### Coupon Generation Service

```csharp
public interface ICouponGenerationService
{
    Task<Coupon> GenerateSingleCouponAsync(Guid promotionId, CancellationToken ct = default);
    Task<IReadOnlyList<Coupon>> GenerateBulkCouponsAsync(
        Guid promotionId, int count, string? poolId = null, CancellationToken ct = default);
    Task<Coupon> GenerateReferralCouponAsync(
        Guid promotionId, Guid referrerId, CancellationToken ct = default);
}

public class CouponGenerationService : ICouponGenerationService
{
    private readonly PromotionDbContext _db;
    private readonly ILogger<CouponGenerationService> _logger;
    private const string CodeAlphabet = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789";
    private const int CodeLength = 8;

    public CouponGenerationService(PromotionDbContext db, ILogger<CouponGenerationService> logger)
    {
        _db = db;
        _logger = logger;
    }

    public async Task<Coupon> GenerateSingleCouponAsync(
        Guid promotionId, CancellationToken ct = default)
    {
        var code = await GenerateUniqueCodeAsync(ct);

        var coupon = new Coupon
        {
            Id = Guid.NewGuid(),
            PromotionId = promotionId,
            Code = code,
            Status = CouponStatus.Active,
            IsSingleUse = true,
            MaxUses = 1,
            TimesUsed = 0,
            CreatedAt = DateTime.UtcNow
        };

        _db.Coupons.Add(coupon);
        await _db.SaveChangesAsync(ct);

        return coupon;
    }

    public async Task<IReadOnlyList<Coupon>> GenerateBulkCouponsAsync(
        Guid promotionId, int count, string? poolId = null, CancellationToken ct = default)
    {
        var coupons = new List<Coupon>(count);
        var existingCodes = await _db.Coupons
            .Select(c => c.Code)
            .ToHashSetAsync(ct);

        for (int i = 0; i < count; i++)
        {
            string code;
            do
            {
                code = GenerateCode();
            } while (existingCodes.Contains(code));

            existingCodes.Add(code);

            coupons.Add(new Coupon
            {
                Id = Guid.NewGuid(),
                PromotionId = promotionId,
                Code = code,
                Status = CouponStatus.Active,
                IsSingleUse = true,
                MaxUses = 1,
                TimesUsed = 0,
                CouponPoolId = poolId ?? Guid.NewGuid().ToString(),
                CreatedAt = DateTime.UtcNow
            });
        }

        _db.Coupons.AddRange(coupons);
        await _db.SaveChangesAsync(ct);

        _logger.LogInformation("Generated {Count} coupons for promotion {PromotionId}",
            count, promotionId);

        return coupons;
    }

    public async Task<Coupon> GenerateReferralCouponAsync(
        Guid promotionId, Guid referrerId, CancellationToken ct = default)
    {
        var code = $"REF-{referrerId.ToString()[..8].ToUpperInvariant()}-{GenerateCode(4)}";

        var coupon = new Coupon
        {
            Id = Guid.NewGuid(),
            PromotionId = promotionId,
            Code = code,
            Status = CouponStatus.Active,
            IsSingleUse = false,
            MaxUses = null,
            TimesUsed = 0,
            AssignedToCustomerId = referrerId,
            CreatedAt = DateTime.UtcNow
        };

        _db.Coupons.Add(coupon);
        await _db.SaveChangesAsync(ct);

        return coupon;
    }

    private async Task<string> GenerateUniqueCodeAsync(CancellationToken ct)
    {
        string code;
        bool exists;

        do
        {
            code = GenerateCode();
            exists = await _db.Coupons.AnyAsync(c => c.Code == code, ct);
        } while (exists);

        return code;
    }

    private static string GenerateCode(int length = CodeLength)
    {
        return RandomNumberGenerator.GetString(CodeAlphabet, length);
    }
}
```

### Coupon Redemption Service

```csharp
public record CouponRedemptionResult(
    bool Success,
    string? ErrorMessage = null,
    Promotion? Promotion = null,
    decimal DiscountAmount = 0);

public class CouponRedemptionService
{
    private readonly PromotionDbContext _db;
    private readonly TimeProvider _timeProvider;
    private readonly ILogger<CouponRedemptionService> _logger;

    public CouponRedemptionService(
        PromotionDbContext db,
        TimeProvider timeProvider,
        ILogger<CouponRedemptionService> logger)
    {
        _db = db;
        _timeProvider = timeProvider;
        _logger = logger;
    }

    public async Task<CouponRedemptionResult> ValidateAndRedeemAsync(
        string code,
        Guid customerId,
        Guid orderId,
        decimal orderTotal,
        CancellationToken ct = default)
    {
        var coupon = await _db.Coupons
            .Include(c => c.Promotion)
                .ThenInclude(p => p.Rules)
            .FirstOrDefaultAsync(c => c.Code == code, ct);

        if (coupon is null)
            return new CouponRedemptionResult(false, "Invalid coupon code.");

        // Validate coupon status
        if (coupon.Status != CouponStatus.Active)
            return new CouponRedemptionResult(false, "This coupon is no longer active.");

        // Validate usage limits
        if (coupon.MaxUses.HasValue && coupon.TimesUsed >= coupon.MaxUses.Value)
            return new CouponRedemptionResult(false, "This coupon has reached its usage limit.");

        // Validate customer assignment
        if (coupon.AssignedToCustomerId.HasValue &&
            coupon.AssignedToCustomerId.Value != customerId)
            return new CouponRedemptionResult(false, "This coupon is not valid for your account.");

        var promotion = coupon.Promotion;
        var now = _timeProvider.GetUtcNow().UtcDateTime;

        // Validate promotion is active
        if (promotion.Status != PromotionStatus.Active)
            return new CouponRedemptionResult(false, "This promotion is not currently active.");

        if (now < promotion.ValidFrom || now >= promotion.ValidTo)
            return new CouponRedemptionResult(false, "This promotion has expired.");

        // Validate minimum order amount
        if (promotion.MinOrderAmount.HasValue && orderTotal < promotion.MinOrderAmount.Value)
            return new CouponRedemptionResult(false,
                $"Minimum order of {promotion.MinOrderAmount:C} required.");

        // Check if customer already used this promotion
        bool alreadyUsed = await _db.DiscountApplications
            .AnyAsync(d => d.PromotionId == promotion.Id && d.CustomerId == customerId, ct);

        if (alreadyUsed && coupon.IsSingleUse)
            return new CouponRedemptionResult(false, "You have already used this promotion.");

        // Apply redemption within a transaction
        await using var transaction = await _db.Database.BeginTransactionAsync(ct);

        try
        {
            coupon.TimesUsed++;
            coupon.RedeemedAt = now;

            if (coupon.IsSingleUse)
                coupon.Status = CouponStatus.Redeemed;

            promotion.CurrentRedemptions++;

            var application = new DiscountApplication
            {
                Id = Guid.NewGuid(),
                PromotionId = promotion.Id,
                OrderId = orderId,
                CouponId = coupon.Id,
                CustomerId = customerId,
                OrderTotalBefore = orderTotal,
                AppliedAt = now
            };

            _db.DiscountApplications.Add(application);
            await _db.SaveChangesAsync(ct);
            await transaction.CommitAsync(ct);

            _logger.LogInformation(
                "Coupon {Code} redeemed by customer {CustomerId} on order {OrderId}",
                code, customerId, orderId);

            return new CouponRedemptionResult(true, Promotion: promotion);
        }
        catch (Exception ex)
        {
            await transaction.RollbackAsync(ct);
            _logger.LogError(ex, "Failed to redeem coupon {Code}", code);
            return new CouponRedemptionResult(false, "An error occurred. Please try again.");
        }
    }
}
```

---

## 4. Promotion Rules Engine

### Condition Evaluators

The rules engine evaluates whether a promotion's conditions are met for a given cart and customer context. Each rule type has a dedicated evaluator implementing a common interface.

```csharp
public record RuleEvaluationContext
{
    public required Cart Cart { get; init; }
    public required Customer Customer { get; init; }
    public required PromotionRule Rule { get; init; }
}

public interface IRuleEvaluator
{
    RuleType RuleType { get; }
    bool Evaluate(RuleEvaluationContext context);
}

public class MinCartTotalEvaluator : IRuleEvaluator
{
    public RuleType RuleType => RuleType.MinCartTotal;

    public bool Evaluate(RuleEvaluationContext context)
    {
        var threshold = decimal.Parse(context.Rule.Value);
        return context.Cart.Subtotal >= threshold;
    }
}

public class MinItemCountEvaluator : IRuleEvaluator
{
    public RuleType RuleType => RuleType.MinItemCount;

    public bool Evaluate(RuleEvaluationContext context)
    {
        var minCount = int.Parse(context.Rule.Value);
        var totalItems = context.Cart.Items.Sum(i => i.Quantity);
        return totalItems >= minCount;
    }
}

public class CustomerSegmentEvaluator : IRuleEvaluator
{
    public RuleType RuleType => RuleType.CustomerSegment;

    public bool Evaluate(RuleEvaluationContext context)
    {
        var allowedSegments = context.Rule.Value
            .Split(',', StringSplitOptions.TrimEntries);
        return allowedSegments.Contains(context.Customer.Segment);
    }
}

public class FirstPurchaseEvaluator : IRuleEvaluator
{
    public RuleType RuleType => RuleType.FirstPurchase;

    public bool Evaluate(RuleEvaluationContext context)
    {
        return context.Customer.OrderCount == 0;
    }
}

public class SpecificProductEvaluator : IRuleEvaluator
{
    public RuleType RuleType => RuleType.SpecificProduct;

    public bool Evaluate(RuleEvaluationContext context)
    {
        var requiredProductIds = JsonSerializer
            .Deserialize<List<Guid>>(context.Rule.Value) ?? new();
        var cartProductIds = context.Cart.Items
            .Select(i => i.ProductId).ToHashSet();
        return requiredProductIds.All(id => cartProductIds.Contains(id));
    }
}
```

### Rules Engine Orchestrator

```csharp
public class PromotionRulesEngine
{
    private readonly Dictionary<RuleType, IRuleEvaluator> _evaluators;
    private readonly ILogger<PromotionRulesEngine> _logger;

    public PromotionRulesEngine(
        IEnumerable<IRuleEvaluator> evaluators,
        ILogger<PromotionRulesEngine> logger)
    {
        _evaluators = evaluators.ToDictionary(e => e.RuleType);
        _logger = logger;
    }

    /// <summary>
    /// Evaluates all rules for a promotion. All required rules must pass.
    /// Optional rules enhance the discount but are not mandatory.
    /// </summary>
    public PromotionEligibility EvaluateEligibility(
        Promotion promotion, Cart cart, Customer customer)
    {
        var failedRules = new List<string>();
        var passedRules = new List<string>();

        foreach (var rule in promotion.Rules)
        {
            if (!_evaluators.TryGetValue(rule.RuleType, out var evaluator))
            {
                _logger.LogWarning("No evaluator for rule type {RuleType}", rule.RuleType);
                if (rule.IsRequired)
                    failedRules.Add($"Unknown rule: {rule.RuleType}");
                continue;
            }

            var context = new RuleEvaluationContext
            {
                Cart = cart,
                Customer = customer,
                Rule = rule
            };

            bool passed = evaluator.Evaluate(context);

            if (passed)
            {
                passedRules.Add(rule.RuleType.ToString());
            }
            else if (rule.IsRequired)
            {
                failedRules.Add($"{rule.RuleType}: requires {rule.Operator} {rule.Value}");
            }
        }

        return new PromotionEligibility
        {
            PromotionId = promotion.Id,
            IsEligible = failedRules.Count == 0,
            PassedRules = passedRules,
            FailedRules = failedRules
        };
    }
}

public class PromotionEligibility
{
    public Guid PromotionId { get; init; }
    public bool IsEligible { get; init; }
    public IReadOnlyList<string> PassedRules { get; init; } = Array.Empty<string>();
    public IReadOnlyList<string> FailedRules { get; init; } = Array.Empty<string>();
}
```

### Combinability and Priority Management

```csharp
public class PromotionStackingService
{
    private readonly PromotionRulesEngine _rulesEngine;
    private readonly DiscountPipeline _pipeline;
    private readonly ILogger<PromotionStackingService> _logger;

    public PromotionStackingService(
        PromotionRulesEngine rulesEngine,
        DiscountPipeline pipeline,
        ILogger<PromotionStackingService> logger)
    {
        _rulesEngine = rulesEngine;
        _pipeline = pipeline;
        _logger = logger;
    }

    /// <summary>
    /// Determines the optimal set of promotions to apply considering
    /// combinability rules and priority ordering.
    /// </summary>
    public IReadOnlyList<DiscountResult> ResolveApplicableDiscounts(
        Cart cart, Customer customer, IReadOnlyList<Promotion> promotions)
    {
        // Evaluate eligibility for all promotions
        var eligible = promotions
            .Select(p => new
            {
                Promotion = p,
                Eligibility = _rulesEngine.EvaluateEligibility(p, cart, customer)
            })
            .Where(x => x.Eligibility.IsEligible)
            .OrderBy(x => x.Promotion.Priority)
            .Select(x => x.Promotion)
            .ToList();

        // Separate combinable and non-combinable promotions
        var nonCombinable = eligible.Where(p => !p.IsCombinable).ToList();
        var combinable = eligible.Where(p => p.IsCombinable).ToList();

        // Calculate the best non-combinable discount alone
        var bestNonCombinable = nonCombinable
            .Select(p => _pipeline.CalculateDiscounts(cart, customer, new[] { p }))
            .OrderByDescending(r => r.Sum(d => d.Amount))
            .FirstOrDefault();

        // Calculate the combined total of all combinable discounts
        var combinedResults = _pipeline.CalculateDiscounts(cart, customer, combinable);
        var combinedTotal = combinedResults.Sum(d => d.Amount);

        var bestNonCombinableTotal = bestNonCombinable?.Sum(d => d.Amount) ?? 0;

        // Return whichever strategy yields the better discount
        if (bestNonCombinableTotal > combinedTotal)
        {
            _logger.LogInformation(
                "Non-combinable discount ({Amount:C}) beats combined ({Combined:C})",
                bestNonCombinableTotal, combinedTotal);
            return bestNonCombinable!;
        }

        return combinedResults;
    }
}
```

### Exclusion Rules

```csharp
public class ExclusionRuleService
{
    /// <summary>
    /// Filters out items that are excluded from promotions (e.g., sale items,
    /// specific brands, gift cards).
    /// </summary>
    public IReadOnlyList<CartItem> FilterExcludedItems(
        IReadOnlyList<CartItem> items, Promotion promotion)
    {
        var exclusions = !string.IsNullOrEmpty(promotion.CustomerSegment)
            ? JsonSerializer.Deserialize<ExclusionConfig>(promotion.CustomerSegment)
            : null;

        if (exclusions is null) return items;

        return items.Where(item =>
        {
            if (exclusions.ExcludedProductIds?.Contains(item.ProductId) == true)
                return false;
            if (exclusions.ExcludedCategories?.Contains(item.Category) == true)
                return false;
            if (exclusions.ExcludeSaleItems && item.IsOnSale)
                return false;
            if (exclusions.ExcludeGiftCards && item.IsGiftCard)
                return false;

            return true;
        }).ToList();
    }
}

public record ExclusionConfig
{
    public List<Guid>? ExcludedProductIds { get; init; }
    public List<string>? ExcludedCategories { get; init; }
    public bool ExcludeSaleItems { get; init; }
    public bool ExcludeGiftCards { get; init; }
}
```

---

## 5. Dynamic Pricing

### Time-Based Pricing

Time-based pricing adjusts product prices based on the time of day, day of the week, or season. This is common in industries like food delivery (lunch specials) and travel (weekend surcharges).

```csharp
public interface IDynamicPricingStrategy
{
    string StrategyName { get; }
    Task<decimal> CalculatePriceAsync(
        DynamicPricingContext context, CancellationToken ct = default);
}

public record DynamicPricingContext
{
    public required Guid ProductId { get; init; }
    public required decimal BasePrice { get; init; }
    public required Customer Customer { get; init; }
    public required DateTime RequestedAt { get; init; }
    public int? CurrentDemandLevel { get; init; }
    public int? InventoryLevel { get; init; }
}

public class TimeBasedPricingStrategy : IDynamicPricingStrategy
{
    public string StrategyName => "TimeBased";

    private readonly IOptionsMonitor<TimeBasedPricingOptions> _options;

    public TimeBasedPricingStrategy(IOptionsMonitor<TimeBasedPricingOptions> options)
    {
        _options = options;
    }

    public Task<decimal> CalculatePriceAsync(
        DynamicPricingContext context, CancellationToken ct = default)
    {
        var rules = _options.CurrentValue.Rules;
        var now = context.RequestedAt;

        foreach (var rule in rules.OrderByDescending(r => r.Priority))
        {
            if (rule.DaysOfWeek?.Contains(now.DayOfWeek) == true &&
                now.TimeOfDay >= rule.StartTime &&
                now.TimeOfDay <= rule.EndTime)
            {
                var adjusted = context.BasePrice * rule.PriceMultiplier;
                return Task.FromResult(Math.Round(adjusted, 2));
            }
        }

        return Task.FromResult(context.BasePrice);
    }
}

public class TimeBasedPricingOptions
{
    public List<TimePricingRule> Rules { get; set; } = new();
}

public class TimePricingRule
{
    public string Name { get; set; } = string.Empty;
    public List<DayOfWeek>? DaysOfWeek { get; set; }
    public TimeSpan StartTime { get; set; }
    public TimeSpan EndTime { get; set; }
    public decimal PriceMultiplier { get; set; } = 1.0m;
    public int Priority { get; set; }
}
```

### Demand-Based Pricing

```csharp
public class DemandBasedPricingStrategy : IDynamicPricingStrategy
{
    public string StrategyName => "DemandBased";

    private readonly IDistributedCache _cache;
    private readonly ILogger<DemandBasedPricingStrategy> _logger;

    public DemandBasedPricingStrategy(
        IDistributedCache cache,
        ILogger<DemandBasedPricingStrategy> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public async Task<decimal> CalculatePriceAsync(
        DynamicPricingContext context, CancellationToken ct = default)
    {
        var demandKey = $"demand:{context.ProductId}";
        var demandStr = await _cache.GetStringAsync(demandKey, ct);
        var demandLevel = demandStr is not null ? int.Parse(demandStr) : 50;

        // Apply pricing curve: higher demand = higher price
        decimal multiplier = demandLevel switch
        {
            >= 90 => 1.25m,  // Very high demand: +25%
            >= 75 => 1.15m,  // High demand: +15%
            >= 50 => 1.0m,   // Normal demand: base price
            >= 25 => 0.95m,  // Low demand: -5%
            _ => 0.90m       // Very low demand: -10%
        };

        // Apply floor and ceiling
        var adjusted = context.BasePrice * multiplier;
        var floor = context.BasePrice * 0.80m;
        var ceiling = context.BasePrice * 1.50m;

        adjusted = Math.Clamp(adjusted, floor, ceiling);

        _logger.LogDebug(
            "Demand pricing for {ProductId}: demand={Demand}, multiplier={Multiplier}, price={Price}",
            context.ProductId, demandLevel, multiplier, adjusted);

        return Math.Round(adjusted, 2);
    }
}
```

### Customer-Segment Pricing and A/B Testing

```csharp
public class CustomerSegmentPricingStrategy : IDynamicPricingStrategy
{
    public string StrategyName => "CustomerSegment";

    private readonly PromotionDbContext _db;

    public CustomerSegmentPricingStrategy(PromotionDbContext db)
    {
        _db = db;
    }

    public async Task<decimal> CalculatePriceAsync(
        DynamicPricingContext context, CancellationToken ct = default)
    {
        // VIP customers get preferential pricing
        var segmentDiscount = context.Customer.Segment switch
        {
            "VIP" => 0.10m,
            "Gold" => 0.07m,
            "Silver" => 0.03m,
            "Employee" => 0.25m,
            _ => 0m
        };

        var adjusted = context.BasePrice * (1 - segmentDiscount);
        return await Task.FromResult(Math.Round(adjusted, 2));
    }
}

public class AbTestPricingStrategy : IDynamicPricingStrategy
{
    public string StrategyName => "ABTest";

    private readonly IDistributedCache _cache;

    public AbTestPricingStrategy(IDistributedCache cache)
    {
        _cache = cache;
    }

    public async Task<decimal> CalculatePriceAsync(
        DynamicPricingContext context, CancellationToken ct = default)
    {
        var testKey = $"ab-test:pricing:{context.ProductId}";
        var testConfig = await _cache.GetStringAsync(testKey, ct);

        if (testConfig is null)
            return context.BasePrice;

        var config = JsonSerializer.Deserialize<AbTestConfig>(testConfig);
        if (config is null || !config.IsActive)
            return context.BasePrice;

        // Assign customer to cohort deterministically using customer ID hash
        var bucket = Math.Abs(context.Customer.Id.GetHashCode()) % 100;
        var isTestGroup = bucket < config.TestGroupPercentage;

        var price = isTestGroup ? config.TestPrice : context.BasePrice;

        // Track assignment for analytics
        var assignmentKey = $"ab-assignment:{config.TestId}:{context.Customer.Id}";
        await _cache.SetStringAsync(assignmentKey,
            isTestGroup ? "test" : "control",
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromDays(30) },
            ct);

        return Math.Round(price, 2);
    }
}

public record AbTestConfig
{
    public string TestId { get; init; } = string.Empty;
    public bool IsActive { get; init; }
    public int TestGroupPercentage { get; init; }
    public decimal TestPrice { get; init; }
}
```

### Dynamic Pricing Orchestrator

```csharp
public class DynamicPricingService
{
    private readonly IEnumerable<IDynamicPricingStrategy> _strategies;
    private readonly ILogger<DynamicPricingService> _logger;

    public DynamicPricingService(
        IEnumerable<IDynamicPricingStrategy> strategies,
        ILogger<DynamicPricingService> logger)
    {
        _strategies = strategies;
        _logger = logger;
    }

    /// <summary>
    /// Applies dynamic pricing strategies in order. Only the first matching
    /// strategy is used to avoid compounding price changes.
    /// </summary>
    public async Task<PricingResult> GetDynamicPriceAsync(
        DynamicPricingContext context,
        string strategyName,
        CancellationToken ct = default)
    {
        var strategy = _strategies.FirstOrDefault(
            s => s.StrategyName == strategyName);

        if (strategy is null)
        {
            _logger.LogWarning("Unknown pricing strategy: {Strategy}", strategyName);
            return new PricingResult(context.BasePrice, context.BasePrice, "None");
        }

        var adjustedPrice = await strategy.CalculatePriceAsync(context, ct);

        return new PricingResult(context.BasePrice, adjustedPrice, strategyName);
    }
}

public record PricingResult(
    decimal BasePrice,
    decimal FinalPrice,
    string AppliedStrategy);
```

---

## 6. Flash Sales and Limited-Time Offers

### Flash Sale Entity and Management

Flash sales require precise timing, inventory tracking, and surge protection to handle traffic spikes when a sale starts.

```csharp
public class FlashSale
{
    public Guid Id { get; set; }
    public Guid PromotionId { get; set; }
    public string Name { get; set; } = string.Empty;
    public DateTime StartsAt { get; set; }
    public DateTime EndsAt { get; set; }
    public int? MaxUnits { get; set; }
    public int UnitsSold { get; set; }
    public bool IsActive { get; set; }
    public int MaxPerCustomer { get; set; } = 1;

    public Promotion Promotion { get; set; } = null!;
}

public class FlashSaleService
{
    private readonly PromotionDbContext _db;
    private readonly IDistributedCache _cache;
    private readonly TimeProvider _timeProvider;
    private readonly ILogger<FlashSaleService> _logger;

    public FlashSaleService(
        PromotionDbContext db,
        IDistributedCache cache,
        TimeProvider timeProvider,
        ILogger<FlashSaleService> logger)
    {
        _db = db;
        _cache = cache;
        _timeProvider = timeProvider;
        _logger = logger;
    }

    /// <summary>
    /// Attempts to reserve a unit in a flash sale using distributed locking
    /// to prevent overselling.
    /// </summary>
    public async Task<FlashSaleReservation> TryReserveAsync(
        Guid flashSaleId, Guid customerId, int quantity, CancellationToken ct = default)
    {
        var sale = await _db.Set<FlashSale>()
            .FirstOrDefaultAsync(s => s.Id == flashSaleId, ct);

        if (sale is null)
            return FlashSaleReservation.Failed("Flash sale not found.");

        var now = _timeProvider.GetUtcNow().UtcDateTime;

        if (now < sale.StartsAt)
            return FlashSaleReservation.Failed("This sale hasn't started yet.");

        if (now >= sale.EndsAt || !sale.IsActive)
            return FlashSaleReservation.Failed("This sale has ended.");

        // Check per-customer limit using cache
        var customerKey = $"flash:{flashSaleId}:customer:{customerId}";
        var customerPurchasesStr = await _cache.GetStringAsync(customerKey, ct);
        var customerPurchases = customerPurchasesStr is not null
            ? int.Parse(customerPurchasesStr)
            : 0;

        if (customerPurchases + quantity > sale.MaxPerCustomer)
            return FlashSaleReservation.Failed(
                $"Maximum {sale.MaxPerCustomer} unit(s) per customer.");

        // Atomic inventory check using distributed counter
        var inventoryKey = $"flash:{flashSaleId}:remaining";
        var remainingStr = await _cache.GetStringAsync(inventoryKey, ct);

        if (remainingStr is null)
        {
            // Initialize cache from database
            var remaining = (sale.MaxUnits ?? int.MaxValue) - sale.UnitsSold;
            await _cache.SetStringAsync(inventoryKey, remaining.ToString(),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpiration = sale.EndsAt
                }, ct);
            remainingStr = remaining.ToString();
        }

        var currentRemaining = int.Parse(remainingStr);

        if (currentRemaining < quantity)
            return FlashSaleReservation.Failed("Sold out!");

        // Decrement inventory (in production, use Redis DECRBY for atomicity)
        await _cache.SetStringAsync(inventoryKey,
            (currentRemaining - quantity).ToString(),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpiration = sale.EndsAt
            }, ct);

        // Track customer purchases
        await _cache.SetStringAsync(customerKey,
            (customerPurchases + quantity).ToString(),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpiration = sale.EndsAt
            }, ct);

        _logger.LogInformation(
            "Flash sale {SaleId}: reserved {Qty} units for customer {CustomerId}. " +
            "Remaining: {Remaining}",
            flashSaleId, quantity, customerId, currentRemaining - quantity);

        return FlashSaleReservation.Succeeded(quantity, sale.PromotionId);
    }
}

public record FlashSaleReservation
{
    public bool Success { get; init; }
    public string? ErrorMessage { get; init; }
    public int QuantityReserved { get; init; }
    public Guid? PromotionId { get; init; }

    public static FlashSaleReservation Failed(string message) =>
        new() { Success = false, ErrorMessage = message };

    public static FlashSaleReservation Succeeded(int quantity, Guid promotionId) =>
        new() { Success = true, QuantityReserved = quantity, PromotionId = promotionId };
}
```

### Scheduled Activation and Deactivation

```csharp
public class FlashSaleScheduler : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<FlashSaleScheduler> _logger;

    public FlashSaleScheduler(
        IServiceScopeFactory scopeFactory,
        ILogger<FlashSaleScheduler> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await using var scope = _scopeFactory.CreateAsyncScope();
                var db = scope.ServiceProvider.GetRequiredService<PromotionDbContext>();
                var now = DateTime.UtcNow;

                // Activate sales that should start
                var toActivate = await db.Set<FlashSale>()
                    .Where(s => !s.IsActive && s.StartsAt <= now && s.EndsAt > now)
                    .ToListAsync(stoppingToken);

                foreach (var sale in toActivate)
                {
                    sale.IsActive = true;
                    _logger.LogInformation("Activated flash sale: {SaleId} - {Name}",
                        sale.Id, sale.Name);
                }

                // Deactivate sales that should end
                var toDeactivate = await db.Set<FlashSale>()
                    .Where(s => s.IsActive && s.EndsAt <= now)
                    .ToListAsync(stoppingToken);

                foreach (var sale in toDeactivate)
                {
                    sale.IsActive = false;
                    _logger.LogInformation("Deactivated flash sale: {SaleId} - {Name}",
                        sale.Id, sale.Name);
                }

                if (toActivate.Count > 0 || toDeactivate.Count > 0)
                    await db.SaveChangesAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in flash sale scheduler");
            }

            await Task.Delay(TimeSpan.FromSeconds(15), stoppingToken);
        }
    }
}
```

### Countdown Timer API

```csharp
[ApiController]
[Route("api/flash-sales")]
public class FlashSaleController : ControllerBase
{
    private readonly PromotionDbContext _db;
    private readonly FlashSaleService _flashSaleService;
    private readonly TimeProvider _timeProvider;

    public FlashSaleController(
        PromotionDbContext db,
        FlashSaleService flashSaleService,
        TimeProvider timeProvider)
    {
        _db = db;
        _flashSaleService = flashSaleService;
        _timeProvider = timeProvider;
    }

    [HttpGet("active")]
    public async Task<ActionResult<IEnumerable<FlashSaleDto>>> GetActiveSales(
        CancellationToken ct)
    {
        var now = _timeProvider.GetUtcNow().UtcDateTime;

        var sales = await _db.Set<FlashSale>()
            .Where(s => s.IsActive && s.EndsAt > now)
            .Select(s => new FlashSaleDto
            {
                Id = s.Id,
                Name = s.Name,
                EndsAt = s.EndsAt,
                SecondsRemaining = (int)(s.EndsAt - now).TotalSeconds,
                PercentSold = s.MaxUnits.HasValue
                    ? (int)(s.UnitsSold * 100.0 / s.MaxUnits.Value)
                    : 0,
                IsAlmostGone = s.MaxUnits.HasValue &&
                    s.UnitsSold >= s.MaxUnits.Value * 0.9
            })
            .ToListAsync(ct);

        return Ok(sales);
    }

    [HttpPost("{id}/reserve")]
    [Authorize]
    public async Task<ActionResult<FlashSaleReservation>> Reserve(
        Guid id,
        [FromBody] FlashSaleReserveRequest request,
        CancellationToken ct)
    {
        var customerId = User.GetCustomerId();
        var result = await _flashSaleService.TryReserveAsync(
            id, customerId, request.Quantity, ct);

        if (!result.Success)
            return BadRequest(result);

        return Ok(result);
    }
}

public record FlashSaleDto
{
    public Guid Id { get; init; }
    public string Name { get; init; } = string.Empty;
    public DateTime EndsAt { get; init; }
    public int SecondsRemaining { get; init; }
    public int PercentSold { get; init; }
    public bool IsAlmostGone { get; init; }
}

public record FlashSaleReserveRequest(int Quantity = 1);
```

---

## 7. Loyalty and Points Programs

### Points Earning Rules

Loyalty programs reward customers for purchases and other activities. Points can be earned at different rates depending on the product, customer tier, and promotional multipliers.

```csharp
public class LoyaltyAccount
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public int PointsBalance { get; set; }
    public int LifetimePointsEarned { get; set; }
    public int LifetimePointsRedeemed { get; set; }
    public string Tier { get; set; } = "Bronze";
    public DateTime TierExpiresAt { get; set; }
    public DateTime CreatedAt { get; set; }

    public ICollection<PointsTransaction> Transactions { get; set; } = new List<PointsTransaction>();
}

public class PointsTransaction
{
    public Guid Id { get; set; }
    public Guid LoyaltyAccountId { get; set; }
    public int Points { get; set; }
    public string Type { get; set; } = string.Empty;  // "earn", "redeem", "expire", "adjust"
    public string Description { get; set; } = string.Empty;
    public Guid? OrderId { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? ExpiresAt { get; set; }

    public LoyaltyAccount Account { get; set; } = null!;
}

public class LoyaltyPointsService
{
    private readonly PromotionDbContext _db;
    private readonly ILogger<LoyaltyPointsService> _logger;

    public LoyaltyPointsService(PromotionDbContext db, ILogger<LoyaltyPointsService> logger)
    {
        _db = db;
        _logger = logger;
    }

    /// <summary>
    /// Calculates and awards points for a completed order.
    /// </summary>
    public async Task<int> EarnPointsForOrderAsync(
        Guid customerId, Guid orderId, decimal orderTotal,
        string productCategory, CancellationToken ct = default)
    {
        var account = await GetOrCreateAccountAsync(customerId, ct);

        int basePoints = (int)Math.Floor(orderTotal);  // 1 point per dollar
        decimal multiplier = GetTierMultiplier(account.Tier);
        decimal categoryBonus = GetCategoryBonus(productCategory);

        int totalPoints = (int)Math.Ceiling(basePoints * multiplier * categoryBonus);

        var transaction = new PointsTransaction
        {
            Id = Guid.NewGuid(),
            LoyaltyAccountId = account.Id,
            Points = totalPoints,
            Type = "earn",
            Description = $"Order {orderId}: {basePoints} base × {multiplier} tier × {categoryBonus} category",
            OrderId = orderId,
            CreatedAt = DateTime.UtcNow,
            ExpiresAt = DateTime.UtcNow.AddYears(1)
        };

        account.PointsBalance += totalPoints;
        account.LifetimePointsEarned += totalPoints;

        _db.Set<PointsTransaction>().Add(transaction);
        await _db.SaveChangesAsync(ct);

        _logger.LogInformation(
            "Customer {CustomerId} earned {Points} points for order {OrderId}",
            customerId, totalPoints, orderId);

        // Check for tier upgrade
        await EvaluateTierAsync(account, ct);

        return totalPoints;
    }

    private static decimal GetTierMultiplier(string tier) => tier switch
    {
        "Platinum" => 2.0m,
        "Gold" => 1.5m,
        "Silver" => 1.25m,
        _ => 1.0m
    };

    private static decimal GetCategoryBonus(string category) => category switch
    {
        "Electronics" => 1.5m,
        "Fashion" => 2.0m,
        "Groceries" => 1.0m,
        _ => 1.0m
    };

    private async Task<LoyaltyAccount> GetOrCreateAccountAsync(
        Guid customerId, CancellationToken ct)
    {
        var account = await _db.Set<LoyaltyAccount>()
            .FirstOrDefaultAsync(a => a.CustomerId == customerId, ct);

        if (account is not null)
            return account;

        account = new LoyaltyAccount
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            PointsBalance = 0,
            Tier = "Bronze",
            TierExpiresAt = DateTime.UtcNow.AddYears(1),
            CreatedAt = DateTime.UtcNow
        };

        _db.Set<LoyaltyAccount>().Add(account);
        await _db.SaveChangesAsync(ct);

        return account;
    }

    /// <summary>
    /// Evaluates and upgrades the customer's loyalty tier based on lifetime points.
    /// </summary>
    private async Task EvaluateTierAsync(LoyaltyAccount account, CancellationToken ct)
    {
        var newTier = account.LifetimePointsEarned switch
        {
            >= 10000 => "Platinum",
            >= 5000 => "Gold",
            >= 1000 => "Silver",
            _ => "Bronze"
        };

        if (newTier != account.Tier)
        {
            _logger.LogInformation(
                "Customer {CustomerId} upgraded from {OldTier} to {NewTier}",
                account.CustomerId, account.Tier, newTier);

            account.Tier = newTier;
            account.TierExpiresAt = DateTime.UtcNow.AddYears(1);
            await _db.SaveChangesAsync(ct);
        }
    }
}
```

### Points Redemption and Expiration

```csharp
public class PointsRedemptionService
{
    private readonly PromotionDbContext _db;
    private readonly ILogger<PointsRedemptionService> _logger;

    // Points-to-currency conversion: 100 points = $1
    private const decimal PointsToCurrencyRate = 0.01m;

    public PointsRedemptionService(
        PromotionDbContext db,
        ILogger<PointsRedemptionService> logger)
    {
        _db = db;
        _logger = logger;
    }

    /// <summary>
    /// Redeems loyalty points as a discount on an order.
    /// </summary>
    public async Task<PointsRedemptionResult> RedeemPointsAsync(
        Guid customerId, int pointsToRedeem, Guid orderId,
        decimal orderTotal, CancellationToken ct = default)
    {
        var account = await _db.Set<LoyaltyAccount>()
            .FirstOrDefaultAsync(a => a.CustomerId == customerId, ct);

        if (account is null)
            return new PointsRedemptionResult(false, "No loyalty account found.");

        if (account.PointsBalance < pointsToRedeem)
            return new PointsRedemptionResult(false,
                $"Insufficient points. Balance: {account.PointsBalance}");

        var discountAmount = pointsToRedeem * PointsToCurrencyRate;

        // Cannot discount more than the order total
        if (discountAmount > orderTotal)
        {
            pointsToRedeem = (int)Math.Ceiling(orderTotal / PointsToCurrencyRate);
            discountAmount = orderTotal;
        }

        account.PointsBalance -= pointsToRedeem;
        account.LifetimePointsRedeemed += pointsToRedeem;

        var transaction = new PointsTransaction
        {
            Id = Guid.NewGuid(),
            LoyaltyAccountId = account.Id,
            Points = -pointsToRedeem,
            Type = "redeem",
            Description = $"Redeemed {pointsToRedeem} points for {discountAmount:C} off order {orderId}",
            OrderId = orderId,
            CreatedAt = DateTime.UtcNow
        };

        _db.Set<PointsTransaction>().Add(transaction);
        await _db.SaveChangesAsync(ct);

        _logger.LogInformation(
            "Customer {CustomerId} redeemed {Points} points for {Discount:C}",
            customerId, pointsToRedeem, discountAmount);

        return new PointsRedemptionResult(true, DiscountAmount: discountAmount,
            PointsRedeemed: pointsToRedeem,
            RemainingBalance: account.PointsBalance);
    }

    /// <summary>
    /// Expires points that have passed their expiration date.
    /// Run as a scheduled background job.
    /// </summary>
    public async Task ExpirePointsAsync(CancellationToken ct = default)
    {
        var now = DateTime.UtcNow;

        var expiredTransactions = await _db.Set<PointsTransaction>()
            .Where(t => t.Type == "earn" && t.ExpiresAt <= now && t.Points > 0)
            .Include(t => t.Account)
            .ToListAsync(ct);

        foreach (var transaction in expiredTransactions)
        {
            transaction.Account.PointsBalance -= transaction.Points;

            _db.Set<PointsTransaction>().Add(new PointsTransaction
            {
                Id = Guid.NewGuid(),
                LoyaltyAccountId = transaction.LoyaltyAccountId,
                Points = -transaction.Points,
                Type = "expire",
                Description = $"Expired {transaction.Points} points from {transaction.CreatedAt:d}",
                CreatedAt = now
            });

            // Zero out the original transaction to prevent re-expiration
            transaction.Points = 0;
        }

        await _db.SaveChangesAsync(ct);

        _logger.LogInformation("Expired points from {Count} transactions", expiredTransactions.Count);
    }
}

public record PointsRedemptionResult(
    bool Success,
    string? ErrorMessage = null,
    decimal DiscountAmount = 0,
    int PointsRedeemed = 0,
    int RemainingBalance = 0);
```

---

## 8. Cart-Level vs. Item-Level Discounts

### Application Order

The order in which discounts are applied significantly affects the final price. A consistent discount application sequence prevents unexpected results and ensures tax compliance.

| Level | Applied To | Example |
|-------|-----------|---------|
| **Item-level** | Individual line items | 20% off specific product |
| **Cart-level** | Entire order subtotal | $10 off orders over $50 |
| **Shipping-level** | Shipping charges | Free shipping on orders over $75 |

> Discounts should be applied in a deterministic order: item-level first, then cart-level, then shipping-level. This ensures that cart-level minimum thresholds are evaluated against item-discounted subtotals.

```csharp
public enum DiscountLevel
{
    Item,
    Cart,
    Shipping
}

public class DiscountApplicationService
{
    private readonly ILogger<DiscountApplicationService> _logger;

    public DiscountApplicationService(ILogger<DiscountApplicationService> logger)
    {
        _logger = logger;
    }

    /// <summary>
    /// Applies discounts in the correct order: item → cart → shipping.
    /// Returns the final order totals with all discounts applied.
    /// </summary>
    public OrderPricing ApplyDiscounts(
        Cart cart,
        IReadOnlyList<AppliedDiscount> discounts,
        decimal shippingCost,
        decimal taxRate)
    {
        var itemDiscounts = discounts.Where(d => d.Level == DiscountLevel.Item).ToList();
        var cartDiscounts = discounts.Where(d => d.Level == DiscountLevel.Cart).ToList();
        var shippingDiscounts = discounts.Where(d => d.Level == DiscountLevel.Shipping).ToList();

        // Step 1: Apply item-level discounts
        var lineItems = cart.Items.Select(item =>
        {
            var itemTotal = item.Price * item.Quantity;
            var discount = itemDiscounts
                .Where(d => d.AffectedItemIds.Contains(item.Id))
                .Sum(d => DistributeDiscount(d, item, cart.Items));

            return new PricedLineItem
            {
                ItemId = item.Id,
                OriginalTotal = itemTotal,
                DiscountAmount = Math.Min(discount, itemTotal),
                FinalTotal = Math.Max(0, itemTotal - discount)
            };
        }).ToList();

        var subtotalAfterItemDiscounts = lineItems.Sum(l => l.FinalTotal);

        // Step 2: Apply cart-level discounts
        var cartDiscountTotal = cartDiscounts.Sum(d => d.Amount);
        cartDiscountTotal = Math.Min(cartDiscountTotal, subtotalAfterItemDiscounts);
        var subtotalAfterAllDiscounts = subtotalAfterItemDiscounts - cartDiscountTotal;

        // Step 3: Calculate tax on discounted subtotal
        var taxAmount = Math.Round(subtotalAfterAllDiscounts * taxRate, 2);

        // Step 4: Apply shipping discounts
        var shippingDiscountTotal = shippingDiscounts.Sum(d => d.Amount);
        var finalShipping = Math.Max(0, shippingCost - shippingDiscountTotal);

        var orderTotal = subtotalAfterAllDiscounts + taxAmount + finalShipping;

        _logger.LogInformation(
            "Order pricing: subtotal={Subtotal:C}, itemDisc={ItemDisc:C}, " +
            "cartDisc={CartDisc:C}, tax={Tax:C}, shipping={Ship:C}, total={Total:C}",
            cart.Items.Sum(i => i.Price * i.Quantity),
            lineItems.Sum(l => l.DiscountAmount), cartDiscountTotal,
            taxAmount, finalShipping, orderTotal);

        return new OrderPricing
        {
            LineItems = lineItems,
            SubtotalBeforeDiscounts = cart.Items.Sum(i => i.Price * i.Quantity),
            ItemDiscountTotal = lineItems.Sum(l => l.DiscountAmount),
            CartDiscountTotal = cartDiscountTotal,
            SubtotalAfterDiscounts = subtotalAfterAllDiscounts,
            TaxAmount = taxAmount,
            ShippingBeforeDiscount = shippingCost,
            ShippingDiscount = shippingDiscountTotal,
            ShippingAfterDiscount = finalShipping,
            OrderTotal = orderTotal
        };
    }

    /// <summary>
    /// Distributes a cart-level discount proportionally across line items
    /// for accurate per-item accounting (needed for partial refunds).
    /// </summary>
    private static decimal DistributeDiscount(
        AppliedDiscount discount, CartItem item, IEnumerable<CartItem> allItems)
    {
        if (discount.Level != DiscountLevel.Item)
            return 0;

        var affectedItems = allItems
            .Where(i => discount.AffectedItemIds.Contains(i.Id))
            .ToList();

        var totalAffected = affectedItems.Sum(i => i.Price * i.Quantity);
        if (totalAffected == 0) return 0;

        var itemTotal = item.Price * item.Quantity;
        var proportion = itemTotal / totalAffected;

        return Math.Round(discount.Amount * proportion, 2);
    }
}

public class AppliedDiscount
{
    public Guid PromotionId { get; init; }
    public DiscountLevel Level { get; init; }
    public decimal Amount { get; init; }
    public IReadOnlyList<Guid> AffectedItemIds { get; init; } = Array.Empty<Guid>();
}

public class PricedLineItem
{
    public Guid ItemId { get; init; }
    public decimal OriginalTotal { get; init; }
    public decimal DiscountAmount { get; init; }
    public decimal FinalTotal { get; init; }
}

public class OrderPricing
{
    public IReadOnlyList<PricedLineItem> LineItems { get; init; } = Array.Empty<PricedLineItem>();
    public decimal SubtotalBeforeDiscounts { get; init; }
    public decimal ItemDiscountTotal { get; init; }
    public decimal CartDiscountTotal { get; init; }
    public decimal SubtotalAfterDiscounts { get; init; }
    public decimal TaxAmount { get; init; }
    public decimal ShippingBeforeDiscount { get; init; }
    public decimal ShippingDiscount { get; init; }
    public decimal ShippingAfterDiscount { get; init; }
    public decimal OrderTotal { get; init; }
}
```

### Refund Impact on Discounts

```csharp
public class DiscountedRefundCalculator
{
    /// <summary>
    /// When a partial refund is issued, the discount must be proportionally
    /// reversed to prevent customers from keeping the full discount benefit
    /// while returning items.
    /// </summary>
    public RefundBreakdown CalculateRefund(
        OrderPricing originalPricing,
        IReadOnlyList<Guid> returnedItemIds)
    {
        var returnedItems = originalPricing.LineItems
            .Where(l => returnedItemIds.Contains(l.ItemId))
            .ToList();

        var keptItems = originalPricing.LineItems
            .Where(l => !returnedItemIds.Contains(l.ItemId))
            .ToList();

        // Proportional share of returned items in the total
        var returnedOriginalTotal = returnedItems.Sum(i => i.OriginalTotal);
        var fullOriginalTotal = originalPricing.SubtotalBeforeDiscounts;

        if (fullOriginalTotal == 0)
            return new RefundBreakdown();

        var returnProportion = returnedOriginalTotal / fullOriginalTotal;

        // Proportional item-level discount reversal
        var itemDiscountReversal = returnedItems.Sum(i => i.DiscountAmount);

        // Proportional cart-level discount reversal
        var cartDiscountReversal = Math.Round(
            originalPricing.CartDiscountTotal * returnProportion, 2);

        // Tax refund on the discounted returned amount
        var returnedSubtotal = returnedOriginalTotal - itemDiscountReversal - cartDiscountReversal;
        var taxRefund = Math.Round(
            returnedSubtotal * (originalPricing.TaxAmount / originalPricing.SubtotalAfterDiscounts), 2);

        return new RefundBreakdown
        {
            ReturnedItemsOriginalTotal = returnedOriginalTotal,
            ItemDiscountReversal = itemDiscountReversal,
            CartDiscountReversal = cartDiscountReversal,
            TaxRefund = taxRefund,
            TotalRefund = returnedSubtotal + taxRefund
        };
    }
}

public record RefundBreakdown
{
    public decimal ReturnedItemsOriginalTotal { get; init; }
    public decimal ItemDiscountReversal { get; init; }
    public decimal CartDiscountReversal { get; init; }
    public decimal TaxRefund { get; init; }
    public decimal TotalRefund { get; init; }
}
```

---

## 9. Promotion Analytics

### Conversion Tracking

Tracking promotion performance is essential for measuring ROI and optimizing future campaigns. The analytics system records impressions, applications, and conversions.

```csharp
public class PromotionEvent
{
    public Guid Id { get; set; }
    public Guid PromotionId { get; set; }
    public string EventType { get; set; } = string.Empty;  // "impression", "applied", "converted", "removed"
    public Guid? CustomerId { get; set; }
    public Guid? OrderId { get; set; }
    public decimal? DiscountAmount { get; set; }
    public decimal? OrderTotal { get; set; }
    public string? Metadata { get; set; }
    public DateTime OccurredAt { get; set; }
}

public class PromotionAnalyticsService
{
    private readonly PromotionDbContext _db;
    private readonly ILogger<PromotionAnalyticsService> _logger;

    public PromotionAnalyticsService(
        PromotionDbContext db,
        ILogger<PromotionAnalyticsService> logger)
    {
        _db = db;
        _logger = logger;
    }

    public async Task TrackEventAsync(
        Guid promotionId, string eventType, Guid? customerId = null,
        Guid? orderId = null, decimal? discountAmount = null,
        decimal? orderTotal = null, CancellationToken ct = default)
    {
        var evt = new PromotionEvent
        {
            Id = Guid.NewGuid(),
            PromotionId = promotionId,
            EventType = eventType,
            CustomerId = customerId,
            OrderId = orderId,
            DiscountAmount = discountAmount,
            OrderTotal = orderTotal,
            OccurredAt = DateTime.UtcNow
        };

        _db.Set<PromotionEvent>().Add(evt);
        await _db.SaveChangesAsync(ct);
    }

    /// <summary>
    /// Generates a performance summary for a promotion over a given time range.
    /// </summary>
    public async Task<PromotionPerformance> GetPerformanceAsync(
        Guid promotionId, DateTime from, DateTime to, CancellationToken ct = default)
    {
        var events = await _db.Set<PromotionEvent>()
            .Where(e => e.PromotionId == promotionId
                     && e.OccurredAt >= from
                     && e.OccurredAt <= to)
            .ToListAsync(ct);

        var impressions = events.Count(e => e.EventType == "impression");
        var applications = events.Count(e => e.EventType == "applied");
        var conversions = events.Count(e => e.EventType == "converted");
        var totalDiscount = events
            .Where(e => e.EventType == "converted" && e.DiscountAmount.HasValue)
            .Sum(e => e.DiscountAmount!.Value);
        var totalRevenue = events
            .Where(e => e.EventType == "converted" && e.OrderTotal.HasValue)
            .Sum(e => e.OrderTotal!.Value);

        return new PromotionPerformance
        {
            PromotionId = promotionId,
            Period = $"{from:yyyy-MM-dd} to {to:yyyy-MM-dd}",
            Impressions = impressions,
            Applications = applications,
            Conversions = conversions,
            ConversionRate = impressions > 0
                ? Math.Round((decimal)conversions / impressions * 100, 2)
                : 0,
            TotalDiscountGiven = totalDiscount,
            TotalRevenue = totalRevenue,
            AverageOrderValue = conversions > 0
                ? Math.Round(totalRevenue / conversions, 2)
                : 0,
            Roi = totalDiscount > 0
                ? Math.Round((totalRevenue - totalDiscount) / totalDiscount * 100, 2)
                : 0
        };
    }
}

public record PromotionPerformance
{
    public Guid PromotionId { get; init; }
    public string Period { get; init; } = string.Empty;
    public int Impressions { get; init; }
    public int Applications { get; init; }
    public int Conversions { get; init; }
    public decimal ConversionRate { get; init; }
    public decimal TotalDiscountGiven { get; init; }
    public decimal TotalRevenue { get; init; }
    public decimal AverageOrderValue { get; init; }
    public decimal Roi { get; init; }
}
```

### Cannibalization Analysis

```csharp
public class CannibalizationAnalyzer
{
    private readonly PromotionDbContext _db;

    public CannibalizationAnalyzer(PromotionDbContext db)
    {
        _db = db;
    }

    /// <summary>
    /// Analyzes whether a promotion is cannibalizing regular-price sales
    /// by comparing sales during the promotion period vs. a baseline period.
    /// </summary>
    public async Task<CannibalizationReport> AnalyzeAsync(
        Guid promotionId, CancellationToken ct = default)
    {
        var promotion = await _db.Promotions.FindAsync(new object[] { promotionId }, ct);
        if (promotion is null)
            throw new InvalidOperationException("Promotion not found.");

        var promoDuration = promotion.ValidTo - promotion.ValidFrom;

        // Baseline: same duration before the promotion started
        var baselineFrom = promotion.ValidFrom - promoDuration;
        var baselineTo = promotion.ValidFrom;

        // Revenue during promotion
        var promoApplications = await _db.DiscountApplications
            .Where(a => a.PromotionId == promotionId)
            .ToListAsync(ct);

        var promoRevenue = promoApplications.Sum(a => a.OrderTotalAfter);
        var promoOrders = promoApplications.Count;
        var totalDiscount = promoApplications.Sum(a => a.DiscountAmount);

        // Estimate baseline revenue from orders in the baseline period
        var baselineOrders = await _db.DiscountApplications
            .Where(a => a.AppliedAt >= baselineFrom && a.AppliedAt < baselineTo)
            .CountAsync(ct);

        var baselineRevenue = await _db.DiscountApplications
            .Where(a => a.AppliedAt >= baselineFrom && a.AppliedAt < baselineTo)
            .SumAsync(a => a.OrderTotalAfter, ct);

        var incrementalRevenue = promoRevenue - baselineRevenue;
        var cannibalizationRate = baselineRevenue > 0
            ? Math.Round((1 - incrementalRevenue / baselineRevenue) * 100, 2)
            : 0;

        return new CannibalizationReport
        {
            PromotionId = promotionId,
            BaselineRevenue = baselineRevenue,
            PromoRevenue = promoRevenue,
            IncrementalRevenue = incrementalRevenue,
            TotalDiscount = totalDiscount,
            NetImpact = incrementalRevenue - totalDiscount,
            CannibalizationRate = cannibalizationRate,
            BaselineOrders = baselineOrders,
            PromoOrders = promoOrders
        };
    }
}

public record CannibalizationReport
{
    public Guid PromotionId { get; init; }
    public decimal BaselineRevenue { get; init; }
    public decimal PromoRevenue { get; init; }
    public decimal IncrementalRevenue { get; init; }
    public decimal TotalDiscount { get; init; }
    public decimal NetImpact { get; init; }
    public decimal CannibalizationRate { get; init; }
    public int BaselineOrders { get; init; }
    public int PromoOrders { get; init; }
}
```

---

## 10. Fraud Prevention for Promotions

### Coupon Abuse Detection

Promotion fraud is a significant cost center. Common abuse patterns include creating multiple accounts to reuse single-use coupons, sharing referral codes fraudulently, and exploiting stacking rules.

```csharp
public class PromotionFraudDetector
{
    private readonly PromotionDbContext _db;
    private readonly IDistributedCache _cache;
    private readonly ILogger<PromotionFraudDetector> _logger;

    public PromotionFraudDetector(
        PromotionDbContext db,
        IDistributedCache cache,
        ILogger<PromotionFraudDetector> logger)
    {
        _db = db;
        _cache = cache;
        _logger = logger;
    }

    /// <summary>
    /// Performs a comprehensive fraud check before allowing a coupon redemption.
    /// </summary>
    public async Task<FraudCheckResult> CheckForFraudAsync(
        Guid customerId, string couponCode, string? ipAddress,
        string? deviceFingerprint, CancellationToken ct = default)
    {
        var signals = new List<FraudSignal>();

        // Check 1: Velocity — too many coupon attempts in a short period
        var velocitySignal = await CheckVelocityAsync(customerId, ct);
        if (velocitySignal is not null) signals.Add(velocitySignal);

        // Check 2: Device fingerprint — same device, different accounts
        if (deviceFingerprint is not null)
        {
            var deviceSignal = await CheckDeviceFingerprintAsync(
                customerId, deviceFingerprint, ct);
            if (deviceSignal is not null) signals.Add(deviceSignal);
        }

        // Check 3: IP address — multiple accounts from the same IP
        if (ipAddress is not null)
        {
            var ipSignal = await CheckIpAddressAsync(customerId, ipAddress, ct);
            if (ipSignal is not null) signals.Add(ipSignal);
        }

        // Check 4: Referral fraud — self-referral detection
        var referralSignal = await CheckReferralFraudAsync(
            customerId, couponCode, ct);
        if (referralSignal is not null) signals.Add(referralSignal);

        var riskScore = signals.Sum(s => s.Score);
        var decision = riskScore switch
        {
            >= 80 => FraudDecision.Block,
            >= 50 => FraudDecision.Review,
            _ => FraudDecision.Allow
        };

        if (decision != FraudDecision.Allow)
        {
            _logger.LogWarning(
                "Fraud detected for customer {CustomerId}, coupon {Code}: " +
                "score={Score}, decision={Decision}, signals=[{Signals}]",
                customerId, couponCode, riskScore, decision,
                string.Join(", ", signals.Select(s => s.Reason)));
        }

        return new FraudCheckResult
        {
            Decision = decision,
            RiskScore = riskScore,
            Signals = signals
        };
    }

    private async Task<FraudSignal?> CheckVelocityAsync(
        Guid customerId, CancellationToken ct)
    {
        var key = $"coupon-velocity:{customerId}";
        var attemptsStr = await _cache.GetStringAsync(key, ct);
        var attempts = attemptsStr is not null ? int.Parse(attemptsStr) : 0;

        // Increment and set 1-hour window
        await _cache.SetStringAsync(key, (attempts + 1).ToString(),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
            }, ct);

        if (attempts > 10)
        {
            return new FraudSignal
            {
                Reason = $"High velocity: {attempts} coupon attempts in 1 hour",
                Score = 40
            };
        }

        return null;
    }

    private async Task<FraudSignal?> CheckDeviceFingerprintAsync(
        Guid customerId, string fingerprint, CancellationToken ct)
    {
        var key = $"device-coupon:{fingerprint}";
        var existingCustomerStr = await _cache.GetStringAsync(key, ct);

        if (existingCustomerStr is not null &&
            existingCustomerStr != customerId.ToString())
        {
            return new FraudSignal
            {
                Reason = "Same device used with a different customer account",
                Score = 50
            };
        }

        await _cache.SetStringAsync(key, customerId.ToString(),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromDays(30)
            }, ct);

        return null;
    }

    private async Task<FraudSignal?> CheckIpAddressAsync(
        Guid customerId, string ipAddress, CancellationToken ct)
    {
        var key = $"ip-coupon:{ipAddress}";
        var accountsStr = await _cache.GetStringAsync(key, ct);
        var accounts = accountsStr is not null
            ? JsonSerializer.Deserialize<HashSet<string>>(accountsStr) ?? new()
            : new HashSet<string>();

        accounts.Add(customerId.ToString());

        await _cache.SetStringAsync(key,
            JsonSerializer.Serialize(accounts),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromDays(7)
            }, ct);

        if (accounts.Count > 3)
        {
            return new FraudSignal
            {
                Reason = $"IP address {ipAddress} used by {accounts.Count} accounts",
                Score = 30
            };
        }

        return null;
    }

    private async Task<FraudSignal?> CheckReferralFraudAsync(
        Guid customerId, string couponCode, CancellationToken ct)
    {
        if (!couponCode.StartsWith("REF-"))
            return null;

        var coupon = await _db.Coupons
            .FirstOrDefaultAsync(c => c.Code == couponCode, ct);

        if (coupon?.AssignedToCustomerId == customerId)
        {
            return new FraudSignal
            {
                Reason = "Self-referral detected: customer using own referral code",
                Score = 80
            };
        }

        return null;
    }
}

public record FraudSignal
{
    public string Reason { get; init; } = string.Empty;
    public int Score { get; init; }
}

public enum FraudDecision
{
    Allow,
    Review,
    Block
}

public record FraudCheckResult
{
    public FraudDecision Decision { get; init; }
    public int RiskScore { get; init; }
    public IReadOnlyList<FraudSignal> Signals { get; init; } = Array.Empty<FraudSignal>();
}
```

### Promotion Stacking Abuse Detection

```csharp
public class StackingAbuseDetector
{
    private readonly PromotionDbContext _db;
    private readonly ILogger<StackingAbuseDetector> _logger;

    public StackingAbuseDetector(
        PromotionDbContext db,
        ILogger<StackingAbuseDetector> logger)
    {
        _db = db;
        _logger = logger;
    }

    /// <summary>
    /// Detects patterns of promotion stacking abuse where a customer
    /// is systematically combining discounts to reduce prices below cost.
    /// </summary>
    public async Task<bool> IsStackingAbuseAsync(
        Guid customerId, IReadOnlyList<Guid> promotionIds,
        decimal orderTotal, decimal totalDiscount,
        CancellationToken ct = default)
    {
        // Flag 1: Discount exceeds 60% of order total
        var discountRatio = totalDiscount / orderTotal;
        if (discountRatio > 0.60m)
        {
            _logger.LogWarning(
                "Stacking abuse: customer {Id} discount ratio {Ratio:P}",
                customerId, discountRatio);
            return true;
        }

        // Flag 2: Customer has stacked 3+ promotions on a single order
        if (promotionIds.Count >= 3)
        {
            _logger.LogWarning(
                "Stacking abuse: customer {Id} stacked {Count} promotions",
                customerId, promotionIds.Count);
            return true;
        }

        // Flag 3: Customer has a history of high-discount orders
        var recentHighDiscountOrders = await _db.DiscountApplications
            .Where(a => a.CustomerId == customerId
                     && a.AppliedAt >= DateTime.UtcNow.AddDays(-30)
                     && a.DiscountAmount > a.OrderTotalBefore * 0.40m)
            .CountAsync(ct);

        if (recentHighDiscountOrders >= 5)
        {
            _logger.LogWarning(
                "Stacking abuse: customer {Id} has {Count} high-discount orders in 30 days",
                customerId, recentHighDiscountOrders);
            return true;
        }

        return false;
    }
}
```

---

## 11. Implementation Checklist

### Phase 1: Foundation (Week 1–2)

- [ ] Define and create Promotion, PromotionRule, Coupon, and DiscountApplication entities
- [ ] Configure EF Core mappings with proper indexes and precision settings
- [ ] Implement percentage and fixed-amount discount calculators
- [ ] Build basic coupon generation and single-code redemption flow
- [ ] Create promotion validity period service with background status updater
- [ ] Add input validation for coupon codes and promotion rules

### Phase 2: Rules Engine (Week 3–4)

- [ ] Implement rule evaluator interface and core evaluators (cart total, item count, customer segment)
- [ ] Build rules engine orchestrator with required/optional rule evaluation
- [ ] Implement combinability and priority-based stacking logic
- [ ] Add exclusion rules for sale items, gift cards, and specific categories
- [ ] Create BOGO, tiered pricing, and bundle discount calculators
- [ ] Implement discount calculation pipeline with ordered application

### Phase 3: Advanced Features (Week 5–6)

- [ ] Build coupon bulk generation service with pool management
- [ ] Implement referral code generation and tracking
- [ ] Add time-based and demand-based dynamic pricing strategies
- [ ] Implement customer-segment pricing and A/B price testing
- [ ] Build flash sale entity, reservation service, and scheduled activation
- [ ] Add countdown timer API and inventory-limited sale support

### Phase 4: Loyalty Program (Week 7–8)

- [ ] Create loyalty account and points transaction entities
- [ ] Implement points earning rules with tier multipliers and category bonuses
- [ ] Build points redemption flow with points-to-currency conversion
- [ ] Add points expiration background job
- [ ] Implement loyalty tier evaluation and upgrade logic
- [ ] Create loyalty dashboard API endpoints

### Phase 5: Cart Integration (Week 9–10)

- [ ] Implement cart-level vs. item-level discount application ordering
- [ ] Build discount distribution across line items for proportional accounting
- [ ] Add tax calculation on discounted subtotals
- [ ] Implement refund calculator that proportionally reverses discounts
- [ ] Create shipping-level discount application
- [ ] End-to-end integration testing of discount application pipeline

### Phase 6: Analytics and Fraud Prevention (Week 11–12)

- [ ] Implement promotion event tracking (impressions, applications, conversions)
- [ ] Build promotion performance dashboard with ROI metrics
- [ ] Add cannibalization analysis comparing promo vs. baseline periods
- [ ] Implement coupon abuse detection with velocity checks
- [ ] Add device fingerprinting and IP-based fraud signals
- [ ] Build referral fraud and self-referral detection
- [ ] Implement promotion stacking abuse detection
- [ ] Add monitoring alerts for anomalous discount patterns

### Phase 7: Production Hardening (Week 13–14)

- [ ] Add distributed caching for promotion rules and flash sale inventory
- [ ] Implement rate limiting on coupon validation endpoints
- [ ] Add audit logging for all promotion creation, modification, and deletion
- [ ] Configure health checks for promotion service dependencies
- [ ] Load test flash sale reservation flow under concurrent traffic
- [ ] Document promotion API endpoints and integration guide

---

> **Next Steps:**
>
> - Review [18-COMMON-SCENARIOS](18-COMMON-SCENARIOS.md) for end-to-end cart checkout scenarios.
> - See [06-DATABASE-DESIGN](06-DATABASE-DESIGN.md) for database schema patterns and migrations.
> - See [08-PERFORMANCE-OPTIMIZATION](08-PERFORMANCE-OPTIMIZATION.md) for caching promotion rules at scale.
> - Review [28-ADDRESS-VERIFICATION-FRAUD-PREVENTION](28-ADDRESS-VERIFICATION-FRAUD-PREVENTION.md) for complementary fraud detection strategies.
