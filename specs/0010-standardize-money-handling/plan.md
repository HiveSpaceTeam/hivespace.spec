# Implementation Plan: Standardize System Money Handling

**Branch:** `0010-standardize-money-handling`
**Date:** 2026-06-29
**Spec:** [spec.md](./spec.md)

> Before writing anything in this file, read `.specify/memory/constitution.md`

---

## Phase 0 - Research

### Existing context (read before planning)

- [x] `services/user-service/README.md` - confirm UserService can own the platform configuration foundation
- [x] `services/catalog-service/README.md` - confirm CatalogService owns product pricing and product read models
- [x] `services/order-service/README.md` and `services/order-service/workflows.md` - confirm OrderService owns coupons, carts, checkout, and saga orchestration
- [x] `services/payment-service/README.md` and `services/payment-service/workflows.md` - confirm PaymentService owns payment amount validation
- [x] `services/api-gateway/README.md` - verify no new gateway prefix is required
- [x] `shared/event-catalog.md` - verify no existing currency-configuration event already exists
- [x] `shared/api-catalog.md` - verify no existing currency-configuration endpoint already exists
- [x] Source repos - confirm current hard-coded `VND` assumptions and scattered frontend money formatting

### Technical unknowns

- How should a UserService-owned currency configuration be distributed to CatalogService, OrderService, and PaymentService without direct database reads?
- Should this feature introduce or change a MassTransit saga state machine?
- How should malformed or unsupported stored currencies remain readable without guessing a replacement currency?
- Where should frontend money formatting live so admin, seller, and buyer apps use one rule?
- How should seller create/edit flows obtain enabled currency options without duplicating policy state?

### Research notes

- Resolved in [research.md](./research.md).

---

## Phase 1 - Architecture & Data Model

### Technical context

| Area | Decision |
| --- | --- |
| Policy owner | UserService owns the persisted platform configuration foundation and the admin configuration API |
| Distribution model | UserService publishes `PlatformCurrencyPolicyUpdatedIntegrationEvent`; CatalogService, OrderService, and PaymentService maintain local `PlatformCurrencyConfigRef` models for validation |
| Supported currencies | `VND`, `USD`, `EUR` only for this feature |
| Conversion | No exchange-rate conversion; mixed-currency operations are rejected |
| Projection consistency | Commerce services fail closed for new or updated money-bearing writes when the required local currency-policy projection is missing, unreadable, or stale for validation |
| Money read contract | Standardize API read models on explicit money metadata using smallest-unit amount, ISO currency code, and invalid-money diagnostics where needed |
| Default currency semantics | Default currency is a client-visible preselection for admin/seller configuration or authoring flows, never a backend fallback for omitted money currency |
| Frontend formatting/input | Add shared `@hivespace/shared` money-formatting and money-input utilities, default cent-backed `USD`/`EUR` values to major-unit display/input, and replace app-local formatters plus generic money input handling |
| Coupon storage model | Standardize coupon on one canonical aggregate `CurrencyCode` plus smallest-unit amount fields for related coupon money values |
| Currency persistence model | Persist one `PlatformCurrency` record per supported currency plus a small `PlatformConfig` record for default/version state |
| Unknowns | None remain; no `NEEDS CLARIFICATION` items remain after Phase 0 |

### Constitution check

| Check | Status | Notes |
| --- | --- | --- |
| Service ownership follows constitution | Pass | UserService owns the config foundation; commerce services keep their own validation and data |
| No direct cross-service database reads | Pass | Policy distribution uses integration events and local projections |
| Money stored as smallest-unit `long` with currency | Pass | Existing value object remains; feature removes implicit `VND` fallback behavior |
| New public endpoints recorded later in catalog | Pass | Plan identifies required API changes for later catalog/task updates |
| New integration events recorded later in catalog | Pass | Plan identifies required event changes for later catalog/task updates |
| No `hivespace.config` dependency | Pass | Runtime/config work stays in backend/frontend repos only |

### Service placement

UserService owns the feature because the clarified requirement assigns platform configuration governance and admin configuration persistence to UserService. CatalogService, OrderService, and PaymentService consume the currency configuration to validate their own domain operations but do not own that configuration themselves.

