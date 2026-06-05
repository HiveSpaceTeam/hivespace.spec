# Contract: Local Backend Runtime

This is a developer workflow contract, not a public business API contract.

## Startup Contract

The backend repo must expose one documented local backend startup entry point.

Expected outcome:

- Starts ApiGateway and all backend API services in the feature scope.
- Starts SQL Server, RabbitMQ, Kafka, Redis, and Azurite local dependencies.
- Preserves existing service and dependency ports.
- Provides one runtime view for status, OpenTelemetry traces, metrics, and logs.
- Uses the standard backend API startup file roles for every included API project where technically possible.
- Supplies infrastructure endpoints and secrets through `ConnectionStrings`, while broker feature flags remain under `Messaging`.

## Startup Standardization Contract

Every included API project must expose a consistent startup surface for AppHost, ServiceDefaults, health, and OpenTelemetry integration:

- `Program.cs` is entry point only and delegates service registration and pipeline setup.
- `HostingExtensions.cs` owns `ConfigureServices()` and `ConfigurePipeline()` or `ConfigurePipelineAsync()` when async setup is required.
- `ServiceCollectionExtensions.cs` owns thin `AddApp*()` service-registration wrappers that delegate to shared helpers where available.
- Development migration and seed work runs after pipeline configuration and before app run for services that own a database.
- ApiGateway follows the same file-role structure while preserving YARP, CORS, token validation, health, WebSocket, CSRF, and session-forwarding behavior.

Document any unavoidable service-specific startup difference. Do not use startup standardization to change public routes, auth policy, domain behavior, integration events, or database ownership.

## Dependency Configuration Contract

The local runtime and deployable app configuration must use a connection-string-first dependency contract:

- Service databases use their existing `ConnectionStrings:{ServiceDbName}` keys.
- Redis uses `ConnectionStrings:Redis`.
- RabbitMQ uses `ConnectionStrings:RabbitMq` for endpoint and credentials.
- Kafka uses `ConnectionStrings:Kafka` for bootstrap endpoint information.
- Azure Service Bus uses `ConnectionStrings:AzureServiceBus`.
- Azure Storage/Azurite uses `ConnectionStrings:AzureStorage`.
- Broker selection remains controlled by `Messaging:EnableRabbitMq`, `Messaging:EnableKafka`, and `Messaging:EnableAzureServiceBus`.
- `Messaging:RabbitMq` and `Messaging:Kafka` may contain non-secret tuning only; they must not contain broker endpoints, usernames, passwords, bootstrap servers, or connection strings.

If a broker flag is enabled and its required `ConnectionStrings` entry is missing, startup must fail with a clear configuration error.

## Address Contract

Existing local addresses remain stable:

| Resource | Address |
| --- | --- |
| ApiGateway | `http://localhost:5000` |
| IdentityService | `http://localhost:5001` |
| CatalogService | `http://localhost:5002` |
| MediaService API | `http://localhost:5003` |
| OrderService | `http://localhost:5004` |
| PaymentService | `http://localhost:5005` |
| NotificationService | `http://localhost:5006` |
| UserService | `http://localhost:5007` |
| SQL Server | `localhost:1433` |
| RabbitMQ AMQP | `localhost:5672` |
| Kafka | `localhost:9092` |
| Redis | `localhost:6379` |
| Azurite Blob | `localhost:10000` |
| Azurite Queue | `localhost:10001` |
| Azurite Table | `localhost:10002` |

## Observability Contract

The runtime view must make these visible:

- Included backend service resources.
- Included local infrastructure resources.
- Running/failed/unhealthy/stopped status.
- Backend service OpenTelemetry traces for Aspire-compatible services.
- Backend service OpenTelemetry metrics for Aspire-compatible services.
- Backend service logs as supporting diagnostics.
- Enough failure detail to identify the failed service or dependency.

Logs alone do not satisfy the monitoring contract. The local monitoring path is Aspire-backed OpenTelemetry telemetry plus health/status, with logs retained for troubleshooting.

## Data Contract

- New-runtime local data should survive normal stop/restart workflows where supported.
- Existing Docker Compose local data is not migrated.
- Resetting local development data is an explicit maintenance action, not part of normal startup.

## Exclusions

- No public API contract changes.
- No integration event contract changes.
- No saga message changes.
- No frontend dev server orchestration in v1.
- No production deployment contract.
