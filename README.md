# Retisio Portal

> **Self-serve retailer onboarding and multi-tenant control plane** — provision sandbox tenants, configure catalog/pricing/inventory, and promote retailers to production.

[![Live Demo](https://img.shields.io/badge/Live%20Demo-Visit-FF5C28?style=for-the-badge)](https://stripathi-retisio.github.io/shopforge-demo)
[![GitHub](https://img.shields.io/badge/GitHub-Repository-181717?style=for-the-badge&logo=github)](https://github.com/stripathi-retisio/shopforge-demo)
[![Phase](https://img.shields.io/badge/Phase-1%20Prototype-38BDF8?style=for-the-badge)](#roadmap)
[![Built with](https://img.shields.io/badge/Built%20with-HTML%2FCSS%2FJS-F59E0B?style=for-the-badge)](#tech-stack)

---

## What is Retisio Portal?

Retisio Portal is a **multi-tenant self-serve retailer onboarding and control plane** built on top of commerce infrastructure.

This is **not** a generic ecommerce SaaS or a Shopify clone. It is an operator platform that:

1. Lets retailers self-onboard and request a sandbox tenant
2. Provisions isolated sandbox environments instantly
3. Gives each retailer a tenant workspace to configure catalog, pricing, and inventory
4. Provisions a starter B2C storefront automatically for each tenant
5. Gives platform operators a control plane to manage lifecycle, provisioning state, and promotions

> **Phase 1 assumption:** channel = tenant. One tenant maps to one commerce channel. Multi-channel per tenant is Phase 2.

---

## The Full Flow

```
Landing
  ├── Start Retailer Onboarding → Package Selection → 5-Step Wizard → Sandbox Provisioned
  │     └── Tenant Workspace → Catalog → Pricing → Inventory → Orders → Users → Storefront Preview
  │           └── "Request Promotion" → Operator approves → Live in Production
  │
  ├── View Guided Demo → Demo selector (Workspace / Storefront / Operator Portal)
  │
  └── Operator Portal → Tenants → Provisioning → Orders → Lifecycle controls
```

---

## Features

### Self-Serve Onboarding
- **Package selection screen** — Sandbox Evaluation / Growth / Enterprise with clear entitlements before signup
- **5-step onboarding wizard:**
  1. Account (name, email, password)
  2. Tenant & Channel Setup (name, slug, category, business model: B2C / B2B / D2C)
  3. Package & Environment (confirm tier, environment defaults to Sandbox)
  4. Starter Storefront Theme (6 theme options)
  5. Sandbox Ready (provisioning state checklist)

### Tenant Workspace
Eight sections in the sidebar:

| Section | What's there |
|---|---|
| Overview | Lifecycle + env badges, setup checklist, revenue chart, recent orders, catalog preview |
| Catalog | Product CRUD — add, edit, delete, publish/unpublish |
| Pricing | Base price and compare price per item, discount display |
| Inventory | Stock levels per item with In Stock / Low Stock / Critical status |
| Orders | Order table with status filters and fulfillment update |
| Users | Team access table (demo data, Phase 2 backend) |
| Storefront Preview | Inline preview of B2C channel + channel metadata |
| Settings | Tenant info, brand color, feature toggles, tier/environment, deprovision |

**"Request Promotion to Production"** is available from both Overview and Settings when the tenant is in Sandbox and not yet Live.

### Starter Storefront
- Branded per-tenant (color, gradient, store name)
- Product catalog with cart sidebar
- Quantity controls, compare prices, stock count
- Checkout modal with customer details + payment method selection
- Order confirmation — order immediately appears in tenant's dashboard

### Operator Portal
Five sections:

| Section | What's there |
|---|---|
| Overview | Platform stats (tenants, live vs sandbox, pending promotions, orders), tier distribution, tenant list |
| Tenants | Full tenant table — lifecycle status dropdown, Activate / Suspend / Archive / Promote actions |
| Provisioning | Per-tenant matrix of 7 provisioning stages with ✓/— indicators |
| All Orders | Cross-tenant order ledger (null-safe, handles all edge cases) |
| DB Schema | Full schema documentation with ERD, tables, and indexing strategy |

### Lifecycle Statuses
```
Requested → Sandbox Ready → Setup In Progress → Ready for Production → Live
                                                                      ↓
                                                               Suspended / Archived
```

### Provisioning Stages (tracked per tenant)
1. Tenant Created
2. Channel Provisioned
3. Starter Storefront Provisioned
4. Catalog Complete
5. Pricing Complete
6. Inventory Complete
7. Production Promotion Complete

---

## Quick Start

```bash
git clone https://github.com/stripathi-retisio/shopforge-demo.git
cd shopforge-demo
open retisio-portal.html
```

No build step. No dependencies. Works in any modern browser. Data persists in `localStorage`.

---

## Demo Credentials

### Retailers (quick-login buttons on the login page)
| Tenant | Email | Password | Status | Env |
|---|---|---|---|---|
| Pixel & Thread | `hello@pixelthread.com` | `demo123` | Live | Production |
| Green Roots | `hi@greenroots.co` | `demo123` | Sandbox Ready | Sandbox |
| Urban Luxe | `contact@urbanluxe.in` | `demo123` | Ready for Production | Sandbox |

### Operator
| Email | Password |
|---|---|
| `operator@retisio.com` | `admin123` |

> Use **"View Guided Demo"** on the landing page — it opens a selector that explains all three demo paths and routes you directly.

---

## Project Structure

```
shopforge-demo/
├── retisio-portal.html     # Entire application (~1,600 lines, single file)
├── README.md               # You are here
├── PLAN.md                 # Vision, phases, roadmap, decisions
├── DESIGN.md               # Architecture, schema, flow design
├── IMPLEMENTATION.md       # Code walkthrough — every system explained
└── CONTRIBUTING.md         # How to contribute
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| UI | Vanilla HTML5 + CSS3 (custom properties, grid, flexbox) |
| Logic | Vanilla JavaScript ES6+ (no framework) |
| Routing | Custom SPA router — no URL hash, no library |
| State | In-memory JS + `localStorage` persistence |
| Fonts | Bricolage Grotesque (headings) · DM Sans (body) · DM Mono (code) |
| Icons | Inline SVG |
| Target DB | PostgreSQL with Row-Level Security per tenant |

---

## Roadmap

See [PLAN.md](PLAN.md) for the full phased plan.

**Phase 1 — complete (this prototype):**
- Self-serve onboarding with package selection
- Tenant workspace with 8 sections
- Starter B2C storefront with full cart/checkout
- Operator Portal with lifecycle management and provisioning tracker

**Phase 2 — planned:**
- Backend API (Node.js + Express or Supabase)
- Real PostgreSQL with RLS
- Real auth (JWT + bcrypt)
- Multi-channel per tenant
- Razorpay payment integration
- Cloudinary image uploads
- Resend/SendGrid transactional email
- Custom subdomain per tenant (`slug.retisio.io`)

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). All contributions welcome.

---

## License

MIT © [Shreyas Tripathi](https://github.com/stripathi-retisio)
