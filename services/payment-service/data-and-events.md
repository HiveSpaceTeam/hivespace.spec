# PaymentService Data And Events

## Data Ownership

PaymentService owns:

- `payments`
- `wallets`
- `transactions`

## Core Data

| Data | Notes |
|---|---|
| Payment | One payment record per order where payment is required |
| Wallet | One wallet per user |
| Transaction | Append-only wallet ledger entry |
| Idempotency key | Prevents duplicate payment creation/processing |

## Consumed Messages

| Message | Purpose |
|---|---|
| `InitiatePayment` | Create payment and payment URL for checkout |

## Published Events

| Event | Purpose |
|---|---|
| `PaymentInitiatedIntegrationEvent` | Saga can redirect/wait for payment |
| `PaymentInitiationFailedIntegrationEvent` | Saga should compensate/fail payment path |
| `PaymentSucceededIntegrationEvent` | OrderService can continue paid checkout |
| `PaymentFailedIntegrationEvent` | OrderService can fail or compensate checkout |

## Publisher Policy

- PaymentService application publishing uses service-owned publisher abstractions for payment integration events.
- Checkout saga participant responses remain MassTransit consume-context workflow messages.

## Invariants

- Payment status changes are idempotent and monotonic for a gateway transaction.
- Gateway transaction IDs must not process twice.
- Wallet running balances must stay consistent with transaction history.
- PaymentService does not directly update orders.
