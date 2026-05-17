# API Contracts — HiveSpace Microservices

All endpoints are versioned under `/api/v{version}/`. The API Gateway at `http://localhost:5000` proxies all requests. Direct service ports are listed in the route table below.

---

## Authorization Policy Reference

| Policy Name | Who it allows |
|---|---|
| `RequireSystemAdmin` | System-level admin role only |
| `RequireAdmin` | Platform admin role |
| `RequireSeller` | User with seller role |
| `RequireUser` | Any authenticated user (any role) |
| `RequireBuyerUser` | Authenticated user in buyer context |
| `RequireAdminOrUser` | Admin or any authenticated user |
| `Authorize` (bare) | Any authenticated principal (valid JWT) |
| Anonymous / Public | No authentication required |

---

## YARP Gateway Route Table

The API Gateway (`http://localhost:5000`) routes based on path prefix to downstream clusters.

| Path Pattern | Downstream Service | Port |
|---|---|---|
| `/identity/**` | UserService | 5001 |
| `/api/v{version}/users/**` | UserService | 5001 |
| `/api/v{version}/accounts/**` | UserService | 5001 |
| `/api/v{version}/admins/**` | UserService | 5001 |
| `/api/v{version}/stores/**` | UserService | 5001 |
| `/api/v{version}/categories/**` | CatalogService | 5002 |
| `/api/v{version}/products/**` | CatalogService | 5002 |
| `/api/v{version}/media/**` | MediaService | 5003 |
| `/api/v{version}/coupons/**` | OrderService | 5004 |
| `/api/v{version}/carts/**` | OrderService | 5004 |
| `/api/v{version}/orders/**` | OrderService | 5004 |
| `/api/v{version}/payments/**` | PaymentService | 5005 |
| `/api/v{version}/wallets/**` | PaymentService | 5005 |
| `/api/v{version}/notifications/**` | NotificationService | 5006 |
| `/api/v{version}/notification-preferences/**` | NotificationService | 5006 |
| `/hubs/notifications/**` | NotificationService | 5006 (WebSocket) |

---

## UserService Endpoints

Base URL (via gateway): `http://localhost:5000`
Direct: `https://localhost:5001`
Framework: MVC controllers (not Carter/Minimal API)

### Profile & Settings

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| GET | `/api/v1/users/me` | `RequireAdminOrUser` | — | User profile DTO |
| PUT | `/api/v1/users/me` | `RequireAdminOrUser` | Update profile body | Updated profile DTO |
| GET | `/api/v1/users/settings` | `RequireAdminOrUser` | — | User settings DTO |
| PUT | `/api/v1/users/settings` | `RequireAdminOrUser` | Settings body | Updated settings DTO |

### Account / Email Verification

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| POST | `/api/v1/accounts/email-verification` | `RequireAdminOrUser` | — | 200 (sends email) |
| POST | `/api/v1/accounts/email-verification/verify` | Anonymous | `{ token }` | 200 or 400 |

### Store Management

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| POST | `/api/v1/stores` | `RequireUser` | Store creation body | 201 `CreateStoreResponseDto` |

### Admin Management

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| POST | `/api/v1/admins` | `RequireAdmin` | Admin creation body | 201 |
| GET | `/api/v1/admins` | `RequireAdmin` | Query params (pagination) | Paginated admin list |
| GET | `/api/v1/admins/users` | `RequireAdmin` | Query params (pagination) | Paginated user list |
| PUT | `/api/v1/admins/users/status` | `RequireAdmin` | `{ userId, status }` | 200 |
| DELETE | `/api/v1/admins/users/{userId}` | `RequireAdmin` | — | 204 |

### Address Management

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| GET | `/api/v1/users/address` | `Authorize` | — | Address list |
| POST | `/api/v1/users/address` | `Authorize` | Address body | 201 |
| PUT | `/api/v1/users/address/{id}` | `Authorize` | Address body | 200 |
| DELETE | `/api/v1/users/address/{id}` | `Authorize` | — | 204 |
| PUT | `/api/v1/users/address/{id}/default` | `Authorize` | — | 200 |

