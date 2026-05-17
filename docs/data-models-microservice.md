# Data Models — HiveSpace Microservices

Each service owns its own SQL Server database. All schemas are managed via EF Core 9.0.1 code-first migrations. Table names use `snake_case` configured via `builder.ToTable(...)`. Column names follow `snake_case` where explicitly configured.

---

## Shared Conventions

### Money Value Object

Stored as two columns per usage site (EF Core owned type):

| Column Suffix | Type | Notes |
|---|---|---|
| `_Amount` | `long` | Smallest currency unit (e.g. VND, no decimals) |
| `_Currency` | `string` / enum | Currency code enum (e.g. VND) |

Example: `SubTotal_Amount bigint`, `SubTotal_Currency nvarchar(10)`.

Methods on the VO:
- `ExceedsCODLimit()`: `Amount > 2_000_000`
- `CalculateServiceFee()`: `Amount * 0.099` → new `Money`

### ID Generation

- `Guid` PKs: UUIDv7 via `GuidV7Generator` (time-ordered)
- `long` PKs: Snowflake via `SnowflakeIdGenerator` (monotonic)

---

## PaymentService Database (`payments`)

### Table: `payments`

Primary store for payment aggregates.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `Id` | `uniqueidentifier` | PK, not null | UUIDv7 |
| `OrderId` | `uniqueidentifier` | not null, unique index | One payment per order |
| `BuyerId` | `uniqueidentifier` | not null | Reference to UserService user |
| `Amount` | `bigint` | not null | Money VO amount |
| `Currency` | `nvarchar(10)` | not null | Money VO currency |
| `PaymentMethod_Type` | `nvarchar(50)` | not null | e.g. COD, VNPay, Wallet |
| `PaymentMethod_CardLast4` | `nvarchar(4)` | nullable | Card payments only |
| `PaymentMethod_CardBrand` | `nvarchar(50)` | nullable | Card payments only |
| `PaymentMethod_WalletProvider` | `nvarchar(50)` | nullable | Wallet payments only |
| `PaymentMethod_BankCode` | `nvarchar(50)` | nullable | Bank transfer payments |
| `Status` | `nvarchar(50)` | not null | Pending, Paid, Failed, Expired, Refunded |
| `Gateway` | `nvarchar(50)` | not null | VNPay, Momo, COD, Wallet |
| `GatewayTransactionId` | `nvarchar(200)` | nullable | Gateway's transaction reference |
| `GatewayPaymentUrl` | `nvarchar(2000)` | nullable | Redirect URL for online payments |
| `GatewayResponse_RawResponse` | `nvarchar(max)` | nullable | Raw response JSON from gateway |
| `GatewayResponse_Success` | `bit` | nullable | Parsed success flag |
| `GatewayResponse_ErrorMessage` | `nvarchar(500)` | nullable | Parsed error message |
| `IdempotencyKey` | `nvarchar(200)` | not null, unique index | Prevents duplicate payment creation |
| `PaidAt` | `datetime2` | nullable | Set when Status = Paid |
| `ExpiresAt` | `datetime2` | nullable | Payment link expiry |
| `CreatedAt` | `datetime2` | not null | |
| `UpdatedAt` | `datetime2` | not null | |

### Table: `wallets`

One wallet per user.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `Id` | `uniqueidentifier` | PK, not null | UUIDv7 |
| `UserId` | `uniqueidentifier` | not null, unique index | One wallet per user |
| `AvailableBalance_Amount` | `bigint` | not null | Spendable balance |
| `AvailableBalance_Currency` | `nvarchar(10)` | not null | |
| `EscrowBalance_Amount` | `bigint` | not null | Held for pending orders |
| `EscrowBalance_Currency` | `nvarchar(10)` | not null | |
| `RewardPoints` | `int` | not null | Loyalty points balance |
| `Status` | `nvarchar(50)` | not null | Active, Suspended |
| `SuspensionReason` | `nvarchar(500)` | nullable | Set when Status = Suspended |
| `CreatedAt` | `datetime2` | not null | |
| `UpdatedAt` | `datetime2` | not null | |

### Table: `transactions`

