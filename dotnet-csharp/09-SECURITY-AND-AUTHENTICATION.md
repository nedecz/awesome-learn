# Security and Authentication in .NET

## Table of Contents

1. [Overview](#overview)
2. [Authentication](#authentication)
3. [Authorization](#authorization)
4. [OAuth 2.0 and OpenID Connect](#oauth-20-and-openid-connect)
5. [JWT Tokens](#jwt-tokens)
6. [CORS](#cors)
7. [Data Protection API](#data-protection-api)
8. [Secrets Management](#secrets-management)
9. [HTTPS and Transport Security](#https-and-transport-security)
10. [Input Validation and Sanitization](#input-validation-and-sanitization)
11. [Security Headers](#security-headers)
12. [Best Practices](#best-practices)
13. [Next Steps](#next-steps)

## Overview

Security is a cross-cutting concern that spans every layer of a .NET application — from transport encryption to authentication, authorization, input validation, and secret management. ASP.NET Core provides a comprehensive security model with middleware-based authentication, policy-based authorization, and the Data Protection API.

### Target Audience

- .NET developers building web APIs and web applications
- Architects designing authentication and authorization strategies
- Teams implementing security hardening and compliance requirements

### Scope

- Authentication schemes (cookies, JWT, OpenID Connect)
- Policy-based and resource-based authorization
- OAuth 2.0 and OpenID Connect integration (Entra ID, Keycloak, Auth0)
- JWT token creation, validation, and refresh
- CORS configuration, Data Protection API, and secrets management
- HTTPS enforcement, input validation, and security headers
- OWASP Top 10 mapping and production security checklist

### Defense in Depth

```
┌─────────────────────────────────────────────────────────────────┐
│                    Defense in Depth                              │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Layer 1: Network                                          │  │
│  │   HTTPS/TLS, firewall rules, DDoS protection             │  │
│  │ ┌─────────────────────────────────────────────────────┐   │  │
│  │ │ Layer 2: Transport                                  │   │  │
│  │ │   HSTS, certificate pinning, TLS 1.2+              │   │  │
│  │ │ ┌───────────────────────────────────────────────┐   │   │  │
│  │ │ │ Layer 3: Authentication                       │   │   │  │
│  │ │ │   JWT, cookies, OpenID Connect                │   │   │  │
│  │ │ │ ┌─────────────────────────────────────────┐   │   │   │  │
│  │ │ │ │ Layer 4: Authorization                  │   │   │   │  │
│  │ │ │ │   Policies, roles, claims, resources    │   │   │   │  │
│  │ │ │ │ ┌───────────────────────────────────┐   │   │   │   │  │
│  │ │ │ │ │ Layer 5: Input Validation         │   │   │   │   │  │
│  │ │ │ │ │   Model validation, anti-XSS,     │   │   │   │   │  │
│  │ │ │ │ │   parameterized queries            │   │   │   │   │  │
│  │ │ │ │ │ ┌─────────────────────────────┐   │   │   │   │   │  │
│  │ │ │ │ │ │ Layer 6: Data Protection    │   │   │   │   │   │  │
│  │ │ │ │ │ │   Encryption at rest,       │   │   │   │   │   │  │
│  │ │ │ │ │ │   key management            │   │   │   │   │   │  │
│  │ │ │ │ │ └─────────────────────────────┘   │   │   │   │   │  │
│  │ │ │ │ └───────────────────────────────────┘   │   │   │   │  │
│  │ │ │ └─────────────────────────────────────────┘   │   │   │  │
│  │ │ └───────────────────────────────────────────────┘   │   │  │
│  │ └─────────────────────────────────────────────────────┘   │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Authentication

### ASP.NET Core Authentication Middleware

Authentication in ASP.NET Core is middleware-based. The authentication middleware reads credentials from the request, validates them, and creates a `ClaimsPrincipal`.

```
┌─────────────────────────────────────────────────────────────────┐
│              Authentication Middleware Pipeline                  │
│                                                                 │
│  HTTP Request                                                   │
│      │                                                          │
│      ▼                                                          │
│  ┌──────────────┐                                               │
│  │   HTTPS      │  Enforce TLS                                  │
│  │  Redirection │                                               │
│  └──────┬───────┘                                               │
│         ▼                                                       │
│  ┌──────────────┐                                               │
│  │Authentication│  Read token/cookie → validate → ClaimsPrincipal│
│  │  Middleware  │  Sets HttpContext.User                         │
│  └──────┬───────┘                                               │
│         ▼                                                       │
│  ┌──────────────┐                                               │
│  │Authorization │  Check policies, roles, claims                │
│  │  Middleware  │  [Authorize] attribute enforcement             │
│  └──────┬───────┘                                               │
│         ▼                                                       │
│  ┌──────────────┐                                               │
│  │   CORS       │  Cross-origin request handling                │
│  └──────┬───────┘                                               │
│         ▼                                                       │
│  ┌──────────────┐                                               │
│  │  Endpoint    │  Controller / Minimal API                     │
│  └──────────────┘                                               │
└─────────────────────────────────────────────────────────────────┘
```

### Cookie Authentication

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(options =>
    {
        options.LoginPath = "/account/login";
        options.LogoutPath = "/account/logout";
        options.AccessDeniedPath = "/account/access-denied";
        options.ExpireTimeSpan = TimeSpan.FromHours(8);
        options.SlidingExpiration = true;
        options.Cookie.HttpOnly = true;
        options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
        options.Cookie.SameSite = SameSiteMode.Strict;
    });

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
```

### Sign In with Cookie Authentication

```csharp
app.MapPost("/account/login", async (
    LoginRequest request,
    HttpContext httpContext,
    IUserService userService) =>
{
    var user = await userService.ValidateCredentialsAsync(request.Username, request.Password);
    if (user is null)
        return Results.Unauthorized();

    var claims = new List<Claim>
    {
        new(ClaimTypes.NameIdentifier, user.Id.ToString()),
        new(ClaimTypes.Name, user.Username),
        new(ClaimTypes.Email, user.Email),
        new(ClaimTypes.Role, user.Role)
    };

    var identity = new ClaimsIdentity(claims, CookieAuthenticationDefaults.AuthenticationScheme);
    var principal = new ClaimsPrincipal(identity);

    await httpContext.SignInAsync(
        CookieAuthenticationDefaults.AuthenticationScheme,
        principal,
        new AuthenticationProperties
        {
            IsPersistent = request.RememberMe,
            ExpiresUtc = DateTimeOffset.UtcNow.AddHours(8)
        });

    return Results.Ok();
});

app.MapPost("/account/logout", async (HttpContext httpContext) =>
{
    await httpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);
    return Results.Ok();
}).RequireAuthorization();
```

### JWT Bearer Authentication

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!)),
            ClockSkew = TimeSpan.FromMinutes(1)
        };
    });
```

### ASP.NET Core Identity

ASP.NET Core Identity provides a complete user management system with user registration, login, password hashing, two-factor authentication, and external login providers.

```csharp
// Add Identity with EF Core stores
builder.Services.AddIdentity<ApplicationUser, IdentityRole>(options =>
{
    options.Password.RequireDigit = true;
    options.Password.RequireLowercase = true;
    options.Password.RequireUppercase = true;
    options.Password.RequiredLength = 12;
    options.Password.RequireNonAlphanumeric = true;

    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
    options.Lockout.MaxFailedAccessAttempts = 5;

    options.User.RequireUniqueEmail = true;
})
.AddEntityFrameworkStores<AppDbContext>()
.AddDefaultTokenProviders();

// Custom user class
public class ApplicationUser : IdentityUser
{
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}
```

## Authorization

### Policy-Based Authorization

```csharp
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"))
    .AddPolicy("MinimumAge", policy =>
        policy.RequireClaim("DateOfBirth")
              .AddRequirements(new MinimumAgeRequirement(18)))
    .AddPolicy("PremiumUser", policy =>
        policy.RequireRole("Premium")
              .RequireClaim("subscription", "active"))
    .AddPolicy("CanEditProduct", policy =>
        policy.AddRequirements(new ProductOwnerRequirement()));
```

### Using Authorization in Controllers and Minimal APIs

```csharp
// Controller-based
[ApiController]
[Route("api/[controller]")]
[Authorize] // Requires authentication for all actions
public class ProductsController(IProductService productService) : ControllerBase
{
    [HttpGet]
    [AllowAnonymous] // Override class-level [Authorize]
    public async Task<IActionResult> GetAll()
        => Ok(await productService.GetAllAsync());

    [HttpPost]
    [Authorize(Policy = "AdminOnly")]
    public async Task<IActionResult> Create(CreateProductRequest request)
    {
        var product = await productService.CreateAsync(request);
        return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
    }

    [HttpGet("{id:int}")]
    public async Task<IActionResult> GetById(int id)
        => Ok(await productService.GetByIdAsync(id));
}

// Minimal API
app.MapGet("/api/products", async (IProductService service) =>
    Results.Ok(await service.GetAllAsync()));

app.MapPost("/api/products", async (CreateProductRequest request, IProductService service) =>
{
    var product = await service.CreateAsync(request);
    return Results.Created($"/api/products/{product.Id}", product);
}).RequireAuthorization("AdminOnly");
```

### Custom Authorization Handler

```csharp
public class MinimumAgeRequirement(int minimumAge) : IAuthorizationRequirement
{
    public int MinimumAge { get; } = minimumAge;
}

public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var dateOfBirthClaim = context.User.FindFirst("DateOfBirth");

        if (dateOfBirthClaim is null)
            return Task.CompletedTask; // Requirement not satisfied

        var dateOfBirth = DateTime.Parse(dateOfBirthClaim.Value);
        var age = DateTime.Today.Year - dateOfBirth.Year;

        if (dateOfBirth.Date > DateTime.Today.AddYears(-age))
            age--;

        if (age >= requirement.MinimumAge)
            context.Succeed(requirement);

        return Task.CompletedTask;
    }
}

// Register the handler
builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
```

### Resource-Based Authorization

```csharp
public class ProductOwnerRequirement : IAuthorizationRequirement { }

public class ProductOwnerHandler : AuthorizationHandler<ProductOwnerRequirement, Product>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        ProductOwnerRequirement requirement,
        Product resource)
    {
        var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;

        if (userId is not null && resource.OwnerId.ToString() == userId)
            context.Succeed(requirement);

        return Task.CompletedTask;
    }
}

// Usage in a controller or endpoint
[HttpPut("{id:int}")]
public async Task<IActionResult> Update(
    int id,
    UpdateProductRequest request,
    IAuthorizationService authorizationService)
{
    var product = await _productService.GetByIdAsync(id);
    if (product is null)
        return NotFound();

    var authResult = await authorizationService.AuthorizeAsync(User, product, "CanEditProduct");
    if (!authResult.Succeeded)
        return Forbid();

    await _productService.UpdateAsync(id, request);
    return NoContent();
}
```

### Claims-Based Authorization

```csharp
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("EmployeesOnly", policy =>
        policy.RequireClaim("EmployeeId"))
    .AddPolicy("SeniorEmployees", policy =>
        policy.RequireClaim("EmployeeLevel", "Senior", "Lead", "Principal"))
    .AddPolicy("SpecificDepartment", policy =>
        policy.RequireAssertion(context =>
        {
            var department = context.User.FindFirst("Department")?.Value;
            return department is "Engineering" or "Product";
        }));
```

## OAuth 2.0 and OpenID Connect

### OAuth 2.0 Authorization Code Flow

```
┌─────────────────────────────────────────────────────────────────┐
│           OAuth 2.0 Authorization Code Flow with PKCE           │
│                                                                 │
│  ┌──────┐                                        ┌───────────┐ │
│  │ User │                                        │  Identity  │ │
│  │      │                                        │  Provider  │ │
│  └──┬───┘                                        │(Entra ID / │ │
│     │                                            │ Auth0)     │ │
│     │  1. Click "Login"                          └─────┬──────┘ │
│     ▼                                                  │        │
│  ┌──────────┐  2. Redirect to /authorize               │        │
│  │  Client  │ ────────────────────────────────────────►│        │
│  │  (SPA /  │                                          │        │
│  │  Web App)│  3. User authenticates                   │        │
│  │          │◄─────── (login form) ───────────────────►│        │
│  │          │                                          │        │
│  │          │  4. Redirect back with authorization code│        │
│  │          │◄─────────────────────────────────────────│        │
│  │          │                                          │        │
│  │          │  5. Exchange code for tokens              │        │
│  │          │  (code + code_verifier → access_token)   │        │
│  │          │ ────────────────────────────────────────►│        │
│  │          │◄─────────────────────────────────────────│        │
│  │          │  6. Access token + ID token + refresh    │        │
│  └──────────┘                                          │        │
│     │                                                  │        │
│     │  7. Call API with access_token                   │        │
│     ▼                                                  │        │
│  ┌──────────┐                                          │        │
│  │  API     │  8. Validate token                       │        │
│  │  Server  │ ────────────────────────────────────────►│        │
│  └──────────┘                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Client Credentials Flow (Service-to-Service)

```
┌─────────────────────────────────────────────────────────────────┐
│              Client Credentials Flow                             │
│                                                                 │
│  ┌──────────────┐  1. POST /token                               │
│  │  Service A   │     client_id + client_secret    ┌──────────┐│
│  │  (API Client)│ ──────────────────────────────► │ Identity ││
│  │              │◄────────────────────────────────  │ Provider ││
│  │              │  2. Access token                  └──────────┘│
│  │              │                                               │
│  │              │  3. Call API with Bearer token                 │
│  │              │ ──────────────────────────────► ┌──────────┐ │
│  │              │                                 │ Service B │ │
│  └──────────────┘                                 │  (API)    │ │
│                                                   └──────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Microsoft Entra ID (Azure AD) Integration

```csharp
// Install: Microsoft.Identity.Web

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));

// appsettings.json
// {
//   "AzureAd": {
//     "Instance": "https://login.microsoftonline.com/",
//     "TenantId": "your-tenant-id",
//     "ClientId": "your-client-id",
//     "Audience": "api://your-client-id"
//   }
// }
```

### Keycloak Integration

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://keycloak.example.com/realms/my-realm";
        options.Audience = "my-api";
        options.RequireHttpsMetadata = true;

        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = "https://keycloak.example.com/realms/my-realm",
            ValidateAudience = true,
            ValidAudience = "my-api",
            ValidateLifetime = true,
            RoleClaimType = "realm_access.roles"
        };
    });
```

### Auth0 Integration

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = $"https://{builder.Configuration["Auth0:Domain"]}";
        options.Audience = builder.Configuration["Auth0:Audience"];

        options.TokenValidationParameters = new TokenValidationParameters
        {
            NameClaimType = "name",
            RoleClaimType = "https://myapp.com/roles"
        };
    });
