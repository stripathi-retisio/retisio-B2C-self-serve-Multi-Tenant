# IMPLEMENTATION.md — How It's Built

## Overview

ShopForge lives entirely in `shopforge.html` — ~1,200 lines of HTML, CSS, and JavaScript with no build step, no dependencies, and no backend. This document walks through every major system in the codebase.

---

## File Structure (inside shopforge.html)

```
shopforge.html
│
├── <style>                  CSS — design tokens, components, layouts
│
└── <script>
    ├── DB                   Mock database (localStorage abstraction)
    ├── seedData()           Pre-populates 3 merchants, products, orders
    ├── Router               SPA routing engine
    ├── toast()              Toast notification system
    ├── I (icons)            SVG icon map
    │
    ├── renderLanding()      Landing page + attachLanding()
    ├── renderSignup()       Onboarding shell + attachSignup()
    │   └── renderSignupStep()  Step 1–4 wizard logic
    ├── renderLogin()        Merchant login + attachLogin()
    │
    ├── renderDashboard()    Dashboard shell + attachDashboard()
    │   ├── renderOverviewTab()
    │   ├── renderProductsTab()
    │   │   └── renderProductCard()
    │   ├── renderOrdersTab()
    │   │   └── renderOrderRows()
    │   └── renderSettingsTab()
    │
    ├── renderStore()        Public storefront + attachStore()
    │   ├── addToCart()
    │   ├── renderCart()
    │   ├── toggleCart()
    │   ├── checkout()
    │   └── showCheckoutModal()
    │
    ├── renderAdminLogin()   Admin auth + attachAdminLogin()
    ├── renderAdminDashboard() Super-admin + attachAdminDashboard()
    │   ├── renderAdminOverview()
    │   ├── renderAdminMerchants()
    │   └── renderAdminOrders()
    │
    └── renderSchema()       DB schema documentation page
```

---

## 1. The DB Layer

The `DB` object is a complete mock database abstraction over `localStorage`. It mimics what a real ORM or query layer would expose.

```js
const DB = {
  get(key)          // JSON.parse from localStorage
  set(key, val)     // JSON.stringify to localStorage
  del(key)          // Remove key

  // Tenants
  getTenants()                    // → Tenant[]
  getTenant(id)                   // → Tenant | null
  getTenantBySlug(slug)           // → Tenant | null
  addTenant(tenant)               // → Tenant (auto-assigns id, status, plan)
  updateTenant(id, updates)       // → Tenant | null
  deleteTenant(id)                // cascades to products + orders

  // Products
  getProducts(tenantId)           // → Product[]
  addProduct(tenantId, product)   // → Product
  updateProduct(tenantId, id, updates)
  deleteProduct(tenantId, id)

  // Orders
  getOrders(tenantId)             // → Order[]
  addOrder(tenantId, order)       // → Order (also increments tenant.revenue)
  updateOrder(tenantId, id, updates)

  // Auth
  getAdmin()                      // hardcoded admin credentials
  getSession()                    // → Session | null
  setSession(session)
  clearSession()
}
```

**Key design decisions:**
- All data is namespaced under `sf_` to avoid localStorage collisions
- `deleteTenant()` cascades — deletes all related products and orders
- `addOrder()` automatically increments `tenant.revenue` (denormalized for fast reads)
- Tenant products and orders are stored under separate keys: `sf_products_{tenantId}` and `sf_orders_{tenantId}` — this mirrors a sharded or RLS-enforced Postgres setup

---

## 2. The Router

A minimal SPA router — no library, no URL hash required.

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
    // 3. Call the correct render + attach functions
  }
}
```

**Route protection:** The router checks `DB.getSession()` before rendering protected routes:
- `/dashboard` requires `session.role === 'merchant'`
- `/admin-dashboard` requires `session.role === 'admin'`
- Any violation redirects to the appropriate login page

**Usage throughout the app:**
```js
Router.go('landing')
Router.go('dashboard', { tab: 'products' })
Router.go('store', { slug: 'pixel-thread' })
Router.go('admin-dashboard', { tab: 'merchants' })
```

---

## 3. Rendering Pattern

Every view follows the same two-function pattern:

```js
function renderX() {
  return `<html string>`   // Pure — returns HTML, touches nothing
}

function attachX() {
  // Imperative — binds event listeners, initialises sub-components
  document.getElementById('...').onclick = () => { ... }
}
```

The router calls both in sequence:
```js
app.innerHTML = renderDashboard(tenant, tab);
attachDashboard(tenant, tab);
```

This separation keeps rendering predictable and re-renderable — call `render` again and you get a fresh, consistent DOM.

**Inline event handlers** are used for dynamic list items (product cards, order rows) where elements don't exist at `attach` time:
```html
<button onclick="deleteProduct('${tenantId}','${p.id}')">Delete</button>
```

---

## 4. The Signup Wizard

The onboarding flow is a 4-step wizard managed by two module-level variables:

```js
let signupStep = 1;   // Current step (1–4)
let signupData = {};  // Accumulated form data across steps
```

Each step renders into `#signupCard` via `renderSignupStep()`. Validation happens before advancing:

