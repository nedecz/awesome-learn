# Entity Framework Core and Data Access

## Table of Contents

1. [Overview](#overview)
2. [DbContext and Configuration](#dbcontext-and-configuration)
3. [Entity Configuration](#entity-configuration)
4. [Migrations](#migrations)
5. [Querying](#querying)
6. [Change Tracking](#change-tracking)
7. [Performance](#performance)
8. [Repository Pattern with EF Core](#repository-pattern-with-ef-core)
9. [Dapper](#dapper)
10. [Database Providers](#database-providers)
11. [Best Practices](#best-practices)
12. [Next Steps](#next-steps)

## Overview

Entity Framework Core (EF Core) is the recommended data access technology for .NET applications. It is an object-relational mapper (ORM) that enables developers to work with databases using .NET objects, eliminating most of the data-access plumbing code that would otherwise need to be written by hand.

### Target Audience

- .NET developers building data-driven applications
- Architects choosing between EF Core, Dapper, and raw ADO.NET
- Teams establishing data access patterns and migration strategies

### Scope

- DbContext lifetime management and configuration
- Entity configuration with Fluent API and conventions
- Code-first migrations and production deployment strategies
- LINQ querying, change tracking, and concurrency handling
- Performance optimization and N+1 prevention
- Repository pattern considerations
- Dapper for high-performance scenarios
- Database provider selection and configuration

### EF Core vs Dapper vs ADO.NET

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Data Access Technology Spectrum                    │
│                                                                     │
│  More Abstraction                              Less Abstraction     │
│  ◄──────────────────────────────────────────────────────────────►   │
│                                                                     │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐  │
│  │  EF Core     │    │   Dapper     │    │    Raw ADO.NET       │  │
│  │              │    │              │    │                      │  │
│  │ • Full ORM   │    │ • Micro-ORM  │    │ • Full control       │  │
│  │ • LINQ       │    │ • Raw SQL    │    │ • SqlCommand         │  │
│  │ • Migrations │    │ • Fast maps  │    │ • SqlDataReader      │  │
│  │ • Change     │    │ • No change  │    │ • Manual mapping     │  │
│  │   tracking   │    │   tracking   │    │ • Max performance    │  │
│  │ • Lazy load  │    │ • Simple API │    │ • Max complexity     │  │
│  └──────────────┘    └──────────────┘    └──────────────────────┘  │
│                                                                     │
│  Best for:           Best for:           Best for:                  │
│  CRUD-heavy apps,    Read-heavy,         Stored procs, legacy DB,  │
│  rapid development   reporting, perf-    absolute performance       │
│                      critical reads      requirements               │
└─────────────────────────────────────────────────────────────────────┘
```

| Feature | EF Core | Dapper | ADO.NET |
|---|---|---|---|
| Query language | LINQ | Raw SQL | Raw SQL |
| Change tracking | ✅ | ❌ | ❌ |
| Migrations | ✅ | ❌ | ❌ |
| Performance overhead | Moderate | Minimal | None |
| Learning curve | Moderate | Low | Low |
| Batch operations | ✅ (.NET 8+) | Manual | Manual |
| Relationship navigation | ✅ | Manual joins | Manual joins |

## DbContext and Configuration

### DbContext Lifecycle

The `DbContext` is the primary class for interacting with the database. It represents a session with the database and provides APIs for querying, saving, and configuring entities.

```
┌─────────────────────────────────────────────────────────────────┐
│                    DbContext Lifecycle                           │
│                                                                 │
│  ┌──────────────┐    ┌───────────────┐    ┌───────────────┐    │
│  │   Created    │───►│   In Use      │───►│   Disposed    │    │
│  │              │    │               │    │               │    │
│  │ • DI resolves│    │ • Query       │    │ • Connection  │    │
│  │ • Connection │    │ • Track       │    │   returned    │    │
│  │   not yet    │    │ • SaveChanges │    │ • Change      │    │
│  │   opened     │    │ • Transactions│    │   tracker     │    │
│  └──────────────┘    └───────────────┘    │   cleared     │    │
│                                           └───────────────┘    │
│                                                                 │
│  HTTP Request Scoped (default in ASP.NET Core):                 │
│                                                                 │
│  Request Start ──► DbContext Created ──► Used ──► Request End   │
│                    (scoped lifetime)              (disposed)     │
└─────────────────────────────────────────────────────────────────┘
```

### Basic Registration

```csharp
// Program.cs — register DbContext with DI
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
```

### DbContext Definition

```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Apply all configurations from the assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

### DbContextFactory

Use `IDbContextFactory<T>` when you need DbContext instances outside of the request scope — in background services, Blazor components, or parallel operations.

```csharp
// Registration
builder.Services.AddDbContextFactory<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

// Usage in a background service
public class OrderProcessingService(IDbContextFactory<AppDbContext> contextFactory)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // Each iteration gets its own short-lived DbContext
            await using var context = await contextFactory.CreateDbContextAsync(stoppingToken);

            var pendingOrders = await context.Orders
                .Where(o => o.Status == OrderStatus.Pending)
                .ToListAsync(stoppingToken);

            foreach (var order in pendingOrders)
            {
                order.Status = OrderStatus.Processing;
            }

            await context.SaveChangesAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
        }
    }
}
```

### DbContext Pooling

DbContext pooling reuses DbContext instances to reduce allocation overhead. The pool resets tracked entities when an instance is returned.

```csharp
// Pooling — reuses DbContext instances (pool size defaults to 1024)
builder.Services.AddDbContextPool<AppDbContext>(options =>
    options.UseSqlServer(connectionString), poolSize: 256);

// Pooled factory — combines pooling with factory pattern
builder.Services.AddPooledDbContextFactory<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
```

### Connection String Configuration

```json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=MyApp;Trusted_Connection=true;TrustServerCertificate=true;",
    "ReadReplica": "Server=readonly-server;Database=MyApp;Trusted_Connection=true;"
  }
}
```

## Entity Configuration

### Fluent API vs Data Annotations

Fluent API is the recommended approach — it keeps domain entities clean and provides the most configuration options.

| Feature | Fluent API | Data Annotations |
|---|---|---|
| Keeps entities clean | ✅ | ❌ (attributes on class) |
| Full feature access | ✅ | Partial |
| Composite keys | ✅ | ❌ |
| Advanced indexes | ✅ | Limited |
| Table splitting | ✅ | ❌ |
| Separation of concerns | ✅ | ❌ |

### IEntityTypeConfiguration

```csharp
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Products");
        builder.HasKey(p => p.Id);

        builder.Property(p => p.Name)
            .IsRequired()
            .HasMaxLength(200);

        builder.Property(p => p.Price)
            .HasPrecision(18, 2);

        builder.Property(p => p.Sku)
            .IsRequired()
            .HasMaxLength(50);

        builder.HasIndex(p => p.Sku)
            .IsUnique();

        // Relationships
        builder.HasOne(p => p.Category)
            .WithMany(c => c.Products)
            .HasForeignKey(p => p.CategoryId)
            .OnDelete(DeleteBehavior.Restrict);

        // Query filter for soft delete
        builder.HasQueryFilter(p => !p.IsDeleted);
    }
}
```

### Entity Classes

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Sku { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public bool IsDeleted { get; set; }
    public int CategoryId { get; set; }
    public Category Category { get; set; } = null!;
    public List<OrderItem> OrderItems { get; set; } = [];
}

public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public List<Product> Products { get; set; } = [];
}
```

### Owned Types

Owned types map value objects from domain-driven design. They share the table with the owner entity.

```csharp
public class Order
{
    public int Id { get; set; }
    public DateTime OrderDate { get; set; }
    public Address ShippingAddress { get; set; } = null!;
    public Address BillingAddress { get; set; } = null!;
    public Money TotalAmount { get; set; } = null!;
}

// Value objects — no identity of their own
public class Address
{
    public string Street { get; set; } = string.Empty;
    public string City { get; set; } = string.Empty;
    public string State { get; set; } = string.Empty;
    public string ZipCode { get; set; } = string.Empty;
}

public class Money
{
    public decimal Amount { get; set; }
    public string Currency { get; set; } = "USD";
}

// Configuration
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.OwnsOne(o => o.ShippingAddress, address =>
        {
            address.Property(a => a.Street).HasColumnName("ShippingStreet").HasMaxLength(200);
            address.Property(a => a.City).HasColumnName("ShippingCity").HasMaxLength(100);
            address.Property(a => a.State).HasColumnName("ShippingState").HasMaxLength(50);
            address.Property(a => a.ZipCode).HasColumnName("ShippingZip").HasMaxLength(20);
        });

        builder.OwnsOne(o => o.BillingAddress, address =>
        {
            address.Property(a => a.Street).HasColumnName("BillingStreet").HasMaxLength(200);
            address.Property(a => a.City).HasColumnName("BillingCity").HasMaxLength(100);
            address.Property(a => a.State).HasColumnName("BillingState").HasMaxLength(50);
            address.Property(a => a.ZipCode).HasColumnName("BillingZip").HasMaxLength(20);
        });

        builder.OwnsOne(o => o.TotalAmount, money =>
        {
            money.Property(m => m.Amount).HasColumnName("TotalAmount").HasPrecision(18, 2);
            money.Property(m => m.Currency).HasColumnName("TotalCurrency").HasMaxLength(3);
        });
    }
}
```

### Value Converters

Value converters translate property values between the .NET type and the database column type.

```csharp
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        // Enum to string converter
        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasMaxLength(50);

        // Custom converter — DateOnly to DateTime
        builder.Property(o => o.DueDate)
            .HasConversion(
                v => v.ToDateTime(TimeOnly.MinValue),
                v => DateOnly.FromDateTime(v));

        // Store list as JSON (EF Core 8+)
        builder.OwnsMany(o => o.Tags, tag =>
        {
            tag.ToJson();
        });
    }
}
```

### Shadow Properties

Shadow properties exist in the EF Core model but not on the .NET entity class. Useful for audit columns.

```csharp
public class AuditableEntityConfiguration<T> : IEntityTypeConfiguration<T> where T : class
{
    public void Configure(EntityTypeBuilder<T> builder)
    {
        // Shadow properties — not on the entity class
        builder.Property<DateTime>("CreatedAt").HasDefaultValueSql("GETUTCDATE()");
        builder.Property<DateTime>("UpdatedAt");
        builder.Property<string>("CreatedBy").HasMaxLength(100);
    }
}

// Setting shadow properties in SaveChanges override
public override int SaveChanges()
{
    foreach (var entry in ChangeTracker.Entries()
        .Where(e => e.State == EntityState.Added || e.State == EntityState.Modified))
    {
        entry.Property("UpdatedAt").CurrentValue = DateTime.UtcNow;

        if (entry.State == EntityState.Added)
        {
            entry.Property("CreatedAt").CurrentValue = DateTime.UtcNow;
        }
    }

    return base.SaveChanges();
}
```

## Migrations

### Code-First Migration Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                    Migration Workflow                            │
│                                                                 │
│  1. Modify Entity / Configuration                               │
│     │                                                           │
│     ▼                                                           │
│  2. dotnet ef migrations add <MigrationName>                    │
│     │  (creates migration files in /Migrations folder)          │
│     ▼                                                           │
│  3. Review generated migration                                  │
│     │  (check Up/Down methods for correctness)                  │
│     ▼                                                           │
│  4. dotnet ef database update                                   │
│     │  (applies migration to database)                          │
│     ▼                                                           │
│  5. Commit migration files to source control                    │
└─────────────────────────────────────────────────────────────────┘
```

### CLI Commands

```bash
# Install EF Core tools globally
dotnet tool install --global dotnet-ef

# Add a migration
dotnet ef migrations add InitialCreate --project src/MyApp.Data --startup-project src/MyApp.Api

# Apply migrations to the database
dotnet ef database update

# Remove the last unapplied migration
dotnet ef migrations remove

# Generate an idempotent SQL script (safe for production)
dotnet ef migrations script --idempotent --output migrate.sql

# Generate a migration bundle (self-contained executable)
dotnet ef migrations bundle --self-contained -r linux-x64
```

### Migration File Structure

```csharp
public partial class AddProductTable : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "Products",
            columns: table => new
            {
                Id = table.Column<int>(nullable: false)
                    .Annotation("SqlServer:Identity", "1, 1"),
                Name = table.Column<string>(maxLength: 200, nullable: false),
                Price = table.Column<decimal>(precision: 18, scale: 2, nullable: false),
                Sku = table.Column<string>(maxLength: 50, nullable: false)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_Products", x => x.Id);
            });

        migrationBuilder.CreateIndex(
            name: "IX_Products_Sku",
            table: "Products",
            column: "Sku",
            unique: true);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "Products");
    }
}
```

### Custom Migration Operations

```csharp
public partial class SeedCategories : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Seed data within migration
        migrationBuilder.InsertData(
            table: "Categories",
            columns: ["Id", "Name"],
            values: new object[,]
            {
                { 1, "Electronics" },
                { 2, "Clothing" },
                { 3, "Books" }
            });

        // Raw SQL for complex operations
        migrationBuilder.Sql("""
            CREATE VIEW vw_ActiveProducts AS
            SELECT Id, Name, Price, Sku
            FROM Products
            WHERE IsDeleted = 0
            """);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql("DROP VIEW IF EXISTS vw_ActiveProducts");
        migrationBuilder.DeleteData(table: "Categories", keyColumn: "Id", keyValues: [1, 2, 3]);
    }
}
```

### Production Migration Strategies

| Strategy | When to Use | Pros | Cons |
|---|---|---|---|
| Idempotent SQL scripts | CI/CD pipelines, DBA review | Reviewable, repeatable | Manual execution |
| Migration bundles | Containerized deployments | Self-contained executable | Binary artifact |
| `Database.Migrate()` at startup | Simple apps, dev environments | Automatic | Risky in multi-instance |
| Separate migration service | Kubernetes, distributed systems | Controlled rollout | Extra infrastructure |

```csharp
// Apply migrations at startup (development only)
if (app.Environment.IsDevelopment())
{
    using var scope = app.Services.CreateScope();
    var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await dbContext.Database.MigrateAsync();
}
```

## Querying

### Basic LINQ Queries

```csharp
public class ProductService(AppDbContext context)
{
    // Simple query with filtering and ordering
    public async Task<List<Product>> GetActiveProductsAsync(decimal minPrice)
    {
        return await context.Products
            .Where(p => p.Price >= minPrice)
            .OrderBy(p => p.Name)
            .ToListAsync();
    }

    // Projection — only load what you need
    public async Task<List<ProductDto>> GetProductSummariesAsync()
    {
        return await context.Products
            .Select(p => new ProductDto
            {
                Id = p.Id,
                Name = p.Name,
                Price = p.Price,
                CategoryName = p.Category.Name
            })
            .ToListAsync();
    }

    // Pagination
    public async Task<PagedResult<Product>> GetPagedAsync(int page, int pageSize)
    {
        var totalCount = await context.Products.CountAsync();

        var items = await context.Products
            .OrderBy(p => p.Id)
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync();

        return new PagedResult<Product>(items, totalCount, page, pageSize);
    }
}
```

### Eager, Explicit, and Lazy Loading

```csharp
// Eager loading — Include related data in the initial query
var orders = await context.Orders
    .Include(o => o.Customer)
    .Include(o => o.OrderItems)
        .ThenInclude(oi => oi.Product)
    .ToListAsync();

// Filtered include (EF Core 5+)
var orders = await context.Orders
    .Include(o => o.OrderItems.Where(oi => oi.Quantity > 0))
    .ToListAsync();

// Explicit loading — load related data on demand
var order = await context.Orders.FindAsync(orderId);
await context.Entry(order!).Collection(o => o.OrderItems).LoadAsync();
await context.Entry(order!).Reference(o => o.Customer).LoadAsync();

// Lazy loading — requires Microsoft.EntityFrameworkCore.Proxies (use sparingly)
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString)
           .UseLazyLoadingProxies());
```

### Split Queries

Split queries avoid the Cartesian explosion problem that occurs when loading multiple collection navigations.

```csharp
// Single query (default) — one SQL statement, potential Cartesian explosion
var orders = await context.Orders
    .Include(o => o.OrderItems)
    .Include(o => o.Payments)
    .ToListAsync();

// Split query — separate SQL queries for each Include
var orders = await context.Orders
    .Include(o => o.OrderItems)
    .Include(o => o.Payments)
    .AsSplitQuery()
    .ToListAsync();

// Configure as default for the DbContext
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString, sql =>
        sql.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery)));
```

### Compiled Queries

Compiled queries avoid LINQ expression tree compilation on every execution.

```csharp
public class ProductService(AppDbContext context)
{
    // Define once — reuse across invocations
    private static readonly Func<AppDbContext, decimal, IAsyncEnumerable<Product>>
        GetExpensiveProductsQuery = EF.CompileAsyncQuery(
            (AppDbContext ctx, decimal minPrice) =>
                ctx.Products.Where(p => p.Price >= minPrice).OrderBy(p => p.Name));

    public async Task<List<Product>> GetExpensiveProductsAsync(decimal minPrice)
    {
        var results = new List<Product>();

        await foreach (var product in GetExpensiveProductsQuery(context, minPrice))
        {
            results.Add(product);
        }

        return results;
    }
}
```

### Raw SQL Queries

```csharp
// Raw SQL with parameterized queries (SQL injection safe)
var minPrice = 100m;
var products = await context.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE Price > {minPrice}")
    .OrderBy(p => p.Name)
    .ToListAsync();

// Unmapped types with SqlQuery (EF Core 8+)
var stats = await context.Database
    .SqlQuery<ProductStats>($"""
        SELECT CategoryId, COUNT(*) AS ProductCount, AVG(Price) AS AveragePrice
        FROM Products
        GROUP BY CategoryId
        """)
    .ToListAsync();

// Non-query raw SQL
await context.Database.ExecuteSqlInterpolatedAsync(
    $"UPDATE Products SET Price = Price * {1.1m} WHERE CategoryId = {categoryId}");
```

### Global Query Filters

```csharp
// Applied in OnModelCreating or entity configuration
builder.HasQueryFilter(p => !p.IsDeleted);

// All queries automatically exclude soft-deleted products
var products = await context.Products.ToListAsync(); // WHERE IsDeleted = 0

// Bypass the filter when needed
var allProducts = await context.Products.IgnoreQueryFilters().ToListAsync();
```

### Tracking vs No-Tracking

```csharp
// Tracked (default) — entities are tracked for changes
var product = await context.Products.FirstOrDefaultAsync(p => p.Id == id);
product!.Price = 29.99m;
await context.SaveChangesAsync(); // UPDATE is generated

// No-tracking — read-only, better performance
var products = await context.Products
    .AsNoTracking()
    .Where(p => p.CategoryId == categoryId)
    .ToListAsync();

// No-tracking with identity resolution — no tracking but deduplicates entities
var orders = await context.Orders
    .AsNoTrackingWithIdentityResolution()
    .Include(o => o.OrderItems)
    .ToListAsync();

// Set no-tracking as default for the context
context.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
```

## Change Tracking

### Entity States

```
┌─────────────────────────────────────────────────────────────────┐
│                    Entity State Machine                          │
│                                                                 │
│  ┌───────────┐     Query      ┌────────────┐                   │
│  │ Detached  │ ──────────────►│ Unchanged  │                   │
│  └───────────┘                └─────┬──────┘                   │
│       ▲                             │                           │
│       │                      Modify property                    │
│       │                             │                           │
│       │                             ▼                           │
│       │                       ┌──────────┐                     │
│       │                       │ Modified │                     │
│       │                       └─────┬────┘                     │
│       │                             │                           │
│       │                       SaveChanges()                     │
│       │                             │                           │
│       │                             ▼                           │
│       │                       ┌────────────┐                   │
│       └───────────────────────│ Unchanged  │                   │
│                               └────────────┘                   │
│                                                                 │
│  ┌───────────┐   Add()       ┌───────┐    SaveChanges()       │
│  │ Detached  │ ─────────────►│ Added │ ──────────────►Unchanged│
│  └───────────┘               └───────┘                         │
│                                                                 │
│  ┌────────────┐  Remove()    ┌─────────┐  SaveChanges()       │
│  │ Unchanged  │ ────────────►│ Deleted │ ──────────────►Detached│
│  └────────────┘              └─────────┘                       │
└─────────────────────────────────────────────────────────────────┘
```

### SaveChanges Flow

```csharp
public class OrderService(AppDbContext context)
{
    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        var order = new Order
        {
            CustomerId = request.CustomerId,
            OrderDate = DateTime.UtcNow,
            Status = OrderStatus.Pending
        };

        foreach (var item in request.Items)
        {
            order.OrderItems.Add(new OrderItem
            {
                ProductId = item.ProductId,
                Quantity = item.Quantity,
                UnitPrice = item.UnitPrice
            });
        }

        context.Orders.Add(order);     // State = Added
        await context.SaveChangesAsync(); // INSERT statements generated

        return order; // order.Id is now populated
    }

    public async Task UpdateOrderStatusAsync(int orderId, OrderStatus newStatus)
    {
        var order = await context.Orders.FindAsync(orderId)
            ?? throw new InvalidOperationException($"Order {orderId} not found");

        order.Status = newStatus; // State = Modified (only Status column)
        await context.SaveChangesAsync(); // UPDATE only the changed column
    }
}
```

### Optimistic Concurrency

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }

    // Row version for concurrency — SQL Server uses rowversion/timestamp
    [Timestamp]
    public byte[] RowVersion { get; set; } = [];
}

// Fluent API configuration
builder.Property(p => p.RowVersion)
    .IsRowVersion();

// Handling concurrency conflicts
public async Task UpdateProductPriceAsync(int id, decimal newPrice)
{
    var product = await context.Products.FindAsync(id)
        ?? throw new InvalidOperationException("Product not found");

    product.Price = newPrice;

    try
    {
        await context.SaveChangesAsync();
    }
    catch (DbUpdateConcurrencyException ex)
    {
        var entry = ex.Entries.Single();
        var databaseValues = await entry.GetDatabaseValuesAsync()
            ?? throw new InvalidOperationException("Product was deleted");

        var databasePrice = databaseValues.GetValue<decimal>(nameof(Product.Price));

        // Strategy 1: Last write wins — overwrite with current values
        entry.OriginalValues.SetValues(databaseValues);
        await context.SaveChangesAsync();

        // Strategy 2: Inform the user
        // throw new ConcurrencyConflictException(
        //     $"Price was changed to {databasePrice} by another user.");
    }
}
```

### Bulk Operations (EF Core 7+)

```csharp
// ExecuteUpdate — update without loading entities
await context.Products
    .Where(p => p.CategoryId == oldCategoryId)
    .ExecuteUpdateAsync(s => s
        .SetProperty(p => p.CategoryId, newCategoryId)
        .SetProperty(p => p.Price, p => p.Price * 1.1m));

// ExecuteDelete — delete without loading entities
await context.Products
    .Where(p => p.IsDeleted && p.DeletedAt < DateTime.UtcNow.AddDays(-30))
    .ExecuteDeleteAsync();
```

## Performance

### The N+1 Problem

```csharp
// ❌ N+1 problem — one query per order to load items
var orders = await context.Orders.ToListAsync();
foreach (var order in orders)
{
    // Each iteration triggers a separate SQL query
    var items = await context.OrderItems
        .Where(i => i.OrderId == order.Id)
        .ToListAsync();
}

// ✅ Fix with eager loading — single query with JOIN
var orders = await context.Orders
    .Include(o => o.OrderItems)
    .ToListAsync();

// ✅ Fix with projection — only load what you need
var orderDtos = await context.Orders
    .Select(o => new OrderDto
    {
        Id = o.Id,
        ItemCount = o.OrderItems.Count,
        Total = o.OrderItems.Sum(i => i.UnitPrice * i.Quantity)
    })
    .ToListAsync();
```

### Indexing Strategies

```csharp
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        // Single column index
        builder.HasIndex(p => p.Sku).IsUnique();

        // Composite index
        builder.HasIndex(p => new { p.CategoryId, p.Price });

        // Filtered index (SQL Server)
        builder.HasIndex(p => p.Name)
            .HasFilter("[IsDeleted] = 0");

        // Include columns in index (covering index)
        builder.HasIndex(p => p.CategoryId)
            .IncludeProperties(p => new { p.Name, p.Price });
    }
}
```

### Connection Pooling

EF Core uses ADO.NET connection pooling automatically. Configure pool sizes via the connection string.

```csharp
// Connection string with pool settings
var connectionString = "Server=localhost;Database=MyApp;Min Pool Size=5;Max Pool Size=100;";

// DbContext pooling (reuse DbContext instances)
builder.Services.AddDbContextPool<AppDbContext>(options =>
    options.UseSqlServer(connectionString), poolSize: 256);
```

### Compiled Models (EF Core 6+)

Compiled models reduce startup time for large models by pre-generating the model configuration.

```bash
# Generate compiled model
dotnet ef dbcontext optimize --output-dir CompiledModels --namespace MyApp.CompiledModels
```

```csharp
// Use compiled model
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString)
           .UseModel(MyApp.CompiledModels.AppDbContextModel.Instance));
```

### Performance Comparison Table

| Technique | Impact | Effort | When to Use |
|---|---|---|---|
| `AsNoTracking()` | Medium | Low | Read-only queries |
| Projection (`Select`) | High | Low | API responses, DTOs |
| Compiled queries | Low–Medium | Low | Hot path queries |
| Split queries | Medium | Low | Multiple collection includes |
| `ExecuteUpdate/Delete` | High | Low | Bulk data changes |
| Proper indexing | High | Medium | Frequently filtered columns |
| Compiled models | High (startup) | Low | Large models (100+ entities) |
| DbContext pooling | Low–Medium | Low | High-throughput APIs |

## Repository Pattern with EF Core

### The Debate

DbContext already implements the Unit of Work pattern and DbSet acts as a repository. Adding another repository layer is often unnecessary.

```
┌─────────────────────────────────────────────────────────────────┐
│              DbContext IS a Unit of Work                         │
│                                                                 │
│  ┌─────────────────────────────────────────────────┐            │
│  │ DbContext                                       │            │
│  │  ┌──────────────────────┐                       │            │
│  │  │ DbSet<Product>       │ ◄── Repository        │            │
│  │  │ DbSet<Order>         │ ◄── Repository        │            │
│  │  │ DbSet<Customer>      │ ◄── Repository        │            │
│  │  └──────────────────────┘                       │            │
│  │  ChangeTracker              ◄── Unit of Work    │            │
│  │  SaveChanges()              ◄── Commit          │            │
│  └─────────────────────────────────────────────────┘            │
│                                                                 │
│  Adding IRepository<T> + IUnitOfWork on top often duplicates    │
│  what DbContext already provides with no added benefit.         │
└─────────────────────────────────────────────────────────────────┘
```

### When to Use a Repository

| Use Repository | Skip Repository |
|---|---|
| Abstracting away the data provider (e.g., swap EF for Dapper) | Application exclusively uses EF Core |
| Domain-driven design with rich domain models | Simple CRUD applications |
| Complex query encapsulation | Direct DbContext injection is sufficient |
| Shared libraries consumed by multiple projects | Small team, single project |

### Practical Repository (If You Choose to Use One)

```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<List<T>> GetAllAsync(CancellationToken ct = default);
    void Add(T entity);
    void Remove(T entity);
}

