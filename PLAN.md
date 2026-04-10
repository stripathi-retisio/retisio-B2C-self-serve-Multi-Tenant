# PLAN.md — Product Plan

## Vision

Build a **self-serve, multi-tenant retailer onboarding and control plane** that proves the full product thinking behind a Retisio-style platform — from package selection and sandbox provisioning, to tenant workspace configuration, starter storefront, and operator lifecycle management.

The goal is a prototype that is self-explanatory as a Retisio Phase 1 product — not a generic ecommerce demo, not a strategy slide — a working control plane UI that tells the product story on its own.

---

## Problem Statement

Onboarding a retailer onto a multi-tenant commerce platform involves multiple parties (the retailer, the platform operator), multiple environments (sandbox, production), and multiple provisioning steps (tenant creation, channel setup, catalog, pricing, inventory, storefront). Today this is often manual, opaque, and error-prone.

Retisio solves this with:
- **Self-serve onboarding** — the retailer drives their own setup
- **Structured lifecycle management** — the operator can see exactly where each tenant is
- **Provisioning transparency** — both parties know what is done and what is pending
- **Promotion to production** — a deliberate, visible step instead of a vague handoff

---

## Phase 1 Assumptions

| Assumption | Rationale |
|---|---|
| Channel = Tenant | Simplifies the data model for Phase 1. Multi-channel per tenant is Phase 2. |
| Sandbox and Production only | Staging environment is in schema but not required for Phase 1 demo. |
| No real backend | localStorage as mock DB. All flows work without a server. |
| INR as default currency | Built for the Indian market. Trivially configurable. |
| Emoji as product image | No image upload infrastructure needed for prototype. |
| Passwords stored in plaintext | Prototype only. Production uses bcrypt + JWT. |

---

## Scope

### In Scope — Phase 1 (current)

| Area | What's built |
|---|---|
| Landing page | 3 clear CTAs, Retisio positioning, feature grid, package tiers |
| Guided Demo | Demo selector modal routing to Workspace / Storefront / Operator Portal |
| Package selection | Sandbox / Growth / Enterprise with entitlements, before signup |
| Onboarding wizard | 5 steps: Account → Tenant Setup → Package → Storefront → Sandbox Ready |
| Tenant workspace | 8-section sidebar: Overview, Catalog, Pricing, Inventory, Orders, Users, Storefront Preview, Settings |
| Setup checklist | Visible on Overview, tracks 8 provisioning milestones |
| Starter storefront | Per-tenant B2C channel with cart, checkout, order confirmation |
| Operator Portal | Tenants, Provisioning matrix, All Orders, Overview stats, DB Schema |
| Lifecycle management | 7 statuses, dropdown in Operator, "Request Promotion" in Tenant Workspace |
| Provisioning tracker | 7 stages per tenant, visible in Operator's Provisioning tab |
| DB schema page | Full PostgreSQL schema with RLS note, indexes, ERD |

### Out of Scope — Phase 1

- Real backend / database
- Payment processing
- Image uploads
- Email notifications
- Custom subdomains
- Multi-channel per tenant
- Real-time updates
- Mobile app

---

## Phases

### Phase 0 — Design ✅
- Define multi-tenancy strategy (tenant ID scoping + RLS)
- Design lifecycle statuses and provisioning stages
- Map all user flows (retailer, customer, operator)
- Choose single-file HTML approach for zero-friction demo

### Phase 1 — Prototype ✅
- Landing page with clear Retisio positioning
- Package selection screen
- 5-step onboarding wizard with all required fields
- Tenant workspace with 8 navigation sections
- Pricing and Inventory tabs (UI with real data from catalog)
- Users and Storefront Preview tabs
- Setup checklist on Overview
- "Request Promotion to Production" in workspace
- Starter B2C storefront with full cart/checkout/order flow
- Operator Portal with Provisioning matrix, lifecycle controls, Activate/Suspend/Archive/Promote
- Guided Demo selector on landing
- Full DB schema page with phase 1 assumption noted

### Phase 2 — Backend (Planned)
- [ ] Node.js + Express REST API (or Supabase)
- [ ] PostgreSQL with Row-Level Security per tenant
- [ ] JWT-based auth (access + refresh tokens)
- [ ] Real session management
- [ ] All CRUD endpoints mirroring current DB layer methods
- [ ] Tenant provisioning webhook events

### Phase 3 — Production Features (Planned)
- [ ] Razorpay / Stripe payment integration
- [ ] Cloudinary / S3 image uploads for catalog
- [ ] Transactional email (Resend / SendGrid)
- [ ] Custom subdomain per tenant (`slug.retisio.io`)
- [ ] Multi-channel per tenant (Phase 1 assumption removed)
- [ ] Real-time order updates (SSE or WebSocket)
- [ ] Analytics time-series data

### Phase 4 — Scale (Future)
- [ ] App marketplace / plugin system
- [ ] Storefront theme builder
- [ ] Multi-currency support
- [ ] Inventory alerts and reorder rules
- [ ] Affiliate / referral system
- [ ] Mobile apps (React Native)
- [ ] White-label Operator Portal

---

## Success Metrics (Phase 1 Prototype)

| Goal | Measure |
|---|---|
| Flow is self-explanatory | Someone unfamiliar with Retisio can understand the product from the landing page alone |
| Onboarding is complete | Retailer can go from landing → sandbox provisioned in under 3 minutes |
| Workspace is useful | All 8 tabs render correctly with real data |
| Operator has control | Operator can view, filter, promote, suspend, and archive any tenant |
| Schema is production-ready | DB schema can be applied to a real PostgreSQL instance with no changes |

---

## Decision Log

| Decision | Rationale |
|---|---|
| Single HTML file | Zero setup friction — open it, fork it, deploy it, share it |
| localStorage DB | Lets the schema, lifecycle, and provisioning story shine without backend complexity |
| Vanilla JS (no framework) | Demonstrates routing, state, and DOM logic from first principles; no build tools |
| Package selection before signup | Makes entitlements visible upfront, sets expectation before account creation |
| Theme selection in onboarding (not post-signup) | Gives immediate visual reward; can be changed in Settings |
| 5 onboarding steps (not 4) | Separating Package/Environment into its own step makes the sandbox-first model explicit |
| Provisioning stages on Operator | Operators need to know exactly what a tenant has and hasn't configured — not just a status label |
| "Request Promotion" from tenant + "Promote" from operator | Both sides of the workflow are represented; operator retains final control |
| `channel = tenant` note in schema | Transparent about the Phase 1 simplification; sets up Phase 2 correctly |

---

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| localStorage 5MB cap | Fine for demo; production uses Postgres |
| Passwords in plaintext | Documented — production uses bcrypt + JWT |
| No real payments | Checkout UI is complete; payment gateway is a drop-in |
| Single file becomes unwieldy | Phase 2 splits into proper frontend + API |
| Demo data wiped on localStorage clear | Seed function is idempotent — runs on every boot if empty |
