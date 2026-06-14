# Backend Tasks: System Testing Quality Gate

**Repo**: `../hivespace.microservice`
**Prerequisites**: Read `../hivespace.microservice/AGENTS.md` and `CLAUDE.md` before starting.

---

## Shared Backend Test Infrastructure

### Update

- [ ] B001 Update `Directory.Packages.props` — add test package versions
  - File: `../hivespace.microservice/Directory.Packages.props`
  - Verify existing entries: `Microsoft.NET.Test.Sdk`, `xunit`, `xunit.runner.visualstudio`, `FluentAssertions`, `coverlet.collector`
  - Add `<PackageVersion>` entries for any missing packages using the same version as `HiveSpace.PaymentService.Tests` references
  - Do not add `Version=` attributes in any `.csproj`; all versions must remain in `Directory.Packages.props`
  - Acceptance: `dotnet restore` succeeds after adding new test projects that reference these packages without any `Version=` override

### Create

- [ ] B002 Create `HiveSpace.Testing.Shared` project
  - File: `../hivespace.microservice/tests/HiveSpace.Testing.Shared/HiveSpace.Testing.Shared.csproj`
  - Target: `netX.0` matching the service projects (use the same `TargetFramework` as `HiveSpace.PaymentService.Tests`)
  - PackageReference (no `Version=`): `xunit`, `FluentAssertions`, `Microsoft.AspNetCore.Mvc.Testing` (if applicable)
  - Folder layout:
    - `Fakes/` — stateful test doubles that simulate external provider behavior
    - `Doubles/` — lightweight non-stateful helpers used by any test layer
    - `Capture/` — utilities that capture in-process messages or events for assertion
    - `Builders/` — test data factories
  - `Doubles/DeterministicClock.cs`: implements `TimeProvider` or `IDateTimeProvider`; constructor accepts `DateTime utcNow`; returns injected value for all date/time calls
  - `Doubles/FakeCurrentUser.cs`: builds a `ClaimsPrincipal` with configurable role (`buyer`, `seller`, `admin`), user ID (ULID string), and optional `StoreId` claim
  - `Capture/InMemoryMessageCapture.cs`: collects `IIntegrationEvent` instances published during a test; exposes `Published` list for assertion; implements `IPublishEndpoint` or the relevant MassTransit publish interface
  - `Fakes/PaymentProviderFake.cs`: returns configurable VNPay-shaped response stubs via `SetupReturn(TransactionRef, VNPayResult)` and `GetStub(TransactionRef)`; throws if no stub configured
  - `Fakes/EmailDeliveryFake.cs`: captures outgoing email payloads (`To`, `Subject`, `Body`) in a `Sent` list; does not make any SMTP or API call
  - `Fakes/BlobStorageFake.cs`: simulates presign URL generation returning a deterministic URL string; simulates upload confirmation; tracks `ConfirmedKeys` list
  - `Fakes/SignalRHubFake.cs`: captures realtime hub invocations in an `Invocations` list of `(MethodName, Args)`; exposes `Emit(methodName, args)` for test setup
  - `Builders/TestDataBuilder.cs`: static helpers `NewUlid()` (deterministic increment), `MoneyStub(decimal amount, string currency = "VND")`, `AddressStub(string city = "Hanoi")`
  - Do not include: product business logic, EF Core migrations, live API calls, or any infrastructure that requires a running process
  - Acceptance: project compiles; each fake/builder instantiates in an xUnit test with `[Fact]` without any running infrastructure dependency

---

## Quality Gate Runner

### Create

- [ ] B003 [US1] Create backend quality-gate runner `quality-gate.ps1`
  - File: `../hivespace.microservice/quality-gate.ps1`
  - Accept parameter `-Scope` (string); validate against: `docs-only`, `backend:IdentityService`, `backend:UserService`, `backend:CatalogService`, `backend:OrderService`, `backend:PaymentService`, `backend:MediaService`, `backend:NotificationService`, `shared`, `release`
  - For `docs-only`: emit CheckResult with `status: not_applicable` for all runtime checks; do not invoke `dotnet test`
  - For `backend:<serviceName>`: run shared baseline (`dotnet test tests/HiveSpace.Testing.Shared`) then `dotnet test tests/HiveSpace.<serviceName>.Tests`
  - For `shared`: run `dotnet test tests/HiveSpace.Testing.Shared`
  - For `release`: run `dotnet test` from repo root (all test projects)
  - Rerun policy: on non-zero exit, re-invoke once; if second run also non-zero and exit code differs from first, set `status: unstable_check`; if both runs have same non-zero exit, set `status: fail`
  - Emit JSON output matching `contracts/quality-gate-report.md`: one `gate` object, one `checkResult` per test project, `coverageGaps: []`, `acceptedRisks: []`
  - Map `dotnet test` pass to `status: pass, blockingDecision: none`; failure to `status: fail, failureCategory: product_behavior, blockingDecision: merge_blocking`
  - Do not use real payment credentials; do not make any external network call; reference `HiveSpace.Testing.Shared` fakes only
  - Acceptance: `.\quality-gate.ps1 -Scope release` produces valid JSON; `.\quality-gate.ps1 -Scope backend:PaymentService` invokes only `HiveSpace.PaymentService.Tests`; `.\quality-gate.ps1 -Scope docs-only` produces `not_applicable` results without running tests

---

## IdentityService Tests

### Create