Append-only ledger of wallet movements.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `Id` | `uniqueidentifier` | PK, not null | UUIDv7 |
| `WalletId` | `uniqueidentifier` | not null, FK → `wallets(Id)` CASCADE | |
| `Amount` | `bigint` | not null | Transaction amount |
| `Currency` | `nvarchar(10)` | not null | |
| `RunningBalance` | `bigint` | not null | Available balance after transaction |
| `RunningBalanceCurrency` | `nvarchar(10)` | not null | |
| `Direction` | `nvarchar(10)` | not null | `Credit` or `Debit` |
| `Type` | `nvarchar(50)` | not null | e.g. Purchase, Refund, TopUp, Fee |
| `Reference` | `nvarchar(200)` | nullable | External reference (order ID, payment ID) |
| `Description` | `nvarchar(500)` | nullable | Human-readable description |
| `CreatedAt` | `datetime2` | not null | |

**Relationships**:
- `wallets` 1 → N `transactions` (cascade delete)
- `payments` is independent; `OrderId` is a logical foreign key (no DB constraint across services)

---

## NotificationService Database (`notifications`)

### Table: `notifications`

Core notification record.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `Id` | `uniqueidentifier` | PK, not null | UUIDv7 |
| `UserId` | `uniqueidentifier` | not null, indexed | Recipient |
| `Channel` | `nvarchar(20)` | not null | `InApp` or `Email` |
| `EventType` | `nvarchar(100)` | not null | e.g. `order.confirmed`, `order.rejected` |
| `IdempotencyKey` | `nvarchar(200)` | not null, unique index | Prevents duplicate delivery |
| `Status` | `nvarchar(20)` | not null | `Pending`, `Sent`, `Failed`, `Throttled`, `Dead`, `Read` |
| `Payload` | `nvarchar(max)` | not null | JSON — template variables |
| `AttemptCount` | `int` | not null, default 0 | Delivery attempt counter |
| `CreatedAt` | `datetime2` | not null | |
| `SentAt` | `datetime2` | nullable | Set when Status = Sent |
| `ReadAt` | `datetime2` | nullable | Set when Status = Read (InApp only) |
| `ErrorMessage` | `nvarchar(1000)` | nullable | Last delivery error |

### Table: `delivery_attempts`

Per-attempt audit log.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `Id` | `uniqueidentifier` | PK, not null | |
| `NotificationId` | `uniqueidentifier` | not null, FK → `notifications(Id)` | |
| `AttemptedAt` | `datetime2` | not null | |
| `Success` | `bit` | not null | |
| `ErrorMessage` | `nvarchar(1000)` | nullable | |
| `DurationMs` | `int` | nullable | Delivery duration |

### Table: `notification_templates`

Scriban templates per event type per channel.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `Id` | `uniqueidentifier` | PK, not null | |
| `EventType` | `nvarchar(100)` | not null | Matches `notifications.EventType` |
| `Channel` | `nvarchar(20)` | not null | `InApp` or `Email` |
| `Subject` | `nvarchar(200)` | nullable | Email subject (Email channel only) |
| `Body` | `nvarchar(max)` | not null | Scriban template string |
| `IsActive` | `bit` | not null, default 1 | |
| `CreatedAt` | `datetime2` | not null | |
| `UpdatedAt` | `datetime2` | not null | |

Unique constraint: `(EventType, Channel)`.

### Table: `user_channel_preferences`

Per-user channel opt-in/out.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `UserId` | `uniqueidentifier` | not null, composite PK | |
| `Channel` | `nvarchar(20)` | not null, composite PK | `InApp`, `Email` |
| `Enabled` | `bit` | not null, default 1 | |

### Table: `user_group_preferences`

Per-user event-group opt-in/out per channel.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `UserId` | `uniqueidentifier` | not null, composite PK | |
| `Channel` | `nvarchar(20)` | not null, composite PK | |
| `EventGroup` | `nvarchar(100)` | not null, composite PK | e.g. `OrderUpdates`, `PromotionAlerts` |
| `Enabled` | `bit` | not null, default 1 | |

### Table: `user_refs`

Local projection of user data from UserService.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `Id` | `uniqueidentifier` | PK, not null | Matches UserService userId |
| `Email` | `nvarchar(256)` | not null | For email delivery |
| `FullName` | `nvarchar(200)` | nullable | For template personalization |
| `UpdatedAt` | `datetime2` | not null | Last sync timestamp |

