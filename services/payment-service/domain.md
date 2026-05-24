# PaymentService Domain Model

## Purpose

PaymentService owns payment processing, gateway interaction, wallets, wallet transactions, and idempotent payment state transitions.

Implementation source:

```text
../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Domain
```

## Core Model

| Model | Type | Meaning |
|---|---|---|
| `Payment` | Aggregate root | Payment record for an order with buyer, amount, method, gateway, gateway identifiers, idempotency key, status, and expiry |
| `Wallet` | Aggregate root | User balance account with available balance, escrow balance, reward points, status, and transaction history |
| `Transaction` | Entity under `Wallet` | Append-only ledger row with direction, type, amount, balance after, reference, and description |
| `PaymentMethod`, `GatewayResponse`, `BankAccount` | Value objects | Payment method metadata, gateway response data, and payout account data |

## Payment Rules

- A payment requires order ID, buyer ID, positive amount, payment method, gateway, and idempotency key.
- New payments start as `Pending` and expire 15 minutes after creation.
- A payment can move from `Pending` to `Processing` only before expiry and only when a gateway payment URL is available.
- A payment can move from `Processing` to `Succeeded` only once a gateway transaction ID and response are recorded.
- A succeeded payment cannot later be failed or cancelled.
- Failed payments record gateway response data when available.
- Pending or processing payments may be marked `Expired`; other states ignore expiry marking.
- Gateway transaction handling must remain idempotent so the same gateway callback cannot create duplicate success/failure effects.

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
| Payment | `Pending` -> `Processing` -> `Succeeded`; `Pending`/`Processing` -> `Expired`; non-succeeded payments may become `Failed` or `Cancelled` |
| Wallet | `Active`, `Suspended`, `Closed`; current domain behavior supports suspend and reactivate |
| Transaction | Append-only ledger entries with `Credit` or `Debit` direction |

## Cross-Service Facts

- PaymentService consumes `InitiatePayment` from the checkout saga to create a payment and gateway URL.
- PaymentService publishes `PaymentInitiatedIntegrationEvent` or `PaymentInitiationFailedIntegrationEvent` for saga progression.
- PaymentService publishes `PaymentSucceededIntegrationEvent` and `PaymentFailedIntegrationEvent` after gateway/IPN outcomes.
- OrderService owns order lifecycle and reacts to payment events; PaymentService does not mutate order state directly.
- PaymentService owns gateway identifiers and idempotency; other services must not infer payment state from order state alone.
