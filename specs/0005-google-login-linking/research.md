# Research: Google Login and Account Linking

## Decision: Put Google browser auth under IdentityService account endpoints and expose it on sign-in and sign-up pages

**Rationale**: IdentityService already owns credentials, external login state, account status, lockout, email verification, token cookies, and account REST endpoints under `/api/v1/accounts/**`. Adding Google challenge/callback/link endpoints under that prefix keeps browser auth consistent with password login and lets ApiGateway reuse the existing account route. Buyer and seller sign-in and sign-up pages should both start the same Google challenge because the callback determines whether to sign in, create a normal user account, or start account linking.

**Alternatives considered**:

| Alternative | Why rejected |
| --- | --- |
| Keep `Pages/ExternalLogin/*` as the primary flow | It preserves old IdentityServer-hosted user-facing pages, which the feature explicitly removes. |
| Add Google flow to each frontend app without IdentityService endpoints | Frontend apps must not own provider tokens, credentials, or durable account-link decisions. |
| Route Google auth through `/Account/**` | Old account URL compatibility is not required during development and should not remain the target user journey. |
| Add separate Google sign-in and Google sign-up backend endpoints | Duplicates the provider challenge without changing the backend decision tree. |

## Decision: Use ASP.NET Identity external login storage for durable Google links

**Rationale**: IdentityService already uses ASP.NET Identity. `AspNetUserLogins` is the correct identity-owned store for provider name and provider user key. It avoids a parallel linking table and keeps external login behavior aligned with IdentityService ownership.

**Alternatives considered**:

| Alternative | Why rejected |
| --- | --- |
| New custom `google_account_links` table | Duplicates ASP.NET Identity external login semantics without a clear need. |
| Store only Google email on `ApplicationUser` | Email can change and is not a stable provider identity. |
| Store Google link in UserService profile | Violates the boundary because UserService must not own credentials or external login state. |

## Decision: Require password-confirmed linking for existing email/password accounts

**Rationale**: A Google email match alone is not enough to prove ownership of an existing HiveSpace password account. The link flow must require explicit consent and correct password confirmation, while applying the same lockout/account-status protections as password login. The link prompt is only for a same-email local password account that has no Google link; if such an account exists, the user must link, sign in with password, reset password, or use another Google account. The system must not create a second Google-only account for the same email. If no same-email password account exists, the callback continues the normal Google sign-in/sign-up path.

**Alternatives considered**:

| Alternative | Why rejected |
| --- | --- |
| Auto-link on verified Google email match | Higher account takeover risk if an account's email control has changed or was compromised. |
| Let the user continue as a separate Google-only account with the same email | Creates duplicate HiveSpace identities, split profile/order/store state, confusing support cases, and possible role/profile conflicts. |
| Block linking and require support/admin action | Too high friction for normal users and does not meet the requested self-service flow. |
| Email one-time code instead of password | Adds a notification dependency and a second verification workflow not requested for this feature. |

## Decision: New Google users are normal user accounts

**Rationale**: Seller role and store ownership are already granted through UserService seller onboarding and `StoreCreatedIntegrationEvent`. Google sign-in from the seller app must not bypass that flow.

**Alternatives considered**:

| Alternative | Why rejected |
| --- | --- |
| Create seller-capable accounts from seller Google sign-in | Bypasses the existing seller/store onboarding workflow. |
| Allow Google sign-up only from buyer app | Blocks a reasonable seller entry point and adds avoidable UX friction. |

## Decision: No MassTransit saga

**Rationale**: The flow is an IdentityService-local browser authentication and linking operation. Cross-service effects reuse the existing account-created event for new users. There is no multi-service compensation, timeout, or long-running state machine.

**Alternatives considered**:

| Alternative | Why rejected |
| --- | --- |
| Add a saga for account creation and profile creation | Existing profile creation already uses `IdentityUserCreatedIntegrationEvent`; adding a saga would over-coordinate unchanged behavior. |

## Decision: Delete IdentityService `Pages/` and `AccountCompatibilityEndpoints`

**Rationale**: The target auth experience is frontend-owned. Keeping Razor Pages or compatibility redirects would preserve a second account URL surface and conflict with the requirement that old user-facing IdentityService account pages are not part of the supported journey.

**Alternatives considered**:

| Alternative | Why rejected |
| --- | --- |
| Keep compatibility redirects for `/Account/Login`, `/Account/Register`, and `/Account/Logout` | Compatibility is not required during development and the user explicitly asked to delete `AccountCompatibilityEndpoints`. |
| Keep non-auth Razor pages such as diagnostics, grants, device, and CIBA | The user asked to remove the entire `Pages` folder; required protocol behavior must be preserved through non-page endpoints only. |
| Hide page links but leave page files mapped | Old URLs could still render stale user-facing pages. |

## Decision: Create ADR-0004

**Rationale**: The feature makes a security-sensitive auth decision with meaningful alternatives: auto-link vs password-confirmed link, Razor hosted flow vs frontend-owned flow, and provider identity storage choices. Future maintainers need the reason recorded outside the feature plan.

**Alternatives considered**:

| Alternative | Why rejected |
| --- | --- |
| Only reference ADR-0003 | ADR-0003 covers cookie browser sessions, not Google account linking or old external-login page removal. |
