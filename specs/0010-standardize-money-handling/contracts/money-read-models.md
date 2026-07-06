# Contract - Money Read Models

## Standard response shape

Money-bearing read models should converge on one explicit shape for frontend rendering:

```json
{
  "amount": 1599000,
  "currencyCode": "VND",
  "isValid": true,
  "issueCode": null
}
```

If the record is readable but the stored currency is missing or unsupported:

```json
{
  "amount": 1599000,
  "currencyCode": null,
  "isValid": false,
  "issueCode": "missing_currency"
}
```

## Affected endpoint families

### CatalogService

- Seller product detail/list responses
- Buyer storefront product detail and summary responses

Required behavior:

- Stop returning money values that force the client to infer `VND`
- Normalize currency representation away from numeric enum assumptions in app types

### OrderService

- Coupon detail/list responses
- Cart summary response
- Checkout preview response
- Order detail/list responses

Required behavior:

- Coupon responses must carry one canonical coupon `currencyCode`
- Cart and checkout payloads must expose one calculation currency or explicit invalid state
- Mixed-currency contexts fail on write/calculation flows instead of returning guessed totals

### PaymentService

- Payment detail/by-order responses
- Payment result UI payloads

Required behavior:

- Preserve explicit payment currency
- Do not fall back to `VND` when response currency is null or unsupported

## Frontend formatting rules

- `VND` renders with no fractional digits
- `USD` stored as cents renders by default as dollar major units with standard USD symbol/code and two fractional digits where needed
- `EUR` stored as cents renders by default as euro major units with standard EUR symbol/code and two fractional digits where needed
- Invalid or missing currency renders an explicit placeholder, not a guessed symbol or locale-only amount
- Shared formatter APIs may expose an optional explicit raw-smallest-unit mode, but normal user-facing rendering must not show `USD`/`EUR` raw cent values by default

## Frontend input rules

- Shared money inputs accept major-unit editing for `USD` and `EUR` and convert the parsed value to smallest-unit payloads before submit
- Shared money inputs accept whole-unit editing for `VND` and do not allow fractional digits by default
- Shared money input and shared money display helpers must use the same currency precision rules so edit/display round-trips stay lossless
- Generic integer-only number input helpers must not be used for user-facing money entry in `USD` or `EUR`

## Backward-compatibility notes

- Existing historical records in now-disabled currencies remain readable if the currency code is still recognized
- Existing malformed records remain readable only through invalid-money metadata and placeholder rendering
- This feature does not introduce amount conversion between currencies
- Coupon contract compatibility does not require migration/backfill in this feature; dev seed and fixture data should be updated to the new canonical coupon currency model