public class Repository<T>(AppDbContext context) : IRepository<T> where T : class
{
    public async Task<T?> GetByIdAsync(int id, CancellationToken ct = default)
        => await context.Set<T>().FindAsync([id], ct);

    public async Task<List<T>> GetAllAsync(CancellationToken ct = default)
        => await context.Set<T>().ToListAsync(ct);

    public void Add(T entity) => context.Set<T>().Add(entity);

    public void Remove(T entity) => context.Set<T>().Remove(entity);
}

// Specialized repository with domain-specific queries
public interface IProductRepository : IRepository<Product>
{
    Task<List<Product>> GetByCategoryAsync(int categoryId, CancellationToken ct = default);
    Task<Product?> GetBySkuAsync(string sku, CancellationToken ct = default);
}

public class ProductRepository(AppDbContext context) : Repository<Product>(context), IProductRepository
{
    public async Task<List<Product>> GetByCategoryAsync(int categoryId, CancellationToken ct = default)
        => await context.Products
            .Where(p => p.CategoryId == categoryId)
            .OrderBy(p => p.Name)
            .ToListAsync(ct);

    public async Task<Product?> GetBySkuAsync(string sku, CancellationToken ct = default)
        => await context.Products.FirstOrDefaultAsync(p => p.Sku == sku, ct);
}

