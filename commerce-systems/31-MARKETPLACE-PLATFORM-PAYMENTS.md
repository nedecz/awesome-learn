# Marketplace and Platform Payments

## Overview

Comprehensive guide to building marketplace and platform payment systems — covering multi-party payment flows, seller onboarding, split payments, escrow patterns, platform fee management, dispute arbitration, and regulatory compliance. This document addresses the unique challenges of facilitating transactions between buyers and multiple sellers on a single platform, with .NET/C# implementation examples using domain-driven design, Stripe Connect, and PayPal for Marketplaces.

> **Related documents:**
>
> - [16-STRIPE-INTEGRATION](16-STRIPE-INTEGRATION.md) — Stripe Connect setup and API usage
> - [17-PAYPAL-INTEGRATION](17-PAYPAL-INTEGRATION.md) — PayPal for Marketplaces integration
> - [28-ADDRESS-VERIFICATION-FRAUD-PREVENTION](28-ADDRESS-VERIFICATION-FRAUD-PREVENTION.md) — KYC verification and fraud prevention
> - [18-COMMON-SCENARIOS](18-COMMON-SCENARIOS.md) — Marketplace checkout flows and common patterns

## Table of Contents

1. [Marketplace Payment Models](#1-marketplace-payment-models)
2. [Seller Onboarding & KYC](#2-seller-onboarding--kyc)
3. [Multi-Party Payouts](#3-multi-party-payouts)
4. [Escrow & Hold Patterns](#4-escrow--hold-patterns)
5. [Platform Fee Management](#5-platform-fee-management)
6. [Dispute & Chargeback Arbitration](#6-dispute--chargeback-arbitration)
7. [Seller Dashboard & Reporting](#7-seller-dashboard--reporting)
8. [Multi-Vendor Cart & Checkout](#8-multi-vendor-cart--checkout)
9. [Refund Distribution](#9-refund-distribution)
10. [Compliance & Money Transmission](#10-compliance--money-transmission)
11. [Implementation Checklist](#11-implementation-checklist)

---

## 1. Marketplace Payment Models

### Aggregator vs. Facilitator Model

A **payment aggregator** processes payments under its own merchant account and distributes funds to sellers. A **payment facilitator** (PayFac) underwrites sub-merchants and enables them to accept payments directly. The choice determines regulatory burden, onboarding speed, and liability allocation.

| Aspect | Aggregator | Payment Facilitator |
|--------|-----------|-------------------|
| **Merchant Account** | Single (platform's) | Per-seller sub-accounts |
| **Onboarding Speed** | Fast — no per-seller underwriting | Moderate — KYC per seller |
| **Regulatory Burden** | High — platform is liable | Shared — card network rules apply |
| **Chargeback Liability** | Platform | Seller (with platform backstop) |
| **Example** | Early-stage marketplace | Stripe Connect, Adyen for Platforms |

### Payment Flow Types

Marketplace payments typically follow one of three patterns. Each has trade-offs around control, complexity, and settlement timing.

```csharp
/// <summary>
/// Enumerates the supported marketplace charge models.
/// </summary>
public enum MarketplaceChargeModel
{
    /// <summary>
    /// Platform creates a charge on a connected account directly.
    /// The connected account is the merchant of record.
    /// </summary>
    DirectCharge,

    /// <summary>
    /// Platform creates a charge on its own account and transfers
    /// a portion to the connected (seller) account.
    /// </summary>
    DestinationCharge,

    /// <summary>
    /// Platform creates a charge on its own account then creates
    /// separate transfers to one or more connected accounts.
    /// </summary>
    SeparateChargeAndTransfer
}

/// <summary>
/// Selects the appropriate charge model based on marketplace configuration.
/// </summary>
public class ChargeModelSelector
{
    public MarketplaceChargeModel SelectModel(MarketplaceConfig config)
    {
        // Single-seller orders where the seller should be merchant of record
        if (config.SellerIsMerchantOfRecord && !config.HasMultiSellerCart)
            return MarketplaceChargeModel.DirectCharge;

        // Single-seller orders where the platform wants control over the charge
        if (!config.HasMultiSellerCart)
            return MarketplaceChargeModel.DestinationCharge;

        // Multi-seller carts require separate charge and transfer
        return MarketplaceChargeModel.SeparateChargeAndTransfer;
    }
}
```

### Direct Charge Flow

```csharp
public class DirectChargeService
{
    private readonly IStripeConnectClient _stripe;
    private readonly IPlatformFeeCalculator _feeCalculator;

    public DirectChargeService(
        IStripeConnectClient stripe,
        IPlatformFeeCalculator feeCalculator)
    {
        _stripe = stripe;
        _feeCalculator = feeCalculator;
    }

    /// <summary>
    /// Creates a charge directly on the seller's connected account.
    /// The platform collects an application fee.
    /// </summary>
    public async Task<MarketplacePaymentResult> ChargeDirectAsync(
        DirectChargeRequest request,
        CancellationToken ct = default)
    {
        var platformFee = _feeCalculator.Calculate(
            request.Amount, request.SellerId, request.CategoryId);

        var chargeOptions = new ChargeCreateOptions
        {
            Amount = request.Amount.MinorUnits,
            Currency = request.Amount.Currency.Code,
            Source = request.PaymentSourceToken,
            ApplicationFeeAmount = platformFee.MinorUnits,
            Metadata = new Dictionary<string, string>
            {
                ["order_id"] = request.OrderId.ToString(),
                ["platform_order_ref"] = request.PlatformOrderRef
            }
        };

        var charge = await _stripe.CreateChargeOnConnectedAccountAsync(
            request.SellerStripeAccountId, chargeOptions, ct);

        return new MarketplacePaymentResult
        {
            ChargeId = charge.Id,
            Model = MarketplaceChargeModel.DirectCharge,
            SellerAccountId = request.SellerStripeAccountId,
            GrossAmount = request.Amount,
            PlatformFee = platformFee,
            SellerNetAmount = request.Amount - platformFee
        };
    }
}
```

### Destination Charge Flow

```csharp
public class DestinationChargeService
{
    private readonly IStripeConnectClient _stripe;
    private readonly IPlatformFeeCalculator _feeCalculator;

    public DestinationChargeService(
        IStripeConnectClient stripe,
        IPlatformFeeCalculator feeCalculator)
    {
        _stripe = stripe;
        _feeCalculator = feeCalculator;
    }

    /// <summary>
    /// Creates a charge on the platform account with automatic transfer
    /// to the seller's connected account.
    /// </summary>
    public async Task<MarketplacePaymentResult> ChargeWithDestinationAsync(
        DestinationChargeRequest request,
        CancellationToken ct = default)
    {
        var platformFee = _feeCalculator.Calculate(
            request.Amount, request.SellerId, request.CategoryId);

        var sellerAmount = request.Amount - platformFee;

        var options = new PaymentIntentCreateOptions
        {
            Amount = request.Amount.MinorUnits,
            Currency = request.Amount.Currency.Code,
            PaymentMethodTypes = new List<string> { "card" },
            TransferData = new PaymentIntentTransferDataOptions
            {
                Destination = request.SellerStripeAccountId,
                Amount = sellerAmount.MinorUnits
            },
            Metadata = new Dictionary<string, string>
            {
                ["order_id"] = request.OrderId.ToString(),
                ["seller_id"] = request.SellerId.ToString()
            }
        };

        var paymentIntent = await _stripe.CreatePaymentIntentAsync(options, ct);

        return new MarketplacePaymentResult
        {
            PaymentIntentId = paymentIntent.Id,
            Model = MarketplaceChargeModel.DestinationCharge,
            SellerAccountId = request.SellerStripeAccountId,
            GrossAmount = request.Amount,
            PlatformFee = platformFee,
            SellerNetAmount = sellerAmount
        };
    }
}
```

### Separate Charge and Transfer (Multi-Seller)

```csharp
public class SeparateChargeAndTransferService
{
    private readonly IStripeConnectClient _stripe;
    private readonly IPlatformFeeCalculator _feeCalculator;
    private readonly IUnitOfWork _unitOfWork;

    public SeparateChargeAndTransferService(
        IStripeConnectClient stripe,
        IPlatformFeeCalculator feeCalculator,
        IUnitOfWork unitOfWork)
    {
        _stripe = stripe;
        _feeCalculator = feeCalculator;
        _unitOfWork = unitOfWork;
    }

    /// <summary>
    /// Charges the buyer once on the platform account, then creates
    /// individual transfers to each seller's connected account.
    /// </summary>
    public async Task<MultiSellerPaymentResult> ChargeAndTransferAsync(
        MultiSellerChargeRequest request,
        CancellationToken ct = default)
    {
        // Step 1: Create a single charge on the platform account
        var totalAmount = request.SellerLineItems.Sum(s => s.Amount.MinorUnits);

        var paymentIntent = await _stripe.CreatePaymentIntentAsync(
            new PaymentIntentCreateOptions
            {
                Amount = totalAmount,
                Currency = request.Currency.Code,
                PaymentMethodTypes = new List<string> { "card" },
                TransferGroup = request.OrderGroupId,
                Metadata = new Dictionary<string, string>
                {
                    ["order_group_id"] = request.OrderGroupId
                }
            }, ct);

        // Step 2: Create transfers to each seller
        var transfers = new List<SellerTransferResult>();

        foreach (var sellerItem in request.SellerLineItems)
        {
            var fee = _feeCalculator.Calculate(
                sellerItem.Amount, sellerItem.SellerId, sellerItem.CategoryId);
            var sellerNet = sellerItem.Amount - fee;

            var transfer = await _stripe.CreateTransferAsync(
                new TransferCreateOptions
                {
                    Amount = sellerNet.MinorUnits,
                    Currency = request.Currency.Code,
                    Destination = sellerItem.SellerStripeAccountId,
                    TransferGroup = request.OrderGroupId,
                    Metadata = new Dictionary<string, string>
                    {
                        ["seller_id"] = sellerItem.SellerId.ToString(),
                        ["sub_order_id"] = sellerItem.SubOrderId.ToString()
                    }
                }, ct);

            transfers.Add(new SellerTransferResult
            {
                SellerId = sellerItem.SellerId,
                TransferId = transfer.Id,
                GrossAmount = sellerItem.Amount,
                PlatformFee = fee,
                SellerNetAmount = sellerNet
            });
        }

        // Persist the payment record
        await _unitOfWork.MarketplacePayments.AddAsync(
            MarketplacePayment.Create(
                request.OrderGroupId, paymentIntent.Id, transfers), ct);
        await _unitOfWork.SaveChangesAsync(ct);

        return new MultiSellerPaymentResult
        {
            PaymentIntentId = paymentIntent.Id,
            TransferGroup = request.OrderGroupId,
            Transfers = transfers
        };
    }
}
```

### Commission Structure Models

```csharp
public record CommissionStructure
{
    public Guid SellerId { get; init; }
    public CommissionType Type { get; init; }
    public decimal PercentageRate { get; init; }
    public Money FixedFee { get; init; }
    public Money MinimumFee { get; init; }
    public Money MaximumFee { get; init; }
}

public enum CommissionType
{
    FlatPercentage,
    FixedPerTransaction,
    PercentagePlusFixed,
    TieredByVolume,
    CategoryBased
}
```

---

## 2. Seller Onboarding & KYC

### Seller Verification Workflow

Marketplace platforms must verify seller identity and business information before enabling payment processing. This involves collecting identity documents, performing KYC (Know Your Customer) and KYB (Know Your Business) checks, and managing progressive onboarding to reduce friction.

```csharp
public enum SellerOnboardingStatus
{
    Started,
    BasicInfoCollected,
    IdentitySubmitted,
    IdentityVerified,
    BankAccountLinked,
    TaxInfoCollected,
    UnderReview,
    Approved,
    Restricted,
    Rejected,
    Suspended
}

public class SellerOnboardingState
{
    public Guid SellerId { get; private set; }
    public SellerOnboardingStatus Status { get; private set; }
    public SellerType SellerType { get; private set; }
    public bool IdentityVerified { get; private set; }
    public bool BankAccountLinked { get; private set; }
    public bool TaxInfoProvided { get; private set; }
    public Money? PayoutThreshold { get; private set; }
    public List<string> PendingRequirements { get; private set; } = new();
    public DateTimeOffset CreatedAt { get; private set; }
    public DateTimeOffset? ApprovedAt { get; private set; }

    public bool CanAcceptPayments =>
        Status == SellerOnboardingStatus.Approved && IdentityVerified;

    public bool CanReceivePayouts =>
        CanAcceptPayments && BankAccountLinked && TaxInfoProvided;
}

public enum SellerType
{
    Individual,
    SoleProprietor,
    LLC,
    Corporation,
    Partnership,
    NonProfit
}
```

### Identity Document Collection

```csharp
public interface ISellerVerificationService
{
    Task<VerificationResult> VerifyIdentityAsync(
        Guid sellerId, IdentityVerificationRequest request, CancellationToken ct);
    Task<VerificationResult> VerifyBusinessAsync(
        Guid sellerId, BusinessVerificationRequest request, CancellationToken ct);
    Task<OnboardingRequirements> GetPendingRequirementsAsync(
        Guid sellerId, CancellationToken ct);
}

public class IdentityVerificationRequest
{
    public string FirstName { get; set; } = null!;
    public string LastName { get; set; } = null!;
    public DateOnly DateOfBirth { get; set; }
    public string SsnLast4 { get; set; } = null!;
    public Address LegalAddress { get; set; } = null!;
    public DocumentUpload? GovernmentIdFront { get; set; }
    public DocumentUpload? GovernmentIdBack { get; set; }
}

public class BusinessVerificationRequest
{
    public string LegalBusinessName { get; set; } = null!;
    public string TaxId { get; set; } = null!;
    public SellerType BusinessType { get; set; }
    public Address BusinessAddress { get; set; } = null!;
    public string? BusinessUrl { get; set; }
    public string IndustryCode { get; set; } = null!;
    public DocumentUpload? ArticlesOfIncorporation { get; set; }
}

public record DocumentUpload(
    string FileName,
    string ContentType,
    byte[] Content,
    DocumentPurpose Purpose);

public enum DocumentPurpose
{
    GovernmentIdFront,
    GovernmentIdBack,
    ProofOfAddress,
    ArticlesOfIncorporation,
    BankStatement,
    TaxReturn
}
```

### Stripe Connect Onboarding

```csharp
public class StripeConnectOnboardingService : ISellerOnboardingProvider
{
    private readonly IStripeConnectClient _stripe;
    private readonly ISellerRepository _sellerRepo;
    private readonly ILogger<StripeConnectOnboardingService> _logger;

    public StripeConnectOnboardingService(
        IStripeConnectClient stripe,
        ISellerRepository sellerRepo,
        ILogger<StripeConnectOnboardingService> logger)
    {
        _stripe = stripe;
        _sellerRepo = sellerRepo;
        _logger = logger;
    }

    /// <summary>
    /// Creates a Stripe Connect account for a new seller and returns
    /// an onboarding link for hosted identity verification.
    /// </summary>
    public async Task<OnboardingSession> StartOnboardingAsync(
        Guid sellerId, SellerOnboardingRequest request, CancellationToken ct)
    {
        var accountOptions = new AccountCreateOptions
        {
            Type = "express",
            Country = request.Country,
            Email = request.Email,
            BusinessType = MapBusinessType(request.SellerType),
            Capabilities = new AccountCapabilitiesOptions
            {
                CardPayments = new AccountCapabilitiesCardPaymentsOptions
                {
                    Requested = true
                },
                Transfers = new AccountCapabilitiesTransfersOptions
                {
                    Requested = true
                }
            },
            Metadata = new Dictionary<string, string>
            {
                ["platform_seller_id"] = sellerId.ToString()
            }
        };

        var account = await _stripe.CreateAccountAsync(accountOptions, ct);

        // Create an Account Link for hosted onboarding
        var linkOptions = new AccountLinkCreateOptions
        {
            Account = account.Id,
            RefreshUrl = $"{request.BaseUrl}/sellers/onboarding/refresh",
            ReturnUrl = $"{request.BaseUrl}/sellers/onboarding/complete",
            Type = "account_onboarding"
        };

        var accountLink = await _stripe.CreateAccountLinkAsync(linkOptions, ct);

        // Persist the mapping
        var seller = await _sellerRepo.GetByIdAsync(sellerId, ct);
        seller.SetStripeAccountId(account.Id);
        seller.UpdateOnboardingStatus(SellerOnboardingStatus.Started);
        await _sellerRepo.UpdateAsync(seller, ct);

        _logger.LogInformation(
            "Started Stripe Connect onboarding for seller {SellerId}, account {AccountId}",
            sellerId, account.Id);

        return new OnboardingSession
        {
            SellerId = sellerId,
            StripeAccountId = account.Id,
            OnboardingUrl = accountLink.Url,
            ExpiresAt = accountLink.ExpiresAt
        };
    }

    /// <summary>
    /// Handles the account.updated webhook to track onboarding progress.
    /// </summary>
    public async Task HandleAccountUpdatedAsync(
        string stripeAccountId, StripeAccount account, CancellationToken ct)
    {
        var seller = await _sellerRepo.GetByStripeAccountIdAsync(stripeAccountId, ct);
        if (seller is null) return;

        if (account.ChargesEnabled && account.PayoutsEnabled)
        {
            seller.UpdateOnboardingStatus(SellerOnboardingStatus.Approved);
            seller.MarkIdentityVerified();
            seller.MarkBankAccountLinked();
        }
        else if (account.Requirements?.CurrentlyDue?.Any() == true)
        {
            seller.UpdateOnboardingStatus(SellerOnboardingStatus.Restricted);
            seller.SetPendingRequirements(
                account.Requirements.CurrentlyDue.ToList());
        }

        await _sellerRepo.UpdateAsync(seller, ct);
    }

    private static string MapBusinessType(SellerType sellerType) =>
        sellerType switch
        {
            SellerType.Individual => "individual",
            SellerType.SoleProprietor => "individual",
            SellerType.LLC => "company",
            SellerType.Corporation => "company",
            SellerType.Partnership => "company",
            SellerType.NonProfit => "non_profit",
            _ => "individual"
        };
}
```

### Progressive Onboarding

Progressive onboarding reduces friction by collecting only the minimum information needed for each stage, requesting more as the seller's volume grows.

```csharp
public class ProgressiveOnboardingService
{
    private readonly ISellerRepository _sellerRepo;
    private readonly ISellerVerificationService _verification;

    public ProgressiveOnboardingService(
        ISellerRepository sellerRepo,
        ISellerVerificationService verification)
    {
        _sellerRepo = sellerRepo;
        _verification = verification;
    }

    /// <summary>
    /// Determines onboarding requirements based on seller tier.
    /// Lower tiers require less documentation but have lower payout limits.
    /// </summary>
    public async Task<OnboardingRequirements> GetRequirementsAsync(
        Guid sellerId, CancellationToken ct)
    {
        var seller = await _sellerRepo.GetByIdAsync(sellerId, ct);
        var totalVolume = await _sellerRepo.GetLifetimeVolumeAsync(sellerId, ct);

        return totalVolume.Amount switch
        {
            < 500_00 => new OnboardingRequirements
            {
                Tier = OnboardingTier.Starter,
                RequiredFields = new[] { "email", "name", "dob", "ssn_last4" },
                PayoutLimit = Money.USD(500_00),
                PayoutFrequency = PayoutFrequency.Weekly
            },
            < 10_000_00 => new OnboardingRequirements
            {
                Tier = OnboardingTier.Growing,
                RequiredFields = new[]
                {
                    "email", "name", "dob", "full_ssn",
                    "address", "bank_account"
                },
                PayoutLimit = Money.USD(10_000_00),
                PayoutFrequency = PayoutFrequency.Daily
            },
            _ => new OnboardingRequirements
            {
                Tier = OnboardingTier.Established,
                RequiredFields = new[]
                {
                    "email", "name", "dob", "full_ssn",
                    "address", "bank_account", "government_id",
                    "tax_info"
                },
                PayoutLimit = null,
                PayoutFrequency = PayoutFrequency.Daily
            }
        };
    }
}

public enum OnboardingTier
{
    Starter,
    Growing,
    Established,
    Enterprise
}

public enum PayoutFrequency
{
    Daily,
    Weekly,
    Biweekly,
    Monthly,
    Manual
}
```

---

## 3. Multi-Party Payouts

### Split Payment Engine

Splitting payments across multiple parties — sellers, the platform, affiliates — requires precise calculation and idempotent transfer execution.

```csharp
public interface IPayoutService
{
    Task<PayoutResult> ExecutePayoutAsync(
        Guid sellerId, PayoutRequest request, CancellationToken ct);
    Task<BatchPayoutResult> ExecuteBatchPayoutsAsync(
        IReadOnlyList<PayoutRequest> requests, CancellationToken ct);
}

public class PaymentSplitEngine
{
    private readonly IPlatformFeeCalculator _feeCalculator;

    public PaymentSplitEngine(IPlatformFeeCalculator feeCalculator)
    {
        _feeCalculator = feeCalculator;
    }

    /// <summary>
    /// Calculates the split for a marketplace transaction across all parties.
    /// </summary>
    public PaymentSplit CalculateSplit(
        Money orderTotal,
        Guid sellerId,
        Guid? affiliateId,
        string categoryId)
    {
        var platformFee = _feeCalculator.Calculate(orderTotal, sellerId, categoryId);

        var affiliateFee = affiliateId.HasValue
            ? Money.Of(orderTotal.MinorUnits * 5 / 100, orderTotal.Currency)
            : Money.Zero(orderTotal.Currency);

        var stripeFee = Money.Of(
            (long)(orderTotal.MinorUnits * 0.029m + 30), orderTotal.Currency);

        var sellerNet = orderTotal - platformFee - affiliateFee;

        return new PaymentSplit
        {
            OrderTotal = orderTotal,
            SellerNet = sellerNet,
            PlatformFee = platformFee,
            AffiliateFee = affiliateFee,
            ProcessingFee = stripeFee,
            Parties = new List<SplitParty>
            {
                new(PartyRole.Seller, sellerId, sellerNet),
                new(PartyRole.Platform, Guid.Empty, platformFee),
                affiliateId.HasValue
                    ? new(PartyRole.Affiliate, affiliateId.Value, affiliateFee)
                    : null
            }.Where(p => p is not null).ToList()!
        };
    }
}

public record PaymentSplit
{
    public Money OrderTotal { get; init; }
    public Money SellerNet { get; init; }
    public Money PlatformFee { get; init; }
    public Money AffiliateFee { get; init; }
    public Money ProcessingFee { get; init; }
    public List<SplitParty> Parties { get; init; } = new();
}

public record SplitParty(PartyRole Role, Guid PartyId, Money Amount);

public enum PartyRole { Seller, Platform, Affiliate }
```

### Payout Scheduling & Batching

```csharp
public class PayoutScheduler : IHostedService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<PayoutScheduler> _logger;
    private Timer? _timer;

    public PayoutScheduler(
        IServiceScopeFactory scopeFactory,
        ILogger<PayoutScheduler> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    public Task StartAsync(CancellationToken ct)
    {
        _timer = new Timer(ProcessPayouts, null, TimeSpan.Zero, TimeSpan.FromHours(1));
        return Task.CompletedTask;
    }

    private async void ProcessPayouts(object? state)
    {
        using var scope = _scopeFactory.CreateScope();
        var payoutService = scope.ServiceProvider.GetRequiredService<IPayoutService>();
        var sellerRepo = scope.ServiceProvider.GetRequiredService<ISellerRepository>();

        var eligibleSellers = await sellerRepo
            .GetSellersEligibleForPayoutAsync(CancellationToken.None);

        foreach (var seller in eligibleSellers)
        {
            try
            {
                var balance = await sellerRepo.GetSellerBalanceAsync(
                    seller.Id, CancellationToken.None);

                if (balance < seller.MinimumPayoutThreshold)
                {
                    _logger.LogDebug(
                        "Seller {SellerId} balance {Balance} below threshold {Threshold}",
                        seller.Id, balance, seller.MinimumPayoutThreshold);
                    continue;
                }

                var result = await payoutService.ExecutePayoutAsync(
                    seller.Id,
                    new PayoutRequest
                    {
                        SellerId = seller.Id,
                        Amount = balance,
                        Currency = seller.PayoutCurrency,
                        DestinationBankAccountId = seller.DefaultBankAccountId
                    },
                    CancellationToken.None);

                _logger.LogInformation(
                    "Payout {PayoutId} of {Amount} sent to seller {SellerId}",
                    result.PayoutId, balance, seller.Id);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex,
                    "Failed to process payout for seller {SellerId}", seller.Id);
            }
        }
    }

    public Task StopAsync(CancellationToken ct)
    {
        _timer?.Change(Timeout.Infinite, 0);
        return Task.CompletedTask;
    }
}
```

### Payout Currency Conversion

```csharp
public class CrossCurrencyPayoutService
{
    private readonly ICurrencyExchangeService _exchange;
    private readonly IPayoutService _payoutService;

    public CrossCurrencyPayoutService(
        ICurrencyExchangeService exchange,
        IPayoutService payoutService)
    {
        _exchange = exchange;
        _payoutService = payoutService;
    }

    /// <summary>
    /// Converts the seller's platform balance to their local payout currency
    /// and initiates the payout.
    /// </summary>
    public async Task<PayoutResult> PayoutInLocalCurrencyAsync(
        Guid sellerId,
        Money platformBalance,
        Currency targetCurrency,
        CancellationToken ct)
    {
        if (platformBalance.Currency == targetCurrency)
        {
            return await _payoutService.ExecutePayoutAsync(
                sellerId,
                new PayoutRequest { SellerId = sellerId, Amount = platformBalance },
                ct);
        }

        var exchangeQuote = await _exchange.GetQuoteAsync(
            platformBalance.Currency, targetCurrency, platformBalance, ct);

        var convertedAmount = exchangeQuote.ConvertedAmount;

        var result = await _payoutService.ExecutePayoutAsync(
            sellerId,
            new PayoutRequest
            {
                SellerId = sellerId,
                Amount = convertedAmount,
                Currency = targetCurrency,
                ExchangeRateId = exchangeQuote.QuoteId,
                OriginalAmount = platformBalance
            },
            ct);

        return result;
    }
}
```

---

## 4. Escrow & Hold Patterns

### Payment Hold Service

Holding funds in escrow protects both buyer and seller. Funds are captured but not released to the seller until fulfillment conditions are met.

```csharp
public enum EscrowStatus
{
    Pending,
    Funded,
    PartiallyReleased,
    Released,
    Refunded,
    Disputed,
    Expired
}

public class EscrowAccount
{
    public Guid Id { get; private set; }
    public Guid OrderId { get; private set; }
    public Guid BuyerId { get; private set; }
    public Guid SellerId { get; private set; }
    public Money HeldAmount { get; private set; }
    public Money ReleasedAmount { get; private set; }
    public EscrowStatus Status { get; private set; }
    public DateTimeOffset CreatedAt { get; private set; }
    public DateTimeOffset? ReleasedAt { get; private set; }
    public DateTimeOffset ExpiresAt { get; private set; }
    public TimeSpan HoldDuration { get; private set; }

    public Money RemainingBalance => HeldAmount - ReleasedAmount;

    public static EscrowAccount Create(
        Guid orderId, Guid buyerId, Guid sellerId,
        Money amount, TimeSpan holdDuration)
    {
        return new EscrowAccount
        {
            Id = Guid.NewGuid(),
            OrderId = orderId,
            BuyerId = buyerId,
            SellerId = sellerId,
            HeldAmount = amount,
            ReleasedAmount = Money.Zero(amount.Currency),
            Status = EscrowStatus.Pending,
            CreatedAt = DateTimeOffset.UtcNow,
            ExpiresAt = DateTimeOffset.UtcNow.Add(holdDuration),
            HoldDuration = holdDuration
        };
    }

    public void Fund()
    {
        if (Status != EscrowStatus.Pending)
            throw new InvalidOperationException(
                $"Cannot fund escrow in status {Status}");
        Status = EscrowStatus.Funded;
    }

    public void Release(Money amount)
    {
        if (Status is not (EscrowStatus.Funded or EscrowStatus.PartiallyReleased))
            throw new InvalidOperationException(
                $"Cannot release from escrow in status {Status}");

        if (amount > RemainingBalance)
            throw new InvalidOperationException(
                "Release amount exceeds remaining balance");

        ReleasedAmount += amount;
        Status = ReleasedAmount >= HeldAmount
            ? EscrowStatus.Released
            : EscrowStatus.PartiallyReleased;
        ReleasedAt = DateTimeOffset.UtcNow;
    }
}
```

### Escrow Management Service

```csharp
public class EscrowService
{
    private readonly IEscrowRepository _escrowRepo;
    private readonly IStripeConnectClient _stripe;
    private readonly ILogger<EscrowService> _logger;

    public EscrowService(
        IEscrowRepository escrowRepo,
        IStripeConnectClient stripe,
        ILogger<EscrowService> logger)
    {
        _escrowRepo = escrowRepo;
        _stripe = stripe;
        _logger = logger;
    }

    /// <summary>
    /// Creates an escrow hold by capturing the payment but delaying
    /// the transfer to the seller.
    /// </summary>
    public async Task<EscrowAccount> CreateEscrowAsync(
        EscrowRequest request, CancellationToken ct)
    {
        var holdDuration = GetHoldDuration(request.CategoryId);

        var escrow = EscrowAccount.Create(
            request.OrderId, request.BuyerId, request.SellerId,
            request.Amount, holdDuration);

        // Capture the payment on the platform account
        var paymentIntent = await _stripe.CreatePaymentIntentAsync(
            new PaymentIntentCreateOptions
            {
                Amount = request.Amount.MinorUnits,
                Currency = request.Amount.Currency.Code,
                CaptureMethod = "manual",
                PaymentMethodTypes = new List<string> { "card" },
                Metadata = new Dictionary<string, string>
                {
                    ["escrow_id"] = escrow.Id.ToString(),
                    ["order_id"] = request.OrderId.ToString()
                }
            }, ct);

        await _stripe.CapturePaymentIntentAsync(paymentIntent.Id, ct);
        escrow.Fund();

        await _escrowRepo.AddAsync(escrow, ct);

        _logger.LogInformation(
            "Created escrow {EscrowId} for order {OrderId}, hold duration {Duration}",
            escrow.Id, request.OrderId, holdDuration);

        return escrow;
    }

    /// <summary>
    /// Releases escrow funds to the seller after fulfillment conditions are met.
    /// </summary>
    public async Task ReleaseEscrowAsync(
        Guid escrowId, EscrowReleaseTrigger trigger, CancellationToken ct)
    {
        var escrow = await _escrowRepo.GetByIdAsync(escrowId, ct)
            ?? throw new EntityNotFoundException(nameof(EscrowAccount), escrowId);

        var releaseAmount = trigger switch
        {
            EscrowReleaseTrigger.DeliveryConfirmed => escrow.RemainingBalance,
            EscrowReleaseTrigger.BuyerApproved => escrow.RemainingBalance,
            EscrowReleaseTrigger.HoldExpired => escrow.RemainingBalance,
            EscrowReleaseTrigger.PartialRelease p => p.Amount,
            _ => throw new ArgumentException($"Unknown trigger: {trigger}")
        };

        escrow.Release(releaseAmount);

        // Transfer funds to the seller's connected account
        await _stripe.CreateTransferAsync(new TransferCreateOptions
        {
            Amount = releaseAmount.MinorUnits,
            Currency = releaseAmount.Currency.Code,
            Destination = escrow.SellerId.ToString(),
            Metadata = new Dictionary<string, string>
            {
                ["escrow_id"] = escrowId.ToString(),
                ["trigger"] = trigger.GetType().Name
            }
        }, ct);

        await _escrowRepo.UpdateAsync(escrow, ct);
    }

    /// <summary>
    /// Returns configurable hold durations by product category.
    /// </summary>
    private static TimeSpan GetHoldDuration(string categoryId) =>
        categoryId switch
        {
            "digital_goods" => TimeSpan.FromHours(24),
            "physical_goods" => TimeSpan.FromDays(7),
            "services" => TimeSpan.FromDays(14),
            "high_value" => TimeSpan.FromDays(30),
            _ => TimeSpan.FromDays(7)
        };
}

public abstract record EscrowReleaseTrigger
{
    public static readonly EscrowReleaseTrigger DeliveryConfirmed = new DeliveryConfirmedTrigger();
    public static readonly EscrowReleaseTrigger BuyerApproved = new BuyerApprovedTrigger();
    public static readonly EscrowReleaseTrigger HoldExpired = new HoldExpiredTrigger();
    public static EscrowReleaseTrigger PartialRelease(Money amount) => new PartialReleaseTrigger(amount);

    private sealed record DeliveryConfirmedTrigger : EscrowReleaseTrigger;
    private sealed record BuyerApprovedTrigger : EscrowReleaseTrigger;
    private sealed record HoldExpiredTrigger : EscrowReleaseTrigger;
    public sealed record PartialReleaseTrigger(Money Amount) : EscrowReleaseTrigger;
}
```

---

## 5. Platform Fee Management

### Fee Calculation Engine

Platform fees can be calculated using multiple strategies depending on seller tier, product category, volume thresholds, and subscription plans.

```csharp
public interface IPlatformFeeCalculator
{
    Money Calculate(Money transactionAmount, Guid sellerId, string categoryId);
}

public class CompositeFeeCalculator : IPlatformFeeCalculator
{
    private readonly ISellerRepository _sellerRepo;
    private readonly IFeeConfigRepository _feeConfigRepo;

    public CompositeFeeCalculator(
        ISellerRepository sellerRepo,
        IFeeConfigRepository feeConfigRepo)
    {
        _sellerRepo = sellerRepo;
        _feeConfigRepo = feeConfigRepo;
    }

    public Money Calculate(Money transactionAmount, Guid sellerId, string categoryId)
    {
        var seller = _sellerRepo.GetById(sellerId);
        var feeConfig = _feeConfigRepo.GetActiveConfig();

        // Check if the seller has a subscription that modifies fees
        if (seller.ActiveSubscription is not null)
        {
            return CalculateSubscriptionFee(
                transactionAmount, seller.ActiveSubscription);
        }

        // Check for category-specific fee override
        var categoryFee = feeConfig.CategoryFees
            .FirstOrDefault(cf => cf.CategoryId == categoryId);
        if (categoryFee is not null)
        {
            return CalculateFromRate(transactionAmount, categoryFee.Rate);
        }

        // Apply tiered fee based on seller's monthly volume
        var monthlyVolume = _sellerRepo.GetMonthlyVolume(sellerId);
        var tier = feeConfig.VolumeTiers
            .OrderByDescending(t => t.MinVolume)
            .FirstOrDefault(t => monthlyVolume >= t.MinVolume);

        var rate = tier?.Rate ?? feeConfig.DefaultRate;
        var fee = CalculateFromRate(transactionAmount, rate);

        // Apply minimum and maximum fee caps
        if (fee < feeConfig.MinimumFee) fee = feeConfig.MinimumFee;
        if (feeConfig.MaximumFee.HasValue && fee > feeConfig.MaximumFee.Value)
            fee = feeConfig.MaximumFee.Value;

        return fee;
    }

    private static Money CalculateFromRate(Money amount, FeeRate rate) =>
        rate.Type switch
        {
            FeeRateType.Percentage =>
                Money.Of((long)(amount.MinorUnits * rate.Percentage / 100m),
                    amount.Currency),
            FeeRateType.Fixed =>
                rate.FixedAmount,
            FeeRateType.PercentagePlusFixed =>
                Money.Of((long)(amount.MinorUnits * rate.Percentage / 100m),
                    amount.Currency) + rate.FixedAmount,
            _ => throw new ArgumentException($"Unknown fee rate type: {rate.Type}")
        };

    private static Money CalculateSubscriptionFee(
        Money amount, SellerSubscription subscription) =>
        Money.Of(
            (long)(amount.MinorUnits * subscription.TransactionFeeRate / 100m),
            amount.Currency);
}

public class FeeConfiguration
{
    public FeeRate DefaultRate { get; set; } = null!;
    public Money MinimumFee { get; set; }
    public Money? MaximumFee { get; set; }
    public List<CategoryFee> CategoryFees { get; set; } = new();
    public List<VolumeTier> VolumeTiers { get; set; } = new();
}

public record FeeRate
{
    public FeeRateType Type { get; init; }
    public decimal Percentage { get; init; }
    public Money FixedAmount { get; init; }
}

public enum FeeRateType { Percentage, Fixed, PercentagePlusFixed }

public record CategoryFee(string CategoryId, FeeRate Rate);

public record VolumeTier(Money MinVolume, FeeRate Rate);
```

### Seller Subscription Plans

```csharp
public class SellerSubscription
{
    public Guid Id { get; private set; }
    public Guid SellerId { get; private set; }
    public SellerPlan Plan { get; private set; }
    public decimal TransactionFeeRate { get; private set; }
    public Money MonthlyFee { get; private set; }
    public DateTimeOffset StartDate { get; private set; }
    public DateTimeOffset? EndDate { get; private set; }
    public bool IsActive => EndDate is null || EndDate > DateTimeOffset.UtcNow;
}

public enum SellerPlan
{
    /// <summary>No subscription — default commission rates apply.</summary>
    Free,

    /// <summary>Lower transaction fees, basic analytics.</summary>
    Professional,

    /// <summary>Lowest transaction fees, priority support, advanced analytics.</summary>
    Enterprise
}

public class SellerPlanConfiguration
{
    public static readonly Dictionary<SellerPlan, PlanDetails> Plans = new()
    {
        [SellerPlan.Free] = new PlanDetails(
            MonthlyFee: Money.USD(0),
            TransactionFeePercent: 15.0m,
            Features: new[] { "Basic storefront", "Standard support" }),

        [SellerPlan.Professional] = new PlanDetails(
            MonthlyFee: Money.USD(39_99),
            TransactionFeePercent: 10.0m,
            Features: new[]
            {
                "Custom storefront", "Priority support",
                "Analytics dashboard", "Promoted listings"
            }),

        [SellerPlan.Enterprise] = new PlanDetails(
            MonthlyFee: Money.USD(199_99),
            TransactionFeePercent: 5.0m,
            Features: new[]
            {
                "White-label storefront", "Dedicated account manager",
                "Advanced analytics", "API access", "Bulk listing tools"
            })
    };
}

public record PlanDetails(
    Money MonthlyFee,
    decimal TransactionFeePercent,
    string[] Features);
```

### Fee Invoicing

```csharp
public class PlatformFeeInvoiceService
{
    private readonly ISellerRepository _sellerRepo;
    private readonly IInvoiceRepository _invoiceRepo;

    public PlatformFeeInvoiceService(
        ISellerRepository sellerRepo,
        IInvoiceRepository invoiceRepo)
    {
        _sellerRepo = sellerRepo;
        _invoiceRepo = invoiceRepo;
    }

    /// <summary>
    /// Generates monthly fee invoices for sellers on paid subscription plans.
    /// </summary>
    public async Task GenerateMonthlyInvoicesAsync(
        int year, int month, CancellationToken ct)
    {
        var sellers = await _sellerRepo.GetSellersWithActiveSubscriptionsAsync(ct);

        foreach (var seller in sellers)
        {
            var subscription = seller.ActiveSubscription!;
            var transactionFees = await _sellerRepo
                .GetMonthlyPlatformFeesAsync(seller.Id, year, month, ct);

            var invoice = new PlatformFeeInvoice
            {
                Id = Guid.NewGuid(),
                SellerId = seller.Id,
                Period = new DateOnly(year, month, 1),
                SubscriptionFee = subscription.MonthlyFee,
                TransactionFeesTotal = transactionFees,
                TotalDue = subscription.MonthlyFee + transactionFees,
                Status = InvoiceStatus.Pending,
                DueDate = new DateOnly(year, month, 1).AddMonths(1).AddDays(14)
            };

            await _invoiceRepo.AddAsync(invoice, ct);
        }
    }
}
```

---

## 6. Dispute & Chargeback Arbitration

### Marketplace Dispute Flow

In a marketplace, disputes involve three parties: buyer, seller, and platform. The platform acts as arbiter, collecting evidence from sellers and making decisions based on marketplace policies.

```csharp
public enum MarketplaceDisputeStatus
{
    Opened,
    SellerNotified,
    SellerEvidenceSubmitted,
    PlatformReviewing,
    EscalatedToPaymentProvider,
    ResolvedBuyerFavor,
    ResolvedSellerFavor,
    Closed
}

public class MarketplaceDispute
{
    public Guid Id { get; private set; }
    public Guid OrderId { get; private set; }
    public Guid BuyerId { get; private set; }
    public Guid SellerId { get; private set; }
    public string? PaymentProviderDisputeId { get; private set; }
    public MarketplaceDisputeStatus Status { get; private set; }
    public DisputeReason Reason { get; private set; }
    public Money DisputedAmount { get; private set; }
    public string BuyerDescription { get; private set; } = null!;
    public SellerEvidence? SellerEvidence { get; private set; }
    public PlatformDecision? PlatformDecision { get; private set; }
    public DateTimeOffset OpenedAt { get; private set; }
    public DateTimeOffset SellerResponseDeadline { get; private set; }
    public DateTimeOffset? ResolvedAt { get; private set; }

    public static MarketplaceDispute Open(
        Guid orderId, Guid buyerId, Guid sellerId,
        DisputeReason reason, Money amount, string description)
    {
        return new MarketplaceDispute
        {
            Id = Guid.NewGuid(),
            OrderId = orderId,
            BuyerId = buyerId,
            SellerId = sellerId,
            Status = MarketplaceDisputeStatus.Opened,
            Reason = reason,
            DisputedAmount = amount,
            BuyerDescription = description,
            OpenedAt = DateTimeOffset.UtcNow,
            SellerResponseDeadline = DateTimeOffset.UtcNow.AddDays(3)
        };
    }

    public void SubmitSellerEvidence(SellerEvidence evidence)
    {
        SellerEvidence = evidence;
        Status = MarketplaceDisputeStatus.SellerEvidenceSubmitted;
    }

    public void Resolve(PlatformDecision decision)
    {
        PlatformDecision = decision;
        Status = decision.InFavorOf == DisputeParty.Buyer
            ? MarketplaceDisputeStatus.ResolvedBuyerFavor
            : MarketplaceDisputeStatus.ResolvedSellerFavor;
        ResolvedAt = DateTimeOffset.UtcNow;
    }
}

public enum DisputeReason
{
    ItemNotReceived,
    ItemNotAsDescribed,
    Unauthorized,
    DuplicateCharge,
    Other
}

public record SellerEvidence(
    string Response,
    string? TrackingNumber,
    string? Carrier,
    List<DocumentUpload> SupportingDocuments,
    DateTimeOffset SubmittedAt);

public record PlatformDecision(
    DisputeParty InFavorOf,
    string Reasoning,
    Money? RefundAmount,
    Guid DecidedBy,
    DateTimeOffset DecidedAt);

public enum DisputeParty { Buyer, Seller }
```

### Dispute Arbitration Service

```csharp
public class DisputeArbitrationService
{
    private readonly IMarketplaceDisputeRepository _disputeRepo;
    private readonly IRefundService _refundService;
    private readonly INotificationService _notificationService;
    private readonly ILogger<DisputeArbitrationService> _logger;

    public DisputeArbitrationService(
        IMarketplaceDisputeRepository disputeRepo,
        IRefundService refundService,
        INotificationService notificationService,
        ILogger<DisputeArbitrationService> logger)
    {
        _disputeRepo = disputeRepo;
        _refundService = refundService;
        _notificationService = notificationService;
        _logger = logger;
    }

    /// <summary>
    /// Opens a dispute and notifies the seller to provide evidence.
    /// </summary>
    public async Task<MarketplaceDispute> OpenDisputeAsync(
        OpenDisputeRequest request, CancellationToken ct)
    {
        var dispute = MarketplaceDispute.Open(
            request.OrderId, request.BuyerId, request.SellerId,
            request.Reason, request.Amount, request.Description);

        await _disputeRepo.AddAsync(dispute, ct);

        dispute.Status = MarketplaceDisputeStatus.SellerNotified;
        await _disputeRepo.UpdateAsync(dispute, ct);

        await _notificationService.NotifySellerDisputeOpenedAsync(
            dispute.SellerId, dispute.Id, dispute.SellerResponseDeadline, ct);

        return dispute;
    }

    /// <summary>
    /// Platform reviews evidence and makes a decision.
    /// Automated decision for clear-cut cases, manual for complex ones.
    /// </summary>
    public async Task<PlatformDecision> ArbitrateAsync(
        Guid disputeId, Guid reviewerId, CancellationToken ct)
    {
        var dispute = await _disputeRepo.GetByIdAsync(disputeId, ct)
            ?? throw new EntityNotFoundException(nameof(MarketplaceDispute), disputeId);

        dispute.Status = MarketplaceDisputeStatus.PlatformReviewing;

        // Attempt automated resolution for common scenarios
        var autoDecision = TryAutoResolve(dispute);
        if (autoDecision is not null)
        {
            dispute.Resolve(autoDecision);
            await _disputeRepo.UpdateAsync(dispute, ct);

            if (autoDecision.RefundAmount.HasValue)
            {
                await _refundService.ProcessDisputeRefundAsync(
                    dispute.OrderId, autoDecision.RefundAmount.Value, ct);
            }

            return autoDecision;
        }

        // Queue for manual review
        await _disputeRepo.UpdateAsync(dispute, ct);
        return null!;
    }

    private static PlatformDecision? TryAutoResolve(MarketplaceDispute dispute)
    {
        // Auto-resolve in buyer's favor if seller didn't respond in time
        if (dispute.SellerEvidence is null &&
            DateTimeOffset.UtcNow > dispute.SellerResponseDeadline)
        {
            return new PlatformDecision(
                InFavorOf: DisputeParty.Buyer,
                Reasoning: "Seller did not respond within the deadline.",
                RefundAmount: dispute.DisputedAmount,
                DecidedBy: Guid.Empty,
                DecidedAt: DateTimeOffset.UtcNow);
        }

        // Auto-resolve in seller's favor if tracking shows delivered
        if (dispute.Reason == DisputeReason.ItemNotReceived &&
            dispute.SellerEvidence?.TrackingNumber is not null)
        {
            return new PlatformDecision(
                InFavorOf: DisputeParty.Seller,
                Reasoning: "Tracking information confirms delivery.",
                RefundAmount: null,
                DecidedBy: Guid.Empty,
                DecidedAt: DateTimeOffset.UtcNow);
        }

        return null;
    }
}
```

### Chargeback Liability Management

```csharp
public class ChargebackLiabilityService
{
    private readonly ISellerRepository _sellerRepo;
    private readonly ILedgerService _ledger;

    public ChargebackLiabilityService(
        ISellerRepository sellerRepo,
        ILedgerService ledger)
    {
        _sellerRepo = sellerRepo;
        _ledger = ledger;
    }

    /// <summary>
    /// Determines who bears the cost of a chargeback — seller or platform —
    /// based on marketplace policy and dispute details.
    /// </summary>
    public async Task<ChargebackLiabilityResult> DetermineLiabilityAsync(
        Guid orderId, Guid sellerId, Money chargebackAmount,
        ChargebackReasonCode reasonCode, CancellationToken ct)
    {
        var seller = await _sellerRepo.GetByIdAsync(sellerId, ct);

        // Platform absorbs fraud-related chargebacks if 3DS was used
        if (reasonCode == ChargebackReasonCode.Fraud &&
            await WasThreeDSecureUsedAsync(orderId, ct))
        {
            return new ChargebackLiabilityResult
            {
                LiableParty = DisputeParty.Seller,
                PlatformAbsorbs = true,
                Reason = "Liability shift: 3DS authentication was completed"
            };
        }

        // Seller is liable if chargeback rate exceeds threshold
        var chargebackRate = await _sellerRepo.GetChargebackRateAsync(sellerId, ct);
        if (chargebackRate > 0.01m)
        {
            await _ledger.DebitSellerAccountAsync(
                sellerId, chargebackAmount, "chargeback_debit", ct);

            return new ChargebackLiabilityResult
            {
                LiableParty = DisputeParty.Seller,
                PlatformAbsorbs = false,
                Reason = $"Seller chargeback rate ({chargebackRate:P2}) exceeds threshold"
            };
        }

        // Default: seller liable, deducted from next payout
        await _ledger.DebitSellerAccountAsync(
            sellerId, chargebackAmount, "chargeback_debit", ct);

        return new ChargebackLiabilityResult
        {
            LiableParty = DisputeParty.Seller,
            PlatformAbsorbs = false,
            Reason = "Standard marketplace policy: seller bears chargeback cost"
        };
    }

    private Task<bool> WasThreeDSecureUsedAsync(Guid orderId, CancellationToken ct)
    {
        // Check payment records for 3DS authentication
        return Task.FromResult(false);
    }
}

public record ChargebackLiabilityResult
{
    public DisputeParty LiableParty { get; init; }
    public bool PlatformAbsorbs { get; init; }
    public string Reason { get; init; } = null!;
}

public enum ChargebackReasonCode
{
    Fraud,
    ProductNotReceived,
    ProductNotAsDescribed,
    DuplicateTransaction,
    SubscriptionCanceled,
    Other
}
```

---

## 7. Seller Dashboard & Reporting

### Seller Balance Tracking

```csharp
public class SellerBalanceService
{
    private readonly ILedgerService _ledger;
    private readonly IPayoutRepository _payoutRepo;

    public SellerBalanceService(
        ILedgerService ledger,
        IPayoutRepository payoutRepo)
    {
        _ledger = ledger;
        _payoutRepo = payoutRepo;
    }

    public async Task<SellerBalance> GetBalanceAsync(
        Guid sellerId, CancellationToken ct)
    {
        var availableBalance = await _ledger.GetAvailableBalanceAsync(sellerId, ct);
        var pendingBalance = await _ledger.GetPendingBalanceAsync(sellerId, ct);
        var pendingPayouts = await _payoutRepo
            .GetPendingPayoutsAsync(sellerId, ct);
        var lifetimeEarnings = await _ledger
            .GetLifetimeEarningsAsync(sellerId, ct);

        return new SellerBalance
        {
            SellerId = sellerId,
            Available = availableBalance,
            Pending = pendingBalance,
            InTransit = pendingPayouts.Sum(p => p.Amount),
            LifetimeEarnings = lifetimeEarnings,
            LastPayoutDate = pendingPayouts
                .OrderByDescending(p => p.CreatedAt)
                .FirstOrDefault()?.CreatedAt
        };
    }
}

public record SellerBalance
{
    public Guid SellerId { get; init; }
    public Money Available { get; init; }
    public Money Pending { get; init; }
    public Money InTransit { get; init; }
    public Money LifetimeEarnings { get; init; }
    public DateTimeOffset? LastPayoutDate { get; init; }
}
```

### Settlement Reports

```csharp
public class SettlementReportService
{
    private readonly ITransactionRepository _transactionRepo;
    private readonly IPayoutRepository _payoutRepo;
    private readonly IRefundRepository _refundRepo;

    public SettlementReportService(
        ITransactionRepository transactionRepo,
        IPayoutRepository payoutRepo,
        IRefundRepository refundRepo)
    {
        _transactionRepo = transactionRepo;
        _payoutRepo = payoutRepo;
        _refundRepo = refundRepo;
    }

    /// <summary>
    /// Generates a detailed settlement report for a seller for a given period.
    /// </summary>
    public async Task<SellerSettlementReport> GenerateReportAsync(
        Guid sellerId, DateOnly from, DateOnly to, CancellationToken ct)
    {
        var transactions = await _transactionRepo
            .GetBySellerAndPeriodAsync(sellerId, from, to, ct);
        var payouts = await _payoutRepo
            .GetBySellerAndPeriodAsync(sellerId, from, to, ct);
        var refunds = await _refundRepo
            .GetBySellerAndPeriodAsync(sellerId, from, to, ct);

        var grossSales = transactions.Sum(t => t.GrossAmount);
        var totalRefunds = refunds.Sum(r => r.Amount);
        var totalFees = transactions.Sum(t => t.PlatformFee);
        var totalPayouts = payouts.Sum(p => p.Amount);

        return new SellerSettlementReport
        {
            SellerId = sellerId,
            PeriodStart = from,
            PeriodEnd = to,
            GrossSales = grossSales,
            Refunds = totalRefunds,
            PlatformFees = totalFees,
            ProcessingFees = transactions.Sum(t => t.ProcessingFee),
            NetRevenue = grossSales - totalRefunds - totalFees,
            TotalPayouts = totalPayouts,
            TransactionCount = transactions.Count,
            RefundCount = refunds.Count,
            Transactions = transactions.Select(t => new TransactionLineItem
            {
                TransactionId = t.Id,
                Date = t.CreatedAt,
                OrderId = t.OrderId,
                GrossAmount = t.GrossAmount,
                PlatformFee = t.PlatformFee,
                ProcessingFee = t.ProcessingFee,
                NetAmount = t.NetAmount
            }).ToList()
        };
    }
}

public record SellerSettlementReport
{
    public Guid SellerId { get; init; }
    public DateOnly PeriodStart { get; init; }
    public DateOnly PeriodEnd { get; init; }
    public Money GrossSales { get; init; }
    public Money Refunds { get; init; }
    public Money PlatformFees { get; init; }
    public Money ProcessingFees { get; init; }
    public Money NetRevenue { get; init; }
    public Money TotalPayouts { get; init; }
    public int TransactionCount { get; init; }
    public int RefundCount { get; init; }
    public List<TransactionLineItem> Transactions { get; init; } = new();
}

public record TransactionLineItem
{
    public Guid TransactionId { get; init; }
    public DateTimeOffset Date { get; init; }
    public Guid OrderId { get; init; }
    public Money GrossAmount { get; init; }
    public Money PlatformFee { get; init; }
    public Money ProcessingFee { get; init; }
    public Money NetAmount { get; init; }
}
```

### Tax Document Generation (1099-K)

```csharp
public class TaxDocumentService
{
    private readonly ISellerRepository _sellerRepo;
    private readonly ITransactionRepository _transactionRepo;
    private readonly ITaxDocumentRepository _taxDocRepo;

    public TaxDocumentService(
        ISellerRepository sellerRepo,
        ITransactionRepository transactionRepo,
        ITaxDocumentRepository taxDocRepo)
    {
        _sellerRepo = sellerRepo;
        _transactionRepo = transactionRepo;
        _taxDocRepo = taxDocRepo;
    }

    /// <summary>
    /// Generates 1099-K forms for US sellers who meet the IRS reporting threshold.
    /// As of 2024, the threshold is $5,000 in gross payments.
    /// </summary>
    public async Task<List<Tax1099K>> GenerateAnnual1099KAsync(
        int taxYear, CancellationToken ct)
    {
        var sellers = await _sellerRepo.GetUsSellersAsync(ct);
        var documents = new List<Tax1099K>();

        foreach (var seller in sellers)
        {
            var grossPayments = await _transactionRepo
                .GetAnnualGrossPaymentsAsync(seller.Id, taxYear, ct);
            var transactionCount = await _transactionRepo
                .GetAnnualTransactionCountAsync(seller.Id, taxYear, ct);

            // IRS threshold check
            if (grossPayments.MinorUnits < 500_000)
                continue;

            var monthlyBreakdown = new Money[12];
            for (int month = 1; month <= 12; month++)
            {
                monthlyBreakdown[month - 1] = await _transactionRepo
                    .GetMonthlyGrossPaymentsAsync(seller.Id, taxYear, month, ct);
            }

            var document = new Tax1099K
            {
                TaxYear = taxYear,
                SellerId = seller.Id,
                SellerTin = seller.TaxId,
                SellerLegalName = seller.LegalName,
                SellerAddress = seller.LegalAddress,
                GrossAmount = grossPayments,
                TransactionCount = transactionCount,
                MonthlyAmounts = monthlyBreakdown,
                PlatformName = "Marketplace Platform",
                PlatformEin = "XX-XXXXXXX",
                GeneratedAt = DateTimeOffset.UtcNow
            };

            documents.Add(document);
            await _taxDocRepo.AddAsync(document, ct);
        }

        return documents;
    }
}

public record Tax1099K
{
    public int TaxYear { get; init; }
    public Guid SellerId { get; init; }
    public string SellerTin { get; init; } = null!;
    public string SellerLegalName { get; init; } = null!;
    public Address SellerAddress { get; init; } = null!;
    public Money GrossAmount { get; init; }
    public int TransactionCount { get; init; }
    public Money[] MonthlyAmounts { get; init; } = null!;
    public string PlatformName { get; init; } = null!;
    public string PlatformEin { get; init; } = null!;
    public DateTimeOffset GeneratedAt { get; init; }
}
```

---

## 8. Multi-Vendor Cart & Checkout

### Multi-Seller Cart Model

When a buyer adds items from multiple sellers to a single cart, the platform must handle per-seller shipping, order splitting, and payment routing.

```csharp
public class MultiVendorCart
{
    public Guid Id { get; private set; }
    public Guid BuyerId { get; private set; }
    public List<CartItem> Items { get; private set; } = new();

    /// <summary>
    /// Groups cart items by seller for display, shipping, and payment splitting.
    /// </summary>
    public IReadOnlyList<SellerCartGroup> GetSellerGroups()
    {
        return Items
            .GroupBy(i => i.SellerId)
            .Select(g => new SellerCartGroup
            {
                SellerId = g.Key,
                SellerName = g.First().SellerName,
                Items = g.ToList(),
                Subtotal = Money.Of(
                    g.Sum(i => i.LineTotal.MinorUnits),
                    g.First().LineTotal.Currency)
            })
            .ToList();
    }

    public Money GetCartTotal() =>
        Money.Of(Items.Sum(i => i.LineTotal.MinorUnits),
            Items.First().LineTotal.Currency);
}

public record CartItem
{
    public Guid ProductId { get; init; }
    public Guid SellerId { get; init; }
    public string SellerName { get; init; } = null!;
    public string ProductName { get; init; } = null!;
    public int Quantity { get; init; }
    public Money UnitPrice { get; init; }
    public Money LineTotal => Money.Of(
        UnitPrice.MinorUnits * Quantity, UnitPrice.Currency);
    public string CategoryId { get; init; } = null!;
}

public record SellerCartGroup
{
    public Guid SellerId { get; init; }
    public string SellerName { get; init; } = null!;
    public List<CartItem> Items { get; init; } = new();
    public Money Subtotal { get; init; }
    public Money? ShippingCost { get; set; }
    public Money Total => ShippingCost.HasValue
        ? Subtotal + ShippingCost.Value
        : Subtotal;
}
```

### Per-Seller Shipping Calculation

```csharp
public class MultiVendorShippingService
{
    private readonly IShippingRateProvider _shippingProvider;

    public MultiVendorShippingService(IShippingRateProvider shippingProvider)
    {
        _shippingProvider = shippingProvider;
    }

    /// <summary>
    /// Calculates shipping for each seller group independently, since
    /// each seller ships from a different location.
    /// </summary>
    public async Task<MultiVendorShippingResult> CalculateShippingAsync(
        IReadOnlyList<SellerCartGroup> sellerGroups,
        Address shippingAddress,
        CancellationToken ct)
    {
        var results = new List<SellerShippingQuote>();

        // Fetch shipping rates in parallel
        var tasks = sellerGroups.Select(async group =>
        {
            var rates = await _shippingProvider.GetRatesAsync(
                new ShippingRateRequest
                {
                    SellerId = group.SellerId,
                    Items = group.Items.Select(i => new ShippableItem
                    {
                        ProductId = i.ProductId,
                        Quantity = i.Quantity
                    }).ToList(),
                    Destination = shippingAddress
                }, ct);

            return new SellerShippingQuote
            {
                SellerId = group.SellerId,
                AvailableRates = rates,
                SelectedRate = rates.OrderBy(r => r.Cost.MinorUnits).First()
            };
        });

        results.AddRange(await Task.WhenAll(tasks));

        return new MultiVendorShippingResult
        {
            SellerQuotes = results,
            TotalShipping = Money.Of(
                results.Sum(r => r.SelectedRate.Cost.MinorUnits),
                results.First().SelectedRate.Cost.Currency)
        };
    }
}
```

### Order Splitting

```csharp
public class MultiVendorCheckoutService
{
    private readonly SeparateChargeAndTransferService _paymentService;
    private readonly IOrderRepository _orderRepo;
    private readonly IUnitOfWork _unitOfWork;

    public MultiVendorCheckoutService(
        SeparateChargeAndTransferService paymentService,
        IOrderRepository orderRepo,
        IUnitOfWork unitOfWork)
    {
        _paymentService = paymentService;
        _orderRepo = orderRepo;
        _unitOfWork = unitOfWork;
    }

    /// <summary>
    /// Processes a multi-vendor checkout: creates a parent order, sub-orders
    /// per seller, and processes payment with per-seller transfers.
    /// </summary>
    public async Task<MultiVendorOrderResult> CheckoutAsync(
        MultiVendorCart cart, PaymentMethod paymentMethod,
        MultiVendorShippingResult shipping, CancellationToken ct)
    {
        var orderGroupId = Guid.NewGuid().ToString();
        var sellerGroups = cart.GetSellerGroups();

        // Create parent order
        var parentOrder = Order.CreateParent(
            cart.BuyerId, orderGroupId, cart.GetCartTotal());

        // Create sub-orders for each seller
        var subOrders = sellerGroups.Select(group =>
        {
            var shippingQuote = shipping.SellerQuotes
                .First(q => q.SellerId == group.SellerId);

            return SubOrder.Create(
                parentOrderId: parentOrder.Id,
                sellerId: group.SellerId,
                items: group.Items,
                subtotal: group.Subtotal,
                shippingCost: shippingQuote.SelectedRate.Cost);
        }).ToList();

        // Process single charge with multi-seller transfers
        var paymentResult = await _paymentService.ChargeAndTransferAsync(
            new MultiSellerChargeRequest
            {
                OrderGroupId = orderGroupId,
                Currency = cart.GetCartTotal().Currency,
                SellerLineItems = subOrders.Select(so => new SellerLineItem
                {
                    SellerId = so.SellerId,
                    SellerStripeAccountId = so.SellerStripeAccountId,
                    SubOrderId = so.Id,
                    Amount = so.Total,
                    CategoryId = so.PrimaryCategoryId
                }).ToList()
            }, ct);

        // Persist all orders
        await _orderRepo.AddAsync(parentOrder, ct);
        foreach (var subOrder in subOrders)
            await _orderRepo.AddSubOrderAsync(subOrder, ct);

        await _unitOfWork.SaveChangesAsync(ct);

        return new MultiVendorOrderResult
        {
            ParentOrderId = parentOrder.Id,
            SubOrders = subOrders.Select(so => new SubOrderSummary
            {
                SubOrderId = so.Id,
                SellerId = so.SellerId,
                Total = so.Total,
                TransferId = paymentResult.Transfers
                    .First(t => t.SellerId == so.SellerId).TransferId
            }).ToList(),
            PaymentIntentId = paymentResult.PaymentIntentId
        };
    }
}
```

---

## 9. Refund Distribution

### Multi-Party Refund Logic

Refunds in a marketplace must reverse funds across multiple parties: the buyer gets their money back, the seller forfeits the item revenue, and the platform may or may not refund its commission.

```csharp
public enum RefundCommissionPolicy
{
    /// <summary>Platform refunds its commission on refunded items.</summary>
    FullCommissionRefund,

    /// <summary>Platform retains commission even on refunds.</summary>
    RetainCommission,

    /// <summary>Platform refunds commission only on full order refunds.</summary>
    RefundOnFullOrderOnly
}

public class MarketplaceRefundService
{
    private readonly IStripeConnectClient _stripe;
    private readonly IOrderRepository _orderRepo;
    private readonly IPlatformFeeCalculator _feeCalculator;
    private readonly ILedgerService _ledger;
    private readonly MarketplaceRefundConfig _config;

    public MarketplaceRefundService(
        IStripeConnectClient stripe,
        IOrderRepository orderRepo,
        IPlatformFeeCalculator feeCalculator,
        ILedgerService ledger,
        IOptions<MarketplaceRefundConfig> config)
    {
        _stripe = stripe;
        _orderRepo = orderRepo;
        _feeCalculator = feeCalculator;
        _ledger = ledger;
        _config = config.Value;
    }

    /// <summary>
    /// Processes a refund for a marketplace order, distributing the refund
    /// across buyer, seller, and platform according to policy.
    /// </summary>
    public async Task<MarketplaceRefundResult> ProcessRefundAsync(
        MarketplaceRefundRequest request, CancellationToken ct)
    {
        var order = await _orderRepo.GetByIdAsync(request.OrderId, ct)
            ?? throw new EntityNotFoundException(nameof(Order), request.OrderId);

        var refundBreakdown = CalculateRefundDistribution(
            request.RefundAmount, order, request.SellerId);

        // Reverse the transfer to the seller
        if (refundBreakdown.SellerDebit.MinorUnits > 0)
        {
            await _stripe.CreateTransferReversalAsync(
                order.SellerTransferId!,
                new TransferReversalCreateOptions
                {
                    Amount = refundBreakdown.SellerDebit.MinorUnits
                }, ct);
        }

        // Refund the buyer
        await _stripe.CreateRefundAsync(new RefundCreateOptions
        {
            PaymentIntent = order.PaymentIntentId,
            Amount = request.RefundAmount.MinorUnits,
            Metadata = new Dictionary<string, string>
            {
                ["order_id"] = request.OrderId.ToString(),
                ["reason"] = request.Reason.ToString()
            }
        }, ct);

        // Record ledger entries
        await _ledger.RecordRefundAsync(new RefundLedgerEntry
        {
            OrderId = request.OrderId,
            SellerId = request.SellerId,
            BuyerRefund = request.RefundAmount,
            SellerDebit = refundBreakdown.SellerDebit,
            PlatformFeeRefund = refundBreakdown.PlatformFeeRefund,
            PlatformRetainedFee = refundBreakdown.PlatformRetainedFee
        }, ct);

        return new MarketplaceRefundResult
        {
            RefundId = Guid.NewGuid(),
            BuyerRefundAmount = request.RefundAmount,
            SellerDebitAmount = refundBreakdown.SellerDebit,
            PlatformFeeRefunded = refundBreakdown.PlatformFeeRefund,
            Policy = _config.CommissionPolicy
        };
    }

    private RefundDistribution CalculateRefundDistribution(
        Money refundAmount, Order order, Guid sellerId)
    {
        var originalFee = order.PlatformFee;
        var refundRatio = (decimal)refundAmount.MinorUnits / order.Total.MinorUnits;
        var proportionalFee = Money.Of(
            (long)(originalFee.MinorUnits * refundRatio), originalFee.Currency);

        return _config.CommissionPolicy switch
        {
            RefundCommissionPolicy.FullCommissionRefund => new RefundDistribution
            {
                SellerDebit = refundAmount - proportionalFee,
                PlatformFeeRefund = proportionalFee,
                PlatformRetainedFee = Money.Zero(originalFee.Currency)
            },
            RefundCommissionPolicy.RetainCommission => new RefundDistribution
            {
                SellerDebit = refundAmount,
                PlatformFeeRefund = Money.Zero(originalFee.Currency),
                PlatformRetainedFee = proportionalFee
            },
            RefundCommissionPolicy.RefundOnFullOrderOnly => new RefundDistribution
            {
                SellerDebit = refundAmount == order.Total
                    ? refundAmount - originalFee
                    : refundAmount,
                PlatformFeeRefund = refundAmount == order.Total
                    ? originalFee
                    : Money.Zero(originalFee.Currency),
                PlatformRetainedFee = refundAmount == order.Total
                    ? Money.Zero(originalFee.Currency)
                    : proportionalFee
            },
            _ => throw new ArgumentException("Unknown commission policy")
        };
    }
}

public record RefundDistribution
{
    public Money SellerDebit { get; init; }
    public Money PlatformFeeRefund { get; init; }
    public Money PlatformRetainedFee { get; init; }
}

public record MarketplaceRefundResult
{
    public Guid RefundId { get; init; }
    public Money BuyerRefundAmount { get; init; }
    public Money SellerDebitAmount { get; init; }
    public Money PlatformFeeRefunded { get; init; }
    public RefundCommissionPolicy Policy { get; init; }
}

public class MarketplaceRefundConfig
{
    public RefundCommissionPolicy CommissionPolicy { get; set; }
        = RefundCommissionPolicy.FullCommissionRefund;
    public int MaxRefundWindowDays { get; set; } = 30;
}
```

### Multi-Seller Partial Refund

```csharp
public class MultiSellerRefundService
{
    private readonly MarketplaceRefundService _refundService;
    private readonly IOrderRepository _orderRepo;

    public MultiSellerRefundService(
        MarketplaceRefundService refundService,
        IOrderRepository orderRepo)
    {
        _refundService = refundService;
        _orderRepo = orderRepo;
    }

    /// <summary>
    /// Processes a partial refund that spans multiple sellers in a multi-vendor order.
    /// Each seller's sub-order is refunded independently.
    /// </summary>
    public async Task<MultiSellerRefundResult> RefundMultiSellerOrderAsync(
        Guid parentOrderId,
        List<SellerRefundItem> sellerRefunds,
        CancellationToken ct)
    {
        var results = new List<MarketplaceRefundResult>();

        foreach (var sellerRefund in sellerRefunds)
        {
            var subOrder = await _orderRepo.GetSubOrderAsync(
                parentOrderId, sellerRefund.SellerId, ct);

            if (subOrder is null) continue;

            var result = await _refundService.ProcessRefundAsync(
                new MarketplaceRefundRequest
                {
                    OrderId = subOrder.Id,
                    SellerId = sellerRefund.SellerId,
                    RefundAmount = sellerRefund.RefundAmount,
                    Reason = sellerRefund.Reason
                }, ct);

            results.Add(result);
        }

        return new MultiSellerRefundResult
        {
            ParentOrderId = parentOrderId,
            SellerRefunds = results,
            TotalRefunded = Money.Of(
                results.Sum(r => r.BuyerRefundAmount.MinorUnits),
                results.First().BuyerRefundAmount.Currency)
        };
    }
}

public record SellerRefundItem(
    Guid SellerId,
    Money RefundAmount,
    RefundReason Reason);

public record MultiSellerRefundResult
{
    public Guid ParentOrderId { get; init; }
    public List<MarketplaceRefundResult> SellerRefunds { get; init; } = new();
    public Money TotalRefunded { get; init; }
}

public enum RefundReason
{
    CustomerRequest,
    ItemNotReceived,
    ItemDamaged,
    ItemNotAsDescribed,
    ReturnReceived,
    DisputeResolution
}
```

---

## 10. Compliance & Money Transmission

### Money Transmitter Licensing

Marketplace platforms that hold or move funds on behalf of sellers may be classified as money transmitters. Understanding when you cross that regulatory line is critical.

```csharp
/// <summary>
/// Evaluates whether the platform's payment model requires
/// a money transmitter license based on the flow of funds.
/// </summary>
public class MoneyTransmissionAnalyzer
{
    /// <summary>
    /// Determines regulatory classification based on the marketplace's
    /// payment architecture and fund flow.
    /// </summary>
    public RegulatoryAssessment Analyze(MarketplacePaymentArchitecture architecture)
    {
        var risks = new List<RegulatoryRisk>();

        // Platform holds funds in its own account before transferring
        if (architecture.PlatformHoldsFunds)
        {
            risks.Add(new RegulatoryRisk(
                RiskLevel.High,
                "Platform custody of funds may require MTL",
                "Consider using a registered payment facilitator (e.g., Stripe Connect) " +
                "to avoid direct fund custody."));
        }

        // Platform delays payouts beyond settlement period
        if (architecture.AveragePayoutDelayDays > 7)
        {
            risks.Add(new RegulatoryRisk(
                RiskLevel.Medium,
                "Extended payout delays may indicate fund custody",
                "Reduce payout delays or use escrow through a licensed entity."));
        }

        // Platform converts currencies
        if (architecture.PerformsCurrencyConversion)
        {
            risks.Add(new RegulatoryRisk(
                RiskLevel.Medium,
                "Currency conversion may require additional licensing",
                "Delegate currency conversion to the payment processor."));
        }

        // Platform operates in multiple states/countries
        if (architecture.OperatingJurisdictions.Count > 1)
        {
            risks.Add(new RegulatoryRisk(
                RiskLevel.High,
                "Multi-jurisdiction operation requires per-state/country licensing",
                "Use a PayFac model to shift licensing requirements."));
        }

        return new RegulatoryAssessment
        {
            RequiresMoneyTransmitterLicense = risks.Any(r => r.Level == RiskLevel.High),
            Risks = risks,
            RecommendedModel = risks.Any(r => r.Level == RiskLevel.High)
                ? "Use Stripe Connect or Adyen for Platforms as a registered PayFac"
                : "Current model may be compliant with agent-of-seller exemption"
        };
    }
}

public record RegulatoryAssessment
{
    public bool RequiresMoneyTransmitterLicense { get; init; }
    public List<RegulatoryRisk> Risks { get; init; } = new();
    public string RecommendedModel { get; init; } = null!;
}

public record RegulatoryRisk(
    RiskLevel Level,
    string Description,
    string Mitigation);

public enum RiskLevel { Low, Medium, High }
```

### PSD2 Compliance for EU Platforms

```csharp
/// <summary>
/// Implements PSD2 Strong Customer Authentication (SCA) requirements
/// for EU marketplace transactions.
/// </summary>
public class Psd2ComplianceService
{
    private readonly IStripeConnectClient _stripe;

    public Psd2ComplianceService(IStripeConnectClient stripe)
    {
        _stripe = stripe;
    }

    /// <summary>
    /// Determines if a transaction requires SCA and applies the
    /// appropriate authentication flow.
    /// </summary>
    public async Task<ScaResult> ApplyScaAsync(
        PaymentIntentRequest request, CancellationToken ct)
    {
        var requiresSca = EvaluateScaRequirement(request);

        if (!requiresSca)
        {
            return new ScaResult
            {
                ScaRequired = false,
                ExemptionApplied = ScaExemption.None
            };
        }

        // Check for applicable exemptions
        var exemption = DetermineExemption(request);

        var options = new PaymentIntentCreateOptions
        {
            Amount = request.Amount.MinorUnits,
            Currency = request.Amount.Currency.Code,
            PaymentMethodTypes = new List<string> { "card" },
            PaymentMethodOptions = new PaymentIntentPaymentMethodOptionsOptions
            {
                Card = new PaymentIntentPaymentMethodOptionsCardOptions
                {
                    RequestThreeDSecure = exemption == ScaExemption.None
                        ? "any"
                        : "automatic"
                }
            }
        };

        var intent = await _stripe.CreatePaymentIntentAsync(options, ct);

        return new ScaResult
        {
            ScaRequired = true,
            ExemptionApplied = exemption,
            PaymentIntentId = intent.Id,
            ClientSecret = intent.ClientSecret,
            RequiresAction = intent.Status == "requires_action"
        };
    }

    private static bool EvaluateScaRequirement(PaymentIntentRequest request) =>
        request.BuyerCountry is "EU" or "EEA" or "UK" &&
        request.Amount.MinorUnits > 30_00;

    private static ScaExemption DetermineExemption(PaymentIntentRequest request)
    {
        // Low-value transactions under €30
        if (request.Amount.MinorUnits <= 30_00)
            return ScaExemption.LowValue;

        // Trusted beneficiary (returning customer with established history)
        if (request.IsTrustedBeneficiary)
            return ScaExemption.TrustedBeneficiary;

        // Transaction Risk Analysis (TRA) exemption based on fraud rate
        if (request.Amount.MinorUnits <= 500_00 &&
            request.PlatformFraudRate < 0.006m)
            return ScaExemption.TransactionRiskAnalysis;

        return ScaExemption.None;
    }
}

public enum ScaExemption
{
    None,
    LowValue,
    TrustedBeneficiary,
    TransactionRiskAnalysis,
    RecurringTransaction,
    SecureCorporatePayment
}

public record ScaResult
{
    public bool ScaRequired { get; init; }
    public ScaExemption ExemptionApplied { get; init; }
    public string? PaymentIntentId { get; init; }
    public string? ClientSecret { get; init; }
    public bool RequiresAction { get; init; }
}
```

### Anti-Money Laundering (AML)

```csharp
public class AmlScreeningService
{
    private readonly IAmlProvider _amlProvider;
    private readonly ISellerRepository _sellerRepo;
    private readonly ILogger<AmlScreeningService> _logger;

    public AmlScreeningService(
        IAmlProvider amlProvider,
        ISellerRepository sellerRepo,
        ILogger<AmlScreeningService> logger)
    {
        _amlProvider = amlProvider;
        _sellerRepo = sellerRepo;
        _logger = logger;
    }

    /// <summary>
    /// Screens sellers against sanctions lists and PEP databases
    /// during onboarding and periodically.
    /// </summary>
    public async Task<AmlScreeningResult> ScreenSellerAsync(
        Guid sellerId, CancellationToken ct)
    {
        var seller = await _sellerRepo.GetByIdAsync(sellerId, ct);

        var result = await _amlProvider.ScreenAsync(new AmlScreeningRequest
        {
            FullName = seller.LegalName,
            DateOfBirth = seller.DateOfBirth,
            Country = seller.Country,
            NationalId = seller.TaxId
        }, ct);

        if (result.HasMatches)
        {
            _logger.LogWarning(
                "AML screening flagged seller {SellerId}: {MatchCount} matches found",
                sellerId, result.Matches.Count);

            seller.UpdateOnboardingStatus(SellerOnboardingStatus.UnderReview);
            await _sellerRepo.UpdateAsync(seller, ct);
        }

        return result;
    }

    /// <summary>
    /// Monitors transaction patterns for suspicious activity.
    /// </summary>
    public async Task<SuspiciousActivityReport?> MonitorTransactionsAsync(
        Guid sellerId, CancellationToken ct)
    {
        var seller = await _sellerRepo.GetByIdAsync(sellerId, ct);
        var recentTransactions = await _sellerRepo
            .GetRecentTransactionsAsync(sellerId, TimeSpan.FromDays(30), ct);

        var indicators = new List<string>();

        // Unusual volume spike
        var avgDailyVolume = recentTransactions
            .GroupBy(t => t.CreatedAt.Date)
            .Average(g => g.Sum(t => t.GrossAmount.MinorUnits));
        var todayVolume = recentTransactions
            .Where(t => t.CreatedAt.Date == DateTime.UtcNow.Date)
            .Sum(t => t.GrossAmount.MinorUnits);

        if (todayVolume > avgDailyVolume * 5)
            indicators.Add("Volume spike: 5x above daily average");

        // High refund rate
        var refundRate = recentTransactions.Count(t => t.IsRefunded)
            / (decimal)Math.Max(recentTransactions.Count, 1);
        if (refundRate > 0.3m)
            indicators.Add($"High refund rate: {refundRate:P0}");

        // Rapid successive transactions (structuring)
        var rapidTransactions = recentTransactions
            .OrderBy(t => t.CreatedAt)
            .Zip(recentTransactions.OrderBy(t => t.CreatedAt).Skip(1),
                (a, b) => b.CreatedAt - a.CreatedAt)
            .Count(gap => gap < TimeSpan.FromMinutes(1));

        if (rapidTransactions > 10)
            indicators.Add($"Possible structuring: {rapidTransactions} rapid transactions");

        if (indicators.Count == 0) return null;

        return new SuspiciousActivityReport
        {
            SellerId = sellerId,
            Indicators = indicators,
            ReportedAt = DateTimeOffset.UtcNow,
            RequiresManualReview = true
        };
    }
}

public record SuspiciousActivityReport
{
    public Guid SellerId { get; init; }
    public List<string> Indicators { get; init; } = new();
    public DateTimeOffset ReportedAt { get; init; }
    public bool RequiresManualReview { get; init; }
}
```

---

## 11. Implementation Checklist

### Phase 1: Foundation (Week 1–2)

- [ ] Choose marketplace charge model (direct, destination, or separate charge & transfer)
- [ ] Set up Stripe Connect or PayPal for Marketplaces account
- [ ] Implement seller onboarding flow with hosted identity verification
- [ ] Define platform fee structure (percentage, fixed, or tiered)
- [ ] Create seller entity with onboarding state machine
- [ ] Set up webhook handlers for `account.updated` events

### Phase 2: Payment Flows (Week 3–4)

- [ ] Implement payment splitting for single-seller orders
- [ ] Implement separate charge and transfer for multi-seller orders
- [ ] Build platform fee calculation engine
- [ ] Create multi-vendor cart with per-seller grouping
- [ ] Implement per-seller shipping calculation
- [ ] Build order splitting for multi-vendor checkout

### Phase 3: Payouts & Escrow (Week 5–6)

- [ ] Implement seller balance tracking and ledger
- [ ] Build payout scheduling service (daily/weekly/manual)
- [ ] Set minimum payout thresholds
- [ ] Implement escrow hold patterns for high-risk categories
- [ ] Build escrow release triggers (delivery confirmed, hold expired)
- [ ] Add cross-currency payout support

### Phase 4: Disputes & Refunds (Week 7–8)

- [ ] Build marketplace dispute workflow (open → evidence → arbitrate → resolve)
- [ ] Implement seller evidence collection flow
- [ ] Create automated dispute resolution rules
- [ ] Define chargeback liability policy (seller vs. platform)
- [ ] Implement multi-party refund distribution
- [ ] Handle commission refund policies (retain vs. refund)

### Phase 5: Reporting & Compliance (Week 9–10)

- [ ] Build seller dashboard with balance, payouts, and transaction history
- [ ] Generate settlement reports per seller per period
- [ ] Implement 1099-K generation for US sellers
- [ ] Add AML screening during seller onboarding
- [ ] Implement transaction monitoring for suspicious activity
- [ ] Ensure PSD2/SCA compliance for EU transactions

### Phase 6: Production Hardening (Week 11–12)

- [ ] Evaluate money transmitter licensing requirements
- [ ] Implement idempotency for all payment and transfer operations
- [ ] Add comprehensive audit logging for all fund movements
- [ ] Set up monitoring and alerting for failed payouts and transfers
- [ ] Load-test payment splitting under peak multi-seller checkout
- [ ] Perform security review of seller onboarding data handling
- [ ] Document platform terms of service and seller payment policies

---

## Additional Resources

- **Stripe Connect Documentation** — https://stripe.com/docs/connect
- **PayPal for Marketplaces** — https://developer.paypal.com/docs/multiparty/
- **FinCEN Money Transmitter Guidance** — https://www.fincen.gov/resources/statutes-regulations
- **PSD2 Regulatory Technical Standards** — https://www.eba.europa.eu/regulation-and-policy/payment-services-and-electronic-money
- **IRS 1099-K Reporting Requirements** — https://www.irs.gov/businesses/understanding-your-form-1099-k

---

> **Next Steps:**
>
> - Review [16-STRIPE-INTEGRATION](16-STRIPE-INTEGRATION.md) for Stripe Connect setup and API details.
> - See [17-PAYPAL-INTEGRATION](17-PAYPAL-INTEGRATION.md) for PayPal for Marketplaces integration.
> - See [28-ADDRESS-VERIFICATION-FRAUD-PREVENTION](28-ADDRESS-VERIFICATION-FRAUD-PREVENTION.md) for KYC verification patterns.
> - See [18-COMMON-SCENARIOS](18-COMMON-SCENARIOS.md) for marketplace checkout flows.
