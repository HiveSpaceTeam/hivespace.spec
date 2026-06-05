# Config Tasks: Local Backend Runtime Orchestration

## hivespace.config Coordination

### Update

- [ ] C001 [US1] [US3] Update `config repo synchronized backend appsettings`
  - File: `../hivespace.config/**`
  - Run the backend repo's required config sync process if backend appsettings or environment source files changed.
  - Propagate only the new connection-string contract and local runtime settings needed by the backend source repo.
  - Preserve existing frontend gateway base URL assumptions and do not add frontend dev server orchestration.
  - Do not migrate existing Docker Compose local data or delete Docker Compose assets unless explicitly required by backend repo workflow docs.
  - Acceptance: config repo contains synchronized settings for `ConnectionStrings` broker/storage keys and no stale nested broker endpoint/secret source remains where sync owns those files.

- [ ] C002 [US1] [US3] Update `Docker Compose replacement notes`
  - File: `../hivespace.config/README.md`, `../hivespace.config/docker/**`
  - If config docs still present Docker Compose as the primary backend local development path, update them to point backend developers to the Aspire AppHost flow.
  - Keep Docker Compose references only for infrastructure/config maintenance that remains valid outside the backend-local startup path.
  - Do not claim existing Docker Compose data is migrated to Aspire-managed resources.
  - Acceptance: config docs no longer instruct backend developers to start the backend stack through Docker Compose as the preferred flow.

