# Contract: Buyer Avatar APIs

All browser-facing calls go through the ApiGateway under `/api/v1`.

## Update Current User Profile With Optional Avatar

```http
PUT /api/v1/users/me
Authorization: Bearer <access-token>
Content-Type: application/json
```

### Request

```json
{
  "userName": "buyer123",
  "fullName": "Buyer Name",
  "phoneNumber": "84901234567",
  "gender": 2,
  "dateOfBirth": "1995-01-01T00:00:00Z",
  "avatarFileId": "4f4f68da-3db8-4eb3-b18c-d1ed67dfddbc"
}
```

### Response

```http
204 No Content
```

### Validation

- Existing profile update field validation still applies.
- `avatarFileId` is optional.
- When `avatarFileId` is null or omitted, the saved avatar is unchanged.
- When `avatarFileId` is provided, it must be a GUID-compatible media file ID.
- The endpoint always targets the authenticated current user; clients cannot pass another user ID.
- On success, UserService stores `AvatarFileId` and keeps the current `AvatarUrl` unchanged until media processing resolves the replacement public URL.

### Errors

| Status | Meaning |
|---|---|
| 400 | Invalid profile field or invalid `avatarFileId` when provided |
| 401 | User is not authenticated |
| 404 | Current user was not found |

## Existing Media Upload Flow

### Presign Upload URL

```http
POST /api/v1/media/presign-url
Authorization: Bearer <access-token>
Content-Type: application/json
```

```json
{
  "fileName": "avatar.webp",
  "contentType": "image/webp",
  "fileSize": 321000,
  "entityType": "user_avatar",
  "entityId": null
}
```

### Upload Bytes

```http
PUT <uploadUrl>
x-ms-blob-type: BlockBlob
Content-Type: image/webp
```

Body is the selected image file bytes.

### Confirm Upload

```http
POST /api/v1/media/{fileId}/confirm
Authorization: Bearer <access-token>
Content-Type: application/json
```

```json
{
  "entityId": "<current-user-id>"
}
```

The buyer app should call `PUT /api/v1/users/me` with `avatarFileId` before confirming upload so UserService can resolve the later `MediaAssetProcessedIntegrationEvent`. The buyer app resolves `entityId` from the authenticated buyer subject/user ID rather than editable profile form state.