| Service | Classification | Reason | Documentation/catalog action |
| ------- | -------------- | ------ | ---------------------------- |
| UserService | Owning service | Owns the persisted platform configuration foundation, the currency config type, admin management API, and read-only authenticated query | Update service docs; add new/changed API rows; add new config event row |
| CatalogService | Changed supporting service | Product and SKU pricing must stop assuming `VND`, validate enabled currencies, and return normalized money metadata | Update service docs; update changed product endpoint rows if response contracts change |
| OrderService | Changed supporting service | Coupon money fields, cart/coupon guards, checkout preview, checkout initiation, and saga request payloads must reject mixed/disabled currency states | Update service docs; update changed coupon/order/cart endpoint rows; add config event consumer row if event catalog tracks consumer set change |
| PaymentService | Changed supporting service | Payment creation and return models must validate and preserve order currency without `VND` fallback | Update service docs; update changed payment endpoint rows if response contracts change |
| ApiGateway | Reused supporting service | Existing `/api/v1/users/**` and `/api/v1/admins/**` ownership covers the new endpoints; no new gateway prefix or business logic | Verification only; no doc rewrite unless route ownership table changes materially |

### New aggregates and value objects

1. `PlatformConfig` aggregate in UserService
   - Single config-level settings record for one config type
   - Field: `ConfigType = currency` for this feature
   - Fields: `DefaultCurrency`, `Version`, audit timestamps
   - Behavior: change default currency, validate default remains enabled, coordinate concurrency/versioning for the currency set
   - EF table: `platform_configs`
   - Seed/bootstrap only in this feature; no migration/backfill work required in dev phase

2. `PlatformCurrency` aggregate/member record in UserService
   - One record per supported currency
   - Fields: `CurrencyCode`, `IsEnabled`, optional display/sort metadata
   - Used to persist the enable/disable state cleanly and allow future currencies without reshaping the policy record

3. `PlatformCurrencyConfigRef` in CatalogService, OrderService, and PaymentService
   - Stores current enabled/default currencies plus version from integration event
   - Used for synchronous domain validation without cross-service calls
   - If missing, unreadable, or stale for a write validation path, the service rejects the write instead of guessing or accepting with fallback state

4. `MoneyIssue` read-model metadata
   - Not a persisted aggregate
   - Captures `isValid`, `issueCode`, and optional `displayPlaceholder`
   - Used only when a read model encounters missing or unsupported currency data

5. `CouponCurrencyContext` in OrderService
   - Single canonical `CurrencyCode` stored on the coupon aggregate
   - Governs `DiscountAmount`, `MaxDiscountAmount`, and `MinOrderAmount`
   - Prevents repeated persisted currency fields across coupon money values

6. `MoneyInputModel` in shared frontend
   - Currency-aware major-unit input for `USD`/`EUR`
   - Whole-unit input for `VND`
   - Produces smallest-unit payload values for write APIs

### New repository interfaces

- UserService:
  - `IPlatformConfigRepository`
- CatalogService:
  - `IPlatformCurrencyConfigRefRepository`
- OrderService:
  - `IPlatformCurrencyConfigRefRepository`
- PaymentService:
  - `IPlatformCurrencyConfigRefRepository`

New money factory/API direction:

- Replace ambiguous `Money.Create(long amount, string currencyCode)` usage in feature-touched flows with explicit factories for:
  - smallest-unit integer creation
  - major-unit decimal creation

### Integration events (must go through Outbox)

| Event | Topic/Exchange | Producer | Consumer(s) |
| --- | --- | --- | --- |
| `PlatformCurrencyPolicyUpdatedIntegrationEvent` | `platform.currency-policy.updated` | user-service | catalog-service, order-service, payment-service |

Reused unchanged existing contracts:

- Checkout and fulfillment saga messages remain structurally unchanged unless payment/order request payloads need normalized currency code fields. The feature does not add a new saga.
- Existing product, order, and payment domain events continue to express service-owned facts; no currency conversion event is introduced.

### API endpoints

New endpoints:

| Method | Path | Auth | Handler |
| --- | --- | --- | --- |
| GET | `/api/v1/admins/configuration/currencies` | `RequireAdmin` | `GetPlatformCurrencyConfigHandler` |
| PUT | `/api/v1/admins/configuration/currencies` | `RequireAdmin` | `UpdatePlatformCurrencyConfigHandler` |
| GET | `/api/v1/users/platform-currency-policy` | `RequireAdminOrUser` | `GetActivePlatformCurrencyConfigHandler` |

Changed existing endpoints:

