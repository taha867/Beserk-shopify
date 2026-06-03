# Team Development Workflow — Beserk Shopify Theme

## Overview

```
Jira task assigned  →  feature/KAN-XX branch  →  PR review  →  staging branch  →  Shopify staging  →  test  →  main branch  →  Shopify live
```

---

## Tools Used

| Tool | Purpose | Link |
|------|---------|-------|
| **Jira** | Task tracking — all issues live here | https://mmjawaan.atlassian.net/jira/software/projects/KAN/boards/2 |
| **GitHub** | Code repository and PR reviews | https://github.com/taha867/Beserk-shopify |
| **Shopify Staging** | Test environment before going live | Preview ID: 140520685702 |
| **Shopify Live** | Production store | beserk.com.au |

---

## Branch Structure

| Branch | Purpose | Shopify Theme |
|--------|---------|---------------|
| `main` | Production-ready code | Live theme (beserk.com.au) |
| `staging` | Tested code awaiting QA | Staging theme (ID: 140520685702) |
| `feature/KAN-XX-*` | Individual developer features | Local dev only |
| `fix/KAN-XX-*` | Bug fix branches | Local dev only |

---

## Branch Naming Convention

Every branch must include the Jira ticket number so work is traceable:

```
feature/KAN-XX-short-description
fix/KAN-XX-short-description
```

**Examples:**
| Jira Ticket | Branch Name |
|-------------|-------------|
| KAN-5 (Cart drawer not opening) | `fix/KAN-5-cart-drawer-not-opening` |
| KAN-18 (Cart quantity 340 requests) | `fix/KAN-18-cart-quantity-requests` |
| KAN-30 (Search overlay broken) | `fix/KAN-30-search-overlay` |
| KAN-31 (Mobile menu broken) | `fix/KAN-31-mobile-menu` |
| KAN-10 (Linux malicious script) | `fix/KAN-10-linux-malicious-script` |

**Rules:**
- Always lowercase, words separated by hyphens
- Keep it short — 3 to 5 words after the ticket number
- Use `fix/` for bug fixes, `feature/` for new functionality

---

## Commit Message Convention

Reference the Jira ticket in every commit message:

```
KAN-XX: short description of what changed and why
```

**Examples:**
```
KAN-5: open cart drawer after add to cart click
KAN-30: wire search overlay open on header icon click
KAN-18: replace full reload with fetch DOM swap on quantity change
```

---

## Developer Workflow

### Step 1 — One-time clone setup
```bash
git clone https://github.com/taha867/Beserk-shopify.git
cd Beserk-shopify
git checkout staging
```

### Step 2 — Pick your Jira task
1. Go to https://mmjawaan.atlassian.net/jira/software/projects/KAN/boards/2
2. Find the task assigned to you
3. Move it to **"In Progress"** on the board
4. Note the ticket number (e.g. `KAN-30`)

### Step 3 — Create your branch from staging
Always branch off `staging`, never off `main`:
```bash
git checkout staging
git pull origin staging
git checkout -b fix/KAN-30-search-overlay
```

### Step 4 — Make your changes
Edit the relevant Liquid/CSS/JS files:
```
sections/     → page sections
snippets/     → reusable partials
blocks/       → inline content blocks
assets/       → theme.js and theme.css
templates/    → page templates
locales/      → text/translation strings
```

### Step 5 — Commit and push your branch
```bash
git add sections/your-file.liquid snippets/your-file.liquid
git commit -m "KAN-30: wire search overlay open on header icon click"
git push origin fix/KAN-30-search-overlay
```

### Step 6 — Open a Pull Request on GitHub
1. Go to https://github.com/taha867/Beserk-shopify
2. Click **"Compare & pull request"** on your branch
3. Set base branch to **`staging`** (never `main`)
4. Fill in:
   - **Title:** `KAN-30: fix search overlay not opening`
   - **Description:** what changed, why, and how you tested it
   - **Screenshots:** required for any visual changes
5. Request review from **@taha867**
6. Move the Jira ticket to **"In Review"**
7. Wait for approval — do not merge your own PR

---

## Team Lead — Making Your Own Code Changes

If you (Muhammad Taha) make code changes yourself — fixing a bug, updating a section, or any direct edit — follow this flow instead of the developer PR flow.

### Step 1 — Always start from latest staging
```bash
git checkout staging
git pull origin staging
```

### Step 2 — Create your own branch (same convention)
```bash
git checkout -b fix/KAN-XX-short-description
```
Even as team lead, never edit `staging` or `main` directly — always branch off.

