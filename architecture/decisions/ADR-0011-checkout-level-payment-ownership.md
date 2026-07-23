# ADR-0011: Checkout-Level Payment Ownership

- **Status**: Implemented
- **Date**: 2026-07-11
- **Feature**: `0011-multi-order-payment-methods`
- **Deciders**: Project maintainers

## Context

HiveSpace checkout can generate multiple orders from one buyer action, commonly split by store. The existing payment model is documented as one payment per order, which does not satisfy the requirement for one buyer payment action, one reconciliation reference, and one shared COD/VNPay payment outcome across all generated orders. The platform also needs COD, VNPay, and future Stripe metadata to be consistent across buyer, seller, and admin apps.

The constitution assigns payment gateway processing, payment lifecycle, wallets, transactions, and payment idempotency to PaymentService. OrderService owns checkout orchestration, order records, and order lifecycle transitions. ApiGateway must stay thin and must not own business rules.

## Decision

PaymentService owns checkout-level payment records, child payment attempts, payment `ReferenceNo` generation, payment-to-order links, canonical payment method configuration, and gateway reconciliation. OrderService continues to own checkout saga orchestration, per-order `OrderCode` generation, current attempt tracking, and order lifecycle transitions caused by payment outcomes.

OrderService sends the final linked order set and final checkout total to PaymentService through the changed `InitiatePayment` saga command. PaymentService validates the request, creates one payment and initial COD or VNPay attempt, and publishes payment initiation/outcome events containing the shared payment reference, attempt identity, and linked order set. Retries create additional child attempts under the same payment reference. All apps load method metadata from PaymentService.

## Consequences

### Positive

- One checkout can be paid with one buyer action and one payment reference.
- COD and VNPay share the same traceability model.
- VNPay retry and COD switch preserve one stable support/reconciliation reference while keeping full attempt history.
- PaymentService remains the source of truth for payment methods, gateway identifiers, and reconciliation.
- OrderService keeps order ownership and fulfillment separated by store/order.
- Frontend apps stop hardcoding divergent payment method lists.

### Negative / Trade-offs

- PaymentService must store order-link snapshots and attempt history even though OrderService remains order truth.
- Checkout saga messages become larger because they carry the linked order set.
- Payment events must include attempt identity, and stale callback handling becomes part of the payment contract.
- Payment/order read models need careful authorization so support traceability does not leak other users' orders.

### Risks

- Mismatched totals between OrderService and PaymentService could create invalid payment records; mitigate by validating amount, currency, and linked order set before payment creation.
- Duplicate or stale gateway callbacks could duplicate or overwrite order transitions; mitigate with PaymentService idempotent attempt state changes, current-attempt checks, and OrderService saga idempotency guards.
- Historical payment/order identifiers may not match new formats; mitigate by preserving reads/search for legacy identifiers.

## Alternatives Considered

| Option | Why rejected |
| --- | --- |
| Keep one payment per order and group only in UI | Does not provide one payment action or one reconciliation reference. |
| Let OrderService own payment group records | Violates PaymentService ownership of payment truth and gateway reconciliation. |
| Create a new checkout payment for each retry | Creates multiple payment references for one checkout and weakens support traceability. |
| Put canonical payment methods in frontend configuration | Risks drift across apps and conflicts with backend checkout validation. |
| Put canonical payment methods in ApiGateway | Gateway owns routing, not payment business semantics. |

## Implementation Notes

- Implemented in feature `0011-multi-order-payment-methods`.
- API and event catalogs now document the checkout-level payment endpoints, retry attempt endpoint, canonical payment methods endpoint, and changed checkout payment messages.
- PaymentService and OrderService service docs now describe checkout-level payment ownership, `PAY-{ULID}` references, `ORD-{ULID}` order codes, linked order/payment summaries, current-attempt validation, retry attempts, and stale callback handling.
