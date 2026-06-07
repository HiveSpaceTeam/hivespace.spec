# Backend Tasks: Local Backend Runtime Orchestration

## Repository Preparation

### Verify

- [ ] B001 [US1] [US4] Verify `../hivespace.microservice` agent and startup instructions
  - File: `../hivespace.microservice/AGENTS.md`, `../hivespace.microservice/CLAUDE.md`, `../hivespace.microservice/docs/agent/startup-conventions.md`
  - Read both agent instruction files before editing backend code and follow any stricter backend-repo rules.
  - Read the startup convention doc and use it as the file-role source of truth for all included API projects.
  - Do not begin source edits before confirming whether backend repo instructions require additional package, build, or config-sync steps.
  - Acceptance: implementation notes or PR summary can cite the backend repo instructions and startup convention as read.

## Backend Solution And Packages

### Update

- [ ] B002 [US1] Update `.NET Aspire package and SDK policy`
  - File: `../hivespace.microservice/Directory.Packages.props`, `../hivespace.microservice/global.json`, `../hivespace.microservice/HiveSpace.sln`
  - Add centrally managed package versions needed for Aspire AppHost, ServiceDefaults, health checks, OpenTelemetry, and hosting integrations.
  - Add solution references for `HiveSpace.AppHost` and `HiveSpace.ServiceDefaults`.
  - If Aspire requires a newer .NET 8 SDK than the current `global.json`, update `global.json` deliberately and document the required SDK in backend docs.
  - Do not add `Version=` attributes to individual `.csproj` package references.
  - Acceptance: `dotnet restore` resolves Aspire packages and the solution includes both new projects.

## HiveSpace.ServiceDefaults

### Create

- [ ] B003 [US2] Create `HiveSpace.ServiceDefaults`
  - File: `../hivespace.microservice/HiveSpace.ServiceDefaults/HiveSpace.ServiceDefaults.csproj`, `../hivespace.microservice/HiveSpace.ServiceDefaults/Extensions.cs`
  - Include ServiceDefaults extension methods for OpenTelemetry tracing, OpenTelemetry metrics, health checks, service discovery compatibility where appropriate, and Aspire dashboard export.
  - Provide `AddServiceDefaults()` and default endpoint mapping helpers that can be called from standardized API startup paths.
  - Provide `AddDefaultAuthentication(this IHostApplicationBuilder builder, string scope, Action<JwtBearerOptions>? configure = null)` that delegates to the existing `AddHiveSpaceJwtBearerAuthentication(...)` helper and keeps service scopes explicit.
  - Provide `AddDefaultOpenApi(this IServiceCollection services, string title, string description = "")` that delegates to the existing `AddHiveSpaceSwaggerGen(...)` helper and keeps service titles/descriptions explicit.
  - Preserve existing service-specific Serilog, MassTransit, EF, auth, IdentityServer, YARP, SignalR, OpenAPI, and health behavior by adding defaults conservatively.
  - Do not adopt the eShop `Identity:Url`/`Identity:Audience` configuration shape, do not replace existing `Authentication:*` keys, and do not disable audience validation.
  - Do not make logs the only monitoring signal; traces, metrics, and health/status must be wired.
  - Acceptance: project builds and API projects can reference it without changing business behavior, auth policy behavior, or OpenAPI route availability.

## HiveSpace.AppHost

### Create

- [ ] B004 [US1] [US3] Create `HiveSpace.AppHost` infrastructure resources
  - File: `../hivespace.microservice/HiveSpace.AppHost/HiveSpace.AppHost.csproj`, `../hivespace.microservice/HiveSpace.AppHost/Program.cs`
  - Model SQL Server on `localhost:1433`, RabbitMQ AMQP on `localhost:5672`, Kafka on `localhost:9092`, Redis on `localhost:6379`, and Azurite Blob/Queue/Table on `localhost:10000`, `localhost:10001`, and `localhost:10002`.
  - Configure persistent local data behavior where Aspire resource support allows normal stop/restart without deleting new-runtime data.
  - Use resource names and connection-string names that map to `ConnectionStrings` keys: service database keys, `RabbitMq`, `Redis`, `AzureServiceBus` when enabled, and `AzureStorage`.
  - Declare Kafka as an AppHost infrastructure resource on `localhost:9092`, but do not attach Kafka to any API project or function through references, waits, connection strings, or enabled broker flags.
  - Use Aspire resource references or dependency ordering where supported so dependent API projects do not require manual restart solely because local infrastructure starts slowly.
  - Do not model production deployment settings or migrate existing Docker Compose volumes.
  - Acceptance: AppHost can instantiate all required infrastructure resources, exposes fixed local endpoints, and represents dependency relationships where Aspire supports them.