```

## JWT Tokens

### Creating JWT Tokens

```csharp
public class TokenService(IConfiguration configuration)
{
    public TokenResult GenerateTokens(ApplicationUser user, IList<string> roles)
    {
        var key = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(configuration["Jwt:Key"]!));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id),
            new(JwtRegisteredClaimNames.Email, user.Email!),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new(JwtRegisteredClaimNames.Iat,
                DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(),
                ClaimValueTypes.Integer64)
        };

        // Add role claims
        claims.AddRange(roles.Select(role => new Claim(ClaimTypes.Role, role)));

        var accessToken = new JwtSecurityToken(
            issuer: configuration["Jwt:Issuer"],
            audience: configuration["Jwt:Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(15),
            signingCredentials: credentials);

        var refreshToken = GenerateRefreshToken();

        return new TokenResult
        {
            AccessToken = new JwtSecurityTokenHandler().WriteToken(accessToken),
            RefreshToken = refreshToken,
            ExpiresAt = accessToken.ValidTo
        };
    }

    private static string GenerateRefreshToken()
    {
        var randomBytes = new byte[64];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(randomBytes);
        return Convert.ToBase64String(randomBytes);
    }
}

public record TokenResult
{
    public required string AccessToken { get; init; }
    public required string RefreshToken { get; init; }
    public DateTime ExpiresAt { get; init; }
}
```

### Refresh Token Endpoint

```csharp
app.MapPost("/api/auth/refresh", async (
    RefreshTokenRequest request,
    TokenService tokenService,
    UserManager<ApplicationUser> userManager,
    AppDbContext dbContext) =>
{
    // Validate the refresh token
    var storedToken = await dbContext.RefreshTokens
        .FirstOrDefaultAsync(t => t.Token == request.RefreshToken && !t.IsRevoked);

    if (storedToken is null || storedToken.ExpiresAt < DateTime.UtcNow)
        return Results.Unauthorized();

    var user = await userManager.FindByIdAsync(storedToken.UserId);
    if (user is null)
        return Results.Unauthorized();

    // Revoke old refresh token
    storedToken.IsRevoked = true;

    // Generate new tokens
    var roles = await userManager.GetRolesAsync(user);
    var tokens = tokenService.GenerateTokens(user, roles);

    // Store new refresh token
    dbContext.RefreshTokens.Add(new RefreshToken
    {
        Token = tokens.RefreshToken,
        UserId = user.Id,
        ExpiresAt = DateTime.UtcNow.AddDays(7),
        CreatedAt = DateTime.UtcNow
    });

    await dbContext.SaveChangesAsync();

    return Results.Ok(tokens);
});
```

### Token Lifetime Recommendations

| Token Type | Recommended Lifetime | Notes |
|---|---|---|
| Access token | 5–15 minutes | Short-lived, stateless |
| Refresh token | 7–30 days | Long-lived, stored securely, revocable |
| ID token | Same as access token | Used for user info, not API access |

## CORS

### CORS Configuration

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddCors(options =>
{
    // Named policy for specific origins
    options.AddPolicy("AllowFrontend", policy =>
    {
        policy.WithOrigins(
                "https://app.example.com",
                "https://admin.example.com")
            .WithMethods("GET", "POST", "PUT", "DELETE")
            .WithHeaders("Authorization", "Content-Type")
            .AllowCredentials()
            .SetPreflightMaxAge(TimeSpan.FromMinutes(10));
    });

    // Restrictive default policy
    options.AddDefaultPolicy(policy =>
    {
        policy.WithOrigins("https://app.example.com")
            .AllowAnyMethod()
            .AllowAnyHeader();
    });
});

var app = builder.Build();

// Apply CORS middleware (must be before auth middleware in the pipeline)
app.UseCors("AllowFrontend");
```

