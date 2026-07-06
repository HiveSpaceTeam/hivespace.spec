# PaymentService

## Responsibility

PaymentService owns payment processing, payment gateway interaction, wallets, and wallet transactions.

Source path:

```text
../hivespace.microservice/src/HiveSpace.PaymentService
```

## Owns

- Payment aggregate lifecycle.
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
- Payment initiation validates explicit order currency against the local currency-policy projection and fails closed when that projection is missing, unreadable, or stale for a write path.
- Payment read models return explicit money metadata and invalid-money diagnostics instead of inferring `VND`.
- OrderService reacts to payment outcomes; PaymentService does not mutate order state directly.
- Wallet ledger entries should be append-only.
- Payment workflow events use standardized `*IntegrationEvent` names and service-owned publisher policy per [ADR-0002](../../architecture/decisions/ADR-0002-standardized-integration-event-contracts.md).
- Currency-policy ownership remains in UserService per [ADR-0010](../../architecture/decisions/ADR-0010-user-service-platform-currency-policy.md); PaymentService only consumes the projection for validation.

## Detail

- Domain model: `domain.md`
- Public API: `api.md`
- Data and events: `data-and-events.md`
- Workflows: `workflows.md`
