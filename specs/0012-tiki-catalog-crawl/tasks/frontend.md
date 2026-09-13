# Frontend Tasks: Tiki Catalog Crawl

## Admin App: Catalog Imports

### Create

- [ ] F001 [US1] Create `admin catalog import types`
  - File: `../hivespace.web/apps/admin/src/types/catalog-import.types.ts`, `../hivespace.web/apps/admin/src/types/index.ts`
  - Define source, crawl, seller, category provisioning, category link, product, SKU, image, attribute, validation issue, duplicate group, summary, seller ownership approval, provisioning result, import result, job submission, job history row, job status/detail, job progress, paginated request/response, upload request, and response types matching `contracts/category-provisioning-schema.md`, `contracts/import-bundle-schema.md`, and `contracts/catalog-import-api.md`.
  - Include `sourceFileName` as optional display metadata on upload submissions, bundles, job history rows, and job detail responses.
  - Use canonical string `currencyCode` and do not add numeric currency enums or local VND allowlists.
  - Acceptance: admin `pnpm type-check` sees exported types without cross-app imports.

- [ ] F002 [US1] Create `admin catalog import service`
  - File: `../hivespace.web/apps/admin/src/services/catalog-import.service.ts`
  - Use the app singleton API service/build URL convention to call `/api/v1/admins/catalog-imports/categories/provisioning`, `/api/v1/admins/catalog-imports/categories/attributes/provisioning`, paginated `/api/v1/admins/catalog-imports/bundles`, paginated `/api/v1/admins/catalog-imports/jobs`, `/api/v1/admins/catalog-imports/jobs/{jobId}`, `/api/v1/admins/catalog-imports/bundles/{bundleId}`, paginated bundle section endpoints for category links, sellers, products, duplicate groups, and validation issues, validation, seller provisioning, seller ownership approval, import endpoints, and optional retry.
  - Send original upload file names through the `X-Source-File-Name` header for category and product bundle submissions when available.
  - Treat long-running operation responses as job submissions instead of completed result payloads.
  - Pass pagination/filter parameters for job history and all bundle-detail collection tables.
  - Keep this file as a thin transport wrapper with no branching beyond request/response mapping.
  - Do not construct `ApiService` directly and do not call IdentityService/UserService provisioning endpoints from the frontend.
  - Acceptance: admin type-check passes and endpoint paths match `contracts/catalog-import-api.md`.

- [ ] F003 [US1] Create `catalog import store tests - bundle history, side pane jobs, and detail`
  - File: `../hivespace.web/apps/admin/src/stores/catalog-import.store.test.ts`
  - Test: `should load paginated catalog import bundle history from the API`
  - Test: `should load selected bundle summary and related jobs for the side pane`
  - Test: `should load job detail and linked bundle detail with paginated sellers products issues and duplicate groups`
  - Test: `should preserve source file name on upload-created job submissions`
  - Mock `catalogImportService`; assert loading/error state, bundle/category pagination state, selected bundle side-pane rows, selected job detail, and linked bundle sections.
  - Acceptance: tests compile and fail before F004 implements the store.

- [ ] F004 [US1] Create `catalog import store`
  - File: `../hivespace.web/apps/admin/src/stores/catalog-import.store.ts`, `../hivespace.web/apps/admin/src/stores/index.ts`
  - Implement Pinia setup store state for category provisioning upload, product bundle upload, paginated bundle history, selected bundle side-pane summary and related jobs, category provisioning upload history, selected job detail, linked bundle detail, per-table pagination, loading, error, filters, active jobs, job progress, provisioning summary, validation summary, and import summary.
  - Actions: `provisionCategories`, `submitBundle`, `fetchBundleHistory`, `fetchSelectedBundleJobs`, `fetchCategoryProvisioningHistory`, `fetchJobHistory`, `fetchJobDetail`, `fetchBundleDetail`, `validateBundle`, `provisionSellers`, `approveSellerOwnership`, `importReadyProducts`, `fetchJobStatus`, and optional `retryJob`.
  - Use `useAppStore().setLoading(true/false)` in `try/finally` and app notifications for success/error.
  - Do not call service methods from pages/components directly.
  - Acceptance: F003 tests pass.