- [ ] B005 [US1] [US2] [US3] Create `AppHost API project resources`
  - File: `../hivespace.microservice/HiveSpace.AppHost/Program.cs`
  - Add project resources for `src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway`, `src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api`, `src/HiveSpace.UserService/HiveSpace.UserService.Api`, `src/HiveSpace.CatalogService/HiveSpace.CatalogService.Api`, `src/HiveSpace.MediaService/HiveSpace.MediaService.Api`, `src/HiveSpace.OrderService/HiveSpace.OrderService.Api`, `src/HiveSpace.PaymentService/HiveSpace.PaymentService.Api`, and `src/HiveSpace.NotificationService/HiveSpace.NotificationService.Api`.
  - Preserve local HTTP URLs: gateway `5000`, Identity `5001`, Catalog `5002`, Media API `5003`, Order `5004`, Payment `5005`, Notification `5006`, User `5007`.
  - Inject `Authentication__Authority`, gateway `ReverseProxy__Clusters__*__Destinations__destination1__Address`, service database connection strings, `ConnectionStrings__RabbitMq`, `ConnectionStrings__Redis`, `ConnectionStrings__AzureStorage`, and broker feature flags.
  - Do not inject `ConnectionStrings__Kafka` into any service/function. Keep `Messaging__EnableKafka=false` only where needed to override legacy local settings.
  - Do not change public API paths, auth policies, integration events, or service ownership.
  - Acceptance: AppHost resource graph contains every required API project and dependency with stable local addresses.

## Shared Messaging And Configuration

### Update

- [ ] B006 Update `MessagingOptions connection-string contract`
  - File: `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging/**`
  - Refactor messaging options so `Messaging:EnableRabbitMq`, `Messaging:EnableKafka`, and `Messaging:EnableAzureServiceBus` remain feature flags.
  - Move RabbitMQ host/port/username/password and Azure Service Bus connection string reads to `ConnectionStrings:RabbitMq` and `ConnectionStrings:AzureServiceBus`.
  - Keep Kafka disabled for every v1 service/function and remove Kafka bootstrap endpoint reads from active service startup paths.
  - Keep only non-secret tuning under nested `Messaging` provider sections, such as RabbitMQ prefetch/outbox/heartbeat.
  - Do not rename existing integration contracts or change MassTransit saga messages.
  - Acceptance: messaging configuration compiles and endpoint/secret material is no longer read from nested `Messaging` provider objects.

- [ ] B007 Update `broker fail-fast validation`
  - File: `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging/**`
  - Add clear startup validation when a broker feature flag is enabled but its required `ConnectionStrings` entry is missing or empty.
  - Include broker name and missing key in the error message.
  - Preserve existing disabled-broker behavior when the corresponding feature flag is false.
  - Do not silently disable a broker that was explicitly enabled.
  - Acceptance: intentionally removing `ConnectionStrings:RabbitMq` while `Messaging:EnableRabbitMq=true` causes a clear startup failure.

- [ ] B008 Update `messaging callers for new options`
  - File: `../hivespace.microservice/src/**/appsettings*.json`, `../hivespace.microservice/src/**/Program.cs`, `../hivespace.microservice/src/**/HostingExtensions.cs`, `../hivespace.microservice/src/**/ServiceCollectionExtensions.cs`
  - Update every service that enables RabbitMQ or Azure Service Bus to pass connection-string-based values into shared messaging registration.
  - Ensure no service/function enables Kafka or requires Kafka connection-string values in v1.
  - Preserve transactional outbox, idempotent consumer registration, retry/dead-letter behavior, and existing saga configuration.
  - Do not introduce direct cross-service database reads or duplicate event contracts.
  - Acceptance: all messaging-enabled services build with the new connection-string contract.

## API Project ServiceDefaults And Startup Standardization

### Update

