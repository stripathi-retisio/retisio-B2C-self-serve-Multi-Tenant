# IMPLEMENTATION.md — How It's Built

## Approver - Rajiv/Tony
## Build - Sameer

## Overview

Retisio Portal lives entirely in `retisio-portal.html` — roughly 1,600 lines of HTML, CSS, and JavaScript with no build step, no dependencies, and no backend. This document walks through every major system in the codebase.

---

## File Structure (inside retisio-portal.html)

```
retisio-portal.html
│
├── <style>                     CSS — design tokens, components, layouts
│
└── <script>
    ├── DB                      Mock database (localStorage abstraction)
    ├── seedData()              Pre-populates 3 tenants with products, orders, provisioning stages
    ├── Router                  SPA routing engine
    ├── toast()                 Notification system
    ├── I (icons)               SVG icon map
    ├── Helper functions        statusBadge, tenantStatusLabel, tenantStatusBadgeClass, hexToRgb
    │
    ├── renderLanding()         Landing page (3 CTAs + features + packages)
    ├── openDemoSelector()      Guided demo modal with 3 route options
    │
    ├── renderPackage()         Package selection screen
    │   └── attachPackage()     Wires "Continue" button
    │   └── selectPackage()     Highlights selected card
    │
    ├── renderSignup()          Onboarding wizard shell
    │   └── renderSignupStep()  Steps 1–5 (rendered into #signupCard)
    │   └── attachSignup()      Resets state and renders step 1
    │
    ├── renderLogin()           Retailer login + quick-login buttons
    │   └── attachLogin()       Wires submit and quickLogin
    │
    ├── renderDashboard()       Tenant workspace shell + 8-tab sidebar
    │   ├── renderOverviewTab()         Setup checklist, stats, revenue chart
    │   ├── renderProductsTab()         Catalog grid
    │   ├── renderPricingTab()          Price table per catalog item
    │   ├── renderInventoryTab()        Stock levels table
    │   ├── renderOrdersTab()           Order table with filters
    │   │   └── renderOrderRows()       Null-safe order row renderer
    │   ├── renderUsersTab()            Demo team table
    │   ├── renderStorefrontPreviewTab() Inline storefront preview + channel info
    │   └── renderSettingsTab()         Tenant info, theme, toggles, deprovision
    │
    ├── renderProductCard()     Single catalog card (used in grid + modal refresh)
    ├── showProductModal()      Add/edit catalog item modal
    │
    ├── renderStore()           Starter storefront (public, per-tenant)
    │   ├── attachStore()       Resets cart, renders cart sidebar
    │   ├── addToCart()
    │   ├── renderCart()
    │   ├── toggleCart()
    │   ├── checkout()
    │   └── showCheckoutModal()
    │
    ├── renderAdminLogin()      Operator login
    │   └── attachAdminLogin()
    │
    ├── renderAdminDashboard()  Operator Portal shell + 5-tab sidebar
    │   ├── renderAdminOverview()       Stats, tier distribution, tenant list
    │   ├── renderAdminTenants()        Full tenant table with actions
    │   ├── renderAdminProvisioning()   7-stage provisioning matrix
    │   └── renderAdminOrders()         Cross-tenant order ledger (null-safe)
    │
    ├── Operator action fns     adminUpdateTenantStatus, adminPromoteToProduction,
    │                           adminSuspendTenant, adminActivateTenant, adminArchiveTenant,
    │                           adminDeleteTenant
    │
    └── renderSchema()          Full DB schema documentation page
```

---

## 1. The DB Layer

The `DB` object is a complete mock database abstraction over `localStorage`. It mirrors what a real ORM or query layer would expose.

```js
const DB = {
  get(key)      // JSON.parse from localStorage (namespaced with 'sf_')
  set(key, val) // JSON.stringify to localStorage
  del(key)      // Remove key

  // Tenants
  getTenants()
  getTenant(id)
  getTenantBySlug(slug)
  addTenant(tenant)          // Auto-assigns id, status, provisioningStages, theme, settings
  updateTenant(id, updates)  // Shallow merge
  deleteTenant(id)           // Cascades to products + orders

  // Products
  getProducts(tenantId)
  addProduct(tenantId, product)
  updateProduct(tenantId, productId, updates)
  deleteProduct(tenantId, productId)

  // Orders
  getOrders(tenantId)
  addOrder(tenantId, order)      // Also increments tenant.revenue
  updateOrder(tenantId, id, updates)

  // Auth
  getAdmin()     // Returns hardcoded operator credentials
  getSession()   // → { role, tenantId?, email, name } | null
  setSession(s)
  clearSession()
}
```

**Key design decisions:**
- All data namespaced under `sf_` to avoid localStorage collisions
- `deleteTenant()` cascades — removes all related products and orders
- `addOrder()` automatically increments `tenant.revenue` (denormalized for fast reads)
- Tenant products and orders stored under separate keys: `sf_products_{id}` and `sf_orders_{id}`
- `addTenant()` sets `provisioningStages` with first 3 stages pre-completed (tenant created, channel provisioned, storefront provisioned)

