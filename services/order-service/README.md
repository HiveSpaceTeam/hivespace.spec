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
- Seller order confirmation/rejection.
- Coupon creation, usage, and early ending.
- CheckoutSaga and FulfillmentSaga state.
- Product, SKU, and store projections needed for order processing.

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

## Planning Notes

- Use a saga when checkout/fulfillment crosses CatalogService, PaymentService, NotificationService, or compensation is needed.
- Order item/product data should be snapshotted at purchase time.
- Seller order actions must enforce seller/store ownership.
- Payment result handling belongs to PaymentService; OrderService reacts to payment success/failure events.

## Detail

- Domain model: `domain.md`
- Public API: `api.md`
- Data and events: `data-and-events.md`
- Workflows: `workflows.md`
