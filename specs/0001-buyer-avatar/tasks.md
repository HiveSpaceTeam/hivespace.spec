# Tasks: Buyer Avatar

- **Input**: Design documents from `/specs/0001-buyer-avatar/`
- **Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/avatar-api.md, quickstart.md
- **Tests**: Automated test tasks are not generated because TDD/automated tests were not explicitly requested. Verification tasks use targeted builds, lint/type-check, and the manual quickstart flow.
- **Organization**: Tasks are grouped by user story to enable independent implementation and testing.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel with other tasks in the same phase when files do not overlap
- **[Story]**: User story label from spec.md
- Each task includes exact file paths

## Phase 1: Setup (Shared Preparation)

**Purpose**: Confirm implementation scope and target repo rules before editing source repos.

- [ ] T001 Review backend implementation instructions in `../hivespace.microservice/AGENTS.md` and `../hivespace.microservice/CLAUDE.md`
- [ ] T002 Review frontend implementation instructions in `../hivespace.web/AGENTS.md` and `../hivespace.web/CLAUDE.md`
- [ ] T003 [P] Review buyer-avatar requirements in `specs/0001-buyer-avatar/spec.md`, `specs/0001-buyer-avatar/plan.md`, and `specs/0001-buyer-avatar/research.md`
- [ ] T004 [P] Review existing API and event catalogs in `shared/api-catalog.md` and `shared/event-catalog.md` to confirm no new endpoint or event is required

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Establish shared backend/frontend contracts and reusable primitives needed by all stories.

**CRITICAL**: Complete this phase before any user story implementation.

- [ ] T005 Inspect existing UserService profile avatar fields and update flow in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Domain/Aggregates/User/User.cs`, `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Application/DTOs/User/UpdateUserProfileRequestDto.cs`, and `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Application/Services/UserService.cs`
- [ ] T006 Inspect existing MediaService upload validation and confirm flow in `../hivespace.microservice/src/HiveSpace.MediaService/HiveSpace.MediaService.Core/Features/Media/Commands/GeneratePresignedUrl/GeneratePresignedUrlValidator.cs`, `../hivespace.microservice/src/HiveSpace.MediaService/HiveSpace.MediaService.Api/Endpoints/MediaEndpoints.cs`, and `../hivespace.microservice/src/HiveSpace.MediaService/HiveSpace.MediaService.Core/Features/Media/Commands/ConfirmUpload/ConfirmUploadCommandHandler.cs`
- [ ] T007 Inspect existing UserService media processed consumer in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Consumers/Sync/MediaAssetProcessedConsumer.cs`
- [ ] T008 [P] Inspect shared frontend media upload feature in `../hivespace.web/packages/shared/src/features/media-upload/media-upload.types.ts`, `../hivespace.web/packages/shared/src/features/media-upload/media-upload.service.ts`, and `../hivespace.web/packages/shared/src/features/media-upload/createMediaUploadStore.ts`
- [ ] T009 [P] Inspect buyer profile page/store/service structure in `../hivespace.web/apps/buyer/src/pages/Profile/ProfilePage.vue`, `../hivespace.web/apps/buyer/src/stores/profile.store.ts`, and `../hivespace.web/apps/buyer/src/services/user.service.ts`
- [ ] T010 [P] Inspect shared file input and avatar primitives in `../hivespace.web/packages/shared/src/components/FileInput.vue` and `../hivespace.web/packages/shared/src/components/Avatar.vue`

**Checkpoint**: Backend contracts and frontend reuse points are understood; user story work can begin.

---

## Phase 3: User Story 1 - Upload a New Avatar (Priority: P1) MVP

**Goal**: An authenticated buyer can upload and save a new avatar through the buyer profile page and see the saved avatar after processing.

**Independent Test**: Sign in as a buyer, open `/profile`, choose a valid JPEG/PNG/WebP under 5 MB, save, and confirm the profile page shows the new avatar and one success/pending toast.

### Implementation for User Story 1

