# Quickstart: Local Backend Runtime Orchestration

This quickstart describes the intended developer workflow after implementation.

## Prerequisites

- Backend repo available at `../hivespace.microservice`.
- .NET SDK compatible with the backend `global.json` and Aspire projects.
- A local container runtime supported by Aspire.
- Required development secrets and local appsettings already configured as they are today.

## Start Backend Runtime

From `../hivespace.microservice`:

```powershell
dotnet run --project .\HiveSpace.AppHost\HiveSpace.AppHost.csproj
```

Expected result:

- Aspire dashboard/runtime view opens or prints its local URL.
- API projects and infrastructure dependencies start from one entry point.
- OpenTelemetry traces and metrics appear in the Aspire dashboard for compatible backend services after requests run.
- Existing gateway address remains `http://localhost:5000`.
- Included API projects use the standard startup extension path for AppHost, ServiceDefaults, health, and OpenTelemetry wiring.
- Infrastructure endpoints and secrets are injected through `ConnectionStrings`, with broker runtime flags under `Messaging`.

## Smoke Checks

```powershell
Invoke-WebRequest http://localhost:5000/health
Invoke-WebRequest http://localhost:5001/.well-known/openid-configuration
```

Expected result:

- Gateway health responds.
- Identity authority metadata responds.
- Runtime view shows included services and dependencies with visible status.
- Runtime view shows OpenTelemetry traces and metrics for compatible backend services after the smoke checks run.

## Startup Structure Check

For each included API project, inspect startup files before considering the runtime setup complete:

- `Program.cs` should only create the builder, call startup extensions, run development migration/seed where needed, and run the app.
- `HostingExtensions.cs` should own `ConfigureServices()` plus `ConfigurePipeline()` or `ConfigurePipelineAsync()`.
- `ServiceCollectionExtensions.cs` should own thin `AddApp*()` wrappers.

Expected result:

- ApiGateway and all backend API services expose the same startup file roles where technically possible.
- Service-specific behavior is preserved for IdentityServer, UserService controllers, NotificationService SignalR/auth/logging, ApiGateway YARP/session/CSRF, and MediaService migration-only seeding.

## Dependency Configuration Check

Before considering the runtime setup complete, verify the configuration shape:

- RabbitMQ endpoint and credentials are supplied by `ConnectionStrings:RabbitMq`.
- Kafka bootstrap endpoint is supplied by `ConnectionStrings:Kafka`.
- Azure Service Bus endpoint/secret is supplied by `ConnectionStrings:AzureServiceBus` when enabled.
- Redis and Azure Storage/Azurite are supplied by `ConnectionStrings:Redis` and `ConnectionStrings:AzureStorage`.
- `Messaging:EnableRabbitMq`, `Messaging:EnableKafka`, and `Messaging:EnableAzureServiceBus` remain the runtime broker switches.
- Nested `Messaging` provider sections contain only non-secret tuning.

## Stop And Restart

Stop the AppHost process from the terminal or IDE.

Restart with the same command:

```powershell
dotnet run --project .\HiveSpace.AppHost\HiveSpace.AppHost.csproj
```

Expected result:

- Runtime starts again without service-by-service commands.
- Data created by the new runtime survives normal restart when persistent volumes are configured.
- Existing Docker Compose local data is not migrated.

## Troubleshooting Expectations

- If a required port is already in use, the runtime should report the conflicting resource.
- If a dependency is unhealthy, the runtime view should identify the dependency and affected services.
- If frontend apps are running separately, they should continue using the existing gateway base URL.
