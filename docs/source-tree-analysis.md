# Source Tree Analysis

## Repository Structure Overview

HiveSpace is organized across two primary repositories. This document shows annotated directory trees for both.

---

## hivespace.microservice (Backend)

```
hivespace.microservice/
├── src/                                    # Microservice implementations
│   ├── HiveSpace.ApiGateway/
│   │   └── HiveSpace.YarpApiGateway/      # ENTRY POINT: YARP reverse proxy (port 5000/7001)
│   │       ├── Program.cs                  # Startup: YARP config, JWT validation
│   │       └── appsettings*.json           # Route config to downstream services
│   │
│   ├── HiveSpace.UserService/              # Identity & auth (Duende IdentityServer, port 5001)
│   │   ├── HiveSpace.UserService.Api/      # ENTRY POINT: Carter modules, IdentityServer middleware
│   │   ├── HiveSpace.UserService.Application/   # Commands/queries (MediatR), user use cases
│   │   ├── HiveSpace.UserService.Domain/        # User, Role aggregates, value objects
│   │   ├── HiveSpace.UserService.Infrastructure/ # EF Core (UserDbContext), email, JWT issuance
│   │   ├── README.md                            # Seeded accounts, client registration
│   │   └── SEEDED_ACCOUNTS.md                   # Dev seed data
│   │
│   ├── HiveSpace.CatalogService/           # Product catalog, inventory
│   │   ├── HiveSpace.CatalogService.Api/        # ENTRY POINT: Carter modules, JWT auth
│   │   ├── HiveSpace.CatalogService.Application/ # Product/category commands & queries
│   │   ├── HiveSpace.CatalogService.Domain/      # Product, Category, Stock aggregates
│   │   └── HiveSpace.CatalogService.Infrastructure/ # EF Core (CatalogDbContext), messaging consumers
│   │
│   ├── HiveSpace.OrderService/             # Order management + checkout sagas
│   │   ├── HiveSpace.OrderService.Api/          # ENTRY POINT: Carter modules
│   │   ├── HiveSpace.OrderService.Application/  # Cart/, Orders/, Contracts/, Coupons/ use cases
│   │   ├── HiveSpace.OrderService.Domain/        # Order, Cart, Coupon aggregates
│   │   └── HiveSpace.OrderService.Infrastructure/ # EF Core, CheckoutSaga, FulfillmentSaga
│   │
│   ├── HiveSpace.PaymentService/           # Payment processing (in development)
│   │   ├── HiveSpace.PaymentService.Api/        # ENTRY POINT: Carter modules
│   │   ├── HiveSpace.PaymentService.Application/
│   │   ├── HiveSpace.PaymentService.Domain/      # Payment, Wallet, Refund, Escrow aggregates
│   │   └── HiveSpace.PaymentService.Infrastructure/
│   │
│   ├── HiveSpace.NotificationService/      # Notifications (lightweight 2-project structure)
│   │   ├── HiveSpace.NotificationService.Api/   # ENTRY POINT: Carter modules + SignalR hub
│   │   └── HiveSpace.NotificationService.Core/  # Consumers, email/in-app logic (no DDD layer)
│   │
│   └── HiveSpace.MediaService/             # File upload/processing (Azure Functions)
│       └── (Azure Functions Worker, HTTP triggers for upload/resize)
│
├── libs/                                   # Shared cross-service libraries
│   ├── HiveSpace.Core/                     # JWT helpers, core utilities
│   ├── HiveSpace.Domain.Shared/            # Base classes:
│   │   ├── Entities/                       #   Entity<TId> (private setters + protected ctor)
│   │   ├── ValueObjects/                   #   ValueObject base + Copy<T>()
│   │   ├── Exceptions/                     #   NotFoundException, InvalidFieldException
│   │   ├── Enumerations/                   #   Enumeration base class
│   │   ├── Specifications/                 #   Specification pattern
│   │   └── Interfaces/                     #   IRepository<T>, IAggregateRoot
│   ├── HiveSpace.Application.Shared/       # MediatR base patterns:
│   │   ├── Commands/                       #   ICommand, ICommandHandler<TCmd, TResult>
│   │   ├── Queries/                        #   IQuery, IQueryHandler<TQuery, TResult>
│   │   ├── Behaviors/                      #   Validation pipeline behavior (FluentValidation)
│   │   └── Handlers/                       #   Base handler utilities
│   ├── HiveSpace.Infrastructure.Authorization/   # JWT/policy authorization helpers
│   ├── HiveSpace.Infrastructure.Messaging/       # MassTransit + RabbitMQ setup abstractions
│   ├── HiveSpace.Infrastructure.Messaging.Shared/ # Integration event contracts (shared across services)
│   └── HiveSpace.Infrastructure.Persistence/     # Generic EF Core repository, DbContext patterns
│
├── infra/                                  # Azure Bicep IaC
│   ├── api-gateway.bicep                   # Azure API Management deployment
│   └── parameters/                         # Per-env parameter files
│
├── scripts/                                # Dev helper scripts
├── templates/                              # Code scaffolding templates
├── docs/                                   # Existing docs (copilot instructions, agent workflows)
├── Directory.Packages.props                # CENTRALIZED NuGet version management (all versions here)
├── global.json                             # .NET SDK version pin
└── hivespace.microservice.sln              # Solution file (all projects)
```

