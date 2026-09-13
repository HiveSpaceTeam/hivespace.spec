# Verification Tasks: Tiki Catalog Crawl

## Python Crawler

### Verify

- [x] V001 [US1] Verify `crawler unit and contract tests`
  - File: `../hivespace.crawler/tests/`
  - Run the crawler repository unittest suite after B001-B008.
  - Confirm tests use fixtures and do not call live Tiki endpoints.
  - Acceptance: pytest exits 0 and schema/CLI/client/normalizer tests pass.

- [ ] V002 [US1] Verify `crawler lint/type checks`
  - File: `../hivespace.crawler/pyproject.toml`, `../hivespace.crawler/src/hivespace_crawler/`
  - Run the crawler repository lint and type-check commands defined in `pyproject.toml`.
  - Confirm no task-created temporary files remain under `../hivespace.crawler`.
  - Acceptance: lint/type checks exit 0 or any baseline issue is documented before implementation starts.

- [x] V003 [US1] Verify `sample crawl output contract`
  - File: `../hivespace.crawler/samples/tiki-category-1846.bundle.json`
  - Run `python -m hivespace_crawler tiki crawl --source-type category --source-value 1846 --limit 20 --out <temp-output>` against the approved operator test source when live-source access is permitted.
  - If live access is not permitted, run the same command with fixture-backed test mode if implemented and record that live source validation is deferred.
  - Acceptance: output bundle includes product, category, SKU, variant, price, stock, image, attribute, seller, summary, and validation hint data where available.

## Backend

### Verify

- [ ] V004 [US1] [US2] [US3] Verify `CatalogService tests and quality gate`
  - File: `../hivespace.microservice/tests/HiveSpace.CatalogService.Tests/`
  - Run CatalogService targeted tests and `.\quality-gate.ps1 -Scope backend:CatalogService` after B009-B026, B033-B039, and B043-B048.
  - Confirm long-running import endpoints return `202 Accepted` with job IDs and source file names where available, and that job history/status tests cover pagination, all operation types, bundle section endpoints, pending, running, completed, failed, progress counts, result summary, error summary, and idempotent retry behavior.
  - Confirm measured Domain/Application/API consumer coverage is at least 80%; add tests for missing measured behavior if below threshold.
  - Acceptance: targeted tests pass and CatalogService quality gate meets the coverage target.

- [ ] V005 [US2] Verify `IdentityService tests and quality gate`
  - File: `../hivespace.microservice/tests/HiveSpace.IdentityService.Tests/`
  - Run IdentityService targeted tests and `.\quality-gate.ps1 -Scope backend:IdentityService` after B027-B029.
  - Confirm the provisioning endpoint does not issue cookies, access tokens, refresh tokens, passwords, or browser session state.
  - Acceptance: targeted tests pass and IdentityService quality gate meets the coverage target.

- [ ] V006 [US2] Verify `UserService tests and quality gate`
  - File: `../hivespace.microservice/tests/HiveSpace.UserService.Tests/`
  - Run UserService targeted tests and `.\quality-gate.ps1 -Scope backend:UserService` after B030-B032.
  - Confirm store provisioning publishes existing store-created behavior and does not grant identity roles directly.
  - Acceptance: targeted tests pass and UserService quality gate meets the coverage target.

- [ ] V007 [US1] [US2] [US3] Verify `backend build and gateway route coverage`
  - File: `../hivespace.microservice/`
  - Run `dotnet build` from the backend repo root after all backend tasks.
  - Search gateway route config for `/api/v1/admins/catalog-imports`, `/api/v1/admins/imported-seller-accounts`, and `/api/v1/admins/imported-seller-stores`; confirm the catalog-import catchall covers seller ownership approval, job status, and retry endpoints.
  - Search backend source to confirm v1 import workers are hosted in CatalogService and no Azure Function, separate worker project, central executor, or MassTransit saga was introduced for catalog import.
  - Acceptance: backend builds with only documented baseline warnings and gateway routes cover every new public endpoint.

