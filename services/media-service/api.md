# MediaService API

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/media/presign-url` | `Authorize` | Create direct upload URL and file/media ID |
| POST | `/api/v1/media/{fileId}/confirm` | `Authorize` | Confirm upload and associate media with entity |

## Upload Flow

```text
Frontend
  -> POST /api/v1/media/presign-url
  -> PUT file bytes to returned storage URL
  -> POST /api/v1/media/{fileId}/confirm
  -> MediaService marks uploaded and processes asset
```

## API Rules

- Do not send large file bytes through business services.
- Keep upload confirmation idempotent.
- Validate ownership before associating an asset with an entity.
