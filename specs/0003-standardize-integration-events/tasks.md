# Tasks: Standardize Integration Event Contracts

- **Input**: Design documents from `/specs/0003-standardize-integration-events/`
- **Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/standardized-message-contracts.md, quickstart.md, ADR-0002
- **Detailed tasks**: Implementation tasks live under `/specs/0003-standardize-integration-events/tasks/`
- **Organization**: `tasks.md` is the compatibility entrypoint and high-level tracker. Detailed task files are grouped by implementation ownership, then service/library/docs area, then action.

## Summary

This feature standardizes backend integration event contracts and durable documentation. It has no frontend, public API, database schema, or config-file work. The implementation touches the backend shared messaging contracts, affected backend publishers/consumers/sagas, service docs, event catalog, and coding conventions.

## Task Index

| Area | File | Status | Notes |
| --- | --- | --- | --- |
| Backend | `tasks/backend.md` | Not started | Shared message contracts, event base inheritance, workflow folder split, publisher abstractions, service consumers/sagas |
| Docs/Catalog | `tasks/docs-catalog.md` | Not started | Event catalog, affected service docs, coding conventions, ADR status |
| Verification | `tasks/verification.md` | Not started | Build, contract inventory searches, publisher-policy checks, docs searches, rollout gate evidence |

## Dependency Order

1. Complete backend source orientation and GitNexus impact checks: `B001`.
2. Standardize shared message contracts before updating service references: `B002-B007`.
3. Align service-owned publisher abstractions and application/background publishing paths: `B008-B013`.
4. Update consuming services, checkout saga, and fulfillment saga references after target contract names exist: `B014-B021`.
5. Update durable docs and catalogs after source names are known: `D001-D010`.
6. Run verification and deployment-readiness checks: `V001-V008`.

## Story Traceability

| Story | Detailed task IDs | Independent acceptance |
| --- | --- | --- |
| US1 | B001-B007, B014, B021, D001, D010, V001-V004, V007 | Shared contract inventory shows every event is named `*IntegrationEvent`, derives from `IntegrationEvent`, and commands/DTOs remain non-events. |
| US2 | B003-B006, B015-B021, D001, D005, D008, D010, V002, V004, V007 | Checkout, fulfillment, and shared handoff contracts are grouped by responsibility without changing saga business outcomes. |
| US3 | B008-B013, D002-D004, D006-D010, V003, V006, V007 | Application/background publishes use service-owned publisher abstractions while MassTransit saga orchestration remains context-driven. |
| US4 | D001-D010, V004-V008 | Current event catalog, affected service docs, coding conventions, and rollout evidence match standardized names and policies. |

## Documentation And Catalog Scope

Editable service docs:

- `services/identity-service/README.md`
- `services/identity-service/data-and-events.md`
- `services/user-service/README.md`
- `services/user-service/data-and-events.md`
- `services/catalog-service/README.md`
- `services/catalog-service/data-and-events.md`
- `services/media-service/README.md`
- `services/media-service/data-and-events.md`
- `services/order-service/README.md`
- `services/order-service/data-and-events.md`
- `services/order-service/workflows.md`
- `services/payment-service/README.md`
- `services/payment-service/data-and-events.md`
- `services/payment-service/workflows.md`
- `services/notification-service/README.md`
- `services/notification-service/data-and-events.md`
- `services/notification-service/workflows.md`

Editable shared docs:

- `shared/event-catalog.md`
- `shared/coding-conventions.md`
- `architecture/decisions/ADR-0002-standardized-integration-event-contracts.md`

Verification-only context:

- `shared/api-catalog.md` because no public endpoint changes are planned.
- `services/api-gateway/` because ApiGateway is reused unchanged.
- Historical completed feature specs may keep old event names as historical context.

## Suggested MVP Scope

MVP for US1 is `B001-B007`, `D001`, `D010`, and `V001-V004`. This proves the core contract standard before service publisher policy and durable service-documentation cleanup are completed.

## Independent Test Criteria

- **US1**: Run shared-contract searches and confirm all event records end with `IntegrationEvent`, derive from `IntegrationEvent`, and non-event commands/DTOs were not renamed.
- **US2**: Review the shared messaging folder tree and event catalog to confirm checkout, fulfillment, and handoff grouping is discoverable in under 2 minutes.
- **US3**: Search application/background publishing paths and confirm only service-owned publishers publish integration events, while saga request/response/timeout/continuation paths remain MassTransit context-driven.
- **US4**: Search durable docs and catalogs for suffixless current saga event names and confirm only historical feature specs may retain them.

## Completion Checklist

- [ ] All detailed task files listed in Task Index exist.
- [ ] All detailed tasks have unique IDs across files.
- [ ] Every implementation task includes file path, exact change detail, forbidden behavior, dependencies/callers where relevant, and acceptance.
- [ ] `tasks.md` dependency order matches the detailed task dependencies.
- [ ] Verification tasks cover backend build, source searches, docs/catalog searches, and rollout gate evidence.
