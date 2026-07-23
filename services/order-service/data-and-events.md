# OrderService Data And Events

## Data Ownership

OrderService owns:

- `orders`
- `order_items`
- `order_checkouts`
- `order_discounts`
- linked payment summary fields on order read/state records
- cart tables and selected cart state
- coupon tables and usage records
- saga state tables for checkout and fulfillment
- MassTransit inbox/outbox tables
- `product_refs`
- `sku_refs`
- `store_refs`
- `platform_currency_config_refs`

## Consumed Projection Events

| Event | Purpose |
|---|---|
| `ProductCreatedIntegrationEvent` | Create product projection |
| `ProductUpdatedIntegrationEvent` | Refresh product projection |
| `ProductDeletedIntegrationEvent` | Deactivate product projection |
| `ProductSkuUpdatedIntegrationEvent` | Refresh SKU projection |
| `StoreCreatedIntegrationEvent` | Create store projection |
| `StoreUpdatedIntegrationEvent` | Refresh store projection |
| `PlatformCurrencyPolicyUpdatedIntegrationEvent` | Refresh local currency-policy validation projection for coupon, cart, checkout, and order money rules |

## Consumed Saga Events

| Event | Purpose |
|---|---|
| `InventoryReservedIntegrationEvent` / `InventoryReservationFailedIntegrationEvent` | Continue or compensate checkout |
| `PaymentSucceededIntegrationEvent` / `PaymentFailedIntegrationEvent` | Continue or fail payment path only when checkout correlation and attempt identity match the current payment attempt |
| `PaymentInitiatedIntegrationEvent` / `PaymentInitiationFailedIntegrationEvent` | Continue or compensate payment initiation with checkout-level payment reference, current attempt identity, and linked order context |
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
| `InitiatePayment` | PaymentService; sent once per checkout with canonical method, final total, currency, and full linked order set |
| `NotifySellerNewOrder` | NotificationService |
| `NotifyBuyerOrderConfirmed` | NotificationService |
| `NotifyBuyerOrderCancelled` | NotificationService |

## Publisher Policy

- OrderService application publishing uses service-owned publisher abstractions for order-owned integration events.
- Checkout and fulfillment saga request/response, timeout, schedule, handoff, and continuation messages remain MassTransit orchestration messages.

## Invariants

- OrderService coordinates checkout but does not own catalog truth or payment gateway truth.
- OrderService owns per-order `ORD-{ULID}` generation and keeps historical order codes readable/searchable.
- OrderService stores linked payment summaries for order reads, but PaymentService remains the payment truth.
- Payment outcome handling must validate checkout correlation and current attempt identity before changing order lifecycle.
- Currency policy ownership remains in UserService; OrderService uses the local projection only for synchronous validation.
- Order records keep purchase-time snapshots of product, SKU, address, and pricing data.
- Coupon usage is committed only after the checkout path reaches the appropriate success point.
- Saga state and outbox/inbox tables must remain configured together.
