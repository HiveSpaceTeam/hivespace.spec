# MediaService

## Responsibility

MediaService owns upload orchestration and media asset processing state.

Source path:

```text
../hivespace.microservice/src/HiveSpace.MediaService
```

## Owns

- Presigned/direct upload URL generation.
- Media asset records.
- Upload confirmation.
- Image processing and thumbnail lifecycle.
- Blob/Azurite storage interaction.

## Must Not Own

- Product, order, user, or store business records.
- Product image association rules beyond media ownership metadata.
- Catalog, payment, order, or identity decisions.

## Architecture

MediaService is a lighter service. New feature work should still follow the repo's CQRS/Minimal API direction and keep storage/processing concerns isolated from business-domain ownership.

## Runtime

| Item | Value |
|---|---|
| Local HTTP | `http://localhost:5003` |
| Gateway prefix | `/api/v1/media` |
| Storage | Azure Blob Storage or Azurite |
| Database | SQL Server, media asset records |

Backend local development starts MediaService API through Aspire AppHost in `../hivespace.microservice/src/HiveSpace.AppHost`. The media workflow also includes MediaService Function on `http://localhost:7072`, which runs through Azure Functions Core Tools under AppHost orchestration.

## Planning Notes

- Frontends upload file bytes directly to the issued storage URL.
- Other domains store references to media IDs/URLs, but MediaService owns media processing state.
- If a domain needs to react to processing completion, use `MediaAssetProcessedIntegrationEvent`.
- Media processing publishes through a service-owned publisher according to [ADR-0002](../../architecture/decisions/ADR-0002-standardized-integration-event-contracts.md).

## Detail

- Domain model: `domain.md`
- Public API: `api.md`
- Data and events: `data-and-events.md`
