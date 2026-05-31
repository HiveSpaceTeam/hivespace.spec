# Config Tasks

## Backend Config

### Update

- [ ] C001 [US1] [US2] [US3] [US4] Update browser token cookie configuration
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/appsettings.json`, `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/appsettings.json`, `../hivespace.config/**/appsettings*.json`
  - Add cookie names `__Host-HiveSpace.AccessToken`, `__Host-HiveSpace.RefreshToken`, and `HiveSpace.Csrf`, plus CSRF header `X-HiveSpace-CSRF`, where code reads options.
  - Do not add shared Data Protection key-ring settings for the auth cookies; token cookies are stored without an application-level encrypted envelope.
  - Configure access-token lifetime/validation options consistently between IdentityService issuance and ApiGateway validation.
  - Do not hardcode secrets or environment-specific absolute paths that cannot run on another machine.
  - Acceptance: IdentityService-issued access-token cookie can be read and validated by ApiGateway locally without Data Protection decryption.

- [ ] C002 [US1] [US2] [US3] [US4] Update frontend origin and cookie settings
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/appsettings.json`, `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/appsettings.json`
  - Confirm allowed origins include `http://localhost:5173`, `http://localhost:5174`, and `http://localhost:5175`.
  - Add frontend fallback URLs for old account URL redirects by app: admin, seller, buyer.
  - Ensure token/CSRF cookie names and CSRF header values match C001.
  - Acceptance: config supports shared local session across all three frontend app origins.

## Frontend Env

### Update

- [ ] C003 [US1] [US2] [US3] [US4] Update app env examples for gateway session auth
  - File: `../hivespace.web/apps/admin/.env.example`, `../hivespace.web/apps/seller/.env.example`, `../hivespace.web/apps/buyer/.env.example`
  - Set or document `VITE_GATEWAY_BASE_URL=http://localhost:5000` for local HTTP gateway verification.
  - Remove OIDC-only variables from required local auth setup if no runtime code uses them after migration.
  - Keep unrelated app config keys unchanged.
  - Acceptance: a developer can run auth flows locally through ApiGateway on port 5000 using env examples.

- [ ] C004 [US1] [US2] [US3] [US4] Update frontend app config typing for browser auth
  - File: `../hivespace.web/apps/admin/src/config/index.ts`, `../hivespace.web/apps/seller/src/config/index.ts`, `../hivespace.web/apps/buyer/src/config/index.ts`
  - Add `auth.app` or equivalent app id and `auth.gatewayBaseUrl` if shared auth initialization requires it.
  - Keep `config.api.baseUrl` gateway fallback behavior compatible with existing services.
  - Do not keep required redirect URI/post-logout URI config in the critical browser auth path.
  - Acceptance: shared auth initialization can be configured without OIDC redirect settings.

## Config Sync

### Verify

- [ ] C005 Verify config sync expectations before PR
  - File: `../hivespace.microservice/AGENTS.md`, `../hivespace.config/**`
  - If backend appsettings files are changed during implementation, run or plan `bash scripts/sync-config.sh` from `../hivespace.microservice` before PR.
  - Do not stage JSON files automatically; source repo guardrails require user-handled JSON staging for commits.
  - Acceptance: config changes are mirrored to `hivespace.config` and JSON staging constraints are documented in implementation notes.
