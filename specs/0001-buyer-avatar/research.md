# Research: Buyer Avatar

## Decision: Reuse MediaService Direct Upload

**Rationale:** MediaService already owns presigned URL generation, direct blob upload, confirmation, processing state, and `MediaAssetProcessedIntegrationEvent`. Sending image bytes through UserService would duplicate ownership and violate the existing media boundary.

**Alternatives considered:**

- Upload bytes through UserService: rejected because business services must not own large file transfer.
- Store avatar as a URL typed directly by the user: rejected because it bypasses media ownership and validation.

## Decision: Reuse Current-User Profile Update Endpoint

**Rationale:** The implementation should use the existing `PUT /api/v1/users/me` profile update endpoint and add optional `AvatarFileId` to its request. When the buyer does not change their avatar, `AvatarFileId` remains null/omitted and avatar state is unchanged. When provided, UserService stores the new file ID and keeps the current `AvatarUrl` unchanged until MediaService processing resolves the replacement public URL.

**Alternatives considered:**

- Add `PUT /api/v1/users/me/avatar`: rejected by product direction to keep avatar changes in the existing profile update flow.
- Let MediaService update UserService directly: rejected because MediaService must not own user business state.

## Decision: Assign Avatar Before Upload Confirmation

**Rationale:** The existing UserService media consumer finds the user by `AvatarFileId` when `MediaAssetProcessedIntegrationEvent` arrives. Assigning `AvatarFileId` before confirming the media upload avoids a race where processing completes before UserService knows the file belongs to the user.

**Alternatives considered:**

- Assign after confirmation: rejected because processing could publish before assignment.
- Add a saga: rejected because there is no compensation or orchestrator state requirement.

## Decision: Use Existing Events

**Rationale:** `MediaAssetProcessedIntegrationEvent` already covers file processing completion and `UserUpdatedIntegrationEvent` already includes `AvatarUrl`. No new message contract is necessary for this release.

**Alternatives considered:**

- Add `UserAvatarChangedIntegrationEvent`: rejected as duplicate semantics unless a future consumer needs avatar-specific facts.
- Add notification events: rejected because avatar changes do not require user-visible notifications.

## Decision: Sync User Avatar URL From Media Processing Event

**Rationale:** MediaService owns processing and public URL generation, while UserService owns the profile avatar fields. UserService must consume `MediaAssetProcessedIntegrationEvent` for `EntityType = "user_avatar"` and update the matching user where `AvatarFileId` equals the processed `FileId`. Matching on `AvatarFileId` prevents an older processed event from replacing a newer avatar selection.

**Alternatives considered:**

- Have the buyer app copy the media public URL into UserService: rejected because the browser should not own cross-service synchronization or trust derived media URLs.
- Have MediaService write UserService profile fields directly: rejected because MediaService must not write another service database.
- Match only on event `EntityId`: rejected because it could allow a stale avatar processing event to overwrite a newer `AvatarFileId`.

## Decision: Scope Avatar Display To Buyer App Surfaces

**Rationale:** The requested app is the buyer app. Existing buyer app shell already exposes `currentUserAvatarSrc` from the profile store to header/account surfaces, so refreshing the profile store is enough to update existing buyer avatar locations.

**Alternatives considered:**

- Profile page only: rejected because existing buyer account surfaces already consume profile avatar state.
- All apps: deferred because seller/admin UI changes were not requested.

## Decision: Initialize Avatar Picker From Saved Or Default Avatar

**Rationale:** The profile page avatar picker should not appear empty when profile data already has an avatar, and it should not show a blank image state when the buyer has no custom avatar. The shared `FileInput` should support an opt-in avatar display mode that renders a selected local preview first, then the saved `avatarUrl`, then the shared default avatar fallback. Existing non-avatar upload surfaces keep the current upload placeholder behavior.

**Alternatives considered:**

- Render the saved avatar separately from the upload control: rejected because it creates two avatar visuals and makes the picker state less clear.
- Replace the empty state globally for all file inputs: rejected because product image and document upload surfaces may still need the current upload placeholder.

## Decision: Hide Selected File Name In Avatar Picker

**Rationale:** The buyer profile page should show the image preview and validation/status feedback without displaying the selected file name below the avatar image section.

**Alternatives considered:**

- Keep shared `FileInput` file-name display: rejected for this page because it adds visual noise under the avatar section.
- Remove file-name display globally from the shared component: deferred because other upload surfaces may rely on it.

## Decision: Use One Combined Toast When Profile And Avatar Change Together

**Rationale:** When the buyer changes profile fields and selects a new avatar in the same save action, the UI should show one toast for the whole save result instead of separate profile and avatar toasts. A single toast avoids duplicate feedback and makes the async avatar state clear. If the avatar URL is resolved within the polling window, the toast should say that both profile and avatar were updated. If the profile fields are saved but the avatar is still processing, the toast should say the profile was updated and the new avatar will appear after processing finishes. If profile fields are saved but avatar confirmation/processing cannot complete, the toast should say the profile was updated but the avatar could not be completed.

**Alternatives considered:**

- Show a profile success toast followed by an avatar success/pending toast: rejected because two toasts for one save action are noisy and can look contradictory.
- Always show the avatar-only success toast when an avatar was selected: rejected because it hides that profile fields were also changed.
- Always show the generic profile success toast: rejected because it does not explain the avatar processing state.
