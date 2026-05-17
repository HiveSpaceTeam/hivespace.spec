# Source Tree — HiveSpace Microservices

Annotated directory tree of `hivespace.microservice/`. Excludes `bin/`, `obj/`, `.git/`, `.codex*/`, `.gitnexus/`, `.agent-source/`.

---

## Root

```
hivespace.microservice/
├── src/                          # All deployable service projects
├── libs/                         # Shared library projects
├── infra/                        # Azure Bicep deployment templates
├── scripts/                      # Developer utility scripts
├── templates/                    # dotnet new service templates
├── docs/                         # Existing documentation (agent/, workflows/)
├── .github/workflows/            # GitHub Actions CI/CD pipelines
├── hivespace.microservice.sln    # Solution file (all projects registered)
├── Directory.Packages.props      # Centralized NuGet version management
└── global.json                   # .NET SDK version pin
```

---

## `src/` — Services

### ApiGateway

```
src/HiveSpace.ApiGateway/
└── HiveSpace.YarpApiGateway/             # ENTRY POINT: port 5000
    ├── Program.cs                        # YARP + CORS + WebSocket setup
    └── appsettings.json                  # Route table + cluster config (all service ports)
```

---

### UserService

```
src/HiveSpace.UserService/
├── HiveSpace.UserService.Domain/         # Domain layer
│   ├── Aggregates/
│   │   ├── User/
│   │   │   ├── User.cs                   # AGGREGATE: User (Identity, profile, addresses, settings)
│   │   │   ├── Address.cs                # Entity owned by User
│   │   │   ├── Email.cs                  # Value object
│   │   │   ├── PhoneNumber.cs            # Value object
│   │   │   ├── DateOfBirth.cs            # Value object
│   │   │   ├── Role.cs                   # Value object (Seller/Admin/SystemAdmin/Buyer)
│   │   │   └── UserSettings.cs           # Value object (Theme + Culture)
│   │   └── Store/
│   │       └── Store.cs                  # AGGREGATE: Store
│   ├── Enums/                            # UserStatus, Gender, StoreStatus, AddressType
│   ├── Exceptions/                       # Domain-specific exceptions
│   ├── Repositories/                     # Repository interfaces
│   └── Services/                         # UserManager, StoreManager (domain services)
│
├── HiveSpace.UserService.Application/    # Application layer
│   ├── Services/                         # IUserService, IAdminService, IStoreService, IUserAddressService
│   ├── DTOs/                             # Request/response DTOs per feature
│   ├── Interfaces/                       # Service interfaces
│   ├── Validators/                       # FluentValidation validators
│   ├── Extensions/                       # ApplicationServiceCollectionExtensions
│   └── Constant/                         # Enum definitions
│
├── HiveSpace.UserService.Infrastructure/ # Infrastructure layer
│   ├── Data/                             # UserDbContext
│   ├── Identity/                         # ApplicationUser (extends IdentityUser<Guid>), CustomUserStore
│   ├── EntityConfigurations/
│   │   ├── ApplicationUserEntityConfiguration.cs  # EF config: users table
│   │   ├── AddressEntityConfiguration.cs          # EF config: addresses table
│   │   └── StoreEntityConfiguration.cs            # EF config: stores table
│   ├── Migrations/                       # EF Core migration files
│   ├── Repositories/                     # Concrete repository implementations
│   ├── Messaging/                        # MassTransit consumers + event publishers
│   ├── Mappers/                          # Domain ↔ Infrastructure mappers
│   ├── DataSeeder.cs                     # Seed SystemAdmin, sellers, buyers
│   ├── DataSeeder.Admin.cs
│   ├── DataSeeder.Sellers.cs
│   ├── DataSeeder.TestUsers.cs
│   └── UserInfrastructureExtension.cs    # IServiceCollection extensions for Infrastructure
│
└── HiveSpace.UserService.Api/            # ENTRY POINT: port 5001
    ├── Program.cs                        # Bootstrap + Serilog + DataSeeder call
    ├── Controllers/
    │   ├── AccountController.cs          # /api/v1/accounts — email verification
    │   ├── UserController.cs             # /api/v1/users — profile + settings
    │   ├── UserAddressController.cs      # /api/v1/users/address
    │   ├── AdminController.cs            # /api/v1/admins
    │   └── StoreController.cs            # /api/v1/stores
    ├── Extensions/
    │   ├── HostingExtensions.cs          # ConfigureServices + ConfigurePipeline
    │   └── ServiceCollectionExtensions.cs # Identity, IdentityServer, JWT, CORS setup
    ├── Configs/                          # IdentityServer client/scope config
    ├── Consumers/                        # MassTransit consumers
    ├── Middleware/                       # Custom middleware
    ├── Services/                         # Localization service
    ├── appsettings.json                  # DB, IdentityServer clients, Messaging config
    └── Dockerfile
```