- [ ] B004 [US2] Create `HiveSpace.IdentityService.Tests` project
  - File: `../hivespace.microservice/tests/HiveSpace.IdentityService.Tests/HiveSpace.IdentityService.Tests.csproj`
  - Reference: `HiveSpace.Testing.Shared`; no `Version=` in any `<PackageReference>`
  - Folder layout:
    - `Application/` — handler-level integration tests; each class uses `IClassFixture<IdentityServiceFixture>`
    - `Fixtures/IdentityServiceFixture.cs` — creates in-memory `IdentityDbContext` with `Guid.NewGuid()` database name in `InitializeAsync`
  - Application/ folder mirrors source Application/ layer; one file per handler in sub-feature sub-folders:
  - `Application/AccountRegistration/RegisterBuyerCommandHandlerTests.cs` (uses `IdentityServiceFixture`):
    - `Handle_WithValidEmailAndPassword_CreatesAccountWithBuyerRole`
    - `Handle_WithDuplicateEmail_ReturnsConflictError`
  - `Application/AccountRegistration/RegisterSellerCommandHandlerTests.cs` (uses `IdentityServiceFixture`):
    - `Handle_WithValidInputs_CreatesAccountWithPendingSellerStatus`
  - `Application/EmailVerification/VerifyEmailCommandHandlerTests.cs` (uses `IdentityServiceFixture`, `DeterministicClock`):
    - `Handle_WithValidTokenWithinExpiry_VerifiesAccountAndSetsActive`
    - `Handle_WithExpiredToken_ReturnsError`
    - `Handle_WithInvalidToken_ReturnsError`
  - `Application/Session/LoginCommandHandlerTests.cs` (uses `IdentityServiceFixture`, `FakeCurrentUser`):
    - `Handle_WithVerifiedCredentials_ReturnsSessionWithCorrectClaims`
    - `Handle_WithUnverifiedAccount_IsRejected`
    - `Handle_WithAdminCredentials_GrantsAdminClaims`
  - `Application/Session/LogoutCommandHandlerTests.cs` (uses `IdentityServiceFixture`):
    - `Handle_ClearsSessionForAuthenticatedUser`
  - `Application/AdminAccounts/SuspendAccountCommandHandlerTests.cs` (uses `IdentityServiceFixture`):
    - `Handle_WithActiveAccount_SetsStatusToSuspended`
    - `Handle_ProducesAuditRecord`
  - `Application/AdminAccounts/ActivateAccountCommandHandlerTests.cs` (uses `IdentityServiceFixture`):
    - `Handle_WithSuspendedAccount_SetsStatusToActive`
    - `Handle_ProducesAuditRecord`
  - Use `EmailDeliveryFake` — no real emails sent
  - Acceptance: `dotnet test tests/HiveSpace.IdentityService.Tests` exits with code 0; no live SMTP or identity provider calls

---

## UserService Tests

### Create

- [ ] B006 [US2] Create `HiveSpace.UserService.Tests` project
  - File: `../hivespace.microservice/tests/HiveSpace.UserService.Tests/HiveSpace.UserService.Tests.csproj`
  - Reference: `HiveSpace.Testing.Shared`; no `Version=`
  - Folder layout:
    - `Domain/` — pure aggregate/value-object unit tests; no DB, no fakes, no DI. Plain `[Fact]`.
    - `Application/` — handler-level integration tests; each class uses `IClassFixture<UserServiceFixture>`
    - `Fixtures/UserServiceFixture.cs` — creates in-memory `UserDbContext` with `Guid.NewGuid()` database name in `InitializeAsync`
  - `Domain/UserTests.cs` (no fixture — plain `[Fact]` only):
    - `Create_WithUsernameTooShort_ThrowsDomainException`: username < 3 characters rejected
    - `Create_WithUsernameTooLong_ThrowsDomainException`: username > 50 characters rejected
    - `Create_WithUsernameContainingInvalidCharacters_ThrowsDomainException`: special characters rejected by domain rule
    - `UpdateProfile_WithValidFields_ChangesStoredValues`: updated fields readable after mutation
    - `MarkAddressAsDefault_ClearsPreviousDefaultFlag`: only one address can be default at a time
    - `RemoveAddress_WhenOnlyAddress_CanBeRemovedReturnsFalse`: domain guard prevents removing last address
  - `Domain/EmailTests.cs` (no fixture — plain `[Fact]` only):
    - `Create_WithValidFormat_Succeeds`: well-formed email constructs without error
    - `Create_WithInvalidFormat_ThrowsInvalidEmailException`: malformed email rejected
    - `TwoEmailsWithSameAddress_AreEqual`: value object equality via `GetEqualityComponents()`
  - `Domain/StoreTests.cs` (no fixture — plain `[Fact]` only):
    - `Create_WithValidFields_StartsInActiveStatus`: newly created store has active status
    - `Deactivate_ActiveStore_TransitionsToInactive`: deactivation changes status
    - `Activate_InactiveStore_TransitionsToActive`: activation restores status
    - `Create_WithNameTooShort_ThrowsDomainException`: store name < 2 characters rejected
    - `Create_WithDescriptionTooLong_ThrowsDomainException`: description > 500 characters rejected
  - `Domain/StoreManagerTests.cs` (no fixture — uses mock repository stub, not EF):
    - `RegisterStore_WhenUserAlreadyOwnsStore_ThrowsUserStoreExistsException`: one-store-per-user enforced
    - `RegisterStore_WithDuplicateNameCaseInsensitive_ThrowsConflictException`: name uniqueness is case-insensitive
    - `RegisterStore_WithForbiddenCharactersInName_ThrowsInvalidStoreInformationException`: `<>"|\\/*?:` characters rejected
  - `Domain/AddressTests.cs` (no fixture — plain `[Fact]` only):
    - `SetAsDefault_ClearsPreviousDefaultFlag`: `SetAsDefault()` removes default from other addresses
    - `Create_WithPhoneNumberExceedingMaxLength_ThrowsDomainException`: phone > 20 chars rejected
    - `Create_WithStreetExceedingMaxLength_ThrowsDomainException`: street > 200 chars rejected
  - Application/ folder mirrors source Application/ layer; one file per handler in sub-feature sub-folders:
  - `Application/Profile/CreateProfileCommandHandlerTests.cs` (uses `UserServiceFixture`, `BlobStorageFake`, `FakeCurrentUser`):
    - `Handle_WithValidFields_PersistsNameEmailReferenceAndAvatarReference`
  - `Application/Profile/UpdateProfileCommandHandlerTests.cs` (uses `UserServiceFixture`, `BlobStorageFake`, `FakeCurrentUser`):
    - `Handle_WithNewDisplayName_ChangesStoredField`
    - `Handle_AvatarReference_StoredWithoutLiveBlobCall`
  - `Application/Settings/UpdateNotificationPreferenceCommandHandlerTests.cs` (uses `UserServiceFixture`):
    - `Handle_PersistsChannelPreference`
  - `Application/Settings/GetSettingsQueryHandlerTests.cs` (uses `UserServiceFixture`):
    - `Handle_ReturnsCurrentValues`
  - `Application/Addresses/AddAddressCommandHandlerTests.cs` (uses `UserServiceFixture`):
    - `Handle_NewAddressAppearsInAddressList`
  - `Application/Addresses/SetDefaultAddressCommandHandlerTests.cs` (uses `UserServiceFixture`):
    - `Handle_UpdatesDefaultFlag_ClearsOldDefault`
  - `Application/Addresses/RemoveAddressCommandHandlerTests.cs` (uses `UserServiceFixture`):
    - `Handle_RemovedAddressNoLongerInList`
  - `Application/StoreOnboarding/SubmitStoreRegistrationCommandHandlerTests.cs` (uses `UserServiceFixture`, `FakeCurrentUser`):
    - `Handle_CreatesPendingStoreProfile`
  - `Application/StoreOnboarding/ApproveStoreCommandHandlerTests.cs` (uses `UserServiceFixture`, `FakeCurrentUser`):
    - `Handle_AssignsSellerRoleToOwner`
  - Use `DeterministicClock` in fixtures
  - Acceptance: `dotnet test tests/HiveSpace.UserService.Tests` exits with code 0