---

## 2. Seed Data

`seedData()` runs once on boot. It checks if any tenants exist and exits early if so — idempotent and safe to call on every page load.

```js
function seedData() {
  if (DB.getTenants().length > 0) return;  // Already seeded

  const t1 = DB.addTenant({ ..., tenantStatus: 'live', environment: 'production',
    provisioningStages: { ...all true } });
  const t2 = DB.addTenant({ ..., tenantStatus: 'sandbox_ready', environment: 'sandbox',
    provisioningStages: { tenantCreated: true, channelProvisioned: true, storefrontProvisioned: true, ...rest false } });
  const t3 = DB.addTenant({ ..., tenantStatus: 'ready_for_production', environment: 'sandbox',
    provisioningStages: { ...first 6 true, productionPromoted: false } });

  // Add products and orders for each...
}
```

The three seed tenants demonstrate three different lifecycle stages simultaneously, making the Operator Portal immediately useful on first load.

---

## 3. The Router

A minimal SPA router — no library, no URL hash.

```js
const Router = {
  current: '',
  params: {},

  go(path, params = {}) {
    this.current = path;
    this.params = params;
    this.render();
  },

  render() {
    // 1. Clear any open modals
    // 2. Check session for protected routes
    // 3. Call the correct render function
  }
}
```

**Route protection:**
- `/dashboard` requires `session.role === 'merchant'`
- `/admin-dashboard` requires `session.role === 'admin'`
- Violations redirect to the appropriate login page

---

## 4. Onboarding Wizard

The wizard is managed by two module-level variables:

```js
let signupStep = 1;   // 1–5
let signupData = { plan: selectedPackage };  // Accumulated across steps
```

Each step renders into `#signupCard`. `renderSignupStep()` re-renders the card and step indicator on every step change.

**Step breakdown:**

| Step | Content | Key validation |
|---|---|---|
| 1 | Account (name, email, password) | Email uniqueness check |
| 2 | Tenant setup (name, slug, category, business model) | Slug format + uniqueness check |
| 3 | Package + environment confirmation | Editable plan dropdown |
| 4 | Starter storefront theme (6 options) | None — visual choice |
| 5 | Sandbox Ready state (checklist + next steps) | None — completion screen |

On step 5 completion, `DB.addTenant()` is called with `tenantStatus: 'sandbox_ready'`, `environment: 'sandbox'`, and pre-set `provisioningStages`. Session is set and the retailer can navigate to their workspace or preview their storefront.

---

## 5. Tenant Workspace Tabs

The workspace has 8 tabs, all rendered by `renderDashboard()` via a tab parameter.

**Overview tab** is the most information-dense:
- Page header shows lifecycle status badge, environment badge, business model badge
- "Request Promotion to Production" button (shown when not yet Live/Production)
- 4 stat cards (revenue, orders, catalog items, pending orders)
- Setup checklist card tracking 8 milestones
- Revenue bar chart (static demo data)
- Recent orders and catalog preview cards

**Pricing tab** renders a table from the same `products` array, showing price, comparePrice, and computed discount percentage. Edit links open the catalog item modal.

**Inventory tab** renders stock levels with a computed status:
- > 20 units → In Stock (green)
- 6–20 units → Low Stock (orange)
- ≤ 5 units → Critical (red)

**Users tab** shows a demo team table (3 hardcoded users: Admin, Editor, Viewer). Phase 2 will wire this to a real users table.

**Storefront Preview tab** renders an inline mini-preview of the B2C channel without leaving the dashboard, plus a metadata card with storefront URL, brand color, business model, and published item count.

---

## 6. Operator Portal — Provisioning Tab

The Provisioning tab is a matrix table:

```
Tenant | Env | Created | Channel | Storefront | Catalog | Pricing | Inventory | Promoted | Actions
```

Each stage cell shows `✓` (green badge) or `—` (gray badge) based on the tenant's `provisioningStages` object. This gives operators an at-a-glance view of where every tenant is in the setup process.

The "Promote to Production" button in this tab (and the Tenants tab) calls `adminPromoteToProduction()`:

```js
function adminPromoteToProduction(id) {
  if (!confirm('Promote this tenant to Production?')) return;
  DB.updateTenant(id, {
    environment: 'production',
    tenantStatus: 'live',
    status: 'active',
    provisioningStages: {
      ...DB.getTenant(id)?.provisioningStages,
      productionPromoted: true
    }
  });
  toast('🚀 Tenant promoted to Production!');
  Router.go('admin-dashboard', { tab: 'merchants' });
}
```

---

## 7. Operator Orders — Null Safety

The Operator Orders tab was previously broken due to unguarded access on potentially null order objects. The fix:

```js
function renderAdminOrders(tenants, allOrders) {
  const safe = (allOrders || []).filter(Boolean);  // Remove any null/undefined
  const allWithStore = safe.map(o => ({
    ...o,
    storeName: (tenants || []).find(t => t && t.id === o.tenantId)?.name || o.tenantId || 'Unknown'
  }));
  // ...render rows with o.id||'—', o.total||0, o.createdAt?...:'—', (o.items||[])
}
```

`renderAdminDashboard()` also wraps `DB.getOrders()` in try/catch:
```js
const allOrders = tenants.flatMap(t => {
  try { return DB.getOrders(t.id) || []; } catch { return []; }
});
```

---

## 8. Cart System

Cart state is a module-level array — intentionally outside the DB layer since it's ephemeral session state:

```js
let cart = [];  // [{ ...product, qty: number }]
```

Key operations: `addToCart`, `updateQty`, `removeFromCart`, `renderCart`, `updateCartCount`, `toggleCart`, `checkout`, `showCheckoutModal`.

On checkout completion, `DB.addOrder()` persists the order, the cart is cleared, and the order immediately appears in the tenant's Orders tab.

---

## 9. Modals

Modals are created dynamically and appended to `document.body`:

```js
const overlay = document.createElement('div');
overlay.className = 'modal-overlay';
overlay.innerHTML = `<div class="modal">...</div>`;
document.body.appendChild(overlay);

// Click outside to close
overlay.addEventListener('click', e => {
  if (e.target === overlay) overlay.remove();
});
```

Used for: product add/edit, checkout, order confirmation, guided demo selector.

The router clears all open modals before rendering a new route:
```js
document.querySelectorAll('.modal-overlay').forEach(m => m.remove());
```

---

## 10. Helper Functions

```js
statusBadge(status)
// Returns CSS class for order status badges
// pending→orange, processing→blue, shipped→blue, delivered→green, cancelled→red

tenantStatusLabel(s)
// Returns human-readable label for tenant lifecycle status
// 'sandbox_ready' → 'Sandbox Ready', 'ready_for_production' → 'Ready for Production', etc.

tenantStatusBadgeClass(s)
// Returns CSS class for tenant status badges
// requested→gray, sandbox_ready→blue, setup_in_progress→orange,
// ready_for_production→brand, live→green, suspended→red, archived→purple

hexToRgb(hex)
// Decomposes hex color into 'r,g,b' string for rgba() usage in storefront gradients
```

---

## 11. CSS Architecture

All styles use CSS custom properties defined on `:root`:

```css
:root {
  --brand: #FF5C28;      /* Primary orange */
  --bg: #0B0B0F;         /* Page background */
  --surface: #141418;    /* Card background */
  --surface2: #1C1C23;   /* Elevated surface */
  --text: #F0EFE9;       /* Primary text */
  --text2: #9997A0;      /* Secondary text */
  --border: rgba(255,255,255,0.08);
  --radius: 12px;
  --font-head: 'Bricolage Grotesque', sans-serif;
  --font-body: 'DM Sans', sans-serif;
  --font-mono: 'DM Mono', monospace;
}
```

**New CSS classes added in v2:**
- `.badge-purple` — for Archived tenant status
- `.btn-success` — green action button (Activate, Promote)
- `.checklist` / `.check-item` / `.check-dot` — setup checklist component
- `.package-grid` / `.package-card` — package selection cards
- `.demo-grid` / `.demo-card` — guided demo selector
- `.placeholder-section` — dashed placeholder for UI-only tabs (Pricing, Inventory, Users)

---

## 12. Known Limitations (v1 Prototype)

| Limitation | Notes |
|---|---|
| Passwords in plaintext | Production: bcrypt |
| No real auth | localStorage session has no expiry; production: JWT + httpOnly cookie |
| Pricing/Inventory/Users are UI-only | Phase 2 backend needed for real logic |
| No image upload | Emoji placeholder — Phase 2: Cloudinary |
| No real-time updates | Orders don't push to dashboard without navigation |
| Single file | Maintainability degrades past ~2000 lines; Phase 2 splits into frontend + API |

---

## 13. Migration Path to Production

1. Extract frontend into a proper SPA (React, Next.js, or SvelteKit)
2. Build REST API (Node + Express or Supabase)
3. Apply the schema from [DESIGN.md](DESIGN.md) to PostgreSQL
4. Enable RLS on all tenant-scoped tables
5. Replace `DB.*` calls with `fetch('/api/...')` equivalents
6. Replace localStorage session with JWT in httpOnly cookie
7. Add Razorpay/Stripe as payment provider
8. Add Cloudinary/S3 for product image uploads
9. Add Resend/SendGrid for transactional email
10. Set up DNS-level subdomain routing per tenant (`slug.retisio.io`)

The data model, lifecycle logic, provisioning stages, and all business rules are already correct in the prototype — the migration is primarily plumbing.
