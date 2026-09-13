# Feature Specification: Tiki Catalog Crawl

- **Feature Branch**: `0013-tiki-catalog-crawl`
- **Created**: 2026-07-23
- **Status**: Implemented
- **Implemented**: 2026-09-13
- **Input**: User description: "I want to add a new spec to setup a crawl tool to crawl data from some tiki api for this project, can refer to e:\Project\HivespaceProject\hivespace.crawl-data, but I want to use python and need to reflect the current backend model"

## Clarifications

### Session 2026-07-23

- Q: How should preserved Tiki sellers become HiveSpace store ownership for imported products? -> A: Automatically create HiveSpace stores for every new Tiki seller before product import.
- Q: What identity model should be used for automatically created Tiki sellers? -> A: Create full seller user/account records automatically for each new Tiki seller.
- Q: What publication state should valid imported products use after import? -> A: Import valid products using the existing `publicationState` request field with operator-selected `Draft`, `Unpublish`, or `Available`, defaulting to `Draft`; `Available` makes imported products eligible for immediate storefront product-summary visibility subject to existing storefront filters.
- Q: How should Tiki categories be handled before product import? -> A: Operators must crawl Tiki categories first and submit them to CatalogService for category provisioning before crawling and submitting products; there is no operator category-linking step.
- Q: How should imported Tiki prices and currency be validated? -> A: Require VND price validation; unsupported or missing currency blocks affected SKUs.
- Q: Should the production crawler be built in the existing `hivespace.crawl-data` reference repo? -> A: Create a new crawler repository for production work; keep `hivespace.crawl-data` as reference material only.

### Session 2026-07-29

- Q: How should all-category crawling work? -> A: Crawl SellerCenter categories first, save the discovered category cache locally, then crawl products category-by-category.
- Q: How should interrupted all-category crawls resume? -> A: Save local per-category manifest state and skip completed category bundles on rerun; retry failed or unfinished categories.
- Q: How should Tiki anti-bot/challenge responses be handled? -> A: Detect HTML challenge content from Tiki product listing APIs and stop the run so the operator can resume later instead of marking the full remaining category set failed.

### Session 2026-07-30

- Q: What is the required import order for categories and products? -> A: The operator must crawl categories first, submit/provision those categories in CatalogService, and only later crawl and submit product bundles after provisioned categories exist.

### Session 2026-08-16

- Q: How should category-scoped attributes be crawled and handed off for import? -> A: Crawl category attributes as a separate crawler output file after category discovery, then submit/provision that attribute file in CatalogService before product bundle validation or import.

### Session 2026-08-02

- Q: How should crawled Tiki sellers and products behave when existing seed stores or products have similar names? -> A: Existing HiveSpace seed data remains authoritative and must not be deleted, overwritten, or automatically merged. Exact external source identity may be matched automatically; similar store or product names must be surfaced for operator review as conflicts or duplicate-risk warnings before import readiness.
- Q: Can an operator approve a crawled Tiki seller to sync products into an existing HiveSpace store after a similar-name conflict? -> A: Yes. The conflict must remain blocked until an authorized operator explicitly approves linking that external seller identity to the existing HiveSpace seller account and store. The approval creates a durable ownership link for future crawls, must be auditable, and must not overwrite the existing store profile from Tiki data.

### Session 2026-08-03

- Q: How should long-running catalog import operations execute? -> A: Catalog import operations that can exceed normal API request timeouts run as CatalogService-owned asynchronous jobs. The job workers live in the same CatalogService host as background consumers. v1 does not use Azure Functions, a separate worker project, or a central executor. A future central monitor may consume lifecycle events for cross-service observability only and must not execute service-owned import work.

### Session 2026-08-04

- Q: How should the admin Catalog Import UI be organized? -> A: Use a list/upload page for initial category and product file submission plus all import job history, and a separate detail page for inspecting one job or linked bundle, running follow-up workflow actions, resolving seller ownership conflicts, and monitoring progress.
- Q: How should import history and review tables behave? -> A: The list page must show all CatalogService import jobs, not only file-upload submissions, and every backend-backed table in the Catalog Import UI must use pagination.

