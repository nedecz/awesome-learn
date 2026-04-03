# Returns, Refunds, and RMA (Return Merchandise Authorization)

## Overview

Returns, refunds, and RMA processing form a critical post-purchase subsystem in any commerce platform. A well-designed return flow protects revenue while maintaining customer trust — balancing generous return policies with fraud prevention and operational efficiency. This document covers the full lifecycle of returns: from eligibility validation and RMA issuance through inspection, disposition, and refund settlement. Each section includes .NET implementation examples using domain-driven design, EF Core, and async patterns suitable for production e-commerce systems.

> **Related documents:**
>
> - [05-ERROR-HANDLING](05-ERROR-HANDLING.md) — Error handling patterns for resilient return processing
> - [16-STRIPE-INTEGRATION](16-STRIPE-INTEGRATION.md) — Stripe refund APIs and payment reversal
> - [18-COMMON-SCENARIOS](18-COMMON-SCENARIOS.md) — Return/refund scenario walkthroughs
> - [22-EVENT-DRIVEN-ARCHITECTURE](22-EVENT-DRIVEN-ARCHITECTURE.md) — Domain events for return lifecycle

## Table of Contents

1. [Return Data Model](#1-return-data-model)
2. [RMA Workflow State Machine](#2-rma-workflow-state-machine)
3. [Return Eligibility Engine](#3-return-eligibility-engine)
4. [Return Shipping & Logistics](#4-return-shipping--logistics)
5. [Inspection & Quality Assessment](#5-inspection--quality-assessment)
6. [Refund Processing](#6-refund-processing)
7. [Exchange Flows](#7-exchange-flows)
8. [Warranty Claims](#8-warranty-claims)
9. [Analytics & Loss Prevention](#9-analytics--loss-prevention)
10. [Customer Communication](#10-customer-communication)
11. [Implementation Checklist](#11-implementation-checklist)

---

## 1. Return Data Model

The return data model captures every aspect of a return request — the items being returned, the reason, the current status, and the financial outcome. These entities form the aggregate root for the returns bounded context.

### Core Entities

```csharp
namespace Returns.Domain.Entities;

public enum ReturnStatus
{
    Requested,
    Approved,
    Rejected,
    LabelGenerated,
    Shipped,
    Received,
    Inspected,
    RefundIssued,
    ExchangeShipped,
    Closed,
    Cancelled
}

public enum ReturnReason
{
    Defective,
    WrongItem,
    NotAsDescribed,
    DoesNotFit,
    ChangedMind,
    ArrivedTooLate,
    BetterPriceFound,
    DamagedInShipping,
    MissingParts,
    Unauthorized,
    Other
}

public enum ResolutionType
{
    Refund,
    StoreCredit,
    Exchange,
    Repair,
    Replacement
}

public class ReturnRequest
{
    public Guid Id { get; private set; }
    public string RmaNumber { get; private set; } = default!;
    public Guid OrderId { get; private set; }
    public Guid CustomerId { get; private set; }
    public ReturnStatus Status { get; private set; }
    public ResolutionType RequestedResolution { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? ApprovedAt { get; private set; }
    public DateTime? ReceivedAt { get; private set; }
    public DateTime? ResolvedAt { get; private set; }
    public DateTime ReturnWindowDeadline { get; private set; }
    public string? CustomerNotes { get; private set; }
    public string? InternalNotes { get; private set; }
    public string? TrackingNumber { get; private set; }
    public string? ReturnLabelUrl { get; private set; }

    private readonly List<ReturnItem> _items = new();
    public IReadOnlyCollection<ReturnItem> Items => _items.AsReadOnly();

    private readonly List<ReturnStatusHistory> _statusHistory = new();
    public IReadOnlyCollection<ReturnStatusHistory> StatusHistory => _statusHistory.AsReadOnly();

    private ReturnRequest() { } // EF Core

    public static ReturnRequest Create(
        Guid orderId,
        Guid customerId,
        ResolutionType resolution,
        DateTime returnWindowDeadline,
        string? customerNotes = null)
    {
        var request = new ReturnRequest
        {
            Id = Guid.NewGuid(),
            RmaNumber = GenerateRmaNumber(),
            OrderId = orderId,
            CustomerId = customerId,
            Status = ReturnStatus.Requested,
            RequestedResolution = resolution,
            CreatedAt = DateTime.UtcNow,
            ReturnWindowDeadline = returnWindowDeadline,
            CustomerNotes = customerNotes
        };

        request.AddStatusHistory(ReturnStatus.Requested, "Return request created");
        return request;
    }

    public void AddItem(ReturnItem item)
    {
        if (Status != ReturnStatus.Requested)
            throw new InvalidOperationException("Cannot add items after request is submitted.");

        _items.Add(item);
    }

    public void Approve(string? internalNotes = null)
    {
        if (Status != ReturnStatus.Requested)
            throw new InvalidOperationException($"Cannot approve return in {Status} status.");

        Status = ReturnStatus.Approved;
        ApprovedAt = DateTime.UtcNow;
        InternalNotes = internalNotes;
        AddStatusHistory(ReturnStatus.Approved, "Return approved");
    }

    public void Reject(string reason)
    {
        if (Status != ReturnStatus.Requested)
            throw new InvalidOperationException($"Cannot reject return in {Status} status.");

        Status = ReturnStatus.Rejected;
        InternalNotes = reason;
        AddStatusHistory(ReturnStatus.Rejected, reason);
    }

    public void SetTrackingInfo(string trackingNumber, string? labelUrl = null)
    {
        TrackingNumber = trackingNumber;
        ReturnLabelUrl = labelUrl;
        Status = ReturnStatus.LabelGenerated;
        AddStatusHistory(ReturnStatus.LabelGenerated, "Return label generated");
    }

    public void MarkShipped()
    {
        Status = ReturnStatus.Shipped;
        AddStatusHistory(ReturnStatus.Shipped, "Return shipment in transit");
    }

    public void MarkReceived()
    {
        Status = ReturnStatus.Received;
        ReceivedAt = DateTime.UtcNow;
        AddStatusHistory(ReturnStatus.Received, "Return received at warehouse");
    }

    public void MarkInspected()
    {
        Status = ReturnStatus.Inspected;
        AddStatusHistory(ReturnStatus.Inspected, "Inspection complete");
    }

    public void Resolve()
    {
        Status = RequestedResolution == ResolutionType.Exchange
            ? ReturnStatus.ExchangeShipped
            : ReturnStatus.RefundIssued;

        ResolvedAt = DateTime.UtcNow;
        AddStatusHistory(Status, "Return resolved");
    }

    public void Close()
    {
        Status = ReturnStatus.Closed;
        AddStatusHistory(ReturnStatus.Closed, "Return closed");
    }

    private void AddStatusHistory(ReturnStatus status, string description)
    {
        _statusHistory.Add(new ReturnStatusHistory
        {
            Id = Guid.NewGuid(),
            ReturnRequestId = Id,
            Status = status,
            Description = description,
            Timestamp = DateTime.UtcNow
        });
    }

    private static string GenerateRmaNumber()
    {
        var timestamp = DateTime.UtcNow.ToString("yyyyMMdd");
        var random = Random.Shared.Next(100000, 999999);
        return $"RMA-{timestamp}-{random}";
    }
}
```

### Return Item and Status History

```csharp
namespace Returns.Domain.Entities;

public enum ItemCondition
{
    Unopened,
    LikeNew,
    Used,
    Damaged,
    Defective
}

public class ReturnItem
{
    public Guid Id { get; set; }
    public Guid ReturnRequestId { get; set; }
    public Guid OrderLineId { get; set; }
    public string Sku { get; set; } = default!;
    public string ProductName { get; set; } = default!;
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    public ReturnReason Reason { get; set; }
    public string? ReasonDetail { get; set; }
    public ItemCondition? ReceivedCondition { get; set; }
    public decimal? RefundAmount { get; set; }
    public decimal? RestockingFee { get; set; }
    public string? DispositionAction { get; set; }
}

public class ReturnStatusHistory
{
    public Guid Id { get; set; }
    public Guid ReturnRequestId { get; set; }
    public ReturnStatus Status { get; set; }
    public string Description { get; set; } = default!;
    public DateTime Timestamp { get; set; }
}
```

### EF Core Configuration

```csharp
namespace Returns.Infrastructure.Persistence;

using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Returns.Domain.Entities;

public class ReturnRequestConfiguration : IEntityTypeConfiguration<ReturnRequest>
{
    public void Configure(EntityTypeBuilder<ReturnRequest> builder)
    {
        builder.ToTable("ReturnRequests");
        builder.HasKey(r => r.Id);

        builder.Property(r => r.RmaNumber)
            .HasMaxLength(30)
            .IsRequired();

        builder.HasIndex(r => r.RmaNumber).IsUnique();
        builder.HasIndex(r => r.OrderId);
        builder.HasIndex(r => r.CustomerId);
        builder.HasIndex(r => r.Status);

        builder.Property(r => r.Status)
            .HasConversion<string>()
            .HasMaxLength(30);

        builder.Property(r => r.RequestedResolution)
            .HasConversion<string>()
            .HasMaxLength(30);

        builder.Property(r => r.CustomerNotes).HasMaxLength(2000);
        builder.Property(r => r.InternalNotes).HasMaxLength(2000);
        builder.Property(r => r.TrackingNumber).HasMaxLength(100);
        builder.Property(r => r.ReturnLabelUrl).HasMaxLength(500);

        builder.HasMany(r => r.Items)
            .WithOne()
            .HasForeignKey(i => i.ReturnRequestId)
            .OnDelete(DeleteBehavior.Cascade);

        builder.HasMany(r => r.StatusHistory)
            .WithOne()
            .HasForeignKey(h => h.ReturnRequestId)
            .OnDelete(DeleteBehavior.Cascade);

        builder.Navigation(r => r.Items).UsePropertyAccessMode(PropertyAccessMode.Field);
        builder.Navigation(r => r.StatusHistory).UsePropertyAccessMode(PropertyAccessMode.Field);
    }
}

public class ReturnItemConfiguration : IEntityTypeConfiguration<ReturnItem>
{
    public void Configure(EntityTypeBuilder<ReturnItem> builder)
    {
        builder.ToTable("ReturnItems");
        builder.HasKey(i => i.Id);

        builder.Property(i => i.Sku).HasMaxLength(50).IsRequired();
        builder.Property(i => i.ProductName).HasMaxLength(300).IsRequired();

        builder.Property(i => i.UnitPrice).HasPrecision(18, 2);
        builder.Property(i => i.RefundAmount).HasPrecision(18, 2);
        builder.Property(i => i.RestockingFee).HasPrecision(18, 2);

        builder.Property(i => i.Reason)
            .HasConversion<string>()
            .HasMaxLength(30);

        builder.Property(i => i.ReceivedCondition)
            .HasConversion<string>()
            .HasMaxLength(20);

        builder.Property(i => i.DispositionAction).HasMaxLength(50);
        builder.Property(i => i.ReasonDetail).HasMaxLength(1000);
    }
}
```

### Return Window Policies

```csharp
namespace Returns.Domain.Policies;

public record ReturnWindowPolicy(
    string Name,
    int ReturnWindowDays,
    bool AllowUsedItems,
    decimal RestockingFeePercent,
    bool RequiresOriginalPackaging);

public static class DefaultReturnPolicies
{
    public static readonly ReturnWindowPolicy Standard = new(
        Name: "Standard",
        ReturnWindowDays: 30,
        AllowUsedItems: false,
        RestockingFeePercent: 0m,
        RequiresOriginalPackaging: true);

    public static readonly ReturnWindowPolicy Extended = new(
        Name: "Extended",
        ReturnWindowDays: 90,
        AllowUsedItems: true,
        RestockingFeePercent: 0m,
        RequiresOriginalPackaging: false);

    public static readonly ReturnWindowPolicy Electronics = new(
        Name: "Electronics",
        ReturnWindowDays: 15,
        AllowUsedItems: false,
        RestockingFeePercent: 15m,
        RequiresOriginalPackaging: true);

    public static readonly ReturnWindowPolicy FinalSale = new(
        Name: "FinalSale",
        ReturnWindowDays: 0,
        AllowUsedItems: false,
        RestockingFeePercent: 100m,
        RequiresOriginalPackaging: true);
}
```

---

## 2. RMA Workflow State Machine

The RMA state machine governs valid transitions through the return lifecycle. Using explicit state transitions prevents invalid operations and emits domain events for downstream processing.

### State Transition Diagram

| From State       | To State         | Trigger              | Guard Condition                    |
|------------------|------------------|----------------------|------------------------------------|
| **Requested**    | Approved         | Agent/Auto Approve   | Passes eligibility check           |
| **Requested**    | Rejected         | Agent Reject         | Fails eligibility or policy        |
| **Requested**    | Cancelled        | Customer Cancel      | Within cancellation window         |
| **Approved**     | LabelGenerated   | Label Created        | Carrier API success                |
| **LabelGenerated** | Shipped       | Tracking Update      | Carrier scan detected              |
| **Shipped**      | Received         | Warehouse Scan       | Item scanned at facility           |
| **Received**     | Inspected        | QA Complete          | All items inspected                |
| **Inspected**    | RefundIssued     | Refund Processed     | Payment provider confirms          |
| **Inspected**    | ExchangeShipped  | Exchange Shipped     | Replacement order created          |
| **RefundIssued** | Closed           | Auto/Manual Close    | Refund settled                     |
| **ExchangeShipped** | Closed       | Delivery Confirmed   | Exchange delivered                 |

### State Machine Implementation

```csharp
namespace Returns.Domain.StateMachine;

using Returns.Domain.Entities;

public class ReturnStateMachine
{
    private static readonly Dictionary<(ReturnStatus From, ReturnStatus To), Func<ReturnRequest, bool>> _transitions = new()
    {
        { (ReturnStatus.Requested, ReturnStatus.Approved), r => r.Items.Any() },
        { (ReturnStatus.Requested, ReturnStatus.Rejected), _ => true },
        { (ReturnStatus.Requested, ReturnStatus.Cancelled), _ => true },
        { (ReturnStatus.Approved, ReturnStatus.LabelGenerated), _ => true },
        { (ReturnStatus.LabelGenerated, ReturnStatus.Shipped), r => !string.IsNullOrEmpty(r.TrackingNumber) },
        { (ReturnStatus.Shipped, ReturnStatus.Received), _ => true },
        { (ReturnStatus.Received, ReturnStatus.Inspected), r => r.Items.All(i => i.ReceivedCondition.HasValue) },
        { (ReturnStatus.Inspected, ReturnStatus.RefundIssued), r => r.RequestedResolution == ResolutionType.Refund
                                                                   || r.RequestedResolution == ResolutionType.StoreCredit },
        { (ReturnStatus.Inspected, ReturnStatus.ExchangeShipped), r => r.RequestedResolution == ResolutionType.Exchange },
        { (ReturnStatus.RefundIssued, ReturnStatus.Closed), _ => true },
        { (ReturnStatus.ExchangeShipped, ReturnStatus.Closed), _ => true },
    };

    public bool CanTransition(ReturnRequest request, ReturnStatus targetStatus)
    {
        var key = (request.Status, targetStatus);
        return _transitions.TryGetValue(key, out var guard) && guard(request);
    }

    public IReadOnlyList<ReturnStatus> GetAvailableTransitions(ReturnRequest request)
    {
        return _transitions
            .Where(kvp => kvp.Key.From == request.Status && kvp.Value(request))
            .Select(kvp => kvp.Key.To)
            .ToList();
    }
}
```

### Domain Events

```csharp
namespace Returns.Domain.Events;

public abstract record ReturnDomainEvent(Guid ReturnRequestId, DateTime OccurredAt);

public record ReturnRequestedEvent(
    Guid ReturnRequestId,
    Guid OrderId,
    Guid CustomerId,
    string RmaNumber,
    DateTime OccurredAt) : ReturnDomainEvent(ReturnRequestId, OccurredAt);

public record ReturnApprovedEvent(
    Guid ReturnRequestId,
    string RmaNumber,
    DateTime OccurredAt) : ReturnDomainEvent(ReturnRequestId, OccurredAt);

public record ReturnRejectedEvent(
    Guid ReturnRequestId,
    string RmaNumber,
    string Reason,
    DateTime OccurredAt) : ReturnDomainEvent(ReturnRequestId, OccurredAt);

public record ReturnReceivedEvent(
    Guid ReturnRequestId,
    string RmaNumber,
    int ItemCount,
    DateTime OccurredAt) : ReturnDomainEvent(ReturnRequestId, OccurredAt);

public record ReturnInspectedEvent(
    Guid ReturnRequestId,
    string RmaNumber,
    IReadOnlyList<InspectionResult> Results,
    DateTime OccurredAt) : ReturnDomainEvent(ReturnRequestId, OccurredAt);

public record RefundIssuedEvent(
    Guid ReturnRequestId,
    string RmaNumber,
    decimal RefundTotal,
    string RefundMethod,
    DateTime OccurredAt) : ReturnDomainEvent(ReturnRequestId, OccurredAt);

public record InspectionResult(
    Guid ReturnItemId,
    string Sku,
    ItemCondition Condition,
    string DispositionAction);

public record ExchangeShippedEvent(
    Guid ReturnRequestId,
    Guid ExchangeOrderId,
    string RmaNumber,
    DateTime OccurredAt) : ReturnDomainEvent(ReturnRequestId, OccurredAt);
```

### Workflow Orchestrator

```csharp
namespace Returns.Application.Services;

using Returns.Domain.Entities;
using Returns.Domain.Events;
using Returns.Domain.StateMachine;

public interface IReturnWorkflowService
{
    Task<ReturnRequest> ApproveReturnAsync(Guid returnId, string? notes = null, CancellationToken ct = default);
    Task<ReturnRequest> RejectReturnAsync(Guid returnId, string reason, CancellationToken ct = default);
    Task ProcessReturnReceivedAsync(Guid returnId, CancellationToken ct = default);
}

public class ReturnWorkflowService : IReturnWorkflowService
{
    private readonly IReturnRepository _repository;
    private readonly ReturnStateMachine _stateMachine;
    private readonly IEventPublisher _eventPublisher;
    private readonly IReturnLabelService _labelService;

    public ReturnWorkflowService(
        IReturnRepository repository,
        ReturnStateMachine stateMachine,
        IEventPublisher eventPublisher,
        IReturnLabelService labelService)
    {
        _repository = repository;
        _stateMachine = stateMachine;
        _eventPublisher = eventPublisher;
        _labelService = labelService;
    }

    public async Task<ReturnRequest> ApproveReturnAsync(
        Guid returnId, string? notes, CancellationToken ct)
    {
        var request = await _repository.GetByIdAsync(returnId, ct)
            ?? throw new InvalidOperationException($"Return {returnId} not found.");

        if (!_stateMachine.CanTransition(request, ReturnStatus.Approved))
            throw new InvalidOperationException(
                $"Cannot approve return in {request.Status} status.");

        request.Approve(notes);

        var label = await _labelService.GenerateReturnLabelAsync(request, ct);
        request.SetTrackingInfo(label.TrackingNumber, label.LabelUrl);

        await _repository.UpdateAsync(request, ct);

        await _eventPublisher.PublishAsync(new ReturnApprovedEvent(
            request.Id, request.RmaNumber, DateTime.UtcNow), ct);

        return request;
    }

    public async Task<ReturnRequest> RejectReturnAsync(
        Guid returnId, string reason, CancellationToken ct)
    {
        var request = await _repository.GetByIdAsync(returnId, ct)
            ?? throw new InvalidOperationException($"Return {returnId} not found.");

        if (!_stateMachine.CanTransition(request, ReturnStatus.Rejected))
            throw new InvalidOperationException(
                $"Cannot reject return in {request.Status} status.");

        request.Reject(reason);
        await _repository.UpdateAsync(request, ct);

        await _eventPublisher.PublishAsync(new ReturnRejectedEvent(
            request.Id, request.RmaNumber, reason, DateTime.UtcNow), ct);

        return request;
    }

    public async Task ProcessReturnReceivedAsync(Guid returnId, CancellationToken ct)
    {
        var request = await _repository.GetByIdAsync(returnId, ct)
            ?? throw new InvalidOperationException($"Return {returnId} not found.");

        if (!_stateMachine.CanTransition(request, ReturnStatus.Received))
            throw new InvalidOperationException(
                $"Cannot mark return as received in {request.Status} status.");

        request.MarkReceived();
        await _repository.UpdateAsync(request, ct);

        await _eventPublisher.PublishAsync(new ReturnReceivedEvent(
            request.Id, request.RmaNumber, request.Items.Count, DateTime.UtcNow), ct);
    }
}
```

---

## 3. Return Eligibility Engine

The eligibility engine evaluates whether a return request is valid before an RMA is issued. Rules are composable and can vary by product category, customer tier, or promotional context.

### Eligibility Rule Interface

```csharp
namespace Returns.Domain.Eligibility;

using Returns.Domain.Entities;

public record EligibilityContext(
    ReturnRequest Request,
    DateTime OrderDeliveredAt,
    string ProductCategory,
    bool IsFinalSale,
    bool IsGiftOrder,
    int CustomerReturnCountLast90Days,
    decimal CustomerReturnRateLast90Days);

public record EligibilityResult(
    bool IsEligible,
    string? DenialReason = null,
    decimal RestockingFeePercent = 0m);

public interface IReturnEligibilityRule
{
    int Priority { get; }
    Task<EligibilityResult> EvaluateAsync(EligibilityContext context, CancellationToken ct);
}
```

### Built-In Rules

```csharp
namespace Returns.Domain.Eligibility.Rules;

public class ReturnWindowRule : IReturnEligibilityRule
{
    public int Priority => 1;

    public Task<EligibilityResult> EvaluateAsync(
        EligibilityContext context, CancellationToken ct)
    {
        var daysSinceDelivery = (DateTime.UtcNow - context.OrderDeliveredAt).Days;
        var windowDays = (context.Request.ReturnWindowDeadline - context.OrderDeliveredAt).Days;

        if (daysSinceDelivery > windowDays)
        {
            return Task.FromResult(new EligibilityResult(
                false,
                $"Return window expired. Items must be returned within {windowDays} days of delivery."));
        }

        return Task.FromResult(new EligibilityResult(true));
    }
}

public class FinalSaleRule : IReturnEligibilityRule
{
    public int Priority => 2;

    public Task<EligibilityResult> EvaluateAsync(
        EligibilityContext context, CancellationToken ct)
    {
        if (context.IsFinalSale)
        {
            return Task.FromResult(new EligibilityResult(
                false, "Final sale items are not eligible for return."));
        }

        return Task.FromResult(new EligibilityResult(true));
    }
}

public class CategoryRestockingFeeRule : IReturnEligibilityRule
{
    private static readonly Dictionary<string, decimal> _categoryFees = new()
    {
        ["Electronics"] = 15m,
        ["OpenBoxSoftware"] = 25m,
        ["CustomizedItems"] = 50m,
    };

    public int Priority => 5;

    public Task<EligibilityResult> EvaluateAsync(
        EligibilityContext context, CancellationToken ct)
    {
        var fee = _categoryFees.GetValueOrDefault(context.ProductCategory, 0m);
        return Task.FromResult(new EligibilityResult(true, RestockingFeePercent: fee));
    }
}

public class SerialReturnerRule : IReturnEligibilityRule
{
    private const int MaxReturnsIn90Days = 10;
    private const decimal MaxReturnRate = 0.40m;

    public int Priority => 10;

    public Task<EligibilityResult> EvaluateAsync(
        EligibilityContext context, CancellationToken ct)
    {
        if (context.CustomerReturnCountLast90Days >= MaxReturnsIn90Days)
        {
            return Task.FromResult(new EligibilityResult(
                false,
                "Return limit reached. Please contact customer support for assistance."));
        }

        if (context.CustomerReturnRateLast90Days > MaxReturnRate)
        {
            return Task.FromResult(new EligibilityResult(
                false,
                "Unusual return activity detected. Please contact customer support."));
        }

        return Task.FromResult(new EligibilityResult(true));
    }
}
```

### Eligibility Engine

```csharp
namespace Returns.Domain.Eligibility;

public class ReturnEligibilityEngine
{
    private readonly IEnumerable<IReturnEligibilityRule> _rules;

    public ReturnEligibilityEngine(IEnumerable<IReturnEligibilityRule> rules)
    {
        _rules = rules.OrderBy(r => r.Priority);
    }

    public async Task<EligibilityResult> EvaluateAsync(
        EligibilityContext context, CancellationToken ct = default)
    {
        var maxRestockingFee = 0m;

        foreach (var rule in _rules)
        {
            var result = await rule.EvaluateAsync(context, ct);

            if (!result.IsEligible)
                return result;

            if (result.RestockingFeePercent > maxRestockingFee)
                maxRestockingFee = result.RestockingFeePercent;
        }

        return new EligibilityResult(true, RestockingFeePercent: maxRestockingFee);
    }
}
```

### Registration in DI

```csharp
namespace Returns.Infrastructure;

using Microsoft.Extensions.DependencyInjection;
using Returns.Domain.Eligibility;
using Returns.Domain.Eligibility.Rules;

public static class EligibilityServiceRegistration
{
    public static IServiceCollection AddReturnEligibility(this IServiceCollection services)
    {
        services.AddScoped<IReturnEligibilityRule, ReturnWindowRule>();
        services.AddScoped<IReturnEligibilityRule, FinalSaleRule>();
        services.AddScoped<IReturnEligibilityRule, CategoryRestockingFeeRule>();
        services.AddScoped<IReturnEligibilityRule, SerialReturnerRule>();
        services.AddScoped<ReturnEligibilityEngine>();
        return services;
    }
}
```

---

## 4. Return Shipping & Logistics

Return shipping integrates with carrier APIs to generate prepaid labels, track return shipments, and handle cross-border customs. Providing customers a frictionless shipping experience reduces return abandonment.

### Return Label Service

```csharp
namespace Returns.Application.Shipping;

using Returns.Domain.Entities;

public record ReturnLabel(
    string TrackingNumber,
    string LabelUrl,
    string Carrier,
    decimal ShippingCost,
    DateTime ExpiresAt);

public interface IReturnLabelService
{
    Task<ReturnLabel> GenerateReturnLabelAsync(
        ReturnRequest request, CancellationToken ct = default);
    Task<ReturnLabel> GenerateCrossBorderLabelAsync(
        ReturnRequest request, string originCountry, string destinationCountry,
        CancellationToken ct = default);
}

public record ShipmentStatus(
    string TrackingNumber,
    string Status,
    string? Location,
    DateTime LastUpdated,
    DateTime? EstimatedDelivery);

public interface IReturnTrackingService
{
    Task<ShipmentStatus> GetStatusAsync(string trackingNumber, CancellationToken ct = default);
    Task SubscribeToUpdatesAsync(string trackingNumber, string webhookUrl, CancellationToken ct = default);
}
```

### Carrier Integration

```csharp
namespace Returns.Infrastructure.Shipping;

using Returns.Application.Shipping;
using Returns.Domain.Entities;

public class CarrierReturnLabelService : IReturnLabelService
{
    private readonly HttpClient _httpClient;
    private readonly ShippingConfiguration _config;

    public CarrierReturnLabelService(
        HttpClient httpClient, ShippingConfiguration config)
    {
        _httpClient = httpClient;
        _config = config;
    }

    public async Task<ReturnLabel> GenerateReturnLabelAsync(
        ReturnRequest request, CancellationToken ct)
    {
        var payload = new
        {
            shipment = new
            {
                from_address = new
                {
                    // Customer address from order
                    name = "Customer",
                    street1 = "Loaded from order shipping address",
                    city = "City",
                    state = "ST",
                    zip = "00000",
                    country = "US"
                },
                to_address = new
                {
                    name = _config.ReturnWarehouseName,
                    street1 = _config.ReturnWarehouseStreet,
                    city = _config.ReturnWarehouseCity,
                    state = _config.ReturnWarehouseState,
                    zip = _config.ReturnWarehouseZip,
                    country = _config.ReturnWarehouseCountry
                },
                parcel = new
                {
                    weight = EstimateReturnWeight(request),
                    length = 12, height = 8, width = 6
                },
                is_return = true
            }
        };

        var response = await _httpClient.PostAsJsonAsync(
            $"{_config.CarrierApiBaseUrl}/shipments", payload, ct);

        response.EnsureSuccessStatusCode();
        var result = await response.Content.ReadFromJsonAsync<CarrierShipmentResponse>(ct);

        return new ReturnLabel(
            TrackingNumber: result!.TrackingCode,
            LabelUrl: result.PostageLabel.LabelUrl,
            Carrier: result.SelectedRate.Carrier,
            ShippingCost: decimal.Parse(result.SelectedRate.Rate),
            ExpiresAt: DateTime.UtcNow.AddDays(30));
    }

    public async Task<ReturnLabel> GenerateCrossBorderLabelAsync(
        ReturnRequest request, string originCountry, string destinationCountry,
        CancellationToken ct)
    {
        // Cross-border returns require customs declarations
        var label = await GenerateReturnLabelAsync(request, ct);

        // Additional customs form generation would happen here
        // including HS codes, declared values, and origin certificates

        return label;
    }

    private static double EstimateReturnWeight(ReturnRequest request)
    {
        return request.Items.Sum(i => i.Quantity) * 1.5;
    }
}

public class ShippingConfiguration
{
    public string CarrierApiBaseUrl { get; set; } = default!;
    public string CarrierApiKey { get; set; } = default!;
    public string ReturnWarehouseName { get; set; } = default!;
    public string ReturnWarehouseStreet { get; set; } = default!;
    public string ReturnWarehouseCity { get; set; } = default!;
    public string ReturnWarehouseState { get; set; } = default!;
    public string ReturnWarehouseZip { get; set; } = default!;
    public string ReturnWarehouseCountry { get; set; } = default!;
}

// Carrier API response models
public record CarrierShipmentResponse(
    string TrackingCode,
    PostageLabel PostageLabel,
    SelectedRate SelectedRate);

public record PostageLabel(string LabelUrl);
public record SelectedRate(string Carrier, string Rate);
```

### Return Tracking Webhook Handler

```csharp
namespace Returns.Application.Shipping;

using Returns.Domain.Entities;

public class ReturnTrackingWebhookHandler
{
    private readonly IReturnRepository _repository;
    private readonly IEventPublisher _eventPublisher;

    public ReturnTrackingWebhookHandler(
        IReturnRepository repository, IEventPublisher eventPublisher)
    {
        _repository = repository;
        _eventPublisher = eventPublisher;
    }

    public async Task HandleTrackingUpdateAsync(
        string trackingNumber, string status, CancellationToken ct)
    {
        var request = await _repository.GetByTrackingNumberAsync(trackingNumber, ct);
        if (request is null) return;

        switch (status.ToUpperInvariant())
        {
            case "IN_TRANSIT":
                if (request.Status == ReturnStatus.LabelGenerated)
                    request.MarkShipped();
                break;

            case "DELIVERED":
                if (request.Status == ReturnStatus.Shipped)
                    request.MarkReceived();
                break;

            case "RETURNED_TO_SENDER":
                request.Reject("Package returned to sender by carrier.");
                break;
        }

        await _repository.UpdateAsync(request, ct);
    }
}
```

---

## 5. Inspection & Quality Assessment

Once a return is physically received, the inspection workflow grades item condition and determines disposition — whether items should be restocked, refurbished, or disposed. Accurate grading directly impacts inventory value and refund amounts.

### Inspection Models

```csharp
namespace Returns.Domain.Inspection;

using Returns.Domain.Entities;

public enum DispositionAction
{
    Restock,
    RestockAsOpen,
    Refurbish,
    LiquidateBulk,
    Dispose,
    ReturnToVendor
}

public record InspectionCriteria(
    string Category,
    bool RequiresOriginalPackaging,
    bool RequiresAllAccessories,
    bool CheckForTampering,
    int MaxAllowedScratchCount);

public record ItemInspectionRecord
{
    public Guid Id { get; init; }
    public Guid ReturnItemId { get; init; }
    public Guid InspectorId { get; init; }
    public DateTime InspectedAt { get; init; }
    public ItemCondition ConditionGrade { get; init; }
    public bool OriginalPackagingPresent { get; init; }
    public bool AllAccessoriesPresent { get; init; }
    public bool TamperingDetected { get; init; }
    public string? DamageDescription { get; init; }
    public List<string> PhotoUrls { get; init; } = new();
    public DispositionAction Disposition { get; init; }
    public string? Notes { get; init; }
}
```

### Inspection Service

```csharp
namespace Returns.Application.Inspection;

using Returns.Domain.Entities;
using Returns.Domain.Events;
using Returns.Domain.Inspection;

public interface IInspectionService
{
    Task<ItemInspectionRecord> InspectItemAsync(
        Guid returnItemId, InspectionInput input, CancellationToken ct = default);
    Task CompleteInspectionAsync(Guid returnRequestId, CancellationToken ct = default);
}

public record InspectionInput(
    Guid InspectorId,
    ItemCondition ConditionGrade,
    bool OriginalPackagingPresent,
    bool AllAccessoriesPresent,
    bool TamperingDetected,
    string? DamageDescription,
    List<string>? PhotoUrls,
    string? Notes);

public class InspectionService : IInspectionService
{
    private readonly IReturnRepository _returnRepository;
    private readonly IInspectionRecordRepository _inspectionRepo;
    private readonly IDispositionEngine _dispositionEngine;
    private readonly IEventPublisher _eventPublisher;

    public InspectionService(
        IReturnRepository returnRepository,
        IInspectionRecordRepository inspectionRepo,
        IDispositionEngine dispositionEngine,
        IEventPublisher eventPublisher)
    {
        _returnRepository = returnRepository;
        _inspectionRepo = inspectionRepo;
        _dispositionEngine = dispositionEngine;
        _eventPublisher = eventPublisher;
    }

    public async Task<ItemInspectionRecord> InspectItemAsync(
        Guid returnItemId, InspectionInput input, CancellationToken ct)
    {
        var disposition = _dispositionEngine.Determine(
            input.ConditionGrade,
            input.OriginalPackagingPresent,
            input.AllAccessoriesPresent,
            input.TamperingDetected);

        var record = new ItemInspectionRecord
        {
            Id = Guid.NewGuid(),
            ReturnItemId = returnItemId,
            InspectorId = input.InspectorId,
            InspectedAt = DateTime.UtcNow,
            ConditionGrade = input.ConditionGrade,
            OriginalPackagingPresent = input.OriginalPackagingPresent,
            AllAccessoriesPresent = input.AllAccessoriesPresent,
            TamperingDetected = input.TamperingDetected,
            DamageDescription = input.DamageDescription,
            PhotoUrls = input.PhotoUrls ?? new List<string>(),
            Disposition = disposition,
            Notes = input.Notes
        };

        await _inspectionRepo.SaveAsync(record, ct);
        return record;
    }

    public async Task CompleteInspectionAsync(Guid returnRequestId, CancellationToken ct)
    {
        var request = await _returnRepository.GetByIdAsync(returnRequestId, ct)
            ?? throw new InvalidOperationException($"Return {returnRequestId} not found.");

        var inspections = await _inspectionRepo
            .GetByReturnRequestIdAsync(returnRequestId, ct);

        // Update each return item with inspection results
        foreach (var item in request.Items)
        {
            var inspection = inspections.FirstOrDefault(i => i.ReturnItemId == item.Id);
            if (inspection is null)
                throw new InvalidOperationException(
                    $"Item {item.Sku} has not been inspected.");

            item.ReceivedCondition = inspection.ConditionGrade;
            item.DispositionAction = inspection.Disposition.ToString();
        }

        request.MarkInspected();
        await _returnRepository.UpdateAsync(request, ct);

        var results = inspections.Select(i => new InspectionResult(
            i.ReturnItemId, request.Items.First(it => it.Id == i.ReturnItemId).Sku,
            i.ConditionGrade, i.Disposition.ToString())).ToList();

        await _eventPublisher.PublishAsync(new ReturnInspectedEvent(
            request.Id, request.RmaNumber, results, DateTime.UtcNow), ct);
    }
}
```

### Disposition Engine

```csharp
namespace Returns.Application.Inspection;

using Returns.Domain.Entities;
using Returns.Domain.Inspection;

public interface IDispositionEngine
{
    DispositionAction Determine(
        ItemCondition condition,
        bool hasOriginalPackaging,
        bool hasAllAccessories,
        bool tamperingDetected);
}

public class DispositionEngine : IDispositionEngine
{
    public DispositionAction Determine(
        ItemCondition condition,
        bool hasOriginalPackaging,
        bool hasAllAccessories,
        bool tamperingDetected)
    {
        if (tamperingDetected)
            return DispositionAction.Dispose;

        return condition switch
        {
            ItemCondition.Unopened when hasOriginalPackaging
                => DispositionAction.Restock,

            ItemCondition.LikeNew when hasOriginalPackaging && hasAllAccessories
                => DispositionAction.RestockAsOpen,

            ItemCondition.LikeNew
                => DispositionAction.Refurbish,

            ItemCondition.Used when hasAllAccessories
                => DispositionAction.Refurbish,

            ItemCondition.Used
                => DispositionAction.LiquidateBulk,

            ItemCondition.Damaged
                => DispositionAction.ReturnToVendor,

            ItemCondition.Defective
                => DispositionAction.ReturnToVendor,

            _ => DispositionAction.Dispose
        };
    }
}
```

---

## 6. Refund Processing

Refund processing must route funds back through the correct payment method, handle partial refunds, apply restocking fees, and respect refund timing SLAs. Multi-provider support is essential for platforms accepting multiple payment methods on a single order.

### Refund Calculation

```csharp
namespace Returns.Application.Refunds;

using Returns.Domain.Entities;

public record RefundBreakdown(
    decimal ItemSubtotal,
    decimal RestockingFee,
    decimal TaxRefund,
    decimal ShippingRefund,
    decimal PromoAdjustment,
    decimal TotalRefund);

public class RefundCalculator
{
    public RefundBreakdown Calculate(
        ReturnRequest request,
        decimal originalTaxRate,
        decimal originalShippingCost,
        bool refundShipping,
        decimal appliedPromoDiscount)
    {
        var itemSubtotal = 0m;
        var totalRestockingFee = 0m;

        foreach (var item in request.Items)
        {
            var lineTotal = item.UnitPrice * item.Quantity;
            var fee = item.RestockingFee ?? 0m;

            itemSubtotal += lineTotal;
            totalRestockingFee += fee;
        }

        var netItemRefund = itemSubtotal - totalRestockingFee;
        var taxRefund = netItemRefund * originalTaxRate;
        var shippingRefund = refundShipping ? originalShippingCost : 0m;

        // Adjust for promos that applied to returned items
        var promoAdjustment = appliedPromoDiscount > 0
            ? -appliedPromoDiscount
            : 0m;

        var totalRefund = netItemRefund + taxRefund + shippingRefund + promoAdjustment;
        totalRefund = Math.Max(totalRefund, 0m);

        return new RefundBreakdown(
            ItemSubtotal: itemSubtotal,
            RestockingFee: totalRestockingFee,
            TaxRefund: Math.Round(taxRefund, 2),
            ShippingRefund: shippingRefund,
            PromoAdjustment: promoAdjustment,
            TotalRefund: Math.Round(totalRefund, 2));
    }
}
```

### Multi-Provider Refund Service

```csharp
namespace Returns.Application.Refunds;

using Returns.Domain.Entities;
using Returns.Domain.Events;

public interface IPaymentRefundProvider
{
    string ProviderName { get; }
    Task<RefundResult> ProcessRefundAsync(RefundRequest refundRequest, CancellationToken ct);
}

public record RefundRequest(
    string PaymentTransactionId,
    decimal Amount,
    string Currency,
    string Reason,
    Dictionary<string, string>? Metadata = null);

public record RefundResult(
    bool Success,
    string? RefundTransactionId,
    string? ErrorMessage,
    DateTime ProcessedAt);

public class RefundOrchestrator
{
    private readonly Dictionary<string, IPaymentRefundProvider> _providers;
    private readonly IReturnRepository _returnRepository;
    private readonly IRefundRecordRepository _refundRecordRepo;
    private readonly IEventPublisher _eventPublisher;
    private readonly RefundCalculator _calculator;

    public RefundOrchestrator(
        IEnumerable<IPaymentRefundProvider> providers,
        IReturnRepository returnRepository,
        IRefundRecordRepository refundRecordRepo,
        IEventPublisher eventPublisher,
        RefundCalculator calculator)
    {
        _providers = providers.ToDictionary(p => p.ProviderName);
        _returnRepository = returnRepository;
        _refundRecordRepo = refundRecordRepo;
        _eventPublisher = eventPublisher;
        _calculator = calculator;
    }

    public async Task<RefundResult> ProcessRefundAsync(
        Guid returnRequestId,
        string paymentProvider,
        string paymentTransactionId,
        decimal originalTaxRate,
        decimal originalShippingCost,
        bool refundShipping,
        decimal appliedPromoDiscount,
        CancellationToken ct = default)
    {
        var request = await _returnRepository.GetByIdAsync(returnRequestId, ct)
            ?? throw new InvalidOperationException($"Return {returnRequestId} not found.");

        if (request.Status != ReturnStatus.Inspected)
            throw new InvalidOperationException(
                "Return must be inspected before refund can be processed.");

        var breakdown = _calculator.Calculate(
            request, originalTaxRate, originalShippingCost,
            refundShipping, appliedPromoDiscount);

        if (!_providers.TryGetValue(paymentProvider, out var provider))
            throw new InvalidOperationException(
                $"Payment provider '{paymentProvider}' not registered.");

        var refundRequest = new RefundRequest(
            PaymentTransactionId: paymentTransactionId,
            Amount: breakdown.TotalRefund,
            Currency: "USD",
            Reason: $"Return {request.RmaNumber}",
            Metadata: new Dictionary<string, string>
            {
                ["rma_number"] = request.RmaNumber,
                ["return_id"] = request.Id.ToString()
            });

        var result = await provider.ProcessRefundAsync(refundRequest, ct);

        if (result.Success)
        {
            // Update each item with its refund amount
            foreach (var item in request.Items)
            {
                item.RefundAmount = (item.UnitPrice * item.Quantity)
                    - (item.RestockingFee ?? 0m);
            }

            request.Resolve();
            await _returnRepository.UpdateAsync(request, ct);

            await _refundRecordRepo.SaveAsync(new RefundRecord
            {
                Id = Guid.NewGuid(),
                ReturnRequestId = request.Id,
                PaymentProvider = paymentProvider,
                TransactionId = result.RefundTransactionId!,
                Amount = breakdown.TotalRefund,
                Breakdown = breakdown,
                ProcessedAt = result.ProcessedAt
            }, ct);

            await _eventPublisher.PublishAsync(new RefundIssuedEvent(
                request.Id, request.RmaNumber, breakdown.TotalRefund,
                paymentProvider, DateTime.UtcNow), ct);
        }

        return result;
    }
}

public class RefundRecord
{
    public Guid Id { get; set; }
    public Guid ReturnRequestId { get; set; }
    public string PaymentProvider { get; set; } = default!;
    public string TransactionId { get; set; } = default!;
    public decimal Amount { get; set; }
    public RefundBreakdown Breakdown { get; set; } = default!;
    public DateTime ProcessedAt { get; set; }
}
```

### Stripe Refund Provider

```csharp
namespace Returns.Infrastructure.Payments;

using Returns.Application.Refunds;
using Stripe;

public class StripeRefundProvider : IPaymentRefundProvider
{
    private readonly RefundService _refundService;

    public StripeRefundProvider(RefundService refundService)
    {
        _refundService = refundService;
    }

    public string ProviderName => "Stripe";

    public async Task<RefundResult> ProcessRefundAsync(
        RefundRequest refundRequest, CancellationToken ct)
    {
        try
        {
            var options = new RefundCreateOptions
            {
                PaymentIntent = refundRequest.PaymentTransactionId,
                Amount = (long)(refundRequest.Amount * 100), // Convert to cents
                Reason = RefundReasons.RequestedByCustomer,
                Metadata = refundRequest.Metadata
            };

            var refund = await _refundService.CreateAsync(options, cancellationToken: ct);

            return new RefundResult(
                Success: refund.Status == "succeeded",
                RefundTransactionId: refund.Id,
                ErrorMessage: null,
                ProcessedAt: DateTime.UtcNow);
        }
        catch (StripeException ex)
        {
            return new RefundResult(
                Success: false,
                RefundTransactionId: null,
                ErrorMessage: ex.Message,
                ProcessedAt: DateTime.UtcNow);
        }
    }
}
```

### Store Credit Issuance

```csharp
namespace Returns.Application.Refunds;

public record StoreCredit(
    Guid Id,
    Guid CustomerId,
    decimal Amount,
    string Currency,
    string Code,
    DateTime IssuedAt,
    DateTime? ExpiresAt,
    string Reason);

public interface IStoreCreditService
{
    Task<StoreCredit> IssueAsync(
        Guid customerId, decimal amount, string currency,
        string reason, DateTime? expiresAt = null, CancellationToken ct = default);
}

public class StoreCreditService : IStoreCreditService
{
    private readonly IStoreCreditRepository _repository;

    public StoreCreditService(IStoreCreditRepository repository)
    {
        _repository = repository;
    }

    public async Task<StoreCredit> IssueAsync(
        Guid customerId, decimal amount, string currency,
        string reason, DateTime? expiresAt, CancellationToken ct)
    {
        var credit = new StoreCredit(
            Id: Guid.NewGuid(),
            CustomerId: customerId,
            Amount: amount,
            Currency: currency,
            Code: GenerateCreditCode(),
            IssuedAt: DateTime.UtcNow,
            ExpiresAt: expiresAt ?? DateTime.UtcNow.AddYears(1),
            Reason: reason);

        await _repository.SaveAsync(credit, ct);
        return credit;
    }

    private static string GenerateCreditCode()
    {
        return $"SC-{Guid.NewGuid().ToString("N")[..12].ToUpperInvariant()}";
    }
}
```

---

## 7. Exchange Flows

Exchanges add complexity by combining a return with a new outbound order. The system must handle even exchanges, exchanges with price differences, and instant exchanges where the replacement ships before the return is received.

### Exchange Service

```csharp
namespace Returns.Application.Exchanges;

using Returns.Domain.Entities;
using Returns.Domain.Events;

public record ExchangeRequest(
    Guid ReturnRequestId,
    List<ExchangeLineItem> NewItems);

public record ExchangeLineItem(
    string Sku,
    string ProductName,
    int Quantity,
    decimal UnitPrice,
    string? VariantDescription);

public record ExchangeResult(
    Guid ExchangeOrderId,
    decimal PriceDifference,
    bool RequiresAdditionalPayment,
    bool RefundDueToCustomer);

public interface IExchangeService
{
    Task<ExchangeResult> CreateExchangeAsync(
        ExchangeRequest request, CancellationToken ct = default);
    Task<ExchangeResult> CreateInstantExchangeAsync(
        ExchangeRequest request, string paymentMethodId,
        CancellationToken ct = default);
}

public class ExchangeService : IExchangeService
{
    private readonly IReturnRepository _returnRepository;
    private readonly IOrderService _orderService;
    private readonly IPaymentService _paymentService;
    private readonly IInventoryService _inventoryService;
    private readonly IEventPublisher _eventPublisher;

    public ExchangeService(
        IReturnRepository returnRepository,
        IOrderService orderService,
        IPaymentService paymentService,
        IInventoryService inventoryService,
        IEventPublisher eventPublisher)
    {
        _returnRepository = returnRepository;
        _orderService = orderService;
        _paymentService = paymentService;
        _inventoryService = inventoryService;
        _eventPublisher = eventPublisher;
    }

    public async Task<ExchangeResult> CreateExchangeAsync(
        ExchangeRequest request, CancellationToken ct)
    {
        var returnRequest = await _returnRepository
            .GetByIdAsync(request.ReturnRequestId, ct)
            ?? throw new InvalidOperationException("Return request not found.");

        var returnTotal = returnRequest.Items.Sum(i => i.UnitPrice * i.Quantity);
        var exchangeTotal = request.NewItems.Sum(i => i.UnitPrice * i.Quantity);
        var priceDifference = exchangeTotal - returnTotal;

        // Reserve inventory for exchange items
        foreach (var item in request.NewItems)
        {
            await _inventoryService.ReserveAsync(
                item.Sku, item.Quantity, $"Exchange-{returnRequest.RmaNumber}", ct);
        }

        // Create the exchange order
        var exchangeOrderId = await _orderService.CreateExchangeOrderAsync(
            returnRequest.CustomerId,
            returnRequest.RmaNumber,
            request.NewItems,
            ct);

        return new ExchangeResult(
            ExchangeOrderId: exchangeOrderId,
            PriceDifference: priceDifference,
            RequiresAdditionalPayment: priceDifference > 0,
            RefundDueToCustomer: priceDifference < 0);
    }

    public async Task<ExchangeResult> CreateInstantExchangeAsync(
        ExchangeRequest request, string paymentMethodId, CancellationToken ct)
    {
        var returnRequest = await _returnRepository
            .GetByIdAsync(request.ReturnRequestId, ct)
            ?? throw new InvalidOperationException("Return request not found.");

        var returnTotal = returnRequest.Items.Sum(i => i.UnitPrice * i.Quantity);
        var exchangeTotal = request.NewItems.Sum(i => i.UnitPrice * i.Quantity);

        // Place a hold for the full exchange value as protection
        var holdAmount = exchangeTotal;
        var hold = await _paymentService.PlaceHoldAsync(
            paymentMethodId, holdAmount, "USD",
            $"Instant exchange hold for {returnRequest.RmaNumber}", ct);

        // Reserve inventory and create order immediately
        foreach (var item in request.NewItems)
        {
            await _inventoryService.ReserveAsync(
                item.Sku, item.Quantity, $"InstantExchange-{returnRequest.RmaNumber}", ct);
        }

        var exchangeOrderId = await _orderService.CreateExchangeOrderAsync(
            returnRequest.CustomerId,
            returnRequest.RmaNumber,
            request.NewItems,
            ct);

        // Ship the exchange immediately
        await _orderService.ShipOrderAsync(exchangeOrderId, ct);

        await _eventPublisher.PublishAsync(new ExchangeShippedEvent(
            returnRequest.Id, exchangeOrderId, returnRequest.RmaNumber,
            DateTime.UtcNow), ct);

        return new ExchangeResult(
            ExchangeOrderId: exchangeOrderId,
            PriceDifference: exchangeTotal - returnTotal,
            RequiresAdditionalPayment: false,
            RefundDueToCustomer: false);
    }
}
```

### Instant Exchange Hold Reconciliation

```csharp
namespace Returns.Application.Exchanges;

public class ExchangeHoldReconciler
{
    private readonly IReturnRepository _returnRepository;
    private readonly IPaymentService _paymentService;
    private readonly IRefundRecordRepository _refundRecordRepo;

    public ExchangeHoldReconciler(
        IReturnRepository returnRepository,
        IPaymentService paymentService,
        IRefundRecordRepository refundRecordRepo)
    {
        _returnRepository = returnRepository;
        _paymentService = paymentService;
        _refundRecordRepo = refundRecordRepo;
    }

    public async Task ReconcileAsync(
        Guid returnRequestId, string holdId, CancellationToken ct)
    {
        var request = await _returnRepository.GetByIdAsync(returnRequestId, ct)
            ?? throw new InvalidOperationException("Return not found.");

        if (request.Status == ReturnStatus.Inspected)
        {
            // Return received and inspected — release the hold
            await _paymentService.ReleaseHoldAsync(holdId, ct);
        }
        else if (request.Status is ReturnStatus.Rejected or ReturnStatus.Cancelled)
        {
            // Return not received — capture the hold
            await _paymentService.CaptureHoldAsync(holdId, ct);
        }
    }

    public async Task HandleExpiredHoldsAsync(CancellationToken ct)
    {
        // Holds typically expire after 7 days; escalate unresolved exchanges
        var unresolvedExchanges = await _returnRepository
            .GetPendingInstantExchangesOlderThanAsync(TimeSpan.FromDays(7), ct);

        foreach (var exchange in unresolvedExchanges)
        {
            // Notify operations team for manual review
            // Log for reconciliation audit
        }
    }
}
```

---

## 8. Warranty Claims

Warranty claims extend beyond standard returns, covering defects discovered after the return window. The warranty system tracks registrations, validates claim eligibility, and routes to repair or replacement workflows — including manufacturer RMA integration.

### Warranty Models

```csharp
namespace Returns.Domain.Warranty;

public enum WarrantyType
{
    Standard,
    Extended,
    AccidentalDamage,
    ManufacturerLimited
}

public enum ClaimStatus
{
    Submitted,
    UnderReview,
    Approved,
    RepairInProgress,
    ReplacementShipped,
    Denied,
    Closed
}

public class WarrantyRegistration
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public Guid OrderId { get; set; }
    public string Sku { get; set; } = default!;
    public string SerialNumber { get; set; } = default!;
    public WarrantyType Type { get; set; }
    public DateTime PurchaseDate { get; set; }
    public DateTime WarrantyStartDate { get; set; }
    public DateTime WarrantyEndDate { get; set; }
    public bool IsActive => DateTime.UtcNow <= WarrantyEndDate;
}

public class WarrantyClaim
{
    public Guid Id { get; set; }
    public Guid WarrantyRegistrationId { get; set; }
    public Guid CustomerId { get; set; }
    public string ClaimNumber { get; set; } = default!;
    public ClaimStatus Status { get; set; }
    public string IssueDescription { get; set; } = default!;
    public List<string> EvidencePhotoUrls { get; set; } = new();
    public DateTime SubmittedAt { get; set; }
    public DateTime? ResolvedAt { get; set; }
    public string? ResolutionNotes { get; set; }
    public string? ManufacturerRmaNumber { get; set; }
}
```

### Warranty Claim Service

```csharp
namespace Returns.Application.Warranty;

using Returns.Domain.Warranty;

public interface IWarrantyClaimService
{
    Task<WarrantyClaim> SubmitClaimAsync(
        Guid warrantyRegistrationId, string issueDescription,
        List<string>? photoUrls = null, CancellationToken ct = default);
    Task<WarrantyClaim> ReviewClaimAsync(
        Guid claimId, bool approved, string notes, CancellationToken ct = default);
    Task<WarrantyClaim> InitiateRepairAsync(
        Guid claimId, CancellationToken ct = default);
    Task<WarrantyClaim> ShipReplacementAsync(
        Guid claimId, CancellationToken ct = default);
}

public class WarrantyClaimService : IWarrantyClaimService
{
    private readonly IWarrantyRepository _warrantyRepo;
    private readonly IWarrantyClaimRepository _claimRepo;
    private readonly IManufacturerRmaService _mfrRmaService;
    private readonly IOrderService _orderService;

    public WarrantyClaimService(
        IWarrantyRepository warrantyRepo,
        IWarrantyClaimRepository claimRepo,
        IManufacturerRmaService mfrRmaService,
        IOrderService orderService)
    {
        _warrantyRepo = warrantyRepo;
        _claimRepo = claimRepo;
        _mfrRmaService = mfrRmaService;
        _orderService = orderService;
    }

    public async Task<WarrantyClaim> SubmitClaimAsync(
        Guid warrantyRegistrationId, string issueDescription,
        List<string>? photoUrls, CancellationToken ct)
    {
        var warranty = await _warrantyRepo.GetByIdAsync(warrantyRegistrationId, ct)
            ?? throw new InvalidOperationException("Warranty registration not found.");

        if (!warranty.IsActive)
            throw new InvalidOperationException(
                $"Warranty expired on {warranty.WarrantyEndDate:yyyy-MM-dd}.");

        var claim = new WarrantyClaim
        {
            Id = Guid.NewGuid(),
            WarrantyRegistrationId = warrantyRegistrationId,
            CustomerId = warranty.CustomerId,
            ClaimNumber = $"WC-{DateTime.UtcNow:yyyyMMdd}-{Random.Shared.Next(10000, 99999)}",
            Status = ClaimStatus.Submitted,
            IssueDescription = issueDescription,
            EvidencePhotoUrls = photoUrls ?? new List<string>(),
            SubmittedAt = DateTime.UtcNow
        };

        await _claimRepo.SaveAsync(claim, ct);
        return claim;
    }

    public async Task<WarrantyClaim> ReviewClaimAsync(
        Guid claimId, bool approved, string notes, CancellationToken ct)
    {
        var claim = await _claimRepo.GetByIdAsync(claimId, ct)
            ?? throw new InvalidOperationException("Claim not found.");

        claim.Status = approved ? ClaimStatus.Approved : ClaimStatus.Denied;
        claim.ResolutionNotes = notes;

        if (!approved)
            claim.ResolvedAt = DateTime.UtcNow;

        await _claimRepo.UpdateAsync(claim, ct);
        return claim;
    }

    public async Task<WarrantyClaim> InitiateRepairAsync(Guid claimId, CancellationToken ct)
    {
        var claim = await _claimRepo.GetByIdAsync(claimId, ct)
            ?? throw new InvalidOperationException("Claim not found.");

        var warranty = await _warrantyRepo.GetByIdAsync(claim.WarrantyRegistrationId, ct);

        // Route to manufacturer if under manufacturer warranty
        if (warranty!.Type == WarrantyType.ManufacturerLimited)
        {
            var mfrRma = await _mfrRmaService.CreateRmaAsync(
                warranty.SerialNumber, claim.IssueDescription, ct);
            claim.ManufacturerRmaNumber = mfrRma.RmaNumber;
        }

        claim.Status = ClaimStatus.RepairInProgress;
        await _claimRepo.UpdateAsync(claim, ct);
        return claim;
    }

    public async Task<WarrantyClaim> ShipReplacementAsync(
        Guid claimId, CancellationToken ct)
    {
        var claim = await _claimRepo.GetByIdAsync(claimId, ct)
            ?? throw new InvalidOperationException("Claim not found.");

        var warranty = await _warrantyRepo
            .GetByIdAsync(claim.WarrantyRegistrationId, ct);

        await _orderService.CreateReplacementOrderAsync(
            warranty!.CustomerId, warranty.Sku, 1, ct);

        claim.Status = ClaimStatus.ReplacementShipped;
        claim.ResolvedAt = DateTime.UtcNow;
        await _claimRepo.UpdateAsync(claim, ct);
        return claim;
    }
}
```

### Manufacturer RMA Integration

```csharp
namespace Returns.Infrastructure.Warranty;

public record ManufacturerRmaResponse(
    string RmaNumber,
    string ShippingInstructions,
    string? PrepaidLabelUrl,
    int EstimatedRepairDays);

public interface IManufacturerRmaService
{
    Task<ManufacturerRmaResponse> CreateRmaAsync(
        string serialNumber, string issueDescription,
        CancellationToken ct = default);
    Task<string> GetRepairStatusAsync(
        string rmaNumber, CancellationToken ct = default);
}

public class ManufacturerRmaService : IManufacturerRmaService
{
    private readonly HttpClient _httpClient;
    private readonly ManufacturerApiConfig _config;

    public ManufacturerRmaService(HttpClient httpClient, ManufacturerApiConfig config)
    {
        _httpClient = httpClient;
        _config = config;
    }

    public async Task<ManufacturerRmaResponse> CreateRmaAsync(
        string serialNumber, string issueDescription, CancellationToken ct)
    {
        var payload = new
        {
            serial_number = serialNumber,
            issue = issueDescription,
            dealer_id = _config.DealerId,
            return_address = _config.WarehouseAddress
        };

        var response = await _httpClient.PostAsJsonAsync(
            $"{_config.BaseUrl}/rma/create", payload, ct);

        response.EnsureSuccessStatusCode();

        return await response.Content
            .ReadFromJsonAsync<ManufacturerRmaResponse>(ct)
            ?? throw new InvalidOperationException("Invalid manufacturer response.");
    }

    public async Task<string> GetRepairStatusAsync(string rmaNumber, CancellationToken ct)
    {
        var response = await _httpClient.GetAsync(
            $"{_config.BaseUrl}/rma/{rmaNumber}/status", ct);

        response.EnsureSuccessStatusCode();
        var result = await response.Content.ReadFromJsonAsync<RmaStatusResponse>(ct);
        return result?.Status ?? "Unknown";
    }
}

public class ManufacturerApiConfig
{
    public string BaseUrl { get; set; } = default!;
    public string DealerId { get; set; } = default!;
    public string WarehouseAddress { get; set; } = default!;
}

public record RmaStatusResponse(string Status, string? EstimatedCompletion);
```

---

## 9. Analytics & Loss Prevention

Return analytics drive operational improvements and flag fraudulent patterns. Tracking return rates by category, identifying serial returners, and analyzing return reasons enable data-driven policy adjustments.

### Return Analytics Models

```csharp
namespace Returns.Application.Analytics;

public record ReturnRateReport(
    DateTime PeriodStart,
    DateTime PeriodEnd,
    int TotalOrders,
    int TotalReturns,
    decimal ReturnRate,
    decimal TotalRefundAmount,
    Dictionary<string, int> ReturnsByReason,
    Dictionary<string, decimal> ReturnRateByCategory);

public record CustomerReturnProfile(
    Guid CustomerId,
    int TotalOrders,
    int TotalReturns,
    decimal ReturnRate,
    decimal TotalRefundAmount,
    DateTime? LastReturnDate,
    List<string> TopReturnReasons,
    bool IsFlaggedAsHighRisk);
```

### Analytics Service

```csharp
namespace Returns.Application.Analytics;

public interface IReturnAnalyticsService
{
    Task<ReturnRateReport> GetReturnRateReportAsync(
        DateTime from, DateTime to, CancellationToken ct = default);
    Task<CustomerReturnProfile> GetCustomerProfileAsync(
        Guid customerId, CancellationToken ct = default);
    Task<IReadOnlyList<CustomerReturnProfile>> GetHighRiskCustomersAsync(
        decimal returnRateThreshold, int minimumOrders,
        CancellationToken ct = default);
    Task<ReturnCostAnalysis> GetCostAnalysisAsync(
        DateTime from, DateTime to, CancellationToken ct = default);
}

public record ReturnCostAnalysis(
    decimal TotalRefunds,
    decimal TotalRestockingFeesCollected,
    decimal TotalShippingCosts,
    decimal TotalProcessingCosts,
    decimal NetReturnCost,
    decimal AverageCostPerReturn,
    Dictionary<string, decimal> CostByCategory);

public class ReturnAnalyticsService : IReturnAnalyticsService
{
    private readonly IReturnRepository _returnRepository;
    private readonly IOrderRepository _orderRepository;

    public ReturnAnalyticsService(
        IReturnRepository returnRepository,
        IOrderRepository orderRepository)
    {
        _returnRepository = returnRepository;
        _orderRepository = orderRepository;
    }

    public async Task<ReturnRateReport> GetReturnRateReportAsync(
        DateTime from, DateTime to, CancellationToken ct)
    {
        var orders = await _orderRepository.GetOrderCountAsync(from, to, ct);
        var returns = await _returnRepository.GetReturnsInPeriodAsync(from, to, ct);

        var returnsByReason = returns
            .SelectMany(r => r.Items)
            .GroupBy(i => i.Reason.ToString())
            .ToDictionary(g => g.Key, g => g.Count());

        var returnsByCategory = returns
            .SelectMany(r => r.Items)
            .GroupBy(i => i.Sku[..3]) // Category prefix
            .ToDictionary(
                g => g.Key,
                g => orders > 0 ? (decimal)g.Count() / orders * 100 : 0);

        var totalRefunds = returns
            .SelectMany(r => r.Items)
            .Sum(i => i.RefundAmount ?? 0m);

        return new ReturnRateReport(
            PeriodStart: from,
            PeriodEnd: to,
            TotalOrders: orders,
            TotalReturns: returns.Count,
            ReturnRate: orders > 0 ? (decimal)returns.Count / orders * 100 : 0,
            TotalRefundAmount: totalRefunds,
            ReturnsByReason: returnsByReason,
            ReturnRateByCategory: returnsByCategory);
    }

    public async Task<CustomerReturnProfile> GetCustomerProfileAsync(
        Guid customerId, CancellationToken ct)
    {
        var customerOrders = await _orderRepository
            .GetCustomerOrderCountAsync(customerId, ct);
        var customerReturns = await _returnRepository
            .GetByCustomerIdAsync(customerId, ct);

        var returnRate = customerOrders > 0
            ? (decimal)customerReturns.Count / customerOrders * 100
            : 0;

        var topReasons = customerReturns
            .SelectMany(r => r.Items)
            .GroupBy(i => i.Reason.ToString())
            .OrderByDescending(g => g.Count())
            .Take(3)
            .Select(g => g.Key)
            .ToList();

        return new CustomerReturnProfile(
            CustomerId: customerId,
            TotalOrders: customerOrders,
            TotalReturns: customerReturns.Count,
            ReturnRate: returnRate,
            TotalRefundAmount: customerReturns
                .SelectMany(r => r.Items)
                .Sum(i => i.RefundAmount ?? 0m),
            LastReturnDate: customerReturns
                .MaxBy(r => r.CreatedAt)?.CreatedAt,
            TopReturnReasons: topReasons,
            IsFlaggedAsHighRisk: returnRate > 30 && customerOrders >= 5);
    }

    public async Task<IReadOnlyList<CustomerReturnProfile>> GetHighRiskCustomersAsync(
        decimal returnRateThreshold, int minimumOrders, CancellationToken ct)
    {
        var allCustomerIds = await _returnRepository.GetDistinctCustomerIdsAsync(ct);
        var highRisk = new List<CustomerReturnProfile>();

        foreach (var customerId in allCustomerIds)
        {
            var profile = await GetCustomerProfileAsync(customerId, ct);

            if (profile.TotalOrders >= minimumOrders
                && profile.ReturnRate >= returnRateThreshold)
            {
                highRisk.Add(profile);
            }
        }

        return highRisk.OrderByDescending(p => p.ReturnRate).ToList();
    }

    public async Task<ReturnCostAnalysis> GetCostAnalysisAsync(
        DateTime from, DateTime to, CancellationToken ct)
    {
        var returns = await _returnRepository.GetReturnsInPeriodAsync(from, to, ct);

        var totalRefunds = returns
            .SelectMany(r => r.Items).Sum(i => i.RefundAmount ?? 0m);
        var restockingFees = returns
            .SelectMany(r => r.Items).Sum(i => i.RestockingFee ?? 0m);

        // Estimated processing cost per return
        var processingCostPerReturn = 8.50m;
        var shippingCostPerReturn = 7.25m;

        var totalProcessing = returns.Count * processingCostPerReturn;
        var totalShipping = returns.Count * shippingCostPerReturn;

        return new ReturnCostAnalysis(
            TotalRefunds: totalRefunds,
            TotalRestockingFeesCollected: restockingFees,
            TotalShippingCosts: totalShipping,
            TotalProcessingCosts: totalProcessing,
            NetReturnCost: totalRefunds + totalShipping + totalProcessing - restockingFees,
            AverageCostPerReturn: returns.Count > 0
                ? (totalRefunds + totalShipping + totalProcessing) / returns.Count
                : 0,
            CostByCategory: new Dictionary<string, decimal>());
    }
}
```

### Fraud Pattern Detection

```csharp
namespace Returns.Application.Analytics;

public record FraudSignal(
    Guid CustomerId,
    string SignalType,
    string Description,
    decimal Severity,
    DateTime DetectedAt);

public interface IReturnFraudDetector
{
    Task<IReadOnlyList<FraudSignal>> AnalyzeCustomerAsync(
        Guid customerId, CancellationToken ct = default);
}

public class ReturnFraudDetector : IReturnFraudDetector
{
    private readonly IReturnRepository _returnRepository;
    private readonly IOrderRepository _orderRepository;

    public ReturnFraudDetector(
        IReturnRepository returnRepository, IOrderRepository orderRepository)
    {
        _returnRepository = returnRepository;
        _orderRepository = orderRepository;
    }

    public async Task<IReadOnlyList<FraudSignal>> AnalyzeCustomerAsync(
        Guid customerId, CancellationToken ct)
    {
        var signals = new List<FraudSignal>();
        var returns = await _returnRepository.GetByCustomerIdAsync(customerId, ct);
        var now = DateTime.UtcNow;

        // Pattern: High-value items returned just before window closes
        var lastMinuteReturns = returns.Where(r =>
        {
            var daysBeforeDeadline = (r.ReturnWindowDeadline - r.CreatedAt).Days;
            return daysBeforeDeadline <= 2
                && r.Items.Any(i => i.UnitPrice > 200);
        }).ToList();

        if (lastMinuteReturns.Count >= 3)
        {
            signals.Add(new FraudSignal(
                customerId, "LastMinuteHighValue",
                $"Customer has {lastMinuteReturns.Count} last-minute returns of high-value items.",
                0.7m, now));
        }

        // Pattern: Returning different items than ordered (wardrobing)
        var usedReturns = returns.Count(r =>
            r.Items.Any(i => i.ReceivedCondition == ItemCondition.Used));

        if (usedReturns >= 5)
        {
            signals.Add(new FraudSignal(
                customerId, "Wardrobing",
                $"Customer has returned {usedReturns} items in used condition.",
                0.8m, now));
        }

        // Pattern: Repeated returns of the same SKU
        var skuCounts = returns
            .SelectMany(r => r.Items)
            .GroupBy(i => i.Sku)
            .Where(g => g.Count() >= 3);

        foreach (var group in skuCounts)
        {
            signals.Add(new FraudSignal(
                customerId, "RepeatedSkuReturn",
                $"SKU {group.Key} returned {group.Count()} times.",
                0.6m, now));
        }

        // Pattern: Burst of returns in short period
        var last30DaysReturns = returns
            .Where(r => r.CreatedAt >= now.AddDays(-30)).ToList();

        if (last30DaysReturns.Count >= 8)
        {
            signals.Add(new FraudSignal(
                customerId, "ReturnBurst",
                $"{last30DaysReturns.Count} returns in the last 30 days.",
                0.9m, now));
        }

        return signals;
    }
}
```

---

## 10. Customer Communication

Clear communication throughout the return lifecycle reduces support contacts and improves customer satisfaction. Templates cover every status transition, and a self-service portal empowers customers to initiate and track returns independently.

### Notification Templates

```csharp
namespace Returns.Application.Notifications;

using Returns.Domain.Entities;

public interface IReturnNotificationService
{
    Task SendReturnConfirmationAsync(ReturnRequest request, CancellationToken ct = default);
    Task SendReturnApprovedAsync(ReturnRequest request, CancellationToken ct = default);
    Task SendReturnRejectedAsync(ReturnRequest request, CancellationToken ct = default);
    Task SendReturnReceivedAsync(ReturnRequest request, CancellationToken ct = default);
    Task SendRefundProcessedAsync(ReturnRequest request, decimal refundAmount,
        string refundMethod, CancellationToken ct = default);
    Task SendExchangeShippedAsync(ReturnRequest request, string exchangeTrackingNumber,
        CancellationToken ct = default);
}

public class ReturnNotificationService : IReturnNotificationService
{
    private readonly IEmailService _emailService;
    private readonly ICustomerRepository _customerRepo;

    public ReturnNotificationService(
        IEmailService emailService, ICustomerRepository customerRepo)
    {
        _emailService = emailService;
        _customerRepo = customerRepo;
    }

    public async Task SendReturnConfirmationAsync(
        ReturnRequest request, CancellationToken ct)
    {
        var customer = await _customerRepo.GetByIdAsync(request.CustomerId, ct);

        await _emailService.SendTemplateAsync(
            to: customer!.Email,
            templateId: "return-confirmation",
            data: new Dictionary<string, object>
            {
                ["customer_name"] = customer.FullName,
                ["rma_number"] = request.RmaNumber,
                ["items"] = request.Items.Select(i => new
                {
                    i.ProductName,
                    i.Quantity,
                    i.Reason
                }),
                ["return_by_date"] = request.ReturnWindowDeadline.ToString("MMMM dd, yyyy"),
                ["tracking_url"] = $"https://returns.example.com/track/{request.RmaNumber}"
            }, ct);
    }

    public async Task SendReturnApprovedAsync(
        ReturnRequest request, CancellationToken ct)
    {
        var customer = await _customerRepo.GetByIdAsync(request.CustomerId, ct);

        await _emailService.SendTemplateAsync(
            to: customer!.Email,
            templateId: "return-approved",
            data: new Dictionary<string, object>
            {
                ["customer_name"] = customer.FullName,
                ["rma_number"] = request.RmaNumber,
                ["return_label_url"] = request.ReturnLabelUrl ?? "",
                ["tracking_number"] = request.TrackingNumber ?? "",
                ["instructions"] = "Please print the label and attach it to your package."
            }, ct);
    }

    public async Task SendReturnRejectedAsync(
        ReturnRequest request, CancellationToken ct)
    {
        var customer = await _customerRepo.GetByIdAsync(request.CustomerId, ct);

        await _emailService.SendTemplateAsync(
            to: customer!.Email,
            templateId: "return-rejected",
            data: new Dictionary<string, object>
            {
                ["customer_name"] = customer.FullName,
                ["rma_number"] = request.RmaNumber,
                ["reason"] = request.InternalNotes ?? "Policy requirements not met.",
                ["support_url"] = "https://support.example.com/returns"
            }, ct);
    }

    public async Task SendReturnReceivedAsync(
        ReturnRequest request, CancellationToken ct)
    {
        var customer = await _customerRepo.GetByIdAsync(request.CustomerId, ct);

        await _emailService.SendTemplateAsync(
            to: customer!.Email,
            templateId: "return-received",
            data: new Dictionary<string, object>
            {
                ["customer_name"] = customer.FullName,
                ["rma_number"] = request.RmaNumber,
                ["received_date"] = request.ReceivedAt?.ToString("MMMM dd, yyyy") ?? "",
                ["next_steps"] = "We are inspecting your return. You will receive an update within 2-3 business days."
            }, ct);
    }

    public async Task SendRefundProcessedAsync(
        ReturnRequest request, decimal refundAmount,
        string refundMethod, CancellationToken ct)
    {
        var customer = await _customerRepo.GetByIdAsync(request.CustomerId, ct);

        await _emailService.SendTemplateAsync(
            to: customer!.Email,
            templateId: "refund-processed",
            data: new Dictionary<string, object>
            {
                ["customer_name"] = customer.FullName,
                ["rma_number"] = request.RmaNumber,
                ["refund_amount"] = refundAmount.ToString("C"),
                ["refund_method"] = refundMethod,
                ["estimated_arrival"] = refundMethod == "StoreCredit"
                    ? "Immediately"
                    : "5-10 business days"
            }, ct);
    }

    public async Task SendExchangeShippedAsync(
        ReturnRequest request, string exchangeTrackingNumber, CancellationToken ct)
    {
        var customer = await _customerRepo.GetByIdAsync(request.CustomerId, ct);

        await _emailService.SendTemplateAsync(
            to: customer!.Email,
            templateId: "exchange-shipped",
            data: new Dictionary<string, object>
            {
                ["customer_name"] = customer.FullName,
                ["rma_number"] = request.RmaNumber,
                ["tracking_number"] = exchangeTrackingNumber,
                ["tracking_url"] = $"https://track.example.com/{exchangeTrackingNumber}"
            }, ct);
    }
}
```

### Self-Service Return Portal API

```csharp
namespace Returns.Api.Controllers;

using Microsoft.AspNetCore.Mvc;
using Returns.Application.Services;
using Returns.Domain.Eligibility;
using Returns.Domain.Entities;

[ApiController]
[Route("api/returns")]
public class ReturnsController : ControllerBase
{
    private readonly IReturnService _returnService;
    private readonly ReturnEligibilityEngine _eligibilityEngine;
    private readonly IReturnWorkflowService _workflowService;

    public ReturnsController(
        IReturnService returnService,
        ReturnEligibilityEngine eligibilityEngine,
        IReturnWorkflowService workflowService)
    {
        _returnService = returnService;
        _eligibilityEngine = eligibilityEngine;
        _workflowService = workflowService;
    }

    [HttpPost("check-eligibility")]
    public async Task<ActionResult<EligibilityResult>> CheckEligibility(
        [FromBody] CheckEligibilityRequest request, CancellationToken ct)
    {
        var context = await _returnService
            .BuildEligibilityContextAsync(request.OrderId, request.ItemIds, ct);

        var result = await _eligibilityEngine.EvaluateAsync(context, ct);
        return Ok(result);
    }

    [HttpPost]
    public async Task<ActionResult<ReturnResponse>> CreateReturn(
        [FromBody] CreateReturnRequest request, CancellationToken ct)
    {
        var context = await _returnService
            .BuildEligibilityContextAsync(request.OrderId, request.ItemIds, ct);

        var eligibility = await _eligibilityEngine.EvaluateAsync(context, ct);
        if (!eligibility.IsEligible)
            return BadRequest(new { error = eligibility.DenialReason });

        var returnRequest = await _returnService.CreateReturnAsync(
            request.OrderId, request.CustomerId, request.Items,
            request.Resolution, request.CustomerNotes, ct);

        return CreatedAtAction(
            nameof(GetReturn),
            new { rmaNumber = returnRequest.RmaNumber },
            MapToResponse(returnRequest));
    }

    [HttpGet("{rmaNumber}")]
    public async Task<ActionResult<ReturnResponse>> GetReturn(
        string rmaNumber, CancellationToken ct)
    {
        var returnRequest = await _returnService
            .GetByRmaNumberAsync(rmaNumber, ct);

        if (returnRequest is null)
            return NotFound();

        return Ok(MapToResponse(returnRequest));
    }

    [HttpGet("{rmaNumber}/status")]
    public async Task<ActionResult<ReturnStatusResponse>> GetReturnStatus(
        string rmaNumber, CancellationToken ct)
    {
        var returnRequest = await _returnService
            .GetByRmaNumberAsync(rmaNumber, ct);

        if (returnRequest is null)
            return NotFound();

        return Ok(new ReturnStatusResponse(
            returnRequest.RmaNumber,
            returnRequest.Status.ToString(),
            returnRequest.StatusHistory
                .OrderByDescending(h => h.Timestamp)
                .Select(h => new StatusEntry(h.Status.ToString(), h.Description, h.Timestamp))
                .ToList()));
    }

    [HttpPost("{rmaNumber}/cancel")]
    public async Task<ActionResult> CancelReturn(
        string rmaNumber, CancellationToken ct)
    {
        var returnRequest = await _returnService.GetByRmaNumberAsync(rmaNumber, ct);

        if (returnRequest is null)
            return NotFound();

        if (returnRequest.Status != ReturnStatus.Requested
            && returnRequest.Status != ReturnStatus.Approved)
        {
            return BadRequest(new { error = "Return cannot be cancelled in its current status." });
        }

        await _returnService.CancelReturnAsync(returnRequest.Id, ct);
        return NoContent();
    }

    private static ReturnResponse MapToResponse(ReturnRequest request) => new(
        request.RmaNumber,
        request.Status.ToString(),
        request.RequestedResolution.ToString(),
        request.CreatedAt,
        request.TrackingNumber,
        request.ReturnLabelUrl,
        request.Items.Select(i => new ReturnItemResponse(
            i.ProductName, i.Quantity, i.Reason.ToString(),
            i.RefundAmount)).ToList());
}

// API DTOs
public record CheckEligibilityRequest(Guid OrderId, List<Guid> ItemIds);

public record CreateReturnRequest(
    Guid OrderId,
    Guid CustomerId,
    List<CreateReturnItemRequest> Items,
    ResolutionType Resolution,
    string? CustomerNotes,
    List<Guid> ItemIds);

public record CreateReturnItemRequest(
    Guid OrderLineId, string Sku, string ProductName,
    int Quantity, decimal UnitPrice, ReturnReason Reason,
    string? ReasonDetail);

public record ReturnResponse(
    string RmaNumber, string Status, string Resolution,
    DateTime CreatedAt, string? TrackingNumber, string? ReturnLabelUrl,
    List<ReturnItemResponse> Items);

public record ReturnItemResponse(
    string ProductName, int Quantity, string Reason, decimal? RefundAmount);

public record ReturnStatusResponse(
    string RmaNumber, string CurrentStatus, List<StatusEntry> History);

public record StatusEntry(string Status, string Description, DateTime Timestamp);
```

---

## 11. Implementation Checklist

### Phase 1: Core Return Flow (Weeks 1–3)

- [ ] Define return data model (ReturnRequest, ReturnItem, ReturnStatusHistory)
- [ ] Implement EF Core configurations and migrations
- [ ] Build RMA state machine with valid transitions
- [ ] Create return eligibility engine with base rules (return window, final sale)
- [ ] Implement return request creation API endpoint
- [ ] Set up domain event infrastructure for return lifecycle
- [ ] Add return status query endpoint

### Phase 2: Shipping & Inspection (Weeks 4–6)

- [ ] Integrate carrier API for prepaid return label generation
- [ ] Build tracking webhook handler for status updates
- [ ] Implement warehouse receiving workflow
- [ ] Build inspection service with condition grading
- [ ] Implement disposition engine (restock, refurbish, dispose)
- [ ] Add cross-border return label support
- [ ] Create inspection photo upload endpoint

### Phase 3: Refund & Exchange (Weeks 7–9)

- [ ] Build refund calculator with restocking fees and tax
- [ ] Implement multi-provider refund routing (Stripe, store credit)
- [ ] Create store credit issuance service
- [ ] Build exchange service with price difference handling
- [ ] Implement instant exchange with payment hold
- [ ] Add hold reconciliation for expired exchanges
- [ ] Integrate partial refund support

### Phase 4: Warranty & Advanced Features (Weeks 10–12)

- [ ] Build warranty registration system
- [ ] Implement warranty claim submission and review
- [ ] Integrate manufacturer RMA API for repair routing
- [ ] Add replacement shipment workflow
- [ ] Implement warranty extension support
- [ ] Build warranty expiration notification scheduler

### Phase 5: Analytics & Communication (Weeks 13–15)

- [ ] Build return rate analytics dashboard queries
- [ ] Implement customer return profile scoring
- [ ] Deploy fraud pattern detection rules
- [ ] Create email notification templates for all return statuses
- [ ] Build self-service return portal API
- [ ] Add return status tracking page support
- [ ] Implement return cost analysis reporting

### Phase 6: Hardening & Operations (Weeks 16–18)

- [ ] Load test return creation and refund processing
- [ ] Add idempotency keys to refund operations
- [ ] Implement retry policies for carrier and payment APIs
- [ ] Set up monitoring alerts for refund failures and SLA breaches
- [ ] Configure rate limiting on return creation endpoints
- [ ] Add audit logging for all return state transitions
- [ ] Document runbooks for manual return intervention

---

## Additional Resources

- [Stripe Refund API Documentation](https://stripe.com/docs/refunds)
- [EasyPost Returns API](https://www.easypost.com/docs/api#returns)
- [Shopify Returns Management](https://shopify.dev/docs/apps/fulfillment/returns)
- [Domain-Driven Design — Eric Evans](https://www.domainlanguage.com/ddd/)
- [Saga Pattern for Distributed Transactions](https://microservices.io/patterns/data/saga.html)

---

> **Next Steps:**
>
> - Review [18-COMMON-SCENARIOS](18-COMMON-SCENARIOS.md) for return/refund scenario walkthroughs.
> - See [22-EVENT-DRIVEN-ARCHITECTURE](22-EVENT-DRIVEN-ARCHITECTURE.md) for domain event patterns.
> - Refer to [16-STRIPE-INTEGRATION](16-STRIPE-INTEGRATION.md) for Stripe refund API details.
> - Check [05-ERROR-HANDLING](05-ERROR-HANDLING.md) for resilient error handling in return workflows.
