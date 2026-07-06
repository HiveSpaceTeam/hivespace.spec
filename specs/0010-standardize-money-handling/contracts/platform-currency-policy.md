# Contract - Platform Currency Policy

This contract remains currency-specific for feature `0010`, but the payload shape is intentionally compatible with a broader platform-configuration foundation: config-level metadata plus typed currency item rows.

## Admin management API

### GET `/api/v1/admins/configuration/currencies`

Auth: `RequireAdmin`

Response:

```json
{
  "currencies": [
    { "currencyCode": "VND", "isEnabled": true },
    { "currencyCode": "USD", "isEnabled": true },
    { "currencyCode": "EUR", "isEnabled": false }
  ],
  "defaultCurrencyCode": "VND",
  "supportedCurrencyCodes": ["VND", "USD", "EUR"],
  "version": 3,
  "updatedAt": "2026-06-29T10:30:00Z"
}
```

### PUT `/api/v1/admins/configuration/currencies`

Auth: `RequireAdmin`

Request:

```json
{
  "currencies": [
    { "currencyCode": "VND", "isEnabled": true },
    { "currencyCode": "USD", "isEnabled": true },
    { "currencyCode": "EUR", "isEnabled": true }
  ],
  "defaultCurrencyCode": "USD",
  "version": 3
}
```

Validation:

- `defaultCurrencyCode` must refer to a currency record whose `isEnabled` is `true`
- All codes must be within `VND`, `USD`, `EUR`
- At least one enabled currency record is required
- Reject disabling the current default unless a replacement default is saved in the same request

Error conditions:

- Disabled default conflict
- Unsupported currency code
- Empty enabled-currency set
- Concurrency/version mismatch

## Authenticated read API

### GET `/api/v1/users/platform-currency-policy`

Auth: `RequireAdminOrUser`

Purpose:

- Supply admin, seller, and buyer authenticated clients with the current enabled/default policy
- Support seller coupon/product authoring flows without duplicating currency configuration in each app
- Allow clients to preselect the current platform default currency in eligible authoring or management flows without treating it as a backend write fallback

Response:

```json
{
  "currencies": [
    { "currencyCode": "VND", "isEnabled": true },
    { "currencyCode": "USD", "isEnabled": true },
    { "currencyCode": "EUR", "isEnabled": true }
  ],
  "defaultCurrencyCode": "USD",
  "supportedCurrencyCodes": ["VND", "USD", "EUR"],
  "version": 4
}
```

## Integration event

### `PlatformCurrencyPolicyUpdatedIntegrationEvent`

Producer: UserService

Consumers: CatalogService, OrderService, PaymentService

Payload:

```json
{
  "policyId": "6bdbff54-d0a9-42a6-bdb1-e6d2ccf2f312",
  "currencies": [
    { "currencyCode": "VND", "isEnabled": true },
    { "currencyCode": "USD", "isEnabled": true },
    { "currencyCode": "EUR", "isEnabled": true }
  ],
  "defaultCurrencyCode": "USD",
  "version": 4,
  "updatedAt": "2026-06-29T10:45:00Z"
}
```

Consumer expectations:

- Apply idempotently by `EventId` and/or `version`
- Update local validation projection only
- Do not mutate foreign domain data directly

## Admin UI behavior notes

- Render one row or card per persisted currency record rather than a hard-coded chip list.
- Use a dedicated toggle for `isEnabled` and a separate explicit control for choosing the default currency.
- Show the current default as a badge, but do not rely on the badge itself as the only way to understand or change the default.
- Treat `defaultCurrencyCode` as a client-visible preselection/default-display hint only unless a consuming flow defines another explicit use.
- Do not rely on `defaultCurrencyCode` as permission for backend APIs to accept missing money currency fields.

