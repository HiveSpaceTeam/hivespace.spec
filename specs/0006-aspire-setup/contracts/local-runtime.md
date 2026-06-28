# Contract: Local Backend Runtime

This is a developer workflow contract, not a public business API contract.

## Startup Contract

The backend repo must expose one documented local backend startup entry point.

Expected outcome:

- Starts ApiGateway, all backend API services, and MediaService Function in the feature scope.
- Starts SQL Server, RabbitMQ, Kafka, Redis, and Azurite local dependencies.
- Declares Kafka for future compatibility only; no service or function uses Kafka in v1.
- Launches MediaService Function through Azure Functions Core Tools on the documented local function port.
- Preserves existing service and dependency ports.
- Provides one runtime view for status, OpenTelemetry traces, metrics, and logs.
- Uses the standard backend API startup file roles for every included API project where technically possible.
- Supplies infrastructure endpoints and secrets through `ConnectionStrings`, while broker feature flags remain under `Messaging`.

## Startup Standardization Contract

Every included API project must expose a consistent startup surface for AppHost, ServiceDefaults, health, and OpenTelemetry integration:

- `Program.cs` is entry point only and delegates service registration and pipeline setup.
- `HostingExtensions.cs` owns `ConfigureServices()` and `ConfigurePipeline()` or `ConfigurePipelineAsync()` when async setup is required.
- `ServiceCollectionExtensions.cs` owns thin `AddApp*()` service-registration wrappers that delegate to shared helpers where available.
- Compatible service `AddAppAuthentication()` wrappers may call ServiceDefaults `AddDefaultAuthentication(...)`, but they must pass the existing service scope explicitly and preserve service-specific JWT callbacks.
- Compatible service `AddAppOpenApi()` wrappers may call ServiceDefaults `AddDefaultOpenApi(...)`, but they must pass the existing title and description explicitly.
- Development migration and seed work runs after pipeline configuration and before app run for services that own a database.
- ApiGateway follows the same file-role structure while preserving YARP, CORS, token validation, health, WebSocket, CSRF, and session-forwarding behavior.

Document any unavoidable service-specific startup difference. Do not use startup standardization to change public routes, auth policy, token validation contract, OpenAPI endpoint contract, domain behavior, integration events, or database ownership.

## Authentication And OpenAPI Wrapper Contract

ServiceDefaults may expose shared wrappers for standard HiveSpace authentication and OpenAPI registration. These wrappers are internal startup helpers, not public business contracts.

- `AddDefaultAuthentication` delegates to the existing HiveSpace JWT/auth helper and uses the existing `Authentication:Authority`, `Authentication:Audience`, and `Authentication:RequireHttpsMetadata` configuration shape.
- `AddDefaultAuthentication` must not introduce `Identity:Url` or `Identity:Audience` as replacement configuration keys and must not disable configured audience validation.
- ApiGateway must not use `AddDefaultAuthentication`; its gateway-specific token validation, CSRF, cookie/session forwarding, and YARP behavior remains local.
- NotificationService keeps its SignalR query-token JWT behavior through the wrapper callback.
- IdentityService keeps Google external auth, cookies, ASP.NET Identity, and IdentityServer setup local.
- `AddDefaultOpenApi` delegates to the existing HiveSpace OpenAPI helper and preserves each service's title and description.

## Dependency Configuration Contract

The local runtime and deployable app configuration must use a connection-string-first dependency contract:

- Service databases use their existing `ConnectionStrings:{ServiceDbName}` keys.
- Redis uses `ConnectionStrings:Redis`.
- RabbitMQ uses `ConnectionStrings:RabbitMq` for endpoint and credentials.
- Kafka is declared by AppHost on the existing local port, but services and functions must not receive or require `ConnectionStrings:Kafka` in v1.
- Azure Service Bus uses `ConnectionStrings:AzureServiceBus`.
- Azure Storage/Azurite uses `ConnectionStrings:AzureStorage`.
- Broker selection remains controlled by `Messaging:EnableRabbitMq`, `Messaging:EnableKafka`, and `Messaging:EnableAzureServiceBus`.
- `Messaging:RabbitMq` may contain non-secret tuning only; it must not contain broker endpoints, usernames, passwords, bootstrap servers, or connection strings.
- `Messaging:EnableKafka` must remain false or absent for every v1 service/function, and service/function appsettings must not contain Kafka bootstrap endpoints or Kafka tuning.

If a broker flag is enabled and its required `ConnectionStrings` entry is missing, startup must fail with a clear configuration error.

## Address Contract

Existing local addresses remain stable:

| Resource | Address |
| --- | --- |
| ApiGateway | `http://localhost:5000` |
| IdentityService | `http://localhost:5001` |
| CatalogService | `http://localhost:5002` |
| MediaService API | `http://localhost:5003` |
| MediaService Function | `http://localhost:7072` |
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
- MediaService Function resource.
- Included local infrastructure resources.
- Running/failed/unhealthy/stopped status.
- Backend service OpenTelemetry traces for Aspire-compatible services.
- Backend service OpenTelemetry metrics for Aspire-compatible services.
- Backend service logs as supporting diagnostics.
- Enough failure detail to identify the failed service or dependency.

Logs alone do not satisfy the monitoring contract. The local monitoring path is Aspire-backed OpenTelemetry telemetry plus health/status, with logs retained for troubleshooting.

## MediaService Function Contract

MediaService Function is in v1 local runtime scope.

- It runs from `../hivespace.microservice/src/HiveSpace.MediaService/HiveSpace.MediaService.Func`.
- It uses Azure Functions Core Tools as the local function host.
- It shares the MediaService database, RabbitMQ, and Azurite blob/queue resources managed by AppHost.
- It must use connection-string-based storage and broker configuration compatible with AppHost injection.
- It must not change image-processing behavior, queue payload shape, media public API contracts, or integration event contracts.

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
