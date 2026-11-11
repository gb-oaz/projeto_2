---
name: devops-azure
description: DevOps: GitHub Actions, Docker, Docker Compose, Terraform, Azure Container Apps, CI/CD, observabilidade, custos
metadata:
  type: reference
  category: devops
  agents: [deboaz-arc, deboaz-backend, deboaz-frontend, deboaz-qa]
---

# Skill: DevOps e Infraestrutura (Azure)

## Responsabilidade e Escopo

Competência de referência para práticas de DevOps, CI/CD, infraestrutura como código, containerização, deploy no Azure, observabilidade, monitoramento, logs, gestão de custos e segurança.

## Stack Principal

| Categoria | Tecnologias |
|---|---|
| **Source Control** | Git, GitHub, GitHub Actions |
| **Containerização** | Docker, Docker Compose, Multi-stage builds |
| **IaC** | Terraform, Azure Bicep (alternativa) |
| **Cloud** | Microsoft Azure, Azure Container Apps, Azure Container Registry |
| **Orquestração** | Azure Container Apps (serverless containers), Dapr (sidecar) |
| **Banco de Dados** | Azure Cosmos DB (MongoDB API), Azure Cache for Redis |
| **Observabilidade** | Azure Monitor, Application Insights, Log Analytics, Grafana |
| **CI/CD** | GitHub Actions, Environments (dev/staging/prod), Approval gates |
| **Segurança** | Azure Key Vault, Managed Identities, Microsoft Defender for Cloud |
| **Custos** | Azure Cost Management, Budgets, Advisor |

## 1. Docker & Docker Compose

### Multi-stage Dockerfile (Backend .NET)
```dockerfile
# Dockerfile.backend
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["src/Arteris.Backend/Arteris.Backend.csproj", "Arteris.Backend/"]
COPY ["src/Arteris.Domain/Arteris.Domain.csproj", "Arteris.Domain/"]
COPY ["src/Arteris.Application/Arteris.Application.csproj", "Arteris.Application/"]
COPY ["src/Arteris.Infrastructure/Arteris.Infrastructure.csproj", "Arteris.Infrastructure/"]
RUN dotnet restore "Arteris.Backend/Arteris.Backend.csproj"
COPY . .
WORKDIR "/src/Arteris.Backend"
RUN dotnet publish -c Release -o /app/publish /p:UseAppHost=false

# Stage 2: Runtime
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_ENVIRONMENT=Production
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "Arteris.Backend.dll"]
```

### Multi-stage Dockerfile (Frontend Vue + Vite)
```dockerfile
# Dockerfile.frontend
# Stage 1: Build
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Nginx Runtime
FROM nginx:alpine AS final
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Docker Compose (Desenvolvimento Local)
```yaml
# docker-compose.yml
version: '3.8'

services:
  # Infraestrutura
  mongodb:
    image: mongo:7
    ports: ["27017:27017"]
    volumes: [mongo_data:/data/db]
    environment:
      MONGO_INITDB_DATABASE: arteris
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh localhost:27017/test --quiet
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    volumes: [redis_data:/data]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  # Aplicações
  backend:
    build:
      context: .
      dockerfile: 5_DEV/backend/Dockerfile
    ports: ["8080:8080"]
    environment:
      ASPNETCORE_ENVIRONMENT: Development
      ConnectionStrings__MongoDB: mongodb://mongodb:27017
      ConnectionStrings__Redis: redis:6379
      JwtSettings__SecretKey: ${JWT_SECRET_KEY}
    depends_on:
      mongodb:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./5_DEV/backend:/src:cached

  frontend:
    build:
      context: .
      dockerfile: 5_DEV/frontend/Dockerfile
    ports: ["3000:80"]
    environment:
      VITE_API_BASE_URL: http://localhost:8080/api
    depends_on: [backend]
    volumes:
      - ./5_DEV/frontend:/app:cached

volumes:
  mongo_data:
  redis_data:
```

## 2. Terraform (Infrastructure as Code)

### Estrutura de Módulos
```
4_CI_CD/terraform/
├── main.tf                 # Root module
├── variables.tf
├── outputs.tf
├── providers.tf
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── terraform.tfvars
│   │   └── backend.hcl
│   ├── staging/
│   └── prod/
└── modules/
    ├── resource-group/
    ├── container-apps-env/
    ├── container-app/
    ├── cosmos-db-mongodb/
    ├── redis-cache/
    ├── container-registry/
    ├── key-vault/
    ├── log-analytics/
    ├── application-insights/
    └── managed-identity/
```

### Exemplo: Azure Container App Module
```hcl
# modules/container-app/main.tf
resource "azurerm_container_app" "main" {
  name                = var.name
  resource_group_name = var.resource_group_name
  container_app_environment_id = var.container_app_environment_id
  
  configuration {
    active_revision_mode = "Single"
    ingress {
      external_enabled = var.external_enabled
      target_port      = var.target_port
      transport        = "http"
      allow_insecure   = false
    }
    registries {
      server   = var.acr_login_server
      identity = "system"
    }
    secrets = [
      { name = "jwt-secret", value = var.jwt_secret }
      { name = "mongodb-conn", value = var.mongodb_connection_string }
      { name = "redis-conn", value = var.redis_connection_string }
    ]
  }

  template {
    container {
      name  = var.name
      image = var.image
      cpu   = var.cpu
      memory = var.memory
      env = [
        { name = "ASPNETCORE_ENVIRONMENT", value = var.environment },
        { name = "ConnectionStrings__MongoDB", secret_ref = "mongodb-conn" },
        { name = "ConnectionStrings__Redis", secret_ref = "redis-conn" },
        { name = "JwtSettings__SecretKey", secret_ref = "jwt-secret" }
      ]
      probes {
        liveness_probe {
          http_get { path = "/health/live", port = var.target_port }
          initial_delay_seconds = 10
          period_seconds = 10
        }
        readiness_probe {
          http_get { path = "/health/ready", port = var.target_port }
          initial_delay_seconds = 5
          period_seconds = 10
        }
      }
    }
    scale {
      min_replicas = var.min_replicas
      max_replicas = var.max_replicas
      rule {
        name = "http-scaling"
        http {
          metadata = { concurrent_requests = "50" }
        }
      }
    }
  }
  
  identity {
    type = "SystemAssigned"
  }
}
```

## 3. GitHub Actions (CI/CD)

### Pipeline Backend
```yaml
# .github/workflows/backend-ci.yml
name: Backend CI

on:
  push:
    branches: [main, develop]
    paths: ['5_DEV/backend/**', '4_CI_CD/**']
  pull_request:
    branches: [main]
    paths: ['5_DEV/backend/**']

env:
  REGISTRY: ${{ secrets.AZURE_CONTAINER_REGISTRY }}
  IMAGE_NAME: arteris-backend

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with: { dotnet-version: '8.0.x' }
      
      - name: Restore
        run: dotnet restore 5_DEV/backend/Arteris.Backend.sln
      
      - name: Build
        run: dotnet build 5_DEV/backend/Arteris.Backend.sln --no-restore -c Release
      
      - name: Unit Tests
        run: dotnet test 5_DEV/backend/tests/Arteris.UnitTests --no-build -c Release --collect:"XPlat Code Coverage"
      
      - name: Integration Tests (Testcontainers)
        run: dotnet test 5_DEV/backend/tests/Arteris.IntegrationTests --no-build -c Release
        env:
          TESTCONTAINERS_RYUK_DISABLED: true
      
      - name: Upload Coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/**/coverage.cobertura.xml
          flags: backend

  docker:
    needs: build-and-test
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to ACR
        uses: azure/docker-login@v1
        with:
          login-server: ${{ env.REGISTRY }}
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix=
            type=raw,value=latest,enable={{is_default_branch}}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: 5_DEV/backend
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-dev:
    needs: docker
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: development
    steps:
      - name: Deploy to Azure Container Apps (Dev)
        uses: azure/container-apps-deploy-action@v1
        with:
          resourceGroup: ${{ secrets.AZURE_RG_DEV }}
          containerAppName: arteris-backend-dev
          imageToDeploy: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          acrName: ${{ secrets.AZURE_ACR_NAME }}

  deploy-staging:
    needs: docker
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to Azure Container Apps (Staging)
        uses: azure/container-apps-deploy-action@v1
        with:
          resourceGroup: ${{ secrets.AZURE_RG_STAGING }}
          containerAppName: arteris-backend-staging
          imageToDeploy: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          acrName: ${{ secrets.AZURE_ACR_NAME }}

  deploy-prod:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy to Azure Container Apps (Prod)
        uses: azure/container-apps-deploy-action@v1
        with:
          resourceGroup: ${{ secrets.AZURE_RG_PROD }}
          containerAppName: arteris-backend-prod
          imageToDeploy: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          acrName: ${{ secrets.AZURE_ACR_NAME }}