### Per-Endpoint CORS

```csharp
// Allow specific policy on a single endpoint
app.MapGet("/api/public/health", () => Results.Ok("Healthy"))
    .RequireCors("AllowFrontend");

// Disable CORS on a specific endpoint
[DisableCors]
[HttpGet("internal-only")]
public IActionResult InternalEndpoint() => Ok();
```

## Data Protection API

The Data Protection API provides cryptographic services for protecting data at rest. It is used internally by ASP.NET Core for cookie encryption, anti-forgery tokens, and session state.

### Basic Usage

```csharp
// Register Data Protection
builder.Services.AddDataProtection()
    .SetApplicationName("MyApp")
    .PersistKeysToFileSystem(new DirectoryInfo("/var/keys"))
    .SetDefaultKeyLifetime(TimeSpan.FromDays(90));

// Usage in a service
public class SecureDataService(IDataProtectionProvider provider)
{
    private readonly IDataProtector _protector = provider.CreateProtector("SecureDataService.v1");

    public string Protect(string plainText) => _protector.Protect(plainText);

    public string Unprotect(string protectedText) => _protector.Unprotect(protectedText);
}
```

### Purpose Strings

Purpose strings create isolated protectors — data protected with one purpose string cannot be unprotected with another.

```csharp
public class TokenProtectionService(IDataProtectionProvider provider)
{
    // Different protectors for different use cases
    private readonly IDataProtector _emailConfirmation =
        provider.CreateProtector("EmailConfirmation");
    private readonly IDataProtector _passwordReset =
        provider.CreateProtector("PasswordReset");

    public string CreateEmailConfirmationToken(string userId)
        => _emailConfirmation.Protect(userId);

    public string? ValidateEmailConfirmationToken(string token)
    {
        try { return _emailConfirmation.Unprotect(token); }
        catch (CryptographicException) { return null; }
    }

    public string CreatePasswordResetToken(string userId)
        => _passwordReset.Protect(userId);
}
```

