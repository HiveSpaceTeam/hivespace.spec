# Contract: Payment APIs

## GET `/api/v1/payments/{paymentId}`

Auth: `Authorize`

Returns payment detail with shared reference, linked orders, latest attempt, and attempt history.

### Response `200 OK`

```json
{
  "id": "4a11a1d4-3839-41e7-9857-4a994f7d79c5",
  "referenceNo": "PAY-01JZXYZABCDEABCDEABCDEABC",
  "buyerId": "f6f1c282-49d5-49d0-a0f1-3f77887bb156",
  "method": {
    "code": "VNPAY",
    "displayName": "VNPay",
    "kind": "Online",
    "availability": "Available"
  },
  "status": "Succeeded",
  "amount": {
    "amount": 12500000,
    "currencyCode": "VND"
  },
  "gateway": {
    "code": "VNPAY",
    "merchantReference": "PAY-01JZXYZABCDEABCDEABCDEABC",
    "gatewayTransactionId": "VNPAY-TRANSACTION-ID"
  },
  "latestAttempt": {
    "id": "1f7ad774-5345-42f5-b866-2e56176e1ad3",
    "attemptNo": 2,
    "methodCode": "VNPAY",
    "gatewayCode": "VNPAY",
    "status": "Succeeded",
    "redirectUrl": null,
    "gatewayTransactionId": "VNPAY-TRANSACTION-ID",
    "failureReasonCode": null,
    "createdAt": "2026-07-11T10:15:00Z",
    "expiresAt": "2026-07-11T10:30:00Z",
    "completedAt": "2026-07-11T10:18:00Z"
  },
  "attempts": [
    {
      "id": "6e6fda1d-6c5b-4074-b15a-6571b525d54c",
      "attemptNo": 1,
      "methodCode": "VNPAY",
      "gatewayCode": "VNPAY",
      "status": "Expired",
      "redirectUrl": null,
      "gatewayTransactionId": null,
      "failureReasonCode": "Expired",
      "createdAt": "2026-07-11T09:55:00Z",
      "expiresAt": "2026-07-11T10:10:00Z",
      "completedAt": "2026-07-11T10:10:00Z"
    }
  ],
  "linkedOrders": [
    {
      "orderId": "3181d3ff-c4f8-4ab9-92df-1fb65cb2f8a5",
      "orderCode": "ORD-01JZXYZABCDEABCDEABCDEABD",
      "storeId": "88c559cb-a60f-411b-9c21-00503150f2bb",
      "amount": {
        "amount": 6500000,
        "currencyCode": "VND"
      }
    }
  ]
}
```

Buyer-facing reads may omit full `attempts` when the caller is not privileged, but must include `latestAttempt`. Admin/support reads must include full attempt history.

## POST `/api/v1/payments/{paymentId}/attempts`

Auth: `Authorize`

Creates a new payment attempt for an existing checkout-level payment after a failed, expired, or cancelled attempt.

### Request

```json
{
  "methodCode": "VNPAY",
  "idempotencyKey": "buyer-retry-key"
}
```

Rules:

- Allowed only when no attempt has succeeded and linked orders are still eligible for payment.
- `VNPAY` creates a new gateway attempt and returns a redirect URL.
- `COD` creates an offline attempt only before fulfillment starts.
- `STRIPE` is rejected until Stripe checkout is implemented.
- Repeating the same idempotency key with the same request shape returns the same attempt.

### Response `201 Created`

```json
{
  "paymentId": "4a11a1d4-3839-41e7-9857-4a994f7d79c5",
  "referenceNo": "PAY-01JZXYZABCDEABCDEABCDEABC",
  "attempt": {
    "id": "1f7ad774-5345-42f5-b866-2e56176e1ad3",
    "attemptNo": 2,
    "methodCode": "VNPAY",
    "gatewayCode": "VNPAY",
    "status": "Processing",
    "redirectUrl": "https://sandbox.vnpay.vn/payment-url",
    "createdAt": "2026-07-11T10:15:00Z",
    "expiresAt": "2026-07-11T10:30:00Z"
  }
}
```

## GET `/api/v1/payments/by-reference/{referenceNo}`

Auth: `Authorize`

Looks up a payment by public `PAY-{ULID}` reference for support/admin reconciliation. Authorization must enforce buyer ownership or privileged access according to existing policies.

## GET `/api/v1/payments/by-order/{orderId}`

Auth: `Authorize`

Returns the shared checkout payment for any linked order. Existing endpoint behavior changes from one-order payment lookup to linked checkout payment lookup.

## Gateway callbacks

Existing callbacks stay in place:

- `GET /api/v1/payments/vnpay/return`
- `GET /api/v1/payments/webhook/{gateway}`

VNPay initiation must send `referenceNo` as the merchant transaction reference and include attempt identity in service-side callback correlation. Gateway transaction IDs are stored on the attempt after return/IPN.
