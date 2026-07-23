# Implementation Plan: Multi-Order Payment Methods

**Branch:** `0012-multi-order-payment-methods`
**Date:** 2026-07-11
**Spec:** [spec.md](spec.md)

> Constitution loaded from `.specify/memory/constitution.md` before planning.

---

## Phase 0 - Research

### Existing context read before planning

- [x] `services/order-service/README.md`, `domain.md`, `api.md`, `data-and-events.md`, `workflows.md`
- [x] `services/payment-service/README.md`, `domain.md`, `api.md`, `data-and-events.md`, `workflows.md`
- [x] `services/api-gateway/README.md`
- [x] `services/notification-service/README.md`
- [x] `shared/event-catalog.md`
- [x] `shared/api-catalog.md`
- [x] Existing checkout saga documentation

### Technical unknowns

- None remain after [research.md](research.md). Payment grouping, method ownership, COD modeling, retry-attempt modeling, Stripe availability, public identifiers, and VNPay reference use are resolved.

### Research notes

See [research.md](research.md).

---

## Phase 1 - Architecture & Data Model

### Service placement

PaymentService owns checkout-level payment truth, payment attempts, payment reference numbers, payment-to-order links, canonical payment method configuration, and gateway interaction. OrderService owns order creation, order codes, checkout saga state, and applying current-attempt payment outcomes to order lifecycle. ApiGateway is changed only if source routing needs to expose new PaymentService endpoints under the existing `/api/v1/payments` prefix.

| Service | Classification | Reason | Documentation/catalog action |
| ------- | -------------- | ------ | ---------------------------- |
| PaymentService | Owning service | Owns payment aggregate refactor, checkout-level payment records, child attempts, payment references, payment methods, VNPay merchant reference, idempotent gateway outcomes, retry eligibility, and payment read APIs. | Update PaymentService docs during implementation wrap-up; update API and event catalogs now for new/changed contracts. |
| OrderService | Owning service | Owns checkout saga changes, per-order `OrderCode` generation, linked payment reference display in order reads, current attempt tracking, and idempotent order payment transitions from shared payment outcomes. | Update OrderService docs during implementation wrap-up; update API and event catalogs now for changed checkout/payment contracts. |
| ApiGateway | Changed supporting service | May need route config only if existing `/api/v1/payments/**` route does not cover new PaymentService method/detail endpoints. No business logic. | Verify existing route coverage; update gateway config only if required. |
| NotificationService | Reused supporting service | Existing fulfillment and notification flows continue to consume order workflow messages unchanged for this feature. | Verification-only; no catalog/doc rewrite. |
| CatalogService | Reused supporting service | Existing inventory reservation/release/confirmation messages remain unchanged. | Verification-only; no catalog/doc rewrite. |
| UserService | Reused supporting service | Existing currency-policy projection event is consumed unchanged. | Verification-only; no catalog/doc rewrite. |
| Buyer app | Changed client | Checkout method selection must load PaymentService method metadata and hide/disable Stripe. | Frontend tasks update types/services/stores/components/i18n. |
| Seller app | Changed client | Seller order/payment displays must use canonical method labels and show linked payment references. | Frontend tasks update types/services/stores/components/i18n. |
| Admin app | Changed client | Admin payment/order reconciliation views must use canonical method metadata and linked order/payment traceability. | Frontend tasks update types/services/stores/components/i18n. |

### New and changed domain/data model

Detailed model: [data-model.md](data-model.md).

PaymentService changes:

- Refactor `Payment` from one-order-only to checkout-level payment with one or more linked orders.
- Add public `ReferenceNo` generated as `PAY-{26-character uppercase ULID}` and keep it separate from idempotency keys.
- Add child `PaymentAttempt` records for initial payment and each retry while keeping the same `ReferenceNo`.
- Add linked order rows/snapshots for order ID, `OrderCode`, store ID, amount, currency, and status summary.
- Add canonical payment method metadata for `COD`, `VNPAY`, and `STRIPE`.
- Preserve gateway transaction identifiers separately from `ReferenceNo`.

OrderService changes:

- Replace new-order code generation with `ORD-{26-character uppercase ULID}` while preserving historical reads.
- Store the shared payment ID/reference and current attempt identity on every generated order after payment initiation/COD payment creation.
- Apply shared payment success/failure exactly once to all linked orders and only for the current accepted attempt.

### New repository interfaces

PaymentService:

```csharp
public interface IPaymentRepository : IRepository<Payment, PaymentId>
{
    Task<Payment?> GetByReferenceNoAsync(string referenceNo, CancellationToken ct);
    Task<Payment?> GetByOrderIdAsync(Guid orderId, CancellationToken ct);
    Task<bool> ReferenceNoExistsAsync(string referenceNo, CancellationToken ct);
}
```