### Session 2026-08-11

- Q: How should the Catalog Import list page prioritize bundles vs jobs? -> A: The list page should be bundle-first for product imports, with a primary paginated bundle table, a right-side pane that shows jobs for the selected bundle, and a separate category provisioning uploads section.
- Q: Should selecting a bundle collapse the main list? -> A: No. Keep the main bundle table visible and open bundle-specific job history in a side pane similar to the account page pattern.
- Q: How should times be shown in import tables? -> A: Format operator-facing timestamps in local time with an absolute datetime plus a relative hint rather than raw API timestamps.

### Session 2026-09-10

- Q: How should the crawler's current product-level `seller_logo_url` source field be represented in the import specification? -> A: Preserve it as required imported seller metadata in the product bundle using `sellers[].logoUrl`; imported products continue to reference the seller by external seller identity and do not own a separate final product logo field.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Provision Categories Before Product Crawl (Priority: P1)

As a platform operator, I want to crawl Tiki categories first and submit them to HiveSpace so CatalogService has provisioned category records before any product bundle is crawled or submitted.

**Why this priority**: Product imports depend on category ownership. The category tree must exist in CatalogService before products can be validated or attached to catalog categories.

**Independent Test**: Can be fully tested by running the category crawl, submitting the category output to CatalogService, receiving a job ID immediately, polling job status until completion, and confirming categories are created or matched idempotently before product bundle submission is accepted for validation.

**Acceptance Scenarios**:

1. **Given** SellerCenter categories are available, **When** the operator runs the category crawl, **Then** the crawler writes a category output containing external category IDs, parent IDs, names, paths, product set IDs, and source metadata.
2. **Given** SellerCenter categories with product set IDs are available, **When** the operator runs the category attribute crawl, **Then** the crawler writes a separate category-attribute output containing category-scoped attribute definitions, source attribute IDs, required flags, input types, and known selectable values where available.
3. **Given** category output or category-attribute output has not been submitted, **When** the operator tries to submit or validate a product bundle for those categories, **Then** affected products are blocked because provisioned category links or provisioned category-attribute definitions do not exist.
4. **Given** a category output is submitted, **When** CatalogService accepts the provisioning request, **Then** the API returns `202 Accepted` with a catalog import job ID and source fingerprint without waiting for all categories to be processed.
5. **Given** a category-attribute output is submitted to `POST /api/v1/admins/catalog-imports/categories/attributes/provisioning`, **When** CatalogService accepts the provisioning request, **Then** the API returns `202 Accepted` with a catalog import job ID and source fingerprint without waiting for all category attributes to be processed.
6. **Given** a category provisioning or category-attribute provisioning job has been accepted, **When** the operator checks the job status, **Then** the status shows pending, running, completed, or failed state with progress counts and any result or error summary.
7. **Given** CatalogService runs the category provisioning job, **When** provisioning completes, **Then** it creates or matches HiveSpace categories idempotently and records the external Tiki category links.
8. **Given** CatalogService runs the category-attribute provisioning job, **When** provisioning completes, **Then** it creates or matches category-scoped attribute definitions and selectable values idempotently for the provisioned categories.
9. **Given** the same category output or category-attribute output is submitted again, **When** provisioning runs, **Then** existing category links, category attribute definitions, and known selectable values are matched instead of duplicate records being created.
10. **Given** product crawling or product bundle submission depends on category and category-attribute provisioning, **When** a required provisioning job has not completed successfully, **Then** product crawling/import steps must wait or affected product submission/validation remains blocked.
11. **Given** the operator opens the Catalog Import list page, **When** the operator selects a category provisioning file, category-attribute provisioning file, or product bundle file, **Then** the page submits the file and records visible upload history with source file name, requested time, and status in the appropriate bundle or preparation section.
12. **Given** previous product bundle uploads exist, **When** the operator views the Catalog Import list page, **Then** the primary table shows paginated product bundles rather than a flat mixed job list.
13. **Given** a product bundle is selected from the list page, **When** the side pane opens, **Then** the operator can inspect the bundle summary and the paginated related jobs for that bundle without leaving the page.