| Method | Path | Auth | Change |
| --- | --- | --- | --- |
| POST | `/api/v1/products` | `RequireSeller` | Validate currency against enabled policy |
| PUT | `/api/v1/products/{id}` | `RequireSeller` | Validate and persist explicit price currency |
| GET | `/api/v1/products/{id}` | `RequireSeller` | Return normalized money metadata for SKU prices |
| GET | `/api/v1/products/detail/{id}` | Anonymous | Return normalized money metadata for storefront prices |
| POST | `/api/v1/coupons` | `RequireSeller` | Validate `DiscountAmount`, `MaxDiscountAmount`, and `MinOrderAmount` against one explicit currency |
| PUT | `/api/v1/coupons/{id}` | `RequireSeller` | Preserve/validate coupon currency consistency |
| GET | `/api/v1/coupons` | `RequireSeller` | Return normalized coupon money metadata |
| GET | `/api/v1/coupons/{id}` | `RequireSeller` | Return normalized coupon money metadata |
| POST | `/api/v1/carts/summary` | `RequireUser` | Reject mixed-currency cart state and expose explicit currency metadata |
| POST | `/api/v1/orders/checkout/preview` | `Authorize` | Reject mixed/invalid currency calculation contexts |
| POST | `/api/v1/orders/checkout` | `Authorize` | Block mixed/disabled currency checkout initiation |
| GET | `/api/v1/orders/{orderId}` | `Authorize` | Return explicit order money metadata and invalid-money placeholders where needed |
| GET | `/api/v1/payments/{paymentId}` | `Authorize` | Return explicit payment currency metadata |
| GET | `/api/v1/payments/by-order/{orderId}` | `Authorize` | Return explicit payment currency metadata |

Plans must treat existing `/api/v1/admins/**`, `/api/v1/users/**`, `/api/v1/products/**`, `/api/v1/coupons/**`, `/api/v1/orders/**`, and `/api/v1/payments/**` prefixes as reused ownership; no new gateway prefix is required.

Coupon contract normalization in this feature:

- Write requests use one top-level `CurrencyCode` plus smallest-unit integer amount fields.
- Read responses use one canonical coupon `CurrencyCode` instead of repeating per-field coupon currency values.
- `defaultCurrencyCode` supports client preselection only; omitted currency fields remain validation errors on backend writes.

---

## Phase 2 - Implementation Plan

### Backend implementation plan

#### UserService

**Domain layer**

- [ ] Add `PlatformConfig` aggregate with guarded default/version transitions
- [ ] Add `PlatformCurrency` entity/aggregate record with per-currency enabled state
- [ ] Add domain validation for supported currency whitelist (`VND`, `USD`, `EUR`)
- [ ] Add domain event for policy updates if the service uses domain-event-to-integration-event mapping

**Application layer**

- [ ] `GetPlatformCurrencyConfigQuery` + handler for admin workspace
- [ ] `UpdatePlatformCurrencyConfigCommand` + handler + validator
- [ ] `GetActivePlatformCurrencyConfigQuery` + handler for authenticated cross-app reads
- [ ] DTOs for enabled/default currencies and optimistic version metadata

**Infrastructure layer**

- [ ] EF configuration for `platform_configs`
- [ ] EF configuration for `platform_currencies`
- [ ] Repository implementation
- [ ] Service-owned publisher abstraction for `PlatformCurrencyPolicyUpdatedIntegrationEvent`
- [ ] Seed/bootstrap logic so the platform has an initial `currency` config with default `VND` and `USD`/`EUR` disabled unless already configured

**API layer**

- [ ] Minimal API endpoints under existing admin/user route groups
- [ ] Admin authorization on update endpoint
- [ ] Concurrency/error handling for default-currency disable attempts

#### CatalogService

**Domain/application**

- [ ] Replace any `Money.FromVND(...)` or equivalent defaulting in product create/update flows with explicit currency-aware creation
- [ ] Normalize seller/storefront product DTOs to return explicit currency code and smallest-unit amount
- [ ] Validate new and updated SKU prices against local `PlatformCurrencyConfigRef`
- [ ] Reject product writes when the required local currency-policy projection is missing, unreadable, or stale for validation
- [ ] Keep malformed historical records readable via invalid-money response metadata rather than inferred `VND`

**Infrastructure**

- [ ] Add `PlatformCurrencyPolicyUpdatedIntegrationEvent` consumer with idempotent `PlatformCurrencyConfigRef` update
- [ ] Add `PlatformCurrencyConfigRef` entity/table if required
- [ ] Update seed data and fixture helpers so non-VND pricing is possible without breaking current data
- [ ] Make projection freshness observable so policy-sync failures can be diagnosed without allowing fallback writes

#### OrderService

**Domain/application**