**Relationships**:
- `notifications` 1 → N `delivery_attempts` (no cascade; retain on notification delete for audit)
- `notification_templates` is a standalone config table
- `user_channel_preferences`, `user_group_preferences`, `user_refs` are standalone lookup tables

---

## OrderService Database (`orders`)

### Table: `orders`

Order aggregate root.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `Id` | `uniqueidentifier` | PK, not null | UUIDv7 |
| `OrderCode` | `nvarchar(50)` | not null, unique index | Human-readable order number |
| `UserId` | `uniqueidentifier` | not null, indexed | Buyer |
| `StoreId` | `uniqueidentifier` | not null, indexed | Seller's store |
| `Status` | `nvarchar(50)` | not null | Pending, Confirmed, Rejected, Cancelled, Completed |
| `RejectionReason` | `nvarchar(500)` | nullable | Set when Status = Rejected |
| `DeliveryAddress_RecipientName` | `nvarchar(200)` | not null | Owned address (flattened) |
| `DeliveryAddress_Phone` | `nvarchar(20)` | not null | |
| `DeliveryAddress_Street` | `nvarchar(500)` | not null | |
| `DeliveryAddress_Ward` | `nvarchar(100)` | nullable | |
| `DeliveryAddress_District` | `nvarchar(100)` | not null | |
| `DeliveryAddress_Province` | `nvarchar(100)` | not null | |
| `SubTotal_Amount` | `bigint` | not null | Sum of line totals |
| `SubTotal_Currency` | `nvarchar(10)` | not null | |
| `TotalDiscount_Amount` | `bigint` | not null | Total coupon discount |
| `TotalDiscount_Currency` | `nvarchar(10)` | not null | |
| `ShippingFee_Amount` | `bigint` | not null | |
| `ShippingFee_Currency` | `nvarchar(10)` | not null | |
| `TotalAmount_Amount` | `bigint` | not null | `SubTotal - TotalDiscount + ShippingFee` |
| `TotalAmount_Currency` | `nvarchar(10)` | not null | |
| `IsShippingPaidBySeller` | `bit` | not null | |
| `ShippingId` | `nvarchar(100)` | nullable | External shipping provider tracking ID |
| `CreatedAt` | `datetime2` | not null | |
| `UpdatedAt` | `datetime2` | not null | |

### Table: `order_items`

Line items belonging to an order.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `Id` | `uniqueidentifier` | PK, not null | |
| `OrderId` | `uniqueidentifier` | not null, FK → `orders(Id)` CASCADE | |
| `ProductId` | `bigint` | not null | Snowflake ID from CatalogService |
| `SkuId` | `bigint` | not null | Snowflake ID from CatalogService |
| `Quantity` | `int` | not null | |
| `UnitPrice_Amount` | `bigint` | not null | Price at time of order |
| `UnitPrice_Currency` | `nvarchar(10)` | not null | |
| `LineTotal_Amount` | `bigint` | not null | `Quantity * UnitPrice` |
| `LineTotal_Currency` | `nvarchar(10)` | not null | |
| `IsCOD` | `bit` | not null | Whether this item is COD eligible |
| `ProductSnapshot_Name` | `nvarchar(500)` | not null | Product name snapshot at order time |
| `ProductSnapshot_ImageUrl` | `nvarchar(2000)` | nullable | |
| `ProductSnapshot_SkuAttributes` | `nvarchar(max)` | nullable | JSON array of variant attributes |

### Table: `order_checkouts`

Checkout metadata associated with an order.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `OrderId` | `uniqueidentifier` | PK (shadow), FK → `orders(Id)` | One per order |
| `PaymentMethod` | `nvarchar(50)` | not null | COD, VNPay, Wallet |
| `Amount` | `bigint` | not null | Amount charged |
| `Currency` | `nvarchar(10)` | not null | |

### Table: `order_discounts`

Coupon discounts applied to an order.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `Id` | `uniqueidentifier` | PK (shadow) | |
| `OrderId` | `uniqueidentifier` | not null, FK → `orders(Id)` CASCADE | |
| `CouponCode` | `nvarchar(50)` | not null | |
| `CouponOwnerType` | `nvarchar(20)` | not null | `Platform` or `Store` |
| `Scope` | `nvarchar(50)` | not null | e.g. `Order`, `Product`, `Category` |
| `Amount` | `bigint` | not null | Discount amount |
| `Currency` | `nvarchar(10)` | not null | |

