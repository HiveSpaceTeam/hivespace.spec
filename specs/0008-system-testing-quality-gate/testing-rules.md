# Testing Rules: System Testing Quality Gate

Consolidated from `../hivespace.microservice/TESTING.md` and `../hivespace.web/TESTING.md`. These rules apply to all tests created under this feature.

---

## Coverage Thresholds

| Repo / Workspace | Measured scope | Target | Enforcement |
|------------------|---------------|--------|-------------|
| Backend - all services | Domain + Application layers (per `coverage.runsettings`) | 80% line | `quality-gate.ps1` (B021 adds check) |
| `hivespace.web/apps/buyer` | `src/pages/**, src/stores/**, src/composables/**, src/router/**` | 80% line | existing `coverage.ps1` |
| `hivespace.web/apps/seller` | `src/pages/**, src/stores/**, src/composables/**, src/router/**` | 80% line | existing `coverage.ps1` |
| `hivespace.web/apps/admin` | `src/pages/**, src/stores/**, src/composables/**, src/router/**` | 80% line | existing `coverage.ps1` |
| `hivespace.web/packages/shared` | `src/features/**, src/composables/**, src/test-utils/**` | 80% line | existing `coverage.ps1` |

Coverage below threshold is treated as an incomplete deliverable, not a passing result.

---

## Backend Rules (.NET / xUnit)

### Framework stack

- **Test runner**: xUnit 2.5+
- **Assertions**: FluentAssertions 6.x
- **Coverage**: Coverlet collector (`--collect:"XPlat Code Coverage"`) with `coverage.runsettings`
- **In-memory persistence**: `Microsoft.EntityFrameworkCore.InMemory`
- **Shared fakes**: `HiveSpace.Testing.Shared`

### File structure

One test file per production class. Mirror the source folder structure inside `Application/`:

```text
tests/HiveSpace.<Service>.Tests/
|-- Domain/                    # Pure unit tests - no DB, no fakes, no DI
|   `-- <Aggregate>Tests.cs
|-- Application/               # Handler/query tests - real application unit, using the lightest valid test pattern
|   `-- <Feature>/
|       `-- <Handler>Tests.cs
|-- Consumers/                 # Consumer/event handler tests
|   `-- <Consumer>Tests.cs
`-- Fixtures/
    `-- <Service>Fixture.cs    # Owns in-memory DbContext when fixture-backed tests are required
```

### Naming convention

```text
Method_Condition_ExpectedOutcome
```

Examples:
- `Handle_WithValidEmailAndPassword_CreatesAccountWithBuyerRole`
- `Handle_WithExpiredToken_ReturnsError`
- `UpdateProfile_WithValidFields_ChangesStoredValues`

### Layer constraints

| Layer | Test type | Fixture | Fakes allowed |
|-------|-----------|---------|---------------|
| Domain | Pure `[Fact]` | None - no `IClassFixture` | None - pure in-process logic only |
| Application | `[Fact]` executing the real handler/query/service unit | Either NSubstitute with no fixture when orchestration verification is sufficient, or `IClassFixture<ServiceFixture>` with in-memory EF Core when persistence/query round-tripping must be observed | `PaymentProviderFake`, `EmailDeliveryFake`, `BlobStorageFake`, `SignalRHubFake`, `DeterministicClock`, `FakeCurrentUser` |
| Consumers | `[Fact]` with `IClassFixture<ServiceFixture>` or MassTransit harness, depending on the consumer type | Same as Application when persistence is needed | `InMemoryMessageCapture` |

### Application test decision rule

- Use **NSubstitute** when the test verifies handler orchestration and mock verification is sufficient.
- Use **`IClassFixture<ServiceFixture>` with in-memory EF Core** when the test must round-trip through persistence or observe query behavior from stored data.
- Application tests must still execute the real Application-layer unit for the service architecture in use.

### Fake catalogue (from `HiveSpace.Testing.Shared`)

| Class | Purpose |
|-------|---------|
| `PaymentProviderFake` | Configurable VNPay-shaped stub; `SetupReturn(ref, result)` / `GetStub(ref)` |
| `EmailDeliveryFake` | Captures `(To, Subject, Body)` in `Sent` list; no SMTP calls |
| `BlobStorageFake` | Deterministic presign URLs; tracks `ConfirmedKeys`; no network calls |
| `SignalRHubFake` | Captures hub invocations in `Invocations` list; `Emit(method, args)` for setup |
| `DeterministicClock` | Implements `TimeProvider`; constructor accepts `DateTime utcNow` |
| `FakeCurrentUser` | Builds `ClaimsPrincipal` with configurable role, user ID (ULID), `StoreId` |
| `InMemoryMessageCapture` | Implements `IPublishEndpoint`; collects `IIntegrationEvent` in `Published` list |
| `TestDataBuilder` | `NewUlid()`, `MoneyStub(decimal, currency)`, `AddressStub(city)` |

### Coverage scope (from `coverage.runsettings`)

**Included:**
- `[HiveSpace.*.Application]*`
- `[HiveSpace.*.Domain]*`
- `[HiveSpace.*.Core]*`

