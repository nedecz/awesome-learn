# Middleware and HTTP Pipeline in ASP.NET Core

## Table of Contents

1. [Overview](#overview)
2. [Request Pipeline Architecture](#request-pipeline-architecture)
3. [Built-in Middleware](#built-in-middleware)
4. [Custom Middleware](#custom-middleware)
5. [Error Handling Middleware](#error-handling-middleware)
6. [Request/Response Manipulation](#requestresponse-manipulation)
7. [Filters in ASP.NET Core](#filters-in-aspnet-core)
8. [Minimal APIs Pipeline](#minimal-apis-pipeline)
9. [Rate Limiting](#rate-limiting)
10. [Output Caching](#output-caching)
11. [Best Practices](#best-practices)
12. [Next Steps](#next-steps)

## Overview

The ASP.NET Core HTTP pipeline is a chain of middleware components that process every HTTP request and response. Each middleware component can inspect, modify, short-circuit, or pass a request to the next component. Understanding this pipeline is essential for building performant, secure, and maintainable web applications.

### Target Audience

- ASP.NET Core developers building web APIs and web applications
- Architects designing cross-cutting concerns (logging, auth, caching, rate limiting)
- Teams migrating from controller-centric pipelines to Minimal APIs with endpoint filters

### Scope

- Request/response pipeline architecture and middleware ordering
- Built-in middleware components and their correct registration order
- Convention-based and factory-based custom middleware
- Global error handling with ProblemDetails
- ASP.NET Core filters (action, authorization, resource, exception, result)
- Minimal API endpoint filters, route groups, and request delegates
- Rate limiting (fixed window, sliding window, token bucket, concurrency)
- Output caching with cache policies and invalidation (.NET 7+)
- Production best practices, ordering checklists, and performance tips

## Request Pipeline Architecture

Every HTTP request enters the ASP.NET Core pipeline through the server (Kestrel) and flows through a series of middleware components before reaching an endpoint. The response flows back through the same middleware in reverse order.

### Pipeline Flow

```
  HTTP Request
       │
       ▼
┌─────────────┐
│   Kestrel   │  (HTTP server)
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│                    Middleware Pipeline                       │
│                                                             │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐       │
│  │ Middleware 1 │──▶│ Middleware 2 │──▶│ Middleware 3 │──▶ …  │
│  │ (Exception  │   │ (Auth)      │   │ (Routing)   │       │
│  │  Handler)   │   │             │   │             │       │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘       │
│         │◀─────────────────│◀─────────────────│◀────── …    │
│       response           response           response        │
│       (reverse)          (reverse)          (reverse)        │
└─────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────┐
│  Endpoint   │  (Controller action / Minimal API handler)
└─────────────┘
```

### Middleware vs Filters

Middleware operates on every request across the entire application. Filters operate within the MVC/Minimal API endpoint execution model.

| Aspect | Middleware | Filters |
|---|---|---|
| Scope | All requests | Only requests routed to endpoints |
| Access to | `HttpContext` | `HttpContext` + endpoint metadata, model binding, action arguments |
| Ordering | Registration order in `Program.cs` | Attribute order, global filters, filter pipelines |
| Short-circuit | Call `next()` or return early | Set `context.Result` or throw |
| Use cases | Logging, CORS, compression, auth, rate limiting | Validation, authorization policies, caching, exception mapping |

### Request Delegate Chain

Each middleware is a `RequestDelegate` that wraps the next delegate in the pipeline:

```csharp
// Conceptual model — each middleware wraps the next
RequestDelegate pipeline = context =>
{
    // Middleware 3 (innermost)
    return Task.CompletedTask;
};

pipeline = next => async context =>
{
    // Middleware 2 — before
    await next(context);
    // Middleware 2 — after
};

pipeline = next => async context =>
{
    // Middleware 1 — before
    await next(context);
    // Middleware 1 — after
};
```

## Built-in Middleware

ASP.NET Core ships with many built-in middleware components. Their registration order in `Program.cs` determines execution order — and getting this wrong is a common source of bugs.

### Recommended Ordering

```csharp
var builder = WebApplication.CreateBuilder(args);

// --- Service registration ---
builder.Services.AddAuthentication();
builder.Services.AddAuthorization();
builder.Services.AddCors();
builder.Services.AddResponseCompression();
builder.Services.AddOutputCache();
builder.Services.AddRateLimiter(options => { /* ... */ });
builder.Services.AddControllers();

var app = builder.Build();

// --- Middleware pipeline (ORDER MATTERS) ---

// 1. Exception handling — outermost so it catches everything
app.UseExceptionHandler("/error");

// 2. HSTS — redirect HTTP to HTTPS
app.UseHsts();

// 3. HTTPS redirection
app.UseHttpsRedirection();

// 4. Static files — serve before routing to short-circuit early
app.UseStaticFiles();

// 5. Response compression
app.UseResponseCompression();

// 6. Routing — matches endpoints but does not execute them
app.UseRouting();

// 7. CORS — must be between UseRouting and UseAuthorization
app.UseCors();

// 8. Rate limiting
app.UseRateLimiter();

// 9. Authentication — identify the user
app.UseAuthentication();

// 10. Authorization — check permissions
app.UseAuthorization();

// 11. Output caching — after auth so cached responses respect policies
app.UseOutputCache();

// 12. Endpoints — terminal middleware
app.MapControllers();

app.Run();
```

### Ordering Rules

```
┌──────────────────────────────────────────────────────────────┐
│               MIDDLEWARE ORDERING RULES                       │
│                                                              │
│  UseExceptionHandler    ← FIRST (catches all exceptions)     │
│  UseHsts                                                     │
│  UseHttpsRedirection                                         │
│  UseStaticFiles         ← BEFORE routing (short-circuit)     │
│  UseResponseCompression                                      │
│  UseRouting             ← AFTER static files                 │
│  UseCors                ← BETWEEN routing and auth           │
│  UseRateLimiter         ← AFTER routing, BEFORE auth         │
│  UseAuthentication      ← BEFORE authorization               │
│  UseAuthorization       ← AFTER authentication               │
│  UseOutputCache         ← AFTER authorization                │
│  MapControllers / MapGet / MapPost  ← TERMINAL               │
└──────────────────────────────────────────────────────────────┘
```

### Built-in Middleware Reference

| Middleware | Purpose | NuGet Package |
|---|---|---|
| `UseExceptionHandler` | Global exception handling, ProblemDetails | Built-in |
| `UseHsts` | HTTP Strict Transport Security header | Built-in |
| `UseHttpsRedirection` | Redirect HTTP → HTTPS | Built-in |
| `UseStaticFiles` | Serve static files (wwwroot) | Built-in |
| `UseRouting` | Match URL to endpoint | Built-in |
| `UseCors` | Cross-Origin Resource Sharing | Built-in |
| `UseAuthentication` | Authenticate the request | Built-in |
| `UseAuthorization` | Authorize the request | Built-in |
| `UseResponseCompression` | Gzip/Brotli compression | `Microsoft.AspNetCore.ResponseCompression` |
| `UseRateLimiter` | Rate limiting (.NET 7+) | Built-in |
| `UseOutputCache` | Output caching (.NET 7+) | Built-in |
| `UseResponseCaching` | HTTP response caching | Built-in |
| `UseRequestLocalization` | Request culture/localization | Built-in |

## Custom Middleware

There are two patterns for writing custom middleware: convention-based and factory-based (`IMiddleware`).

### Convention-Based Middleware

The convention-based pattern uses a class with an `InvokeAsync` method and a `RequestDelegate` constructor parameter. The class is instantiated once (singleton lifetime) by the framework.

```csharp
public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTimingMiddleware> _logger;

    public RequestTimingMiddleware(
        RequestDelegate next,
        ILogger<RequestTimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();

        // Add timing header before passing to next middleware
        context.Response.OnStarting(() =>
        {
            context.Response.Headers["X-Response-Time"] =
                $"{stopwatch.ElapsedMilliseconds}ms";
            return Task.CompletedTask;
        });

        await _next(context);

        stopwatch.Stop();
        _logger.LogInformation(
            "{Method} {Path} completed in {ElapsedMs}ms",
            context.Request.Method,
            context.Request.Path,
            stopwatch.ElapsedMilliseconds);
    }
}

// Registration
app.UseMiddleware<RequestTimingMiddleware>();
```

### Factory-Based Middleware (IMiddleware)

The `IMiddleware` interface creates a new instance per request, enabling scoped dependencies. You must register the middleware in DI.

```csharp
public class CorrelationIdMiddleware : IMiddleware
{
    private readonly ILogger<CorrelationIdMiddleware> _logger;

    public CorrelationIdMiddleware(ILogger<CorrelationIdMiddleware> logger)
    {
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var correlationId = context.Request.Headers["X-Correlation-Id"]
            .FirstOrDefault() ?? Guid.NewGuid().ToString();

        context.Items["CorrelationId"] = correlationId;
        context.Response.Headers["X-Correlation-Id"] = correlationId;

        using (_logger.BeginScope(
            new Dictionary<string, object> { ["CorrelationId"] = correlationId }))
        {
            _logger.LogInformation("Request started — CorrelationId: {Id}", correlationId);
            await next(context);
        }
    }
}

// Registration — must register in DI for IMiddleware
builder.Services.AddTransient<CorrelationIdMiddleware>();
app.UseMiddleware<CorrelationIdMiddleware>();
```

### Convention-Based vs Factory-Based

| Aspect | Convention-Based | Factory-Based (IMiddleware) |
|---|---|---|
| Lifetime | Singleton (one instance) | Per-request (transient) |
| DI registration | Not required | Required (`AddTransient` / `AddScoped`) |
| Constructor DI | Singleton services only | Any lifetime (scoped services allowed) |
| Method DI | Additional params in `InvokeAsync` | No method injection |
| Performance | Slightly faster (no allocation) | Slight overhead per request |
| Use when | Stateless, no scoped dependencies | Need scoped services (e.g., DbContext) |

### Inline Middleware with Use / Map / Run

```csharp
// Use — calls next middleware
app.Use(async (context, next) =>
{
    context.Response.Headers["X-Powered-By"] = "awesome-learn";
    await next(context);
});

// Map — branches the pipeline by path prefix
app.Map("/health", healthApp =>
{
    healthApp.Run(async context =>
    {
        await context.Response.WriteAsync("OK");
    });
});

// Run — terminal middleware (does not call next)
app.Run(async context =>
{
    await context.Response.WriteAsync("Hello, World!");
});
```

## Error Handling Middleware

Proper error handling is critical for security and observability. ASP.NET Core provides built-in middleware for global exception handling.

### Global Exception Handler with ProblemDetails

```csharp
var builder = WebApplication.CreateBuilder(args);

// Register ProblemDetails services
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = ctx =>
    {
        ctx.ProblemDetails.Extensions["traceId"] =
            ctx.HttpContext.TraceIdentifier;
        ctx.ProblemDetails.Extensions["instance"] =
            ctx.HttpContext.Request.Path;
    };
});

var app = builder.Build();

// Global exception handler — returns ProblemDetails JSON
app.UseExceptionHandler();

// Status code pages for non-exception errors (404, 403, etc.)
app.UseStatusCodePages();

app.MapGet("/", () => "Hello, World!");

app.Run();
```

### Custom Exception Handler

```csharp
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        context.Response.ContentType = "application/problem+json";

        var exception = context.Features
            .Get<IExceptionHandlerFeature>()?.Error;

        var (statusCode, title) = exception switch
        {
            ArgumentException => (StatusCodes.Status400BadRequest,
                "Invalid argument"),
            KeyNotFoundException => (StatusCodes.Status404NotFound,
                "Resource not found"),
            UnauthorizedAccessException => (StatusCodes.Status403Forbidden,
                "Access denied"),
            _ => (StatusCodes.Status500InternalServerError,
                "An unexpected error occurred")
        };

        context.Response.StatusCode = statusCode;

        var problem = new ProblemDetails
        {
            Status = statusCode,
            Title = title,
            Detail = app.Environment.IsDevelopment()
                ? exception?.Message : null,
            Instance = context.Request.Path
        };

        await context.Response.WriteAsJsonAsync(problem);
    });
});
```

### IExceptionHandler (.NET 8+)

.NET 8 introduced `IExceptionHandler` for structured, DI-friendly exception handling:

```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken cancellationToken)
    {
        _logger.LogError(exception,
            "Unhandled exception for {Method} {Path}",
            context.Request.Method, context.Request.Path);

        var problem = new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Title = "Server Error",
            Type = "https://tools.ietf.org/html/rfc9110#section-15.6.1"
        };

        context.Response.StatusCode = problem.Status.Value;
        await context.Response.WriteAsJsonAsync(problem, cancellationToken);

        return true; // Exception is handled
    }
}

// Registration
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

var app = builder.Build();
app.UseExceptionHandler();
```

### Developer Exception Page

```csharp
if (app.Environment.IsDevelopment())
{
    // Shows detailed exception info — NEVER enable in production
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler("/error");
    app.UseHsts();
}
```

## Request/Response Manipulation

Middleware can read, buffer, and transform both requests and responses.

### Reading Request Bodies

By default, the request body stream is forward-only. Enable buffering to read it multiple times:

```csharp
app.Use(async (context, next) =>
{
    // Enable buffering so the body can be read multiple times
    context.Request.EnableBuffering();

    using var reader = new StreamReader(
        context.Request.Body,
        leaveOpen: true);

    var body = await reader.ReadToEndAsync();

    // Reset the stream position for downstream middleware
    context.Request.Body.Position = 0;

    if (!string.IsNullOrEmpty(body))
    {
        // Log or inspect the body
        var logger = context.RequestServices
            .GetRequiredService<ILogger<Program>>();
        logger.LogDebug("Request body: {Body}", body);
    }

    await next(context);
});
```

### Response Buffering and Manipulation

```csharp
public class ResponseWrappingMiddleware
{
    private readonly RequestDelegate _next;

    public ResponseWrappingMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        var originalBody = context.Response.Body;

        using var memoryStream = new MemoryStream();
        context.Response.Body = memoryStream;

        await _next(context);

        memoryStream.Position = 0;
        var responseBody = await new StreamReader(memoryStream)
            .ReadToEndAsync();

        // Manipulate the response if needed
        memoryStream.Position = 0;
        await memoryStream.CopyToAsync(originalBody);
        context.Response.Body = originalBody;
    }
}
```

### Response Compression

```csharp
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();
    options.MimeTypes = ResponseCompressionDefaults
        .MimeTypes.Concat(["application/json"]);
});

builder.Services.Configure<BrotliCompressionProviderOptions>(options =>
{
    options.Level = CompressionLevel.Fastest;
});

var app = builder.Build();
app.UseResponseCompression();
```

### URL Rewriting

```csharp
var rewriteOptions = new RewriteOptions()
    .AddRedirect("old-path/(.*)", "new-path/$1", 301)
    .AddRewrite(@"^api/v1/(.*)", "api/v2/$1", skipRemainingRules: true);

app.UseRewriter(rewriteOptions);
```

## Filters in ASP.NET Core

Filters run within the ASP.NET Core endpoint invocation pipeline — after routing but around the action/endpoint execution. They provide hooks at specific stages of request processing.

### Filter Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│                   FILTER PIPELINE                            │
│                                                             │
│  Request                                                    │
│    │                                                        │
│    ▼                                                        │
│  ┌───────────────────────┐                                  │
│  │  Authorization Filter │  ← Can short-circuit (403)       │
│  └──────────┬────────────┘                                  │
│             ▼                                               │
│  ┌───────────────────────┐                                  │
│  │   Resource Filter     │  ← Before/after model binding    │
│  │   (OnResourceExecuting)                                  │
│  └──────────┬────────────┘                                  │
│             ▼                                               │
│  ┌───────────────────────┐                                  │
│  │   Model Binding       │                                  │
│  └──────────┬────────────┘                                  │
│             ▼                                               │
│  ┌───────────────────────┐                                  │
│  │   Action Filter       │  ← Before/after action method    │
│  │   (OnActionExecuting) │                                  │
│  └──────────┬────────────┘                                  │
│             ▼                                               │
│  ┌───────────────────────┐                                  │
│  │   ACTION METHOD       │  ← Your code runs here          │
│  └──────────┬────────────┘                                  │
│             ▼                                               │
│  ┌───────────────────────┐                                  │
│  │   Action Filter       │                                  │
│  │   (OnActionExecuted)  │                                  │
│  └──────────┬────────────┘                                  │
│             ▼                                               │
│  ┌───────────────────────┐                                  │
│  │   Exception Filter    │  ← Catches unhandled exceptions  │
│  └──────────┬────────────┘                                  │
│             ▼                                               │
│  ┌───────────────────────┐                                  │
│  │   Result Filter       │  ← Before/after result execution │
│  │   (OnResultExecuting) │                                  │
│  └──────────┬────────────┘                                  │
│             ▼                                               │
│  ┌───────────────────────┐                                  │
│  │   Resource Filter     │                                  │
│  │   (OnResourceExecuted)│                                  │
│  └───────────────────────┘                                  │
│    │                                                        │
│    ▼                                                        │
│  Response                                                   │
└─────────────────────────────────────────────────────────────┘
```

### Authorization Filter

```csharp
public class ApiKeyAuthorizationFilter : IAuthorizationFilter
{
    private const string ApiKeyHeader = "X-Api-Key";

    public void OnAuthorization(AuthorizationFilterContext context)
    {
        if (!context.HttpContext.Request.Headers
            .TryGetValue(ApiKeyHeader, out var apiKey)
            || apiKey != "expected-key")
        {
            context.Result = new UnauthorizedResult();
        }
    }
}

// Usage on a controller or action
[ServiceFilter(typeof(ApiKeyAuthorizationFilter))]
public class SecureController : ControllerBase { }
```

### Action Filter

```csharp
public class ValidateModelAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        if (!context.ModelState.IsValid)
        {
            context.Result = new BadRequestObjectResult(
                new ValidationProblemDetails(context.ModelState));
        }
    }
}

// Async action filter
public class AuditLogFilter : IAsyncActionFilter
{
    private readonly ILogger<AuditLogFilter> _logger;

    public AuditLogFilter(ILogger<AuditLogFilter> logger) => _logger = logger;

    public async Task OnActionExecutionAsync(
        ActionExecutingContext context,
        ActionExecutionDelegate next)
    {
        _logger.LogInformation("Executing {Action} with {Args}",
            context.ActionDescriptor.DisplayName,
            context.ActionArguments);

        var result = await next();

        if (result.Exception is not null)
        {
            _logger.LogError(result.Exception,
                "Action {Action} threw an exception",
                context.ActionDescriptor.DisplayName);
        }
    }
}
```

### Exception Filter

```csharp
public class DomainExceptionFilter : IExceptionFilter
{
    public void OnException(ExceptionContext context)
    {
        if (context.Exception is DomainException domainEx)
        {
            context.Result = new ObjectResult(new ProblemDetails
            {
                Status = StatusCodes.Status422UnprocessableEntity,
                Title = "Domain Error",
                Detail = domainEx.Message
            })
            {
                StatusCode = StatusCodes.Status422UnprocessableEntity
            };

            context.ExceptionHandled = true;
        }
    }
}

// Register globally
builder.Services.AddControllers(options =>
{
    options.Filters.Add<DomainExceptionFilter>();
});
```

### Result Filter

```csharp
public class AddPaginationHeadersFilter : IAsyncResultFilter
{
    public async Task OnResultExecutionAsync(
        ResultExecutingContext context,
        ResultExecutionDelegate next)
    {
        if (context.Result is ObjectResult { Value: IPagedResult paged })
        {
            context.HttpContext.Response.Headers["X-Total-Count"] =
                paged.TotalCount.ToString();
            context.HttpContext.Response.Headers["X-Page-Size"] =
                paged.PageSize.ToString();
        }

        await next();
    }
}
```

### Filter Ordering

Filters execute by order value (lower runs first) and scope (Global → Controller → Action):

```csharp
// Global filter with explicit order
builder.Services.AddControllers(options =>
{
    options.Filters.Add<AuditLogFilter>(order: 1);
    options.Filters.Add<ValidateModelAttribute>(order: 2);
});

// Controller-level filter
[ServiceFilter(typeof(AuditLogFilter), Order = 0)]
public class OrdersController : ControllerBase { }

// Action-level filter
[ValidateModel(Order = -1)]  // Runs before global filters
public IActionResult CreateOrder([FromBody] OrderRequest request) { }
```

## Minimal APIs Pipeline

Minimal APIs use endpoint filters instead of MVC filters, and route groups for organizing endpoints.

### Endpoint Filters

```csharp
var app = WebApplication.CreateBuilder(args).Build();

app.MapPost("/api/orders", (OrderRequest request) =>
    {
        return Results.Created($"/api/orders/{request.Id}", request);
    })
    .AddEndpointFilter(async (context, next) =>
    {
        var request = context.GetArgument<OrderRequest>(0);

        if (string.IsNullOrEmpty(request.CustomerId))
        {
            return Results.ValidationProblem(
                new Dictionary<string, string[]>
                {
                    ["CustomerId"] = ["Customer ID is required"]
                });
        }

        return await next(context);
    });

app.Run();
```

### Typed Endpoint Filters

```csharp
public class ValidationFilter<T> : IEndpointFilter where T : class
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var argument = context.GetArgument<T>(0);

        var validator = context.HttpContext.RequestServices
            .GetService<IValidator<T>>();

        if (validator is not null)
        {
            var result = await validator.ValidateAsync(argument);

            if (!result.IsValid)
            {
                return Results.ValidationProblem(
                    result.ToDictionary());
            }
        }

        return await next(context);
    }
}

// Usage
app.MapPost("/api/orders", (OrderRequest request) =>
        Results.Created($"/api/orders/1", request))
    .AddEndpointFilter<ValidationFilter<OrderRequest>>();
```

### Route Groups with MapGroup

```csharp
var app = WebApplication.CreateBuilder(args).Build();

var api = app.MapGroup("/api")
    .AddEndpointFilter(async (ctx, next) =>
    {
        // Logging filter applied to all /api endpoints
        var logger = ctx.HttpContext.RequestServices
            .GetRequiredService<ILogger<Program>>();
        logger.LogInformation("API request: {Path}",
            ctx.HttpContext.Request.Path);
        return await next(ctx);
    });

var orders = api.MapGroup("/orders")
    .RequireAuthorization();

orders.MapGet("/", async (IOrderService service) =>
    Results.Ok(await service.GetAllAsync()));

orders.MapGet("/{id:int}", async (int id, IOrderService service) =>
    await service.GetByIdAsync(id) is { } order
        ? Results.Ok(order)
        : Results.NotFound());

orders.MapPost("/", async (OrderRequest request, IOrderService service) =>
{
    var order = await service.CreateAsync(request);
    return Results.Created($"/api/orders/{order.Id}", order);
})
.AddEndpointFilter<ValidationFilter<OrderRequest>>();

orders.MapDelete("/{id:int}", async (int id, IOrderService service) =>
{
    await service.DeleteAsync(id);
    return Results.NoContent();
})
.RequireAuthorization("AdminPolicy");

app.Run();
```

### Request Delegates

```csharp
// Inline delegate
app.MapGet("/hello", () => "Hello, World!");

// Method group
app.MapGet("/orders/{id}", GetOrderById);

static async Task<IResult> GetOrderById(
    int id,
    IOrderService service,
    CancellationToken ct)
{
    var order = await service.GetByIdAsync(id, ct);
    return order is not null ? Results.Ok(order) : Results.NotFound();
}

// TypedResults for OpenAPI metadata
app.MapGet("/products/{id}", async Task<Results<Ok<Product>, NotFound>>
    (int id, IProductService service) =>
{
    var product = await service.GetByIdAsync(id);
    return product is not null
        ? TypedResults.Ok(product)
        : TypedResults.NotFound();
});
```

## Rate Limiting

.NET 7+ includes built-in rate limiting middleware supporting four algorithms. Rate limiting protects APIs from abuse and ensures fair resource distribution.

### Rate Limiting Algorithms

```
┌────────────────────────────────────────────────────────────┐
│               RATE LIMITING ALGORITHMS                      │
│                                                            │
│  Fixed Window        Sliding Window                        │
│  ┌──────┬──────┐     ┌──┬──┬──┬──┬──┐                     │
│  │ 10/m │ 10/m │     │  │  │  │  │  │  overlapping         │
│  │      │      │     │  segments  │  │  windows             │
│  └──────┴──────┘     └──┴──┴──┴──┴──┘                     │
│  Resets at boundary  Smooths burst traffic                 │
│                                                            │
│  Token Bucket        Concurrency                           │
│  ┌──────────┐        ┌──────────┐                          │
│  │ ● ● ● ●  │ refill │ [1] [2]  │ max concurrent          │
│  │ tokens    │ rate   │ [3] [ ]  │ requests                │
│  └──────────┘        └──────────┘                          │
│  Allows bursts       Limits parallelism                    │
└────────────────────────────────────────────────────────────┘
```

### Fixed Window Limiter

Permits a fixed number of requests per time window. Resets at the window boundary.

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", config =>
    {
        config.PermitLimit = 100;
        config.Window = TimeSpan.FromMinutes(1);
        config.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        config.QueueLimit = 10;
    });

    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

var app = builder.Build();
app.UseRateLimiter();

app.MapGet("/api/data", () => Results.Ok("data"))
    .RequireRateLimiting("fixed");
```

### Sliding Window Limiter

Divides the window into segments for smoother rate enforcement.

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddSlidingWindowLimiter("sliding", config =>
    {
        config.PermitLimit = 100;
        config.Window = TimeSpan.FromMinutes(1);
        config.SegmentsPerWindow = 6;  // 10-second segments
        config.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        config.QueueLimit = 5;
    });
});
```

### Token Bucket Limiter

Allows bursts up to the bucket size, then limits to the replenishment rate.

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddTokenBucketLimiter("token", config =>
    {
        config.TokenLimit = 50;
        config.ReplenishmentPeriod = TimeSpan.FromSeconds(10);
        config.TokensPerPeriod = 10;
        config.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        config.QueueLimit = 5;
        config.AutoReplenishment = true;
    });
});
```

### Concurrency Limiter

Limits the number of concurrent requests rather than requests per time window.

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddConcurrencyLimiter("concurrent", config =>
    {
        config.PermitLimit = 20;
        config.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        config.QueueLimit = 10;
    });
});
```

### Per-User Rate Limiting

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    options.AddPolicy("per-user", context =>
    {
        var userId = context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value
            ?? context.Connection.RemoteIpAddress?.ToString()
            ?? "anonymous";

        return RateLimitPartition.GetFixedWindowLimiter(userId,
            _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 60,
                Window = TimeSpan.FromMinutes(1)
            });
    });

    options.OnRejected = async (context, ct) =>
    {
        context.HttpContext.Response.ContentType = "application/problem+json";

        var problem = new ProblemDetails
        {
            Status = StatusCodes.Status429TooManyRequests,
            Title = "Too Many Requests",
            Detail = "Rate limit exceeded. Try again later."
        };

        if (context.Lease.TryGetMetadata(
            MetadataName.RetryAfter, out var retryAfter))
        {
            context.HttpContext.Response.Headers.RetryAfter =
                ((int)retryAfter.TotalSeconds).ToString();
            problem.Detail +=
                $" Retry after {(int)retryAfter.TotalSeconds} seconds.";
        }

        await context.HttpContext.Response.WriteAsJsonAsync(problem, ct);
    };
});
```

## Output Caching

Output caching (.NET 7+) caches entire HTTP responses on the server, avoiding endpoint re-execution. Unlike response caching, output caching does not depend on HTTP cache headers.

### Basic Output Cache

```csharp
builder.Services.AddOutputCache(options =>
{
    // Default policy — cache for 60 seconds
    options.AddBasePolicy(builder => builder.Expire(TimeSpan.FromSeconds(60)));

    // Named policy
    options.AddPolicy("CacheLong", builder =>
        builder.Expire(TimeSpan.FromMinutes(10)));

    // Vary by query string
    options.AddPolicy("VaryByQuery", builder =>
        builder.SetVaryByQuery("page", "pageSize")
               .Expire(TimeSpan.FromMinutes(5)));

    // Vary by header
    options.AddPolicy("VaryByLocale", builder =>
        builder.SetVaryByHeader("Accept-Language")
               .Expire(TimeSpan.FromMinutes(15)));
});

var app = builder.Build();
app.UseOutputCache();

app.MapGet("/api/products", async (IProductService service) =>
        Results.Ok(await service.GetAllAsync()))
    .CacheOutput("CacheLong");

app.MapGet("/api/products/{id}", async (int id, IProductService service) =>
        Results.Ok(await service.GetByIdAsync(id)))
    .CacheOutput(policy => policy.Expire(TimeSpan.FromMinutes(2)));
```

### Cache Tag Invalidation

```csharp
builder.Services.AddOutputCache(options =>
{
    options.AddPolicy("ProductCache", builder =>
        builder.Tag("products")
               .Expire(TimeSpan.FromMinutes(30)));
});

// Endpoints
app.MapGet("/api/products", async (IProductService service) =>
        Results.Ok(await service.GetAllAsync()))
    .CacheOutput("ProductCache");

app.MapPost("/api/products",
    async (ProductRequest req, IProductService service,
           IOutputCacheStore cache, CancellationToken ct) =>
    {
        var product = await service.CreateAsync(req);

        // Invalidate all entries tagged "products"
        await cache.EvictByTagAsync("products", ct);

        return Results.Created($"/api/products/{product.Id}", product);
    });
```

### Custom Output Cache Policy

```csharp
public class AuthenticatedCachePolicy : IOutputCachePolicy
{
    public ValueTask CacheRequestAsync(
        OutputCacheContext context, CancellationToken ct)
    {
        // Only cache for authenticated users
        context.EnableOutputCaching = context.HttpContext.User.Identity?.IsAuthenticated == true;
        context.AllowCacheLookup = context.EnableOutputCaching;
        context.AllowCacheStorage = context.EnableOutputCaching;
        context.AllowLocking = true;

        return ValueTask.CompletedTask;
    }

    public ValueTask ServeFromCacheAsync(
        OutputCacheContext context, CancellationToken ct)
    {
        return ValueTask.CompletedTask;
    }

    public ValueTask ServeResponseAsync(
        OutputCacheContext context, CancellationToken ct)
    {
        context.Tags.Add("authenticated");
        return ValueTask.CompletedTask;
    }
}

// Registration
builder.Services.AddOutputCache(options =>
{
    options.AddPolicy("AuthCache", new AuthenticatedCachePolicy());
});
```

## Best Practices

### Middleware Ordering Checklist

- ✅ Place `UseExceptionHandler` first so it catches all pipeline exceptions
- ✅ Place `UseStaticFiles` before `UseRouting` to short-circuit for static assets
- ✅ Place `UseCors` between `UseRouting` and `UseAuthorization`
- ✅ Place `UseAuthentication` immediately before `UseAuthorization`
- ✅ Place `UseRateLimiter` after `UseRouting` so rate limit policies can access endpoint metadata
- ✅ Place `UseOutputCache` after `UseAuthorization` to respect authorization results
- ✅ Place terminal middleware (`MapControllers`, `MapGet`) last

### Performance Tips

- ✅ Keep middleware lightweight — avoid blocking I/O in the hot path
- ✅ Use `Response.OnStarting` callbacks to set headers instead of buffering the entire response
- ✅ Prefer convention-based middleware for stateless logic (singleton, no allocation)
- ✅ Use `IMiddleware` only when scoped dependencies are required
- ✅ Enable response compression for JSON APIs (`AddResponseCompression`)
- ✅ Use output caching for read-heavy endpoints to avoid re-execution
- ✅ Use `EnableBuffering()` only when the request body must be read multiple times

### Do / Don't

- ✅ Do register `IExceptionHandler` (.NET 8+) for structured, DI-friendly error handling
- ✅ Do use `ProblemDetails` for consistent error responses across all endpoints
- ✅ Do use endpoint filters for Minimal API validation instead of MVC action filters
- ✅ Do use `MapGroup` to organize related endpoints and apply shared filters
- ✅ Do use rate limiting with per-user partitioning to prevent abuse
- ✅ Do use tag-based cache invalidation with output caching
- ✅ Do return `Retry-After` headers when rate limiting rejects a request
- ✅ Do test middleware in isolation with `DefaultHttpContext` or `WebApplicationFactory`

- ❌ Don't modify `HttpContext.Response` after the response has started — it throws `InvalidOperationException`
- ❌ Don't call `next()` after writing to the response body in terminal middleware
- ❌ Don't enable `UseDeveloperExceptionPage` in production — it exposes stack traces
- ❌ Don't cache responses that contain user-specific data without vary-by-user policies
- ❌ Don't use `UseResponseCaching` and `UseOutputCache` together — pick one strategy
- ❌ Don't add middleware after `MapControllers()` or `app.Run()` — it will never execute
- ❌ Don't perform heavy computation in middleware — offload to background services
- ❌ Don't ignore `CancellationToken` in middleware — always propagate it

## Next Steps

Return to the [Learning Path](LEARNING-PATH.md) for the complete curriculum. Review [Security and Authentication](09-SECURITY-AND-AUTHENTICATION.md) for authentication schemes, authorization policies, and identity management that integrate with the middleware pipeline covered here.

### Related Documents

1. **[Learning Path](LEARNING-PATH.md)** — Complete .NET learning curriculum
2. **[Services](02-SERVICES.md)** — Web APIs, worker services, gRPC, health checks
3. **[Dependency Injection](06-DEPENDENCY-INJECTION.md)** — Service lifetimes, registration patterns
4. **[Security and Authentication](09-SECURITY-AND-AUTHENTICATION.md)** — Auth schemes, policies, identity
5. **[Best Practices](03-BEST-PRACTICES.md)** — Async/await, memory, security, testing

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial middleware and HTTP pipeline documentation |