### Table: `order_trackings`

Immutable event log for order lifecycle events.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `Id` | `uniqueidentifier` | PK, not null | |
| `OrderId` | `uniqueidentifier` | not null, FK → `orders(Id)` | |
| `Type` | `nvarchar(100)` | not null | Event type name |
| `ExecutorType` | `nvarchar(50)` | not null | `System`, `Seller`, `Buyer`, `Admin` |
| `ExecutorId` | `uniqueidentifier` | nullable | Who triggered the event |
| `Message` | `nvarchar(500)` | nullable | Human-readable description |
| `CreatedAt` | `datetime2` | not null | |

### Table: `coupons`

Coupon aggregate.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `Id` | `uniqueidentifier` | PK, not null | UUIDv7 |
| `Code` | `nvarchar(50)` | not null, unique index | |
| `Name` | `nvarchar(200)` | not null | |
| `DiscountType` | `nvarchar(20)` | not null | `Fixed` or `Percentage` |
| `Scope` | `nvarchar(50)` | not null | `Order`, `Product`, `Category` |
| `OwnerType` | `nvarchar(20)` | not null | `Platform` or `Store` |
| `StoreId` | `uniqueidentifier` | nullable | Null for platform coupons |
| `DiscountAmount_Amount` | `bigint` | nullable | Fixed discount (DiscountType = Fixed) |
| `DiscountAmount_Currency` | `nvarchar(10)` | nullable | |
| `MaxDiscountAmount_Amount` | `bigint` | nullable | Cap for percentage discounts |
| `MaxDiscountAmount_Currency` | `nvarchar(10)` | nullable | |
| `MinOrderAmount_Amount` | `bigint` | nullable | Minimum order total to apply |
| `MinOrderAmount_Currency` | `nvarchar(10)` | nullable | |
| `DiscountPercentage` | `decimal(5,2)` | nullable | Percentage (DiscountType = Percentage) |
| `IsHidden` | `bit` | not null, default 0 | Hidden coupons not shown in public listings |
| `ApplicableProductIds` | `nvarchar(max)` | nullable | JSON array of `long` product IDs |
| `ApplicableCategoryIds` | `nvarchar(max)` | nullable | JSON array of category IDs |
| `StartDateTime` | `datetime2` | not null | |
| `EndDateTime` | `datetime2` | not null | |
| `EarlySaveDateTime` | `datetime2` | nullable | Early access start for hidden coupons |
| `CreatedAt` | `datetime2` | not null | |
| `UpdatedAt` | `datetime2` | not null | |

### Table: `coupon_rules`

Additional eligibility rules for a coupon.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `Id` | `uniqueidentifier` | PK, not null | |
| `CouponId` | `uniqueidentifier` | not null, FK → `coupons(Id)` CASCADE | |
| `RuleType` | `nvarchar(50)` | not null | e.g. `MaxUsesTotal`, `MaxUsesPerUser` |
| `Value` | `nvarchar(200)` | not null | Rule value |

### Table: `coupon_usages`

Tracks coupon redemptions per user.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `Id` | `uniqueidentifier` | PK, not null | |
| `CouponId` | `uniqueidentifier` | not null, FK → `coupons(Id)` CASCADE | |
| `UserId` | `uniqueidentifier` | not null | |
| `OrderId` | `uniqueidentifier` | not null | |
| `UsedAt` | `datetime2` | not null | |

### Table: `carts`

One cart per user.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `Id` | `uniqueidentifier` | PK, not null | |
| `UserId` | `uniqueidentifier` | not null, unique index | |
| `PlatformCouponCode` | `nvarchar(50)` | nullable | Applied platform coupon |
| `CreatedAt` | `datetime2` | not null | |
| `UpdatedAt` | `datetime2` | not null | |

### Table: `cart_items`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `Id` | `uniqueidentifier` | PK, not null | |
| `CartId` | `uniqueidentifier` | not null, FK → `carts(Id)` CASCADE | |
| `ProductId` | `bigint` | not null | |
| `SkuId` | `bigint` | not null | |
| `Quantity` | `int` | not null | |
| `IsSelected` | `bit` | not null, default 1 | |
| `StoreId` | `uniqueidentifier` | not null | |
| `StoreCouponCode` | `nvarchar(50)` | nullable | Applied store coupon for this store's items |
| `AddedAt` | `datetime2` | not null | |