- [ ] F005 [US1] Create `catalog import list/upload and detail page tests`
  - File: `../hivespace.web/apps/admin/src/pages/catalog-imports/CatalogImportListPage.test.ts`, `../hivespace.web/apps/admin/src/pages/catalog-imports/CatalogImportJobDetailPage.test.ts`
  - Test: `should render category and product upload controls on the list page`
  - Test: `should render paginated bundle history on the list page`
  - Test: `should open the selected bundle side pane without collapsing the main bundle table`
  - Test: `should navigate to job detail when a related job row is clicked from the side pane`
  - Test: `should render separate category provisioning upload history on the list page`
  - Test: `should render formatted local datetime plus relative hint on list-page tables`
  - Test: `should render job detail bundle summary and validation sections from the store`
  - Test: `should show warning and blocked issue counts without hardcoded display strings`
  - Use `createTestI18n` and a test router; mock the store/service boundary.
  - Acceptance: tests compile and fail before F006-F007 create the page/components.

- [ ] F006 [US1] [US3] Create `catalog import review components`
  - File: `../hivespace.web/apps/admin/src/components/catalog-imports/{CatalogImportUploadPanel.vue,CatalogImportBundleHistoryTable.vue,CatalogImportRelatedJobsPane.vue,CatalogImportJobHistoryTable.vue,JobSummaryPanel.vue,BundleSummaryPanel.vue,ValidationIssueTable.vue,ImportedProductsTable.vue,ImportedSellersTable.vue,DuplicateGroupsTable.vue,CategoryProvisioningPanel.vue}`
  - Use existing shared table, modal, loading, status, date formatting, money formatting, and confirmation primitives where available.
  - Render list/upload controls, paginated product bundle history, the selected bundle side pane with related jobs, separate category provisioning upload history, job status/progress, product/SKU/category/seller traceability, seller ownership candidates, warning/blocking validation issues, and duplicate groups.
  - All user-facing text must use `catalogImports.*` or `common.*` i18n keys.
  - Every table component must expose pagination props/events and must not assume the full dataset is already loaded.
  - Do not add raw spinner markup, local modal state with `ref<boolean>`, hardcoded money formatting, or raw timestamp rendering.
  - Acceptance: components compile and F005 rendered assertions can find summary, issue, product, seller, duplicate, and category provisioning UI through i18n-resolved labels.

- [ ] F007 [US1] Create `catalog import list/upload and job detail pages`
  - File: `../hivespace.web/apps/admin/src/pages/catalog-imports/CatalogImportListPage.vue`, `../hivespace.web/apps/admin/src/pages/catalog-imports/CatalogImportJobDetailPage.vue`
  - List/upload page: wire category provisioning file submission, product bundle file submission, paginated product bundle history, selected-bundle side pane jobs, separate category provisioning upload history, and row navigation from the side pane to job detail.
  - Detail page: wire selected job detail, linked bundle summary, paginated category links, imported sellers, products/SKUs, duplicate groups, validation issues from the dedicated section endpoints, and follow-up job actions.
  - Keep initial file submission only on the list/upload page; keep bundle review, validation, seller provisioning, seller ownership approval, final import, retry, and polling on the detail page.
  - Keep the main bundle table visible while the selected bundle side pane is open on desktop; use a stacked fallback on narrow screens.
  - Keep page orchestration in pages; keep reusable tables/modals in `components/catalog-imports`.
  - Use `storeToRefs` for store state.
  - Acceptance: F005 tests pass and no component calls service methods directly.

- [ ] F008 [US2] Create `seller provisioning store/page tests - AC2.2 ownership before activation`
  - File: `../hivespace.web/apps/admin/src/stores/catalog-import.store.test.ts`, `../hivespace.web/apps/admin/src/pages/catalog-imports/CatalogImportJobDetailPage.test.ts`
  - Test: `should provision sellers and refresh bundle detail`
  - Test: `should keep mixed-bundle import enabled for eligible rows while conflicted sellers remain excluded`
  - Test: `should render seller conflict reason when provisioning fails`
  - Test: `should render existing store candidate for similar-name seller conflicts`
  - Acceptance: tests compile and fail before F009 extends the store/page.

