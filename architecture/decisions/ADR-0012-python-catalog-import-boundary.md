# ADR-0012: Python Catalog Import Boundary

- **Status**: Accepted
- **Date**: 2026-07-24
- **Feature**: `0012-tiki-catalog-crawl`
- **Deciders**: Project maintainers

## Context

HiveSpace needs to crawl Tiki catalog data, preserve external seller identity, create seller account/store ownership, validate records against HiveSpace catalog rules, and import valid products with an operator-selected `Draft`, `Unpublish`, or `Available` publication state.

The user prefers Python for crawling, referenced `../hivespace.crawl-data` as prior crawl experimentation, and clarified that production crawler work should use a new repository. HiveSpace backend service boundaries require IdentityService to own accounts, UserService to own stores, CatalogService to own catalog records, and MediaService to own binary media processing.

## Decision

Create a new sibling Python crawler/exporter repository at `../hivespace.crawler` as the external-source adapter and make CatalogService the authoritative import-bundle and catalog-validation owner.

The Python tool emits category and category-attribute provisioning payloads first, then product import bundles only after categories and category attributes have been provisioned. It does not write HiveSpace databases and does not decide final import readiness. The existing `../hivespace.crawl-data` repo remains reference material only. CatalogService provisions categories and category attributes, stores product bundles, validates them, groups duplicates, coordinates seller provisioning through owning services, and creates products through existing catalog rules with the accepted publication state. IdentityService and UserService expose idempotent `RequireCatalogImportProvisioning` APIs for imported seller accounts and stores.

No new MassTransit saga is introduced. Existing identity, store, product, SKU, and media events remain the propagation mechanism after owning services commit state.

## Consequences

### Positive

- External Tiki access is isolated from backend service runtime reliability.
- Python can evolve quickly as Tiki response shapes change.
- HiveSpace service boundaries remain intact.
- Import bundles are reviewable, versioned, and testable.
- CatalogService remains the single owner of category provisioning and product import readiness.

### Negative / Trade-offs

- The import workflow spans a new Python repo, backend services, and admin frontend tasks.
- CatalogService needs application ports/clients for IdentityService and UserService provisioning.
- Operators need an explicit category-first provision/review/import flow instead of one fully automatic crawl-to-publish path.

### Risks

- Tiki source shape or access behavior may change unexpectedly; mitigate with versioned source adapters, fixture tests, and source validation hints.
- Automated seller account/store creation can create low-quality or conflicting seller records; mitigate with idempotent matching, conflict status, admin review, and audit metadata.
- Media URLs may become stale before import; mitigate by preserving source URLs as external references and copying through MediaService before publication where required.

## Alternatives Considered

| Option | Why rejected |
| --- | --- |
| Put Tiki crawling inside CatalogService | Couples external-source volatility to catalog runtime and expands CatalogService beyond catalog ownership. |
| Let Python write directly to service databases | Violates database ownership and bypasses domain validation, authorization, and outbox behavior. |
| Build production crawler code in `../hivespace.crawl-data` | Keeps experimental/reference code as a long-term production boundary and makes ownership unclear. |
| Create a new ImportService | Adds a new service boundary before there is enough need and still depends on CatalogService for authoritative validation. |
| Use a MassTransit import saga | The first release is admin-triggered and idempotent; no required long-running state machine or compensation workflow exists. |

## Follow-Up

- User-owned end-to-end verification remains tracked in `specs/0012-tiki-catalog-crawl/tasks/verification.md`.
