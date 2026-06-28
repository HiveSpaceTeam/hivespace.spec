# Implementation Plan: Local Backend Runtime Orchestration

**Branch:** `0006-aspire-setup`
**Date:** 2026-06-04
**Spec:** [spec.md](spec.md)

> This feature replaces the backend-local Docker Compose workflow with a project-based .NET Aspire local runtime. It is local development orchestration only, not production deployment.

---

## Phase 0 - Research

### Existing context read before planning

- [x] `.specify/memory/constitution.md`
- [x] `architecture/overview.md`
- [x] `services/_inventory.md`
- [x] `shared/api-catalog.md`
- [x] `shared/event-catalog.md`
- [x] `shared/coding-conventions.md`
- [x] `shared/glossary.md`
- [x] `services/api-gateway/README.md`
- [x] `services/identity-service/README.md`
- [x] `services/user-service/README.md`
- [x] `services/catalog-service/README.md`
- [x] `services/media-service/README.md`
- [x] `services/order-service/README.md`
- [x] `services/payment-service/README.md`
- [x] `services/notification-service/README.md`
- [x] `../hivespace.microservice/AGENTS.md` and `CLAUDE.md`
- [x] Backend solution, launch settings, and appsettings inspected for current projects, ports, and local dependencies

### Technical unknowns

No unresolved clarification markers remain for planning. Key decisions are captured in [research.md](research.md) and [ADR-0005](../../architecture/decisions/ADR-0005-aspire-local-runtime.md).

### Research notes

Research is captured in [research.md](research.md). The selected approach is a project-based Aspire AppHost plus ServiceDefaults in `../hivespace.microservice`, with Aspire starting backend API projects, MediaService Function, and local infrastructure containers on the existing local ports. Kafka is declared as a local AppHost infrastructure resource only; no backend service or function uses Kafka in v1.

---

## Phase 1 - Architecture & Data Model

### Service placement

This is a cross-service local development runtime feature. No business service owns domain behavior for the feature, and no public API/event contract changes are introduced.

| Service / area | Classification | Reason | Documentation/catalog action |
| --- | --- | --- | --- |
| Backend solution runtime | Owning area | Owns the new local orchestration entry point, ServiceDefaults integration, startup standardization, MediaService Function launch, and backend developer workflow. | Update backend root run docs in `AGENTS.md` and `CLAUDE.md`; add AppHost/ServiceDefaults projects in source repo during implementation. |
| ApiGateway | Changed supporting service | Participates in Aspire startup and observability while preserving route behavior and fixed `http://localhost:5000`; startup must move out of inline `Program.cs` into the standard extension shape. | Update service docs only if startup/runtime notes change; no API catalog changes. |
| IdentityService | Changed supporting service | Participates in Aspire startup, health/log visibility, SQL/RabbitMQ references, and fixed `http://localhost:5001`; keep IdentityServer/Razor behavior while matching startup file roles. | Runtime documentation only if needed; no endpoint/event catalog changes. |
| UserService | Changed supporting service | Participates in Aspire startup, health/log visibility, SQL/RabbitMQ references, and fixed `http://localhost:5007`; keep legacy controller behavior while matching startup file roles where technically possible. Kafka is not used by UserService in v1. | Runtime documentation only if needed; no endpoint/event catalog changes. |
| CatalogService | Changed supporting service | Participates in Aspire startup, health/log visibility, SQL/RabbitMQ reference, fixed `http://localhost:5002`, and standard startup order. | Runtime documentation only if needed; no endpoint/event catalog changes. |
| MediaService API | Changed supporting service | Participates in Aspire startup, health/log visibility, SQL/RabbitMQ/Azurite references, fixed `http://localhost:5003`, and standard startup order. | Runtime documentation only if needed; no endpoint/event catalog changes. |
| MediaService Function | Changed supporting runtime resource | Participates in AppHost startup as the media queue-processing Azure Function while preserving existing function behavior, queue payloads, and media processing contracts. | Document Azure Functions Core Tools prerequisite and runtime wiring; no API/event catalog changes. |
| OrderService | Changed supporting service | Participates in Aspire startup, health/log visibility, SQL/RabbitMQ references, fixed `http://localhost:5004`, and standard startup order. Kafka is not used by OrderService in v1. | Runtime documentation only if needed; no endpoint/event catalog changes. |
| PaymentService | Changed supporting service | Participates in Aspire startup, health/log visibility, SQL/RabbitMQ references, fixed `http://localhost:5005`, and standard startup order. Kafka is not used by PaymentService in v1. | Runtime documentation only if needed; no endpoint/event catalog changes. |
| NotificationService | Changed supporting service | Participates in Aspire startup, health/log visibility, SQL/RabbitMQ/Redis references, and fixed `http://localhost:5006`; keep Serilog and SignalR-specific behavior while matching startup file roles. | Runtime documentation only if needed; no endpoint/event catalog changes. |