### Tables: `product_refs`, `sku_refs`, `store_refs`

Local projections synced from CatalogService and UserService events. Used to avoid inter-service REST calls at checkout/cart time.

`product_refs`:

| Column | Type | Notes |
|---|---|---|
| `Id` | `bigint` | PK, Snowflake ID from CatalogService |
| `Name` | `nvarchar(500)` | |
| `StoreId` | `uniqueidentifier` | |
| `Status` | `nvarchar(50)` | |
| `UpdatedAt` | `datetime2` | |

`sku_refs`:

| Column | Type | Notes |
|---|---|---|
| `Id` | `bigint` | PK, Snowflake ID |
| `ProductId` | `bigint` | FK → product_refs |
| `Price_Amount` | `bigint` | Current price |
| `Price_Currency` | `nvarchar(10)` | |
| `StockQuantity` | `int` | |
| `Attributes` | `nvarchar(max)` | JSON array of variant attributes |
| `UpdatedAt` | `datetime2` | |

`store_refs`:

| Column | Type | Notes |
|---|---|---|
| `Id` | `uniqueidentifier` | PK |
| `Name` | `nvarchar(200)` | |
| `OwnerId` | `uniqueidentifier` | Seller's user ID |
| `UpdatedAt` | `datetime2` | |

### MassTransit Saga State Tables

#### `CheckoutSagaState`

Persisted state for the `CheckoutSaga` MassTransit state machine.

| Column | Type | Notes |
|---|---|---|
| `CorrelationId` | `uniqueidentifier` | PK — saga instance ID |
| `CurrentState` | `nvarchar(50)` | Current state machine state name |
| `OrderIds` | `nvarchar(max)` | JSON array of created order IDs |
| `BuyerId` | `uniqueidentifier` | |
| `PaymentMethod` | `nvarchar(50)` | COD or online payment type |
| `TotalAmount_Amount` | `bigint` | |
| `TotalAmount_Currency` | `nvarchar(10)` | |
| `PaymentId` | `uniqueidentifier` | nullable — set after payment created |
| `FailureReason` | `nvarchar(500)` | nullable |
| `CreatedAt` | `datetime2` | |
| `UpdatedAt` | `datetime2` | |

#### `FulfillmentSagaState`

Persisted state for the `FulfillmentSaga` MassTransit state machine (one instance per seller sub-order).

| Column | Type | Notes |
|---|---|---|
| `CorrelationId` | `uniqueidentifier` | PK — saga instance ID |
| `CurrentState` | `nvarchar(50)` | Current state name |
| `OrderId` | `uniqueidentifier` | Associated order |
| `StoreId` | `uniqueidentifier` | Seller's store |
| `SellerId` | `uniqueidentifier` | Seller's user ID |
| `BuyerId` | `uniqueidentifier` | |
| `ConfirmationDeadline` | `datetime2` | 3 days from NotifyingSeller entry |
| `FailureReason` | `nvarchar(500)` | nullable |
| `CreatedAt` | `datetime2` | |
| `UpdatedAt` | `datetime2` | |

---

## UserService Database (`users`)

**Framework**: ASP.NET Identity + Duende IdentityServer EF tables (standard schemas, not documented in full here).

### Custom Extension: `app_users` (extends `AspNetUsers`)

| Column | Type | Notes |
|---|---|---|
| `Id` | `nvarchar(450)` | PK — matches ASP.NET Identity user ID |
| `DateOfBirth` | `datetime2` | nullable |
| `PhoneNumber` | `nvarchar(20)` | nullable (also on Identity base) |
| `AvatarUrl` | `nvarchar(2000)` | nullable |
| `Gender` | `nvarchar(20)` | nullable |
| `Theme` | `nvarchar(20)` | nullable — UI theme preference |
| `Status` | `nvarchar(20)` | not null — Active, Suspended, Deleted |

