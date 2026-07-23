# Verification Tasks

## Source Repo Instructions

### Verify

- [ ] V001 Verify `backend repo implementation instructions`
  - File: `../hivespace.microservice/AGENTS.md`; `../hivespace.microservice/CLAUDE.md`; `../hivespace.microservice/TESTING.md`
  - Read both agent instruction files before backend implementation, plus TESTING.md if present.
  - Follow the target repo's coverage workflow and naming conventions for PaymentService and OrderService tests.
  - Do not start backend source work using only spec-repo instructions.
  - Acceptance: backend implementation notes reference the loaded target repo rules.

- [ ] V002 Verify `frontend repo implementation instructions`
  - File: `../hivespace.web/AGENTS.md`; `../hivespace.web/CLAUDE.md`; `../hivespace.web/TESTING.md`
  - Read both agent instruction files before frontend implementation, plus TESTING.md if present.
  - Confirm frontend tests use `should ...` names and shared package import rules.
  - Do not add app-to-app imports or bypass `@hivespace/shared`.
  - Acceptance: frontend implementation notes reference the loaded target repo rules.

## Backend

### Verify

- [ ] V003 [US1] Verify `PaymentService tests and coverage`
  - File: `../hivespace.microservice/tests/HiveSpace.PaymentService.Tests/**`; `../hivespace.microservice/coverage.ps1`
  - Run the target repo coverage flow for PaymentService after B001-B017.
  - Confirm tests cover checkout payment creation, reference collision retry, initiation validation, retry attempts, COD offline attempts, VNPay merchant reference, duplicate callbacks, stale callbacks, and payment read authorization.
  - Add measured-scope tests if PaymentService affected coverage is below 80%.
  - Acceptance: PaymentService targeted tests are green and measured coverage policy is satisfied or documented with added tasks.

- [ ] V004 [US1] Verify `OrderService tests and coverage`
  - File: `../hivespace.microservice/tests/HiveSpace.OrderService.Tests/**`; `../hivespace.microservice/coverage.ps1`
  - Run the target repo coverage flow for OrderService after B018-B028.
  - Confirm tests cover `ORD-{ULID}` generation, one `InitiatePayment` for multi-order checkout, current-attempt success/failure, stale attempt ignoring, COD path, and order read payment summaries.
  - Add measured-scope tests if OrderService affected coverage is below 80%.
  - Acceptance: OrderService targeted tests are green and measured coverage policy is satisfied or documented with added tasks.

- [ ] V005 [US2] Verify `backend build and gateway route coverage`
  - File: `../hivespace.microservice/HiveSpace.sln`; `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/appsettings*.json`
  - Run the narrowest backend build or solution build required by target repo instructions after backend changes.
  - Confirm gateway routes cover `GET /api/v1/payments/methods`, `GET /api/v1/payments/by-reference/{referenceNo}`, and `POST /api/v1/payments/{paymentId}/attempts`.
  - Do not treat `../hivespace.config` as a required feature artifact.
  - Acceptance: backend build is green and route coverage is confirmed.

## Frontend

### Verify

- [ ] V006 [US2] Verify `shared frontend package checks`
  - File: `../hivespace.web/packages/shared/src/**`
  - Run targeted shared package tests and type-check after F001-F006.
  - Confirm payment service/type tests pass and i18n resources compile for English and Vietnamese.
  - Do not run formatters that rewrite unrelated files.
  - Acceptance: shared frontend checks are green.

- [ ] V007 [US1] Verify `buyer app checkout and order checks`
  - File: `../hivespace.web/apps/buyer/src/**`
  - Run targeted buyer app tests/type-check/build after F007-F011.
  - Confirm checkout loads canonical methods, hides/disables Stripe, supports VNPay retry under same reference, supports COD without redirect, and displays order code/payment reference.
  - Do not accept hardcoded buyer-local method source of truth.
  - Acceptance: buyer checks are green and reviewed surfaces use i18n keys.

- [ ] V008 [US2] Verify `seller and admin payment displays`
  - File: `../hivespace.web/apps/seller/src/**`; `../hivespace.web/apps/admin/src/**`
  - Run targeted seller/admin tests/type-check/build after F012-F017.
  - Confirm seller order views and admin payment/order views use canonical method metadata, show shared payment references, and support admin by-reference lookup with linked orders/attempt history.
  - Do not expose cross-store linked order details in seller surfaces unless backend contract authorizes them.
  - Acceptance: seller/admin checks are green and final search finds no app-local method source of truth.

## User-Owned E2E

### Verify

- [ ] V009 [US1] Verify `multi-order VNPay browser journey`
  - File: `specs/0011-multi-order-payment-methods/quickstart.md`
  - User-owned E2E: user runs a browser checkout for a cart that splits into two or more orders, chooses VNPay, completes one payment, opens each generated order, and confirms distinct `ORD-{ULID}` values share the same `PAY-{ULID}` reference.
  - User-owned E2E: user repeats a failed/expired VNPay attempt and confirms retry creates a new attempt under the same payment reference, while stale older callbacks do not change paid orders.
  - Do not mark this complete from automated agent execution.
  - Acceptance: user explicitly confirms the browser journey results.

## Docs

### Verify

- [ ] V010 Verify `docs and catalog consistency`
  - File: `shared/api-catalog.md`; `shared/event-catalog.md`; `services/payment-service/*.md`; `services/order-service/*.md`; `architecture/decisions/ADR-0011-checkout-level-payment-ownership.md`; `specs/0011-multi-order-payment-methods/tasks/*.md`
  - Confirm API/event catalog rows match implemented contracts and service docs do not contradict catalogs or ADR-0011.
  - Search for stale one-payment-per-order wording in PaymentService docs and stale `ORD-{timestamp}-{random}` wording in OrderService docs after D003-D004.
  - Confirm all detailed tasks keep checkbox ID/story/action/file/detail/acceptance format.
  - Acceptance: docs are non-empty, no stale contradictory wording remains, and task format validation passes.
