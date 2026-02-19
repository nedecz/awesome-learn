# C# Coding Style Guide

## Table of Contents

1. [Overview](#overview)
2. [Naming Conventions](#naming-conventions)
3. [File and Class Organization](#file-and-class-organization)
4. [Code Formatting](#code-formatting)
5. [Using Directives](#using-directives)
6. [Null Handling Patterns](#null-handling-patterns)
7. [Pattern Matching](#pattern-matching)
8. [Record Types vs Classes](#record-types-vs-classes)
9. [Expression-Bodied Members](#expression-bodied-members)
10. [String Handling](#string-handling)
11. [EditorConfig and Analyzers](#editorconfig-and-analyzers)
12. [Next Steps](#next-steps)

## Overview

This style guide defines the coding conventions and formatting standards for C# code in this project. It is aligned with the [Microsoft C# Coding Conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions) and extended with additional team preferences.

### Target Audience

- All developers writing or reviewing C# code
- New team members onboarding to the codebase
- CI/CD engineers configuring code analysis tooling

### Scope

- Naming conventions for types, members, variables, and files
- Code formatting (braces, indentation, line length)
- Using directive organization and global usings
- Pattern matching, record types, and expression-bodied member style
- String handling best practices
- EditorConfig and Roslyn analyzer setup

## Naming Conventions

### Summary Table

| Element | Convention | Example |
|---------|-----------|---------|
| **Namespace** | PascalCase | `MyApp.Services` |
| **Class / Struct** | PascalCase | `OrderService` |
| **Interface** | `I` + PascalCase | `IOrderRepository` |
| **Method** | PascalCase | `GetOrderAsync` |
| **Property** | PascalCase | `OrderId` |
| **Event** | PascalCase | `OrderCreated` |
| **Enum type** | PascalCase (singular) | `OrderStatus` |
| **Enum value** | PascalCase | `OrderStatus.Completed` |
| **Public field** | PascalCase | `MaxRetryCount` |
| **Private field** | `_camelCase` | `_orderRepository` |
| **Local variable** | camelCase | `orderTotal` |
| **Parameter** | camelCase | `customerId` |
| **Constant** | PascalCase | `DefaultPageSize` |
| **Type parameter** | `T` + PascalCase | `TEntity`, `TKey` |
| **Async method** | PascalCase + `Async` suffix | `GetOrderAsync` |
| **Boolean** | `Is`, `Has`, `Can`, `Should` prefix | `IsActive`, `HasPermission` |

### Naming Examples

```csharp
// ✅ Good naming
public class OrderService : IOrderService
{
    private readonly IOrderRepository _orderRepository;
    private readonly ILogger<OrderService> _logger;
    private const int DefaultPageSize = 20;

    public OrderService(
        IOrderRepository orderRepository,
        ILogger<OrderService> logger)
    {
        _orderRepository = orderRepository;
        _logger = logger;
    }

    public async Task<Order?> GetOrderByIdAsync(
        int orderId, CancellationToken cancellationToken = default)
    {
        var order = await _orderRepository
            .GetByIdAsync(orderId, cancellationToken);

        if (order is null)
        {
            _logger.LogWarning("Order {OrderId} not found", orderId);
        }

        return order;
    }

    public bool IsOrderComplete(Order order) =>
        order.Status == OrderStatus.Completed;
}
```

```csharp
// ❌ Bad naming
public class orderSvc : IorderSvc           // Wrong casing
{
    private IOrderRepository orderRepo;      // Missing _ prefix
    private ILogger<orderSvc> Logger;        // Wrong casing for field
    private const int DEFAULT_PAGE_SIZE = 20; // SCREAMING_CASE

    public async Task<Order?> getOrder(      // Missing PascalCase, Async suffix
        int OrderId)                          // PascalCase parameter
    {
        var Order = await orderRepo           // PascalCase local variable
            .GetByIdAsync(OrderId);
        return Order;
    }
}
```

### Abbreviation Rules

```csharp
// ✅ Two-letter acronyms — both uppercase
public string IO { get; set; }
public int Id { get; set; }   // Exception: "Id" is always two-letter cased

// ✅ Three+ letter acronyms — PascalCase
public string XmlDocument { get; set; }
public string JsonSerializer { get; set; }
public string HttpClient { get; set; }

// ❌ All-caps for three+ letter acronyms
public string XMLDocument { get; set; }   // Wrong
public string JSONSerializer { get; set; } // Wrong
public string HTTPClient { get; set; }     // Wrong
```

## File and Class Organization

### One Type Per File

```
// ✅ Each public type in its own file
Services/
├── OrderService.cs           // Contains: class OrderService
├── IOrderService.cs          // Contains: interface IOrderService
├── OrderStatus.cs            // Contains: enum OrderStatus
└── OrderCreatedEvent.cs      // Contains: record OrderCreatedEvent
```

### Class Member Ordering

```csharp
public class OrderService : IOrderService
{
    // 1. Constants
    private const int MaxRetryAttempts = 3;

    // 2. Static fields
    private static readonly TimeSpan DefaultTimeout = TimeSpan.FromSeconds(30);

    // 3. Instance fields
    private readonly IOrderRepository _repository;
    private readonly ILogger<OrderService> _logger;

    // 4. Constructors
    public OrderService(IOrderRepository repository,
        ILogger<OrderService> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    // 5. Properties
    public int OrderCount { get; private set; }

    // 6. Public methods
    public async Task<Order> CreateOrderAsync(CreateOrderRequest request,
        CancellationToken ct = default)
    {
        // Implementation
    }

    // 7. Internal / protected methods
    internal void ValidateOrder(Order order) { }

    // 8. Private methods
    private async Task NotifyOrderCreated(Order order) { }

    // 9. Nested types (use sparingly)
    private record OrderCacheKey(int OrderId, string Region);
}
```

### File Layout

```csharp
// 1. File-scoped namespace (C# 10+)
namespace MyApp.Services;

// 2. Type definition
public class OrderService : IOrderService
{
    // Members...
}
```

## Code Formatting

### Braces

```csharp
// ✅ Allman style (new line for braces) — C# standard
public class OrderService
{
    public void Process()
    {
        if (condition)
        {
            DoSomething();
        }
        else
        {
            DoSomethingElse();
        }
    }
}

// ✅ Single-line body — braces optional for simple statements
if (order is null)
    return NotFound();

// ✅ But prefer braces for multi-line or nested blocks
if (order is null)
{
    _logger.LogWarning("Order not found");
    return NotFound();
}
```

### Indentation and Spacing

| Setting | Value |
|---------|-------|
| Indent style | Spaces |
| Indent size | 4 spaces |
| Max line length | 120 characters (soft limit) |
| Blank lines between members | 1 |
| Blank lines between sections | 1 |

```csharp
// ✅ Method parameters — wrap when line exceeds 120 characters
public async Task<OrderResult> CreateOrderAsync(
    CreateOrderRequest request,
    OrderOptions options,
    CancellationToken cancellationToken = default)
{
    // Method body
}

// ✅ Fluent chain — one call per line
var result = await _dbContext.Orders
    .Where(o => o.CustomerId == customerId)
    .OrderByDescending(o => o.CreatedAt)
    .Select(o => new OrderSummary(o.Id, o.Total))
    .ToListAsync(ct);
```

### Ternary and Null-Coalescing

```csharp
// ✅ Simple ternary — single line
var label = isActive ? "Active" : "Inactive";

// ✅ Complex ternary — multi-line
var result = order.Status switch
{
    OrderStatus.Pending => ProcessPending(order),
    OrderStatus.Completed => ProcessCompleted(order),
    _ => throw new InvalidOperationException($"Unknown status: {order.Status}")
};
```

## Using Directives

### Organization Order

```csharp
// 1. System namespaces
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

// 2. Microsoft namespaces
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;

// 3. Third-party namespaces
using FluentValidation;
using Serilog;

// 4. Project namespaces
using MyApp.Models;
using MyApp.Services;
```

### Global Usings (C# 10+)

```csharp
// GlobalUsings.cs — one file per project
global using System.Collections.Generic;
global using System.Linq;
global using System.Threading;
global using System.Threading.Tasks;
global using Microsoft.Extensions.Logging;
global using MyApp.Models;
```

### Implicit Usings (.NET 6+)

```xml
<!-- .csproj — enable implicit usings -->
<PropertyGroup>
  <ImplicitUsings>enable</ImplicitUsings>
</PropertyGroup>

<!-- Opt out of specific implicit usings -->
<ItemGroup>
  <Using Remove="System.Net.Http" />
</ItemGroup>

<!-- Add additional global usings via .csproj -->
<ItemGroup>
  <Using Include="MyApp.Models" />
  <Using Include="MyApp.Extensions" />
</ItemGroup>
```

## Null Handling Patterns

### Preferred Patterns

```csharp
// ✅ Guard clauses at method entry
public async Task<Order> ProcessOrderAsync(Order order)
{
    ArgumentNullException.ThrowIfNull(order);
    ArgumentException.ThrowIfNullOrWhiteSpace(order.CustomerId);

    // Method logic...
}

// ✅ Null-conditional for optional chains
var city = user?.Address?.City;
var count = collection?.Count ?? 0;

// ✅ Null-coalescing assignment
options.Timeout ??= TimeSpan.FromSeconds(30);

// ✅ Pattern matching for null + type checks
if (result is Order { Status: OrderStatus.Completed } completedOrder)
{
    await ProcessCompletedAsync(completedOrder);
}

// ✅ is not null (preferred over != null)
if (order is not null)
{
    await ProcessAsync(order);
}
```

### Nullable Annotations

```csharp
// ✅ Use [NotNullWhen] for Try-pattern methods
public bool TryGetOrder(int id, [NotNullWhen(true)] out Order? order)
{
    order = _cache.Get(id);
    return order is not null;
}

// ✅ Use [MemberNotNull] for initialization helpers
[MemberNotNull(nameof(_connection))]
private void EnsureConnected()
{
    _connection ??= CreateConnection();
}
```

## Pattern Matching

### Switch Expressions

```csharp
// ✅ Switch expression for mapping
public string GetStatusLabel(OrderStatus status) => status switch
{
    OrderStatus.Pending => "Pending Review",
    OrderStatus.Processing => "In Progress",
    OrderStatus.Completed => "Delivered",
    OrderStatus.Cancelled => "Cancelled",
    _ => throw new ArgumentOutOfRangeException(nameof(status))
};

// ✅ Property pattern matching
public decimal CalculateDiscount(Order order) => order switch
{
    { Total: > 1000 } => order.Total * 0.15m,
    { Total: > 500 } => order.Total * 0.10m,
    { Total: > 100, CustomerType: CustomerType.Premium } => order.Total * 0.08m,
    _ => 0m
};

// ✅ Relational and logical patterns
public string GetTemperatureLabel(double temp) => temp switch
{
    < 0 => "Freezing",
    >= 0 and < 15 => "Cold",
    >= 15 and < 25 => "Comfortable",
    >= 25 and < 35 => "Warm",
    >= 35 => "Hot"
};
```

### List Patterns (C# 11+)

```csharp
// ✅ List pattern matching
public string DescribeList(int[] numbers) => numbers switch
{
    [] => "Empty",
    [var single] => $"Single: {single}",
    [var first, .., var last] => $"Range: {first} to {last}",
};
```

## Record Types vs Classes

### When to Use Each

| Use | Record | Class |
|-----|--------|-------|
| DTOs / API responses | ✅ Preferred | ❌ Avoid |
| Value objects (DDD) | ✅ Preferred | ❌ Avoid |
| Event payloads | ✅ Preferred | ❌ Avoid |
| Services with behavior | ❌ Avoid | ✅ Preferred |
| Mutable state / entities | ❌ Avoid | ✅ Preferred |
| Configuration options | Depends | ✅ Preferred (mutable) |

```csharp
// ✅ Record for immutable data transfer
public record OrderDto(
    int Id,
    string CustomerId,
    decimal Total,
    OrderStatus Status,
    DateTimeOffset CreatedAt);

// ✅ Record with computed property
public record LineItem(string ProductId, int Quantity, decimal UnitPrice)
{
    public decimal Total => Quantity * UnitPrice;
}

// ✅ Record struct for small, stack-allocated values
public readonly record struct Coordinate(double Latitude, double Longitude);

// ✅ Class for service with behavior and mutable state
public class OrderService : IOrderService
{
    private readonly IOrderRepository _repository;

    public OrderService(IOrderRepository repository)
    {
        _repository = repository;
    }

    public async Task<Order> CreateAsync(CreateOrderRequest request) { }
}
```

## Expression-Bodied Members

```csharp
// ✅ Use for simple, single-expression members
public class Circle
{
    public double Radius { get; }

    public Circle(double radius) => Radius = radius;

    // Expression-bodied property
    public double Area => Math.PI * Radius * Radius;

    // Expression-bodied method
    public double Circumference() => 2 * Math.PI * Radius;

    // Expression-bodied method with string
    public override string ToString() => $"Circle(r={Radius})";
}

// ❌ Don't use expression body for complex logic
// ❌ public decimal CalculateTotal() => items.Where(i => i.IsActive)
//     .Sum(i => i.Price * i.Quantity) * (1 - discount) + shippingCost;

// ✅ Use block body for complex logic
public decimal CalculateTotal()
{
    var subtotal = _items
        .Where(i => i.IsActive)
        .Sum(i => i.Price * i.Quantity);

    var discountedTotal = subtotal * (1 - _discount);
    return discountedTotal + _shippingCost;
}
```

## String Handling

### String Interpolation

```csharp
// ✅ String interpolation (preferred over string.Format)
var message = $"Order {orderId} created for {customerName}";

// ✅ Verbatim interpolated string (multi-line)
var query = $@"
    SELECT *
    FROM Orders
    WHERE CustomerId = {customerId}
    AND Status = {status}";

// ✅ Raw string literals (C# 11+) — great for JSON, XML, SQL
var json = """
    {
        "name": "John",
        "age": 30,
        "email": "john@example.com"
    }
    """;

// ✅ Raw string with interpolation
var jsonWithValues = $$"""
    {
        "orderId": {{orderId}},
        "total": {{total}}
    }
    """;
```

### String Comparison

```csharp
// ✅ Always specify StringComparison for culture-dependent operations
var isMatch = name.Equals("admin", StringComparison.OrdinalIgnoreCase);
var contains = text.Contains("search", StringComparison.OrdinalIgnoreCase);
var index = path.IndexOf(".json", StringComparison.Ordinal);

// ✅ Use Ordinal for technical/non-linguistic strings
// (file paths, identifiers, protocol values)
if (contentType.StartsWith("application/json", StringComparison.Ordinal))
{
    // Process JSON
}

// ✅ Use OrdinalIgnoreCase for case-insensitive technical comparisons
var headers = new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase);
```

### String Performance

```csharp
// ✅ Use StringBuilder for concatenation in loops
var sb = new StringBuilder();
foreach (var item in items)
{
    sb.Append(item.Name).Append(": ").AppendLine(item.Value);
}
var result = sb.ToString();

// ✅ Use string.Create for performance-critical formatting
var formatted = string.Create(16, value, (span, v) =>
{
    v.TryFormat(span, out _);
});

// ✅ Use Span<char> for zero-allocation string operations
ReadOnlySpan<char> trimmed = input.AsSpan().Trim();
```

## EditorConfig and Analyzers

### Example .editorconfig

```ini
# .editorconfig
root = true

[*]
indent_style = space
indent_size = 4
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.cs]
# Namespace
csharp_style_namespace_declarations = file_scoped:warning

# Using directives
dotnet_sort_system_directives_first = true
csharp_using_directive_placement = outside_namespace:warning

# var preferences
csharp_style_var_for_built_in_types = false:suggestion
csharp_style_var_when_type_is_apparent = true:suggestion
csharp_style_var_elsewhere = false:suggestion

# Expression-bodied members
csharp_style_expression_bodied_methods = when_on_single_line:suggestion
csharp_style_expression_bodied_constructors = false:suggestion
csharp_style_expression_bodied_properties = true:suggestion
csharp_style_expression_bodied_accessors = true:suggestion

# Pattern matching
csharp_style_pattern_matching_over_is_with_cast_check = true:warning
csharp_style_pattern_matching_over_as_with_null_check = true:warning
csharp_style_prefer_switch_expression = true:suggestion
csharp_style_prefer_pattern_matching = true:suggestion

# Null checking
csharp_style_prefer_is_null_check_over_reference_equality_method = true:warning
dotnet_style_coalesce_expression = true:suggestion
dotnet_style_null_propagation = true:suggestion

# Modifier preferences
dotnet_style_require_accessibility_modifiers = for_non_interface_members:warning
csharp_preferred_modifier_order = public,private,protected,internal,static,extern,new,virtual,abstract,sealed,override,readonly,unsafe,volatile,async:warning

# Naming conventions
dotnet_naming_rule.private_fields_should_be_camel_case.severity = warning
dotnet_naming_rule.private_fields_should_be_camel_case.symbols = private_fields
dotnet_naming_rule.private_fields_should_be_camel_case.style = camel_case_underscore

dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private
dotnet_naming_symbols.private_fields.required_modifiers =

dotnet_naming_style.camel_case_underscore.capitalization = camel_case
dotnet_naming_style.camel_case_underscore.required_prefix = _

dotnet_naming_rule.interfaces_should_begin_with_i.severity = warning
dotnet_naming_rule.interfaces_should_begin_with_i.symbols = interfaces
dotnet_naming_rule.interfaces_should_begin_with_i.style = begins_with_i

dotnet_naming_symbols.interfaces.applicable_kinds = interface
dotnet_naming_style.begins_with_i.capitalization = pascal_case
dotnet_naming_style.begins_with_i.required_prefix = I

# Code style
csharp_style_prefer_primary_constructors = true:suggestion
csharp_style_prefer_top_level_statements = true:suggestion
csharp_prefer_braces = when_multiline:suggestion
csharp_prefer_simple_using_statement = true:suggestion

# New line preferences
csharp_new_line_before_open_brace = all
csharp_new_line_before_else = true
csharp_new_line_before_catch = true
csharp_new_line_before_finally = true

# Indentation
csharp_indent_case_contents = true
csharp_indent_switch_labels = true

# Spacing
csharp_space_after_cast = false
csharp_space_after_keywords_in_control_flow_statements = true
```

### Roslyn Analyzer Recommendations

| Analyzer Package | Purpose |
|-----------------|---------|
| `Microsoft.CodeAnalysis.NetAnalyzers` | Built-in .NET code quality rules (included by default) |
| `StyleCop.Analyzers` | Enforces consistent code style (ordering, spacing, documentation) |
| `Meziantou.Analyzer` | Additional .NET best practice rules |
| `SonarAnalyzer.CSharp` | OWASP security and code smell detection |
| `Roslynator.Analyzers` | 500+ code analysis and refactoring rules |

```xml
<!-- .csproj — enable recommended analyzers -->
<PropertyGroup>
  <AnalysisLevel>latest-recommended</AnalysisLevel>
  <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
</PropertyGroup>

<ItemGroup>
  <PackageReference Include="Meziantou.Analyzer" Version="2.*">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
  </PackageReference>
  <PackageReference Include="Roslynator.Analyzers" Version="4.*">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
  </PackageReference>
</ItemGroup>
```

### Suppressing Warnings

```csharp
// ✅ Suppress with justification
#pragma warning disable CA1822 // Member does not access instance data
public string GetVersion() => "1.0.0";
#pragma warning restore CA1822

// ✅ Suppress via attribute
[SuppressMessage("Design", "CA1062:Validate arguments of public methods",
    Justification = "Validated by framework model binding")]
public IActionResult Create(CreateRequest request) { }

// ✅ Suppress globally in .editorconfig
// [*.cs]
// dotnet_diagnostic.CA1062.severity = none
```

## Next Steps

Continue to the [Learning Path](LEARNING-PATH.md) for a structured 12-week curriculum covering C# fundamentals through production-ready .NET application development.

### Related Documents

1. **[Overview](00-OVERVIEW.md)** — .NET ecosystem and C# language evolution
2. **[Best Practices](03-BEST-PRACTICES.md)** — Async/await, memory, security, testing
3. **[Learning Path](LEARNING-PATH.md)** — Structured 12-week curriculum

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial C# coding style guide |
