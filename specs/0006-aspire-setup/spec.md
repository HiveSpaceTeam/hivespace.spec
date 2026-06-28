# Feature Specification: Local Backend Runtime Orchestration

- **Feature Branch**: `0006-aspire-setup`
- **Created**: 2026-06-04
- **Status**: Implemented
- **Implemented**: 2026-06-07
- **Input**: User description: "Set up Aspire for the HiveSpace backend microservice solution so local developers can launch the backend services and required infrastructure together."

## Clarifications

### Session 2026-06-04

- Q: How should the new local runtime relate to the existing Docker Compose backend development workflow? -> A: Fully replace Docker Compose for backend local development.
- Q: How should local development data be handled when replacing Docker Compose? -> A: Preserve data across new-runtime restarts only; no migration from existing Docker Compose data.

### Session 2026-06-05

- Q: How strict should the startup standardization requirement be for this Aspire feature? -> A: Strict all services; every included API project should use the same startup file roles wherever technically possible.
- Q: How should dependency connection details be shaped for Aspire and deployment while keeping runtime broker selection? -> A: Use `ConnectionStrings` for endpoints and secrets, keep `Messaging:EnableRabbitMq`, `Messaging:EnableKafka`, and `Messaging:EnableAzureServiceBus` as runtime feature flags, and keep only non-secret broker tuning under `Messaging`.
- Q: Should ServiceDefaults include authentication and OpenAPI setup? -> A: Add optional default authentication and OpenAPI wrapper helpers that delegate to existing HiveSpace shared helpers, keep per-service auth scopes explicit, preserve service-specific auth/OpenAPI exceptions, and do not adopt the eShop `Identity:*` config shape or disable audience validation.

### Session 2026-06-07

- Q: Should the MediaService Azure Function be part of the v1 local backend runtime scope? -> A: Include MediaService Function in v1 AppHost orchestration, require Azure Functions Core Tools locally, and preserve function processing behavior while wiring it to the same media database, RabbitMQ, and Azurite resources.
- Q: How should Kafka be handled in the v1 local runtime? -> A: Declare Kafka as an AppHost-managed local infrastructure resource on the existing port for future compatibility, but do not use, reference, wait for, inject, or enable Kafka in any backend service for v1.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Start backend stack from one place (Priority: P1)

As a HiveSpace developer, I want one documented local backend startup flow so I can run the backend services and their required dependencies without manually starting each service one by one.

**Why this priority**: This is the core value of the feature. Without a unified startup flow, local development remains slow, error-prone, and dependent on individual service knowledge.

**Independent Test**: A developer can follow the documented startup flow from a clean local checkout and reach a running backend stack that includes the gateway, backend services, MediaService Function, and required local dependencies.

**Acceptance Scenarios**:

1. **Given** a developer has the required local tooling installed, **When** they start the backend runtime using the documented flow, **Then** all required backend services, MediaService Function, and infrastructure dependencies begin startup from one entry point.
2. **Given** the backend runtime is starting, **When** a required dependency is unavailable or fails to start, **Then** the developer can identify which dependency failed without checking each service manually.

---

### User Story 2 - Monitor runtime telemetry in one view (Priority: P2)

As a HiveSpace developer, I want a consolidated local runtime view so I can monitor service health, OpenTelemetry traces, metrics, logs, and dependency status while debugging backend flows.

**Why this priority**: Running many services is only useful if developers can quickly see which service or dependency is unhealthy and inspect telemetry for the backend flow under development.

**Independent Test**: A developer can open the local runtime view and determine whether each backend service and infrastructure dependency is running, unhealthy, or stopped, then inspect OpenTelemetry traces and metrics for compatible backend services.

**Acceptance Scenarios**:

1. **Given** the local backend runtime is active, **When** the developer opens the runtime view, **Then** they can see every backend service, MediaService Function, and required dependency with a clear status.
2. **Given** a backend request or message flow runs through an Aspire-compatible service, **When** the developer inspects the runtime view, **Then** they can inspect OpenTelemetry traces and metrics without relying on logs as the primary monitoring signal.
3. **Given** a service fails during startup, **When** the developer inspects the runtime view, **Then** they can find the service status and recent logs from the same local workflow.

---

### User Story 3 - Preserve existing local client assumptions (Priority: P3)

As a HiveSpace developer, I want the existing local gateway URL and backend service ports to continue working so frontend apps and documented manual checks do not need a separate migration.

**Why this priority**: The new runtime flow should improve developer productivity without forcing unrelated frontend or API contract changes.

**Independent Test**: After starting the new local backend runtime, existing frontend apps can still reach the backend through `http://localhost:5000`, and documented backend service ports remain usable for manual checks.

**Acceptance Scenarios**:

1. **Given** a frontend app is configured to call `http://localhost:5000`, **When** the backend runtime is started through the new flow, **Then** the frontend app can still call the gateway without changing its local configuration.
2. **Given** a developer uses existing documented local service URLs for manual checks, **When** the backend runtime is active, **Then** those URLs remain valid for the same service responsibilities.

