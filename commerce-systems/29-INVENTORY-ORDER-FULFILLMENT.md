# Inventory Management and Order Fulfillment

## Overview

Comprehensive guide to building inventory management and order fulfillment systems for e-commerce platforms. Covers the full lifecycle from stock tracking and allocation through order processing, warehouse management, shipping integration, and returns handling — all with .NET implementation examples using domain-driven design, EF Core, and event-driven patterns.

> **Related documents:**
>
> - [06-DATABASE-DESIGN](06-DATABASE-DESIGN.md) — Database schemas
> - [22-EVENT-DRIVEN-ARCHITECTURE](22-EVENT-DRIVEN-ARCHITECTURE.md) — Domain events
> - [18-COMMON-SCENARIOS](18-COMMON-SCENARIOS.md) — Checkout flows
> - [23-MICROSERVICES-PATTERNS](23-MICROSERVICES-PATTERNS.md) — Service decomposition

## Table of Contents

1. [Inventory Data Model](#1-inventory-data-model)
2. [Stock Allocation and Reservation](#2-stock-allocation-and-reservation)
3. [Inventory Tracking and Adjustments](#3-inventory-tracking-and-adjustments)
4. [Order Lifecycle State Machine](#4-order-lifecycle-state-machine)
5. [Order Fulfillment Workflow](#5-order-fulfillment-workflow)
6. [Shipping Integration](#6-shipping-integration)
7. [Warehouse Management Patterns](#7-warehouse-management-patterns)
8. [Real-Time Inventory Sync](#8-real-time-inventory-sync)
9. [Backorder and Pre-Order Management](#9-backorder-and-pre-order-management)
10. [Returns Impact on Inventory](#10-returns-impact-on-inventory)
11. [Reporting and Analytics](#11-reporting-and-analytics)
12. [Implementation Checklist](#12-implementation-checklist)

---

## 1. Inventory Data Model

Inventory systems begin with a well-structured data model that represents products, variants, physical locations, and current stock levels. The model must capture the distinction between different inventory states (available, reserved, damaged, in-transit) and support multi-warehouse topologies.

### SKU and Variant Entities

A **SKU** (Stock Keeping Unit) uniquely identifies a specific product variant — a combination of product attributes like size, color, and material. Each SKU maps to a single purchasable unit.

```csharp
namespace Inventory.Domain.Entities;

public class Product
{
    public Guid Id { get; private set; }
    public string Name { get; private set; } = string.Empty;
    public string Brand { get; private set; } = string.Empty;
    public string Category { get; private set; } = string.Empty;
    public bool IsActive { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime UpdatedAt { get; private set; }

    private readonly List<ProductVariant> _variants = new();
    public IReadOnlyCollection<ProductVariant> Variants => _variants.AsReadOnly();

    public static Product Create(string name, string brand, string category)
    {
        return new Product
        {
            Id = Guid.NewGuid(),
            Name = name,
            Brand = brand,
            Category = category,
            IsActive = true,
            CreatedAt = DateTime.UtcNow,
            UpdatedAt = DateTime.UtcNow
        };
    }

    public ProductVariant AddVariant(string sku, string size, string color, decimal weight)
    {
        if (_variants.Any(v => v.Sku == sku))
            throw new InvalidOperationException($"SKU {sku} already exists on this product.");

        var variant = ProductVariant.Create(Id, sku, size, color, weight);
        _variants.Add(variant);
        UpdatedAt = DateTime.UtcNow;
        return variant;
    }
}

public class ProductVariant
{
    public Guid Id { get; private set; }
    public Guid ProductId { get; private set; }
    public string Sku { get; private set; } = string.Empty;
    public string Size { get; private set; } = string.Empty;
    public string Color { get; private set; } = string.Empty;
    public decimal WeightKg { get; private set; }
    public string? Barcode { get; private set; }
    public bool IsActive { get; private set; }
    public DateTime CreatedAt { get; private set; }

    public Product Product { get; private set; } = null!;
    private readonly List<InventoryRecord> _inventoryRecords = new();
    public IReadOnlyCollection<InventoryRecord> InventoryRecords => _inventoryRecords.AsReadOnly();

    public static ProductVariant Create(
        Guid productId, string sku, string size, string color, decimal weight)
    {
        return new ProductVariant
        {
            Id = Guid.NewGuid(),
            ProductId = productId,
            Sku = sku,
            Size = size,
            Color = color,
            WeightKg = weight,
            IsActive = true,
            CreatedAt = DateTime.UtcNow
        };
    }
}
```

### Warehouse and Location Model

Physical inventory is tracked per warehouse and per location (aisle, shelf, bin) within each warehouse:

```csharp
public class Warehouse
{
    public Guid Id { get; private set; }
    public string Code { get; private set; } = string.Empty;
    public string Name { get; private set; } = string.Empty;
    public Address Address { get; private set; } = null!;
    public bool IsActive { get; private set; }
    public int Priority { get; private set; } // lower = preferred for fulfillment

    private readonly List<WarehouseLocation> _locations = new();
    public IReadOnlyCollection<WarehouseLocation> Locations => _locations.AsReadOnly();

    public static Warehouse Create(string code, string name, Address address, int priority)
    {
        return new Warehouse
        {
            Id = Guid.NewGuid(),
            Code = code,
            Name = name,
            Address = address,
            IsActive = true,
            Priority = priority
        };
    }
}

public class WarehouseLocation
{
    public Guid Id { get; private set; }
    public Guid WarehouseId { get; private set; }
    public string Aisle { get; private set; } = string.Empty;
    public string Shelf { get; private set; } = string.Empty;
    public string Bin { get; private set; } = string.Empty;
    public LocationType Type { get; private set; }

    public string FullCode => $"{Aisle}-{Shelf}-{Bin}";
    public Warehouse Warehouse { get; private set; } = null!;
}

public enum LocationType
{
    Pickable,
    Bulk,
    Receiving,
    Staging,
    Returns
}

public record Address(
    string Street,
    string City,
    string State,
    string PostalCode,
    string Country,
    double Latitude,
    double Longitude);
```

### Inventory Record with Status Tracking

Each `InventoryRecord` represents the stock of a specific SKU at a specific warehouse location, broken down by status:

```csharp
public class InventoryRecord
{
    public Guid Id { get; private set; }
    public Guid VariantId { get; private set; }
    public Guid WarehouseId { get; private set; }
    public Guid? LocationId { get; private set; }

    public int QuantityOnHand { get; private set; }
    public int QuantityReserved { get; private set; }
    public int QuantityDamaged { get; private set; }
    public int QuantityInTransit { get; private set; }

    public int QuantityAvailable =>
        QuantityOnHand - QuantityReserved - QuantityDamaged;

    public uint RowVersion { get; private set; } // optimistic concurrency

    public ProductVariant Variant { get; private set; } = null!;
    public Warehouse Warehouse { get; private set; } = null!;
    public WarehouseLocation? Location { get; private set; }

    public static InventoryRecord CreateNew(Guid variantId, Guid warehouseId)
    {
        return new InventoryRecord
        {
            Id = Guid.NewGuid(),
            VariantId = variantId,
            WarehouseId = warehouseId,
            QuantityOnHand = 0,
            QuantityReserved = 0,
            QuantityDamaged = 0,
            QuantityInTransit = 0
        };
    }

    public void Reserve(int quantity)
    {
        if (quantity > QuantityAvailable)
            throw new InsufficientStockException(VariantId, QuantityAvailable, quantity);

        QuantityReserved += quantity;
    }

    public void ReleaseReservation(int quantity)
    {
        QuantityReserved = Math.Max(0, QuantityReserved - quantity);
    }

    public void ConfirmOutbound(int quantity)
    {
        if (quantity > QuantityReserved)
            throw new InvalidOperationException("Cannot ship more than reserved.");

        QuantityOnHand -= quantity;
        QuantityReserved -= quantity;
    }

    public void ReceiveInbound(int quantity)
    {
        QuantityOnHand += quantity;
        QuantityInTransit = Math.Max(0, QuantityInTransit - quantity);
    }

    public void MarkDamaged(int quantity)
    {
        if (quantity > QuantityAvailable)
            throw new InvalidOperationException("Not enough available stock to mark as damaged.");

        QuantityDamaged += quantity;
    }
}

public class InsufficientStockException : Exception
{
    public Guid VariantId { get; }
    public int Available { get; }
    public int Requested { get; }

    public InsufficientStockException(Guid variantId, int available, int requested)
        : base($"Insufficient stock for variant {variantId}. Available: {available}, Requested: {requested}")
    {
        VariantId = variantId;
        Available = available;
        Requested = requested;
    }
}
```

### EF Core Configuration with Indexes

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace Inventory.Infrastructure.Persistence;

public class InventoryDbContext : DbContext
{
    public DbSet<Product> Products => Set<Product>();
    public DbSet<ProductVariant> Variants => Set<ProductVariant>();
    public DbSet<Warehouse> Warehouses => Set<Warehouse>();
    public DbSet<WarehouseLocation> WarehouseLocations => Set<WarehouseLocation>();
    public DbSet<InventoryRecord> InventoryRecords => Set<InventoryRecord>();

    public InventoryDbContext(DbContextOptions<InventoryDbContext> options)
        : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(InventoryDbContext).Assembly);
    }
}

public class ProductVariantConfiguration : IEntityTypeConfiguration<ProductVariant>
{
    public void Configure(EntityTypeBuilder<ProductVariant> builder)
    {
        builder.ToTable("product_variants");
        builder.HasKey(v => v.Id);
        builder.Property(v => v.Sku).HasMaxLength(50).IsRequired();
        builder.Property(v => v.Size).HasMaxLength(20);
        builder.Property(v => v.Color).HasMaxLength(30);
        builder.Property(v => v.Barcode).HasMaxLength(50);
        builder.Property(v => v.WeightKg).HasPrecision(8, 3);

        builder.HasIndex(v => v.Sku).IsUnique();
        builder.HasIndex(v => v.Barcode).IsUnique().HasFilter("barcode IS NOT NULL");
        builder.HasIndex(v => v.ProductId);

        builder.HasOne(v => v.Product)
            .WithMany(p => p.Variants)
            .HasForeignKey(v => v.ProductId);
    }
}

public class InventoryRecordConfiguration : IEntityTypeConfiguration<InventoryRecord>
{
    public void Configure(EntityTypeBuilder<InventoryRecord> builder)
    {
        builder.ToTable("inventory_records");
        builder.HasKey(r => r.Id);

        builder.Property(r => r.RowVersion)
            .IsRowVersion();

        builder.HasIndex(r => new { r.VariantId, r.WarehouseId })
            .IsUnique()
            .HasFilter("location_id IS NULL");

        builder.HasIndex(r => new { r.VariantId, r.WarehouseId, r.LocationId })
            .IsUnique()
            .HasFilter("location_id IS NOT NULL");

        builder.HasIndex(r => r.WarehouseId);

        builder.HasOne(r => r.Variant)
            .WithMany(v => v.InventoryRecords)
            .HasForeignKey(r => r.VariantId);

        builder.HasOne(r => r.Warehouse)
            .WithMany()
            .HasForeignKey(r => r.WarehouseId);

        builder.HasOne(r => r.Location)
            .WithMany()
            .HasForeignKey(r => r.LocationId);
    }
}

public class WarehouseConfiguration : IEntityTypeConfiguration<Warehouse>
{
    public void Configure(EntityTypeBuilder<Warehouse> builder)
    {
        builder.ToTable("warehouses");
        builder.HasKey(w => w.Id);
        builder.Property(w => w.Code).HasMaxLength(10).IsRequired();
        builder.HasIndex(w => w.Code).IsUnique();

        builder.OwnsOne(w => w.Address, a =>
        {
            a.Property(x => x.Street).HasColumnName("street");
            a.Property(x => x.City).HasColumnName("city");
            a.Property(x => x.State).HasColumnName("state");
            a.Property(x => x.PostalCode).HasColumnName("postal_code");
            a.Property(x => x.Country).HasColumnName("country");
            a.Property(x => x.Latitude).HasColumnName("latitude");
            a.Property(x => x.Longitude).HasColumnName("longitude");
        });
    }
}
```

---

## 2. Stock Allocation and Reservation

Stock reservation prevents overselling by temporarily holding inventory when a customer begins checkout. Reservations must be time-limited to avoid deadlocking stock indefinitely.

### Reserve-on-Checkout Pattern

When a customer adds items to their cart and proceeds to checkout, the system reserves inventory. This reservation has a TTL (Time-To-Live) — if the checkout is not completed, the reservation expires and the stock is released.

```csharp
namespace Inventory.Domain.Entities;

public class StockReservation
{
    public Guid Id { get; private set; }
    public Guid VariantId { get; private set; }
    public Guid WarehouseId { get; private set; }
    public Guid OrderId { get; private set; }
    public int Quantity { get; private set; }
    public ReservationStatus Status { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime ExpiresAt { get; private set; }
    public DateTime? ConfirmedAt { get; private set; }
    public DateTime? ReleasedAt { get; private set; }

    public bool IsExpired => Status == ReservationStatus.Active && DateTime.UtcNow > ExpiresAt;

    public static StockReservation Create(
        Guid variantId, Guid warehouseId, Guid orderId, int quantity, TimeSpan ttl)
    {
        return new StockReservation
        {
            Id = Guid.NewGuid(),
            VariantId = variantId,
            WarehouseId = warehouseId,
            OrderId = orderId,
            Quantity = quantity,
            Status = ReservationStatus.Active,
            CreatedAt = DateTime.UtcNow,
            ExpiresAt = DateTime.UtcNow.Add(ttl)
        };
    }

    public void Confirm()
    {
        if (Status != ReservationStatus.Active)
            throw new InvalidOperationException($"Cannot confirm reservation in {Status} state.");

        Status = ReservationStatus.Confirmed;
        ConfirmedAt = DateTime.UtcNow;
    }

    public void Release()
    {
        if (Status is ReservationStatus.Confirmed or ReservationStatus.Released)
            return;

        Status = ReservationStatus.Released;
        ReleasedAt = DateTime.UtcNow;
    }
}

public enum ReservationStatus
{
    Active,
    Confirmed,
    Released,
    Expired
}
```

### Reservation Service with Optimistic Concurrency

The reservation service uses **optimistic concurrency** with row versioning to prevent overselling when multiple checkout sessions target the same SKU simultaneously:

```csharp
using Microsoft.EntityFrameworkCore;

namespace Inventory.Application.Services;

public interface IStockReservationService
{
    Task<StockReservation> ReserveAsync(
        Guid variantId, Guid orderId, int quantity, CancellationToken ct = default);
    Task ConfirmAsync(Guid reservationId, CancellationToken ct = default);
    Task ReleaseAsync(Guid reservationId, CancellationToken ct = default);
    Task ReleaseExpiredAsync(CancellationToken ct = default);
}

public class StockReservationService : IStockReservationService
{
    private readonly InventoryDbContext _db;
    private readonly IWarehouseSelector _warehouseSelector;
    private readonly TimeSpan _reservationTtl = TimeSpan.FromMinutes(15);
    private const int MaxRetries = 3;

    public StockReservationService(InventoryDbContext db, IWarehouseSelector warehouseSelector)
    {
        _db = db;
        _warehouseSelector = warehouseSelector;
    }

    public async Task<StockReservation> ReserveAsync(
        Guid variantId, Guid orderId, int quantity, CancellationToken ct = default)
    {
        for (int attempt = 0; attempt < MaxRetries; attempt++)
        {
            try
            {
                var warehouseId = await _warehouseSelector
                    .SelectAsync(variantId, quantity, ct);

                var record = await _db.InventoryRecords
                    .FirstOrDefaultAsync(r =>
                        r.VariantId == variantId &&
                        r.WarehouseId == warehouseId, ct)
                    ?? throw new InvalidOperationException(
                        $"No inventory record for variant {variantId} at warehouse {warehouseId}.");

                record.Reserve(quantity);

                var reservation = StockReservation.Create(
                    variantId, warehouseId, orderId, quantity, _reservationTtl);

                _db.Set<StockReservation>().Add(reservation);
                await _db.SaveChangesAsync(ct);

                return reservation;
            }
            catch (DbUpdateConcurrencyException) when (attempt < MaxRetries - 1)
            {
                // Another checkout beat us — reload and retry
                foreach (var entry in _db.ChangeTracker.Entries())
                    await entry.ReloadAsync(ct);
            }
        }

        throw new InsufficientStockException(variantId, 0, quantity);
    }

    public async Task ConfirmAsync(Guid reservationId, CancellationToken ct = default)
    {
        var reservation = await _db.Set<StockReservation>()
            .FirstOrDefaultAsync(r => r.Id == reservationId, ct)
            ?? throw new InvalidOperationException("Reservation not found.");

        reservation.Confirm();
        await _db.SaveChangesAsync(ct);
    }

    public async Task ReleaseAsync(Guid reservationId, CancellationToken ct = default)
    {
        var reservation = await _db.Set<StockReservation>()
            .FirstOrDefaultAsync(r => r.Id == reservationId, ct)
            ?? throw new InvalidOperationException("Reservation not found.");

        if (reservation.Status == ReservationStatus.Active)
        {
            var record = await _db.InventoryRecords
                .FirstAsync(r =>
                    r.VariantId == reservation.VariantId &&
                    r.WarehouseId == reservation.WarehouseId, ct);

            record.ReleaseReservation(reservation.Quantity);
        }

        reservation.Release();
        await _db.SaveChangesAsync(ct);
    }

    public async Task ReleaseExpiredAsync(CancellationToken ct = default)
    {
        var expired = await _db.Set<StockReservation>()
            .Where(r => r.Status == ReservationStatus.Active && r.ExpiresAt < DateTime.UtcNow)
            .ToListAsync(ct);

        foreach (var reservation in expired)
        {
            var record = await _db.InventoryRecords
                .FirstOrDefaultAsync(r =>
                    r.VariantId == reservation.VariantId &&
                    r.WarehouseId == reservation.WarehouseId, ct);

            record?.ReleaseReservation(reservation.Quantity);
            reservation.Release();
        }

        await _db.SaveChangesAsync(ct);
    }
}
```

### TTL-Based Reservation Expiry Background Job

A hosted service periodically sweeps for expired reservations:

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

namespace Inventory.Infrastructure.BackgroundJobs;

public class ReservationExpiryWorker : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<ReservationExpiryWorker> _logger;
    private readonly TimeSpan _interval = TimeSpan.FromMinutes(1);

    public ReservationExpiryWorker(
        IServiceScopeFactory scopeFactory,
        ILogger<ReservationExpiryWorker> logger)
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
                using var scope = _scopeFactory.CreateScope();
                var service = scope.ServiceProvider
                    .GetRequiredService<IStockReservationService>();

                await service.ReleaseExpiredAsync(stoppingToken);
                _logger.LogDebug("Reservation expiry sweep completed");
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error during reservation expiry sweep");
            }

            await Task.Delay(_interval, stoppingToken);
        }
    }
}
```

---

## 3. Inventory Tracking and Adjustments

Every stock change must be recorded as an auditable **stock movement**. This provides a complete audit trail and enables cycle counting, shrinkage detection, and inventory reconciliation.

### Stock Movement Entity

```csharp
namespace Inventory.Domain.Entities;

public class StockMovement
{
    public Guid Id { get; private set; }
    public Guid VariantId { get; private set; }
    public Guid WarehouseId { get; private set; }
    public MovementType Type { get; private set; }
    public int Quantity { get; private set; }
    public int BalanceAfter { get; private set; }
    public string? ReferenceNumber { get; private set; }
    public Guid? RelatedOrderId { get; private set; }
    public Guid? RelatedTransferId { get; private set; }
    public string Reason { get; private set; } = string.Empty;
    public string PerformedBy { get; private set; } = string.Empty;
    public DateTime CreatedAt { get; private set; }

    public static StockMovement Create(
        Guid variantId,
        Guid warehouseId,
        MovementType type,
        int quantity,
        int balanceAfter,
        string reason,
        string performedBy,
        string? referenceNumber = null,
        Guid? relatedOrderId = null,
        Guid? relatedTransferId = null)
    {
        return new StockMovement
        {
            Id = Guid.NewGuid(),
            VariantId = variantId,
            WarehouseId = warehouseId,
            Type = type,
            Quantity = quantity,
            BalanceAfter = balanceAfter,
            Reason = reason,
            PerformedBy = performedBy,
            ReferenceNumber = referenceNumber,
            RelatedOrderId = relatedOrderId,
            RelatedTransferId = relatedTransferId,
            CreatedAt = DateTime.UtcNow
        };
    }
}

public enum MovementType
{
    Inbound,       // purchase order received
    Outbound,      // order shipped
    Transfer,      // between warehouses
    Adjustment,    // cycle count correction
    Return,        // customer return
    Damaged,       // marked as damaged
    Shrinkage      // unexplained loss
}
```

### Inventory Adjustment Service

```csharp
namespace Inventory.Application.Services;

public interface IInventoryAdjustmentService
{
    Task ReceiveStockAsync(Guid variantId, Guid warehouseId, int quantity,
        string purchaseOrderNumber, string performedBy, CancellationToken ct = default);
    Task AdjustStockAsync(Guid variantId, Guid warehouseId, int actualCount,
        string reason, string performedBy, CancellationToken ct = default);
    Task RecordShrinkageAsync(Guid variantId, Guid warehouseId, int lostQuantity,
        string reason, string performedBy, CancellationToken ct = default);
}

public class InventoryAdjustmentService : IInventoryAdjustmentService
{
    private readonly InventoryDbContext _db;
    private readonly IEventPublisher _eventPublisher;

    public InventoryAdjustmentService(InventoryDbContext db, IEventPublisher eventPublisher)
    {
        _db = db;
        _eventPublisher = eventPublisher;
    }

    public async Task ReceiveStockAsync(
        Guid variantId, Guid warehouseId, int quantity,
        string purchaseOrderNumber, string performedBy, CancellationToken ct = default)
    {
        var record = await GetOrCreateRecordAsync(variantId, warehouseId, ct);
        record.ReceiveInbound(quantity);

        var movement = StockMovement.Create(
            variantId, warehouseId, MovementType.Inbound,
            quantity, record.QuantityOnHand,
            $"Received from PO {purchaseOrderNumber}", performedBy,
            referenceNumber: purchaseOrderNumber);

        _db.Set<StockMovement>().Add(movement);
        await _db.SaveChangesAsync(ct);

        await _eventPublisher.PublishAsync(new StockReceivedEvent(
            variantId, warehouseId, quantity, record.QuantityAvailable), ct);
    }

    public async Task AdjustStockAsync(
        Guid variantId, Guid warehouseId, int actualCount,
        string reason, string performedBy, CancellationToken ct = default)
    {
        var record = await GetOrCreateRecordAsync(variantId, warehouseId, ct);
        int difference = actualCount - record.QuantityOnHand;

        var movementType = difference >= 0 ? MovementType.Adjustment : MovementType.Shrinkage;
        record.ReceiveInbound(difference); // negative adjustments reduce count

        var movement = StockMovement.Create(
            variantId, warehouseId, movementType,
            difference, record.QuantityOnHand,
            $"Cycle count adjustment: {reason}", performedBy);

        _db.Set<StockMovement>().Add(movement);
        await _db.SaveChangesAsync(ct);
    }

    public async Task RecordShrinkageAsync(
        Guid variantId, Guid warehouseId, int lostQuantity,
        string reason, string performedBy, CancellationToken ct = default)
    {
        var record = await GetOrCreateRecordAsync(variantId, warehouseId, ct);
        record.ReceiveInbound(-lostQuantity);

        var movement = StockMovement.Create(
            variantId, warehouseId, MovementType.Shrinkage,
            -lostQuantity, record.QuantityOnHand,
            $"Shrinkage: {reason}", performedBy);

        _db.Set<StockMovement>().Add(movement);
        await _db.SaveChangesAsync(ct);
    }

    private async Task<InventoryRecord> GetOrCreateRecordAsync(
        Guid variantId, Guid warehouseId, CancellationToken ct)
    {
        var record = await _db.InventoryRecords
            .FirstOrDefaultAsync(r => r.VariantId == variantId && r.WarehouseId == warehouseId, ct);

        if (record is null)
        {
            record = InventoryRecord.CreateNew(variantId, warehouseId);
            _db.InventoryRecords.Add(record);
        }

        return record;
    }
}
```

### Cycle Counting

Cycle counting verifies physical inventory against system records on a rolling basis rather than performing a full physical inventory:

```csharp
namespace Inventory.Domain.Entities;

public class CycleCount
{
    public Guid Id { get; private set; }
    public Guid WarehouseId { get; private set; }
    public CycleCountStatus Status { get; private set; }
    public DateTime ScheduledDate { get; private set; }
    public DateTime? CompletedDate { get; private set; }
    public string AssignedTo { get; private set; } = string.Empty;

    private readonly List<CycleCountItem> _items = new();
    public IReadOnlyCollection<CycleCountItem> Items => _items.AsReadOnly();

    public void RecordCount(Guid variantId, int physicalCount)
    {
        var item = _items.FirstOrDefault(i => i.VariantId == variantId)
            ?? throw new InvalidOperationException($"Variant {variantId} not in this cycle count.");

        item.RecordPhysicalCount(physicalCount);
    }

    public void Complete()
    {
        if (_items.Any(i => !i.IsCounted))
            throw new InvalidOperationException("All items must be counted before completing.");

        Status = CycleCountStatus.Completed;
        CompletedDate = DateTime.UtcNow;
    }
}

public class CycleCountItem
{
    public Guid Id { get; private set; }
    public Guid CycleCountId { get; private set; }
    public Guid VariantId { get; private set; }
    public int SystemCount { get; private set; }
    public int? PhysicalCount { get; private set; }
    public int Variance => (PhysicalCount ?? SystemCount) - SystemCount;
    public bool IsCounted => PhysicalCount.HasValue;

    public void RecordPhysicalCount(int count)
    {
        PhysicalCount = count;
    }
}

public enum CycleCountStatus
{
    Scheduled,
    InProgress,
    Completed,
    Cancelled
}
```

---

## 4. Order Lifecycle State Machine

Orders progress through a series of well-defined states. Each transition is enforced to prevent invalid state changes and emits a domain event for downstream processing.

### Order Entity with State Machine

```csharp
namespace Inventory.Domain.Entities;

public class Order
{
    public Guid Id { get; private set; }
    public string OrderNumber { get; private set; } = string.Empty;
    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Address ShippingAddress { get; private set; } = null!;
    public decimal SubTotal { get; private set; }
    public decimal ShippingCost { get; private set; }
    public decimal TaxAmount { get; private set; }
    public decimal TotalAmount { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime UpdatedAt { get; private set; }

    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    private readonly List<DomainEvent> _domainEvents = new();
    public IReadOnlyCollection<DomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    public void ClearDomainEvents() => _domainEvents.Clear();

    public static Order Create(
        Guid customerId, Address shippingAddress, IEnumerable<OrderItem> items)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            OrderNumber = GenerateOrderNumber(),
            CustomerId = customerId,
            Status = OrderStatus.Created,
            ShippingAddress = shippingAddress,
            CreatedAt = DateTime.UtcNow,
            UpdatedAt = DateTime.UtcNow
        };

        foreach (var item in items)
            order._items.Add(item);

        order.RecalculateTotals();
        order._domainEvents.Add(new OrderCreatedEvent(order.Id, order.OrderNumber));
        return order;
    }

    public void Confirm()
    {
        EnsureValidTransition(OrderStatus.Confirmed);
        Status = OrderStatus.Confirmed;
        UpdatedAt = DateTime.UtcNow;
        _domainEvents.Add(new OrderConfirmedEvent(Id));
    }

    public void StartProcessing()
    {
        EnsureValidTransition(OrderStatus.Processing);
        Status = OrderStatus.Processing;
        UpdatedAt = DateTime.UtcNow;
        _domainEvents.Add(new OrderProcessingStartedEvent(Id));
    }

    public void MarkShipped(string trackingNumber, string carrier)
    {
        EnsureValidTransition(OrderStatus.Shipped);
        Status = OrderStatus.Shipped;
        UpdatedAt = DateTime.UtcNow;
        _domainEvents.Add(new OrderShippedEvent(Id, trackingNumber, carrier));
    }

    public void MarkDelivered()
    {
        EnsureValidTransition(OrderStatus.Delivered);
        Status = OrderStatus.Delivered;
        UpdatedAt = DateTime.UtcNow;
        _domainEvents.Add(new OrderDeliveredEvent(Id));
    }

    public void Complete()
    {
        EnsureValidTransition(OrderStatus.Completed);
        Status = OrderStatus.Completed;
        UpdatedAt = DateTime.UtcNow;
        _domainEvents.Add(new OrderCompletedEvent(Id));
    }

    public void Cancel(string reason)
    {
        if (Status is OrderStatus.Shipped or OrderStatus.Delivered or OrderStatus.Completed)
            throw new InvalidOrderTransitionException(Status, OrderStatus.Cancelled);

        Status = OrderStatus.Cancelled;
        UpdatedAt = DateTime.UtcNow;
        _domainEvents.Add(new OrderCancelledEvent(Id, reason));
    }

    private void EnsureValidTransition(OrderStatus target)
    {
        var allowed = GetAllowedTransitions(Status);
        if (!allowed.Contains(target))
            throw new InvalidOrderTransitionException(Status, target);
    }

    private static OrderStatus[] GetAllowedTransitions(OrderStatus current) => current switch
    {
        OrderStatus.Created    => new[] { OrderStatus.Confirmed, OrderStatus.Cancelled },
        OrderStatus.Confirmed  => new[] { OrderStatus.Processing, OrderStatus.Cancelled },
        OrderStatus.Processing => new[] { OrderStatus.Shipped, OrderStatus.Cancelled },
        OrderStatus.Shipped    => new[] { OrderStatus.Delivered },
        OrderStatus.Delivered  => new[] { OrderStatus.Completed },
        _ => Array.Empty<OrderStatus>()
    };

    private void RecalculateTotals()
    {
        SubTotal = _items.Sum(i => i.UnitPrice * i.Quantity);
        TotalAmount = SubTotal + ShippingCost + TaxAmount;
    }

    private static string GenerateOrderNumber()
        => $"ORD-{DateTime.UtcNow:yyyyMMdd}-{Guid.NewGuid().ToString()[..8].ToUpper()}";
}

public class OrderItem
{
    public Guid Id { get; private set; }
    public Guid OrderId { get; private set; }
    public Guid VariantId { get; private set; }
    public string Sku { get; private set; } = string.Empty;
    public string ProductName { get; private set; } = string.Empty;
    public int Quantity { get; private set; }
    public decimal UnitPrice { get; private set; }
    public decimal LineTotal => UnitPrice * Quantity;
    public FulfillmentStatus FulfillmentStatus { get; private set; }

    public void MarkFulfilled() => FulfillmentStatus = FulfillmentStatus.Fulfilled;
    public void MarkPartiallyFulfilled() => FulfillmentStatus = FulfillmentStatus.PartiallyFulfilled;
    public void MarkBackordered() => FulfillmentStatus = FulfillmentStatus.Backordered;
}

public enum OrderStatus
{
    Created,
    Confirmed,
    Processing,
    Shipped,
    Delivered,
    Completed,
    Cancelled
}

public enum FulfillmentStatus
{
    Pending,
    Allocated,
    Fulfilled,
    PartiallyFulfilled,
    Backordered
}
```

### State Transition Validation and Domain Events

```csharp
namespace Inventory.Domain.Exceptions;

public class InvalidOrderTransitionException : Exception
{
    public OrderStatus CurrentStatus { get; }
    public OrderStatus TargetStatus { get; }

    public InvalidOrderTransitionException(OrderStatus current, OrderStatus target)
        : base($"Cannot transition order from {current} to {target}.")
    {
        CurrentStatus = current;
        TargetStatus = target;
    }
}
```

```csharp
namespace Inventory.Domain.Events;

public abstract record DomainEvent
{
    public Guid EventId { get; init; } = Guid.NewGuid();
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}

public record OrderCreatedEvent(Guid OrderId, string OrderNumber) : DomainEvent;
public record OrderConfirmedEvent(Guid OrderId) : DomainEvent;
public record OrderProcessingStartedEvent(Guid OrderId) : DomainEvent;
public record OrderShippedEvent(Guid OrderId, string TrackingNumber, string Carrier) : DomainEvent;
public record OrderDeliveredEvent(Guid OrderId) : DomainEvent;
public record OrderCompletedEvent(Guid OrderId) : DomainEvent;
public record OrderCancelledEvent(Guid OrderId, string Reason) : DomainEvent;
public record StockReceivedEvent(
    Guid VariantId, Guid WarehouseId, int Quantity, int NewAvailable) : DomainEvent;
```

### Order State Transition Diagram

```
┌─────────┐     ┌───────────┐     ┌────────────┐     ┌─────────┐
│ Created ├────►│ Confirmed ├────►│ Processing ├────►│ Shipped │
└────┬────┘     └─────┬─────┘     └──────┬─────┘     └────┬────┘
     │                │                   │                 │
     │                │                   │                 ▼
     │                │                   │           ┌───────────┐
     │                │                   │           │ Delivered │
     │                │                   │           └─────┬─────┘
     │                │                   │                 │
     │                │                   │                 ▼
     ▼                ▼                   ▼           ┌───────────┐
┌───────────┐   ┌───────────┐      ┌───────────┐     │ Completed │
│ Cancelled │   │ Cancelled │      │ Cancelled │     └───────────┘
└───────────┘   └───────────┘      └───────────┘
```

---

## 5. Order Fulfillment Workflow

Fulfillment is the process of picking, packing, and shipping items from warehouse to customer. Complex orders may involve split shipments across multiple warehouses or partial fulfillment when some items are out of stock.

### Fulfillment Entity Model

```csharp
namespace Inventory.Domain.Entities;

public class Fulfillment
{
    public Guid Id { get; private set; }
    public Guid OrderId { get; private set; }
    public Guid WarehouseId { get; private set; }
    public FulfillmentState State { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? PickedAt { get; private set; }
    public DateTime? PackedAt { get; private set; }
    public DateTime? ShippedAt { get; private set; }
    public string? TrackingNumber { get; private set; }
    public string? CarrierCode { get; private set; }

    private readonly List<FulfillmentLine> _lines = new();
    public IReadOnlyCollection<FulfillmentLine> Lines => _lines.AsReadOnly();

    public static Fulfillment Create(Guid orderId, Guid warehouseId, IEnumerable<FulfillmentLine> lines)
    {
        var fulfillment = new Fulfillment
        {
            Id = Guid.NewGuid(),
            OrderId = orderId,
            WarehouseId = warehouseId,
            State = FulfillmentState.Pending,
            CreatedAt = DateTime.UtcNow
        };

        foreach (var line in lines)
            fulfillment._lines.Add(line);

        return fulfillment;
    }

    public void StartPicking()
    {
        if (State != FulfillmentState.Pending)
            throw new InvalidOperationException("Can only start picking from Pending state.");

        State = FulfillmentState.Picking;
    }

    public void CompletePicking()
    {
        State = FulfillmentState.Picked;
        PickedAt = DateTime.UtcNow;
    }

    public void CompletePacking(string packageWeight, string dimensions)
    {
        State = FulfillmentState.Packed;
        PackedAt = DateTime.UtcNow;
    }

    public void Ship(string trackingNumber, string carrierCode)
    {
        State = FulfillmentState.Shipped;
        ShippedAt = DateTime.UtcNow;
        TrackingNumber = trackingNumber;
        CarrierCode = carrierCode;
    }
}

public class FulfillmentLine
{
    public Guid Id { get; private set; }
    public Guid FulfillmentId { get; private set; }
    public Guid OrderItemId { get; private set; }
    public Guid VariantId { get; private set; }
    public int QuantityRequested { get; private set; }
    public int QuantityPicked { get; private set; }
    public Guid? LocationId { get; private set; }

    public void RecordPick(int quantity, Guid locationId)
    {
        QuantityPicked = quantity;
        LocationId = locationId;
    }
}

public enum FulfillmentState
{
    Pending,
    Picking,
    Picked,
    Packed,
    Shipped,
    Delivered,
    Cancelled
}
```

### Fulfillment Orchestrator

The fulfillment orchestrator decides how to split an order across warehouses and creates fulfillment records:

```csharp
namespace Inventory.Application.Services;

public interface IFulfillmentOrchestrator
{
    Task<IReadOnlyList<Fulfillment>> CreateFulfillmentsAsync(
        Guid orderId, CancellationToken ct = default);
}

public class FulfillmentOrchestrator : IFulfillmentOrchestrator
{
    private readonly InventoryDbContext _db;
    private readonly IWarehouseSelector _warehouseSelector;
    private readonly IEventPublisher _eventPublisher;

    public FulfillmentOrchestrator(
        InventoryDbContext db,
        IWarehouseSelector warehouseSelector,
        IEventPublisher eventPublisher)
    {
        _db = db;
        _warehouseSelector = warehouseSelector;
        _eventPublisher = eventPublisher;
    }

    public async Task<IReadOnlyList<Fulfillment>> CreateFulfillmentsAsync(
        Guid orderId, CancellationToken ct = default)
    {
        var order = await _db.Set<Order>()
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == orderId, ct)
            ?? throw new InvalidOperationException($"Order {orderId} not found.");

        // Group items by optimal warehouse
        var warehouseAllocations = new Dictionary<Guid, List<FulfillmentLine>>();
        var backorderItems = new List<OrderItem>();

        foreach (var item in order.Items)
        {
            var allocation = await TryAllocateAsync(item, ct);
            if (allocation is not null)
            {
                if (!warehouseAllocations.ContainsKey(allocation.Value.WarehouseId))
                    warehouseAllocations[allocation.Value.WarehouseId] = new();

                warehouseAllocations[allocation.Value.WarehouseId].Add(new FulfillmentLine
                {
                    // initialize properties
                });
            }
            else
            {
                backorderItems.Add(item);
                item.MarkBackordered();
            }
        }

        // Create one fulfillment per warehouse (split shipment)
        var fulfillments = new List<Fulfillment>();
        foreach (var (warehouseId, lines) in warehouseAllocations)
        {
            var fulfillment = Fulfillment.Create(orderId, warehouseId, lines);
            _db.Set<Fulfillment>().Add(fulfillment);
            fulfillments.Add(fulfillment);
        }

        // Handle backorders
        foreach (var item in backorderItems)
        {
            _db.Set<BackorderEntry>().Add(BackorderEntry.Create(
                orderId, item.VariantId, item.Quantity));
        }

        await _db.SaveChangesAsync(ct);

        if (fulfillments.Count > 1)
        {
            await _eventPublisher.PublishAsync(
                new OrderSplitIntoShipmentsEvent(orderId, fulfillments.Count), ct);
        }

        return fulfillments;
    }

    private async Task<(Guid WarehouseId, int Quantity)?> TryAllocateAsync(
        OrderItem item, CancellationToken ct)
    {
        var records = await _db.InventoryRecords
            .Where(r => r.VariantId == item.VariantId && r.QuantityOnHand - r.QuantityReserved > 0)
            .OrderBy(r => r.Warehouse.Priority)
            .ToListAsync(ct);

        var best = records.FirstOrDefault(r => r.QuantityAvailable >= item.Quantity);
        if (best is null) return null;

        return (best.WarehouseId, item.Quantity);
    }
}

public record OrderSplitIntoShipmentsEvent(Guid OrderId, int ShipmentCount) : DomainEvent;
```

### Pick List Generation

```csharp
namespace Inventory.Application.Services;

public record PickListItem(
    string Sku,
    string ProductName,
    int Quantity,
    string LocationCode,
    string Aisle,
    string Shelf,
    string Bin);

public class PickListGenerator
{
    private readonly InventoryDbContext _db;

    public PickListGenerator(InventoryDbContext db) => _db = db;

    public async Task<IReadOnlyList<PickListItem>> GenerateAsync(
        Guid fulfillmentId, CancellationToken ct = default)
    {
        var fulfillment = await _db.Set<Fulfillment>()
            .Include(f => f.Lines)
            .FirstOrDefaultAsync(f => f.Id == fulfillmentId, ct)
            ?? throw new InvalidOperationException("Fulfillment not found.");

        var pickList = new List<PickListItem>();

        foreach (var line in fulfillment.Lines)
        {
            var variant = await _db.Variants
                .Include(v => v.Product)
                .FirstAsync(v => v.Id == line.VariantId, ct);

            var location = await _db.WarehouseLocations
                .Where(l => l.WarehouseId == fulfillment.WarehouseId && l.Type == LocationType.Pickable)
                .FirstOrDefaultAsync(ct);

            pickList.Add(new PickListItem(
                variant.Sku,
                variant.Product.Name,
                line.QuantityRequested,
                location?.FullCode ?? "UNASSIGNED",
                location?.Aisle ?? "",
                location?.Shelf ?? "",
                location?.Bin ?? ""));
        }

        // Sort by location for efficient warehouse traversal
        return pickList
            .OrderBy(p => p.Aisle)
            .ThenBy(p => p.Shelf)
            .ThenBy(p => p.Bin)
            .ToList();
    }
}
```

---

## 6. Shipping Integration

A clean abstraction over carrier APIs allows the system to support multiple carriers without coupling business logic to any single provider.

### Carrier Abstraction

```csharp
namespace Inventory.Domain.Interfaces;

public interface IShippingCarrier
{
    string CarrierCode { get; }
    Task<ShippingRate[]> GetRatesAsync(ShipmentRequest request, CancellationToken ct = default);
    Task<ShippingLabel> CreateLabelAsync(ShipmentRequest request, CancellationToken ct = default);
    Task<TrackingInfo> GetTrackingAsync(string trackingNumber, CancellationToken ct = default);
    Task CancelShipmentAsync(string trackingNumber, CancellationToken ct = default);
}

public record ShipmentRequest(
    Address FromAddress,
    Address ToAddress,
    IReadOnlyList<PackageInfo> Packages,
    string ServiceLevel = "ground");

public record PackageInfo(
    decimal WeightKg,
    decimal LengthCm,
    decimal WidthCm,
    decimal HeightCm);

public record ShippingRate(
    string CarrierCode,
    string ServiceLevel,
    string ServiceName,
    decimal TotalCost,
    string Currency,
    int EstimatedDays,
    DateTime? EstimatedDeliveryDate);

public record ShippingLabel(
    string TrackingNumber,
    string CarrierCode,
    string ServiceLevel,
    byte[] LabelData,
    string LabelFormat,
    decimal TotalCost);

public record TrackingInfo(
    string TrackingNumber,
    string CarrierCode,
    TrackingStatus Status,
    DateTime? EstimatedDelivery,
    IReadOnlyList<TrackingEvent> Events);

public record TrackingEvent(
    DateTime Timestamp,
    string Location,
    string Description,
    TrackingStatus Status);

public enum TrackingStatus
{
    LabelCreated,
    PickedUp,
    InTransit,
    OutForDelivery,
    Delivered,
    Exception,
    Returned
}
```

### Carrier Implementations

```csharp
using Microsoft.Extensions.Options;

namespace Inventory.Infrastructure.Shipping;

public class FedExCarrier : IShippingCarrier
{
    private readonly HttpClient _httpClient;
    private readonly FedExOptions _options;

    public string CarrierCode => "FEDEX";

    public FedExCarrier(HttpClient httpClient, IOptions<FedExOptions> options)
    {
        _httpClient = httpClient;
        _options = options.Value;
    }

    public async Task<ShippingRate[]> GetRatesAsync(
        ShipmentRequest request, CancellationToken ct = default)
    {
        var rateRequest = new
        {
            accountNumber = new { value = _options.AccountNumber },
            requestedShipment = new
            {
                shipper = MapAddress(request.FromAddress),
                recipient = MapAddress(request.ToAddress),
                requestedPackageLineItems = request.Packages.Select(p => new
                {
                    weight = new { value = p.WeightKg, units = "KG" },
                    dimensions = new
                    {
                        length = p.LengthCm,
                        width = p.WidthCm,
                        height = p.HeightCm,
                        units = "CM"
                    }
                })
            }
        };

        var response = await _httpClient.PostAsJsonAsync(
            "/rate/v1/rates/quotes", rateRequest, ct);
        response.EnsureSuccessStatusCode();

        var result = await response.Content
            .ReadFromJsonAsync<FedExRateResponse>(cancellationToken: ct);

        return result?.Output.RateReplyDetails
            .Select(r => new ShippingRate(
                CarrierCode,
                r.ServiceType,
                r.ServiceName,
                r.RatedShipmentDetails.First().TotalNetCharge,
                "USD",
                r.TransitTime,
                r.DeliveryTimestamp))
            .ToArray() ?? Array.Empty<ShippingRate>();
    }

    public async Task<ShippingLabel> CreateLabelAsync(
        ShipmentRequest request, CancellationToken ct = default)
    {
        // FedEx Ship API call
        var response = await _httpClient.PostAsJsonAsync(
            "/ship/v1/shipments", MapToShipRequest(request), ct);
        response.EnsureSuccessStatusCode();

        var result = await response.Content
            .ReadFromJsonAsync<FedExShipResponse>(cancellationToken: ct);

        var piece = result!.Output.TransactionShipments.First()
            .PieceResponses.First();

        return new ShippingLabel(
            piece.TrackingNumber,
            CarrierCode,
            request.ServiceLevel,
            Convert.FromBase64String(piece.PackageDocuments.First().EncodedLabel),
            "PDF",
            piece.NetCharge);
    }

    public async Task<TrackingInfo> GetTrackingAsync(
        string trackingNumber, CancellationToken ct = default)
    {
        var response = await _httpClient.PostAsJsonAsync(
            "/track/v1/trackingnumbers",
            new { trackingInfo = new[] { new { trackingNumberInfo = new { trackingNumber } } } },
            ct);
        response.EnsureSuccessStatusCode();

        var result = await response.Content
            .ReadFromJsonAsync<FedExTrackResponse>(cancellationToken: ct);

        var track = result!.Output.CompleteTrackResults.First().TrackResults.First();

        return new TrackingInfo(
            trackingNumber,
            CarrierCode,
            MapStatus(track.LatestStatusDetail.Code),
            track.EstimatedDeliveryTimeWindow?.Window?.Ends,
            track.ScanEvents.Select(e => new TrackingEvent(
                e.Date, e.ScanLocation.City, e.EventDescription, MapStatus(e.EventType)
            )).ToList());
    }

    public async Task CancelShipmentAsync(string trackingNumber, CancellationToken ct = default)
    {
        await _httpClient.PutAsJsonAsync(
            "/ship/v1/shipments/cancel",
            new { trackingNumber, accountNumber = new { value = _options.AccountNumber } },
            ct);
    }

    private static object MapAddress(Address addr) => new
    {
        address = new
        {
            streetLines = new[] { addr.Street },
            city = addr.City,
            stateOrProvinceCode = addr.State,
            postalCode = addr.PostalCode,
            countryCode = addr.Country
        }
    };

    private static object MapToShipRequest(ShipmentRequest request) => new { /* mapping */ };
    private static TrackingStatus MapStatus(string code) => code switch
    {
        "DL" => TrackingStatus.Delivered,
        "IT" => TrackingStatus.InTransit,
        "OD" => TrackingStatus.OutForDelivery,
        "PU" => TrackingStatus.PickedUp,
        _ => TrackingStatus.InTransit
    };
}

public class FedExOptions
{
    public string AccountNumber { get; set; } = string.Empty;
    public string ApiKey { get; set; } = string.Empty;
    public string SecretKey { get; set; } = string.Empty;
}
```

### Carrier Factory and Rate Comparison

```csharp
namespace Inventory.Application.Services;

public interface IShippingService
{
    Task<ShippingRate[]> GetAllRatesAsync(ShipmentRequest request, CancellationToken ct = default);
    Task<ShippingRate> GetCheapestRateAsync(ShipmentRequest request, CancellationToken ct = default);
    Task<ShippingLabel> ShipAsync(
        ShipmentRequest request, string carrierCode, CancellationToken ct = default);
    Task<TrackingInfo> TrackAsync(
        string trackingNumber, string carrierCode, CancellationToken ct = default);
}

public class ShippingService : IShippingService
{
    private readonly IEnumerable<IShippingCarrier> _carriers;

    public ShippingService(IEnumerable<IShippingCarrier> carriers)
    {
        _carriers = carriers;
    }

    public async Task<ShippingRate[]> GetAllRatesAsync(
        ShipmentRequest request, CancellationToken ct = default)
    {
        var tasks = _carriers.Select(c => c.GetRatesAsync(request, ct));
        var results = await Task.WhenAll(tasks);
        return results.SelectMany(r => r).OrderBy(r => r.TotalCost).ToArray();
    }

    public async Task<ShippingRate> GetCheapestRateAsync(
        ShipmentRequest request, CancellationToken ct = default)
    {
        var rates = await GetAllRatesAsync(request, ct);
        return rates.FirstOrDefault()
            ?? throw new InvalidOperationException("No shipping rates available.");
    }

    public async Task<ShippingLabel> ShipAsync(
        ShipmentRequest request, string carrierCode, CancellationToken ct = default)
    {
        var carrier = GetCarrier(carrierCode);
        return await carrier.CreateLabelAsync(request, ct);
    }

    public async Task<TrackingInfo> TrackAsync(
        string trackingNumber, string carrierCode, CancellationToken ct = default)
    {
        var carrier = GetCarrier(carrierCode);
        return await carrier.GetTrackingAsync(trackingNumber, ct);
    }

    private IShippingCarrier GetCarrier(string code)
    {
        return _carriers.FirstOrDefault(c =>
            c.CarrierCode.Equals(code, StringComparison.OrdinalIgnoreCase))
            ?? throw new InvalidOperationException($"Carrier {code} not registered.");
    }
}
```

### Shipping Webhook Handler

Carrier tracking webhooks update order status automatically:

```csharp
namespace Inventory.Api.Controllers;

[ApiController]
[Route("api/webhooks/shipping")]
public class ShippingWebhookController : ControllerBase
{
    private readonly InventoryDbContext _db;
    private readonly IEventPublisher _eventPublisher;
    private readonly ILogger<ShippingWebhookController> _logger;

    public ShippingWebhookController(
        InventoryDbContext db,
        IEventPublisher eventPublisher,
        ILogger<ShippingWebhookController> logger)
    {
        _db = db;
        _eventPublisher = eventPublisher;
        _logger = logger;
    }

    [HttpPost("fedex")]
    public async Task<IActionResult> HandleFedExWebhook(
        [FromBody] FedExWebhookPayload payload)
    {
        var fulfillment = await _db.Set<Fulfillment>()
            .FirstOrDefaultAsync(f => f.TrackingNumber == payload.TrackingNumber);

        if (fulfillment is null)
        {
            _logger.LogWarning(
                "Received tracking update for unknown tracking number: {TrackingNumber}",
                payload.TrackingNumber);
            return Ok(); // acknowledge but ignore
        }

        switch (payload.EventType)
        {
            case "DELIVERED":
                fulfillment.Ship(payload.TrackingNumber, "FEDEX");
                var order = await _db.Set<Order>()
                    .FirstAsync(o => o.Id == fulfillment.OrderId);
                order.MarkDelivered();
                await _eventPublisher.PublishAsync(
                    new OrderDeliveredEvent(order.Id));
                break;

            case "EXCEPTION":
                _logger.LogWarning(
                    "Shipping exception for {TrackingNumber}: {Description}",
                    payload.TrackingNumber, payload.Description);
                await _eventPublisher.PublishAsync(
                    new ShippingExceptionEvent(
                        fulfillment.OrderId, payload.TrackingNumber, payload.Description));
                break;
        }

        await _db.SaveChangesAsync();
        return Ok();
    }
}

public record FedExWebhookPayload(
    string TrackingNumber,
    string EventType,
    string Description,
    DateTime Timestamp);

public record ShippingExceptionEvent(
    Guid OrderId, string TrackingNumber, string Description) : DomainEvent;
```

---

## 7. Warehouse Management Patterns

Multi-warehouse operations require intelligent inventory distribution, fulfillment routing, and rebalancing to optimize delivery speed and cost.

### Warehouse Selection Strategy

```csharp
namespace Inventory.Application.Services;

public interface IWarehouseSelector
{
    Task<Guid> SelectAsync(Guid variantId, int quantity, CancellationToken ct = default);
    Task<Guid> SelectNearestAsync(
        Guid variantId, int quantity, Address destination, CancellationToken ct = default);
}

public class WarehouseSelector : IWarehouseSelector
{
    private readonly InventoryDbContext _db;

    public WarehouseSelector(InventoryDbContext db) => _db = db;

    public async Task<Guid> SelectAsync(
        Guid variantId, int quantity, CancellationToken ct = default)
    {
        var record = await _db.InventoryRecords
            .Include(r => r.Warehouse)
            .Where(r => r.VariantId == variantId &&
                        r.Warehouse.IsActive &&
                        (r.QuantityOnHand - r.QuantityReserved) >= quantity)
            .OrderBy(r => r.Warehouse.Priority)
            .FirstOrDefaultAsync(ct)
            ?? throw new InsufficientStockException(variantId, 0, quantity);

        return record.WarehouseId;
    }

    public async Task<Guid> SelectNearestAsync(
        Guid variantId, int quantity, Address destination, CancellationToken ct = default)
    {
        var candidates = await _db.InventoryRecords
            .Include(r => r.Warehouse)
            .Where(r => r.VariantId == variantId &&
                        r.Warehouse.IsActive &&
                        (r.QuantityOnHand - r.QuantityReserved) >= quantity)
            .ToListAsync(ct);

        if (candidates.Count == 0)
            throw new InsufficientStockException(variantId, 0, quantity);

        // Select warehouse closest to the shipping destination
        return candidates
            .OrderBy(r => HaversineDistance(
                r.Warehouse.Address.Latitude, r.Warehouse.Address.Longitude,
                destination.Latitude, destination.Longitude))
            .First()
            .WarehouseId;
    }

    private static double HaversineDistance(
        double lat1, double lon1, double lat2, double lon2)
    {
        const double R = 6371; // Earth radius in km
        var dLat = ToRadians(lat2 - lat1);
        var dLon = ToRadians(lon2 - lon1);
        var a = Math.Sin(dLat / 2) * Math.Sin(dLat / 2) +
                Math.Cos(ToRadians(lat1)) * Math.Cos(ToRadians(lat2)) *
                Math.Sin(dLon / 2) * Math.Sin(dLon / 2);
        return R * 2 * Math.Atan2(Math.Sqrt(a), Math.Sqrt(1 - a));
    }

    private static double ToRadians(double degrees) => degrees * Math.PI / 180;
}
```

### Safety Stock and Reorder Points

Automatic reorder triggers prevent stockouts by monitoring inventory levels:

```csharp
namespace Inventory.Domain.Entities;

public class ReorderPolicy
{
    public Guid Id { get; private set; }
    public Guid VariantId { get; private set; }
    public Guid WarehouseId { get; private set; }
    public int ReorderPoint { get; private set; }
    public int SafetyStock { get; private set; }
    public int ReorderQuantity { get; private set; }
    public int MaxStock { get; private set; }
    public int LeadTimeDays { get; private set; }
    public bool IsActive { get; private set; }

    public bool NeedsReorder(int currentAvailable) =>
        IsActive && currentAvailable <= ReorderPoint;

    public static ReorderPolicy Create(
        Guid variantId, Guid warehouseId,
        int reorderPoint, int safetyStock, int reorderQuantity,
        int maxStock, int leadTimeDays)
    {
        return new ReorderPolicy
        {
            Id = Guid.NewGuid(),
            VariantId = variantId,
            WarehouseId = warehouseId,
            ReorderPoint = reorderPoint,
            SafetyStock = safetyStock,
            ReorderQuantity = reorderQuantity,
            MaxStock = maxStock,
            LeadTimeDays = leadTimeDays,
            IsActive = true
        };
    }
}
```

```csharp
namespace Inventory.Application.Services;

public class ReorderMonitorService
{
    private readonly InventoryDbContext _db;
    private readonly IEventPublisher _eventPublisher;
    private readonly ILogger<ReorderMonitorService> _logger;

    public ReorderMonitorService(
        InventoryDbContext db,
        IEventPublisher eventPublisher,
        ILogger<ReorderMonitorService> logger)
    {
        _db = db;
        _eventPublisher = eventPublisher;
        _logger = logger;
    }

    public async Task CheckReorderPointsAsync(CancellationToken ct = default)
    {
        var policies = await _db.Set<ReorderPolicy>()
            .Where(p => p.IsActive)
            .ToListAsync(ct);

        foreach (var policy in policies)
        {
            var record = await _db.InventoryRecords
                .FirstOrDefaultAsync(r =>
                    r.VariantId == policy.VariantId &&
                    r.WarehouseId == policy.WarehouseId, ct);

            if (record is null) continue;

            if (policy.NeedsReorder(record.QuantityAvailable))
            {
                _logger.LogInformation(
                    "Reorder triggered for variant {VariantId} at warehouse {WarehouseId}. " +
                    "Available: {Available}, Reorder Point: {ReorderPoint}",
                    policy.VariantId, policy.WarehouseId,
                    record.QuantityAvailable, policy.ReorderPoint);

                await _eventPublisher.PublishAsync(new ReorderTriggeredEvent(
                    policy.VariantId,
                    policy.WarehouseId,
                    policy.ReorderQuantity,
                    record.QuantityAvailable,
                    policy.LeadTimeDays), ct);
            }
        }
    }
}

public record ReorderTriggeredEvent(
    Guid VariantId,
    Guid WarehouseId,
    int ReorderQuantity,
    int CurrentAvailable,
    int LeadTimeDays) : DomainEvent;
```

### Inventory Transfer Between Warehouses

```csharp
namespace Inventory.Domain.Entities;

public class InventoryTransfer
{
    public Guid Id { get; private set; }
    public Guid VariantId { get; private set; }
    public Guid SourceWarehouseId { get; private set; }
    public Guid DestinationWarehouseId { get; private set; }
    public int Quantity { get; private set; }
    public TransferStatus Status { get; private set; }
    public string Reason { get; private set; } = string.Empty;
    public string InitiatedBy { get; private set; } = string.Empty;
    public DateTime CreatedAt { get; private set; }
    public DateTime? ShippedAt { get; private set; }
    public DateTime? ReceivedAt { get; private set; }

    public static InventoryTransfer Create(
        Guid variantId, Guid sourceWarehouseId, Guid destinationWarehouseId,
        int quantity, string reason, string initiatedBy)
    {
        return new InventoryTransfer
        {
            Id = Guid.NewGuid(),
            VariantId = variantId,
            SourceWarehouseId = sourceWarehouseId,
            DestinationWarehouseId = destinationWarehouseId,
            Quantity = quantity,
            Status = TransferStatus.Requested,
            Reason = reason,
            InitiatedBy = initiatedBy,
            CreatedAt = DateTime.UtcNow
        };
    }

    public void MarkShipped()
    {
        Status = TransferStatus.InTransit;
        ShippedAt = DateTime.UtcNow;
    }

    public void MarkReceived()
    {
        Status = TransferStatus.Received;
        ReceivedAt = DateTime.UtcNow;
    }

    public void Cancel()
    {
        if (Status == TransferStatus.Received)
            throw new InvalidOperationException("Cannot cancel a received transfer.");

        Status = TransferStatus.Cancelled;
    }
}

public enum TransferStatus
{
    Requested,
    Approved,
    InTransit,
    Received,
    Cancelled
}
```

```csharp
namespace Inventory.Application.Services;

public class InventoryTransferService
{
    private readonly InventoryDbContext _db;
    private readonly IEventPublisher _eventPublisher;

    public InventoryTransferService(InventoryDbContext db, IEventPublisher eventPublisher)
    {
        _db = db;
        _eventPublisher = eventPublisher;
    }

    public async Task<InventoryTransfer> InitiateTransferAsync(
        Guid variantId, Guid sourceWarehouseId, Guid destWarehouseId,
        int quantity, string reason, string initiatedBy,
        CancellationToken ct = default)
    {
        var sourceRecord = await _db.InventoryRecords
            .FirstOrDefaultAsync(r =>
                r.VariantId == variantId && r.WarehouseId == sourceWarehouseId, ct)
            ?? throw new InvalidOperationException("Source inventory record not found.");

        if (sourceRecord.QuantityAvailable < quantity)
            throw new InsufficientStockException(variantId, sourceRecord.QuantityAvailable, quantity);

        sourceRecord.Reserve(quantity);

        var transfer = InventoryTransfer.Create(
            variantId, sourceWarehouseId, destWarehouseId, quantity, reason, initiatedBy);

        _db.Set<InventoryTransfer>().Add(transfer);
        await _db.SaveChangesAsync(ct);

        return transfer;
    }

    public async Task ShipTransferAsync(Guid transferId, CancellationToken ct = default)
    {
        var transfer = await _db.Set<InventoryTransfer>()
            .FirstAsync(t => t.Id == transferId, ct);

        var sourceRecord = await _db.InventoryRecords
            .FirstAsync(r =>
                r.VariantId == transfer.VariantId &&
                r.WarehouseId == transfer.SourceWarehouseId, ct);

        sourceRecord.ConfirmOutbound(transfer.Quantity);
        transfer.MarkShipped();

        // Mark as in-transit at destination
        var destRecord = await GetOrCreateRecordAsync(
            transfer.VariantId, transfer.DestinationWarehouseId, ct);
        // QuantityInTransit tracked separately

        _db.Set<StockMovement>().Add(StockMovement.Create(
            transfer.VariantId, transfer.SourceWarehouseId,
            MovementType.Transfer, -transfer.Quantity,
            sourceRecord.QuantityOnHand,
            $"Transfer to warehouse", transfer.InitiatedBy,
            relatedTransferId: transfer.Id));

        await _db.SaveChangesAsync(ct);
    }

    public async Task ReceiveTransferAsync(Guid transferId, CancellationToken ct = default)
    {
        var transfer = await _db.Set<InventoryTransfer>()
            .FirstAsync(t => t.Id == transferId, ct);

        var destRecord = await GetOrCreateRecordAsync(
            transfer.VariantId, transfer.DestinationWarehouseId, ct);

        destRecord.ReceiveInbound(transfer.Quantity);
        transfer.MarkReceived();

        _db.Set<StockMovement>().Add(StockMovement.Create(
            transfer.VariantId, transfer.DestinationWarehouseId,
            MovementType.Transfer, transfer.Quantity,
            destRecord.QuantityOnHand,
            $"Transfer from warehouse", transfer.InitiatedBy,
            relatedTransferId: transfer.Id));

        await _db.SaveChangesAsync(ct);

        await _eventPublisher.PublishAsync(new TransferCompletedEvent(
            transfer.Id, transfer.VariantId,
            transfer.SourceWarehouseId, transfer.DestinationWarehouseId,
            transfer.Quantity), ct);
    }

    private async Task<InventoryRecord> GetOrCreateRecordAsync(
        Guid variantId, Guid warehouseId, CancellationToken ct)
    {
        var record = await _db.InventoryRecords
            .FirstOrDefaultAsync(r =>
                r.VariantId == variantId && r.WarehouseId == warehouseId, ct);

        if (record is null)
        {
            record = new InventoryRecord();
            _db.InventoryRecords.Add(record);
        }

        return record;
    }
}

public record TransferCompletedEvent(
    Guid TransferId, Guid VariantId,
    Guid SourceWarehouseId, Guid DestinationWarehouseId,
    int Quantity) : DomainEvent;
```

---

## 8. Real-Time Inventory Sync

Multi-channel retail requires real-time inventory synchronization across web, mobile, POS, and marketplace channels. An event-driven architecture ensures eventual consistency without tight coupling.

### Inventory Event Publisher

```csharp
namespace Inventory.Application.Interfaces;

public interface IEventPublisher
{
    Task PublishAsync<TEvent>(TEvent @event, CancellationToken ct = default)
        where TEvent : DomainEvent;
}

public interface IInventoryBroadcaster
{
    Task BroadcastAvailabilityAsync(
        Guid variantId, int totalAvailable, CancellationToken ct = default);
    Task BroadcastStockoutAsync(
        Guid variantId, CancellationToken ct = default);
}
```

### Message Broker Integration

```csharp
using System.Text.Json;
using RabbitMQ.Client;

namespace Inventory.Infrastructure.Messaging;

public class RabbitMqEventPublisher : IEventPublisher
{
    private readonly IConnection _connection;
    private readonly ILogger<RabbitMqEventPublisher> _logger;
    private const string ExchangeName = "inventory.events";

    public RabbitMqEventPublisher(IConnection connection, ILogger<RabbitMqEventPublisher> logger)
    {
        _connection = connection;
        _logger = logger;
    }

    public async Task PublishAsync<TEvent>(TEvent @event, CancellationToken ct = default)
        where TEvent : DomainEvent
    {
        using var channel = await _connection.CreateChannelAsync(ct);

        await channel.ExchangeDeclareAsync(
            exchange: ExchangeName,
            type: "topic",
            durable: true,
            cancellationToken: ct);

        var routingKey = GetRoutingKey(@event);
        var body = JsonSerializer.SerializeToUtf8Bytes(@event, new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        });

        var properties = new BasicProperties
        {
            MessageId = @event.EventId.ToString(),
            Timestamp = new AmqpTimestamp(DateTimeOffset.UtcNow.ToUnixTimeSeconds()),
            ContentType = "application/json",
            DeliveryMode = DeliveryModes.Persistent
        };

        await channel.BasicPublishAsync(
            exchange: ExchangeName,
            routingKey: routingKey,
            mandatory: false,
            basicProperties: properties,
            body: body,
            cancellationToken: ct);

        _logger.LogInformation(
            "Published event {EventType} with ID {EventId} to {RoutingKey}",
            typeof(TEvent).Name, @event.EventId, routingKey);
    }

    private static string GetRoutingKey<TEvent>(TEvent @event) => @event switch
    {
        StockReceivedEvent => "inventory.stock.received",
        ReorderTriggeredEvent => "inventory.stock.reorder",
        OrderCreatedEvent => "order.lifecycle.created",
        OrderShippedEvent => "order.lifecycle.shipped",
        OrderDeliveredEvent => "order.lifecycle.delivered",
        TransferCompletedEvent => "inventory.transfer.completed",
        _ => $"inventory.{typeof(TEvent).Name.ToLowerInvariant()}"
    };
}
```

### Channel-Specific Inventory Projections

Different sales channels may need different views of inventory. A projection service maintains channel-optimized read models:

```csharp
namespace Inventory.Application.Services;

public class ChannelInventoryProjection
{
    private readonly InventoryDbContext _db;
    private readonly IInventoryBroadcaster _broadcaster;

    public ChannelInventoryProjection(InventoryDbContext db, IInventoryBroadcaster broadcaster)
    {
        _db = db;
        _broadcaster = broadcaster;
    }

    public async Task<ChannelAvailability> GetAvailabilityAsync(
        Guid variantId, string channel, CancellationToken ct = default)
    {
        var records = await _db.InventoryRecords
            .Where(r => r.VariantId == variantId)
            .ToListAsync(ct);

        int totalAvailable = records.Sum(r => r.QuantityAvailable);

        // Apply channel-specific buffer
        int channelBuffer = channel switch
        {
            "web" => 2,         // hold back 2 units as safety buffer
            "marketplace" => 5, // higher buffer for marketplace lag
            "pos" => 0,         // POS sees real-time stock
            _ => 0
        };

        int channelAvailable = Math.Max(0, totalAvailable - channelBuffer);

        return new ChannelAvailability(
            variantId,
            channel,
            channelAvailable,
            channelAvailable > 0,
            totalAvailable <= 10 ? "Low Stock" : "In Stock");
    }

    public async Task RebuildProjectionsAsync(
        Guid variantId, CancellationToken ct = default)
    {
        var records = await _db.InventoryRecords
            .Where(r => r.VariantId == variantId)
            .ToListAsync(ct);

        int totalAvailable = records.Sum(r => r.QuantityAvailable);
        await _broadcaster.BroadcastAvailabilityAsync(variantId, totalAvailable, ct);

        if (totalAvailable <= 0)
            await _broadcaster.BroadcastStockoutAsync(variantId, ct);
    }
}

public record ChannelAvailability(
    Guid VariantId,
    string Channel,
    int AvailableQuantity,
    bool InStock,
    string StockStatus);
```

### SignalR Real-Time Broadcast

For web and mobile clients, SignalR pushes inventory changes in real time:

```csharp
using Microsoft.AspNetCore.SignalR;

namespace Inventory.Infrastructure.Hubs;

public class InventoryHub : Hub
{
    public async Task SubscribeToSku(string sku)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"sku:{sku}");
    }

    public async Task UnsubscribeFromSku(string sku)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"sku:{sku}");
    }
}

public class SignalRInventoryBroadcaster : IInventoryBroadcaster
{
    private readonly IHubContext<InventoryHub> _hubContext;
    private readonly InventoryDbContext _db;

    public SignalRInventoryBroadcaster(
        IHubContext<InventoryHub> hubContext, InventoryDbContext db)
    {
        _hubContext = hubContext;
        _db = db;
    }

    public async Task BroadcastAvailabilityAsync(
        Guid variantId, int totalAvailable, CancellationToken ct = default)
    {
        var variant = await _db.Variants.FirstOrDefaultAsync(v => v.Id == variantId, ct);
        if (variant is null) return;

        await _hubContext.Clients.Group($"sku:{variant.Sku}")
            .SendAsync("StockUpdated", new
            {
                sku = variant.Sku,
                available = totalAvailable,
                inStock = totalAvailable > 0,
                updatedAt = DateTime.UtcNow
            }, ct);
    }

    public async Task BroadcastStockoutAsync(
        Guid variantId, CancellationToken ct = default)
    {
        var variant = await _db.Variants.FirstOrDefaultAsync(v => v.Id == variantId, ct);
        if (variant is null) return;

        await _hubContext.Clients.Group($"sku:{variant.Sku}")
            .SendAsync("StockOut", new
            {
                sku = variant.Sku,
                available = 0,
                inStock = false,
                updatedAt = DateTime.UtcNow
            }, ct);
    }
}
```

---

## 9. Backorder and Pre-Order Management

When inventory is unavailable, backorder and pre-order systems allow customers to place orders for future fulfillment. These require careful handling of payment capture timing and customer communication.

### Backorder Entity and Queue

```csharp
namespace Inventory.Domain.Entities;

public class BackorderEntry
{
    public Guid Id { get; private set; }
    public Guid OrderId { get; private set; }
    public Guid VariantId { get; private set; }
    public int Quantity { get; private set; }
    public BackorderStatus Status { get; private set; }
    public int QueuePosition { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? FulfilledAt { get; private set; }
    public DateTime? EstimatedAvailableDate { get; private set; }

    public static BackorderEntry Create(Guid orderId, Guid variantId, int quantity)
    {
        return new BackorderEntry
        {
            Id = Guid.NewGuid(),
            OrderId = orderId,
            VariantId = variantId,
            Quantity = quantity,
            Status = BackorderStatus.Queued,
            CreatedAt = DateTime.UtcNow
        };
    }

    public void AssignQueuePosition(int position) => QueuePosition = position;

    public void SetEstimatedDate(DateTime date) => EstimatedAvailableDate = date;

    public void MarkFulfilled()
    {
        Status = BackorderStatus.Fulfilled;
        FulfilledAt = DateTime.UtcNow;
    }

    public void Cancel()
    {
        Status = BackorderStatus.Cancelled;
    }
}

public enum BackorderStatus
{
    Queued,
    AwaitingStock,
    ReadyToFulfill,
    Fulfilled,
    Cancelled
}
```

### Pre-Order Management

Pre-orders have specific rules around payment capture — authorization at order time, capture at ship time:

```csharp
namespace Inventory.Domain.Entities;

public class PreOrder
{
    public Guid Id { get; private set; }
    public Guid OrderId { get; private set; }
    public Guid VariantId { get; private set; }
    public int Quantity { get; private set; }
    public PreOrderStatus Status { get; private set; }
    public DateTime ReleaseDate { get; private set; }
    public DateTime? PaymentAuthorizedAt { get; private set; }
    public DateTime? PaymentCapturedAt { get; private set; }
    public string? PaymentIntentId { get; private set; }
    public DateTime CreatedAt { get; private set; }

    public bool IsReleaseDateReached => DateTime.UtcNow >= ReleaseDate;

    public static PreOrder Create(
        Guid orderId, Guid variantId, int quantity,
        DateTime releaseDate, string paymentIntentId)
    {
        return new PreOrder
        {
            Id = Guid.NewGuid(),
            OrderId = orderId,
            VariantId = variantId,
            Quantity = quantity,
            Status = PreOrderStatus.Pending,
            ReleaseDate = releaseDate,
            PaymentIntentId = paymentIntentId,
            CreatedAt = DateTime.UtcNow
        };
    }

    public void AuthorizePayment()
    {
        Status = PreOrderStatus.Authorized;
        PaymentAuthorizedAt = DateTime.UtcNow;
    }

    public void CapturePayment()
    {
        if (Status != PreOrderStatus.Authorized)
            throw new InvalidOperationException("Payment must be authorized before capture.");

        Status = PreOrderStatus.Captured;
        PaymentCapturedAt = DateTime.UtcNow;
    }

    public void MarkReadyToShip() => Status = PreOrderStatus.ReadyToShip;
    public void Cancel() => Status = PreOrderStatus.Cancelled;
}

public enum PreOrderStatus
{
    Pending,
    Authorized,
    Captured,
    ReadyToShip,
    Fulfilled,
    Cancelled
}
```

### Backorder Fulfillment Processor

When stock arrives, backorders are fulfilled in FIFO order:

```csharp
namespace Inventory.Application.Services;

public class BackorderProcessor
{
    private readonly InventoryDbContext _db;
    private readonly IFulfillmentOrchestrator _fulfillmentOrchestrator;
    private readonly IEventPublisher _eventPublisher;
    private readonly ILogger<BackorderProcessor> _logger;

    public BackorderProcessor(
        InventoryDbContext db,
        IFulfillmentOrchestrator fulfillmentOrchestrator,
        IEventPublisher eventPublisher,
        ILogger<BackorderProcessor> logger)
    {
        _db = db;
        _fulfillmentOrchestrator = fulfillmentOrchestrator;
        _eventPublisher = eventPublisher;
        _logger = logger;
    }

    public async Task ProcessBackordersForVariantAsync(
        Guid variantId, CancellationToken ct = default)
    {
        var available = await _db.InventoryRecords
            .Where(r => r.VariantId == variantId)
            .SumAsync(r => r.QuantityOnHand - r.QuantityReserved - r.QuantityDamaged, ct);

        if (available <= 0) return;

        var pendingBackorders = await _db.Set<BackorderEntry>()
            .Where(b => b.VariantId == variantId && b.Status == BackorderStatus.Queued)
            .OrderBy(b => b.QueuePosition)
            .ThenBy(b => b.CreatedAt)
            .ToListAsync(ct);

        int remaining = available;

        foreach (var backorder in pendingBackorders)
        {
            if (remaining < backorder.Quantity) break;

            backorder.MarkFulfilled();
            remaining -= backorder.Quantity;

            _logger.LogInformation(
                "Fulfilling backorder {BackorderId} for order {OrderId}, quantity {Quantity}",
                backorder.Id, backorder.OrderId, backorder.Quantity);

            await _eventPublisher.PublishAsync(
                new BackorderFulfilledEvent(backorder.Id, backorder.OrderId, backorder.VariantId), ct);
        }

        await _db.SaveChangesAsync(ct);
    }
}

public record BackorderFulfilledEvent(
    Guid BackorderId, Guid OrderId, Guid VariantId) : DomainEvent;
```

---

## 10. Returns Impact on Inventory

Returned items must go through an inspection process before being added back to sellable inventory. Quality gates determine whether returned items are restocked, refurbished, or disposed.

### Return-to-Stock Flow

```csharp
namespace Inventory.Domain.Entities;

public class ReturnRequest
{
    public Guid Id { get; private set; }
    public Guid OrderId { get; private set; }
    public ReturnStatus Status { get; private set; }
    public string Reason { get; private set; } = string.Empty;
    public DateTime CreatedAt { get; private set; }
    public DateTime? ReceivedAt { get; private set; }
    public DateTime? CompletedAt { get; private set; }

    private readonly List<ReturnItem> _items = new();
    public IReadOnlyCollection<ReturnItem> Items => _items.AsReadOnly();

    public static ReturnRequest Create(Guid orderId, string reason, IEnumerable<ReturnItem> items)
    {
        var request = new ReturnRequest
        {
            Id = Guid.NewGuid(),
            OrderId = orderId,
            Status = ReturnStatus.Requested,
            Reason = reason,
            CreatedAt = DateTime.UtcNow
        };

        foreach (var item in items)
            request._items.Add(item);

        return request;
    }

    public void MarkReceived()
    {
        Status = ReturnStatus.Received;
        ReceivedAt = DateTime.UtcNow;
    }

    public void Complete()
    {
        if (_items.Any(i => i.InspectionResult == InspectionResult.Pending))
            throw new InvalidOperationException("All items must be inspected before completing.");

        Status = ReturnStatus.Completed;
        CompletedAt = DateTime.UtcNow;
    }
}

public class ReturnItem
{
    public Guid Id { get; private set; }
    public Guid ReturnRequestId { get; private set; }
    public Guid VariantId { get; private set; }
    public int Quantity { get; private set; }
    public InspectionResult InspectionResult { get; private set; }
    public string? InspectionNotes { get; private set; }
    public InventoryDisposition Disposition { get; private set; }
    public DateTime? InspectedAt { get; private set; }

    public void RecordInspection(
        InspectionResult result, InventoryDisposition disposition, string? notes = null)
    {
        InspectionResult = result;
        Disposition = disposition;
        InspectionNotes = notes;
        InspectedAt = DateTime.UtcNow;
    }
}

public enum ReturnStatus
{
    Requested,
    Approved,
    Received,
    Inspecting,
    Completed,
    Rejected
}

public enum InspectionResult
{
    Pending,
    PassedAsNew,
    MinorDefect,
    MajorDefect,
    Unsalvageable
}

public enum InventoryDisposition
{
    ReturnToStock,
    Refurbished,
    Damaged,
    Disposed
}
```

### Return Processing Service

```csharp
namespace Inventory.Application.Services;

public class ReturnProcessingService
{
    private readonly InventoryDbContext _db;
    private readonly IEventPublisher _eventPublisher;
    private readonly ILogger<ReturnProcessingService> _logger;

    public ReturnProcessingService(
        InventoryDbContext db,
        IEventPublisher eventPublisher,
        ILogger<ReturnProcessingService> logger)
    {
        _db = db;
        _eventPublisher = eventPublisher;
        _logger = logger;
    }

    public async Task ProcessInspectionAsync(
        Guid returnItemId,
        InspectionResult result,
        InventoryDisposition disposition,
        Guid warehouseId,
        string? notes,
        CancellationToken ct = default)
    {
        var item = await _db.Set<ReturnItem>()
            .FirstOrDefaultAsync(i => i.Id == returnItemId, ct)
            ?? throw new InvalidOperationException("Return item not found.");

        item.RecordInspection(result, disposition, notes);

        var record = await _db.InventoryRecords
            .FirstOrDefaultAsync(r =>
                r.VariantId == item.VariantId && r.WarehouseId == warehouseId, ct);

        if (record is null) return;

        switch (disposition)
        {
            case InventoryDisposition.ReturnToStock:
                record.ReceiveInbound(item.Quantity);
                _db.Set<StockMovement>().Add(StockMovement.Create(
                    item.VariantId, warehouseId, MovementType.Return,
                    item.Quantity, record.QuantityOnHand,
                    $"Return to stock: {result}", "system"));

                _logger.LogInformation(
                    "Returned {Quantity} units of {VariantId} to stock",
                    item.Quantity, item.VariantId);
                break;

            case InventoryDisposition.Damaged:
                record.MarkDamaged(item.Quantity);
                _db.Set<StockMovement>().Add(StockMovement.Create(
                    item.VariantId, warehouseId, MovementType.Damaged,
                    item.Quantity, record.QuantityOnHand,
                    $"Returned damaged: {notes}", "system"));
                break;

            case InventoryDisposition.Refurbished:
                // Add to refurbished inventory pool
                record.ReceiveInbound(item.Quantity);
                _db.Set<StockMovement>().Add(StockMovement.Create(
                    item.VariantId, warehouseId, MovementType.Return,
                    item.Quantity, record.QuantityOnHand,
                    $"Refurbished return: {notes}", "system"));
                break;

            case InventoryDisposition.Disposed:
                _db.Set<StockMovement>().Add(StockMovement.Create(
                    item.VariantId, warehouseId, MovementType.Shrinkage,
                    -item.Quantity, record.QuantityOnHand,
                    $"Disposed: {notes}", "system"));
                break;
        }

        await _db.SaveChangesAsync(ct);

        await _eventPublisher.PublishAsync(new ReturnProcessedEvent(
            item.ReturnRequestId, item.VariantId, disposition, item.Quantity), ct);
    }
}

public record ReturnProcessedEvent(
    Guid ReturnRequestId,
    Guid VariantId,
    InventoryDisposition Disposition,
    int Quantity) : DomainEvent;
```

---

## 11. Reporting and Analytics

Inventory analytics drive purchasing decisions, identify slow-moving stock, and measure fulfillment performance.

### Key Inventory Metrics

| Metric | Formula | Target |
|--------|---------|--------|
| **Inventory Turnover** | Cost of Goods Sold / Average Inventory | 4–8x per year |
| **Days of Supply** | On-Hand Inventory / Average Daily Sales | 30–60 days |
| **Stockout Rate** | Stockout Events / Total SKU-Days | < 2% |
| **Fill Rate** | Items Fulfilled / Items Ordered | > 95% |
| **Order Cycle Time** | Ship Date − Order Date | < 2 days |
| **Dead Stock %** | SKUs with 0 Sales (90d) / Total SKUs | < 5% |

### Analytics Query Service

```csharp
namespace Inventory.Application.Services;

public class InventoryAnalyticsService
{
    private readonly InventoryDbContext _db;

    public InventoryAnalyticsService(InventoryDbContext db) => _db = db;

    public async Task<InventoryTurnoverReport> GetTurnoverReportAsync(
        DateTime startDate, DateTime endDate, CancellationToken ct = default)
    {
        var movements = await _db.Set<StockMovement>()
            .Where(m => m.CreatedAt >= startDate && m.CreatedAt <= endDate)
            .ToListAsync(ct);

        var outbound = movements
            .Where(m => m.Type == MovementType.Outbound)
            .GroupBy(m => m.VariantId)
            .Select(g => new
            {
                VariantId = g.Key,
                TotalSold = g.Sum(m => Math.Abs(m.Quantity))
            })
            .ToList();

        var currentStock = await _db.InventoryRecords
            .GroupBy(r => r.VariantId)
            .Select(g => new
            {
                VariantId = g.Key,
                TotalOnHand = g.Sum(r => r.QuantityOnHand)
            })
            .ToListAsync(ct);

        var turnoverItems = outbound.Select(o =>
        {
            var stock = currentStock.FirstOrDefault(s => s.VariantId == o.VariantId);
            var avgInventory = (stock?.TotalOnHand ?? 0 + o.TotalSold) / 2.0m;
            var turnover = avgInventory > 0 ? o.TotalSold / avgInventory : 0;

            return new SkuTurnover(o.VariantId, o.TotalSold, stock?.TotalOnHand ?? 0, turnover);
        }).ToList();

        return new InventoryTurnoverReport(startDate, endDate, turnoverItems);
    }

    public async Task<IReadOnlyList<DeadStockItem>> GetDeadStockAsync(
        int daysSinceLastSale = 90, CancellationToken ct = default)
    {
        var cutoffDate = DateTime.UtcNow.AddDays(-daysSinceLastSale);

        var allVariants = await _db.Variants
            .Include(v => v.Product)
            .Where(v => v.IsActive)
            .ToListAsync(ct);

        var recentSales = await _db.Set<StockMovement>()
            .Where(m => m.Type == MovementType.Outbound && m.CreatedAt >= cutoffDate)
            .Select(m => m.VariantId)
            .Distinct()
            .ToListAsync(ct);

        var stockByVariant = await _db.InventoryRecords
            .GroupBy(r => r.VariantId)
            .Select(g => new { VariantId = g.Key, TotalOnHand = g.Sum(r => r.QuantityOnHand) })
            .ToDictionaryAsync(x => x.VariantId, x => x.TotalOnHand, ct);

        var deadStock = allVariants
            .Where(v => !recentSales.Contains(v.Id))
            .Select(v => new DeadStockItem(
                v.Id,
                v.Sku,
                v.Product.Name,
                stockByVariant.GetValueOrDefault(v.Id, 0)))
            .Where(d => d.QuantityOnHand > 0)
            .OrderByDescending(d => d.QuantityOnHand)
            .ToList();

        return deadStock;
    }

    public async Task<FillRateReport> GetFillRateAsync(
        DateTime startDate, DateTime endDate, CancellationToken ct = default)
    {
        var orders = await _db.Set<Order>()
            .Include(o => o.Items)
            .Where(o => o.CreatedAt >= startDate && o.CreatedAt <= endDate)
            .ToListAsync(ct);

        int totalItemsOrdered = orders.Sum(o => o.Items.Sum(i => i.Quantity));
        int totalItemsFulfilled = orders
            .SelectMany(o => o.Items)
            .Where(i => i.FulfillmentStatus == FulfillmentStatus.Fulfilled)
            .Sum(i => i.Quantity);

        decimal fillRate = totalItemsOrdered > 0
            ? (decimal)totalItemsFulfilled / totalItemsOrdered * 100
            : 0;

        return new FillRateReport(startDate, endDate, totalItemsOrdered, totalItemsFulfilled, fillRate);
    }

    public async Task<DaysOfSupplyReport> GetDaysOfSupplyAsync(
        Guid variantId, int lookbackDays = 30, CancellationToken ct = default)
    {
        var cutoff = DateTime.UtcNow.AddDays(-lookbackDays);

        var totalSold = await _db.Set<StockMovement>()
            .Where(m => m.VariantId == variantId &&
                        m.Type == MovementType.Outbound &&
                        m.CreatedAt >= cutoff)
            .SumAsync(m => Math.Abs(m.Quantity), ct);

        decimal avgDailySales = totalSold / (decimal)lookbackDays;

        var currentStock = await _db.InventoryRecords
            .Where(r => r.VariantId == variantId)
            .SumAsync(r => r.QuantityAvailable, ct);

        decimal daysOfSupply = avgDailySales > 0
            ? currentStock / avgDailySales
            : decimal.MaxValue;

        return new DaysOfSupplyReport(variantId, currentStock, avgDailySales, daysOfSupply);
    }
}

public record InventoryTurnoverReport(
    DateTime StartDate, DateTime EndDate, IReadOnlyList<SkuTurnover> Items);

public record SkuTurnover(Guid VariantId, int TotalSold, int CurrentStock, decimal TurnoverRatio);

public record DeadStockItem(Guid VariantId, string Sku, string ProductName, int QuantityOnHand);

public record FillRateReport(
    DateTime StartDate, DateTime EndDate,
    int TotalOrdered, int TotalFulfilled, decimal FillRatePercent);

public record DaysOfSupplyReport(
    Guid VariantId, int CurrentStock, decimal AvgDailySales, decimal DaysOfSupply);
```

### Inventory Dashboard Endpoint

```csharp
namespace Inventory.Api.Controllers;

[ApiController]
[Route("api/inventory/analytics")]
public class InventoryAnalyticsController : ControllerBase
{
    private readonly InventoryAnalyticsService _analytics;

    public InventoryAnalyticsController(InventoryAnalyticsService analytics)
    {
        _analytics = analytics;
    }

    [HttpGet("turnover")]
    public async Task<IActionResult> GetTurnover(
        [FromQuery] DateTime startDate,
        [FromQuery] DateTime endDate,
        CancellationToken ct)
    {
        var report = await _analytics.GetTurnoverReportAsync(startDate, endDate, ct);
        return Ok(report);
    }

    [HttpGet("dead-stock")]
    public async Task<IActionResult> GetDeadStock(
        [FromQuery] int days = 90, CancellationToken ct = default)
    {
        var items = await _analytics.GetDeadStockAsync(days, ct);
        return Ok(items);
    }

    [HttpGet("fill-rate")]
    public async Task<IActionResult> GetFillRate(
        [FromQuery] DateTime startDate,
        [FromQuery] DateTime endDate,
        CancellationToken ct)
    {
        var report = await _analytics.GetFillRateAsync(startDate, endDate, ct);
        return Ok(report);
    }

    [HttpGet("days-of-supply/{variantId:guid}")]
    public async Task<IActionResult> GetDaysOfSupply(
        Guid variantId,
        [FromQuery] int lookbackDays = 30,
        CancellationToken ct = default)
    {
        var report = await _analytics.GetDaysOfSupplyAsync(variantId, lookbackDays, ct);
        return Ok(report);
    }
}
```

---

## 12. Implementation Checklist

### Phase 1: Data Model and Core Inventory (Week 1–2)

- [ ] Design SKU, variant, and warehouse entities with EF Core
- [ ] Configure proper indexes (unique SKU, composite warehouse+variant)
- [ ] Implement row versioning for optimistic concurrency on inventory records
- [ ] Set up inventory status tracking (available, reserved, damaged, in-transit)
- [ ] Create stock movement audit table with complete event trail

### Phase 2: Reservation and Allocation (Week 2–3)

- [ ] Implement reserve-on-checkout with TTL-based expiry
- [ ] Add optimistic concurrency retry logic for race conditions
- [ ] Build background job for expired reservation cleanup
- [ ] Verify overselling prevention under concurrent load
- [ ] Integration test reservation → confirmation → fulfillment flow

### Phase 3: Order Lifecycle (Week 3–4)

- [ ] Implement order state machine with transition validation
- [ ] Emit domain events for every state transition
- [ ] Enforce valid state transitions (prevent Created → Shipped, etc.)
- [ ] Build order cancellation with inventory release

### Phase 4: Fulfillment and Shipping (Week 4–5)

- [ ] Build fulfillment orchestrator with split-shipment support
- [ ] Implement pick list generation sorted by warehouse location
- [ ] Integrate carrier API abstraction (FedEx, UPS, etc.)
- [ ] Implement rate comparison and label generation
- [ ] Set up shipping webhook handlers for tracking updates

### Phase 5: Warehouse Operations (Week 5–6)

- [ ] Implement multi-warehouse inventory with nearest-warehouse routing
- [ ] Build inter-warehouse transfer flow (request → ship → receive)
- [ ] Configure safety stock and reorder point policies
- [ ] Set up reorder monitoring background job

### Phase 6: Channels and Sync (Week 6–7)

- [ ] Implement event-driven inventory updates via message broker
- [ ] Build channel-specific inventory projections with buffers
- [ ] Add SignalR real-time stock broadcasting
- [ ] Test eventual consistency across web, mobile, and POS

### Phase 7: Backorders, Returns, and Analytics (Week 7–8)

- [ ] Implement backorder queue with FIFO processing
- [ ] Build pre-order flow with authorize-now / capture-at-ship
- [ ] Implement return-to-stock with quality inspection gates
- [ ] Build inventory analytics (turnover, fill rate, days of supply, dead stock)
- [ ] Set up monitoring alerts for stockout rate and reorder triggers

---

> **Next Steps:**
>
> - Review [06-DATABASE-DESIGN](06-DATABASE-DESIGN.md) for database schema patterns.
> - See [22-EVENT-DRIVEN-ARCHITECTURE](22-EVENT-DRIVEN-ARCHITECTURE.md) for domain event infrastructure.
> - See [18-COMMON-SCENARIOS](18-COMMON-SCENARIOS.md) for end-to-end checkout flows.
> - See [23-MICROSERVICES-PATTERNS](23-MICROSERVICES-PATTERNS.md) for service decomposition.
