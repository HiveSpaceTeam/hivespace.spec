# Feature Specification: Standardize System Money Handling

- **Feature Branch**: `0010-standardize-money-handling`
- **Created**: 2026-06-28
- **Status**: Implemented
- **Implemented**: 2026-07-06
- **Input**: User description: "$speckit-specify I want standardize the way to handle money in this system, include Coupon aggregate has multiple currency field, for DiscountAmount, MaxDiscountAmount, MinOrderAmount; multiple place in the microservice is fixing the currency to VND, instead of check or use USD or EUR, need consistency; the way to display money in fe also not consistency, some use d for VND, not check the amount, like in CouponList or CouponDetail, Product Detail, need to check all project or maybe create share method or composable to handle it; update Configuration page in admin to manage (enable, disable) currency to use in the system"

## Wrap-Up

- User-owned E2E follow-up remains explicit in `tasks/verification.md` for `V004` and `V005`. This spec is marked implemented, but those user-run end-to-end validations are not marked complete here on the user's behalf.

## Clarifications

### Session 2026-06-29

- Q: How broad should the frontend money-formatting consistency pass be? -> A: Audit and standardize all currently shipped money-displaying surfaces across admin, seller, and buyer apps.
- Q: What should happen when an admin tries to disable the current default currency? -> A: Block disabling the default currency until the admin selects another enabled currency as the new default.
- Q: Which service should own the persisted platform configuration foundation and admin configuration API? -> A: UserService owns the persisted platform configuration foundation and the admin configuration API.
- Q: How should read flows handle stored money values whose currency is missing or unsupported? -> A: Return/display the record, but show an explicit invalid-money placeholder and log/flag the issue.
- Q: When should the platform block mixed-currency cart or coupon state? -> A: Prevent carts or coupon applications from creating a mixed-currency state in the first place.
- Q: How should shared frontend money handling treat editable money inputs? -> A: Shared frontend money handling must cover both display and input; `USD`/`EUR` inputs use major-unit UX and convert to smallest-unit payloads underneath.
- Q: How should coupon-like aggregates store related money values that must always share one currency? -> A: Standardize on one canonical aggregate currency field instead of repeating currency across each persisted money value in that aggregate.
- Q: Should this feature include data migration for the new coupon money storage model? -> A: No; the system is still in dev phase, so seed and fixture data should be updated instead of adding migration/backfill work.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Manage Platform Currencies (Priority: P1)

A platform admin manages the currency entries in the platform configuration workspace, enabling or disabling supported currencies and selecting the default so all services and clients follow one source of truth.

**Why this priority**: Currency governance is the root decision for the rest of the feature. Without an authoritative enabled-currency configuration, backend validations and frontend rendering remain inconsistent.

**Independent Test**: Can be fully tested by editing persisted currency configuration rows in the admin configuration workspace and verifying that the saved configuration is reflected consistently in downstream admin, seller, and buyer workflows.

**Acceptance Scenarios**:

1. **Given** a platform admin opens currency configuration, **When** the admin enables `USD` and `EUR` and keeps `VND` as default, **Then** the platform stores that policy and exposes it consistently to all supported clients.
2. **Given** a currency is disabled by the platform admin, **When** a user attempts to create or update new commerce data with that currency, **Then** the system rejects the operation and explains that the currency is not currently enabled.
3. **Given** historical records already exist in a currency that is later disabled, **When** users view those records, **Then** the records remain readable and their amounts remain visible without allowing new or updated use of that disabled currency.
4. **Given** the current default currency is still enabled, **When** a platform admin attempts to disable it without first selecting another enabled default currency, **Then** the system rejects the change and instructs the admin to choose a replacement default first.

---

### User Story 2 - Create And Maintain Currency-Aware Commerce Data (Priority: P2)

A seller can create and update coupons and product prices using an enabled currency, and the platform validates those values consistently instead of silently assuming `VND`.

**Why this priority**: Sellers are the main source of coupon and product price data. If create and update flows remain inconsistent, downstream checkout and display logic cannot be trusted.

**Independent Test**: Can be fully tested by creating and editing coupons and product prices in each enabled currency and confirming that validation, storage, retrieval, and later edits all preserve the selected currency correctly.

**Acceptance Scenarios**:

1. **Given** a seller creates a coupon with `USD`, **When** the seller enters `DiscountAmount`, `MaxDiscountAmount`, and `MinOrderAmount`, **Then** all coupon money values are stored and returned using the selected coupon currency consistently.
2. **Given** a seller updates a coupon that uses `EUR`, **When** the seller saves changes, **Then** the updated coupon keeps a single consistent money currency across all comparable coupon money fields.
3. **Given** a seller tries to use a disabled currency for a new coupon or product price, **When** the save is submitted, **Then** the system rejects the submission before that change becomes active.

