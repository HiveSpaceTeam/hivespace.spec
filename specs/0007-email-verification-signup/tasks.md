# Tasks: Email Verification Before First Sign-In

- **Input**: Design documents from `/specs/0007-email-verification-signup/`
- **Prerequisites**: plan.md, spec.md, research.md, data-model.md, `contracts/account-activation-api.md`, quickstart.md, `architecture/decisions/ADR-0006-email-verification-activation-boundary.md`
- **Detailed tasks**: Implementation tasks live under `/specs/0007-email-verification-signup/tasks/`
- **Organization**: `tasks.md` is the compatibility entrypoint and high-level tracker. Detailed task files are grouped by implementation ownership, then service/app/package/lib, then action.

## Detailed Task Files

| File | Purpose | Task ID prefix |
| --- | --- | --- |
| `tasks/backend.md` | Backend services, APIs, domain/application/infrastructure code, shared backend libs, migrations | `B###` |
| `tasks/frontend.md` | Frontend apps, shared package, services, stores, pages, routes, i18n | `F###` |
| `tasks/docs-catalog.md` | Service docs, API catalog, event catalog | `D###` |
| `tasks/verification.md` | Repo instruction checks, builds, type-check, quickstart/manual validation, final searches | `V###` |

## Task Index

| Area | File | Status | Notes |
| --- | --- | --- | --- |
| Backend | `tasks/backend.md` | Not started | Shared messaging, IdentityService lifecycle/auth flows, UserService provisioning consumer |
| Frontend | `tasks/frontend.md` | Not started | Shared auth package plus buyer/seller sign-up, reusable seller email-action flow, and sign-in recovery flows |
| Docs/Catalog | `tasks/docs-catalog.md` | Not started | IdentityService/UserService service docs and shared API/event catalogs |
| Verification | `tasks/verification.md` | Not started | Repo instruction reads, builds, type-check, quickstart/manual validation, unchanged-supporting-service verification |

## Dependency Order

1. Complete `V001` to read target repo instructions and verify the implementation surface before editing code.
2. Complete shared backend contract work `B001` first so IdentityService and UserService can compile against the new readiness event.
3. Complete IdentityService lifecycle and API changes in order: `B002` -> `B003` -> `B004` -> `B005` -> `B006` -> `B007`.
4. Complete the downstream UserService consumer migration in `B008` after the new readiness event publisher exists.
5. Complete shared frontend auth contract work `F001` before app-specific sign-up and recovery flows.
6. Complete buyer frontend work `F002` -> `F003`, then seller frontend work `F004` -> `F005`, then shared pending-login recovery handling in `F006`.
7. Complete docs/catalog tasks `D001` -> `D002` after code paths and public/shared contract names are finalized.
8. Finish verification tasks `V002` -> `V005`.

## Story Traceability

| Story | Detailed task IDs | Independent acceptance |
| --- | --- | --- |
| US1 | B002, B003, F001, F002, F003, F004, D001 | A new buyer or seller registration returns a verification-sent success payload without `status` or `message`, issues no auth cookies, publishes the verification request, and lands on a verification-sent confirmation path with local UI copy instead of a signed-in session. |
| US2 | B001, B006, B007, B008, F003, F005, D002 | A valid verification link activates the pending account, returns `204 No Content`, publishes `IdentityUserReadyIntegrationEvent`, creates the UserService profile exactly once, and routes the user to sign-in with success copy. |
| US3 | B004, B005, F003, F005, F006, D001 | Pending-account sign-in is blocked with the standard exception envelope, resend is anonymous and returns `204 No Content` with generic success semantics, and buyer/seller apps route users into resend recovery without leaking account existence. |

## Implementation Groups

1. Shared backend contract and publisher foundation
2. IdentityService pending-registration, login-block, resend, and verify flows
3. UserService downstream provisioning migration
4. Shared frontend auth contract update
5. Buyer verification UX plus reusable seller email-action/verification UX
6. Service docs and shared catalog updates
7. Final verification and unchanged-supporting-service checks

## Suggested MVP Scope

Minimal scope for User Story 1:

- Backend: `B002`, `B003`
- Frontend: `F001`, `F002`, `F003`, `F004`
- Docs/Catalog: `D001`
- Verification: `V001`, `V002`, `V003`, `V004`

US2 and US3 can then layer on top with `B001`, `B004`-`B008`, `F005`, `F006`, `D002`, and `V005`.

## Completion Checklist

- [ ] All detailed task files listed in Task Index exist.
- [ ] All detailed tasks have unique IDs across files.
- [ ] Every implementation task includes file path, exact change detail, forbidden behavior, dependencies/callers where relevant, and acceptance.
- [ ] `tasks.md` dependency order matches the detailed task dependencies.
- [ ] Verification tasks cover repo instructions, builds/type-check, quickstart/manual validation, and unchanged-supporting-service checks.
