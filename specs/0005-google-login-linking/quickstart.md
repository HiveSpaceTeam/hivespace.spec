# Quickstart: Google Login and Account Linking

## Prerequisites

- Local infrastructure from `../hivespace.config/docker` is running.
- IdentityService, ApiGateway, buyer app, and seller app are configured with matching Google OAuth settings and frontend origins. Real Google OAuth client secrets may be stored only in local unpushed backend appsettings or .NET user-secrets.
- Google OAuth redirect URI points to the implemented completion endpoint under `/api/v1/accounts/external/google/complete`.
- Feature implementation settings are supplied from backend/frontend source or local developer configuration.

## Backend Checks

1. Build IdentityService:

   ```bash
   cd ../hivespace.microservice
   dotnet build src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/HiveSpace.IdentityService.Api.csproj
   ```

2. If gateway route/config changes were made, build ApiGateway:

   ```bash
   dotnet build src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/HiveSpace.YarpApiGateway.csproj
   ```

3. Confirm no response contract exposes access or refresh tokens in JSON.

## Frontend Checks

1. Build or type-check shared package and affected apps using the commands available in `../hivespace.web`.
2. Confirm buyer and seller sign-in pages expose Google.
3. Confirm buyer and seller sign-up pages expose Google and start the same Google challenge flow.
4. Confirm admin sign-in does not expose Google sign-in.
5. Confirm English and Vietnamese auth copy are updated together.

## Manual Scenarios

### New Buyer Google User

1. Open buyer app sign-in.
2. Choose Google using an email with no matching HiveSpace email/password account.
3. Complete Google authentication.
4. Expect a normal user account, signed-in buyer session, no browser-readable access/refresh token, and matching UserService profile creation through the existing event flow.

Repeat from buyer app sign-up and expect the same backend outcome.

### New Seller Google User

1. Open seller app sign-in.
2. Choose Google using an email with no matching HiveSpace email/password account.
3. Complete Google authentication.
4. Expect a normal user account, not seller access.
5. Expect existing seller guard/onboarding to route the user through email verification or store registration as currently designed.

Repeat from seller app sign-up and expect the same backend outcome.

### Existing Password Account Link

1. Create or use an existing email/password buyer or seller account with no Google login linked.
2. Start Google sign-in using the same verified email.
3. Expect the frontend account-link prompt.
4. Decline once and verify no durable link, no signed-in Google session, and no duplicate Google-only HiveSpace account are created.
5. Confirm the declined-link outcome offers only safe exits: password sign-in, password reset, or choosing another Google account.
6. Repeat, leave explicit consent unchecked, enter the correct password if the UI allows it, and verify no link is created, no duplicate account is created, and the UI or API returns a safe validation outcome.
7. Repeat, accept explicit consent, enter a wrong password, and verify no link is created, no duplicate account is created, and lockout rules apply.
8. Repeat, accept explicit consent, enter the correct password, and verify:
   - Google external login is linked;
   - account email is marked verified;
   - browser session cookies and CSRF token are issued;
   - future Google sign-in skips the link prompt.

### No Matching Password Account

1. Choose Google using a verified email that does not belong to an existing local email/password account.
2. Expect no account-link prompt.
3. Expect the normal Google path: linked Google accounts sign in directly; otherwise a normal user account is created and signed in.

### Matching Password Account Cannot Continue Without Linking

1. Use a verified Google email that belongs to an existing local email/password account with no Google link.
2. Complete Google authentication and reach the account-link prompt.
3. Attempt to continue without linking.
4. Expect no duplicate Google-only account, no session, and no account mutation.
5. Expect the UI to guide the user to password sign-in, password reset, or choosing another Google account.

### Blocked Accounts

1. Use a disabled, suspended, inactive, or locked account.
2. Start Google sign-in with a linked or matching Google email.
3. Expect sign-in/linking to be blocked with a safe user-facing error.

### Google Email Problems

1. Simulate missing or unverified Google email.
2. Expect account creation and linking to be rejected.

### Legacy Pages

1. Visit old user-facing account URLs such as `/Account/Login`, `/Account/Register`, `/Account/Logout`, consent, grants, device, diagnostics, CIBA, and old external login pages.
2. Expect no legacy user-facing IdentityServer account page to render.
3. Required protocol endpoints such as `/.well-known/**` and `/connect/**` must remain available.
4. Confirm the old `AccountCompatibilityEndpoints` redirect behavior is gone.
