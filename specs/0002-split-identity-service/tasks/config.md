# Config Tasks

## Backend Runtime Config

### Update

- [ ] C001 [US1] Update IdentityService local launch and appsettings port
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Properties/launchSettings.json`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/appsettings.json`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/appsettings.*.json`
  - Set local service URL, issuer, authority, and metadata references to `http://localhost:5001`.
  - Remove stale IdentityService `5007` values unless explicitly documented as previous-state migration notes.
  - Acceptance: IdentityService runs locally on `http://localhost:5001`.

- [ ] C002 [US2] Update UserService local launch and appsettings port
  - File: `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Properties/launchSettings.json`, `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/appsettings.json`, `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/appsettings.*.json`
  - Set UserService local service URL to `http://localhost:5007` while JWT validation authority points to IdentityService `http://localhost:5001`.
  - Remove UserService-hosted IdentityServer/ASP.NET Identity config sections no longer owned by UserService.
  - Acceptance: UserService runs locally on `http://localhost:5007` and validates IdentityService-issued JWTs.

- [ ] C003 [US1] Update downstream service JWT authority config
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/**/appsettings*.json`, `../hivespace.microservice/src/HiveSpace.OrderService/**/appsettings*.json`, `../hivespace.microservice/src/HiveSpace.PaymentService/**/appsettings*.json`, `../hivespace.microservice/src/HiveSpace.MediaService/**/appsettings*.json`, `../hivespace.microservice/src/HiveSpace.NotificationService/**/appsettings*.json`
  - Point JWT authority, issuer, audience, and metadata address references to IdentityService `http://localhost:5001` where present.
  - Do not change unrelated service business configuration.
  - Acceptance: all protected services validate IdentityService-issued tokens.

- [ ] C004 [US1] Update ApiGateway route clusters
  - File: `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/appsettings.json`
  - Route IdentityService-owned REST APIs such as retained `/api/v1/accounts/**` and identity-affecting admin APIs to IdentityService `http://localhost:5001`.
  - Route UserService-owned `/api/v1/users/**` and `/api/v1/stores/**` to UserService `http://localhost:5007`.
  - Remove `/identity/**` and do not forward `/.well-known/**`, `/connect/**`, or `/Account/**`.
  - Acceptance: gateway config matches `contracts/api-routes.md`.

## Config Repo

### Update

- [ ] C005 [US4] Update Docker Compose service definitions
  - File: `../hivespace.config/docker/docker-compose.yml`, `../hivespace.config/docker/`
  - Add or update IdentityService container/service entries on port `5001` and UserService entries on port `5007`.
  - Keep dependencies for SQL Server, RabbitMQ, Redis, Azurite, and other infrastructure unchanged unless required by service startup.
  - Acceptance: local compose config can run both services without port collision.

- [ ] C006 [US4] Update synced appsettings/local settings sources
  - File: `../hivespace.config/`
  - Update config-source copies for IdentityService `5001`, UserService `5007`, gateway clusters, and downstream JWT authority.
  - Do not manually diverge synced config from backend appsettings after `scripts/sync-config.sh`.
  - Acceptance: config repo contains the same target port ownership as backend appsettings.

- [ ] C007 [US4] Update frontend environment defaults
  - File: `../hivespace.web/apps/admin/.env*`, `../hivespace.web/apps/seller/.env*`, `../hivespace.web/apps/buyer/.env*`, `../hivespace.web/packages/shared/`
  - Point OIDC authority/env defaults to IdentityService `http://localhost:5001`; keep gateway base URL on ApiGateway defaults.
  - Do not introduce `/api/v1/identity/**`.
  - Acceptance: frontend development env uses direct IdentityService authority and existing gateway base for REST APIs.
