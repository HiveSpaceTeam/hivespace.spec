# API Contracts: HiveSpace Web Frontend (hivespace.web)

## Base URL Pattern

All API calls route through the YARP API Gateway. The base URL is constructed from environment variables:

```
{VITE_GATEWAY_BASE_URL}/api/{VITE_API_VERSION}/{path}
```

- `VITE_GATEWAY_BASE_URL` default: `https://localhost:7001`
- `VITE_API_VERSION` default: `v1`
- Full base: `https://localhost:7001/api/v1`

The `ApiService` in `@hivespace/shared` handles this construction. It also falls back through `VITE_API_BASE_URL` → `VITE_API_URL` if `VITE_GATEWAY_BASE_URL` is not set.

## Standard Request Headers

Every request from `ApiService` includes:

| Header | Value |
|--------|-------|
| `Authorization` | `Bearer {access_token}` |
| `X-Correlation-ID` | Generated UUID per request (tracing) |
| `Content-Type` | `application/json` (for POST/PUT/PATCH) |

---

## Admin App API Calls

### Admin Account Management

| Method | Endpoint | Purpose | Notes |
|--------|----------|---------|-------|
| `POST` | `/admins` | Create admin account | Request: `{ email, role, ... }` |
| `GET` | `/admins` | List admin accounts | Paginated |
| `PUT` | `/admins/users/status` | Update admin account status | Request: `{ userId, status }` |
| `DELETE` | `/admins/users/:userId` | Delete user account | Path param: `userId` |

### Buyer Account Management (Admin View)

| Method | Endpoint | Purpose | Notes |
|--------|----------|---------|-------|
| `GET` | `/admins/users` | List buyer accounts | Paginated |
| `PUT` | `/admins/users/status` | Update buyer account status | Request: `{ userId, status }` |

### Notifications (shared with seller/buyer)

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `GET` | `/notifications` | List notifications |
| `GET` | `/notifications/unread-count` | Get unread notification count |
| `PUT` | `/notifications/:id/read` | Mark notification as read |

### User Profile & Settings (shared with seller/buyer)

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `GET` | `/users/me` | Get authenticated user profile |
| `GET` | `/users/settings` | Get user settings (locale, theme) |
| `PUT` | `/users/settings` | Update user settings |

---

## Seller App API Calls

### Product Management

| Method | Endpoint | Purpose | Notes |
|--------|----------|---------|-------|
| `POST` | `/products` | Create product | Multipart or JSON with media IDs |
| `GET` | `/products` | List seller's products | Paginated |
| `GET` | `/products/:id` | Get product detail | — |
| `PUT` | `/products/:id` | Update product | — |
| `DELETE` | `/products/:id` | Delete product | — |

### Category & Attributes

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `GET` | `/categories` | List all categories |
| `GET` | `/categories/:id/attributes` | Get attributes for a category |

### Order Management

| Method | Endpoint | Purpose | Notes |
|--------|----------|---------|-------|
| `GET` | `/orders/seller` | List seller's orders | Query: filter params, pagination |
| `POST` | `/orders/:id/confirm` | Confirm order | — |
| `POST` | `/orders/:id/reject` | Reject order | — |

### Coupon Management

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `POST` | `/coupons` | Create coupon |
| `GET` | `/coupons` | List seller's coupons |
| `GET` | `/coupons/:id` | Get coupon detail |
| `PUT` | `/coupons/:id` | Update coupon |
| `DELETE` | `/coupons/:id` | Delete coupon |
| `POST` | `/coupons/:id/end` | End (deactivate) coupon early |

### Store Registration

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `POST` | `/stores` | Register seller store |

After successful store registration, the app performs a forced token refresh to obtain the `StoreOwner` role claim from the identity server.

### Media Upload (2-step presigned URL flow)

| Method | Endpoint | Purpose | Notes |
|--------|----------|---------|-------|
| `POST` | `/media/presign-url` | Get presigned upload URL | Returns: `{ uploadUrl, mediaId }` |
| `POST` | `/media/:id/confirm` | Confirm media upload complete | Called after PUT to `uploadUrl` |

The actual file bytes are PUT directly to the presigned URL (S3/compatible storage), not through the gateway.

### Notifications, Profile, Settings