---

### User Story 2 - Preserve Seller Identity and Create Seller Ownership (Priority: P2)

As a platform operator, I want the import bundle to preserve Tiki seller identity and create full HiveSpace seller ownership for new Tiki sellers before products are activated.

**Why this priority**: HiveSpace product ownership depends on seller/store context. Preserving seller identity and creating seller account plus store ownership prevents imported products from being assigned to the wrong seller.

**Independent Test**: Can be fully tested by crawling products from multiple Tiki sellers and confirming every imported product references its external seller and has a created, matched, or conflict status for HiveSpace seller account and store ownership.

**Acceptance Scenarios**:

1. **Given** categories and category attributes have been provisioned, **When** the operator crawls products from Tiki, **Then** the product bundle preserves seller, seller logo, product, category, SKU, price, stock, image, and attribute traceability including source attribute identity and value-shaping metadata where available.
2. **Given** Tiki returns products with missing optional fields, **When** the bundle is produced, **Then** required facts are preserved, optional gaps are recorded, and the bundle remains reviewable.
3. **Given** crawled products from multiple Tiki sellers, **When** the import bundle is reviewed, **Then** each product is associated with the Tiki seller identity captured from the source.
4. **Given** a Tiki seller has no HiveSpace seller ownership, **When** the import workflow prepares products for activation, **Then** CatalogService accepts seller provisioning as an asynchronous import job when the operation is long-running and creates the required HiveSpace seller account and store through the account- and store-owning workflows before any products from that seller become ready.
5. **Given** a crawled Tiki seller already has an approved HiveSpace ownership link for the same external seller identity, **When** seller provisioning runs, **Then** that existing ownership is matched instead of creating another store.
6. **Given** a crawled Tiki seller only has a similar name to an existing HiveSpace store, **When** seller provisioning runs, **Then** the similarity is reported for operator review and affected products remain not ready until the conflict is resolved.
7. **Given** a crawled Tiki seller has no approved ownership link and no blocking store conflict, **When** seller provisioning runs, **Then** seller account and store ownership are created through the owning services before affected products can become ready, and the operator can inspect the job result counts after completion.
8. **Given** a crawled Tiki seller is blocked by a similar-name store conflict, **When** an operator approves linking that Tiki seller to the existing HiveSpace seller account and store, **Then** the seller conflict is resolved, future crawls for the same external seller identity match automatically, and affected products may become ready after revalidation.
9. **Given** a job is linked to an import bundle, **When** the operator opens the job detail page, **Then** the operator can inspect paginated imported seller rows and resolve eligible seller ownership conflicts from that detail context.
10. **Given** the operator selects a bundle from the Catalog Import list page, **When** the related-jobs side pane loads, **Then** the operator can understand the submit, validate, seller provisioning, and import jobs for that bundle in one grouped view.

---

### User Story 3 - Validate Against HiveSpace Catalog Rules (Priority: P3)

As a platform operator, I want the import bundle to show validation issues against HiveSpace catalog expectations so bad or incomplete records can be fixed before import.

**Why this priority**: Imported data must fit existing product, SKU, category, attribute, currency, stock, and media-reference rules before it can safely enter seller catalog workflows.

**Independent Test**: Can be fully tested by validating a product bundle that references unprovisioned categories, invalid prices, unavailable stock, seller ownership conflicts, and image issues, then confirming each issue appears in a validation report.

**Acceptance Scenarios**:

1. **Given** a bundle contains records that do not match HiveSpace catalog requirements, **When** validation runs as a long-running job, **Then** every blocked product includes a reason and the affected field or relationship after the job completes.
2. **Given** a bundle contains products with seller account and store ownership plus a mix of eligible and blocked records, **When** validation completes and an import-ready-products job runs for selected products or for all eligible products in the bundle with operator-selected `Draft`, `Unpublish`, or `Available` publication state, **Then** products with `Ready` status and products with warning-only issues are imported using that selected state, defaulting to `Draft`, while blocked records remain excluded from the import and only `Available` imports are eligible for immediate storefront product-summary visibility subject to existing storefront filters.
3. **Given** a crawled product has a similar title to an existing HiveSpace product, **When** validation runs, **Then** the validation report shows duplicate risk and prevents automatic overwrite or silent merge with the existing product.
4. **Given** product bundle submission, validation, seller provisioning, or ready-product import may exceed normal request timeouts, **When** the operator submits the operation, **Then** the API returns a job ID immediately and the operator monitors progress through the job status endpoint.
5. **Given** the operator opens a catalog import job detail page, **When** the job has a linked bundle, **Then** the page shows bundle summary plus paginated category links, products and SKUs, validation issues, duplicate groups, and imported sellers.
6. **Given** a bundle has blocking category, seller ownership, validation, or duplicate issues alongside `Ready` or warning-only products, **When** the operator views the detail page, **Then** ready-product import remains enabled for the currently eligible products while blocked records remain excluded until the blocking conditions are resolved and validation has been rerun.
7. **Given** a bundle has multiple import-eligible products across paginated results, **When** the operator chooses to import all ready products, **Then** the API accepts one async import job without requiring the frontend to enumerate every ready product ID client-side.
8. **Given** the operator submits a bundle-wide import request, **When** the import job is accepted, **Then** CatalogService captures a submit-time snapshot of eligible imported product IDs and executes the job against that fixed set.
9. **Given** a bundle contains `Blocked`, `Ready`, and warning-only products, **When** the operator imports all eligible products or a selected eligible subset, **Then** the accepted import job includes only the products that are currently eligible and leaves blocked products pending revalidation or conflict resolution.
10. **Given** the operator reviews bundle or category history on the list page, **When** timestamps are shown, **Then** the UI displays operator-friendly local time formatting with a relative hint instead of raw timestamp strings.
11. **Given** a crawled product resolves to a provisioned HiveSpace category, **When** validation evaluates imported attributes, **Then** the attributes are matched against that category's provisioned attribute definitions using stable source attribute identity where available, with selected value IDs preferred over display-text matching and free-text fallback retained for operator review when no selectable value match exists.

### Edge Cases