## Frontend

### Verify

- [ ] V008 [US1] [US2] [US3] Verify `admin tests, lint, type-check, and coverage`
  - File: `../hivespace.web/apps/admin/`
  - Run admin Jest tests for catalog imports, then `pnpm lint`, `pnpm type-check`, and `.\coverage.ps1 -Workspace admin` from the frontend repo root after F001-F017.
  - Treat known admin baseline lint/type-check failures as baseline only if they predate this feature; fix all introduced issues.
  - Confirm async operation UI treats `202 Accepted` as job submission, refreshes paginated bundle history and separate category upload history on the list page, shows selected-bundle related jobs in the side pane, polls job status on the detail page, refreshes linked bundle detail only after completion, keeps mixed-bundle import enabled when at least one row is eligible, and blocks explicit selected-product import when any selected row is no longer eligible at request acceptance time.
  - Acceptance: catalog import tests pass and admin policy-scoped line coverage is at least 80% or new tests are added until it is.

- [ ] V009 [US1] [US2] [US3] Verify `frontend route and i18n consistency searches`
  - File: `../hivespace.web/apps/admin/src/`
  - Search for stale route imports, stale `CatalogImportReviewPage` references, missing `CatalogImportListPage` or `CatalogImportJobDetailPage` references, hardcoded user-facing catalog import strings, and missing `catalogImports` imports in i18n setup.
  - Confirm shell/navigation labels use `common.*` and feature copy uses `catalogImports.*`.
  - Acceptance: searches find no stale filenames, no unresolved i18n roots, and no hardcoded catalog import UI copy.

## Docs And Catalogs

### Verify

- [ ] V010 [US1] [US2] [US3] Verify `planning docs and catalogs are synchronized`
  - File: `services/catalog-service/`, `services/identity-service/`, `services/user-service/`, `shared/api-catalog.md`, `shared/event-catalog.md`
  - Confirm service docs, API catalog, and event catalog agree on endpoint ownership, auth policies, async job response semantics, paginated job history, bundle section pagination endpoints, job status/retry routes, background job lifecycle events, reused domain events, and no saga requirement.
  - Confirm no docs claim MediaService or the Python crawler owns product/store/account business data.
  - Confirm no docs claim a central monitor executes CatalogService import jobs.
  - Acceptance: docs/catalogs are internally consistent and all changed markdown files are non-empty.

## User-Owned End-to-End

### Verify

- [ ] V011 [US1] [US2] [US3] Verify `full operator catalog import journey`
  - File: `specs/0012-tiki-catalog-crawl/quickstart.md`
  - User-owned E2E: Start backend AppHost, start the admin frontend, crawl Tiki categories and category attributes from `../hivespace.crawler`, submit/provision categories and category attributes from the Catalog Import list/upload page, confirm job IDs and source file names are returned, confirm the submissions appear in the separate preparation upload histories, open the job detail page and monitor provisioning to completion, then crawl products, submit a product bundle from the list/upload page, confirm the bundle appears in the primary bundle table, open the selected bundle's side pane, review the related jobs, open the detail page from a related job row, validate, provision sellers, approve any intended existing-store seller ownership link, revalidate, and import all eligible ready products with `Draft`, `Unpublish`, or `Available` publication state from the detail page while monitoring each async job to completion.
  - Confirm invalid VND prices, missing category provisioning, unresolved seller conflicts, ineligible ownership approvals, invalid stock, required field gaps, unusable media references, and duplicate products remain blocked or warned as appropriate while unrelated eligible rows in the same bundle can still be imported.
  - Confirm newly approved seller-conflict rows do not become importable until validation is rerun and the refreshed readiness state marks them eligible.
  - Confirm imported products remain hidden from buyer storefront results until existing publication rules make them visible.
  - Acceptance: user runs and confirms the browser/API journey; agents must leave this task pending until user confirmation.

