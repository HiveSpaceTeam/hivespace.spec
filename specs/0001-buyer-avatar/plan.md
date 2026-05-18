# Implementation Plan: Buyer Avatar

**Branch:** `001-buyer-avatar`
**Date:** 2026-05-18
**Spec:** [spec.md](spec.md)

> Constitution loaded from `.specify/memory/constitution.md` before planning.

---

## Phase 0 - Research

### Existing context read before planning

- [x] `architecture/overview.md`
- [x] `services/_inventory.md`
- [x] `services/user-service/README.md`
- [x] `services/user-service/api.md`
- [x] `services/user-service/domain.md`
- [x] `services/user-service/data-and-events.md`
- [x] `services/media-service/README.md`
- [x] `services/media-service/api.md`
- [x] `services/media-service/domain.md`
- [x] `services/media-service/data-and-events.md`
- [x] `services/api-gateway/README.md`
- [x] `services/api-gateway/api.md`
- [x] `shared/event-catalog.md`
- [x] `shared/api-catalog.md`
- [x] `shared/coding-conventions.md`
- [x] `shared/glossary.md`
- [x] `../hivespace.microservice/AGENTS.md`
- [x] `../hivespace.microservice/CLAUDE.md`
- [x] `../hivespace.web/AGENTS.md`
- [x] `../hivespace.web/CLAUDE.md`

### Technical unknowns

- Avatar upload ownership and media validation: resolved in [research.md](research.md).
- Whether to add a new avatar endpoint or reuse profile update: resolved by clarification and [research.md](research.md).
- Event needs for processed avatar URL sync: resolved in [research.md](research.md).
- Frontend shared component reuse for file preview and media upload: resolved in [research.md](research.md).
- Combined profile/avatar save toast behavior: resolved in [research.md](research.md).

### Research notes

The selected design reuses MediaService direct upload, `PUT /api/v1/users/me`, the existing `MediaAssetProcessedIntegrationEvent`, and shared frontend media/profile primitives. No unresolved clarification remains.

---

## Phase 1 - Architecture & Data Model

### Service placement

| Area | Owner | Reason |
|---|---|---|
| Profile avatar reference (`AvatarFileId`, `AvatarUrl`) | UserService | UserService owns account and profile data. |
| Presigned URL, file storage, media lifecycle, processing URL | MediaService | MediaService owns media assets and processing state. |
| Browser-facing routing | ApiGateway | Existing `/api/v1/users/**` and `/api/v1/media/**` prefixes already route to the owning services. |
| Buyer profile UI and account avatar display | Buyer app | The requested surface is buyer-facing profile/account UI. |
| Shared upload/profile primitives | `@hivespace/shared` | Existing reusable frontend media upload and user profile modules should be reused instead of duplicating app-local infrastructure. |

### Data model

See [data-model.md](data-model.md).

The feature uses existing UserService user profile fields and MediaService `MediaAsset` records. No new aggregate is required.

### New aggregates

None.

### Repository interfaces

No new repository interface is required. UserService continues through existing user repository/profile service logic. MediaService continues through existing media command handlers.

### Integration events

No new integration event is required.

| Event | Producer | Consumer(s) | Purpose |
|---|---|---|---|
| `MediaAssetProcessedIntegrationEvent` | MediaService | UserService | For `EntityType = "user_avatar"`, sync processed `PublicUrl` to the user whose `AvatarFileId` equals event `FileId`. |
| `UserUpdatedIntegrationEvent` | UserService | Existing user projection consumers | Existing event already carries `AvatarUrl`; use it if profile update publication is extended or already triggered by existing profile update flow. |

Catalog status: `shared/event-catalog.md` already contains these contracts. No catalog update is needed for plan phase.

### API endpoints

No new public endpoint is required.

| Method | Path | Auth | Handler/owner | Change |
|---|---|---|---|---|
| `PUT` | `/api/v1/users/me` | `RequireAdminOrUser` | UserService `UserController.UpdateUserProfile` | Accept optional `avatarFileId` only when the buyer changes avatar. |
| `GET` | `/api/v1/users/me` | `RequireAdminOrUser` | UserService `UserController.GetUserProfile` | Return latest resolved `avatarUrl`. |
| `POST` | `/api/v1/media/presign-url` | `Authorize` | MediaService `MediaEndpoints` | Reuse with `entityType = "user_avatar"` and avatar MIME/size validation. |
| `POST` | `/api/v1/media/{fileId}/confirm` | `Authorize` | MediaService `MediaEndpoints` | Reuse after UserService has stored `avatarFileId`. |

Catalog status: `shared/api-catalog.md` already contains these public endpoints. No new route prefix is needed.

### Contracts

See [contracts/avatar-api.md](contracts/avatar-api.md).

### Saga design

No saga required. The feature reuses a single MediaService processing event consumed by UserService, with no compensation path, timeout-driven orchestration, or MassTransit state machine.

### Architecture decision

No ADR required. This feature reuses existing service boundaries, existing endpoints, and existing event contracts; the media sync ordering is captured in [research.md](research.md) and [contracts/avatar-api.md](contracts/avatar-api.md).

---

## Phase 2 - Implementation Plan

### Backend layer order

**UserService domain/application**

