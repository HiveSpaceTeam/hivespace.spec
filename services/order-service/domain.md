# OrderService Domain Model

## Purpose

OrderService owns carts, checkout/order records, coupon rules and usage, seller fulfillment decisions, saga state, and the local product/SKU/store projections required to process orders.

Implementation source:

```text
../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Domain
```

## Core Model

| Model | Type | Meaning |
|---|---|---|
| `Cart` | Aggregate root | Buyer's selected items plus applied platform/store coupon choices |
| `CartItem` | Entity under `Cart` | Product/SKU/quantity row with selected state |
| `Coupon` | Aggregate root | Platform or store promotion with discount type, scope, date window, usage limits, applicability rules, and usage history |
| `CouponUsage` / `CouponRule` | Entities under `Coupon` | Coupon usage audit and custom validation rules |
| `Order` | Aggregate root | Purchase record split by store with delivery snapshot, item snapshots, checkout breakdown, discounts, totals, status, and tracking entries |
| `OrderItem`, `Checkout`, `Discount`, `OrderTracking` | Child entities/value objects | Purchase-time details and audit trail under an order |
| `ProductRef`, `SkuRef`, `StoreRef` | Projection models | Local copies of catalog/store facts required for cart, checkout, and notifications |
| `DeliveryAddress`, `ProductSnapshot`, `PackageDimensions`, `PhoneNumber` | Value objects | Immutable checkout/order snapshots |

## Cart Rules

- Cart user ID, product ID, SKU ID, and item quantity are required.
- Adding an existing product/SKU pair increments its quantity.
- Updating a cart item requires a positive quantity and an existing item.
- New cart items are selected by default.
- Checkout validation fails for an empty cart.
- Clearing selected items also clears platform coupons and removes store coupons for purchased stores.
- Store coupons are one-per-store in the cart; applying a new store coupon replaces the existing coupon for that store.
- Platform coupon codes are normalized to uppercase and duplicate applications are ignored.

## Coupon Rules

- Coupon codes are normalized to uppercase.
- Coupons may be platform-owned or store-owned.
- Discounts are either fixed amount or percentage.
- Fixed amount discounts must be positive.
- Percentage discounts must be greater than 0 and no more than 100.
- Percentage coupons may have a maximum discount amount; if supplied, it must be large enough for the configured minimum order amount.
- Coupons validate active state, date window, minimum order amount, total usage limit, per-user limit, product applicability, category applicability, and store applicability.
- Coupon usage increments only when marked as used for an order; usage can be released by order ID during compensation.
- Coupon end time must remain after start time.
- Early-save time must be before start time and cannot be changed after early-save display has already started.

## Order Rules

- An order requires buyer user ID, store ID, and delivery address.
- New orders start as `Created`, receive a generated `ORD-{timestamp}-{random}` code, and expire after 24 hours.
- Items, shipping fee, and discounts may be changed only while the order is in a valid pre-payment state.
- Discounts cannot be applied before all item/shipping changes are complete; item/shipping changes are blocked after discounts exist.
- Store coupons may apply only to orders for the same store.
- Platform discounts can be prorated across store orders.
- Totals are recalculated from item line totals, discounts, and buyer-paid shipping.
- Online payment moves `Created` orders to `Paid`; duplicate paid marking is idempotent.
- COD marking is allowed only from `Created` and fails when total amount exceeds the COD limit.
- Seller confirmation/rejection is allowed only from `Paid` or `COD`.
- Confirmed orders can be assigned shipping and moved through `ReadyToShip`, `Shipped`, `Delivered`, and `Completed`.
- Orders can be cancelled only while `Created`, `Paid`, `COD`, or `Confirmed`.
- Expiration is valid only from `Created`.
- Every major lifecycle action appends an `OrderTracking` entry.

## Lifecycle

| Lifecycle | States / transitions |
|---|---|
| Order status | `Created`, `Paid`, `COD`, `Confirmed`, `Rejected`, `ReadyToShip`, `Shipped`, `Delivered`, `Completed`, `Cancelled`, `Claimed`, `Refunding`, `Refunded`, `Solved`, `Expired` |
| Confirm/reject | `Paid` or `COD` only |
| Ship | `Confirmed` or `ReadyToShip`; shipping ID is required before shipment |
| Final states | `Completed`, `Refunded`, `Solved`, `Expired`; cancelled/rejected are terminal business outcomes even though they are not `IsFinal()` in the enumeration helper |
| Checkout saga | Creates orders, reserves inventory, handles COD/payment path, commits coupon usage, clears cart, and starts fulfillment |
| Fulfillment saga | Notifies seller, waits for seller action/timeout, confirms inventory or compensates, then notifies buyer |

## Cross-Service Facts

- OrderService consumes product, SKU, and store events to maintain local projections.
- Product and SKU projections are snapshots for order processing; CatalogService remains the catalog source of truth.
- PaymentService owns gateway/payment truth; OrderService reacts to payment success/failure events.
- CatalogService owns inventory reservation/confirmation; OrderService coordinates those steps through saga messages.
- NotificationService owns delivery; OrderService requests buyer/seller notifications through saga messages.
