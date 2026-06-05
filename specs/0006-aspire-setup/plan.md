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

No unresolved clarification markers remain for planning. Key decisions are captured in [research.md](research.md) and [ADR-0004](../../architecture/decisions/ADR-0004-aspire-local-runtime.md).

### Research notes

Research is captured in [research.md](research.md). The selected approach is a project-based Aspire AppHost plus ServiceDefaults in `../hivespace.microservice`, with Aspire starting backend API projects and local infrastructure containers on the existing local ports.

---

## Phase 1 - Architecture & Data Model

### Service placement

This is a cross-service local development runtime feature. No business service owns domain behavior for the feature, and no public API/event contract changes are introduced.

| Service / area | Classification | Reason | Documentation/catalog action |
| --- | --- | --- | --- |
| Backend solution runtime | Owning area | Owns the new local orchestration entry point, ServiceDefaults integration, startup standardization, and backend developer workflow. | Update backend root run docs in `AGENTS.md` and `CLAUDE.md`; add AppHost/ServiceDefaults projects in source repo during implementation. |
| ApiGateway | Changed supporting service | Participates in Aspire startup and observability while preserving route behavior and fixed `http://localhost:5000`; startup must move out of inline `Program.cs` into the standard extension shape. | Update service docs only if startup/runtime notes change; no API catalog changes. |
| IdentityService | Changed supporting service | Participates in Aspire startup, health/log visibility, SQL/RabbitMQ references, and fixed `http://localhost:5001`; keep IdentityServer/Razor behavior while matching startup file roles. | Runtime documentation only if needed; no endpoint/event catalog changes. |
| UserService | Changed supporting service | Participates in Aspire startup, health/log visibility, SQL/RabbitMQ/Kafka references, and fixed `http://localhost:5007`; keep legacy controller behavior while matching startup file roles where technically possible. | Runtime documentation only if needed; no endpoint/event catalog changes. |
| CatalogService | Changed supporting service | Participates in Aspire startup, health/log visibility, SQL/RabbitMQ reference, fixed `http://localhost:5002`, and standard startup order. | Runtime documentation only if needed; no endpoint/event catalog changes. |
| MediaService API | Changed supporting service | Participates in Aspire startup, health/log visibility, SQL/RabbitMQ/Azurite references, fixed `http://localhost:5003`, and standard startup order. | Runtime documentation only if needed; no endpoint/event catalog changes. |
| OrderService | Changed supporting service | Participates in Aspire startup, health/log visibility, SQL/RabbitMQ/Kafka references, fixed `http://localhost:5004`, and standard startup order. | Runtime documentation only if needed; no endpoint/event catalog changes. |
| PaymentService | Changed supporting service | Participates in Aspire startup, health/log visibility, SQL/RabbitMQ/Kafka references, fixed `http://localhost:5005`, and standard startup order. | Runtime documentation only if needed; no endpoint/event catalog changes. |
| NotificationService | Changed supporting service | Participates in Aspire startup, health/log visibility, SQL/RabbitMQ/Redis references, and fixed `http://localhost:5006`; keep Serilog and SignalR-specific behavior while matching startup file roles. | Runtime documentation only if needed; no endpoint/event catalog changes. |
| MediaService Function | Reused supporting service | Existing function code may remain documented separately unless a simple Aspire launch path is available without changing function behavior. | Verification only for v1; do not change function behavior. |
| Config repo | Changed supporting repo | Docker Compose is no longer the backend local development entry point, but config may still hold appsettings sync outputs. | Update config docs/settings only if implementation changes synchronized appsettings; no catalog changes. |

### Runtime model

See [data-model.md](data-model.md).

This feature introduces no domain aggregate, repository, EF Core table, business migration, public endpoint, integration message, or saga state. Runtime entities are documentation/planning concepts only: Local Runtime, Backend Service, Infrastructure Dependency, Runtime Status, and Local Development Data.

### Local runtime resources

