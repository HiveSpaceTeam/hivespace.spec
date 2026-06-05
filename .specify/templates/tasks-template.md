---

description: "Task index template for feature implementation"
---

# Tasks: [FEATURE NAME]

- **Input**: Design documents from `/specs/[NNNN-feature-name]/`
- **Prerequisites**: plan.md (required), spec.md (required for user-story traceability), research.md, data-model.md, contracts/, saga-design.md (conditional), linked ADRs (conditional)
- **Detailed tasks**: Implementation tasks live under `/specs/[NNNN-feature-name]/tasks/`
- **Organization**: `tasks.md` is the compatibility entrypoint and high-level tracker. Detailed task files are grouped by implementation ownership, then service/app/package/lib, then action.

## Detailed Task Files

Generate only the files needed by the feature, using these defaults:

| File | Purpose | Task ID prefix |
| --- | --- | --- |
| `tasks/backend.md` | Backend services, APIs, domain/application/infrastructure code, shared backend libs, migrations | `B###` |
| `tasks/frontend.md` | Frontend apps, shared package, services, stores, components, pages, routes, i18n | `F###` |
| `tasks/docs-catalog.md` | Service docs, API catalog, event catalog, ADRs, architecture docs | `D###` |
| `tasks/verification.md` | Builds, tests, lint/type-check, quickstart/manual validation, final searches | `V###` |

## Detailed Task Format

Every detailed task must be a markdown checkbox with a stable ID and enough implementation detail to execute without guessing.

```md
- [ ] B001 [US1] Create `ApplicationUser.cs`
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Identity/ApplicationUser.cs`
  - Include fields: `Status`, `CreatedAt`, `UpdatedAt`, `LastLoginAt`, `RoleName`, nullable `StoreId`
  - Inherit from: `IdentityUser`
  - Add/update methods:
    - `MarkLoginSuccess(DateTime utcNow)` updates `LastLoginAt` and `UpdatedAt`
    - `Suspend(DateTime utcNow)` sets status to suspended and updates `UpdatedAt`
    - `Activate(DateTime utcNow)` sets status to active and updates `UpdatedAt`
    - `AssignSellerRole(Guid storeId, DateTime utcNow)` sets `RoleName`, `StoreId`, and `UpdatedAt`
  - Do not include: profile fields, address fields, settings fields, store aggregate data, avatar fields
  - Acceptance: class compiles, EF can map custom fields, and identity-owned fields are no longer owned by UserService
```

Each task that touches code, config, or docs must include:

- target file path or explicit file set
- exact fields, types, methods, endpoints, routes, events, config keys, or docs sections to create/update/remove/move
- forbidden behavior or values that must not be introduced
- affected callers, dependencies, or ordering constraints when relevant
- acceptance check that can be verified locally

Do not generate `tasks/config.md`, `C###` task IDs, or feature tasks that edit `../hivespace.config`. Source-repo runtime settings, appsettings, gateway route config, and frontend environment typing belong in `backend.md` or `frontend.md` based on the owning source repo. The config repo may remain referenced only as local infrastructure context, such as starting Docker Compose.

## Grouping Rules

Detailed files must use this hierarchy:

1. Repository area file, such as `backend.md` or `frontend.md`
2. Service, app, package, library, or config/docs area
3. Action heading: `Create`, `Update`, `Remove`, `Move/Rename`, `Verify`
4. Detailed tasks with user-story labels for traceability

Keep user-story labels (`[US1]`, `[US2]`, etc.) on detailed tasks, but do not use user stories as the primary grouping.

## Documentation And Catalog Scope

Generated tasks must distinguish editable docs from verification-only context:

- Owning service docs: add update tasks when the shipped feature changes that service.
- Changed supporting service docs: add update tasks only when that service's API, event, validation, workflow, ownership, or behavior changed.
- Reused supporting service docs: add verification-only tasks; do not generate edit tasks.
- Shared API/event catalogs: add or update only new/changed contracts. Existing common rows reused unchanged are verification-only.

<!--
  ============================================================================
  IMPORTANT: The sections below are SAMPLE INDEX CONTENT for illustration only.

  The task generation command MUST replace them with actual content based on:
  - User stories from spec.md, used only for labels and traceability
  - Feature requirements and implementation structure from plan.md
  - Entities from data-model.md
  - Endpoints/events from contracts/
  - Saga design and linked ADRs when present

  DO NOT keep these sample tasks in generated tasks.md or tasks/*.md files.
  ============================================================================
-->

## Task Index

| Area | File | Status | Notes |
| --- | --- | --- | --- |
| Backend | `tasks/backend.md` | Not started | Services, shared libs, migrations |
| Frontend | `tasks/frontend.md` | Not started | Apps, shared package, routes, i18n |
| Docs/Catalog | `tasks/docs-catalog.md` | Not started | Service docs, catalogs, ADRs |
| Verification | `tasks/verification.md` | Not started | Builds, tests, quickstart |

## Dependency Order

1. Read target repo instructions and verify source layout.
2. Complete foundational contract and shared model/config tasks.
3. Complete backend service/lib tasks needed by the MVP story.
4. Complete frontend app/package tasks for the same story.
5. Complete docs/catalog updates for changed ownership, APIs, events, and ADR decisions.
6. Complete verification tasks.

## Story Traceability

| Story | Detailed task IDs | Independent acceptance |
| --- | --- | --- |
| US1 | B001-B00X, F001-F00X, V001 | [Describe independent validation] |
| US2 | B00X-B00Y, F00X-F00Y, V002 | [Describe independent validation] |

## Completion Checklist

- [ ] All detailed task files listed in Task Index exist.
- [ ] All detailed tasks have unique IDs across files.
- [ ] Every implementation task includes file path, exact change detail, forbidden behavior, dependencies/callers where relevant, and acceptance.
- [ ] `tasks.md` dependency order matches the detailed task dependencies.
- [ ] Verification tasks cover builds/tests/checks and quickstart/manual validation.