- [ ] Ensure `User.AvatarFileId` and `User.AvatarUrl` remain profile-owned fields with private setters.
- [ ] Ensure `UpdateUserProfileRequestDto.AvatarFileId` is optional and validated as a GUID-compatible media file ID when present.
- [ ] In the existing profile update flow, call `SetAvatar(avatarFileId)` only when the request includes avatar data.
- [ ] Keep existing `AvatarUrl` unchanged when only a new `AvatarFileId` is saved.

**UserService infrastructure/api**

- [ ] Keep `PUT /api/v1/users/me` as the only profile save endpoint for this feature.
- [ ] Ensure the media processed consumer handles `EntityType = "user_avatar"` and finds the user by matching `AvatarFileId` to event `FileId`.
- [ ] Update the matched user's `AvatarUrl` from `PublicUrl`.
- [ ] Throw observable failures for required missing user data instead of silently swallowing processing events.

**MediaService application/api**

- [ ] Reuse `POST /api/v1/media/presign-url`.
- [ ] For `entityType = "user_avatar"`, enforce JPEG, PNG, WebP, and max 5 MB.
- [ ] Reuse direct blob upload and `POST /api/v1/media/{fileId}/confirm`.
- [ ] Ensure processing publishes `MediaAssetProcessedIntegrationEvent` with `FileId`, `EntityType`, `EntityId`, and `PublicUrl`.

**ApiGateway**

- [ ] No route changes expected because `/api/v1/users/**` and `/api/v1/media/**` already exist.

### Frontend plan

Surface:

- [x] buyer
- [ ] seller
- [ ] admin

Implementation order:

| Step | Files | Notes |
|---|---|---|
| Types | `apps/buyer/src/types/profile.types.ts` | Include optional `avatarFileId` in `UpdateProfileRequest`; keep `avatarUrl` nullable on profile response. |
| Services | `apps/buyer/src/services/user.service.ts`, `apps/buyer/src/services/profile.service.ts`, `apps/buyer/src/services/media.service.ts` | Reuse app `apiService` and shared profile/media upload factories. |
| Stores | `apps/buyer/src/stores/profile.store.ts`, `apps/buyer/src/stores/media.store.ts` | Store owns upload orchestration, save state, profile refresh, and avatar error state. |
| Components | `packages/shared/src/components/FileInput.vue`, `apps/buyer/src/components/profile/ProfileSidebar.vue`, `apps/buyer/src/components/layout/AppHeader.vue` | Reuse shared `FileInput` with avatar fallback and hidden file-name option. |
| Page | `apps/buyer/src/pages/Profile/ProfilePage.vue` | Add avatar picker to existing profile page; preview selected valid image; no file name below avatar. |
| Routes | Existing `/profile` route | No new route. |
| i18n | `apps/buyer/src/i18n/locales/en/profile-page.json`, `apps/buyer/src/i18n/locales/vi/profile-page.json` | Add/update profile-only, avatar-only, combined profile+avatar success, combined profile+avatar pending, partial profile-saved/avatar-failed, unsupported type, oversized, and unauthenticated messages in both languages. |

Toast behavior:

- Profile fields only changed: show one profile success toast.
- Avatar only changed and avatar resolves within the profile polling window: show one avatar success toast.
- Avatar only changed and avatar is still processing after the polling window: show one avatar pending toast.
- Profile fields and avatar both changed, and avatar resolves within the polling window: show one combined success toast, such as "Profile and avatar updated successfully".
- Profile fields and avatar both changed, and avatar is still processing after the polling window: show one combined pending toast, such as "Profile updated. Your new avatar will appear after processing finishes."
- Profile fields are saved but avatar confirmation or processing cannot complete: show one partial-success error/warning toast, such as "Profile updated, but the avatar could not be completed. Please try again."
- Do not show separate profile and avatar success toasts for one save action.

### Verification plan

Backend:

- [ ] `dotnet build src/HiveSpace.UserService/HiveSpace.UserService.Api/HiveSpace.UserService.Api.csproj`
- [ ] `dotnet build src/HiveSpace.MediaService/HiveSpace.MediaService.Api/HiveSpace.MediaService.Api.csproj`
- [ ] Manual or targeted integration check that `MediaAssetProcessedIntegrationEvent` for `user_avatar` updates only the user whose `AvatarFileId` matches the event `FileId`.

Frontend:

- [ ] From `../hivespace.web/apps/buyer`, run `pnpm lint`.
- [ ] From `../hivespace.web/apps/buyer`, run `pnpm type-check`.
- [ ] Manual `/profile` flow from [quickstart.md](quickstart.md).

### Constitution compliance check

- [x] Planning-only repo: no runnable product code added here.
- [x] Service ownership respected: UserService owns profile fields; MediaService owns file lifecycle.
- [x] No cross-service database reads.
- [x] No new hard-delete behavior.
- [x] No money values introduced.
- [x] No new package version changes planned.
- [x] Integration event reuse documented; no duplicate event added.
- [x] No new public endpoint added; existing endpoints are cataloged.
- [x] Frontend uses shared-first approach and i18n in both English and Vietnamese.
- [x] No saga artifact required; this feature has no compensation path, timeout-driven orchestration, or MassTransit state machine.
- [x] No ADR required; this feature reuses existing service boundaries, endpoints, and event contracts.

### Post-design gate

PASS. The plan uses existing service boundaries, existing public APIs, and existing messaging contracts. No unresolved clarification or unjustified constitution violation remains.
