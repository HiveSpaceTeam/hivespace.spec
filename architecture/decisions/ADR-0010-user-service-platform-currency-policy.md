# ADR-0010: UserService Platform Configuration Foundation

- **Status**: Accepted
- **Date**: 2026-06-29
- **Feature**: `0010-standardize-money-handling`
- **Deciders**: Project maintainers

## Context

HiveSpace currently handles money inconsistently across backend and frontend code. Several flows assume `VND` implicitly, coupon money fields do not consistently preserve one currency, and multiple UI surfaces format values with hard-coded symbols or locale-only rules. The feature also requires a platform admin to enable or disable supported currencies and choose the default currency from the admin configuration workspace.

The main decision is where this platform-level configuration foundation should live and how other commerce services should consume currency configuration without violating service boundaries.

## Decision

UserService will own the persisted platform configuration foundation and the admin configuration API used to manage it.

The foundation will use a small config-level record for type, default selected value, and optimistic version state, plus typed config-item tables for selectable entries. Currency is the first implemented config type in feature `0010`, using one row per supported currency with enabled state and optional UI ordering metadata.

UserService will publish a currency configuration update event whenever the currency config changes. CatalogService, OrderService, and PaymentService will consume that event into local currency-config ref models and use those refs for synchronous validation in product, coupon, cart, checkout, order, and payment workflows.

Frontend apps will standardize on shared money display and money-input helpers in `@hivespace/shared`, while backend APIs will expose explicit money metadata instead of relying on client-side `VND` assumptions. Those shared helpers will default cent-backed `USD` and `EUR` values to major-unit dollar/euro display and input rather than raw cent display, with an optional explicit override for specialized cases.

Coupon will standardize on one canonical aggregate `CurrencyCode` plus smallest-unit amount fields for related coupon money values, instead of repeating coupon-specific persisted currency fields across each money sub-value. This feature will update seed and fixture data only; it will not include migration/backfill work because the project is still in development phase.

## Consequences

### Positive

- One authoritative owner governs enabled/default currencies.
- The same ownership model can support future list-based config types such as payment methods or payment providers without another persistence redesign.
- Commerce services keep fast local validation without direct database reads or runtime coupling to UserService.
- Admin, seller, and buyer apps can render money consistently from explicit contracts.
- The design satisfies the clarified requirement that disabled currencies block new writes while historical records remain readable.

### Negative / Trade-offs

- The feature adds a new cross-service integration event and projection maintenance burden.
- The foundation introduces one more layer of abstraction than a currency-only design.
- Existing API contracts for product, coupon, order, checkout, and payment money fields need coordinated updates across backend and frontend repos.
- Coupon persistence and API contracts need a coordinated reshape to one canonical coupon currency plus smallest-unit amount fields.
- Projection lag must be considered when rolling out policy changes.

### Risks

- If a projection consumer falls behind, a service could validate against stale currency configuration.
- Broad frontend surface area means some hard-coded formatters may be missed without a disciplined audit.
- Historical malformed data may still surface unexpected invalid-money states until data quality improves.

## Alternatives Considered

| Option | Why rejected |
| --- | --- |
| Make OrderService the owner | Coupons and checkout are only part of the problem; product pricing and admin configuration would become awkward supporting concerns |
| Make PaymentService the owner | Payment ownership does not justify platform-wide governance over product and coupon authoring |
| Use synchronous HTTP validation from commerce services to UserService | Adds runtime coupling and availability dependency to core write/check flows |
| Store currency-only state in a dedicated aggregate with embedded enabled values | Rejected because the admin configuration UI and expected future config types point toward a reusable list-based foundation |
| Store config rows with free-form JSON item payloads | Rejected because typed currency rows keep validation and contracts simpler for the first implementation |
| Store currency configuration in shared config/appsettings | The feature requires admin-managed persisted state, not deployment-time configuration |

## Follow-Up

- Add API catalog rows for the new UserService currency-configuration endpoints and changed money-bearing endpoint contracts
- Add event catalog row for the currency configuration update event
- Update UserService, CatalogService, OrderService, and PaymentService service docs during implementation
