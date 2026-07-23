# PaymentService Data And Events

## Data Ownership

PaymentService owns:

- `payments`
- `payment_attempts`
- `payment_linked_orders`
- `wallets`
- `transactions`
- `platform_currency_config_refs`

## Core Data

| Data | Notes |
|---|---|
| Payment | One checkout-level payment record that may cover one or more generated orders and owns one stable `PAY-{ULID}` reference |
| PaymentAttempt | Child attempt history under a payment; retries add attempts while keeping the payment reference stable |
| PaymentLinkedOrder | Snapshot of each generated order linked to the checkout payment for support/reconciliation reads |
| Payment method metadata | PaymentService-owned COD, VNPay, and future/unavailable Stripe metadata used by all apps |
| Wallet | One wallet per user |
| Transaction | Append-only wallet ledger entry |
| Idempotency key | Prevents duplicate payment creation/processing |

## Consumed Messages

| Message | Purpose |
|---|---|
| `InitiatePayment` | Create one checkout-level payment and initial payment attempt for a checkout correlation, method, final total, currency, and linked order set |
| `PlatformCurrencyPolicyUpdatedIntegrationEvent` | Refresh local currency-policy validation projection for payment initiation and payment read normalization |

## Published Events

| Event | Purpose |
|---|---|
| `PaymentInitiatedIntegrationEvent` | Saga receives payment ID/reference, current attempt identity, method, total, currency, redirect URL when applicable, and linked order set |
| `PaymentInitiationFailedIntegrationEvent` | Saga should compensate/fail payment path |
| `PaymentSucceededIntegrationEvent` | OrderService can continue paid checkout when checkout correlation and attempt identity match the current attempt |
| `PaymentFailedIntegrationEvent` | OrderService can fail, retry, or compensate checkout when checkout correlation and attempt identity match the current attempt |

## Publisher Policy

- PaymentService application publishing uses service-owned publisher abstractions for payment integration events.
- Checkout saga participant responses remain MassTransit consume-context workflow messages.

## Invariants

- Payment status changes are idempotent and monotonic for a gateway transaction.
- Payment attempts are numbered monotonically within one checkout-level payment.
- A succeeded payment rejects further retry attempts.
- Stale gateway callbacks from older attempts do not publish order-changing outcomes after a newer current attempt exists or the payment has succeeded.
- Currency policy ownership remains in UserService; PaymentService uses the local projection only for synchronous validation.
- Gateway transaction IDs are stored on payment attempts and must not process twice.
- Wallet running balances must stay consistent with transaction history.
- PaymentService does not directly update orders.
