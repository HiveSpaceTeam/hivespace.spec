# Contract: Payment Methods API

## GET `/api/v1/payments/methods`

Returns PaymentService-owned canonical payment method metadata for buyer, seller, and admin apps.

Auth: `Authorize`

### Response `200 OK`

```json
{
  "methods": [
    {
      "code": "COD",
      "displayName": "Cash on delivery",
      "kind": "Offline",
      "gatewayCode": null,
      "isEnabled": true,
      "isCheckoutSelectable": true,
      "availability": "Available",
      "sortOrder": 10
    },
    {
      "code": "VNPAY",
      "displayName": "VNPay",
      "kind": "Online",
      "gatewayCode": "VNPAY",
      "isEnabled": true,
      "isCheckoutSelectable": true,
      "availability": "Available",
      "sortOrder": 20
    },
    {
      "code": "STRIPE",
      "displayName": "Stripe",
      "kind": "Online",
      "gatewayCode": "STRIPE",
      "isEnabled": false,
      "isCheckoutSelectable": false,
      "availability": "Future",
      "sortOrder": 30
    }
  ]
}
```

Rules:

- Buyer checkout must only allow methods where `isEnabled` and `isCheckoutSelectable` are both true.
- Non-checkout displays may show unavailable/future methods.
- Clients must not hardcode a separate method list.
