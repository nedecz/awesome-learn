# Data Privacy and Compliance

## Overview

Data privacy and regulatory compliance are critical requirements for any e-commerce or payment system handling personal information. Organizations must navigate a complex landscape of regulations including GDPR, CCPA/CPRA, PCI DSS, and industry-specific requirements while maintaining seamless customer experiences. This document covers privacy data modeling, consent management, data subject rights, retention policies, payment data privacy, and privacy-by-design patterns in .NET — providing production-ready implementations for building compliant commerce systems.

> **Related documents:**
>
> - [15-SECURITY-PATTERNS-PRACTICES](15-SECURITY-PATTERNS-PRACTICES.md) — PCI DSS, encryption, and security architecture
> - [06-DATABASE-DESIGN](06-DATABASE-DESIGN.md) — Database schemas and data modeling
> - [19-MONITORING-ALERTING](19-MONITORING-ALERTING.md) — Logging, observability, and audit trails
> - [28-ADDRESS-VERIFICATION-FRAUD-PREVENTION](28-ADDRESS-VERIFICATION-FRAUD-PREVENTION.md) — KYC data handling and identity verification

## Table of Contents

1. [Privacy Data Model](#1-privacy-data-model)
2. [GDPR Compliance](#2-gdpr-compliance)
3. [CCPA/CPRA Compliance](#3-ccpacpra-compliance)
4. [Consent Management Platform](#4-consent-management-platform)
5. [Right to Erasure (Right to Be Forgotten)](#5-right-to-erasure-right-to-be-forgotten)
6. [Data Retention Policies](#6-data-retention-policies)
7. [Data Subject Access Requests (DSAR)](#7-data-subject-access-requests-dsar)
8. [Payment Data Privacy](#8-payment-data-privacy)
9. [Data Minimization & Purpose Limitation](#9-data-minimization--purpose-limitation)
10. [Privacy by Design in .NET](#10-privacy-by-design-in-net)
11. [Implementation Checklist](#11-implementation-checklist)

---

## 1. Privacy Data Model

Effective privacy compliance starts with a well-designed data model that captures consent records, data subject requests, and a complete inventory of personal data across systems. Classification of data by sensitivity level enables automated policy enforcement.

### Core Entities

```csharp
public enum DataClassification
{
    Public = 0,
    Internal = 1,
    Confidential = 2,
    Restricted = 3,       // PII: names, emails, addresses
    HighlyRestricted = 4  // PCI, SSN, health data
}

public enum LawfulBasis
{
    Consent,
    Contract,
    LegalObligation,
    VitalInterest,
    PublicTask,
    LegitimateInterest
}

public enum ConsentStatus
{
    Granted,
    Denied,
    Withdrawn,
    Expired
}

public enum DsarRequestType
{
    Access,
    Erasure,
    Rectification,
    Portability,
    Restriction,
    Objection
}

public enum DsarRequestStatus
{
    Received,
    IdentityVerificationPending,
    InProgress,
    AwaitingThirdParty,
    OnLegalHold,
    Completed,
    Denied
}
```

### ConsentRecord Entity

```csharp
public class ConsentRecord
{
    public Guid Id { get; private set; }
    public Guid DataSubjectId { get; private set; }
    public string ConsentCategory { get; private set; } = default!;
    public string Purpose { get; private set; } = default!;
    public LawfulBasis LawfulBasis { get; private set; }
    public ConsentStatus Status { get; private set; }
    public string ConsentVersion { get; private set; } = default!;
    public string PolicyVersion { get; private set; } = default!;
    public string CollectionMethod { get; private set; } = default!;
    public string? IpAddress { get; private set; }
    public string? UserAgent { get; private set; }
    public DateTime GrantedAtUtc { get; private set; }
    public DateTime? WithdrawnAtUtc { get; private set; }
    public DateTime? ExpiresAtUtc { get; private set; }
    public string? ProofOfConsentHash { get; private set; }

    private ConsentRecord() { }

    public static ConsentRecord Grant(
        Guid dataSubjectId,
        string category,
        string purpose,
        LawfulBasis lawfulBasis,
        string consentVersion,
        string policyVersion,
        string collectionMethod,
        string? ipAddress = null,
        string? userAgent = null,
        DateTime? expiresAtUtc = null)
    {
        return new ConsentRecord
        {
            Id = Guid.NewGuid(),
            DataSubjectId = dataSubjectId,
            ConsentCategory = category,
            Purpose = purpose,
            LawfulBasis = lawfulBasis,
            Status = ConsentStatus.Granted,
            ConsentVersion = consentVersion,
            PolicyVersion = policyVersion,
            CollectionMethod = collectionMethod,
            IpAddress = ipAddress,
            UserAgent = userAgent,
            GrantedAtUtc = DateTime.UtcNow,
            ExpiresAtUtc = expiresAtUtc
        };
    }

    public void Withdraw()
    {
        Status = ConsentStatus.Withdrawn;
        WithdrawnAtUtc = DateTime.UtcNow;
    }
}
```

### DataSubjectRequest Entity

```csharp
public class DataSubjectRequest
{
    public Guid Id { get; private set; }
    public Guid DataSubjectId { get; private set; }
    public DsarRequestType RequestType { get; private set; }
    public DsarRequestStatus Status { get; private set; }
    public string RequestDetails { get; private set; } = default!;
    public string? VerificationMethod { get; private set; }
    public bool IdentityVerified { get; private set; }
    public DateTime ReceivedAtUtc { get; private set; }
    public DateTime DeadlineUtc { get; private set; }
    public DateTime? CompletedAtUtc { get; private set; }
    public string? ResponseSummary { get; private set; }
    public string? DenialReason { get; private set; }
    public string Jurisdiction { get; private set; } = default!;

    private readonly List<DsarAuditEntry> _auditTrail = new();
    public IReadOnlyCollection<DsarAuditEntry> AuditTrail => _auditTrail.AsReadOnly();

    private DataSubjectRequest() { }

    public static DataSubjectRequest Create(
        Guid dataSubjectId,
        DsarRequestType requestType,
        string details,
        string jurisdiction,
        int slaBusinessDays = 30)
    {
        return new DataSubjectRequest
        {
            Id = Guid.NewGuid(),
            DataSubjectId = dataSubjectId,
            RequestType = requestType,
            Status = DsarRequestStatus.Received,
            RequestDetails = details,
            Jurisdiction = jurisdiction,
            ReceivedAtUtc = DateTime.UtcNow,
            DeadlineUtc = CalculateDeadline(DateTime.UtcNow, slaBusinessDays)
        };
    }

    public void VerifyIdentity(string method)
    {
        VerificationMethod = method;
        IdentityVerified = true;
        Status = DsarRequestStatus.InProgress;
        _auditTrail.Add(new DsarAuditEntry(Id, "Identity verified", method));
    }

    public void Complete(string responseSummary)
    {
        Status = DsarRequestStatus.Completed;
        CompletedAtUtc = DateTime.UtcNow;
        ResponseSummary = responseSummary;
        _auditTrail.Add(new DsarAuditEntry(Id, "Request completed", responseSummary));
    }

    public bool IsOverdue => DateTime.UtcNow > DeadlineUtc
                             && Status != DsarRequestStatus.Completed
                             && Status != DsarRequestStatus.Denied;

    private static DateTime CalculateDeadline(DateTime start, int businessDays)
    {
        var deadline = start;
        var daysAdded = 0;
        while (daysAdded < businessDays)
        {
            deadline = deadline.AddDays(1);
            if (deadline.DayOfWeek != DayOfWeek.Saturday &&
                deadline.DayOfWeek != DayOfWeek.Sunday)
                daysAdded++;
        }
        return deadline;
    }
}

public class DsarAuditEntry
{
    public Guid Id { get; private set; }
    public Guid RequestId { get; private set; }
    public string Action { get; private set; } = default!;
    public string Details { get; private set; } = default!;
    public DateTime TimestampUtc { get; private set; }

    public DsarAuditEntry(Guid requestId, string action, string details)
    {
        Id = Guid.NewGuid();
        RequestId = requestId;
        Action = action;
        Details = details;
        TimestampUtc = DateTime.UtcNow;
    }

    private DsarAuditEntry() { }
}
```

### PersonalDataInventory Entity

```csharp
public class PersonalDataInventory
{
    public Guid Id { get; private set; }
    public string SystemName { get; private set; } = default!;
    public string DataCategory { get; private set; } = default!;
    public string DataElements { get; private set; } = default!;
    public DataClassification Classification { get; private set; }
    public LawfulBasis LawfulBasis { get; private set; }
    public string Purpose { get; private set; } = default!;
    public string DataController { get; private set; } = default!;
    public string? DataProcessor { get; private set; }
    public string StorageLocation { get; private set; } = default!;
    public string RetentionPeriod { get; private set; } = default!;
    public bool CrossBorderTransfer { get; private set; }
    public string? TransferMechanism { get; private set; }
    public DateTime LastReviewedUtc { get; private set; }

    public static PersonalDataInventory Register(
        string systemName,
        string dataCategory,
        string dataElements,
        DataClassification classification,
        LawfulBasis lawfulBasis,
        string purpose,
        string controller,
        string storageLocation,
        string retentionPeriod)
    {
        return new PersonalDataInventory
        {
            Id = Guid.NewGuid(),
            SystemName = systemName,
            DataCategory = dataCategory,
            DataElements = dataElements,
            Classification = classification,
            LawfulBasis = lawfulBasis,
            Purpose = purpose,
            DataController = controller,
            StorageLocation = storageLocation,
            RetentionPeriod = retentionPeriod,
            LastReviewedUtc = DateTime.UtcNow
        };
    }
}
```

### EF Core Configuration

```csharp
public class PrivacyDbContext : DbContext
{
    public DbSet<ConsentRecord> ConsentRecords => Set<ConsentRecord>();
    public DbSet<DataSubjectRequest> DataSubjectRequests => Set<DataSubjectRequest>();
    public DbSet<PersonalDataInventory> PersonalDataInventories => Set<PersonalDataInventory>();

    public PrivacyDbContext(DbContextOptions<PrivacyDbContext> options)
        : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<ConsentRecord>(entity =>
        {
            entity.ToTable("consent_records");
            entity.HasKey(e => e.Id);
            entity.Property(e => e.ConsentCategory).HasMaxLength(100).IsRequired();
            entity.Property(e => e.Purpose).HasMaxLength(500).IsRequired();
            entity.Property(e => e.LawfulBasis).HasConversion<string>().HasMaxLength(50);
            entity.Property(e => e.Status).HasConversion<string>().HasMaxLength(20);
            entity.Property(e => e.ConsentVersion).HasMaxLength(20);
            entity.Property(e => e.PolicyVersion).HasMaxLength(20);
            entity.Property(e => e.CollectionMethod).HasMaxLength(50);
            entity.Property(e => e.IpAddress).HasMaxLength(45);
            entity.Property(e => e.ProofOfConsentHash).HasMaxLength(128);

            entity.HasIndex(e => e.DataSubjectId);
            entity.HasIndex(e => new { e.DataSubjectId, e.ConsentCategory })
                  .HasDatabaseName("ix_consent_subject_category");
        });

        modelBuilder.Entity<DataSubjectRequest>(entity =>
        {
            entity.ToTable("data_subject_requests");
            entity.HasKey(e => e.Id);
            entity.Property(e => e.RequestType).HasConversion<string>().HasMaxLength(30);
            entity.Property(e => e.Status).HasConversion<string>().HasMaxLength(40);
            entity.Property(e => e.Jurisdiction).HasMaxLength(10).IsRequired();
            entity.HasMany(e => e.AuditTrail)
                  .WithOne()
                  .HasForeignKey(a => a.RequestId)
                  .OnDelete(DeleteBehavior.Cascade);

            entity.HasIndex(e => e.DataSubjectId);
            entity.HasIndex(e => e.Status);
            entity.HasIndex(e => e.DeadlineUtc);
        });

        modelBuilder.Entity<DsarAuditEntry>(entity =>
        {
            entity.ToTable("dsar_audit_entries");
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Action).HasMaxLength(200);
            entity.Property(e => e.Details).HasMaxLength(2000);
        });

        modelBuilder.Entity<PersonalDataInventory>(entity =>
        {
            entity.ToTable("personal_data_inventory");
            entity.HasKey(e => e.Id);
            entity.Property(e => e.SystemName).HasMaxLength(200).IsRequired();
            entity.Property(e => e.Classification).HasConversion<string>().HasMaxLength(30);
            entity.Property(e => e.LawfulBasis).HasConversion<string>().HasMaxLength(50);
            entity.Property(e => e.RetentionPeriod).HasMaxLength(50);
            entity.HasIndex(e => e.Classification);
        });
    }
}
```

---

## 2. GDPR Compliance

The General Data Protection Regulation (GDPR) applies to any organization processing personal data of EU/EEA residents. Commerce systems must implement lawful basis tracking, consent management, data processing agreements, and cross-border transfer mechanisms.

### Lawful Basis Service

```csharp
public interface ILawfulBasisService
{
    Task<bool> HasValidBasisAsync(Guid dataSubjectId, string purpose);
    Task RecordBasisAsync(Guid dataSubjectId, string purpose, LawfulBasis basis);
    Task<IReadOnlyList<LawfulBasisRecord>> GetBasisHistoryAsync(Guid dataSubjectId);
}

public class LawfulBasisRecord
{
    public Guid Id { get; init; }
    public Guid DataSubjectId { get; init; }
    public string Purpose { get; init; } = default!;
    public LawfulBasis Basis { get; init; }
    public string Justification { get; init; } = default!;
    public DateTime RecordedAtUtc { get; init; }
    public DateTime? ReviewDueUtc { get; init; }
}

public class GdprComplianceService : ILawfulBasisService
{
    private readonly PrivacyDbContext _db;
    private readonly ILogger<GdprComplianceService> _logger;

    public GdprComplianceService(PrivacyDbContext db, ILogger<GdprComplianceService> logger)
    {
        _db = db;
        _logger = logger;
    }

    public async Task<bool> HasValidBasisAsync(Guid dataSubjectId, string purpose)
    {
        var consent = await _db.ConsentRecords
            .Where(c => c.DataSubjectId == dataSubjectId
                        && c.Purpose == purpose
                        && c.Status == ConsentStatus.Granted
                        && (c.ExpiresAtUtc == null || c.ExpiresAtUtc > DateTime.UtcNow))
            .AnyAsync();

        return consent;
    }

    public async Task RecordBasisAsync(
        Guid dataSubjectId, string purpose, LawfulBasis basis)
    {
        var record = ConsentRecord.Grant(
            dataSubjectId,
            category: "gdpr-lawful-basis",
            purpose: purpose,
            lawfulBasis: basis,
            consentVersion: "1.0",
            policyVersion: "2024-01",
            collectionMethod: "system-recorded");

        _db.ConsentRecords.Add(record);
        await _db.SaveChangesAsync();

        _logger.LogInformation(
            "Recorded lawful basis {Basis} for subject {SubjectId}, purpose: {Purpose}",
            basis, dataSubjectId, purpose);
    }

    public async Task<IReadOnlyList<LawfulBasisRecord>> GetBasisHistoryAsync(
        Guid dataSubjectId)
    {
        return await _db.ConsentRecords
            .Where(c => c.DataSubjectId == dataSubjectId)
            .Select(c => new LawfulBasisRecord
            {
                Id = c.Id,
                DataSubjectId = c.DataSubjectId,
                Purpose = c.Purpose,
                Basis = c.LawfulBasis,
                Justification = c.CollectionMethod,
                RecordedAtUtc = c.GrantedAtUtc
            })
            .OrderByDescending(r => r.RecordedAtUtc)
            .ToListAsync();
    }
}
```

### Cross-Border Transfer Validator

```csharp
public enum TransferMechanism
{
    AdequacyDecision,
    StandardContractualClauses,
    BindingCorporateRules,
    ExplicitConsent,
    Derogation
}

public class CrossBorderTransferValidator
{
    private static readonly HashSet<string> AdequateCountries = new(StringComparer.OrdinalIgnoreCase)
    {
        "Andorra", "Argentina", "Canada", "Faroe Islands", "Guernsey",
        "Israel", "Isle of Man", "Japan", "Jersey", "New Zealand",
        "Republic of Korea", "Switzerland", "United Kingdom", "Uruguay"
    };

    public TransferValidationResult Validate(
        string sourceCountry, string destinationCountry, TransferMechanism mechanism)
    {
        if (IsEeaCountry(sourceCountry) && IsEeaCountry(destinationCountry))
            return TransferValidationResult.Allowed("Intra-EEA transfer");

        if (AdequateCountries.Contains(destinationCountry))
            return TransferValidationResult.Allowed(
                $"Adequacy decision exists for {destinationCountry}");

        return mechanism switch
        {
            TransferMechanism.StandardContractualClauses =>
                TransferValidationResult.AllowedWithConditions(
                    "SCCs required — ensure transfer impact assessment is completed"),
            TransferMechanism.BindingCorporateRules =>
                TransferValidationResult.AllowedWithConditions(
                    "BCRs must be approved by lead supervisory authority"),
            TransferMechanism.ExplicitConsent =>
                TransferValidationResult.AllowedWithConditions(
                    "Explicit, informed consent required — document risks disclosed"),
            _ => TransferValidationResult.Denied(
                $"No valid transfer mechanism for {sourceCountry} → {destinationCountry}")
        };
    }

    private static bool IsEeaCountry(string country) =>
        EeaCountries.Contains(country, StringComparer.OrdinalIgnoreCase);

    private static readonly string[] EeaCountries =
    {
        "Austria", "Belgium", "Bulgaria", "Croatia", "Cyprus", "Czech Republic",
        "Denmark", "Estonia", "Finland", "France", "Germany", "Greece", "Hungary",
        "Iceland", "Ireland", "Italy", "Latvia", "Liechtenstein", "Lithuania",
        "Luxembourg", "Malta", "Netherlands", "Norway", "Poland", "Portugal",
        "Romania", "Slovakia", "Slovenia", "Spain", "Sweden"
    };
}

public record TransferValidationResult(
    bool IsAllowed, bool HasConditions, string Reason)
{
    public static TransferValidationResult Allowed(string reason) =>
        new(true, false, reason);
    public static TransferValidationResult AllowedWithConditions(string reason) =>
        new(true, true, reason);
    public static TransferValidationResult Denied(string reason) =>
        new(false, false, reason);
}
```

---

## 3. CCPA/CPRA Compliance

The California Consumer Privacy Act (CCPA) and its amendment, the California Privacy Rights Act (CPRA), grant California consumers specific rights over their personal information. Commerce systems must support the rights to know, delete, opt-out of sale, and correct personal information.

### Consumer Rights Service

```csharp
public interface ICcpaConsumerRightsService
{
    Task<CcpaDisclosureReport> HandleRightToKnowAsync(Guid consumerId);
    Task<DeletionResult> HandleRightToDeleteAsync(Guid consumerId);
    Task HandleDoNotSellAsync(Guid consumerId);
    Task HandleRightToCorrectAsync(Guid consumerId, Dictionary<string, string> corrections);
    Task<bool> HasOptedOutOfSaleAsync(Guid consumerId);
}

public class CcpaDisclosureReport
{
    public Guid ConsumerId { get; init; }
    public DateTime GeneratedAtUtc { get; init; }
    public List<CollectedDataCategory> CategoriesCollected { get; init; } = new();
    public List<string> SourcesOfCollection { get; init; } = new();
    public List<string> BusinessPurposes { get; init; } = new();
    public List<ThirdPartyDisclosure> ThirdPartyDisclosures { get; init; } = new();
}

public record CollectedDataCategory(string Category, string Examples, string RetentionPeriod);
public record ThirdPartyDisclosure(string ThirdParty, string Category, string Purpose);

public class CcpaComplianceService : ICcpaConsumerRightsService
{
    private readonly PrivacyDbContext _db;
    private readonly IPersonalDataCollector _dataCollector;
    private readonly IDataDeletionService _deletionService;
    private readonly ILogger<CcpaComplianceService> _logger;

    public CcpaComplianceService(
        PrivacyDbContext db,
        IPersonalDataCollector dataCollector,
        IDataDeletionService deletionService,
        ILogger<CcpaComplianceService> logger)
    {
        _db = db;
        _dataCollector = dataCollector;
        _deletionService = deletionService;
        _logger = logger;
    }

    public async Task<CcpaDisclosureReport> HandleRightToKnowAsync(Guid consumerId)
    {
        var personalData = await _dataCollector.CollectAllDataAsync(consumerId);

        var report = new CcpaDisclosureReport
        {
            ConsumerId = consumerId,
            GeneratedAtUtc = DateTime.UtcNow,
            CategoriesCollected = new List<CollectedDataCategory>
            {
                new("Identifiers", "Name, email, account ID", "Duration of account + 3 years"),
                new("Commercial Information", "Purchase history, payment methods", "7 years"),
                new("Internet Activity", "Browsing history, search queries", "90 days"),
                new("Geolocation Data", "Shipping addresses", "Duration of account")
            },
            SourcesOfCollection = new() { "Direct from consumer", "Payment processors", "Analytics" },
            BusinessPurposes = new() { "Order fulfillment", "Customer support", "Fraud prevention" }
        };

        _logger.LogInformation(
            "Generated CCPA disclosure report for consumer {ConsumerId}", consumerId);

        return report;
    }

    public async Task<DeletionResult> HandleRightToDeleteAsync(Guid consumerId)
    {
        var result = await _deletionService.DeleteConsumerDataAsync(consumerId, new DeletionOptions
        {
            PreserveLegalHolds = true,
            PreserveFraudRecords = true,
            PreserveTaxRecords = true,
            NotifyServiceProviders = true
        });

        return result;
    }

    public async Task HandleDoNotSellAsync(Guid consumerId)
    {
        var consent = ConsentRecord.Grant(
            consumerId,
            category: "ccpa-do-not-sell",
            purpose: "Opt-out of sale of personal information",
            lawfulBasis: LawfulBasis.Consent,
            consentVersion: "1.0",
            policyVersion: "2024-01",
            collectionMethod: "consumer-initiated");

        _db.ConsentRecords.Add(consent);
        await _db.SaveChangesAsync();

        _logger.LogInformation(
            "Consumer {ConsumerId} opted out of sale of personal information", consumerId);
    }

    public async Task HandleRightToCorrectAsync(
        Guid consumerId, Dictionary<string, string> corrections)
    {
        // Delegate correction to downstream systems
        foreach (var (field, newValue) in corrections)
        {
            _logger.LogInformation(
                "Correcting {Field} for consumer {ConsumerId}", field, consumerId);
        }

        await _db.SaveChangesAsync();
    }

    public async Task<bool> HasOptedOutOfSaleAsync(Guid consumerId)
    {
        return await _db.ConsentRecords
            .Where(c => c.DataSubjectId == consumerId
                        && c.ConsentCategory == "ccpa-do-not-sell"
                        && c.Status == ConsentStatus.Granted)
            .AnyAsync();
    }
}
```

### Do Not Sell Middleware

```csharp
public class DoNotSellMiddleware
{
    private readonly RequestDelegate _next;

    public DoNotSellMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(
        HttpContext context, ICcpaConsumerRightsService ccpaService)
    {
        if (context.Request.Headers.TryGetValue("Sec-GPC", out var gpcHeader)
            && gpcHeader == "1")
        {
            var consumerId = context.User.FindFirst("sub")?.Value;
            if (consumerId != null && Guid.TryParse(consumerId, out var id))
            {
                var alreadyOptedOut = await ccpaService.HasOptedOutOfSaleAsync(id);
                if (!alreadyOptedOut)
                {
                    await ccpaService.HandleDoNotSellAsync(id);
                }
            }
        }

        await _next(context);
    }
}
```

---

## 4. Consent Management Platform

A robust consent management platform (CMP) captures, stores, and enforces user consent preferences across all data processing activities. This includes cookie consent, marketing preferences, and data sharing opt-ins with full versioning and audit trails.

### Consent Categories and Preferences

```csharp
public static class ConsentCategories
{
    public const string StrictlyNecessary = "strictly_necessary";
    public const string Performance = "performance";
    public const string Functional = "functional";
    public const string Targeting = "targeting";
    public const string SocialMedia = "social_media";
    public const string MarketingEmail = "marketing_email";
    public const string MarketingSms = "marketing_sms";
    public const string ThirdPartySharing = "third_party_sharing";
    public const string Analytics = "analytics";
    public const string Personalization = "personalization";
}

public class ConsentPreference
{
    public string Category { get; init; } = default!;
    public bool IsGranted { get; init; }
    public string? GranularDetail { get; init; }
}

public class ConsentBundle
{
    public Guid DataSubjectId { get; init; }
    public List<ConsentPreference> Preferences { get; init; } = new();
    public string ConsentVersion { get; init; } = default!;
    public string PolicyVersion { get; init; } = default!;
    public string CollectionPoint { get; init; } = default!;
    public string? IpAddress { get; init; }
    public string? UserAgent { get; init; }
}
```

### Consent Management Service

```csharp
public interface IConsentManagementService
{
    Task RecordConsentBundleAsync(ConsentBundle bundle);
    Task WithdrawConsentAsync(Guid dataSubjectId, string category);
    Task<bool> HasConsentAsync(Guid dataSubjectId, string category);
    Task<IReadOnlyList<ConsentRecord>> GetConsentHistoryAsync(Guid dataSubjectId);
    Task<ConsentSummary> GetCurrentPreferencesAsync(Guid dataSubjectId);
}

public class ConsentSummary
{
    public Guid DataSubjectId { get; init; }
    public Dictionary<string, bool> Preferences { get; init; } = new();
    public DateTime LastUpdatedUtc { get; init; }
    public string CurrentPolicyVersion { get; init; } = default!;
}

public class ConsentManagementService : IConsentManagementService
{
    private readonly PrivacyDbContext _db;
    private readonly ILogger<ConsentManagementService> _logger;

    public ConsentManagementService(
        PrivacyDbContext db, ILogger<ConsentManagementService> logger)
    {
        _db = db;
        _logger = logger;
    }

    public async Task RecordConsentBundleAsync(ConsentBundle bundle)
    {
        var records = bundle.Preferences.Select(pref =>
        {
            if (pref.IsGranted)
            {
                return ConsentRecord.Grant(
                    bundle.DataSubjectId,
                    pref.Category,
                    purpose: $"Consent for {pref.Category}",
                    lawfulBasis: LawfulBasis.Consent,
                    consentVersion: bundle.ConsentVersion,
                    policyVersion: bundle.PolicyVersion,
                    collectionMethod: bundle.CollectionPoint,
                    ipAddress: bundle.IpAddress,
                    userAgent: bundle.UserAgent);
            }

            var denied = ConsentRecord.Grant(
                bundle.DataSubjectId,
                pref.Category,
                purpose: $"Consent denied for {pref.Category}",
                lawfulBasis: LawfulBasis.Consent,
                consentVersion: bundle.ConsentVersion,
                policyVersion: bundle.PolicyVersion,
                collectionMethod: bundle.CollectionPoint);
            denied.Withdraw();
            return denied;
        });

        _db.ConsentRecords.AddRange(records);
        await _db.SaveChangesAsync();
    }

    public async Task WithdrawConsentAsync(Guid dataSubjectId, string category)
    {
        var activeConsent = await _db.ConsentRecords
            .Where(c => c.DataSubjectId == dataSubjectId
                        && c.ConsentCategory == category
                        && c.Status == ConsentStatus.Granted)
            .FirstOrDefaultAsync();

        if (activeConsent != null)
        {
            activeConsent.Withdraw();
            await _db.SaveChangesAsync();
            _logger.LogInformation(
                "Consent withdrawn for subject {SubjectId}, category {Category}",
                dataSubjectId, category);
        }
    }

    public async Task<bool> HasConsentAsync(Guid dataSubjectId, string category)
    {
        if (category == ConsentCategories.StrictlyNecessary)
            return true;

        return await _db.ConsentRecords
            .Where(c => c.DataSubjectId == dataSubjectId
                        && c.ConsentCategory == category
                        && c.Status == ConsentStatus.Granted
                        && (c.ExpiresAtUtc == null || c.ExpiresAtUtc > DateTime.UtcNow))
            .AnyAsync();
    }

    public async Task<IReadOnlyList<ConsentRecord>> GetConsentHistoryAsync(
        Guid dataSubjectId)
    {
        return await _db.ConsentRecords
            .Where(c => c.DataSubjectId == dataSubjectId)
            .OrderByDescending(c => c.GrantedAtUtc)
            .ToListAsync();
    }

    public async Task<ConsentSummary> GetCurrentPreferencesAsync(Guid dataSubjectId)
    {
        var latestPerCategory = await _db.ConsentRecords
            .Where(c => c.DataSubjectId == dataSubjectId)
            .GroupBy(c => c.ConsentCategory)
            .Select(g => g.OrderByDescending(c => c.GrantedAtUtc).First())
            .ToListAsync();

        return new ConsentSummary
        {
            DataSubjectId = dataSubjectId,
            Preferences = latestPerCategory.ToDictionary(
                c => c.ConsentCategory,
                c => c.Status == ConsentStatus.Granted),
            LastUpdatedUtc = latestPerCategory.Max(c => c.GrantedAtUtc),
            CurrentPolicyVersion = latestPerCategory
                .OrderByDescending(c => c.GrantedAtUtc)
                .First().PolicyVersion
        };
    }
}
```

### Consent Enforcement Filter

```csharp
[AttributeUsage(AttributeTargets.Method | AttributeTargets.Class)]
public class RequiresConsentAttribute : Attribute, IAsyncActionFilter
{
    public string Category { get; }

    public RequiresConsentAttribute(string category)
    {
        Category = category;
    }

    public async Task OnActionExecutionAsync(
        ActionExecutingContext context, ActionExecutionDelegate next)
    {
        var consentService = context.HttpContext.RequestServices
            .GetRequiredService<IConsentManagementService>();

        var userId = context.HttpContext.User.FindFirst("sub")?.Value;
        if (userId == null || !Guid.TryParse(userId, out var subjectId))
        {
            context.Result = new ForbidResult();
            return;
        }

        var hasConsent = await consentService.HasConsentAsync(subjectId, Category);
        if (!hasConsent)
        {
            context.Result = new ObjectResult(new
            {
                Error = "ConsentRequired",
                Message = $"User consent for '{Category}' is required.",
                ConsentUrl = "/api/consent/preferences"
            })
            { StatusCode = 451 }; // Unavailable For Legal Reasons
            return;
        }

        await next();
    }
}

// Usage
[ApiController]
[Route("api/[controller]")]
public class RecommendationsController : ControllerBase
{
    [HttpGet]
    [RequiresConsent(ConsentCategories.Personalization)]
    public async Task<IActionResult> GetPersonalizedRecommendations()
    {
        // Only executes if user has granted personalization consent
        return Ok(new { recommendations = new[] { "Product A", "Product B" } });
    }
}
```

---

## 5. Right to Erasure (Right to Be Forgotten)

The right to erasure requires organizations to delete personal data upon request, subject to legal exceptions. Implementation must handle data discovery across systems, cascading deletions, anonymization alternatives, and third-party notification.

### Data Deletion Orchestrator

```csharp
public class DeletionOptions
{
    public bool PreserveLegalHolds { get; set; } = true;
    public bool PreserveFraudRecords { get; set; } = true;
    public bool PreserveTaxRecords { get; set; } = true;
    public bool NotifyServiceProviders { get; set; } = true;
    public bool AnonymizeInsteadOfDelete { get; set; }
}

public class DeletionResult
{
    public Guid DataSubjectId { get; init; }
    public bool IsComplete { get; init; }
    public List<DeletionSystemResult> SystemResults { get; init; } = new();
    public List<string> RetainedDataReasons { get; init; } = new();
    public DateTime ProcessedAtUtc { get; init; }
}

public record DeletionSystemResult(
    string SystemName, bool Success, int RecordsAffected, string? Reason);

public interface IDataDeletionService
{
    Task<DeletionResult> DeleteConsumerDataAsync(
        Guid dataSubjectId, DeletionOptions options);
}

public interface IDataDeletionHandler
{
    string SystemName { get; }
    Task<DeletionSystemResult> DeleteAsync(Guid dataSubjectId, DeletionOptions options);
}

public class DataDeletionOrchestrator : IDataDeletionService
{
    private readonly IEnumerable<IDataDeletionHandler> _handlers;
    private readonly ILegalHoldService _legalHoldService;
    private readonly ILogger<DataDeletionOrchestrator> _logger;

    public DataDeletionOrchestrator(
        IEnumerable<IDataDeletionHandler> handlers,
        ILegalHoldService legalHoldService,
        ILogger<DataDeletionOrchestrator> logger)
    {
        _handlers = handlers;
        _legalHoldService = legalHoldService;
        _logger = logger;
    }

    public async Task<DeletionResult> DeleteConsumerDataAsync(
        Guid dataSubjectId, DeletionOptions options)
    {
        var result = new DeletionResult
        {
            DataSubjectId = dataSubjectId,
            ProcessedAtUtc = DateTime.UtcNow
        };

        // Check for legal holds
        var legalHold = await _legalHoldService.HasActiveHoldAsync(dataSubjectId);
        if (legalHold)
        {
            result.RetainedDataReasons.Add("Active legal hold — deletion deferred");
            _logger.LogWarning(
                "Erasure request for {SubjectId} blocked by legal hold", dataSubjectId);
            return result with { IsComplete = false };
        }

        // Execute deletion across all registered systems
        var tasks = _handlers.Select(async handler =>
        {
            try
            {
                return await handler.DeleteAsync(dataSubjectId, options);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex,
                    "Deletion failed in {System} for {SubjectId}",
                    handler.SystemName, dataSubjectId);
                return new DeletionSystemResult(
                    handler.SystemName, false, 0, ex.Message);
            }
        });

        var systemResults = await Task.WhenAll(tasks);
        result.SystemResults.AddRange(systemResults);

        return result with { IsComplete = systemResults.All(r => r.Success) };
    }
}
```

### Anonymization Handler

```csharp
public class CustomerDatabaseDeletionHandler : IDataDeletionHandler
{
    public string SystemName => "CustomerDatabase";
    private readonly PrivacyDbContext _db;

    public CustomerDatabaseDeletionHandler(PrivacyDbContext db) => _db = db;

    public async Task<DeletionSystemResult> DeleteAsync(
        Guid dataSubjectId, DeletionOptions options)
    {
        var recordsAffected = 0;

        if (options.AnonymizeInsteadOfDelete)
        {
            recordsAffected = await AnonymizeCustomerAsync(dataSubjectId);
        }
        else
        {
            recordsAffected = await HardDeleteCustomerAsync(dataSubjectId);
        }

        return new DeletionSystemResult(SystemName, true, recordsAffected, null);
    }

    private async Task<int> AnonymizeCustomerAsync(Guid dataSubjectId)
    {
        var anonymizedToken = $"ANON-{Guid.NewGuid():N}";

        // Anonymize customer record while preserving aggregate data
        var affected = await _db.Database.ExecuteSqlInterpolatedAsync($@"
            UPDATE customers SET
                email = {anonymizedToken + "@deleted.example.com"},
                first_name = 'REDACTED',
                last_name = 'REDACTED',
                phone = NULL,
                date_of_birth = NULL,
                ip_address = NULL,
                is_anonymized = 1,
                anonymized_at_utc = {DateTime.UtcNow}
            WHERE id = {dataSubjectId}");

        return affected;
    }

    private async Task<int> HardDeleteCustomerAsync(Guid dataSubjectId)
    {
        var affected = await _db.Database.ExecuteSqlInterpolatedAsync(
            $"DELETE FROM customers WHERE id = {dataSubjectId}");
        return affected;
    }
}
```

### Legal Hold Service

```csharp
public interface ILegalHoldService
{
    Task<bool> HasActiveHoldAsync(Guid dataSubjectId);
    Task PlaceHoldAsync(Guid dataSubjectId, string caseReference, string reason);
    Task ReleaseHoldAsync(Guid dataSubjectId, string caseReference);
}

public class LegalHoldService : ILegalHoldService
{
    private readonly PrivacyDbContext _db;

    public LegalHoldService(PrivacyDbContext db) => _db = db;

    public async Task<bool> HasActiveHoldAsync(Guid dataSubjectId)
    {
        return await _db.Database
            .SqlQuery<int>(
                $"SELECT COUNT(*) AS Value FROM legal_holds WHERE data_subject_id = {dataSubjectId} AND released_at_utc IS NULL")
            .AnyAsync(c => c > 0);
    }

    public async Task PlaceHoldAsync(
        Guid dataSubjectId, string caseReference, string reason)
    {
        await _db.Database.ExecuteSqlInterpolatedAsync($@"
            INSERT INTO legal_holds (id, data_subject_id, case_reference, reason, placed_at_utc)
            VALUES ({Guid.NewGuid()}, {dataSubjectId}, {caseReference}, {reason}, {DateTime.UtcNow})");
    }

    public async Task ReleaseHoldAsync(Guid dataSubjectId, string caseReference)
    {
        await _db.Database.ExecuteSqlInterpolatedAsync($@"
            UPDATE legal_holds SET released_at_utc = {DateTime.UtcNow}
            WHERE data_subject_id = {dataSubjectId}
            AND case_reference = {caseReference}
            AND released_at_utc IS NULL");
    }
}
```

---

## 6. Data Retention Policies

Data retention policies define how long different categories of personal data may be stored. Automated purging ensures data is deleted when retention periods expire, while regulatory requirements such as tax law and PCI DSS mandate minimum retention for certain records.

### Retention Schedule Configuration

```csharp
public class RetentionPolicy
{
    public string DataCategory { get; init; } = default!;
    public TimeSpan RetentionPeriod { get; init; }
    public string LegalBasis { get; init; } = default!;
    public RetentionAction Action { get; init; }
    public bool RequiresApproval { get; init; }
}

public enum RetentionAction
{
    Delete,
    Anonymize,
    Archive
}

public static class RetentionSchedules
{
    public static readonly IReadOnlyList<RetentionPolicy> Policies = new List<RetentionPolicy>
    {
        new()
        {
            DataCategory = "transaction_records",
            RetentionPeriod = TimeSpan.FromDays(7 * 365), // 7 years
            LegalBasis = "Tax law (IRS) / SOX compliance",
            Action = RetentionAction.Archive
        },
        new()
        {
            DataCategory = "payment_card_data",
            RetentionPeriod = TimeSpan.FromDays(90),
            LegalBasis = "PCI DSS Requirement 3.1",
            Action = RetentionAction.Delete
        },
        new()
        {
            DataCategory = "customer_browsing_history",
            RetentionPeriod = TimeSpan.FromDays(90),
            LegalBasis = "GDPR data minimization",
            Action = RetentionAction.Delete
        },
        new()
        {
            DataCategory = "customer_account_data",
            RetentionPeriod = TimeSpan.FromDays(3 * 365), // 3 years post-closure
            LegalBasis = "Contractual obligation",
            Action = RetentionAction.Anonymize
        },
        new()
        {
            DataCategory = "consent_records",
            RetentionPeriod = TimeSpan.FromDays(7 * 365), // 7 years
            LegalBasis = "Proof of consent / GDPR Art. 7",
            Action = RetentionAction.Archive
        },
        new()
        {
            DataCategory = "audit_logs",
            RetentionPeriod = TimeSpan.FromDays(5 * 365), // 5 years
            LegalBasis = "SOX / internal compliance",
            Action = RetentionAction.Archive
        },
        new()
        {
            DataCategory = "marketing_preferences",
            RetentionPeriod = TimeSpan.FromDays(2 * 365), // 2 years
            LegalBasis = "Legitimate interest review cycle",
            Action = RetentionAction.Delete
        },
        new()
        {
            DataCategory = "fraud_investigation_data",
            RetentionPeriod = TimeSpan.FromDays(10 * 365), // 10 years
            LegalBasis = "Fraud prevention / legal proceedings",
            Action = RetentionAction.Archive
        }
    };
}
```

### Automated Purge Service

```csharp
public interface IDataPurgeService
{
    Task<PurgeReport> ExecutePurgeAsync(CancellationToken cancellationToken = default);
}

public class PurgeReport
{
    public DateTime ExecutedAtUtc { get; init; }
    public List<PurgeCategoryResult> Results { get; init; } = new();
    public int TotalRecordsPurged => Results.Sum(r => r.RecordsPurged);
}

public record PurgeCategoryResult(
    string DataCategory, int RecordsPurged, RetentionAction ActionTaken, string? Error);

public class DataPurgeService : IDataPurgeService
{
    private readonly PrivacyDbContext _db;
    private readonly ILegalHoldService _legalHoldService;
    private readonly ILogger<DataPurgeService> _logger;

    public DataPurgeService(
        PrivacyDbContext db,
        ILegalHoldService legalHoldService,
        ILogger<DataPurgeService> logger)
    {
        _db = db;
        _legalHoldService = legalHoldService;
        _logger = logger;
    }

    public async Task<PurgeReport> ExecutePurgeAsync(
        CancellationToken cancellationToken = default)
    {
        var report = new PurgeReport { ExecutedAtUtc = DateTime.UtcNow };

        foreach (var policy in RetentionSchedules.Policies)
        {
            try
            {
                var cutoffDate = DateTime.UtcNow - policy.RetentionPeriod;
                var recordsPurged = policy.Action switch
                {
                    RetentionAction.Delete =>
                        await PurgeByDeletionAsync(policy.DataCategory, cutoffDate),
                    RetentionAction.Anonymize =>
                        await PurgeByAnonymizationAsync(policy.DataCategory, cutoffDate),
                    RetentionAction.Archive =>
                        await PurgeByArchivalAsync(policy.DataCategory, cutoffDate),
                    _ => 0
                };

                report.Results.Add(new PurgeCategoryResult(
                    policy.DataCategory, recordsPurged, policy.Action, null));

                _logger.LogInformation(
                    "Purged {Count} records from {Category} (action: {Action})",
                    recordsPurged, policy.DataCategory, policy.Action);
            }
            catch (Exception ex)
            {
                report.Results.Add(new PurgeCategoryResult(
                    policy.DataCategory, 0, policy.Action, ex.Message));
                _logger.LogError(ex, "Purge failed for {Category}", policy.DataCategory);
            }
        }

        return report;
    }

    private async Task<int> PurgeByDeletionAsync(string category, DateTime cutoff)
    {
        return await _db.Database.ExecuteSqlInterpolatedAsync(
            $"DELETE FROM {category} WHERE created_at_utc < {cutoff} AND NOT EXISTS (SELECT 1 FROM legal_holds lh WHERE lh.data_subject_id = {category}.subject_id AND lh.released_at_utc IS NULL)");
    }

    private async Task<int> PurgeByAnonymizationAsync(string category, DateTime cutoff)
    {
        return await _db.Database.ExecuteSqlInterpolatedAsync($@"
            UPDATE {category} SET
                email = CONCAT('anon-', id, '@deleted.local'),
                first_name = 'REDACTED', last_name = 'REDACTED',
                phone = NULL, is_anonymized = 1
            WHERE created_at_utc < {cutoff} AND is_anonymized = 0");
    }

    private async Task<int> PurgeByArchivalAsync(string category, DateTime cutoff)
    {
        // Move to cold storage / archive table
        return await _db.Database.ExecuteSqlInterpolatedAsync($@"
            INSERT INTO {category}_archive SELECT * FROM {category}
            WHERE created_at_utc < {cutoff};
            DELETE FROM {category} WHERE created_at_utc < {cutoff}");
    }
}
```

### Retention Purge Background Job

```csharp
public class DataRetentionPurgeJob : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<DataRetentionPurgeJob> _logger;
    private readonly TimeSpan _interval = TimeSpan.FromHours(24);

    public DataRetentionPurgeJob(
        IServiceProvider serviceProvider, ILogger<DataRetentionPurgeJob> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = _serviceProvider.CreateScope();
                var purgeService = scope.ServiceProvider
                    .GetRequiredService<IDataPurgeService>();

                var report = await purgeService.ExecutePurgeAsync(stoppingToken);
                _logger.LogInformation(
                    "Retention purge completed: {Total} records processed",
                    report.TotalRecordsPurged);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Data retention purge job failed");
            }

            await Task.Delay(_interval, stoppingToken);
        }
    }
}
```

---

## 7. Data Subject Access Requests (DSAR)

DSAR processing requires a structured workflow from intake through identity verification, data collection across systems, response formatting, and SLA tracking. GDPR mandates a 30-day response window (extendable to 90 days for complex requests).

### DSAR Workflow Service

```csharp
public interface IDsarWorkflowService
{
    Task<DataSubjectRequest> SubmitRequestAsync(DsarSubmission submission);
    Task VerifyIdentityAsync(Guid requestId, IdentityVerificationResult verification);
    Task<DsarDataPackage> CollectDataAsync(Guid requestId);
    Task CompleteRequestAsync(Guid requestId, DsarDataPackage package);
    Task<IReadOnlyList<DataSubjectRequest>> GetOverdueRequestsAsync();
}

public class DsarSubmission
{
    public string Email { get; init; } = default!;
    public DsarRequestType RequestType { get; init; }
    public string Details { get; init; } = default!;
    public string Jurisdiction { get; init; } = default!;
}

public record IdentityVerificationResult(bool Verified, string Method, string? FailureReason);

public class DsarDataPackage
{
    public Guid RequestId { get; init; }
    public Guid DataSubjectId { get; init; }
    public DateTime GeneratedAtUtc { get; init; }
    public Dictionary<string, object> CollectedData { get; init; } = new();
    public string Format { get; init; } = "JSON"; // JSON or CSV
    public byte[]? ExportFile { get; init; }
}
```

### Personal Data Collector

```csharp
public interface IPersonalDataCollector
{
    Task<Dictionary<string, object>> CollectAllDataAsync(Guid dataSubjectId);
}

public interface ISystemDataProvider
{
    string SystemName { get; }
    Task<object?> GetDataForSubjectAsync(Guid dataSubjectId);
}

public class AggregatingPersonalDataCollector : IPersonalDataCollector
{
    private readonly IEnumerable<ISystemDataProvider> _providers;
    private readonly ILogger<AggregatingPersonalDataCollector> _logger;

    public AggregatingPersonalDataCollector(
        IEnumerable<ISystemDataProvider> providers,
        ILogger<AggregatingPersonalDataCollector> logger)
    {
        _providers = providers;
        _logger = logger;
    }

    public async Task<Dictionary<string, object>> CollectAllDataAsync(Guid dataSubjectId)
    {
        var result = new Dictionary<string, object>();

        var tasks = _providers.Select(async provider =>
        {
            try
            {
                var data = await provider.GetDataForSubjectAsync(dataSubjectId);
                return (provider.SystemName, Data: data);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex,
                    "Failed to collect data from {System} for {SubjectId}",
                    provider.SystemName, dataSubjectId);
                return (provider.SystemName, Data: (object?)new { Error = "Collection failed" });
            }
        });

        var results = await Task.WhenAll(tasks);
        foreach (var (system, data) in results)
        {
            if (data != null)
                result[system] = data;
        }

        return result;
    }
}

// Example data providers
public class OrderSystemDataProvider : ISystemDataProvider
{
    public string SystemName => "Orders";
    private readonly PrivacyDbContext _db;

    public OrderSystemDataProvider(PrivacyDbContext db) => _db = db;

    public async Task<object?> GetDataForSubjectAsync(Guid dataSubjectId)
    {
        return await _db.Database
            .SqlQuery<OrderSummary>(
                $"SELECT order_id, order_date, total, status FROM orders WHERE customer_id = {dataSubjectId}")
            .ToListAsync();
    }
}

public record OrderSummary(string OrderId, DateTime OrderDate, decimal Total, string Status);

public class ConsentDataProvider : ISystemDataProvider
{
    public string SystemName => "ConsentRecords";
    private readonly PrivacyDbContext _db;

    public ConsentDataProvider(PrivacyDbContext db) => _db = db;

    public async Task<object?> GetDataForSubjectAsync(Guid dataSubjectId)
    {
        return await _db.ConsentRecords
            .Where(c => c.DataSubjectId == dataSubjectId)
            .Select(c => new
            {
                c.ConsentCategory,
                c.Purpose,
                Status = c.Status.ToString(),
                c.GrantedAtUtc,
                c.WithdrawnAtUtc
            })
            .ToListAsync();
    }
}
```

### DSAR SLA Monitor

```csharp
public class DsarSlaMonitorJob : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<DsarSlaMonitorJob> _logger;

    public DsarSlaMonitorJob(
        IServiceProvider serviceProvider, ILogger<DsarSlaMonitorJob> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = _serviceProvider.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<PrivacyDbContext>();

            var nearingDeadline = await db.DataSubjectRequests
                .Where(r => r.Status != DsarRequestStatus.Completed
                            && r.Status != DsarRequestStatus.Denied
                            && r.DeadlineUtc <= DateTime.UtcNow.AddDays(5))
                .ToListAsync(stoppingToken);

            foreach (var request in nearingDeadline)
            {
                var daysRemaining = (request.DeadlineUtc - DateTime.UtcNow).Days;
                if (daysRemaining < 0)
                {
                    _logger.LogCritical(
                        "DSAR {RequestId} is OVERDUE by {Days} days! Type: {Type}",
                        request.Id, Math.Abs(daysRemaining), request.RequestType);
                }
                else
                {
                    _logger.LogWarning(
                        "DSAR {RequestId} deadline in {Days} days. Type: {Type}, Status: {Status}",
                        request.Id, daysRemaining, request.RequestType, request.Status);
                }
            }

            await Task.Delay(TimeSpan.FromHours(6), stoppingToken);
        }
    }
}
```

---

## 8. Payment Data Privacy

Payment data privacy focuses on PCI DSS compliance, ensuring cardholder data is properly tokenized, masked, and isolated within a secure cardholder data environment (CDE). Card data must never be logged, and PAN display follows strict masking rules.

### PCI-Compliant Token Vault

```csharp
public interface ITokenVault
{
    Task<string> TokenizeAsync(string pan, Guid customerId);
    Task<string> DetokenizeAsync(string token, string purpose);
    Task<bool> DeleteTokenAsync(string token);
    string MaskPan(string pan);
}

public class TokenVaultService : ITokenVault
{
    private readonly IEncryptionService _encryption;
    private readonly PrivacyDbContext _db;
    private readonly ILogger<TokenVaultService> _logger;

    public TokenVaultService(
        IEncryptionService encryption,
        PrivacyDbContext db,
        ILogger<TokenVaultService> logger)
    {
        _encryption = encryption;
        _db = db;
        _logger = logger;
    }

    public async Task<string> TokenizeAsync(string pan, Guid customerId)
    {
        var token = GenerateToken();
        var encryptedPan = await _encryption.EncryptAsync(pan);

        await _db.Database.ExecuteSqlInterpolatedAsync($@"
            INSERT INTO payment_tokens (token, encrypted_pan, customer_id, created_at_utc)
            VALUES ({token}, {encryptedPan}, {customerId}, {DateTime.UtcNow})");

        // Never log the PAN
        _logger.LogInformation(
            "Tokenized card ending in {Last4} for customer {CustomerId}",
            pan[^4..], customerId);

        return token;
    }

    public async Task<string> DetokenizeAsync(string token, string purpose)
    {
        _logger.LogInformation(
            "Detokenization requested for token {Token}, purpose: {Purpose}",
            token[..8] + "...", purpose);

        var encryptedPan = await _db.Database
            .SqlQuery<string>(
                $"SELECT encrypted_pan AS Value FROM payment_tokens WHERE token = {token}")
            .FirstOrDefaultAsync();

        if (encryptedPan == null)
            throw new InvalidOperationException("Token not found");

        // Log access for PCI audit trail
        await _db.Database.ExecuteSqlInterpolatedAsync($@"
            INSERT INTO token_access_log (token, purpose, accessed_at_utc, accessed_by)
            VALUES ({token}, {purpose}, {DateTime.UtcNow}, {purpose})");

        return await _encryption.DecryptAsync(encryptedPan);
    }

    public async Task<bool> DeleteTokenAsync(string token)
    {
        var affected = await _db.Database.ExecuteSqlInterpolatedAsync(
            $"DELETE FROM payment_tokens WHERE token = {token}");
        return affected > 0;
    }

    public string MaskPan(string pan)
    {
        if (string.IsNullOrEmpty(pan) || pan.Length < 13)
            return "****";

        // PCI DSS: display only first 6 and last 4 digits
        var first6 = pan[..6];
        var last4 = pan[^4..];
        var masked = new string('*', pan.Length - 10);
        return $"{first6}{masked}{last4}";
    }

    private static string GenerateToken()
    {
        var bytes = new byte[16];
        using var rng = System.Security.Cryptography.RandomNumberGenerator.Create();
        rng.GetBytes(bytes);
        return $"tok_{Convert.ToBase64String(bytes).Replace("/", "_").Replace("+", "-").TrimEnd('=')}";
    }
}

public interface IEncryptionService
{
    Task<string> EncryptAsync(string plaintext);
    Task<string> DecryptAsync(string ciphertext);
}
```

### CDE Isolation Middleware

```csharp
public class CdeIsolationMiddleware
{
    private readonly RequestDelegate _next;
    private static readonly HashSet<string> CdeEndpoints = new()
    {
        "/api/payments/tokenize",
        "/api/payments/charge",
        "/api/payments/refund"
    };

    public CdeIsolationMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        var path = context.Request.Path.Value ?? string.Empty;

        if (CdeEndpoints.Any(e => path.StartsWith(e, StringComparison.OrdinalIgnoreCase)))
        {
            // Enforce TLS 1.2+
            if (context.Connection.ClientCertificate == null &&
                !context.Request.IsHttps)
            {
                context.Response.StatusCode = 403;
                await context.Response.WriteAsJsonAsync(new
                {
                    Error = "HTTPS required for payment endpoints"
                });
                return;
            }

            // Add security headers for CDE
            context.Response.Headers.Append("X-Content-Type-Options", "nosniff");
            context.Response.Headers.Append("Cache-Control", "no-store");
            context.Response.Headers.Append("Pragma", "no-cache");
        }

        await _next(context);
    }
}
```

---

## 9. Data Minimization & Purpose Limitation

Data minimization ensures only necessary data is collected for specified purposes. Purpose limitation prevents using data beyond its original collection purpose without re-consent. Both principles reduce exposure surface and simplify compliance.

### Purpose-Bound Data Access

```csharp
public enum DataPurpose
{
    OrderFulfillment,
    PaymentProcessing,
    FraudPrevention,
    CustomerSupport,
    Marketing,
    Analytics,
    LegalCompliance,
    ProductRecommendation
}

[AttributeUsage(AttributeTargets.Property)]
public class DataPurposeAttribute : Attribute
{
    public DataPurpose[] AllowedPurposes { get; }

    public DataPurposeAttribute(params DataPurpose[] purposes)
    {
        AllowedPurposes = purposes;
    }
}

[AttributeUsage(AttributeTargets.Property)]
public class PersonalDataAttribute : Attribute
{
    public DataClassification Classification { get; }

    public PersonalDataAttribute(
        DataClassification classification = DataClassification.Restricted)
    {
        Classification = classification;
    }
}

// Domain model with PII annotations
public class Customer
{
    public Guid Id { get; set; }

    [PersonalData(DataClassification.Restricted)]
    [DataPurpose(DataPurpose.OrderFulfillment, DataPurpose.CustomerSupport)]
    public string FirstName { get; set; } = default!;

    [PersonalData(DataClassification.Restricted)]
    [DataPurpose(DataPurpose.OrderFulfillment, DataPurpose.CustomerSupport)]
    public string LastName { get; set; } = default!;

    [PersonalData(DataClassification.Restricted)]
    [DataPurpose(DataPurpose.OrderFulfillment, DataPurpose.Marketing, DataPurpose.CustomerSupport)]
    public string Email { get; set; } = default!;

    [PersonalData(DataClassification.Restricted)]
    [DataPurpose(DataPurpose.OrderFulfillment)]
    public string? Phone { get; set; }

    [PersonalData(DataClassification.Confidential)]
    [DataPurpose(DataPurpose.Analytics)]
    public DateTime? DateOfBirth { get; set; }

    [PersonalData(DataClassification.HighlyRestricted)]
    [DataPurpose(DataPurpose.PaymentProcessing)]
    public string? TaxId { get; set; }
}
```

### Purpose-Scoped Data Projection

```csharp
public interface IPurposeScopedProjector
{
    T ProjectForPurpose<T>(T entity, DataPurpose purpose) where T : class;
}

public class PurposeScopedProjector : IPurposeScopedProjector
{
    private readonly ILogger<PurposeScopedProjector> _logger;

    public PurposeScopedProjector(ILogger<PurposeScopedProjector> logger)
    {
        _logger = logger;
    }

    public T ProjectForPurpose<T>(T entity, DataPurpose purpose) where T : class
    {
        var clone = MemberwiseCloneEntity(entity);
        var properties = typeof(T).GetProperties();

        foreach (var prop in properties)
        {
            var purposeAttr = prop.GetCustomAttribute<DataPurposeAttribute>();
            if (purposeAttr != null && !purposeAttr.AllowedPurposes.Contains(purpose))
            {
                // Redact fields not allowed for this purpose
                if (prop.PropertyType == typeof(string))
                    prop.SetValue(clone, null);
                else if (Nullable.GetUnderlyingType(prop.PropertyType) != null)
                    prop.SetValue(clone, null);

                _logger.LogDebug(
                    "Redacted {Property} from {Type} — not authorized for {Purpose}",
                    prop.Name, typeof(T).Name, purpose);
            }
        }

        return clone;
    }

    private static T MemberwiseCloneEntity<T>(T source) where T : class
    {
        var clone = Activator.CreateInstance<T>();
        foreach (var prop in typeof(T).GetProperties().Where(p => p.CanWrite))
        {
            prop.SetValue(clone, prop.GetValue(source));
        }
        return clone;
    }
}
```

### Data Collection Validator

```csharp
public class MinimalDataCollectionValidator
{
    private static readonly Dictionary<string, HashSet<string>> RequiredFieldsByPurpose = new()
    {
        ["OrderFulfillment"] = new() { "FirstName", "LastName", "Email", "ShippingAddress" },
        ["PaymentProcessing"] = new() { "BillingAddress", "PaymentToken" },
        ["Marketing"] = new() { "Email" },
        ["Analytics"] = new() { } // No PII needed
    };

    public ValidationResult ValidateCollection(
        string purpose, IReadOnlyDictionary<string, object?> collectedFields)
    {
        if (!RequiredFieldsByPurpose.TryGetValue(purpose, out var required))
            return ValidationResult.Failure($"Unknown purpose: {purpose}");

        var excessFields = collectedFields.Keys
            .Except(required, StringComparer.OrdinalIgnoreCase)
            .ToList();

        if (excessFields.Count > 0)
        {
            return ValidationResult.Warning(
                $"Excessive data collection for purpose '{purpose}': " +
                $"{string.Join(", ", excessFields)} are not required");
        }

        var missingFields = required
            .Except(collectedFields.Keys, StringComparer.OrdinalIgnoreCase)
            .ToList();

        if (missingFields.Count > 0)
        {
            return ValidationResult.Failure(
                $"Missing required fields for '{purpose}': {string.Join(", ", missingFields)}");
        }

        return ValidationResult.Success();
    }
}

public record ValidationResult(bool IsValid, bool HasWarnings, string? Message)
{
    public static ValidationResult Success() => new(true, false, null);
    public static ValidationResult Warning(string message) => new(true, true, message);
    public static ValidationResult Failure(string message) => new(false, false, message);
}
```

---

## 10. Privacy by Design in .NET

Privacy by Design integrates data protection into the software development lifecycle. In .NET, this means attribute-based PII marking, automatic log redaction, encrypted field storage, privacy-aware serialization, and data masking in non-production environments.

### Automatic Log Redaction

```csharp
[AttributeUsage(AttributeTargets.Property)]
public class SensitiveDataAttribute : Attribute
{
    public string RedactedValue { get; }

    public SensitiveDataAttribute(string redactedValue = "***REDACTED***")
    {
        RedactedValue = redactedValue;
    }
}

public class PrivacyAwareSerializer
{
    public static string SerializeSafe<T>(T obj) where T : class
    {
        var options = new JsonSerializerOptions
        {
            WriteIndented = false,
            Converters = { new SensitiveDataJsonConverter() }
        };

        return JsonSerializer.Serialize(obj, options);
    }
}

public class SensitiveDataJsonConverter : JsonConverterFactory
{
    public override bool CanConvert(Type typeToConvert) =>
        typeToConvert.GetProperties()
            .Any(p => p.GetCustomAttribute<SensitiveDataAttribute>() != null);

    public override JsonConverter CreateConverter(
        Type typeToConvert, JsonSerializerOptions options)
    {
        var converterType = typeof(SensitiveDataConverter<>).MakeGenericType(typeToConvert);
        return (JsonConverter)Activator.CreateInstance(converterType)!;
    }

    private class SensitiveDataConverter<T> : JsonConverter<T> where T : class
    {
        public override T? Read(
            ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
            => JsonSerializer.Deserialize<T>(ref reader);

        public override void Write(
            Utf8JsonWriter writer, T value, JsonSerializerOptions options)
        {
            writer.WriteStartObject();
            foreach (var prop in typeof(T).GetProperties())
            {
                var sensitive = prop.GetCustomAttribute<SensitiveDataAttribute>();
                var propValue = prop.GetValue(value);

                writer.WritePropertyName(prop.Name);
                if (sensitive != null)
                    writer.WriteStringValue(sensitive.RedactedValue);
                else
                    JsonSerializer.Serialize(writer, propValue, prop.PropertyType, options);
            }
            writer.WriteEndObject();
        }
    }
}
```

### Encrypted Field Value Converter (EF Core)

```csharp
public class EncryptedStringConverter : ValueConverter<string, string>
{
    public EncryptedStringConverter(IEncryptionService encryption)
        : base(
            v => encryption.EncryptAsync(v).GetAwaiter().GetResult(),
            v => encryption.DecryptAsync(v).GetAwaiter().GetResult())
    {
    }
}

// Usage in EF Core configuration
public static class EncryptedFieldExtensions
{
    public static PropertyBuilder<string> IsEncrypted(
        this PropertyBuilder<string> builder, IEncryptionService encryption)
    {
        return builder.HasConversion(new EncryptedStringConverter(encryption));
    }
}
```

### Non-Production Data Masking

```csharp
public interface IDataMaskingService
{
    Task<int> MaskEnvironmentDataAsync(CancellationToken cancellationToken = default);
}

public class NonProductionDataMasker : IDataMaskingService
{
    private readonly PrivacyDbContext _db;
    private readonly IHostEnvironment _env;
    private readonly ILogger<NonProductionDataMasker> _logger;

    public NonProductionDataMasker(
        PrivacyDbContext db,
        IHostEnvironment env,
        ILogger<NonProductionDataMasker> logger)
    {
        _db = db;
        _env = env;
        _logger = logger;
    }

    public async Task<int> MaskEnvironmentDataAsync(CancellationToken cancellationToken)
    {
        if (_env.IsProduction())
        {
            throw new InvalidOperationException(
                "Data masking cannot be executed in production");
        }

        _logger.LogWarning(
            "Masking PII data in {Environment} environment", _env.EnvironmentName);

        var totalAffected = 0;

        // Mask customer emails with deterministic fake values
        totalAffected += await _db.Database.ExecuteSqlRawAsync(@"
            UPDATE customers SET
                email = CONCAT('user-', id, '@test.example.com'),
                first_name = CONCAT('FirstName-', LEFT(CAST(id AS VARCHAR(36)), 8)),
                last_name = CONCAT('LastName-', LEFT(CAST(id AS VARCHAR(36)), 8)),
                phone = '+1-555-000-' + RIGHT('0000' + CAST(ABS(CHECKSUM(id)) % 10000 AS VARCHAR(4)), 4),
                date_of_birth = DATEADD(DAY, ABS(CHECKSUM(id)) % 10000, '1970-01-01')
            WHERE is_anonymized = 0", cancellationToken);

        // Mask addresses
        totalAffected += await _db.Database.ExecuteSqlRawAsync(@"
            UPDATE addresses SET
                line1 = CONCAT(ABS(CHECKSUM(id)) % 9999, ' Test Street'),
                line2 = NULL,
                city = 'Testville',
                postal_code = '00000'
            WHERE 1=1", cancellationToken);

        _logger.LogInformation(
            "Masked {Count} records in {Environment}", totalAffected, _env.EnvironmentName);

        return totalAffected;
    }
}
```

### Privacy Impact Assessment Helper

```csharp
public class PrivacyImpactAssessment
{
    public string FeatureName { get; init; } = default!;
    public List<PiaDataElement> DataElements { get; init; } = new();
    public List<PiaRisk> IdentifiedRisks { get; init; } = new();
    public bool RequiresDpia { get; private set; }

    public PiaResult Evaluate()
    {
        var hasHighRiskData = DataElements
            .Any(d => d.Classification >= DataClassification.Restricted);
        var hasLargeScaleProcessing = DataElements
            .Any(d => d.EstimatedRecordCount > 100_000);
        var hasCrossBorderTransfer = DataElements
            .Any(d => d.CrossBorderTransfer);
        var hasAutomatedDecisionMaking = DataElements
            .Any(d => d.InvolvesAutomatedDecisionMaking);

        RequiresDpia = hasHighRiskData && (hasLargeScaleProcessing ||
                                            hasCrossBorderTransfer ||
                                            hasAutomatedDecisionMaking);

        return new PiaResult
        {
            FeatureName = FeatureName,
            RiskLevel = CalculateRiskLevel(),
            RequiresDpia = RequiresDpia,
            Recommendations = GenerateRecommendations()
        };
    }

    private string CalculateRiskLevel()
    {
        var maxClassification = DataElements
            .Max(d => d.Classification);
        return maxClassification switch
        {
            DataClassification.HighlyRestricted => "Critical",
            DataClassification.Restricted => "High",
            DataClassification.Confidential => "Medium",
            _ => "Low"
        };
    }

    private List<string> GenerateRecommendations()
    {
        var recs = new List<string>();
        if (DataElements.Any(d => d.Classification >= DataClassification.Restricted))
            recs.Add("Implement field-level encryption for restricted data");
        if (DataElements.Any(d => d.CrossBorderTransfer))
            recs.Add("Ensure SCCs or adequacy decision covers transfer");
        if (DataElements.Any(d => d.InvolvesAutomatedDecisionMaking))
            recs.Add("Provide mechanism for human review of automated decisions");
        if (DataElements.Any(d => d.RetentionPeriod == null))
            recs.Add("Define explicit retention periods for all data elements");
        return recs;
    }
}

public class PiaDataElement
{
    public string Name { get; init; } = default!;
    public DataClassification Classification { get; init; }
    public string Purpose { get; init; } = default!;
    public LawfulBasis LawfulBasis { get; init; }
    public int EstimatedRecordCount { get; init; }
    public bool CrossBorderTransfer { get; init; }
    public bool InvolvesAutomatedDecisionMaking { get; init; }
    public string? RetentionPeriod { get; init; }
}

public class PiaRisk
{
    public string Description { get; init; } = default!;
    public string Likelihood { get; init; } = default!;
    public string Impact { get; init; } = default!;
    public string Mitigation { get; init; } = default!;
}

public class PiaResult
{
    public string FeatureName { get; init; } = default!;
    public string RiskLevel { get; init; } = default!;
    public bool RequiresDpia { get; init; }
    public List<string> Recommendations { get; init; } = new();
}
```

---

## 11. Implementation Checklist

### Phase 1: Foundation (Weeks 1–3)

- [ ] Deploy privacy data model (ConsentRecord, DataSubjectRequest, PersonalDataInventory)
- [ ] Run EF Core migrations for all privacy tables
- [ ] Classify all existing data fields by DataClassification level
- [ ] Build personal data inventory across all systems
- [ ] Implement lawful basis tracking service
- [ ] Set up DSAR intake API endpoint with identity verification

### Phase 2: Consent & Rights (Weeks 4–6)

- [ ] Deploy consent management service with category-based consent
- [ ] Implement cookie consent banner with preference center
- [ ] Build consent enforcement filter (RequiresConsentAttribute)
- [ ] Implement right to erasure with anonymization fallback
- [ ] Implement right to data portability (JSON/CSV export)
- [ ] Build Do Not Sell / Global Privacy Control (GPC) signal handling
- [ ] Configure legal hold service to block erasure when required

### Phase 3: Retention & Payment Privacy (Weeks 7–9)

- [ ] Define and configure retention schedules for all data categories
- [ ] Deploy automated purge background job
- [ ] Implement token vault for PCI-compliant card storage
- [ ] Deploy CDE isolation middleware with TLS enforcement
- [ ] Set up PAN masking across all display and logging surfaces
- [ ] Audit all log statements for inadvertent PII/card data exposure

### Phase 4: Privacy Engineering (Weeks 10–12)

- [ ] Apply `[PersonalData]` and `[SensitiveData]` attributes to all domain models
- [ ] Implement automatic log redaction for sensitive fields
- [ ] Deploy encrypted field storage for restricted data (EF Core converters)
- [ ] Build non-production data masking pipeline
- [ ] Implement purpose-scoped data projection service
- [ ] Run Privacy Impact Assessment for all major features

### Phase 5: Monitoring & Compliance (Weeks 13–15)

- [ ] Deploy DSAR SLA monitoring with escalation alerts
- [ ] Set up compliance dashboards (consent rates, DSAR turnaround, purge metrics)
- [ ] Configure cross-border transfer validation for all data flows
- [ ] Conduct GDPR Article 30 Records of Processing Activities audit
- [ ] Schedule quarterly privacy review cadence
- [ ] Train development team on privacy-by-design practices
- [ ] Engage external DPO or assign internal DPO role
- [ ] Document data processing agreements with all third-party processors

---

> **Next Steps:**
>
> - Review [15-SECURITY-PATTERNS-PRACTICES](15-SECURITY-PATTERNS-PRACTICES.md) for PCI DSS encryption integration and security architecture patterns.
> - See [06-DATABASE-DESIGN](06-DATABASE-DESIGN.md) for database schema patterns that support privacy data models.
> - Refer to [19-MONITORING-ALERTING](19-MONITORING-ALERTING.md) for setting up privacy compliance dashboards and DSAR SLA alerting.
> - Consult [28-ADDRESS-VERIFICATION-FRAUD-PREVENTION](28-ADDRESS-VERIFICATION-FRAUD-PREVENTION.md) for KYC data handling and identity verification flows.
