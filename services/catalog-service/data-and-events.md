# CatalogService Data And Events

## Data Ownership

CatalogService owns:

- `products`
- `product_variants`
- `product_variant_options`
- `sku_variants`
- `catalog_import_jobs`
- `catalog_import_bundles`
- `imported_sellers`
- `imported_products`
- `imported_skus`
- `imported_attributes`
- `imported_image_references`
- `import_validation_issues`
- `import_duplicate_groups`
- `categories`
- `external_category_links`
- `external_category_attribute_links`
- `seller_ownership_links`
- `product_categories`
- `attribute_definitions`
- `attribute_values`
- `product_attributes`
- `category_attributes`
- `store_refs`
- `platform_currency_config_refs`

## Consumed Events

| Event | Purpose |
|---|---|
| `StoreCreatedIntegrationEvent` | Create local store reference |
| `StoreUpdatedIntegrationEvent` | Refresh local store reference |
| `PlatformCurrencyPolicyUpdatedIntegrationEvent` | Refresh local currency-policy validation projection for product and SKU writes |

## Published Events

| Event | Purpose |
|---|---|
| `ProductCreatedIntegrationEvent` | Let downstream services create product projections |
| `ProductUpdatedIntegrationEvent` | Refresh product projections |
| `ProductDeletedIntegrationEvent` | Deactivate product projections |
| `ProductSkuUpdatedIntegrationEvent` | Refresh SKU price/availability projections |
| `BackgroundJobQueuedIntegrationEvent` | Observer-only monitoring when a CatalogService-owned import job is queued |
| `BackgroundJobStartedIntegrationEvent` | Observer-only monitoring when a CatalogService-owned import job starts |
| `BackgroundJobProgressedIntegrationEvent` | Observer-only monitoring of import job progress counts or milestones |
| `BackgroundJobCompletedIntegrationEvent` | Observer-only monitoring when a CatalogService-owned import job completes |
| `BackgroundJobFailedIntegrationEvent` | Observer-only monitoring when a CatalogService-owned import job fails |

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
- CatalogService publishes background job lifecycle events for monitoring only; consumers must not execute catalog import domain work from those events.
- Saga participant responses remain MassTransit consume-context workflow messages.

## Invariants

- Product/SKU facts are authoritative only in CatalogService.
- Currency policy ownership remains in UserService; CatalogService uses the local projection only for synchronous validation.
- Product/SKU projections in OrderService are read models, not catalog ownership.
- Anonymous storefront APIs must not expose seller-only draft or private data.
- Long-running Tiki catalog import operations, including category-attribute provisioning, are CatalogService-owned asynchronous jobs processed by same-host background consumers and exposed through job status endpoints.
- Tiki product import uses previously provisioned `external_category_links`; product import must not create categories.
- Tiki product import validation uses previously provisioned `external_category_attribute_links`; product import must not create category attribute definitions or selectable values.
- Tiki seller import uses approved `seller_ownership_links` keyed by external seller identity; similar-name conflicts remain blocked until operator approval creates a link.
- Operator-approved seller ownership links do not change UserService-owned store profile data.