| Resource | Requirement |
| --- | --- |
| AppHost | Project-based C# Aspire AppHost in the backend solution. |
| ServiceDefaults | Shared .NET project referenced by API projects where compatible; adds OpenTelemetry traces, metrics, default health endpoints, and Aspire dashboard export without replacing existing service-specific behavior. |
| Connection string contract | Dependency endpoints and secrets exposed through `ConnectionStrings`; broker enablement remains in `Messaging` feature flags. |
| API projects | ApiGateway, IdentityService, UserService, CatalogService, MediaService API, OrderService, PaymentService, NotificationService. |
| SQL Server | Aspire-managed local container on `localhost:1433`; databases remain service-owned. |
| RabbitMQ | Aspire-managed local container on `localhost:5672`; UI may stay on the conventional local management port if configured. |
| Kafka | Aspire-managed local container on `localhost:9092` for services with existing Kafka-enabled local config. |
| Redis | Aspire-managed local container on `localhost:6379` for NotificationService cache/throttle dependencies. |
| Azurite / Azure Storage emulator | Aspire-managed local storage emulator on existing blob/queue/table ports. |

### Integration events

No new integration events are introduced. Existing MassTransit/RabbitMQ and Kafka contracts are reused unchanged. No `shared/event-catalog.md` update is required.

### API endpoints

No public endpoint path, method, auth policy, request/response contract, or route ownership changes are introduced. Existing gateway and service addresses remain usable. No `shared/api-catalog.md` update is required.

### Contracts

This feature has no public business API contract. The local developer runtime interface is documented in [contracts/local-runtime.md](contracts/local-runtime.md).

### Saga design

No saga required. The feature does not add or change a MassTransit saga state machine, cross-service workflow, compensation path, timeout event, or saga message contract.

### Architecture decision

Use [ADR-0004: Aspire Local Backend Runtime](../../architecture/decisions/ADR-0004-aspire-local-runtime.md). It records the decision to replace Docker Compose for backend local development with an Aspire AppHost while preserving existing local ports and avoiding Docker Compose data migration.

---

## Phase 2 - Implementation Plan

### Backend source repo changes

Implementation occurs in `../hivespace.microservice` after reading that repo's `AGENTS.md` and `CLAUDE.md`.

- Add project-based Aspire orchestration:
  - `HiveSpace.AppHost`
  - `HiveSpace.ServiceDefaults`
  - solution references for all orchestrated API projects
- Add Aspire hosting package versions to `Directory.Packages.props`; do not use `Version=` in individual `.csproj` package references.
- Keep the backend `global.json` SDK policy explicit. If Aspire templates or packages require a newer .NET 8 SDK than `8.0.203`, update `global.json` deliberately and document the tooling requirement.
- Model local infrastructure in AppHost with fixed known host ports:
  - SQL Server `1433`
  - RabbitMQ `5672`
  - Kafka `9092`
  - Redis `6379`
  - Azurite Blob `10000`, Queue `10001`, Table `10002`
- Add AppHost project references for:
  - `src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway`
  - `src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api`
  - `src/HiveSpace.UserService/HiveSpace.UserService.Api`
  - `src/HiveSpace.CatalogService/HiveSpace.CatalogService.Api`
  - `src/HiveSpace.MediaService/HiveSpace.MediaService.Api`
  - `src/HiveSpace.OrderService/HiveSpace.OrderService.Api`
  - `src/HiveSpace.PaymentService/HiveSpace.PaymentService.Api`
  - `src/HiveSpace.NotificationService/HiveSpace.NotificationService.Api`
- Configure each project resource with existing fixed application URLs from launch settings.
- Inject dependency endpoints and secrets through AppHost connection strings/resource references so services continue to use current local assumptions:
  - `ConnectionStrings__*` for service databases
  - `ConnectionStrings__Redis`
  - `ConnectionStrings__RabbitMq`
  - `ConnectionStrings__Kafka`
  - `ConnectionStrings__AzureServiceBus` when enabled
  - `ConnectionStrings__AzureStorage` for Azurite/Azure Storage
  - `Authentication__Authority`
  - `ReverseProxy__Clusters__*__Destinations__destination1__Address`
  - `Messaging__EnableRabbitMq`, `Messaging__EnableKafka`, and `Messaging__EnableAzureServiceBus` for runtime broker selection
