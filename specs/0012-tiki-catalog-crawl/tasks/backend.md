# Backend And Source Tasks: Tiki Catalog Crawl

## Python Crawler Repository: `../hivespace.crawler`

### Create

- [x] B001 [US1] Create `crawler project skeleton`
  - File: `../hivespace.crawler/pyproject.toml`, `../hivespace.crawler/src/hivespace_crawler/__init__.py`, `../hivespace.crawler/src/hivespace_crawler/tiki/__init__.py`, `../hivespace.crawler/tests/`
  - Define package name `hivespace-crawler`, Python entry package `hivespace_crawler`, stdlib `unittest` discovery, and initial CLI/bundle modules without requiring third-party packages.
  - Keep production crawler work out of `../hivespace.crawl-data`; use that repo only as reference material.
  - Acceptance: `python -m hivespace_crawler --help` starts after install and no file is created in `../hivespace.config`.

- [x] B002 [US1] Create `crawler contract tests for CLI and bundle schema`
  - File: `../hivespace.crawler/tests/test_tiki_cli_contract.py`, `../hivespace.crawler/tests/test_import_bundle_schema.py`
  - Test: `should_write_schema_versioned_bundle_for_valid_category_source`
  - Test: `should_reject_unsupported_source_type_with_exit_code_1`
  - Test: `should_validate_required_seller_product_sku_price_fields`
  - Use `specs/0012-tiki-catalog-crawl/contracts/import-bundle-schema.md` and `contracts/crawler-cli.md` as source contracts.
  - Acceptance: tests compile and fail before B003-B008 implement the CLI, schema, and normalizer.

- [x] B003 [US1] Create `crawler CLI entrypoint`
  - File: `../hivespace.crawler/src/hivespace_crawler/__main__.py`, `../hivespace.crawler/src/hivespace_crawler/tiki/cli.py`
  - Implement `python -m hivespace_crawler tiki crawl --source-type <category|search|product_list> --source-value <value> --out <path>`.
  - Add options `--checkpoint-dir`, `--limit`, `--request-timeout-seconds`, `--rate-limit-per-minute`, and `--user-agent`.
  - Return exit codes 0, 1, 2, 3, and 4 exactly as `contracts/crawler-cli.md` defines.
  - Do not call HiveSpace backend APIs from the crawler; it only writes the import bundle JSON.
  - Acceptance: B002 CLI tests pass and console output includes source selection, product count, seller count, category count, warning count, and output path.

- [x] B004 [US1] Create `Tiki client fixture tests`
  - File: `../hivespace.crawler/tests/fixtures/tiki/*.json`, `../hivespace.crawler/tests/test_tiki_client.py`
  - Test: `should_collect_products_from_category_response`
  - Test: `should_record_missing_optional_fields_as_validation_hints`
  - Test: `should_resume_from_checkpoint_after_partial_collection`
  - Use recorded fixture JSON only; do not hit live Tiki endpoints in automated tests.
  - Acceptance: tests compile and fail before B005 implements the source client.

- [x] B005 [US1] Create `Tiki source client and checkpoint support`
  - File: `../hivespace.crawler/src/hivespace_crawler/tiki/client.py`, `../hivespace.crawler/src/hivespace_crawler/tiki/checkpoint.py`
  - Support `category`, `search`, and `product_list` source selectors, request timeout, configurable user agent, per-host rate limit, and resumable checkpoint writes.
  - Preserve source URLs, external product IDs, external seller IDs, category IDs, raw price values, stock values, image URLs, attributes, variants, and SKUs where available.
  - Missing optional Tiki fields must become validation hints instead of process failures.
  - Acceptance: B004 tests pass without network access.

- [x] B006 [US1] Create `import bundle normalization tests`
  - File: `../hivespace.crawler/tests/test_import_bundle_normalizer.py`
  - Test: `should_normalize_product_seller_category_sku_image_attribute_records`
  - Test: `should_require_vnd_positive_integer_prices_for_ready_sku_candidates`
  - Test: `should_group_duplicate_source_products_by_external_product_id`
  - Test: `should_keep_partial_bundle_reviewable_when_optional_fields_are_missing`
  - Acceptance: tests compile and fail before B007 implements normalization.

- [x] B007 [US1] Create `import bundle normalizer and writer`
  - File: `../hivespace.crawler/src/hivespace_crawler/tiki/normalizer.py`, `../hivespace.crawler/src/hivespace_crawler/bundle.py`
  - Emit top-level `schemaVersion`, `source`, `crawl`, `sellers`, `products`, and `validationHints` for product bundles.
  - Shape sellers, products, SKUs, attributes, images, and validation hints exactly as `contracts/import-bundle-schema.md` describes.
  - Compute stable `sourceFingerprint` from source selector plus normalized external product IDs; preserve `checkpointId` when provided.
  - Do not silently coerce missing or unsupported currency to VND; emit blocking validation hints.
  - Acceptance: B006 tests pass and output JSON validates against the local schema checks.

