# DESIGN.md — Architecture & Design

## System Overview

Retisio Portal is a single-page application that simulates the complete architecture of a multi-tenant retailer onboarding and control plane. The current implementation uses `localStorage` as a mock database, but every layer — data model, routing, lifecycle logic, access control — is designed to map 1:1 to a production backend.

```
┌──────────────────────────────────────────────────────────┐
│                   retisio-portal.html                     │
│                                                           │
│  ┌──────────┐  ┌────────────────────┐  ┌───────────────┐ │
│  │  Router  │→ │   View Layer       │  │  DB Layer     │ │
│  │  (SPA)   │  │   (render fns)     │  │ localStorage  │ │
│  └──────────┘  └────────────────────┘  └───────────────┘ │
│                                                           │
│  Routes:                                                  │
│  ├── landing          (public)                            │
│  ├── package          (package selection)                 │
│  ├── signup           (5-step onboarding)                 │
│  ├── login            (retailer auth)                     │
│  ├── dashboard        (tenant workspace, 8 tabs)          │
│  ├── store            (starter storefront, per-tenant)    │
│  ├── admin            (operator login)                    │
│  ├── admin-dashboard  (operator portal, 5 tabs)           │
│  └── schema           (DB documentation)                  │
└──────────────────────────────────────────────────────────┘
```

---

## Multi-Tenancy Design

### Strategy: Tenant ID Scoping + Row-Level Security

Every retailer is a **tenant**. Every piece of data — products, orders — is scoped to a `tenantId`. There is no shared catalog, no shared order table from a user perspective.

```
tenants
  └── products    (tenantId FK)
  └── orders      (tenantId FK)
  └── sessions    (userId FK)
```

In the mock DB (`localStorage`), each tenant's data is stored under namespaced keys:
```
sf_products_{tenantId}
sf_orders_{tenantId}
```

In production PostgreSQL, this maps to Row-Level Security:
```sql
ALTER TABLE products ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON products
  USING (tenant_id = current_setting('app.current_tenant')::TEXT);
```

Operators bypass RLS via a `BYPASSRLS` role. Retailers only ever see their own data.

---

## Lifecycle & Provisioning Model

### Lifecycle Statuses

Tenants move through a defined set of statuses:

| Status | Who sets it | Meaning |
|---|---|---|
| `requested` | System (on signup request) | Tenant record created, not yet provisioned |
| `sandbox_ready` | System (on signup completion) | Sandbox provisioned, workspace available |
| `setup_in_progress` | Operator / auto-detected | Retailer is actively configuring |
| `ready_for_production` | Retailer (via "Request Promotion") | Retailer considers setup complete |
| `live` | Operator (via "Promote to Production") | Running in production environment |
| `suspended` | Operator | Temporarily disabled |
| `archived` | Operator | Permanently deactivated |

### Provisioning Stages

Each tenant has a `provisioningStages` JSONB object tracking 7 milestones:

```json
{
  "tenantCreated": true,
  "channelProvisioned": true,
  "storefrontProvisioned": true,
  "catalogComplete": false,
  "pricingComplete": false,
  "inventoryComplete": false,
  "productionPromoted": false
}
```

These are set automatically (first 3 on signup) and updated as the retailer configures their workspace. Visible to the operator in the Provisioning tab.

### Environment Model

```
sandbox → staging (optional) → production
```

Phase 1 uses `sandbox` and `production` only. Staging is in the schema for Phase 2.

---

## Database Schema

### Phase 1 Note: Channel = Tenant

In Phase 1, one tenant = one commerce channel. The `tenants` table represents both the retailer entity and their single commerce channel. Multi-channel per tenant is Phase 2.

### Entity Relationship Diagram

```
plans ──────────────────── tenants ──────────────── products
                               │
                               └─────────────────── orders
                               │
                               └── sessions ──────── (tenants | operators)

operators ──────────────── sessions
```

---

### `tenants`
The core multi-tenancy entity. One row = one retailer + their commerce channel.

| Column | Type | Notes |
|---|---|---|
| id | VARCHAR(30) | PK — `tenant_TIMESTAMP` |
| name | VARCHAR(255) | Tenant / retailer display name |
| slug | VARCHAR(100) | UNIQUE — URL identifier (`retisio.io/slug`) |
| email | VARCHAR(255) | UNIQUE — retailer login email |
| password | VARCHAR(255) | bcrypt hash in production |
| ownerName | VARCHAR(255) | Full name of retailer owner |
| plan | ENUM | `starter \| growth \| enterprise` |
| businessModel | ENUM | `B2C \| B2B \| D2C` |
| status | ENUM | `active \| suspended \| deleted` |
| tenantStatus | ENUM | `requested \| sandbox_ready \| setup_in_progress \| ready_for_production \| live \| suspended \| archived` |
| environment | ENUM | `sandbox \| staging \| production` |
| provisioningStages | JSONB | 7 boolean flags tracking setup milestones |
| revenue | DECIMAL(12,2) | Cumulative gross revenue (denormalized) |
| theme | JSONB | `{ primaryColor, bgGradient }` — starter storefront branding |
| settings | JSONB | `{ maintenanceMode, allowReviews, freeShipping, currency }` |
| createdAt | TIMESTAMP | |
| updatedAt | TIMESTAMP | |

---

### `products`
Catalog items for each retailer tenant. Fully scoped and isolated per tenant.