### Key Rotation and Azure Key Vault

```csharp
// Persist keys to Azure Blob Storage, protected by Azure Key Vault
builder.Services.AddDataProtection()
    .SetApplicationName("MyApp")
    .PersistKeysToAzureBlobStorage(new Uri("https://mystore.blob.core.windows.net/keys/keys.xml"))
    .ProtectKeysWithAzureKeyVault(
        new Uri("https://myvault.vault.azure.net/keys/dataprotection"),
        new DefaultAzureCredential());
```

## Secrets Management

### User Secrets (Development)

```bash
# Initialize user secrets for a project
dotnet user-secrets init --project src/MyApp.Api

# Set a secret
dotnet user-secrets set "Jwt:Key" "my-super-secret-key-for-development" --project src/MyApp.Api

# List secrets
dotnet user-secrets list --project src/MyApp.Api
```

```csharp
// User secrets are automatically loaded in Development environment
var builder = WebApplication.CreateBuilder(args);
// builder.Configuration already includes user secrets
var jwtKey = builder.Configuration["Jwt:Key"];
```

### Azure Key Vault

```csharp
// Install: Azure.Extensions.AspNetCore.Configuration.Secrets
builder.Configuration.AddAzureKeyVault(
    new Uri("https://myvault.vault.azure.net"),
    new DefaultAzureCredential());

// Secrets are accessible via IConfiguration
var connectionString = builder.Configuration["ConnectionStrings:Default"];
```

