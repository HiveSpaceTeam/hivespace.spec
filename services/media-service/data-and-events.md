# MediaService Data And Events

## Data Ownership

MediaService owns:

- `media_assets`
- Upload state: pending, uploaded, processed, failed.
- Storage object path/container metadata.
- Processing results such as thumbnail URLs.

## Published Events

| Event | Purpose |
|---|---|
| `MediaAssetProcessedIntegrationEvent` | Tell owning domains that media processing completed |

## External Dependencies

| Dependency | Purpose |
|---|---|
| Azure Blob Storage | Production media object storage |
| Azurite | Local media object storage emulator |
| Image processing library/runtime | Thumbnail and image derivative generation |

## Invariants

- MediaService owns file lifecycle, not the business meaning of the entity that references a file.
- A media asset may be referenced by another service only after upload confirmation.
- Failed processing must be observable and retryable where supported.

## Publisher Policy

- Media background processing publishes `MediaAssetProcessedIntegrationEvent` through a service-owned media event publisher after processing state is saved.