OrderService:

```csharp
public interface IOrderRepository : IRepository<Order, OrderId>
{
    Task<bool> OrderCodeExistsAsync(string orderCode, CancellationToken ct);
    Task<IReadOnlyList<Order>> GetByCheckoutCorrelationIdAsync(Guid correlationId, CancellationToken ct);
}
```

### Integration events and saga messages

Changed contracts are documented in [contracts/checkout-payment-messages.md](contracts/checkout-payment-messages.md).

| Contract | Producer | Consumer(s) | Change |
| -------- | -------- | ----------- | ------ |
| `InitiatePayment` | OrderService checkout saga | PaymentService | Include checkout correlation, payment method, final total, currency, and full linked order set. |
| `PaymentInitiatedIntegrationEvent` | PaymentService | OrderService checkout saga | Return one payment ID/reference and linked order set for the checkout. |
| `PaymentInitiationFailedIntegrationEvent` | PaymentService | OrderService checkout saga | Include checkout correlation, method, linked order set, and failure reason. |
| `PaymentSucceededIntegrationEvent` | PaymentService | OrderService checkout saga | Outcome includes attempt identity and applies to one shared payment and all linked orders only when it matches the current attempt. |
| `PaymentFailedIntegrationEvent` | PaymentService | OrderService checkout saga | Failure/expiry/cancel includes attempt identity and applies to one shared payment and all linked orders only when it matches the current attempt. |

Existing CatalogService, NotificationService, and currency-policy projection messages are reused unchanged; no catalog update is required for those reused contracts.

### API endpoints

New/changed contracts are documented in [contracts/payment-api.md](contracts/payment-api.md) and [contracts/payment-methods-api.md](contracts/payment-methods-api.md).

| Method | Path | Auth | Owner | Change |
| ------ | ---- | ---- | ----- | ------ |
| GET | `/api/v1/payments/methods` | `Authorize` | PaymentService | New canonical method metadata for all apps. |
| GET | `/api/v1/payments/{paymentId}` | `Authorize` | PaymentService | Include `ReferenceNo`, linked orders, method metadata, and explicit money. |
| GET | `/api/v1/payments/by-reference/{referenceNo}` | `Authorize` | PaymentService | New support/admin lookup by public payment reference. |
| GET | `/api/v1/payments/by-order/{orderId}` | `Authorize` | PaymentService | Return the shared checkout payment for any linked order. |
| POST | `/api/v1/payments/{paymentId}/attempts` | `Authorize` | PaymentService | New retry endpoint that creates a child attempt under the same checkout payment reference. |
| POST | `/api/v1/orders/checkout` | `Authorize` | OrderService | Existing endpoint starts checkout with canonical method code and shared payment flow. |
| GET | `/api/v1/orders/{orderId}` | `Authorize` | OrderService | Existing endpoint includes order code and linked payment summary. |

No `../hivespace.config` updates are planned. Source-repo runtime settings, gateway route config, and frontend environment typing stay under backend/frontend repos if implementation requires them.

---

## Phase 2 - Implementation Plan

### Layer order

**PaymentService domain layer**

- [ ] Refactor `Payment` to represent one checkout-level payment with `ReferenceNo`, amount, currency, aggregate status, idempotency key, linked orders, current attempt, and attempt collection.
- [ ] Add `PaymentAttempt` child entity with attempt number, method, gateway, amount, currency, status, gateway transaction ID, redirect URL, failure reason, and timestamps.
- [ ] Add `PaymentLinkedOrder` entity/value object with order ID, order code, store ID, buyer/store totals, currency, and relationship to payment.
- [ ] Add `PaymentReferenceNo` generation helper with uniqueness retry using `PAY-{ULID}`.
- [ ] Add canonical `PaymentMethodCode`, `PaymentMethodAvailability`, and payment method metadata model for `COD`, `VNPAY`, and `STRIPE`.
- [ ] Preserve historical/unknown method reads without rejecting legacy data.

**PaymentService application layer**

- [ ] Update `InitiatePayment` handler to validate linked order set, amount, currency, method, and idempotency against final checkout state.
- [ ] Create COD offline attempt records without gateway transactions.
- [ ] Create VNPay checkout-level payment records with attempt `1` and send `ReferenceNo` as the VNPay merchant transaction reference.
- [ ] Add retry command/query handling for `POST /api/v1/payments/{paymentId}/attempts`, allowing VNPay retry and COD switch only when no attempt has succeeded and linked orders remain eligible.
- [ ] Add queries for payment detail, payment by reference, payment by order, payment methods, latest attempt, and full attempt history for privileged reads.
- [ ] Enforce idempotent duplicate initiation/return/IPN behavior and stale callback protection by attempt identity.

**PaymentService infrastructure layer**