- [ ] Add one canonical coupon `CurrencyCode` on the aggregate and stop persisting repeated currency fields for `DiscountAmount`, `MaxDiscountAmount`, and `MinOrderAmount`
- [ ] Update coupon aggregate/application flows so the coupon currency governs every related coupon money amount
- [ ] Replace ambiguous `Money.Create(long, string)` calls in coupon/cart/checkout paths with explicit smallest-unit money factories
- [ ] Remove `VND` fallback defaults from cart summary, checkout preview, and coupon evaluation paths
- [ ] Add guards that reject mixed-currency carts, coupon application, checkout preview, and checkout initiation
- [ ] Reject coupon/cart/checkout writes when the required local currency-policy projection is missing, unreadable, or stale for validation
- [ ] Preserve readable historical data by returning invalid-money diagnostics for malformed records

**Infrastructure**

- [ ] Add `PlatformCurrencyPolicyUpdatedIntegrationEvent` consumer with local `PlatformCurrencyConfigRef` update
- [ ] Update saga request payload creation so checkout/payment handoff keeps the order currency explicitly
- [ ] Update repository projections or SQL read models that currently infer missing currency as `VND`
- [ ] Update seed data and dev fixtures to match the new coupon storage model; do not add migration/backfill work in this feature
- [ ] Make policy-projection freshness/version visible enough to diagnose stale validation state during admin policy changes

#### PaymentService

**Domain/application**

- [ ] Validate initiated payment currency against local `PlatformCurrencyConfigRef`
- [ ] Remove implicit `VND` fallback in payment read models and gateway result mapping
- [ ] Reject payment initiation when the required local currency-policy projection is missing, unreadable, or stale for validation
- [ ] Preserve readable historical payment records with invalid-money diagnostics when needed

**Infrastructure**

- [ ] Add `PlatformCurrencyPolicyUpdatedIntegrationEvent` consumer with local `PlatformCurrencyConfigRef` update
- [ ] Update seed data and fixtures that currently assume `VND`
- [ ] Make projection freshness observable so stale currency-policy validation can be diagnosed without accepting fallback writes

### Saga design

No saga required.

Reason: the feature changes validation and payload consistency inside existing checkout/payment flows but does not add or change MassTransit saga states, participants, compensation branches, or timeout behavior. Existing checkout and fulfillment state machines remain in place and only receive cleaner currency guards before or during their existing requests.

### Architecture decision

ADR required: [ADR-0010-user-service-platform-currency-policy.md](../../architecture/decisions/ADR-0010-user-service-platform-currency-policy.md)

Reason: the feature makes a non-obvious cross-service data-ownership decision by placing the platform configuration foundation in UserService while distributing currency validation state to other commerce services through integration events and local projections.

---

## Phase 3 - Frontend Plan

### Surfaces

- [x] admin
- [x] seller
- [x] buyer
- [x] shared package

### Shared package work (mandatory first)

Files to add or extend in `../hivespace.web/packages/shared/src`:

| File | Notes |
| --- | --- |
| `types/money.types.ts` | Shared `MoneyDisplay`, `MoneyIssue`, and currency-config contracts |
| `utils/format-money.ts` or `composables/useMoneyFormatter.ts` | One money-formatting rule for `VND`, `USD`, `EUR`, plus invalid-money placeholder handling; default cent-backed `USD`/`EUR` values to major-unit display and optionally allow raw-smallest-unit output |
| `composables/useMoneyInput.ts` or equivalent | Currency-aware money input parsing/formatting that accepts major-unit `USD`/`EUR` entry, whole-unit `VND` entry, and emits smallest-unit payloads |
| `internal.ts` exports | Re-export shared money helpers/types for all apps |
| Shared i18n locale files | Add invalid-money placeholder and currency display labels if shared text is needed |

### Admin app

Files to add or extend:

| File | Notes |
| --- | --- |
| `src/types/configuration.types.ts` | Currency configuration request/response contracts on the generic foundation |
| `src/services/configuration.service.ts` | Load/save currency configuration through UserService admin endpoints |
| `src/stores/configuration.store.ts` | Manage persisted config data, optimistic save state, and validation errors |
| `src/pages/Configuration/ConfigurationPage.vue` | Replace mock currency chips with persisted currency-record rows/toggles and an explicit default-currency selector |
| `src/pages/Buyers/BuyersPage.vue` and other discovered admin money-bearing pages | Replace hard-coded or locale-only money formatting with the shared formatter while preserving existing page ownership |
| `src/i18n/locales/en/configuration.json` | Add currency-policy management copy |
| `src/i18n/locales/vi/configuration.json` | Keep Vietnamese translations in sync |

### Seller app

Files to add or extend:

