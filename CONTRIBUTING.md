# Team Development Workflow

## Branch Structure

```
main        →  Live Shopify theme (production)
staging     →  Staging Shopify theme (ID: 140520685702)
feature/*   →  Developer feature branches
```

## Developer Workflow (all 4 devs follow this)

### 1. One-time setup
```bash
git clone <repo-url>
cd shopify-project
git checkout staging
```

### 2. Start a new feature
Always branch off `staging`, never off `main`:
```bash
git checkout staging
git pull origin staging
git checkout -b feature/your-feature-name
```

**Branch naming examples:**
- `feature/cart-drawer-redesign`
- `feature/product-card-badges`
- `feature/header-mobile-fix`
- `fix/checkout-button-color`

### 3. Make changes & push
```bash
git add sections/your-file.liquid snippets/your-file.liquid
git commit -m "Short description of what and why"
git push origin feature/your-feature-name
```

### 4. Open a Pull Request
- Go to GitHub → open PR from `feature/your-feature-name` → **into `staging`**
- Describe what changed and why
- Add screenshots if it's a visual change
- Request review from team lead (@m-taha)

---

## Team Lead Workflow (Muhammad Taha)

### Review & merge to staging
1. Review the PR on GitHub
2. Approve and merge into `staging`
3. Push `staging` to Shopify staging theme:
```bash
git checkout staging
git pull origin staging
shopify theme push --store=beserk.myshopify.com --theme=140520685702
```
4. Test at: https://beserk.myshopify.com/?preview_theme_id=140520685702

### Deploy staging → production (when tested & approved)
```bash
git checkout main
git merge staging
git push origin main
shopify theme push --store=beserk.myshopify.com
```

---

## Rules

- Never push directly to `main` or `staging`
- Never open a PR into `main` — only into `staging`
- Always pull latest `staging` before creating a new feature branch
- One feature per branch — keep PRs small and focused
- The team lead is the only one who pushes to Shopify (staging or live)