---

### CatalogService

```
src/HiveSpace.CatalogService/
├── HiveSpace.CatalogService.Domain/
│   ├── Aggregates/
│   │   ├── ProductAggregate/
│   │   │   ├── Product.cs                # AGGREGATE: Product (owns Skus, Variants, Images, Categories, Attributes)
│   │   │   ├── Sku.cs                    # Entity: SKU with Price (Money VO)
│   │   │   ├── ProductVariant.cs
│   │   │   ├── ProductVariantOption.cs
│   │   │   ├── ProductAttribute.cs
│   │   │   ├── ProductCategory.cs
│   │   │   ├── ProductImage.cs
│   │   │   ├── SkuImage.cs
│   │   │   └── SkuVariant.cs
│   │   ├── CategoryAggregate/
│   │   │   ├── Category.cs               # AGGREGATE: Category (owns CategoryAttributes)
│   │   │   └── CategoryAttribute.cs
│   │   └── AttributeAggregate/           # Attribute + AttributeValue aggregates
│   ├── ValueObjects/                     # Weight, Dimensions, Price (Money)
│   ├── Enums/                            # ProductStatus, ProductCondition
│   ├── Exceptions/                       # CatalogDomainException
│   └── Repositories/                     # Repository interfaces
│
├── HiveSpace.CatalogService.Application/
│   ├── Products/
│   │   ├── Commands/                     # CreateProduct, UpdateProduct, DeleteProduct
│   │   └── Queries/                      # GetProduct, GetProducts, GetProductDetail, GetProductSummaries
│   ├── Categories/
│   │   └── Queries/                      # GetCategories, GetHomepageCategories, GetAttributesByCategoryId
│   ├── Contracts/                        # ProductUpsertRequestDto, ProductSearchRequestDto
│   ├── Interfaces/                       # Application service interfaces
│   └── Helpers/                          # Query helpers
│
├── HiveSpace.CatalogService.Infrastructure/
│   ├── EntityConfigurations/
│   │   ├── ProductConfiguration.cs       # products, product_categories, product_attributes, product_images
│   │   ├── SkuConfiguration.cs           # skus, sku_images, sku_variants
│   │   ├── CategoryConfiguration.cs      # categories, category_attributes
│   │   ├── ProductVariantConfiguration.cs
│   │   ├── AttributeConfiguration.cs
│   │   ├── AttributeValueConfiguration.cs
│   │   └── StoreRefValueConfiguration.cs
│   ├── Migrations/                       # 5 migration files
│   ├── Repositories/                     # Concrete implementations
│   ├── Messaging/                        # MassTransit consumers
│   ├── DataSeeder.cs
│   └── CatalogInfrastructureExtensions.cs
│
└── HiveSpace.CatalogService.Api/         # ENTRY POINT: port 5002
    ├── Program.cs
    ├── Endpoints/
    │   ├── ProductEndpoints.cs           # /api/v1/products — static extension method
    │   └── CategoryEndpoints.cs          # /api/v1/categories — static extension method
    ├── Extensions/                        # ConfigureServices, ConfigurePipeline
    ├── Consumers/                         # MassTransit consumers
    ├── appsettings.json
    └── Dockerfile
```

---

### OrderService

