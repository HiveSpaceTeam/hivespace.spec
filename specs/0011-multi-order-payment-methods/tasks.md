# Tasks: Multi-Order Payment Methods

- **Input**: Design documents from `/specs/0011-multi-order-payment-methods/`
- **Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, saga-design.md, ADR-0011
- **Detailed tasks**: Implementation tasks live under `/specs/0011-multi-order-payment-methods/tasks/`
- **Organization**: `tasks.md` is the compatibility entrypoint and high-level tracker. Detailed task files are grouped by implementation ownership, then service/app/package/lib, then action.

## Detailed Task Files

| File | Purpose | Task ID prefix | Task count |
| --- | --- | --- | ---: |
| `tasks/backend.md` | Backend services, APIs, domain/application/infrastructure code, shared backend libs, migrations | `B###` | 29 |
| `tasks/frontend.md` | Frontend apps, shared package, services, stores, components, pages, routes, i18n | `F###` | 15 |
| `tasks/docs-catalog.md` | Service docs, API catalog, event catalog, ADRs, architecture docs | `D###` | 6 |
| `tasks/verification.md` | Builds, tests, lint/type-check, quickstart/manual validation, final searches | `V###` | 10 |

## Documentation And Catalog Scope

Editable docs/catalog entries:

- PaymentService service docs: checkout-level payment aggregate, attempts, payment methods, linked-order reads, retry rules, VNPay reference behavior.
- OrderService service docs: checkout saga payment state, `ORD-{ULID}` order codes, linked payment summaries, current-attempt outcome handling.
- ApiGateway route config/docs only if source inspection shows existing `/api/v1/payments/**` routing does not cover new PaymentService endpoints.
- `shared/api-catalog.md`: new/changed PaymentService and OrderService public contracts.
- `shared/event-catalog.md`: changed checkout payment saga command/event payload responsibilities.
- `architecture/decisions/ADR-0011-checkout-level-payment-ownership.md`: keep status/follow-up aligned with implementation outcome.

Verification-only supporting services:

- CatalogService inventory reservation/release/confirmation contracts are reused unchanged.
- NotificationService fulfillment notification contracts are reused unchanged.
- UserService currency-policy projection is reused unchanged.
- Existing common API/event catalog rows outside the changed payment/order contracts are verification-only.

## Task Index

| Area | File | Status | Notes |
| --- | --- | --- | --- |
| Backend | `tasks/backend.md` | Not started | PaymentService, OrderService, shared messaging contracts, optional ApiGateway route verification |
| Frontend | `tasks/frontend.md` | Not started | Shared package plus buyer, seller, and admin app surfaces |
| Docs/Catalog | `tasks/docs-catalog.md` | Not started | Catalogs, service docs, ADR follow-up |
| Verification | `tasks/verification.md` | Not started | Target repo instructions, tests, coverage, builds, user-owned E2E |

## Dependency Order

1. Read target repo instructions: V001 and V002 before source repo implementation.
2. Backend contracts and tests: B001-B006 before paired PaymentService implementation; B016-B020 before paired OrderService implementation.
3. PaymentService implementation: B007-B015, then B029 route verification/update if needed.
4. OrderService implementation: B021-B028 after shared message contract changes in B011 and B012.
5. Frontend shared foundation: F003-F006 before app-specific work.
6. Buyer app checkout/order work: F007-F011 for US1/US2/US3 buyer surfaces.
7. Seller and admin app work: F012-F017 for US2/US3 displays and reconciliation.
8. Docs/catalog updates: D001-D006 after contracts and implementation shape are confirmed.
9. Verification: V003-V010 after relevant backend/frontend/docs tasks are complete; V009 is user-owned E2E and not agent-executable.

## Story Traceability

| Story | Detailed task IDs | Independent acceptance |
| --- | --- | --- |
| US1 | B001, B003-B005, B007-B008, B010-B014, B018, B022-B026, B028, F003, F007-F011, D002-D004, V003-V004, V007, V009 | A cart checkout that splits into multiple orders produces one VNPay payment request, one `PAY-{ULID}` reference, and all linked orders transition exactly once on current-attempt payment outcome. |
| US2 | B006, B015, B029, F004, F006, F012, F014, F017, D001, D005, V005-V006, V008 | Buyer, seller, and admin surfaces load COD, VNPay, and future/unavailable Stripe metadata from PaymentService and do not hardcode divergent method lists. |
| US3 | B002, B009, B016-B017, B019-B021, B027, F005, F010, F013, F015-F016, D006 | Order and payment detail views expose distinct `ORD-{ULID}` values, one shared `PAY-{ULID}`, linked order lists, latest attempt, and privileged attempt history. |

## Suggested MVP Scope

Complete B001-B014, B016-B027, F003-F011, D001-D004, V001-V004, V007, and V009 to satisfy User Story 1 with the minimum backend, buyer frontend, catalog, and validation coverage.

## Completion Checklist

- [ ] All detailed task files listed in Task Index exist.
- [ ] All detailed tasks have unique IDs across files.
- [ ] Every implementation task includes file path, exact change detail, forbidden behavior, dependencies/callers where relevant, and acceptance.
- [ ] Test-code tasks precede their paired implementation tasks in each detailed file and in the dependency order.
- [ ] Verification tasks cover backend tests/coverage, frontend tests/type-check/build, docs validation, and user-owned browser E2E.
- [ ] No task edits `../hivespace.config` or creates `tasks/config.md`.
