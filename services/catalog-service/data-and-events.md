# CatalogService Data And Events

## Data Ownership

CatalogService owns:

- `products`
- `product_variants`
- `product_variant_options`
- `sku_variants`
- `categories`
- `product_categories`
- `attribute_definitions`
- `attribute_values`
- `product_attributes`
- `category_attributes`
- `store_refs`

## Consumed Events

| Event | Purpose |
|---|---|
| `StoreCreatedIntegrationEvent` | Create local store reference |
| `StoreUpdatedIntegrationEvent` | Refresh local store reference |

## Published Events

| Event | Purpose |
|---|---|
| `ProductCreatedIntegrationEvent` | Let downstream services create product projections |
| `ProductUpdatedIntegrationEvent` | Refresh product projections |
| `ProductDeletedIntegrationEvent` | Deactivate product projections |
| `ProductSkuUpdatedIntegrationEvent` | Refresh SKU price/availability projections |

## Checkout Saga Participation

| Message | Role |
|---|---|
| `ReserveInventory` | Reserve stock for checkout |
| `InventoryReserved` | Report successful reservation |
| `InventoryReservationFailed` | Report reservation failure |
| `ReleaseInventory` | Release reservation during compensation |
| `InventoryReleased` | Report release success |
| `ConfirmInventory` | Finalize inventory after seller confirmation |
| `InventoryConfirmed` | Report final confirmation |
| `InventoryConfirmationFailed` | Report final confirmation failure |

## Invariants

- Product/SKU facts are authoritative only in CatalogService.
- Product/SKU projections in OrderService are read models, not catalog ownership.
- Anonymous storefront APIs must not expose seller-only draft or private data.
