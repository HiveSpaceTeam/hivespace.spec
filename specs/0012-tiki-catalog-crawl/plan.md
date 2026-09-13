# Implementation Plan: [FEATURE_NAME]

**Branch:** [NNNN-feature-name]
**Date:** [DATE]
**Spec:** [link to spec.md]

> Before writing anything in this file, read `.specify/memory/constitution.md`

---

## Phase 0 — Research

### Existing context (read before planning)

- [ ] `services/<owner-or-supporting>/README.md` — existing aggregates, events, endpoints
- [ ] `shared/event-catalog.md` — verify no event name conflicts
- [ ] `shared/api-catalog.md` — verify no endpoint conflicts
- [ ] Existing saga code (if this feature extends a saga)

### Technical unknowns

- [List anything that needs investigation before implementation]

### Research notes

- Current design artifacts are captured in `research.md`, `data-model.md`, `contracts/`, and `tasks/`; this file still contains the original plan template.
- Catalog import long-running operations use CatalogService-owned asynchronous jobs processed by background consumers in the same CatalogService host.
- No MassTransit saga is required because catalog import is operator-triggered, idempotent, retryable, and does not need cross-service compensation state.
- No separate Azure Function, worker project, or central executor is planned for v1. A future central monitoring service may consume background job lifecycle events for observability only.
- No separate ADR was added in this update because `research.md` captures the service-owned async job decision and alternatives; revisit before implementation if this becomes a reusable platform-wide job pattern.

---

## Phase 1 — Architecture & Data Model

### Service placement

Which service owns this feature and why?
Reference constitution Article I (service boundaries).

Classify every service mentioned by the feature:

| Service | Classification | Reason | Documentation/catalog action |
| ------- | -------------- | ------ | ---------------------------- |
| [Service] | Owning service / Changed supporting service / Reused supporting service | [Why this service is involved] | [Update docs/catalogs, or verification-only] |

Rules:

- Owning services have feature-owned domain, data, workflow, API, or event changes and may need service doc updates.
- Changed supporting services may need docs/catalog updates only for actual API, event, validation, workflow, ownership, or behavior changes.
- Reused supporting services are context only. Do not update their docs or shared catalog rows when existing contracts are reused unchanged.
- Shared catalogs change only for new contracts or actual changes to existing endpoint/message contract, owner, auth, semantics, or consumer set.

### New aggregates

Catalog import entities are defined in `data-model.md`: `CatalogImportJob`, `CatalogImportBundle`, `ImportedSeller`, `SellerOwnershipLink`, `ExternalCategoryLink`, `ImportedProduct`, `ImportedSku`, `ImportedAttribute`, `ImportedImageReference`, `ImportValidationIssue`, and `ImportDuplicateGroup`.

EF Core tables use snake_case names under CatalogService. The implementation tasks specify the migration as `AddCatalogImports`.

### New repository interfaces

CatalogService needs repository methods for adding and idempotently matching import jobs and bundles, paginated all-operation job history, job detail by ID, paginated bundle summaries, paginated bundle detail sections, and job progress/result/error updates.

### Integration events (must go through Outbox)

| Event | Producer | Consumer(s) |
| ----- | -------- | ----------- |
| `BackgroundJobQueuedIntegrationEvent` | CatalogService | Observer-only monitoring |
| `BackgroundJobStartedIntegrationEvent` | CatalogService | Observer-only monitoring |
| `BackgroundJobProgressedIntegrationEvent` | CatalogService | Observer-only monitoring |
| `BackgroundJobCompletedIntegrationEvent` | CatalogService | Observer-only monitoring |
| `BackgroundJobFailedIntegrationEvent` | CatalogService | Observer-only monitoring |

Existing `IdentityUserReadyIntegrationEvent`, `StoreCreatedIntegrationEvent`, `ProductCreatedIntegrationEvent`, `ProductSkuUpdatedIntegrationEvent`, and `MediaAssetProcessedIntegrationEvent` are reused where their existing ownership semantics apply.

