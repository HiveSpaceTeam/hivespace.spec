# Research - Standardize System Money Handling

## Decision 1: UserService owns the persisted platform configuration foundation

- Decision: Store a generic platform-configuration foundation in UserService, with a config-level record for default/version state and config-type-specific item records for selectable entries. Currency is the first implemented config type and is exposed through admin management and authenticated read APIs in this feature.
- Rationale: The clarified feature requirement already assigns ownership to UserService, and the constitution allows UserService to own platform-facing profile/settings style data without moving commerce aggregates out of CatalogService, OrderService, or PaymentService. The current admin configuration UI already behaves like a catalog of configurable items, so a reusable list-based foundation fits the product direction better than a currency-only aggregate.
- Alternatives considered:
  - OrderService ownership: rejected because coupons and checkout are only part of the scope; product pricing and admin configuration would become awkward supporting concerns.
  - PaymentService ownership: rejected because gateway/payment ownership does not justify platform-wide governance over product and coupon authoring.
  - Config repo or appsettings ownership: rejected because the feature requires admin-managed persisted data, not deployment-time configuration.

## Decision 2: Distribute configuration changes through an integration event plus local `Ref` models

- Decision: Publish a currency-configuration update event from UserService and have CatalogService, OrderService, and PaymentService maintain local currency-configuration refs for synchronous validation.
- Rationale: The constitution forbids direct cross-service database reads. Product, coupon, checkout, and payment validations must execute locally and consistently, so projection-based reads are safer and more resilient than runtime HTTP dependency chains between services.
- Alternatives considered:
  - Synchronous HTTP calls from each service to UserService: rejected because it adds runtime coupling to every create/update/check workflow and makes validation dependent on another service's availability.
  - Shared database/config table: rejected because it violates service-owned data boundaries.
  - Hard-coded currency list in each service: rejected because the feature explicitly requires one persisted admin-managed configuration.

## Decision 3: Do not introduce exchange-rate conversion or cross-currency totals

- Decision: Reject mixed-currency cart, coupon, checkout, and payment contexts instead of converting values.
- Rationale: The spec explicitly excludes conversion and requires mixed-currency prevention. This keeps the feature tractable and avoids introducing rate sourcing, historical rate snapshots, or rounding policy decisions.
- Alternatives considered:
  - Runtime conversion using static rates: rejected because it would still require a rate owner, update strategy, and reconciliation rules.
  - Allow mixed-currency carts and convert only at checkout: rejected because it breaks the requirement to prevent mixed-currency state before preview or checkout.

## Decision 4: Standardize API money metadata instead of letting clients infer currency

- Decision: Normalize money-bearing responses to explicit smallest-unit amount plus ISO currency code and attach invalid-money diagnostics when stored currency is missing or unsupported.
- Rationale: Current implementations show multiple inconsistencies: numeric enum currency shapes in some product types, `VND` fallback behavior in checkout/payment flows, and frontend pages that guess symbols by locale or hard-coded currency. Explicit API metadata is the simplest contract for consistent rendering.
- Alternatives considered:
  - Keep existing inconsistent response shapes and patch each frontend page: rejected because the underlying contract drift would remain.
  - Return preformatted money strings only: rejected because clients still need structured values for calculations, tests, and placeholder handling.

## Decision 5: Add shared frontend money display and input helpers in `@hivespace/shared`

- Decision: Create one shared money-formatting and money-input helper/composable set in `@hivespace/shared` and replace app-local `formatMoney`, `formatCurrency`, hard-coded dong-suffix patterns, and generic integer-only number formatters on money fields. Display defaults `USD` and `EUR` smallest-unit amounts to major-unit dollar/euro output, while input accepts major-unit user entry and converts it to smallest-unit payloads.
- Rationale: The source audit found scattered money formatting in admin (`BuyersPage`), seller (`CouponListPage`, `CouponDetailPage`, `ProductListPage`, `OrderManagementPage`), and buyer (`CartPage`, `CheckoutPage`, `OrderDetailPage`, `OrdersPage`, `ProductDetailPage`, `PaymentResultPage`, and shared coupon/product widgets). The shared package also currently exposes a generic `useNumberInputFormatter` that is integer-only and not currency-aware, so it cannot safely handle dollar/euro entry without cent-conversion bugs.
- Alternatives considered:
  - Leave formatting local to each app: rejected because it caused the current inconsistency.
  - Use locale-only formatting without currency metadata: rejected because `VND`, `USD`, and `EUR` need different symbols and precision rules.

## Decision 6: Use one aggregate currency for coupon money fields

- Decision: Standardize coupon storage and write contracts on one canonical aggregate `CurrencyCode`, while persisting related coupon money fields as smallest-unit amounts that inherit that currency rather than repeating currency on each persisted money sub-value.
- Rationale: Coupon currently carries multiple related money values that must always share one currency. Repeating currency across each persisted value object or app-layer write field adds duplication and makes inconsistent states easier to create. A single aggregate currency matches the business invariant more directly.
- Alternatives considered:
  - Keep repeated per-money currency storage: rejected because it duplicates the same invariant across several values and leaves more room for drift.
  - Apply the same refactor to every money-bearing entity immediately: rejected because this feature only needs it for coupon-like multi-money-field aggregates; single-money-field entities can keep the existing `Money` model for now.

## Decision 7: Do not add migration/backfill work in this feature

- Decision: Update seed and fixture data only; do not add migration or historical backfill work for the coupon storage reshaping in this feature.
- Rationale: The project is still in development phase, and the user explicitly prefers seed-data updates over migration complexity. This keeps the feature focused on the new steady-state model instead of carrying temporary compatibility work.
- Alternatives considered:
  - Add a database migration and backfill logic now: rejected because it adds work with no meaningful value in the current dev-only lifecycle.

## Decision 8: No new saga design artifact is required

- Decision: Do not create `saga-design.md` for this feature.
- Rationale: The checkout saga already exists. This feature only changes currency validation and payload correctness before or within existing requests; it does not add new saga states, new async participants, new compensation paths, or new timeout handling.
- Alternatives considered:
  - Create a new saga for configuration propagation: rejected because this is a simple integration-event projection pattern, not a long-running workflow.
  - Modify checkout saga structure for currency validation: rejected because validation can happen before or inside existing handlers without changing the state machine.

## Decision 9: Keep config-level default/version state separate from per-item rows

- Decision: Keep default selection and optimistic versioning on a generic config-level record rather than on individual currency rows.
- Rationale: One config-level record gives the admin save flow a single concurrency token and one coherent snapshot for downstream projections. The row set itself remains flexible for future config types such as payment methods or providers.
- Alternatives considered:
  - Put `IsDefault` on one currency row: rejected because it complicates set-wide concurrency and makes future config types harder to standardize.
  - Store default redundantly on both a config row and item rows: rejected because it introduces avoidable consistency risk.

## Decision 10: Keep config-type item models typed, not free-form JSON

- Decision: Use a common config foundation for identity/version/default behavior, but keep item-specific models typed per config type instead of introducing a generic JSON settings blob.
- Rationale: Currency rows in this feature only need typed fields such as code, enabled state, and display order. Typed item models preserve validation clarity and keep the first implementation simpler.
- Alternatives considered:
  - Generic JSON payload on each item row: rejected because it weakens validation and adds complexity before there is a concrete cross-type need.
  - Delay the foundation entirely and stay currency-only: rejected because the admin configuration UI and expected future payment-method/provider work already point toward reusable list-based configuration.