- [x] B008 [US1] Create `crawler README and sample bundle`
  - File: `../hivespace.crawler/README.md`, `../hivespace.crawler/samples/tiki-category-1846.bundle.json`
  - Document setup, CLI examples, exit codes, checkpoint behavior, fixture-only tests, and the rule that production crawler code lives in `../hivespace.crawler`.
  - Include one small sample bundle with at least one seller, category, product, SKU, image, attribute, and validation hint.
  - Acceptance: README commands match `contracts/crawler-cli.md` and sample bundle validates with B002 schema tests.

- [x] B040 [US1] Create `SellerCenter all-category discovery and cache`
  - File: `../hivespace.crawler/src/hivespace_crawler/tiki/client.py`, `../hivespace.crawler/tests/test_tiki_client.py`, `../hivespace.crawler/tests/fixtures/tiki/seller-categories-parent-*.json`
  - Add recursive SellerCenter category discovery from `parent_id=2`, read bearer token from `TIKI_SELLER_BEARER_TOKEN`, preserve category ID, name, parent ID, and product set ID, and include nested child categories when returned in the response payload.
  - Save discovered categories to local crawler state so later runs can reuse existing category data unless explicitly refreshed.
  - Do not hardcode SellerCenter bearer tokens or add secrets to repo files.
  - Acceptance: fixture tests cover recursive and nested child category discovery; live validation on 2026-07-29 discovered 6,101 categories from SellerCenter.

- [x] B041 [US1] Create `all-category crawl command with per-category output and resume`
  - File: `../hivespace.crawler/src/hivespace_crawler/tiki/cli.py`, `../hivespace.crawler/src/hivespace_crawler/tiki/state.py`, `../hivespace.crawler/tests/test_tiki_cli_contract.py`
  - Add `python -m hivespace_crawler tiki crawl-all-categories --state-dir <dir> --out-dir <dir> --limit-per-category <n>`.
  - Write `categories.json`, `manifest.json`, and `bundles/category-{categoryId}.bundle.json`; skip completed categories when rerun and retry failed/running categories from the start of the category.
  - Preserve existing single-source `tiki crawl` behavior unchanged.
  - Acceptance: CLI tests cover all-category bundle writing and category-level resume.

- [x] B042 [US1] Create `Tiki HTML challenge detection`
  - File: `../hivespace.crawler/src/hivespace_crawler/tiki/client.py`, `../hivespace.crawler/src/hivespace_crawler/tiki/cli.py`, `../hivespace.crawler/tests/test_tiki_cli_contract.py`
  - Detect product API responses that return HTML instead of JSON and stop the all-category run after recording the current category as failed.
  - Do not continue through the remaining category list and mark every category failed when Tiki starts serving challenge content.
  - Acceptance: tests cover stop-on-challenge behavior; live validation on 2026-07-29 wrote 6 category bundles before Tiki returned HTML challenge content.

## CatalogService: Import Domain And Validation

### Create

- [ ] B009 [US3] Create `CatalogImportBundle domain tests - AC3.1 validation issue summary`
  - File: `../hivespace.microservice/tests/HiveSpace.CatalogService.Tests/Domain/CatalogImports/CatalogImportBundleTests.cs`
  - Test: `Create_WithRequiredSourceMetadata_InitializesSubmittedBundle`
  - Test: `ApplyValidationResults_WithBlockingIssues_SetsNeedsAttentionSummary`
  - Test: `MarkReadyToImport_WithoutBlockingIssues_SetsReadyStatus`
  - Use pure domain tests; no EF, no live database.
  - Acceptance: tests compile and fail before B012 creates the aggregate.

- [ ] B010 [US3] Create `imported seller/category/product domain tests`
  - File: `../hivespace.microservice/tests/HiveSpace.CatalogService.Tests/Domain/CatalogImports/ImportedCatalogRecordsTests.cs`
  - Test: `ImportedSeller_WithoutDisplayName_ThrowsDomainException`
  - Test: `ExternalCategoryLink_MissingProvisionedCategory_BlocksProducts`
  - Test: `ImportedSku_WithNegativeStock_RecordsBlockingIssue`
  - Test: `ImportedSku_WithMissingVndPrice_RecordsBlockingIssue`
  - Acceptance: tests compile and fail before B013 creates imported record entities.

- [ ] B011 [US3] Create `duplicate grouping domain tests`
  - File: `../hivespace.microservice/tests/HiveSpace.CatalogService.Tests/Domain/CatalogImports/ImportDuplicateGroupTests.cs`
  - Test: `Create_WithSameExternalProductId_GroupsDuplicateMembers`
  - Test: `Resolve_WithRepresentativeProduct_SetsResolvedStatus`
  - Test: `ReadyValidation_WithUnresolvedDuplicate_BlocksMembers`
  - Acceptance: tests compile and fail before B014 creates duplicate grouping behavior.

