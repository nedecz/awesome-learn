# Address Verification and Fraud Prevention

## Overview

Comprehensive guide to verifying customer addresses, validating customer identity, and preventing fraudulent activity in e-commerce and payment systems. Covers Address Verification Service (AVS), identity verification (KYC), device fingerprinting, 3D Secure authentication, machine-learning fraud scoring, and chargeback prevention strategies — all with .NET implementation examples.

> **Related documents:**
>
> - [15-SECURITY-PATTERNS-PRACTICES](15-SECURITY-PATTERNS-PRACTICES.md) — PCI DSS, encryption, API security
> - [18-COMMON-SCENARIOS](18-COMMON-SCENARIOS.md) — checkout and subscription flows
> - [24-PAYMENT-METHODS-AND-FLOWS](24-PAYMENT-METHODS-AND-FLOWS.md) — payment method details

## Table of Contents

1. [Address Verification Service (AVS)](#1-address-verification-service-avs)
2. [Address Validation and Standardization](#2-address-validation-and-standardization)
3. [Customer Identity Verification (KYC)](#3-customer-identity-verification-kyc)
4. [3D Secure Authentication](#4-3d-secure-authentication)
5. [Device Fingerprinting](#5-device-fingerprinting)
6. [Fraud Risk Scoring Engine](#6-fraud-risk-scoring-engine)
7. [Velocity and Behavioral Analysis](#7-velocity-and-behavioral-analysis)
8. [Order Screening and Manual Review](#8-order-screening-and-manual-review)
9. [Chargeback Prevention and Dispute Management](#9-chargeback-prevention-and-dispute-management)
10. [Machine Learning Fraud Detection](#10-machine-learning-fraud-detection)
11. [Provider-Specific Fraud Tools](#11-provider-specific-fraud-tools)
12. [Implementation Checklist](#12-implementation-checklist)

---

## 1. Address Verification Service (AVS)

### What Is AVS?

AVS is a fraud-prevention mechanism provided by card networks (Visa, Mastercard, etc.) that compares the **billing address** submitted with a transaction against the address the card issuer has on file. The issuer returns a single-character response code indicating whether the street number, ZIP/postal code, both, or neither matched.

> AVS is available primarily in the US, UK, and Canada. Many other countries do not support AVS.

### AVS Response Codes

| Code | Meaning | Risk | Recommendation |
|------|---------|------|----------------|
| **Y** | Street address and ZIP match | Low | Accept |
| **A** | Street address matches, ZIP does not | Medium | Accept with monitoring |
| **Z** | ZIP matches, street address does not | Medium | Accept with monitoring |
| **N** | Neither match | High | Decline or flag for review |
| **U** | Information unavailable (non-US issuer) | Unknown | Accept with caution |
| **R** | Retry — system unavailable | N/A | Retry or accept with monitoring |
| **S** | AVS not supported by issuer | Unknown | Accept with other signals |
| **G** | Global (non-US) issuer, AVS not supported | Unknown | Use alternative verification |

### AVS Implementation

```csharp
/// <summary>
/// Evaluates AVS response codes and returns a risk assessment.
/// </summary>
public class AvsVerificationService
{
    public AvsResult Evaluate(string avsCode, PaymentContext context)
    {
        var result = avsCode?.ToUpperInvariant() switch
        {
            "Y" or "X" or "D" or "M" => new AvsResult
            {
                Status = AvsStatus.FullMatch,
                RiskScore = 0,
                Action = AvsAction.Accept
            },
            "A" => new AvsResult
            {
                Status = AvsStatus.StreetMatchOnly,
                RiskScore = 15,
                Action = AvsAction.AcceptWithMonitoring
            },
            "Z" or "W" or "P" => new AvsResult
            {
                Status = AvsStatus.ZipMatchOnly,
                RiskScore = 15,
                Action = AvsAction.AcceptWithMonitoring
            },
            "N" => new AvsResult
            {
                Status = AvsStatus.NoMatch,
                RiskScore = 50,
                Action = context.TransactionAmount > 200
                    ? AvsAction.Decline
                    : AvsAction.ManualReview
            },
            "U" or "S" or "G" or "I" => new AvsResult
            {
                Status = AvsStatus.Unavailable,
                RiskScore = 10,
                Action = AvsAction.AcceptWithMonitoring
            },
            "R" => new AvsResult
            {
                Status = AvsStatus.Retry,
                RiskScore = 5,
                Action = AvsAction.Retry
            },
            _ => new AvsResult
            {
                Status = AvsStatus.Unknown,
                RiskScore = 20,
                Action = AvsAction.ManualReview
            }
        };

        return result;
    }
}

public enum AvsStatus
{
    FullMatch, StreetMatchOnly, ZipMatchOnly, NoMatch,
    Unavailable, Retry, Unknown
}

public enum AvsAction
{
    Accept, AcceptWithMonitoring, ManualReview, Decline, Retry
}

public record AvsResult
{
    public AvsStatus Status { get; init; }
    public int RiskScore { get; init; }
    public AvsAction Action { get; init; }
}
```

### AVS + CVV Combined Decision Matrix

Use AVS together with CVV (Card Verification Value) checks for stronger fraud prevention:

```csharp
public class AvsAndCvvDecisionEngine
{
    public TransactionDecision Evaluate(string avsCode, string cvvCode)
    {
        var avsMatch = IsAvsMatch(avsCode);   // Y, X, D, M
        var cvvMatch = IsCvvMatch(cvvCode);   // M

        return (avsMatch, cvvMatch) switch
        {
            (true, true)   => TransactionDecision.Accept,
            (true, false)  => TransactionDecision.ManualReview,
            (false, true)  => TransactionDecision.ManualReview,
            (false, false) => TransactionDecision.Decline
        };
    }

    private static bool IsAvsMatch(string code)
        => code is "Y" or "X" or "D" or "M";

    private static bool IsCvvMatch(string code)
        => code is "M";
}
```

### Handling International Addresses

```csharp
public class InternationalAddressVerifier
{
    private readonly IGeocodingService _geocoding;
    private readonly IAddressValidationService _validator;

    /// <summary>
    /// For countries where AVS is not supported, fall back
    /// to geocoding and postal-database validation.
    /// </summary>
    public async Task<AddressVerificationResult> VerifyAsync(Address address)
    {
        // 1. Standardize using postal database
        var standardized = await _validator.StandardizeAsync(address);
        if (!standardized.IsValid)
        {
            return AddressVerificationResult.Invalid(standardized.Errors);
        }

        // 2. Geocode to confirm the address exists
        var geocode = await _geocoding.GeocodeAsync(standardized.Address);
        if (!geocode.IsResolved)
        {
            return AddressVerificationResult.Unverified(
                "Address could not be geocoded.");
        }

        // 3. Compare billing and shipping proximity
        if (address.Type == AddressType.Shipping)
        {
            var distanceKm = GeoUtil.DistanceKm(
                geocode.Location, address.BillingGeoLocation);
            if (distanceKm > 5000)
            {
                return AddressVerificationResult.FlaggedForReview(
                    $"Shipping address is {distanceKm:N0} km from billing address.");
            }
        }

        return AddressVerificationResult.Verified(standardized.Address);
    }
}
```

---

## 2. Address Validation and Standardization

### Why Validate Addresses?

- **Reduce shipping errors** — invalid addresses cause failed deliveries and return costs.
- **Prevent fraud** — fake or mismatched addresses are a strong fraud signal.
- **Improve AVS accuracy** — standardized addresses yield better AVS match rates.
- **Tax compliance** — accurate addresses ensure correct tax calculation.

### Using Third-Party Address Validation APIs

```csharp
public interface IAddressValidationProvider
{
    Task<AddressValidationResult> ValidateAsync(AddressInput input);
}

/// <summary>
/// Example: Google Address Validation API adapter.
/// Real implementations should also consider
/// SmartyStreets, Melissa, Loqate, or EasyPost.
/// </summary>
public class GoogleAddressValidationProvider : IAddressValidationProvider
{
    private readonly HttpClient _httpClient;

    public async Task<AddressValidationResult> ValidateAsync(AddressInput input)
    {
        var request = new
        {
            address = new
            {
                addressLines = new[] { input.Line1, input.Line2 },
                locality = input.City,
                administrativeArea = input.State,
                postalCode = input.PostalCode,
                regionCode = input.CountryCode
            }
        };

        var response = await _httpClient.PostAsJsonAsync(
            "v1:validateAddress", request);
        response.EnsureSuccessStatusCode();

        var body = await response.Content
            .ReadFromJsonAsync<GoogleValidationResponse>();

        return new AddressValidationResult
        {
            IsValid = body.Result.Verdict.ValidationGranularity
                is "PREMISE" or "SUB_PREMISE",
            StandardizedAddress = MapToStandardAddress(body),
            Confidence = MapConfidence(body.Result.Verdict)
        };
    }
}
```

### Address Standardization Pipeline

```csharp
public class AddressStandardizationPipeline
{
    private readonly IAddressValidationProvider _provider;
    private readonly ILogger<AddressStandardizationPipeline> _logger;

    public async Task<StandardizedAddress> StandardizeAsync(AddressInput raw)
    {
        // Step 1 — Trim and normalize whitespace
        var cleaned = CleanInput(raw);

        // Step 2 — Expand abbreviations (St → Street, Ave → Avenue)
        var expanded = ExpandAbbreviations(cleaned);

        // Step 3 — Validate via external provider
        var validation = await _provider.ValidateAsync(expanded);

        if (!validation.IsValid)
        {
            _logger.LogWarning("Address failed validation: {@Input}", raw);
        }

        // Step 4 — Return standardized result
        return new StandardizedAddress
        {
            Line1 = validation.StandardizedAddress.Line1,
            Line2 = validation.StandardizedAddress.Line2,
            City = validation.StandardizedAddress.City,
            StateOrProvince = validation.StandardizedAddress.StateOrProvince,
            PostalCode = validation.StandardizedAddress.PostalCode,
            CountryCode = validation.StandardizedAddress.CountryCode,
            IsValid = validation.IsValid,
            Confidence = validation.Confidence
        };
    }

    private static AddressInput CleanInput(AddressInput raw) => new()
    {
        Line1 = raw.Line1?.Trim(),
        Line2 = raw.Line2?.Trim(),
        City = raw.City?.Trim(),
        State = raw.State?.Trim(),
        PostalCode = raw.PostalCode?.Trim(),
        CountryCode = raw.CountryCode?.Trim().ToUpperInvariant()
    };

    private static AddressInput ExpandAbbreviations(AddressInput input)
    {
        // Simplified — a real implementation uses a full
        // abbreviation dictionary per country
        var abbreviations = new Dictionary<string, string>(
            StringComparer.OrdinalIgnoreCase)
        {
            ["St"] = "Street", ["Ave"] = "Avenue",
            ["Blvd"] = "Boulevard", ["Dr"] = "Drive",
            ["Ln"] = "Lane", ["Rd"] = "Road",
            ["Ct"] = "Court", ["Pl"] = "Place"
        };

        var line = input.Line1 ?? string.Empty;
        foreach (var (abbr, full) in abbreviations)
        {
            line = Regex.Replace(
                line,
                $@"\b{Regex.Escape(abbr)}\b",
                full,
                RegexOptions.IgnoreCase);
        }

        return input with { Line1 = line };
    }
}
```

### Database Schema for Verified Addresses

```sql
CREATE TABLE verified_addresses (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    customer_id         UNIQUEIDENTIFIER NOT NULL,
    address_type        VARCHAR(20) NOT NULL,          -- billing, shipping
    line1               NVARCHAR(255) NOT NULL,
    line2               NVARCHAR(255),
    city                NVARCHAR(100) NOT NULL,
    state_province      NVARCHAR(100),
    postal_code         VARCHAR(20) NOT NULL,
    country_code        CHAR(2) NOT NULL,
    is_verified         BIT NOT NULL DEFAULT 0,
    verification_source VARCHAR(50),                   -- google, smarty, loqate
    verified_at         DATETIME2,
    confidence_score    DECIMAL(3,2),                  -- 0.00 – 1.00
    latitude            DECIMAL(10,7),
    longitude           DECIMAL(10,7),
    created_at          DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    updated_at          DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    CONSTRAINT FK_verified_addresses_customer
        FOREIGN KEY (customer_id) REFERENCES customers(id)
);

CREATE INDEX IX_verified_addresses_customer
    ON verified_addresses (customer_id, address_type);
```

---

## 3. Customer Identity Verification (KYC)

### Why KYC Matters in E-Commerce

- **Regulatory compliance** — financial regulations require knowing your customer for high-value or recurring transactions.
- **Fraud reduction** — verifying identity before allowing high-risk actions (large orders, account changes, payouts).
- **Trust building** — verified sellers/buyers improve marketplace confidence.
- **Chargeback reduction** — verified identities make friendly-fraud disputes easier to fight.

### KYC Verification Levels

```csharp
public enum KycLevel
{
    /// <summary>No verification — guest checkout, low-value orders.</summary>
    None = 0,

    /// <summary>Email + phone verified.</summary>
    Basic = 1,

    /// <summary>Government ID verified via document scan.</summary>
    Enhanced = 2,

    /// <summary>Full identity verification — ID + liveness check + address proof.</summary>
    Full = 3
}
```

### Tiered Verification Based on Risk

```csharp
public class KycPolicyEngine
{
    /// <summary>
    /// Determines the required KYC level based on the action
    /// and the customer's transaction history.
    /// </summary>
    public KycLevel DetermineRequiredLevel(CustomerAction action, CustomerProfile profile)
    {
        return action switch
        {
            // Guest checkout under $50 — no verification
            { Type: ActionType.Purchase, Amount: < 50, IsGuest: true }
                => KycLevel.None,

            // Registered user, low-value purchase
            { Type: ActionType.Purchase, Amount: < 500 }
                => KycLevel.Basic,

            // High-value purchase or first-time large order
            { Type: ActionType.Purchase, Amount: >= 500 }
                when profile.TotalOrders < 3
                => KycLevel.Enhanced,

            // Payout or withdrawal
            { Type: ActionType.Payout }
                => KycLevel.Full,

            // Account changes (email, password, payment method)
            { Type: ActionType.AccountChange }
                => KycLevel.Basic,

            // Seller onboarding on a marketplace
            { Type: ActionType.SellerOnboarding }
                => KycLevel.Full,

            _ => KycLevel.Basic
        };
    }
}
```

### Email and Phone Verification

```csharp
public class EmailPhoneVerificationService
{
    private readonly IDistributedCache _cache;
    private readonly IEmailSender _email;
    private readonly ISmsSender _sms;

    public async Task SendEmailVerificationAsync(string emailAddress)
    {
        var code = GenerateSecureCode(6);
        await _cache.SetStringAsync(
            $"verify:email:{emailAddress}",
            code,
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15)
            });

        await _email.SendAsync(emailAddress,
            "Verify your email",
            $"Your verification code is: {code}");
    }

    public async Task<bool> VerifyEmailCodeAsync(string emailAddress, string code)
    {
        var stored = await _cache.GetStringAsync($"verify:email:{emailAddress}");
        if (stored == null || !CryptographicOperations.FixedTimeEquals(
                Encoding.UTF8.GetBytes(stored),
                Encoding.UTF8.GetBytes(code)))
        {
            return false;
        }

        await _cache.RemoveAsync($"verify:email:{emailAddress}");
        return true;
    }

    public async Task SendPhoneVerificationAsync(string phoneNumber)
    {
        var code = GenerateSecureCode(6);
        await _cache.SetStringAsync(
            $"verify:phone:{phoneNumber}",
            code,
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
            });

        await _sms.SendAsync(phoneNumber,
            $"Your verification code is: {code}");
    }

    private static string GenerateSecureCode(int length)
    {
        return string.Create(length, 0, static (span, _) =>
        {
            ReadOnlySpan<char> digits = "0123456789";
            Span<byte> random = stackalloc byte[span.Length];
            RandomNumberGenerator.Fill(random);
            for (var i = 0; i < span.Length; i++)
            {
                span[i] = digits[random[i] % digits.Length];
            }
        });
    }
}
```

### Document-Based Identity Verification

```csharp
/// <summary>
/// Orchestrates identity document verification via a third-party
/// provider (e.g., Stripe Identity, Jumio, Onfido).
/// </summary>
public class IdentityDocumentVerificationService
{
    private readonly IIdentityVerificationProvider _provider;
    private readonly ILogger<IdentityDocumentVerificationService> _logger;

    public async Task<VerificationSession> CreateSessionAsync(
        Guid customerId, KycLevel requiredLevel)
    {
        var session = await _provider.CreateVerificationSessionAsync(
            new VerificationRequest
            {
                CustomerId = customerId.ToString(),
                RequiredChecks = requiredLevel switch
                {
                    KycLevel.Enhanced => new[]
                    {
                        VerificationCheck.DocumentScan
                    },
                    KycLevel.Full => new[]
                    {
                        VerificationCheck.DocumentScan,
                        VerificationCheck.LivenessCheck,
                        VerificationCheck.AddressProof
                    },
                    _ => Array.Empty<VerificationCheck>()
                }
            });

        _logger.LogInformation(
            "Created verification session {SessionId} for customer {CustomerId}",
            session.Id, customerId);

        return session;
    }

    /// <summary>
    /// Called by a webhook when the verification provider
    /// has finished processing.
    /// </summary>
    public async Task HandleVerificationCompletedAsync(
        string sessionId, VerificationOutcome outcome)
    {
        if (outcome.Status == VerificationStatus.Verified)
        {
            await UpdateCustomerKycLevelAsync(
                outcome.CustomerId, KycLevel.Full);
        }
        else
        {
            _logger.LogWarning(
                "Verification failed for session {SessionId}: {Reason}",
                sessionId, outcome.FailureReason);
        }
    }
}
```

---

## 4. 3D Secure Authentication

### What Is 3D Secure?

3D Secure (3DS) adds an extra authentication step during online card payments. The cardholder is prompted to verify their identity through the card issuer (e.g., a one-time password, biometric, or mobile app approval). This shifts fraud liability from the merchant to the issuer.

### 3DS 2.0 vs 1.0

| Feature | 3DS 1.0 | 3DS 2.0 |
|---------|---------|---------|
| User experience | Full-page redirect, high cart abandonment | Inline iframe/modal, frictionless flow possible |
| Data points | Few | 100+ data elements sent to issuer |
| Frictionless flow | No | Yes — issuer can approve without challenge |
| Mobile support | Poor | Native SDK support |
| Regulation | Optional | Mandatory under PSD2/SCA (EU) |

### When to Require 3DS

```csharp
public class ThreeDSecurePolicyService
{
    /// <summary>
    /// Determines whether to request 3DS authentication.
    /// </summary>
    public ThreeDSecureDecision ShouldRequire3DS(
        PaymentRequest payment, CustomerProfile customer, RiskScore risk)
    {
        // 1. Regulatory requirement — PSD2/SCA for EU transactions
        if (IsEuropeanTransaction(payment))
        {
            return ThreeDSecureDecision.Required(
                "SCA required for EU transactions under PSD2.");
        }

        // 2. High-risk transactions
        if (risk.Level is RiskLevel.High or RiskLevel.Critical)
        {
            return ThreeDSecureDecision.Required(
                $"High risk score: {risk.Score}");
        }

        // 3. First-time customer with large order
        if (customer.TotalOrders == 0 && payment.Amount > 100)
        {
            return ThreeDSecureDecision.Recommended(
                "First-time customer with order > $100.");
        }

        // 4. Known trusted customer with low-value order
        if (customer.TotalOrders > 10 && payment.Amount < 50)
        {
            return ThreeDSecureDecision.Optional(
                "Trusted customer, low-value order.");
        }

        // 5. Default — recommended for liability shift
        return ThreeDSecureDecision.Recommended(
            "Default recommendation for liability shift.");
    }

    private static bool IsEuropeanTransaction(PaymentRequest payment)
    {
        var euCountries = new HashSet<string>
        {
            "AT","BE","BG","HR","CY","CZ","DK","EE","FI","FR",
            "DE","GR","HU","IE","IT","LV","LT","LU","MT","NL",
            "PL","PT","RO","SK","SI","ES","SE","GB","IS","LI","NO"
        };
        return euCountries.Contains(payment.BillingCountry);
    }
}
```

### Implementing 3DS with Stripe

```csharp
public class Stripe3DSecureService
{
    private readonly PaymentIntentService _intentService;

    /// <summary>
    /// Creates a PaymentIntent configured to handle 3DS when required.
    /// </summary>
    public async Task<PaymentIntentResult> CreatePaymentWithAuth(
        decimal amount, string currency, string paymentMethodId)
    {
        var options = new PaymentIntentCreateOptions
        {
            Amount = (long)(amount * 100),
            Currency = currency,
            PaymentMethod = paymentMethodId,
            ConfirmationMethod = "manual",
            Confirm = true,
            // Use automatic to let Stripe decide when 3DS is needed
            PaymentMethodOptions = new PaymentIntentPaymentMethodOptionsOptions
            {
                Card = new PaymentIntentPaymentMethodOptionsCardOptions
                {
                    RequestThreeDSecure = "automatic"
                }
            }
        };

        var intent = await _intentService.CreateAsync(options);

        return intent.Status switch
        {
            "requires_action" => PaymentIntentResult.RequiresAuthentication(
                intent.ClientSecret),
            "succeeded" => PaymentIntentResult.Succeeded(intent.Id),
            _ => PaymentIntentResult.Failed(intent.Status)
        };
    }
}
```

---

## 5. Device Fingerprinting

### What Is Device Fingerprinting?

Device fingerprinting collects technical characteristics of a user's browser or device (screen resolution, installed fonts, browser plugins, timezone, etc.) to create a unique identifier. This helps detect:

- **Account takeover** — login from an unknown device.
- **Multi-account fraud** — same device creating many accounts.
- **Bot detection** — automated scripts with unusual fingerprints.

### Collecting Device Data

```csharp
public class DeviceFingerprint
{
    public string FingerprintHash { get; set; }
    public string UserAgent { get; set; }
    public string AcceptLanguage { get; set; }
    public string Timezone { get; set; }
    public string ScreenResolution { get; set; }
    public bool CookiesEnabled { get; set; }
    public bool JavaScriptEnabled { get; set; }
    public string IpAddress { get; set; }
    public string IpCountry { get; set; }
    public bool IsProxy { get; set; }
    public bool IsVpn { get; set; }
    public bool IsTor { get; set; }
}
```

### Server-Side Device Analysis

```csharp
public class DeviceAnalysisService
{
    private readonly IIpIntelligenceService _ipIntel;
    private readonly IDeviceFingerprintStore _store;

    public async Task<DeviceRiskAssessment> AnalyzeAsync(
        HttpContext context, Guid? customerId)
    {
        var fingerprint = ExtractFingerprint(context);
        var assessment = new DeviceRiskAssessment { Score = 0 };

        // 1. IP reputation check
        var ipInfo = await _ipIntel.LookupAsync(fingerprint.IpAddress);
        if (ipInfo.IsProxy || ipInfo.IsVpn)
        {
            assessment.Score += 20;
            assessment.Flags.Add("Proxy/VPN detected");
        }
        if (ipInfo.IsTor)
        {
            assessment.Score += 30;
            assessment.Flags.Add("Tor exit node detected");
        }

        // 2. Known device check
        if (customerId.HasValue)
        {
            var known = await _store.IsKnownDeviceAsync(
                customerId.Value, fingerprint.FingerprintHash);
            if (!known)
            {
                assessment.Score += 15;
                assessment.Flags.Add("Unknown device for this customer");
            }
        }

        // 3. Device velocity — same device across multiple accounts
        var accountCount = await _store.GetAccountCountForDeviceAsync(
            fingerprint.FingerprintHash, TimeSpan.FromDays(7));
        if (accountCount > 3)
        {
            assessment.Score += 25;
            assessment.Flags.Add(
                $"Device used by {accountCount} accounts in 7 days");
        }

        // 4. Geolocation mismatch
        if (customerId.HasValue)
        {
            var customerCountry = await GetCustomerCountryAsync(customerId.Value);
            if (customerCountry != null && ipInfo.Country != customerCountry)
            {
                assessment.Score += 15;
                assessment.Flags.Add(
                    $"IP country ({ipInfo.Country}) differs from customer country ({customerCountry})");
            }
        }

        // 5. Bot indicators
        if (string.IsNullOrEmpty(fingerprint.UserAgent)
            || fingerprint.UserAgent.Contains("bot", StringComparison.OrdinalIgnoreCase)
            || fingerprint.UserAgent.Contains("crawl", StringComparison.OrdinalIgnoreCase))
        {
            assessment.Score += 40;
            assessment.Flags.Add("Possible bot user-agent");
        }

        assessment.Score = Math.Min(assessment.Score, 100);
        return assessment;
    }

    private static DeviceFingerprint ExtractFingerprint(HttpContext context)
    {
        var headers = context.Request.Headers;
        var ip = context.Connection.RemoteIpAddress?.ToString() ?? "unknown";

        return new DeviceFingerprint
        {
            UserAgent = headers.UserAgent.ToString(),
            AcceptLanguage = headers.AcceptLanguage.ToString(),
            IpAddress = ip,
            FingerprintHash = ComputeHash(
                headers.UserAgent.ToString(),
                headers.AcceptLanguage.ToString(),
                ip)
        };
    }

    private static string ComputeHash(params string[] values)
    {
        var combined = string.Join("|", values);
        var bytes = SHA256.HashData(Encoding.UTF8.GetBytes(combined));
        return Convert.ToHexString(bytes);
    }
}
```

---

## 6. Fraud Risk Scoring Engine

### Multi-Signal Risk Engine

Combine AVS, CVV, device, velocity, and behavioral data into a single risk score:

```csharp
public class ComprehensiveFraudEngine
{
    private readonly AvsVerificationService _avs;
    private readonly DeviceAnalysisService _device;
    private readonly VelocityCheckService _velocity;
    private readonly ICustomerHistoryService _history;

    public async Task<FraudAssessment> EvaluateAsync(
        PaymentRequest payment, HttpContext context)
    {
        var signals = new List<RiskSignal>();
        var totalScore = 0.0;

        // ── AVS Check ──
        var avsResult = _avs.Evaluate(payment.AvsResponseCode, new PaymentContext
        {
            TransactionAmount = payment.Amount
        });
        totalScore += avsResult.RiskScore * 0.20; // 20% weight
        signals.Add(new RiskSignal("AVS", avsResult.RiskScore, avsResult.Status.ToString()));

        // ── CVV Check ──
        var cvvScore = payment.CvvResponseCode == "M" ? 0 : 30;
        totalScore += cvvScore * 0.15; // 15% weight
        signals.Add(new RiskSignal("CVV", cvvScore,
            payment.CvvResponseCode == "M" ? "Match" : "No match"));

        // ── Device Analysis ──
        var deviceResult = await _device.AnalyzeAsync(
            context, payment.CustomerId);
        totalScore += deviceResult.Score * 0.20; // 20% weight
        signals.Add(new RiskSignal("Device", deviceResult.Score,
            string.Join("; ", deviceResult.Flags)));

        // ── Velocity ──
        var cardVelocity = await _velocity.CheckVelocityAsync(
            payment.Last4, VelocityRules.PaymentPerCard);
        var velocityScore = cardVelocity.IsBlocked ? 50 : 0;
        totalScore += velocityScore * 0.15; // 15% weight
        signals.Add(new RiskSignal("Velocity", velocityScore,
            cardVelocity.IsBlocked ? "Velocity limit exceeded" : "OK"));

        // ── Customer History ──
        var historyScore = await CalculateHistoryScoreAsync(payment.CustomerId);
        totalScore += historyScore * 0.15; // 15% weight
        signals.Add(new RiskSignal("History", historyScore, "Based on past orders"));

        // ── Order Characteristics ──
        var orderScore = EvaluateOrderCharacteristics(payment);
        totalScore += orderScore * 0.15; // 15% weight
        signals.Add(new RiskSignal("Order", orderScore, "Based on order attributes"));

        var finalScore = Math.Min(totalScore, 100);

        return new FraudAssessment
        {
            Score = finalScore,
            Level = ClassifyRisk(finalScore),
            Decision = DetermineAction(finalScore),
            Signals = signals,
            EvaluatedAt = DateTimeOffset.UtcNow
        };
    }

    private int EvaluateOrderCharacteristics(PaymentRequest payment)
    {
        var score = 0;

        // High-value order
        if (payment.Amount > 500) score += 10;
        if (payment.Amount > 2000) score += 15;

        // Billing/shipping country mismatch
        if (payment.BillingCountry != payment.ShippingCountry) score += 15;

        // Digital goods (no shipping address) — higher fraud rate
        if (payment.IsDigitalGoods) score += 10;

        // Expedited shipping on first order — common fraud pattern
        if (payment.ShippingMethod == "express" && payment.IsFirstOrder) score += 15;

        return Math.Min(score, 50);
    }

    private static RiskLevel ClassifyRisk(double score) => score switch
    {
        < 20 => RiskLevel.Low,
        < 40 => RiskLevel.Medium,
        < 70 => RiskLevel.High,
        _    => RiskLevel.Critical
    };

    private static FraudDecision DetermineAction(double score) => score switch
    {
        < 20 => FraudDecision.Accept,
        < 40 => FraudDecision.AcceptWithMonitoring,
        < 70 => FraudDecision.ManualReview,
        _    => FraudDecision.Decline
    };
}
```

### Risk Signal Model

```csharp
public record RiskSignal(string Source, double Score, string Detail);

public record FraudAssessment
{
    public double Score { get; init; }
    public RiskLevel Level { get; init; }
    public FraudDecision Decision { get; init; }
    public List<RiskSignal> Signals { get; init; } = new();
    public DateTimeOffset EvaluatedAt { get; init; }
}

public enum FraudDecision
{
    Accept,
    AcceptWithMonitoring,
    ManualReview,
    Decline
}
```

---

## 7. Velocity and Behavioral Analysis

### Advanced Velocity Rules

Beyond the basic velocity checks in [15-SECURITY-PATTERNS-PRACTICES](15-SECURITY-PATTERNS-PRACTICES.md), add these e-commerce-specific rules:

```csharp
public static class EcommerceVelocityRules
{
    // Same email creating multiple accounts
    public static readonly VelocityRule AccountsPerEmail = new()
    {
        Name = "accounts_per_email",
        MaxAttempts = 2,
        WindowMinutes = 1440 // 24 hours
    };

    // Same card used on different accounts
    public static readonly VelocityRule CardAcrossAccounts = new()
    {
        Name = "card_across_accounts",
        MaxAttempts = 3,
        WindowMinutes = 10080 // 7 days
    };

    // Shipping to same address from different accounts
    public static readonly VelocityRule ShippingAddressReuse = new()
    {
        Name = "shipping_address_reuse",
        MaxAttempts = 5,
        WindowMinutes = 10080 // 7 days
    };

    // Rapid address changes before a purchase
    public static readonly VelocityRule AddressChangesBeforePurchase = new()
    {
        Name = "address_changes_before_purchase",
        MaxAttempts = 3,
        WindowMinutes = 60
    };

    // Coupon/promo code abuse
    public static readonly VelocityRule PromoCodePerDevice = new()
    {
        Name = "promo_per_device",
        MaxAttempts = 1,
        WindowMinutes = 43200 // 30 days
    };
}
```

### Behavioral Analysis

```csharp
public class BehavioralAnalysisService
{
    /// <summary>
    /// Scores an order based on behavioral patterns
    /// that indicate potential fraud.
    /// </summary>
    public BehavioralScore Analyze(OrderContext order)
    {
        var score = 0;
        var flags = new List<string>();

        // 1. Time on site before purchase — bots/fraudsters are fast
        if (order.TimeOnSiteSeconds < 30)
        {
            score += 20;
            flags.Add("Suspiciously fast checkout (< 30s on site)");
        }

        // 2. Cart composition — all high-value, easily resellable items
        if (order.Items.All(i => i.Category is "electronics" or "gift_cards"))
        {
            score += 15;
            flags.Add("Cart contains only high-resale-value items");
        }

        // 3. Unusual quantity
        if (order.Items.Any(i => i.Quantity > 5))
        {
            score += 10;
            flags.Add("Unusually high item quantity");
        }

        // 4. Gift card purchase with new account
        if (order.Items.Any(i => i.Category == "gift_cards")
            && order.IsNewAccount)
        {
            score += 25;
            flags.Add("Gift card purchase from new account");
        }

        // 5. Mismatched billing name and account name
        if (!string.Equals(
                order.BillingName, order.AccountName,
                StringComparison.OrdinalIgnoreCase))
        {
            score += 10;
            flags.Add("Billing name differs from account name");
        }

        return new BehavioralScore
        {
            Score = Math.Min(score, 100),
            Flags = flags
        };
    }
}
```

---

## 8. Order Screening and Manual Review

### Automated Order Screening Pipeline

```
Order Placed
    │
    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  AVS + CVV   │───▶│   Device     │───▶│  Velocity    │
│  Check       │    │ Fingerprint  │    │  Check       │
└──────────────┘    └──────────────┘    └──────────────┘
    │                                        │
    ▼                                        ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Behavioral  │───▶│   Risk       │───▶│  Decision    │
│  Analysis    │    │  Scoring     │    │  Engine      │
└──────────────┘    └──────────────┘    └──────────────┘
                                             │
                    ┌────────────────────┬────┴────┬──────────┐
                    ▼                    ▼         ▼          ▼
               ✅ Accept          ⚠️ Monitor   👁️ Review  ❌ Decline
```

### Manual Review Queue

```csharp
public class OrderReviewService
{
    private readonly IOrderRepository _orders;
    private readonly INotificationService _notifications;
    private readonly ILogger<OrderReviewService> _logger;

    /// <summary>
    /// Flags an order for manual review and notifies the fraud team.
    /// </summary>
    public async Task FlagForReviewAsync(
        Guid orderId, FraudAssessment assessment)
    {
        var order = await _orders.GetByIdAsync(orderId);
        order.Status = OrderStatus.PendingReview;
        order.FraudScore = assessment.Score;
        order.FraudFlags = assessment.Signals
            .Where(s => s.Score > 0)
            .Select(s => s.Detail)
            .ToList();

        await _orders.UpdateAsync(order);

        await _notifications.NotifyFraudTeamAsync(new ReviewNotification
        {
            OrderId = orderId,
            Score = assessment.Score,
            Level = assessment.Level,
            Flags = order.FraudFlags,
            CustomerEmail = order.CustomerEmail,
            OrderTotal = order.Total,
            CreatedAt = DateTimeOffset.UtcNow
        });

        _logger.LogWarning(
            "Order {OrderId} flagged for review. Score: {Score}, Flags: {Flags}",
            orderId, assessment.Score, string.Join(", ", order.FraudFlags));
    }

    /// <summary>
    /// Called by a fraud analyst to approve or reject an order.
    /// </summary>
    public async Task ResolveReviewAsync(
        Guid orderId, ReviewDecision decision, string analystId, string notes)
    {
        var order = await _orders.GetByIdAsync(orderId);

        switch (decision)
        {
            case ReviewDecision.Approve:
                order.Status = OrderStatus.Approved;
                await ProcessPaymentAsync(order);
                break;

            case ReviewDecision.Reject:
                order.Status = OrderStatus.Rejected;
                await CancelAuthorizationAsync(order);
                break;

            case ReviewDecision.Escalate:
                order.Status = OrderStatus.Escalated;
                break;
        }

        order.ReviewedBy = analystId;
        order.ReviewNotes = notes;
        order.ReviewedAt = DateTimeOffset.UtcNow;
        await _orders.UpdateAsync(order);
    }
}
```

### Review Dashboard Data Model

```sql
CREATE TABLE fraud_reviews (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    order_id        UNIQUEIDENTIFIER NOT NULL,
    fraud_score     DECIMAL(5,2) NOT NULL,
    risk_level      VARCHAR(20) NOT NULL,
    flags           NVARCHAR(MAX),             -- JSON array
    decision        VARCHAR(20),               -- approve, reject, escalate
    reviewed_by     NVARCHAR(100),
    review_notes    NVARCHAR(MAX),
    created_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    reviewed_at     DATETIME2,
    sla_deadline    DATETIME2,                 -- review must happen by this time
    CONSTRAINT FK_fraud_reviews_order
        FOREIGN KEY (order_id) REFERENCES orders(id)
);

CREATE INDEX IX_fraud_reviews_pending
    ON fraud_reviews (decision, created_at)
    WHERE decision IS NULL;
```

---

## 9. Chargeback Prevention and Dispute Management

### Common Chargeback Reasons

| Reason Code Category | Description | Prevention Strategy |
|----------------------|-------------|---------------------|
| **Fraud** | Cardholder did not authorize | 3DS, AVS, device fingerprinting |
| **Not Received** | Goods/services not delivered | Tracking numbers, delivery confirmation |
| **Not as Described** | Product differs from listing | Accurate descriptions, photos, reviews |
| **Duplicate** | Charged more than once | Idempotency keys, duplicate detection |
| **Subscription** | Customer forgot they subscribed | Clear billing descriptors, reminders |
| **Credit Not Processed** | Refund not issued | Prompt refund processing |

### Chargeback Prevention Service

```csharp
public class ChargebackPreventionService
{
    private readonly IPaymentRepository _payments;
    private readonly IShippingService _shipping;

    /// <summary>
    /// Proactively checks for conditions that commonly lead to chargebacks.
    /// </summary>
    public async Task<ChargebackRisk> AssessChargebackRiskAsync(Guid orderId)
    {
        var order = await _payments.GetOrderWithPaymentsAsync(orderId);
        var risks = new List<string>();

        // 1. Check for duplicate charges
        var duplicates = await _payments.FindDuplicateChargesAsync(
            order.CustomerId, order.Total, TimeSpan.FromHours(24));
        if (duplicates.Count > 1)
        {
            risks.Add("Potential duplicate charge detected");
        }

        // 2. Check shipping status — unshipped orders get "not received" disputes
        if (order.ShippingStatus == ShippingStatus.NotShipped
            && order.CreatedAt < DateTimeOffset.UtcNow.AddDays(-3))
        {
            risks.Add("Order not shipped within 3 days");
        }

        // 3. Check for tracking number
        if (order.RequiresShipping && string.IsNullOrEmpty(order.TrackingNumber))
        {
            risks.Add("No tracking number provided");
        }

        // 4. Check billing descriptor clarity
        if (string.IsNullOrEmpty(order.BillingDescriptor)
            || order.BillingDescriptor.Length < 5)
        {
            risks.Add("Unclear billing descriptor — customer may not recognize charge");
        }

        // 5. Subscription without recent reminder
        if (order.IsSubscription
            && !await HasRecentBillingReminderAsync(order.CustomerId))
        {
            risks.Add("Subscription renewal without advance billing reminder");
        }

        return new ChargebackRisk
        {
            OrderId = orderId,
            Risks = risks,
            Level = risks.Count switch
            {
                0 => RiskLevel.Low,
                1 => RiskLevel.Medium,
                _ => RiskLevel.High
            }
        };
    }
}
```

### Dispute Response Automation

```csharp
public class DisputeResponseService
{
    private readonly IPaymentRepository _payments;
    private readonly IShippingService _shipping;
    private readonly ICustomerService _customers;
    private readonly IFileStorageService _files;

    /// <summary>
    /// Automatically gathers evidence for disputing a chargeback.
    /// </summary>
    public async Task<DisputeEvidence> GatherEvidenceAsync(Guid disputeId)
    {
        var dispute = await _payments.GetDisputeAsync(disputeId);
        var order = await _payments.GetOrderAsync(dispute.OrderId);
        var customer = await _customers.GetAsync(order.CustomerId);

        var evidence = new DisputeEvidence
        {
            DisputeId = disputeId,
            OrderId = order.Id,
            GatheredAt = DateTimeOffset.UtcNow
        };

        // 1. Transaction evidence
        evidence.TransactionReceipt = await GenerateReceiptAsync(order);
        evidence.PaymentConfirmationEmail = await GetConfirmationEmailAsync(order.Id);
        evidence.AvsResult = order.AvsResponseCode;
        evidence.CvvResult = order.CvvResponseCode;

        // 2. Shipping evidence
        if (order.RequiresShipping)
        {
            var tracking = await _shipping.GetTrackingAsync(order.TrackingNumber);
            evidence.TrackingNumber = order.TrackingNumber;
            evidence.CarrierName = tracking.Carrier;
            evidence.DeliveryConfirmation = tracking.DeliveredAt?.ToString("o");
            evidence.ShippingAddress = order.ShippingAddress.ToString();

            if (tracking.SignatureOnFile)
            {
                evidence.DeliverySignature =
                    await _files.GetUrlAsync(tracking.SignatureFileId);
            }
        }

        // 3. Customer evidence
        evidence.CustomerEmail = customer.Email;
        evidence.CustomerIpAddress = order.CustomerIpAddress;
        evidence.CustomerDeviceFingerprint = order.DeviceFingerprint;
        evidence.CustomerPriorTransactions = await _payments
            .GetSuccessfulTransactionCountAsync(customer.Id);

        // 4. Product evidence
        evidence.ProductDescription = string.Join(", ",
            order.Items.Select(i => $"{i.Name} x{i.Quantity}"));

        return evidence;
    }
}
```

### Key Chargeback Prevention Best Practices

1. **Use clear billing descriptors** — include your company name and a recognizable identifier so customers know who charged them.
2. **Send pre-billing notifications** — for subscriptions, email customers 3–7 days before renewal.
3. **Provide easy refund/cancel options** — customers who can self-serve are less likely to file chargebacks.
4. **Ship quickly and provide tracking** — delivery confirmation is your strongest evidence.
5. **Use 3DS for liability shift** — when 3DS authentication succeeds, fraud liability shifts to the issuer.
6. **Keep detailed records** — IP addresses, device fingerprints, email confirmations, and delivery signatures.
7. **Monitor chargeback ratios** — card networks penalize merchants exceeding 0.9–1% chargeback rates.

---

## 10. Machine Learning Fraud Detection

### ML-Based Fraud Scoring Architecture

```
┌─────────────────────────────────────────────┐
│              Feature Extraction              │
│  ┌─────┐ ┌────────┐ ┌──────┐ ┌───────────┐ │
│  │ AVS │ │ Device │ │ Geo  │ │ Behavioral│ │
│  └──┬──┘ └───┬────┘ └──┬───┘ └─────┬─────┘ │
│     └────────┴─────────┴───────────┘        │
│                    │                         │
│              Feature Vector                  │
└────────────────────┬────────────────────────┘
                     │
        ┌────────────┴────────────┐
        ▼                         ▼
  ┌───────────┐           ┌─────────────┐
  │ Rule-Based│           │  ML Model   │
  │  Engine   │           │ (gradient   │
  │           │           │  boosting)  │
  └─────┬─────┘           └──────┬──────┘
        │                        │
        └───────────┬────────────┘
                    ▼
           ┌──────────────┐
           │  Ensemble    │
           │  Decision    │
           └──────────────┘
```

### Feature Engineering

```csharp
public class FraudFeatureExtractor
{
    /// <summary>
    /// Converts a transaction into a feature vector
    /// that can be fed to an ML model.
    /// </summary>
    public FraudFeatureVector Extract(TransactionContext tx)
    {
        return new FraudFeatureVector
        {
            // Transaction features
            Amount = tx.Amount,
            AmountLog = Math.Log(tx.Amount + 1),
            Currency = tx.Currency,
            IsDigitalGoods = tx.IsDigitalGoods ? 1 : 0,

            // AVS features
            AvsMatch = tx.AvsCode is "Y" or "X" or "D" or "M" ? 1 : 0,
            AvsPartialMatch = tx.AvsCode is "A" or "Z" or "W" or "P" ? 1 : 0,
            AvsNoMatch = tx.AvsCode == "N" ? 1 : 0,

            // CVV features
            CvvMatch = tx.CvvCode == "M" ? 1 : 0,

            // Customer features
            AccountAgeDays = (DateTimeOffset.UtcNow - tx.AccountCreatedAt).TotalDays,
            TotalPriorOrders = tx.PriorOrderCount,
            TotalPriorAmount = tx.PriorOrderTotal,
            DaysSinceLastOrder = tx.DaysSinceLastOrder ?? -1,
            HasVerifiedEmail = tx.EmailVerified ? 1 : 0,
            HasVerifiedPhone = tx.PhoneVerified ? 1 : 0,

            // Device features
            IsKnownDevice = tx.IsKnownDevice ? 1 : 0,
            IsProxy = tx.IsProxy ? 1 : 0,
            IsVpn = tx.IsVpn ? 1 : 0,

            // Geo features
            BillingShippingCountryMatch = tx.BillingCountry == tx.ShippingCountry ? 1 : 0,
            IpBillingCountryMatch = tx.IpCountry == tx.BillingCountry ? 1 : 0,
            DistanceBillingShippingKm = tx.BillingShippingDistanceKm,

            // Time features
            HourOfDay = tx.TransactionTime.Hour,
            DayOfWeek = (int)tx.TransactionTime.DayOfWeek,
            IsWeekend = tx.TransactionTime.DayOfWeek
                is DayOfWeek.Saturday or DayOfWeek.Sunday ? 1 : 0,

            // Velocity features
            TransactionsLast1Hour = tx.TransactionsInLastHour,
            TransactionsLast24Hours = tx.TransactionsInLast24Hours,
            UniqueCardsLast7Days = tx.UniqueCardsInLast7Days
        };
    }
}
```

### Integrating ML Model Predictions

```csharp
public class MlFraudScoringService
{
    private readonly HttpClient _mlServiceClient;
    private readonly ComprehensiveFraudEngine _ruleEngine;
    private readonly ILogger<MlFraudScoringService> _logger;

    /// <summary>
    /// Combines rule-based and ML scoring for a final fraud decision.
    /// The ML model runs as a separate microservice.
    /// </summary>
    public async Task<FraudAssessment> ScoreAsync(
        TransactionContext tx, HttpContext httpContext)
    {
        // 1. Rule-based scoring (always runs)
        var ruleAssessment = await _ruleEngine.EvaluateAsync(
            tx.PaymentRequest, httpContext);

        // 2. ML scoring (best-effort — fall back to rules if unavailable)
        double? mlScore = null;
        try
        {
            var features = new FraudFeatureExtractor().Extract(tx);
            var response = await _mlServiceClient.PostAsJsonAsync(
                "/predict", features);
            response.EnsureSuccessStatusCode();

            var prediction = await response.Content
                .ReadFromJsonAsync<MlPrediction>();
            mlScore = prediction?.FraudProbability * 100;
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex,
                "ML fraud service unavailable — using rules only");
        }

        // 3. Ensemble — weighted combination
        var finalScore = mlScore.HasValue
            ? (ruleAssessment.Score * 0.4) + (mlScore.Value * 0.6)
            : ruleAssessment.Score;

        return ruleAssessment with
        {
            Score = Math.Min(finalScore, 100),
            Level = ClassifyRisk(finalScore),
            Decision = DetermineAction(finalScore)
        };
    }
}
```

---

## 11. Provider-Specific Fraud Tools

### Stripe Radar

Stripe Radar is a built-in ML-powered fraud detection tool:

```csharp
public class StripeRadarIntegration
{
    /// <summary>
    /// Creates a PaymentIntent with Stripe Radar risk evaluation.
    /// Stripe automatically scores every payment through Radar.
    /// </summary>
    public async Task<PaymentResult> ProcessWithRadarAsync(
        PaymentRequest request, PaymentIntentService service)
    {
        var options = new PaymentIntentCreateOptions
        {
            Amount = (long)(request.Amount * 100),
            Currency = request.Currency,
            PaymentMethod = request.PaymentMethodId,
            Confirm = true,
            // Send additional data to improve Radar accuracy
            Shipping = new ChargeShippingOptions
            {
                Name = request.ShippingName,
                Address = new AddressOptions
                {
                    Line1 = request.ShippingLine1,
                    City = request.ShippingCity,
                    State = request.ShippingState,
                    PostalCode = request.ShippingPostalCode,
                    Country = request.ShippingCountry
                }
            },
            Metadata = new Dictionary<string, string>
            {
                ["customer_email"] = request.CustomerEmail,
                ["customer_account_age_days"] =
                    request.AccountAgeDays.ToString(),
                ["order_item_count"] = request.ItemCount.ToString()
            }
        };

        var intent = await service.CreateAsync(options);

        // Access Radar's risk evaluation
        var outcome = intent.LatestCharge?.Outcome;

        return new PaymentResult
        {
            IntentId = intent.Id,
            Status = intent.Status,
            RadarRiskLevel = outcome?.RiskLevel,  // "normal", "elevated", "highest"
            RadarRiskScore = outcome?.RiskScore,  // 0–100
            RadarRule = outcome?.Rule?.Id          // if a custom Radar rule triggered
        };
    }
}
```

### PayPal Fraud Protection

```csharp
public class PayPalFraudProtection
{
    /// <summary>
    /// Configures PayPal order with fraud protection signals.
    /// PayPal uses its own risk engine to evaluate transactions.
    /// </summary>
    public OrderRequest BuildOrderWithFraudSignals(
        PaymentRequest request, DeviceFingerprint device)
    {
        return new OrderRequest
        {
            Intent = "CAPTURE",
            PurchaseUnits = new List<PurchaseUnitRequest>
            {
                new()
                {
                    Amount = new AmountWithBreakdown
                    {
                        Value = request.Amount.ToString("F2"),
                        CurrencyCode = request.Currency
                    },
                    Shipping = new ShippingDetail
                    {
                        Name = new Name { FullName = request.ShippingName },
                        AddressPortable = new AddressPortable
                        {
                            AddressLine1 = request.ShippingLine1,
                            AdminArea2 = request.ShippingCity,
                            AdminArea1 = request.ShippingState,
                            PostalCode = request.ShippingPostalCode,
                            CountryCode = request.ShippingCountry
                        }
                    }
                }
            },
            // PayPal Risk — provide client-side device data token
            ApplicationContext = new ApplicationContext
            {
                ShippingPreference = "SET_PROVIDED_ADDRESS",
                UserAction = "PAY_NOW"
            }
        };
    }
}
```

### Third-Party Fraud Services

| Service | Strengths | Best For |
|---------|-----------|----------|
| **Stripe Radar** | Built-in, no integration overhead | Stripe-only merchants |
| **Signifyd** | Guaranteed fraud protection, chargeback insurance | Large retailers |
| **Sift** | Real-time ML, account protection, content integrity | Marketplaces, platforms |
| **Kount (Equifax)** | Identity trust, device intelligence | Enterprise merchants |
| **Riskified** | Guaranteed approval, chargeback guarantee | High-volume e-commerce |
| **Forter** | Automated decisioning, identity-based | Omnichannel retailers |
| **ClearSale** | High approval rates, manual review included | LATAM markets |

---

## 12. Implementation Checklist

### Phase 1: Foundation (Week 1–2)

- [ ] Implement AVS response code evaluation
- [ ] Implement CVV response code evaluation
- [ ] Create AVS + CVV combined decision matrix
- [ ] Add input validation for addresses (FluentValidation)
- [ ] Integrate an address validation API for shipping addresses

### Phase 2: Customer Verification (Week 3)

- [ ] Implement email verification flow
- [ ] Implement phone/SMS verification flow
- [ ] Define KYC level tiers and policy engine
- [ ] Integrate 3DS 2.0 for card payments
- [ ] Set up identity document verification for high-risk actions

### Phase 3: Device and Behavioral Analysis (Week 4)

- [ ] Collect device fingerprint data on checkout
- [ ] Implement IP reputation checking
- [ ] Add behavioral analysis (time-on-site, cart composition)
- [ ] Configure e-commerce-specific velocity rules

### Phase 4: Risk Scoring and Review (Week 5–6)

- [ ] Build multi-signal fraud scoring engine
- [ ] Implement automatic accept/decline thresholds
- [ ] Create manual review queue and dashboard
- [ ] Set up fraud team notification workflow
- [ ] Implement order hold and release process

### Phase 5: Chargeback Prevention (Week 6–7)

- [ ] Configure clear billing descriptors
- [ ] Implement pre-billing notifications for subscriptions
- [ ] Automate dispute evidence gathering
- [ ] Track and monitor chargeback ratio
- [ ] Set up alerts when chargeback ratio exceeds 0.5%

### Phase 6: ML and Optimization (Ongoing)

- [ ] Evaluate provider-specific fraud tools (Stripe Radar, etc.)
- [ ] Collect labeled fraud/legitimate transaction data
- [ ] Train and deploy ML fraud model
- [ ] Implement ensemble scoring (rules + ML)
- [ ] Continuously retrain model with new data
- [ ] A/B test scoring thresholds to optimize approval vs. fraud rates

---

## Additional Resources

- **PCI Security Standards Council** — https://www.pcisecuritystandards.org/
- **EMVCo 3D Secure** — https://www.emvco.com/emv-technologies/3d-secure/
- **NIST Digital Identity Guidelines (SP 800-63)** — https://pages.nist.gov/800-63-3/
- **Stripe Radar Documentation** — https://stripe.com/docs/radar
- **OWASP Fraud Prevention Cheat Sheet** — https://cheatsheetseries.owasp.org/cheatsheets/Transaction_Authorization_Cheat_Sheet.html

---

> **Next Steps:**
>
> - Review [15-SECURITY-PATTERNS-PRACTICES](15-SECURITY-PATTERNS-PRACTICES.md) for PCI DSS, encryption, and API security.
> - See [18-COMMON-SCENARIOS](18-COMMON-SCENARIOS.md) for end-to-end checkout flows.
> - See [24-PAYMENT-METHODS-AND-FLOWS](24-PAYMENT-METHODS-AND-FLOWS.md) for payment method details.
