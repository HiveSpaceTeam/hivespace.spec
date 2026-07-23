# PaymentService

## Responsibility

PaymentService owns checkout-level payment processing, payment gateway interaction, canonical payment method metadata, wallets, and wallet transactions.

Source path:

```text
../hivespace.microservice/src/HiveSpace.PaymentService
```

## Owns

- Payment aggregate lifecycle.
- Checkout-level payment records that can cover one or more generated orders.
- Payment attempts, including retry attempts under one stable payment reference.
- Linked order payment snapshots used for payment reconciliation.
- Payment `ReferenceNo` generation using `PAY-{ULID}`.
- Canonical payment method metadata for COD, VNPay, and future/unavailable Stripe.
- Payment gateway redirect/IPN handling.
- VNPay integration.
- Wallet balances and escrow balances.
- Wallet transaction ledger.
- Payment idempotency.
- Local currency-policy validation projection used for payment initiation and money read normalization.

## Must Not Own

- Order lifecycle decisions.
- Inventory reservation.
- Product catalog.
- User identity.
- Notification delivery.

## Architecture

PaymentService follows standard Clean Architecture / DDD with CQRS and Minimal API endpoints.

## Runtime

| Item | Value |
|---|---|
| Local HTTP | `http://localhost:5005` |
| Gateway prefixes | `/api/v1/payments`, `/api/v1/wallets` |
| Database | SQL Server, PaymentService-owned payments/wallets schema |
| Gateway dependency | VNPay |

Backend local development starts PaymentService through Aspire AppHost in `../hivespace.microservice/src/HiveSpace.AppHost`; frontend dev servers remain separate.

## Planning Notes

- Payment webhooks/IPN handlers must be idempotent.
- Gateway callbacks should not cause duplicate payment success/failure events.
- Payment initiation validates the checkout correlation, linked order set, final amount, method, and explicit currency against the local currency-policy projection and fails closed when that projection is missing, unreadable, or stale for a write path.
- Payment `ReferenceNo` is a public reconciliation value and is separate from idempotency keys.
- VNPay uses payment `ReferenceNo` as the merchant transaction reference; gateway transaction identifiers are stored on payment attempts.
- COD creates an offline checkout-level payment attempt without a gateway transaction.
- Retry creates a child attempt under the same checkout-level payment only while no attempt has succeeded and linked orders remain eligible.
- Payment read models return explicit money metadata and invalid-money diagnostics instead of inferring `VND`.
- OrderService reacts to payment outcomes; PaymentService does not mutate order state directly.
- Wallet ledger entries should be append-only.
- Payment workflow events use standardized `*IntegrationEvent` names and service-owned publisher policy per [ADR-0002](../../architecture/decisions/ADR-0002-standardized-integration-event-contracts.md).
- Currency-policy ownership remains in UserService per [ADR-0010](../../architecture/decisions/ADR-0010-user-service-platform-currency-policy.md); PaymentService only consumes the projection for validation.
- Checkout-level payment ownership and payment method ownership are documented in [ADR-0011](../../architecture/decisions/ADR-0011-checkout-level-payment-ownership.md).

## Detail

- Domain model: `domain.md`
- Public API: `api.md`
- Data and events: `data-and-events.md`
- Workflows: `workflows.md`