- [ ] T011 [US1] Ensure `AvatarFileId` and `AvatarUrl` remain profile-owned fields with private setters and no hard-delete behavior in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Domain/Aggregates/User/User.cs`
- [ ] T012 [US1] Ensure optional `AvatarFileId` validation accepts only non-empty GUID-compatible values when provided in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Application/Validators/User/UpdateUserProfileValidator.cs`
- [ ] T013 [US1] Ensure the existing profile update flow calls `SetAvatar(avatarFileId)` only when avatar data is included and preserves current `AvatarUrl` until processing completes in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Application/Services/UserService.cs`
- [ ] T014 [US1] Ensure `GET /api/v1/users/me` returns nullable `avatarUrl` and `PUT /api/v1/users/me` accepts optional `avatarFileId` through existing DTOs in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Application/DTOs/User/GetUserProfileResponseDto.cs` and `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Application/DTOs/User/UpdateUserProfileRequestDto.cs`
- [ ] T015 [US1] Ensure UserService consumes `MediaAssetProcessedIntegrationEvent` for `EntityType = "user_avatar"` and updates only the user whose current `AvatarFileId` matches event `FileId` in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Consumers/Sync/MediaAssetProcessedConsumer.cs`
- [ ] T016 [US1] Ensure MediaService validates `user_avatar` uploads as JPEG, PNG, or WebP with max 5 MB in `../hivespace.microservice/src/HiveSpace.MediaService/HiveSpace.MediaService.Core/Features/Media/Commands/GeneratePresignedUrl/GeneratePresignedUrlValidator.cs`
- [ ] T017 [US1] Ensure buyer profile types include nullable `avatarUrl` and optional `avatarFileId` in `../hivespace.web/apps/buyer/src/types/profile.types.ts`
- [ ] T018 [US1] Ensure buyer media upload service/store reuses shared media upload factories and targets `/media/presign-url` plus `/media/{fileId}/confirm` in `../hivespace.web/apps/buyer/src/services/media.service.ts` and `../hivespace.web/apps/buyer/src/stores/media.store.ts`
- [ ] T019 [US1] Implement buyer profile save orchestration to presign/upload bytes, call `PUT /api/v1/users/me` with `avatarFileId`, then confirm upload in `../hivespace.web/apps/buyer/src/stores/profile.store.ts`
- [ ] T020 [US1] Implement profile refresh polling after avatar confirmation and keep previous `avatarUrl` visible until a new processed URL arrives in `../hivespace.web/apps/buyer/src/stores/profile.store.ts`
- [ ] T021 [US1] Display the saved/latest avatar in buyer profile sidebar and app header from profile store state in `../hivespace.web/apps/buyer/src/components/profile/ProfileSidebar.vue`, `../hivespace.web/apps/buyer/src/components/layout/AppHeader.vue`, and `../hivespace.web/apps/buyer/src/App.vue`
- [ ] T022 [US1] Add English and Vietnamese avatar-only and combined profile/avatar success/pending toast keys in `../hivespace.web/apps/buyer/src/i18n/locales/en/profile-page.json` and `../hivespace.web/apps/buyer/src/i18n/locales/vi/profile-page.json`
- [ ] T023 [US1] Implement exactly one save toast for profile-only, avatar-only, profile+avatar success, and profile+avatar pending outcomes in `../hivespace.web/apps/buyer/src/stores/profile.store.ts`

**Checkpoint**: User Story 1 is independently functional and can be validated from `/profile`.

---

## Phase 4: User Story 2 - Preview Before Saving (Priority: P2)

**Goal**: The buyer sees a local preview of the selected avatar before saving, can replace the selection, and never sees the selected file name below the avatar section.

**Independent Test**: Open `/profile`, select a valid image, confirm the preview appears before saving, select another valid image, confirm the preview updates, and confirm no selected file name appears below the avatar section.

### Implementation for User Story 2

- [ ] T024 [US2] Add or verify `FileInput` support for saved avatar source, circular avatar preview, default avatar fallback, and hidden file-name display in `../hivespace.web/packages/shared/src/components/FileInput.vue`
- [ ] T025 [US2] Ensure `FileInput` object URL cleanup handles replacement and unmount without stale previews in `../hivespace.web/packages/shared/src/components/FileInput.vue`
- [ ] T026 [US2] Wire buyer profile avatar picker to `FileInput` with `accept="image/jpeg,image/png,image/webp"`, `maxSize="5242880"`, circular preview, saved `profile.avatarUrl`, avatar fallback, and `showFileName=false` in `../hivespace.web/apps/buyer/src/pages/Profile/ProfilePage.vue`
- [ ] T027 [US2] Ensure clearing or replacing a selected avatar resets only unsaved preview state and keeps saved profile avatar unchanged in `../hivespace.web/apps/buyer/src/pages/Profile/ProfilePage.vue`
- [ ] T028 [US2] Add English and Vietnamese avatar picker labels/help text/no-file-name-safe validation copy in `../hivespace.web/apps/buyer/src/i18n/locales/en/profile-page.json` and `../hivespace.web/apps/buyer/src/i18n/locales/vi/profile-page.json`

**Checkpoint**: User Story 2 is independently functional without saving a file.

---

## Phase 5: User Story 3 - Handle Invalid or Failed Uploads (Priority: P3)

**Goal**: Unsupported, oversized, interrupted, unauthenticated, or partially failed avatar saves give clear feedback and preserve the existing saved avatar.

**Independent Test**: Try unsupported and oversized files plus simulated failed upload/confirm/profile-save paths; confirm the buyer sees actionable feedback and the saved avatar is not replaced by a broken image.

### Implementation for User Story 3

- [ ] T029 [US3] Ensure unsupported avatar file types and images over 5 MB are rejected client-side before upload in `../hivespace.web/packages/shared/src/components/FileInput.vue` and `../hivespace.web/apps/buyer/src/pages/Profile/ProfilePage.vue`
- [ ] T030 [US3] Ensure MediaService returns validation failures for unsupported `user_avatar` content types and oversized files in `../hivespace.microservice/src/HiveSpace.MediaService/HiveSpace.MediaService.Core/Features/Media/Commands/GeneratePresignedUrl/GeneratePresignedUrlValidator.cs`
- [ ] T031 [US3] Ensure unauthenticated avatar save is rejected and prompts sign-in feedback before upload/confirm in `../hivespace.web/apps/buyer/src/stores/profile.store.ts`
- [ ] T032 [US3] Ensure failed presign/blob upload/profile update/confirm paths preserve previous `avatarUrl`, reset only unsaved avatar state, and set actionable avatar error state in `../hivespace.web/apps/buyer/src/stores/profile.store.ts`
- [ ] T033 [US3] Implement partial-success toast behavior when profile fields save but avatar confirmation or processing cannot complete in `../hivespace.web/apps/buyer/src/stores/profile.store.ts`
- [ ] T034 [US3] Add English and Vietnamese unsupported type, oversized file, upload failed, save failed, unauthenticated, and partial profile-saved/avatar-failed messages in `../hivespace.web/apps/buyer/src/i18n/locales/en/profile-page.json` and `../hivespace.web/apps/buyer/src/i18n/locales/vi/profile-page.json`
- [ ] T035 [US3] Ensure UserService media processed consumer throws observable not-found failures instead of silently swallowing required missing user data in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Consumers/Sync/MediaAssetProcessedConsumer.cs`

