# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

This is a customized **Prestige** theme (v10.9.2) by Maestrooo for the **Beserk** (beserk.com.au) Shopify store. Theme documentation: https://support.maestrooo.com/category/749-technical-documentation

## Development Commands

Requires [Shopify CLI](https://shopify.dev/docs/themes/tools/cli):

```bash
# Start local dev server (proxies to live store)
shopify theme dev --store=beserk.myshopify.com

# Push local changes to store
shopify theme push --store=beserk.myshopify.com

# Pull latest theme from store
shopify theme pull --store=beserk.myshopify.com

# Preview without pushing (creates unpublished theme)
shopify theme push --unpublished --store=beserk.myshopify.com
```

There is no build step — `assets/theme.js` and `assets/theme.css` are pre-compiled bundles. Edit them directly for JS/CSS changes.

## Architecture

### Directory Layout

| Directory | Purpose |
|-----------|---------|
| `layout/theme.liquid` | Root HTML shell — `<head>`, global scripts, `{{ content_for_layout }}` |
| `templates/` | Maps URL patterns to sections. JSON templates are preferred; `.liquid` templates are legacy or special-case |
| `sections/` | Reusable content blocks with merchant-facing schema settings. `main-*` sections are page-specific; others are reusable |
| `snippets/` | Stateless partials rendered with `{% render %}`. No schema. |
| `blocks/` | Inline content blocks (rich-text, headings, buttons, etc.) used inside sections |
| `config/settings_schema.json` | Defines global theme settings UI |
| `config/settings_data.json` | Saved values for those settings |
| `locales/` | i18n strings — `en.default.json` is the source of truth |
| `assets/` | Only 9 files: `theme.js`, `theme.css`, `vendor.min.js`, `photoswipe.min.js`, `splide.min.js`, and a few SVGs |

### JavaScript

`theme.js` uses **ES modules with importmap**. The importmap is injected in `theme.liquid`:

```
"vendor" → vendor.min.js
"theme"  → theme.js
"photoswipe" → photoswipe.min.js
```

Theme settings and media query breakpoints are exposed to JS via `window.themeVariables` (set in `snippets/js-variables.liquid`). Breakpoints: `sm`=700px, `md`=1000px, `lg`=1150px, `xl`=1400px, `2xl`=1600px.

Custom elements (Web Components) are the primary JS pattern — each interactive UI component extends `HTMLElement`.

### CSS / Theming

CSS custom properties are set in `snippets/css-variables.liquid` (injected in `<head>`). Color theming uses two approaches:
- **Color scheme classes**: `class="color-scheme color-scheme--{{ color_scheme.id }}"` for scheme-based coloring
- **`surface` snippet**: for manual inline color overrides via CSS variables

### Third-Party Integrations

Several apps inject directly into the theme:
- **Boost Commerce** — collection filtering (`bc-sf-filter`) and search; `theme.liquid` patches `window.fetch` to append `sort_first` params for specific collections
- **Swym** — wishlist functionality (`swymowishlist-icon.liquid`, `swym-custom.liquid`)
- **Smile.io** — loyalty/rewards points
- **Flits** — enhanced customer account pages (`flits_account*.liquid`)
- **PreOrder Me** — pre-order widgets (`preorder-me-*.liquid`)
- **SMSBump** — SMS marketing opt-in at checkout
- **YouPay** — cart sharing

### Key Snippets

- `product-card.liquid` / `product-card-horizontal.liquid` — product grid cards
- `variant-picker.liquid` — variant selection UI
- `buy-buttons.liquid` — ATC and buy-now buttons
- `facets.liquid` — collection filter UI
- `price-list.liquid` / `price-range.liquid` — price display logic
- `icon.liquid` — SVG icon renderer (pass `icon` variable)

### Template Customization Pattern

Custom page layouts use named templates: e.g., `templates/page.faq.json` renders the FAQ page using a specific set of sections. The template filename suffix after `page.` maps to the template assigned in the Shopify admin.