- [ ] B009 [US2] [US4] Update `ServiceDefaults references in API projects`
  - File: `../hivespace.microservice/src/**/**/*.Api.csproj`, `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/*.csproj`
  - Add project references to `HiveSpace.ServiceDefaults` for ApiGateway and all included API projects where compatible.
  - Wire `AddServiceDefaults()` through `ConfigureServices()` and map default endpoints through `ConfigurePipeline()` or `ConfigurePipelineAsync()`.
  - Route compatible service `AddAppAuthentication()` methods through `AddDefaultAuthentication(...)`, passing each existing service scope unchanged: `identity.fullaccess`, `user.fullaccess`, `catalog.fullaccess`, `media.fullaccess`, `order.fullaccess`, `payment.fullaccess`, and `notification.fullaccess`.
  - Route compatible service `AddAppOpenApi()` methods through `AddDefaultOpenApi(...)`, passing existing service titles and descriptions unchanged.
  - Keep NotificationService SignalR JWT query-token handling in its local `AddAppAuthentication()` callback passed to the wrapper.
  - Keep IdentityService Google external auth, cookies, ASP.NET Identity, and IdentityServer setup local; only standard bearer API registration may use the wrapper.
  - Keep ApiGateway outside the default authentication wrapper because its token validation, CSRF, cookie/session forwarding, and YARP behavior are gateway-specific.
  - Do not add per-service inline Aspire setup in `Program.cs`.
  - Do not change public API paths, auth policy names, service scopes, token format, or generated business OpenAPI contract intent.
  - Acceptance: each included API project compiles with ServiceDefaults wired through the standardized startup path, and compatible services use the default auth/OpenAPI wrappers without behavior changes.

- [ ] B010 [US4] Update `ApiGateway startup structure`
  - File: `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/Program.cs`, `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/HostingExtensions.cs`, `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/ServiceCollectionExtensions.cs`
  - Move inline YARP, CORS, token validation, health checks, CSRF/session middleware, WebSockets, and reverse proxy mapping out of `Program.cs`.
  - Keep `Program.cs` limited to builder creation, `ConfigureServices()`, `ConfigurePipeline()`, and `Run()`.
  - Preserve cookie mediation, forwarded authorization, CSRF rejection, route table behavior, and `/health`.
  - Do not route IdentityServer direct authority endpoints through the gateway.
  - Acceptance: gateway health and representative downstream routing still work through `http://localhost:5000`.

- [ ] B011 [US4] Update `IdentityService startup structure`
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Program.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/HostingExtensions.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/ServiceCollectionExtensions.cs`
  - Normalize startup file roles while preserving Duende IdentityServer, ASP.NET Identity, Razor compatibility redirects, cookie/session behavior, seed behavior, and MassTransit consumers.
  - Keep direct authority endpoints on `http://localhost:5001`.
  - Do not expose tokens to browser scripts or alter account/public OIDC contracts.
  - Acceptance: `/.well-known/openid-configuration` responds and identity API behavior is unchanged.

- [ ] B012 [US4] Update `UserService startup structure`
  - File: `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Program.cs`, `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/HostingExtensions.cs`, `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/ServiceCollectionExtensions.cs`
  - Normalize startup file roles while preserving controller behavior, profile/settings/address/store routes, existing migrations/seeding, and integration event consumers/publishers.
  - Keep local URL `http://localhost:5007`.
  - Do not move identity-owned credentials, roles, claims, or email verification behavior into UserService.
  - Acceptance: UserService builds and existing gateway prefixes remain service-owned.

- [ ] B013 [US4] Update `CatalogService startup structure`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Api/Program.cs`, `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Api/HostingExtensions.cs`, `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Api/ServiceCollectionExtensions.cs`
  - Normalize startup file roles while preserving CQRS handlers, Minimal API endpoint modules, EF setup, MassTransit product/store projection consumers, and local URL `http://localhost:5002`.
  - Do not change product/category public endpoints or catalog ownership rules.
  - Acceptance: CatalogService builds and existing `/api/v1/products` and `/api/v1/categories` ownership is unchanged.

- [ ] B014 [US4] Update `MediaService API startup structure`
  - File: `../hivespace.microservice/src/HiveSpace.MediaService/HiveSpace.MediaService.Api/Program.cs`, `../hivespace.microservice/src/HiveSpace.MediaService/HiveSpace.MediaService.Api/HostingExtensions.cs`, `../hivespace.microservice/src/HiveSpace.MediaService/HiveSpace.MediaService.Api/ServiceCollectionExtensions.cs`
  - Normalize startup file roles while preserving upload URL generation, confirm-upload routes, Azure Storage/Azurite configuration, migration-only seeding behavior, and local URL `http://localhost:5003`.
  - Keep media processing behavior unchanged.
  - Do not change `/api/v1/media` request/response contracts.
  - Acceptance: MediaService API builds and media public API contracts remain unchanged.

