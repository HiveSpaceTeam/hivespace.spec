# Tasks: Cookie Auth Migration

- **Input**: Design documents from `specs/0004-cookie-auth-migration/`
- **Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/browser-auth-api.md](contracts/browser-auth-api.md), [quickstart.md](quickstart.md), [ADR-0003](../../architecture/decisions/ADR-0003-gateway-mediated-cookie-browser-sessions.md)
- **Detailed tasks**: Implementation tasks live under `specs/0004-cookie-auth-migration/tasks/`
- **Organization**: Detailed tasks are grouped by implementation ownership. User-story labels are traceability only.

## Task Index

| Area | File | Status | Notes |
| --- | --- | --- | --- |
| Backend | [tasks/backend.md](tasks/backend.md) | Not started | IdentityService REST auth/session endpoints, raw HttpOnly token cookies, Razor UI removal, ApiGateway cookie/CSRF mediation |
| Frontend | [tasks/frontend.md](tasks/frontend.md) | Not started | Shared cookie-session auth, shared AuthLayout from demo templates, app sign-in/sign-up forms, app images, routes, i18n |
| Config | [tasks/config.md](tasks/config.md) | Not started | Token cookie names/lifetimes, frontend gateway env, CORS/cookie settings |
| Docs/Catalog | [tasks/docs-catalog.md](tasks/docs-catalog.md) | Not started | IdentityService and ApiGateway docs, API catalog verification, event catalog no-change evidence |
| Verification | [tasks/verification.md](tasks/verification.md) | Not started | Builds/type-checks, browser/manual quickstart, security/storage searches |

## Dependency Order

1. Complete setup and impact-analysis tasks in backend/frontend detailed files before editing source symbols.
2. Complete config tasks for token cookie names/lifetimes, gateway origin, CORS, and frontend env defaults.
3. Complete IdentityService account session application services and endpoint contracts.
4. Complete ApiGateway cookie-to-bearer and CSRF mediation.
5. Complete shared frontend auth service/store and shared HTTP/realtime changes.
6. Complete shared `AuthLayout` from `packages/demo/src/Auth/Signin.vue` and `Signup.vue`, then app auth pages/routes that provide only their form slot plus page-specific center image/text.
7. Complete docs/catalog updates and verification-only checks for unchanged events/supporting services.
8. Run verification tasks in dependency order: backend builds, frontend checks, storage/security searches, and quickstart/manual flows.

## Story Traceability

| Story | Detailed task IDs | Independent acceptance |
| --- | --- | --- |
| US1 Sign In From Frontend-Owned UI | B001-B007, B011-B019, F001-F016, F018-F023, F025-F030, C001-C005, D001-D004, D006, V001-V011, V015 | Open admin, seller, and buyer while signed out; sign in with valid credentials; network shows `/api/v1/accounts/login` through ApiGateway; no IdentityService UI renders; app routes according to role rules. |
| US2 Register From Frontend-Owned UI | B001-B006, B008, B011-B019, F001-F010, F016-F030, C001-C005, D001-D006, V001-V005, V007-V010, V012, V015 | Register from buyer or seller frontend page; network shows `/api/v1/accounts/register`; one identity account is created; existing `IdentityUserCreatedIntegrationEvent` profile flow is preserved; duplicate email and validation errors are safe. |
| US3 Use Protected APIs Through Gateway Session | B001, B004-B006, B009, B011, B014-B019, F001-F009, F013-F014, F019-F021, F026-F027, F029, C001-C005, D003-D004, D006, V001-V003, V005-V009, V013-V014 | After sign-in, protected API calls through ApiGateway succeed with downstream bearer context, no access/refresh token appears in web storage, refresh after reload uses `/session/refresh`, and missing CSRF is rejected at gateway. |
| US4 Sign Out And Session Cleanup | B001, B004-B006, B010-B019, F001-F009, F013-F014, F019-F020, F026-F027, F029, C001-C005, D001-D004, D006, V001-V006, V009, V015 | Sign out from any app; session and CSRF cookies are cleared; reload requires sign-in; protected requests are unauthenticated after logout. |

## Suggested MVP Scope

MVP for US1 requires:

- Backend: IdentityService login/session issuance plus ApiGateway cookie-to-bearer/CSRF path needed for protected calls.
- Frontend: shared account session auth service/store, `ApiService` credentials/CSRF support, and sign-in pages/routes for admin, seller, and buyer.
- Config: token cookie names/lifetimes, CSRF header, frontend origins, and `VITE_GATEWAY_BASE_URL=http://localhost:5000`.
- Verification: backend builds, frontend type checks, sign-in happy/failure paths, and storage check.

Registration polish, cross-app/session edge cases, and old URL redirect verification can follow after the initial US1 sign-in path is smoke-tested, but old URL compatibility remains required before the feature is complete.

## Completion Checklist

- [ ] All detailed task files listed in Task Index exist.
- [ ] All detailed tasks have unique IDs across files.
- [ ] Every implementation task includes file path, exact change detail, forbidden behavior, dependencies/callers where relevant, and acceptance.
- [ ] `tasks.md` dependency order matches detailed task dependencies.
- [ ] Verification tasks cover backend builds, frontend checks, browser/manual flows, stale-token searches, CSRF rejection, and old URL redirects.
- [ ] Reused event contracts remain unchanged and are verified rather than edited.
