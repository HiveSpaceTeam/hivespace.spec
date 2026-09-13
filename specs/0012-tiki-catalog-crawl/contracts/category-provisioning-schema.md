# Contract: Tiki Category Provisioning Schema

## Purpose

This JSON contract is produced by the Python crawler after SellerCenter category discovery and submitted to CatalogService before any product bundle is crawled or submitted. It is category-only source data used to create or match HiveSpace categories and persist external Tiki category links.

## Top-Level Shape

```json
{
  "schemaVersion": "2026-07-30",
  "source": {
    "system": "tiki",
    "type": "sellercenter_categories",
    "value": "parent:2",
    "url": "https://sellercenter.tiki.vn/api/tiki_api?path=catalog%2Fcategories"
  },
  "crawl": {
    "startedAt": "2026-07-30T10:00:00Z",
    "completedAt": "2026-07-30T10:05:00Z",
    "sourceFingerprint": "sha256:...",
    "checkpointId": "optional-checkpoint-id"
  },
  "categories": []
}
```

## Category

```json
{
  "externalCategoryId": "1846",
  "externalParentCategoryId": null,
  "name": "Nha sach Tiki",
  "path": ["Nha sach Tiki"],
  "productSetId": "optional-product-set-id",
  "imageUrl": "https://salt.tikicdn.com/ts/category/example.png",
  "imageFileId": null,
  "metadata": {}
}
```

Required fields: `externalCategoryId`, `name`.

Rules:

- Parent categories must be processed before children when both are present in the same submission.
- Re-submission must match existing external category links instead of creating duplicates.
- Missing parents, duplicate external IDs with conflicting names or parents, and invalid names produce per-category conflict results.
- `imageUrl` and `imageFileId` are optional. When present, CatalogService stores them on the matched or created HiveSpace category.
- Category provisioning is complete before product bundle submission; product bundles only reference `externalCategoryIds`.
