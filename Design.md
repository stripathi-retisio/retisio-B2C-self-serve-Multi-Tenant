# DESIGN.md — Architecture & Design

## System Overview

ShopForge is a single-page application that simulates the full architecture of a multi-tenant SaaS ecommerce platform. The current implementation uses `localStorage` as a mock database, but every layer — the data model, the routing, the access control logic — is designed to map 1:1 to a production backend.

```
┌─────────────────────────────────────────────────────────┐
│                     shopforge.html                       │
│                                                          │
│  ┌─────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │ Router  │  │  View Layer  │  │    DB Layer (mock)  │  │
│  │  (SPA)  │→ │  (renderers) │  │    localStorage     │  │
│  └─────────┘  └──────────────┘  └────────────────────┘  │
│                                                          │
│  Views:                                                  │
│  ├── Landing        (public)                             │
│  ├── Signup         (merchant onboarding)                │
│  ├── Login          (merchant auth)                      │
│  ├── Dashboard      (merchant portal)                    │
│  ├── Store          (public storefront, per-tenant)      │
│  ├── Admin Login    (platform admin)                     │
│  ├── Admin Dashboard(super-admin portal)                 │
│  └── Schema         (DB documentation)                   │
└─────────────────────────────────────────────────────────┘
```

---

## Multi-Tenancy Design

### Strategy: Tenant ID Scoping

Every merchant is a **tenant**. Every piece of data — products, orders — is scoped to a `tenantId`. There is no shared product catalog, no shared order table from a user perspective.

```
tenants
  └── products (tenantId FK)
  └── orders   (tenantId FK)
  └── sessions (userId FK)
```

In the mock DB (localStorage), each tenant's data is stored under namespaced keys:
```
sf_products_{tenantId}
sf_orders_{tenantId}
```

In production PostgreSQL, this maps to **Row-Level Security**:
```sql
-- Enable RLS on products table
ALTER TABLE products ENABLE ROW LEVEL SECURITY;

-- Only the owning tenant can see/modify their rows
CREATE POLICY tenant_isolation ON products
  USING (tenant_id = current_setting('app.current_tenant')::TEXT);
```

### Tenant Isolation Guarantees
- Merchant A **cannot** see Merchant B's products or orders
- Admin can see **all** tenants' data (bypasses RLS via `BYPASSRLS` role)
- Public storefront only shows **published** products for the requested tenant

---

## Database Schema

### Entity Relationship Diagram

```
plans ─────────────── tenants ──────────────── products
                          │
                          └───────────────────── orders
                          │
                          └── sessions ──────── (admin | tenant)

admins ────────────── sessions
```

### Tables

#### `tenants`
The core multi-tenancy entity. One row = one merchant store.

| Column | Type | Notes |
|---|---|---|
| id | VARCHAR(30) | PK — `tenant_TIMESTAMP` |
| name | VARCHAR(255) | Store display name |
| slug | VARCHAR(100) | UNIQUE — URL identifier (`/slug`) |
| email | VARCHAR(255) | UNIQUE — login email |
| password | VARCHAR(255) | bcrypt hash in production |
| ownerName | VARCHAR(255) | Full name |
| plan | ENUM | `starter \| growth \| enterprise` |
| status | ENUM | `active \| suspended \| deleted` |
| category | VARCHAR(100) | Business category |
| revenue | DECIMAL(12,2) | Cumulative gross revenue |
| theme | JSONB | `{ primaryColor, bgGradient }` |
| settings | JSONB | `{ maintenanceMode, allowReviews, freeShipping, currency }` |
| createdAt | TIMESTAMP | |
| updatedAt | TIMESTAMP | |

#### `products`
Fully scoped to a tenant. No cross-tenant sharing.

| Column | Type | Notes |
|---|---|---|
| id | VARCHAR(30) | PK — `prod_TIMESTAMP` |
| tenantId | VARCHAR(30) | FK → tenants.id |
| name | VARCHAR(255) | |
| description | TEXT | |
| price | DECIMAL(10,2) | Selling price |
| comparePrice | DECIMAL(10,2) | Crossed-out original price, nullable |
| category | VARCHAR(100) | |
| stock | INTEGER | Inventory count |
| emoji | VARCHAR(10) | Placeholder for image URL |
| published | BOOLEAN | Visibility on storefront |
| createdAt | TIMESTAMP | |

#### `orders`
Created when a customer checks out on any tenant storefront.

| Column | Type | Notes |
|---|---|---|
| id | VARCHAR(20) | PK — `ORD-XXXXXX` |
| tenantId | VARCHAR(30) | FK → tenants.id |
| customerName | VARCHAR(255) | |
| customerEmail | VARCHAR(255) | |
| address | TEXT | Shipping address |
| items | JSONB | `[{ productId, name, price, qty, emoji }]` |
| total | DECIMAL(10,2) | |
| paymentMethod | VARCHAR(50) | UPI \| Card \| Net Banking \| COD |
| status | ENUM | `pending \| processing \| shipped \| delivered \| cancelled` |
| createdAt | TIMESTAMP | |
| updatedAt | TIMESTAMP | |

