# Quickstart: Multi-Order Payment Methods

## Scope

Use this after implementation tasks exist. This is a planning quickstart, not runnable product code.

## Backend verification

1. Read `../hivespace.microservice/AGENTS.md` and `../hivespace.microservice/CLAUDE.md`.
2. Start backend dependencies through Aspire AppHost from `../hivespace.microservice`.
3. Run targeted PaymentService and OrderService tests for:
   - `PAY-{ULID}` uniqueness and collision retry.
   - `ORD-{ULID}` uniqueness and collision retry.
   - COD checkout creates one offline payment linked to all generated orders.
   - VNPay checkout creates one payment with one reference linked to all generated orders.
   - VNPay retry creates a new attempt under the same payment reference after failure/expiry.
   - COD retry/switch creates an offline attempt only before fulfillment starts.
   - Duplicate VNPay return/IPN does not duplicate order transitions.
   - Stale callback from an older attempt does not change orders after a newer attempt succeeds.
   - Payment amount/currency/order-set mismatch rejects initiation.
4. Run the target repo coverage flow for affected services and add tests if measured scope is below 80%.

## Frontend verification

1. Read `../hivespace.web/AGENTS.md` and `../hivespace.web/CLAUDE.md`.
2. Confirm buyer checkout loads methods from `GET /api/v1/payments/methods`.
3. Confirm buyer checkout allows COD and VNPay only.
4. Confirm buyer retry after failed/expired VNPay uses the same payment reference and new attempt.
5. Confirm Stripe appears as unavailable/future metadata outside checkout where payment methods are displayed.
6. Confirm buyer, seller, and admin surfaces display the same canonical method labels.
7. Confirm order detail shows `OrderCode`, shared `PaymentReferenceNo`, and latest payment attempt state.
8. Update and verify English and Vietnamese i18n resources together.

## User-owned E2E

The user should run the browser journey for a cart that splits into two or more orders:

1. Checkout with VNPay.
2. Complete payment once.
3. Open each generated order.
4. Confirm every order has a distinct `ORD-{ULID}` and the same `PAY-{ULID}` payment reference.
