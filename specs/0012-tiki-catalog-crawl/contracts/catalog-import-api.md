# Contract: Catalog Import Admin API

All endpoints are browser-facing through ApiGateway under `{VITE_GATEWAY_BASE_URL}/api/v1`.

## Submit Bundle

`POST /api/v1/admins/catalog-imports/bundles`

Auth: `RequireAdmin`

Request: the product bundle JSON shape from [import-bundle-schema.md](import-bundle-schema.md). Category provisioning must already have run for every referenced `externalCategoryId`.

Optional header:

- `X-Source-File-Name`: original uploaded file name for operator traceability.

Response: `202 Accepted`

```json
{
  "jobId": "018f...",
  "status": "Pending",
  "operationType": "SubmitBundle",
  "sourceFingerprint": "sha256:...",
  "sourceFileName": "tiki-category-1846.bundle.json",
  "bundleId": null
}
```

Idempotency:

- Re-submitting the same `sourceFingerprint` while an equivalent job is pending or running returns the existing job submission response.
- Re-submitting a completed `sourceFingerprint` returns the existing completed job or bundle reference unless an explicit supersede option is added in a later version.

## List Bundles

`GET /api/v1/admins/catalog-imports/bundles`

Auth: `RequireAdmin`

Query:

- `pageNumber`: optional integer, default follows backend pagination convention.
- `pageSize`: optional integer, default follows backend pagination convention.

Response: paginated bundle summaries with `bundleId`, source metadata, source file name where available, crawl timestamps, status, counts, and submitted actor.

## Get Bundle Detail

`GET /api/v1/admins/catalog-imports/bundles/{bundleId}`

Auth: `RequireAdmin`

Response: bundle metadata and summary only. Paginated review tables are loaded through the bundle section endpoints below.

- `bundle`

## List Bundle Category Links

`GET /api/v1/admins/catalog-imports/bundles/{bundleId}/category-links`

Auth: `RequireAdmin`

Query: `pageNumber`, `pageSize`, optional `status`.

Response: paginated provisioned external category links for the bundle or category provisioning job context.

## Map Bundle Category Link

`POST /api/v1/admins/catalog-imports/bundles/{bundleId}/category-links/{externalCategoryId}/mapping`

Auth: `RequireAdmin`

Request:

```json
{
  "hiveSpaceCategoryId": 123
}
```

Response:

```json
{
  "bundleId": "018f...",
  "externalCategoryId": "1017",
  "hiveSpaceCategoryId": 123,
  "status": "Mapped",
  "affectedProductCount": 42
}
```

Behavior:

- Operators use this endpoint when validation reports `UnprovisionedCategory` because an imported product references an external category without a provisioned HiveSpace category link.
- The target HiveSpace category must exist and be active.
- Mapping a category link does not automatically make affected products ready; operators must re-run bundle validation after mapping so readiness and blocking issues are recalculated.
- Product import remains blocked while any product `externalCategoryId` is unmapped or mapped to a non-active category.

## List Bundle Imported Sellers

`GET /api/v1/admins/catalog-imports/bundles/{bundleId}/sellers`

Auth: `RequireAdmin`

Query: `pageNumber`, `pageSize`, optional `provisioningStatus`.

Response: paginated imported sellers with ownership status, conflict reason, and existing-store candidates where available.

## List Bundle Imported Products

`GET /api/v1/admins/catalog-imports/bundles/{bundleId}/products`

Auth: `RequireAdmin`

Query: `pageNumber`, `pageSize`, optional `readinessStatus`, `importStatus`, and `sellerId`.

Response: paginated imported products with SKU summaries, category traceability, seller traceability, readiness status, and import status.

## List Bundle Duplicate Groups

`GET /api/v1/admins/catalog-imports/bundles/{bundleId}/duplicate-groups`

Auth: `RequireAdmin`

Query: `pageNumber`, `pageSize`, optional `resolutionStatus`.

Response: paginated duplicate groups and representative product state.

## List Bundle Validation Issues

`GET /api/v1/admins/catalog-imports/bundles/{bundleId}/validation-issues`

