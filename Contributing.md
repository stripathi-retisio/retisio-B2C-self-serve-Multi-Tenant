# CONTRIBUTING.md — Contributing to ShopForge

First off — thank you for considering contributing! ShopForge is a learning-friendly project and welcomes contributions of all kinds, from bug fixes to new features to documentation improvements.

---

## Ways to Contribute

- 🐛 **Bug reports** — something broken? File an issue
- 💡 **Feature suggestions** — have an idea? Open a discussion
- 🔧 **Code contributions** — fix a bug or build something new
- 📝 **Documentation** — improve any of the docs
- 🎨 **Design** — better UI, new themes, accessibility improvements

---

## Getting Started

### 1. Fork and clone

```bash
# Fork via GitHub UI, then:
git clone https://github.com/YOUR_USERNAME/shopforge-demo.git
cd shopforge-demo
```

### 2. Open the project

No build step needed — just open the file:

```bash
open index.html
# or drag it into your browser
```

For a better dev experience, use a local server to avoid localStorage quirks:

```bash
# Python
python3 -m http.server 8080

# Node (npx)
npx serve .

# VS Code
# Install "Live Server" extension and click "Go Live"
```

Then visit `http://localhost:8080/shopforge.html`.

### 3. Make your changes

All application code lives in `shopforge.html`. See [IMPLEMENTATION.md](IMPLEMENTATION.md) for a full walkthrough of the codebase.

---

## Contribution Guidelines

### Code Style

- Use `const` / `let` — never `var`
- Arrow functions for callbacks
- Template literals for HTML strings
- Descriptive variable names — `tenantId` not `tid`
- Follow the existing `render*() / attach*()` pattern for new views
- CSS: add new design tokens to `:root` rather than hardcoding values inline

### HTML / CSS

- New components should use existing CSS variables (`--brand`, `--surface`, etc.)
- Don't add new external font or library dependencies without discussion
- Maintain the existing dark-theme-first aesthetic

### Commit Messages

Use conventional commit format:

```
feat: add product variant support
fix: cart qty not updating on remove
docs: update DB schema in DESIGN.md
style: improve mobile layout for dashboard
refactor: extract cart logic into separate module
```

---

## Pull Request Process

1. **Create a branch** from `main`:
   ```bash
   git checkout -b feat/your-feature-name
   ```

2. **Make your changes** with clear, focused commits

3. **Test manually:**
   - [ ] Merchant signup flow works end-to-end
   - [ ] Products can be added, edited, deleted
   - [ ] Orders appear in dashboard after storefront checkout
   - [ ] Admin dashboard shows correct data
   - [ ] No `console.error` in the browser console
   - [ ] Works in Chrome and Firefox

4. **Open a PR** against `main` with:
   - A clear title and description of what changed and why
   - Screenshots or a screen recording if UI changed
   - Reference any related issue with `Closes #123`

5. A maintainer will review within a few days

---

## Good First Issues

Look for issues tagged `good first issue` on the [issue tracker](https://github.com/stripathi-retisio/shopforge-demo/issues). These are well-scoped and documented.

Some ideas to get started:

| Idea | Difficulty |
|---|---|
| Add product search/filter on the storefront | Easy |
| Add an order count badge to the sidebar | Easy |
| Add a "copy store link" button in settings | Easy |
| Dark/light mode toggle | Medium |
| Add product categories filter on storefront | Medium |
| CSV export of orders | Medium |
| Add a discount/coupon code field at checkout | Medium |
| Multi-image product support (carousel) | Hard |
| Extract app into separate HTML/CSS/JS files | Hard |

---

## Reporting Bugs

Please include:

1. **What you expected to happen**
2. **What actually happened**
3. **Steps to reproduce**
4. **Browser + OS** (e.g. Chrome 124 on macOS)
5. **Any console errors** (open DevTools → Console)

---

## Questions?

Open a [GitHub Discussion](https://github.com/stripathi-retisio/shopforge-demo/discussions) — happy to help.

---

## Code of Conduct

Be kind. Be constructive. Assume good intent. This project follows the [Contributor Covenant](https://www.contributor-covenant.org/version/2/1/code_of_conduct/).