### Environment Variables

```csharp
// Environment variables are loaded by default
// Use double underscore (__) to represent section separators
// e.g., ConnectionStrings__Default maps to ConnectionStrings:Default

// Docker / Kubernetes
// ENV ConnectionStrings__Default="Server=db;Database=MyApp;..."
```

### Configuration Priority

```
┌─────────────────────────────────────────────────────────────────┐
│              Configuration Priority (highest to lowest)          │
│                                                                 │
│  1. Command-line arguments                         (highest)    │
│  2. Environment variables                                       │
│  3. User secrets (Development only)                             │
│  4. appsettings.{Environment}.json                              │
│  5. appsettings.json                               (lowest)    │
│  6. Azure Key Vault (if configured)                             │
│                                                                 │
│  Higher priority sources override lower priority sources.       │
│  Never store secrets in appsettings.json — use user secrets     │
│  in development and Key Vault / environment variables in prod.  │
└─────────────────────────────────────────────────────────────────┘
```

## HTTPS and Transport Security

### HTTPS Redirection and HSTS

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHsts(options =>
{
    options.MaxAge = TimeSpan.FromDays(365);
    options.IncludeSubDomains = true;
    options.Preload = true;
});

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}

app.UseHttpsRedirection();
```

### Certificate Management

```csharp
// Kestrel HTTPS configuration
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(5001, listenOptions =>
    {
        listenOptions.UseHttps("certificate.pfx", "password");
    });
});