### Table: `user_addresses`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `Id` | `uniqueidentifier` | PK, not null | |
| `UserId` | `nvarchar(450)` | not null, FK → `AspNetUsers(Id)` | |
| `RecipientName` | `nvarchar(200)` | not null | |
| `Phone` | `nvarchar(20)` | not null | |
| `Street` | `nvarchar(500)` | not null | |
| `Ward` | `nvarchar(100)` | nullable | |
| `District` | `nvarchar(100)` | not null | |
| `Province` | `nvarchar(100)` | not null | |
| `IsDefault` | `bit` | not null, default 0 | Only one default per user |
| `CreatedAt` | `datetime2` | not null | |
| `UpdatedAt` | `datetime2` | not null | |

---

## CatalogService Database (`catalog`)

### Table: `products`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `Id` | `bigint` | PK — Snowflake ID | |
| `StoreId` | `uniqueidentifier` | not null, indexed | |
| `CategoryId` | `uniqueidentifier` | not null, FK → `categories(Id)` | |
| `Name` | `nvarchar(500)` | not null | |
| `Description` | `nvarchar(max)` | nullable | |
| `Status` | `nvarchar(50)` | not null | Draft, Active, Inactive |
| `IsCOD` | `bit` | not null | |
| `CreatedAt` | `datetime2` | not null | |
| `UpdatedAt` | `datetime2` | not null | |

### Table: `product_variants`

Groups of variant options (e.g. "Color", "Size").

| Column | Type | Notes |
|---|---|---|
| `Id` | `bigint` | PK — Snowflake |
| `ProductId` | `bigint` | FK → `products(Id)` CASCADE |
| `Name` | `nvarchar(100)` | e.g. "Color" |
| `DisplayOrder` | `int` | |

### Table: `product_variant_options`

Individual option values per variant group.

| Column | Type | Notes |
|---|---|---|
| `Id` | `bigint` | PK |
| `ProductVariantId` | `bigint` | FK → `product_variants(Id)` CASCADE |
| `Value` | `nvarchar(100)` | e.g. "Red" |
| `DisplayOrder` | `int` | |

### Table: `sku_variants`

Concrete SKUs combining variant options.

| Column | Type | Notes |
|---|---|---|
| `Id` | `bigint` | PK — Snowflake |
| `ProductId` | `bigint` | FK → `products(Id)` CASCADE |
| `Price_Amount` | `bigint` | |
| `Price_Currency` | `nvarchar(10)` | |
| `StockQuantity` | `int` | |
| `ImageUrl` | `nvarchar(2000)` | nullable |
| `Attributes` | `nvarchar(max)` | JSON — selected option IDs per variant |
| `CreatedAt` | `datetime2` | |
| `UpdatedAt` | `datetime2` | |

### Table: `categories`

Hierarchical category tree.

| Column | Type | Notes |
|---|---|---|
| `Id` | `uniqueidentifier` | PK |
| `ParentId` | `uniqueidentifier` | nullable, FK → `categories(Id)` |
| `Name` | `nvarchar(200)` | |
| `Slug` | `nvarchar(200)` | unique |
| `ImageUrl` | `nvarchar(2000)` | nullable |
| `DisplayOrder` | `int` | |
| `IsActive` | `bit` | |

### Table: `attribute_definitions`

Defines attribute types per category (e.g. "Brand", "Material").

| Column | Type | Notes |
|---|---|---|
| `Id` | `uniqueidentifier` | PK |
| `CategoryId` | `uniqueidentifier` | FK → `categories(Id)` |
| `Name` | `nvarchar(200)` | |
| `IsRequired` | `bit` | |
| `DisplayOrder` | `int` | |

### Table: `attribute_values`

Allowed values for an attribute definition.

| Column | Type | Notes |
|---|---|---|
| `Id` | `uniqueidentifier` | PK |
| `AttributeDefinitionId` | `uniqueidentifier` | FK → `attribute_definitions(Id)` |
| `Value` | `nvarchar(200)` | |

### Table: `product_attributes`

Values assigned to a product for its category attributes.

| Column | Type | Notes |
|---|---|---|
| `ProductId` | `bigint` | composite PK, FK → `products(Id)` |
| `AttributeDefinitionId` | `uniqueidentifier` | composite PK |
| `Value` | `nvarchar(500)` | |

### Table: `store_refs`

Local projection of store data from UserService.

| Column | Type | Notes |
|---|---|---|
| `Id` | `uniqueidentifier` | PK |
| `Name` | `nvarchar(200)` | |
| `OwnerId` | `uniqueidentifier` | |
| `UpdatedAt` | `datetime2` | |