// Unit of Work (if needed)
public interface IUnitOfWork
{
    IProductRepository Products { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}

public class UnitOfWork(AppDbContext context, IProductRepository products) : IUnitOfWork
{
    public IProductRepository Products => products;

    public async Task<int> SaveChangesAsync(CancellationToken ct = default)
        => await context.SaveChangesAsync(ct);
}
```

## Dapper

### When to Use Dapper

Dapper is a micro-ORM that maps SQL results to .NET objects with minimal overhead. Use it for read-heavy scenarios where you need full SQL control and maximum performance.

### Basic Dapper Usage

```csharp
using Dapper;

public class ProductReadService(IDbConnection connection)
{
    public async Task<IEnumerable<ProductDto>> GetProductsAsync(decimal minPrice)
    {
        const string sql = """
            SELECT p.Id, p.Name, p.Price, c.Name AS CategoryName
            FROM Products p
            INNER JOIN Categories c ON p.CategoryId = c.Id
            WHERE p.Price >= @MinPrice AND p.IsDeleted = 0
            ORDER BY p.Name
            """;

        return await connection.QueryAsync<ProductDto>(sql, new { MinPrice = minPrice });
    }

    public async Task<ProductDto?> GetByIdAsync(int id)
    {
        const string sql = "SELECT Id, Name, Price, Sku FROM Products WHERE Id = @Id";
        return await connection.QuerySingleOrDefaultAsync<ProductDto>(sql, new { Id = id });
    }

