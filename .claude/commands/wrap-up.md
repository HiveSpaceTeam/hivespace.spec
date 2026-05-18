Update documentation after a feature has shipped.
Usage: `/wrap-up [NNNN-feature-name]`

Step 1 - Resolve the feature
- If a feature name is supplied, use `specs/[feature-name]`.
- Otherwise read `.specify/feature.json`; if absent, use the most recently modified folder under `specs/`.
- Read `spec.md`, `plan.md`, and `tasks.md`.

Step 2 - Update service docs
Classify every service mentioned by the feature before editing:
- Owning service: domain data, business rules, workflow, API, or events changed for the feature.
- Changed supporting service: a supporting/common service had an actual API, event, validation, workflow, ownership, or behavior change.
- Reused supporting service: the feature only consumed an existing common capability.

For owning services and changed supporting services:
- Read `services/<name>/README.md` and related `api.md`, `data-and-events.md`, or `workflows.md` files when present.
- Add shipped endpoints.
- Add published and consumed events.
- Add workflow or saga changes.
- Add ADR links for decisions that shaped the implementation.
- Remove known limitations that the shipped feature resolved.

For reused supporting services:
- Read docs as verification context only.
- Do not update their service docs just to mention feature-specific use of an existing common capability.
- Example: if UserService stores a buyer avatar reference and consumes existing MediaService upload/processed-event contracts unchanged, update UserService docs only.

Step 3 - Verify catalogs
- Read `shared/event-catalog.md` and verify all shipped events/messages are present with correct ownership and consumers.
- Read `shared/api-catalog.md` and verify all shipped endpoints are present with correct auth and service ownership.
- Add missing catalog entries only for new contracts.
- Update existing catalog entries only when the actual endpoint/message contract, owner, auth policy, semantics, or consumer set changed.
- Do not rewrite common catalog rows just to mention one feature-specific use.

Step 4 - Mark spec done
- In the feature `spec.md`, set status to `Implemented`.
- Add the implementation date.

Step 5 - Summary
- Show a table of every file changed with a one-line description of the change.
