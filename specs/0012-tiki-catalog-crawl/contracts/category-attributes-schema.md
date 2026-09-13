# Contract: Tiki Category Attributes Schema

## Purpose

This JSON contract is produced by the Python crawler after SellerCenter category discovery. It is submitted separately from `categories.json` and contains category-scoped attribute definitions and known selectable values used before product bundle validation or import. The crawler writes one manifest plus one or more chunk files.

## Manifest Shape

```json
{
  "schemaVersion": "2026-08-16",
  "source": {
    "system": "tiki",
    "type": "sellercenter_category_attributes",
    "value": "parent:2",
    "url": "https://api-sellercenter.tiki.vn/katana/v1/productsets/{productSetId}/attributes"
  },
  "crawl": {
    "categorySourceFingerprint": "sha256:...",
    "chunkSize": 100,
    "totalCategories": 6101,
    "totalChunks": 62,
    "startedAt": "2026-08-16T10:00:00Z",
    "completedAt": "2026-08-16T10:30:00Z"
  },
  "chunks": []
}
```

## Chunk Shape

```json
{
  "schemaVersion": "2026-08-16",
  "source": {
    "system": "tiki",
    "type": "sellercenter_category_attributes",
    "value": "parent:2",
    "url": "https://api-sellercenter.tiki.vn/katana/v1/productsets/{productSetId}/attributes"
  },
  "crawl": {
    "startedAt": "2026-08-16T10:00:00Z",
    "completedAt": "2026-08-16T10:05:00Z",
    "sourceFingerprint": "sha256:...",
    "categorySourceFingerprint": "sha256:...",
    "checkpointId": null,
    "chunkIndex": 1
  },
  "categories": []
}
```

## Category Attribute Group

```json
{
  "externalCategoryId": "1846",
  "productSetId": "9001",
  "status": "complete",
  "errorSummary": null,
  "attributes": []
}
```

Required fields: `externalCategoryId`. `productSetId` should be present when SellerCenter exposes it for that category.

## Attribute Definition

```json
{
  "sourceAttributeId": "501",
  "name": "Brand",
  "isRequired": true,
  "inputType": "select",
  "maxValueCount": 1,
  "values": []
}
```

Required fields: `sourceAttributeId`, `name`, `inputType`, `maxValueCount`.

## Selectable Value

```json
{
  "sourceValueId": "1001",
  "value": "HiveSpace Press",
  "displayName": "HiveSpace Press",
  "isActive": true
}
```

Required fields: `sourceValueId`, `value`.

## Rules

- The crawler writes `category-attributes/manifest.json` plus one or more `chunk-*.json` files separately from `categories.json` so category hierarchy and category-attribute preparation can be retried or provisioned independently.
- Chunk files are grouped by a fixed category-count range, not by one-file-per-category.
- Categories without `productSetId` may appear with an empty `attributes` array.
- Attribute discovery should preserve stable source attribute IDs and known selectable values where SellerCenter exposes them.
- A chunk may be marked failed while still saving successful category attribute results for other categories in that same chunk.
- Re-submission must allow CatalogService to match previously provisioned category attributes and values instead of creating duplicates.