### Step 3 — Start local dev server
```bash
shopify theme dev --store=beserk.myshopify.com --theme=140520685702
```
Open `http://127.0.0.1:9292` — your changes appear instantly on save.

### Step 4 — Make and test your changes
Edit the relevant files, verify the fix works in the browser at `127.0.0.1:9292`.

### Step 5 — Commit and push
```bash
git add sections/your-file.liquid
git commit -m "KAN-XX: what you changed and why"
git push origin fix/KAN-XX-short-description
```

### Step 6 — Merge directly into staging (no PR review needed)
Since you are the team lead, you can merge your own branch directly:
```bash
git checkout staging
git merge fix/KAN-XX-short-description
git push origin staging
```

### Step 7 — Push to Shopify staging and test
```bash
shopify theme push --store=beserk.myshopify.com --theme=140520685702
```
Preview at: `https://beserk.myshopify.com/?preview_theme_id=140520685702`
Test the change on both desktop and mobile.

### Step 8 — Deploy to production when ready
```bash
git checkout main
git merge staging
git push origin main
shopify theme push --store=beserk.myshopify.com
```

### Step 9 — Update Jira
Move the Jira ticket to **Done**.

---

## Team Lead Workflow (Muhammad Taha)

### Step 1 — Review PR on GitHub
- Check the code diff
- Leave comments on anything that needs changes
- Approve or request changes

### Step 2 — Update Jira ticket
- If changes requested → move ticket back to **"In Progress"**
- If approved → move ticket to **"Done"** after merging

### Step 3 — Merge approved PR into staging
- Click **"Merge pull request"** on GitHub
- Select **"Squash and merge"** for clean history

### Step 4 — Pull and push to Shopify staging
```bash
git checkout staging
git pull origin staging
shopify theme push --store=beserk.myshopify.com --theme=140520685702
```

### Step 5 — Test on staging
Preview the staging theme at:
```
https://beserk.myshopify.com/?preview_theme_id=140520685702
```
Test all affected pages. Check mobile and desktop.

### Step 6 — Deploy to production (after staging is approved)
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
Jira Board
    │
    └─ Team lead assigns KAN-XX to developer
                    │
            Developer picks up task
                    │
            Move Jira ticket → "In Progress"
                    │
            git checkout staging
            git pull origin staging
            git checkout -b fix/KAN-XX-description
                    │
            make changes to theme files
                    │
            git commit -m "KAN-XX: what changed and why"
            git push origin fix/KAN-XX-description
                    │
            Open PR on GitHub → into staging
            Move Jira ticket → "In Review"
                    │
              Team Lead reviews PR
                    │
         ┌──────────┴──────────┐
     changes?              approved?
         │                     │
   request fixes         Merge into staging
   ticket → In Progress  ticket → Done
                               │
                  shopify theme push --theme=140520685702
                               │
                         test on staging
                               │
                  ┌────────────┴────────────┐
               issues?                 all good?
                  │                         │
            fix & re-test           merge staging → main
                                           │
                                  shopify theme push (live)
                                           │
                                   beserk.com.au updated ✓
```

---

## Jira Ticket Status Flow

```
To Do  →  In Progress  →  In Review  →  Done
```

| Status | When to set it |
|--------|---------------|
| **To Do** | Task is assigned but not started |
| **In Progress** | Developer has created their branch and is working |
| **In Review** | PR is open on GitHub, waiting for team lead review |
| **Done** | PR merged into staging and tested |

---

## Rules

- Never push directly to `main` or `staging` — always use a PR
- Never open a PR into `main` — only into `staging`
- Always pull latest `staging` before creating a new branch
- Every branch must include the Jira ticket number (`KAN-XX`)
- Every commit must reference the Jira ticket (`KAN-XX: description`)
- One Jira ticket = one branch = one PR — keep PRs small and focused
- Resolve all PR review comments before merging
- Update the Jira ticket status as you progress
- Only the team lead pushes to Shopify (staging or live)
- Never share GitHub tokens or credentials in chat or code

---

## Quick Reference

| Task | Command |
|------|---------|
| Start new task | `git checkout -b fix/KAN-XX-description` |
| Save changes | `git add <files> && git commit -m "KAN-XX: description"` |
| Push branch | `git push origin fix/KAN-XX-description` |
| Update local staging | `git checkout staging && git pull origin staging` |
| Push to Shopify staging | `shopify theme push --store=beserk.myshopify.com --theme=140520685702` |
| Push to Shopify live | `shopify theme push --store=beserk.myshopify.com` |
| Preview staging locally | `shopify theme dev --store=beserk.myshopify.com --theme=140520685702` |
