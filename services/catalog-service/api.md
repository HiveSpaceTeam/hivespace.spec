# CatalogService API

## Seller Products

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/products` | `RequireSeller` | Create product |
| GET | `/api/v1/products` | `RequireSeller` | List seller products |
| GET | `/api/v1/products/{id}` | `RequireSeller` | Get seller product detail |
| PUT | `/api/v1/products/{id}` | `RequireSeller` | Update product |
| DELETE | `/api/v1/products/{id}` | `RequireSeller` | Delete or deactivate product |

## Storefront Products

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/products/summaries` | Anonymous | Search/list storefront product summaries |
| GET | `/api/v1/products/detail/{id}` | Anonymous | Get storefront product detail with SKUs |

## Categories

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/categories` | Anonymous | Get full category tree |
| GET | `/api/v1/categories/homepage` | Anonymous | Get homepage category list |
| GET | `/api/v1/categories/{id}/attributes` | Anonymous | Get category attribute definitions |
