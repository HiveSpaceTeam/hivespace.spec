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

## Checkout And Fulfillment Workflow Participation

| Message | Role |
|---|---|
| `ReserveInventory` | Reserve stock for checkout |
| `InventoryReservedIntegrationEvent` | Report successful reservation |
| `InventoryReservationFailedIntegrationEvent` | Report reservation failure |
| `ReleaseInventory` | Release reservation during compensation |
| `InventoryReleasedIntegrationEvent` | Report release success |
| `ConfirmInventory` | Finalize inventory after seller confirmation |
| `InventoryConfirmedIntegrationEvent` | Report final confirmation |
| `InventoryConfirmationFailedIntegrationEvent` | Report final confirmation failure |

## Publisher Policy

- CatalogService application publishing uses service-owned publisher abstractions for product/SKU integration events.
- Saga participant responses remain MassTransit consume-context workflow messages.

## Invariants

- Product/SKU facts are authoritative only in CatalogService.
- Product/SKU projections in OrderService are read models, not catalog ownership.
- Anonymous storefront APIs must not expose seller-only draft or private data.
