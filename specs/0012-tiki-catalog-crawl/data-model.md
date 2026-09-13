# Data Model: Tiki Catalog Crawl

## CatalogImportJob

Durable CatalogService-owned execution record for long-running import operations.

Fields:

- `Id`: CatalogService import job ID.
- `OperationType`: `ProvisionCategories`, `ProvisionCategoryAttributes`, `SubmitBundle`, `ValidateBundle`, `ProvisionSellers`, or `ImportReadyProducts`.
- `Status`: `Pending`, `Running`, `Completed`, `Failed`.
- `SourceSystem`: `tiki`.
- `SourceFingerprint`: nullable stable hash for idempotent duplicate detection where the source payload provides one.
- `SourceFileName`: nullable original uploaded file name for operator traceability.
- `BundleId`: nullable import bundle ID for bundle-scoped jobs.
- `RequestedByUserId`: admin/operator user ID.
- `RequestedAt`, `StartedAt`, `CompletedAt`: timestamps.
- `TotalCount`, `ProcessedCount`, `CreatedCount`, `MatchedCount`, `SkippedCount`, `BlockedCount`, `WarningCount`, `DuplicateCount`, `FailedCount`, `ConflictCount`: nullable or zero progress/result counters by operation.
- `ResultSummaryJson`: nullable final operation-specific result summary.
- `ErrorSummary`: nullable failure reason/detail safe for operators.
- `CorrelationId`: nullable request/message correlation ID used by API, worker, and lifecycle events.

Relationships:

- May reference one `CatalogImportBundle`.
- May reference `ExternalCategoryLink`, `ImportedSeller`, `ImportedProduct`, or validation result rows through operation-specific result summaries rather than owning those records.

Validation:

- `OperationType`, `Status`, `SourceSystem`, `RequestedByUserId`, and `RequestedAt` are required.
- Jobs must transition only through `Pending -> Running -> Completed` or `Pending -> Running -> Failed`; retry creates or reuses a new `Pending` execution record according to operation idempotency.
- CatalogService job state is the source of truth for progress, final result details, and retry eligibility.
- Domain work remains inside CatalogService background consumers in the same service host for v1.
- A central monitoring service may observe lifecycle events, but it must not execute CatalogService-owned work or own result state.

Idempotency:

- Category provisioning is idempotent by `OperationType` plus `SourceSystem` plus `SourceFingerprint`.
- Category-attribute provisioning is idempotent by `OperationType` plus `SourceSystem` plus `SourceFingerprint`.
- Product bundle submission is idempotent by `OperationType` plus `SourceSystem` plus `SourceFingerprint`.
- Bundle validation and seller provisioning are idempotent by `OperationType` plus `BundleId`.
- Ready-product import is idempotent by `OperationType` plus `BundleId` plus the accepted import selection snapshot, whether that snapshot comes from explicit `productIds` or bundle-wide eligibility resolution at submit time.
- Re-submitting an equivalent pending or running operation returns the existing job reference.
- Retrying a completed category provisioning fingerprint must match existing category links and must not create duplicate categories or external links.

## CatalogImportBundle

Owns one submitted crawl output and its validation/import lifecycle.

Fields:

- `Id`: CatalogService import bundle ID.
- `SchemaVersion`: import bundle contract version.
- `SourceSystem`: `tiki`.
- `SourceType`: `category`, `search`, or `product_list`.
- `SourceValue`: source category ID/URL, keyword, or product list identifier.
- `SourceFingerprint`: stable hash for idempotent duplicate bundle detection.
- `SourceFileName`: nullable original uploaded file name for operator traceability.
- `CrawledAt`: source crawl timestamp.
- `SubmittedAt`: HiveSpace submission timestamp.
- `SubmittedByUserId`: admin/operator user ID.
- `SubmissionJobId`: nullable `CatalogImportJob` ID that created or updated the bundle.
- `Status`: `Submitted`, `Validated`, `NeedsAttention`, `ProvisioningSellers`, `ReadyToImport`, `PartiallyImported`, `Imported`, `Failed`.
- `TotalProducts`, `ReadyProducts`, `BlockedProducts`, `WarningCount`, `DuplicateCount`.

Relationships:

- Has many `ImportedSeller`, `ImportedProduct`, `ImportValidationIssue`, and `ImportDuplicateGroup`.

Validation:

- `SchemaVersion`, `SourceSystem`, `SourceType`, `SourceValue`, and `SourceFingerprint` are required.
- Duplicate `SourceFingerprint` submissions are idempotent unless explicitly superseded.
- `SourceFileName` is display metadata only and must not participate in idempotency or validation decisions.
- Bundle import is allowed only when validation has no blocking issues for selected products.