```
src/HiveSpace.OrderService/
├── HiveSpace.OrderService.Domain/
│   ├── Aggregates/
│   │   ├── Orders/
│   │   │   ├── Order.cs                  # AGGREGATE: Order (owns Items, Checkouts, Discounts, Trackings)
│   │   │   ├── OrderItem.cs
│   │   │   ├── Checkout.cs               # VO: payment method + amount
│   │   │   ├── Discount.cs               # VO: coupon applied discount
│   │   │   └── OrderTracking.cs          # VO: audit trail entry
│   │   ├── Carts/
│   │   │   ├── Cart.cs                   # AGGREGATE: Cart
│   │   │   ├── CartItem.cs
│   │   │   ├── CartAppliedPlatformCoupon.cs
│   │   │   └── CartAppliedStoreCoupon.cs
│   │   └── Coupons/
│   │       ├── Coupon.cs                 # AGGREGATE: Coupon (owns Rules + Usages)
│   │       ├── CouponRule.cs
│   │       ├── CouponUsage.cs
│   │       └── CouponValidationResult.cs
│   ├── ValueObjects/                     # DeliveryAddress, Money (re-exported)
│   ├── Enumerations/                     # OrderStatus (Enumeration), PaymentMethod
│   ├── Exceptions/
│   ├── External/                         # External reference VOs (store snapshots, etc.)
│   └── Repositories/                     # IOrderRepository, ICartRepository, ICouponRepository
│
├── HiveSpace.OrderService.Application/
│   ├── Orders/
│   │   ├── Commands/                     # ConfirmOrder, RejectOrder
│   │   └── Queries/                      # GetOrderById, GetOrderList, GetSellerOrders
│   ├── Cart/
│   │   ├── Commands/                     # AddCartItem, RemoveCartItem, UpdateCartItems, ApplyCoupon, RemoveCoupon
│   │   └── Queries/                      # GetCartSummary, GetCheckoutPreview, GetSelectedCartItemsCount
│   ├── Coupons/
│   │   ├── Commands/                     # CreateCoupon, UpdateCoupon, DeleteCoupon, EndCoupon
│   │   └── Queries/                      # GetCouponById, GetCouponList, GetAvailableCoupons
│   ├── Interfaces/                       # ICheckoutQuery, IOrderEventPublisher
│   └── Contracts/                        # Shared DTOs
│
├── HiveSpace.OrderService.Infrastructure/
│   ├── EntityConfigurations/
│   │   ├── Orders/
│   │   │   ├── OrderEntityConfiguration.cs      # orders table (with DeliveryAddress VO, Money VOs)
│   │   │   ├── OrderItemEntityConfiguration.cs
│   │   │   └── OrderTrackingEntityConfiguration.cs
│   │   ├── Carts/
│   │   │   ├── CartEntityConfiguration.cs
│   │   │   └── CartItemEntityConfiguration.cs
│   │   ├── Coupons/
│   │   │   ├── CouponEntityConfiguration.cs
│   │   │   ├── CouponRuleEntityConfiguration.cs
│   │   │   └── CouponUsageEntityConfiguration.cs
│   │   └── External/
│   ├── Sagas/
│   │   ├── CheckoutSagaState.cs                  # EF-persisted saga state
│   │   ├── CheckoutSagaStateEntityConfiguration.cs  # checkout_saga_states table
│   │   ├── FulfillmentSagaState.cs
│   │   └── FulfillmentSagaStateEntityConfiguration.cs
│   ├── Migrations/                       # 10+ migration files
│   ├── Repositories/                     # Concrete repository implementations
│   ├── Messaging/                        # Event publishers, consumers
│   ├── Data/                             # OrderDbContext
│   ├── DataSeeder.cs
│   └── OrderInfrastructureExtension.cs
│
└── HiveSpace.OrderService.Api/           # ENTRY POINT: port 5004
    ├── Program.cs
    ├── Endpoints/
    │   ├── CartEndpoints.cs              # /api/v1/carts/* — Minimal API
    │   ├── CheckoutEndpoints.cs          # /api/v1/orders/checkout — Minimal API
    │   ├── OrderEndpoints.cs             # /api/v1/orders/* — Minimal API
    │   ├── CouponEndpoints.cs            # /api/v1/coupons/* — Minimal API
    │   └── HealthEndpoints.cs
    ├── Sagas/
    │   ├── CheckoutSaga/
    │   │   └── CheckoutSagaStateMachine.cs   # KEY: Full checkout orchestration saga
    │   └── FulfillmentSaga/
    │       └── FulfillmentSagaStateMachine.cs # KEY: Per-order seller confirmation saga
    ├── Consumers/                         # MassTransit consumers (order operations)
    ├── Models/                            # Request model records
    ├── Extensions/                        # ConfigureServices, ConfigurePipeline
    └── appsettings.json
```