Auth: `RequireAdmin`

Query: `pageNumber`, `pageSize`, optional `severity`, `entityType`, and `reasonCode`.

Response: paginated blocking and warning validation issues.

Product-level `UnprovisionedCategory` issues keep the affected product identity in `entityType` and `entitySourceId`, and include the unresolved category repair targets in optional metadata:

```json
{
  "issueId": "03404ec7-9ee3-4736-8300-6e1d88f92309",
  "entityType": "Product",
  "entitySourceId": "5445063",
  "field": "category",
  "severity": "Blocking",
  "reasonCode": "UnprovisionedCategory",
  "message": "Every imported product category must resolve to a provisioned HiveSpace category link.",
  "metadata": {
    "missingExternalCategoryIds": ["914"]
  },
  "createdAt": "2026-09-12T03:49:53.6495046+00:00"
}
```

## Provision Categories

`POST /api/v1/admins/catalog-imports/categories/provisioning`

Auth: `RequireAdmin`

Request: the JSON shape from [category-provisioning-schema.md](category-provisioning-schema.md).

Optional header:

- `X-Source-File-Name`: original uploaded file name for operator traceability.

Response: `202 Accepted`

```json
{
  "jobId": "018f...",
  "sourceFingerprint": "sha256:...",
  "sourceFileName": "categories.json",
  "status": "Pending",
  "operationType": "ProvisionCategories"
}
```

Behavior:

- Operators submit category crawl output before product crawling and product bundle submission.
- The request returns immediately after CatalogService persists the import job and queues CatalogService-owned background processing.
- CatalogService creates or matches real HiveSpace categories and stores external Tiki category links.
- Re-submitting the same category `sourceFingerprint` is idempotent.
- Product import never creates categories implicitly.
- Product crawling and product bundle submission must wait for the required category provisioning job to complete successfully.

Completed job result shape:

```json
{
  "totalCategories": 6101,
  "created": 25,
  "matched": 6070,
  "failed": 0,
  "conflict": 6,
  "results": [
    {
      "externalCategoryId": "1846",
      "categoryId": "018f...",
      "status": "Matched",
      "conflictReason": null
    }
  ]
}
```

## Provision Category Attributes

`POST /api/v1/admins/catalog-imports/categories/attributes/provisioning`

Auth: `RequireAdmin`

Request: the chunk JSON shape from [category-attributes-schema.md](category-attributes-schema.md).

Optional header:

- `X-Source-File-Name`: original uploaded chunk file name for operator traceability.

Response: `202 Accepted`

```json
{
  "jobId": "018f...",
  "sourceFingerprint": "sha256:...",
  "sourceFileName": "chunk-001.json",
  "status": "Pending",
  "operationType": "ProvisionCategoryAttributes"
}
```

Behavior:

- Operators submit category-attribute chunks after category discovery and category provisioning.
- The request returns immediately after CatalogService persists the import job and queues CatalogService-owned background processing.
- CatalogService creates or matches category-scoped attribute definitions and known selectable values for already provisioned category links.
- Re-submitting the same category-attribute `sourceFingerprint` is idempotent.
- Product bundle validation remains blocked for affected categories until required category attributes have been provisioned successfully.

## Validate Bundle

`POST /api/v1/admins/catalog-imports/bundles/{bundleId}/validate`

Auth: `RequireAdmin`

Response: `202 Accepted`

```json
{
  "jobId": "018f...",
  "status": "Pending",
  "operationType": "ValidateBundle",
  "bundleId": "018f..."
}
```

Completed job result: validation summary and issue counts.

Validation responsibilities:

- Seller ownership status.
- Provisioned category links for every product `externalCategoryId`.
- Unprovisioned product category IDs remain visible as unmapped bundle category-link placeholders so an operator can map them later.
- Operator-provided mappings for imported category links when category provisioning did not create or match the required HiveSpace category.
- Required product fields.
- Attribute requirements for provisioned categories.
- SKU presence.
- Positive VND money values.
- Enabled VND currency projection.
- Non-negative stock.
- Media-reference usability.
- Duplicate grouping/resolution.