If this feature reuses an existing common event unchanged, list it as reused and state that no catalog update is required.

### API endpoints

| Method | Path | Auth | Purpose |
| ------ | ---- | ---- | ------- |
| POST | `/api/v1/admins/catalog-imports/categories/provisioning` | `RequireAdmin` | Queue category provisioning from uploaded category JSON |
| POST | `/api/v1/admins/catalog-imports/categories/attributes/provisioning` | `RequireAdmin` | Queue category-attribute provisioning from uploaded category-attribute chunk JSON |
| POST | `/api/v1/admins/catalog-imports/bundles` | `RequireAdmin` | Queue product bundle submission from uploaded bundle JSON |
| GET | `/api/v1/admins/catalog-imports/jobs` | `RequireAdmin` | Paginated all-operation import job history for the list/upload page |
| GET | `/api/v1/admins/catalog-imports/jobs/{jobId}` | `RequireAdmin` | Job detail, progress, result, and linked bundle reference |
| GET | `/api/v1/admins/catalog-imports/bundles/{bundleId}` | `RequireAdmin` | Paginated linked bundle review sections for the detail page |
| POST | `/api/v1/admins/catalog-imports/bundles/{bundleId}/validate` | `RequireAdmin` | Queue bundle validation from the detail page |
| POST | `/api/v1/admins/catalog-imports/bundles/{bundleId}/seller-provisioning` | `RequireAdmin` | Queue seller account/store provisioning from the detail page |
| POST | `/api/v1/admins/catalog-imports/bundles/{bundleId}/sellers/{importedSellerId}/ownership-link` | `RequireAdmin` | Approve a seller ownership link from the detail page |
| POST | `/api/v1/admins/catalog-imports/bundles/{bundleId}/import` | `RequireAdmin` | Queue ready-product import with `Draft`, `Unpublish`, or `Available` publication state from the detail page |

If this feature reuses an existing common endpoint unchanged, list it as reused and state that no catalog update is required.

---

## Phase 2 — Implementation Plan

### Layer order (always domain → application → infrastructure → api)

**Domain layer**

- [ ] Catalog import job, bundle, imported record, validation issue, seller ownership link, and duplicate group entities from `data-model.md`
- [ ] Repository interfaces for paginated job history, job detail, bundle list/detail sections, idempotency lookup, and worker progress updates

**Application layer**

- [ ] Category provisioning, category-attribute provisioning, and product bundle submission commands that return job submissions with source file name metadata when available
- [ ] Paginated job history, job detail, bundle list, and bundle detail queries
- [ ] Validation, seller provisioning, seller ownership approval, selected or bundle-wide ready-product import, retry, and same-host worker orchestration

**Infrastructure layer**

- [ ] EF configurations, `AddCatalogImports` migration, repository implementation, Identity/User provisioning HTTP clients, and same-host job workers
- [ ] Transactional-outbox publishing for observer-only background job lifecycle events

**API layer**

- [ ] CatalogService admin catalog import endpoints from `contracts/catalog-import-api.md`
- [ ] IdentityService and UserService system-admin imported seller provisioning endpoints
- [ ] ApiGateway route coverage for `/api/v1/admins/catalog-imports/**`, `/api/v1/admins/imported-seller-accounts`, and `/api/v1/admins/imported-seller-stores`

### Saga design (if applicable)

Create `saga-design.md` alongside this file from `.specify/templates/saga-design-template.md` only when this feature introduces or changes a MassTransit saga state machine.

Do not create `saga-design.md` for ordinary cross-service events, direct-upload flows, simple async consumers, or REST workflows that do not add/change saga state.

If a saga is needed, the plan must link to `saga-design.md`, and the saga design must define owner service, participants, states, messages, compensation, timeouts, idempotency, and observability.

If no saga is needed: state "No saga required" and explain why the feature does not add or change MassTransit saga state.

