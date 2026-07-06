# Quickstart - Standardize System Money Handling

## Goal

Implement a reusable platform-configuration foundation in UserService, deliver currency as the first config type on that foundation, propagate currency state to commerce services for validation, and standardize money rendering across admin, seller, and buyer apps.

## Recommended implementation order

1. Backend ownership in `../hivespace.microservice`
   - UserService: add generic config-level metadata plus typed `PlatformCurrency` rows, admin/read APIs, and the currency configuration update event
   - CatalogService, OrderService, PaymentService: add consumer-side currency config refs and consumers
2. Backend domain behavior
   - Remove hard-coded `VND` defaults
   - Normalize coupon/product/payment/cart/order money contracts
   - Standardize coupon on one aggregate `CurrencyCode` plus smallest-unit amount fields
   - Replace ambiguous money factory usage with explicit smallest-unit vs major-unit creation paths
   - Add mixed-currency and disabled-currency guards
3. Frontend shared package in `../hivespace.web/packages/shared`
   - Add shared money types, formatter, and currency-aware money input utilities with default major-unit display/input for cent-backed `USD`/`EUR` amounts and an optional raw-smallest-unit override
4. Frontend apps
   - Admin configuration workspace with persisted currency rows, enable/disable toggles, and explicit default selection
   - Seller coupon/product/order surfaces
   - Buyer cart/checkout/order/payment/product surfaces
5. Docs and catalogs
   - Update affected service docs
   - Add API/event catalog rows after task generation/update-catalog flow

## Backend verification targets

From `../hivespace.microservice`:

```powershell
dotnet restore
dotnet build
.\\quality-gate.ps1 -Scope backend:UserService
.\\quality-gate.ps1 -Scope backend:CatalogService
.\\quality-gate.ps1 -Scope backend:OrderService
.\\quality-gate.ps1 -Scope backend:PaymentService
```

Focus tests on:

- config update validation
- projection consumer idempotency
- coupon currency consistency
- explicit smallest-unit vs major-unit money factory behavior
- mixed-currency cart/checkout rejection
- payment currency validation
- malformed historical money read handling

## Frontend verification targets

From `../hivespace.web`:

```powershell
pnpm --filter @hivespace/shared test
pnpm --filter @hivespace/admin type-check
pnpm --filter @hivespace/seller type-check
pnpm --filter @hivespace/buyer type-check
.\\coverage.ps1 -Workspace admin
.\\coverage.ps1 -Workspace seller
.\\coverage.ps1 -Workspace buyer
.\\coverage.ps1 -Workspace shared
```

Focus tests on:

- admin currency configuration load/save states
- seller coupon and product currency options
- shared formatter output for `VND`, `USD`, and `EUR`
- shared money input parsing for `VND`, `USD`, and `EUR`
- invalid-money placeholder rendering
- buyer cart/checkout/order/payment surfaces using the shared formatter

## Non-goals

- No exchange-rate conversion
- No new saga state machine
- No required updates to `../hivespace.config`
- No migration/backfill work; update dev seed and fixture data instead
- No active payment-method/provider behavior changes in this feature; only the shared foundation is prepared for those future config types
