# Docs And Catalog Tasks: Tiki Catalog Crawl

## Service Documentation

### Update

- [ ] D001 [US1] [US3] Update `CatalogService docs for import bundle ownership`
  - File: `services/catalog-service/README.md`, `services/catalog-service/domain.md`, `services/catalog-service/api.md`, `services/catalog-service/data-and-events.md`
  - Document CatalogService ownership of async catalog import jobs, same-host background consumers, category provisioning, import bundles, imported catalog records, validation, duplicate grouping, seller provisioning orchestration, operator-approved seller ownership links, and selectable import publication state with `Draft` default plus `Unpublish` and `Available` support.
  - Add the admin catalog import endpoint table rows, including paginated job history, bundle section pagination endpoints, job status, and optional retry endpoints.
  - Add new CatalogService-owned tables, job statuses, and state transitions.
  - Do not describe IdentityService accounts, UserService stores, or MediaService binary storage as CatalogService-owned.
  - Acceptance: docs match `data-model.md` and `contracts/catalog-import-api.md`.

- [ ] D002 [US2] Update `IdentityService docs for imported seller account provisioning`
  - File: `services/identity-service/README.md`, `services/identity-service/api.md`, `services/identity-service/data-and-events.md`
  - Document `POST /api/v1/admins/imported-seller-accounts`, `RequireCatalogImportProvisioning`, idempotent create-or-match behavior, conflict reporting, and no token/password/session issuance.
  - Note that existing `IdentityUserReadyIntegrationEvent` is reused when a created imported seller account becomes usable.
  - Do not add store/profile ownership to IdentityService docs.
  - Acceptance: docs describe only identity-owned behavior and match the API catalog.

- [ ] D003 [US2] Update `UserService docs for imported seller store provisioning`
  - File: `services/user-service/README.md`, `services/user-service/api.md`, `services/user-service/data-and-events.md`
  - Document `POST /api/v1/admins/imported-seller-stores`, `RequireCatalogImportProvisioning`, idempotent create-or-match behavior, uniqueness/conflict behavior, and existing `StoreCreatedIntegrationEvent` publication for new stores.
  - Do not change seller self-registration semantics.
  - Acceptance: docs describe only UserService store ownership and match the API catalog.

## Shared Catalogs

### Update

- [ ] D004 [US1] [US2] [US3] Update `API catalog for catalog import endpoints`
  - File: `shared/api-catalog.md`
  - Add CatalogService admin category provisioning, category-attribute provisioning, and catalog import endpoints from `contracts/catalog-import-api.md`.
  - Add the paginated CatalogService import job history endpoint and paginated bundle section endpoints; note that job history returns all operation types, not only upload submissions.
  - Add the CatalogService seller ownership approval endpoint for explicitly linking a conflicted imported seller to an existing eligible HiveSpace account/store.
  - Add IdentityService imported seller account provisioning endpoint.
  - Add UserService imported seller store provisioning endpoint.
  - Confirm route ownership remains under `/api/v1/admins/**` and auth policies are `RequireAdmin` or `RequireCatalogImportProvisioning` as specified.
  - Acceptance: no duplicate endpoint rows exist and all new public routes are cataloged.

### Verify

- [ ] D005 [US1] [US2] [US3] Update `event catalog for observer-only background job lifecycle events`
  - File: `shared/event-catalog.md`
  - Add `BackgroundJobQueuedIntegrationEvent`, `BackgroundJobStartedIntegrationEvent`, `BackgroundJobProgressedIntegrationEvent`, `BackgroundJobCompletedIntegrationEvent`, and `BackgroundJobFailedIntegrationEvent`.
  - State that these events are for observer-only monitoring and that CatalogService job status remains the source of truth for catalog import progress, results, errors, and retry eligibility.
  - Confirm `IdentityUserReadyIntegrationEvent`, `StoreCreatedIntegrationEvent`, and `StoreUpdatedIntegrationEvent` still cover seller account/profile/store readiness and projection refresh.
  - Do not add new seller-provisioned events or central executor commands.
  - Acceptance: event catalog has the lifecycle events and no duplicate seller/account/store messages.

- [ ] D006 [US1] [US3] Verify `event catalog uses existing product and media domain events unchanged`
  - File: `shared/event-catalog.md`
  - Confirm `ProductCreatedIntegrationEvent`, `ProductSkuUpdatedIntegrationEvent`, and `MediaAssetProcessedIntegrationEvent` cover product/SKU projection and media processing completion.
  - Do not add a saga or new crawler-to-broker command/event; lifecycle events must remain generic observer facts, not product/media/import domain facts.
  - Acceptance: event catalog has no duplicate product/media/import-domain messages.

- [ ] D007 [US1] [US2] [US3] Verify `ADR need after implementation details settle`
  - File: `architecture/decisions/`
  - Re-check whether a new ADR is required for the new `../hivespace.crawler` repository boundary and CatalogService-owned import orchestration.
  - If required, create the next `ADR-[NNNN]-*.md` from `.specify/templates/architecture-decision-template.md` and link it from the feature plan/docs.
  - If not required, leave a short note in `specs/0012-tiki-catalog-crawl/plan.md` or implementation summary explaining that `research.md` already captured the decision.
  - Acceptance: architecture decisions either contain a new numbered ADR or the task result documents why no ADR was added.