- [ ] B012 [US1] [US3] Create `CatalogImportBundle aggregate`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Domain/CatalogImports/CatalogImportBundle.cs`
  - Include fields from `data-model.md`: ID, schema/source/crawl/submission metadata, status, summary counts, submitted actor, and child collections.
  - Add static `Create(...)`, validation summary update methods, seller provisioning state methods, import result methods, private setters, and protected EF constructor.
  - Link bundle creation/update to the `CatalogImportJob` that accepted or processed the submission.
  - Do not publish lifecycle integration events from the aggregate; job lifecycle events are emitted by application/background job orchestration through outbox-backed publisher abstractions.
  - Acceptance: B009 tests pass and aggregate follows existing CatalogService entity/value-object patterns.

- [ ] B045 [US1] [US2] [US3] Create `CatalogImportJob domain tests and aggregate`
  - File: `../hivespace.microservice/tests/HiveSpace.CatalogService.Tests/Domain/CatalogImports/CatalogImportJobTests.cs`, `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Domain/CatalogImports/CatalogImportJob.cs`
  - Test: `Create_WithOperationAndRequester_StartsPending`
  - Test: `Start_FromPending_SetsRunningAndStartedAt`
  - Test: `Complete_WithCounts_SetsCompletedResultSummary`
  - Test: `Fail_WithErrorSummary_SetsFailed`
  - Include fields, statuses, progress counts, timestamps, requester, source fingerprint, optional source file name, bundle ID, operation type, result summary, error summary, and correlation ID from `data-model.md`.
  - Do not put import execution logic in the job entity.
  - Acceptance: tests pass and invalid state transitions are rejected.

- [ ] B013 [US1] [US2] [US3] Create `imported catalog record entities`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Domain/CatalogImports/{ImportedSeller.cs,SellerOwnershipLink.cs,ExternalCategoryLink.cs,ImportedProduct.cs,ImportedSku.cs,ImportedAttribute.cs,ImportedImageReference.cs,ImportValidationIssue.cs}`
  - Include statuses and fields from `data-model.md`, using ULID/GUID conventions already used by CatalogService.
  - Implement guards for required external IDs, display names, product titles, SKU price shape, category provisioning, seller ownership, media reference status, and non-negative stock.
  - Keep external seller identity separate from HiveSpace user/store ownership fields; include suggested existing-store candidate fields and manual approval audit fields from `data-model.md`.
  - Acceptance: B010 tests pass and no entity contains credentials, raw passwords, profile fields, or media binary data.

- [ ] B014 [US3] Create `duplicate group domain entity`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Domain/CatalogImports/ImportDuplicateGroup.cs`
  - Include `BundleId`, `DuplicateKey`, external product IDs, nullable representative imported product ID, and resolution status.
  - Require duplicate groups to be resolved before any member can become ready.
  - Acceptance: B011 tests pass and duplicate source products cannot be counted as unrelated ready products.

- [ ] B015 [US1] [US2] [US3] Create `catalog import repository interfaces`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Domain/CatalogImports/ICatalogImportBundleRepository.cs`
  - Include bundle methods for add, get by ID, get by `SourceFingerprint`, list paginated summaries, load paginated detail sections, and save changes.
  - Include job methods for add, get by ID, list paginated all-operation history with filters, find active/completed idempotent job by operation plus source fingerprint or bundle/selection identity, list retry candidates, and save progress.
  - Do not expose IQueryable or cross-service database access.
  - Acceptance: Domain project compiles and application handlers can depend only on the interface.

