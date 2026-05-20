# Feature Specification: Split Identity Service

- **Feature Branch**: `0002-split-identity-service`
- **Created**: 2026-05-20
- **Status**: Draft
- **Input**: User description: "Split UserService into two services: IdentityService for credentials, authentication, authorization, roles, IdentityServer, MS Identity, Identity User; remaining UserService focuses on profile, address, settings, and stores. Route changes are acceptable. Store registration publishes an event from UserService to IdentityService for role propagation. Breaking migration is acceptable during development."

## Clarifications

### Session 2026-05-20

- Q: How should the profile record be created after a new identity account is created? -> A: IdentityService publishes account-created event; UserService creates profile.
- Q: When admin user-management actions need to affect both identity data and profile data, what should be the source of truth for account status? -> A: IdentityService owns account status; UserService does not keep a separate profile status.
- Q: Which service should own email verification after the split? -> A: IdentityService owns email verification.
- Q: Should UserService profile records use the same public user ID as IdentityService, or have separate profile IDs linked to identity IDs? -> A: Same public user ID across IdentityService and UserService.
- Q: Should the split keep a `/identity/**` route and what service shape should IdentityService use? -> A: No `/identity/**` route; create IdentityService as a LiteService.
- Q: Which service should own temporary lockout and suspended/inactive account status? -> A: IdentityService owns both lockout and suspended/inactive account status.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Authenticate Through Dedicated Identity Boundary (Priority: P1)

Platform users can register, sign in, receive access, and keep their role-based access through a dedicated identity boundary that is separate from profile, address, settings, and store management.

**Why this priority**: Authentication and authorization are required before any buyer, seller, or admin workflow can function.

**Independent Test**: Can be tested by completing sign-in for an existing buyer, seller, and admin account and confirming each user reaches only the application areas allowed by their role.

**Acceptance Scenarios**:

1. **Given** an existing active buyer account, **When** the buyer signs in, **Then** the buyer receives authenticated access and can use buyer-only areas without profile data being required by the identity boundary.
2. **Given** an existing admin account, **When** the admin signs in, **Then** the admin receives admin access and non-admin users remain blocked from admin-only areas.
3. **Given** an inactive or invalid account, **When** sign-in is attempted, **Then** access is denied and no profile or store data is exposed.

---

### User Story 2 - Manage Profile Data In User Boundary (Priority: P2)

Authenticated users can manage profile, settings, addresses, and store registration through the user boundary while identity data remains owned by the identity boundary.

**Why this priority**: The split must preserve the user-facing account and store workflows while reducing responsibility overlap between services.

**Independent Test**: Can be tested by updating profile details, settings, and addresses after sign-in and confirming identity credentials and roles are not modified by those profile-only actions.

**Acceptance Scenarios**:

1. **Given** an authenticated user, **When** the user updates profile details or settings, **Then** the changes are reflected in profile views without changing credentials or roles.
2. **Given** an authenticated buyer, **When** the buyer creates, edits, selects, or deletes an address according to existing address rules, **Then** the address state is maintained under the user boundary.
3. **Given** an authenticated eligible user, **When** the user registers a store, **Then** the store is created under the user boundary and a role-change request is emitted for the identity boundary.

---

### User Story 3 - Propagate Seller Access From Store Registration (Priority: P3)

After a user registers a store, the platform grants seller access through identity role propagation so seller workflows become available after the user's authorization state refreshes.

**Why this priority**: Store registration crosses the new boundary and is the highest-risk business workflow affected by the split.

**Independent Test**: Can be tested by registering a store as a buyer and confirming seller access is unavailable before role propagation and available after role propagation plus authorization refresh.

**Acceptance Scenarios**:

1. **Given** a buyer without seller access, **When** store registration succeeds, **Then** the system records the store and requests seller-role assignment from the identity boundary.
2. **Given** a successful store registration, **When** role propagation completes, **Then** the user can refresh authorization state and access seller workflows.
3. **Given** role propagation cannot complete, **When** the user attempts seller-only access, **Then** access remains denied and the failed propagation is observable for remediation.

---

### User Story 4 - Migrate Development Data To Split Ownership (Priority: P4)

Developers can reset or migrate development data into the split ownership model so identity-owned records and user-owned records are clearly separated.

**Why this priority**: The project is in development, so a breaking migration is acceptable, but the resulting data ownership must be unambiguous.

**Independent Test**: Can be tested by applying the migration path in a development environment and confirming account access, profile access, address access, and store access use the new ownership model.

**Acceptance Scenarios**:

1. **Given** a development environment with current user data, **When** the breaking migration path is applied, **Then** identity records are owned by the identity boundary and profile/store records are owned by the user boundary.
2. **Given** migrated development data, **When** a user signs in and views profile, address, settings, or store data, **Then** the expected data is available through the proper boundary.

### Edge Cases

