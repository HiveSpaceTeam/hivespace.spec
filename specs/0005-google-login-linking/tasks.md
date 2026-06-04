# Tasks: Google Login and Account Linking

- **Input**: Design documents from `/specs/0005-google-login-linking/`
- **Prerequisites**: `plan.md`, `spec.md`, `research.md`, `data-model.md`, `contracts/browser-auth-api.md`, `quickstart.md`, `architecture/decisions/ADR-0004-google-account-linking.md`
- **Detailed tasks**: Implementation tasks live under `/specs/0005-google-login-linking/tasks/`
- **Organization**: `tasks.md` is the compatibility entrypoint and high-level tracker. Detailed task files are grouped by implementation ownership, then service/app/package/lib, then action.

## Detailed Task Files

| Area | File | Status | Task count | Notes |
| --- | --- | --- | ---: | --- |
| Backend | `tasks/backend.md` | Not started | 17 | IdentityService Google endpoint file, external login handlers, source/local IdentityService settings, page/static asset removal, gateway route verification |
| Frontend | `tasks/frontend.md` | Not started | 15 | Shared auth helpers, buyer/seller Google entry points, link-confirmation routes, frontend env verification, admin exclusion |
| Docs/Catalog | `tasks/docs-catalog.md` | Not started | 6 | IdentityService docs, API catalog validation, event catalog verification, ADR follow-up |
| Verification | `tasks/verification.md` | Not started | 10 | Builds, type-checks, manual quickstart scenarios, legacy URL checks |

## Documentation And Catalog Scope

- Editable service docs: `services/identity-service/README.md`, `services/identity-service/api.md`, `services/identity-service/data-and-events.md`.
- Editable catalogs: `shared/api-catalog.md` only if implementation changes endpoint path, method, auth, request/response, owner, or purpose from the current Google rows.
- Verification-only service docs: `services/user-service/*`, `services/api-gateway/*`, and NotificationService references, unless implementation changes their API, event, validation, workflow, ownership, or behavior.
- Verification-only event catalog: `shared/event-catalog.md`; no new event, command, saga message, failure event, timeout event, projection event, or consumer-set change is planned.

## Dependency Order

1. Backend source prep and impact analysis: `B001`, then inspect exact IdentityService symbols before modifying code.
2. Backend foundation: `B002-B004` before endpoint mapping or frontend integration.
3. Backend flow implementation: `B005-B012` in order, then legacy cleanup `B013-B015`.
4. Source/local runtime configuration checks: `B016-B017` and `F015`, coordinated with backend endpoint URLs and frontend origins.
5. Shared frontend auth foundation: `F001-F004` before app pages.
6. Buyer flow: `F005-F009`; seller flow: `F010-F013`; admin exclusion: `F014`.
7. Docs/catalog: `D001-D006` after implementation choices are final.
8. Verification: `V001-V010`, with manual quickstart checks after source config and builds pass.

## Story Traceability

| Story | Detailed task IDs | Independent acceptance |
| --- | --- | --- |
| US1 - Sign In With Google | B002, B003, B005, B006, B009-B012, B016, B017, F001-F004, F007-F009, F012, F013, F015, D001, D002, V001, V003-V005, V007 | From buyer or seller sign-in/sign-up, choose Google with a verified email that has no local password account, create/sign in a normal user, issue HttpOnly session cookies, and avoid legacy IdentityServer pages. |
| US2 - Link Existing Password Account | B003, B004, B006-B010, B012, B017, F001-F006, F009-F011, F013, D001, D002, V008 | Start Google with a verified email matching an existing password account, show link confirmation, require consent and correct password, block duplicate same-email Google-only account creation, link Google, mark email verified, and sign in. |
| US3 - Return With Already Linked Google Account | B006, B009-B012, F003, V007, V008 | After a successful link, sign out and choose Google again; the user signs in directly without the link prompt and account status rules still block disabled or locked accounts. |
| US4 - Remove Legacy Account Page Flow | B013-B015, B017, F014, D001, D002, V002, V006, V009, V010 | Known old `/Account/**` and page URLs no longer render legacy user-facing pages or static assets; buyer/seller use frontend-owned auth screens and admin has no Google option. |

## Completion Checklist

- [ ] All detailed task files listed in Task Index exist.
- [ ] All detailed tasks have unique IDs across files.
- [ ] Every implementation task includes file path, exact change detail, forbidden behavior, dependencies/callers where relevant, and acceptance.
- [ ] `tasks.md` dependency order matches the detailed task dependencies.
- [ ] Verification tasks cover builds/tests/checks and quickstart/manual validation.
- [ ] Optional hooks below have been considered before and after task generation.

## Extension Hooks

**Optional Pre-Hook**: git  
Command: `/speckit.git.commit`  
Description: Auto-commit before task generation

Prompt: Commit outstanding changes before task generation?  
To execute: `/speckit.git.commit`

**Optional Hook**: git  
Command: `/speckit.git.commit`  
Description: Auto-commit after task generation

Prompt: Commit task changes?  
To execute: `/speckit.git.commit`