#### `admins`
Platform-level administrator accounts.

| Column | Type | Notes |
|---|---|---|
| id | VARCHAR(30) | PK |
| email | VARCHAR(255) | UNIQUE |
| password | VARCHAR(255) | bcrypt hash |
| name | VARCHAR(255) | |
| role | ENUM | `super_admin \| support \| finance` |
| createdAt | TIMESTAMP | |

#### `sessions`
Active sessions for merchants and admins.

| Column | Type | Notes |
|---|---|---|
| id | VARCHAR(36) | UUID session token |
| userId | VARCHAR(30) | FK → tenants.id or admins.id |
| role | ENUM | `merchant \| admin` |
| expiresAt | TIMESTAMP | |
| createdAt | TIMESTAMP | |

#### `plans`
Subscription plan definitions.

| Column | Type | Notes |
|---|---|---|
| id | VARCHAR(20) | PK — `starter \| growth \| enterprise` |
| name | VARCHAR(100) | |
| priceMonthly | DECIMAL(8,2) | |
| priceAnnual | DECIMAL(8,2) | |
| maxProducts | INTEGER | NULL = unlimited |
| transactionFeePercent | DECIMAL(4,2) | |
| features | JSONB | Array of feature flag strings |

---

## Indexing Strategy

```sql
-- Storefront lookup (most frequent query)
CREATE UNIQUE INDEX tenants_slug_idx ON tenants(slug);

-- Merchant login
CREATE UNIQUE INDEX tenants_email_idx ON tenants(email);

-- Product catalog (storefront)
CREATE INDEX products_tenant_pub_idx ON products(tenant_id, published)
  WHERE published = true;

-- Merchant dashboard product list
CREATE INDEX products_tenant_idx ON products(tenant_id);

-- Order listing (merchant dashboard)
CREATE INDEX orders_tenant_idx ON orders(tenant_id);

-- Order status filter
CREATE INDEX orders_status_idx ON orders(tenant_id, status);

-- Chronological order feed
CREATE INDEX orders_created_idx ON orders(created_at DESC);
```

---

## SPA Router Design

ShopForge implements its own lightweight router — no library, no URL hash.

```
Router.go(route, params)
  └── sets Router.current + Router.params
  └── checks session validity
  └── calls the correct render function
  └── calls the corresponding attach function (event binding)
```

**Route table:**

| Route | Auth Required | Renders |
|---|---|---|
| `landing` | No | Landing page |
| `signup` | No | 4-step onboarding wizard |
| `login` | No | Merchant login |
| `dashboard` | Merchant session | Merchant portal |
| `store` | No | Public storefront (by slug) |
| `admin` | No | Admin login |
| `admin-dashboard` | Admin session | Super-admin portal |
| `schema` | No | DB schema documentation |

---

## Session & Auth Design

Current (mock):
```js
DB.setSession({ role: 'merchant', tenantId, email, name })
DB.setSession({ role: 'admin', email, name })
DB.clearSession()
```

Production design:
```
POST /auth/login
  → validate email + bcrypt.compare(password, hash)
  → issue JWT access token (15 min) + refresh token (7 days)
  → store refresh token in httpOnly cookie

All API requests:
  → Authorization: Bearer <access_token>
  → Middleware extracts tenantId from JWT
  → Sets app.current_tenant for RLS
```

---

## Storefront Theming

Each tenant has a `theme` object stored in their row:

```json
{
  "primaryColor": "#FF5C28",
  "bgGradient": "linear-gradient(135deg, #0b0b0f, #1c0a00)"
}
```

At render time, the storefront injects these as CSS inline styles and RGB-decomposed values for gradient overlays. This makes every storefront visually distinct with zero extra configuration.

---

## UI Architecture

### Component Patterns
- All UI is rendered via `render*()` functions that return HTML strings
- Event handlers are attached via `attach*()` functions called after render
- Modals are created dynamically and appended to `document.body`
- Toasts are self-contained and auto-remove after 2.8 seconds

### Design Tokens (CSS Variables)
```css
--brand: #FF5C28        /* Primary orange */
--bg: #0B0B0F           /* Page background */
--surface: #141418      /* Card background */
--surface2: #1C1C23     /* Elevated surface */
--text: #F0EFE9         /* Primary text */
--text2: #9997A0        /* Secondary text */
--border: rgba(255,255,255,0.08)
--radius: 12px
```

---

## Production Architecture (Planned)

```
                    ┌─────────────────────┐
                    │   CDN / Edge         │
                    │  (static assets)     │
                    └────────┬────────────┘
                             │
              ┌──────────────┴──────────────┐
              │         Load Balancer        │
              └──────────────┬──────────────┘
                             │
         ┌───────────────────┴──────────────────┐
         │              API Server               │
         │         (Node.js / Express)           │
         │                                       │
         │  /auth      /products    /orders      │
         │  /tenants   /admin       /webhooks    │
         └───────────────────┬──────────────────┘
                             │
              ┌──────────────┴──────────────┐
              │        PostgreSQL            │
              │   (RLS per tenant)           │
              └─────────────────────────────┘
```