---

### User Story 3 - View Money Consistently Across Apps (Priority: P3)

A buyer, seller, or admin sees money displayed consistently across storefront, seller, and admin interfaces, including correct symbols, decimal precision, and currency codes for the stored amount.

**Why this priority**: User trust drops when the same amount is displayed differently across pages or when fixed `VND` symbols appear for non-`VND` values.

**Independent Test**: Can be fully tested by viewing the same `VND`, `USD`, and `EUR` values across all currently shipped money-displaying surfaces in the admin, seller, and buyer apps and confirming the output format is consistent.

**Acceptance Scenarios**:

1. **Given** a money value is stored as `VND`, **When** it is shown in any supported app surface, **Then** it is rendered with the platform's standard `VND` display format and without fractional digits.
2. **Given** a money value is stored as `USD` or `EUR` using smallest-unit cents, **When** it is shown in any supported app surface, **Then** it is rendered by default in major units (`dollars`/`euros`), not raw cents, and with the correct precision.
3. **Given** a user compares money values across coupon detail, coupon list, and product detail views, **When** those pages load, **Then** the same currency is never shown with conflicting symbols or precision rules.
4. **Given** a seller edits a `USD` or `EUR` money field in a supported frontend form, **When** the seller types a major-unit value such as `10.50`, **Then** the shared money input parses it correctly and submits the matching smallest-unit amount instead of treating it as raw cents.

### Edge Cases

- Disabling the current default currency must be rejected until the admin saves another enabled currency as the new default.
- Cart and coupon application flows must reject any action that would introduce a mixed-currency calculation context before preview or checkout can proceed.
- Pages must keep the record readable when a stored money value has a missing or unsupported currency, but they must show an explicit invalid-money placeholder and keep the issue observable instead of guessing a currency.
- Historical coupons, orders, products, or payments created before the platform currency configuration change must remain readable without amount conversion or data rewriting.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST provide a UserService-owned platform configuration foundation for list-based platform settings, with one config-level record controlling version/default state and config-type-specific item records controlling the selectable entries.
- **FR-001a**: The currency capability in this feature MUST be implemented as the first config type on that foundation, using one persisted currency item record per supported currency plus one config-level record that stores the default currency and version.
- **FR-002**: Platform admins MUST be able to view and update the persisted currency configuration from the admin configuration workspace.
- **FR-002a**: The system MUST NOT allow the current default currency to be disabled until a different enabled currency has been saved as the new default.
- **FR-002b**: The persisted default currency MUST act as the platform's preselected currency for admin-managed configuration displays and seller/admin currency-authoring flows unless a specific consumer flow defines a different explicit behavior.
- **FR-002c**: Backend write APIs MUST NOT infer, substitute, or silently apply the persisted default currency when a required money currency is omitted; missing currency remains a validation error.
- **FR-003**: The system MUST support `VND`, `USD`, and `EUR` as the currencies in scope for this feature.
- **FR-004**: The system MUST preserve money values as an amount plus currency, and all currency-aware operations MUST use the stored currency instead of silently assuming `VND`.
- **FR-005**: Coupon create, update, retrieval, and validation flows MUST treat `DiscountAmount`, `MaxDiscountAmount`, and `MinOrderAmount` as currency-aware values that stay consistent with the coupon currency.
- **FR-006**: Product price create, update, retrieval, and validation flows MUST recognize enabled currencies and MUST reject new or updated product prices that use disabled currencies.
- **FR-007**: Checkout, cart, order, coupon, and payment workflows MUST reject mixed-currency calculation contexts instead of auto-converting or silently falling back to `VND`.
- **FR-007a**: Cart and coupon application flows MUST prevent users from creating a mixed-currency state before preview or checkout by rejecting the action that introduces the conflicting currency.
- **FR-008**: The system MUST NOT introduce exchange-rate conversion as part of this feature.
- **FR-009**: The system MUST allow historical records in disabled currencies to remain readable after policy changes.
- **FR-010**: The system MUST prevent new or updated coupons, product prices, checkout attempts, or payment initiations from using currencies that are not enabled by the current persisted currency configuration.
- **FR-010a**: Commerce services MUST fail closed for new or updated money-bearing writes when the local currency-policy projection required for validation is missing, unreadable, or older than the latest successfully applied policy version known to that service.
- **FR-011**: All user-facing money rendering in supported admin, seller, and buyer surfaces MUST follow one shared formatting rule for symbol, code, grouping, and decimal precision based on currency.
- **FR-011a**: The shared money formatter MUST treat stored `USD` and `EUR` amounts as cent-based smallest units and render them by default as major-unit `dollar` and `euro` amounts rather than raw cents.
- **FR-011b**: The shared money formatter MAY expose an optional explicit mode to render raw smallest-unit values when a surface intentionally needs that behavior, but normal user-facing display MUST default to major units for `USD` and `EUR`.
- **FR-011c**: Shared frontend money handling MUST also cover editable money inputs, with `USD` and `EUR` entered using major-unit UX and converted to smallest-unit values before submission.
- **FR-011d**: Shared frontend money inputs MUST keep `VND` as whole-unit entry without fractional digits.
- **FR-012**: All currently shipped frontend money-displaying surfaces in the admin, seller, and buyer apps, including coupon list, coupon detail, product detail, admin buyer-facing money pages such as `BuyersPage`, and other discovered money-bearing pages, MUST stop using hard-coded `VND` symbols or locale-only assumptions for non-`VND` values.
- **FR-013**: The admin configuration workspace MUST display the current enabled/default currency configuration using real persisted platform data rather than mock-only values.
- **FR-013a**: The admin configuration workspace MUST render currency entries as persisted list items with explicit enable/disable controls and a separate explicit default-currency selector rather than hard-coded currency chips alone.
- **FR-014**: The system MUST expose enough currency metadata in read models and user-facing responses for clients to render stored money values consistently.
- **FR-015**: When a money value cannot be safely used because its currency is missing, unsupported, or inconsistent with the calculation context, the system MUST fail the operation with an observable validation or business error rather than guessing a currency.
- **FR-016**: Read flows for historical or malformed records MUST keep the parent record readable, but any money value with a missing or unsupported currency MUST render as an explicit invalid-money placeholder and MUST remain observable through logging or equivalent diagnostics.
- **FR-017**: Coupon storage and write contracts MUST use one canonical coupon `CurrencyCode` for `DiscountAmount`, `MaxDiscountAmount`, and `MinOrderAmount` instead of storing or accepting repeated currency fields for each of those values.
- **FR-018**: Backend money factories and mapping code MUST distinguish smallest-unit integer creation from major-unit decimal creation so `USD` and `EUR` inputs cannot be inflated by incorrect cent-to-major-unit conversion.