### Runtime model

See [data-model.md](data-model.md).

This feature introduces no domain aggregate, repository, EF Core table, business migration, public endpoint, integration message, or saga state. Runtime entities are documentation/planning concepts only: Local Runtime, Backend Service, Infrastructure Dependency, Runtime Status, and Local Development Data.

### Local runtime resources

| Resource | Requirement |
| --- | --- |
| AppHost | Project-based C# Aspire AppHost in the backend solution. |
| ServiceDefaults | Shared .NET project referenced by API projects where compatible; adds OpenTelemetry traces, metrics, default health endpoints, Aspire dashboard export, and optional wrappers for standard HiveSpace authentication/OpenAPI registration without replacing service-specific behavior. |
| Connection string contract | Dependency endpoints and secrets exposed through `ConnectionStrings`; broker enablement remains in `Messaging` feature flags. |
| API projects | ApiGateway, IdentityService, UserService, CatalogService, MediaService API, OrderService, PaymentService, NotificationService. |
| Function resource | MediaService Function launched by AppHost through Azure Functions Core Tools on the existing local function port. |
| SQL Server | Aspire-managed local container on `localhost:1433`; databases remain service-owned. |
| RabbitMQ | Aspire-managed local container on `localhost:5672`; UI may stay on the conventional local management port if configured. |
| Kafka | Aspire-managed local container on `localhost:9092`, declared for future compatibility only. No service or function references, waits for, receives a connection string for, or enables Kafka in v1. |
| Redis | Aspire-managed local container on `localhost:6379` for NotificationService cache/throttle dependencies. |
| Azurite / Azure Storage emulator | Aspire-managed local storage emulator on existing blob/queue/table ports. |

### Integration events

No new integration events are introduced. Existing MassTransit/RabbitMQ contracts are reused unchanged. Kafka is declared as local infrastructure only and is not used by any service or function in v1. No `shared/event-catalog.md` update is required.

### API endpoints

No public endpoint path, method, auth policy, request/response contract, or route ownership changes are introduced. Existing gateway and service addresses remain usable. No `shared/api-catalog.md` update is required.

### Contracts

This feature has no public business API contract. The local developer runtime interface is documented in [contracts/local-runtime.md](contracts/local-runtime.md).

### Saga design

No saga required. The feature does not add or change a MassTransit saga state machine, cross-service workflow, compensation path, timeout event, or saga message contract.

### Architecture decision

Use [ADR-0005: Aspire Local Backend Runtime](../../architecture/decisions/ADR-0005-aspire-local-runtime.md). It records the decision to replace Docker Compose for backend local development with an Aspire AppHost while preserving existing local ports, including MediaService Function in v1, declaring Kafka without service usage, and avoiding Docker Compose data migration.

---

## Phase 2 - Implementation Plan

### Backend source repo changes

Implementation occurs in `../hivespace.microservice` after reading that repo's `AGENTS.md` and `CLAUDE.md`.

- Add project-based Aspire orchestration:
  - `HiveSpace.AppHost`
  - `HiveSpace.ServiceDefaults`
  - solution references for all orchestrated API projects and MediaService Function
