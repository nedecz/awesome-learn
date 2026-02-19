# Testing in .NET

## Table of Contents

1. [Overview](#overview)
2. [Unit Testing Frameworks](#unit-testing-frameworks)
3. [xUnit Deep Dive](#xunit-deep-dive)
4. [Mocking](#mocking)
5. [FluentAssertions](#fluentassertions)
6. [Integration Testing](#integration-testing)
7. [Test Architecture](#test-architecture)
8. [Testing ASP.NET Core](#testing-aspnet-core)
9. [Testing EF Core](#testing-ef-core)
10. [Code Coverage](#code-coverage)
11. [Best Practices](#best-practices)
12. [Next Steps](#next-steps)

## Overview

Testing is a foundational practice for building reliable .NET applications. A well-structured test suite catches regressions early, documents expected behavior, and gives teams confidence to refactor.

### Target Audience

- .NET developers writing unit, integration, and end-to-end tests
- Teams establishing testing conventions and CI/CD quality gates
- Architects designing testable application layers

### Scope

- Unit testing frameworks (xUnit, NUnit, MSTest) and when to use each
- Mocking with Moq and NSubstitute
- FluentAssertions for readable test assertions
- Integration testing with `WebApplicationFactory` and test containers
- Test architecture patterns (Arrange-Act-Assert, test data builders, AutoFixture)
- Testing ASP.NET Core controllers, minimal APIs, middleware, and EF Core
- Code coverage tools and meaningful coverage targets

### The Testing Pyramid

```
┌─────────────────────────────────────────────────────────────────┐
│                      Testing Pyramid                            │
│                                                                 │
│                         ╱╲                                      │
│                        ╱  ╲                                     │
│                       ╱ E2E╲         Few — slow, expensive      │
│                      ╱      ╲        UI / browser tests         │
│                     ╱────────╲                                  │
│                    ╱          ╲                                  │
│                   ╱Integration ╲     Some — test real I/O       │
│                  ╱              ╲    HTTP, database, queues      │
│                 ╱────────────────╲                               │
│                ╱                  ╲                              │
│               ╱    Unit Tests     ╲  Many — fast, isolated      │
│              ╱                    ╲  Business logic, services    │
│             ╱──────────────────────╲                             │
│                                                                 │
│  Speed:     Fast ◄──────────────────────────────────► Slow      │
│  Cost:      Cheap ◄─────────────────────────────────► Expensive │
│  Quantity:  Many ◄──────────────────────────────────► Few       │
│  Scope:     Narrow ◄────────────────────────────────► Broad     │
└─────────────────────────────────────────────────────────────────┘
```

### Test-Driven Development (TDD)

TDD follows a **Red → Green → Refactor** cycle:

1. **Red** — Write a failing test that describes the desired behavior
2. **Green** — Write the minimum code to make the test pass
3. **Refactor** — Improve the code while keeping all tests green

```
┌──────────┐      ┌──────────┐      ┌────────────┐
│   RED    │─────►│  GREEN   │─────►│  REFACTOR  │
│  Write   │      │  Make it │      │  Clean up  │
│  failing │      │  pass    │      │  code      │
│  test    │      │          │      │            │
└──────────┘      └──────────┘      └─────┬──────┘
     ▲                                     │
     └─────────────────────────────────────┘
```

## Unit Testing Frameworks

### Framework Comparison

| Feature | xUnit | NUnit | MSTest |
|---|---|---|---|
| Test attribute | `[Fact]` / `[Theory]` | `[Test]` / `[TestCase]` | `[TestMethod]` |
| Parameterized tests | `[InlineData]`, `[ClassData]`, `[MemberData]` | `[TestCase]`, `[TestCaseSource]` | `[DataRow]`, `[DynamicData]` |
| Setup/Teardown | Constructor / `IDisposable` | `[SetUp]` / `[TearDown]` | `[TestInitialize]` / `[TestCleanup]` |
| Class-level setup | `IClassFixture<T>` | `[OneTimeSetUp]` | `[ClassInitialize]` |
| Parallel by default | ✅ Yes (per class) | ❌ No | ❌ No |
| Assertions | `Assert.Equal`, etc. | `Assert.That` (constraint model) | `Assert.AreEqual`, etc. |
| Community adoption | Most popular for new projects | Established, mature | Microsoft-backed |
| .NET template | `dotnet new xunit` | `dotnet new nunit` | `dotnet new mstest` |

### Recommendation

**xUnit** is the recommended framework for new .NET projects. It is the most widely used in the .NET ecosystem, has first-class support in Visual Studio and JetBrains Rider, and follows modern design patterns (constructor injection for setup, `IDisposable` for teardown).

### Project Setup

```bash
# Create xUnit test project
dotnet new xunit -n MyApp.Tests

# Add references
dotnet add MyApp.Tests reference ../MyApp/MyApp.csproj

# Add common test packages
dotnet add MyApp.Tests package Moq
dotnet add MyApp.Tests package FluentAssertions
dotnet add MyApp.Tests package Microsoft.AspNetCore.Mvc.Testing
```

## xUnit Deep Dive

### Facts and Theories

```csharp
public class CalculatorTests
{
    // [Fact] — a test that is always true, no parameters
    [Fact]
    public void Add_TwoPositiveNumbers_ReturnsSum()
    {
        var calculator = new Calculator();

        var result = calculator.Add(2, 3);

        Assert.Equal(5, result);
    }

    // [Theory] — a parameterized test, runs once per data set
    [Theory]
    [InlineData(2, 3, 5)]
    [InlineData(-1, 1, 0)]
    [InlineData(0, 0, 0)]
    [InlineData(int.MaxValue, 0, int.MaxValue)]
    public void Add_WithVariousInputs_ReturnsExpectedSum(int a, int b, int expected)
    {
        var calculator = new Calculator();

        var result = calculator.Add(a, b);

        Assert.Equal(expected, result);
    }
}
```

### ClassData and MemberData

```csharp
// [ClassData] — test data from a dedicated class
public class DivisionTestData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        yield return [10, 2, 5.0];
        yield return [9, 3, 3.0];
        yield return [7, 2, 3.5];
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

public class CalculatorDivisionTests
{
    [Theory]
    [ClassData(typeof(DivisionTestData))]
    public void Divide_ValidInputs_ReturnsExpectedResult(int a, int b, double expected)
    {
        var calculator = new Calculator();

        var result = calculator.Divide(a, b);

        Assert.Equal(expected, result);
    }

    // [MemberData] — test data from a static property or method
    public static IEnumerable<object[]> NegativeTestCases =>
    [
        [0, "Divisor cannot be zero"],
        [-1, null!]  // null expected means no exception
    ];

    [Theory]
    [MemberData(nameof(NegativeTestCases))]
    public void Divide_ByZero_ThrowsException(int divisor, string? expectedMessage)
    {
        var calculator = new Calculator();

        if (expectedMessage is not null)
        {
            var ex = Assert.Throws<DivideByZeroException>(() => calculator.Divide(10, divisor));
            Assert.Equal(expectedMessage, ex.Message);
        }
    }
}
```

### Test Lifecycle

```csharp
// Constructor runs before EACH test
// IDisposable.Dispose runs after EACH test
public class DatabaseTests : IDisposable
{
    private readonly TestDatabase _db;

    public DatabaseTests()
    {
        // Arrange — runs before each test
        _db = new TestDatabase();
        _db.Seed();
    }

    [Fact]
    public void Query_ReturnsSeededData()
    {
        var result = _db.GetAll();
        Assert.NotEmpty(result);
    }

    public void Dispose()
    {
        // Cleanup — runs after each test
        _db.Dispose();
    }
}

// IAsyncLifetime — for async setup and teardown
public class AsyncDatabaseTests : IAsyncLifetime
{
    private AppDbContext _context = null!;

    public async Task InitializeAsync()
    {
        // Async setup
        _context = await TestDbContextFactory.CreateAsync();
        await _context.Database.MigrateAsync();
    }

    [Fact]
    public async Task InsertAndQuery_ReturnsEntity()
    {
        _context.Products.Add(new Product { Name = "Test", Price = 9.99m, Sku = "TST-001" });
        await _context.SaveChangesAsync();

        var product = await _context.Products.FirstAsync();
        Assert.Equal("Test", product.Name);
    }

    public async Task DisposeAsync()
    {
        // Async cleanup
        await _context.Database.EnsureDeletedAsync();
        await _context.DisposeAsync();
    }
}
```

### Shared Context with Fixtures

```csharp
// Class fixture — shared across all tests in a single test class
public class DatabaseFixture : IAsyncLifetime
{
    public AppDbContext Context { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        Context = await TestDbContextFactory.CreateAsync();
        await Context.Database.EnsureCreatedAsync();
    }

    public async Task DisposeAsync()
    {
        await Context.Database.EnsureDeletedAsync();
        await Context.DisposeAsync();
    }
}

public class ProductTests(DatabaseFixture fixture) : IClassFixture<DatabaseFixture>
{
    [Fact]
    public async Task Create_ValidProduct_Persists()
    {
        fixture.Context.Products.Add(new Product { Name = "Widget", Price = 5m, Sku = "WDG-001" });
        await fixture.Context.SaveChangesAsync();

        var count = await fixture.Context.Products.CountAsync();
        Assert.True(count > 0);
    }
}

// Collection fixture — shared across multiple test classes
[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture> { }

[Collection("Database")]
public class OrderTests(DatabaseFixture fixture)
{
    [Fact]
    public async Task CreateOrder_InsertsRecord()
    {
        fixture.Context.Orders.Add(new Order { OrderDate = DateTime.UtcNow });
        await fixture.Context.SaveChangesAsync();

        Assert.True(await fixture.Context.Orders.AnyAsync());
    }
}
```

## Mocking

### Moq

```csharp
public class OrderServiceTests
{
    private readonly Mock<IOrderRepository> _repoMock = new();
    private readonly Mock<IEmailService> _emailMock = new();
    private readonly OrderService _sut;

    public OrderServiceTests()
    {
        _sut = new OrderService(_repoMock.Object, _emailMock.Object);
    }

    [Fact]
    public async Task PlaceOrder_ValidOrder_SavesAndSendsEmail()
    {
        // Arrange
        var order = new Order { Id = 1, CustomerId = 42 };

        _repoMock
            .Setup(r => r.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()))
            .Returns(Task.CompletedTask);

        _emailMock
            .Setup(e => e.SendOrderConfirmationAsync(It.IsAny<int>(), It.IsAny<string>()))
            .Returns(Task.CompletedTask);

        // Act
        await _sut.PlaceOrderAsync(order);

        // Assert
        _repoMock.Verify(r => r.AddAsync(order, default), Times.Once);
        _emailMock.Verify(
            e => e.SendOrderConfirmationAsync(42, It.Is<string>(s => s.Contains("Order #1"))),
            Times.Once);
    }

    [Fact]
    public async Task GetOrder_NotFound_ReturnsNull()
    {
        _repoMock
            .Setup(r => r.GetByIdAsync(It.IsAny<int>(), default))
            .ReturnsAsync((Order?)null);

        var result = await _sut.GetOrderAsync(999);

        Assert.Null(result);
    }

    // Argument matchers
    [Fact]
    public async Task PlaceOrder_SetsTimestamp()
    {
        _repoMock
            .Setup(r => r.AddAsync(
                It.Is<Order>(o => o.OrderDate > DateTime.UtcNow.AddMinutes(-1)),
                It.IsAny<CancellationToken>()))
            .Returns(Task.CompletedTask);

        await _sut.PlaceOrderAsync(new Order { CustomerId = 1 });

        _repoMock.Verify(r => r.AddAsync(
            It.Is<Order>(o => o.OrderDate != default),
            default));
    }
}
```

### NSubstitute

```csharp
public class OrderServiceNSubTests
{
    private readonly IOrderRepository _repo = Substitute.For<IOrderRepository>();
    private readonly IEmailService _email = Substitute.For<IEmailService>();
    private readonly OrderService _sut;

    public OrderServiceNSubTests()
    {
        _sut = new OrderService(_repo, _email);
    }

    [Fact]
    public async Task PlaceOrder_ValidOrder_SavesAndSendsEmail()
    {
        // Arrange — NSubstitute uses natural syntax
        var order = new Order { Id = 1, CustomerId = 42 };

        // Act
        await _sut.PlaceOrderAsync(order);

        // Assert
        await _repo.Received(1).AddAsync(order, Arg.Any<CancellationToken>());
        await _email.Received(1).SendOrderConfirmationAsync(42, Arg.Is<string>(s => s.Contains("Order #1")));
    }

    [Fact]
    public async Task GetOrder_ReturnsConfiguredResult()
    {
        var expected = new Order { Id = 5, CustomerId = 10 };
        _repo.GetByIdAsync(5, default).Returns(expected);

        var result = await _sut.GetOrderAsync(5);

        Assert.Equal(expected, result);
    }
}
```

### Moq vs NSubstitute Comparison

| Feature | Moq | NSubstitute |
|---|---|---|
| Setup syntax | `.Setup(x => x.Method()).Returns(value)` | `sub.Method().Returns(value)` |
| Verification | `.Verify(x => x.Method(), Times.Once)` | `sub.Received(1).Method()` |
| Argument matching | `It.Is<T>(predicate)`, `It.IsAny<T>()` | `Arg.Is<T>(predicate)`, `Arg.Any<T>()` |
| Syntax style | Lambda-heavy, explicit | Natural, reads like production code |
| Strict mode | `MockBehavior.Strict` | `Substitute.For<T>()` (loose by default) |
| Community usage | Most popular | Growing, especially in new projects |

## FluentAssertions

### Readable Assertions

```csharp
using FluentAssertions;

public class ProductServiceTests
{
    [Fact]
    public void Create_ValidProduct_SetsProperties()
    {
        var product = ProductFactory.Create("Widget", 29.99m);

        // Standard assertions (harder to read)
        Assert.Equal("Widget", product.Name);
        Assert.Equal(29.99m, product.Price);
        Assert.True(product.IsActive);

        // FluentAssertions (reads like natural language)
        product.Name.Should().Be("Widget");
        product.Price.Should().Be(29.99m);
        product.IsActive.Should().BeTrue();
    }

    // Object comparison
    [Fact]
    public void Map_ProductToDto_MapsAllProperties()
    {
        var product = new Product { Id = 1, Name = "Widget", Price = 29.99m };
        var dto = Mapper.Map<ProductDto>(product);

        dto.Should().BeEquivalentTo(product, options => options
            .ExcludingMissingMembers()
            .Excluding(d => d.Id));
    }

    // Collection assertions
    [Fact]
    public async Task GetAll_ReturnsProductsInOrder()
    {
        var products = await _service.GetAllAsync();

        products.Should().NotBeEmpty()
            .And.HaveCountGreaterThan(2)
            .And.BeInAscendingOrder(p => p.Name)
            .And.OnlyContain(p => p.Price > 0);
    }

    // Exception assertions
    [Fact]
    public async Task GetById_InvalidId_ThrowsNotFoundException()
    {
        var act = async () => await _service.GetByIdAsync(-1);

        await act.Should().ThrowAsync<NotFoundException>()
            .WithMessage("*not found*")
            .Where(ex => ex.EntityId == -1);
    }

    // String assertions
    [Fact]
    public void GenerateSlug_ReturnsLowerKebabCase()
    {
        var slug = SlugGenerator.Generate("Hello World Test");

        slug.Should()
            .StartWith("hello")
            .And.EndWith("test")
            .And.NotContainAny(" ", "_")
            .And.Be("hello-world-test");
    }

    // Numeric assertions
    [Fact]
    public void CalculateDiscount_ReturnsValueWithinRange()
    {
        var discount = _calculator.CalculateDiscount(100m, 15);

        discount.Should().BeApproximately(15m, precision: 0.01m);
        discount.Should().BeInRange(0m, 100m);
    }
}
```

## Integration Testing

### WebApplicationFactory

`WebApplicationFactory<T>` creates an in-memory test server for your ASP.NET Core application, allowing real HTTP integration tests.

```csharp
public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Remove the real database registration
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (descriptor is not null)
                services.Remove(descriptor);

            // Add SQLite in-memory database for testing
            services.AddDbContext<AppDbContext>(options =>
            {
                options.UseSqlite("Data Source=:memory:");
            });

            // Ensure the database is created
            var sp = services.BuildServiceProvider();
            using var scope = sp.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            db.Database.OpenConnection();
            db.Database.EnsureCreated();
        });

        builder.UseEnvironment("Testing");
    }
}
```

### Integration Test with HttpClient

```csharp
public class ProductApiTests(CustomWebApplicationFactory factory)
    : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client = factory.CreateClient();

    [Fact]
    public async Task GetProducts_ReturnsSuccessAndList()
    {
        var response = await _client.GetAsync("/api/products");

        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var products = await response.Content.ReadFromJsonAsync<List<ProductDto>>();
        products.Should().NotBeNull();
    }

    [Fact]
    public async Task CreateProduct_ValidPayload_ReturnsCreated()
    {
        var request = new CreateProductRequest
        {
            Name = "Test Product",
            Price = 19.99m,
            Sku = "TST-001"
        };

        var response = await _client.PostAsJsonAsync("/api/products", request);

        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();

        var created = await response.Content.ReadFromJsonAsync<ProductDto>();
        created!.Name.Should().Be("Test Product");
    }

    [Fact]
    public async Task CreateProduct_InvalidPayload_ReturnsBadRequest()
    {
        var request = new CreateProductRequest { Name = "", Price = -1 };

        var response = await _client.PostAsJsonAsync("/api/products", request);

        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }
}
```

### Testcontainers

Testcontainers start real database instances in Docker for realistic integration tests.

```csharp
public class PostgresFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .WithDatabase("testdb")
        .WithUsername("test")
        .WithPassword("test")
        .Build();

    public string ConnectionString => _container.GetConnectionString();

    public async Task InitializeAsync() => await _container.StartAsync();

    public async Task DisposeAsync() => await _container.DisposeAsync();
}

public class ProductRepositoryIntegrationTests(PostgresFixture fixture)
    : IClassFixture<PostgresFixture>
{
    [Fact]
    public async Task Add_ValidProduct_PersistsToPostgres()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(fixture.ConnectionString)
            .Options;

        await using var context = new AppDbContext(options);
        await context.Database.MigrateAsync();

        context.Products.Add(new Product { Name = "Widget", Price = 5m, Sku = "WDG-001" });
        await context.SaveChangesAsync();

        var product = await context.Products.FirstAsync();
        product.Name.Should().Be("Widget");
    }
}
```

## Test Architecture

### Arrange-Act-Assert (AAA)

```csharp
[Fact]
public async Task ApplyDiscount_ValidPercentage_ReducesPrice()
{
    // Arrange — set up the test scenario
    var product = new Product { Name = "Widget", Price = 100m, Sku = "WDG-001" };
    var service = new PricingService();

    // Act — execute the behavior under test
    var discountedPrice = service.ApplyDiscount(product, 20);

    // Assert — verify the expected outcome
    discountedPrice.Should().Be(80m);
}
```

### Test Naming Conventions

| Convention | Example |
|---|---|
| `Method_Condition_ExpectedResult` | `Add_TwoPositiveNumbers_ReturnsSum` |
| `Should_ExpectedBehavior_When_Condition` | `Should_ReturnNull_When_IdNotFound` |
| `Given_When_Then` | `GivenEmptyCart_WhenAddItem_ThenCartHasOneItem` |

The `Method_Condition_ExpectedResult` convention is most common in .NET projects.

### Test Data Builders

```csharp
public class ProductBuilder
{
    private string _name = "Default Product";
    private decimal _price = 9.99m;
    private string _sku = "DEF-001";
    private int _categoryId = 1;
    private bool _isDeleted;

    public ProductBuilder WithName(string name) { _name = name; return this; }
    public ProductBuilder WithPrice(decimal price) { _price = price; return this; }
    public ProductBuilder WithSku(string sku) { _sku = sku; return this; }
    public ProductBuilder WithCategoryId(int id) { _categoryId = id; return this; }
    public ProductBuilder Deleted() { _isDeleted = true; return this; }

    public Product Build() => new()
    {
        Name = _name,
        Price = _price,
        Sku = _sku,
        CategoryId = _categoryId,
        IsDeleted = _isDeleted
    };
}

// Usage in tests
[Fact]
public void Discount_ExpensiveProduct_AppliesTierDiscount()
{
    var product = new ProductBuilder()
        .WithName("Premium Widget")
        .WithPrice(500m)
        .Build();

    var discount = _service.CalculateDiscount(product);

    discount.Should().BeGreaterThan(0);
}
```

### AutoFixture

```csharp
public class OrderServiceAutoFixtureTests
{
    private readonly IFixture _fixture = new Fixture()
        .Customize(new AutoMoqCustomization { ConfigureMembers = true });

    [Fact]
    public async Task PlaceOrder_GeneratesOrderNumber()
    {
        // AutoFixture creates an OrderService with auto-mocked dependencies
        var sut = _fixture.Create<OrderService>();
        var order = _fixture.Create<Order>();

        await sut.PlaceOrderAsync(order);

        order.OrderNumber.Should().NotBeNullOrEmpty();
    }

    // Combine with [Theory] using AutoData
    [Theory, AutoMoqData]
    public async Task GetOrder_ExistingId_ReturnsOrder(
        [Frozen] Mock<IOrderRepository> repoMock,
        OrderService sut,
        Order expected)
    {
        repoMock.Setup(r => r.GetByIdAsync(expected.Id, default)).ReturnsAsync(expected);

        var result = await sut.GetOrderAsync(expected.Id);

        result.Should().BeEquivalentTo(expected);
    }
}

// Custom AutoData attribute
public class AutoMoqDataAttribute() : AutoDataAttribute(() =>
    new Fixture().Customize(new AutoMoqCustomization { ConfigureMembers = true }));
```

## Testing ASP.NET Core

### Controller Tests

```csharp
public class ProductsControllerTests
{
    private readonly Mock<IProductService> _serviceMock = new();
    private readonly ProductsController _controller;

    public ProductsControllerTests()
    {
        _controller = new ProductsController(_serviceMock.Object);
    }

    [Fact]
    public async Task GetById_ExistingProduct_ReturnsOkWithProduct()
    {
        var product = new ProductDto { Id = 1, Name = "Widget", Price = 9.99m };
        _serviceMock.Setup(s => s.GetByIdAsync(1)).ReturnsAsync(product);

        var result = await _controller.GetById(1);

        var okResult = result.Result.Should().BeOfType<OkObjectResult>().Subject;
        var returnedProduct = okResult.Value.Should().BeOfType<ProductDto>().Subject;
        returnedProduct.Name.Should().Be("Widget");
    }

    [Fact]
    public async Task GetById_NonExistent_ReturnsNotFound()
    {
        _serviceMock.Setup(s => s.GetByIdAsync(999)).ReturnsAsync((ProductDto?)null);

        var result = await _controller.GetById(999);

        result.Result.Should().BeOfType<NotFoundResult>();
    }
}
```

### Minimal API Tests

```csharp
public class MinimalApiTests(CustomWebApplicationFactory factory)
    : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client = factory.CreateClient();

    [Fact]
    public async Task GetProduct_ReturnsOk()
    {
        // Seed data
        using var scope = factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        db.Products.Add(new Product { Name = "Test", Price = 5m, Sku = "TST-001" });
        await db.SaveChangesAsync();

        var response = await _client.GetAsync("/api/products/1");

        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }

    [Fact]
    public async Task PostProduct_InvalidBody_ReturnsValidationProblem()
    {
        var response = await _client.PostAsJsonAsync("/api/products", new { });

        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }
}
```

### Middleware Tests

```csharp
public class ExceptionMiddlewareTests
{
    [Fact]
    public async Task Invoke_WhenExceptionThrown_Returns500WithProblemDetails()
    {
        // Arrange
        var middleware = new ExceptionHandlingMiddleware(
            next: _ => throw new InvalidOperationException("Test error"),
            logger: NullLogger<ExceptionHandlingMiddleware>.Instance);

        var context = new DefaultHttpContext();
        context.Response.Body = new MemoryStream();

        // Act
        await middleware.InvokeAsync(context);

        // Assert
        context.Response.StatusCode.Should().Be(500);
        context.Response.Body.Seek(0, SeekOrigin.Begin);
        var body = await new StreamReader(context.Response.Body).ReadToEndAsync();
        body.Should().Contain("Test error");
    }
}
```

### Filter Tests

```csharp
public class ValidationFilterTests
{
    [Fact]
    public async Task OnActionExecuting_InvalidModel_ReturnsBadRequest()
    {
        var filter = new ValidationFilter();

        var actionContext = new ActionContext(
            new DefaultHttpContext(),
            new RouteData(),
            new ActionDescriptor());
        actionContext.ModelState.AddModelError("Name", "Name is required");

        var context = new ActionExecutingContext(
            actionContext,
            [],
            new Dictionary<string, object?>(),
            controller: null!);

        filter.OnActionExecuting(context);

        context.Result.Should().BeOfType<BadRequestObjectResult>();
    }
}
```

## Testing EF Core

### In-Memory Provider vs SQLite

| Feature | In-Memory Provider | SQLite (In-Memory) |
|---|---|---|
| Relational behavior | ❌ No (not a relational DB) | ✅ Yes |
| Foreign key enforcement | ❌ No | ✅ Yes |
| Transaction support | ❌ No | ✅ Yes |
| Migration support | ❌ No | ✅ Yes |
| LINQ translation fidelity | Low | High |
| Speed | Fastest | Fast |
| Recommended | ❌ Avoid for new tests | ✅ Preferred |

### SQLite In-Memory Testing

```csharp
public class TestDbContextFactory
{
    public static AppDbContext Create()
    {
        var connection = new SqliteConnection("Data Source=:memory:");
        connection.Open();

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlite(connection)
            .Options;

        var context = new AppDbContext(options);
        context.Database.EnsureCreated();

        return context;
    }
}

public class ProductRepositoryTests : IDisposable
{
    private readonly AppDbContext _context;

    public ProductRepositoryTests()
    {
        _context = TestDbContextFactory.Create();
    }

    [Fact]
    public async Task Add_ValidProduct_PersistsToDatabase()
    {
        var product = new Product { Name = "Widget", Price = 9.99m, Sku = "WDG-001" };
        _context.Products.Add(product);
        await _context.SaveChangesAsync();

        var saved = await _context.Products.FirstAsync();
        saved.Name.Should().Be("Widget");
        saved.Id.Should().BeGreaterThan(0);
    }

    [Fact]
    public async Task Delete_ExistingProduct_RemovesFromDatabase()
    {
        var product = new Product { Name = "Widget", Price = 9.99m, Sku = "WDG-001" };
        _context.Products.Add(product);
        await _context.SaveChangesAsync();

        _context.Products.Remove(product);
        await _context.SaveChangesAsync();

        var count = await _context.Products.CountAsync();
        count.Should().Be(0);
    }

    public void Dispose() => _context.Dispose();
}
```

### Test Database Strategies

```
┌──────────────────────────────────────────────────────────────────┐
│                Test Database Strategies                           │
│                                                                  │
│  ┌──────────────────┐  Best for: unit tests, fast feedback      │
│  │ SQLite In-Memory │  Limitations: some SQL Server features     │
│  └──────────────────┘  missing (JSON columns, specific types)    │
│                                                                  │
│  ┌──────────────────┐  Best for: CI/CD, realistic integration   │
│  │  Testcontainers  │  Runs real PostgreSQL/SQL Server in Docker│
│  └──────────────────┘  Slow startup, highest fidelity            │
│                                                                  │
│  ┌──────────────────┐  Best for: shared dev/test environments   │
│  │  Dedicated Test  │  Requires infrastructure management       │
│  │    Database      │  Risk of test pollution                    │
│  └──────────────────┘                                            │
│                                                                  │
│  ┌──────────────────┐  Best for: nothing (avoid)                │
│  │ EF In-Memory     │  Not relational, masks real issues        │
│  │   Provider       │  May pass tests that fail in production   │
│  └──────────────────┘                                            │
└──────────────────────────────────────────────────────────────────┘
```

## Code Coverage

### Tools

| Tool | Type | Integration |
|---|---|---|
| Coverlet | Open source, cross-platform | CLI, MSBuild, CI/CD |
| dotCover | JetBrains commercial | Rider, TeamCity |
| Visual Studio | Built-in (Enterprise) | Visual Studio only |

### Coverlet Setup and Usage

```bash
# Install coverlet as a global tool
dotnet tool install --global coverlet.console

# Run tests with coverage collection
dotnet test --collect:"XPlat Code Coverage"

# Generate HTML report
dotnet tool install --global dotnet-reportgenerator-globaltool
reportgenerator -reports:"**/coverage.cobertura.xml" -targetdir:"coveragereport" -reporttypes:Html

# MSBuild integration — add to test project
# <PackageReference Include="coverlet.msbuild" />
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=cobertura /p:Threshold=80
```

### Meaningful Coverage

Coverage is a metric, not a goal. Focus on testing critical business logic, not chasing a percentage.

```
┌─────────────────────────────────────────────────────────────────┐
│              Coverage Target Guidelines                          │
│                                                                 │
│  100%  ┤ ❌ Diminishing returns, brittle tests                  │
│   90%  ┤                                                        │
│   80%  ┤ ✅ Good target for business logic layers               │
│   70%  ┤ ✅ Reasonable for overall project                      │
│   60%  ┤                                                        │
│   50%  ┤ ⚠️  Minimum acceptable for mature projects             │
│   40%  ┤                                                        │
│        └────────────────────────────────────────────            │
│                                                                 │
│  Focus coverage on:                                             │
│  • Business logic and domain services                           │
│  • Edge cases and error handling                                │
│  • Public API contracts                                         │
│                                                                 │
│  Don't chase coverage on:                                       │
│  • DTOs, models, and trivial properties                         │
│  • Generated code and third-party wrappers                      │
│  • Framework plumbing (Program.cs)                              │
└─────────────────────────────────────────────────────────────────┘
```

### Mutation Testing

Mutation testing modifies your code and checks whether tests detect the change. If a mutation survives, the test suite has a gap.

```bash
# Install Stryker.NET
dotnet tool install --global dotnet-stryker

# Run mutation testing
dotnet stryker --project MyApp.csproj --test-project MyApp.Tests.csproj
```

## Best Practices

### Production Checklist

- ✅ Use xUnit for new .NET test projects — it is the ecosystem standard
- ✅ Follow Arrange-Act-Assert in every test method
- ✅ Use `Method_Condition_ExpectedResult` naming for test methods
- ✅ Use FluentAssertions for readable and expressive assertions
- ✅ Use `IClassFixture<T>` for expensive shared setup (databases, containers)
- ✅ Use `IAsyncLifetime` for async setup and teardown
- ✅ Use `WebApplicationFactory<T>` for ASP.NET Core integration tests
- ✅ Use SQLite in-memory for EF Core unit tests — not the In-Memory provider
- ✅ Use Testcontainers for realistic database integration tests in CI
- ✅ Use test data builders for complex object construction
- ✅ Run tests in parallel by default (xUnit does this per class)
- ✅ Set a code coverage threshold (70–80%) in CI/CD pipelines
- ✅ Keep unit tests fast — under 100ms per test
- ✅ Mock external dependencies (HTTP, database, file system) in unit tests

- ❌ Don't test implementation details — test behavior and outcomes
- ❌ Don't use the EF Core In-Memory provider — it hides relational issues
- ❌ Don't share mutable state between tests without proper isolation
- ❌ Don't assert more than one logical concept per test
- ❌ Don't mock types you don't own — wrap them and mock the wrapper
- ❌ Don't write tests that depend on execution order
- ❌ Don't ignore flaky tests — fix the root cause or quarantine them
- ❌ Don't target 100% code coverage — focus on meaningful coverage of critical paths

## Next Steps

Continue to [Security and Authentication](09-SECURITY-AND-AUTHENTICATION.md) to learn how to secure .NET applications with authentication, authorization, JWT tokens, and security best practices.

### Related Documents

1. **[Learning Path](LEARNING-PATH.md)** — Complete .NET learning curriculum
2. **[Entity Framework Core](07-ENTITY-FRAMEWORK-CORE.md)** — Data access, querying, migrations
3. **[Services](02-SERVICES.md)** — Web APIs, worker services, gRPC
4. **[Best Practices](03-BEST-PRACTICES.md)** — Async/await, memory, security, testing

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial testing in .NET documentation |
