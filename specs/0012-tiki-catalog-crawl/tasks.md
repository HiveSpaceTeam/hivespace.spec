# Tasks: Tiki Catalog Crawl

- **Input**: Design documents from `specs/0012-tiki-catalog-crawl/`
- **Prerequisites**: `spec.md`, `research.md`, `data-model.md`, `contracts/`, `quickstart.md`; `plan.md` exists but still contains the unfilled template, so this task set is generated from the completed design artifacts.
- **Detailed tasks**: Implementation tasks live under `specs/0012-tiki-catalog-crawl/tasks/`
- **Organization**: `tasks.md` is the compatibility entrypoint and high-level tracker. Detailed task files are grouped by implementation ownership, then service/app/package/lib, then action.

## Task Index

| Area | File | Status | Notes |
| --- | --- | --- | --- |
| Backend/source work | `tasks/backend.md` | Not started | Python crawler repo, CatalogService async import job APIs/workers, seller ownership approval, IdentityService/UserService seller provisioning APIs, gateway routes, migrations |
| Frontend | `tasks/frontend.md` | Not started | Admin catalog import types, services, paginated bundle history/store state, selected-bundle side pane, category upload history, job detail page, seller ownership approval UI, routes, i18n |
| Docs/Catalog | `tasks/docs-catalog.md` | Not started | Catalog/Identity/User service docs, API catalog updates, and background job lifecycle event catalog updates |
| Verification | `tasks/verification.md` | Not started | Backend/frontend/crawler checks plus user-owned E2E |

## Documentation And Catalog Scope

| Service or artifact | Classification | Editable? | Reason |
| --- | --- | --- | --- |
| CatalogService | Owning service | Yes | Owns category provisioning, category-attribute provisioning, import bundle persistence, catalog validation, duplicate grouping, seller provisioning orchestration, operator-approved seller ownership links, and publication-state product import |
| IdentityService | Changed supporting service | Yes | Adds idempotent imported seller account provisioning API |
| UserService | Changed supporting service | Yes | Adds idempotent imported seller store provisioning API |
| ApiGateway | Changed supporting service | Yes, route config only | New admin route prefixes must be browser-facing through `/api/v1` |
| MediaService | Reused supporting service | No | Existing media upload/processing ownership is reused unchanged for later image copy/association |
| Event catalog | Changed shared monitoring contracts | Yes | Reuses domain events unchanged and adds generic `BackgroundJob*IntegrationEvent` contracts for observer-only monitoring of service-owned async import jobs |
| `../hivespace.config` | Infrastructure context only | No | Feature tasks must not edit the config repo |

## Dependency Order

1. Crawler contract foundation: B001-B008.
2. CatalogService domain and validation foundation: B009-B016.
3. CatalogService async job application commands/queries, background consumers, seller ownership approval, and ports: B017-B026, B043-B044, B045-B048.
4. IdentityService and UserService provisioning endpoints: B027-B034.
5. CatalogService infrastructure, API endpoints, gateway routes, and migrations: B033-B039.
6. Admin frontend contracts, store tests, services, store, seller ownership approval UI, pages, route, i18n: F001-F015.
7. Documentation/catalog updates: D001-D007.
8. Verification and user-owned E2E: V001-V011.

Test-code tasks precede their paired implementation tasks within each implementation group.

## Story Traceability

| Story | Detailed task IDs | Independent acceptance |
| --- | --- | --- |
| US1 | B001-B008, B017-B020, B035-B038, B045-B048, F001-F007, F012-F013, F016-F017, D001, D004-D006, V001-V004, V007-V011 | Crawl categories first, submit/provision categories through an async job, then submit a product bundle and confirm bundle history, selected-bundle related jobs, category upload history, product, SKU, seller, price, stock, image, attribute, optional-gap, job status, and summary data are reviewable |
| US2 | B021-B022, B027-B037, B043-B044, F008-F009, F014-F015, D002-D004, V004-V006, V009-V011 | Submit products from multiple Tiki sellers, provision sellers, resolve similar-name store conflicts through explicit operator approval, and confirm each imported product has matched or created account/store ownership before readiness/import |
| US3 | B009-B016, B023-B026, B033-B035, B038-B039, B045-B048, F010-F011, F016-F017, D001, D004-D007, V004, V007-V011 | Validate bundles with category provisioning, price, stock, duplicate, seller, media, required-field issues, and async job status; confirm blocked/warning/ready/duplicate reports are actionable |

## Suggested MVP Scope

Complete US1 first with B001-B008, B017-B020, B035-B038, B045-B048, F001-F007, F012-F013, F016-F017, D001, D004-D006, and V001-V004/V007-V011. That creates category-first crawler output, async category provisioning, bundle submission, paginated bundle history, selected-bundle side-pane jobs, list/upload page, job detail page, job status polling, and operator review surface before seller provisioning or final import is attempted.

## Completion Checklist

- [ ] All detailed task files listed in Task Index exist.
- [ ] All detailed tasks have unique IDs across files.
- [ ] Every implementation task includes file path, exact change detail, forbidden behavior, dependencies/callers where relevant, and acceptance.
- [ ] Required backend/frontend test-code tasks precede paired implementation tasks.
- [ ] Verification tasks cover builds/tests/checks, coverage gates, quickstart validation, and user-owned E2E.
- [ ] No task edits `../hivespace.config`.