---

### PaymentService

```
src/HiveSpace.PaymentService/
├── HiveSpace.PaymentService.Domain/
│   ├── Aggregates/
│   │   ├── Payments/
│   │   │   ├── Payment.cs                # AGGREGATE: Payment (Pending→Processing→Succeeded/Failed/Cancelled/Expired)
│   │   │   └── Enumerations/             # PaymentStatus, PaymentGateway
│   │   └── Wallets/
│   │       ├── Wallet.cs                 # AGGREGATE: Wallet (owns Transactions)
│   │       └── Transaction.cs
│   ├── ValueObjects/                     # GatewayResponse, PaymentMethod
│   ├── Exceptions/
│   └── Repositories/
│
├── HiveSpace.PaymentService.Application/
│   ├── Payments/
│   │   ├── Commands/                     # ProcessPaymentWebhook (VNPay IPN handler)
│   │   └── Queries/                      # GetPayment, GetPaymentByOrderId
│   ├── Wallets/
│   │   └── Queries/                      # GetWallet, GetTransactionHistory
│   ├── Interfaces/                       # Payment gateway interface
│   └── ApplicationServiceCollectionExtensions.cs
│
├── HiveSpace.PaymentService.Infrastructure/
│   ├── EntityConfigurations/
│   │   ├── Payments/
│   │   │   └── PaymentEntityConfiguration.cs   # payments table
│   │   └── Wallets/
│   │       ├── WalletEntityConfiguration.cs
│   │       └── TransactionEntityConfiguration.cs
│   ├── Gateways/                         # VNPayGateway implementation
│   ├── Messaging/                        # MassTransit consumers (InitiatePayment, MarkOrderAsPaid)
│   ├── Migrations/
│   ├── Repositories/
│   ├── DataSeeder.cs
│   └── PaymentInfrastructureExtensions.cs
│
├── HiveSpace.PaymentService.Api/         # ENTRY POINT: port 5005
│   ├── Program.cs
│   ├── Endpoints/
│   │   ├── PaymentEndpoints.cs           # /api/v1/payments/* — VNPay return + webhook + queries
│   │   ├── WalletEndpoints.cs            # /api/v1/wallets/*
│   │   └── HealthEndpoints.cs
│   ├── Consumers/                         # API-layer consumers
│   ├── Extensions/
│   └── appsettings.json
│
└── HiveSpace.PaymentService.Tests/       # xUnit test project (in progress)
```

---

### NotificationService

