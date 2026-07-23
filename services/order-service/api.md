# OrderService API

## Cart

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/carts/summary` | `RequireUser` | Get cart summary with explicit money metadata and reject mixed-currency cart state |
| POST | `/api/v1/carts/items` | `RequireUser` | Add item to cart |
| PUT | `/api/v1/carts/items` | `RequireUser` | Update cart item quantities |
| DELETE | `/api/v1/carts/items/{cartItemId}` | `RequireUser` | Remove cart item |
| GET | `/api/v1/carts/items/selected/count` | `RequireUser` | Get selected item count |

## Cart Coupons

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/carts/coupons/platform` | `RequireUser` | Apply platform coupon |
| DELETE | `/api/v1/carts/coupons/platform` | `RequireUser` | Remove platform coupon |
| PUT | `/api/v1/carts/coupons/stores/{storeId}` | `RequireUser` | Apply store coupon |
| DELETE | `/api/v1/carts/coupons/stores/{storeId}` | `RequireUser` | Remove store coupon |

## Checkout

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/orders/checkout/preview` | `Authorize` | Preview checkout totals with explicit money metadata and reject mixed or invalid currency calculation contexts |
| POST | `/api/v1/orders/checkout` | `Authorize` | Start checkout saga with a canonical payment method code and one checkout-level payment covering all generated orders when cart, coupon, and payment currency state is enabled and internally consistent |

## Buyer Orders

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/orders` | `Authorize` | List buyer orders |
| GET | `/api/v1/orders/{orderId}` | `Authorize` | Get order detail with explicit money metadata, order code, linked checkout payment reference, and invalid-money diagnostics when needed |

## Seller Orders

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/orders/seller` | `RequireSeller` | List seller orders |
| POST | `/api/v1/orders/{orderId}/confirm` | `RequireSeller` | Confirm order |
| POST | `/api/v1/orders/{orderId}/reject` | `RequireSeller` | Reject order |

## Coupons

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/coupons/available` | `RequireUser` | List available coupons |
| POST | `/api/v1/coupons` | `RequireSeller` | Create coupon using one canonical `currencyCode` for all coupon money fields and reject disabled currencies |
| GET | `/api/v1/coupons` | `RequireSeller` | List seller coupons with normalized coupon money metadata under one canonical `currencyCode` |
| GET | `/api/v1/coupons/{id}` | `RequireSeller` | Get coupon detail with normalized coupon money metadata under one canonical `currencyCode` |
| PUT | `/api/v1/coupons/{id}` | `RequireSeller` | Update coupon while preserving one canonical `currencyCode` across all coupon money fields |
| DELETE | `/api/v1/coupons/{id}` | `RequireSeller` | Delete or deactivate coupon |
| POST | `/api/v1/coupons/{id}/end` | `RequireSeller` | End coupon early |