| File | Notes |
| --- | --- |
| `src/types/coupon.types.ts` | Normalize coupon money contracts and validity metadata |
| `src/types/product.types.ts` | Normalize product/SKU money contracts |
| `src/services/configuration.service.ts` or shared config adapter | Fetch active currency configuration for forms if needed |
| `src/stores/coupon.store.ts` | Consume normalized coupon money metadata |
| `src/pages/Marketing/CouponDetailPage.vue` | Replace local symbol logic and generic number formatters with shared formatter/input and policy-driven currency options |
| `src/pages/Marketing/CouponListPage.vue` | Replace manual `formatMoney` helper |
| `src/composables/useCouponValidation.ts` | Remove hard-coded symbol/locale assumptions |
| `src/pages/Products/ProductListPage.vue` | Replace `USD`-fixed formatter |
| `src/pages/Products/UpsertProductPage.vue` | Use enabled currency options and shared money input for explicit price currency |
| `src/pages/Orders/OrderManagementPage.vue` | Replace `vi-VN` plus hard-coded dong-symbol formatting |
| `src/i18n/locales/en/*.json` | Update feature-owned copy where needed |
| `src/i18n/locales/vi/*.json` | Update matching Vietnamese copy |

### Buyer app

Files to add or extend:

| File | Notes |
| --- | --- |
| `src/types/cart.types.ts` | Preserve explicit currency and invalid-money metadata |
| `src/types/checkout.types.ts` | Preserve explicit currency and invalid-money metadata |
| `src/types/order.types.ts` | Preserve explicit currency and invalid-money metadata |
| `src/types/payment.types.ts` | Preserve explicit currency and invalid-money metadata |
| `src/types/product.types.ts` | Normalize storefront price contract away from numeric currency enum assumptions |
| `src/pages/Product/ProductDetailPage.vue` | Replace local price formatting logic with shared formatter |
| `src/pages/Cart/CartPage.vue` | Replace fixed `VND` formatter |
| `src/pages/Checkout/CheckoutPage.vue` | Replace fixed `VND` formatter and surface mixed-currency errors |
| `src/pages/Payment/PaymentResultPage.vue` | Replace fallback `VND` formatter |
| `src/pages/Profile/OrdersPage.vue` | Replace local `vi-VN` plus hard-coded dong-symbol formatting |
| `src/pages/Account/OrderDetailPage.vue` | Replace local price formatter |
| `src/components/common/AvailableCouponPopover.vue` | Replace fixed `VND` formatter |
| `src/components/home/ProductCard.vue` and `src/components/home/FlashSale.vue` | Replace locale-only formatting with shared money formatter |

### i18n keys to add

- `configuration.localization.currencies.*` for admin policy controls and validation
- `configuration.localization.defaultCurrency` and related helper/error text for explicit default selection
- Shared invalid-money placeholder key(s) such as `common.money.invalid`
- Admin feature-owned money-surface copy for invalid-money or policy-sync failure states where surfaced
- Feature-owned backend/business error labels for disabled currency and mixed-currency rejection paths

### Frontend verification intent

- Standardize all currently shipped money surfaces across admin, seller, and buyer apps
- Ensure admin money-bearing pages beyond the configuration workspace also use the shared formatter
- Ensure the same `VND`, `USD`, and `EUR` values render consistently everywhere
- Ensure cent-backed `USD` and `EUR` values render as dollars/euros by default rather than raw cents
- Ensure shared frontend money inputs accept major-unit `USD`/`EUR` entry and submit smallest-unit amounts correctly
- Ensure invalid/malformed currency values show the explicit placeholder instead of guessed formatting
- Ensure admin policy changes that have not yet reached a consumer projection surface an explicit failure instead of allowing fallback writes

---

## Post-Design Constitution Check

| Check | Status | Notes |
| --- | --- | --- |
| Owning service remains single source of truth | Pass | UserService owns the config foundation; other services only project and validate |
| Outbox/event rules preserved | Pass | New config event is published through the service-owned publisher/outbox path |
| No new saga introduced unnecessarily | Pass | Existing saga behavior only receives validation updates |
| Frontend shared-first rule followed | Pass | Plan introduces shared formatter/input helpers before app-local changes |
| English and Vietnamese translations updated together | Pass | Admin/seller/buyer work explicitly includes paired i18n updates |
| Config repo excluded from implementation scope | Pass | No `../hivespace.config` work required |

## Implementation sequencing summary

1. UserService config foundation, currency rows, admin/read APIs, and integration event
2. Currency config `Ref` consumers in CatalogService, OrderService, and PaymentService
3. Product, coupon, cart, checkout, order, and payment contract normalization
4. Shared frontend money formatter/input/types
5. Admin configuration UI
6. Seller and buyer surface standardization
7. Catalog and service doc updates during implementation/task execution