---

### User Story 4 - Standardize backend service startup structure (Priority: P3)

As a HiveSpace backend maintainer, I want every included API project to follow the same startup file roles so Aspire ServiceDefaults, OpenTelemetry, health checks, and future runtime changes can be applied consistently.

**Why this priority**: The current backend has startup drift across service entry points, hosting setup, and service-registration helpers. Aspire integration should not add another cross-service pattern on top of inconsistent startup wiring.

**Independent Test**: A maintainer can inspect every included API project and verify startup wiring follows the repository startup convention while preserving each service's existing behavior.

**Acceptance Scenarios**:

1. **Given** an included API project, **When** a maintainer inspects its startup wiring, **Then** entry point, service registration, pipeline setup, and service-specific extension roles follow the repository startup convention.
2. **Given** an included API project has shared runtime defaults applied, **When** a maintainer inspects its startup wiring, **Then** the shared defaults are applied through the standard startup path rather than ad hoc inline setup.
3. **Given** ApiGateway or a legacy-shaped service has service-specific startup behavior, **When** startup is standardized, **Then** existing routing, auth, session, CSRF, IdentityServer, controllers, SignalR, logging, and seeding behavior remains unchanged.

### Edge Cases

- A required local port is already in use before startup.
- A required local dependency starts slower than the backend service that needs it.
- A service starts but is unhealthy because its owned database or broker dependency is unavailable.
- The local runtime is stopped and restarted without deleting data created by the new runtime.
- Existing Docker Compose local data is present before the new runtime is introduced.
- A service has a legitimate service-specific startup requirement while still needing the standard startup file roles.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST provide a single documented local startup flow for the backend services and required local infrastructure dependencies.
- **FR-002**: The local runtime MUST include the browser-facing gateway and every backend service listed in the architecture overview.
- **FR-003**: The local runtime MUST include required local dependencies for service databases, messaging, caching, and media storage.
- **FR-004**: Developers MUST be able to view health or running status for each backend service and required dependency from one local runtime workflow.
- **FR-005**: Developers MUST be able to access OpenTelemetry traces and metrics for backend services integrated with Aspire ServiceDefaults from the local runtime workflow.
- **FR-006**: The local runtime MUST preserve the existing frontend gateway base URL `http://localhost:5000` and documented backend service ports after the new runtime flow is introduced.
- **FR-007**: The feature MUST preserve existing service ownership boundaries and must not introduce new business APIs, user-facing routes, integration events, or service responsibilities.
- **FR-008**: The feature MUST document the new local runtime as the replacement for the existing Docker Compose backend local development workflow.
- **FR-009**: The feature MUST make startup failures observable enough for developers to identify the failing service or dependency.
- **FR-010**: The feature MUST support normal stop and restart workflows without requiring developers to delete local development data.
- **FR-011**: The feature MUST NOT require migration of existing Docker Compose local data into the new runtime.
- **FR-012**: Backend logs MUST remain available as supplemental diagnostics, but logs alone MUST NOT satisfy the local monitoring requirement.
- **FR-013**: The local runtime SHOULD preserve correlation context across HTTP requests and async messages where existing service behavior and Aspire/OpenTelemetry integration support it.
- **FR-014**: Included backend API projects MUST use a consistent startup structure that allows shared runtime defaults, health checks, telemetry, and future startup changes to be applied predictably while preserving documented service-specific exceptions.
- **FR-015**: ApiGateway MUST follow the same startup convention wherever technically possible while preserving existing gateway routing, CORS, token validation, health check, session, CSRF, WebSocket, and reverse proxy behavior.
- **FR-016**: Startup standardization MUST preserve existing service-specific behavior for IdentityService, UserService, NotificationService, ApiGateway, and MediaService API, including auth, routing, middleware, logging, controllers, SignalR, migration, and seeding behavior.
- **FR-017**: Aspire ServiceDefaults and OpenTelemetry setup MUST be applied consistently through each compatible service's startup path rather than through ad hoc per-service wiring.
- **FR-018**: Infrastructure dependency endpoints and secrets MUST be configured through `ConnectionStrings`, including service databases, Redis, RabbitMQ, Azure Service Bus when enabled, and Azure Storage/Azurite.
- **FR-019**: Messaging broker runtime selection MUST remain controlled by `Messaging:EnableRabbitMq`, `Messaging:EnableKafka`, and `Messaging:EnableAzureServiceBus`.
- **FR-020**: Nested `Messaging` provider objects MUST NOT contain broker endpoint or secret fields such as RabbitMQ host/port/username/password, Kafka bootstrap servers, or Azure Service Bus connection strings.
- **FR-021**: Non-secret broker tuning MAY remain under `Messaging` provider sections for active brokers, such as RabbitMQ prefetch/outbox/heartbeat settings. Kafka tuning MUST NOT remain in v1 service/function configuration because Kafka is declared but unused.
- **FR-022**: Services MUST fail startup with a clear configuration error when a broker feature flag is enabled and the required `ConnectionStrings` entry is missing.
- **FR-023**: Shared authentication and OpenAPI startup helpers MAY be introduced only when they preserve existing HiveSpace authentication, authorization scopes, OpenAPI metadata, callbacks, and service-specific exceptions.
- **FR-024**: Authentication configuration MUST preserve the existing HiveSpace authentication contract and MUST NOT weaken configured token audience validation.
- **FR-025**: ApiGateway MUST preserve its gateway-specific authentication, cookie/session forwarding, CSRF, and reverse proxy behavior instead of being forced through shared defaults intended for ordinary API services.
- **FR-026**: The local runtime MUST include MediaService Function as an AppHost-managed function resource on the existing local function port, using Azure Functions Core Tools and the same media database, RabbitMQ, and Azurite resources as the MediaService API.
- **FR-027**: MediaService Function orchestration MUST NOT change image-processing behavior, queue payload shape, media API request/response contracts, or integration event contracts.
- **FR-028**: Kafka MUST be declared as a local AppHost infrastructure resource for future compatibility, but no backend service or function may use Kafka, reference Kafka, wait for Kafka, receive `ConnectionStrings:Kafka`, or enable `Messaging:EnableKafka` in v1.