**Excluded:**
- `[HiveSpace.*.Api]*`
- `[HiveSpace.*.Infrastructure]*`
- `[HiveSpace.ServiceDefaults]*`, `[HiveSpace.AppHost]*`
- `[HiveSpace.Core]*` (generic shared lib)
- `[HiveSpace.Application.Shared]*`, `[HiveSpace.Domain.Shared]*`
- `[HiveSpace.*.Tests]*`, `[HiveSpace.Testing.Shared]*`
- Sub-paths: `Core/Infrastructure/**`, `Core/Persistence/**`, `Core/Interfaces/**`, `Core/Extensions/**`, `Migrations/**`

### Smoke tests do not count

Do not count these as Application-layer coverage:

- `typeof(Handler).Should().NotBeNull()`
- Multi-handler smoke tests that never execute the orchestration logic
- Direct aggregate mutation in an `Application/` test when the handler is the real unit under test

### What NOT to test

- `Api/` controller plumbing (request routing, model binding boilerplate)
- `Infrastructure/` EF Core migrations and context registration
- `Infrastructure/` extension methods and DI registration
- Generated code (`[GeneratedCode]`, `[CompilerGenerated]`)
- Classes decorated with `[ExcludeFromCodeCoverage]`

---

## Frontend Rules (Jest)

### Framework stack

- **Test runner**: Jest 29 (`jest.base.cjs` factory)
- **Browser environment**: jsdom
- **Component testing**: `@testing-library/vue`
- **Store testing**: `@pinia/testing`
- **Coverage provider**: v8 (configured in `jest.base.cjs`)

### File structure

Co-locate test files next to source files. Never use a top-level `__tests__/` folder.

```text
apps/<app>/src/
|-- stores/
|   |-- cart.ts
|   `-- cart.test.ts          # store test alongside store
`-- pages/
    `-- Cart/
        |-- CartPage.vue
        `-- CartPage.test.ts  # page test alongside component
```

Jest `testMatch`: `src/**/*.test.ts`

### Naming convention

| Test type | Pattern | Example |
|-----------|---------|---------|
| Store | `should …` plain English sentence | `should increment cart count when item is added` |
| View/Page | `should …` plain English sentence | `should render empty state when cart has no items` |
| Route guard | `should …` plain English sentence | `should redirect to login when user is unauthenticated` |

### Layer constraints

| Test type | Mount component? | Use store? | HTTP calls? |
|-----------|-----------------|-----------|-------------|
| Store test | No | `@pinia/testing` | No - stub via `stubApiResponse` |
| View/page test | Yes - `@testing-library/vue` | Yes - wire up real Pinia | No - stub via `stubApiResponse` |
| Composable test | No | Call directly | No |
| Router test | No | Call guard directly | No |

### Shared test-utils (`@hivespace/shared/test-utils`)

| Export | Signature | Use for |
|--------|-----------|---------|
| `createFakeAuthUser` | `(overrides?: Partial<AppUser>): AppUser` | Build typed user with role, id, storeId |
| `createFakeAuthState` | `(role: 'buyer' \| 'seller' \| 'admin')` | Pinia-compatible auth store state |
| `createMockAxios` | `(): AxiosInstance` | Axios instance with interceptors stripped |
| `stubApiResponse` | `(instance, method, path, data, status?)` | Queue a response for path+method |
| `createFakeSignalRHub` | `(): {invocations, emit}` | Capture hub calls; trigger events |
| `createFakePresignResponse` | `(overrides?): {url, uploadRef}` | Fake presign URL response |
| `simulateUploadConfirm` | `(uploadRef): {confirmed, mediaId}` | Fake upload confirmation response |
| `createTestI18n` | `(messages?): I18n` | vue-i18n instance with English locale |

### i18n assertion rule

All text assertions MUST use i18n key lookups. Never assert on hardcoded display strings.

```ts
// Correct
expect(wrapper.text()).toContain(i18n.t('cart.emptyState.title'))

// Wrong
expect(wrapper.text()).toContain('Your cart is empty')
```

### Coverage scope (from `coverage.ps1`)

**Included (apps: buyer, seller, admin):**
- `src/pages/**/*.ts`, `src/pages/**/*.vue`
- `src/stores/**/*.ts`
- `src/composables/**/*.ts`
- `src/router/**/*.ts`

**Included (shared package):**
- `src/features/**/*.ts`, `src/features/**/*.vue`
- `src/composables/**/*.ts`
- `src/test-utils/**/*.ts`

**Excluded everywhere:**
- `**/index.ts` (barrel files)
- `src/assets/**`, `src/styles/**`
- `src/components/**` (presentational only)
- `src/icons/**`
- `src/i18n/**` (locale JSON files)
- `src/config/**`
- `src/services/**` (thin axios wrappers)
- `src/types/**`
- `src/test/**`

### What NOT to test

- Locale JSON files (`en.json`, `vi.json`)
- Icon SVG wrapper components
- Barrel `index.ts` files
- Config constants (`src/config/constants.ts`)
- Thin service wrappers (`src/services/*.service.ts` that only call axios)
- Type-only files
