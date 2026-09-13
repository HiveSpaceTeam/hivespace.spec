# CatalogService API

## Admin Catalog Imports

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/admins/catalog-imports/categories/provisioning` | `RequireAdmin` | Queue a CatalogService-owned async job to create or match categories from crawled Tiki category data before product crawling and store external category links; returns `202 Accepted` with job ID |
| POST | `/api/v1/admins/catalog-imports/categories/attributes/provisioning` | `RequireAdmin` | Queue a CatalogService-owned async job to create or match category-scoped attribute definitions and selectable values from crawled Tiki category-attribute data; returns `202 Accepted` with job ID |
| POST | `/api/v1/admins/catalog-imports/bundles` | `RequireAdmin` | Queue a CatalogService-owned async job to persist a Python-generated product import bundle after category provisioning; returns `202 Accepted` with job ID |
| GET | `/api/v1/admins/catalog-imports/bundles` | `RequireAdmin` | List paginated catalog import bundles with source metadata, source file name where available, validation status, and summary counts |
| GET | `/api/v1/admins/catalog-imports/bundles/{bundleId}` | `RequireAdmin` | Get catalog import bundle metadata and summary for the detail page |
| GET | `/api/v1/admins/catalog-imports/bundles/{bundleId}/category-links` | `RequireAdmin` | List paginated provisioned external category links for a catalog import bundle |
| GET | `/api/v1/admins/catalog-imports/bundles/{bundleId}/sellers` | `RequireAdmin` | List paginated imported sellers with ownership status, conflict reason, and existing-store candidates where available |
| GET | `/api/v1/admins/catalog-imports/bundles/{bundleId}/seller-ownership-candidates` | `RequireAdmin` | List paginated seller ownership candidates for imported seller conflict review |
| GET | `/api/v1/admins/catalog-imports/bundles/{bundleId}/products` | `RequireAdmin` | List paginated imported products with SKU summaries, category traceability, seller traceability, readiness status, and import status |
| GET | `/api/v1/admins/catalog-imports/bundles/{bundleId}/duplicate-groups` | `RequireAdmin` | List paginated duplicate product groups and representative product state for a catalog import bundle |
| GET | `/api/v1/admins/catalog-imports/bundles/{bundleId}/validation-issues` | `RequireAdmin` | List paginated blocking and warning validation issues for a catalog import bundle, including optional metadata such as missing external category IDs |
| POST | `/api/v1/admins/catalog-imports/bundles/{bundleId}/category-links/{externalCategoryId}/mapping` | `RequireAdmin` | Map an imported category link to an existing active HiveSpace category so `UnprovisionedCategory` validation blockers can be cleared after revalidation |
| POST | `/api/v1/admins/catalog-imports/bundles/{bundleId}/validate` | `RequireAdmin` | Queue a CatalogService-owned async job to re-run catalog import validation after category provisioning or seller provisioning state changes; returns `202 Accepted` with job ID |
| POST | `/api/v1/admins/catalog-imports/bundles/{bundleId}/seller-provisioning` | `RequireAdmin` | Queue a CatalogService-owned async job for idempotent imported seller account and store provisioning through IdentityService and UserService ownership boundaries; returns `202 Accepted` with job ID |
| POST | `/api/v1/admins/catalog-imports/bundles/{bundleId}/sellers/{importedSellerId}/ownership-link` | `RequireAdmin` | Approve linking a conflicted imported Tiki seller to an existing eligible HiveSpace seller account/store without overwriting existing store or product data |
| POST | `/api/v1/admins/catalog-imports/bundles/{bundleId}/import` | `RequireAdmin` | Queue a CatalogService-owned async job to import ready bundle products with operator-selected `Draft`, `Unpublish`, or `Available` publication state; returns `202 Accepted` with job ID |
| GET | `/api/v1/admins/catalog-imports/jobs` | `RequireAdmin` | List paginated all-operation CatalogService catalog import job history with job ID, source file name where available, operation type, requested/completed times, status, linked bundle, progress counts, and result/error summary |
| GET | `/api/v1/admins/catalog-imports/jobs/{jobId}` | `RequireAdmin` | Get CatalogService-owned catalog import job status, progress counts, final result summary, and error summary |
| POST | `/api/v1/admins/catalog-imports/jobs/{jobId}/retry` | `RequireAdmin` | Optionally retry an idempotent failed or retryable completed catalog import job; returns `202 Accepted` with job ID |

## Seller Products

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/products` | `RequireSeller` | Create product with explicit price currency validated against the enabled platform currency policy |
| GET | `/api/v1/products` | `RequireSeller` | List seller products |
| GET | `/api/v1/products/{id}` | `RequireSeller` | Get seller product detail with explicit SKU money metadata and invalid-money diagnostics when needed |
| PUT | `/api/v1/products/{id}` | `RequireSeller` | Update product with explicit price currency validated against the enabled platform currency policy |
| DELETE | `/api/v1/products/{id}` | `RequireSeller` | Delete or deactivate product |

## Storefront Products

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/products/summaries` | Anonymous | Search/list storefront product summaries |
| GET | `/api/v1/products/detail/{id}` | Anonymous | Get storefront product detail with SKUs, explicit money metadata, and invalid-money diagnostics when needed |

## Categories

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/categories` | Anonymous | Get full category tree |
| GET | `/api/v1/categories/homepage` | Anonymous | Get homepage category list |
| GET | `/api/v1/categories/{id}/attributes` | Anonymous | Get category attribute definitions |