---

## CatalogService Endpoints

Base URL (via gateway): `http://localhost:5000`
Direct: `http://localhost:5002`
Framework: Carter (Minimal API)

### Products (Seller-facing)

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| POST | `/api/v1/products` | `RequireSeller` | Product creation body | 201 |
| GET | `/api/v1/products` | `RequireSeller` | Query params (pagination, filters) | Paginated product list |
| GET | `/api/v1/products/{id}` | `RequireSeller` | — | Product detail |
| PUT | `/api/v1/products/{id}` | `RequireSeller` | Product update body | 200 |
| DELETE | `/api/v1/products/{id}` | `RequireSeller` | — | 204 |

### Products (Storefront / Anonymous)

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| GET | `/api/v1/products/summaries` | Anonymous | Query params (pagination, search, filters) | Paginated product summaries |
| GET | `/api/v1/products/detail/{id}` | Anonymous | — | Full product detail with SKUs |

### Categories

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| GET | `/api/v1/categories` | Anonymous | — | Category tree |
| GET | `/api/v1/categories/homepage` | Anonymous | — | Homepage category list |
| GET | `/api/v1/categories/{id}/attributes` | Anonymous | — | Category attribute definitions |

---

## OrderService Endpoints

Base URL (via gateway): `http://localhost:5000`
Direct: `http://localhost:5004`
Framework: Carter (Minimal API)

### Cart

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| POST | `/api/v1/carts/summary` | `RequireUser` | Pagination body | Paginated cart summary |
| POST | `/api/v1/carts/items` | `RequireUser` | `{ productId, skuId, quantity }` | 201 |
| PUT | `/api/v1/carts/items` | `RequireUser` | Array of item updates | 200 |
| DELETE | `/api/v1/carts/items/{cartItemId}` | `RequireUser` | — | 204 |
| GET | `/api/v1/carts/items/selected/count` | `RequireUser` | — | `{ count }` |

### Cart Coupons

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| POST | `/api/v1/carts/coupons/platform` | `RequireUser` | `{ couponCode }` | 200 |
| DELETE | `/api/v1/carts/coupons/platform` | `RequireUser` | — | 204 |
| PUT | `/api/v1/carts/coupons/stores/{storeId}` | `RequireUser` | `{ couponCode }` | 200 |
| DELETE | `/api/v1/carts/coupons/stores/{storeId}` | `RequireUser` | — | 204 |

### Checkout

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| POST | `/api/v1/orders/checkout/preview` | `Authorize` | Checkout preview body | Checkout preview with totals |
| POST | `/api/v1/orders/checkout` | `Authorize` | Checkout initiation body | `CheckoutResponse { paymentUrl }` |

Notes:
- COD path: `paymentUrl` is null; order proceeds directly to FulfillmentSaga.
- Online payment path: `paymentUrl` is a VNPay/Momo redirect URL.

### Orders (Buyer)

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| GET | `/api/v1/orders` | `Authorize` | Query params (page, pageSize, status) | Paginated order list |
| GET | `/api/v1/orders/{orderId}` | `Authorize` | — | Order detail |

### Orders (Seller)

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| GET | `/api/v1/orders/seller` | `RequireSeller` | Query params (page, pageSize, status) | Paginated seller order list |
| POST | `/api/v1/orders/{orderId}/confirm` | `RequireSeller` | — | 200 |
| POST | `/api/v1/orders/{orderId}/reject` | `RequireSeller` | `{ reason }` | 200 |

### Coupons

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| GET | `/api/v1/coupons/available` | `RequireUser` | Query: `storeId`, `productIds` (array) | Available coupons list |
| POST | `/api/v1/coupons` | `RequireSeller` | Coupon creation body | 201 |
| GET | `/api/v1/coupons` | `RequireSeller` | Query params (pagination) | Paginated coupon list |
| PUT | `/api/v1/coupons/{id}` | `RequireSeller` | Coupon update body | 200 |
| DELETE | `/api/v1/coupons/{id}` | `RequireSeller` | — | 204 |
| POST | `/api/v1/coupons/{id}/end` | `RequireSeller` | — | 200 (ends coupon early) |

