# Tasks: Local Backend Runtime Orchestration

- **Input**: Design documents from `/specs/0006-aspire-setup/`
- **Prerequisites**: `plan.md`, `spec.md`, `research.md`, `data-model.md`, `contracts/local-runtime.md`, `quickstart.md`, `architecture/decisions/ADR-0005-aspire-local-runtime.md`
- **Detailed tasks**: Implementation tasks live under `/specs/0006-aspire-setup/tasks/`
- **Organization**: `tasks.md` is the compatibility entrypoint and high-level tracker. Detailed task files are grouped by implementation ownership, then service/app/package/lib, then action.

## Detailed Task Files

| Area | File | Status | Notes |
| --- | --- | --- | --- |
| Backend | `tasks/backend.md` | Not started | AppHost, ServiceDefaults, startup standardization, messaging configuration |
| Docs/Catalog | `tasks/docs-catalog.md` | Not started | Backend run docs, service/runtime notes, catalog no-change verification |
| Verification | `tasks/verification.md` | Not started | Builds, static checks, AppHost smoke checks, telemetry checks |

No `frontend.md` is generated because frontend dev servers are out of scope and existing frontend gateway configuration must remain unchanged.

## Dependency Order

1. Complete backend source preparation and package/tooling tasks: `B001-B002`.
2. Create Aspire foundation projects and AppHost resources: `B003-B005`.
3. Standardize dependency configuration and messaging connection-string behavior: `B006-B008`, then `B020`.
4. Wire ServiceDefaults and normalize API startup file roles: `B009-B018`.
5. Add and verify MediaService Function AppHost orchestration: `B019`.
6. Update runtime documentation and confirm catalogs remain unchanged: `D001-D005`.
7. Execute verification in order: `V001-V012`.

## Story Traceability

| Story | Detailed task IDs | Independent acceptance |
| --- | --- | --- |
| US1 | `B001-B005`, `B009-B019`, `D001-D003`, `V001-V004`, `V007`, `V011` | A developer can start ApiGateway, all backend API services, MediaService Function, and required infrastructure from one documented AppHost command. |
| US2 | `B003`, `B005`, `B009-B019`, `D001-D003`, `V006`, `V008` | The Aspire dashboard shows service/function/dependency status plus OpenTelemetry traces and metrics for compatible backend services. |
| US3 | `B004-B005`, `B020`, `D001-D003`, `V003`, `V007`, `V009` | Existing local gateway/service ports and frontend gateway assumptions still work after AppHost startup. |
| US4 | `B001`, `B009-B018`, `D002-D003`, `V002`, `V005`, `V010` | Every included API project follows the standard startup file roles or documents a concrete technical exception. |
| Cross-cutting config | `B006-B008`, `B020`, `V004`, `V008` | Active broker endpoints/secrets are supplied through `ConnectionStrings`; broker enablement remains under `Messaging`; Kafka is declared in AppHost but disabled and unused by all v1 services/functions. |
| Cross-cutting catalog scope | `D004-D005`, `V012` | API and event catalogs remain unchanged because no public endpoint or integration contract changes are introduced. |

## Suggested MVP Scope

The MVP for User Story 1 is `B001-B005`, `B009-B020`, `D001-D003`, and `V001-V006`. This gives one AppHost startup path, all required backend services, MediaService Function, dependencies, preserved local ports, and enough verification to prove the backend stack starts.

## Completion Checklist

- [ ] All detailed task files listed in the Task Index exist.
- [ ] All detailed tasks have unique IDs across files.
- [ ] Every implementation task includes file path, exact change detail, forbidden behavior, dependencies/callers where relevant, and acceptance.
- [ ] `tasks.md` dependency order matches the detailed task dependencies.
- [ ] Verification tasks cover builds/checks and quickstart/manual validation.
- [ ] API and event catalogs are unchanged unless implementation discovers a real public contract change.