### Key Entities

- **Local Runtime**: The developer-facing backend startup experience that groups services and dependencies.
- **Backend Service**: A HiveSpace API or gateway process that participates in local backend development.
- **MediaService Function**: The Azure Functions worker that processes media queue messages during local development and participates in the AppHost runtime without becoming a public API.
- **Infrastructure Dependency**: A local database, broker, cache, or storage dependency required by one or more backend services.
- **Runtime Status**: The visible state of a service or dependency, such as starting, running, unhealthy, stopped, or failed.
- **OpenTelemetry Monitoring**: Local traces and metrics emitted by backend services through Aspire-compatible telemetry configuration and surfaced in the runtime view.
- **Startup Convention**: The backend API startup file roles defined by `../hivespace.microservice/docs/agent/startup-conventions.md`.
- **Dependency Connection String**: A `ConnectionStrings` entry that contains a local or deployment endpoint and any required secret material for an infrastructure dependency.
- **Messaging Feature Flag**: A runtime `Messaging` boolean that enables or disables a broker integration without carrying endpoint or credential data.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A developer can start the local backend stack from a clean checkout using one documented command or launch action.
- **SC-002**: 100% of backend services listed in the architecture overview plus MediaService Function are represented in the local runtime workflow.
- **SC-003**: 100% of required local infrastructure dependencies for the represented backend services are represented in the local runtime workflow.
- **SC-004**: A developer can identify the running or failed status of any backend service or required dependency within 30 seconds of opening the local runtime view.
- **SC-005**: Existing frontend apps can continue using the current local gateway URL without configuration changes.
- **SC-006**: The local runtime documentation is sufficient for a developer to start, stop, and restart the backend stack without service-by-service startup instructions.
- **SC-007**: A developer can inspect OpenTelemetry traces and metrics for at least one successful gateway-to-service request in the local runtime view without using log search as the primary monitoring mechanism.
- **SC-008**: 100% of included API projects either follow the repository startup convention or document a concrete technical reason why a specific startup step must remain different.
- **SC-009**: 100% of configured infrastructure endpoints and secrets used by the local runtime are supplied through `ConnectionStrings`, while broker enablement remains controlled through `Messaging` feature flags.

## Assumptions

- The target users are HiveSpace developers running the backend locally.
- The first version covers backend services, MediaService Function, and local infrastructure dependencies only; frontend dev servers remain outside this feature.
- This feature improves local development orchestration and OpenTelemetry-based monitoring; it does not define production deployment behavior.
- The new local runtime replaces Docker Compose for backend local development.
- Local development data created by the new runtime should survive normal stop and restart workflows, but existing Docker Compose local data is not migrated.
- Existing public API paths, service ownership, integration events, and domain behavior remain unchanged.
- Developers have the required local platform tooling installed before using the runtime flow.
- Developers have Azure Functions Core Tools installed before using the full AppHost runtime because MediaService Function is included in v1.
- Partial runtime startup is not a supported v1 workflow; developers should start the full backend runtime from the AppHost command.
- Startup standardization is part of this Aspire feature because consistent ServiceDefaults, OpenTelemetry, health, and AppHost integration depend on a consistent service startup surface.
- Connection-string standardization is part of this Aspire feature because Aspire resource references and deployment platforms naturally expose dependencies as connection strings, while broker feature flags still provide runtime selection.
- ServiceDefaults authentication and OpenAPI wrappers are startup consistency helpers only; they do not change public route ownership, token format, authorization policy names, service scopes, OpenAPI endpoint availability, or generated business contracts.
