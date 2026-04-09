# ShopForge 🔥

> **A self-serve, multi-tenant ecommerce platform** — Built from scratch in a single HTML file.

[![Live Demo](https://img.shields.io/badge/Live%20Demo-Visit-FF5C28?style=for-the-badge)](https://stripathi-retisio.github.io/shopforge-demo)
[![GitHub Stars](https://img.shields.io/github/stars/stripathi-retisio/shopforge-demo?style=for-the-badge)](https://github.com/stripathi-retisio/shopforge-demo/stargazers)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)](LICENSE)
[![Made with](https://img.shields.io/badge/Built%20with-HTML%2FCSS%2FJS-blue?style=for-the-badge)](#)

---

## What is ShopForge?

ShopForge is a fully functional, multi-tenant ecommerce platform prototype. Every merchant who signs up gets their own isolated store — their own storefront, products, orders, branding, and settings — all within a single application.

It demonstrates the complete architecture of a Shopify-style SaaS product:

- **Merchants** self-onboard, configure their store, and manage it from a dashboard
- **Customers** browse the public storefront and place orders with a cart + checkout flow
- **Platform admins** oversee all merchants, orders, and revenue from a super-dashboard

No frameworks. No build tools. No backend (yet). Just the platform, raw.

---

## Screenshots

| Landing | Merchant Dashboard | Storefront | Admin |
|---|---|---|---|
| Hero + Pricing | Products, Orders, Analytics | Cart + Checkout | Merchant oversight |

---

## Features

### Merchant Self-Serve
- **4-step onboarding** — Account → Store Info → Theme Picker → Launch
- **Product management** — Add, edit, delete, publish/unpublish products
- **Order management** — View all orders, filter by status, update fulfillment status
- **Store settings** — Rename store, change brand color, toggle features, danger zone
- **Storefront preview** — Live public-facing store with custom branding

### Public Storefront
- Per-tenant branding (colors, store name, category)
- Product catalog with emoji icons, pricing, compare prices
- Add-to-cart, quantity controls, cart sidebar
- Checkout modal with customer details + payment method
- Order confirmation

### Admin Super-Dashboard
- Platform-wide stats (merchants, revenue, orders)
- Plan distribution breakdown
- Per-merchant controls — suspend/activate, delete, view store
- Full cross-tenant order ledger

### Data Layer
- `localStorage`-backed mock database (zero backend required to run)
- Full DB schema documented for PostgreSQL migration
- Row-level-security-ready schema design
- Seed data included (3 merchants, products, orders pre-loaded)

---

## Quick Start

```bash
# Clone the repo
git clone https://github.com/stripathi-retisio/shopforge-demo.git
cd shopforge-demo

# Open in browser — no build step needed
open index.html
```

Or just open `index.html` directly in any modern browser.

---

## Demo Credentials

### Merchants
| Store | Email | Password |
|---|---|---|
| Pixel & Thread | `hello@pixelthread.com` | `demo123` |
| Green Roots | `hi@greenroots.co` | `demo123` |
| Urban Luxe | `contact@urbanluxe.in` | `demo123` |

### Platform Admin
| Email | Password |
|---|---|
| `admin@shopforge.io` | `admin123` |

> **Tip:** The login page shows quick-login buttons for all demo merchants.

---

## Project Structure

```
shopforge-demo/
├── index.html        # Entire application (single file)
├── README.md             # You are here
├── PLAN.md               # Project goals, scope, and roadmap
├── DESIGN.md             # Architecture, schema, and design decisions
├── IMPLEMENTATION.md     # How it's built — code walkthrough
└── CONTRIBUTING.md       # How to contribute
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| UI | Vanilla HTML5 + CSS3 (custom properties, grid, flexbox) |
| Logic | Vanilla JavaScript (ES6+) |
| Routing | Custom hash-free SPA router |
| State | In-memory JS + `localStorage` persistence |
| Fonts | Google Fonts (Bricolage Grotesque + DM Sans + DM Mono) |
| Icons | Inline SVG |
| DB (planned) | PostgreSQL with Row-Level Security |

---

## Roadmap

See [PLAN.md](PLAN.md) for the full roadmap.

**Near-term:**
- [ ] Backend API (Node.js / Express or Supabase)
- [ ] Real PostgreSQL database with RLS
- [ ] Image uploads via Cloudinary / S3
- [ ] Payment integration (Razorpay / Stripe)
- [ ] Custom domain support per tenant
- [ ] Email notifications (order confirmation, status updates)

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) — contributions are welcome!

---

## License

MIT © [Surabhi Tripathi](https://github.com/stripathi-retisio)
