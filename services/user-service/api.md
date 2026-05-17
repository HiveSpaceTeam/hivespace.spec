# UserService API

## Profile and Settings

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/users/me` | `RequireAdminOrUser` | Get authenticated user profile |
| PUT | `/api/v1/users/me` | `RequireAdminOrUser` | Update authenticated user profile |
| GET | `/api/v1/users/settings` | `RequireAdminOrUser` | Get user settings |
| PUT | `/api/v1/users/settings` | `RequireAdminOrUser` | Update user settings |

## Email Verification

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/accounts/email-verification` | `RequireAdminOrUser` | Send verification email |
| POST | `/api/v1/accounts/email-verification/verify` | Anonymous | Verify email token |

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

## Admin

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/admins` | `RequireAdmin` | Create admin account |
| GET | `/api/v1/admins` | `RequireAdmin` | List admin accounts |
| GET | `/api/v1/admins/users` | `RequireAdmin` | List user accounts |
| PUT | `/api/v1/admins/users/status` | `RequireAdmin` | Update user/admin status |
| DELETE | `/api/v1/admins/users/{userId}` | `RequireAdmin` | Delete or deactivate user |