## Provision Sellers

`POST /api/v1/admins/catalog-imports/bundles/{bundleId}/seller-provisioning`

Auth: `RequireAdmin`

Response: `202 Accepted`

```json
{
  "jobId": "018f...",
  "status": "Pending",
  "operationType": "ProvisionSellers",
  "bundleId": "018f..."
}
```

Completed job result: seller provisioning summary with created, matched, failed, and conflict counts.

Behavior:

- CatalogService calls IdentityService for imported seller accounts.
- CatalogService calls UserService for imported seller stores.
- The command is idempotent for each external seller ID.

## Approve Existing Store For Imported Seller

`POST /api/v1/admins/catalog-imports/bundles/{bundleId}/sellers/{importedSellerId}/ownership-link`

Auth: `RequireAdmin`

Request:

```json
{
  "targetUserId": "018f...",
  "targetStoreId": "018f...",
  "approvalReason": "Existing seed store is the intended HiveSpace owner for this Tiki seller"
}
```

Response:

```json
{
  "importedSellerId": "018f...",
  "externalSellerId": "123",
  "userId": "018f...",
  "storeId": "018f...",
  "status": "Matched",
  "conflictReason": null
}
```

Behavior:

- Operators use this endpoint only after review determines that a conflicted imported seller should sync into an existing HiveSpace seller account/store.
- CatalogService records an approved seller ownership link for `sourceSystem` plus `externalSellerId` so future crawls match by external identity, not by store name.
- The target account and store must be eligible for seller ownership; ineligible targets return a conflict or validation error and leave affected products blocked.
- Approval does not overwrite the existing store profile, name, metadata, owner, lifecycle state, or existing products from Tiki seller metadata.
- Re-run bundle validation after approval so affected products can become ready only when all other validation checks pass.

## Import Ready Products

`POST /api/v1/admins/catalog-imports/bundles/{bundleId}/import`

Auth: `RequireAdmin`

Request:

```json
{
  "publicationState": "Draft"
}
```

`publicationState` is an existing request field. Allowed operator-facing values are `Draft`, `Unpublish`, and `Available`. When omitted, the UI defaults this field to `Draft`.

Optional selected import:

```json
{
  "productIds": ["018f..."],
  "publicationState": "Draft"
}
```

Response: `202 Accepted`

```json
{
  "jobId": "018f...",
  "status": "Pending",
  "operationType": "ImportReadyProducts",
  "bundleId": "018f..."
}
```

Completed job result: imported, skipped, blocked, and failed counts plus per-product results.

Rules:

- `productIds` is optional. When omitted or sent as an empty array, CatalogService imports all eligible products in the bundle.
- When `productIds` is provided and non-empty, CatalogService imports only the selected imported products that remain eligible.
- CatalogService captures the eligible imported product ID set at request acceptance time and executes the async job against that snapshot.
- Retries for the accepted job use the same captured snapshot instead of recalculating eligibility.
- Only products without blocking issues or unresolved duplicate blocks can be imported.
- Products with warning-only issues remain importable when seller ownership, category links, and SKU requirements are satisfied.
- Every product category must resolve to an already provisioned CatalogService category link.
- Imported products are created as seller-owned catalog records using the submitted `publicationState` value.
- `Draft` is the default import state when the operator does not choose another value.
- `Draft` and `Unpublish` remain non-public states and are not expected to appear in storefront product-summary reads.
- `Available` imports are expected to appear immediately in existing storefront product-summary reads, subject to existing storefront filters.

## List Catalog Import Jobs

`GET /api/v1/admins/catalog-imports/jobs`

Auth: `RequireAdmin`

Query:

- `pageNumber`: optional integer, default follows backend pagination convention.
- `pageSize`: optional integer, default follows backend pagination convention.
- `status`: optional `Pending`, `Running`, `Completed`, or `Failed`.
- `operationType`: optional `ProvisionCategories`, `ProvisionCategoryAttributes`, `SubmitBundle`, `ValidateBundle`, `ProvisionSellers`, or `ImportReadyProducts`.
- `sourceSystem`: optional source system filter, initially `tiki`.
- `bundleId`: optional bundle ID filter.
- `requestedFrom` / `requestedTo`: optional requested timestamp range.