```
Step 1 (Account)   → validate name, email uniqueness, password length
Step 2 (Store)     → validate store name, slug uniqueness, format check
Step 3 (Theme)     → theme index stored in signupData
Step 4 (Done)      → tenant created, session set, celebration screen
```

The step indicator re-renders on every step transition, applying `.active` and `.done` classes.

---

## 5. The Cart System

Cart state is a module-level array — intentionally kept outside the DB layer since it's ephemeral session state, not persisted data.

```js
let cart = [];  // [{ ...product, qty: number }]
```

**Key operations:**
```js
addToCart(product)           // Merge or push
updateQty(productId, delta)  // Increment/decrement, remove if qty <= 0
removeFromCart(productId)
renderCart(tenant)           // Redraws cart sidebar
updateCartCount()            // Updates badge on cart button
```

**Checkout flow:**
1. Customer fills checkout modal (name, email, address, payment method)
2. `DB.addOrder(tenantId, {...})` persists the order
3. Cart is cleared, sidebar closes, confirmation modal shown
4. Order immediately appears in the merchant's dashboard

---

## 6. Modals

Modals are created dynamically and appended to `document.body` — they're not part of any view's HTML. This lets them work from any page without routing.

```js
function showProductModal(tenantId, product) {
  const overlay = document.createElement('div');
  overlay.className = 'modal-overlay';
  overlay.innerHTML = `<div class="modal">...</div>`;
  document.body.appendChild(overlay);

  // Click outside to close
  overlay.addEventListener('click', e => {
    if (e.target === overlay) overlay.remove();
  });

  // Save handler
  document.getElementById('pm_save').onclick = () => {
    // validate → save → remove modal → refresh UI
  };
}
```

The same pattern is used for the checkout modal and order confirmation.

---

## 7. Toast Notifications

Toasts are self-managing — they create, show, and destroy themselves:

```js
function toast(msg, type = 'success') {
  const el = document.createElement('div');
  el.className = `toast ${type}`;
  el.innerHTML = `<span>${icon}</span><span>${msg}</span>`;
  document.getElementById('toastContainer').appendChild(el);

  setTimeout(() => el.style.opacity = '0', 2500);   // Start fade
  setTimeout(() => el.remove(), 2800);               // Remove from DOM
}
```

Types: `success` (green left border), `error` (red), `info` (blue).

---

## 8. Seed Data

`seedData()` runs once on page load. It checks if any tenants exist and exits early if so — idempotent and safe to call on every boot.

```js
function seedData() {
  if (DB.getTenants().length > 0) return;  // Already seeded

  const t1 = DB.addTenant({ name: 'Pixel & Thread', ... });
  DB.addProduct(t1.id, { name: 'Minimal Tote Bag', price: 899, ... });
  // ...
  DB.addOrder(t1.id, { customerName: 'Riya Nair', total: 1798, ... });
}
```

Seeded data: 3 merchants × (3–4 products + 8 orders each) = ready-to-explore state on first open.

---

## 9. Storefront Theming

The storefront dynamically applies each merchant's theme at render time:

```js
function renderStore(tenant) {
  const color = tenant.theme?.primaryColor || '#FF5C28';
  const bg = tenant.theme?.bgGradient || '...';

  return `
    <style>:root { --store-brand: ${color}; }</style>
    <div style="background: ${bg}">...hero...</div>
    <button style="background: ${color}">Shop Now</button>
  `;
}
```

The `hexToRgb()` helper decomposes the brand color for use in `rgba()` gradient overlays — so the hero glow always matches the brand color, no matter what the merchant chose.

---

## 10. CSS Architecture

All styles use **CSS custom properties** (variables) defined on `:root`. This makes theming and consistency trivial:

```css
:root {
  --brand: #FF5C28;
  --bg: #0B0B0F;
  --surface: #141418;
  --surface2: #1C1C23;
  --text: #F0EFE9;
  --text2: #9997A0;
  --border: rgba(255,255,255,0.08);
  --radius: 12px;
}
```

Component classes (`.btn`, `.card`, `.badge`, `.form-input`, etc.) are defined once and composed — no inline style repetition. Utility classes handle spacing, flex, and typography.

---

## Known Limitations (v1)

| Limitation | Notes |
|---|---|
| Passwords stored in plaintext | Production must use bcrypt |
| No real auth (JWT/sessions) | localStorage session has no expiry |
| localStorage ~5MB cap | Sufficient for demo, not production |
| No image upload | Emoji used as placeholder |
| No real-time updates | Orders don't push to dashboard without page navigation |
| Single file | Maintainability degrades past ~2000 lines |

---

## Migration Path to Production

To turn this into a real product:

1. **Extract frontend** into a proper SPA (React, Vue, or SvelteKit)
2. **Build REST API** (Node + Express or Next.js API routes)
3. **Stand up PostgreSQL** — apply the schema from [DESIGN.md](DESIGN.md)
4. **Enable RLS** on all tenant-scoped tables
5. **Replace `DB.*` calls** with `fetch('/api/...')` equivalents
6. **Replace localStorage session** with JWT in httpOnly cookie
7. **Add Razorpay/Stripe** as the payment provider
8. **Add Cloudinary/S3** for product image uploads
9. **Set up Resend/SendGrid** for transactional email

The data model and all business logic are already correct — the migration is largely plumbing.