```

### Pipeline Frontend
```yaml
# .github/workflows/frontend-ci.yml
name: Frontend CI

on:
  push:
    branches: [main, develop]
    paths: ['5_DEV/frontend/**', '4_CI_CD/**']
  pull_request:
    branches: [main]
    paths: ['5_DEV/frontend/**']

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node
        uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm', cache-dependency-path: '5_DEV/frontend/package-lock.json' }
      
      - name: Install
        run: cd 5_DEV/frontend && npm ci
      
      - name: Lint
        run: cd 5_DEV/frontend && npm run lint
      
      - name: Type Check
        run: cd 5_DEV/frontend && npm run type-check
      
      - name: Unit Tests
        run: cd 5_DEV/frontend && npm run test:unit -- --coverage
      
      - name: Component Tests
        run: cd 5_DEV/frontend && npm run test:component
      
      - name: Upload Coverage
        uses: codecov/codecov-action@v3
        with:
          flags: frontend
          files: ./5_DEV/frontend/coverage/**/coverage.xml

  e2e:
    needs: build-and-test
    runs-on: ubuntu-latest
    services:
      backend:
        image: ${{ secrets.AZURE_CONTAINER_REGISTRY }}/arteris-backend:latest
        ports: [8080:8080]
        env:
          ConnectionStrings__MongoDB: mongodb://mongodb:27017
          ConnectionStrings__Redis: redis:6379
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node
        uses: actions/setup-node@v4
        with: { node-version: '20' }
      - name: Install
        run: cd 5_DEV/frontend && npm ci
      - name: Build
        run: cd 5_DEV/frontend && npm run build
      - name: Run E2E Tests
        uses: cypress-io/github-action@v6
        with:
          build: npm run build
          start: npm run preview
          working-directory: 5_DEV/frontend
        env:
          CYPRESS_BASE_URL: http://localhost:4173
          CYPRESS_API_URL: http://localhost:8080/api

  deploy-preview:
    needs: build-and-test
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Azure Static Web Apps (Preview)
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_SWA_TOKEN }}
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          action: "upload"
          app_location: "5_DEV/frontend"
          output_location: "dist"
          environment_name: preview
```

## 4. Observabilidade

### Application Insights + OpenTelemetry (.NET)
```csharp
// Program.cs
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddMongoDBInstrumentation()
        .AddRedisInstrumentation()
        .AddSource("Arteris.*")
        .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("Arteris.Backend"))
        .AddOtlpExporter(o => o.Endpoint = new Uri(builder.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]))
    )
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddMeter("Arteris.*")
        .AddOtlpExporter()
    );

// Structured Logging com Serilog
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithProperty("Application", "Arteris.Backend")
    .Enrich.WithProperty("Environment", builder.Environment.EnvironmentName)
    .WriteTo.Console(outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}")
    .WriteTo.ApplicationInsights(builder.Configuration["APPLICATIONINSIGHTS_CONNECTION_STRING"], TelemetryConverter.Traces)
    .CreateLogger();
