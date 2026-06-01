# Team Development Workflow — Beserk Shopify Theme

## Overview

```
feature/xxx  →  PR review  →  staging branch  →  Shopify staging  →  test  →  main branch  →  Shopify live
```

## Branch Structure

| Branch | Purpose | Shopify Theme |
|--------|---------|---------------|
| `main` | Production-ready code | Live theme (beserk.com.au) |
| `staging` | Tested code awaiting QA | Staging theme (ID: 140520685702) |
| `feature/*` | Individual developer features | Local dev only |
| `fix/*` | Bug fix branches | Local dev only |

---

## Developer Workflow

### Step 1 — One-time clone setup
```bash
git clone https://github.com/taha867/Beserk-shopify.git
cd Beserk-shopify
git checkout staging
```

### Step 2 — Start every new feature from staging
Always branch off `staging`, never off `main`:
```bash
git checkout staging
git pull origin staging
git checkout -b feature/your-feature-name
```

**Branch naming:**
- `feature/cart-drawer-redesign`
- `feature/product-card-badges`
- `feature/header-mobile-fix`
- `fix/checkout-button-color`
- `fix/announcement-bar-typo`

### Step 3 — Make your changes
Edit the relevant Liquid/CSS/JS files:
```
sections/     → page sections
snippets/     → reusable partials
blocks/       → inline content blocks
assets/       → theme.js and theme.css
templates/    → page templates
locales/      → text/translation strings
```

### Step 4 — Commit and push your branch
```bash
git add sections/your-file.liquid snippets/your-file.liquid
git commit -m "Brief description of what changed and why"
git push origin feature/your-feature-name
```

### Step 5 — Open a Pull Request on GitHub
1. Go to https://github.com/taha867/Beserk-shopify
2. Click **"Compare & pull request"** on your branch
3. Set base branch to **`staging`** (never `main`)
4. Fill in:
   - **Title:** short description of the feature/fix
   - **Description:** what changed, why, and any notes for the reviewer
   - **Screenshots:** required for any visual changes
5. Request review from **@taha867**
6. Wait for approval — do not merge your own PR

---

## Team Lead Workflow (Muhammad Taha)

### Step 1 — Review PR on GitHub
- Check the code diff
- Leave comments on anything that needs changes
- Request changes or approve

### Step 2 — Merge approved PR into staging
- Click **"Merge pull request"** on GitHub
- Select **"Squash and merge"** for clean history

### Step 3 — Pull and push to Shopify staging
```bash
git checkout staging
git pull origin staging
shopify theme push --store=beserk.myshopify.com --theme=140520685702
```

### Step 4 — Test on staging
Preview the staging theme at:
```
https://beserk.myshopify.com/?preview_theme_id=140520685702
```
Test all affected pages. Check mobile and desktop.

### Step 5 — Deploy to production (after staging is approved)
Once all features are tested and approved on staging:
```bash
git checkout main
git merge staging
git push origin main
shopify theme push --store=beserk.myshopify.com
```
This updates the live theme at beserk.com.au.

---

## Full Flow Diagram

```
Developer
    │
    ├─ git checkout staging
    ├─ git pull origin staging
    ├─ git checkout -b feature/xxx
    ├─ make changes
    ├─ git push origin feature/xxx
    └─ open PR → into staging
                    │
              Team Lead reviews
                    │
              ┌─────┴─────┐
           changes?      approved?
              │               │
        request fixes    merge into staging
                              │
                   shopify theme push (staging ID)
                              │
                        test on staging
                              │
                   ┌──────────┴──────────┐
                 issues?            all good?
                   │                     │
             fix & re-test         merge staging → main
                                         │
                                shopify theme push (live)
```

---

## Rules

- Never push directly to `main` or `staging` — always use a PR
- Never open a PR into `main` — only into `staging`
- Always pull latest `staging` before creating a new branch
- One feature per branch — keep PRs small and focused
- Resolve all PR review comments before merging
- Only the team lead pushes to Shopify (staging or live)
- Never share GitHub tokens or credentials in chat or code

---

## Quick Reference

| Task | Command |
|------|---------|
| Start new feature | `git checkout -b feature/name` |
| Save changes | `git add <files> && git commit -m "message"` |
| Push branch | `git push origin feature/name` |
| Update local staging | `git checkout staging && git pull origin staging` |
| Push to Shopify staging | `shopify theme push --store=beserk.myshopify.com --theme=140520685702` |
| Push to Shopify live | `shopify theme push --store=beserk.myshopify.com` |
| Preview staging | `shopify theme dev --store=beserk.myshopify.com --theme=140520685702` |
