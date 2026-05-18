# Quickstart: Buyer Avatar

## Prerequisites

- SQL Server, RabbitMQ, Azurite/Blob storage, and required local infrastructure are running.
- UserService, MediaService API, MediaService processing function, ApiGateway, and buyer app are available.
- Buyer account can sign in through the existing OIDC flow.

## Backend Verification

From `../hivespace.microservice`:

```bash
dotnet build src/HiveSpace.UserService/HiveSpace.UserService.Api/HiveSpace.UserService.Api.csproj
dotnet build src/HiveSpace.MediaService/HiveSpace.MediaService.Api/HiveSpace.MediaService.Api.csproj
```

Avatar media sync verification:

- Confirm MediaService processing publishes `MediaAssetProcessedIntegrationEvent` after a `user_avatar` image is processed.
- Confirm UserService has `MediaAssetProcessedConsumer` registered when RabbitMQ messaging is enabled.
- Confirm a `user_avatar` processed event updates `AvatarUrl` only for the user whose `AvatarFileId` matches the event `FileId`.

## Frontend Verification

From `../hivespace.web/apps/buyer`:

```bash
pnpm lint
pnpm type-check
```

## Manual Flow

1. Start ApiGateway, UserService, MediaService API, MediaService processing function, and the buyer app.
2. Sign in as a buyer.
3. Open `/profile`.
4. If the buyer already has a saved avatar, confirm the avatar upload component displays that current avatar before any new file is selected.
5. If the buyer has no saved avatar, confirm the avatar upload component displays the shared default avatar fallback rather than a blank preview.
6. Select a JPEG, PNG, or WebP image under 5 MB.
7. Confirm the preview displays the selected image before saving and temporarily replaces the saved/default avatar display.
8. Clear the selected image before saving and confirm the upload component returns to the current saved avatar or default avatar fallback.
9. Select a valid image again and confirm the selected file name is not displayed below the avatar image section.
10. Save the profile/avatar change through the existing profile save action.
11. Confirm exactly one toast appears for the save action.
12. When only profile fields changed, confirm the toast says the profile was updated.
13. When only the avatar changed and processing resolves within the polling window, confirm the toast says the avatar was updated.
14. When both profile fields and avatar changed and processing resolves within the polling window, confirm the toast says both profile and avatar were updated.
15. When both profile fields and avatar changed but the avatar is still processing after the polling window, confirm the toast says the profile was updated and the new avatar will appear after processing finishes.
16. When profile fields are saved but avatar confirmation or processing cannot complete, confirm the toast says the profile was updated but the avatar could not be completed, and confirm the existing saved avatar remains visible.
17. Confirm MediaService processing publishes `MediaAssetProcessedIntegrationEvent` for `EntityType = "user_avatar"` and UserService syncs the event `PublicUrl` into the matching user's `AvatarUrl`.
18. Confirm the profile refreshes after processing completes and the latest avatar appears on profile, header, account sidebar, and the avatar upload component without a manual browser refresh when processing completes within the polling window.
19. Try an unsupported file type and confirm it is rejected without replacing the saved avatar.
20. Try an image over 5 MB and confirm it is rejected without replacing the saved avatar.

## Expected Data Flow

```text
Buyer app
  -> MediaService presign-url
  -> Blob/Azurite direct upload
  -> UserService users/me with AvatarFileId
  -> MediaService confirm
  -> Buyer app polls current profile while showing previous avatar or pending state
  -> MediaService processing
  -> MediaAssetProcessedIntegrationEvent
  -> UserService MediaAssetProcessedConsumer matches FileId to AvatarFileId
  -> UserService updates AvatarUrl from event PublicUrl
  -> Buyer app refreshes profile
```
