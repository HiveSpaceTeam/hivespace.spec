# ADR-0004: Google Account Linking

- **Status**: Draft
- **Date**: 2026-05-31
- **Feature**: `0005-google-login-linking`
- **Deciders**: Project maintainers

## Context

HiveSpace already migrated browser password login and registration to frontend-owned pages backed by IdentityService `/api/v1/accounts/**` endpoints that issue HttpOnly token cookies. IdentityService still contains legacy IdentityServer/Razor external login pages that auto-provision external users and redirect to old `/Account/Login` error paths.

The new feature requires buyer and seller users to sign in with Google, link matching email/password accounts only after explicit consent and password confirmation, mark the account email verified after successful linking with a verified Google email, and remove old user-facing IdentityServer account page flows. Admin Google login is out of scope. If a verified Google email matches an existing unlinked password account, HiveSpace must not create a second Google-only account for that same email.

## Decision

Implement Google login as IdentityService-owned browser account endpoints under `/api/v1/accounts/external/google/**`, exposed through the existing ApiGateway account route prefix. Durable Google links use ASP.NET Identity external login storage. If Google returns a verified email that matches an existing password account, IdentityService creates short-lived pending-link state and the frontend asks the user to consent and enter the existing password. Only after correct password confirmation does IdentityService add the Google login, mark the email verified, clear pending state, and issue the normal browser session cookies. Declined, failed, abandoned, or expired linking must not fall through to separate same-email Google account creation.

New Google-authenticated users are created as normal user accounts. Seller access still requires the existing UserService seller/store onboarding flow and `StoreCreatedIntegrationEvent` role propagation. Admin accounts cannot be created or signed in through Google.

## Consequences

### Positive

- Keeps external login state, account status, lockout, roles, email verification, and token issuance inside IdentityService.
- Prevents email-match-only account linking and reduces account takeover risk.
- Prevents duplicate same-email HiveSpace identities and split profile/order/store state.
- Reuses the existing cookie browser session contract after successful Google sign-in or linking.
- Reuses the existing `IdentityUserCreatedIntegrationEvent` profile creation path for new Google users.
- Removes old IdentityServer-hosted user-facing account pages from the supported HiveSpace auth journey.

### Negative / Trade-offs

- Requires a temporary pending-link state and an additional frontend link-confirmation flow.
- Google sign-in from seller app may first land users in existing seller onboarding rather than direct seller access.
- Admins cannot use Google sign-in convenience in this feature.
- Old user-facing account URL compatibility is intentionally not preserved during development.

### Risks

- Misconfigured Google redirect URIs can break sign-in; mitigate with config sync and quickstart checks.
- Pending-link state could expose provider identifiers if implemented incorrectly; mitigate by keeping provider keys/tokens in HttpOnly or server-side state only.
- Removing Razor pages could accidentally remove protocol pages still needed by Duende IdentityServer; mitigate by removing only user-facing HiveSpace account UI and preserving required `/.well-known/**` and `/connect/**` behavior.
- Email verification event behavior may diverge if linking updates `EmailConfirmed` outside the existing verification path; implementation must either reuse the existing email-verified event path or document why no downstream event is needed.

## Alternatives Considered

| Option | Why rejected |
| --- | --- |
| Auto-link accounts when Google verified email matches | Email match alone is insufficient protection for existing password accounts. |
| Let users continue as a separate Google-only account with the same email | Creates duplicate identities, split user data, support ambiguity, and possible role/profile conflicts. |
| Keep the old Razor `ExternalLogin` pages as the primary Google flow | Conflicts with frontend-owned auth and legacy page removal. |
| Create a custom Google-link table | Duplicates ASP.NET Identity external login storage. |
| Store Google link state in UserService | Violates service boundaries because UserService must not own credentials or external login state. |
| Allow Google for admin accounts | Increases administrative account risk and was explicitly scoped out. |

## Follow-Up

- Add Google account endpoint rows to `shared/api-catalog.md`.
- Update IdentityService docs for Google external login ownership and legacy page removal.
- Generate implementation tasks for backend, shared frontend auth, buyer/seller pages, admin exclusion verification, config sync, and verification.