---

## MediaService Database (`media`)

### Table: `media_assets`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `Id` | `uniqueidentifier` | PK, not null | UUIDv7 |
| `FileName` | `nvarchar(500)` | not null | System-generated storage file name |
| `OriginalFileName` | `nvarchar(500)` | not null | Client-provided file name |
| `StoragePath` | `nvarchar(2000)` | not null | Azure Blob path |
| `PublicUrl` | `nvarchar(2000)` | nullable | CDN or public blob URL; set after Uploaded |
| `ThumbnailUrl` | `nvarchar(2000)` | nullable | Set after Processed |
| `MimeType` | `nvarchar(100)` | not null | e.g. `image/jpeg` |
| `FileSize` | `bigint` | not null | Bytes |
| `EntityType` | `nvarchar(100)` | nullable | e.g. `Product`, `User`, `Store` |
| `EntityId` | `nvarchar(200)` | nullable | ID of owning entity; set on confirm |
| `Status` | `nvarchar(20)` | not null | `Pending`, `Uploaded`, `Processed`, `Failed` |
| `CreatedAt` | `datetime2` | not null | |
| `UpdatedAt` | `datetime2` | not null | |

---

## MassTransit Infrastructure Tables (all publishing services)

Present in the databases of: OrderService, PaymentService, CatalogService, NotificationService.

### Table: `outbox_message`

| Column | Type | Notes |
|---|---|---|
| `SequenceNumber` | `bigint` | PK, auto-increment |
| `EnqueueTime` | `datetime2` | nullable — scheduled delivery |
| `SentTime` | `datetime2` | not null |
| `Headers` | `nvarchar(max)` | nullable — JSON |
| `Properties` | `nvarchar(max)` | nullable — JSON |
| `InboxMessageId` | `uniqueidentifier` | nullable |
| `InboxConsumerId` | `uniqueidentifier` | nullable |
| `OutboxId` | `uniqueidentifier` | nullable |
| `MessageId` | `uniqueidentifier` | not null |
| `ContentType` | `nvarchar(256)` | not null |
| `MessageType` | `nvarchar(max)` | not null |
| `Body` | `nvarchar(max)` | not null — JSON message body |
| `ConversationId` | `uniqueidentifier` | nullable |
| `CorrelationId` | `uniqueidentifier` | nullable |
| `InitiatorId` | `uniqueidentifier` | nullable |
| `RequestId` | `uniqueidentifier` | nullable |
| `SourceAddress` | `nvarchar(256)` | nullable |
| `DestinationAddress` | `nvarchar(256)` | nullable |
| `ResponseAddress` | `nvarchar(256)` | nullable |
| `FaultAddress` | `nvarchar(256)` | nullable |
| `ExpirationTime` | `datetime2` | nullable |

### Table: `outbox_state`

Tracks outbox delivery state per EF DbContext instance.

| Column | Type | Notes |
|---|---|---|
| `OutboxId` | `uniqueidentifier` | PK |
| `LockId` | `uniqueidentifier` | Optimistic concurrency |
| `LockUntil` | `datetime2` | nullable |
| `LastSequenceNumber` | `bigint` | nullable |
| `Created` | `datetime2` | |
| `Delivered` | `datetime2` | nullable |
| `RowVersion` | `rowversion` | Concurrency token |

### Table: `inbox_state`

Deduplication store for consumed messages.

| Column | Type | Notes |
|---|---|---|
| `Id` | `bigint` | PK, auto-increment |
| `MessageId` | `uniqueidentifier` | not null, unique |
| `ConsumerId` | `uniqueidentifier` | not null |
| `LockId` | `uniqueidentifier` | |
| `RowVersion` | `rowversion` | |
| `Received` | `datetime2` | |
| `ReceiveCount` | `int` | |
| `ExpirationTime` | `datetime2` | nullable |
| `Consumed` | `datetime2` | nullable |
| `Delivered` | `datetime2` | nullable |
| `LastSequenceNumber` | `bigint` | nullable |

**Pattern note**: The outbox guarantees at-least-once delivery. The inbox deduplicates by `MessageId` + `ConsumerId` pair to achieve effectively-once consumer processing.
