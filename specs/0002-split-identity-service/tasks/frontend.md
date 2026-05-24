# Frontend Tasks

## Source Repo Preparation

### Verify

- [ ] F001 Verify frontend repo instructions and shared-first inventory
  - File: `../hivespace.web/AGENTS.md`, `../hivespace.web/CLAUDE.md`, `../hivespace.web/packages/shared/src/`
  - Read both frontend instruction files and inspect existing shared auth, profile, settings, address, media, modal, notification, route, service, store, type, and i18n modules.
  - Do not create app-local shared behavior when an existing `@hivespace/shared` capability can be extended.
  - Acceptance: implementation notes identify shared modules to update before app-local changes.

## Shared Package

### Update

- [ ] F002 [US1] Update shared OIDC authority and auth refresh service
  - File: `../hivespace.web/packages/shared/src/features/auth/refresh.service.ts`, `../hivespace.web/packages/shared/src/features/auth/`
  - Point login, token refresh, account, and email verification calls at IdentityService-owned authority/routes, with local authority `http://localhost:5001` where direct authority config is used.
  - Do not route IdentityServer public endpoints through ApiGateway or `/api/v1/identity/**`.
  - Acceptance: shared auth uses IdentityService authority directly for OIDC public endpoints.

- [ ] F003 [US2] Update shared user profile service routes
  - File: `../hivespace.web/packages/shared/src/features/user-profile/user-profile.service.ts`, `../hivespace.web/packages/shared/src/features/user-settings/`
  - Keep profile/settings calls on UserService-owned `/api/v1/users/**` routes through the shared HTTP client.
  - Do not call identity-owned account/admin routes for profile-only behavior.
  - Acceptance: profile/settings reads and writes use UserService routes only.

- [ ] F004 [US1] Update shared auth/backend error i18n keys
  - File: `../hivespace.web/packages/shared/src/i18n/locales/en/backend-errors.json`, `../hivespace.web/packages/shared/src/i18n/locales/vi/backend-errors.json`
  - Add or move keys for lockout, suspended/inactive account, delayed profile creation, and identity-owned email verification errors as needed.
  - Update English and Vietnamese together and do not reuse one key as both string and object namespace.
  - Acceptance: shared error copy resolves in both locales without hardcoded UI strings.

## Admin App

### Update

- [ ] F005 [US1] Update admin auth and identity account routes
  - File: `../hivespace.web/apps/admin/src/config/index.ts`, `../hivespace.web/apps/admin/src/services/user.service.ts`
  - Point OIDC authority directly to IdentityService `http://localhost:5001`.
  - Split identity-affecting admin account actions from profile/store review routes according to `contracts/api-routes.md`.
  - Acceptance: admin app signs in through IdentityService and identity account management calls IdentityService-owned APIs.

- [ ] F006 [US2] Update admin profile/store review flows
  - File: `../hivespace.web/apps/admin/src/services/user.service.ts`, `../hivespace.web/apps/admin/src/router/index.ts`
  - Keep profile/store review routes on UserService-owned APIs where those workflows remain profile/store data.
  - Do not use profile status to allow, deny, lock, or suspend authentication.
  - Acceptance: admin profile/store views compile against split ownership.

## Seller App

### Update

- [ ] F007 [US1] Update seller auth and email verification routes
  - File: `../hivespace.web/apps/seller/src/config/index.ts`, `../hivespace.web/apps/seller/src/services/account.service.ts`
  - Point OIDC authority directly to IdentityService `http://localhost:5001` and move email verification calls to IdentityService-owned account routes.
  - Do not use `/identity/**` or `/api/v1/identity/**`.
  - Acceptance: seller auth and email verification flow use IdentityService endpoints.

- [ ] F008 [US3] Update seller store registration token refresh flow
  - File: `../hivespace.web/apps/seller/src/stores/store.store.ts`, `../hivespace.web/apps/seller/src/router/index.ts`, `../hivespace.web/apps/seller/src/i18n/locales/en/register-store.json`, `../hivespace.web/apps/seller/src/i18n/locales/vi/register-store.json`
  - Keep store registration on UserService `/api/v1/stores/**`; after success, handle delayed seller-role propagation and require authorization refresh before seller-only access.
  - Update English and Vietnamese copy for propagation/delay state.
  - Acceptance: seller app does not assume immediate role availability until refreshed token contains seller role/claims.

## Buyer App

### Update

- [ ] F009 [US2] Update buyer profile, settings, address, and auth config
  - File: `../hivespace.web/apps/buyer/src/config/index.ts`, `../hivespace.web/apps/buyer/src/services/profile.service.ts`, `../hivespace.web/apps/buyer/src/services/user-settings.service.ts`, `../hivespace.web/apps/buyer/src/services/address.service.ts`
  - Point OIDC authority directly to IdentityService `http://localhost:5001`.
  - Keep profile/settings/address APIs on UserService-owned `/api/v1/users/**` routes.
  - Acceptance: buyer sign-in and profile/settings/address workflows compile against split route ownership.
