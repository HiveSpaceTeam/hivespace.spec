# Contract: Tiki Import Bundle Schema

## Purpose

This JSON contract is produced by the Python crawler after category provisioning has completed and submitted to CatalogService as product import data. It is not raw Tiki JSON; it is a HiveSpace-shaped product bundle with source traceability.

## Top-Level Shape

```json
{
  "schemaVersion": "2026-07-24",
  "source": {
    "system": "tiki",
    "type": "category",
    "value": "1846",
    "url": "https://tiki.vn/..."
  },
  "crawl": {
    "startedAt": "2026-07-24T10:00:00Z",
    "completedAt": "2026-07-24T10:05:00Z",
    "sourceFingerprint": "sha256:...",
    "checkpointId": "optional-checkpoint-id"
  },
  "sellers": [],
  "products": [],
  "validationHints": []
}
```

## Seller

```json
{
  "externalSellerId": "123",
  "displayName": "Tiki Trading",
  "slug": "tiki-trading",
  "url": "https://tiki.vn/cua-hang/tiki-trading",
  "logoUrl": "https://vcdn.tikicdn.com/ts/seller/tiki-seller-1.jpg",
  "metadata": {}
}
```

Required fields: `externalSellerId`, `displayName`, `logoUrl`.

`logoUrl` is normalized from the crawler's product-level `seller_logo_url` source value and belongs to the seller record. Product records reference sellers by `externalSellerId` and do not duplicate seller logo as product-owned catalog data.

## Product

```json
{
  "externalProductId": "271001",
  "externalSellerId": "123",
  "externalCategoryIds": ["1846"],
  "url": "https://tiki.vn/product.html",
  "title": "Product name",
  "description": "Optional product description",
  "thumbnailUrl": "https://...",
  "attributes": [],
  "images": [],
  "variants": [],
  "skus": []
}
```

Required fields: `externalProductId`, `externalSellerId`, `externalCategoryIds`, `title`, `skus`.

Rules:

- Every `externalCategoryId` must have been submitted through [category-provisioning-schema.md](category-provisioning-schema.md) and provisioned by CatalogService before the product can become ready.
- Product bundle submission and product import must not create categories.

## SKU

```json
{
  "externalSkuId": "271001-01",
  "skuNumber": "TIKI-271001-01",
  "variantSelections": {
    "Color": "Black"
  },
  "price": {
    "amount": 125000,
    "currencyCode": "VND",
    "sourceRawValue": "125000"
  },
  "stockQuantity": 10,
  "imageUrls": []
}
```

Required fields: `externalSkuId`, `price`.

Rules:

- `price.amount` is a positive integer in the smallest VND unit.
- `price.currencyCode` must be `VND`.
- `stockQuantity` must be null or a non-negative integer.

## Attribute

```json
{
  "name": "Brand",
  "value": "Example",
  "sourceAttributeId": "optional"
}
```

## Image Reference

```json
{
  "url": "https://...",
  "role": "ProductImage",
  "sourceImageId": "optional"
}
```

`role` values: `Thumbnail`, `ProductImage`, `SkuImage`.

## Validation Hint

Python may include source-level hints, but CatalogService remains authoritative for final validation.

```json
{
  "entityType": "Product",
  "entitySourceId": "271001",
  "field": "description",
  "severity": "Warning",
  "reasonCode": "MissingOptionalDescription",
  "message": "Description was not available from source."
}
```

`severity` values: `Warning`, `Blocking`.
