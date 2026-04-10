# CONTRIBUTING.md — Contributing to Retisio Portal

Thank you for your interest in contributing. Retisio Portal is a learning-friendly prototype and welcomes contributions of all kinds — bug fixes, new features, UI improvements, and documentation.

---

## Ways to Contribute

- 🐛 **Bug reports** — something broken? Open an issue
- 💡 **Feature ideas** — have a suggestion? Start a discussion
- 🔧 **Code** — fix a bug or build something from the ideas list below
- 📝 **Documentation** — improve any of the docs in this repo
- 🎨 **Design** — UI improvements, accessibility, new themes

---

## Getting Started

### 1. Fork and clone

```bash
# Fork via GitHub UI, then:
git clone https://github.com/YOUR_USERNAME/retisio-demo.git
cd retisio-demo
```

### 2. Open the project

No build step needed:

```bash
open retisio-portal.html
# or drag into your browser
```

For a better dev experience, serve locally to avoid localStorage quirks:

```bash
# Python
python3 -m http.server 8080

# Node
npx serve .

# VS Code
# Install "Live Server" extension → "Go Live"
```

Then visit `http://localhost:8080/retisio-portal.html`.

### 3. Understand the codebase

Read [IMPLEMENTATION.md](IMPLEMENTATION.md) before making changes. It walks through every system in the file — the DB layer, router, onboarding wizard, tenant workspace tabs, operator portal, and more.

---

## Code Style

- Use `const` / `let` — never `var`
- Arrow functions for callbacks
- Template literals for HTML strings
- Descriptive names — `tenantId` not `tid`, `tenantStatus` not `status`
- For new views: follow the `render*()` pattern — pure functions returning HTML strings
- For new state changes: go through `DB.*` methods, not direct `localStorage` access
- CSS: use existing custom properties (`--brand`, `--surface`, etc.) — don't hardcode values inline

---

## Pull Request Process

1. **Branch from `main`:**
   ```bash
   git checkout -b feat/your-feature-name
   ```

2. **Keep changes focused.** One PR per feature or fix.

3. **Test manually before opening a PR:**
   - [ ] Retailer signup flow (all 5 steps)
   - [ ] Tenant workspace — all 8 tabs render without errors
   - [ ] Catalog add/edit/delete/publish works
   - [ ] Storefront loads, cart and checkout work, order appears in dashboard
   - [ ] Operator Portal — Overview, Tenants, Provisioning, All Orders all load
   - [ ] Operator promote / suspend / archive actions work
   - [ ] Demo selector modal opens and all 3 paths work
   - [ ] No `console.error` in browser DevTools
   - [ ] Works in Chrome and Firefox

4. **Open a PR** with:
   - A clear title and description
   - Screenshots or screen recording if UI changed
   - Reference any related issue with `Closes #123`

---

## Good First Issues

Look for issues tagged `good first issue` on the [issue tracker](https://github.com/stripathi-retisio/retisio-demo/issues).

Ideas for contributions:

| Idea | Difficulty |
|---|---|
| Add a "Copy storefront URL" button in Storefront Preview tab | Easy |
| Add a filter/search bar to the Catalog tab | Easy |
| Add a total catalog value stat to the Pricing tab | Easy |
| Make the setup checklist items clickable (navigate to that tab) | Easy |
| Add a "Low stock" alert badge in the sidebar Inventory nav item | Easy |
| Add a stock adjustment input directly in the Inventory table | Medium |
| Add a download CSV button for orders | Medium |
| Add a coupon/discount code field in checkout | Medium |
| Add an "Archived" filter to the Operator Tenants table | Medium |
| Add per-tenant revenue to the Operator Overview chart | Medium |
| Extract all CSS into a separate `<style>` reference and JS into `<script src="">` | Hard |
| Add a proper staging environment option in the onboarding wizard | Hard |
| Multi-channel per tenant (Phase 2 foundation) | Hard |

---

## Reporting Bugs

Please include:

1. **What you expected**
2. **What happened instead**
3. **Steps to reproduce** (be specific)
4. **Browser + OS** (e.g., Chrome 124 on macOS 14)
5. **Any console errors** — open DevTools (F12) → Console tab and paste the full error

For localStorage-related issues, also try: DevTools → Application → Storage → Local Storage → clear all `sf_` keys and reload.

---

## Project Context

Before contributing, it helps to understand what this project is and is not:

**This IS:**
- A self-serve retailer onboarding and control plane prototype
- A demonstration of multi-tenant architecture and lifecycle management
- A Retisio Phase 1 product story told through a working UI

**This is NOT:**
- A generic ecommerce SaaS demo
- A Shopify clone
- A production-ready application

Keep contributions aligned with the Retisio positioning. Features that make it look more like a generic store builder are out of scope.

---

## Questions?

Open a [GitHub Discussion](https://github.com/stripathi-retisio/retisio-demo/discussions).

---

## Code of Conduct

Be kind. Be constructive. Assume good intent. This project follows the [Contributor Covenant v2.1](https://www.contributor-covenant.org/version/2/1/code_of_conduct/).
