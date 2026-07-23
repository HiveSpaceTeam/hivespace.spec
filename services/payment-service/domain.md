# PaymentService Domain Model

## Purpose

PaymentService owns checkout-level payment processing, gateway interaction, payment attempts, canonical payment method metadata, wallets, wallet transactions, and idempotent payment state transitions.

Implementation source:

```text
../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Domain
```

## Core Model

| Model | Type | Meaning |
|---|---|---|
| `Payment` | Aggregate root | Checkout-level payment record for one or more generated orders with buyer, amount, currency, method, public `ReferenceNo`, linked order snapshots, current attempt, attempt history, idempotency key, status, and expiry |
| `PaymentAttempt` | Entity under `Payment` | One COD or VNPay attempt with attempt number, method, gateway, amount, currency, status, redirect URL, gateway transaction identifier, failure reason, and lifecycle timestamps |
| `PaymentLinkedOrder` | Entity/value object under `Payment` | Snapshot linking one generated order to the checkout payment, including order ID, order code, store ID, amount, currency, and status summary |
| `PaymentMethod` | Value object/provider metadata | Canonical method metadata for COD, VNPay, and future/unavailable Stripe, including online/offline kind, gateway code, enabled state, checkout-selectable state, availability, and sort order |
| `Wallet` | Aggregate root | User balance account with available balance, escrow balance, reward points, status, and transaction history |
| `Transaction` | Entity under `Wallet` | Append-only ledger row with direction, type, amount, balance after, reference, and description |
| `GatewayResponse`, `BankAccount` | Value objects | Gateway response data and payout account data |

## Payment Rules

- A payment requires buyer ID, positive amount, currency, recognized method, idempotency key, and at least one linked order.
- New payment `ReferenceNo` values use `PAY-{26-character uppercase ULID}` and must be unique before persistence.
- Payment `ReferenceNo` is public reconciliation data and must not be reused as an idempotency key.
- New payments start as `Pending` with one initial attempt and expire according to the current payment window.
- A VNPay attempt can move from `Pending` to `Processing` only before expiry and only when a gateway payment URL is available.
- COD creates an offline attempt without a gateway transaction or redirect URL.
- A payment can move to `Succeeded` only once a current attempt succeeds; gateway transaction identifiers are stored on the attempt.
- A succeeded payment cannot later be failed or cancelled.
- Failed, expired, or cancelled attempts record gateway/failure data when available.
- Retry creates a new child attempt under the same payment while no attempt has succeeded and linked orders remain eligible.
- Switching from a failed, expired, or cancelled VNPay attempt to COD is allowed only before fulfillment starts and before any online attempt succeeds.
- Stripe is exposed as unavailable/future metadata and rejected for checkout attempts until its gateway implementation is enabled.
- Gateway transaction handling must remain idempotent so the same gateway callback cannot create duplicate success/failure effects.
- Stale callbacks from older attempts must not change the aggregate outcome after a newer current attempt exists or the payment has succeeded.

## Wallet Rules

- A wallet requires a user ID and starts `Active` with zero available balance, zero escrow balance, and zero reward points.
- Credit/debit amounts must be positive.
- Credit/debit is allowed only while the wallet is active.
- Debit requires sufficient available balance.
- Every credit/debit creates a transaction with copied `Money` values and the resulting balance.
- Transaction type is derived from the reference prefix: `PAYMENT-`, `REFUND-`, `WITHDRAWAL-`, `ESCROW-`, or `Adjustment`.
- Wallet suspension blocks balance changes until reactivated.

## Lifecycle

| Lifecycle | States / transitions |
|---|---|
| Payment | `Pending` -> `Processing` -> `Succeeded`; `Pending`/`Processing` -> `Expired`; non-succeeded payments may become `Failed` or `Cancelled`; retries add attempts under the same aggregate/reference |
| PaymentAttempt | Attempt-specific `Pending`/`Processing`/`Succeeded`/`Failed`/`Expired`/`Cancelled` lifecycle with monotonically increasing attempt number |
| Wallet | `Active`, `Suspended`, `Closed`; current domain behavior supports suspend and reactivate |
| Transaction | Append-only ledger entries with `Credit` or `Debit` direction |

## Cross-Service Facts

- PaymentService consumes `InitiatePayment` from the checkout saga to create one checkout-level payment and initial attempt for the checkout correlation, method, final total, currency, and linked order set.
- PaymentService publishes `PaymentInitiatedIntegrationEvent` or `PaymentInitiationFailedIntegrationEvent` with payment reference, attempt identity, method, total, currency, and linked order context for saga progression.
- PaymentService publishes `PaymentSucceededIntegrationEvent` and `PaymentFailedIntegrationEvent` after current-attempt gateway/IPN outcomes.
- OrderService owns order lifecycle and reacts to payment events; PaymentService does not mutate order state directly.
- PaymentService owns gateway identifiers and idempotency; other services must not infer payment state from order state alone.