### Architecture decision (if applicable)

Create `architecture/decisions/ADR-[NNNN]-[short-slug].md` from `.specify/templates/architecture-decision-template.md` when this feature makes a non-obvious architecture, service-boundary, data-ownership, messaging, or cross-repo decision with meaningful alternatives.

Number ADRs sequentially by checking `architecture/decisions/`, keep status `Draft` during planning, and link the ADR from this plan.

---

## Phase 3 — Frontend Plan

### Surface(s)

- [ ] buyer
- [ ] seller
- [x] admin

### Files to create (mandatory order: types → service → store → components → view → route → i18n)

| File                          | Notes                            |
| ----------------------------- | -------------------------------- |
| `types/catalog-import.types.ts` | Import job, bundle, upload, pagination, seller/product/category/issue/duplicate DTOs |
| `services/catalog-import.service.ts` | Thin API wrapper for upload submissions, paginated bundle history, related job history, job detail, bundle detail, workflow actions, and retry |
| `stores/catalog-import.store.ts` | Pinia store for list/upload state, bundle history pagination, selected bundle side-pane state, category upload history, selected job detail, linked bundle detail, per-table pagination, polling, and workflow actions |
| `components/catalog-imports/*` | Upload panel, bundle history table, related jobs side pane, category upload history table, job summary panel, bundle summary, category links, sellers, products/SKUs, duplicate groups, validation issue tables |
| `pages/catalog-imports/CatalogImportListPage.vue` | Upload category/product files, show paginated product bundle history, and show selected-bundle jobs in a side pane |
| `pages/catalog-imports/CatalogImportJobDetailPage.vue` | Inspect one job, poll status, review linked bundle, run validation/provision/import actions, resolve seller ownership conflicts |
| `router/index.ts` | Add admin-only `/catalog-imports` and `/catalog-imports/jobs/:jobId` routes |
| `i18n/locales/{en,vi}/catalog-imports.json` | List/detail page, upload, job history, pagination, statuses, actions, summaries, conflicts, validation, and import copy |

List/upload page responsibilities:

- Submit/provision crawled Tiki category files.
- Upload/submit product import bundle files.
- Show a paginated product-bundle table as the primary review surface.
- Keep category provisioning uploads in a separate paginated section.
- Keep the main bundle table visible while a selected bundle opens a right-side related-jobs pane.
- Load bundle-specific jobs into the side pane without forcing navigation away from the page.
- Navigate to the job detail page from bundle-related job rows.
- Format list-page timestamps as operator-friendly local datetime plus relative hint.

Detail page responsibilities:

- Poll and inspect the selected import job.
- Show linked bundle summary and paginated category links, imported sellers, products/SKUs, duplicate groups, and validation issues.
- Start validation, seller provisioning, selected ready-product import, and bundle-wide import-all-ready jobs.
- Resolve seller/store ownership conflicts with explicit operator approval.
- Keep import actions disabled until categories, seller ownership, validation, and duplicate checks are ready.
- Submit bundle-wide import without client-side aggregation of every ready product ID from paginated tables.
- Execute ready-product import jobs against a submit-time snapshot of eligible imported product IDs.

### i18n keys to add (both en.json and vi.json)

```json
{
  "catalogImports": {
    "list": {},
    "detail": {},
    "upload": {},
    "jobs": {},
    "pagination": {},
    "statuses": {},
    "sellers": {},
    "products": {},
    "categories": {},
    "duplicates": {},
    "validation": {},
    "actions": {},
    "notifications": {}
  }
}
```

---

## Constitution Compliance Check

Before implementation starts, verify all of these:

- [ ] No hard-deletes planned
- [ ] All money values are long
- [ ] All IDs use ULID
- [ ] All events go through MassTransit Outbox
- [ ] No Version= in any new .csproj dependencies
- [ ] Both en.json and vi.json will be updated
- [ ] Frontend text uses $t() keys only