- [ ] B016 [US3] Create `catalog import validation domain service`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Domain/CatalogImports/CatalogImportValidator.cs`
  - Validate seller ownership, provisioned category links, required product fields, mandatory attributes, SKU presence, positive VND money, enabled VND currency projection, non-negative stock, media reference usability, and duplicate resolution.
  - Return structured `ImportValidationIssue` records with entity type, source ID, field, severity, reason code, message, and optional metadata for actionable repair context such as missing external category IDs.
  - Acceptance: B009-B011 tests cover blocking and warning states through domain behavior.

## CatalogService: Application And API

### Create

- [ ] B017 [US1] Create `submit/list/detail application tests`
  - File: `../hivespace.microservice/tests/HiveSpace.CatalogService.Tests/Application/CatalogImports/{SubmitCatalogImportBundleCommandHandlerTests.cs,ListCatalogImportBundlesQueryHandlerTests.cs,ListCatalogImportJobsQueryHandlerTests.cs,GetCatalogImportBundleDetailQueryHandlerTests.cs}`
  - Test: `Handle_WithNewSourceFingerprint_ReturnsAcceptedJobSubmission`
  - Test: `Handle_WithExistingActiveSourceFingerprint_ReturnsExistingJob`
  - Test: `Worker_WithAcceptedBundleJob_PersistsSubmittedBundle`
  - Test: `Handle_WithSourceFileName_ReturnsDisplayFileNameWithoutChangingIdempotency`
  - Test: `Handle_WithRecentBundles_ReturnsPaginatedSummaryCounts`
  - Test: `Handle_WithImportJobs_ReturnsPaginatedAllOperationHistory`
  - Test: `Handle_WithBundleId_ReturnsPaginatedSellersProductsIssuesAndDuplicateGroups`
  - Use `IClassFixture<CatalogServiceFixture>` for persistence-backed behavior.
  - Acceptance: tests compile and fail before B018-B020 implement handlers.

- [ ] B018 [US1] Create `submit bundle command and DTO mapping`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Application/CatalogImports/Commands/SubmitCatalogImportBundle/{SubmitCatalogImportBundleCommand.cs,SubmitCatalogImportBundleCommandHandler.cs,SubmitCatalogImportBundleValidator.cs}`
  - Accept the versioned JSON bundle shape from `contracts/import-bundle-schema.md` plus the operator-visible original uploaded file name from `X-Source-File-Name` when provided.
  - Persist or reuse a `CatalogImportJob` and return `202 Accepted` job submission data, including `sourceFileName`, when bundle persistence or initial validation can exceed normal request timeouts.
  - Background execution persists sellers, products, SKUs, attributes, image references, source metadata, validation hints, unmapped placeholders for product category IDs without provisioned category links, and initial summary counts. Product `externalCategoryIds` must resolve to previously provisioned category links during validation.
  - Re-submitting the same `sourceFingerprint` returns the existing active/completed job or bundle reference unless a future supersede option is added; do not use file name for idempotency.
  - Acceptance: B017 submit tests pass, invalid required top-level fields return the existing service validation error convention, and the endpoint does not block on long-running persistence/validation.

- [ ] B019 [US1] Create `import bundle query handlers`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Application/CatalogImports/Queries/{ListCatalogImportBundles,ListCatalogImportBundlesQuery.cs,ListCatalogImportJobs,ListCatalogImportJobsQuery.cs,GetCatalogImportBundleDetail,GetCatalogImportBundleDetailQuery.cs}`
  - Return paginated bundle summaries, bundle metadata/detail summary, and dedicated paginated section queries for `sellers`, `sellerOwnershipCandidates`, `categoryLinks`, `products`, `duplicateGroups`, and `validationIssues`.
  - Return paginated all-operation job history with filters for status, operation type, source system, bundle ID, and requested timestamp range.
  - Keep response DTOs in `Application/CatalogImports/Dtos/` and do not expose domain entities directly.
  - Acceptance: B017 list/detail tests pass and summary counts match persisted bundle data.

- [ ] B020 [US1] Create `catalog import API endpoints for submit/list/detail`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Api/Endpoints/CatalogImportEndpoints.cs`
  - Map `POST /api/v1/admins/catalog-imports/bundles`, `GET /api/v1/admins/catalog-imports/bundles`, paginated `GET /api/v1/admins/catalog-imports/jobs`, `GET /api/v1/admins/catalog-imports/bundles/{bundleId}`, paginated bundle section endpoints for category links, sellers, products, duplicate groups, and validation issues, and `POST /api/v1/admins/catalog-imports/bundles/{bundleId}/category-links/{externalCategoryId}/mapping` for operator category-link repair.
  - For submit, return `202 Accepted` with `{ jobId, status, operationType, sourceFingerprint, sourceFileName, bundleId }`.
  - Apply backend pagination conventions to bundle list, job history, and bundle detail collection sections.
  - Use `RequireAdmin`, Mediator, current-user context, existing result/exception conventions, and OpenAPI tags consistent with CatalogService.
  - Do not add business validation to endpoint handlers.
  - Acceptance: CatalogService builds and routes match `contracts/catalog-import-api.md`.

- [ ] B021 [US2] Create `seller provisioning application tests`
  - File: `../hivespace.microservice/tests/HiveSpace.CatalogService.Tests/Application/CatalogImports/ProvisionImportedSellersCommandHandlerTests.cs`
  - Test: `Handle_WithBundleId_ReturnsAcceptedProvisionSellersJob`
  - Test: `Worker_WithUnmatchedSeller_CallsIdentityThenUserProvisioningPorts`
  - Test: `Worker_WithExistingOwnershipLink_SkipsAlreadyProvisionedSeller`
  - Test: `Worker_WithIdentityConflict_MarksSellerConflictAndBlocksProducts`
  - Test: `Worker_WithStoreConflict_MarksSellerConflictAndBlocksProducts`
  - Test: `Worker_WithSimilarExistingStoreName_MarksSellerConflictAndBlocksProducts`
  - Use substitutes for IdentityService and UserService client ports; no cross-service database access.
  - Acceptance: tests compile and fail before B022 implements provisioning orchestration.