```
src/HiveSpace.NotificationService/
├── HiveSpace.NotificationService.Core/   # All domain + application + infrastructure logic
│   ├── DomainModels/
│   │   ├── Notification.cs               # Domain model: Notification (Create factory, status transitions)
│   │   ├── DeliveryAttempt.cs
│   │   ├── NotificationTemplate.cs       # Scriban template stored in DB
│   │   ├── NotificationEventGroup.cs     # Event group definitions (Order, Account)
│   │   ├── UserChannelPreference.cs
│   │   ├── UserGroupPreference.cs
│   │   ├── Enum/                         # NotificationChannel, NotificationStatus
│   │   └── External/                     # UserRef (cross-service reference)
│   ├── Features/
│   │   ├── Notifications/
│   │   │   ├── Commands/                 # MarkNotificationRead
│   │   │   └── Queries/                  # GetNotifications, GetUnreadCount
│   │   └── Preferences/
│   │       ├── Commands/                 # UpsertChannelPreference, UpsertGroupPreference
│   │       └── Queries/                  # GetPreferences
│   ├── Persistence/
│   │   ├── NotificationDbContext.cs      # DbContext with outbox config
│   │   ├── EntityConfigurations/         # 6 EF config files (snake_case tables)
│   │   ├── Repositories/
│   │   └── Migrations/
│   ├── Infrastructure/
│   │   ├── Channels/
│   │   │   ├── InApp/                    # In-app delivery (DB write + SignalR push)
│   │   │   └── Email/                    # Email delivery (Resend API + Scriban template)
│   │   ├── Storage/                      # Redis cache
│   │   └── Configuration/
│   ├── BackgroundJobs/                   # Hangfire job definitions
│   ├── Dispatch/                         # Notification dispatch orchestration
│   ├── Services/                         # Template rendering (Scriban)
│   ├── Extensions/
│   └── Interfaces/
│
└── HiveSpace.NotificationService.Api/    # ENTRY POINT: port 5006
    ├── Program.cs                        # Serilog bootstrap
    ├── Endpoints/
    │   ├── NotificationEndpoints.cs      # /api/v1/notifications
    │   ├── PreferenceEndpoints.cs        # /api/v1/notification-preferences
    │   └── DevEndpoints.cs               # Development test endpoints
    ├── Hubs/
    │   ├── NotificationHub.cs            # SignalR hub at /hubs/notifications
    │   ├── NotificationHubContext.cs     # IHubContext wrapper
    │   └── SubClaimUserIdProvider.cs     # Maps 'sub' claim to user ID
    ├── Consumers/
    │   ├── NotifySellerNewOrderConsumer.cs
    │   ├── NotifyBuyerOrderConfirmedConsumer.cs
    │   ├── NotifyBuyerOrderCancelledConsumer.cs
    │   ├── EmailVerificationRequestedConsumer.cs
    │   ├── EmailVerifiedConsumer.cs
    │   └── Sync/                         # User sync consumers (from UserService events)
    ├── Extensions/
    │   └── ApplicationServiceCollectionExtensions.cs  # Hangfire, Redis, SignalR, MassTransit
    └── appsettings.json
```

---

### MediaService

```
src/HiveSpace.MediaService/
├── HiveSpace.MediaService.Core/          # Domain model + features + infrastructure
│   ├── DomainModels/
│   │   ├── MediaAsset.cs                 # AGGREGATE: MediaAsset (Pending→Uploaded→Processed/Failed)
│   │   ├── MediaStatus.cs                # Enum
│   │   └── Enum/
│   ├── Features/
│   │   └── Media/
│   │       └── Commands/
│   │           ├── GeneratePresignedUrl/  # Generates Azure SAS URL + creates MediaAsset record
│   │           └── ConfirmUpload/         # Marks asset as uploaded, queues processing
│   ├── Persistence/
│   │   └── Repositories/
│   ├── Infrastructure/
│   │   ├── Storage/                      # Azure Blob Storage client wrapper
│   │   ├── Data/                         # MediaDbContext
│   │   └── Configuration/
│   ├── Migrations/                       # 1 migration: InitialCreate
│   ├── Interfaces/
│   └── Services/
│
├── HiveSpace.MediaService.Api/           # ENTRY POINT: port 5003
│   ├── Program.cs
│   ├── Endpoints/
│   │   └── MediaEndpoints.cs             # /api/v1/media/presign-url + /api/v1/media/{id}/confirm
│   ├── Extensions/
│   ├── appsettings.json                  # AzureStorage, Database (note: not ConnectionStrings)
│   └── Dockerfile
│
└── HiveSpace.MediaService.Func/          # Azure Functions project
    ├── Program.cs                        # Worker host setup
    ├── Functions/                        # Azure Function triggers (Storage Queue)
    ├── host.json                         # Function host configuration
    └── local.settings.json               # Local dev settings (Azure Storage, Application Insights)
```

---

## `libs/` — Shared Libraries