---

## CatalogService Tests

### Create

- [ ] B007 [US2] Create `HiveSpace.CatalogService.Tests` project
  - File: `../hivespace.microservice/tests/HiveSpace.CatalogService.Tests/HiveSpace.CatalogService.Tests.csproj`
  - Reference: `HiveSpace.Testing.Shared`; no `Version=`
  - Folder layout:
    - `Domain/` — value-object unit tests; no DB, no fakes. Plain `[Fact]`.
    - `Application/` — handler-level integration tests; each class uses `IClassFixture<CatalogServiceFixture>`
    - `Consumers/` — projection consumer tests using `InMemoryTestHarness` (not `IClassFixture`)
    - `Fixtures/CatalogServiceFixture.cs` — creates in-memory `CatalogDbContext` with `Guid.NewGuid()` database name in `InitializeAsync`
  - `Domain/DimensionsTests.cs` (no fixture — plain `[Fact]` only):
    - `Create_WithNegativeValue_ThrowsDomainException`: non-positive dimension rejected
    - `TwoDimensionsWithSameValuesAndUnit_AreEqual`: value object equality
  - `Domain/WeightTests.cs` (no fixture — plain `[Fact]` only):
    - `Create_WithNegativeValue_ThrowsDomainException`: non-positive weight rejected
    - `TwoWeightsWithSameValueAndUnit_AreEqual`: value object equality
  - Domain/ tests for `Product`, `Category`, and `Sku` are in task **B015** — write them there, not here. Write a Domain/ test for any aggregate that has guard invariants, factory validation, or state transitions (e.g. `Product.Deactivate`, `Product.CreateProduct` throwing on negative price).
  - Remove the existing `Application/Products/CatalogApplicationSmokeTests.cs` file if present — per TESTING.md, multi-handler smoke tests do not count as Application coverage and must be replaced by per-handler test files.
  - Application/ folder mirrors source Application/ layer; one file per handler in sub-feature sub-folders:
  - `Application/StorefrontDiscovery/SearchProductsQueryHandlerTests.cs` (uses `CatalogServiceFixture`):
    - `Handle_ReturnsOnlyActiveProducts`
    - `Handle_WithCategoryFilter_ScopesResults`
  - `Application/StorefrontDiscovery/GetProductDetailQueryHandlerTests.cs` (uses `CatalogServiceFixture`):
    - `Handle_ReturnsTitle_Price_AndVariants`
  - `Application/SellerProducts/CreateProductCommandHandlerTests.cs` (uses `CatalogServiceFixture`, `FakeCurrentUser`):
    - `Handle_WithValidInputs_PersistsRequiredFields`
  - `Application/SellerProducts/UpdateProductCommandHandlerTests.cs` (uses `CatalogServiceFixture`, `FakeCurrentUser`):
    - `Handle_ChangesStoredTitleAndPrice`
    - `Handle_ByUnauthorizedSeller_IsRejected`
  - `Application/SellerProducts/DeactivateProductCommandHandlerTests.cs` (uses `CatalogServiceFixture`, `FakeCurrentUser`):
    - `Handle_ExcludesProductFromStorefrontSearchResults`
  - `Application/Categories/ListCategoriesQueryHandlerTests.cs` (uses `CatalogServiceFixture`):
    - `Handle_WithSeededCategories_ReturnsAllSeededCategories`: seed N categories, verify all N returned
    - `Handle_WithNoCategories_ReturnsEmptyList`
  - `Application/Categories/GetCategoryAttributesQueryHandlerTests.cs` (uses `CatalogServiceFixture`):
    - `Handle_WithValidCategoryId_ReturnsAttributeDefinitions`
    - `Handle_WithInvalidCategoryId_ReturnsNotFound`
  - `Consumers/ProjectionInputTests.cs` (uses `InMemoryTestHarness`; not `IClassFixture`):
    - `ProductDomainEvent_ProducesCorrectProjectionInput`: event published to harness bus has expected product fields captured by `InMemoryMessageCapture`
    - `ProjectionConsumer_ReadsCorrectFields`: consumer processes captured event without error
  - Use `DeterministicClock` in fixture
  - Acceptance: `dotnet test tests/HiveSpace.CatalogService.Tests` exits with code 0

- [ ] B014 Add missing CatalogService application handler tests
  - All files go in: `../hivespace.microservice/tests/HiveSpace.CatalogService.Tests/`
  - All application handler tests use `IClassFixture<CatalogServiceFixture>`; minimum 2 `[Fact]` per file (happy path + failure/edge case); naming `Handle_<Condition>_<ExpectedOutcome>`
  - `Application/Categories/GetHomepageCategoriesQueryHandlerTests.cs`:
    - `Handle_WithSeededCategories_ReturnsFeaturedCategorySubset`
    - `Handle_WhenNoCategoriesExist_ReturnsEmptyList`
  - `Application/SellerProducts/DeleteProductCommandHandlerTests.cs`:
    - `Handle_WithExistingProduct_RemovesProductFromDb`
    - `Handle_ByUnauthorizedSeller_IsRejected`
    - `Handle_WithNonExistentProductId_ReturnsNotFound`
  - `Application/SellerProducts/GetProductQueryHandlerTests.cs`:
    - `Handle_WithValidProductId_ReturnsProductDetails`
    - `Handle_WithNonExistentId_ReturnsNotFound`
  - `Application/StorefrontDiscovery/GetProductsQueryHandlerTests.cs`:
    - `Handle_WithNoFilters_ReturnsPagedActiveProducts`
    - `Handle_WithCategoryFilter_ScopesToCategory`
    - `Handle_WithInactiveProducts_ExcludesFromResults`
  - `Application/StorefrontDiscovery/GetProductSummariesQueryHandlerTests.cs`:
    - `Handle_WithProductIds_ReturnsMatchingSnapshots`
    - `Handle_WithMissingProductIds_SkipsMissing`
  - Acceptance: `dotnet test tests/HiveSpace.CatalogService.Tests` exits with code 0