| Column | Type | Notes |
|---|---|---|
| id | VARCHAR(30) | PK — `prod_TIMESTAMP` |
| tenantId | VARCHAR(30) | FK → tenants.id |
| name | VARCHAR(255) | |
| description | TEXT | |
| price | DECIMAL(10,2) | Selling price |
| comparePrice | DECIMAL(10,2) | Original / strike-through price, nullable |
| category | VARCHAR(100) | |
| stock | INTEGER | Available inventory count |
| emoji | VARCHAR(10) | Placeholder for image URL (Phase 1) |
| published | BOOLEAN | Visible on starter storefront |
| createdAt | TIMESTAMP | |

---

### `orders`
Customer purchase orders placed on any tenant storefront.

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

---

### `plans`
Subscription package definitions.

| Column | Type | Notes |
|---|---|---|
| id | VARCHAR(20) | PK — `starter \| growth \| enterprise` |
| name | VARCHAR(100) | Display name |
| priceMonthly | DECIMAL(8,2) | |
| priceAnnual | DECIMAL(8,2) | |
| maxTenants | INTEGER | NULL = unlimited |
| maxProducts | INTEGER | NULL = unlimited |
| environments | JSONB | Allowed environments per plan |
| features | JSONB | Array of feature flag strings |

---

### `operators`
Retisio platform operator accounts.

| Column | Type | Notes |
|---|---|---|
| id | VARCHAR(30) | PK |
| email | VARCHAR(255) | UNIQUE |
| password | VARCHAR(255) | bcrypt hash |
| name | VARCHAR(255) | |
| role | ENUM | `super_operator \| support \| finance` |
| createdAt | TIMESTAMP | |

---

### `sessions`
Active sessions for both retailers and operators.

| Column | Type | Notes |
|---|---|---|
| id | VARCHAR(36) | UUID session token |
| userId | VARCHAR(30) | FK → tenants.id or operators.id |
| role | ENUM | `retailer \| operator` |
| expiresAt | TIMESTAMP | |
| createdAt | TIMESTAMP | |

---

## Indexing Strategy

```sql
-- Storefront lookup (most frequent query)
CREATE UNIQUE INDEX tenants_slug_idx ON tenants(slug);

-- Retailer login
CREATE UNIQUE INDEX tenants_email_idx ON tenants(email);

-- Lifecycle and environment filtering (Operator Portal)
CREATE INDEX tenants_status_idx ON tenants(tenant_status, environment);

-- Storefront catalog (published items only)
CREATE INDEX products_tenant_pub_idx ON products(tenant_id, published)
  WHERE published = true;

-- Retailer dashboard catalog list
CREATE INDEX products_tenant_idx ON products(tenant_id);

-- Order listing (retailer dashboard)
CREATE INDEX orders_tenant_idx ON orders(tenant_id);

-- Order status filter
CREATE INDEX orders_status_idx ON orders(tenant_id, status);

-- Chronological order feeds
CREATE INDEX orders_created_idx ON orders(created_at DESC);
```

---

## SPA Router Design

Custom lightweight router — no library, no URL hash.

```js
Router.go(route, params)
  └── sets Router.current + Router.params
  └── checks session validity for protected routes
  └── calls the correct render function
  └── (no separate attach function — event handlers inline or via global fns)
```

### Route Table

| Route | Auth | Renders |
|---|---|---|
| `landing` | No | Landing page with 3 CTAs |
| `package` | No | Package selection screen |
| `signup` | No | 5-step onboarding wizard |
| `login` | No | Retailer login |
| `dashboard` | Merchant session | Tenant workspace (8 tabs) |
| `store` | No | Starter storefront (by slug) |
| `admin` | No | Operator login |
| `admin-dashboard` | Admin session | Operator Portal (5 tabs) |
| `schema` | No | DB schema documentation |

---

## Session & Auth Design

**Current (mock):**
```js
DB.setSession({ role: 'merchant', tenantId, email, name })
DB.setSession({ role: 'admin', email, name })
DB.clearSession()
```

**Production design:**
```
POST /auth/login
  → validate email + bcrypt.compare(password, hash)
  → issue JWT access token (15 min) + refresh token (7 days)
  → store refresh token in httpOnly cookie

All API requests:
  → Authorization: Bearer <access_token>
  → Middleware extracts tenantId from JWT claims
  → Sets app.current_tenant for RLS
  → Operators bypass RLS via BYPASSRLS role
```

---

## Storefront Theming

Each tenant has a `theme` object in their row:

```json
{
  "primaryColor": "#FF5C28",
  "bgGradient": "linear-gradient(135deg, #0b0b0f, #1c0a00)"
}
```

At render time, the storefront injects these as inline CSS and uses `hexToRgb()` to decompose the brand color for `rgba()` gradient overlays — so the hero glow always matches the brand color.

---

## Production Architecture (Planned)

```
                    ┌──────────────────────┐
                    │   CDN / Edge          │
                    │  (static assets)      │
                    └────────┬─────────────┘
                             │
              ┌──────────────┴─────────────┐
              │          Load Balancer      │
              └──────────────┬─────────────┘
                             │
         ┌───────────────────┴──────────────────────┐
         │               API Server                  │
         │           (Node.js / Express)             │
         │                                           │
         │  /auth         /tenants     /products     │
         │  /orders       /operator    /webhooks     │
         └───────────────────┬──────────────────────┘
                             │
              ┌──────────────┴─────────────┐
              │         PostgreSQL          │
              │    (RLS per tenant)         │
              └────────────────────────────┘
```
