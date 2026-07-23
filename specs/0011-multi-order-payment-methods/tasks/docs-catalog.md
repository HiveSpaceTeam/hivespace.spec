# Docs And Catalog Tasks

## Shared Catalogs

### Update

- [ ] D001 [US2] Update `shared/api-catalog.md` for PaymentService method and retry APIs
  - File: `shared/api-catalog.md`
  - Ensure entries include `GET /api/v1/payments/methods`, `GET /api/v1/payments/{paymentId}`, `GET /api/v1/payments/by-reference/{referenceNo}`, `GET /api/v1/payments/by-order/{orderId}`, and `POST /api/v1/payments/{paymentId}/attempts` with auth and purpose aligned to `contracts/payment-api.md`.
  - Document COD/VNPay checkout availability and Stripe future/unavailable metadata in PaymentService notes.
  - Do not add duplicate payment endpoints under another service.
  - Acceptance: API catalog matches implemented public routes and does not mention frontend-owned method lists.

- [ ] D002 [US1] Update `shared/event-catalog.md` for changed checkout payment messages
  - File: `shared/event-catalog.md`
  - Update `InitiatePayment`, `PaymentInitiatedIntegrationEvent`, `PaymentInitiationFailedIntegrationEvent`, `PaymentSucceededIntegrationEvent`, and `PaymentFailedIntegrationEvent` descriptions to include checkout correlation, one shared payment, linked order set, `ReferenceNo`, and attempt identity.
  - Keep CatalogService, NotificationService, and UserService reused messages unchanged except for verification-only references.
  - Do not create new event names with duplicate meanings.
  - Acceptance: event catalog reflects `contracts/checkout-payment-messages.md`.

## Service Docs

### Update

- [ ] D003 [US1] Update `PaymentService service docs`
  - File: `services/payment-service/README.md`; `services/payment-service/domain.md`; `services/payment-service/api.md`; `services/payment-service/data-and-events.md`; `services/payment-service/workflows.md`
  - Document PaymentService ownership of checkout-level payments, child attempts, linked order snapshots, `PAY-{ULID}` reference generation, canonical payment methods, VNPay merchant reference behavior, retry rules, and stale callback handling.
  - Preserve wallet documentation and existing gateway callback notes.
  - Do not state that PaymentService mutates OrderService order lifecycle directly.
  - Acceptance: PaymentService docs describe the shipped behavior without contradicting ADR-0011 or the API/event catalogs.

- [ ] D004 [US1] Update `OrderService service docs`
  - File: `services/order-service/README.md`; `services/order-service/domain.md`; `services/order-service/api.md`; `services/order-service/data-and-events.md`; `services/order-service/workflows.md`
  - Document `ORD-{ULID}` order codes for new orders, linked payment summary fields, one `InitiatePayment` command per checkout, current-attempt outcome validation, COD offline payment path, and stale attempt event ignoring.
  - Preserve OrderService ownership of order lifecycle and fulfillment separation by order/store.
  - Do not imply OrderService owns gateway payment truth.
  - Acceptance: OrderService docs align with saga-design.md and implemented checkout saga behavior.

- [ ] D005 [US2] Update `ApiGateway docs`
  - File: `services/api-gateway/README.md`; `services/api-gateway/api.md`
  - If B029 changes route config, document that existing `/api/v1/payments/**` routes cover methods, by-reference lookup, by-order lookup, and retry attempts.
  - If no route config changed, leave ApiGateway docs unchanged and record verification in V008.
  - Do not add business validation or payment ownership language to gateway docs.
  - Acceptance: gateway docs are changed only when source route behavior changed.

## Architecture Decisions

### Update

- [ ] D006 [US3] Update `ADR-0011` implementation follow-up status
  - File: `architecture/decisions/ADR-0011-checkout-level-payment-ownership.md`
  - After implementation, update status/follow-up notes to reflect completed catalog/doc updates and any implementation constraints discovered.
  - Keep the decision that PaymentService owns checkout-level payment truth and OrderService owns checkout saga/order lifecycle.
  - Do not rewrite unrelated ADR context.
  - Acceptance: ADR follow-up no longer lists completed work as pending after feature ships.
