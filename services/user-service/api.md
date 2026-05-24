# UserService API

## Profile and Settings

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/users/me` | `RequireAdminOrUser` | Get authenticated user profile, including nullable `avatarUrl` |
| PUT | `/api/v1/users/me` | `RequireAdminOrUser` | Update authenticated user profile; accepts optional `avatarFileId` when the current user changes avatar |
| GET | `/api/v1/users/settings` | `RequireAdminOrUser` | Get user settings |
| PUT | `/api/v1/users/settings` | `RequireAdminOrUser` | Update user settings |

### Profile Avatar Flow

- Avatar changes reuse `PUT /api/v1/users/me`; clients omit `avatarFileId` when no avatar change is requested.
- `avatarFileId` must reference a MediaService file ID from the direct-upload flow and targets only the authenticated current user.
- UserService stores `AvatarFileId` immediately and keeps the previous `AvatarUrl` until MediaService processing publishes the replacement URL.
- `GET /api/v1/users/me` returns the latest resolved `avatarUrl` after the media processing event is consumed.

## Addresses

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/users/address` | `Authorize` | List addresses |
| GET | `/api/v1/users/address/default` | `Authorize` | Get default address |
| GET | `/api/v1/users/address/{id}` | `Authorize` | Get address by ID |
| POST | `/api/v1/users/address` | `Authorize` | Create address |
| PUT | `/api/v1/users/address/{id}` | `Authorize` | Update address |
| DELETE | `/api/v1/users/address/{id}` | `Authorize` | Delete address |
| PUT | `/api/v1/users/address/{id}/default` | `Authorize` | Set default address |

## Stores

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/stores` | `RequireUser` | Register seller store |

## Admin Profile And Store Review

Admin workflows that read or update profile/store data remain UserService-owned when they do not change identity-owned credentials, roles, lockout, email verification, or account status. Identity-affecting admin account management is owned by IdentityService.