```
libs/
├── HiveSpace.Core/                       # Cross-cutting utilities
│   ├── Contexts/
│   │   └── IUserContext.cs               # Current user abstraction (sub claim extraction)
│   ├── Exceptions/                       # BadRequestException, ConflictException, DomainException
│   ├── Extensions/                       # Service collection extensions
│   ├── Filters/                          # Exception handling filters
│   ├── Helpers/
│   │   └── ValidationHelper.cs          # Throws BadRequestException on FluentValidation errors
│   ├── Middlewares/                      # Global exception handling middleware
│   ├── Models/                           # ApiResponse, PaginatedResult
│   ├── OpenApi/                          # Scalar/Swagger setup extensions
│   ├── IdGeneration/                     # Snowflake ID generator
│   └── CoreServiceCollectionExtensions.cs # Registers JWT bearer, CORS, MediatR behaviors
│
├── HiveSpace.Domain.Shared/              # Base DDD building blocks
│   ├── Entities/
│   │   ├── Entity.cs                     # Base entity with Id + equality
│   │   ├── AggregateRoot.cs              # Extends Entity, implements IAggregateRoot
│   │   ├── Enumeration.cs                # Smart enum base (Name, Id, FromValue, FromDisplayName)
│   │   └── ValueObject.cs                # Equality by components, Copy<T>()
│   ├── ValueObjects/
│   │   └── Money.cs                      # Multi-currency money VO (long-based, VND/USD/EUR)
│   ├── Enumerations/                     # PaymentMethod, Currency, Theme, Culture, OrderStatus shared types
│   ├── Interfaces/                       # IAuditable, ISoftDeletable, IAggregateRoot, ISpecification
│   ├── Exceptions/                       # NotFoundException, InvalidFieldException, DomainException bases
│   ├── Errors/                           # DomainErrorCode
│   ├── IdGeneration/                     # SnowflakeIdGenerator
│   └── Specifications/                   # ISpecification<T> base
│
├── HiveSpace.Application.Shared/         # MediatR-based CQRS base patterns
│   ├── Commands/
│   │   └── ICommand.cs                   # Marker: ICommand<TResult> : IRequest<TResult>
│   ├── Queries/
│   │   └── IQuery.cs                     # Marker: IQuery<TResult> : IRequest<TResult>
│   ├── Behaviors/                        # MediatR pipeline behaviors (logging, validation)
│   └── Handlers/                         # Base handler types
│
├── HiveSpace.Infrastructure.Authorization/   # Role-based authorization
│   ├── Attributes/
│   │   └── HiveSpaceAuthorizeAttribute.cs  # Policies: RequireSeller, RequireAdmin, RequireUser, etc.
│   └── Extensions/                         # AddHiveSpaceAuthorization (policy registration)
│
├── HiveSpace.Infrastructure.Messaging/       # MassTransit wiring
│   ├── Abstractions/
│   │   ├── IMessageBus.cs                # Abstraction over MassTransit IBus
│   │   └── IEventPublisher.cs
│   ├── Configurations/
│   │   ├── MessagingOptions.cs           # Section: "Messaging"
│   │   ├── RabbitMqOptions.cs            # Section: "Messaging:RabbitMq"
│   │   ├── KafkaOptions.cs               # Section: "Messaging:Kafka"
│   │   └── AzureServiceBusOptions.cs
│   ├── Extensions/
│   │   ├── MassTransitExtensions.cs      # AddMessagingCore, AddEntityOutBox (outbox model builder)
│   │   ├── RabbitMqExtensions.cs         # AddMassTransitWithRabbitMq<TDbContext> — main registration
│   │   ├── KafkaExtensions.cs
│   │   └── AzureServiceBusExtensions.cs
│   └── Services/
│       └── MassTransitMessageBus.cs      # IMessageBus implementation
│
├── HiveSpace.Infrastructure.Messaging.Shared/  # Message contracts (shared across services)
│   ├── CheckoutSaga/
│   │   ├── Commands/                     # CheckoutInitiated, CreateOrder, ReserveInventory,
│   │   │                                 # MarkOrderAsCOD, CommitCouponUsage, ClearCart,
│   │   │                                 # InitiatePayment, MarkOrderAsPaid, ReleaseInventory,
│   │   │                                 # CancelOrder, NotifySellerNewOrder, NotifyBuyerOrderCancelled,
│   │   │                                 # NotifyBuyerOrderConfirmed, ConfirmInventory
│   │   ├── Events/                       # OrderCreated, InventoryReserved, OrderMarkedAsCOD,
│   │   │                                 # CouponUsageCommitted, CartCleared, PaymentInitiated,
│   │   │                                 # OrderMarkedAsPaid, InventoryReleased, OrderCancelled,
│   │   │                                 # OrderReadyForFulfillment, OrderConfirmedBySeller,
│   │   │                                 # OrderRejectedBySeller, PaymentTimeout, SagaStepExpired,
│   │   │                                 # SellerConfirmationExpired, BuyerNotified, SellerNewOrderNotified,
│   │   │                                 # InventoryConfirmed, CheckoutResponse, CheckoutFailed,
│   │   │                                 # *Failed events for each step
│   │   └── Dtos/                         # DeliveryAddressDto, CheckoutCouponSelectionDto, etc.
│   ├── Events/
│   │   ├── Media/                        # MediaAssetProcessed, etc.
│   │   ├── Products/                     # ProductCreated, ProductUpdated, etc.
│   │   ├── Stores/                       # StoreCreated, StoreUpdated
│   │   └── Users/                        # UserCreated, UserUpdated (for sync to other services)
│   └── IntegrationEvents/               # PaymentSucceededIntegrationEvent, PaymentFailedIntegrationEvent
│
└── HiveSpace.Infrastructure.Persistence/    # Generic persistence utilities
    ├── Idempotence/
    │   ├── IncomingRequestRepository.cs  # Generic idempotency check (incoming request tracking)
    │   └── IncomingRequestEntityConfiguration.cs  # incoming_requests table config
    ├── Interceptors/
    │   ├── AuditableInterceptor.cs       # Sets CreatedAt/UpdatedAt on IAuditable
    │   └── SoftDeleteInterceptor.cs      # Sets IsDeleted/DeletedAt on ISoftDeletable
    ├── Transaction/
    │   └── TransactionService.cs         # Generic DbContext transaction wrapper
    ├── Outbox/                           # Outbox-related utilities
    ├── Repositories/                     # IRepository<T, TKey> generic base
    ├── Seeding/                          # IDbSeeder base
    └── PersistenceServiceCollectionExtensions.cs  # AddPersistenceInfrastructure<TContext>
```

