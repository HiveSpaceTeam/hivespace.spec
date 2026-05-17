# Deployment Guide

## Local Development Infrastructure

All local dependencies run via Docker Compose. The compose file is at:
`hivespace.config/docker/docker-compose.yml`

### Start All Services

```bash
cd hivespace.config/docker
docker compose up -d
```

### Services Started

| Container | Image | Port(s) | Purpose |
|-----------|-------|---------|---------|
| mssql-server | mcr.microsoft.com/mssql/server:2022-latest | 1433 | SQL Server (all services) |
| messagebroker | rabbitmq:4-management | 5672 (AMQP), 15672 (UI) | RabbitMQ — MassTransit transport |
| redis | redis | 6379 | Cache (StackExchange.Redis) |
| servicebus-emulator | azure-messaging/servicebus-emulator | 5673 | Azure Service Bus emulator |
| azurite | azure-storage/azurite | 10000 (Blob), 10001 (Queue), 10002 (Table) | Azure Storage emulator |

**Credentials:**
- SQL Server: `sa` / `Passw0rd123!`
- RabbitMQ: `guest` / `guest`

---

## CI/CD Pipelines (Backend)

GitHub Actions workflows live at `hivespace.microservice/.github/workflows/`. Each service has its own pipeline triggered on pushes to `master` for its relevant paths.

| Workflow | File | Trigger Paths |
|----------|------|---------------|
| API Gateway CI | `api-gateway-ci.yml` | `infra/api-gateway.bicep`, `infra/parameters/api-gateway-dev.json` |
| Catalog Service CI | `catalog-service-ci.yml` | Catalog service source |
| Media Service API CI | `media-service-api-ci.yml` | Media service API source |
| Media Service Function CI | `media-service-func-ci.yml` | Media Azure Function source |
| User Service CI | `user-service-ci.yml` | User service source |

### Deployment Targets (Cloud)

| Service | Azure Resource Type |
|---------|-------------------|
| UserService | Azure Container App |
| CatalogService | Azure App Service (Web App) |
| MediaService | Azure Function App (Flex Consumption) |
| API Gateway | Azure API Management (deployed via Bicep) |

**Bicep templates** are in `hivespace.microservice/infra/`:
- `api-gateway.bicep` — Azure API Management configuration
- `parameters/api-gateway-dev.json` — Dev environment parameters

**Secrets** used by CI:
- `AZURE_CREDENTIALS_SERVICE` — Service principal for `az login`
- `KEY_VAULT_NAME` / `CERT_SECRET_NAME` — Custom domain certificate
- Various `AZURE_CONTAINER_APP_*_NAME`, `AZURE_WEBAPP_*_NAME`, `AZURE_FUNCTION_APP_*_NAME` — Resource names per environment
- `AZURE_RESOURCE_GROUP` — Resource group

### API Gateway CI Flow

1. Checkout code
2. Azure login via service principal
3. Fetch backend service FQDNs from Azure (Container Apps, App Service, Function App)
4. Deploy API Gateway via `az deployment group create` with Bicep template
5. Injects backend URLs and custom domain cert from Key Vault into Bicep parameters

---

## CI/CD Pipelines (Frontend)

The web repo (`hivespace.web`) does not have GitHub Actions workflows in the repo itself. Per the project context, frontend apps are deployed to **Azure Static Web Apps** via GitHub Actions (deploy on push to `master` or release branches). The workflow configuration is likely managed at the Azure portal or parent `.github/` level.

---

## Backend Service Startup Order (Local Dev)

Run infrastructure containers first, then services in this order (each can run independently once DB + broker are up):

1. `docker compose up -d` (SQL Server, RabbitMQ, Redis, Service Bus Emulator, Azurite)
2. Apply EF Core migrations for each service (see `development-guide-microservice.md`)
3. Start UserService first (other services may need JWT validation config from it)
4. Start remaining services in any order

---

## Environment Configuration

Each backend service reads from `appsettings.json` and `appsettings.{Environment}.json`. Key connection strings to configure for local development:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost,1433;Database=HiveSpace{Service}Db;User Id=sa;Password=Passw0rd123!;TrustServerCertificate=True"
  },
  "RabbitMq": {
    "Host": "localhost",
    "Port": 5672,
    "Username": "guest",
    "Password": "guest"
  },
  "Redis": {
    "ConnectionString": "localhost:6379"
  }
}
```

### Frontend Environment Variables

Create `.env` in each app directory (`apps/admin/.env`, `apps/seller/.env`, `apps/buyer/.env`):

```env
VITE_APP_CLIENT_ID=<your-oidc-client-id>
VITE_GATEWAY_BASE_URL=https://localhost:7001
VITE_APP_REDIRECT_URI=http://localhost:{PORT}/callback/login
VITE_APP_POST_LOGOUT_REDIRECT_URI=http://localhost:{PORT}/callback/logout
VITE_APP_SCOPE=openid profile email offline_access
VITE_APP_ENVIRONMENT=development
VITE_ENABLE_LOGGING=true
VITE_ENABLE_DEBUG=true
```

Port per app: admin=5173, seller=5174, buyer=5175.

Reference `.env.development` files already present in each app directory as a template.

---

_Last generated: 2026-05-16_