- Configure `HiveSpace.ServiceDefaults` for OpenTelemetry tracing, metrics, health checks, and Aspire dashboard export.
- Add `AddServiceDefaults()` and default health endpoint mapping to API startup paths where compatible, preserving existing service-specific health checks, Serilog, auth, YARP, MassTransit, and EF setup.
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
- Wire `AddServiceDefaults()` through the standardized `ConfigureServices()` path and map default endpoints through the standardized pipeline path; do not add per-service inline Aspire setup in `Program.cs`.
- Standardize shared dependency configuration:
  - Keep database and Redis configuration under existing `ConnectionStrings` keys.
  - Move RabbitMQ endpoint and credentials from nested `Messaging:RabbitMq` fields to `ConnectionStrings:RabbitMq`.
  - Move Kafka bootstrap endpoint from `Messaging:Kafka:BootstrapServers` to `ConnectionStrings:Kafka`.
  - Move Azure Service Bus endpoint/secret from `Messaging:AzureServiceBus:ConnectionString` to `ConnectionStrings:AzureServiceBus`.
  - Move Azurite/Azure Storage endpoint/secret to `ConnectionStrings:AzureStorage`.
  - Keep `Messaging:EnableRabbitMq`, `Messaging:EnableKafka`, and `Messaging:EnableAzureServiceBus` as broker feature flags.
  - Keep only non-secret broker tuning under `Messaging`, such as RabbitMQ prefetch/outbox/heartbeat settings and Kafka client ID, consumer group, security protocol, and non-secret client behavior.
  - Fail fast with a clear configuration error when a broker flag is enabled but its required connection string is missing.
- Refactor `HiveSpace.Infrastructure.Messaging` options and extensions so broker host/bootstrap/secret values are read from `ConnectionStrings`, while `MessagingOptions` remains the source for feature flags and non-secret tuning.
- Keep MediaService Function out of v1 unless it can be launched by Aspire without changing function behavior or requiring a separate runtime contract.

### Documentation and config updates

- Update backend `AGENTS.md` and `CLAUDE.md` together to make the Aspire AppHost the preferred backend local startup command and mark Docker Compose as replaced for backend local development.
- Update spec repo docs after task generation through `$update-catalog` only for runtime documentation references; API and event catalogs remain unchanged.
- If appsettings are changed in the backend repo, run the backend repo's required config sync and coordinate with `../hivespace.config`.
- Do not migrate existing Docker Compose local data. Document that new Aspire-managed data should survive normal AppHost stop/restart when persistent resource volumes are configured.

### Verification

- `dotnet restore` in `../hivespace.microservice`.
- `dotnet build` in `../hivespace.microservice`.
- Static startup convention check: all included API `Program.cs` files are entry-point only, all service registration and middleware setup lives in `HostingExtensions.cs`/`ServiceCollectionExtensions.cs`, and any unavoidable service-specific startup difference is documented in code or service docs.
- Static config check: no service appsettings uses nested `Messaging:RabbitMq` endpoint/credential fields, `Messaging:Kafka:BootstrapServers`, or `Messaging:AzureServiceBus:ConnectionString`.
- Run AppHost and verify the Aspire dashboard/resource view shows all API projects and infrastructure resources.
- Verify the Aspire dashboard shows OpenTelemetry traces and metrics for compatible backend services, including at least one gateway-to-service request.
- Smoke checks:
  - `http://localhost:5000/health`
  - `http://localhost:5001/.well-known/openid-configuration`
  - service health endpoints where available
  - RabbitMQ, Kafka, Redis, SQL Server, and Azurite reachable by dependent services
  - broker-enabled services fail with a clear error when the required broker connection string is intentionally omitted
- Verify existing frontend gateway base URL still points to `http://localhost:5000` without config changes.
- Run `npx gitnexus analyze` in the backend repo before PR if code changes are committed.

---

## Conditional Design Artifacts

| Artifact | Decision |
| --- | --- |
| `saga-design.md` | Not created; no saga state machine is introduced or changed. |
| ADR | Created: [ADR-0004](../../architecture/decisions/ADR-0004-aspire-local-runtime.md). |
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

### Post-design

The design remains compliant. The cross-cutting architectural changes are local developer orchestration, startup standardization, and local OpenTelemetry monitoring, documented in ADR-0004. They do not alter service ownership, business APIs, integration events, sagas, data ownership, or browser-facing contracts.
