# PaymentService Workflows

## Online Payment Initiation

```text
OrderService CheckoutSaga
  -> InitiatePayment
  -> PaymentService validates checkout correlation, linked orders, method, amount, and currency against local currency-policy projection
  -> PaymentService creates one checkout-level payment with PAY-{ULID} reference
  -> PaymentService creates attempt 1
  -> for VNPay: requests gateway URL using ReferenceNo as merchant reference
  -> for COD: records an offline attempt without gateway URL
  -> PaymentInitiatedIntegrationEvent or PaymentInitiationFailedIntegrationEvent
```

## Payment Retry

```text
Buyer app
  -> POST /api/v1/payments/{paymentId}/attempts
  -> PaymentService validates no attempt has succeeded and linked orders remain eligible
  -> creates next attempt under the same payment ReferenceNo
  -> returns COD offline attempt or VNPay redirect URL
```

## Gateway Return / IPN

```text
Gateway
  -> /api/v1/payments/vnpay/return or /api/v1/payments/webhook/{gateway}
  -> PaymentService validates gateway payload
  -> current attempt status changes once
  -> payment status changes only for the effective current attempt
  -> PaymentSucceededIntegrationEvent or PaymentFailedIntegrationEvent

Payment read APIs must preserve the stored payment currency explicitly and surface invalid-money diagnostics rather than inferring `VND` when historical data is malformed.
```

## Workflow Rules

- Treat gateway callbacks as potentially duplicated.
- Ignore stale callbacks from older attempts for order-changing outcomes after a newer attempt exists or the payment has succeeded.
- Store raw gateway response where supported for diagnostics.
- Publish payment result events once per effective current-attempt outcome and include attempt identity.
- Publish payment integration events through the service-owned payment event publisher outside saga consume-context responses.
- Do not mutate OrderService data directly.