- Add Aspire hosting package versions to `Directory.Packages.props`; do not use `Version=` in individual `.csproj` package references.
- Keep the backend `global.json` SDK policy explicit. If Aspire templates or packages require a newer .NET 8 SDK than `8.0.203`, update `global.json` deliberately and document the tooling requirement.
- Model local infrastructure in AppHost with fixed known host ports:
  - SQL Server `1433`
  - RabbitMQ `5672`
  - Kafka `9092` declared only; no dependent service/function resources in v1
  - Redis `6379`
  - Azurite Blob `10000`, Queue `10001`, Table `10002`
- Add AppHost project references for:
  - `src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway`
  - `src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api`
  - `src/HiveSpace.UserService/HiveSpace.UserService.Api`
  - `src/HiveSpace.CatalogService/HiveSpace.CatalogService.Api`
  - `src/HiveSpace.MediaService/HiveSpace.MediaService.Api`
  - `src/HiveSpace.MediaService/HiveSpace.MediaService.Func`
  - `src/HiveSpace.OrderService/HiveSpace.OrderService.Api`
  - `src/HiveSpace.PaymentService/HiveSpace.PaymentService.Api`
  - `src/HiveSpace.NotificationService/HiveSpace.NotificationService.Api`
- Add MediaService Function as an AppHost-managed function/executable resource:
  - Local function port `7072`
  - References to MediaService database, RabbitMQ, and Azurite blob/queue resources
  - Required local prerequisite: Azure Functions Core Tools
- Configure each API project resource with existing fixed application URLs from launch settings.
- Configure AppHost project resources to use Aspire resource references or dependency ordering where supported so services that depend on SQL Server, RabbitMQ, Redis, or Azurite either wait/recover during slow dependency startup or surface the dependency failure in the Aspire workflow.
- Configure MediaService Function dependency ordering so it waits for media database, RabbitMQ, and Azurite where AppHost can represent those dependencies.
- Do not attach Kafka as a reference, wait dependency, connection string, or enabled broker to any API project or function resource in v1.
- Inject dependency endpoints and secrets through AppHost connection strings/resource references so services continue to use current local assumptions:
  - `ConnectionStrings__*` for service databases
  - `ConnectionStrings__Redis`
  - `ConnectionStrings__RabbitMq`
  - `ConnectionStrings__AzureServiceBus` when enabled
  - `ConnectionStrings__AzureStorage` for Azurite/Azure Storage
  - `Authentication__Authority`
  - `ReverseProxy__Clusters__*__Destinations__destination1__Address`
  - `Messaging__EnableRabbitMq` and `Messaging__EnableAzureServiceBus` for runtime broker selection when enabled
  - `Messaging__EnableKafka=false` where an explicit disabled flag is needed to override legacy local settings
- Configure `HiveSpace.ServiceDefaults` for OpenTelemetry tracing, metrics, health checks, Aspire dashboard export, and thin wrappers for standard HiveSpace authentication/OpenAPI registration.
- Add `AddServiceDefaults()` and default health endpoint mapping to API startup paths where compatible, preserving existing service-specific health checks, Serilog, auth, YARP, MassTransit, and EF setup.
- Add ServiceDefaults wrapper helpers for default registration consistency:
  - `AddDefaultAuthentication(this IHostApplicationBuilder builder, string scope, Action<JwtBearerOptions>? configure = null)` delegates to `AddHiveSpaceJwtBearerAuthentication(...)`.
  - `AddDefaultOpenApi(this IServiceCollection services, string title, string description = "")` delegates to `AddHiveSpaceSwaggerGen(...)`.
  - Existing service `AddAppAuthentication()` and `AddAppOpenApi()` methods remain the service-owned place to pass scopes, titles, descriptions, and service-specific callbacks.
  - Do not adopt the eShop `Identity:Url`/`Identity:Audience` config shape, do not replace HiveSpace `Authentication:*` keys, and do not set `ValidateAudience = false`.
