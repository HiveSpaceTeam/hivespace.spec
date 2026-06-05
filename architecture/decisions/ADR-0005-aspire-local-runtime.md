# ADR-0005: Aspire Local Backend Runtime

- **Status**: Draft
- **Date**: 2026-06-04
- **Feature**: `0006-aspire-setup`
- **Deciders**: Project maintainers

## Context

HiveSpace backend local development currently depends on service-by-service `dotnet run` commands plus Docker Compose-managed infrastructure. The backend has a .NET solution with ApiGateway and seven API services that depend on SQL Server, RabbitMQ, Kafka, Redis, and Azurite. The feature requirement is to let developers start and observe the backend stack from one local runtime workflow, and clarification states that the new runtime fully replaces Docker Compose for backend local development.

The runtime must preserve current local addresses so frontend apps and documented manual checks continue to work. Existing Docker Compose local data does not need migration, but data created by the new runtime should survive normal stop/restart workflows where supported.

The backend also has existing startup drift across `Program.cs`, `HostingExtensions.cs`, and `ServiceCollectionExtensions.cs`. Aspire ServiceDefaults, OpenTelemetry, and health wiring should use a consistent startup surface instead of adding more per-service inline setup.

Current messaging configuration uses nested RabbitMQ and Kafka provider objects for endpoint details, while databases and Redis already use `ConnectionStrings`. Aspire and deployment platforms fit better when dependency endpoints and secrets are exposed as connection strings.

## Decision

Use a project-based .NET Aspire AppHost in `../hivespace.microservice` as the backend local development runtime. The AppHost will launch ApiGateway, backend API services, and local infrastructure dependencies, while preserving current localhost ports. Add a ServiceDefaults project conservatively for local OpenTelemetry traces, metrics, health behavior, and Aspire dashboard export without replacing service-specific auth, messaging, persistence, logging, or gateway logic.

Standardize startup file roles across all included API projects where technically possible: `Program.cs` as entry point only, `HostingExtensions.cs` for service/pipeline orchestration, and `ServiceCollectionExtensions.cs` for thin `AddApp*()` wrappers. ApiGateway must be brought into the same structure while preserving YARP, session forwarding, CSRF, CORS, token validation, and health behavior.

Standardize dependency endpoint configuration around `ConnectionStrings`: databases, Redis, RabbitMQ, Kafka, Azure Service Bus, and Azure Storage/Azurite all expose endpoints/secrets through connection-string keys. Keep `Messaging:EnableRabbitMq`, `Messaging:EnableKafka`, and `Messaging:EnableAzureServiceBus` as runtime feature flags. Keep only non-secret broker tuning under nested `Messaging` provider sections.

Docker Compose is replaced as the backend local development entry point. Existing Docker Compose local data will not be migrated into the Aspire-managed runtime.

## Consequences

### Positive

- Developers get one supported backend startup flow.
- Service and dependency status, OpenTelemetry traces, metrics, and logs are visible from one local runtime view.
- Service startup wiring becomes consistent across AppHost, ServiceDefaults, health, and OpenTelemetry integration.
- Aspire and deployment configuration can inject dependency endpoints and secrets through one connection-string convention.
- Existing frontend gateway assumptions remain stable.
- Backend runtime setup becomes part of the .NET solution rather than a separate compose-only workflow.

### Negative / Trade-offs

- Developers need Aspire-compatible .NET tooling and a supported local container runtime.
- Fixed ports can conflict with existing local processes more often than fully dynamic ports.
- Docker Compose data created before this feature is not automatically available in the new runtime.

### Risks

- SDK pinning in backend `global.json` may need adjustment if Aspire templates/packages require a newer .NET 8 SDK. Mitigation: verify tooling before implementation and update/document the SDK deliberately if needed.
- ServiceDefaults could conflict with existing health/logging/middleware if applied broadly. Mitigation: integrate conservatively, preserve existing service-specific behavior, and treat logs as diagnostics rather than the primary monitoring mechanism.
- Startup standardization could accidentally alter gateway, identity, notification, or legacy controller behavior. Mitigation: move wiring without changing behavior and verify gateway, identity metadata, health, and representative routed calls.
- Messaging config migration could break broker startup if feature flags and connection strings drift. Mitigation: fail fast when an enabled broker lacks a required connection string and smoke-test RabbitMQ/Kafka/Azure Service Bus paths where enabled.
- Kafka/Azurite hosting details may require package or container configuration differences. Mitigation: validate each infrastructure resource with AppHost smoke checks.

## Alternatives Considered

| Option | Why rejected |
| --- | --- |
| Keep Docker Compose and add Aspire only for API projects | Does not satisfy the clarified replacement requirement. |
| Keep Docker Compose as a fallback workflow | Lower migration risk, but maintains duplicate backend startup workflows. |
| Use dynamic ports and service discovery everywhere | More idiomatic for orchestration but would force broader config/frontend changes than the feature requires. |
| Add Aspire hooks without startup standardization | Lower immediate refactor cost, but leaves per-service drift around ServiceDefaults, health, and OpenTelemetry wiring. |
| Keep nested messaging endpoint objects | Avoids config migration, but keeps broker endpoints/secrets outside the connection-string convention used by Aspire and deployment platforms. |
| Encode broker feature flags into connection strings | Reduces one config section, but makes runtime broker selection less explicit. |
| Use a file-based AppHost | Less solution-integrated for an existing multi-project .NET backend. |
| Use a TypeScript AppHost | Better fit for Node-centered repos, not the backend .NET solution. |

## Follow-Up

- Add AppHost and ServiceDefaults projects in `../hivespace.microservice`.
- Normalize included API startup files against `docs/agent/startup-conventions.md`.
- Move broker endpoints/secrets to `ConnectionStrings` while retaining `Messaging` broker feature flags and non-secret tuning.
- Verify Aspire-backed OpenTelemetry traces and metrics in the local dashboard.
- Update backend run docs and agent instructions.
- Keep API and event catalogs unchanged unless implementation discovers an actual contract change.