- Store registration succeeds but seller-role propagation is delayed or fails.
- A user has identity data but no matching profile record after a development migration.
- A profile or store record references a user that no longer exists in identity ownership after a development reset.
- Public route changes leave an outdated frontend or gateway client calling old routes.
- An inactive account attempts profile, address, settings, or store actions after the split.
- Admin account management crosses identity and user ownership and must not partially update both boundaries without an observable outcome.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The platform MUST define a dedicated identity boundary named `IdentityService` that owns account credentials, sign-in, sign-out, token/session issuance, authorization roles, claims, and identity user state.
- **FR-002**: The platform MUST retain a user boundary named `UserService` that owns user profile, user settings, user addresses, store registration, and store lifecycle data.
- **FR-003**: Identity-owned data MUST be separated from profile/store-owned data so each boundary has one authoritative owner for its records.
- **FR-004**: Users MUST be able to authenticate and receive role-appropriate access after the split.
- **FR-005**: Authenticated users MUST be able to view and update profile data, settings, addresses, and store data through the user boundary after the split.
- **FR-006**: Store registration MUST request seller-role assignment from the identity boundary through an asynchronous cross-boundary message after the store is successfully created.
- **FR-007**: Seller-only access MUST depend on authorization state owned by the identity boundary, not on direct reads of store data by authorization checks.
- **FR-008**: The system MUST make role propagation failures observable so developers or operators can diagnose and retry or repair failed seller-role assignment.
- **FR-009**: Public client routes MAY change as part of the split, but all changed routes MUST be documented and all affected clients MUST be updated together.
- **FR-010**: The migration path MAY be breaking for development environments, but it MUST leave a clear way to recreate or migrate development accounts, roles, profiles, addresses, settings, and stores.
- **FR-011**: Existing downstream user and store reference consumers MUST continue to receive the user/store facts they need after the split, unless planning explicitly replaces those contracts.
- **FR-012**: Admin account and user-management workflows MUST clearly state which actions affect identity-owned data, user-owned data, or both.
- **FR-013**: No service outside the owning boundary MAY directly read or write identity, profile, address, settings, or store records from another boundary.
- **FR-014**: Identity account creation MUST publish an account-created fact that UserService consumes to create the matching user profile record.
- **FR-015**: UserService MUST make account-created processing failures observable so missing profile records can be diagnosed and repaired during development.
- **FR-016**: IdentityService MUST be the source of truth for account status used by authentication and authorization decisions.
- **FR-017**: UserService MUST NOT define a separate user/profile status for account availability; account availability belongs to IdentityService.
- **FR-018**: IdentityService MUST own email verification state and verification workflows because verified email affects account security and authorization claims.
- **FR-019**: UserService MUST NOT be the authoritative source for email verification, but it MAY display verification state received from identity-owned facts when needed for user experience.
- **FR-020**: IdentityService and UserService MUST use the same public user ID for a user so existing client and downstream references can remain stable while data ownership is separated.
- **FR-021**: UserService MUST treat the shared public user ID as a reference to identity ownership, not as permission to read or write identity-owned records directly.
- **FR-022**: IdentityService MUST own temporary failed-login lockout state and suspended/inactive account status because both affect sign-in and authorization.
- **FR-023**: UserService MUST NOT use profile data to allow, deny, lock, or suspend authentication.

### Key Entities

- **Identity User**: The identity-owned account record used for credentials, sign-in state, account status relevant to authentication, roles, claims, email verification, and authorization decisions.
- **User Profile**: The user-owned profile record containing display and contact information that is not required to authenticate; it uses the same public user ID as the matching identity user.
- **User Settings**: User-owned preferences such as language, theme, and other account experience settings.
- **Address**: User-owned delivery or contact address associated with a profile.
- **Store**: User-owned seller store record created through store registration and linked to the account that owns it.
- **Role Assignment Request**: A cross-boundary request that asks identity ownership to grant or update a user's role after a user-owned workflow succeeds.
- **Account Created Fact**: A cross-boundary fact emitted after identity account creation so user ownership can create the corresponding profile record.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of identity capabilities are assigned to `IdentityService` and 100% of profile, address, settings, and store capabilities are assigned to `UserService` in the final planning artifacts.
- **SC-002**: Buyer, seller, and admin users can complete sign-in and reach their role-appropriate landing areas in a development environment after the split.
- **SC-003**: Users can complete profile update, settings update, address management, and store registration workflows after the split without direct cross-boundary data access.
- **SC-004**: A newly registered store owner can gain seller access after role propagation and authorization refresh in at least 95% of normal development test runs.
- **SC-005**: All changed public routes and all new or changed cross-boundary messages are documented before implementation begins.
- **SC-006**: Development environment setup documents a repeatable data reset or migration path that can be completed in under 30 minutes by a developer familiar with the project.

## Assumptions

- The target shape is two deployable backend services: `IdentityService` and `UserService`.
- Route changes are acceptable, provided the gateway, frontend clients, and catalogs are updated together.
- The planned identity route family does not include `/identity/**`; browser-facing identity routes should use versioned API routes.
- The migration may be breaking because the project is still in development.
- Store registration remains owned by UserService, while seller authorization remains owned by IdentityService.
- Role propagation after store registration is asynchronous and requires the user's authorization state to refresh before seller-only access is available.
- New user profile creation is event-driven from IdentityService to UserService after account creation.
- Account status for sign-in and authorization belongs to IdentityService; UserService does not keep a separate profile status.
- Temporary failed-login lockout and suspended/inactive account status belong to IdentityService.
- Email verification belongs to IdentityService.
- IdentityService and UserService share the same public user ID for a user, while keeping separate data ownership.
- Planning will determine whether existing user/store integration events are reused, renamed, or supplemented.
- Planning will include an architecture decision record because this changes service boundaries and data ownership.
