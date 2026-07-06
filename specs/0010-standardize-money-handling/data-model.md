# Data Model - Standardize System Money Handling

## 1. PlatformConfig

Owning service: UserService

| Field | Type | Notes |
| --- | --- | --- |
| `Id` | `Guid` or service-standard identifier | Single config/settings row for one config type |
| `ConfigType` | `string` | `currency` for this feature |
| `DefaultCurrencyCode` | `string` | Must be one of `VND`, `USD`, `EUR` |
| `Version` | `long` or concurrency token | Used for optimistic update handling and projection freshness |
| `CreatedAtUtc` | timestamp | Audit |
| `UpdatedAt` | timestamp | Audit |

Validation rules:

- Default currency must always be enabled.
- Supported currencies are fixed to `VND`, `USD`, `EUR` for this feature.
- Disabling the active default currency is invalid until another enabled default is saved.
- At least one enabled `PlatformCurrency` record must remain.

State transitions:

- `ChangeDefaultCurrency(code)`
- `ValidateDefaultCurrency(code)`
- `BumpVersion()`

## 2. PlatformCurrency

Owning service: UserService

| Field | Type | Notes |
| --- | --- | --- |
| `Id` | `Guid` or service-standard identifier | One record per supported platform currency |
| `CurrencyCode` | `string` | ISO code in current feature scope |
| `IsEnabled` | `bool` | True when allowed for new commerce operations |
| `DisplayOrder` | `int` | Optional stable UI ordering for admin and form consumers |
| `CreatedAtUtc` | timestamp | Audit |
| `UpdatedAt` | timestamp | Audit |

Validation rules:

- Duplicate currency codes are invalid.
- Unknown currency codes are invalid.
- At least one record must stay enabled across the set.
- If `IsEnabled` is set to `false`, `CurrencyCode` must not match the current `PlatformConfig.DefaultCurrencyCode`.

## 3. PlatformCurrencyConfigRef

Owning services: CatalogService, OrderService, PaymentService (local consumer-side ref only)

| Field | Type | Notes |
| --- | --- | --- |
| `PolicyVersion` | `long` | Last processed event version |
| `DefaultCurrencyCode` | `string` | Current platform default |
| `Currencies` | collection or child rows | Each entry carries `CurrencyCode` plus `IsEnabled` for local validation |
| `UpdatedAt` | timestamp | Observability and replay support |

Validation rules:

- Projection updates must be idempotent by policy version or event id.
- Services reject new writes if the requested currency is not enabled in the current `PlatformCurrencyConfigRef`.

## 4. MoneyValue

Existing shared value object in backend domain model.

| Field | Type | Notes |
| --- | --- | --- |
| `Amount` | `long` | Smallest-unit amount |
| `Currency` | enum/value | Must be normalized to explicit code in API responses |

Feature-specific rules:

- No flow may silently replace a missing or unsupported currency with `VND`.
- New writes must use an enabled currency from the current `PlatformCurrencyConfigRef`.
- Historical disabled-currency values remain readable if the stored currency is still recognized.
- Backend factories must distinguish smallest-unit integer creation from major-unit decimal creation so `USD` and `EUR` values are not inflated by ambiguous factory calls.

## 5. CouponCurrencyContext

Owning service: OrderService

| Field | Type | Notes |
| --- | --- | --- |
| `CurrencyCode` | `string` | Canonical coupon currency used by every coupon money amount |

Validation rules:

- `CurrencyCode` must be enabled in the current platform currency configuration for new and updated coupons.
- `CurrencyCode` must be within the supported feature scope.

## 6. CouponMoneyFields

Owning service: OrderService

| Field | Type | Notes |
| --- | --- | --- |
| `DiscountAmount` | `long?` | Smallest-unit amount for fixed discounts; interpreted using `CouponCurrencyContext.CurrencyCode` |
| `MaxDiscountAmount` | `long?` | Optional smallest-unit cap for percentage discounts; interpreted using `CouponCurrencyContext.CurrencyCode` |
| `MinOrderAmount` | `long` | Smallest-unit qualifying threshold; interpreted using `CouponCurrencyContext.CurrencyCode` |

Validation rules:

- `DiscountAmount`, `MaxDiscountAmount`, and `MinOrderAmount` must not persist separate coupon-specific currencies.
- New and updated coupons cannot use a disabled currency.
- Coupon application to cart/order contexts fails if the coupon currency conflicts with the cart/order currency.
- Domain behavior may materialize transient `MoneyValue` instances from `(amount, CouponCurrencyContext.CurrencyCode)` for comparison and arithmetic, but persisted coupon storage stays on one aggregate currency plus smallest-unit amounts.

## 7. ProductPrice

Owning service: CatalogService

| Field | Type | Notes |
| --- | --- | --- |
| `SkuPrice` | `MoneyValue` | Seller-authored SKU price |
| `OriginalPrice` | `MoneyValue?` | Optional compare-at price if already modeled |

Validation rules:

- New and updated prices must use an enabled currency.
- Storefront and seller product read models must expose explicit amount + currency code.
- Invalid stored currency remains readable only through invalid-money metadata, not fallback formatting.

## 8. Calculation Context

Owning services: OrderService and PaymentService

| Field | Type | Notes |
| --- | --- | --- |
| `CurrencyCode` | `string` | Currency shared by all values participating in one calculation |
| `SourceValues` | collection of money-bearing items | Cart items, coupon thresholds, totals, payment amount |
| `Status` | enum or implicit validation result | Valid, mixed currency, disabled currency, invalid currency |

Validation rules:

- Mixed-currency cart or coupon application is rejected at the action that introduces the conflict.
- Checkout preview/initiation fails when calculation context is mixed, disabled, or invalid.
- Payment initiation fails when order/payment currency is disabled or malformed.

## 9. MoneyDisplayMetadata

Frontend/backend response contract helper.

| Field | Type | Notes |
| --- | --- | --- |
| `amount` | `long` | Smallest-unit amount |
| `currencyCode` | `string or null` | Explicit currency code |
| `isValid` | `bool` | False when currency is missing or unsupported |
| `issueCode` | `string or null` | Example: `missing_currency`, `unsupported_currency`, `mixed_currency_context` |
| `displayPlaceholder` | `string or null` | Optional explicit invalid-money placeholder |
| `displayMode` | `string or null` | Optional display hint such as `major_unit_default` or `raw_smallest_unit` |

Usage rules:

- Used in read models when the parent record must stay readable despite malformed money.
- Not used to hide domain failures on write/update flows; writes fail explicitly instead.
- Frontend shared formatting defaults `USD` and `EUR` smallest-unit amounts to major-unit display (`dollars`/`euros`) rather than raw cents.
- Formatter APIs may expose an optional explicit raw-smallest-unit mode for admin/debug or specialized displays, but ordinary user-facing surfaces should not use it by default.

## 10. MoneyInputModel

Frontend shared input contract helper.

| Field | Type | Notes |
| --- | --- | --- |
| `currencyCode` | `string` | Determines decimal precision, separators, and smallest-unit conversion |
| `displayValue` | `string` | User-edited text in major units for `USD`/`EUR` and whole units for `VND` |
| `smallestUnitAmount` | `long or null` | Parsed output ready for API submission |
| `inputMode` | `string` | Default `major_unit`; optional `raw_smallest_unit` only for explicit exceptional usage |

Usage rules:

- Shared frontend money inputs must parse and format using the same currency rules as shared display helpers.
- `USD` and `EUR` inputs default to major-unit editing with two fractional digits.
- `VND` inputs default to whole-unit editing with no fractional digits.

## 11. Admin Currency Configuration View Model

Frontend/admin helper model, backed by persisted data from UserService.

| Field | Type | Notes |
| --- | --- | --- |
| `currencies[]` | collection | One row/card per currency record |
| `currencies[].currencyCode` | `string` | Example: `VND`, `USD`, `EUR` |
| `currencies[].isEnabled` | `bool` | Drives toggle/switch state |
| `defaultCurrencyCode` | `string` | Selected separately from enable/disable state |
| `version` | `long` | Used for optimistic save |

Usage rules:

- The admin UI should treat currencies as a list of persisted records, not as a hard-coded chip array.
- Default-currency selection must be explicit in the UI; showing a badge only is insufficient.
- Disabling the current default currency must either be blocked immediately or require selecting a replacement default in the same save flow.

