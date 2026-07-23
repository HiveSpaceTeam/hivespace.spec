# PaymentService API

## Payments

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/payments/methods` | `Authorize` | Get PaymentService-owned canonical payment method metadata for COD, VNPay, and future/unavailable Stripe across buyer, seller, and admin apps |
| GET | `/api/v1/payments/vnpay/return` | Anonymous | Handle VNPay browser return |
| GET | `/api/v1/payments/webhook/{gateway}` | Anonymous | Handle payment gateway webhook/IPN |
| GET | `/api/v1/payments/{paymentId}` | `Authorize` | Get checkout-level payment detail with payment reference number, linked orders, canonical method metadata, explicit money metadata, latest attempt, and privileged attempt history when authorized |
| GET | `/api/v1/payments/by-reference/{referenceNo}` | `Authorize` | Get checkout-level payment detail by public `PAY-{ULID}` reference for support/admin reconciliation |
| GET | `/api/v1/payments/by-order/{orderId}` | `Authorize` | Get the shared checkout-level payment linked to an order |
| POST | `/api/v1/payments/{paymentId}/attempts` | `Authorize` | Create an idempotent COD or VNPay retry attempt under an existing checkout-level payment |

## Wallets

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/wallets/me` | `Authorize` | Get current wallet balance |
| GET | `/api/v1/wallets/me/transactions` | `Authorize` | List wallet transactions |

## API Rules

- Payment gateway callbacks must be idempotent.
- Webhook/IPN endpoints should acknowledge gateway delivery and record internal errors for retry/diagnosis.
- Buyer-facing payment reads must enforce ownership.
- Admin/support reads may include full attempt history; buyer-facing reads must at least include the latest attempt.
- `POST /api/v1/payments/{paymentId}/attempts` is allowed only after failed, expired, or cancelled attempts, while linked orders remain eligible and no attempt has succeeded.
- Buyer checkout must only allow methods where PaymentService metadata says the method is enabled and checkout-selectable.