- Tiki returns a product without SKU or variant detail.
- Tiki returns price data without a supported or recognizable currency.
- Product bundles reference Tiki categories that have not been provisioned in CatalogService.
- Category provisioning receives renamed, duplicate, reordered, or missing-parent Tiki categories.
- Tiki returns duplicate products across category and search crawls.
- Tiki category discovery returns nested child categories inside a parent response.
- Tiki returns HTML challenge content instead of product listing JSON.
- Tiki seller identity changes or conflicts with an existing HiveSpace seller account or store.
- A crawled Tiki seller has a display name similar to an existing seed store but no approved external seller ownership link.
- An operator approves a crawled Tiki seller to use an existing store whose owner is not eligible for seller ownership.
- A crawled Tiki product has a title similar to an existing seed product but no exact external product identity match.
- Product image links are missing, inaccessible, duplicated, or point to unsupported media.
- Tiki seller logo URL is missing, empty, inaccessible, duplicated, or malformed.
- Stock quantity is missing, unknown, or inconsistent across SKU records.
- A crawl is interrupted after only part of a source selection is collected.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST require the operator to crawl and submit Tiki category data before product crawl output can be considered ready for validation or import.
- **FR-001a**: System MUST require the operator to crawl and submit category-scoped Tiki attribute-definition data as a separate output file after category discovery and before product bundle validation or import can be considered ready.
- **FR-002**: System MUST produce an import bundle that preserves source traceability for crawled products, categories, sellers, seller logos, SKUs, variants, images, prices, stock, and attributes.
- **FR-003**: System MUST represent imported products using HiveSpace catalog concepts: product, category, attribute, variant, SKU, price, stock quantity, image reference, and seller/store ownership.
- **FR-004**: System MUST preserve Tiki seller identity and seller metadata separately from HiveSpace store ownership.
- **FR-004a**: System MUST preserve the crawler's Tiki seller logo source value as imported seller `logoUrl` metadata; imported product records MUST reference seller identity rather than treating seller logo as product-owned catalog data.
- **FR-005**: System MUST create HiveSpace seller account and store ownership for every new Tiki seller before products from that seller can be considered ready for catalog activation.
- **FR-006**: System MUST identify products that cannot be imported because of seller account or store ownership conflicts, missing category provisioning, invalid money data, invalid stock data, missing required product fields, or unusable media references.
- **FR-007**: System MUST prevent duplicate imported products from being treated as separate ready-to-import products when they refer to the same Tiki source product.
- **FR-008**: System MUST keep crawled catalog data out of buyer-facing storefront results until the data has passed validation and store ownership checks.
- **FR-009**: System MUST provide a validation report that distinguishes ready records, blocked records, warnings, and duplicate records.
- **FR-009a**: System MUST treat warning-only products as import-eligible on a per-product basis when they have no blocking validation issues, no unresolved duplicate block, valid seller ownership, valid provisioned categories, and at least one importable SKU, even if other products in the same bundle remain blocked.
- **FR-010**: System MUST record enough crawl summary information for an operator to know when the crawl ran, which source selection was used, how many records were collected, and how many records require attention.
- **FR-011**: System MUST import valid products using the existing `publicationState` request field with operator-facing choices limited to `Draft`, `Unpublish`, and `Available`, defaulting to `Draft`.
- **FR-011c**: System MUST treat `Draft` and `Unpublish` imports as non-public states and MUST allow `Available` imports to appear immediately in existing storefront product-summary flows subject to existing storefront filters.
- **FR-011a**: System MUST support both selected-product import and bundle-wide import of all eligible products through the same ready-product import workflow.
- **FR-011b**: System MUST allow selected-product import and bundle-wide import to proceed from a mixed-status bundle when at least one product is currently import-eligible, and MUST exclude non-eligible products from the accepted import snapshot.
- **FR-012**: System MUST provision crawled Tiki categories into CatalogService before product bundle submission; product import MUST NOT create categories.
- **FR-013**: System MUST require VND price validation for imported Tiki SKUs; unsupported, missing, or non-numeric currency/price data MUST block affected SKUs and appear in the validation report.
- **FR-014**: System MUST cache discovered SellerCenter categories locally and reuse the cache unless an operator explicitly refreshes categories.
- **FR-014a**: System MUST write discovered category output separately from category-attribute output so category hierarchy/provisioning data and category attribute-definition data can be retried, diffed, and provisioned independently.
- **FR-014b**: System MUST preserve `productSetId` or equivalent source category metadata required to crawl category-scoped attribute definitions from SellerCenter.
- **FR-015**: System MUST save all-category crawl progress per category so reruns skip completed category bundles and retry failed or unfinished categories.
- **FR-016**: System MUST stop an all-category crawl when Tiki product APIs return HTML challenge content instead of JSON, preserving completed category output and resumable state.
- **FR-017**: System MUST block products whose `externalCategoryIds` do not resolve to previously provisioned CatalogService category links.
- **FR-017a**: System MUST resolve imported product attributes against the attribute definitions of the previously provisioned HiveSpace category or categories; missing or incompatible required category attributes MUST block readiness, and unmatched optional attributes MUST remain reviewable instead of being silently discarded.
- **FR-017b**: System MUST preserve imported attribute source identity using source attribute ID when available instead of relying only on display name/value text matching.
- **FR-017c**: System MUST represent imported attribute values as selected source value IDs when a category-scoped selectable value match exists and as free-text fallback when no selectable value match exists.
- **FR-017d**: System MUST provision category-scoped attribute definitions and known selectable values idempotently from the separate category-attribute output file before product bundle validation can treat attribute readiness as satisfied.
- **FR-018**: System MUST treat existing HiveSpace seed stores and products as authoritative records that cannot be overwritten, deleted, or automatically merged by crawled Tiki data.
- **FR-019**: System MUST automatically match seller ownership only when the crawled seller has the same source system and external seller identity as an existing approved ownership link.
- **FR-020**: System MUST NOT automatically match seller ownership using display-name similarity alone; similar store names MUST be reported for operator review and MUST block affected products from import readiness until resolved.
- **FR-021**: System MUST report duplicate risk when crawled products are similar to existing HiveSpace products, and high-confidence or unresolved duplicate conflicts MUST prevent affected records from becoming ready for import.
- **FR-022**: System MUST prevent imported products from overwriting or silently merging with existing HiveSpace products unless an exact external product source identity match or a separate explicit product-duplicate resolution allows it.
- **FR-023**: System MUST allow an authorized operator to explicitly approve linking a conflicted crawled Tiki seller to an existing HiveSpace seller account and store.
- **FR-024**: System MUST record operator-approved seller ownership links with the approving actor, approval time, source seller identity, target HiveSpace account/store, and reason or note.
- **FR-025**: System MUST reject operator-approved seller ownership links when the target account or store is not eligible to own imported seller products.
- **FR-026**: System MUST NOT update the existing HiveSpace store profile, name, metadata, owner, or lifecycle state from crawled Tiki seller metadata as part of operator approval.
- **FR-027**: System MUST process catalog import operations asynchronously when they can exceed normal request timeouts, including category provisioning, product bundle submission where persistence or initial validation is long-running, bundle validation, seller provisioning, and ready-product import.
- **FR-027a**: System MUST treat an omitted or empty `productIds` selection in the ready-product import request as a bundle-wide import of all currently eligible imported products.
- **FR-027b**: System MUST reject a selected-product ready-product import request when any submitted product ID is not currently import-eligible at request acceptance time instead of silently importing a partial subset from that explicit selection.
- **FR-028**: System MUST persist each catalog import job with status, operation type, requester, source or bundle identity, source fingerprint where available, timestamps, progress counts, and final result or error summary.
- **FR-028a**: System MUST capture the eligible imported product ID set for a bundle-wide ready-product import at request acceptance time so the async job result and retry behavior are deterministic.
- **FR-029**: System MUST return a job ID instead of blocking the API request until long-running catalog import work completes.
- **FR-030**: System MUST allow operators to inspect catalog import job progress and final result.
- **FR-031**: System MUST keep catalog import domain execution inside CatalogService; no Azure Function, separate worker project, or central executor owns v1 import work.
- **FR-032**: System MUST publish catalog import job lifecycle events for observer-only monitoring without transferring domain ownership or result authority away from CatalogService.
- **FR-033**: System MUST provide an admin Catalog Import list/upload page where operators submit category provisioning files, category-attribute provisioning files, and product bundle files.
- **FR-034**: System MUST show paginated product bundle history as the primary list-page table for product imports and MUST allow operators to inspect the paginated related job history for the selected bundle from that page.
- **FR-035**: System MUST provide an admin Catalog Import detail page where operators inspect one job, inspect the linked bundle when present, run follow-up workflow actions, resolve seller ownership conflicts, and monitor job progress.
- **FR-036**: System MUST include operator-visible source file name metadata for category provisioning and product bundle submissions without using file names for idempotency.
- **FR-037**: System MUST paginate every backend-backed table in the Catalog Import UI, including job history, bundle lists, category links, imported sellers, products/SKUs, duplicate groups, and validation issues.
- **FR-038**: System MUST show category provisioning uploads and category-attribute provisioning uploads in separate paginated preparation sections on the Catalog Import list page instead of mixing them into the primary product-bundle table.
- **FR-039**: System MUST keep the main product-bundle table visible when a bundle is selected and show the selected bundle's related jobs in a side pane on the same page.
- **FR-040**: System MUST format operator-facing list-page timestamps in local time with an absolute datetime and a relative hint.