    // Multi-mapping for related data
    public async Task<IEnumerable<Order>> GetOrdersWithCustomerAsync()
    {
        const string sql = """
            SELECT o.*, c.*
            FROM Orders o
            INNER JOIN Customers c ON o.CustomerId = c.Id
            """;

        return await connection.QueryAsync<Order, Customer, Order>(
            sql,
            (order, customer) =>
            {
                order.Customer = customer;
                return order;
            },
            splitOn: "Id");
    }
}
```

### Dapper vs EF Core Comparison

| Feature | EF Core | Dapper |
|---|---|---|
| Query syntax | LINQ | Raw SQL |
| Change tracking | ✅ Automatic | ❌ None |
| Migrations | ✅ Built-in | ❌ External tool needed |
| Performance (reads) | Good | Excellent |
| Performance (writes) | Good (batched) | Manual batching |
| Complex mappings | Automatic | Manual |
| Learning curve | Moderate | Low |
| SQL control | Limited (raw SQL available) | Full |

### Hybrid Approach (EF Core + Dapper)

```csharp
// Use EF Core for writes and complex operations
// Use Dapper for read-heavy, performance-critical queries

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

// Register IDbConnection for Dapper (reuse the same connection string)
builder.Services.AddScoped<IDbConnection>(sp =>
    new SqlConnection(connectionString));

// Service that uses both
public class ProductService(AppDbContext context, IDbConnection connection)
{
    // Dapper for fast reads
    public async Task<IEnumerable<ProductDto>> SearchAsync(string term)
    {
        const string sql = """
            SELECT TOP 50 Id, Name, Price
            FROM Products
            WHERE Name LIKE @Term AND IsDeleted = 0
            ORDER BY Name
            """;
        return await connection.QueryAsync<ProductDto>(sql, new { Term = $"%{term}%" });
    }