**Checkpoint**: User Story 3 protects saved avatar state and provides clear error feedback.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Final verification and documentation/catalog consistency.

- [ ] T036 [P] Verify no new public endpoints are required and leave `shared/api-catalog.md` unchanged unless implementation adds an endpoint
- [ ] T037 [P] Verify no new integration events are required and leave `shared/event-catalog.md` unchanged unless implementation adds a message
- [ ] T038 Run backend build for UserService in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/HiveSpace.UserService.Api.csproj`
- [ ] T039 Run backend build for MediaService API in `../hivespace.microservice/src/HiveSpace.MediaService/HiveSpace.MediaService.Api/HiveSpace.MediaService.Api.csproj`
- [ ] T040 Run buyer frontend lint in `../hivespace.web/apps/buyer`
- [ ] T041 Run buyer frontend type-check in `../hivespace.web/apps/buyer`
- [ ] T042 Run the manual verification flow from `specs/0001-buyer-avatar/quickstart.md`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies.
- **Foundational (Phase 2)**: Depends on Setup completion and blocks all story implementation.
- **User Story 1 (Phase 3)**: Depends on Foundational and is the MVP.
- **User Story 2 (Phase 4)**: Depends on Foundational; can be implemented after or alongside US1 once shared `FileInput` impact is understood.
- **User Story 3 (Phase 5)**: Depends on Foundational and benefits from US1 save orchestration and US2 validation UI.
- **Polish (Phase 6)**: Depends on all desired user stories being complete.

### User Story Dependencies

- **US1 Upload a New Avatar**: No dependency on other stories after Foundational; suggested MVP.
- **US2 Preview Before Saving**: Can be developed independently after Foundational, but final page wiring should be reconciled with US1.
- **US3 Invalid/Failed Upload Handling**: Can start after Foundational, but final failure branches depend on US1 upload orchestration.

### Parallel Opportunities

- T003 and T004 can run in parallel after T001/T002 start.
- T008, T009, and T010 can run in parallel during Foundational.
- T011-T016 backend tasks and T017-T023 frontend tasks can be split across backend/frontend implementers, with T019 depending on T017/T018.
- T024/T025 shared component work can run while US1 backend tasks proceed.
- T029/T030/T034 can run in parallel because they touch separate validation/i18n files.
- T036/T037 can run in parallel during final catalog verification.

---

## Parallel Example: User Story 1

```text
Task: "Ensure UserService DTO/profile update behavior in ../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Application/Services/UserService.cs"
Task: "Ensure MediaService avatar MIME/size validation in ../hivespace.microservice/src/HiveSpace.MediaService/HiveSpace.MediaService.Core/Features/Media/Commands/GeneratePresignedUrl/GeneratePresignedUrlValidator.cs"
Task: "Prepare buyer profile request types in ../hivespace.web/apps/buyer/src/types/profile.types.ts"
Task: "Add buyer profile toast i18n keys in ../hivespace.web/apps/buyer/src/i18n/locales/en/profile-page.json and ../hivespace.web/apps/buyer/src/i18n/locales/vi/profile-page.json"
```

## Parallel Example: User Story 2

```text
Task: "Add shared FileInput avatar preview options in ../hivespace.web/packages/shared/src/components/FileInput.vue"
Task: "Add buyer avatar picker copy in ../hivespace.web/apps/buyer/src/i18n/locales/en/profile-page.json and ../hivespace.web/apps/buyer/src/i18n/locales/vi/profile-page.json"
```

## Parallel Example: User Story 3

```text
Task: "Ensure MediaService rejects invalid user_avatar files in ../hivespace.microservice/src/HiveSpace.MediaService/HiveSpace.MediaService.Core/Features/Media/Commands/GeneratePresignedUrl/GeneratePresignedUrlValidator.cs"
Task: "Add failure and partial-success translations in ../hivespace.web/apps/buyer/src/i18n/locales/en/profile-page.json and ../hivespace.web/apps/buyer/src/i18n/locales/vi/profile-page.json"
Task: "Ensure UserService consumer failures remain observable in ../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Consumers/Sync/MediaAssetProcessedConsumer.cs"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1 and Phase 2.
2. Complete Phase 3 for upload/save/sync/display.
3. Validate by signing in as a buyer, uploading a valid avatar, saving, and confirming the avatar appears after processing.
4. Stop and demo before implementing preview refinements or advanced error handling.

### Incremental Delivery

1. Deliver US1 upload/save/display as the MVP.
2. Add US2 preview replacement, default fallback, and hidden file name.
3. Add US3 invalid/failed upload handling and partial-success toast behavior.
4. Run all final verification tasks in Phase 6.

### Parallel Team Strategy

1. Backend implementer owns T011-T016, T030, T035, T038, and T039.
2. Frontend shared implementer owns T018, T024, T025, and `FileInput` validation behavior.
3. Buyer app implementer owns T017, T019-T023, T026-T034, T040, and T041.
4. Spec/catalog reviewer owns T036, T037, and T042.

---

## Notes

- [P] tasks use different files or can be done without depending on incomplete code.
- No `saga-design.md` or ADR tasks are included because this feature does not add/change MassTransit saga state and reuses existing service boundaries.
- Preserve target repo rules: run GitNexus impact before editing symbols in indexed source repos, and do not stage/commit JSON files from source repos.
- All buyer-facing text must be i18n-backed in both English and Vietnamese.