- [ ] B015 Add CatalogService Product aggregate and specification domain tests
  - All files go in: `../hivespace.microservice/tests/HiveSpace.CatalogService.Tests/Domain/`
  - All domain tests are plain `[Fact]` — no fixture, no DbContext, no fakes
  - `Domain/ProductTests.cs`:
    - `CreateProduct_WithValidFields_SetsNameStatusAndSellerId`
    - `CreateProduct_WithNameExceedingMaxLength_ThrowsDomainException`
    - `CreateProduct_WithNegativePrice_ThrowsDomainException`
    - `UpdateName_WithValidName_ChangesStoredName`
    - `UpdateName_WithEmptyName_ThrowsDomainException`
    - `AddCategory_WithNewCategory_AppearsInCategoryCollection`
    - `RemoveCategory_WithExistingCategory_RemovesFromCollection`
    - `UpdateCategories_ReplacesEntireCategorySet`
    - `Deactivate_ActiveProduct_SetsStatusToInactive`
    - `Deactivate_AlreadyInactiveProduct_IsIdempotent`
  - `Domain/ProductSpecificationTests.cs`:
    - `ProductActiveSpecification_ActiveProduct_ReturnsTrue`
    - `ProductActiveSpecification_InactiveProduct_ReturnsFalse`
    - `ProductOwnedBySellerSpecification_MatchingSellerId_ReturnsTrue`
    - `ProductOwnedBySellerSpecification_DifferentSellerId_ReturnsFalse`
  - Acceptance: domain tests run under 50ms each with no external I/O

---

## OrderService Tests

### Create

- [ ] B008 [US2] Create `HiveSpace.OrderService.Tests` project
  - File: `../hivespace.microservice/tests/HiveSpace.OrderService.Tests/HiveSpace.OrderService.Tests.csproj`
  - Reference: `HiveSpace.Testing.Shared`; no `Version=`
  - Folder layout (most complex service — all four folders present):
    - `Domain/` — pure entity unit tests; no fixtures, no DB, no fakes
    - `Application/` — handler-level integration tests; each class uses `IClassFixture<OrderServiceFixture>`
    - `Consumers/` — saga state machine tests using `InMemoryTestHarness` (not `IClassFixture`)
    - `Fixtures/OrderServiceFixture.cs` — creates in-memory `OrderDbContext` with `Guid.NewGuid()` database name in `InitializeAsync`
  - Remove the existing `Application/Cart/OrderApplicationSmokeTests.cs` file if present — per TESTING.md, smoke tests do not count as Application coverage and must be replaced by per-handler test files.
  - `Domain/CartTests.cs` (no fixture — plain `[Fact]` only):
    - `AddItem_UpdatesCartCount`: item added; cart item count increases
    - `UpdateQuantity_ChangesLineTotal`: quantity change reflected in cart
    - `RemoveItem_LeavesRemainingItems`: removed item absent; others intact
    - `AddSameProductTwice_MergesIntoOneLine`: duplicate add merges quantity onto existing line
    - `InitiateCheckoutOnEmptyCart_ThrowsDomainException`: domain invariant enforced without I/O
  - `Domain/CouponTests.cs` (no fixture — plain `[Fact]` only):
    - `ValidCoupon_ReducesOrderTotal`: applied coupon reduces total by expected amount (pure calculation)
    - `ExpiredCoupon_IsRejected`: coupon past `EndDateTime` throws domain exception
    - `AlreadyUsedCoupon_IsRejected`: already-consumed coupon throws domain exception
    - `Validate_BeforeStartDate_IsRejected`: coupon before `StartDateTime` rejected
    - `Validate_WhenBelowMinOrderAmount_IsRejected`: order total below minimum threshold rejected
    - `Validate_WhenMaxUsageReached_IsRejected`: coupon at `MaxUsageCount` rejects further use
    - `Validate_WhenUserExceedsMaxUsagePerUser_IsRejected`: same user exceeding per-user limit rejected
    - `CalculateDiscount_FixedType_ReturnsFixedAmount`: fixed discount type returns exact amount
    - `CalculateDiscount_PercentageType_CappedAtMaxDiscountAmount`: percentage discount capped at `MaxDiscountAmount`
    - `MarkAsUsed_IncrementsCurrentUsageCount`: usage tracking increments correctly
    - `ReleaseUsage_DecrementsCurrentUsageCount`: released coupon decrements usage count
  - `Domain/OrderTests.cs` (no fixture — plain `[Fact]` only):
    - `Create_WithValidFields_StartsInCreatedStatus`
    - `AddItem_InCreatedStatus_AddsToItemCollection`
    - `AddItem_InConfirmedStatus_ThrowsDomainException`: status precondition enforced
    - `MarkAsPaid_FromCreatedStatus_TransitionsToPaid`
    - `MarkAsPaid_FromInvalidStatus_ThrowsDomainException`
    - `Confirm_WithNoItems_ThrowsDomainException`
    - `Cancel_AfterDelivered_ThrowsDomainException`: terminal state cannot be cancelled
    - `RecalculateTotals_ReflectsItemSumMinusDiscount`: financial calculation matches expected value
    - `CalculateSellerPayout_Applies9Point9PercentServiceFee`: payout deducts platform fee
  - `Domain/OrderStatusTests.cs` (no fixture — plain `[Fact]` only):
    - `Created_IsFinal_ReturnsFalse`
    - `Completed_IsFinal_ReturnsTrue`
    - `Cancelled_IsFinal_ReturnsTrue`
    - `Created_CanBeCancelled_ReturnsTrue`
    - `Shipped_CanBeCancelled_ReturnsFalse`
    - `Confirmed_CanBeShipped_ReturnsTrue`
    - `Cancelled_CanBeConfirmed_ReturnsFalse`
    - `Confirmed_CanBeRejected_ReturnsFalse`: confirmed order cannot be rejected
  - Application/ folder mirrors source Application/ layer; one file per handler in sub-feature sub-folders:
  - `Application/Cart/AddItemToCartCommandHandlerTests.cs` (uses `OrderServiceFixture`, `FakeCurrentUser`):
    - `Handle_WithValidProduct_AddsLineItem`
    - `Handle_WithSameProductTwice_MergesQuantity`
  - `Application/Cart/UpdateCartItemQuantityCommandHandlerTests.cs` (uses `OrderServiceFixture`, `FakeCurrentUser`):
    - `Handle_ChangesLineTotalForAffectedItem`
  - `Application/Cart/RemoveCartItemCommandHandlerTests.cs` (uses `OrderServiceFixture`, `FakeCurrentUser`):
    - `Handle_RemovedItemAbsentFromCart`
  - `Application/Checkout/InitiateCheckoutCommandHandlerTests.cs` (uses `OrderServiceFixture`, `FakeCurrentUser`, `InMemoryMessageCapture`):
    - `Handle_WithValidCart_CreatesPendingOrderAndClearsCart`
    - `Handle_WithEmptyCart_ThrowsDomainException`
  - `Application/BuyerOrders/GetBuyerOrdersQueryHandlerTests.cs` (uses `OrderServiceFixture`, `FakeCurrentUser`):
    - `Handle_ReturnsOnlyAuthenticatedBuyerOrders`
    - `Handle_WithOtherBuyerUserId_ReturnsEmpty`
  - `Application/BuyerOrders/GetOrderDetailQueryHandlerTests.cs` (uses `OrderServiceFixture`, `FakeCurrentUser`):
    - `Handle_ReturnsLineItemsAndCurrentStatus`
  - `Application/SellerOrders/GetSellerOrdersQueryHandlerTests.cs` (uses `OrderServiceFixture`, `FakeCurrentUser`):
    - `Handle_ScopesToSellerStoreId`
  - `Application/SellerOrders/ConfirmOrderCommandHandlerTests.cs` (uses `OrderServiceFixture`, `FakeCurrentUser`):
    - `Handle_TransitionsOrderStatusToConfirmed`
  - `Application/SellerOrders/RejectOrderCommandHandlerTests.cs` (uses `OrderServiceFixture`, `FakeCurrentUser`):
    - `Handle_StoresRejectionReasonAndTransitionsStatus`
  - `Consumers/CheckoutSagaTests.cs` (uses `InMemoryTestHarness`):
    - `SagaStartsOnCheckoutInitiated`: `CheckoutInitiated` event starts saga state
    - `PaymentConfirmed_TransitionsSagaToCompleted`: payment confirmation event advances saga; order status updated
    - `PaymentFailed_TransitionsSagaToFailed`: payment failure event cancels or sets failed state
    - `SagaCompletion_CreatesOrder`: completed saga produces confirmed order
  - `Consumers/FulfillmentSagaTests.cs` (uses `InMemoryTestHarness`):
    - `SagaStartsOnOrderPlaced`: `OrderPlaced` event starts fulfillment saga
    - `FulfillmentConfirmed_TransitionsState`: confirmation event advances state
    - `ShipmentDispatched_EmitsEvent`: shipment dispatched event captured in harness
    - `SagaCompletion_MarksOrderFulfilled`: completed fulfillment saga marks order as fulfilled
  - Use `DeterministicClock` in fixture; `PaymentProviderFake` in saga tests
  - Acceptance: `dotnet test tests/HiveSpace.OrderService.Tests` exits with code 0; `Domain/` tests run without any fixture or DB dependency

