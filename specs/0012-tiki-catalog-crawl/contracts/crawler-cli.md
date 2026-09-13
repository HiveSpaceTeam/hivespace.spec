# Contract: Python Crawler CLI

## Commands

The Python crawler should expose one executable entrypoint, for example:

```bash
python -m hivespace_crawler tiki crawl --source-type category --source-value 1846 --out .\out\bundle.json
```

The crawler also supports category-first all-category crawling:

```bash
python -m hivespace_crawler tiki crawl-all-categories --state-dir .\crawler-state\tiki --out-dir .\crawler-state\tiki\bundles --limit-per-category 100
```

Supported source types:

- `category`
- `search`
- `product_list`

All-category crawling is exposed as the `crawl-all-categories` action instead of a `--source-type` value.
The operator must submit the discovered `categories.json` and the chunked category-attribute output to CatalogService provisioning before crawling and submitting product bundles for import validation.

## Required Options

For `tiki crawl`:

| Option | Meaning |
| --- | --- |
| `--source-type` | Tiki source selector kind. |
| `--source-value` | Category ID/URL, keyword, or product-list file/path depending on source type. |
| `--out` | Output path for the import bundle JSON. |

For `tiki crawl-all-categories`:

| Option | Meaning |
| --- | --- |
| `TIKI_SELLER_BEARER_TOKEN` | Environment variable containing the SellerCenter bearer token used for category discovery. |
| `--state-dir` | Local crawler state directory. Defaults to `crawler-state/tiki`. |
| `--out-dir` | Directory for per-category import bundles. Defaults to `<state-dir>/bundles`. |

## Recommended Options

For `tiki crawl`:

| Option | Meaning |
| --- | --- |
| `--checkpoint-dir` | Directory for resumable crawl state. |
| `--limit` | Maximum products to collect for an operator trial run. |
| `--request-timeout-seconds` | Per-request timeout. |
| `--rate-limit-per-minute` | Per-host rate limit for polite source access. |
| `--user-agent` | Configurable crawler user agent. |

For `tiki crawl-all-categories`:

| Option | Meaning |
| --- | --- |
| `--seller-parent-id` | SellerCenter root parent category ID. Defaults to `2`. |
| `--refresh-categories` | Re-fetch SellerCenter categories instead of reusing `categories.json`. |
| `--refresh-category-attributes` | Re-fetch category-scoped attribute definitions instead of reusing completed category-attribute chunks. |
| `--attribute-chunk-size` | Number of categories per category-attribute chunk file. Defaults to `100`. |
| `--limit-per-category` | Maximum products to collect for each category. Omit to crawl all reachable pages. |
| `--request-timeout-seconds` | Per-request timeout. |
| `--rate-limit-per-minute` | Per-host rate limit for polite source access. |
| `--user-agent` | Configurable crawler user agent. |

## Exit Codes

| Code | Meaning |
| ---: | --- |
| 0 | Bundle written successfully. |
| 1 | Invalid CLI arguments or unsupported source type. |
| 2 | Source access failed before a reviewable bundle could be created. |
| 3 | Bundle write failed. |
| 4 | Partial bundle written with blocking source gaps, category failures, or Tiki HTML challenge content. |

## Output

The `--out` file for `tiki crawl` must match [import-bundle-schema.md](import-bundle-schema.md). Console output should be concise and include source selection, product count, seller count, category count, warning count, and output path.

`crawl-all-categories` writes local state:

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

- Category discovery calls SellerCenter `catalog/categories` recursively from `--seller-parent-id`.
- Category attribute discovery calls SellerCenter product-set attribute APIs for each discovered category `productSetId`.
- `categories.json` must match [category-provisioning-schema.md](category-provisioning-schema.md) and is submitted before product bundle submission.
- `category-attributes/manifest.json` tracks chunk state, and each `chunk-*.json` file contains category-scoped attribute definitions, source attribute IDs, and known selectable values discovered during the preparation phase.
- Existing `categories.json` is reused unless `--refresh-categories` is passed.
- Completed category-attribute chunks are reused unless `--refresh-category-attributes` is passed.
- Completed category bundles are skipped on rerun.
- Failed or running categories are retried from the start of that category.
- If Tiki product listing APIs return HTML challenge content instead of JSON, the run stops and can be resumed later.
