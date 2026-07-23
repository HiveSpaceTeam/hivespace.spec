# Research: Multi-Order Payment Methods

## Decision: Model PaymentService payments at checkout level

**Rationale:** The feature needs one payment action to cover one or more generated orders. PaymentService already owns payment lifecycle, gateway interaction, idempotency, and VNPay integration, while OrderService owns order lifecycle. A checkout-level `Payment` with linked order rows keeps payment truth in PaymentService and lets OrderService react through existing saga outcomes.

**Alternatives considered:**

| Alternative | Why rejected |
| --- | --- |
| Keep one payment per order and group in UI | Still creates multiple payment records and does not satisfy pay-once reconciliation. |
| Move payment grouping into OrderService | Violates the PaymentService boundary for gateway/payment truth. |
| Store only checkout correlation ID without linked order snapshots | Weak support/reconciliation traceability and harder idempotent validation. |

## Decision: PaymentService owns canonical payment method configuration

**Rationale:** Payment methods are payment-domain concepts and must be consistent across buyer, seller, and admin apps. PaymentService can expose method code, label, online/offline type, ordering, and checkout availability without making frontend apps hardcode inconsistent lists.

**Alternatives considered:**

| Alternative | Why rejected |
| --- | --- |
| Frontend-owned constants | Creates drift across apps and cannot reflect backend availability. |
| UserService-owned platform config | UserService does not own payment behavior. |
| ApiGateway static config | Gateway must stay thin and own routing only. |

## Decision: COD creates a checkout-level offline payment record

**Rationale:** COD still needs payment traceability across generated orders, but no gateway transaction exists. Creating a PaymentService-owned offline payment record gives all orders a shared reference and consistent method/status display without pretending gateway processing occurred.

**Alternatives considered:**

| Alternative | Why rejected |
| --- | --- |
| No payment record for COD | Breaks shared payment traceability and inconsistent order display. |
| One COD payment per order | Reintroduces per-order payment grouping and weakens pay-once semantics. |

## Decision: Stripe appears as future/unavailable metadata, not selectable checkout method

**Rationale:** The feature must standardize the method set now while explicitly not implementing Stripe gateway behavior. PaymentService method metadata can expose Stripe for non-checkout displays/configuration and mark it unavailable for checkout.

**Alternatives considered:**

| Alternative | Why rejected |
| --- | --- |
| Hide Stripe everywhere | Does not meet the requirement to show future method metadata outside checkout. |
| Enable Stripe selection with placeholder failure | Creates a poor checkout path and false availability. |

## Decision: Use prefixed uppercase ULIDs for public order/payment references

**Rationale:** `ORD-{ULID}` and `PAY-{ULID}` are sortable, unique, readable, and distinct from database IDs and idempotency keys. Uniqueness is enforced before persistence with retry on collision. Historical identifiers remain readable.

**Alternatives considered:**

| Alternative | Why rejected |
| --- | --- |
| Existing timestamp-random order code | Less consistent and not aligned with the new clarified format. |
| Database identity sequences | Harder to generate consistently across services and environments. |
| Reuse idempotency key as payment reference | Violates FR-018 and exposes an internal duplicate-processing control as public reference. |

## Decision: VNPay merchant transaction reference uses `Payment.ReferenceNo`

**Rationale:** Reconciliation should use the same payment reference visible in APIs/UI. Gateway transaction IDs remain separate fields populated from gateway return/IPN payloads.

**Alternatives considered:**

| Alternative | Why rejected |
| --- | --- |
| Use payment database ID | Less user-friendly and leaks internal identifier semantics. |
| Use idempotency key | Publicly exposes an internal key and conflicts with idempotency responsibilities. |

## Decision: Payment retry uses child attempts under one checkout payment

**Rationale:** Buyers need to retry after failed or expired VNPay attempts without creating multiple checkout payments for the same order set. Keeping one stable `PAY-{ULID}` reference simplifies support and reconciliation, while child `PaymentAttempt` records preserve gateway-specific attempt history and let stale callbacks be ignored safely.

**Alternatives considered:**

| Alternative | Why rejected |
| --- | --- |
| Create a new checkout payment for every retry | Fragments order-to-payment traceability and creates multiple public references for one checkout. |
| Overwrite gateway fields on the payment row | Loses failure history and makes stale callback handling unsafe. |
| Keep retry out of scope | Leaves the buyer with no clear recovery path after payment failure/expiry. |
