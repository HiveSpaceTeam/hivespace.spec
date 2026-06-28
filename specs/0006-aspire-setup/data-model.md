# Data Model: Local Backend Runtime Orchestration

This feature has no persisted business data model. The entities below describe the local runtime concepts that implementation and documentation must cover.

## Local Runtime

Represents the developer-facing backend startup experience.

| Attribute | Description |
| --- | --- |
| Startup entry point | One documented command or launch action. |
| Runtime view | One local place to inspect service/resource status, OpenTelemetry traces, metrics, and logs. |
| Scope | Backend API services, MediaService Function, and required local infrastructure dependencies. |
| Exclusions | Frontend dev servers, production deployment, Docker Compose data migration. |

## Backend Service

Represents a backend API or gateway process included in the runtime.

| Attribute | Description |
| --- | --- |
| Name | Service name from architecture/service inventory. |
| Project path | Source project launched by the runtime. |
| Local URL | Existing fixed localhost address preserved by the runtime. |
| Dependencies | Infrastructure resources required by the service. |
| Runtime status | Visible status such as starting, running, unhealthy, stopped, or failed. |
| Startup surface | Standard `Program.cs`, `HostingExtensions.cs`, and `ServiceCollectionExtensions.cs` file roles used for AppHost, ServiceDefaults, health, and OpenTelemetry integration. |

Required backend services:

| Service | Local URL |
| --- | --- |
| ApiGateway | `http://localhost:5000` |
| IdentityService | `http://localhost:5001` |
| CatalogService | `http://localhost:5002` |
| MediaService API | `http://localhost:5003` |
| OrderService | `http://localhost:5004` |
| PaymentService | `http://localhost:5005` |
| NotificationService | `http://localhost:5006` |
| UserService | `http://localhost:5007` |

Required function resources:

| Resource | Local URL |
| --- | --- |
| MediaService Function | `http://localhost:7072` |

MediaService Function requirements:

- Launched by AppHost through Azure Functions Core Tools.
- Uses the MediaService database, RabbitMQ, and Azurite blob/queue resources.
- Preserves existing media processing behavior and queue payload contracts.

## Infrastructure Dependency

Represents a local service dependency managed by the runtime.

| Attribute | Description |
| --- | --- |
| Name | Dependency name shown in the runtime view. |
| Category | Database, message broker, cache, or storage emulator. |
| Local endpoint | Existing fixed localhost endpoint. |
| Persistence policy | Data created by the new runtime survives normal stop/restart where supported. |
| Connection string key | `ConnectionStrings` key used by Aspire and deployable configuration. |

Required dependencies:

| Dependency | Local endpoint |
| --- | --- |
| SQL Server | `localhost:1433` |
| RabbitMQ | `localhost:5672` |
| Kafka | `localhost:9092` declared only; no v1 service/function dependency |
| Redis | `localhost:6379` |
| Azurite Blob | `localhost:10000` |
| Azurite Queue | `localhost:10001` |
| Azurite Table | `localhost:10002` |

Connection string keys:

| Dependency | Connection string key |
| --- | --- |
| Service databases | Existing service-owned database key, such as `CatalogDb`, `OrderServiceDb`, or `DefaultConnection` |
| RabbitMQ | `RabbitMq` |
| Kafka | AppHost resource only in v1; no service/function `ConnectionStrings:Kafka` |
| Azure Service Bus | `AzureServiceBus` |
| Redis | `Redis` |
| Azure Storage / Azurite | `AzureStorage` |

## Messaging Runtime Selection

Represents runtime broker enablement.

Rules:

- `Messaging:EnableRabbitMq`, `Messaging:EnableKafka`, and `Messaging:EnableAzureServiceBus` choose active broker integrations.
- `Messaging:EnableKafka` must be false or absent for every v1 service/function because Kafka is declared but unused.
- Broker endpoint and secret values belong in `ConnectionStrings`, not nested `Messaging` provider objects.
- `Messaging` provider objects may keep non-secret tuning only.
- A missing connection string for an enabled broker is a startup configuration error.
- Kafka missing-connection validation is not exercised by v1 services because no service/function enables Kafka.

## Runtime Status

Represents the visible state of a service or dependency.

Allowed statuses:

- Starting
- Running
- Unhealthy
- Stopped
- Failed

Rules:

- Every included backend service and infrastructure dependency must expose a visible status.
- MediaService Function must expose visible status/logs in the same runtime workflow.
- Startup failures must identify the failing service or dependency.
- OpenTelemetry traces and metrics must be available for Aspire-compatible backend services from the same runtime workflow.
- Logs must be available for backend services as supplemental diagnostics, but logs alone do not satisfy monitoring.

## Startup Surface

Represents the per-service startup structure used by the local runtime.

Rules:

- `Program.cs` is entry point only.
- `HostingExtensions.cs` orchestrates service registration and middleware/endpoint pipeline setup.
- `ServiceCollectionExtensions.cs` contains thin `AddApp*()` service-registration wrappers.
- ApiGateway, IdentityService, UserService, NotificationService, and MediaService API may keep service-specific behavior, but should still use the same file roles wherever technically possible.
- Startup standardization must not change public API paths, auth policies, integration events, service ownership, or domain behavior.

## Local Development Data

Represents data created by local infrastructure during development.

Rules:

- Data created by the new runtime should survive normal runtime stop/restart workflows when persistent volumes are configured.
- Existing Docker Compose local data is not migrated into the new runtime.
- Reset/reseed behavior can be documented as a separate local maintenance workflow.
