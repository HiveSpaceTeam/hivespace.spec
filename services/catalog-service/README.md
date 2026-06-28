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
- Product attributes and values.
- Storefront product read models.
- Store reference projection used for catalog ownership/display.

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
| Gateway prefixes | `/api/v1/products`, `/api/v1/categories` |
| Database | SQL Server, CatalogService-owned catalog schema |
| Messaging | MassTransit/RabbitMQ for product/store projection events |

Backend local development starts CatalogService through Aspire AppHost in `../hivespace.microservice/src/HiveSpace.AppHost`; frontend dev servers remain separate.

## Planning Notes

- Seller product APIs require seller authorization.
- Storefront product discovery APIs are anonymous.
- OrderService should use product/SKU projections from events rather than querying CatalogService database.
- Media references should point to MediaService assets, but CatalogService owns the product association decision.
- Inventory workflow events use standardized `*IntegrationEvent` names per [ADR-0002](../../architecture/decisions/ADR-0002-standardized-integration-event-contracts.md).

## Detail

- Domain model: `domain.md`
- Public API: `api.md`
- Data and events: `data-and-events.md`