- [ ] B016 Add OrderService Coupon application handler tests (7 handlers)
  - All files go in: `../hivespace.microservice/tests/HiveSpace.OrderService.Tests/Application/Coupons/`
  - All tests use `IClassFixture<OrderServiceFixture>` and `FakeCurrentUser`
  - `CreateCouponCommandHandlerTests.cs`:
    - `Handle_WithValidInputs_PersistsCouponWithCorrectFieldsAndActiveStatus`
    - `Handle_WithEndDateBeforeStartDate_ThrowsValidationError`
    - `Handle_WithZeroMaxUsageCount_ThrowsValidationError`
  - `UpdateCouponCommandHandlerTests.cs`:
    - `Handle_WithValidFields_UpdatesStoredCoupon`
    - `Handle_OnExpiredCoupon_IsRejected`
  - `DeleteCouponCommandHandlerTests.cs`:
    - `Handle_WithExistingCoupon_RemovesCouponFromDb`
    - `Handle_WithNonExistentId_ReturnsNotFound`
  - `EndCouponCommandHandlerTests.cs`:
    - `Handle_WithActiveCoupon_SetsEndDateToNow`
    - `Handle_OnAlreadyEndedCoupon_IsIdempotent`
  - `GetAvailableCouponsQueryHandlerTests.cs`:
    - `Handle_ReturnsOnlyActiveCouponsAboveMinOrderAmount`
    - `Handle_WithNoEligibleCoupons_ReturnsEmptyList`
  - `GetCouponByIdQueryHandlerTests.cs`:
    - `Handle_WithValidId_ReturnsCouponDetails`
    - `Handle_WithNonExistentId_ReturnsNotFound`
  - `GetCouponListQueryHandlerTests.cs`:
    - `Handle_ReturnsPaginatedSellerCoupons`
    - `Handle_WithStatusFilter_ScopesResults`
  - Acceptance: `dotnet test tests/HiveSpace.OrderService.Tests` exits with code 0

