# Deployment Patterns for .NET on Azure

## Multi-Stage Dockerfile

Always use multi-stage builds. Never ship the SDK in your production image.

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["src/YourApp.API/YourApp.API.csproj", "src/YourApp.API/"]
COPY ["src/YourApp.Application/YourApp.Application.csproj", "src/YourApp.Application/"]
# ... copy other project files for layer caching
RUN dotnet restore "src/YourApp.API/YourApp.API.csproj"
COPY . .
RUN dotnet publish "src/YourApp.API/YourApp.API.csproj" -c Release -o /app/publish

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "YourApp.API.dll"]
```

Copy `.csproj` files first and restore before copying source — this leverages Docker layer
caching so a restore doesn't re-run on every code change.

## EF Core Migration Strategy

**Never run `dotnet ef database update` in production from CI.** Use one of:

**Migration bundles (recommended):**
```bash
dotnet ef migrations bundle --self-contained -r linux-x64 -o ./migrationbundle
# Run in deployment pipeline before app starts
./migrationbundle --connection "..."
```

**SQL scripts (most control):**
```bash
dotnet ef migrations script --idempotent -o migrations.sql
# Review, then apply via your DBA or deployment pipeline
```

**On startup (only for dev/small apps):**
```csharp
// In Program.cs — acceptable for dev, risky for prod at scale
using var scope = app.Services.CreateScope();
var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
await db.Database.MigrateAsync();
```

The startup migration approach fails if you have multiple instances starting simultaneously —
they'll all try to migrate at once. Use a distributed lock or an init container instead.

## Azure App Service vs. AKS

**Azure App Service:**
- Start here. Simpler, cheaper, managed TLS, built-in deployment slots (blue/green).
- Deployment slots let you swap staging → production with zero downtime.
- Scale out with auto-scale rules based on CPU/memory.
- Limitations: Windows or Linux containers, no pod-level networking control.

**Azure Kubernetes Service (AKS):**
- Use when: multiple services, complex networking, need sidecar patterns, or you're > 5 services.
- More operational overhead. Need to manage ingress, cert-manager, horizontal pod autoscaler.
- Enables canary deployments, circuit breakers via service mesh.

Start with App Service. Migrate to AKS when App Service becomes the bottleneck.

## Secrets Management

**Never store secrets in appsettings.json or environment variables in code.**

Production secret hierarchy:
1. Azure Key Vault (authoritative store)
2. ASP.NET Core User Secrets (local dev only)
3. Environment variables (injected at runtime from Key Vault via App Service config)

```csharp
// Program.cs — pull secrets from Key Vault at startup
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{keyVaultName}.vault.azure.net/"),
    new DefaultAzureCredential());
```

Use Managed Identity so your app authenticates to Key Vault without storing a credential.

## CI/CD Pipeline (GitHub Actions)

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      - run: dotnet restore
      - run: dotnet build --no-restore -c Release
      - run: dotnet test --no-build -c Release
      - run: dotnet publish src/YourApp.API -c Release -o ./publish

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: azure/webapps-deploy@v3
        with:
          app-name: your-app-name
          slot-name: staging
          publish-profile: ${{ secrets.AZURE_PUBLISH_PROFILE }}
          package: ./publish
      # After smoke tests, swap staging → production
      - uses: azure/cli@v2
        with:
          inlineScript: az webapp deployment slot swap -g rg-yourapp -n your-app-name --slot staging
```

## Environment Promotion Strategy

dev → staging → production

- **dev**: Feature branches auto-deploy to a dev slot. No real data.
- **staging**: Mirrors production config. Used for QA and smoke testing. Uses anonymized prod data snapshot.
- **production**: Only deployable from `main`. Requires passing staging gate.

Feature flags (Azure App Configuration) let you deploy code to production before it's "on" — decouple deployment from release.

## Health Checks

```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>()
    .AddRedis(config.GetConnectionString("Redis"))
    .AddAzureServiceBusTopic(config["ServiceBus:ConnectionString"], topicName: "orders");

app.MapHealthChecks("/health");
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

Configure your load balancer to use `/health/ready` for traffic routing decisions.
Liveness (`/health`) and readiness (`/health/ready`) serve different purposes — don't conflate them.