## ImportedSeller

Preserves external Tiki seller identity separately from HiveSpace ownership.

Fields:

- `Id`: imported seller ID.
- `BundleId`: parent bundle.
- `ExternalSellerId`: Tiki seller ID or stable seller key.
- `ExternalSellerSlug`: optional Tiki seller slug.
- `DisplayName`: Tiki display name.
- `SourceUrl`: optional seller URL.
- `LogoUrl`: Tiki seller logo URL normalized from the crawler's product-level `seller_logo_url` source value.
- `MetadataJson`: optional source-specific seller metadata.
- `ProvisioningStatus`: `Unmatched`, `Matched`, `CreateRequested`, `Created`, `Conflict`, `Failed`.
- `HiveSpaceUserId`: nullable Identity/User shared public user ID.
- `HiveSpaceStoreId`: nullable UserService store ID.
- `ConflictReason`: nullable reason code/detail.
- `SuggestedHiveSpaceStoreId`: nullable existing store candidate when a similar-name conflict is detected.
- `SuggestedHiveSpaceUserId`: nullable existing store owner candidate when available.

Relationships:

- Has zero or one `SellerOwnershipLink`.
- Referenced by `ImportedProduct`.

Validation:

- `ExternalSellerId`, `DisplayName`, and `LogoUrl` are required for product bundle submission.
- A product is not import-ready until the seller is matched or created with both account and store ownership.
- Seller conflicts block affected products.
- Similar display names alone create review conflicts; they do not create active ownership links.
- Imported products reference seller logo only through their `ImportedSeller` relationship; seller logo is not a product-owned catalog field.

## SellerOwnershipLink

Records the approved mapping from a Tiki seller to HiveSpace seller account/store ownership.

Fields:

- `ImportedSellerId`: imported seller.
- `HiveSpaceUserId`: identity/user public ID.
- `HiveSpaceStoreId`: UserService store ID.
- `LinkStatus`: `Pending`, `Active`, `Conflict`, `Retired`.
- `CreatedAt`, `UpdatedAt`.
- `CreatedByUserId`: admin/operator or system actor.
- `ApprovedByUserId`: nullable admin/operator user ID when the link was manually approved.
- `ApprovedAt`: nullable approval timestamp.
- `ApprovalReason`: nullable operator note or reason code.

Validation:

- One active link per `SourceSystem` plus `ExternalSellerId`.
- `HiveSpaceUserId` and `HiveSpaceStoreId` must both be present for `Active`.
- Manual approval may activate a link only after the target account and store are eligible for seller ownership.
- Manual approval must not update the existing store profile, name, metadata, owner, or lifecycle state from Tiki seller metadata.

## ExternalCategoryLink

Records a provisioned link from an external Tiki category to a CatalogService-owned category. These links are created before product bundle submission.

Fields:

- `Id`: category link ID.
- `SourceSystem`: `tiki`.
- `ExternalCategoryId`.
- `ExternalCategoryName`.
- `ExternalParentCategoryId`: nullable.
- `PathJson`: source category path.
- `HiveSpaceCategoryId`: CatalogService category ID.
- `ProvisioningStatus`: `Created`, `Matched`, `Conflict`, `Failed`.
- `ConflictReason`: nullable reason code/detail.
- `SourceFingerprint`: category crawl fingerprint that last provisioned the link.
- `ProvisioningJobId`: `CatalogImportJob` ID for the provisioning execution that last touched this link.
- `ProvisionedByUserId`, `ProvisionedAt`.

Validation:

- One active link per `SourceSystem` plus `ExternalCategoryId`.
- Parent categories are provisioned before children when both are in the submission.
- Product bundle submission validates product `ExternalCategoryIds` against these links.
- Product import does not create categories.

## ImportedProduct

Normalized product candidate shaped for HiveSpace catalog review.

Fields:

- `Id`: imported product ID.
- `BundleId`.
- `ExternalProductId`.
- `ExternalProductUrl`.
- `ExternalSellerId`.
- `Title`.
- `Description`: nullable.
- `ExternalCategoryIds`.
- `HiveSpaceCategoryIds`: resolved from previously provisioned `ExternalCategoryLink` records.
- `Condition`: normalized product condition where available.
- `ThumbnailImageExternalUrl`: nullable.
- `ReadinessStatus`: `Ready`, `Blocked`, `Warning`, `Duplicate`, `Imported`.
- `ImportStatus`: `NotImported`, `Imported`, `Skipped`, `Failed`.
- `ImportedProductId`: nullable CatalogService product ID after import.