- [ ] B017 Add OrderService Cart missing application handler tests (6 handlers)
  - All files go in: `../hivespace.microservice/tests/HiveSpace.OrderService.Tests/Application/Cart/`
  - All tests use `IClassFixture<OrderServiceFixture>` and `FakeCurrentUser`
  - `ApplyPlatformCouponCommandHandlerTests.cs`:
    - `Handle_WithValidCoupon_AppliesDiscountToCart`
    - `Handle_WithExpiredCoupon_IsRejected`
    - `Handle_WhenCouponAlreadyApplied_ReplacesWithNewCoupon`
  - `ApplyStoreCouponCommandHandlerTests.cs`:
    - `Handle_WithValidStoreCoupon_AppliesStoreDiscount`
    - `Handle_WithCouponNotBelongingToStore_IsRejected`
  - `RemovePlatformCouponCommandHandlerTests.cs`:
    - `Handle_RemovesPlatformCouponFromCart`
    - `Handle_WhenNoCouponApplied_IsIdempotent`
  - `RemoveStoreCouponCommandHandlerTests.cs`:
    - `Handle_RemovesStoreCouponFromCart`
    - `Handle_WhenNoCouponApplied_IsIdempotent`
  - `GetCartSummaryQueryHandlerTests.cs`:
    - `Handle_ReturnsItemCountSubtotalAndAppliedCoupons`
    - `Handle_WithEmptyCart_ReturnsZeroItems`
  - `GetSelectedCartItemsCountQueryHandlerTests.cs`:
    - `Handle_ReturnsCountOfSelectedItems`
    - `Handle_WithNoSelectedItems_ReturnsZero`
  - `GetCheckoutPreviewQueryHandlerTests.cs` (file goes in `Application/Checkout/`):
    - `Handle_WithValidCart_ReturnsLineItemsShippingAndTotal`
    - `Handle_WithAppliedCoupons_ReflectsDiscountsInPreview`
  - Acceptance: `dotnet test tests/HiveSpace.OrderService.Tests` exits with code 0

- [ ] B018 Add OrderService Order additional application handler tests (4 handlers)
  - All files go in: `../hivespace.microservice/tests/HiveSpace.OrderService.Tests/Application/BuyerOrders/`
  - All tests use `IClassFixture<OrderServiceFixture>` and `FakeCurrentUser`
  - `CancelOrderCommandHandlerTests.cs`:
    - `Handle_WithCancellableOrder_TransitionsStatusToCancelled`
    - `Handle_AfterOrderDelivered_IsRejected`
    - `Handle_ByDifferentBuyer_IsRejected`
  - `GetCheckoutStatusQueryHandlerTests.cs`:
    - `Handle_WithPendingOrder_ReturnsPendingStatus`
    - `Handle_WithCompletedOrder_ReturnsCompletedStatus`
    - `Handle_WithNonExistentId_ReturnsNotFound`
  - `GetOrderByIdQueryHandlerTests.cs`:
    - `Handle_WithValidId_ReturnsOrderWithLineItemsAndStatus`
    - `Handle_WithOtherBuyerOrderId_IsRejected`
  - `GetOrderListQueryHandlerTests.cs`:
    - `Handle_ReturnsOnlyAuthenticatedBuyerOrders`
    - `Handle_WithStatusFilter_ScopesResults`
    - `Handle_PaginationRespectsPageSizeAndPageNumber`
  - Acceptance: `dotnet test tests/HiveSpace.OrderService.Tests` exits with code 0

- [ ] B019 Add OrderService Checkout domain tests and CheckoutCalculator helper tests
  - `Domain/CheckoutTests.cs` (no fixture — plain `[Fact]` only):
    - `Create_WithValidCartItems_InitializesLineItemsAndCalculatesTotal`
    - `AddItem_UpdatesSubtotal`
    - `ApplyCoupon_ReducesTotalByDiscountAmount`
    - `ApplyCoupon_WhenCouponAlreadyApplied_ReplacesExisting`
    - `RemoveCoupon_RestoresSubtotalWithoutDiscount`
  - `Domain/DiscountTests.cs` (no fixture — plain `[Fact]` only):
    - `CreateFixed_WithPositiveAmount_StoresAmount`
    - `CreateFixed_WithNegativeAmount_ThrowsDomainException`
    - `CreatePercentage_WithValueAbove100_ThrowsDomainException`
    - `Apply_FixedType_ReturnsExactAmount`
    - `Apply_PercentageType_CapsAtMaxDiscountAmount`
  - `Application/Checkout/CheckoutCalculatorTests.cs` (no fixture — static helper, plain `[Fact]` only):
    - `CalculateSubtotal_SumsAllLineItemTotals`
    - `CalculateShippingFee_ReturnsExpectedFeeForWeight`
    - `CalculatePlatformDiscount_WithFixedCoupon_DeductsFixed`
    - `CalculatePlatformDiscount_WithPercentageCoupon_DeductsPercentageCapped`
    - `CalculateTotal_SubtotalPlusShippingMinusDiscounts`
  - Acceptance: all tests run without fixture or I/O; calculation results match expected values

---

## PaymentService Tests

### Update

