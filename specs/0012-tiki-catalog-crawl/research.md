# Research: Tiki Catalog Crawl

## Decision: Use a new Python crawler/exporter repository plus backend import APIs

The production crawler will live in a new sibling repository, `../hivespace.crawler`, as a Python tool that reads Tiki sources and emits a versioned HiveSpace import bundle. HiveSpace backend services consume that bundle through admin/system APIs and remain authoritative for identity, store, catalog, currency, media, and publication rules. The existing `../hivespace.crawl-data` repository remains reference material only.

Rationale: The user explicitly prefers Python for crawling and clarified that production crawler work should use a new repository. Keeping crawling outside the .NET services isolates external-source volatility while allowing backend validation to reflect the current domain model.

Alternatives considered:

| Option | Why rejected |
| --- | --- |
| Put Tiki crawling inside CatalogService | Couples external scraping/API volatility to the catalog runtime and complicates service reliability. |
| Let Python write directly to HiveSpace databases | Violates service data ownership and bypasses domain validation/outbox behavior. |
| Build production code in `../hivespace.crawl-data` | That repo is only an experiment/reference and should not become the durable production crawler boundary. |
| Use only existing seller product APIs | Existing seller APIs require seller context and cannot create Identity/User seller ownership for imported sellers. |

## Decision: All-category crawling uses SellerCenter category discovery with local resume state

The Python crawler supports an all-category mode that first discovers Tiki SellerCenter categories, saves the category cache locally, and expects the operator to submit that category output to CatalogService provisioning before product crawling and product bundle submission. Category discovery follows the working reference pattern from `../hivespace.crawltool`: `GET https://sellercenter.tiki.vn/api/tiki_api?path=catalog%2Fcategories&lang=vi&parent_id={parentId}&limit=999&is_active=1&include_promotion=0`, starting at `parent_id=2` and recursively treating returned category IDs as child parent IDs. Discovery must also include nested child categories when Tiki returns children in the same response payload.

All-category crawler state is local file state under `crawler-state/tiki/`: `categories.json` for discovered categories, `manifest.json` for per-category progress, and `bundles/category-{categoryId}.bundle.json` for completed category bundles. If `categories.json` exists, the crawler reuses it unless the operator passes a refresh option. Reruns skip manifest entries marked complete when the matching bundle file exists; failed or running categories are retried from the start of that category.

Rationale: SellerCenter categories better reflect the import/category/attribute model used by the reference crawler than public menu categories. Submitting categories before product crawling gives CatalogService durable category links before products reference them. Local file state keeps the Python crawler simple and dependency-free while making long all-category crawls resumable after failures or interruptions.

Alternatives considered:

| Option | Why rejected |
| --- | --- |
| Public Tiki desktop menu categories only | Easier to access, but less aligned with the SellerCenter catalog hierarchy and product set metadata used by the reference crawl tool. |
| One combined all-category bundle | Harder to resume, retry, and submit safely when Tiki blocks or one category fails. Per-category bundles preserve progress. |
| SQLite crawler state | More queryable, but adds dependency and operational complexity before local JSON state proves insufficient. |
| Product/page-level resume | More precise, but category-level resume is simpler and enough for first production crawler behavior. |

## Decision: Stop on Tiki HTML challenge responses

When Tiki product listing APIs return HTML content instead of JSON, the crawler treats that as a challenge/block response and stops the all-category run after recording the current category as failed. It must not continue through the remaining category list and mark thousands of categories failed.

Rationale: Live validation on 2026-07-29 discovered 6,101 SellerCenter categories, wrote 6 category bundles, then Tiki began returning `text/html` challenge content from `https://tiki.vn/api/v2/products?...`. Stopping early preserves useful output and leaves a clean resume point for a later run.

Alternatives considered:

| Option | Why rejected |
| --- | --- |
| Continue and mark every category failed | Produces noisy manifest state and hides the real operational issue. |
| Retry indefinitely | Risks worsening source blocking and prevents operator control. |
| Bypass challenge handling in v1 | Makes real all-category crawls brittle and hard to resume. |

## Decision: CatalogService owns import bundle persistence and catalog validation

CatalogService will provision submitted category and category-attribute crawl output first, persist later product import bundles, run validation against product/category-link/attribute/SKU/stock/money/media-reference rules, group duplicate source products, and create catalog records for ready products with the accepted `Draft`, `Unpublish`, or `Available` publication state.

Rationale: CatalogService owns products, SKUs, categories, attributes, product media associations, store refs, and local currency-policy validation projection. Category provisioning and import readiness are catalog decisions, even though account and store provisioning remain owned elsewhere.

Alternatives considered:

| Option | Why rejected |
| --- | --- |
| New ImportService | Adds a new service boundary before the need is proven and would still need CatalogService to validate catalog truth. |
| Admin frontend validates bundle only | Frontend validation cannot be authoritative and would duplicate backend domain rules. |
| Python validates all HiveSpace rules | Python cannot safely own current backend rules without duplicating domain logic and service projections. |

## Decision: IdentityService and UserService expose idempotent system/admin provisioning APIs

IdentityService will provide create-or-match imported seller account behavior. UserService will provide create-or-match imported seller store behavior. CatalogService orchestrates these via application ports when an admin requests seller provisioning.

Rationale: IdentityService owns accounts, credentials, roles, claims, and account status. UserService owns stores. Provisioning imported sellers must respect both boundaries and keep propagation through existing `IdentityUserReadyIntegrationEvent` and `StoreCreatedIntegrationEvent`.

Alternatives considered:

| Option | Why rejected |
| --- | --- |
| CatalogService creates accounts/stores directly | Violates identity and store ownership boundaries. |
| Manual seller onboarding before import | Conflicts with the clarified requirement to create full seller account/store records automatically. |
| Create one platform-owned store for all imported products | Conflicts with preserving Tiki seller identity and seller/store ownership. |

## Decision: Similar-name store conflicts require explicit CatalogService ownership approval

When a crawled Tiki seller has only a similar display name to an existing HiveSpace store, the import workflow must not automatically map products to that store. CatalogService records the conflict and blocks affected products until an authorized operator approves a seller ownership link to an eligible existing HiveSpace seller account/store. Once approved, future imports match by `sourceSystem` plus `externalSellerId`, not by display name. Approval must be audited and must not overwrite existing store profile data from Tiki metadata.

Rationale: Seed stores and production stores are HiveSpace-owned data. Store names are not stable identity, and automatic name matching could assign products to the wrong seller or overwrite trusted seed data. CatalogService owns imported seller records and ownership links, while UserService remains the owner of store eligibility and store profile state.

Alternatives considered:

| Option | Why rejected |
| --- | --- |
| Auto-match by normalized store name | Too risky because names are ambiguous and seed data may only be similar, not the same seller. |
| Always create a new store when names conflict | Avoids bad merges but creates duplicate sellers when the existing store is the intended owner. |
| Let UserService own imported seller resolution | UserService owns stores, but CatalogService owns import bundle readiness and imported seller lifecycle. |

## Decision: Use CatalogService-owned async import jobs, not a saga

Catalog import operations that can exceed normal API request timeouts run as CatalogService-owned asynchronous jobs. The browser-facing APIs return `202 Accepted` with a durable job ID. CatalogService persists job state, processes jobs in background consumers hosted inside the same CatalogService deployment, and exposes job status endpoints for operators. No Azure Function, separate worker project, central executor, or MassTransit saga is planned for v1.

The feature reuses `IdentityUserReadyIntegrationEvent`, `StoreCreatedIntegrationEvent`, `StoreUpdatedIntegrationEvent`, `ProductCreatedIntegrationEvent`, `ProductSkuUpdatedIntegrationEvent`, and `MediaAssetProcessedIntegrationEvent` unchanged for domain facts. CatalogService additionally publishes generic background job lifecycle events so future cross-service monitoring can observe queued, started, progressed, completed, and failed import work without owning or executing the work.

Rationale: The workflow is operator initiated, idempotent, and retryable at each step. The long-running problem is API request duration, not cross-service compensation. Existing service events already propagate profile/store/product/media facts. A saga would add durable orchestration state without a required asynchronous compensation path, while a central job executor would violate service ownership by moving catalog domain execution outside CatalogService.

Alternatives considered:

| Option | Why rejected |
| --- | --- |
| New import saga in CatalogService | No cross-service compensation state machine is required; failures remain visible on CatalogService job and bundle state. |
| Azure Function or separate CatalogService worker project | Adds a second deployment/runtime for v1 without changing ownership; same-host background consumers are simpler and keep import domain execution with CatalogService. |
| Central monitoring service as executor | A monitor may observe lifecycle events, but executing CatalogService import work centrally would transfer domain ownership away from the service that owns catalog data. |
| New seller-provisioned integration event | Existing identity/user/store events already communicate account/profile/store readiness. |
| Direct broker commands from Python | Would bypass backend authorization and operational API audit patterns. |

## Decision: Import bundles use a versioned JSON schema

Python category output and CatalogService category provisioning requests will use `schemaVersion`, `source`, `crawl`, and `categories` sections documented in [contracts/category-provisioning-schema.md](contracts/category-provisioning-schema.md). Later product bundle submissions will use `schemaVersion`, `source`, `crawl`, `sellers`, `products`, and `validationHints` sections documented in [contracts/import-bundle-schema.md](contracts/import-bundle-schema.md).

Rationale: Versioning allows Tiki response changes and backend model changes to evolve without silently breaking imports. The bundle is also reviewable, testable, and easy to archive.

Alternatives considered:

| Option | Why rejected |
| --- | --- |
| CSV files | Too weak for nested variants, SKUs, images, attributes, seller metadata, and validation issues. |
| Backend-specific DTO dump without versioning | Brittle across model changes and hard to validate independently. |
| Raw Tiki JSON storage only | Preserves source data but does not reflect HiveSpace catalog concepts. |

## Decision: Imported images start as external references

Bundles preserve external Tiki image URLs and image metadata. During or after import, media copying must use MediaService's existing upload/confirm/processing ownership; CatalogService stores only media references and validation issue state.

Rationale: MediaService owns binary storage and processing. CatalogService owns product association decisions, not downloaded bytes.

Alternatives considered:

| Option | Why rejected |
| --- | --- |
| Store Tiki URLs directly as product images forever | Leaves buyer-facing catalog dependent on external source availability. |
| Have CatalogService download and process images | Violates MediaService ownership. |
| Block all products until every image is copied | Too strict for first import when image issues can be warnings or targeted blockers. |

## Decision: VND is required and validated as enabled platform currency

Imported SKU prices must be parsed as VND smallest-unit money and validated against CatalogService's currency-policy projection before product import.

Rationale: The spec clarifies that Tiki prices are expected to be VND but must not be silently assumed. CatalogService already validates product/SKU money writes using UserService-owned currency policy projection.

Alternatives considered:

| Option | Why rejected |
| --- | --- |
| Assume missing currency means VND | Contradicts the clarified requirement and hides data quality defects. |
| Accept any Tiki currency | Conflicts with enabled platform currency policy. |
| Convert currencies in the import flow | Out of scope and introduces pricing decisions not requested. |