---

## PaymentService Endpoints

Base URL (via gateway): `http://localhost:5000`
Direct: `http://localhost:5005`
Framework: Carter (Minimal API)

### Payments

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| GET | `/api/v1/payments/vnpay/return` | Anonymous | VNPay redirect query params | Redirect to storefront result page |
| GET | `/api/v1/payments/webhook/{gateway}` | Anonymous | Gateway IPN payload | Always returns HTTP 200 |
| GET | `/api/v1/payments/{paymentId}` | `Authorize` | — | Payment detail DTO |
| GET | `/api/v1/payments/by-order/{orderId}` | `Authorize` | — | Payment detail DTO |

Notes:
- `{gateway}` in webhook route corresponds to gateway name (e.g. `vnpay`).
- VNPay IPN endpoint must always return 200 per VNPay specification; internal errors are logged and processed asynchronously.

### Wallets

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| GET | `/api/v1/wallets/me` | `Authorize` | — | Wallet balance DTO (available, escrow, reward points) |
| GET | `/api/v1/wallets/me/transactions` | `Authorize` | Query: `page` (int), `pageSize` (int, 1–100) | Paginated transaction history |

---

## MediaService Endpoints

Base URL (via gateway): `http://localhost:5000`
Direct: `http://localhost:5003`
Framework: Carter or lightweight controller (2-project pattern)

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| POST | `/api/v1/media/presign-url` | `Authorize` | `{ fileName, contentType, fileSize }` | `PresignUrlResponse { presignedUrl, fileId }` |
| POST | `/api/v1/media/{fileId}/confirm` | `Authorize` | `{ entityId }` | 200 |

**Upload flow**:
1. Call presign-url → get `presignedUrl` and `fileId`
2. PUT file bytes directly to `presignedUrl` (Azure Blob Storage)
3. Call confirm with `entityId` to associate the asset with its owning entity

---

## NotificationService Endpoints

Base URL (via gateway): `http://localhost:5000`
Direct: `http://localhost:5006`
Framework: Carter or lightweight controller (2-project pattern)

### Notifications

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| GET | `/api/v1/notifications` | `Authorize` | Query: `page`, `pageSize`, `unreadOnly` (bool) | Paginated notification list |
| GET | `/api/v1/notifications/unread-count` | `Authorize` | — | `{ count }` |
| PUT | `/api/v1/notifications/{id}/read` | `Authorize` | — | 200 |

### Notification Preferences

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| GET | `/api/v1/notification-preferences` | `Authorize` | — | Full preferences object (channels + event groups) |
| PUT | `/api/v1/notification-preferences/{channel}` | `Authorize` | `{ enabled: bool }` | 200 |
| PUT | `/api/v1/notification-preferences/{channel}/{eventGroup}` | `Authorize` | `{ enabled: bool }` | 200 |

Notes:
- `{channel}` values: `InApp`, `Email` (Phase 1); `Sms`, `Push` (Phase 2).
- `{eventGroup}` is a string key grouping related event types (e.g. `OrderUpdates`, `PromotionAlerts`).

---

## NotificationService SignalR Hub

| Property | Value |
|---|---|
| Hub path | `/hubs/notifications` |
| Gateway path | `/hubs/notifications/**` → NotificationService:5006 |
| Transport | WebSocket (with long-polling fallback) |
| Authentication | JWT Bearer token required in connection handshake |

### Client-Received Events

| Event Name | Payload Shape | Description |
|---|---|---|
| `ReceiveNotification` | `{ id, eventType, payload, createdAt }` | Pushed to connected client when a new in-app notification is created for that user |

### Connection Notes

- Clients connect with a valid JWT in the `Authorization` header or query string (`access_token`).
- Each user is placed in a group identified by their `UserId`; the server pushes to the group on notification creation.
- No client-to-server methods are defined in Phase 1 (read receipts go through the REST API).