### Key Entities

- **Crawl Source**: A Tiki category, keyword, product list, or equivalent source selection used to collect external catalog data.
- **Crawler Category Cache**: Local file state containing SellerCenter category IDs, names, parent IDs, and product set IDs discovered before product crawling.
- **Category Attribute Crawl Output**: Separate crawler-produced file containing category-scoped attribute definitions, source attribute IDs, required flags, input types, selectable values where available, and source linkage to discovered categories or product set IDs.
- **Crawler Manifest**: Local per-category progress state that records pending, running, complete, and failed category crawl status for resume.
- **Category Provisioning Submission**: A category-only payload submitted before product crawling that lets CatalogService create or match HiveSpace categories and store external Tiki category links.
- **Category Attribute Provisioning Submission**: A category-attribute payload submitted after category discovery that lets CatalogService create or match category-scoped attribute definitions and known selectable values before product validation.
- **Import Bundle**: A reviewable set of crawled records and validation results created from one crawl run.
- **Imported Seller**: External seller identity, display metadata, and logo URL captured from Tiki for HiveSpace seller account and store ownership creation or matching.
- **Seller Ownership Link**: Relationship between an imported seller, HiveSpace seller account, and HiveSpace store that authorizes product ownership during import.
- **Seller Ownership Approval**: Operator decision that resolves a seller conflict by linking one external seller identity to an existing HiveSpace seller account and store for current and future imports.
- **Imported Product**: External product record shaped for HiveSpace catalog review, including title, description, category reference, seller reference, images, attributes, variants, and SKUs.
- **Existing HiveSpace Store**: A store already present in HiveSpace, including seed stores, that may be reviewed as a possible seller ownership conflict but is not automatically matched by name alone.
- **Existing HiveSpace Product**: A product already present in HiveSpace, including seed products, that may be reviewed as a duplicate risk but is not overwritten or silently merged by imported data.
- **Imported SKU**: Purchasable product variation with SKU identifier, variant selections, VND price validation status, stock quantity when available, and active/readiness status.
- **Provisioned External Category**: External Tiki category reference linked to a CatalogService category before affected products can be submitted and imported.
- **Imported Attribute**: Product metadata captured from Tiki and matched or flagged against provisioned HiveSpace category attribute expectations using source attribute ID, selected value IDs, or free-text fallback as available.
- **Import Validation Report**: Operator-facing result that explains readiness, warnings, duplicate records, and blocking issues.
- **Catalog Import Job**: Durable CatalogService-owned record for asynchronous category provisioning, bundle submission, validation, seller provisioning, or ready-product import work, including operation type, requester, status, timestamps, counts, source or bundle reference, and result or error summary.
- **Import Selection Snapshot**: The immutable set of imported product IDs resolved when a ready-product import job is accepted, used to execute and retry a selected or bundle-wide import deterministically.
- **Catalog Import List/Upload Page**: Admin page for category/product file submission, a paginated product-bundle table, a side pane for selected-bundle job history, and a separate paginated category provisioning uploads section.
- **Catalog Import Detail Page**: Admin page for one catalog import job and its linked bundle review, paginated related data, follow-up workflow actions, conflict approval, retry, and progress monitoring.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: For a valid Tiki source selection containing at least 20 products, the system produces a reviewable import bundle with product and seller traceability for at least 95% of collected product records.
- **SC-002**: 100% of products from new Tiki sellers have HiveSpace seller account and store ownership created before they are imported using the operator-selected `Draft`, `Unpublish`, or `Available` publication state.
- **SC-003**: 100% of products with missing required catalog fields, unsupported money data, invalid stock data, missing category provisioning, or unusable media references are reported with actionable validation reasons.
- **SC-004**: Duplicate source products collected in the same bundle are detected and grouped so operators do not review them as unrelated products.
- **SC-005**: An operator can identify crawl totals, ready records, blocked records, warnings, and duplicate records from the validation report in under 2 minutes for a 100-product bundle.
- **SC-006**: 100% of crawled sellers with only name similarity to existing HiveSpace stores are reported for operator review instead of being automatically matched.
- **SC-007**: 100% of imported products leave existing seed products unchanged unless the import has an exact external product source identity match or a separate product-duplicate resolution; seller ownership approval alone never changes existing products.
- **SC-008**: 100% of operator-approved seller ownership links can be traced to the approving actor, source seller identity, target store, timestamp, and approval reason.
- **SC-009**: Full Tiki category provisioning for 6,000+ categories starts without HTTP 504 and returns a catalog import job ID within normal API response time.
- **SC-009a**: Full Tiki category-attribute provisioning through `POST /api/v1/admins/catalog-imports/categories/attributes/provisioning` starts without HTTP 504 and returns a catalog import job ID within normal API response time after the crawler has produced the separate category-attribute output file.
- **SC-010**: Operators can see final created, matched, failed, and conflict counts after each completed category provisioning or seller provisioning job.
- **SC-011**: Retrying a duplicate completed category provisioning fingerprint does not create duplicate categories or external category links.
- **SC-012**: Job lifecycle status is visible for every asynchronous catalog import operation.
- **SC-013**: Operators can find a product import bundle from the primary paginated bundle table by file name where available, source, time, and summary status, then inspect its related jobs from the side pane.
- **SC-014**: Operators can open an import job detail page from the selected bundle's related-jobs pane and inspect the linked bundle's paginated sellers, products/SKUs, category links, duplicate groups, and validation issues without loading all rows at once.