- [ ] B009 [US2] Update `HiveSpace.PaymentService.Tests` — extend with idempotency and state tests
  - File: `../hivespace.microservice/tests/HiveSpace.PaymentService.Tests/` (existing directory; add folder structure and new test class files)
  - Add folder layout if not already present in existing project:
    - `Domain/` — pure payment state machine unit tests; no DB, no fakes
    - `Application/` — handler-level integration tests; each class uses `IClassFixture<PaymentServiceFixture>`
    - `Fixtures/PaymentServiceFixture.cs` — create if absent; creates in-memory `PaymentDbContext` with `Guid.NewGuid()` database name
  - Remove the existing `Application/Payments/PaymentApplicationSmokeTests.cs` file if present — per TESTING.md, smoke tests do not count as Application coverage and must be replaced by per-handler test files.
  - Move any existing test classes into `Application/` if they currently sit at the root of the project
  - `Domain/PaymentResultStateTests.cs` (no fixture — plain `[Fact]` only):
    - `Payment_Transitions_PendingToProcessing`: state machine accepts pending → processing
    - `Payment_Transitions_ProcessingToSuccess`: state machine accepts processing → success
    - `Payment_Transitions_ProcessingToFailed`: state machine accepts processing → failed
    - `Payment_InvalidTransition_IsRejected`: disallowed transition throws domain exception
    - `MarkAsProcessing_WhenNotPending_ThrowsDomainException`: precondition enforced
    - `MarkAsSucceeded_WhenNotProcessing_ThrowsDomainException`: precondition enforced
    - `Cancel_AfterSucceeded_ThrowsDomainException`: terminal state protected
  - `Domain/WalletTests.cs` (no fixture — plain `[Fact]` only):
    - `CreateForUser_InitializesWithZeroBalance`
    - `Credit_WithPositiveAmount_IncreasesAvailableBalance`
    - `Credit_WithZeroAmount_ThrowsDomainException`
    - `Debit_WithSufficientBalance_DecreasesAvailableBalance`
    - `Debit_WithInsufficientBalance_ThrowsDomainException`
    - `Debit_OnSuspendedWallet_ThrowsDomainException`
    - `Suspend_ActiveWallet_SetsStatusToSuspended`
    - `Activate_SuspendedWallet_SetsStatusToActive`
  - Application/ folder mirrors source Application/ layer; one file per handler in sub-feature sub-folders:
  - `Application/VNPayReturn/ProcessVNPayReturnCommandHandlerTests.cs` (uses `PaymentServiceFixture`, `PaymentProviderFake`):
    - `Handle_WithFirstReturn_CreatesPaymentRecord`
    - `Handle_WithDuplicateTransactionRef_IsIdempotent`
  - `Application/VNPayWebhook/ProcessVNPayWebhookCommandHandlerTests.cs` (uses `PaymentServiceFixture`, `PaymentProviderFake`):
    - `Handle_WithValidSignature_ProcessesWebhook`
    - `Handle_WithTamperedSignature_IsRejected`
    - `Handle_WithDuplicatePaymentRef_IsIdempotent`
  - `Application/Wallet/GetWalletBalanceQueryHandlerTests.cs` (uses `PaymentServiceFixture`):
    - `Handle_ReturnsCurrentBalance`
    - `Handle_SequentialReads_DoNotMutateState`
  - Migrate any existing tests referencing live VNPay HTTP calls to use `PaymentProviderFake`
  - Do not add `Version=` to any `<PackageReference>` in existing or new `.csproj`
  - Acceptance: `dotnet test tests/HiveSpace.PaymentService.Tests` exits with code 0; `Domain/` tests run without any fixture; idempotency assertions confirm no duplicate processing on second call

- [ ] B020 Add PaymentService missing handler and Payment domain tests
  - `Application/Wallet/GetTransactionHistoryQueryHandlerTests.cs` (uses `PaymentServiceFixture`):
    - `Handle_ReturnsTransactionsForAuthenticatedUser`
    - `Handle_WithOtherUserId_ReturnsEmpty`
    - `Handle_WithDateRangeFilter_ScopesResults`
  - `Domain/PaymentTests.cs` (no fixture — plain `[Fact]` only):
    - `Create_WithValidTransactionRef_StartsInPendingStatus`
    - `MarkAsProcessing_FromPendingStatus_TransitionsCorrectly`
    - `MarkAsSucceeded_FromProcessingStatus_TransitionsCorrectly`
    - `MarkAsFailed_FromProcessingStatus_TransitionsCorrectly`
    - `MarkAsProcessing_WhenAlreadyProcessing_ThrowsDomainException`
    - `MarkAsSucceeded_WhenNotProcessing_ThrowsDomainException`
    - `MarkAsFailed_AfterSucceeded_ThrowsDomainException`
    - `Create_WithDuplicateTransactionRef_IsRejectedByDomainRule`
  - Acceptance: `dotnet test tests/HiveSpace.PaymentService.Tests` exits with code 0; domain tests run with no fixture

---

## MediaService Tests

### Create

- [ ] B010 [US2] Create `HiveSpace.MediaService.Tests` project
  - File: `../hivespace.microservice/tests/HiveSpace.MediaService.Tests/HiveSpace.MediaService.Tests.csproj`
  - Reference: `HiveSpace.Testing.Shared`; no `Version=`
  - Folder layout:
    - `Domain/` — unit tests for media entity state transitions and invariants; no DB, no fakes
    - `Application/` — handler-level integration tests; each class uses `IClassFixture<MediaServiceFixture>`
    - `Fixtures/MediaServiceFixture.cs` — creates in-memory `MediaDbContext` (or relevant context) with `Guid.NewGuid()` database name
  - `Domain/MediaAssetTests.cs` (no fixture — plain `[Fact]` only; actual aggregate class is `MediaAsset` not `MediaUploadReference`):
    - `Create_WithValidFields_StartsInPendingStatus`: freshly created asset has pending status
    - `MarkAsUploaded_FromPendingStatus_TransitionsToUploaded`: upload confirmation changes status
    - `MarkAsUploaded_FromNonPendingStatus_ThrowsDomainException`: precondition enforced
    - `MarkAsProcessed_FromUploadedStatus_TransitionsToProcessed`: processing sets public URL
    - `MarkAsFailed_SetsStatusToFailed`: failure transition from any non-terminal status
    - `SetEntityId_OnProcessedAsset_ThrowsDomainException`: entity ID only settable in Pending status
  - Application/ folder mirrors source Application/ layer; one file per handler in sub-feature sub-folders:
  - `Application/Presign/GeneratePresignUrlCommandHandlerTests.cs` (uses `MediaServiceFixture`, `BlobStorageFake`):
    - `Handle_WithValidMediaType_ReturnsPresignUrlAndUploadRef`
    - `Handle_WithInvalidMediaType_ReturnsValidationError`
  - `Application/UploadConfirmation/ConfirmUploadCommandHandlerTests.cs` (uses `MediaServiceFixture`, `BlobStorageFake`):
    - `Handle_ForValidRef_MarksConfirmedInBlobStorageFake`
    - `Handle_ForAlreadyConfirmedRef_IsIdempotent`
    - `Handle_ForExpiredRef_IsRejected`
  - Acceptance: `dotnet test tests/HiveSpace.MediaService.Tests` exits with code 0; `Domain/` tests run without any fixture; no live blob storage calls

---

## NotificationService Tests

### Create