    // EF Core for writes with change tracking
    public async Task<Product> CreateAsync(CreateProductRequest request)
    {
        var product = new Product
        {
            Name = request.Name,
            Price = request.Price,
            Sku = request.Sku,
            CategoryId = request.CategoryId
        };

        context.Products.Add(product);
        await context.SaveChangesAsync();
        return product;
    }
}
```

## Database Providers

### Supported Providers

| Provider | NuGet Package | Use Case |
|---|---|---|
| SQL Server | `Microsoft.EntityFrameworkCore.SqlServer` | Enterprise, Windows ecosystem |
| PostgreSQL | `Npgsql.EntityFrameworkCore.PostgreSQL` | Open source, Linux, JSON support |
| SQLite | `Microsoft.EntityFrameworkCore.Sqlite` | Embedded, mobile, testing |
| Cosmos DB | `Microsoft.EntityFrameworkCore.Cosmos` | NoSQL, globally distributed |
| In-Memory | `Microsoft.EntityFrameworkCore.InMemory` | Unit testing (limited) |

### SQL Server Configuration

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        sqlOptions.MigrationsAssembly("MyApp.Migrations");
        sqlOptions.EnableRetryOnFailure(
            maxRetryCount: 3,
            maxRetryDelay: TimeSpan.FromSeconds(10),
            errorNumbersToAdd: null);
        sqlOptions.CommandTimeout(30);
        sqlOptions.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery);
    }));
```

### PostgreSQL Configuration

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(connectionString, npgsqlOptions =>
    {
        npgsqlOptions.MigrationsAssembly("MyApp.Migrations");
        npgsqlOptions.EnableRetryOnFailure(maxRetryCount: 3);
        npgsqlOptions.MapEnum<OrderStatus>("order_status");
    }));