### Key Entities *(include if feature involves data)*

- **Platform Configuration Record**: A UserService-owned config-level record that stores configuration metadata such as config type, default selected value, and optimistic version state for a list-based platform setting.
- **Platform Currency Item**: A UserService-owned currency configuration row that stores one supported currency and whether it is enabled for commerce use.
- **Money Value**: A monetary value represented by a smallest-unit amount and an associated currency.
- **Coupon Currency Context**: The single canonical coupon currency that governs every persisted coupon money amount in the aggregate.
- **Coupon Money Fields**: The coupon-specific smallest-unit amounts for discount amount, maximum discount amount, and minimum order amount that must stay currency-consistent under one coupon currency.
- **Currency-Aware Commerce Record**: Any coupon, product price, cart summary, checkout total, order total, or payment amount that carries a money value and must preserve its currency across storage, validation, and display.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of supported admin-managed currency configuration changes are reflected consistently across seller, buyer, and admin clients without requiring manual data correction.
- **SC-002**: 100% of newly created or updated coupon money fields preserve and return the selected enabled currency correctly in supported create, edit, list, and detail flows.
- **SC-003**: 100% of supported frontend money surfaces render `VND`, `USD`, and `EUR` using one consistent formatting rule for symbol/code and decimal precision.
- **SC-003a**: 100% of supported shared frontend money inputs submit the correct smallest-unit values for `VND`, `USD`, and `EUR` when users enter valid currency-formatted amounts.
- **SC-004**: 100% of mixed-currency calculation attempts in supported cart, checkout, coupon, order, and payment flows are rejected explicitly rather than silently treated as `VND`.
- **SC-005**: Support issues caused by incorrect money symbol or precision rendering in the identified coupon and product detail surfaces drop to zero after release validation for this feature.

## Assumptions

- The feature scope for supported currencies is limited to `VND`, `USD`, and `EUR`.
- Exchange-rate conversion, currency rate storage, and cross-currency totals are out of scope for this feature.
- Existing historical records created before this feature do not need amount conversion or data rewriting to remain readable.
- The admin configuration workspace is the intended user-facing location for currency configuration management in this feature.
- UserService is the owning service for the persisted platform configuration foundation and the admin configuration API that manages it.
- Future config types such as payment methods or payment providers are not delivered as active feature behavior in `0010`, but the design in this feature should not block them from reusing the same config foundation later.
- The platform may block new or updated use of a disabled currency while still allowing historical read access to records already stored with that currency.
- Coupon is the only aggregate in scope for the new single-currency multi-money-field storage pattern in this feature; single-money-field entities continue using the existing `Money` value object approach.
