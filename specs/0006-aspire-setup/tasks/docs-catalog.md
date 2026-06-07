# Docs/Catalog Tasks: Local Backend Runtime Orchestration

## Backend Runtime Documentation

### Update

- [ ] D001 [US1] [US2] [US3] Update `backend agent instructions`
  - File: `../hivespace.microservice/AGENTS.md`, `../hivespace.microservice/CLAUDE.md`
  - Update both files together so Aspire AppHost is the preferred backend local startup command: `dotnet run --project .\HiveSpace.AppHost\HiveSpace.AppHost.csproj`.
  - Mark Docker Compose as replaced for backend local development while preserving any remaining config/infrastructure maintenance notes.
  - Document Azure Functions Core Tools as required for the full AppHost runtime because MediaService Function is included.
  - Document Kafka as declared local infrastructure only; no v1 backend service or function uses it.
  - Document that frontend dev servers remain outside the AppHost v1 scope and continue using the existing gateway URL.
  - Do not let `AGENTS.md` and `CLAUDE.md` drift semantically.
  - Acceptance: both instruction files contain equivalent backend local runtime guidance.

- [ ] D002 [US1] [US2] [US4] Update `backend runtime and startup docs`
  - File: `../hivespace.microservice/README.md`, `../hivespace.microservice/docs/agent/startup-conventions.md`, `../hivespace.microservice/docs/**`
  - Document prerequisites, including Azure Functions Core Tools for MediaService Function, AppHost command, dashboard/runtime view, fixed ports, stop/restart expectations, no Docker Compose data migration, Kafka declared-only status, and startup convention expectations.
  - Include the connection-string-first dependency contract and broker feature flag rules.
  - Include any concrete startup exceptions discovered during implementation, especially for IdentityService, UserService, NotificationService, ApiGateway, MediaService API, or MediaService Function.
  - Do not document public endpoint or integration event changes unless implementation actually introduced them.
  - Acceptance: a developer can follow backend docs to start, observe, stop, and restart the local backend runtime.

## Spec Repo Runtime Documentation

### Update

- [ ] D003 [US1] [US2] [US3] [US4] Update `spec repo runtime references`
  - File: `architecture/overview.md`, `services/api-gateway/README.md`, `services/identity-service/README.md`, `services/user-service/README.md`, `services/catalog-service/README.md`, `services/media-service/README.md`, `services/order-service/README.md`, `services/payment-service/README.md`, `services/notification-service/README.md`
  - Update only runtime/local-development notes needed after implementation, such as Aspire AppHost startup, fixed ports, ServiceDefaults/OpenTelemetry, and connection-string dependency shape.
  - Keep service ownership, API ownership, event ownership, domain behavior, and business workflow docs unchanged unless implementation discovers a real service behavior change.
  - Do not rewrite common service descriptions just because a service participates in Aspire startup.
  - Acceptance: spec repo docs mention the new backend runtime where useful without implying public contract or service boundary changes.

## Catalog Scope

### Verify

- [ ] D004 Verify `API catalog remains unchanged`
  - File: `shared/api-catalog.md`
  - Confirm the implementation introduced no new public endpoint path, method, auth policy, request/response shape, route ownership, or SignalR contract.
  - If a real public contract changed unexpectedly, update the catalog row in the same change and explain why it was necessary.
  - Do not add AppHost, health-dashboard, or local-only developer workflow entries as public business API rows.
  - Acceptance: catalog remains unchanged or contains only justified real public API changes.

- [ ] D005 Verify `event catalog remains unchanged`
  - File: `shared/event-catalog.md`
  - Confirm the implementation introduced no new cross-service event, command, saga message, failure event, timeout event, projection event, or consumer set change.
  - If a real integration contract changed unexpectedly, update the catalog row in the same change and explain why it was necessary.
  - Do not add Aspire resources, OpenTelemetry signals, or local dependency status as integration events.
  - Acceptance: catalog remains unchanged or contains only justified real integration contract changes.
