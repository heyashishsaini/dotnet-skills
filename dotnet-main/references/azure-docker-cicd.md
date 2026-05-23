# Azure, Docker & CI/CD Reference

## Docker — Production Dockerfile
```dockerfile
# Multi-stage build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["MyApp.API/MyApp.API.csproj", "MyApp.API/"]
RUN dotnet restore "MyApp.API/MyApp.API.csproj"
COPY . .
RUN dotnet publish "MyApp.API/MyApp.API.csproj" -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .
ENV ASPNETCORE_ENVIRONMENT=Production
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.API.dll"]
```

## docker-compose (local dev)
```yaml
services:
  api:
    build: .
    ports: ["5000:8080"]
    environment:
      - ConnectionStrings__Default=Server=db;Database=MyApp;...
    depends_on: [db]
  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      SA_PASSWORD: "YourStrong!Passw0rd"
      ACCEPT_EULA: "Y"
    ports: ["1433:1433"]
```

## Azure Services Map
| Need | Azure Service |
|---|---|
| Host API | App Service / Container Apps |
| Database | Azure SQL Database |
| Secrets | Key Vault |
| File storage | Blob Storage |
| Queue | Service Bus / Storage Queue |
| Serverless | Azure Functions |
| CDN/Static | Static Web Apps |
| Monitoring | Application Insights |

## Azure Key Vault Integration
```csharp
// Program.cs — never put secrets in appsettings.json!
if (builder.Environment.IsProduction())
{
    builder.Configuration.AddAzureKeyVault(
        new Uri($"https://{vaultName}.vault.azure.net/"),
        new DefaultAzureCredential());
}
```

## CI/CD — GitHub Actions
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
        with: { dotnet-version: '8.x' }
      - run: dotnet restore
      - run: dotnet build --no-restore -c Release
      - run: dotnet test --no-build -c Release
      - run: dotnet publish -c Release -o publish
      - uses: azure/webapps-deploy@v3
        with:
          app-name: my-api
          publish-profile: ${{ secrets.AZURE_PUBLISH_PROFILE }}
          package: publish
```

## Application Insights (Logging in Azure)
```csharp
builder.Services.AddApplicationInsightsTelemetry(
    builder.Configuration["ApplicationInsights:ConnectionString"]);
```

## Common Mistakes
1. Committing secrets to git (use Key Vault or environment variables)
2. Running as root in Docker containers
3. Not using multi-stage builds (bloated images)
4. Deploying without health checks (`/health` endpoint)
5. No rollback strategy in CI/CD pipeline
