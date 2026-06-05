# Verification Tasks: Local Backend Runtime Orchestration

## Backend Static Verification

### Verify

- [ ] V001 [US1] Verify `backend restore and build`
  - File: `../hivespace.microservice/HiveSpace.sln`
  - Run `dotnet restore` from `../hivespace.microservice`.
  - Run `dotnet build` from `../hivespace.microservice`.
  - Capture any known baseline warnings separately from new errors caused by Aspire or startup changes.
  - Do not fix unrelated source issues outside the feature scope.
  - Acceptance: restore and build complete, or remaining failures are clearly documented as pre-existing/unrelated.

- [ ] V002 [US4] Verify `startup file roles`
  - File: `../hivespace.microservice/src/**/Program.cs`, `../hivespace.microservice/src/**/HostingExtensions.cs`, `../hivespace.microservice/src/**/ServiceCollectionExtensions.cs`
  - Inspect ApiGateway and all included API projects for `Program.cs` as entry point only.
  - Confirm service registration and middleware/endpoint setup live in `HostingExtensions.cs` and `ServiceCollectionExtensions.cs`.
  - Confirm unavoidable service-specific exceptions are documented in code or backend docs.
  - Do not accept scattered inline Aspire setup in `Program.cs`.
  - Acceptance: 100% of included API projects follow the startup convention or document a concrete technical exception.

- [ ] V003 [US1] [US3] Verify `fixed local addresses`
  - File: `../hivespace.microservice/HiveSpace.AppHost/Program.cs`, `../hivespace.microservice/src/**/Properties/launchSettings.json`, `../hivespace.microservice/src/**/appsettings*.json`
  - Confirm AppHost and service launch/config files preserve gateway `5000`, Identity `5001`, Catalog `5002`, Media API `5003`, Order `5004`, Payment `5005`, Notification `5006`, and User `5007`.
  - Confirm infrastructure endpoints preserve SQL Server `1433`, RabbitMQ `5672`, Kafka `9092`, Redis `6379`, and Azurite `10000`, `10001`, `10002`.
  - Do not switch frontend or documented local checks to dynamic ports.
  - Acceptance: static inspection shows all required ports preserved.

- [ ] V004 Verify `connection-string dependency contract`
  - File: `../hivespace.microservice/src/**/appsettings*.json`, `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging/**`
  - Search for forbidden nested endpoint/secret keys: RabbitMQ host/port/username/password under `Messaging`, Kafka bootstrap servers under `Messaging`, and Azure Service Bus connection strings under `Messaging`.
  - Confirm required keys are supplied under `ConnectionStrings`: `RabbitMq`, `Kafka`, `AzureServiceBus` when enabled, `Redis`, and `AzureStorage`.
  - Confirm broker flags remain under `Messaging:EnableRabbitMq`, `Messaging:EnableKafka`, and `Messaging:EnableAzureServiceBus`.
  - Acceptance: static search confirms endpoint/secret material uses `ConnectionStrings` and feature flags remain under `Messaging`.

## AppHost Runtime Verification

### Verify

- [ ] V005 [US2] Verify `Aspire dashboard resources and telemetry`
  - File: `../hivespace.microservice/HiveSpace.AppHost/Program.cs`, `../hivespace.microservice/HiveSpace.ServiceDefaults/Extensions.cs`
  - Run `dotnet run --project .\HiveSpace.AppHost\HiveSpace.AppHost.csproj` from `../hivespace.microservice`.
  - Open the Aspire dashboard/runtime view and confirm all included API projects and infrastructure resources appear with visible status.
  - Trigger at least one gateway-to-service request and confirm OpenTelemetry traces and metrics appear for compatible services.
  - Do not count logs alone as satisfying telemetry acceptance.
  - Acceptance: dashboard shows service/dependency status plus traces and metrics for a successful request.

- [ ] V006 [US1] [US3] Verify `quickstart smoke checks`
  - File: `specs/0006-aspire-setup/quickstart.md`
  - With AppHost running, run `Invoke-WebRequest http://localhost:5000/health`.
  - Run `Invoke-WebRequest http://localhost:5001/.well-known/openid-configuration`.
  - Check additional service health endpoints where available.
  - Do not require frontend app configuration changes for these checks.
  - Acceptance: gateway health and Identity authority metadata respond from existing local addresses.

- [ ] V007 [US2] Verify `startup failure observability`
  - File: `../hivespace.microservice/HiveSpace.AppHost/Program.cs`
  - Test a representative failure, such as a required port already in use or a dependency intentionally unavailable.
  - Confirm the runtime view or startup output identifies the failing service/dependency within the same local workflow.
  - Do not require checking each service manually to locate the failure.
  - Acceptance: failure source is visible from AppHost output or Aspire dashboard.

- [ ] V008 [US3] Verify `broker missing-connection failure`
  - File: `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging/**`, `../hivespace.microservice/src/**/appsettings*.json`
  - Temporarily omit `ConnectionStrings:RabbitMq` while `Messaging:EnableRabbitMq=true` for one broker-enabled service.
  - Start the service or AppHost path and confirm startup fails with a clear missing-key error.
  - Restore the connection string after the check.
  - Do not silently disable the enabled broker.
  - Acceptance: enabled broker with missing connection string fails fast and clearly.

## Behavior And Contract Verification

### Verify

- [ ] V009 [US4] Verify `service-specific behavior preservation`
  - File: `../hivespace.microservice/src/**`
  - Check ApiGateway YARP/session/CSRF behavior, IdentityService direct authority metadata, UserService controller routes, NotificationService SignalR authentication, MediaService migration-only seeding, and OrderService saga registration after startup normalization.
  - Use existing tests where available or targeted manual checks when tests are not present.
  - Do not accept startup convention compliance if service-specific behavior regressed.
  - Acceptance: representative checks show preserved service behavior after startup restructuring.

- [ ] V010 [US1] Verify `new-runtime stop and restart`
  - File: `../hivespace.microservice/HiveSpace.AppHost/Program.cs`
  - Start AppHost, create or observe local runtime data where practical, stop the AppHost process, then restart with the same command.
  - Confirm the stack starts again without service-by-service commands and without deleting Aspire-managed local data where persistent resources are configured.
  - Do not migrate or depend on existing Docker Compose data.
  - Acceptance: normal stop/restart works from the AppHost command and data persistence behavior matches documentation.

- [ ] V011 Verify `final catalog and source impact checks`
  - File: `shared/api-catalog.md`, `shared/event-catalog.md`, `../hivespace.microservice`
  - Confirm API and event catalogs have no unjustified changes.
  - Run `npx gitnexus analyze` in `../hivespace.microservice` before PR if backend code changes are committed and the backend repo workflow supports it.
  - Review changed files for accidental frontend orchestration, public contract changes, or service boundary changes.
  - Acceptance: final review shows no unintended API/event/service-boundary changes and any GitNexus findings are addressed or documented.