### Key Files — Backend

| File | Purpose |
|------|---------|
| `Directory.Packages.props` | All NuGet package versions — never add Version= in .csproj |
| `global.json` | Pinned .NET SDK version |
| `src/{Service}/{Service}.Api/Program.cs` | Service startup: DI, Carter, MassTransit, EF Core |
| `src/{Service}/{Service}.Domain/*Aggregate.cs` | Domain aggregates with static Create() factory |
| `src/{Service}/{Service}.Infrastructure/*DbContext.cs` | EF Core context with outbox registration |
| `src/{Service}/{Service}.Infrastructure/Sagas/*Saga.cs` | MassTransit state machines (OrderService) |
| `libs/HiveSpace.Infrastructure.Messaging.Shared/` | All integration event/command contracts |

---

## hivespace.web (Frontend)

```
hivespace.web/
├── apps/                                   # Production SPA applications
│   ├── admin/                              # @hivespace/admin — Platform admin dashboard
│   │   ├── src/
│   │   │   ├── components/                 # Admin-specific UI components
│   │   │   ├── pages/                      # Route-level page components (*Page.vue)
│   │   │   ├── stores/                     # Pinia stores (admin-specific)
│   │   │   ├── services/                   # API service layer (axios calls)
│   │   │   ├── types/                      # TypeScript types (api/, store/)
│   │   │   ├── composables/                # Reusable composition functions
│   │   │   ├── router/                     # Vue Router config + auth guards
│   │   │   ├── i18n/                       # Locales (en/, vi/) per module
│   │   │   ├── config/                     # Env config (constants.ts, index.ts)
│   │   │   ├── App.vue                     # ENTRY POINT: Root component, ModalManager
│   │   │   └── main.ts                     # ENTRY POINT: Vue app bootstrap + OIDC init
│   │   ├── public/
│   │   ├── vite.config.ts                  # Vite + plugins config
│   │   └── package.json                    # @hivespace/admin
│   │
│   ├── seller/                             # @hivespace/seller — Seller dashboard (port 5174)
│   │   └── src/                            # Same layout as admin (components/pages/stores/services/...)
│   │
│   └── buyer/                              # @hivespace/buyer — Customer storefront (port 5175)
│       └── src/                            # Same layout + SignalR connection for real-time
│
├── packages/                               # Shared workspace packages
│   ├── shared/                             # @hivespace/shared — Core shared package
│   │   └── src/
│   │       ├── components/                 # Shared components (Button, Modal, Spinner, layout shells, etc.)
│   │       ├── composables/                # useAuth, useModal, etc.
│   │       ├── features/                   # Feature-level shared modules
│   │       ├── icons/                      # SVG icon components
│   │       ├── pages/                      # Shared page shells (auth callbacks, error pages)
│   │       ├── stores/                     # useAppStore (notifications, loading state)
│   │       ├── styles/                     # Global Tailwind base styles
│   │       ├── types/                      # AppUser, Pagination, Status, UserType, etc.
│   │       ├── utils/                      # Shared utilities
│   │       └── index.ts                    # ENTRY POINT: All public exports
│   │
│   ├── demo/                               # @hivespace/demo — Dev-only demo pages
│   ├── eslint-config/                      # @hivespace/eslint-config — Shared ESLint rules
│   ├── tsconfig/                           # @hivespace/tsconfig — Shared TS configs
│   └── vite-config/                        # @hivespace/vite-config — Shared Vite config factory
│
├── package.json                            # Root workspace (Turbo, pnpm scripts)
├── pnpm-workspace.yaml                     # pnpm workspace definition
├── turbo.json                              # Turbo pipeline (build order, caching)
└── eslint.config.ts                        # Root ESLint config
```

### Key Files — Frontend

| File | Purpose |
|------|---------|
| `packages/shared/src/index.ts` | Single export surface for @hivespace/shared — check this before writing anything new |
| `apps/{app}/src/main.ts` | Vue + Pinia + Router + i18n bootstrap, OIDC init |
| `apps/{app}/src/App.vue` | Root component — must include ModalManager |
| `apps/{app}/src/router/index.ts` | Route definitions + beforeEach auth guards |
| `apps/{app}/src/services/api.ts` | Axios singleton with JWT interceptor — never instantiate directly |
| `apps/{app}/src/stores/` | Pinia stores — all state lives here, not in components |
| `pnpm-workspace.yaml` | Declares all apps/ and packages/ as workspaces |
| `turbo.json` | Build pipeline: shared must build before apps |

---

## Critical Directory Rules

### Backend
- **Never cross-layer**: Domain projects must not reference Infrastructure or Api projects.
- **Shared contracts only**: Cross-service event types live exclusively in `libs/HiveSpace.Infrastructure.Messaging.Shared/`. Services reference only this package — never each other's domain projects.
- **NuGet versions centralized**: All versions in `Directory.Packages.props`. No `Version=` attributes in any `.csproj`.

### Frontend
- **@hivespace/shared first**: Always check `packages/shared/src/` before writing any new component, composable, or type.
- **Apps don't import each other**: Apps may depend on `@hivespace/shared` but never on each other.
- **Shared package changes require rebuild**: After modifying `packages/shared`, run `pnpm build:shared` before the consuming apps will see changes.

---

_Last generated: 2026-05-16_