- Keep Serilog and backend service logs available as supplemental diagnostics; do not treat logs alone as satisfying local monitoring.
- Preserve correlation context across HTTP requests and async messaging where existing service behavior and Aspire/OpenTelemetry integration support it.
- Standardize startup structure for every included API project against `docs/agent/startup-conventions.md`:
  - `Program.cs` remains entry point only: create builder, call `ConfigureServices()`, call `ConfigurePipeline()` or `ConfigurePipelineAsync()`, run development migration/seed after pipeline setup where the service owns a database, and run the app.
  - `HostingExtensions.cs` owns orchestration of `AddApp*()` registrations and middleware/endpoint pipeline setup.
  - `ServiceCollectionExtensions.cs` owns thin `AddApp*()` wrappers and delegates to shared helpers where available.
- Move ApiGateway inline setup out of `Program.cs` into the standard extension structure, including YARP, CORS, token validation parameter setup, health checks, CSRF/session middleware, WebSockets, and reverse proxy mapping.
- Preserve service-specific behavior while normalizing file roles:
  - IdentityService keeps IdentityServer and Razor behavior.
  - UserService keeps controller and existing legacy startup behavior where required.
  - NotificationService keeps Serilog bootstrap and SignalR-specific authentication behavior.
  - ApiGateway keeps YARP, session forwarding, CSRF, CORS, and token-validation behavior.
  - MediaService API keeps migration-only seeding behavior.
  - MediaService Function keeps queue-triggered image-processing behavior unchanged.
- Wire `AddServiceDefaults()` through the standardized `ConfigureServices()` path and map default endpoints through the standardized pipeline path; do not add per-service inline Aspire setup in `Program.cs`.
- Wire default authentication/OpenAPI wrappers through existing service `AddAppAuthentication()` and `AddAppOpenApi()` wrappers where compatible; keep ApiGateway outside the default authentication wrapper and keep IdentityService Google/cookie/IdentityServer behavior local.
- Standardize shared dependency configuration:
  - Keep database and Redis configuration under existing `ConnectionStrings` keys.
  - Move RabbitMQ endpoint and credentials from nested `Messaging:RabbitMq` fields to `ConnectionStrings:RabbitMq`.
  - Remove Kafka bootstrap endpoints from service appsettings because Kafka is declared but unused by v1 services.
  - Move Azure Service Bus endpoint/secret from `Messaging:AzureServiceBus:ConnectionString` to `ConnectionStrings:AzureServiceBus`.
  - Move Azurite/Azure Storage endpoint/secret to `ConnectionStrings:AzureStorage`.
  - Keep `Messaging:EnableRabbitMq` and `Messaging:EnableAzureServiceBus` as broker feature flags where applicable; keep `Messaging:EnableKafka=false` only when needed to override legacy defaults.
  - Keep only non-secret broker tuning under `Messaging`, such as RabbitMQ prefetch/outbox/heartbeat settings. Remove Kafka tuning from service configs unless a future feature enables Kafka usage.
  - Fail fast with a clear configuration error when a broker flag is enabled but its required connection string is missing.
- Refactor `HiveSpace.Infrastructure.Messaging` options and extensions so broker host/bootstrap/secret values are read from `ConnectionStrings`, while `MessagingOptions` remains the source for feature flags and non-secret tuning. Kafka-related messaging paths must remain disabled for all v1 services.
- Include MediaService Function in v1 local orchestration, but do not change image-processing behavior, queue payload shape, media API contracts, or integration event contracts to force inclusion.

### Documentation and config updates

- Update backend `AGENTS.md` and `CLAUDE.md` together to make the Aspire AppHost the preferred backend local startup command and mark Docker Compose as replaced for backend local development.
- Document Azure Functions Core Tools as required for the full AppHost runtime because MediaService Function is included in v1.
- Document Kafka as declared local infrastructure only, with no v1 service/function usage.
- Update spec repo docs after task generation through `$update-catalog` only for runtime documentation references; API and event catalogs remain unchanged.
- Keep `../hivespace.config` as infrastructure context only; feature implementation tasks must keep source-repo runtime settings and appsettings changes under `../hivespace.microservice`.
- Do not migrate existing Docker Compose local data. Document that new Aspire-managed data should survive normal AppHost stop/restart when persistent resource volumes are configured.