Response: paginated import job history for all CatalogService import operations, not only file uploads. Each row includes `jobId`, `operationType`, `status`, `sourceSystem`, `sourceFingerprint`, `sourceFileName`, `bundleId`, requester, timestamps, progress counts, result summary, and error summary when available.

Rules:

- Job history is sorted newest first by default.
- Upload-created and non-upload-created jobs must appear in the same history.
- `sourceFileName` is display metadata only; idempotency uses `sourceFingerprint` or operation-specific keys.

## Get Catalog Import Job

`GET /api/v1/admins/catalog-imports/jobs/{jobId}`

Auth: `RequireAdmin`

Response:

```json
{
  "jobId": "018f...",
  "operationType": "ProvisionCategories",
  "status": "Running",
  "sourceFingerprint": "sha256:...",
  "sourceFileName": "categories.json",
  "bundleId": null,
  "requestedByUserId": "018f...",
  "requestedAt": "2026-08-03T10:15:30Z",
  "startedAt": "2026-08-03T10:15:31Z",
  "completedAt": null,
  "progress": {
    "total": 6101,
    "processed": 3200,
    "created": 20,
    "matched": 3174,
    "failed": 0,
    "conflict": 6
  },
  "resultSummary": null,
  "errorSummary": null
}
```

Rules:

- CatalogService job state is the source of truth for import progress, result details, and retry eligibility.
- `status` is one of `Pending`, `Running`, `Completed`, or `Failed`.
- `operationType` is one of `ProvisionCategories`, `ProvisionCategoryAttributes`, `SubmitBundle`, `ValidateBundle`, `ProvisionSellers`, or `ImportReadyProducts`.
- Detail responses include the linked `bundleId` when present so the admin detail page can load paginated bundle review sections.

## Retry Catalog Import Job

`POST /api/v1/admins/catalog-imports/jobs/{jobId}/retry`

Auth: `RequireAdmin`

Response: `202 Accepted`

```json
{
  "jobId": "018f...",
  "status": "Pending",
  "operationType": "ProvisionCategories",
  "sourceFileName": "categories.json",
  "sourceFingerprint": "sha256:..."
}
```

Rules:

- Retry is optional for v1 but reserved by contract.
- Retry may be allowed only for failed or retryable completed jobs whose operation is idempotent.
- Retrying a duplicate completed category provisioning fingerprint must not create duplicate categories or external links.

## Imported Seller Account Provisioning

`POST /api/v1/admins/imported-seller-accounts`

Owner: IdentityService

Auth: `RequireCatalogImportProvisioning`

Request:

```json
{
  "sourceSystem": "tiki",
  "externalSellerId": "123",
  "displayName": "Tiki Trading",
  "sourceUrl": "https://tiki.vn/cua-hang/tiki-trading"
}
```

Response:

```json
{
  "userId": "018f...",
  "status": "Created",
  "conflictReason": null
}
```

Rules:

- Creates or matches an identity-owned seller account.
- Does not issue browser session cookies, access tokens, refresh tokens, or default passwords to the caller.
- Publishes existing account readiness events when appropriate.

## Imported Seller Store Provisioning

`POST /api/v1/admins/imported-seller-stores`

Owner: UserService

Auth: `RequireCatalogImportProvisioning`

Request:

```json
{
  "sourceSystem": "tiki",
  "externalSellerId": "123",
  "userId": "018f...",
  "storeName": "Tiki Trading",
  "sourceUrl": "https://tiki.vn/cua-hang/tiki-trading"
}
```

Response:

```json
{
  "storeId": "018f...",
  "status": "Created",
  "conflictReason": null
}
```

Rules:

- Creates or matches a UserService-owned store.
- Enforces existing store uniqueness and lifecycle rules.
- Publishes existing `StoreCreatedIntegrationEvent` for new stores.
