# PaymentService API

## Payments

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/payments/vnpay/return` | Anonymous | Handle VNPay browser return |
| GET | `/api/v1/payments/webhook/{gateway}` | Anonymous | Handle payment gateway webhook/IPN |
| GET | `/api/v1/payments/{paymentId}` | `Authorize` | Get payment detail |
| GET | `/api/v1/payments/by-order/{orderId}` | `Authorize` | Get payment for an order |

## Wallets

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/wallets/me` | `Authorize` | Get current wallet balance |
| GET | `/api/v1/wallets/me/transactions` | `Authorize` | List wallet transactions |

## API Rules

- Payment gateway callbacks must be idempotent.
- Webhook/IPN endpoints should acknowledge gateway delivery and record internal errors for retry/diagnosis.
- Buyer-facing payment reads must enforce ownership.