// From configuration
// {
//   "Kestrel": {
//     "Endpoints": {
//       "Https": {
//         "Url": "https://*:5001",
//         "Certificate": {
//           "Path": "/certs/cert.pfx",
//           "Password": "cert-password"
//         }
//       }
//     }
//   }
// }
```

### TLS Best Practices

| Practice | Recommendation |
|---|---|
| Minimum TLS version | TLS 1.2 (TLS 1.3 preferred) |
| Certificate type | Use certificates from a trusted CA |
| HSTS | Enable with `max-age=31536000; includeSubDomains; preload` |
| Certificate rotation | Automate with Let's Encrypt or managed certificates |
| Internal services | Use mTLS (mutual TLS) for service-to-service |

## Input Validation and Sanitization

### Model Validation

```csharp
public class CreateProductRequest
{
    [Required(ErrorMessage = "Name is required")]
    [StringLength(200, MinimumLength = 1)]
    public string Name { get; set; } = string.Empty;

    [Range(0.01, 999999.99, ErrorMessage = "Price must be between 0.01 and 999999.99")]
    public decimal Price { get; set; }

    [Required]
    [RegularExpression(@"^[A-Z]{3}-\d{3}$", ErrorMessage = "SKU must match format XXX-000")]
    public string Sku { get; set; } = string.Empty;

