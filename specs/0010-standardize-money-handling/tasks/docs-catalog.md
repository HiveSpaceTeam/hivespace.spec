# Docs and Catalog Tasks

## Documentation Scope

- Editable service docs: `services/user-service/README.md`, `services/catalog-service/README.md`, `services/order-service/README.md`, `services/order-service/workflows.md`, `services/payment-service/README.md`, `services/payment-service/workflows.md`
- Editable shared catalogs/docs: `shared/api-catalog.md`, `shared/event-catalog.md`, `architecture/decisions/ADR-0010-user-service-platform-currency-policy.md`
- Verification-only supporting docs: `services/api-gateway/README.md` because the feature reuses existing route ownership and does not add a new gateway prefix

## Service Docs

### Update

- [ ] D001 [US1] Update UserService documentation for platform configuration foundation ownership
  - File: `services/user-service/README.md`
  - Document: generic platform configuration foundation ownership, currency as the first config type, admin/authenticated read endpoints, persistence rules, supported currencies, default-currency disable guard, and the currency configuration update event publication
  - Do not describe Catalog/Order/Payment validation logic as UserService-owned behavior
  - Acceptance: UserService docs clearly identify policy ownership, APIs, and event publication responsibilities introduced by feature `0010`

- [ ] D002 [US3] Update CatalogService, OrderService, and PaymentService docs for currency-aware validation and read-model behavior
  - File: `services/catalog-service/README.md`; `services/order-service/README.md`; `services/order-service/workflows.md`; `services/payment-service/README.md`; `services/payment-service/workflows.md`
  - Add only the actual behavior changes: local currency-config ref validation, canonical coupon currency model, mixed-currency rejection points, explicit money metadata, and invalid-money historical-read handling
  - Keep ApiGateway and unrelated services documentation untouched because they are reused supporting context only
  - Acceptance: changed supporting service docs describe the new validation/read-model responsibilities without rewriting unaffected sections

## Catalogs and ADR

### Update

- [ ] D003 [US1] Update API catalog for new currency-policy endpoints and changed money-bearing endpoint contracts
  - File: `shared/api-catalog.md`
  - Add: `GET /api/v1/admins/configuration/currencies`, `PUT /api/v1/admins/configuration/currencies`, and `GET /api/v1/users/platform-currency-policy`
  - Update the affected `products`, `coupons`, `carts`, `orders`, and `payments` rows to note explicit money metadata, canonical coupon currency contract changes, and mixed-currency rejection semantics where the contract changed
  - Acceptance: every new or materially changed public HTTP contract introduced by this feature is reflected once in the API catalog with the correct owning service

- [ ] D004 [US1] Update the event catalog for currency configuration propagation
  - File: `shared/event-catalog.md`
  - Add the currency configuration update event under the Identity/User/Store or cross-service governance section with UserService ownership and CatalogService/OrderService/PaymentService consumers
  - Note that the event updates local validation projections only and does not introduce a new saga or conversion workflow
  - Acceptance: the event catalog records the new integration contract and consumer set exactly once without duplicating existing checkout/payment events

- [ ] D005 [US1] Finalize ADR-0010 to reflect the implemented architecture decision
  - File: `architecture/decisions/ADR-0010-user-service-platform-currency-policy.md`
  - Promote status from `Draft` to the final accepted/implemented state used by the repo, and align the consequences/follow-up section with the generated task scope
  - Do not introduce a second ADR for the same ownership decision
  - Acceptance: ADR-0010 is the single source of truth for why UserService owns the configuration foundation and why other services consume projections instead of using direct reads or synchronous validation calls
