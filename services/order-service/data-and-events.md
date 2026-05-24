# OrderService Data And Events

## Data Ownership

OrderService owns:

- `orders`
- `order_items`
- `order_checkouts`
- `order_discounts`
- cart tables and selected cart state
- coupon tables and usage records
- saga state tables for checkout and fulfillment
- MassTransit inbox/outbox tables
- `product_refs`
- `sku_refs`
- `store_refs`

## Consumed Projection Events

| Event | Purpose |
|---|---|
| `ProductCreatedIntegrationEvent` | Create product projection |
| `ProductUpdatedIntegrationEvent` | Refresh product projection |
| `ProductDeletedIntegrationEvent` | Deactivate product projection |
| `ProductSkuUpdatedIntegrationEvent` | Refresh SKU projection |
| `StoreCreatedIntegrationEvent` | Create store projection |
| `StoreUpdatedIntegrationEvent` | Refresh store projection |

## Consumed Saga Events

| Event | Purpose |
|---|---|
| `InventoryReservedIntegrationEvent` / `InventoryReservationFailedIntegrationEvent` | Continue or compensate checkout |
| `PaymentSucceededIntegrationEvent` / `PaymentFailedIntegrationEvent` | Continue or fail payment path |
| `PaymentInitiatedIntegrationEvent` / `PaymentInitiationFailedIntegrationEvent` | Continue or compensate payment initiation |
| `OrderReadyForFulfillmentIntegrationEvent` | Start fulfillment after checkout completion |
| `SellerNewOrderNotifiedIntegrationEvent` | Continue fulfillment notification step |
| `BuyerNotifiedIntegrationEvent` | Complete buyer notification step |
| `OrderConfirmedBySellerIntegrationEvent` / `OrderRejectedBySellerIntegrationEvent` | Continue fulfillment after seller decision |
| `SellerConfirmationExpiredIntegrationEvent` | Compensate fulfillment after seller confirmation timeout |
| `InventoryConfirmedIntegrationEvent` / `InventoryConfirmationFailedIntegrationEvent` | Continue or compensate final inventory confirmation |

## Published Saga Messages

| Message | Target |
|---|---|
| `ReserveInventory` | CatalogService |
| `ReleaseInventory` | CatalogService |
| `ConfirmInventory` | CatalogService |
| `InitiatePayment` | PaymentService |
| `NotifySellerNewOrder` | NotificationService |
| `NotifyBuyerOrderConfirmed` | NotificationService |
| `NotifyBuyerOrderCancelled` | NotificationService |

## Publisher Policy

- OrderService application publishing uses service-owned publisher abstractions for order-owned integration events.
- Checkout and fulfillment saga request/response, timeout, schedule, handoff, and continuation messages remain MassTransit orchestration messages.

## Invariants

- OrderService coordinates checkout but does not own catalog truth or payment gateway truth.
- Order records keep purchase-time snapshots of product, SKU, address, and pricing data.
- Coupon usage is committed only after the checkout path reaches the appropriate success point.
- Saga state and outbox/inbox tables must remain configured together.