- [ ] Add EF mapping/migration for payment reference number uniqueness, linked order table/owned collection, and payment attempts.
- [ ] Add persisted or code-backed canonical payment method provider; keep Stripe unavailable for checkout.
- [ ] Update VNPay adapter to use `ReferenceNo` as merchant reference while storing gateway IDs on the attempt.
- [ ] Update outbox-published payment outcome events with linked order set and attempt identity.

**PaymentService API layer**

- [ ] Add `GET /api/v1/payments/methods`.
- [ ] Add `GET /api/v1/payments/by-reference/{referenceNo}`.
- [ ] Add `POST /api/v1/payments/{paymentId}/attempts`.
- [ ] Extend existing payment detail and by-order DTOs with reference, method metadata, linked orders, latest attempt, attempt history where authorized, and explicit money metadata.

**OrderService domain layer**

- [ ] Replace new `OrderCode` generation with `ORD-{ULID}` and uniqueness retry before persistence.
- [ ] Add linked payment summary fields to order read model/state where needed without making OrderService own payment truth.
- [ ] Keep historical order codes readable and searchable.

**OrderService application/saga layer**

- [ ] Change checkout saga state data to carry the generated linked order set and checkout-level payment reference.
- [ ] Change `InitiatePayment` publishing to send all generated orders and final total once per checkout.
- [ ] For COD, wait for PaymentService-created offline attempt confirmation before marking all orders as COD.
- [ ] Store current payment attempt ID/number in saga state and order payment summaries.
- [ ] For VNPay success/failure/expiry, apply the outcome to every linked order exactly once only when the event attempt identity matches current saga state.
- [ ] Ignore stale older-attempt events after a newer attempt exists or the payment already succeeded.
- [ ] Reject mismatched amount, currency, method, or linked order set before payment creation.

**OrderService API layer**

- [ ] Extend checkout request/response DTOs to use canonical payment method code and return shared payment reference plus latest attempt when available.
- [ ] Extend buyer/seller/admin order detail DTOs with `OrderCode`, shared payment ID/reference, payment method code/label, payment status summary, and latest attempt summary.

### Saga design

Required because this feature changes the existing MassTransit checkout saga's payment state data and payment initiation/outcome messages. See [saga-design.md](saga-design.md).

### Architecture decision

Required because checkout-level payment ownership, canonical method ownership, and payment/order traceability cross service boundaries. See [ADR-0011](../../architecture/decisions/ADR-0011-checkout-level-payment-ownership.md).

---

## Phase 3 - Frontend Plan

### Surfaces

- [x] buyer
- [x] seller
- [x] admin

### Files to create or update

Implementation must inspect `../hivespace.web/packages/shared/src` first.

| Order | Area | Notes |
| ----- | ---- | ----- |
| 1 | shared types | Add payment method, payment detail, payment attempt, linked order, order payment summary, and method availability types. |
| 2 | shared/payment service | Add `getPaymentMethods`, `getPaymentByReference`, `createPaymentAttempt`, update detail/by-order calls. |
| 3 | shared/order types/services | Add order code and linked payment summary fields. |
| 4 | buyer checkout store/components | Load canonical methods, show COD/VNPay, hide or disable Stripe for checkout, and support retry after failed/expired attempt. |
| 5 | buyer order pages | Display order code and linked payment reference. |
| 6 | seller order pages | Display canonical method label and shared payment reference. |
| 7 | admin payment/order views | Support reference lookup and linked order list display where present. |
| 8 | i18n | Update English and Vietnamese together for method labels, unavailable Stripe state, reference labels, and linked-order text. |

### i18n keys to add

Exact namespace can follow existing app/shared conventions, but both `en` and `vi` resources must include:

```json
{
  "payments": {
    "methods": {
      "cod": "",
      "vnpay": "",
      "stripe": "",
      "unavailable": "",
      "future": ""
    },
    "referenceNo": "",
    "linkedOrders": "",
    "attempt": "",
    "retryPayment": "",
    "checkoutPayment": ""
  },
  "orders": {
    "orderCode": "",
    "paymentReference": ""
  }
}
```

---

## Constitution Compliance Check

- [x] No hard-deletes planned.
- [x] Money values remain `long` smallest currency units with explicit currency metadata.
- [x] New order codes and payment reference numbers use ULID-based public identifiers.
- [x] Cross-service payment messages use MassTransit and outbox-published integration events.
- [x] Payment retry is modeled as child attempts under one checkout payment reference, with stale attempt outcomes ignored.
- [x] No new package versions are planned in `.csproj` files.
- [x] Both English and Vietnamese frontend i18n resources must be updated.
- [x] Frontend text must use i18n keys only.
- [x] No implementation work requires `../hivespace.config` updates.

Post-design check: PASS. Required conditional artifacts and catalog updates are included; no unresolved clarifications remain.
