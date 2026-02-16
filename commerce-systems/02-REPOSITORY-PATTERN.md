# Repository Pattern for Payment Systems

## Overview

The Repository Pattern provides an abstraction layer between domain/business logic and data access logic. In payment systems, this is critical for testability, provider-independence, and maintaining a clean separation of concerns.

## Table of Contents

1. [Why Repository Pattern for Payments](#why-repository-pattern-for-payments)
2. [Basic Repository](#basic-repository)
3. [Generic Repository](#generic-repository)
4. [Specification Pattern Integration](#specification-pattern-integration)
5. [Unit of Work Pattern](#unit-of-work-pattern)
6. [Repository with Caching](#repository-with-caching)
7. [Testing with Repositories](#testing-with-repositories)
8. [Anti-Patterns to Avoid](#anti-patterns-to-avoid)

---

## Why Repository Pattern for Payments

### Benefits

- **Testability**: Mock the repository in unit tests instead of hitting a database
- **Provider Flexibility**: Swap between SQL Server, PostgreSQL, CosmosDB without changing business logic
- **Encapsulated Queries**: Complex payment queries live in one place
- **Separation of Concerns**: Domain logic doesn't know about EF Core, Dapper, or raw SQL

### When to Use

| Scenario | Use Repository? |
|----------|----------------|
| CRUD operations on payments | ✅ Yes |
| Complex payment queries (reports, reconciliation) | ✅ Yes |
| Simple single-table lookups in small apps | ⚠️ Optional |
| Direct EF Core usage in controllers | ❌ Avoid |

---

## Basic Repository

### Interface Definition

```csharp
namespace PaymentSystem.Application.Interfaces
{
    public interface IPaymentRepository
    {
        // Query
        Task<Payment> GetByIdAsync(Guid id);
        Task<Payment> GetByOrderIdAsync(string orderId);
        Task<Payment> GetByTransactionIdAsync(string providerTransactionId);
        Task<IReadOnlyList<Payment>> GetByStatusAsync(PaymentStatus status);
        Task<IReadOnlyList<Payment>> GetByCustomerAsync(string customerEmail, int page, int pageSize);
        Task<IReadOnlyList<Payment>> GetPendingPaymentsAsync(TimeSpan olderThan);
        
        // Command
        Task AddAsync(Payment payment);
        Task UpdateAsync(Payment payment);
        
        // Aggregate queries
        Task<decimal> GetTotalRevenueAsync(DateTime startDate, DateTime endDate, string currency);
        Task<int> GetPaymentCountByStatusAsync(PaymentStatus status);
    }
}
```

### EF Core Implementation

```csharp
namespace PaymentSystem.Infrastructure.Repositories
{
    public class PaymentRepository : IPaymentRepository
    {
        private readonly ApplicationDbContext _context;

        public PaymentRepository(ApplicationDbContext context)
        {
            _context = context;
        }

        public async Task<Payment> GetByIdAsync(Guid id)
        {
            return await _context.Payments.FindAsync(id);
        }

        public async Task<Payment> GetByOrderIdAsync(string orderId)
        {
            return await _context.Payments
                .FirstOrDefaultAsync(p => p.OrderId == orderId);
        }

        public async Task<Payment> GetByTransactionIdAsync(string providerTransactionId)
        {
            return await _context.Payments
                .FirstOrDefaultAsync(p => p.ProviderTransactionId == providerTransactionId);
        }

        public async Task<IReadOnlyList<Payment>> GetByStatusAsync(PaymentStatus status)
        {
            return await _context.Payments
                .Where(p => p.Status == status)
                .OrderByDescending(p => p.CreatedAt)
                .ToListAsync();
        }

        public async Task<IReadOnlyList<Payment>> GetByCustomerAsync(
            string customerEmail, int page, int pageSize)
        {
            return await _context.Payments
                .Where(p => p.CustomerEmail == customerEmail)
                .OrderByDescending(p => p.CreatedAt)
                .Skip((page - 1) * pageSize)
                .Take(pageSize)
                .AsNoTracking()
                .ToListAsync();
        }

        public async Task<IReadOnlyList<Payment>> GetPendingPaymentsAsync(TimeSpan olderThan)
        {
            var cutoff = DateTime.UtcNow - olderThan;
            return await _context.Payments
                .Where(p => (p.Status == PaymentStatus.Pending || p.Status == PaymentStatus.Processing)
                         && p.CreatedAt < cutoff)
                .OrderBy(p => p.CreatedAt)
                .ToListAsync();
        }

        public async Task AddAsync(Payment payment)
        {
            await _context.Payments.AddAsync(payment);
            await _context.SaveChangesAsync();
        }

        public async Task UpdateAsync(Payment payment)
        {
            _context.Payments.Update(payment);
            await _context.SaveChangesAsync();
        }

        public async Task<decimal> GetTotalRevenueAsync(
            DateTime startDate, DateTime endDate, string currency)
        {
            return await _context.Payments
                .Where(p => p.Status == PaymentStatus.Completed
                         && p.Currency == currency
                         && p.CompletedAt >= startDate
                         && p.CompletedAt <= endDate)
                .SumAsync(p => p.Amount);
        }

        public async Task<int> GetPaymentCountByStatusAsync(PaymentStatus status)
        {
            return await _context.Payments
                .CountAsync(p => p.Status == status);
        }
    }
}
```

---

## Generic Repository

Use a generic base when multiple entity types follow the same patterns.

```csharp
public interface IRepository<T> where T : class
{
    Task<T> GetByIdAsync(Guid id);
    Task<IReadOnlyList<T>> GetAllAsync();
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(T entity);
}

public class Repository<T> : IRepository<T> where T : class
{
    protected readonly ApplicationDbContext _context;
    protected readonly DbSet<T> _dbSet;

    public Repository(ApplicationDbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public virtual async Task<T> GetByIdAsync(Guid id)
    {
        return await _dbSet.FindAsync(id);
    }

    public virtual async Task<IReadOnlyList<T>> GetAllAsync()
    {
        return await _dbSet.ToListAsync();
    }

    public virtual async Task AddAsync(T entity)
    {
        await _dbSet.AddAsync(entity);
        await _context.SaveChangesAsync();
    }

    public virtual async Task UpdateAsync(T entity)
    {
        _dbSet.Update(entity);
        await _context.SaveChangesAsync();
    }

    public virtual async Task DeleteAsync(T entity)
    {
        _dbSet.Remove(entity);
        await _context.SaveChangesAsync();
    }
}

// Payment-specific repository extends the generic base
public class PaymentRepository : Repository<Payment>, IPaymentRepository
{
    public PaymentRepository(ApplicationDbContext context) : base(context) { }

    public async Task<Payment> GetByOrderIdAsync(string orderId)
    {
        return await _dbSet.FirstOrDefaultAsync(p => p.OrderId == orderId);
    }

    // Additional payment-specific methods...
}
```

---

## Specification Pattern Integration

Encapsulate query criteria into reusable specification objects.

```csharp
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();

    public bool IsSatisfiedBy(T entity)
    {
        return ToExpression().Compile()(entity);
    }
}

// Concrete specifications
public class CompletedPaymentsSpec : Specification<Payment>
{
    public override Expression<Func<Payment, bool>> ToExpression()
    {
        return p => p.Status == PaymentStatus.Completed;
    }
}

public class PaymentsByDateRangeSpec : Specification<Payment>
{
    private readonly DateTime _start;
    private readonly DateTime _end;

    public PaymentsByDateRangeSpec(DateTime start, DateTime end)
    {
        _start = start;
        _end = end;
    }

    public override Expression<Func<Payment, bool>> ToExpression()
    {
        return p => p.CreatedAt >= _start && p.CreatedAt <= _end;
    }
}

public class HighValuePaymentsSpec : Specification<Payment>
{
    private readonly decimal _threshold;

    public HighValuePaymentsSpec(decimal threshold)
    {
        _threshold = threshold;
    }

    public override Expression<Func<Payment, bool>> ToExpression()
    {
        return p => p.Amount >= _threshold;
    }
}

// Combine specifications
public class AndSpecification<T> : Specification<T>
{
    private readonly Specification<T> _left;
    private readonly Specification<T> _right;

    public AndSpecification(Specification<T> left, Specification<T> right)
    {
        _left = left;
        _right = right;
    }

    public override Expression<Func<T, bool>> ToExpression()
    {
        var leftExpr = _left.ToExpression();
        var rightExpr = _right.ToExpression();

        var parameter = Expression.Parameter(typeof(T));
        var body = Expression.AndAlso(
            Expression.Invoke(leftExpr, parameter),
            Expression.Invoke(rightExpr, parameter));

        return Expression.Lambda<Func<T, bool>>(body, parameter);
    }
}

// Repository with specification support
public interface IPaymentRepository
{
    Task<IReadOnlyList<Payment>> FindAsync(Specification<Payment> spec);
}

public class PaymentRepository : IPaymentRepository
{
    public async Task<IReadOnlyList<Payment>> FindAsync(Specification<Payment> spec)
    {
        return await _context.Payments
            .Where(spec.ToExpression())
            .ToListAsync();
    }
}

// Usage
var spec = new AndSpecification<Payment>(
    new CompletedPaymentsSpec(),
    new HighValuePaymentsSpec(1000m)
);
var highValueCompleted = await _repository.FindAsync(spec);
```

---

## Unit of Work Pattern

Coordinate multiple repository operations within a single transaction.

```csharp
public interface IUnitOfWork : IDisposable
{
    IPaymentRepository Payments { get; }
    IRefundRepository Refunds { get; }
    IWebhookEventRepository WebhookEvents { get; }
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
    Task BeginTransactionAsync();
    Task CommitTransactionAsync();
    Task RollbackTransactionAsync();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly ApplicationDbContext _context;
    private IDbContextTransaction _transaction;

    public IPaymentRepository Payments { get; }
    public IRefundRepository Refunds { get; }
    public IWebhookEventRepository WebhookEvents { get; }

    public UnitOfWork(ApplicationDbContext context)
    {
        _context = context;
        Payments = new PaymentRepository(context);
        Refunds = new RefundRepository(context);
        WebhookEvents = new WebhookEventRepository(context);
    }

    public async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        return await _context.SaveChangesAsync(cancellationToken);
    }

    public async Task BeginTransactionAsync()
    {
        _transaction = await _context.Database.BeginTransactionAsync();
    }

    public async Task CommitTransactionAsync()
    {
        await _transaction.CommitAsync();
    }

    public async Task RollbackTransactionAsync()
    {
        await _transaction.RollbackAsync();
    }

    public void Dispose()
    {
        _transaction?.Dispose();
        _context.Dispose();
    }
}

// Usage: coordinated payment + refund
public class RefundService
{
    private readonly IUnitOfWork _unitOfWork;

    public async Task ProcessRefundAsync(Guid paymentId, decimal amount, string reason)
    {
        await _unitOfWork.BeginTransactionAsync();

        try
        {
            var payment = await _unitOfWork.Payments.GetByIdAsync(paymentId);
            payment.MarkAsRefunded();

            var refund = Refund.Create(paymentId, amount, reason);
            await _unitOfWork.Refunds.AddAsync(refund);

            await _unitOfWork.SaveChangesAsync();
            await _unitOfWork.CommitTransactionAsync();
        }
        catch
        {
            await _unitOfWork.RollbackTransactionAsync();
            throw;
        }
    }
}
```

---

## Repository with Caching

```csharp
public class CachedPaymentRepository : IPaymentRepository
{
    private readonly IPaymentRepository _inner;
    private readonly IMemoryCache _cache;
    private readonly TimeSpan _cacheDuration = TimeSpan.FromMinutes(5);

    public CachedPaymentRepository(IPaymentRepository inner, IMemoryCache cache)
    {
        _inner = inner;
        _cache = cache;
    }

    public async Task<Payment> GetByIdAsync(Guid id)
    {
        var cacheKey = $"payment:{id}";

        if (_cache.TryGetValue(cacheKey, out Payment cached))
            return cached;

        var payment = await _inner.GetByIdAsync(id);

        if (payment != null)
        {
            _cache.Set(cacheKey, payment, _cacheDuration);
        }

        return payment;
    }

    public async Task UpdateAsync(Payment payment)
    {
        await _inner.UpdateAsync(payment);
        _cache.Remove($"payment:{payment.Id}");
    }

    // Delegate other methods to _inner...
    public Task AddAsync(Payment payment) => _inner.AddAsync(payment);
    public Task<Payment> GetByOrderIdAsync(string orderId) => _inner.GetByOrderIdAsync(orderId);
    // etc.
}

// DI Registration
services.AddScoped<PaymentRepository>();
services.AddScoped<IPaymentRepository>(provider =>
    new CachedPaymentRepository(
        provider.GetRequiredService<PaymentRepository>(),
        provider.GetRequiredService<IMemoryCache>()));
```

---

## Testing with Repositories

### In-Memory Repository for Unit Tests

```csharp
public class InMemoryPaymentRepository : IPaymentRepository
{
    private readonly List<Payment> _payments = new();

    public Task<Payment> GetByIdAsync(Guid id)
    {
        return Task.FromResult(_payments.FirstOrDefault(p => p.Id == id));
    }

    public Task<Payment> GetByOrderIdAsync(string orderId)
    {
        return Task.FromResult(_payments.FirstOrDefault(p => p.OrderId == orderId));
    }

    public Task AddAsync(Payment payment)
    {
        _payments.Add(payment);
        return Task.CompletedTask;
    }

    public Task UpdateAsync(Payment payment)
    {
        var index = _payments.FindIndex(p => p.Id == payment.Id);
        if (index >= 0) _payments[index] = payment;
        return Task.CompletedTask;
    }

    // Additional methods...
}

// Usage in tests
[TestClass]
public class PaymentServiceTests
{
    [TestMethod]
    public async Task ProcessPayment_Success_SavesAndUpdatesPayment()
    {
        // Arrange
        var repository = new InMemoryPaymentRepository();
        var mockGateway = new Mock<IPaymentGateway>();
        mockGateway
            .Setup(g => g.ProcessPaymentAsync(It.IsAny<GatewayPaymentRequest>()))
            .ReturnsAsync(new GatewayResult { IsSuccess = true, TransactionId = "txn_123" });

        var service = new PaymentService(mockGateway.Object, repository, Mock.Of<ILogger<PaymentService>>());

        // Act
        var result = await service.ProcessPaymentAsync(new PaymentRequest
        {
            OrderId = "order-1",
            Amount = 50m,
            Currency = "USD",
            PaymentMethodId = "pm_test"
        });

        // Assert
        Assert.IsTrue(result.IsSuccess);
        var saved = await repository.GetByOrderIdAsync("order-1");
        Assert.IsNotNull(saved);
    }
}
```

---

## Anti-Patterns to Avoid

### ❌ Leaking IQueryable Outside the Repository

```csharp
// ❌ BAD: Exposes query details to callers
public interface IPaymentRepository
{
    IQueryable<Payment> Query(); // Don't do this
}

// ✅ GOOD: Return materialized results
public interface IPaymentRepository
{
    Task<IReadOnlyList<Payment>> GetByStatusAsync(PaymentStatus status);
}
```

### ❌ Calling SaveChanges in Every Method

```csharp
// ❌ BAD: No transaction control
public async Task AddAsync(Payment payment)
{
    await _context.Payments.AddAsync(payment);
    await _context.SaveChangesAsync(); // What if caller needs to add refund too?
}

// ✅ BETTER: Use Unit of Work for multi-entity operations
```

### ❌ Fat Repositories with Business Logic

```csharp
// ❌ BAD: Repository has business rules
public async Task ProcessPaymentAsync(PaymentRequest request)
{
    if (request.Amount > 10000) throw new Exception("Too high"); // Business logic doesn't belong here
    var payment = new Payment(...);
    await _context.Payments.AddAsync(payment);
}

// ✅ GOOD: Repository only handles data access; business logic stays in domain/application layer
```

---

## Summary

| Approach | Best For | Complexity |
|----------|----------|------------|
| Basic Repository | Small–medium projects | Low |
| Generic Repository | Multiple similar entities | Medium |
| Specification Pattern | Complex, composable queries | Medium |
| Unit of Work | Multi-entity transactions | Medium |
| Cached Repository | Read-heavy workloads | Medium |

## Next Steps

- [Strategy Pattern](03-STRATEGY-PATTERN.md)
- [Architecture Patterns](01-ARCHITECTURE-PATTERNS.md)
- [Database Design](06-DATABASE-DESIGN.md)
