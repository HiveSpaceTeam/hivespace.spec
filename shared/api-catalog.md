# HiveSpace API Catalog

## Purpose

This catalog records public HTTP and realtime contracts used by HiveSpace clients. It is the planning reference for avoiding duplicate endpoints and for identifying which service owns a new API.

All browser-facing REST calls go through YARP ApiGateway and are versioned under:

```text
{VITE_GATEWAY_BASE_URL}/api/v1
```

Local gateway defaults:

- HTTP: `http://localhost:5000`
- HTTPS: `https://localhost:7001`

## Authorization Policies

| Policy | Allows |
|---|---|
| Anonymous | No token required |
| `Authorize` | Any authenticated JWT principal |
| `RequireUser` | Authenticated user |
| `RequireBuyerUser` | Authenticated buyer context |
| `RequireSeller` | Authenticated seller |
| `RequireAdmin` | Platform admin |
| `RequireSystemAdmin` | System admin |
| `RequireAdminOrUser` | Admin or normal authenticated user |

## Public Route Ownership

| Gateway path | Owner |
|---|---|
| `/.well-known/**` | IdentityService direct authority endpoint, not routed through ApiGateway |
| `/connect/**` | IdentityService direct authority endpoint, not routed through ApiGateway |
| `/Account/**` | IdentityService direct compatibility endpoint for legacy account URLs, not routed through ApiGateway |
| `/api/v1/accounts/**` | IdentityService |
| `/api/v1/admins/**` | IdentityService and UserService split by action |
| `/api/v1/users/**` | UserService |
| `/api/v1/stores/**` | UserService |
| `/api/v1/categories/**` | CatalogService |
| `/api/v1/products/**` | CatalogService |
| `/api/v1/media/**` | MediaService |
| `/api/v1/carts/**` | OrderService |
| `/api/v1/coupons/**` | OrderService |
| `/api/v1/orders/**` | OrderService |
| `/api/v1/payments/**` | PaymentService |
| `/api/v1/wallets/**` | PaymentService |
| `/api/v1/notifications/**` | NotificationService |
| `/api/v1/notification-preferences/**` | NotificationService |
| `/hubs/notifications/**` | NotificationService |

## IdentityService

IdentityServer public OIDC protocol endpoints are served directly by IdentityService on `http://localhost:5001`: `/.well-known/**` and `/connect/**`. Legacy `/Account/**` login, registration, and logout URLs are direct IdentityService compatibility redirects to frontend routes, not ApiGateway routes and not user-facing IdentityService UI. `/identity/**` and `/api/v1/identity/**` are intentionally not part of the target contract.