- [ ] B015 [US4] Update `OrderService startup structure`
  - File: `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Api/Program.cs`, `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Api/HostingExtensions.cs`, `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Api/ServiceCollectionExtensions.cs`
  - Normalize startup file roles while preserving checkout saga, fulfillment saga, cart/order/coupon endpoint modules, EF setup, MassTransit, and local URL `http://localhost:5004`.
  - Confirm Kafka remains disabled and unused by OrderService in v1.
  - Do not change saga states, commands, events, compensation, or public order/cart/coupon endpoints.
  - Acceptance: OrderService builds and saga/event contracts are unchanged.

- [ ] B016 [US4] Update `PaymentService startup structure`
  - File: `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Api/Program.cs`, `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Api/HostingExtensions.cs`, `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Api/ServiceCollectionExtensions.cs`
  - Normalize startup file roles while preserving VNPay redirect/IPN handling, wallet routes, payment idempotency behavior, messaging setup, and local URL `http://localhost:5005`.
  - Do not change gateway callback routes or payment integration event contracts.
  - Acceptance: PaymentService builds and existing payment/wallet route contracts remain unchanged.

- [ ] B017 [US4] Update `NotificationService startup structure`
  - File: `../hivespace.microservice/src/HiveSpace.NotificationService/HiveSpace.NotificationService.Api/Program.cs`, `../hivespace.microservice/src/HiveSpace.NotificationService/HiveSpace.NotificationService.Api/HostingExtensions.cs`, `../hivespace.microservice/src/HiveSpace.NotificationService/HiveSpace.NotificationService.Api/ServiceCollectionExtensions.cs`
  - Normalize startup file roles while preserving Serilog bootstrap, SignalR authentication, notification REST routes, Redis/cache setup, email delivery setup, MassTransit consumers, and local URL `http://localhost:5006`.
  - Keep `/hubs/notifications` and `ReceiveNotification` behavior unchanged.
  - Do not change notification delivery decisions or preference APIs.
  - Acceptance: NotificationService builds and SignalR/notification contracts remain unchanged.

- [ ] B018 [US2] [US4] Update `service health and telemetry mapping`
  - File: `../hivespace.microservice/src/**/HostingExtensions.cs`, `../hivespace.microservice/src/**/ServiceCollectionExtensions.cs`
  - Ensure all compatible included API projects expose health/status through the standardized pipeline and emit Aspire-compatible OpenTelemetry traces and metrics.
  - Preserve existing service-specific health checks and logging as supplemental diagnostics.
  - Do not remove existing health endpoints without replacing them with an equivalent path used by quickstart checks.
  - Acceptance: Aspire dashboard shows compatible services with traces/metrics after a gateway-to-service request.

## MediaService Function

### Update

- [ ] B019 [US1] Update `MediaService Function AppHost orchestration`
  - File: `../hivespace.microservice/src/HiveSpace.MediaService/**`
  - Add MediaService Function to the AppHost runtime as a local function/executable resource using Azure Functions Core Tools on port `7072`.
  - Wire the function to the AppHost-managed MediaService database, RabbitMQ, and Azurite blob/queue resources through connection-string-compatible configuration.
  - Document Azure Functions Core Tools as a local prerequisite wherever backend runtime prerequisites are listed.
  - Preserve image-processing behavior, queue trigger payload shape, storage container/queue names, and published media events.
  - Do not change `/api/v1/media` request/response contracts or add public function endpoints to the API catalog.
  - Acceptance: AppHost resource graph includes MediaService Function, the function starts with the full runtime, and implementation summary confirms processing behavior and contracts are unchanged.

## Backend Appsettings

### Update

- [ ] B020 [US3] Update `backend local appsettings connection-string shape`
  - File: `../hivespace.microservice/src/**/appsettings*.json`, `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/appsettings*.json`
  - Move broker endpoints/secrets to `ConnectionStrings:RabbitMq`, `ConnectionStrings:AzureServiceBus`, and `ConnectionStrings:AzureStorage` as applicable.
  - Keep `Messaging:EnableRabbitMq`, `Messaging:EnableKafka`, and `Messaging:EnableAzureServiceBus` under `Messaging`.
  - Set `Messaging:EnableKafka=false` only where needed to override legacy defaults; no service/function may enable Kafka in v1.
  - Remove nested endpoint/secret fields such as RabbitMQ host/port/username/password, Kafka bootstrap servers, and Azure Service Bus connection strings from `Messaging`.
  - Remove service/function `ConnectionStrings:Kafka` entries because Kafka is declared by AppHost but unused by v1 services.
  - Preserve gateway downstream addresses, service local ports, and all public API route prefixes.
  - Acceptance: static search finds no forbidden nested broker endpoint/secret keys in service appsettings.