- [ ] B022 [US2] Create `seller provisioning command and ports`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Application/CatalogImports/Commands/ProvisionImportedSellers/{ProvisionImportedSellersCommand.cs,ProvisionImportedSellersCommandHandler.cs}`, `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Application/CatalogImports/Ports/{IImportedSellerAccountClient.cs,IImportedSellerStoreClient.cs}`
  - Queue or execute through a `CatalogImportJob`; for each imported seller, call IdentityService create-or-match account, then UserService create-or-match store, then record `SellerOwnershipLink`.
  - Make operation idempotent per `sourceSystem` plus `externalSellerId`.
  - Do not create identity accounts, profiles, stores, roles, or claims inside CatalogService.
  - Acceptance: B021 tests pass and failed/conflict counts are visible on bundle detail.

- [ ] B043 [US2] Create `seller ownership approval application tests`
  - File: `../hivespace.microservice/tests/HiveSpace.CatalogService.Tests/Application/CatalogImports/ApproveImportedSellerOwnershipCommandHandlerTests.cs`
  - Test: `Handle_WithConflictedSellerAndEligibleStore_CreatesActiveOwnershipLink`
  - Test: `Handle_WithIneligibleTargetStore_ReturnsConflictAndKeepsSellerBlocked`
  - Test: `Handle_WithApprovedLink_RevalidationAllowsAffectedProductsWhenOtherChecksPass`
  - Test: `Handle_WithApproval_DoesNotUpdateExistingStoreProfileData`
  - Use CatalogService repository fixtures plus substituted UserService/store eligibility port; no direct UserService database access.
  - Acceptance: tests compile and fail before B044 implements manual ownership approval.

- [ ] B044 [US2] Create `seller ownership approval command`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Application/CatalogImports/Commands/ApproveImportedSellerOwnership/{ApproveImportedSellerOwnershipCommand.cs,ApproveImportedSellerOwnershipCommandHandler.cs,ApproveImportedSellerOwnershipValidator.cs}`
  - Accept `bundleId`, `importedSellerId`, `targetUserId`, `targetStoreId`, approving admin actor, and required `approvalReason`.
  - Verify the imported seller belongs to the bundle, currently needs review/conflict resolution, and has no active ownership link for the same `sourceSystem` plus `externalSellerId`.
  - Verify the target HiveSpace account/store is eligible for seller ownership through a UserService-owned/store-reference eligibility port; do not read UserService data directly.
  - Record an active `SellerOwnershipLink` with approving actor, approval timestamp, source seller identity, target account/store, and approval reason; update the imported seller to matched.
  - Do not update existing store name, profile, metadata, owner, lifecycle state, or existing products from Tiki seller metadata.
  - Acceptance: B043 tests pass and affected products remain blocked until validation is rerun.

- [ ] B023 [US1] [US3] Create `validation and category provisioning application tests`
  - File: `../hivespace.microservice/tests/HiveSpace.CatalogService.Tests/Application/CatalogImports/{ValidateCatalogImportBundleCommandHandlerTests.cs,ProvisionImportedCategoriesCommandHandlerTests.cs}`
  - Test: `ProvisionCategories_WithSourceFingerprint_ReturnsAcceptedJobSubmission`
  - Test: `ValidateBundle_WithBundleId_ReturnsAcceptedJobSubmission`
  - Test: `ValidationWorker_WithUnprovisionedCategory_CreatesBlockingIssue`
  - Test: `ValidationWorker_WithUnprovisionedCategory_IncludesMissingExternalCategoryIdsMetadata`
  - Test: `SubmitBundle_WithMissingCategoryLink_AddsUnmappedCategoryLinkPlaceholder`
  - Test: `ValidationWorker_WithInvalidMoney_CreatesSkuBlockingIssue`
  - Test: `ValidationWorker_WithProvisionedCategory_UpdatesAffectedProductReadiness`
  - Test: `MapImportedCategory_WithActiveCategory_MapsCategoryLinkAndRequiresRevalidation`
  - Test: `ValidationWorker_WithDuplicateGroupUnresolved_BlocksDuplicateMembers`
  - Acceptance: tests compile and fail before B024-B025 implement handlers.

- [ ] B024 [US3] Create `validate bundle command`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Application/CatalogImports/Commands/ValidateCatalogImportBundle/{ValidateCatalogImportBundleCommand.cs,ValidateCatalogImportBundleCommandHandler.cs}`
  - Queue or execute through a `CatalogImportJob`; use `CatalogImportValidator` and local CatalogService category, attribute, store-ref, media-reference, and currency-policy projections.
  - Distinguish ready records, blocked records, warnings, and duplicate records in summary counts.
  - Do not call UserService synchronously for currency policy; use CatalogService local projection.
  - Acceptance: B023 validation tests pass and validation remains repeatable after category provisioning or seller provisioning changes.

- [ ] B025 [US1] [US3] Create `category provisioning command`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Application/CatalogImports/Commands/ProvisionImportedCategories/{ProvisionImportedCategoriesCommand.cs,ProvisionImportedCategoriesCommandHandler.cs,ProvisionImportedCategoriesValidator.cs}`
  - Accept the category-only JSON shape from `contracts/category-provisioning-schema.md` plus the operator-visible original uploaded file name from `X-Source-File-Name` when provided.
  - Return a `CatalogImportJob` submission immediately for long-running category provisioning, including `sourceFileName` when available.
  - Background execution creates or matches CatalogService categories and persists external category links keyed by `sourceSystem` plus `externalCategoryId`.
  - Process parents before children and return created, matched, failed, and conflict counts.
  - Do not require or create a product import bundle as part of this flow.
  - Acceptance: B023 category provisioning tests pass and unprovisioned category issues clear only after revalidation.