## Assumptions

- The first release imports catalog essentials only; reviews, promotions, shipping policies, and order data are out of scope.
- HiveSpace account records remain owned by IdentityService, store records remain owned by UserService, and product catalog records remain owned by CatalogService.
- A product is not eligible for import as `Available` until its external seller has a HiveSpace seller account and store ownership link and the imported catalog record has passed review; `Draft` remains the default import state and `Unpublish` remains a supported non-public alternative.
- Category provisioning creates or matches CatalogService category records before product crawling and product bundle submission.
- Category-attribute provisioning creates or matches CatalogService category attribute definitions and known selectable values after category discovery and before product bundle validation/import.
- Existing HiveSpace category, attribute, currency, stock, and media ownership rules remain authoritative.
- Existing HiveSpace seed stores and seed products are representative production data for conflict handling and must be protected from silent import merges.
- Exact external source identity is the default automatic matching rule for imported sellers and products; similar names are review signals, not ownership proof.
- Operator-approved seller ownership links resolve external seller ownership only; they do not authorize automatic edits to the existing store profile or existing products.
- Tiki seller logo is preserved for import review and seller provisioning traceability; it does not create a new final product field or override existing HiveSpace store logo/profile data by itself.
- Tiki catalog prices are expected to be VND, but the import workflow must validate this expectation instead of silently assuming it.
- The existing `hivespace.crawl-data` repository is a reference for crawl experimentation only; production crawler work belongs in a new sibling repository.
- The crawler may call SellerCenter category-attribute definition APIs and attribute-value lookup APIs, but CatalogService remains authoritative for final attribute validation and import readiness.
- Worker execution for catalog import jobs stays in the same CatalogService deployment for v1.
- A future central monitoring service may consume job lifecycle events, but CatalogService job status remains the source of truth for import progress, results, and errors.
- No MassTransit saga is introduced for catalog import because the workflow is operator-triggered, idempotent, retryable, and does not require cross-service compensation state.
- Original source file names are retained for operator traceability only; source fingerprints remain the idempotency key for category provisioning and product bundle submissions.
- `Inactive` may remain an internal or compatibility lifecycle alias in implementation, but it is not an operator-facing publication-state choice for this import workflow.
- This publication-state update reuses the existing ready-product import endpoint and `publicationState` request field; it does not require a new endpoint, request field, or admin route.