Relationships:

- Belongs to one `ImportedSeller`.
- Has many `ImportedSku`, `ImportedAttribute`, and imported image references.
- Belongs to zero or one `ImportDuplicateGroup`.

Validation:

- `ExternalProductId`, `ExternalSellerId`, `Title`, at least one provisioned category link, and at least one valid SKU are required for import.
- Duplicate source products are grouped and only one selected representative can become ready for import.
- `Warning` means the product remains reviewable and importable when all import-blocking conditions are clear.
- Imported products are created as seller-owned catalog records using the accepted `Draft`, `Unpublish`, or `Available` publication state.

## ImportedSku

Purchasable product variation candidate.

Fields:

- `Id`: imported SKU ID.
- `ImportedProductId`.
- `ExternalSkuId`.
- `SkuNumber`: normalized candidate SKU number.
- `VariantSelectionsJson`.
- `PriceAmount`: nullable long smallest currency unit.
- `CurrencyCode`: expected `VND`.
- `StockQuantity`: nullable integer.
- `IsActiveCandidate`: boolean.
- `ReadinessStatus`: `Ready`, `Blocked`, `Warning`, `Duplicate`.

Validation:

- SKU price must be numeric, positive, and VND.
- VND must be enabled in CatalogService's local currency-policy projection.
- Stock cannot be negative; missing or unknown stock is a warning or blocker according to validation policy.
- At least one SKU with `Ready` or `Warning` status must remain importable before a product can be imported.

## ImportedAttribute

External product metadata matched or flagged against HiveSpace category attributes.

Fields:

- `ImportedProductId`.
- `ExternalAttributeName`.
- `ExternalAttributeValue`.
- `HiveSpaceAttributeDefinitionId`: nullable.
- `MatchStatus`: `Matched`, `Unmatched`, `Ignored`, `Conflict`.

Validation:

- Mandatory HiveSpace category attributes must be present or receive a blocking issue.
- Unmapped optional attributes may remain warnings.

## ImportedImageReference

External image reference captured from Tiki.

Fields:

- `ImportedProductId`.
- `ImportedSkuId`: nullable.
- `ExternalUrl`.
- `SourceImageId`: nullable.
- `Role`: `Thumbnail`, `ProductImage`, `SkuImage`.
- `MediaStatus`: `ExternalOnly`, `CopyRequested`, `Copied`, `Unsupported`, `Inaccessible`, `Duplicate`.
- `MediaFileId`: nullable MediaService file ID.

Validation:

- Missing or inaccessible required thumbnail/media references create validation issues.
- Binary storage and processing remain MediaService-owned.

## ImportValidationIssue

Operator-facing validation result.

Fields:

- `BundleId`.
- `EntityType`: `Bundle`, `Seller`, `Category`, `Product`, `Sku`, `Image`, `Attribute`.
- `EntitySourceId`.
- `Field`: nullable affected field.
- `Severity`: `Blocking` or `Warning`.
- `ReasonCode`.
- `Message`.
- `CreatedAt`.

Validation:

- Blocking issues prevent affected records from being imported.
- Warnings remain visible but do not necessarily block import.

## ImportDuplicateGroup

Groups source products that refer to the same Tiki product.

Fields:

- `BundleId`.
- `DuplicateKey`.
- `ExternalProductIds`.
- `RepresentativeImportedProductId`: nullable.
- `ResolutionStatus`: `Unresolved`, `Resolved`, `Ignored`.

Validation:

- Duplicate group must be resolved before any member is treated as ready.

## State Transitions

Catalog import job:

```text
Pending -> Running -> Completed
Pending -> Running -> Failed
Failed -> Pending (retry creates or reuses a retry job for idempotent operations)
Completed -> Pending (retry only when the operation contract allows idempotent replay)
```

Bundle:

```text
Submitted -> Validated
Submitted -> NeedsAttention
NeedsAttention -> Validated
Validated -> ProvisioningSellers
ProvisioningSellers -> ReadyToImport
ReadyToImport -> PartiallyImported
ReadyToImport -> Imported
PartiallyImported -> Imported
Any non-imported state -> Failed
```

Seller provisioning:

```text
Unmatched -> CreateRequested -> Created
Unmatched -> Matched
CreateRequested -> Failed
Unmatched -> Conflict
Conflict -> Matched
```

Product import:

```text
NotImported -> Imported
NotImported -> Skipped
NotImported -> Failed
```