```

### Health Checks
```csharp
builder.Services.AddHealthChecks()
    .AddMongoDb(sp => sp.GetRequiredService<IMongoDatabase>().Client, name: "mongodb")
    .AddRedis(sp => sp.GetRequiredService<IConnectionMultiplexer>(), name: "redis")
    .AddCheck<ExternalApiHealthCheck>("external-api")
    .AddCheck<DiskSpaceHealthCheck>("disk-space");

// Endpoints
app.MapHealthChecks("/health/live", new HealthCheckOptions 
{ 
    Predicate = _ => false, // Apenas liveness (processo vivo)
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse 
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions 
{ 
    Predicate = check => check.Tags.Contains("ready"), // Readiness (dependências)
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse 
});
```

### Dashboards (Grafana + Azure Monitor)
- **Request Rate/Latency/Errors (RED)** por endpoint
- **Container App**: CPU, Memory, Replicas, HTTP 5xx
- **MongoDB**: Connections, Ops/sec, Latency, Cache hit ratio
- **Redis**: Memory, Hit rate, Connected clients, Evictions
- **Business**: Pedidos/hora, Taxa de conversão, Erros de validação

## 5. Segurança

### Azure Key Vault + Managed Identity
```hcl
# Terraform - Key Vault Access Policy para Container App
resource "azurerm_key_vault_access_policy" "container_app" {
  key_vault_id = azurerm_key_vault.main.id
  object_id    = azurerm_container_app.main.identity[0].principal_id
  secret_permissions = ["get", "list"]
}
```

```csharp
// Program.cs - Configuração
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{keyVaultName}.vault.azure.net/"),
    new DefaultAzureCredential());
```

### Segurança de Pipeline
- **Dependabot** para updates de dependências
- **CodeQL** para SAST
- **Trivy** para container scanning
- **OWASP ZAP** para DAST em staging
- **Policy as Code** (OPA/Gatekeeper) para admission control

## 6. Gestão de Custos

### Estratégias
- **Azure Container Apps**: Scale to zero (dev/staging), pay-per-use
- **Cosmos DB**: Serverless (dev) / Provisioned throughput com autoscale (prod)
- **Redis**: Basic (dev) / Premium com clustering (prod)
- **Budgets & Alertas**: $/mês por environment, alertas em 80%/100%
- **Azure Advisor**: Recomendações de right-sizing, reserved instances
- **Tagging**: `environment`, `team`, `project`, `cost-center` em todos recursos

## Boas Práticas

1. **GitOps**: Terraform state em backend remoto (Azure Storage), PR-based changes
2. **Environments**: dev → staging → prod com approval gates
3. **Immutable tags**: SHA para prod, `latest` apenas dev
4. **Secrets em Key Vault**: Nunca em repo, nem em variáveis de pipeline
5. **Health checks** em todos containers (liveness + readiness)
6. **Observabilidade nativa**: OpenTelemetry → Azure Monitor/Grafana
7. **Cost-aware**: Scale to zero em non-prod, autoscale em prod
8. **Security by default**: Managed Identity, least privilege, container scanning

## Critérios de Qualidade

- [ ] Pipeline CI < 10 min (build + unit + integration)
- [ ] Deploy automatizado dev/staging, manual approval prod
- [ ] Rollback < 5 min (revision anterior no Container Apps)
- [ ] 100% secrets em Key Vault (zero no repo)
- [ ] Health checks respondendo em < 2s
- [ ] Dashboards RED + Business atualizados
- [ ] Alertas configurados (latência, erro, custo, saúde)
- [ ] Cost anomaly detection ativo
- [ ] Security scanning em todo PR (SAST, Container, DAST staging)

## Referências

- `csharp-dotnet` — health checks, OpenTelemetry, Dockerfile
- `desenvolvimento-frontend` — Dockerfile, build, preview deploy
- `arquitetura-hexagonal` — configuração de adapters via DI
- `estrategia-qa` — pipeline quality gates