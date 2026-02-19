# Breaking Changes from .NET Core 3.1

## Table of Contents

1. [Overview](#overview)
2. [Migration Path Overview](#migration-path-overview)
3. [LTS vs STS Release Schedule](#lts-vs-sts-release-schedule)
4. [Breaking Changes: .NET Core 3.1 to .NET 5](#breaking-changes-net-core-31-to-net-5)
5. [Breaking Changes: .NET 5 to .NET 6](#breaking-changes-net-5-to-net-6)
6. [Breaking Changes: .NET 6 to .NET 7](#breaking-changes-net-6-to-net-7)
7. [Breaking Changes: .NET 7 to .NET 8](#breaking-changes-net-7-to-net-8)
8. [Migration Checklist](#migration-checklist)
9. [Next Steps](#next-steps)

## Overview

This document catalogs the significant breaking changes encountered when migrating from .NET Core 3.1 through each subsequent major .NET release up to .NET 8. Each section includes tables of key changes, before/after code examples, and mitigation strategies.

### Target Audience

- Developers migrating applications from .NET Core 3.1 or later versions
- Tech leads planning upgrade timelines for .NET projects
- DevOps engineers preparing CI/CD pipelines for framework upgrades

### Scope

- Breaking API changes, behavioral changes, and removed features per major version
- Code migration patterns with before/after examples
- Mitigation strategies and compatibility shims
- LTS and STS release schedule for planning

## Migration Path Overview

```
┌─────────────────┐    ┌─────────────┐    ┌─────────────┐
│ .NET Core 3.1   │───►│   .NET 5    │───►│   .NET 6    │
│ (LTS)           │    │   (STS)     │    │   (LTS)     │
│ EOL: Dec 2022   │    │ EOL: May 22 │    │ EOL: Nov 24 │
└─────────────────┘    └─────────────┘    └──────┬──────┘
                                                  │
                       ┌─────────────┐    ┌──────▼──────┐
                       │   .NET 8    │◄───│   .NET 7    │
                       │   (LTS)     │    │   (STS)     │
                       │ EOL: Nov 26 │    │ EOL: May 24 │
                       └─────────────┘    └─────────────┘
```

### Recommended Migration Strategy

| Starting Version | Recommended Path | Notes |
|-----------------|-----------------|-------|
| .NET Core 3.1 | 3.1 → 6 → 8 | Skip .NET 5; jump to LTS releases |
| .NET 5 | 5 → 6 → 8 | Migrate to LTS as soon as possible |
| .NET 6 | 6 → 8 | Direct upgrade to latest LTS |
| .NET 7 | 7 → 8 | Direct upgrade to latest LTS |

> **Tip:** Always migrate to LTS releases for production workloads. Skip STS versions unless you need a specific feature.

## LTS vs STS Release Schedule

| Version | Release Date | Support Type | End of Life | Status |
|---------|-------------|-------------|-------------|--------|
| .NET Core 3.1 | Dec 2019 | LTS (3 years) | Dec 13, 2022 | ⚠️ EOL |
| .NET 5 | Nov 2020 | STS (18 months) | May 10, 2022 | ⚠️ EOL |
| .NET 6 | Nov 2021 | LTS (3 years) | Nov 12, 2024 | ⚠️ EOL |
| .NET 7 | Nov 2022 | STS (18 months) | May 14, 2024 | ⚠️ EOL |
| .NET 8 | Nov 2023 | LTS (3 years) | Nov 10, 2026 | ✅ Active |
| .NET 9 | Nov 2024 | STS (18 months) | May 2026 | ✅ Active |

```
Release Cadence (annual November releases):

2019  2020  2021  2022  2023  2024  2025  2026
  │     │     │     │     │     │     │     │
  ▼     ▼     ▼     ▼     ▼     ▼     ▼     ▼
 3.1    5     6     7     8     9    10    11
 LTS   STS   LTS   STS   LTS   STS   LTS   STS
```

## Breaking Changes: .NET Core 3.1 to .NET 5

### Key Breaking Changes

| Category | Change | Impact |
|----------|--------|--------|
| **Target framework** | TFM changed from `netcoreapp3.1` to `net5.0` | All project files |
| **JSON serialization** | `System.Text.Json` becomes the default serializer | API serialization behavior |
| **Blazor** | `Microsoft.AspNetCore.Components.Web` assembly changes | Blazor component namespaces |
| **ASP.NET Core** | `Startup.cs` pattern refined | App configuration |
| **Removed APIs** | Several `System.Security` APIs removed | Security code |
| **Globalization** | ICU library replaces NLS on Linux | String comparison behavior |
| **Kestrel** | Default HTTP/2 enabled over TLS | Network behavior |
| **WinForms** | Some controls removed or reorganized | Desktop apps |

### Target Framework Moniker (TFM)

**Before (.NET Core 3.1):**

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>netcoreapp3.1</TargetFramework>
  </PropertyGroup>
</Project>
```

**After (.NET 5):**

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net5.0</TargetFramework>
  </PropertyGroup>
</Project>
```

### JSON Serialization Changes

.NET 5 strongly encourages `System.Text.Json` over `Newtonsoft.Json`. The default serializer in ASP.NET Core is `System.Text.Json`.

**Before (Newtonsoft.Json — .NET Core 3.1 common pattern):**

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers()
        .AddNewtonsoftJson(options =>
        {
            options.SerializerSettings.ReferenceLoopHandling =
                ReferenceLoopHandling.Ignore;
            options.SerializerSettings.NullValueHandling =
                NullValueHandling.Ignore;
        });
}
```

**After (System.Text.Json — .NET 5+):**

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers()
        .AddJsonOptions(options =>
        {
            options.JsonSerializerOptions.ReferenceHandler =
                ReferenceHandler.IgnoreCycles;
            options.JsonSerializerOptions.DefaultIgnoreCondition =
                JsonIgnoreCondition.WhenWritingNull;
        });
}
```

**Key differences between Newtonsoft.Json and System.Text.Json:**

| Behavior | Newtonsoft.Json | System.Text.Json |
|----------|----------------|-----------------|
| Case sensitivity | Case-insensitive by default | Case-sensitive by default |
| Comments in JSON | Allowed by default | Disallowed by default |
| Trailing commas | Allowed by default | Disallowed by default |
| Number handling | Reads from strings | Strict by default |
| Polymorphic serialization | `TypeNameHandling` | `JsonDerivedType` attribute (.NET 7+) |
| `$ref` support | Built-in | `ReferenceHandler.Preserve` |

**Mitigation:** If you need Newtonsoft.Json compatibility, add the compatibility package:

```xml
<PackageReference Include="Microsoft.AspNetCore.Mvc.NewtonsoftJson" Version="5.0.0" />
```

### Globalization Changes

**Before (.NET Core 3.1 on Linux used NLS):**

```csharp
// Consistent behavior across platforms
"straße".ToUpper(); // "STRASSE" on Windows, potentially different on Linux
```

**After (.NET 5 uses ICU on all platforms):**

```csharp
// ICU-based globalization on all platforms
// Consistent Unicode-compliant behavior
"straße".ToUpper(); // "STRASSE" consistently

// To force invariant globalization (opt-out of ICU):
// Set in runtimeconfig.json or environment variable
// DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=true
```

### ASP.NET Core Hosting Changes

**Before (.NET Core 3.1):**

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

**After (.NET 5 — same pattern, but TFM changed):**

The hosting model in .NET 5 is similar to .NET Core 3.1. The major hosting model change comes in .NET 6 (see next section).

## Breaking Changes: .NET 5 to .NET 6

### Key Breaking Changes

| Category | Change | Impact |
|----------|--------|--------|
| **Hosting model** | Minimal hosting model (no `Startup.cs`) | App startup code |
| **Nullable references** | Enabled by default in templates | All new projects |
| **Middleware** | `UseRouting`/`UseEndpoints` implicit in middleware pipeline | Middleware ordering |
| **Logging** | `LoggerMessage.Define` source generator introduced | Logging patterns |
| **Blazor** | `DynamicComponent` and error boundaries added | Blazor app structure |
| **TFM** | `net5.0` → `net6.0` | Project files |
| **Hot reload** | Supported in development | Development workflow |
| **Minimal APIs** | New simplified API pattern | New project templates |

### Minimal Hosting Model

This is one of the most significant changes in .NET 6.

**Before (.NET 5 — Startup.cs pattern):**

```csharp
// Program.cs
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}

// Startup.cs
public class Startup
{
    public IConfiguration Configuration { get; }

    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();
        services.AddSwaggerGen();
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(Configuration.GetConnectionString("Default")));
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
            app.UseSwagger();
            app.UseSwaggerUI();
        }

        app.UseHttpsRedirection();
        app.UseRouting();
        app.UseAuthorization();
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}
```

**After (.NET 6 — Minimal hosting):**

```csharp
// Program.cs (single file, no Startup.cs)
var builder = WebApplication.CreateBuilder(args);

// Add services (replaces ConfigureServices)
builder.Services.AddControllers();
builder.Services.AddSwaggerGen();
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

var app = builder.Build();

// Configure pipeline (replaces Configure)
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

**Mitigation:** The `Startup.cs` pattern still works in .NET 6+. You can continue using it:

```csharp
var builder = WebApplication.CreateBuilder(args);
var startup = new Startup(builder.Configuration);
startup.ConfigureServices(builder.Services);
var app = builder.Build();
startup.Configure(app, app.Environment);
app.Run();
```

### Minimal APIs

**New in .NET 6 — minimal API pattern:**

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/hello", () => "Hello, World!");

app.MapGet("/users/{id}", async (int id, UserService service) =>
{
    var user = await service.GetByIdAsync(id);
    return user is not null ? Results.Ok(user) : Results.NotFound();
});

app.MapPost("/users", async (CreateUserRequest request, UserService service) =>
{
    var user = await service.CreateAsync(request);
    return Results.Created($"/users/{user.Id}", user);
});

app.Run();
```

### Nullable Reference Types Enabled by Default

**Before (.NET 5 templates — nullable not enabled):**

```xml
<PropertyGroup>
  <TargetFramework>net5.0</TargetFramework>
</PropertyGroup>
```

**After (.NET 6 templates — nullable enabled):**

```xml
<PropertyGroup>
  <TargetFramework>net6.0</TargetFramework>
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
</PropertyGroup>
```

**Impact on code:**

```csharp
// .NET 6+ with nullable enabled — compiler warnings on null misuse
public class UserService
{
    // Warning CS8618: Non-nullable property 'Name' must contain
    // a non-null value when exiting constructor
    public string Name { get; set; }      // ⚠️ Warning

    public string? Email { get; set; }     // ✅ Explicitly nullable

    public string Name { get; set; } = string.Empty; // ✅ Initialized
}
```

### Implicit Usings

.NET 6 introduces `ImplicitUsings` which automatically includes common namespaces:

```csharp
// These are implicitly available in .NET 6+ web projects:
// global using System;
// global using System.Collections.Generic;
// global using System.IO;
// global using System.Linq;
// global using System.Net.Http;
// global using System.Threading;
// global using System.Threading.Tasks;
// global using Microsoft.AspNetCore.Builder;
// global using Microsoft.AspNetCore.Http;
// global using Microsoft.Extensions.DependencyInjection;
// global using Microsoft.Extensions.Hosting;
// global using Microsoft.Extensions.Logging;
```

### Middleware Pipeline Changes

**Before (.NET 5):**

```csharp
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.UseEndpoints(endpoints =>
{
    endpoints.MapControllers();
});
```

**After (.NET 6):**

```csharp
// UseRouting and UseEndpoints are now implicit
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
```

## Breaking Changes: .NET 6 to .NET 7

### Key Breaking Changes

| Category | Change | Impact |
|----------|--------|--------|
| **TFM** | `net6.0` → `net7.0` | Project files |
| **Rate limiting** | New built-in rate limiting middleware | API protection |
| **Output caching** | New output caching middleware | Response caching |
| **gRPC** | gRPC JSON transcoding | gRPC service exposure |
| **Typed results** | `TypedResults` for minimal APIs | Minimal API return types |
| **Auth** | `AuthorizationBuilder` simplifies policy setup | Auth configuration |
| **JSON** | Polymorphic serialization, contract customization | JSON handling |
| **AOT** | Native AOT (preview) | Deployment model |

### Rate Limiting Middleware

**New in .NET 7:**

```csharp
using System.Threading.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", opt =>
    {
        opt.PermitLimit = 10;
        opt.Window = TimeSpan.FromSeconds(10);
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        opt.QueueLimit = 2;
    });

    options.AddSlidingWindowLimiter("sliding", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.SegmentsPerWindow = 6;
    });

    options.AddTokenBucketLimiter("token", opt =>
    {
        opt.TokenLimit = 100;
        opt.ReplenishmentPeriod = TimeSpan.FromSeconds(10);
        opt.TokensPerPeriod = 10;
    });

    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

var app = builder.Build();
app.UseRateLimiter();

app.MapGet("/api/resource", () => "Hello!")
    .RequireRateLimiting("fixed");

app.Run();
```

### Output Caching

**New in .NET 7:**

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(policy => policy.Expire(TimeSpan.FromMinutes(5)));
    options.AddPolicy("NoCache", policy => policy.NoCache());
    options.AddPolicy("ShortCache", policy =>
        policy.Expire(TimeSpan.FromSeconds(30)));
});

var app = builder.Build();
app.UseOutputCache();

app.MapGet("/api/products", async (ProductService service) =>
{
    return await service.GetAllAsync();
}).CacheOutput("ShortCache");

app.Run();
```

### gRPC JSON Transcoding

**New in .NET 7 — expose gRPC services as RESTful JSON APIs:**

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddGrpc();
builder.Services.AddGrpcJsonTranscoding();

var app = builder.Build();
app.MapGrpcService<GreeterService>();
app.Run();
```

```protobuf
// greeter.proto
service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply) {
    option (google.api.http) = {
      get: "/v1/greeter/{name}"
    };
  }
}
```

### Typed Results for Minimal APIs

**Before (.NET 6):**

```csharp
app.MapGet("/users/{id}", async (int id, UserService service) =>
{
    var user = await service.GetByIdAsync(id);
    return user is not null ? Results.Ok(user) : Results.NotFound();
});
```

**After (.NET 7 — TypedResults for better OpenAPI support):**

```csharp
app.MapGet("/users/{id}", async Task<Results<Ok<User>, NotFound>> (
    int id, UserService service) =>
{
    var user = await service.GetByIdAsync(id);
    return user is not null
        ? TypedResults.Ok(user)
        : TypedResults.NotFound();
});
```

### Polymorphic JSON Serialization

**Before (.NET 6 — limited polymorphism):**

```csharp
// No built-in polymorphic serialization
// Required custom converters or Newtonsoft.Json
```

**After (.NET 7 — attribute-based polymorphism):**

```csharp
[JsonDerivedType(typeof(CreditCardPayment), "credit")]
[JsonDerivedType(typeof(BankTransferPayment), "bank")]
public abstract class Payment
{
    public decimal Amount { get; set; }
}

public class CreditCardPayment : Payment
{
    public string CardNumber { get; set; } = string.Empty;
}

public class BankTransferPayment : Payment
{
    public string AccountNumber { get; set; } = string.Empty;
}

// Serializes with "$type" discriminator:
// { "$type": "credit", "amount": 99.99, "cardNumber": "****1234" }
```

## Breaking Changes: .NET 7 to .NET 8

### Key Breaking Changes

| Category | Change | Impact |
|----------|--------|--------|
| **TFM** | `net7.0` → `net8.0` | Project files |
| **Native AOT** | Full AOT support for ASP.NET Core | Deployment model |
| **Identity** | New Identity API endpoints | Authentication |
| **Keyed DI** | Keyed dependency injection services | DI registration |
| **Time provider** | `TimeProvider` abstraction | Time-dependent code |
| **Blazor** | Blazor United (SSR + interactive) | Blazor architecture |
| **JSON** | Source generation improvements | Serialization performance |
| **Metrics** | Built-in metrics with `System.Diagnostics.Metrics` | Observability |
| **Hosting** | `IHostedLifecycleService` | Hosted service lifecycle |
| **Validation** | `ValidateOptionsResultBuilder` | Options validation |

### Native AOT Compilation

**New in .NET 8 — full AOT support for ASP.NET Core:**

```xml
<PropertyGroup>
  <TargetFramework>net8.0</TargetFramework>
  <PublishAot>true</PublishAot>
</PropertyGroup>
```

```csharp
// AOT-compatible minimal API
var builder = WebApplication.CreateSlimBuilder(args);
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.TypeInfoResolverChain.Insert(0,
        AppJsonSerializerContext.Default);
});

var app = builder.Build();

app.MapGet("/hello", () => new HelloResponse("Hello, AOT!"));

app.Run();

public record HelloResponse(string Message);

[JsonSerializable(typeof(HelloResponse))]
internal partial class AppJsonSerializerContext : JsonSerializerContext { }
```

**AOT considerations:**

| Feature | AOT Compatible | Notes |
|---------|---------------|-------|
| Minimal APIs | ✅ Yes | Preferred pattern for AOT |
| Controllers | ⚠️ Limited | Not all features supported |
| System.Text.Json source gen | ✅ Yes | Required for AOT JSON |
| Reflection-based serialization | ❌ No | Use source generators |
| Dynamic assembly loading | ❌ No | Not supported |
| EF Core | ⚠️ Limited | Precompiled queries only |

### Identity API Endpoints

**New in .NET 8 — built-in Identity endpoints:**

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAuthentication().AddBearerToken(IdentityConstants.BearerScheme);
builder.Services.AddAuthorizationBuilder();

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite("Data Source=app.db"));

builder.Services.AddIdentityCore<AppUser>()
    .AddEntityFrameworkStores<AppDbContext>()
    .AddApiEndpoints();

var app = builder.Build();
app.MapIdentityApi<AppUser>();
app.Run();
```

This auto-registers endpoints: `/register`, `/login`, `/refresh`, `/confirmEmail`, `/manage/*`.

### Keyed Dependency Injection Services

**Before (.NET 7 — manual keyed service workarounds):**

```csharp
// Required factory pattern or named options
services.AddSingleton<ICache>(sp =>
    new RedisCache(sp.GetRequiredService<IConfiguration>()));
services.AddSingleton<ICache>(sp =>
    new MemoryCache());
// No way to resolve by key natively
```

**After (.NET 8 — built-in keyed services):**

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddKeyedSingleton<ICache, RedisCache>("redis");
builder.Services.AddKeyedSingleton<ICache, MemoryCache>("memory");

var app = builder.Build();

app.MapGet("/data", ([FromKeyedServices("redis")] ICache cache) =>
{
    return cache.Get("key");
});

app.Run();
```

```csharp
// Constructor injection with keyed services
public class OrderService
{
    private readonly ICache _cache;

    public OrderService([FromKeyedServices("redis")] ICache cache)
    {
        _cache = cache;
    }
}
```

### Time Provider Abstraction

**Before (.NET 7 — hard to test time-dependent code):**

```csharp
public class TokenService
{
    public bool IsExpired(DateTime expiresAt)
    {
        return DateTime.UtcNow > expiresAt; // Hard to test
    }
}
```

**After (.NET 8 — TimeProvider abstraction):**

```csharp
public class TokenService
{
    private readonly TimeProvider _timeProvider;

    public TokenService(TimeProvider timeProvider)
    {
        _timeProvider = timeProvider;
    }

    public bool IsExpired(DateTimeOffset expiresAt)
    {
        return _timeProvider.GetUtcNow() > expiresAt;
    }
}

// Registration
builder.Services.AddSingleton(TimeProvider.System);

// Testing with FakeTimeProvider
var fakeTime = new FakeTimeProvider(new DateTimeOffset(2025, 1, 1, 0, 0, 0, TimeSpan.Zero));
var service = new TokenService(fakeTime);
fakeTime.Advance(TimeSpan.FromHours(1));
```

### IHostedLifecycleService

**Before (.NET 7 — limited lifecycle hooks):**

```csharp
public class MyService : IHostedService
{
    public Task StartAsync(CancellationToken ct) => Task.CompletedTask;
    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

**After (.NET 8 — full lifecycle hooks):**

```csharp
public class MyService : IHostedLifecycleService
{
    public Task StartingAsync(CancellationToken ct)
    {
        // Called before StartAsync — warm up resources
        return Task.CompletedTask;
    }

    public Task StartAsync(CancellationToken ct)
    {
        // Main start logic
        return Task.CompletedTask;
    }

    public Task StartedAsync(CancellationToken ct)
    {
        // Called after StartAsync — service is ready
        return Task.CompletedTask;
    }

    public Task StoppingAsync(CancellationToken ct)
    {
        // Called before StopAsync — begin graceful shutdown
        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken ct)
    {
        // Main stop logic
        return Task.CompletedTask;
    }

    public Task StoppedAsync(CancellationToken ct)
    {
        // Called after StopAsync — cleanup complete
        return Task.CompletedTask;
    }
}
```

### Blazor United (Server-Side Rendering + Interactive)

**Before (.NET 7 — separate Blazor Server and WebAssembly):**

```csharp
// Blazor Server
builder.Services.AddServerSideBlazor();
app.MapBlazorHub();

// OR Blazor WebAssembly (separate project)
builder.Services.AddBlazorWebAssembly();
```

**After (.NET 8 — unified Blazor with per-component render modes):**

```csharp
// Program.cs
builder.Services.AddRazorComponents()
    .AddInteractiveServerComponents()
    .AddInteractiveWebAssemblyComponents();

app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode()
    .AddInteractiveWebAssemblyRenderMode();
```

```razor
@* Static SSR by default, opt-in to interactivity per component *@
@page "/counter"
@rendermode InteractiveServer

<h1>Counter</h1>
<p>Current count: @currentCount</p>
<button @onclick="IncrementCount">Click me</button>

@code {
    private int currentCount = 0;
    private void IncrementCount() => currentCount++;
}
```

## Migration Checklist

Use this checklist when upgrading between .NET versions.

### Pre-Migration

- [ ] Review release notes and breaking changes for the target version
- [ ] Identify all NuGet packages that need version updates
- [ ] Ensure CI/CD pipeline supports the target .NET SDK version
- [ ] Back up the project / create a migration branch
- [ ] Run existing tests to establish a passing baseline
- [ ] Check third-party library compatibility with target framework

### Project File Updates

- [ ] Update `<TargetFramework>` to the new TFM
- [ ] Update all `Microsoft.AspNetCore.*` and `Microsoft.Extensions.*` packages
- [ ] Update third-party NuGet packages to compatible versions
- [ ] Remove deprecated package references
- [ ] Add new `<PropertyGroup>` settings if needed (e.g., `<Nullable>enable</Nullable>`)

### Code Changes

- [ ] Fix compiler errors from removed or changed APIs
- [ ] Address nullable reference type warnings (if enabling `<Nullable>`)
- [ ] Update `Startup.cs` to minimal hosting model (optional but recommended)
- [ ] Replace deprecated middleware with new equivalents
- [ ] Update JSON serialization configuration if switching serializers
- [ ] Review and update authentication/authorization configuration
- [ ] Update Blazor components for new render modes (.NET 8)

### Testing and Validation

- [ ] Run all unit tests and fix failures
- [ ] Run integration tests with updated framework
- [ ] Test API contracts (OpenAPI spec, response formats)
- [ ] Validate globalization behavior (string comparison, formatting)
- [ ] Performance benchmark comparison (before vs. after)
- [ ] Test deployment pipeline with new SDK

### Post-Migration

- [ ] Update Dockerfiles to use the new SDK/runtime images
- [ ] Update CI/CD pipeline SDK version
- [ ] Update documentation and runbooks
- [ ] Monitor production metrics after deployment
- [ ] Remove any compatibility shims that are no longer needed

## Next Steps

Continue to [Services](02-SERVICES.md) to learn about dependency injection, Web API patterns, worker services, and service communication in modern .NET.

### Related Documents

1. **[Services](02-SERVICES.md)** — DI, Web APIs, worker services, gRPC, configuration
2. **[Best Practices](03-BEST-PRACTICES.md)** — Async/await, memory, security, testing
3. **[Style Guide](04-STYLE-GUIDE.md)** — Naming conventions, formatting, analyzers

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial breaking changes documentation covering .NET Core 3.1 through .NET 8 |
