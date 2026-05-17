# Development Guide — HiveSpace Microservices

Repository root: `hivespace.microservice/`

---

## Prerequisites

| Tool | Version | Notes |
|---|---|---|
| .NET SDK | 8.0.x | `global.json` pins SDK. Run `dotnet --version` to verify. |
| SQL Server | 2019+ | Default: `localhost:1433`, sa / `Passw0rd123!` |
| RabbitMQ | 3.x | Default: `localhost:5672`, guest/guest |
| Redis | 6+ | `localhost:6379` — NotificationService only |
| Node.js | Optional | Not needed for microservices |
| Docker | Optional | Recommended for infrastructure |
| Azure Functions Core Tools | v4 | For running `HiveSpace.MediaService.Func` locally |
| Azurite | Latest | Azure Storage emulator for MediaService |

---

## Local Infrastructure Setup

The fastest path is to run SQL Server, RabbitMQ, and Redis via Docker:

```bash
# SQL Server
docker run -d --name sqlserver \
  -e SA_PASSWORD=Passw0rd123! \
  -e ACCEPT_EULA=Y \
  -p 1433:1433 \
  mcr.microsoft.com/mssql/server:2022-latest

# RabbitMQ with management UI
docker run -d --name rabbitmq \
  -p 5672:5672 -p 15672:15672 \
  rabbitmq:3-management

# Redis
docker run -d --name redis \
  -p 6379:6379 \
  redis:7-alpine

# Azurite (Azure Storage emulator for MediaService)
docker run -d --name azurite \
  -p 10000:10000 -p 10001:10001 -p 10002:10002 \
  mcr.microsoft.com/azure-storage/azurite
```

RabbitMQ management UI: http://localhost:15672 (guest/guest)

---

## Building

All packages managed centrally in `Directory.Packages.props`. Do not add `Version=` attributes to individual `.csproj` files.

```bash
# From repo root — restore all projects
dotnet restore

# Build entire solution
dotnet build hivespace.microservice.sln

# Build a single service
dotnet build src/HiveSpace.OrderService/HiveSpace.OrderService.Api/HiveSpace.OrderService.Api.csproj
```

Target framework: `net8.0` for all services.

---

## Running Each Service

All services auto-run EF migrations and seed data in the Development environment. Start services in dependency order: UserService first (IdentityServer), then downstream services.

### 1. API Gateway (port 5000)

```bash
cd src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway
dotnet run
# or: dotnet run --urls "http://localhost:5000"
```

No database required. Reads `appsettings.json` for YARP route config.

### 2. UserService (port 5001)

```bash
cd src/HiveSpace.UserService/HiveSpace.UserService.Api
dotnet run
```

Requires: SQL Server (`UserDb`), RabbitMQ, Kafka (optional).
Seeds: SystemAdmin, test sellers, and test buyers via `DataSeeder.EnsureSeedDataAsync`.
See `src/HiveSpace.UserService/SEEDED_ACCOUNTS.md` for seeded credentials.

### 3. CatalogService (port 5002)

```bash
cd src/HiveSpace.CatalogService/HiveSpace.CatalogService.Api
dotnet run
```

Requires: SQL Server (`CatalogDb`), RabbitMQ.
Seeds: Categories, attributes, sample products.

### 4. MediaService API (port 5003)

```bash
cd src/HiveSpace.MediaService/HiveSpace.MediaService.Api
dotnet run
```

Requires: SQL Server (`MediaDb`), Azurite (Azure Storage emulator), RabbitMQ.
`appsettings.json` key: `Database:MediaServiceDb` (note: non-standard key name).

### 5. OrderService (port 5004)

```bash
cd src/HiveSpace.OrderService/HiveSpace.OrderService.Api
dotnet run
```

Requires: SQL Server (`OrderDb`), RabbitMQ, Kafka (optional).
Seeds: Sample coupons and test data.

### 6. PaymentService (port 5005)

```bash
cd src/HiveSpace.PaymentService/HiveSpace.PaymentService.Api
dotnet run
```

Requires: SQL Server (`PaymentDb`), RabbitMQ.
VNPay sandbox credentials already set in `appsettings.json`.

### 7. NotificationService (port 5006)

```bash
cd src/HiveSpace.NotificationService/HiveSpace.NotificationService.Api
dotnet run
```

Requires: SQL Server (`NotificationDb`), Redis, RabbitMQ.
Seeds: Notification templates.

### 8. MediaService Azure Function (local)

```bash
cd src/HiveSpace.MediaService/HiveSpace.MediaService.Func
func start
# or
dotnet run
```

Requires: Azurite running. Uses `local.settings.json` for connection strings.

---

## Database Migrations

EF Core migrations are code-first. Startup auto-applies pending migrations in Development via `DataSeeder.EnsureSeedDataAsync` (which calls `context.Database.MigrateAsync()`).

For manual migration management:

```bash
# From the solution root (required for Directory.Packages.props to be found)

# UserService
dotnet ef migrations add <MigrationName> \
  --project src/HiveSpace.UserService/HiveSpace.UserService.Infrastructure \
  --startup-project src/HiveSpace.UserService/HiveSpace.UserService.Api

dotnet ef database update \
  --project src/HiveSpace.UserService/HiveSpace.UserService.Infrastructure \
  --startup-project src/HiveSpace.UserService/HiveSpace.UserService.Api

# CatalogService
dotnet ef migrations add <MigrationName> \
  --project src/HiveSpace.CatalogService/HiveSpace.CatalogService.Infrastructure \
  --startup-project src/HiveSpace.CatalogService/HiveSpace.CatalogService.Api

# OrderService
dotnet ef migrations add <MigrationName> \
  --project src/HiveSpace.OrderService/HiveSpace.OrderService.Infrastructure \
  --startup-project src/HiveSpace.OrderService/HiveSpace.OrderService.Api

# PaymentService
dotnet ef migrations add <MigrationName> \
  --project src/HiveSpace.PaymentService/HiveSpace.PaymentService.Infrastructure \
  --startup-project src/HiveSpace.PaymentService/HiveSpace.PaymentService.Api

# NotificationService
dotnet ef migrations add <MigrationName> \
  --project src/HiveSpace.NotificationService/HiveSpace.NotificationService.Core \
  --startup-project src/HiveSpace.NotificationService/HiveSpace.NotificationService.Api \
  --context NotificationDbContext

# MediaService
dotnet ef migrations add <MigrationName> \
  --project src/HiveSpace.MediaService/HiveSpace.MediaService.Core \
  --startup-project src/HiveSpace.MediaService/HiveSpace.MediaService.Api
```

Install EF tools if not present:
```bash
dotnet tool install --global dotnet-ef
# or update:
dotnet tool update --global dotnet-ef
```

---

## Environment Configuration

All configuration is in `appsettings.json` in each service's `Api` project. For secrets and environment-specific overrides use `appsettings.Development.json` or environment variables.

### Connection String Keys

| Service | Key | Database |
|---|---|---|
| UserService | `ConnectionStrings:UserServiceDb` | `UserDb` |
| CatalogService | `ConnectionStrings:CatalogDb` | `CatalogDb` |
| OrderService | `ConnectionStrings:OrderServiceDb` | `OrderDb` |
| PaymentService | `ConnectionStrings:PaymentServiceDb` | `PaymentDb` |
| NotificationService | `ConnectionStrings:DefaultConnection` | `NotificationDb` |
| NotificationService | `ConnectionStrings:Redis` | Redis connection string |
| MediaService | `Database:MediaServiceDb` | `MediaDb` (note: `Database:` section, not `ConnectionStrings:`) |

Default connection string format:
```
Server=localhost,1433;Database={Name}Db;User Id=sa;Password=Passw0rd123!;Encrypt=False;TrustServerCertificate=True
```

### Messaging Configuration

All services share the same `Messaging` section structure:

```json
{
  "Messaging": {
    "EnableRabbitMq": true,
    "EnableKafka": false,
    "RabbitMq": {
      "Host": "localhost",
      "Port": 5672,
      "Username": "guest",
      "Password": "guest",
      "DuplicateDetectionWindowMinutes": 5,
      "OutboxQueryDelaySeconds": 1,
      "OutboxQueryMessageLimit": 100,
      "OutboxQueryTimeoutSeconds": 60,
      "OutboxMessageDeliveryLimit": 50
    },
    "Kafka": {
      "BootstrapServers": "localhost:9092",
      "ClientId": "service-name",
      "ConsumerGroup": "service-name",
      "SecurityProtocol": "PLAINTEXT"
    }
  }
}
```

### Authentication Configuration

Non-UserService services validate JWTs:

```json
{
  "Authentication": {
    "Authority": "http://localhost:5001",
    "Audience": "catalog",
    "RequireHttpsMetadata": false
  }
}
```

Audience values: `catalog`, `order`, `payment`, `media`, `notification`.

### UserService IdentityServer Configuration

Configured in `appsettings.json` under `Clients` section. Three OIDC clients:
- `adminportal` → `http://localhost:5173`
- `sellercenter` → `http://localhost:5174`
- `storefront` → `http://localhost:5175`

Duende Community license key is included in `appsettings.json` (development only).

### VNPay Configuration (PaymentService)

```json
{
  "VNPay": {
    "TmnCode": "28FJPQF8",
    "HashSecret": "IKRTI9KSJY6KR4W5CNLSJ7OAIIKSD120",
    "BaseUrl": "https://sandbox.vnpayment.vn/paymentv2/vpcpay.html",
    "ReturnUrl": "http://localhost:5175/payment/result",
    "IpnUrl": "https://<ngrok-url>/api/v1/payments/webhook/vnpay"
  },
  "FrontendUrl": "http://localhost:5175"
}
```