    [EmailAddress]
    public string? ContactEmail { get; set; }
}

// Validation is automatic in controllers with [ApiController]
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpPost]
    public IActionResult Create([FromBody] CreateProductRequest request)
    {
        // ModelState is automatically validated — invalid requests return 400
        return Ok();
    }
}
```

### FluentValidation

```csharp
// Install: FluentValidation.DependencyInjectionExtensions

public class CreateProductValidator : AbstractValidator<CreateProductRequest>
{
    public CreateProductValidator(AppDbContext context)
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required")
            .MaximumLength(200);

        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("Price must be positive")
            .LessThan(1_000_000);

        RuleFor(x => x.Sku)
            .NotEmpty()
            .Matches(@"^[A-Z]{3}-\d{3}$")
            .MustAsync(async (sku, ct) =>
                !await context.Products.AnyAsync(p => p.Sku == sku, ct))
            .WithMessage("SKU already exists");
    }
}

// Register validators
builder.Services.AddValidatorsFromAssemblyContaining<CreateProductValidator>();
```

### SQL Injection Prevention

```csharp
// ✅ Parameterized query (safe) — EF Core always parameterizes
var products = await context.Products
    .Where(p => p.Name.Contains(searchTerm))
    .ToListAsync();

// ✅ FromSqlInterpolated (safe) — parameters are automatically extracted
var products = await context.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE Name LIKE {'%' + searchTerm + '%'}")
    .ToListAsync();

// ❌ String concatenation (VULNERABLE — never do this)
// var products = await context.Products
//     .FromSqlRaw("SELECT * FROM Products WHERE Name = '" + searchTerm + "'")
//     .ToListAsync();
```

### Anti-XSS

```csharp
using System.Text.Encodings.Web;

public class SafeContentService(HtmlEncoder htmlEncoder)
{
    public string SanitizeHtml(string input)
    {
        // Encodes HTML special characters to prevent XSS
        return htmlEncoder.Encode(input);
    }
}

// Razor pages automatically encode output with @
// @Model.UserInput  ← automatically HTML-encoded
// @Html.Raw(Model.UserInput)  ← DANGEROUS — bypasses encoding
```

### Anti-Forgery Tokens

```csharp
// Automatically added to forms in Razor Pages
builder.Services.AddAntiforgery(options =>
{
    options.HeaderName = "X-CSRF-TOKEN";
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.HttpOnly = true;
});

// Validate in Minimal APIs
app.MapPost("/api/form-submit", async (HttpContext context, IAntiforgery antiforgery) =>
{
    await antiforgery.ValidateRequestAsync(context);
    return Results.Ok();
});
```

## Security Headers

### Middleware for Security Headers

```csharp
public class SecurityHeadersMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext context)
    {
        // Content Security Policy
        context.Response.Headers.Append(
            "Content-Security-Policy",
            "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'");

        // Prevent MIME type sniffing
        context.Response.Headers.Append("X-Content-Type-Options", "nosniff");

        // Prevent clickjacking
        context.Response.Headers.Append("X-Frame-Options", "DENY");

        // Referrer policy
        context.Response.Headers.Append("Referrer-Policy", "strict-origin-when-cross-origin");

        // Permissions policy (formerly Feature-Policy)
        context.Response.Headers.Append(
            "Permissions-Policy",
            "camera=(), microphone=(), geolocation=()");

        // Remove server header
        context.Response.Headers.Remove("Server");
        context.Response.Headers.Remove("X-Powered-By");

        await next(context);
    }
}