### Account and Email Verification

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/accounts/login` | Anonymous | Password login from frontend-owned UI; sets secure HttpOnly browser session cookie and CSRF token |
| POST | `/api/v1/accounts/register` | Anonymous | Frontend-owned public account registration where allowed; creates identity account, triggers existing profile creation flow, and sets browser session |
| POST | `/api/v1/accounts/session/refresh` | Session cookie + CSRF | Bootstrap after reload or refresh/rotate the browser session through the gateway |
| POST | `/api/v1/accounts/logout` | Session cookie + CSRF | Clear the browser session and CSRF cookie |
| POST | `/api/v1/accounts/email-verification` | `RequireAdminOrUser` | Send verification email |
| POST | `/api/v1/accounts/email-verification/verify` | Anonymous | Verify email token |

Browser session endpoints are served through ApiGateway under `/api/v1/accounts/**`. Successful login, registration, and refresh responses must not expose access or refresh tokens to browser scripts. Cookie-authenticated state-changing browser requests require a server-issued CSRF token in a custom header before downstream state changes.

### Admin Identity Management

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/admins` | `RequireAdmin` | Create admin account |
| GET | `/api/v1/admins` | `RequireAdmin` | List admin accounts |
| PUT | `/api/v1/admins/users/status` | `RequireAdmin` | Update identity-owned user/admin account status |
| DELETE | `/api/v1/admins/users/{userId}` | `RequireAdmin` | Delete or deactivate identity-owned account access |

## UserService

### Profile and Settings

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/users/me` | `RequireAdminOrUser` | Get authenticated user profile, including nullable avatar URL |
| PUT | `/api/v1/users/me` | `RequireAdminOrUser` | Update authenticated user profile, including optional avatar file ID |
| GET | `/api/v1/users/settings` | `RequireAdminOrUser` | Get locale/theme/user settings |
| PUT | `/api/v1/users/settings` | `RequireAdminOrUser` | Update locale/theme/user settings |

### Addresses

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/users/address` | `Authorize` | List user addresses |
| GET | `/api/v1/users/address/default` | `Authorize` | Get default address |
| GET | `/api/v1/users/address/{id}` | `Authorize` | Get address by ID |
| POST | `/api/v1/users/address` | `Authorize` | Create address |
| PUT | `/api/v1/users/address/{id}` | `Authorize` | Update address |
| DELETE | `/api/v1/users/address/{id}` | `Authorize` | Delete address |
| PUT | `/api/v1/users/address/{id}/default` | `Authorize` | Set default address |

### Stores

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/stores` | `RequireUser` | Register seller store |

Admin profile/store review APIs that do not change credentials, roles, lockout, email verification, or account status remain UserService-owned.

## CatalogService

### Seller Products

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/products` | `RequireSeller` | Create product |
| GET | `/api/v1/products` | `RequireSeller` | List seller products |
| GET | `/api/v1/products/{id}` | `RequireSeller` | Get seller product detail |
| PUT | `/api/v1/products/{id}` | `RequireSeller` | Update product |
| DELETE | `/api/v1/products/{id}` | `RequireSeller` | Delete or deactivate product |

### Storefront Catalog

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/products/summaries` | Anonymous | Search/list storefront product summaries |
| GET | `/api/v1/products/detail/{id}` | Anonymous | Get storefront product detail with SKUs |
| GET | `/api/v1/categories` | Anonymous | Get category tree |
| GET | `/api/v1/categories/homepage` | Anonymous | Get homepage categories |
| GET | `/api/v1/categories/{id}/attributes` | Anonymous | Get category attribute definitions |

## OrderService

### Cart

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/carts/summary` | `RequireUser` | Get cart summary for checkout |
| POST | `/api/v1/carts/items` | `RequireUser` | Add item to cart |
| PUT | `/api/v1/carts/items` | `RequireUser` | Update cart item quantities |
| DELETE | `/api/v1/carts/items/{cartItemId}` | `RequireUser` | Remove cart item |
| GET | `/api/v1/carts/items/selected/count` | `RequireUser` | Get selected cart item count |

### Cart Coupons

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/carts/coupons/platform` | `RequireUser` | Apply platform coupon |
| DELETE | `/api/v1/carts/coupons/platform` | `RequireUser` | Remove platform coupon |
| PUT | `/api/v1/carts/coupons/stores/{storeId}` | `RequireUser` | Apply store coupon |
| DELETE | `/api/v1/carts/coupons/stores/{storeId}` | `RequireUser` | Remove store coupon |

### Checkout and Orders

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/orders/checkout/preview` | `Authorize` | Preview checkout totals |
| POST | `/api/v1/orders/checkout` | `Authorize` | Start checkout saga |
| GET | `/api/v1/orders` | `Authorize` | List buyer orders |
| GET | `/api/v1/orders/{orderId}` | `Authorize` | Get order detail |
| GET | `/api/v1/orders/seller` | `RequireSeller` | List seller orders |
| POST | `/api/v1/orders/{orderId}/confirm` | `RequireSeller` | Seller confirms order |
| POST | `/api/v1/orders/{orderId}/reject` | `RequireSeller` | Seller rejects order |

### Coupons

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/coupons/available` | `RequireUser` | List available coupons for cart/store/products |
| POST | `/api/v1/coupons` | `RequireSeller` | Create coupon |
| GET | `/api/v1/coupons` | `RequireSeller` | List seller coupons |
| GET | `/api/v1/coupons/{id}` | `RequireSeller` | Get coupon detail |
| PUT | `/api/v1/coupons/{id}` | `RequireSeller` | Update coupon |
| DELETE | `/api/v1/coupons/{id}` | `RequireSeller` | Delete or deactivate coupon |
| POST | `/api/v1/coupons/{id}/end` | `RequireSeller` | End coupon early |

## PaymentService

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/payments/vnpay/return` | Anonymous | VNPay browser return endpoint |
| GET | `/api/v1/payments/webhook/{gateway}` | Anonymous | Payment gateway webhook/IPN |
| GET | `/api/v1/payments/{paymentId}` | `Authorize` | Get payment detail |
| GET | `/api/v1/payments/by-order/{orderId}` | `Authorize` | Get payment by order |
| GET | `/api/v1/wallets/me` | `Authorize` | Get current wallet balance |
| GET | `/api/v1/wallets/me/transactions` | `Authorize` | List wallet transactions |

Webhook endpoints should acknowledge gateway delivery even when internal processing is deferred or logged.

## MediaService

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/media/presign-url` | `Authorize` | Create direct-upload URL and media ID |
| POST | `/api/v1/media/{fileId}/confirm` | `Authorize` | Confirm upload and associate media with entity |

Upload flow: request URL, upload bytes directly to Blob/Azurite, then confirm through MediaService.

## NotificationService

### REST

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/notifications` | `Authorize` | List notifications |
| GET | `/api/v1/notifications/unread-count` | `Authorize` | Get unread count |
| PUT | `/api/v1/notifications/{id}/read` | `Authorize` | Mark notification read |
| GET | `/api/v1/notification-preferences` | `Authorize` | Get notification preferences |
| PUT | `/api/v1/notification-preferences/{channel}` | `Authorize` | Enable/disable channel |
| PUT | `/api/v1/notification-preferences/{channel}/{eventGroup}` | `Authorize` | Enable/disable event group for channel |

### SignalR

| Hub | Auth | Event | Purpose |
|---|---|---|---|
| `/hubs/notifications` | JWT | `ReceiveNotification` | Push notification event to connected clients |

`ReceiveNotification` payload shape:

```ts
interface NotificationHubEvent {
  id: string
  eventType: string
  payload: string
  createdAt: string
}
```

`payload` is serialized JSON and must be parsed by the client according to `eventType`.

## Frontend Consumers

| App | Main API domains |
|---|---|
| Admin | admins, users, notifications, profile, settings |
| Seller | products, categories, orders seller view, coupons, stores, media, notifications, profile, settings |
| Buyer | product discovery, categories, cart, checkout, buyer orders, payment, addresses, coupons, notifications, settings |

Shared HTTP behavior is implemented by `ApiService` in `@hivespace/shared`: bearer token injection, correlation ID, base URL construction, and common error handling.