For local VNPay IPN testing, use ngrok to expose port 5005:
```bash
ngrok http 5005
# Update IpnUrl in appsettings.json with the ngrok URL
```

### Snowflake ID Configuration (OrderService)

```json
{
  "Snowflake": {
    "DatacenterId": 0,
    "MachineId": 0
  }
}
```

### Azure Storage Configuration (MediaService)

```json
{
  "AzureStorage": {
    "ConnectionString": "DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=...;BlobEndpoint=http://127.0.0.1:10000/devstoreaccount1;...",
    "TempContainer": "temp-media-upload",
    "PublicContainer": "public-assets",
    "QueueName": "image-processing-queue",
    "CdnHost": ""
  },
  "MediaService": {
    "PresignUrlExpiryMinutes": 10
  },
  "MediaCleanup": {
    "ExpirationHours": 24,
    "BatchSize": 100
  }
}
```

### Resend Email Configuration (NotificationService)

```json
{
  "Resend": {
    "ApiToken": "re_..."
  }
}
```

---

## Hangfire Dashboard

NotificationService hosts Hangfire for background email dispatch jobs. When running locally, the Hangfire dashboard is accessible at:

```
http://localhost:5006/hangfire
```

(URL depends on registration in `ApplicationServiceCollectionExtensions.cs`.)

---

## Adding a New Service

Use the PowerShell scaffold script:

```powershell
# From repo root
.\scripts\new-service.ps1 -ServiceName "HiveSpace.InventoryService" -TemplateName "<template-short-name>" -AddToSolution
```

The script:
1. Finds or installs the dotnet template from `templates/` directory.
2. Runs `dotnet new` in `src/`.
3. Optionally runs `dotnet sln add` to register projects.

Available templates are in `templates/` at the repo root.

---

## CI/CD Pipeline

GitHub Actions workflows in `.github/workflows/`. Currently covers 4 services:

| Workflow | File | Trigger |
|---|---|---|
| UserService CI | `user-service-ci.yml` | Push/PR on `src/HiveSpace.UserService/**` |
| CatalogService CI | `catalog-service-ci.yml` | Push/PR on `src/HiveSpace.CatalogService/**` |
| MediaService API CI | `media-service-api-ci.yml` | Push/PR on `src/HiveSpace.MediaService/**` |
| MediaService Func CI | `media-service-func-ci.yml` | Push/PR on `src/HiveSpace.MediaService/HiveSpace.MediaService.Func/**` |
| ApiGateway CI | `api-gateway-ci.yml` | Push/PR on `src/HiveSpace.ApiGateway/**` |

### CI Pipeline Steps

1. `actions/checkout@v4`
2. `actions/setup-dotnet@v4` — .NET 8.0.x
3. NuGet cache via `actions/cache@v4` (key: `Directory.Packages.props` + `**/*.csproj` hash)
4. `dotnet restore` (service-specific projects)
5. `dotnet build` Release (per-layer, bottom-up)
6. `dotnet test` (when test projects exist)
7. Docker build + push to Azure Container Registry (on `master`/`release` branch)
8. Deploy to Azure Container Apps via Bicep

### Bicep Infrastructure

Bicep templates in `infra/`:
- `api-gateway.bicep` — YARP gateway container app
- `user-service.bicep` — UserService container app + IdentityServer config
- `catalog-service.bicep` — CatalogService container app
- `media-service.bicep` — MediaService + Func container app
- `media-service-api.bicep` — MediaService API container app
- `parameters/` — environment-specific parameter files

---

## Common Development Tasks

### Seeded Test Accounts

See `src/HiveSpace.UserService/SEEDED_ACCOUNTS.md` for pre-seeded user credentials (SystemAdmin, Admin, Seller, Buyer accounts).

### Testing an Endpoint via Gateway

All API calls in development should go through the gateway at `http://localhost:5000`:

```bash
# Get categories (public)
curl http://localhost:5000/api/v1/categories

# Get auth token first (use oidc-client or direct token endpoint):
curl -X POST http://localhost:5000/identity/connect/token \
  -d "grant_type=password&client_id=sellercenter&username=...&password=..."
```

### Configuring CORS

The gateway allows origins `http://localhost:5173`, `http://localhost:5174`, `http://localhost:5175`. To add more, update `AllowedOrigins` array in `src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/appsettings.json`.

### Checking Outbox Delivery

MassTransit outbox tables (`outbox_message`, `outbox_state`) can be queried directly in SQL Server to debug message delivery. The Quartz background job polls every `OutboxQueryDelaySeconds` seconds (default: 1s).

### Scaling Considerations

- `Snowflake:DatacenterId` and `Snowflake:MachineId` must be unique per OrderService instance to avoid ID collisions.
- Saga state machines use EF Core pessimistic locking — only one instance per `CorrelationId` may process events concurrently.
- SignalR NotificationHub requires Redis backplane when running multiple NotificationService instances.
