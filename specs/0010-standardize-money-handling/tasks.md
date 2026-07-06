# Tasks: Standardize System Money Handling

- **Input**: Design documents from `/specs/0010-standardize-money-handling/`
- **Prerequisites**: `plan.md` (required), `spec.md` (required for story traceability), `research.md`, `data-model.md`, `contracts/`, linked ADRs
- **Detailed tasks**: Implementation tasks live under `/specs/0010-standardize-money-handling/tasks/`
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
| Backend | `tasks/backend.md` | Not started | Shared money contract updates, UserService policy ownership, Catalog/Order/Payment validation and read-model normalization |
| Frontend | `tasks/frontend.md` | Not started | `@hivespace/shared` formatter/input utilities, admin policy UI, seller and buyer money-surface standardization |
| Docs/Catalog | `tasks/docs-catalog.md` | Not started | Service docs, API catalog, event catalog, ADR follow-through |
| Verification | `tasks/verification.md` | Not started | Backend quality gates, frontend coverage/type-checks, hard-coded currency search, user-owned E2E |

## Dependency Order

1. Complete shared and owning-service backend test-code tasks first: `B001`, `B002`, `B007`, `B010`, `B013`.
2. Complete shared backend contract/factory groundwork before service-specific behavior: `B003`, `B004`.
3. Complete UserService configuration-foundation persistence, endpoints, and publishing for US1: `B005`, `B006`.
4. Complete CatalogService policy projection and currency-aware product normalization: `B008`, `B009`.
5. Complete OrderService coupon canonical currency work before cart/checkout guards and query normalization: `B011`, `B012`.
6. Complete PaymentService validation/read-model updates after the configuration event contract and projection pattern are in place: `B014`, `B015`.
7. Complete shared frontend formatter/input/types before app-local UI work: `F001`, `F002`.
8. Complete admin configuration tests and currency-config UI: `F003`, `F004`.
9. Complete admin money-surface standardization outside the configuration workspace: `F012`, `F013`.
10. Complete seller contract/store updates before seller page conversions: `F005`, `F006`, `F007`, `F008`.
11. Complete buyer contract/store updates before buyer page conversions: `F009`, `F010`, `F011`.
12. Complete docs and catalog alignment after the implementation shape is stable: `D001` to `D005`.
13. Finish with verification and user-owned E2E validation: `V001` to `V005`.

## Story Traceability

| Story | Detailed task IDs | Independent acceptance |
| --- | --- | --- |
| US1 | `B001`, `B002`, `B003`, `B005`, `B006`, `B009`, `B012`, `B013`, `B014`, `B015`, `F003`, `F004`, `D001`, `D002`, `D003`, `D004`, `D005`, `V001`, `V002`, `V003`, `V004`, `V005` | An admin can save enabled/default currencies in the configuration workspace, disabling the current default is blocked until a replacement default is chosen, and downstream services reject new writes in disabled currencies while keeping historical records readable. |
| US2 | `B003`, `B004`, `B007`, `B008`, `B009`, `B010`, `B011`, `B012`, `F001`, `F002`, `F005`, `F006`, `F007`, `D002`, `D003`, `D004`, `V001`, `V002`, `V003`, `V005` | A seller can create and edit coupons and product prices in enabled currencies, coupon money fields keep one canonical currency, and mixed or disabled currency write attempts are rejected before activation. |
| US3 | `B003`, `B004`, `B007`, `B008`, `B010`, `B011`, `B012`, `B013`, `B014`, `B015`, `F001`, `F002`, `F005`, `F006`, `F007`, `F008`, `F009`, `F010`, `F011`, `F012`, `F013`, `D002`, `D003`, `V001`, `V002`, `V003`, `V005` | Admin, seller, and buyer surfaces render `VND`, `USD`, and `EUR` consistently, `USD`/`EUR` default to major-unit display, invalid-money placeholders appear instead of guessed currencies, and shared money inputs submit correct smallest-unit payloads. |

## Suggested MVP Scope

The minimal shippable slice for User Story 1 is:

1. `tasks/backend.md`: `B001`, `B002`, `B003`, `B005`, `B006`, `B009`, `B012`, `B013`, `B014`, `B015`
2. `tasks/frontend.md`: `F001`, `F002`, `F003`, `F004`
3. `tasks/docs-catalog.md`: `D001`, `D003`, `D004`, `D005`
4. `tasks/verification.md`: `V001`, `V002`, `V003`, `V004`

## Format Validation

- All detailed task files listed in the Task Index exist.
- All detailed tasks use the required checkbox format with stable IDs.
- All code tasks include file paths or explicit file sets, exact change details, constraints, and acceptance checks.
- Test-code tasks precede their paired implementation tasks in the relevant detailed file.
- Verification tasks cover backend, frontend, hard-coded-currency regression checks, and user-owned E2E validation.

## Completion Checklist

- [ ] All detailed task files listed in the Task Index exist.
- [ ] All detailed tasks have unique IDs across files.
- [ ] Every implementation task includes file path, exact change detail, forbidden behavior, dependencies/callers where relevant, and acceptance.
- [ ] `tasks.md` dependency order matches the detailed task dependencies.
- [ ] Verification tasks cover builds/tests/checks and manual validation.