// Register in the pipeline
app.UseMiddleware<SecurityHeadersMiddleware>();
```

### Security Headers Reference

| Header | Value | Purpose |
|---|---|---|
| `Content-Security-Policy` | `default-src 'self'` | Controls resource loading — prevents XSS |
| `X-Content-Type-Options` | `nosniff` | Prevents MIME type sniffing |
| `X-Frame-Options` | `DENY` | Prevents clickjacking |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Forces HTTPS |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Controls referrer information |
| `Permissions-Policy` | `camera=(), microphone=()` | Restricts browser features |
| `Cache-Control` | `no-store` (for sensitive data) | Prevents caching of sensitive responses |

## Best Practices

### OWASP Top 10 Mapping

| OWASP Risk | .NET Mitigation |
|---|---|
| A01 Broken Access Control | Policy-based authorization, resource-based checks, `[Authorize]` |
| A02 Cryptographic Failures | Data Protection API, HTTPS/TLS, Azure Key Vault |
| A03 Injection | Parameterized queries (EF Core), input validation, `HtmlEncoder` |
| A04 Insecure Design | Threat modeling, defense in depth, least privilege |
| A05 Security Misconfiguration | Security headers, HSTS, remove default error pages |
| A06 Vulnerable Components | `dotnet list package --vulnerable`, Dependabot |
| A07 Auth Failures | ASP.NET Core Identity, account lockout, MFA |
| A08 Data Integrity Failures | Anti-forgery tokens, signed JWTs, code signing |
| A09 Logging Failures | Structured logging (Serilog), audit trails, no PII in logs |
| A10 Server-Side Request Forgery | Validate URLs, restrict outbound traffic, use allow-lists |

### Production Checklist

- ✅ Use HTTPS everywhere — enforce with HSTS and `UseHttpsRedirection()`
- ✅ Use policy-based authorization — not just `[Authorize]` with no policy
- ✅ Validate all inputs with Data Annotations or FluentValidation
- ✅ Use parameterized queries — EF Core and Dapper both support this natively
- ✅ Use short-lived access tokens (5–15 min) with refresh token rotation
- ✅ Store secrets in Azure Key Vault or user secrets — never in `appsettings.json`
- ✅ Add security headers (CSP, X-Content-Type-Options, X-Frame-Options)
- ✅ Use the Data Protection API for encrypting sensitive data at rest
- ✅ Enable account lockout and strong password policies in ASP.NET Core Identity
- ✅ Use `Cookie.HttpOnly = true` and `Cookie.SecurePolicy = Always` for auth cookies
- ✅ Audit dependencies for vulnerabilities with `dotnet list package --vulnerable`
- ✅ Use anti-forgery tokens for form submissions and state-changing requests
- ✅ Log security events (failed logins, authorization failures) with structured logging
- ✅ Use `SameSite=Strict` or `SameSite=Lax` for cookies to prevent CSRF

- ❌ Don't store secrets in source code or `appsettings.json`
- ❌ Don't use `[AllowAnonymous]` on endpoints that handle sensitive data
- ❌ Don't disable HTTPS in production — even for internal services
- ❌ Don't use `Html.Raw()` with user input — it bypasses XSS protection
- ❌ Don't use string concatenation for SQL queries — always parameterize
- ❌ Don't log sensitive information (passwords, tokens, PII)
- ❌ Don't use long-lived access tokens — use short-lived tokens with refresh
- ❌ Don't skip input validation on the server — client-side validation is not enough
- ❌ Don't expose detailed error messages in production — use `ProblemDetails`

## Next Steps

Return to the [Learning Path](LEARNING-PATH.md) for the complete curriculum and additional topics. Also review [Services](02-SERVICES.md) for patterns on securing ASP.NET Core APIs and service-to-service communication.

### Related Documents

1. **[Learning Path](LEARNING-PATH.md)** — Complete .NET learning curriculum
2. **[Services](02-SERVICES.md)** — Web APIs, worker services, gRPC
3. **[Entity Framework Core](07-ENTITY-FRAMEWORK-CORE.md)** — Data access, querying, migrations
4. **[Testing](08-TESTING.md)** — Unit testing, integration testing, security testing
5. **[Best Practices](03-BEST-PRACTICES.md)** — Async/await, memory, security, testing

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial security and authentication documentation |
