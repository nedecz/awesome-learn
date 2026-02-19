# Configuration and Deployment for .NET Applications

## Table of Contents

1. [Overview](#overview)
2. [Configuration System](#configuration-system)
3. [Options Pattern](#options-pattern)
4. [Environment-Specific Configuration](#environment-specific-configuration)
5. [Docker](#docker)
6. [CI/CD Pipelines](#cicd-pipelines)
7. [Publishing and Deployment](#publishing-and-deployment)
8. [Health Checks](#health-checks)
9. [Logging and Diagnostics in Production](#logging-and-diagnostics-in-production)
10. [Feature Flags](#feature-flags)
11. [Best Practices](#best-practices)
12. [Next Steps](#next-steps)

## Overview

This document covers the full lifecycle of configuring and deploying .NET applications — from reading configuration at startup through containerization, CI/CD pipelines, health monitoring, and production diagnostics. The patterns here apply to ASP.NET Core web APIs, worker services, and console applications targeting .NET 10 and C# 13.

### Target Audience

- Developers building and deploying ASP.NET Core applications to cloud or on-premises environments
- DevOps engineers creating CI/CD pipelines and Docker images for .NET workloads
- Architects designing configuration strategies, health monitoring, and feature rollout

### Scope

- Configuration providers: JSON, environment variables, command-line args, user secrets, Azure App Configuration
- Options pattern with validation (DataAnnotations, `IValidateOptions<T>`)
- Environment-specific settings (Development, Staging, Production)
- Docker multi-stage builds and Docker Compose
- GitHub Actions and Azure DevOps CI/CD pipelines
- Publishing modes: framework-dependent, self-contained, single-file, Native AOT, ReadyToRun
- Health checks, liveness/readiness probes, and health check UI
- Structured logging (Serilog, OpenTelemetry), Application Insights, diagnostic tools
- Feature flags with `Microsoft.FeatureManagement`

## Configuration System

.NET uses a layered configuration model where providers are added in order and later providers override earlier ones. The default `WebApplication.CreateBuilder` registers providers automatically.

### Configuration Provider Order

```
┌─────────────────────────────────────────────────────────────┐
│                Configuration Provider Stack                  │
│            (later providers override earlier ones)            │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  7. Command-line arguments         ← HIGHEST PRIORITY  │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  6. Environment variables (non-prefixed)               │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  5. User secrets (Development only)                    │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  4. appsettings.{Environment}.json                     │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  3. appsettings.json                                   │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  2. App-specific providers (Azure, AWS, etc.)          │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  1. Host configuration defaults    ← LOWEST PRIORITY   │  │
│  └────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### appsettings.json

The default JSON configuration file is loaded automatically by `WebApplication.CreateBuilder`.

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=AppDb;Trusted_Connection=true"
  },
  "SmtpSettings": {
    "Host": "smtp.example.com",
    "Port": 587,
    "EnableSsl": true,
    "SenderEmail": "noreply@example.com"
  },
  "AllowedHosts": "*"
}
```

### Reading Configuration Values

```csharp
var builder = WebApplication.CreateBuilder(args);

// Direct access via key path
string connStr = builder.Configuration.GetConnectionString("DefaultConnection")!;
string smtpHost = builder.Configuration["SmtpSettings:Host"]!;
int smtpPort = builder.Configuration.GetValue<int>("SmtpSettings:Port");

// Bind to a section
var smtpSection = builder.Configuration.GetSection("SmtpSettings");
```

### Environment Variables

Environment variables use a `__` (double underscore) separator to represent hierarchy.

```bash
# Equivalent to SmtpSettings:Host in JSON
export SmtpSettings__Host="smtp.production.com"
export SmtpSettings__Port="465"
export ConnectionStrings__DefaultConnection="Server=prod-db;..."

# Prefixed environment variables (common in Docker/Kubernetes)
export DOTNET_ENVIRONMENT=Production
export ASPNETCORE_URLS="http://+:8080"
```

```csharp
// Add prefixed environment variables
builder.Configuration.AddEnvironmentVariables(prefix: "MYAPP_");

// MYAPP_SmtpSettings__Host becomes SmtpSettings:Host
```

### Command-Line Arguments

```bash
dotnet run --SmtpSettings:Host=smtp.override.com --SmtpSettings:Port=465
```

```csharp
// Command-line args are added automatically by CreateBuilder(args)
var builder = WebApplication.CreateBuilder(args);

// Switch mappings for shorthand arguments
builder.Configuration.AddCommandLine(args, new Dictionary<string, string>
{
    { "--smtp-host", "SmtpSettings:Host" },
    { "--smtp-port", "SmtpSettings:Port" },
    { "-e", "Environment" }
});
```

### User Secrets (Development Only)

User secrets store sensitive values outside the project directory and are only loaded when `ASPNETCORE_ENVIRONMENT` is `Development`.

```bash
# Initialize user secrets in a project
dotnet user-secrets init

# Set secret values
dotnet user-secrets set "SmtpSettings:Password" "my-dev-password"
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=localhost;..."

# List all secrets
dotnet user-secrets list
```

```csharp
// User secrets are added automatically in Development by CreateBuilder
// For explicit registration:
if (builder.Environment.IsDevelopment())
{
    builder.Configuration.AddUserSecrets<Program>();
}
```

### Azure App Configuration

```csharp
// Add Azure App Configuration as a provider
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(builder.Configuration["AzureAppConfig:ConnectionString"])
           .Select(KeyFilter.Any, LabelFilter.Null)
           .Select(KeyFilter.Any, builder.Environment.EnvironmentName)
           .ConfigureRefresh(refresh =>
           {
               refresh.Register("Settings:Sentinel", refreshAll: true)
                      .SetRefreshInterval(TimeSpan.FromMinutes(5));
           });
});

var app = builder.Build();

// Enable dynamic refresh middleware
app.UseAzureAppConfiguration();
```

### Custom Configuration Provider

```csharp
public class DatabaseConfigurationSource(string connectionString) : IConfigurationSource
{
    public IConfigurationProvider Build(IConfigurationBuilder builder)
        => new DatabaseConfigurationProvider(connectionString);
}

public class DatabaseConfigurationProvider(string connectionString)
    : ConfigurationProvider
{
    public override void Load()
    {
        using var connection = new SqlConnection(connectionString);
        connection.Open();

        using var command = new SqlCommand(
            "SELECT [Key], [Value] FROM AppSettings WHERE IsActive = 1",
            connection);
        using var reader = command.ExecuteReader();

        var data = new Dictionary<string, string?>(
            StringComparer.OrdinalIgnoreCase);

        while (reader.Read())
        {
            data[reader.GetString(0)] = reader.GetString(1);
        }

        Data = data;
    }
}

// Extension method for clean registration
public static class DatabaseConfigurationExtensions
{
    public static IConfigurationBuilder AddDatabase(
        this IConfigurationBuilder builder, string connectionString)
    {
        builder.Add(new DatabaseConfigurationSource(connectionString));
        return builder;
    }
}

// Usage
builder.Configuration.AddDatabase(
    builder.Configuration.GetConnectionString("ConfigDb")!);
```

## Options Pattern

The Options pattern provides strongly-typed access to configuration sections. It supports validation at startup, change monitoring, and named options.

### Basic Binding with IOptions\<T\>

```csharp
public class SmtpOptions
{
    public const string SectionName = "SmtpSettings";

    public string Host { get; set; } = string.Empty;
    public int Port { get; set; } = 587;
    public bool EnableSsl { get; set; } = true;
    public string SenderEmail { get; set; } = string.Empty;
    public string? Password { get; set; }
}

// Registration in Program.cs
builder.Services.Configure<SmtpOptions>(
    builder.Configuration.GetSection(SmtpOptions.SectionName));

// Injection
public class EmailService(IOptions<SmtpOptions> options)
{
    private readonly SmtpOptions _smtp = options.Value;

    public async Task SendAsync(string to, string subject, string body)
    {
        using var client = new SmtpClient(_smtp.Host, _smtp.Port)
        {
            EnableSsl = _smtp.EnableSsl,
            Credentials = new NetworkCredential(_smtp.SenderEmail, _smtp.Password)
        };

        await client.SendMailAsync(_smtp.SenderEmail, to, subject, body);
    }
}
```

### IOptions vs IOptionsSnapshot vs IOptionsMonitor

| Interface | Lifetime | Reloads on Change | Use When |
|---|---|---|---|
| `IOptions<T>` | Singleton | No | Static config that never changes at runtime |
| `IOptionsSnapshot<T>` | Scoped | Yes (per request) | Config may change between HTTP requests |
| `IOptionsMonitor<T>` | Singleton | Yes (real-time) | Long-running services needing live updates |

### Validation with DataAnnotations

```csharp
using System.ComponentModel.DataAnnotations;

public class SmtpOptions
{
    public const string SectionName = "SmtpSettings";

    [Required(ErrorMessage = "SMTP host is required")]
    public string Host { get; set; } = string.Empty;

    [Range(1, 65535, ErrorMessage = "Port must be between 1 and 65535")]
    public int Port { get; set; } = 587;

    public bool EnableSsl { get; set; } = true;

    [Required, EmailAddress]
    public string SenderEmail { get; set; } = string.Empty;
}

// Registration with validation
builder.Services
    .AddOptions<SmtpOptions>()
    .Bind(builder.Configuration.GetSection(SmtpOptions.SectionName))
    .ValidateDataAnnotations()
    .ValidateOnStart(); // Fail fast at startup if invalid
```

### Custom Validation with IValidateOptions\<T\>

```csharp
public class SmtpOptionsValidator : IValidateOptions<SmtpOptions>
{
    public ValidateOptionsResult Validate(string? name, SmtpOptions options)
    {
        var failures = new List<string>();

        if (options.Port is 25 && options.EnableSsl)
        {
            failures.Add("Port 25 does not support SSL — use port 587 or 465.");
        }

        if (string.IsNullOrWhiteSpace(options.Host))
        {
            failures.Add("SMTP host must not be empty.");
        }

        if (!options.SenderEmail.Contains('@'))
        {
            failures.Add("SenderEmail must be a valid email address.");
        }

        return failures.Count > 0
            ? ValidateOptionsResult.Fail(failures)
            : ValidateOptionsResult.Success;
    }
}

// Registration
builder.Services.AddSingleton<IValidateOptions<SmtpOptions>,
    SmtpOptionsValidator>();
```

## Environment-Specific Configuration

ASP.NET Core uses the `ASPNETCORE_ENVIRONMENT` variable to determine the active environment. The three conventional environments are Development, Staging, and Production.

### Environment Detection Flow

```
┌──────────────────────────────────────────────────────────────┐
│              Environment Resolution Order                     │
│                                                              │
│  1. ASPNETCORE_ENVIRONMENT env var                           │
│  2. DOTNET_ENVIRONMENT env var (non-web hosts)               │
│  3. launchSettings.json (local development only)             │
│  4. Default: "Production" (when no variable is set)          │
│                                                              │
│  ┌────────────┐   ┌────────────┐   ┌──────────────────────┐ │
│  │ Development│   │  Staging   │   │     Production       │ │
│  │            │   │            │   │                      │ │
│  │ User       │   │ Staging    │   │ Env vars / vault     │ │
│  │ Secrets    │   │ config     │   │ Azure App Config     │ │
│  │ Dev        │   │ Reduced    │   │ Minimal logging      │ │
│  │ exceptions │   │ logging    │   │ No dev exceptions    │ │
│  │ Swagger    │   │ Pre-prod   │   │ No Swagger           │ │
│  │ Hot reload │   │ checks     │   │ HTTPS enforced       │ │
│  └────────────┘   └────────────┘   └──────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

### launchSettings.json

This file is used only during local development (`dotnet run`) and is **not** deployed.

```json
{
  "profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "applicationUrl": "http://localhost:5000",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "https": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "applicationUrl": "https://localhost:5001;http://localhost:5000",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

### Environment-Specific appsettings

```csharp
// Automatically loaded by CreateBuilder:
//   appsettings.json                     ← always loaded
//   appsettings.{Environment}.json       ← overlays base settings

// appsettings.Development.json
// {
//   "Logging": {
//     "LogLevel": { "Default": "Debug" }
//   },
//   "ConnectionStrings": {
//     "DefaultConnection": "Server=localhost;Database=AppDb_Dev;..."
//   }
// }

// appsettings.Production.json
// {
//   "Logging": {
//     "LogLevel": { "Default": "Warning" }
//   }
// }
```

### Conditional Logic by Environment

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
    app.MapOpenApi();
    app.UseSwaggerUI();
}

if (app.Environment.IsProduction())
{
    app.UseExceptionHandler("/error");
    app.UseHsts();
}

// Custom environment check
if (app.Environment.IsEnvironment("Staging"))
{
    app.UseMiddleware<StagingBannerMiddleware>();
}
```

## Docker

Containerizing .NET applications with Docker enables consistent builds and deployments across environments.

### Multi-Stage Dockerfile

```dockerfile
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# Copy csproj and restore (layer caching)
COPY ["src/MyApp.Api/MyApp.Api.csproj", "src/MyApp.Api/"]
COPY ["src/MyApp.Core/MyApp.Core.csproj", "src/MyApp.Core/"]
RUN dotnet restore "src/MyApp.Api/MyApp.Api.csproj"

# Copy everything and publish
COPY . .
WORKDIR /src/src/MyApp.Api
RUN dotnet publish -c Release -o /app/publish --no-restore

# Stage 2: Runtime
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser -s /bin/false appuser

COPY --from=build /app/publish .

# Set non-root user
USER appuser

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

### .dockerignore

```
**/.git
**/.vs
**/.vscode
**/bin
**/obj
**/node_modules
**/.dockerignore
**/Dockerfile*
**/*.md
**/docker-compose*.yml
.editorconfig
.gitignore
```

### Base Image Selection

| Image | Size | Use Case |
|---|---|---|
| `mcr.microsoft.com/dotnet/aspnet:10.0` | ~220 MB | ASP.NET Core web apps |
| `mcr.microsoft.com/dotnet/runtime:10.0` | ~190 MB | Console apps, worker services |
| `mcr.microsoft.com/dotnet/runtime-deps:10.0` | ~120 MB | Self-contained / Native AOT |
| `mcr.microsoft.com/dotnet/sdk:10.0` | ~900 MB | Build stage only (never deploy) |
| `mcr.microsoft.com/dotnet/aspnet:10.0-alpine` | ~110 MB | Smaller image, musl-based |
| `mcr.microsoft.com/dotnet/nightly/aspnet:10.0` | ~220 MB | Preview / nightly builds |

### Docker Compose

```yaml
# docker-compose.yml
services:
  api:
    build:
      context: .
      dockerfile: src/MyApp.Api/Dockerfile
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=db;Database=AppDb;User=sa;Password=YourStr0ng!Pass;TrustServerCertificate=true
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - MSSQL_SA_PASSWORD=YourStr0ng!Pass
    ports:
      - "1433:1433"
    volumes:
      - sql-data:/var/opt/mssql
    healthcheck:
      test: /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "YourStr0ng!Pass" -C -Q "SELECT 1" -b
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  sql-data:
```

### Container Best Practices

```
┌─────────────────────────────────────────────────────────────┐
│              Docker Best Practices for .NET                   │
│                                                              │
│  1. Use multi-stage builds (separate SDK from runtime)       │
│  2. Copy .csproj first → restore → copy source (caching)    │
│  3. Run as non-root user (USER appuser)                      │
│  4. Use .dockerignore to exclude build artifacts             │
│  5. Pin image tags (10.0 not latest)                         │
│  6. Set DOTNET_EnableDiagnostics=0 in production             │
│  7. Use health checks in docker-compose                      │
│  8. Use Alpine images when possible for smaller footprint    │
│  9. Never store secrets in the image — use env vars / vault  │
│ 10. Scan images for vulnerabilities (Trivy, Snyk)            │
└─────────────────────────────────────────────────────────────┘
```

## CI/CD Pipelines

### GitHub Actions for .NET

```yaml
# .github/workflows/dotnet-ci.yml
name: .NET CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  DOTNET_VERSION: "10.0.x"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --configuration Release

      - name: Test
        run: dotnet test --no-build --configuration Release --verbosity normal --collect:"XPlat Code Coverage" --results-directory ./coverage

      - name: Publish coverage
        uses: codecov/codecov-action@v4
        with:
          directory: ./coverage

  publish:
    needs: build-and-test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

  deploy:
    needs: publish
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy to Azure App Service
        uses: azure/webapps-deploy@v3
        with:
          app-name: myapp-api
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
```

### Azure DevOps Pipeline

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: "ubuntu-latest"

variables:
  buildConfiguration: "Release"
  dotnetVersion: "10.0.x"

stages:
  - stage: Build
    jobs:
      - job: BuildAndTest
        steps:
          - task: UseDotNet@2
            inputs:
              packageType: "sdk"
              version: $(dotnetVersion)

          - script: dotnet restore
            displayName: "Restore packages"

          - script: dotnet build --configuration $(buildConfiguration) --no-restore
            displayName: "Build"

          - script: dotnet test --configuration $(buildConfiguration) --no-build --logger trx --results-directory $(Agent.TempDirectory)
            displayName: "Run tests"

          - task: PublishTestResults@2
            inputs:
              testResultsFormat: "VSTest"
              testResultsFiles: "**/*.trx"
              searchFolder: $(Agent.TempDirectory)

          - script: dotnet publish src/MyApp.Api/MyApp.Api.csproj --configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)
            displayName: "Publish"

          - task: PublishBuildArtifacts@1
            inputs:
              pathToPublish: $(Build.ArtifactStagingDirectory)
              artifactName: "drop"

  - stage: Deploy
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployToProduction
        environment: "production"
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: "MyAzureSubscription"
                    appType: "webAppLinux"
                    appName: "myapp-api"
                    package: "$(Pipeline.Workspace)/drop/**/*.zip"
```

### Deploying to Kubernetes

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
  labels:
    app: myapp-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp-api
  template:
    metadata:
      labels:
        app: myapp-api
    spec:
      containers:
        - name: myapp-api
          image: ghcr.io/myorg/myapp-api:latest
          ports:
            - containerPort: 8080
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: "Production"
            - name: ConnectionStrings__DefaultConnection
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: db-connection-string
          livenessProbe:
            httpGet:
              path: /healthz/live
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /healthz/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-api
spec:
  selector:
    app: myapp-api
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

## Publishing and Deployment

.NET supports multiple publishing modes that trade off between deployment size, startup time, and runtime requirements.

### Publishing Modes Comparison

| Mode | Command | Requires Runtime | Output Size | Startup | Best For |
|---|---|---|---|---|---|
| Framework-Dependent | `dotnet publish -c Release` | Yes | ~5 MB | Fast | Servers with .NET runtime |
| Self-Contained | `dotnet publish -c Release --self-contained` | No | ~80 MB | Fast | Environments without .NET |
| Single-File | `dotnet publish -c Release --self-contained -p:PublishSingleFile=true` | No | ~70 MB | Moderate | Simplified deployment |
| Native AOT | `dotnet publish -c Release -p:PublishAot=true` | No | ~15 MB | Very fast | Serverless, containers |
| ReadyToRun | `dotnet publish -c Release -p:PublishReadyToRun=true` | Yes | ~8 MB | Faster | Reduced JIT at startup |
| Trimmed | `dotnet publish -c Release --self-contained -p:PublishTrimmed=true` | No | ~30 MB | Fast | Smaller self-contained |

### Framework-Dependent Deployment

```xml
<!-- MyApp.Api.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
  </PropertyGroup>
</Project>
```

```bash
dotnet publish -c Release -o ./publish
# Output: MyApp.Api.dll + dependencies
# Run with: dotnet MyApp.Api.dll
```

### Self-Contained Deployment

```bash
# Publish for Linux x64
dotnet publish -c Release --self-contained -r linux-x64 -o ./publish

# Publish for Windows x64
dotnet publish -c Release --self-contained -r win-x64 -o ./publish
```

### Native AOT

Native AOT compiles directly to native code, eliminating the JIT compiler for faster startup and smaller memory footprint.

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <PublishAot>true</PublishAot>
  </PropertyGroup>
</Project>
```

```bash
dotnet publish -c Release -r linux-x64
# Output: single native binary (~15-25 MB)
```

### Native AOT Dockerfile

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish src/MyApp.Api/MyApp.Api.csproj \
    -c Release -r linux-x64 -o /app/publish

# Use runtime-deps (no .NET runtime needed for AOT)
FROM mcr.microsoft.com/dotnet/runtime-deps:10.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .
USER $APP_UID
ENTRYPOINT ["./MyApp.Api"]
```

## Health Checks

Health checks expose the application's health status for load balancers, orchestrators, and monitoring systems.

### Health Check Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    Health Check System                        │
│                                                              │
│  GET /healthz/live ──→ Liveness Probe                        │
│    "Is the process alive and not deadlocked?"                │
│    Response: 200 Healthy | 503 Unhealthy                     │
│                                                              │
│  GET /healthz/ready ──→ Readiness Probe                      │
│    "Can the app serve traffic? Are dependencies available?"  │
│    Checks: Database, Redis, External APIs                    │
│    Response: 200 Healthy | 503 Unhealthy                     │
│                                                              │
│  GET /healthz ──→ Full Health Report                         │
│    Returns detailed JSON with all check results              │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   SQL Server  │  │    Redis     │  │  External API│       │
│  │   ✅ 12ms    │  │   ✅ 3ms     │  │  ❌ Timeout  │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└──────────────────────────────────────────────────────────────┘
```

### Basic Health Check Implementation

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"])
    .AddCheck<DatabaseHealthCheck>("database", tags: ["ready"])
    .AddCheck<RedisHealthCheck>("redis", tags: ["ready"])
    .AddCheck<ExternalApiHealthCheck>("external-api", tags: ["ready"]);

var app = builder.Build();

// Liveness — is the process alive?
app.MapHealthChecks("/healthz/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live")
});

// Readiness — can the app serve traffic?
app.MapHealthChecks("/healthz/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});

// Full health report with details
app.MapHealthChecks("/healthz", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

### Custom Health Checks

```csharp
public class DatabaseHealthCheck(IServiceScopeFactory scopeFactory) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            await using var scope = scopeFactory.CreateAsyncScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            await db.Database.CanConnectAsync(cancellationToken);

            return HealthCheckResult.Healthy("Database connection is healthy.");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                "Database connection failed.",
                exception: ex,
                data: new Dictionary<string, object>
                {
                    { "Error", ex.Message }
                });
        }
    }
}

public class RedisHealthCheck(IConnectionMultiplexer redis) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var db = redis.GetDatabase();
            var latency = await db.PingAsync();

            return latency.TotalMilliseconds < 100
                ? HealthCheckResult.Healthy($"Redis responding in {latency.TotalMilliseconds}ms.")
                : HealthCheckResult.Degraded($"Redis slow: {latency.TotalMilliseconds}ms.");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Redis unavailable.", exception: ex);
        }
    }
}

public class ExternalApiHealthCheck(IHttpClientFactory httpClientFactory) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var client = httpClientFactory.CreateClient("ExternalApi");
            var response = await client.GetAsync("/health", cancellationToken);

            return response.IsSuccessStatusCode
                ? HealthCheckResult.Healthy("External API is reachable.")
                : HealthCheckResult.Degraded(
                    $"External API returned {response.StatusCode}.");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("External API unreachable.", exception: ex);
        }
    }
}
```

### NuGet Health Check Packages

```csharp
// Popular AspNetCore.Diagnostics.HealthChecks packages
builder.Services.AddHealthChecks()
    .AddSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")!)
    .AddRedis(builder.Configuration.GetConnectionString("Redis")!)
    .AddRabbitMQ(builder.Configuration.GetConnectionString("RabbitMQ")!)
    .AddUrlGroup(new Uri("https://api.example.com/health"), "external-api");
```

## Logging and Diagnostics in Production

### Structured Logging with Serilog

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog((context, services, config) =>
{
    config
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .Enrich.WithMachineName()
        .Enrich.WithEnvironmentName()
        .Enrich.WithProperty("Application", "MyApp.Api")
        .WriteTo.Console(new RenderedCompactJsonFormatter())
        .WriteTo.Seq("http://seq:5341")
        .WriteTo.ApplicationInsights(
            TelemetryConverter.Traces);
});

var app = builder.Build();

// Request logging middleware
app.UseSerilogRequestLogging(options =>
{
    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("UserId",
            httpContext.User.FindFirstValue(ClaimTypes.NameIdentifier));
        diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
    };
});
```

### High-Performance Logging with Source Generators

```csharp
public static partial class LogMessages
{
    [LoggerMessage(Level = LogLevel.Information,
        Message = "Processing order {OrderId} for customer {CustomerId}")]
    public static partial void ProcessingOrder(
        this ILogger logger, int orderId, int customerId);

    [LoggerMessage(Level = LogLevel.Error,
        Message = "Order {OrderId} failed after {ElapsedMs}ms")]
    public static partial void OrderFailed(
        this ILogger logger, int orderId, long elapsedMs, Exception exception);

    [LoggerMessage(Level = LogLevel.Warning,
        Message = "Health check {CheckName} degraded: {Reason}")]
    public static partial void HealthCheckDegraded(
        this ILogger logger, string checkName, string reason);
}
```

### OpenTelemetry Integration

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource
        .AddService("MyApp.Api", serviceVersion: "1.0.0"))
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddOtlpExporter(opts =>
        {
            opts.Endpoint = new Uri("http://otel-collector:4317");
        }))
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddOtlpExporter(opts =>
        {
            opts.Endpoint = new Uri("http://otel-collector:4317");
        }));
```

### Application Insights

```csharp
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
});

// Custom telemetry
public class OrderService(TelemetryClient telemetry)
{
    public async Task<Order> CreateAsync(CreateOrderRequest request)
    {
        using var operation = telemetry.StartOperation<RequestTelemetry>("CreateOrder");

        try
        {
            var order = await ProcessOrder(request);
            telemetry.TrackEvent("OrderCreated", new Dictionary<string, string>
            {
                { "OrderId", order.Id.ToString() },
                { "CustomerId", request.CustomerId.ToString() }
            });
            return order;
        }
        catch (Exception ex)
        {
            telemetry.TrackException(ex);
            operation.Telemetry.Success = false;
            throw;
        }
    }
}
```

### Production Diagnostics Tools

| Tool | Purpose | Example Command |
|---|---|---|
| `dotnet-counters` | Real-time performance counters | `dotnet-counters monitor -p <PID>` |
| `dotnet-trace` | Collect runtime traces | `dotnet-trace collect -p <PID> --duration 00:00:30` |
| `dotnet-dump` | Capture and analyze memory dumps | `dotnet-dump collect -p <PID>` |
| `dotnet-gcdump` | GC heap analysis | `dotnet-gcdump collect -p <PID>` |
| `dotnet-stack` | Print managed stack traces | `dotnet-stack report -p <PID>` |

```bash
# Install global diagnostic tools
dotnet tool install -g dotnet-counters
dotnet tool install -g dotnet-trace
dotnet tool install -g dotnet-dump

# Monitor runtime counters in real-time
dotnet-counters monitor -p $(pidof MyApp.Api) \
  --counters System.Runtime,Microsoft.AspNetCore.Hosting

# Collect a 30-second trace
dotnet-trace collect -p $(pidof MyApp.Api) \
  --duration 00:00:30 \
  --output trace.nettrace

# Collect a memory dump
dotnet-dump collect -p $(pidof MyApp.Api) \
  --output dump.dmp

# Analyze dump
dotnet-dump analyze dump.dmp
```

## Feature Flags

Feature flags enable controlled rollout of new functionality without code deployments using the `Microsoft.FeatureManagement` library.

### Basic Feature Flags

```csharp
// appsettings.json
// {
//   "FeatureManagement": {
//     "NewDashboard": true,
//     "BetaSearch": false,
//     "PremiumFeature": {
//       "EnabledFor": [
//         {
//           "Name": "Percentage",
//           "Parameters": { "Value": 25 }
//         }
//       ]
//     }
//   }
// }

// Program.cs
builder.Services.AddFeatureManagement();

// Usage in a service
public class DashboardService(IFeatureManager featureManager)
{
    public async Task<DashboardModel> GetDashboardAsync()
    {
        if (await featureManager.IsEnabledAsync("NewDashboard"))
        {
            return await BuildNewDashboard();
        }

        return await BuildClassicDashboard();
    }
}
```

### Feature Gates on Controllers and Endpoints

```csharp
// Attribute-based gating on controllers
[FeatureGate("BetaSearch")]
[ApiController]
[Route("api/[controller]")]
public class SearchController : ControllerBase
{
    [HttpGet]
    public IActionResult Search([FromQuery] string query)
    {
        return Ok(new { results = Array.Empty<string>() });
    }
}

// Endpoint filter for minimal APIs
app.MapGet("/api/preview/recommendations", async (IFeatureManager fm) =>
{
    if (!await fm.IsEnabledAsync("Recommendations"))
    {
        return Results.NotFound();
    }

    return Results.Ok(new { items = new[] { "item1", "item2" } });
});
```

### Custom Feature Filters

```csharp
// Time window feature filter
[FilterAlias("TimeWindow")]
public class TimeWindowFilter : IFeatureFilter
{
    public Task<bool> EvaluateAsync(FeatureFilterEvaluationContext context)
    {
        var settings = context.Parameters.Get<TimeWindowSettings>();

        var now = DateTimeOffset.UtcNow;
        var isInWindow = now >= settings.Start && now <= settings.End;

        return Task.FromResult(isInWindow);
    }
}

public class TimeWindowSettings
{
    public DateTimeOffset Start { get; set; }
    public DateTimeOffset End { get; set; }
}

// Registration
builder.Services.AddFeatureManagement()
    .AddFeatureFilter<TimeWindowFilter>();
```

```json
{
  "FeatureManagement": {
    "HolidaySale": {
      "EnabledFor": [
        {
          "Name": "TimeWindow",
          "Parameters": {
            "Start": "2025-11-28T00:00:00Z",
            "End": "2025-12-02T23:59:59Z"
          }
        }
      ]
    }
  }
}
```

### A/B Testing with Targeting

```csharp
// appsettings.json
// {
//   "FeatureManagement": {
//     "NewCheckoutFlow": {
//       "EnabledFor": [
//         {
//           "Name": "Targeting",
//           "Parameters": {
//             "Audience": {
//               "Users": ["admin@example.com"],
//               "Groups": [
//                 { "Name": "BetaTesters", "RolloutPercentage": 100 }
//               ],
//               "DefaultRolloutPercentage": 10
//             }
//           }
//         }
//       ]
//     }
//   }
// }

// Implement targeting context
public class HttpTargetingContextAccessor(
    IHttpContextAccessor httpContextAccessor) : ITargetingContextAccessor
{
    public ValueTask<TargetingContext> GetContextAsync()
    {
        var httpContext = httpContextAccessor.HttpContext!;
        var user = httpContext.User;

        return new ValueTask<TargetingContext>(new TargetingContext
        {
            UserId = user.FindFirstValue(ClaimTypes.NameIdentifier) ?? "anonymous",
            Groups = user.Claims
                .Where(c => c.Type == ClaimTypes.Role)
                .Select(c => c.Value)
                .ToList()
        });
    }
}

// Registration
builder.Services.AddFeatureManagement()
    .WithTargeting<HttpTargetingContextAccessor>();
```

## Best Practices

### Production Deployment Checklist

- ✅ Use environment variables or a vault for secrets — never commit them to source control
- ✅ Validate all options at startup with `ValidateOnStart()` to fail fast on misconfiguration
- ✅ Use multi-stage Docker builds to keep runtime images small and secure
- ✅ Run containers as a non-root user
- ✅ Implement both liveness and readiness health checks for Kubernetes deployments
- ✅ Use structured logging (Serilog or OpenTelemetry) with correlation IDs
- ✅ Enable Native AOT or ReadyToRun for latency-sensitive workloads
- ✅ Pin Docker base image tags to specific versions (e.g., `10.0` not `latest`)
- ✅ Use feature flags to decouple deployments from feature releases
- ✅ Set `DOTNET_EnableDiagnostics=0` in production containers unless actively debugging
- ✅ Configure CI/CD pipelines to run tests before publishing artifacts
- ✅ Use deployment environments with approval gates for production releases
- ✅ Store connection strings in environment variables or Azure Key Vault

- ❌ Don't hardcode configuration values — use the Options pattern or `IConfiguration`
- ❌ Don't deploy the SDK image — use the runtime or `runtime-deps` base image
- ❌ Don't skip health checks — orchestrators need them for reliable routing
- ❌ Don't log sensitive data (passwords, tokens, PII) in any environment
- ❌ Don't use `appsettings.json` for secrets — it is committed to source control
- ❌ Don't ignore trimming warnings when using Native AOT — they indicate real issues
- ❌ Don't run production containers as root
- ❌ Don't skip `ValidateOnStart` — runtime failures from bad config are harder to diagnose

## Next Steps

Return to the [Learning Path](LEARNING-PATH.md) for the complete curriculum and additional topics. This document pairs with [Services](02-SERVICES.md) for building robust ASP.NET Core applications and [Security and Authentication](09-SECURITY-AND-AUTHENTICATION.md) for securing production deployments.

### Related Documents

1. **[Learning Path](LEARNING-PATH.md)** — Complete .NET learning curriculum
2. **[Services](02-SERVICES.md)** — Web APIs, worker services, gRPC, and health checks
3. **[Dependency Injection](06-DEPENDENCY-INJECTION.md)** — DI container, Options pattern, service lifetimes
4. **[Security and Authentication](09-SECURITY-AND-AUTHENTICATION.md)** — Authentication, authorization, HTTPS
5. **[Testing](08-TESTING.md)** — Unit testing, integration testing, WebApplicationFactory
6. **[Best Practices](03-BEST-PRACTICES.md)** — Async/await, memory, security, testing

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial configuration and deployment documentation |
