# Research: Local Backend Runtime Orchestration

## Decision: Use a project-based Aspire AppHost

**Rationale**: The backend is an existing .NET solution with many API projects. A project-based AppHost gives strong project references, solution integration, IDE support, and a conventional place to model service and container resources.

**Alternatives considered**:

| Alternative | Reason not selected |
| --- | --- |
| File-based AppHost | Lighter weight, but weaker fit for a multi-project .NET solution that benefits from solution references. |
| TypeScript AppHost | Better for Node-centered monorepos; HiveSpace backend orchestration is centered in the .NET solution. |

## Decision: Replace Docker Compose for backend local development

**Rationale**: The clarified requirement says the new local runtime fully replaces Docker Compose for backend local development. This reduces duplicated local workflows and makes one command/launch action the supported backend startup path.

**Alternatives considered**:

| Alternative | Reason not selected |
| --- | --- |
| Aspire preferred, Docker Compose fallback | Lower migration risk, but conflicts with the clarified replacement requirement. |
| Both workflows first-class | Keeps ambiguity and duplicate maintenance in local runtime docs. |

## Decision: Preserve existing service and infrastructure ports

**Rationale**: The spec requires existing local gateway and backend addresses to remain usable. Fixed ports avoid frontend configuration churn and preserve documented manual checks.

**Alternatives considered**:

| Alternative | Reason not selected |
| --- | --- |
| Fully dynamic Aspire ports | Better isolation, but would require broader gateway/frontend/config changes. |
| Dynamic infrastructure only | Possible later, but current services already use fixed local dependency settings and the feature prioritizes replacement with minimal behavior churn. |

## Decision: Do not migrate Docker Compose local data

**Rationale**: The clarification requires preserving data created by the new runtime across normal restarts while explicitly avoiding migration of existing Docker Compose local data.

**Alternatives considered**:

| Alternative | Reason not selected |
| --- | --- |
| Migrate existing Docker volumes | Adds fragile one-off migration work for disposable local data. |
| Treat all local data as disposable | Simpler, but conflicts with restart preservation requirements. |

## Decision: Use Aspire hosting integrations for local infrastructure

**Rationale**: SQL Server, RabbitMQ, Kafka, Redis, and Azure Storage/Azurite have Aspire hosting integration paths or container modeling support. Modeling them in AppHost is necessary for the replacement workflow and unified runtime view.

**Alternatives considered**:

| Alternative | Reason not selected |
| --- | --- |
| External existing dependencies through connection strings only | Would still require separate local infrastructure startup and would not replace Docker Compose. |
| Keep Docker Compose for infrastructure | Conflicts with the clarified replacement requirement. |

## Decision: Use connection strings for dependency endpoints and broker feature flags for runtime selection

**Rationale**: Databases and Redis already use `ConnectionStrings`, Azure Service Bus already has a connection-string shape, and Aspire resource references map naturally to connection strings. RabbitMQ and Kafka currently use nested endpoint configuration under `Messaging`, which makes Aspire injection and deployment secret handling less consistent. Keep `Messaging:EnableRabbitMq`, `Messaging:EnableKafka`, and `Messaging:EnableAzureServiceBus` as runtime feature flags, but move broker endpoints and secrets to `ConnectionStrings`.

**Alternatives considered**:

| Alternative | Reason not selected |
| --- | --- |
| Put all broker settings, including feature flags, into connection strings | Makes runtime broker selection opaque and harder to toggle per environment. |
| Keep the current nested provider endpoint objects | Works locally, but fits Aspire and deployment secret injection less cleanly. |
| Remove all provider tuning options | Simpler config, but loses existing RabbitMQ outbox/prefetch tuning and Kafka client behavior. |

## Decision: Add ServiceDefaults conservatively for OpenTelemetry monitoring

**Rationale**: ServiceDefaults provides the preferred local path for Aspire-backed OpenTelemetry traces, metrics, health, and dashboard behavior. Each service already has startup, logging, auth, health, EF, MassTransit, and YARP conventions, so the implementation should add OpenTelemetry monitoring defaults without replacing service-specific behavior. Logs remain available for troubleshooting, but logs alone do not satisfy the monitoring requirement.

**Alternatives considered**:

| Alternative | Reason not selected |
| --- | --- |
| Rewrite service startup around Aspire client integrations | Too broad for a local orchestration feature and risks business-service regressions. |
| AppHost only, no ServiceDefaults | Simpler, but loses the OpenTelemetry monitoring benefits expected from Aspire. |

## Decision: Standardize API startup file roles as part of Aspire adoption

**Rationale**: The backend already documents canonical startup roles in `docs/agent/startup-conventions.md`, but current service startup files are not fully consistent. Aspire ServiceDefaults, OpenTelemetry, health, and AppHost integration should be wired through a consistent startup surface instead of adding service-by-service inline setup. Strict standardization applies to all included API projects where technically possible, including ApiGateway, while preserving service-specific runtime behavior.

**Alternatives considered**:

| Alternative | Reason not selected |
| --- | --- |
| Only add Aspire hooks where needed | Leaves existing startup drift and makes future observability/runtime changes harder to apply consistently. |
| Standardize only full DDD services | Leaves ApiGateway and lite/legacy services as special cases even though they participate in the same local runtime. |
| Rewrite all startup behavior from scratch | Too risky; the goal is consistent file roles and wiring, not behavior replacement. |

## Decision: Keep MediaService Function out of v1 unless trivial

**Rationale**: The core spec names backend services and required infrastructure. The function project has a different runtime model. Including it should not delay or complicate the API/runtime replacement unless it can be launched without changing function behavior.

**Alternatives considered**:

| Alternative | Reason not selected |
| --- | --- |
| Always include the function app | Adds runtime complexity and may require separate tooling beyond the core API stack. |
| Permanently exclude it | Too strong; implementation can include it later if a safe launch path is found. |
