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
| `RequireCatalogImportProvisioning` | System admin or CatalogService internal provisioning client |
| `RequireAdminOrUser` | Admin or normal authenticated user |

## Public Route Ownership

| Gateway path | Owner |
|---|---|
| `/.well-known/**` | IdentityService direct authority endpoint, not routed through ApiGateway |
| `/connect/**` | IdentityService direct authority endpoint, not routed through ApiGateway |
| `/Account/**` | Not a supported HiveSpace contract after legacy page removal; old user-facing account URLs may return not-found/error and must not render account pages or compatibility redirects |
| `/api/v1/accounts/**` | IdentityService |
| `/api/v1/admins/**` | IdentityService, UserService, and CatalogService split by action |
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

IdentityServer public OIDC protocol endpoints are served directly by IdentityService on `http://localhost:5001`: `/.well-known/**` and `/connect/**`. Legacy `/Account/**` URLs are not ApiGateway routes and old HiveSpace user-facing IdentityService account pages are not part of the target contract. Because HiveSpace is still in development, old user-facing account URL compatibility redirects are removed instead of preserved. `/identity/**` and `/api/v1/identity/**` are intentionally not part of the target contract.

### Account and Email Verification

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/accounts/login` | Anonymous | Password login from frontend-owned UI; sets secure HttpOnly access/refresh token cookies and CSRF token |
| POST | `/api/v1/accounts/register` | Anonymous | Frontend-owned public account registration where allowed; creates only a pending identity account, sends verification email, and returns a non-authenticated confirmation result without issuing token cookies/browser session |
| POST | `/api/v1/accounts/session/refresh` | Session cookie + CSRF | Bootstrap after reload or refresh/rotate the browser session through the gateway |
| POST | `/api/v1/accounts/logout` | Session cookie + CSRF | Clear token cookies and CSRF cookie |
| POST | `/api/v1/accounts/email-verification` | `RequireAdminOrUser` | Send verification email |
| POST | `/api/v1/accounts/email-verification/resend` | Anonymous | Request another verification email for pending accounts with generic success semantics, cooldown protection, and `204 No Content` success |
| POST | `/api/v1/accounts/email-verification/verify` | Anonymous | Verify email token and return `204 No Content` on success |
| POST | `/api/v1/accounts/otp/request` | Anonymous | Request a one-time sign-in code sent to an email address; always returns generic `200 OK` with challenge token, expiry timestamp, and cooldown timestamp regardless of account existence or cooldown state |
| POST | `/api/v1/accounts/otp/verify` | Anonymous | Verify OTP code using opaque challenge token; on success issues a browser session (HttpOnly cookies + CSRF token) identical to password sign-in and returns a same-origin redirect URL |
| GET | `/api/v1/accounts/external/google/challenge` | Anonymous | Start Google from buyer/seller sign-in or sign-up context and preserve a safe return URL |
| GET | `/api/v1/accounts/external/google/complete` | Anonymous | Complete Google sign-in, create/sign in a Google-linked normal user account only when no same-email password account exists, or redirect to required frontend account-link confirmation |
| POST | `/api/v1/accounts/external/google/link` | Temporary Google link state + CSRF/link token | Confirm consent and existing account password, link Google, mark verified matching email, and set browser session cookies |
| DELETE | `/api/v1/accounts/external/google/link` | Temporary Google link state + CSRF/link token | Cancel pending Google account linking, clear temporary link state, and create no duplicate same-email account |

Browser session endpoints are served through ApiGateway under `/api/v1/accounts/**`. Successful login and refresh responses must not expose access or refresh tokens to browser scripts. IdentityService stores token material only in secure HttpOnly cookies, and ApiGateway forwards the access-token cookie value as downstream bearer authorization. Cookie-authenticated state-changing browser requests require a server-issued CSRF token in a custom header before downstream state changes. Email/password registration no longer creates a browser session; it returns a pending-verification result for the frontend instead.

Where account-related flows fail through exception handling, they continue to use the existing service exception response convention. Registration returns a plain data-only verification-sent success payload without a server-supplied `message` field and does not issue a browser session.

Google sign-in is buyer/seller only. New Google-authenticated users are normal user accounts only when no same-email local password account exists; seller access still requires existing seller onboarding. If a verified Google email matches an existing unlinked local password account, the user must link, use password sign-in/reset, or choose another Google account. Admin accounts must not be created or signed in through Google.

### Admin Identity Management

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/admins` | `RequireAdmin` | Create admin account |
| GET | `/api/v1/admins` | `RequireAdmin` | List admin accounts |
| PUT | `/api/v1/admins/users/status` | `RequireAdmin` | Update identity-owned user/admin account status |
| DELETE | `/api/v1/admins/users/{userId}` | `RequireAdmin` | Delete or deactivate identity-owned account access |
| POST | `/api/v1/admins/imported-seller-accounts` | `RequireCatalogImportProvisioning` | Create or match an identity-owned seller account for a catalog import seller without issuing browser session cookies, access tokens, refresh tokens, or caller-visible passwords |

## UserService

### Profile and Settings

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/users/me` | `RequireAdminOrUser` | Get authenticated user profile, including nullable avatar URL |
| PUT | `/api/v1/users/me` | `RequireAdminOrUser` | Update authenticated user profile, including optional avatar file ID |
| GET | `/api/v1/users/settings` | `RequireAdminOrUser` | Get locale/theme/user settings |
| PUT | `/api/v1/users/settings` | `RequireAdminOrUser` | Update locale/theme/user settings |

### Platform Configuration

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/users/platform-currency-policy` | `RequireAdminOrUser` | Get the current enabled/default platform currency policy for authenticated admin, seller, and buyer clients |

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
| POST | `/api/v1/admins/imported-seller-stores` | `RequireCatalogImportProvisioning` | Create or match a UserService-owned store for an imported seller account while preserving existing store uniqueness and seller-role propagation rules |

### Admin Configuration

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/admins/configuration/currencies` | `RequireAdmin` | Get persisted platform currency configuration, including enabled currencies, default currency, and policy version |
| PUT | `/api/v1/admins/configuration/currencies` | `RequireAdmin` | Update enabled/default platform currencies; reject disabling the current default without saving a replacement default in the same request |

Admin profile/store review APIs that do not change credentials, roles, lockout, email verification, or account status remain UserService-owned.

## CatalogService

### Admin Catalog Imports

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/admins/catalog-imports/categories/provisioning` | `RequireAdmin` | Queue a CatalogService-owned async job to create or match categories from crawled Tiki category data before product crawling and store external category links; returns `202 Accepted` with job ID |
| POST | `/api/v1/admins/catalog-imports/categories/attributes/provisioning` | `RequireAdmin` | Queue a CatalogService-owned async job to create or match category-scoped attribute definitions and selectable values from crawled Tiki category-attribute data; returns `202 Accepted` with job ID |
| POST | `/api/v1/admins/catalog-imports/bundles` | `RequireAdmin` | Queue a CatalogService-owned async job to persist a Python-generated Tiki import bundle and prepare it for CatalogService-owned validation; returns `202 Accepted` with job ID |
| GET | `/api/v1/admins/catalog-imports/bundles` | `RequireAdmin` | List paginated catalog import bundles with source metadata, source file name where available, validation status, and summary counts |
| GET | `/api/v1/admins/catalog-imports/bundles/{bundleId}` | `RequireAdmin` | Get catalog import bundle metadata and summary for the detail page |
| GET | `/api/v1/admins/catalog-imports/bundles/{bundleId}/category-links` | `RequireAdmin` | List paginated provisioned external category links for a catalog import bundle |
| POST | `/api/v1/admins/catalog-imports/bundles/{bundleId}/category-links/{externalCategoryId}/mapping` | `RequireAdmin` | Map an imported category link to an existing active HiveSpace category so `UnprovisionedCategory` validation blockers can be cleared after revalidation |
| GET | `/api/v1/admins/catalog-imports/bundles/{bundleId}/sellers` | `RequireAdmin` | List paginated imported sellers with ownership status, conflict reason, and existing-store candidates where available |
| GET | `/api/v1/admins/catalog-imports/bundles/{bundleId}/products` | `RequireAdmin` | List paginated imported products with SKU summaries, category traceability, seller traceability, readiness status, and import status |
| GET | `/api/v1/admins/catalog-imports/bundles/{bundleId}/duplicate-groups` | `RequireAdmin` | List paginated duplicate product groups and representative product state for a catalog import bundle |
| GET | `/api/v1/admins/catalog-imports/bundles/{bundleId}/validation-issues` | `RequireAdmin` | List paginated blocking and warning validation issues for a catalog import bundle, including optional metadata such as missing external category IDs |
| POST | `/api/v1/admins/catalog-imports/bundles/{bundleId}/validate` | `RequireAdmin` | Queue a CatalogService-owned async job to re-run catalog import validation after category provisioning or seller provisioning state changes; returns `202 Accepted` with job ID |
| POST | `/api/v1/admins/catalog-imports/bundles/{bundleId}/seller-provisioning` | `RequireAdmin` | Queue a CatalogService-owned async job for idempotent imported seller account and store provisioning through IdentityService and UserService ownership boundaries; returns `202 Accepted` with job ID |
| POST | `/api/v1/admins/catalog-imports/bundles/{bundleId}/sellers/{importedSellerId}/ownership-link` | `RequireAdmin` | Approve linking a conflicted imported Tiki seller to an existing eligible HiveSpace seller account/store without overwriting existing store or product data |
| POST | `/api/v1/admins/catalog-imports/bundles/{bundleId}/import` | `RequireAdmin` | Queue a CatalogService-owned async job to import ready bundle products with operator-selected `Draft`, `Unpublish`, or `Available` publication state; returns `202 Accepted` with job ID |
| GET | `/api/v1/admins/catalog-imports/jobs` | `RequireAdmin` | List paginated all-operation CatalogService catalog import job history with job ID, source file name where available, operation type, requested/completed times, status, linked bundle, progress counts, and result/error summary |
| GET | `/api/v1/admins/catalog-imports/jobs/{jobId}` | `RequireAdmin` | Get CatalogService-owned catalog import job status, progress counts, final result summary, and error summary |
| POST | `/api/v1/admins/catalog-imports/jobs/{jobId}/retry` | `RequireAdmin` | Optionally retry an idempotent failed or retryable completed catalog import job; returns `202 Accepted` with job ID |

### Seller Products

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/products` | `RequireSeller` | Create product with explicit price currency validated against the enabled platform currency policy |
| GET | `/api/v1/products` | `RequireSeller` | List seller products |
| GET | `/api/v1/products/{id}` | `RequireSeller` | Get seller product detail with explicit money metadata for SKU prices and invalid-money diagnostics when needed |
| PUT | `/api/v1/products/{id}` | `RequireSeller` | Update product with explicit price currency validated against the enabled platform currency policy |
| DELETE | `/api/v1/products/{id}` | `RequireSeller` | Delete or deactivate product |

### Storefront Catalog

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/products/summaries` | Anonymous | Search/list storefront product summaries |
| GET | `/api/v1/products/detail/{id}` | Anonymous | Get storefront product detail with SKUs, explicit money metadata, and invalid-money diagnostics when needed |
| GET | `/api/v1/categories` | Anonymous | Get category tree |
| GET | `/api/v1/categories/homepage` | Anonymous | Get homepage categories |
| GET | `/api/v1/categories/{id}/attributes` | Anonymous | Get category attribute definitions |

## OrderService

### Cart

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/carts/summary` | `RequireUser` | Get cart summary for checkout with explicit money metadata and reject mixed-currency cart state |
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
| POST | `/api/v1/orders/checkout/preview` | `Authorize` | Preview checkout totals with explicit money metadata and reject mixed or invalid currency calculation contexts |
| POST | `/api/v1/orders/checkout` | `Authorize` | Start checkout saga with a canonical payment method code and one checkout-level payment covering all generated orders when cart, coupon, and payment currency state is enabled and internally consistent |
| GET | `/api/v1/orders` | `Authorize` | List buyer orders |
| GET | `/api/v1/orders/{orderId}` | `Authorize` | Get order detail with explicit order money metadata, order code, linked checkout payment reference, and invalid-money diagnostics when needed |
| GET | `/api/v1/orders/seller` | `RequireSeller` | List seller orders |
| POST | `/api/v1/orders/{orderId}/confirm` | `RequireSeller` | Seller confirms order |
| POST | `/api/v1/orders/{orderId}/reject` | `RequireSeller` | Seller rejects order |

### Coupons

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/coupons/available` | `RequireUser` | List available coupons for cart/store/products |
| POST | `/api/v1/coupons` | `RequireSeller` | Create coupon using one canonical `currencyCode` for all coupon money fields and reject disabled currencies |
| GET | `/api/v1/coupons` | `RequireSeller` | List seller coupons with normalized coupon money metadata under one canonical `currencyCode` |
| GET | `/api/v1/coupons/{id}` | `RequireSeller` | Get coupon detail with normalized coupon money metadata under one canonical `currencyCode` |
| PUT | `/api/v1/coupons/{id}` | `RequireSeller` | Update coupon while preserving one canonical `currencyCode` across all coupon money fields |
| DELETE | `/api/v1/coupons/{id}` | `RequireSeller` | Delete or deactivate coupon |
| POST | `/api/v1/coupons/{id}/end` | `RequireSeller` | End coupon early |

## PaymentService

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/payments/methods` | `Authorize` | Get PaymentService-owned canonical payment method metadata for COD, VNPay, and future/unavailable Stripe across buyer, seller, and admin apps |
| GET | `/api/v1/payments/vnpay/return` | Anonymous | VNPay browser return endpoint |
| GET | `/api/v1/payments/webhook/{gateway}` | Anonymous | Payment gateway webhook/IPN |
| GET | `/api/v1/payments/{paymentId}` | `Authorize` | Get checkout-level payment detail with payment reference number, linked orders, canonical method metadata, explicit payment money metadata, latest attempt, and privileged attempt history when authorized |
| GET | `/api/v1/payments/by-reference/{referenceNo}` | `Authorize` | Get checkout-level payment detail by public `PAY-{ULID}` reference for support/admin reconciliation with linked orders, latest attempt, attempt history when authorized, and explicit money metadata |
| GET | `/api/v1/payments/by-order/{orderId}` | `Authorize` | Get the shared checkout-level payment linked to an order with linked orders, latest attempt, explicit payment money metadata, and invalid-money diagnostics when needed |
| POST | `/api/v1/payments/{paymentId}/attempts` | `Authorize` | Create an idempotent COD or VNPay retry attempt under an existing checkout-level payment after a failed, expired, or cancelled attempt while linked orders remain eligible and no attempt has succeeded |
| GET | `/api/v1/wallets/me` | `Authorize` | Get current wallet balance |
| GET | `/api/v1/wallets/me/transactions` | `Authorize` | List wallet transactions |

PaymentService owns canonical payment method configuration. COD and VNPay are checkout-selectable in this feature; Stripe is returned as unavailable/future metadata for non-checkout displays until its gateway implementation is explicitly enabled later. VNPay initiation uses payment `ReferenceNo` as the merchant transaction reference, while gateway transaction identifiers are stored separately on payment attempts after return/IPN. Webhook endpoints should acknowledge gateway delivery even when internal processing is deferred or logged.

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
