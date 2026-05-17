# MediaService Domain Model

## Purpose

MediaService owns media asset records and upload/processing lifecycle. It does not own the business meaning of the product, store, user, or order entity that references a file.

Implementation source:

```text
../hivespace.microservice/src/HiveSpace.MediaService/HiveSpace.MediaService.Core/DomainModels
```

## Core Model

| Model | Type | Meaning |
|---|---|---|
| `MediaAsset` | Aggregate root | File record with storage path, owning entity metadata, URLs, MIME type, file size, and processing status |
| `MediaStatus` | Enum | Upload/processing lifecycle: `Pending`, `Uploaded`, `Processed`, `Failed` |
| `StoragePermissions` | Enum | Storage permission flags used when issuing direct upload access |

## Business Rules

- A media asset requires file name, storage path, and entity type.
- New media assets start in `Pending`.
- `EntityId` may be attached only while the asset is still `Pending`.
- `MarkAsUploaded()` is valid only from `Pending`; calling it from any other state is a domain failure.
- `MarkAsProcessed(...)` records the public URL and optional thumbnail URL, then moves the asset to `Processed`.
- `MarkAsFailed()` moves the asset to `Failed` so failed processing remains observable.
- File size and storage metadata may be updated as processing/storage details become known.
- MediaService validates file lifecycle, not whether a given business entity is allowed to use the file.

## Lifecycle

| Step | State / rule |
|---|---|
| Presign request | Create `MediaAsset` in `Pending` with storage path and entity metadata |
| Browser upload | Client uploads bytes directly to storage using the issued URL |
| Upload confirmation | Asset moves from `Pending` to `Uploaded` |
| Processing success | Asset moves to `Processed` with public/thumbnail URLs |
| Processing failure | Asset moves to `Failed` |

## Cross-Service Facts

- Other services store media file IDs or URLs as references; they do not own media processing state.
- `MediaAssetProcessedIntegrationEvent` tells owning domains that a media file has a resolved URL/thumbnail.
- MediaService depends on Azure Blob Storage or Azurite for object storage.
- Product, store, user, and order business validation remains in the owning service, not MediaService.
