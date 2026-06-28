# Tasks: Email OTP Sign-In

- **Input**: Design documents from `/specs/0009-signin-otp-email/`
- **Prerequisites**: `plan.md` (required), `spec.md` (required for story traceability), `research.md`, `data-model.md`, `contracts/`, linked ADRs
- **Detailed tasks**: Implementation tasks live under `/specs/0009-signin-otp-email/tasks/`
- **Organization**: `tasks.md` is the compatibility entrypoint and high-level tracker. Detailed task files are grouped by implementation ownership, then service/app/package/lib, then action.

## Detailed Task Files

| File | Purpose | Task ID prefix |
| --- | --- | --- |
| `tasks/backend.md` | Backend services, APIs, domain/application/infrastructure code, shared backend libs, migrations | `B###` |
| `tasks/frontend.md` | Frontend apps, shared package, services, stores, components, pages, routes, i18n | `F###` |
| `tasks/docs-catalog.md` | Service docs, API catalog, event catalog, ADRs | `D###` |
| `tasks/verification.md` | Builds, tests, lint/type-check, manual validation, final searches | `V###` |

## Task Index

| Area | File | Status | Notes |
| --- | --- | --- | --- |
| Backend | `tasks/backend.md` | Not started | IdentityService, NotificationService, shared messaging contract, persistence, endpoints |
| Frontend | `tasks/frontend.md` | Not started | `@hivespace/shared`, buyer app, seller app OTP sign-in flow |
| Docs/Catalog | `tasks/docs-catalog.md` | Not started | IdentityService docs, NotificationService docs, API catalog, event catalog, ADR follow-through |
| Verification | `tasks/verification.md` | Not started | Backend build/tests, frontend lint/type-check/tests, manual OTP flow validation |

## Dependency Order

1. Complete backend and frontend test-code tasks first: `B001`, `B002`, `B012`, `F004`, `F007`, `F011`, `F014`.
2. Complete shared backend contract/model work: `B003`, `B006`, `B007`.
3. Complete IdentityService request flow for US1 and US3: `B004`, `B008`, `B010`, `B011`, `B015`, `B017`.
4. Complete IdentityService verify flow for US2: `B005`, `B009`, `B016`.
5. Complete NotificationService delivery work after the shared event contract exists: `B013`, `B014`.
6. Complete shared frontend support before app-local UI: `F002`, `F003`.
7. Complete buyer OTP flow: `F005`, `F006`, `F007`, `F008`, `F009`, `F010`.
8. Complete seller OTP flow: `F012`, `F013`, `F014`, `F015`, `F016`.
9. Complete docs and catalog alignment after implementation shape is stable: `D001` to `D005`.
10. Run verification tasks and finish with cross-artifact consistency checks: `V001` to `V005`.

## Story Traceability

| Story | Detailed task IDs | Independent acceptance |
| --- | --- | --- |
| US1 | `B001`, `B003`, `B004`, `B006`, `B007`, `B008`, `B010`, `B012`, `B013`, `B014`, `F002`, `F004`, `F005`, `F007`, `F008`, `F010`, `F011`, `F012`, `F014`, `F016`, `D001`, `D002`, `D003`, `D004`, `D005`, `V001`, `V002`, `V003`, `V004`, `V005` | Buyer and seller sign-in pages expose OTP sign-in, OTP request returns the generic success shape, and no account existence information is exposed. |
| US2 | `B002`, `B005`, `B008`, `B009`, `B016`, `F003`, `F006`, `F009`, `F013`, `F015`, `V001`, `V003`, `V004`, `V005` | User enters a valid challenge token and code, receives the same browser session as password sign-in, and lands on the validated redirect target. |
| US3 | `B001`, `B011`, `B015`, `F003`, `F004`, `F009`, `F011`, `F015`, `V001`, `V003`, `V004`, `V005` | Resend unlocks only after cooldown, old codes stop working after resend, and countdown state stays accurate in the UI. |

## Suggested MVP Scope

The minimal shippable slice for User Story 1 is:

1. `tasks/backend.md`: `B001`, `B003`, `B004`, `B006`, `B007`, `B008`, `B010`, `B012`, `B013`, `B014`
2. `tasks/frontend.md`: `F002`, `F004`, `F005`, `F007`, `F008`, `F010`, `F011`, `F012`, `F014`, `F016`
3. `tasks/docs-catalog.md`: `D001`, `D002`, `D003`, `D004`, `D005`
4. `tasks/verification.md`: `V001`, `V002`, `V003`, `V004`, `V005`

## Format Validation

- All detailed task files listed in the Task Index exist.
- All detailed tasks use the required checkbox format with stable IDs.
- All code tasks include file paths or explicit file sets, exact change details, constraints, and acceptance checks.
- Test-code tasks precede their paired implementation tasks in the relevant detailed file.
- Verification tasks cover backend, frontend, manual OTP flow validation, and final documentation/catalog consistency checks.

## Completion Checklist

- [ ] All detailed task files listed in the Task Index exist.
- [ ] All detailed tasks have unique IDs across files.
- [ ] Every implementation task includes file path, exact change detail, forbidden behavior, dependencies/callers where relevant, and acceptance.
- [ ] `tasks.md` dependency order matches the detailed task dependencies.
- [ ] Verification tasks cover builds/tests/checks and manual validation.