Same endpoints as Admin — see [Admin App API Calls](#notifications-shared-with-sellersbuyer) above.

---

## Buyer App API Calls

### Product Discovery

| Method | Endpoint | Purpose | Notes |
|--------|----------|---------|-------|
| `GET` | `/products/summaries` | List/search products | Query: `keyword`, `sort`, `pageSize`, `page` |
| `GET` | `/products/detail/:id` | Get product detail | — |

### Cart Management

| Method | Endpoint | Purpose | Notes |
|--------|----------|---------|-------|
| `POST` | `/carts/items` | Add item to cart | Request: `{ productId, variantId, quantity }` |
| `PUT` | `/carts/items` | Update cart item quantity | Request: `{ cartItemId, quantity }` |
| `DELETE` | `/carts/items/:cartItemId` | Remove item from cart | — |
| `GET` | `/carts/items/selected/count` | Get selected item count | Used in header badge |
| `POST` | `/carts/summary` | Get cart summary for checkout preview | — |

### Cart Coupon Management

| Method | Endpoint | Purpose | Notes |
|--------|----------|---------|-------|
| `POST` | `/carts/coupons/platform` | Apply platform-wide coupon | — |
| `DELETE` | `/carts/coupons/platform` | Remove platform-wide coupon | — |
| `PUT` | `/carts/coupons/stores/:storeId` | Apply store coupon | Path param: `storeId` |
| `DELETE` | `/carts/coupons/stores/:storeId` | Remove store coupon | — |

### Checkout

| Method | Endpoint | Purpose | Response |
|--------|----------|---------|----------|
| `POST` | `/orders/checkout/preview` | Get checkout preview (prices, shipping) | Preview state |
| `POST` | `/orders/checkout` | Submit checkout | `CheckoutResult { paymentUrl, paymentExpiresAt }` |

After checkout, if `paymentUrl` is returned, the buyer is redirected to the payment gateway (VNPay/MoMo). COD orders may not have a `paymentUrl`.

### Order Management (Buyer)

| Method | Endpoint | Purpose | Notes |
|--------|----------|---------|-------|
| `GET` | `/orders` | List buyer's orders | Query: `page`, `pageSize`, `processStatus`, `searchField`, `searchValue` |
| `GET` | `/orders/:orderId` | Get order detail | — |

### Payment

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `GET` | `/payments/:paymentId` | Get payment by ID |
| `GET` | `/payments/by-order/:orderId` | Get payment for an order |

### Address Management

| Method | Endpoint | Purpose | Notes |
|--------|----------|---------|-------|
| `GET` | `/users/address` | List all addresses | Returned sorted default-first |
| `GET` | `/users/address/default` | Get default address | — |
| `GET` | `/users/address/:id` | Get address by ID | — |
| `POST` | `/users/address` | Create address | — |
| `PUT` | `/users/address/:id` | Update address | — |
| `DELETE` | `/users/address/:id` | Delete address | — |
| `PUT` | `/users/address/:id/default` | Set address as default | — |

### User Profile (Buyer)

| Method | Endpoint | Purpose | Notes |
|--------|----------|---------|-------|
| `GET` | `/users/me` | Get authenticated user profile | — |
| `PUT` | `/users/me` | Update user profile | Includes birth date, gender (int mapping) |

### Coupon Discovery

| Method | Endpoint | Purpose | Notes |
|--------|----------|---------|-------|
| `GET` | `/coupons/available` | Get available coupons for cart | Query: `storeId=`, `productIds=` (comma-separated) |

Results are keyed in `useCouponStore` by `"storeId:productIds"` to prevent duplicate fetches.

### Categories

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `GET` | `/categories/homepage` | Get homepage category tree |

### Notifications, Settings

Same endpoints as Admin — see [Admin App API Calls](#notifications-shared-with-sellersbuyer) above.

---

## SignalR: Real-Time Notification Hub

### Connection

```
Hub URL:   {VITE_GATEWAY_BASE_URL}/hubs/notifications
Auth:      Access token passed during connection negotiation (Bearer)
Transport: WebSockets (with fallback to Long Polling)
```

Managed by `useNotificationHub` composable from `@hivespace/shared`. The `useNotificationRealtime` composable exposes the reactive notification feed.

### Event: `ReceiveNotification`

The hub pushes a single event to connected clients:

```ts
// Event name
'ReceiveNotification'

// Payload shape (NotificationHubEvent)
interface NotificationHubEvent {
  id: string
  eventType: string       // e.g. "ORDER_STATUS_CHANGED", "COUPON_EXPIRING"
  payload: string         // JSON string — must be parsed by recipient
  createdAt: string       // ISO 8601 datetime
}
```

The `payload` field is a serialized JSON string. The receiver must `JSON.parse(payload)` and interpret it based on `eventType`.

### Usage Pattern

```ts
const { notifications } = useNotificationRealtime()
// notifications is a reactive ref<NotificationHubEvent[]>
// New events are prepended as they arrive
```

---

## Token Refresh Endpoint

This call is made **directly to the identity server** (not through the API Gateway). Managed by `createRefreshService()` from `@hivespace/shared`.

```
POST {VITE_AUTH_AUTHORITY_URL}/connect/token
Content-Type: application/x-www-form-urlencoded
```

Request body (form-encoded):

| Field | Value |
|-------|-------|
| `grant_type` | `refresh_token` |
| `client_id` | `{VITE_APP_CLIENT_ID}` |
| `refresh_token` | `{stored_refresh_token}` |
| `scope` | `{VITE_APP_SCOPE}` |

Response on success:

```json
{
  "access_token": "...",
  "refresh_token": "...",
  "id_token": "...",
  "expires_in": 3600,
  "token_type": "Bearer"
}
```

Refresh is triggered proactively when the access token has fewer than **60 seconds** remaining. On failure (e.g., refresh token expired or revoked), the user is redirected to the OIDC login flow.
