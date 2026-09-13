# Quickstart: Tiki Catalog Crawl

## Prerequisites

- Backend local runtime from `../hivespace.microservice` through Aspire AppHost.
- Admin frontend from `../hivespace.web/apps/admin`.
- Python runtime and dependencies in the new `../hivespace.crawler` repository.
- A Tiki category, keyword, or product list source selected by the operator.
- For all-category crawling, a SellerCenter bearer token in `TIKI_SELLER_BEARER_TOKEN`.

## Generate A Bundle

```bash
cd ../hivespace.crawler
python -m hivespace_crawler tiki crawl --source-type category --source-value 1846 --out .\out\tiki-category-1846.json
```

Expected result:

- A JSON import bundle matching `specs/0012-tiki-catalog-crawl/contracts/import-bundle-schema.md`.
- Source traceability for products, sellers, categories, SKUs, prices, stock, images, and attributes where available.
- Missing optional Tiki fields recorded as warnings or validation hints.

## Generate And Submit Categories First

Use this flow before any product crawl. The operator first crawls SellerCenter categories, then provisions both the category output and the chunked category-attribute output before product bundle validation or import.

```powershell
cd ../hivespace.crawler
$env:PYTHONPATH = "src"
$env:PYTHONIOENCODING = "utf-8"
$env:TIKI_SELLER_BEARER_TOKEN = "<sellercenter-token>"

python -m hivespace_crawler tiki crawl-all-categories `
  --state-dir .\crawler-state\tiki `
  --out-dir .\crawler-state\tiki\bundles `
  --limit-per-category 100 `
  --rate-limit-per-minute 30
```

Expected local state:

```text
crawler-state/tiki/
  categories.json
  category-attributes/
    manifest.json
    chunk-0001.json
  manifest.json
  bundles/
    category-{categoryId}.bundle.json
```

Behavior:

- If `categories.json` exists, the crawler reuses it.
- If completed category-attribute chunks exist, the crawler reuses them.
- Pass `--refresh-categories` to discover SellerCenter categories again.
- Pass `--refresh-category-attributes` to recrawl category-scoped attribute definitions and values.
- Pass `--attribute-chunk-size` to control how many categories are written per category-attribute chunk file.
- Submit `categories.json` to `POST /api/v1/admins/catalog-imports/categories/provisioning` before product bundle submission.
- Submit the category-attribute chunks described by `category-attributes/manifest.json` to `POST /api/v1/admins/catalog-imports/categories/attributes/provisioning` before product bundle validation or import.
- The category provisioning API returns `202 Accepted` with a `jobId`; poll `GET /api/v1/admins/catalog-imports/jobs/{jobId}` until the job reaches `Completed` before product crawling or product bundle submission.
- Rerunning the same command skips completed product category bundles and retries failed or unfinished categories.
- If Tiki returns HTML challenge content from product listing APIs, the run stops and preserves completed bundles for later resume.

Live validation note from 2026-07-29:

- SellerCenter category discovery from parent `2` returned 6,101 categories.
- The first live product run wrote 6 category bundles before Tiki began returning HTML challenge content instead of product listing JSON.
- Operators should treat SellerCenter bearer tokens as short-lived secrets and rotate them after sharing or testing.

## Submit And Review

1. Start backend AppHost from `../hivespace.microservice`.
2. Start the admin app from `../hivespace.web`.
3. Open the admin Catalog Import list/upload page.
4. After validation and seller provisioning complete for a bundle, use the job detail page to import selected ready products or use the bundle-wide `Import all ready` action.
4. Upload or submit the generated category output first.
5. Upload or submit the generated category-attribute output next.
6. Confirm the category provisioning submission returns a job ID, source file name, and status without an HTTP timeout.
7. Confirm the job appears in the separate category provisioning upload history with job ID, file name, requested time, operation type, and status.
8. Open the job detail page from the history row.
9. Poll the job status until category provisioning completes with created, matched, failed, and conflict counts.
10. Confirm the category-attribute provisioning submission also returns a job ID and completes before bundle validation/import.
11. Crawl products after categories and category attributes are provisioned.
12. Return to the Catalog Import list/upload page, upload or submit the generated product bundle, and confirm the submission appears in the primary paginated product-bundle table.
13. Select the bundle row, review its related jobs in the right-side pane, and open the product bundle submission job detail page from a related job row to review validation summary, category links, duplicate groups, seller provisioning status, and blocking issues.

## Resolve Readiness

1. Ensure product categories resolve to previously provisioned category links.
2. Run validation and monitor the returned job until completion when validation is long-running.
3. Provision imported sellers and monitor the returned job until completion.
4. For similar-name seller/store conflicts, approve an ownership link only when the existing HiveSpace account/store is the intended owner for that Tiki seller.
5. Re-run validation until selected products have no blocking issues or unresolved duplicate blocks.
6. Import eligible products, including warning-only products, with `Draft`, `Unpublish`, or `Available` publication state and monitor the returned import job until completion.
7. Use the paginated product-bundle table and separate category provisioning upload history on the list/upload page, the selected-bundle related-jobs side pane, and the paginated category link, imported seller, imported product/SKU, duplicate group, and validation issue tables on the detail page.

## Verification

- Products with invalid VND prices, missing category provisioning, seller conflicts, invalid stock, required field gaps, or unusable media references remain blocked.
- Similar-name seller/store matches remain blocked until an operator explicitly approves the ownership link; approval must not overwrite existing store profile data.
- Duplicate source products are grouped rather than reviewed as unrelated products.
- Imported products remain hidden from buyer storefront results until existing publication rules make them visible.
- Seller accounts and stores are created by IdentityService and UserService, not by CatalogService or Python direct database writes.
