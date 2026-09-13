# CatalogService

## Responsibility

CatalogService owns product catalog data and seller-facing product management.

Source path:

```text
../hivespace.microservice/src/HiveSpace.CatalogService
```

## Owns

- Products and product lifecycle.
- SKUs and variant/option data.
- Categories and category attributes.
- Catalog import jobs, same-host background import workers, progress/result state, and retry eligibility.
- Category provisioning links for external Tiki categories before product import.
- Category-attribute provisioning links for external Tiki category attributes and selectable values before product validation.
- Imported seller ownership links that approve an external seller identity for an existing HiveSpace seller account/store.
- Product attributes and values.
- Storefront product read models.
- Store reference projection used for catalog ownership/display.
- Local currency-policy validation projection used for product and SKU money writes.

## Must Not Own

- User identity or store registration.
- Cart and order lifecycle.
- Payment processing.
- Notification delivery.
- Media binary storage.

## Architecture

CatalogService follows the standard backend feature pattern:

```text
Domain -> Application -> Infrastructure -> Api
```

New feature work should use CQRS handlers and Minimal API endpoint modules.

## Runtime

| Item | Value |
|---|---|
| Local HTTP | `http://localhost:5002` |
| Gateway prefixes | `/api/v1/products`, `/api/v1/categories`, `/api/v1/admins/catalog-imports` |
| Database | SQL Server, CatalogService-owned catalog schema |
| Messaging | MassTransit/RabbitMQ for product/store projection events |

Backend local development starts CatalogService through Aspire AppHost in `../hivespace.microservice/src/HiveSpace.AppHost`; frontend dev servers remain separate.

## Planning Notes

- Seller product APIs require seller authorization.
- Storefront product discovery APIs are anonymous.
- Product and SKU writes must validate explicit currency codes against the local currency-policy projection; CatalogService must fail closed when that projection is missing, unreadable, or stale for a write path.
- Product read models expose explicit money metadata and invalid-money diagnostics instead of assuming `VND`.
- OrderService should use product/SKU projections from events rather than querying CatalogService database.
- Media references should point to MediaService assets, but CatalogService owns the product association decision.
- Tiki catalog imports are category-first and job-based: external categories are crawled and submitted to a CatalogService-owned async provisioning job before product bundles are crawled or submitted.
- Tiki category attributes are crawled separately and submitted through a CatalogService-owned async category-attribute provisioning job before product bundle validation/import treats attribute readiness as satisfied.
- Long-running catalog import operations return `202 Accepted` with a job ID and are processed by CatalogService background consumers in the same service host. v1 must not use Azure Functions, a separate worker project, or a central executor for import work.
- CatalogService publishes generic background job lifecycle events for observer-only monitoring. CatalogService job status remains the source of truth for import progress, result summaries, errors, and retry eligibility.
- Product import validation resolves product `externalCategoryIds` through provisioned category links; product import does not create categories.
- Similar-name imported seller/store conflicts require explicit operator approval before CatalogService records an ownership link; approval must not overwrite existing store profile data.
- The Python crawler and CatalogService import boundary are documented in [ADR-0012](../../architecture/decisions/ADR-0012-python-catalog-import-boundary.md).
- Inventory workflow events use standardized `*IntegrationEvent` names per [ADR-0002](../../architecture/decisions/ADR-0002-standardized-integration-event-contracts.md).
- Currency-policy ownership remains in UserService per [ADR-0010](../../architecture/decisions/ADR-0010-user-service-platform-currency-policy.md); CatalogService only consumes the projection for validation.

## Detail

- Domain model: `domain.md`
- Public API: `api.md`
- Data and events: `data-and-events.md`