- [ ] B011 [US2] Create `HiveSpace.NotificationService.Tests` project
  - File: `../hivespace.microservice/tests/HiveSpace.NotificationService.Tests/HiveSpace.NotificationService.Tests.csproj`
  - Reference: `HiveSpace.Testing.Shared`; no `Version=`
  - Folder layout:
    - `Domain/` — unit tests for notification entity state transitions and channel preference invariants; no DB, no fakes
    - `Application/` — handler-level integration tests; each class uses `IClassFixture<NotificationServiceFixture>`
    - `Fixtures/NotificationServiceFixture.cs` — creates in-memory `NotificationDbContext` with `Guid.NewGuid()` database name; initializes `EmailDeliveryFake` and `SignalRHubFake`
  - `Domain/NotificationStatusTests.cs` (no fixture — plain `[Fact]` only):
    - `NewNotification_StartsInPendingDeliveryStatus`: freshly created notification has pending status
    - `MarkDelivered_TransitionsToDelivered`: delivery confirmation changes status to delivered
    - `MarkDelivered_WhenAlreadyDelivered_IsIdempotent`: re-delivering an already-delivered notification does not change state or throw
    - `MarkFailed_TransitionsToFailed`: failure marks notification with failed delivery status
  - `Domain/NotificationChannelPreferenceTests.cs` (no fixture — plain `[Fact]` only):
    - `EnableChannel_SetsChannelActive`: enabling a channel sets its active flag
    - `DisableChannel_SetsChannelInactive`: disabling a channel clears its active flag
    - `UnsupportedChannelType_ThrowsDomainException`: unsupported channel type rejected by domain rule
  - Application/ folder mirrors source Application/ layer; one file per handler in sub-feature sub-folders:
  - `Application/Preferences/EnableNotificationChannelCommandHandlerTests.cs` (uses `NotificationServiceFixture`, `FakeCurrentUser`):
    - `Handle_PersistsChannelAsEnabled`
  - `Application/Preferences/DisableNotificationChannelCommandHandlerTests.cs` (uses `NotificationServiceFixture`, `FakeCurrentUser`):
    - `Handle_PersistsChannelAsDisabled`
  - `Application/Preferences/GetNotificationPreferencesQueryHandlerTests.cs` (uses `NotificationServiceFixture`, `FakeCurrentUser`):
    - `Handle_ReturnsStoredPreferences`
    - `Handle_WithUnsupportedChannelType_ReturnsValidationError`
  - `Application/Notifications/CreateNotificationCommandHandlerTests.cs` (uses `NotificationServiceFixture`, `FakeCurrentUser`):
    - `Handle_StoresNotificationWithPendingDeliveryStatus`
  - `Application/Notifications/GetNotificationsQueryHandlerTests.cs` (uses `NotificationServiceFixture`, `FakeCurrentUser`):
    - `Handle_ReturnsNotificationsForAuthenticatedUser`
    - `Handle_WithOtherUserId_ReturnsEmpty`
  - `Application/Delivery/MarkNotificationDeliveredCommandHandlerTests.cs` (uses `NotificationServiceFixture`, `FakeCurrentUser`):
    - `Handle_UpdatesDeliveryStatusToDelivered`
    - `Handle_ForOtherUserNotification_IsRejected`
  - `Application/RealtimePayload/SendOrderUpdateNotificationCommandHandlerTests.cs` (uses `NotificationServiceFixture`, `SignalRHubFake`):
    - `Handle_EmitsSignalRInvocation_ReceiveOrderUpdate_WithOrderIdAndStatus`
  - `Application/RealtimePayload/SendPaymentResultNotificationCommandHandlerTests.cs` (uses `NotificationServiceFixture`, `SignalRHubFake`):
    - `Handle_EmitsSignalRInvocation_ReceivePaymentResult_WithPaymentRefAndStatus`
  - `Application/RealtimePayload/SendGeneralNotificationCommandHandlerTests.cs` (uses `NotificationServiceFixture`, `SignalRHubFake`):
    - `Handle_EmitsSignalRInvocation_ReceiveNotification_WithIdMessageAndType`
  - Use `EmailDeliveryFake` (no live emails), `DeterministicClock`
  - Acceptance: `dotnet test tests/HiveSpace.NotificationService.Tests` exits with code 0

---

## Solution Integration

### Update

- [ ] B012 [US1] Update solution file to include all new test projects
  - File: `../hivespace.microservice/hivespace.microservice.sln` (or the root `.sln` file)
  - Add solution folder `tests` if it does not already exist
  - Add project entries: `tests/HiveSpace.Testing.Shared/HiveSpace.Testing.Shared.csproj`, `tests/HiveSpace.IdentityService.Tests/HiveSpace.IdentityService.Tests.csproj`, `tests/HiveSpace.UserService.Tests/HiveSpace.UserService.Tests.csproj`, `tests/HiveSpace.CatalogService.Tests/HiveSpace.CatalogService.Tests.csproj`, `tests/HiveSpace.OrderService.Tests/HiveSpace.OrderService.Tests.csproj`, `tests/HiveSpace.MediaService.Tests/HiveSpace.MediaService.Tests.csproj`, `tests/HiveSpace.NotificationService.Tests/HiveSpace.NotificationService.Tests.csproj`
  - Do not remove `HiveSpace.PaymentService.Tests` (already present in solution)
  - Acceptance: `dotnet build` from repo root includes all test projects; `dotnet test` discovers all new and existing test projects

---

## TDD Guide

### Create

- [ ] B013 Verify and update `TESTING.md` developer guide in `../hivespace.microservice`
  - File: `../hivespace.microservice/TESTING.md` — **already exists**; do not recreate from scratch
  - Read the existing file and verify the following sections are present and accurate:
  - **Test pyramid**: `Domain/` pure unit, `Application/` in-memory EF integration — already documented
  - **Domain/ decision rule**: aggregate state transitions, guard invariants, value object equality, domain service orchestration — already documented
  - **Two Application test patterns**: the existing worked example uses NSubstitute (`Substitute.For<IOrderRepository>()`); ensure the guide clearly documents when to use NSubstitute (command handler, mock-verify is sufficient) vs `IClassFixture` (query handler or complex insert-then-read). Add a note if the NSubstitute pattern is not yet distinguished from the IClassFixture pattern.
  - **Smoke-test exclusion rule**: add if absent — "Do not count these as Application-layer coverage: `typeof(Handler).Should().NotBeNull()`, smoke tests that invoke multiple handlers, direct aggregate mutation in an Application test when the handler is the real unit under test."
  - **Coverage scope reference**: add if absent — "Coverage is measured by `coverage.runsettings`: includes `[HiveSpace.*.Application]*`, `[HiveSpace.*.Domain]*`, `[HiveSpace.*.Core]*`; excludes Api, Infrastructure, and test projects."
  - Do not rewrite sections that are already correct; only add what is missing
  - Acceptance: file covers both NSubstitute and IClassFixture patterns; smoke-test exclusion rule is present; coverage scope is referenced

