# Database Design for Payment Systems

## Overview

Database design for payment systems requires careful consideration of consistency, auditability, performance, and compliance. This document covers schema design, best practices, and implementation patterns.

## Table of Contents

1. [Core Database Schema](#core-database-schema)
2. [Database Selection](#database-selection)
3. [Schema Design Patterns](#schema-design-patterns)
4. [Indexing Strategy](#indexing-strategy)
5. [Transaction Management](#transaction-management)
6. [Audit and Compliance](#audit-and-compliance)
7. [Performance Optimization](#performance-optimization)
8. [Backup and Recovery](#backup-and-recovery)

---

## Core Database Schema

### Essential Tables

#### 1. Payments Table

```sql
CREATE TABLE Payments (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    OrderId NVARCHAR(100) NOT NULL,
    Amount DECIMAL(18, 2) NOT NULL,
    Currency CHAR(3) NOT NULL,
    Status NVARCHAR(50) NOT NULL,
    
    -- Provider information
    Provider NVARCHAR(50) NOT NULL, -- 'Stripe', 'PayPal', etc.
    ProviderTransactionId NVARCHAR(255),
    PaymentMethodId NVARCHAR(255),
    
    -- Customer information (minimal)
    CustomerEmail NVARCHAR(256),
    CustomerIpAddress NVARCHAR(45), -- IPv6 compatible
    
    -- Timestamps
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    CompletedAt DATETIME2,
    CanceledAt DATETIME2,
    
    -- Additional fields
    ErrorMessage NVARCHAR(MAX),
    ErrorCode NVARCHAR(50),
    Metadata NVARCHAR(MAX), -- JSON for flexible storage
    
    -- Audit fields
    CreatedBy NVARCHAR(100),
    UpdatedBy NVARCHAR(100),
    Version INT NOT NULL DEFAULT 1, -- For optimistic concurrency
    
    CONSTRAINT CK_Payments_Amount CHECK (Amount >= 0),
    CONSTRAINT CK_Payments_Currency CHECK (LEN(Currency) = 3),
    CONSTRAINT CK_Payments_Status CHECK (Status IN (
        'Pending', 'Processing', 'Completed', 'Failed', 
        'Refunded', 'PartiallyRefunded', 'Canceled'
    ))
);

-- Indexes
CREATE INDEX IX_Payments_OrderId ON Payments(OrderId);
CREATE INDEX IX_Payments_ProviderTransactionId ON Payments(ProviderTransactionId);
CREATE INDEX IX_Payments_Status ON Payments(Status);
CREATE INDEX IX_Payments_CreatedAt ON Payments(CreatedAt DESC);
CREATE INDEX IX_Payments_CustomerEmail ON Payments(CustomerEmail);

-- Composite indexes for common queries
CREATE INDEX IX_Payments_Status_CreatedAt ON Payments(Status, CreatedAt DESC);
CREATE INDEX IX_Payments_Provider_Status ON Payments(Provider, Status);
```

#### 2. Refunds Table

```sql
CREATE TABLE Refunds (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    PaymentId UNIQUEIDENTIFIER NOT NULL,
    Amount DECIMAL(18, 2) NOT NULL,
    Currency CHAR(3) NOT NULL,
    Status NVARCHAR(50) NOT NULL,
    
    -- Provider information
    ProviderRefundId NVARCHAR(255),
    
    -- Reason for refund
    Reason NVARCHAR(500),
    ReasonCode NVARCHAR(50), -- 'customer_request', 'fraud', 'duplicate', etc.
    
    -- Timestamps
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    CompletedAt DATETIME2,
    
    -- Audit
    RequestedBy NVARCHAR(100),
    ApprovedBy NVARCHAR(100),
    
    CONSTRAINT FK_Refunds_Payment FOREIGN KEY (PaymentId) 
        REFERENCES Payments(Id),
    CONSTRAINT CK_Refunds_Amount CHECK (Amount > 0),
    CONSTRAINT CK_Refunds_Status CHECK (Status IN (
        'Pending', 'Processing', 'Completed', 'Failed', 'Canceled'
    ))
);

CREATE INDEX IX_Refunds_PaymentId ON Refunds(PaymentId);
CREATE INDEX IX_Refunds_Status ON Refunds(Status);
CREATE INDEX IX_Refunds_CreatedAt ON Refunds(CreatedAt DESC);
```

#### 3. PaymentEvents (Event Sourcing)

```sql
CREATE TABLE PaymentEvents (
    Id BIGINT IDENTITY(1,1) PRIMARY KEY,
    PaymentId UNIQUEIDENTIFIER NOT NULL,
    EventType NVARCHAR(100) NOT NULL,
    EventData NVARCHAR(MAX) NOT NULL, -- JSON
    
    -- Metadata
    Provider NVARCHAR(50),
    ProviderEventId NVARCHAR(255),
    
    -- Timestamps
    OccurredAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    ProcessedAt DATETIME2,
    
    -- Idempotency
    IdempotencyKey NVARCHAR(255),
    
    CONSTRAINT FK_PaymentEvents_Payment FOREIGN KEY (PaymentId) 
        REFERENCES Payments(Id)
);

CREATE INDEX IX_PaymentEvents_PaymentId ON PaymentEvents(PaymentId, OccurredAt);
CREATE INDEX IX_PaymentEvents_EventType ON PaymentEvents(EventType);
CREATE INDEX IX_PaymentEvents_OccurredAt ON PaymentEvents(OccurredAt DESC);
CREATE UNIQUE INDEX IX_PaymentEvents_IdempotencyKey ON PaymentEvents(IdempotencyKey) 
    WHERE IdempotencyKey IS NOT NULL;
```

#### 4. WebhookEvents Table

```sql
CREATE TABLE WebhookEvents (
    Id BIGINT IDENTITY(1,1) PRIMARY KEY,
    EventId NVARCHAR(255) NOT NULL UNIQUE,
    Provider NVARCHAR(50) NOT NULL,
    EventType NVARCHAR(100) NOT NULL,
    
    -- Payload
    RawPayload NVARCHAR(MAX) NOT NULL,
    
    -- Processing status
    ProcessingStatus NVARCHAR(50) NOT NULL DEFAULT 'Pending',
    RetryCount INT NOT NULL DEFAULT 0,
    MaxRetries INT NOT NULL DEFAULT 3,
    
    -- Related entity
    PaymentId UNIQUEIDENTIFIER,
    
    -- Timestamps
    ReceivedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    ProcessedAt DATETIME2,
    NextRetryAt DATETIME2,
    
    -- Error information
    ErrorMessage NVARCHAR(MAX),
    
    CONSTRAINT CK_WebhookEvents_ProcessingStatus CHECK (ProcessingStatus IN (
        'Pending', 'Processing', 'Completed', 'Failed', 'Skipped'
    ))
);

CREATE INDEX IX_WebhookEvents_EventId ON WebhookEvents(EventId);
CREATE INDEX IX_WebhookEvents_ProcessingStatus ON WebhookEvents(ProcessingStatus);
CREATE INDEX IX_WebhookEvents_NextRetryAt ON WebhookEvents(NextRetryAt) 
    WHERE NextRetryAt IS NOT NULL;
CREATE INDEX IX_WebhookEvents_PaymentId ON WebhookEvents(PaymentId);
```

#### 5. PaymentMethods Table (for saved cards)

```sql
CREATE TABLE PaymentMethods (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    CustomerId UNIQUEIDENTIFIER NOT NULL,
    
    -- Never store actual card numbers!
    ProviderPaymentMethodId NVARCHAR(255) NOT NULL, -- Token from provider
    Provider NVARCHAR(50) NOT NULL,
    
    -- Display information only
    CardBrand NVARCHAR(50), -- 'visa', 'mastercard', etc.
    Last4Digits CHAR(4),
    ExpiryMonth INT,
    ExpiryYear INT,
    
    -- Status
    IsDefault BIT NOT NULL DEFAULT 0,
    IsActive BIT NOT NULL DEFAULT 1,
    
    -- Timestamps
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    DeletedAt DATETIME2, -- Soft delete
    
    CONSTRAINT FK_PaymentMethods_Customer FOREIGN KEY (CustomerId) 
        REFERENCES Customers(Id),
    CONSTRAINT CK_PaymentMethods_ExpiryMonth CHECK (ExpiryMonth BETWEEN 1 AND 12)
);

CREATE INDEX IX_PaymentMethods_CustomerId ON PaymentMethods(CustomerId);
CREATE INDEX IX_PaymentMethods_IsActive ON PaymentMethods(IsActive);
```

#### 6. AuditLog Table

```sql
CREATE TABLE AuditLog (
    Id BIGINT IDENTITY(1,1) PRIMARY KEY,
    
    -- Entity information
    EntityType NVARCHAR(100) NOT NULL, -- 'Payment', 'Refund', etc.
    EntityId NVARCHAR(255) NOT NULL,
    
    -- Action
    Action NVARCHAR(50) NOT NULL, -- 'Created', 'Updated', 'Deleted', 'Refunded'
    
    -- Changes
    OldValues NVARCHAR(MAX), -- JSON
    NewValues NVARCHAR(MAX), -- JSON
    
    -- Context
    UserId NVARCHAR(100),
    IpAddress NVARCHAR(45),
    UserAgent NVARCHAR(500),
    
    -- Timestamp
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    
    -- Additional context
    Metadata NVARCHAR(MAX) -- JSON for flexible storage
);

CREATE INDEX IX_AuditLog_EntityType_EntityId ON AuditLog(EntityType, EntityId);
CREATE INDEX IX_AuditLog_CreatedAt ON AuditLog(CreatedAt DESC);
CREATE INDEX IX_AuditLog_UserId ON AuditLog(UserId);
```

---

## Database Selection

### SQL vs NoSQL Comparison

| Feature | SQL (PostgreSQL/SQL Server) | NoSQL (MongoDB/DynamoDB) |
|---------|----------------------------|--------------------------|
| ACID Transactions | ✅ Full support | ⚠️ Limited |
| Complex Queries | ✅ Excellent | ⚠️ Limited |
| Consistency | ✅ Strong | ⚠️ Eventual (typically) |
| Schema Evolution | ⚠️ Requires migrations | ✅ Flexible |
| Scaling | ⚠️ Vertical mainly | ✅ Horizontal |
| Audit Trail | ✅ Built-in features | ⚠️ Custom implementation |
| **Best for Payments?** | ✅ **Recommended** | ❌ Not recommended |

### Recommended Databases

#### 1. **PostgreSQL** (Recommended)

**Pros:**
- Excellent ACID compliance
- JSON/JSONB support for flexible fields
- Strong community support
- Open source
- Great performance
- Built-in full-text search

**Configuration:**
```sql
-- PostgreSQL Schema
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id VARCHAR(100) NOT NULL,
    amount NUMERIC(18, 2) NOT NULL CHECK (amount >= 0),
    currency CHAR(3) NOT NULL,
    status VARCHAR(50) NOT NULL,
    provider VARCHAR(50) NOT NULL,
    provider_transaction_id VARCHAR(255),
    customer_email VARCHAR(256),
    metadata JSONB, -- Flexible JSON storage
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    CONSTRAINT payments_status_check CHECK (status IN (
        'Pending', 'Processing', 'Completed', 'Failed', 
        'Refunded', 'PartiallyRefunded', 'Canceled'
    ))
);

-- GIN index for JSONB queries
CREATE INDEX idx_payments_metadata ON payments USING GIN (metadata);

-- Partial index for active payments
CREATE INDEX idx_payments_active ON payments(status) 
    WHERE status IN ('Pending', 'Processing');
```

#### 2. **SQL Server** (Enterprise)

**Pros:**
- Excellent tooling and support
- Strong integration with .NET
- Advanced features (temporal tables, columnstore)
- Enterprise-grade reliability

**Configuration:**
```sql
-- Enable temporal tables for automatic history
CREATE TABLE Payments (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    -- ... columns ...
    ValidFrom DATETIME2 GENERATED ALWAYS AS ROW START,
    ValidTo DATETIME2 GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.PaymentsHistory));
```

#### 3. **MySQL/MariaDB** (Alternative)

**Pros:**
- Widely supported
- Good performance
- Replication features

**Cons:**
- Less sophisticated than PostgreSQL
- Weaker JSON support

---

## Schema Design Patterns

### Pattern 1: Separate Tables for State Transitions

```sql
CREATE TABLE PaymentStateTransitions (
    Id BIGINT IDENTITY(1,1) PRIMARY KEY,
    PaymentId UNIQUEIDENTIFIER NOT NULL,
    FromStatus NVARCHAR(50),
    ToStatus NVARCHAR(50) NOT NULL,
    TransitionedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    TransitionedBy NVARCHAR(100),
    Reason NVARCHAR(500),
    
    CONSTRAINT FK_PaymentStateTransitions_Payment 
        FOREIGN KEY (PaymentId) REFERENCES Payments(Id)
);

CREATE INDEX IX_PaymentStateTransitions_PaymentId 
    ON PaymentStateTransitions(PaymentId, TransitionedAt);
```

### Pattern 2: Partitioning by Date

```sql
-- SQL Server - Partition by month
CREATE PARTITION FUNCTION PF_PaymentsByMonth (DATETIME2)
AS RANGE RIGHT FOR VALUES (
    '2024-01-01', '2024-02-01', '2024-03-01', 
    '2024-04-01', '2024-05-01', '2024-06-01'
    -- Add more as needed
);

CREATE PARTITION SCHEME PS_PaymentsByMonth
AS PARTITION PF_PaymentsByMonth
ALL TO ([PRIMARY]);

CREATE TABLE Payments (
    -- ... columns ...
    CreatedAt DATETIME2 NOT NULL
) ON PS_PaymentsByMonth(CreatedAt);
```

### Pattern 3: Soft Deletes

```sql
-- Add to all tables
ALTER TABLE Payments ADD DeletedAt DATETIME2 NULL;
ALTER TABLE Payments ADD IsDeleted AS (CASE WHEN DeletedAt IS NULL THEN 0 ELSE 1 END) PERSISTED;

-- Index for active records
CREATE INDEX IX_Payments_Active ON Payments(OrderId) WHERE IsDeleted = 0;

-- Views for easy querying
CREATE VIEW ActivePayments AS
SELECT * FROM Payments WHERE IsDeleted = 0;
```

---

## Indexing Strategy

### Query Performance Analysis

```sql
-- Common queries to optimize

-- 1. Get payment by order ID
CREATE INDEX IX_Payments_OrderId ON Payments(OrderId) 
    INCLUDE (Status, Amount, CreatedAt);

-- 2. Find pending payments
CREATE INDEX IX_Payments_Status_Created ON Payments(Status, CreatedAt DESC)
    WHERE Status IN ('Pending', 'Processing');

-- 3. Customer payment history
CREATE INDEX IX_Payments_Customer_Created ON Payments(CustomerEmail, CreatedAt DESC);

-- 4. Provider transactions
CREATE INDEX IX_Payments_Provider_Status ON Payments(Provider, Status)
    INCLUDE (ProviderTransactionId, Amount);

-- 5. Date range queries
CREATE INDEX IX_Payments_CreatedAt ON Payments(CreatedAt DESC)
    INCLUDE (Status, Amount, Currency);
```

### Index Maintenance

```sql
-- SQL Server - Rebuild fragmented indexes
DECLARE @TableName NVARCHAR(255) = 'Payments';
DECLARE @IndexName NVARCHAR(255);

DECLARE index_cursor CURSOR FOR
SELECT name FROM sys.indexes
WHERE object_id = OBJECT_ID(@TableName)
AND avg_fragmentation_in_percent > 30;

OPEN index_cursor;
FETCH NEXT FROM index_cursor INTO @IndexName;

WHILE @@FETCH_STATUS = 0
BEGIN
    EXEC('ALTER INDEX ' + @IndexName + ' ON ' + @TableName + ' REBUILD');
    FETCH NEXT FROM index_cursor INTO @IndexName;
END;

CLOSE index_cursor;
DEALLOCATE index_cursor;
```

---

## Transaction Management

### ACID Compliance for Payments

```csharp
public class PaymentRepository : IPaymentRepository
{
    private readonly ApplicationDbContext _context;
    
    public async Task<Payment> ProcessPaymentWithTransactionAsync(
        Payment payment,
        Func<Payment, Task> processAction)
    {
        using var transaction = await _context.Database.BeginTransactionAsync(
            IsolationLevel.Serializable); // Highest isolation
        
        try
        {
            // Lock the payment row
            var existingPayment = await _context.Payments
                .Where(p => p.Id == payment.Id)
                .FirstOrDefaultAsync();
            
            if (existingPayment == null)
            {
                await _context.Payments.AddAsync(payment);
            }
            else
            {
                _context.Payments.Update(payment);
            }
            
            // Process the payment
            await processAction(payment);
            
            // Log the event
            _context.PaymentEvents.Add(new PaymentEvent
            {
                PaymentId = payment.Id,
                EventType = "PaymentProcessed",
                EventData = JsonSerializer.Serialize(payment),
                OccurredAt = DateTime.UtcNow
            });
            
            await _context.SaveChangesAsync();
            await transaction.CommitAsync();
            
            return payment;
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

### Optimistic Concurrency

```csharp
public class Payment
{
    public Guid Id { get; set; }
    
    [Timestamp] // EF Core will handle this
    public byte[] RowVersion { get; set; }
    
    // Other properties...
}

// Usage
try
{
    payment.Status = PaymentStatus.Completed;
    await _context.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    // Handle concurrent update
    var entry = ex.Entries.Single();
    var databaseValues = await entry.GetDatabaseValuesAsync();
    
    if (databaseValues == null)
    {
        // Payment was deleted
        throw new InvalidOperationException("Payment no longer exists");
    }
    
    // Refresh and retry
    await entry.ReloadAsync();
    throw new ConcurrencyException("Payment was modified by another process");
}
```

---

## Audit and Compliance

### Automatic Audit Triggers

```sql
-- SQL Server trigger for audit logging
CREATE TRIGGER TR_Payments_Audit
ON Payments
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert operations
    INSERT INTO AuditLog (EntityType, EntityId, Action, NewValues, UserId, CreatedAt)
    SELECT 
        'Payment',
        CAST(i.Id AS NVARCHAR(255)),
        'Created',
        (SELECT i.* FOR JSON PATH, WITHOUT_ARRAY_WRAPPER),
        SYSTEM_USER,
        GETUTCDATE()
    FROM inserted i
    WHERE NOT EXISTS (SELECT 1 FROM deleted);
    
    -- Update operations
    INSERT INTO AuditLog (EntityType, EntityId, Action, OldValues, NewValues, UserId, CreatedAt)
    SELECT 
        'Payment',
        CAST(i.Id AS NVARCHAR(255)),
        'Updated',
        (SELECT d.* FOR JSON PATH, WITHOUT_ARRAY_WRAPPER),
        (SELECT i.* FOR JSON PATH, WITHOUT_ARRAY_WRAPPER),
        SYSTEM_USER,
        GETUTCDATE()
    FROM inserted i
    INNER JOIN deleted d ON i.Id = d.Id;
    
    -- Delete operations
    INSERT INTO AuditLog (EntityType, EntityId, Action, OldValues, UserId, CreatedAt)
    SELECT 
        'Payment',
        CAST(d.Id AS NVARCHAR(255)),
        'Deleted',
        (SELECT d.* FOR JSON PATH, WITHOUT_ARRAY_WRAPPER),
        NULL,
        SYSTEM_USER,
        GETUTCDATE()
    FROM deleted d
    WHERE NOT EXISTS (SELECT 1 FROM inserted);
END;
```

### EF Core Audit Interceptor

```csharp
public class AuditInterceptor : SaveChangesInterceptor
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    
    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData,
        InterceptionResult<int> result)
    {
        UpdateAuditFields(eventData.Context);
        return base.SavingChanges(eventData, result);
    }
    
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        UpdateAuditFields(eventData.Context);
        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }
    
    private void UpdateAuditFields(DbContext context)
    {
        var entries = context.ChangeTracker.Entries()
            .Where(e => e.Entity is IAuditable && 
                       (e.State == EntityState.Added || e.State == EntityState.Modified));
        
        var currentUser = _httpContextAccessor.HttpContext?.User?.Identity?.Name;
        var timestamp = DateTime.UtcNow;
        
        foreach (var entry in entries)
        {
            var auditable = (IAuditable)entry.Entity;
            
            if (entry.State == EntityState.Added)
            {
                auditable.CreatedAt = timestamp;
                auditable.CreatedBy = currentUser;
            }
            
            auditable.UpdatedAt = timestamp;
            auditable.UpdatedBy = currentUser;
        }
    }
}
```

---

## Performance Optimization

### Connection Pooling

```csharp
// appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=.;Database=PaymentDb;Trusted_Connection=True;MultipleActiveResultSets=true;Min Pool Size=10;Max Pool Size=100;Connection Lifetime=300"
  }
}
```

### Query Optimization

```csharp
// Use compiled queries for frequent operations
private static readonly Func<ApplicationDbContext, string, Task<Payment>> 
    GetPaymentByOrderIdQuery = EF.CompileAsyncQuery(
        (ApplicationDbContext context, string orderId) =>
            context.Payments
                .AsNoTracking()
                .FirstOrDefault(p => p.OrderId == orderId));

public async Task<Payment> GetByOrderIdAsync(string orderId)
{
    return await GetPaymentByOrderIdQuery(_context, orderId);
}

// Use projection to reduce data transfer
public async Task<PaymentSummaryDto> GetPaymentSummaryAsync(Guid paymentId)
{
    return await _context.Payments
        .Where(p => p.Id == paymentId)
        .Select(p => new PaymentSummaryDto
        {
            Id = p.Id,
            Status = p.Status,
            Amount = p.Amount,
            Currency = p.Currency
        })
        .FirstOrDefaultAsync();
}
```

### Caching Strategy

```csharp
public class CachedPaymentRepository : IPaymentRepository
{
    private readonly IPaymentRepository _inner;
    private readonly IDistributedCache _cache;
    private readonly TimeSpan _cacheDuration = TimeSpan.FromMinutes(5);
    
    public async Task<Payment> GetByIdAsync(Guid id)
    {
        var cacheKey = $"payment:{id}";
        var cached = await _cache.GetStringAsync(cacheKey);
        
        if (cached != null)
        {
            return JsonSerializer.Deserialize<Payment>(cached);
        }
        
        var payment = await _inner.GetByIdAsync(id);
        
        if (payment != null)
        {
            await _cache.SetStringAsync(
                cacheKey,
                JsonSerializer.Serialize(payment),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = _cacheDuration
                });
        }
        
        return payment;
    }
}
```

---

## Backup and Recovery

### Backup Strategy

```sql
-- SQL Server - Full backup
BACKUP DATABASE PaymentDb
TO DISK = 'C:\Backups\PaymentDb_Full.bak'
WITH FORMAT, INIT, COMPRESSION;

-- Transaction log backup (every 15 minutes)
BACKUP LOG PaymentDb
TO DISK = 'C:\Backups\PaymentDb_Log.trn'
WITH INIT, COMPRESSION;

-- Verify backup
RESTORE VERIFYONLY 
FROM DISK = 'C:\Backups\PaymentDb_Full.bak';
```

### Point-in-Time Recovery

```sql
-- Restore to specific time
RESTORE DATABASE PaymentDb
FROM DISK = 'C:\Backups\PaymentDb_Full.bak'
WITH NORECOVERY;

RESTORE LOG PaymentDb
FROM DISK = 'C:\Backups\PaymentDb_Log.trn'
WITH RECOVERY, STOPAT = '2024-02-15 14:30:00';
```

### Automated Backup Script

```csharp
public class DatabaseBackupService : BackgroundService
{
    private readonly IConfiguration _configuration;
    private readonly ILogger<DatabaseBackupService> _logger;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await BackupDatabaseAsync();
                await Task.Delay(TimeSpan.FromHours(6), stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Backup failed");
            }
        }
    }
    
    private async Task BackupDatabaseAsync()
    {
        var connectionString = _configuration.GetConnectionString("DefaultConnection");
        var backupPath = $"C:\\Backups\\PaymentDb_{DateTime.UtcNow:yyyyMMddHHmmss}.bak";
        
        using var connection = new SqlConnection(connectionString);
        await connection.OpenAsync();
        
        var command = connection.CreateCommand();
        command.CommandText = $@"
            BACKUP DATABASE PaymentDb
            TO DISK = '{backupPath}'
            WITH FORMAT, INIT, COMPRESSION;
        ";
        
        await command.ExecuteNonQueryAsync();
        _logger.LogInformation("Backup completed: {Path}", backupPath);
    }
}
```

---

## Best Practices Summary

### ✅ DO

1. **Use SQL databases** for payment systems (ACID compliance)
2. **Create comprehensive indexes** for common queries
3. **Implement audit logging** for all payment operations
4. **Use transactions** for multi-step operations
5. **Partition large tables** by date for performance
6. **Implement soft deletes** for data retention
7. **Use optimistic concurrency** to handle conflicts
8. **Cache read-heavy data** (payment methods, customer info)
9. **Regular backups** with point-in-time recovery
10. **Monitor query performance** and optimize slow queries

### ❌ DON'T

1. **Never store raw card data** in any database
2. **Don't use NoSQL** as primary payment database
3. **Don't skip indexes** on foreign keys
4. **Don't use SELECT *** in production queries
5. **Don't ignore transaction isolation levels**
6. **Don't delete audit records** (ever)
7. **Don't use GUID clustering keys** without consideration (sequential GUIDs better)
8. **Don't skip backup testing** (restore regularly)
9. **Don't store sensitive data** without encryption
10. **Don't ignore database monitoring** and alerts

---

## Migration Scripts Example

```csharp
public class AddPaymentIndexes : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql(@"
            CREATE INDEX IX_Payments_OrderId 
            ON Payments(OrderId) 
            INCLUDE (Status, Amount, CreatedAt);
            
            CREATE INDEX IX_Payments_Status_Created 
            ON Payments(Status, CreatedAt DESC)
            WHERE Status IN ('Pending', 'Processing');
        ");
    }
    
    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql(@"
            DROP INDEX IX_Payments_OrderId ON Payments;
            DROP INDEX IX_Payments_Status_Created ON Payments;
        ");
    }
}
```

---

## Monitoring Queries

```sql
-- Find slow queries (SQL Server)
SELECT TOP 10
    qs.execution_count,
    qs.total_elapsed_time / 1000000 AS total_elapsed_time_seconds,
    qs.total_elapsed_time / qs.execution_count / 1000 AS avg_elapsed_time_ms,
    SUBSTRING(qt.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(qt.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2)+1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
ORDER BY qs.total_elapsed_time DESC;

-- Find missing indexes
SELECT 
    migs.avg_user_impact,
    migs.avg_total_user_cost,
    migs.user_seeks + migs.user_scans AS total_reads,
    mid.statement AS table_name,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns
FROM sys.dm_db_missing_index_groups mig
INNER JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
INNER JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
ORDER BY migs.avg_user_impact DESC;
```

## Next Steps

- [Security Patterns & Practices](15-SECURITY-PATTERNS-PRACTICES.md)
- [Performance Optimization](08-PERFORMANCE-OPTIMIZATION.md)
- [CQRS and Event Sourcing](11-CQRS-EVENT-SOURCING.md)