- [ ] B026 [US3] Create `import ready products application tests`
  - File: `../hivespace.microservice/tests/HiveSpace.CatalogService.Tests/Application/CatalogImports/ImportReadyProductsCommandHandlerTests.cs`
  - Test: `Handle_WithSelectedReadyProducts_ReturnsAcceptedImportJob`
  - Test: `Handle_WithoutProductIds_ReturnsAcceptedImportAllReadyJob`
  - Test: `Worker_WithReadyProducts_CreatesSellerOwnedProductsUsingSelectedPublicationState`
  - Test: `Worker_WithBlockedProducts_SkipsAndReportsBlockedResults`
  - Test: `Worker_WithDuplicateRepresentativeOnly_ImportsOneProduct`
  - Test: `Worker_WithImportAllReadyRequest_UsesSubmitTimeSnapshot`
  - Test: `Worker_WithDraftProducts_DoesNotExposeStorefrontStatus`
  - Test: `Worker_WithUnpublishProducts_DoesNotExposeStorefrontStatus`
  - Test: `Worker_WithAvailableProducts_ExposesStorefrontStatusSubjectToExistingFilters`
  - Acceptance: tests compile and fail before B038 implements product import.

## IdentityService: Imported Seller Account Provisioning

### Create

- [ ] B027 [US2] Create `imported seller account provisioning tests`
  - File: `../hivespace.microservice/tests/HiveSpace.IdentityService.Tests/Application/Admin/ProvisionImportedSellerAccountCommandHandlerTests.cs`
  - Test: `Handle_WithNewExternalSeller_CreatesSellerAccountWithoutSessionTokens`
  - Test: `Handle_WithExistingExternalSeller_ReturnsMatchedUserId`
  - Test: `Handle_WithConflictingSellerIdentity_ReturnsConflictStatus`
  - Test: `Handle_WithCreatedUsableAccount_PublishesIdentityUserReadyEvent`
  - Acceptance: tests compile and fail before B028-B029 implement the command and endpoint.

- [ ] B028 [US2] Create `imported seller account provisioning command`
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Admin/ImportedSellerAccounts/{ProvisionImportedSellerAccountCommand.cs,ProvisionImportedSellerAccountCommandHandler.cs,ProvisionImportedSellerAccountValidator.cs}`
  - Accept `sourceSystem`, `externalSellerId`, `displayName`, and optional `sourceUrl`.
  - Create or match an identity-owned seller account idempotently without returning passwords, access tokens, refresh tokens, browser cookies, or CSRF tokens.
  - Preserve existing IdentityService ownership of credentials, roles, claims, account status, and readiness event publication.
  - Acceptance: B027 handler tests pass and conflicts return a reason without duplicate account creation.

- [ ] B029 [US2] Create `imported seller account endpoint`
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Endpoints/AdminIdentityEndpoints.cs`
  - Map `POST /api/v1/admins/imported-seller-accounts` with `RequireCatalogImportProvisioning`.
  - Return `userId`, `status`, and nullable `conflictReason`.
  - Do not expose this route under `/api/v1/accounts/**` or direct IdentityServer authority endpoints.
  - Acceptance: IdentityService builds and endpoint contract matches `contracts/catalog-import-api.md`.

## UserService: Imported Seller Store Provisioning

### Create

- [ ] B030 [US2] Create `imported seller store provisioning tests`
  - File: `../hivespace.microservice/tests/HiveSpace.UserService.Tests/Application/Stores/ProvisionImportedSellerStoreCommandHandlerTests.cs`
  - Test: `Handle_WithNewExternalSellerStore_CreatesStoreForUser`
  - Test: `Handle_WithExistingOwnership_ReturnsMatchedStoreId`
  - Test: `Handle_WithStoreUniquenessConflict_ReturnsConflictStatus`
  - Test: `Handle_WithCreatedStore_PublishesStoreCreatedIntegrationEvent`
  - Acceptance: tests compile and fail before B031-B032 implement the command and endpoint.