- [ ] F009 [US2] Update `catalog import store and page for seller provisioning`
  - File: `../hivespace.web/apps/admin/src/stores/catalog-import.store.ts`, `../hivespace.web/apps/admin/src/pages/catalog-imports/CatalogImportJobDetailPage.vue`, `../hivespace.web/apps/admin/src/components/catalog-imports/ImportedSellersTable.vue`
  - Add provision seller action wiring, created/matched/conflict/failed counts, conflict reason display, paginated seller table state, and refresh-after-provisioning behavior on the detail page.
  - Keep mixed-bundle import actions available for currently eligible rows while preventing conflicted seller rows from being selected or imported.
  - Do not expose system-admin provisioning APIs directly to browser code.
  - Acceptance: F008 tests pass.

- [ ] F014 [US2] Create `seller ownership approval UI tests - AC2.8 approve existing store`
  - File: `../hivespace.web/apps/admin/src/stores/catalog-import.store.test.ts`, `../hivespace.web/apps/admin/src/pages/catalog-imports/CatalogImportJobDetailPage.test.ts`
  - Test: `should approve conflicted seller ownership and refresh bundle detail`
  - Test: `should require an approval reason before linking an existing store`
  - Test: `should keep newly approved seller rows non-importable until validation is rerun`
  - Acceptance: tests compile and fail before F015 implements the approval UI flow.

- [ ] F015 [US2] Update `catalog import UI for seller ownership approval`
  - File: `../hivespace.web/apps/admin/src/stores/catalog-import.store.ts`, `../hivespace.web/apps/admin/src/pages/catalog-imports/CatalogImportJobDetailPage.vue`, `../hivespace.web/apps/admin/src/components/catalog-imports/ImportedSellersTable.vue`
  - Add `approveSellerOwnership` store action that calls the CatalogService bundle seller ownership-link endpoint and refreshes bundle detail.
  - Show similar-name existing store candidates, target account/store IDs, conflict reason, and approval reason input for conflicted imported sellers.
  - Require explicit operator confirmation and a non-empty approval reason before calling the approval action.
  - After approval, show the seller as matched but keep only the affected products not ready until validation is rerun; unrelated eligible rows remain importable.
  - Do not call IdentityService or UserService system-admin endpoints from browser code and do not edit store profile data from the approval UI.
  - Acceptance: F014 tests pass and the UI supports resolving a seller/store name conflict through explicit approval.

- [ ] F010 [US3] Create `validation/import action tests - AC3.1 and AC3.2`
  - File: `../hivespace.web/apps/admin/src/stores/catalog-import.store.test.ts`, `../hivespace.web/apps/admin/src/pages/catalog-imports/CatalogImportJobDetailPage.test.ts`
  - Test: `should validate bundle and update blocked warning duplicate counts`
  - Test: `should provision categories before product bundle validation`
  - Test: `should import only ready selected products with the chosen publication state`
  - Test: `should default the publication state selector to Draft`
  - Test: `should allow Draft Unpublish and Available as publication state choices`
  - Test: `should import all eligible products without loading every paginated product row`
  - Test: `should keep bundle-wide import enabled for mixed bundles with at least one eligible row`
  - Test: `should reject selected import when a selected product is no longer eligible`
  - Acceptance: tests compile and fail before F011 extends validation/import UI.

- [ ] F011 [US1] [US3] Update `catalog import UI for category provisioning, validation, and import`
  - File: `../hivespace.web/apps/admin/src/stores/catalog-import.store.ts`, `../hivespace.web/apps/admin/src/pages/catalog-imports/CatalogImportJobDetailPage.vue`, `../hivespace.web/apps/admin/src/components/catalog-imports/{ValidationIssueTable.vue,ImportedProductsTable.vue,DuplicateGroupsTable.vue,CategoryProvisioningPanel.vue}`
  - Wire validate, resolve readiness display, select ready products, require a publication-state selector with `Draft`, `Unpublish`, and `Available` options defaulting to `Draft`, submit the selected state with ready-product import requests, support a bundle-wide `Import all ready` action, and refresh job status after each async operation submission on the detail page.
  - Show blocking issues by entity/field/reason and warnings without hiding reviewable records, and keep import actions enabled whenever at least one product is currently eligible.
  - Keep category provisioning file upload on the list/upload page; detail page only monitors or inspects category provisioning jobs and category-link results.
  - Submit bundle-wide import requests without fetching every paginated ready product row client-side.
  - Prevent blocked or conflicted rows from being selected for import, and surface the backend rejection when an explicit selected-product request contains a row that is no longer eligible at acceptance time.
  - Do not imply buyer-facing visibility for `Draft` or `Unpublish`; show that only `Available` imports are expected to surface immediately, subject to existing storefront filters.
  - Acceptance: F010 tests pass and operators can identify ready, blocked, warning, and duplicate counts from the page state.