---

## `infra/` — Deployment

```
infra/
├── api-gateway.bicep                     # Azure Container App: YARP gateway
├── user-service.bicep                    # Azure Container App: UserService + IdentityServer
├── catalog-service.bicep                 # Azure Container App: CatalogService
├── media-service.bicep                   # Azure Container App: MediaService (API + Func)
├── media-service-api.bicep               # Azure Container App: MediaService API
└── parameters/                           # Environment-specific parameter files
```

---

## `scripts/`

```
scripts/
├── new-service.ps1                       # Scaffold new service from dotnet template
└── sync-config.sh                        # Sync config files across services
```

---

## `.github/workflows/`

```
.github/workflows/
├── api-gateway-ci.yml                    # Build + push ApiGateway to ACR + deploy to ACA
├── user-service-ci.yml                   # Build + push UserService to ACR + deploy to ACA
├── catalog-service-ci.yml                # Build + push CatalogService to ACR + deploy to ACA
├── media-service-api-ci.yml              # Build + push MediaService API to ACR + deploy to ACA
└── media-service-func-ci.yml             # Build + push MediaService Function to ACR + deploy to ACA
```

---

## Key File Quick Reference

| File | Purpose |
|---|---|
| `Directory.Packages.props` | ALL NuGet versions live here. Never version in `.csproj`. |
| `hivespace.microservice.sln` | Solution file — all 20+ projects registered |
| `src/HiveSpace.ApiGateway/.../appsettings.json` | YARP route table (all service ports) |
| `src/HiveSpace.UserService/.../appsettings.json` | IdentityServer client config + scopes |
| `src/HiveSpace.OrderService/.../Sagas/CheckoutSaga/CheckoutSagaStateMachine.cs` | Full checkout orchestration logic |
| `src/HiveSpace.OrderService/.../Sagas/FulfillmentSaga/FulfillmentSagaStateMachine.cs` | Per-order seller confirmation logic |
| `libs/HiveSpace.Infrastructure.Messaging/Extensions/RabbitMqExtensions.cs` | MassTransit + outbox registration for all services |
| `libs/HiveSpace.Infrastructure.Messaging.Shared/CheckoutSaga/` | All saga message contracts |
| `libs/HiveSpace.Domain.Shared/ValueObjects/Money.cs` | Monetary value object used across all services |
| `libs/HiveSpace.Infrastructure.Authorization/Attributes/HiveSpaceAuthorizeAttribute.cs` | Authorization policies |