- [ ] B031 [US2] Create `imported seller store provisioning command`
  - File: `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Application/Stores/Commands/ProvisionImportedSellerStore/{ProvisionImportedSellerStoreCommand.cs,ProvisionImportedSellerStoreCommandHandler.cs,ProvisionImportedSellerStoreValidator.cs}`
  - Accept `sourceSystem`, `externalSellerId`, `userId`, `storeName`, and optional `sourceUrl`.
  - Create or match a UserService-owned store idempotently while enforcing existing store uniqueness, lifecycle, and event publishing rules.
  - Do not grant identity roles/claims directly; IdentityService remains the consumer of `StoreCreatedIntegrationEvent`.
  - Acceptance: B030 tests pass and conflicts are observable through response status and reason.

- [ ] B032 [US2] Create `imported seller store endpoint`
  - File: `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Endpoints/AdminStoreEndpoints.cs`
  - Map `POST /api/v1/admins/imported-seller-stores` with `RequireCatalogImportProvisioning`.
  - Return `storeId`, `status`, and nullable `conflictReason`.
  - Do not change public seller self-registration route semantics.
  - Acceptance: UserService builds and endpoint contract matches `contracts/catalog-import-api.md`.

## Backend Infrastructure, Gateway, And Migrations

### Create

- [ ] B033 [US1] [US2] [US3] Create `CatalogService EF configurations and repository`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Infrastructure/Persistence/Configurations/CatalogImports/*.cs`, `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Infrastructure/Repositories/CatalogImportBundleRepository.cs`
  - Map `catalog_import_jobs`, `catalog_import_bundles`, imported sellers, ownership links, external category links, imported products, imported SKUs, imported attributes, image references, validation issues, and duplicate groups using snake_case names.
  - Configure uniqueness/idempotency indexes for job operation plus source fingerprint or bundle/selection identity, bundle `SourceFingerprint`, and one active seller ownership link per `SourceSystem` plus `ExternalSellerId`; persist suggested store candidates and manual approval audit fields.
  - Do not put validation business rules in EF configurations.
  - Acceptance: CatalogService build succeeds and repository satisfies B015 interface.

- [ ] B034 [US1] [US2] [US3] Create `CatalogService migration for catalog imports`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Infrastructure/Migrations/<timestamp>_AddCatalogImports.cs`
  - Add tables and indexes for the import bundle data model from `data-model.md`, including nullable source file name columns on jobs and bundles.
  - Include reversible `Down` operations for all new tables/indexes.
  - Do not alter existing product/store/category/currency tables except for foreign-key/index references required by imported catalog records.
  - Acceptance: migration scaffolds from CatalogService Infrastructure and applies against local CatalogService database.

- [ ] B035 [US1] [US2] [US3] Update `CatalogService dependency registration`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Infrastructure/DependencyInjection.cs`, `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Api/Program.cs`
  - Register catalog import repository, validator/domain service, async import job dispatch/worker services, lifecycle publisher, and Identity/User provisioning HTTP clients.
  - Register `CatalogImportEndpoints.MapCatalogImportEndpoints()` in the existing endpoint mapping style.
  - Keep service URLs in source-repo runtime configuration or appsettings, not `../hivespace.config` feature tasks.
  - Acceptance: CatalogService starts with missing external service settings surfaced through existing configuration validation.

- [ ] B036 [US1] Update `ApiGateway routes for catalog import endpoints`
  - File: `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/appsettings.json`, `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/appsettings.Development.json`
  - Route `/api/v1/admins/catalog-imports/{**catch-all}` to CatalogService.
  - Route `/api/v1/admins/imported-seller-accounts` to IdentityService.
  - Route `/api/v1/admins/imported-seller-stores` to UserService.
  - Do not add business logic, transforms that change request bodies, or config repo edits.
  - Acceptance: gateway route table covers all paths in `contracts/catalog-import-api.md`.

- [ ] B037 [US2] Create `CatalogService provisioning HTTP client implementations`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Infrastructure/CatalogImports/{ImportedSellerAccountClient.cs,ImportedSellerStoreClient.cs}`
  - Call IdentityService and UserService provisioning APIs with system/admin authorization mechanism already used for internal service calls.
  - Map `Created`, `Matched`, `Conflict`, and `Failed` responses into application port results.
  - Preserve correlation IDs on outbound calls and keep failures visible to the calling handler.
  - Acceptance: CatalogService build succeeds and B021 tests can substitute the ports without infrastructure dependencies.

- [ ] B038 [US3] Create `import ready products command implementation`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Application/CatalogImports/Commands/ImportReadyProducts/{ImportReadyProductsCommand.cs,ImportReadyProductsCommandHandler.cs,ImportReadyProductsValidator.cs}`
  - Queue or execute through a `CatalogImportJob`; create seller-owned `Product` aggregates from selected ready imported products or the bundle-wide eligible snapshot using existing product/SKU/category/attribute/image patterns.
  - Treat omitted or empty `productIds` as `import all eligible imported products in this bundle` and capture that eligible imported product ID set when the job is accepted.
  - Set imported products to the operator-selected `publicationState`, defaulting to `Draft`, support `Draft`, `Unpublish`, and `Available`, keep `Draft` and `Unpublish` non-public, and allow `Available` imports to surface through existing storefront reads subject to existing publication filters.
  - Publish existing product/SKU integration events through existing CatalogService publisher abstractions where product creation requires projection updates.
  - Do not import blocked products, unresolved duplicates, unprovisioned categories, missing seller ownership, invalid money, invalid stock, or unusable required media references.
  - Do not recalculate a bundle-wide import job against later bundle changes during retry; reuse the accepted snapshot.
  - Acceptance: B026 tests pass and import response reports imported, skipped, blocked, and failed counts.

- [ ] B039 [US1] [US2] [US3] Create `provision/validate/import CatalogService endpoints`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Api/Endpoints/CatalogImportEndpoints.cs`
  - Map `POST /api/v1/admins/catalog-imports/categories/provisioning`, `POST /api/v1/admins/catalog-imports/categories/attributes/provisioning`, `POST /api/v1/admins/catalog-imports/bundles/{bundleId}/validate`, `POST /api/v1/admins/catalog-imports/bundles/{bundleId}/seller-provisioning`, `POST /api/v1/admins/catalog-imports/bundles/{bundleId}/sellers/{importedSellerId}/ownership-link`, `POST /api/v1/admins/catalog-imports/bundles/{bundleId}/category-links/{externalCategoryId}/mapping`, and `POST /api/v1/admins/catalog-imports/bundles/{bundleId}/import`.
  - Return `202 Accepted` job submission responses for category provisioning, validation, seller provisioning, and ready-product import; category provisioning responses include source file name when submitted by upload.
  - Accept optional `productIds` on the import request; omitted or empty `productIds` means bundle-wide import of all eligible imported products.
  - Use `RequireAdmin`, request DTOs from the contract, Mediator, and existing error/result conventions.
  - Do not add seller account creation or store creation logic inside the endpoint layer; category creation/matching belongs in the category provisioning command.
  - Acceptance: CatalogService build succeeds and all CatalogService endpoints match `contracts/catalog-import-api.md`.

- [ ] B046 [US1] [US2] [US3] Create `CatalogService import job history, status, and retry endpoints`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Application/CatalogImports/Queries/GetCatalogImportJob/GetCatalogImportJobQuery.cs`, `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Application/CatalogImports/Commands/RetryCatalogImportJob/RetryCatalogImportJobCommand.cs`, `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Api/Endpoints/CatalogImportEndpoints.cs`
  - Map paginated `GET /api/v1/admins/catalog-imports/jobs`, `GET /api/v1/admins/catalog-imports/jobs/{jobId}`, and optional `POST /api/v1/admins/catalog-imports/jobs/{jobId}/retry`.
  - History returns all import operations, including upload submissions, validation, seller provisioning, ready-product import, and retries, sorted newest first by default.
  - Return status, operation type, source fingerprint, source file name where available, bundle ID, requester, timestamps, progress counts, result summary, and error summary.
  - Do not expose internal stack traces, worker implementation details, or cross-service credentials in job responses.
  - Acceptance: operators can inspect progress/final result for every async catalog import operation and retry respects idempotency rules.

- [ ] B047 [US1] [US2] [US3] Create `CatalogService same-host import job workers`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Application/CatalogImports/Jobs/*`, `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Infrastructure/CatalogImports/*`
  - Process `CatalogImportJob` records through CatalogService background consumers hosted in the same CatalogService deployment.
  - Implement handlers for `ProvisionCategories`, `ProvisionCategoryAttributes`, `SubmitBundle`, `ValidateBundle`, `ProvisionSellers`, and `ImportReadyProducts`.
  - Preserve correlation IDs, update progress counts, and keep failures visible through job state.
  - Do not create an Azure Function, separate worker project, central executor, or cross-service saga for v1 import jobs.
  - Acceptance: long-running category provisioning for 6,000+ categories returns a job ID before normal HTTP timeout and completes through CatalogService-owned worker execution.

- [ ] B048 [US1] [US2] [US3] Create `background job lifecycle event contracts and publishing`
  - File: `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/Events/BackgroundJobs/*.cs`, `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Application/CatalogImports/Jobs/*`
  - Add `BackgroundJobQueuedIntegrationEvent`, `BackgroundJobStartedIntegrationEvent`, `BackgroundJobProgressedIntegrationEvent`, `BackgroundJobCompletedIntegrationEvent`, and `BackgroundJobFailedIntegrationEvent`.
  - Publish lifecycle events through the transactional outbox when import jobs are queued, started, progressed, completed, or failed.
  - Make event payloads observer-safe: job ID, owning service, operation type, status, timestamps, correlation ID, progress counters, and operator-safe summaries only.
  - Do not publish domain entities, raw import payloads, credentials, stack traces, or anything that lets a consumer execute CatalogService import work.
  - Acceptance: event catalog names match `shared/event-catalog.md`, events use existing integration event conventions, and CatalogService job status remains the source of truth.