```

### SQLite Configuration (Testing)

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite("Data Source=app.db"));

// In-memory SQLite for testing
var connection = new SqliteConnection("Data Source=:memory:");
connection.Open();

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite(connection));
```

### Cosmos DB Configuration

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseCosmos(
        connectionString: builder.Configuration["Cosmos:ConnectionString"]!,
        databaseName: "MyAppDb",
        cosmosOptions =>
        {
            cosmosOptions.ConnectionMode(ConnectionMode.Direct);
            cosmosOptions.MaxRequestsPerTcpConnection(16);
        }));
```

## Best Practices

### Production Checklist

- ✅ Use `IEntityTypeConfiguration<T>` with Fluent API — keep entities clean
- ✅ Use `AsNoTracking()` for read-only queries
- ✅ Use projections (`Select`) to load only the columns you need
- ✅ Use `Include` or projections to avoid N+1 queries
- ✅ Use split queries when including multiple collections
- ✅ Use `ExecuteUpdate`/`ExecuteDelete` for bulk operations (EF Core 7+)
- ✅ Use compiled queries for frequently executed hot-path queries
- ✅ Use `IDbContextFactory` in background services and Blazor
- ✅ Use DbContext pooling for high-throughput APIs
- ✅ Always generate idempotent migration scripts for production deployments
- ✅ Use optimistic concurrency (`[Timestamp]`/`IsRowVersion()`) for conflict detection
- ✅ Configure connection resiliency (`EnableRetryOnFailure`)
- ✅ Review generated SQL with logging or `.ToQueryString()`
- ✅ Use global query filters for soft delete and multi-tenancy

- ❌ Don't call `Database.Migrate()` at startup in production — use migration bundles or scripts
- ❌ Don't register `DbContext` as Singleton — it is not thread-safe
- ❌ Don't use the In-Memory provider for integration tests — use SQLite instead
- ❌ Don't use lazy loading unless you have profiled and confirmed it is acceptable
- ❌ Don't return `IQueryable` from services — materialize queries before returning
- ❌ Don't ignore the change tracker for write operations — let EF Core manage entity state
- ❌ Don't create repository abstractions unless there is a clear architectural benefit
- ❌ Don't use `Find` in loops — batch reads with `Where(x => ids.Contains(x.Id))`

## Next Steps

Continue to [Testing](08-TESTING.md) to learn how to test EF Core data access code using xUnit, in-memory databases, and integration testing patterns.

### Related Documents

1. **[Learning Path](LEARNING-PATH.md)** — Complete .NET learning curriculum
2. **[Services](02-SERVICES.md)** — Web APIs, worker services, gRPC
3. **[Dependency Injection](06-DEPENDENCY-INJECTION.md)** — DI patterns, service lifetimes
4. **[Best Practices](03-BEST-PRACTICES.md)** — Async/await, memory, security, testing

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Entity Framework Core and data access documentation |
