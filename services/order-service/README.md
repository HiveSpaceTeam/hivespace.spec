# OrderService

## Responsibility

OrderService owns cart, checkout orchestration, order lifecycle, coupons, and fulfillment sagas.

Source path:

```text
../hivespace.microservice/src/HiveSpace.OrderService
```

## Owns

- Buyer cart and selected cart items.
- Cart coupon selection.
- Checkout preview and checkout initiation.
- Order aggregate and order item snapshots.
- Server-side `OrderCode` generation for new orders using `ORD-{ULID}`.
- Linked checkout payment summaries on order reads.
- Seller order confirmation/rejection.
- Coupon creation, usage, and early ending.
- CheckoutSaga and FulfillmentSaga state.
- Product, SKU, and store projections needed for order processing.
- Local currency-policy validation projection used for coupon, cart, checkout, and order money rules.

## Must Not Own

- Product catalog truth.
- User identity truth.
- Payment gateway processing.
- Notification delivery implementation.
- Media processing.

## Architecture

OrderService follows standard Clean Architecture / DDD with CQRS and Minimal API endpoints. It also owns MassTransit saga state machines for checkout and fulfillment.

## Runtime

| Item | Value |
|---|---|
| Local HTTP | `http://localhost:5004` |
| Gateway prefixes | `/api/v1/carts`, `/api/v1/orders`, `/api/v1/coupons` |
| Database | SQL Server, OrderService-owned order/cart/coupon/saga schema |
| Messaging | MassTransit/RabbitMQ sagas and projections |

Backend local development starts OrderService through Aspire AppHost in `../hivespace.microservice/src/HiveSpace.AppHost`; frontend dev servers remain separate.

## Planning Notes

- Use a saga when checkout/fulfillment crosses CatalogService, PaymentService, NotificationService, or compensation is needed.
- Order item/product data should be snapshotted at purchase time.
- Seller order actions must enforce seller/store ownership.
- Coupon writes use one canonical aggregate `CurrencyCode` for `DiscountAmount`, `MaxDiscountAmount`, and `MinOrderAmount` instead of repeated per-field currency state.
- Cart, coupon application, checkout preview, and checkout initiation must reject mixed or disabled currency states before totals or payment flow proceed.
- Order read models must return explicit money metadata and invalid-money diagnostics for malformed historical money data instead of falling back to `VND`.
- Payment gateway result handling belongs to PaymentService; OrderService tracks the current payment attempt in saga/order state and reacts to current-attempt payment success/failure events.
- Checkout sends one `InitiatePayment` command per checkout with all generated orders and the final checkout total.
- COD checkout waits for PaymentService-created offline payment confirmation before marking linked orders as COD.
- Stale payment events from older attempts must not change order lifecycle after a newer attempt exists or the checkout payment already succeeded.
- Checkout, fulfillment, and shared handoff workflow contracts use standardized `*IntegrationEvent` names per [ADR-0002](../../architecture/decisions/ADR-0002-standardized-integration-event-contracts.md).
- Currency-policy ownership remains in UserService per [ADR-0010](../../architecture/decisions/ADR-0010-user-service-platform-currency-policy.md); OrderService only consumes the projection for validation.
- Checkout-level payment ownership is documented in [ADR-0011](../../architecture/decisions/ADR-0011-checkout-level-payment-ownership.md).

## Detail

- Domain model: `domain.md`
- Public API: `api.md`
- Data and events: `data-and-events.md`
- Workflows: `workflows.md`