### Verification

- `dotnet restore` in `../hivespace.microservice`.
- `dotnet build` in `../hivespace.microservice`.
- Static startup convention check: all included API `Program.cs` files are entry-point only, all service registration and middleware setup lives in `HostingExtensions.cs`/`ServiceCollectionExtensions.cs`, and any unavoidable service-specific startup difference is documented in code or service docs.
- Static auth/OpenAPI wrapper check: default auth/OpenAPI helpers delegate to existing HiveSpace shared helpers, each API service keeps its explicit scope/title/description, NotificationService keeps its SignalR JWT callback, ApiGateway does not use the default auth wrapper, and no implementation introduces `Identity:*` auth config or disables audience validation.
- Static config check: no service appsettings uses nested `Messaging:RabbitMq` endpoint/credential fields, any Kafka bootstrap endpoint or enabled Kafka flag, or `Messaging:AzureServiceBus:ConnectionString`.
- Run AppHost and verify the Aspire dashboard/resource view shows all API projects, MediaService Function, and infrastructure resources.
- Verify the Aspire dashboard/resource view shows MediaService Function as a local function resource and surfaces its status/logs in the same workflow.
- Verify Kafka appears as a declared AppHost infrastructure resource and has no dependent service/function resources.
- Verify the Aspire dashboard shows OpenTelemetry traces and metrics for compatible backend services, including at least one gateway-to-service request.
- Verify that slower infrastructure startup does not require manual service-by-service restarts when Aspire resource references or dependency ordering can represent the dependency; otherwise verify the failure is visible in the AppHost output or Aspire dashboard.
- Smoke checks:
  - `http://localhost:5000/health`
  - `http://localhost:5001/.well-known/openid-configuration`
  - service health endpoints where available
  - MediaService Function starts on the documented local function port and binds to the Azurite queue trigger configuration
  - RabbitMQ, Redis, SQL Server, and Azurite reachable by dependent services
  - Kafka declared on `localhost:9092` but not required by any service/function health path
  - broker-enabled services fail with a clear error when the required broker connection string is intentionally omitted
- Verify existing frontend gateway base URL still points to `http://localhost:5000` without config changes.
- Run `npx gitnexus analyze` in the backend repo before PR if code changes are committed.

---

## Conditional Design Artifacts

| Artifact | Decision |
| --- | --- |
| `saga-design.md` | Not created; no saga state machine is introduced or changed. |
| ADR | Created: [ADR-0005](../../architecture/decisions/ADR-0005-aspire-local-runtime.md). |
| Event catalog | No update required; no new or changed integration message. |
| API catalog | No update required; no new or changed public endpoint. |

---

## Constitution Compliance Check

### Pre-design

- [x] Spec repo remains planning-only; runnable code will be added only in `../hivespace.microservice`.
- [x] Service boundaries are preserved; no service gains business data or decisions from another service.
- [x] No new public endpoints are planned; API catalog remains unchanged.
- [x] No new cross-service messages are planned; event catalog remains unchanged.
- [x] No saga state machine is added or changed.
- [x] No domain aggregate, EF migration, persisted money value, hard-delete behavior, or frontend UI change is involved.
- [x] NuGet package versions will remain centrally managed in `Directory.Packages.props`.
- [x] Local monitoring is based on Aspire/OpenTelemetry traces and metrics plus health/status; logs remain diagnostics only.
- [x] Startup standardization is limited to service startup wiring and must preserve business behavior, public APIs, integration events, and service responsibilities.
- [x] Dependency endpoints and secrets are standardized under `ConnectionStrings`; `Messaging` retains broker feature flags and non-secret tuning only.
- [x] Kafka is declared as local infrastructure only and is not a v1 service dependency.

### Post-design

The design remains compliant. The cross-cutting architectural changes are local developer orchestration, startup standardization, local OpenTelemetry monitoring, MediaService Function orchestration, and Kafka declaration without v1 usage, documented in ADR-0005. They do not alter service ownership, business APIs, integration events, sagas, data ownership, or browser-facing contracts.