### Update

- [ ] F012 [US1] Update `admin router and navigation for catalog imports`
  - File: `../hivespace.web/apps/admin/src/router/index.ts`, `../hivespace.web/apps/admin/src/components/layout/*`, `../hivespace.web/apps/admin/src/i18n/locales/{en,vi}/common.json`
  - Add admin-only routes to `CatalogImportListPage.vue` at `/catalog-imports` and `CatalogImportJobDetailPage.vue` at `/catalog-imports/jobs/:jobId`, plus a navigation entry under the existing catalog/admin operations area.
  - Keep shell/navigation labels in `common`, not inside the feature namespace root.
  - Do not add anonymous or seller/buyer routes.
  - Acceptance: router import resolves, auth guard requires admin/system-admin, and navigation text uses valid i18n keys.

- [ ] F013 [US1] [US2] [US3] Create `catalog import i18n resources`
  - File: `../hivespace.web/apps/admin/src/i18n/locales/en/catalog-imports.json`, `../hivespace.web/apps/admin/src/i18n/locales/vi/catalog-imports.json`, `../hivespace.web/apps/admin/src/i18n/index.ts`
  - Add keys for list page title, detail page title, category upload, product bundle upload, bundle history, selected-bundle side pane, category upload history, file name, pagination labels, statuses, bundle statuses, source fields, summary counts, sellers, existing store candidates, seller ownership approval, categories, products, SKUs, duplicates, validation severities, reason labels, provisioning statuses, job statuses/progress, actions, success/error notifications, import results, and formatted time labels.
  - Update English and Vietnamese resources together.
  - Do not reuse `catalogImports` as both a string and object namespace.
  - Acceptance: i18n files load in admin app and F005/F008/F010 tests resolve translated labels through `createTestI18n`.

- [ ] F016 [US1] [US2] [US3] Create `catalog import job polling tests`
  - File: `../hivespace.web/apps/admin/src/stores/catalog-import.store.test.ts`, `../hivespace.web/apps/admin/src/pages/catalog-imports/CatalogImportJobDetailPage.test.ts`
  - Test: `should store job submission after async operation starts`
  - Test: `should poll job status until completed and refresh affected bundle detail`
  - Test: `should show failed job error summary without hiding existing bundle detail`
  - Acceptance: tests compile and fail before F017 implements job polling UI/state.

- [ ] F017 [US1] [US2] [US3] Update `catalog import UI for async job progress`
  - File: `../hivespace.web/apps/admin/src/stores/catalog-import.store.ts`, `../hivespace.web/apps/admin/src/pages/catalog-imports/CatalogImportListPage.vue`, `../hivespace.web/apps/admin/src/pages/catalog-imports/CatalogImportJobDetailPage.vue`, `../hivespace.web/apps/admin/src/components/catalog-imports/{CatalogImportBundleHistoryTable.vue,CatalogImportRelatedJobsPane.vue,CatalogImportJobHistoryTable.vue,CategoryProvisioningPanel.vue,BundleSummaryPanel.vue,JobSummaryPanel.vue}`
  - List/upload page: show newly submitted bundles in paginated bundle history, refresh separate category provisioning upload history after category submissions, and refresh the selected bundle side pane after bundle-related async submissions.
  - Detail page: show active job status, operation type, processed/total counts, final result summary, and operator-safe error summary for category provisioning, bundle submission, validation, seller provisioning, and ready-product import.
  - Refresh bundle history, selected bundle side-pane jobs, selected job detail, and linked bundle detail after completed jobs whose operation changes state.
  - Keep polling bounded and cancel it when the page unmounts or another job replaces the active operation.
  - Do not treat `202 Accepted` as a completed provisioning, validation, seller provisioning, or import result.
  - Acceptance: F016 tests pass and operators can inspect job lifecycle status for every async catalog import operation.

