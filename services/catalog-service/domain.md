# CatalogService Domain Model

## Purpose

CatalogService owns product catalog truth: products, SKUs, variants, categories, attribute definitions, product media references, and store references used to validate seller catalog ownership.

Implementation source:

```text
../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Domain
```

## Core Model

| Model | Type | Meaning |
|---|---|---|
| `Product` | Aggregate root | Seller-owned product with name, slug, description, status, condition, featured flag, thumbnail, categories, attributes, images, SKUs, and variants |
| `Sku` | Entity under `Product` | Purchasable variant with SKU number, variant selections, image references, stock quantity, active flag, and price |
| `ProductVariant` / `ProductVariantOption` | Entity / value object | Product-level option dimensions and allowed values |
| `SkuVariant` | Value object | Concrete variant selection on a SKU |
| `ProductImage` / `SkuImage` | Value objects | Media file reference plus optional resolved image URL |
| `Category` | Aggregate root | Category hierarchy node with optional parent, image reference, active flag, and assigned category attributes |
| `AttributeDefinition` | Aggregate root | Attribute metadata with type, input behavior, mandatory flag, and value-count limit |
| `AttributeValue` | Entity | Allowed attribute value under an attribute definition |
| `StoreRef` | Projection aggregate | Local copy of UserService store data used for catalog ownership/display |
| `Weight`, `Dimensions` | Value objects | Physical product metadata; negative values are invalid |

## Business Rules

- Product ownership is by seller/store context; seller-only APIs must enforce that sellers can manage only their own products.
- Product status uses the shared `ProductStatus`; storefront reads should expose only active public catalog data.
- Product images and thumbnails store MediaService file IDs first, then receive resolved URLs after media processing.
- Updating an image URL is file-ID based and is ignored when the file ID is not attached to the product/SKU.
- SKU quantity cannot be negative.
- SKU price is represented by the shared `Money` value object and uses the platform money convention.
- Product categories, attributes, variants, and SKUs are replaced as collections during product upsert/update flows.
- Category attributes link categories to attribute definitions; removing a missing attribute is a no-op.
- Attribute definitions describe both value type and input type, including whether a value is mandatory and how many values may be supplied.
- Product physical metadata may include weight and dimensions; each dimension/weight value must be non-negative.

## Lifecycle

| Lifecycle | States / transitions |
|---|---|
| Product | Created or updated by seller workflows; deleted/deactivated products publish projection updates for downstream services |
| SKU inventory | Quantity changes are validated at the SKU level and participate in checkout inventory reservation/confirmation flows |
| Media URL resolution | Product, SKU, thumbnail, and category image URLs are set after MediaService processing completes |
| Store projection | `StoreRef` is created/refreshed from UserService store events |

## Cross-Service Facts

- CatalogService consumes `StoreCreatedIntegrationEvent` and `StoreUpdatedIntegrationEvent` to maintain `StoreRef`.
- CatalogService publishes product and SKU events so OrderService can maintain `product_refs` and `sku_refs`.
- CatalogService participates in checkout by reserving, releasing, and confirming inventory through saga messages.
- CatalogService owns product/SKU truth; OrderService projections are local read models, not catalog ownership.
- MediaService owns binary storage and processing; CatalogService owns product/category association to media references.
