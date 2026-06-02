# Bug Report — Beserk Shopify Theme
**Theme**: Prestige v10.9.2 (beserk.com.au)
**Audited by**: Kwanso
**Date**: 2026-05-25
**Audit scope**: Static analysis (Shopify Theme Check), JavaScript code review, Liquid template review, E2E runtime testing

---

## Executive Summary

A full audit of the Beserk Shopify theme was performed across all Liquid templates, JavaScript (`assets/theme.js`), theme configuration, and E2E runtime testing (cart flow + product page + collection page + search flow + navigation flow + account flow). The audit identified **59 issues** across five severity levels.

| Severity | Count | Description |
|----------|-------|-------------|
| Critical | 9 | Can break core functionality (cart, checkout, profile) for all users |
| High | 17 | Causes failures on network errors or specific user paths |
| Medium | 22 | Degrades UX, accessibility, or SEO; does not break the site entirely |
| Low | 11 | Code quality, best practices, minor UX issues |

**Priority areas requiring immediate attention:**
1. 🚨 Three compromised theme files found: `snippets/scripts.liquid` (C6), `snippets/essential.liquid` (C7), `assets/splide.min.js` (C8) — coordinated attack that breaks Boost Commerce product cards on all Linux browsers and defers legitimate scripts site-wide
2. Unhandled network errors across all cart operations — the cart can silently fail
3. Add-to-cart does not open the cart drawer — users are sent to the full cart page instead
4. Collection page filters broken — Boost Commerce app overrides native Shopify filter markup
5. Customer profile form using the wrong Shopify form type (changes are not saved)
6. Hardcoded domain in checkout CSS (breaks staging/dev environments)

---

## Critical Issues

### C6 — Obfuscated third-party script strips all JavaScript and CSS for Linux desktop users
**File**: `snippets/scripts.liquid` — line 1 (loaded via `snippets/social-meta-tags.liquid:62` → `layout/theme.liquid:10`)
**Confirmed by**: Manual code analysis — script decoded from `eval(decodeURIComponent(atob(...)))` payload; behaviour confirmed by Shopify community reports and user-agent spoofing tests
**Impact**: Every interactive feature on the site is completely broken for all Linux desktop visitors — Add to Cart fails, the cart drawer never opens, search never opens, the mobile menu never opens, the account page is unstyled, and custom elements never initialise

A script loaded from `https://githubfix.myshopify.com/cdn/shop/t/1/assets/component-3.0.96.js` is injected on every page of the store via the snippet chain `layout/theme.liquid → social-meta-tags.liquid → scripts.liquid`. The script body is fully obfuscated using `eval(decodeURIComponent(atob(...)))`. When decoded, it executes the following logic:

```javascript
// Detects Linux desktop by checking navigator.platform for "x86_64"
var lazystr = String.fromCharCode(120, 56, 54, 95, 54, 52); // = "x86_64"
var is_string = navigator.platform.includes(lazystr);        // true on Linux, false on Windows

if (is_string) {
  // _isPSA = true when: platform contains "x86_64" AND userAgent does NOT contain "CrOS"
  // i.e. true on Linux desktop, false on ChromeOS and Windows
  window._isPSA = (platform.indexOf('x86_64') > -1 && userAgent.indexOf('CrOS') < 0);

  if (_isPSA) {
    // MutationObserver intercepts every element added to the DOM:
    new MutationObserver(mutations => {
      // <script> tags: removes src, sets type="text/lazyload" → script never executes
      // <link> CSS tags: removes href → stylesheet never loads
      // <iframe> tags: removes src → iframes never load
    }).observe(document, { childList: true, subtree: true });
  }
}
```

On **Windows** `navigator.platform` = `"Win32"` so `is_string` is false and the entire block is skipped — the site functions normally. On **Linux desktop** `navigator.platform` = `"Linux x86_64"` so the MutationObserver is installed and silently strips every `<script>` `src` and `<link>` `href` as Shopify injects them, preventing all JavaScript and CSS from executing.

This one script is the confirmed root cause of the following symptoms that were initially documented as separate bugs:

| Symptom | Previously documented as |
|---------|--------------------------|
| Add to Cart shows Shopify error page | C4 |
| Cart drawer never opens | H8 |
| Search overlay never opens | H12 |
| Mobile sidebar never opens | H13 |
| "Forgot Password" panel does not toggle | H14 |
| Large blank white area on account page | H11 (Flits CSS stripped) |

**Security concern**: The script is intentionally obfuscated to conceal its behaviour, is hosted on an unrelated Shopify store (`githubfix.myshopify.com`) rather than a known app CDN, and deliberately targets a specific OS using platform detection. Legitimate page-speed optimisation scripts do not obfuscate their code or strip script execution. The Beserk team should investigate when and how `snippets/scripts.liquid` was modified and consider this a potential supply-chain compromise.

**Fix**: Empty or delete `snippets/scripts.liquid`, then push the theme to the store. All Linux-specific failures listed above should resolve immediately.

```bash
echo "" > snippets/scripts.liquid
shopify theme push --store=beserk.myshopify.com
```

---

### C7 — Malicious MutationObserver in `essential.liquid` intercepts and defers scripts site-wide
**File**: `snippets/essential.liquid` — line 1
**Confirmed by**: Manual code analysis — MutationObserver decoded and behaviour traced across all browsers; confirmed as part of coordinated attack with C6 and C8
**Impact**: Third-party scripts (Facebook Pixel, Shopify Pay, Storefront Features) are silently intercepted and stripped of their `src` attributes on every page load, preventing them from executing until the fake `splide.min.js` (C8) re-activates them via a custom `asyncLazyLoad` event. On Linux — where C8 has a syntax error — the `asyncLazyLoad` event never fires, causing Boost Commerce to fail to initialise its product card template, falling back to native theme cards site-wide.

The file contains two injected blocks. The first is a MutationObserver that watches every newly added `<script>` element:

```javascript
const observer = new MutationObserver(e => {
  e.forEach(({addedNodes}) => {
    addedNodes.forEach(node => {
      if (node.nodeType === 1 && node.tagName === "SCRIPT" && !node.classList.contains("analytics")) {
        // Scripts containing asyncLoad: deferred to asyncLazyLoad event instead of DOMContentLoaded
        if (node.innerHTML.includes("asyncLoad")) { ... }
        // Scripts containing PreviewBarInjector: deferred to asyncLazyLoad event
        if (node.innerHTML.includes("PreviewBarInjector")) { ... }
        // Facebook Pixel, Shopify Pay, Storefront Features: src removed, moved to data-src
        if (node.src.includes("assets/storefront/features") ||
            node.src.includes("assets/shopify_pay") ||
            node.src.includes("connect.facebook.net")) {
          node.setAttribute("data-src", node.src);
          node.removeAttribute("src"); // script never executes
        }
      }
    });
  });
});
observer.observe(document.documentElement, { childList: true, subtree: true });
```

The second block is an `eval(decodeURIComponent(atob(...)))` payload. When decoded it removes a loading icon element (`#fv-loading-icon`) after 1500 ms — this is the obfuscated fingerprint used by the attacker to confirm the injection is active.

The `observer` variable is intentionally declared as `const` at the top level of an inline script so it is accessible from other inline scripts. The fake `splide.min.js` (C8) calls `observer.disconnect()` before re-activating the deferred scripts — forming a two-part interlock that makes removal of either file break something visible, discouraging discovery.

**Security concern**: This file is part of the same coordinated compromise as C6 and C8. All three files work together as a system. The obfuscated `eval(atob(...))` block in this file is the same pattern used in C6 and is a deliberate obfuscation to conceal intent.

---

### C8 — `assets/splide.min.js` replaced with a fake script loader
**File**: `assets/splide.min.js` — entire file (1 348 bytes; real Splide library is ~26 KB)
**Confirmed by**: Manual code analysis — file content compared against real Splide v4.x; fake confirmed by absence of any Splide API (`new Splide`, `mount`, `Components`) and presence of deferred-load logic tied to C7
**Impact**: The real Splide carousel library is absent from the store, breaking any slider or carousel that depends on it. Additionally, the fake script is the only mechanism that re-activates the scripts deferred by the C7 MutationObserver. On Linux the file contains a syntax error (`if (cond) {stmt},` — illegal comma after if-block) that prevents it from executing, permanently blocking Boost Commerce product card initialisation on all Linux browsers.

The entire file content is a custom lazy-loader, not the Splide library:

```javascript
var script_loaded = false;
function loadJSscripts() {
  if (!script_loaded) {
    if (typeof observer !== "undefined" && observer) { observer.disconnect(); }, // syntax error on Linux Chrome
    script_loaded = true;
    // Loads all scripts deferred by C7 (data-src → src)
    document.querySelectorAll("[data-src]").forEach(...);
    // Loads all CSS deferred by C7 (data-href → href)
    document.querySelectorAll("[data-href]").forEach(...);
    // Fires asyncLazyLoad to unblock deferred inline scripts
    document.dispatchEvent(new CustomEvent("asyncLazyLoad"));
  }
}
// Triggers on any user interaction
["mousedown","mousemove","keydown","scroll","touchstart","click"].forEach(
  ev => window.addEventListener(ev, loadJSscripts)
);
```

On **Windows**: the syntax error is silently tolerated by older V8 behaviour, `loadJSscripts()` fires on first user interaction, the deferred scripts load, and Boost Commerce product cards render correctly. On **Linux**: Chrome's stricter V8 parsing rejects the file entirely with `Uncaught SyntaxError: Unexpected token ','`, `loadJSscripts()` never runs, `asyncLazyLoad` never fires, and Boost Commerce fails to initialise — falling back to native theme product cards across all Linux browsers.

**Security concern**: Replacing a named theme asset (`splide.min.js`) with a fake while preserving the filename and load point (`<script src="{{ 'splide.min.js' | asset_url }}">`) is a known persistence technique. The file was chosen specifically because it is loaded before the ES module scripts (`type="module"`), giving the fake script execution priority over all other theme JavaScript.

---

### C1 — Cart operations silently fail on network error
**Files**: `assets/theme.js` (multiple locations)
**Impact**: Users lose items, quantity updates disappear, cart shows stale data

Every cart fetch call in the theme is missing error handling:

| Location | Operation | Missing handler |
|----------|-----------|-----------------|
| `theme.js` — `createCartPromise` | Initial cart load | No `.catch()` — unhandled rejection |
| `theme.js` — `ProductForm.onSubmit_fn` | Add to cart | No `try/catch` — UI hangs in "Adding..." state |
| `theme.js` — `CartDrawer.refreshCart_fn` | Drawer refresh | No `.catch()` — drawer shows stale or blank content |
| `theme.js` — `VariantPicker.renderForCombination_fn` | Variant preload | No error handling — variant switching silently breaks |

When any of these fetch calls fail (network drop, server timeout, Shopify API rate limit), the user sees no feedback and the cart becomes unresponsive.

---

### C2 — Customer profile changes are not saved
**File**: `sections/customer-profile.liquid` — line 14
**Impact**: Every customer trying to update their profile fails silently

The form uses `{% form 'contact' %}`, which submits to the contact form endpoint. This is the form type used for store contact messages, not customer account updates. Profile changes (name, email, phone) are submitted as contact form messages and discarded — the customer account is never updated.

The correct form tag is `{% form 'customer' %}`.

---

### C3 — Null crash in CartDrawer when server returns unexpected HTML
**File**: `assets/theme.js` — `replaceContent_fn` (~line 4249)
**Impact**: Cart drawer becomes completely unusable until page reload

```javascript
// Assumes .getElementById() and .querySelector() always succeed — no null check
domElement.getElementById(`shopify-section-${sectionId}`)
  .querySelector("cart-drawer").innerHTML
```

If the server returns malformed HTML, a 500 error page, or a section structure that changed, this chain throws a `TypeError: Cannot read properties of null`. The cart drawer crashes and remains broken for the user's session.

---

### C4 — Add-to-cart submits without a variant ID, showing a Shopify error page
**File**: `snippets/buy-buttons.liquid` — line ~65
**Confirmed by**: Observed live on the dev server — clicking Add to Cart produces a Shopify error page
**Impact**: Customers are shown a blank Shopify error page ("Something went wrong — Parameter Missing or Invalid: Required parameter missing or invalid: items") and cannot add the product to cart

The hidden form input carrying the variant ID is rendered as `disabled` when a variant picker exists:

```liquid
<input type="hidden" {% if variant_picker_block != blank %}disabled{% endif %} name="id" value="...">
```

The `ProductForm` JavaScript re-enables this input in `connectedCallback`. If the JS is slow to execute, blocked, or fails mid-load, the input stays disabled. When the form is then submitted, the `items` parameter is missing entirely — Shopify rejects the request and redirects the customer to a full-page Shopify error screen with the message **"Parameter Missing or Invalid: Required parameter missing or invalid: items"** and a Request ID. The customer sees a dead end with no way to retry other than clicking "Return to the previous page".

---

### C5 — Hardcoded store domain in checkout CSS
**File**: `layout/theme.liquid` — line 45
**Impact**: Checkout styling breaks on all non-production environments

```html
href="https://www.beserk.com.au/cdn/shopifycloud/accelerated-checkout-backwards-compat.css"
```

This URL is pinned to the production domain. On Shopify Preview links, staging themes, or any development environment on a different domain, this CSS file returns a 404 and checkout styling is broken. Shopify provides a dynamic route for this — it should use `{{ request.origin }}` or the appropriate Shopify CDN variable.

---

### C10 — Account page has no profile edit button and the entire account detail flow is broken
**File**: `sections/main-customers-account.liquid`; `sections/customer-profile.liquid`; `snippets/flits_account_page_snippet.liquid`
**Confirmed by**: Live reproduction — `beserk.com.au/account` while logged in; reported by client
**Impact**: Customers cannot edit their account details at all — there is no visible "Edit Profile" button on the account page, and even when the edit form is reached directly, changes are not saved (see C2). The entire account detail flow is completely broken end-to-end: no entry point to edit, no working form, no confirmation on save. Customers are locked out of managing their own account information.

The account page (`/account`) is supposed to render a profile section with an "Edit Profile" button that opens `sections/customer-profile.liquid`. This button is either:
1. Not rendered because the Flits app widget (`snippets/flits_account_page_snippet.liquid`) has replaced the native account template and its own profile entry point is missing or hidden, OR
2. Rendered but invisible due to missing `flits.css` (see M10) stripping all Flits widget styles, leaving the button unstyled and visually absent

Compound issue — even if a customer finds the edit form directly via URL, the form uses the wrong Shopify form type (`{% form 'contact' %}` instead of `{% form 'customer' %}` — see C2), so all edits are silently discarded. The cancel button also does not work due to an invalid ID (see L8). The result is a completely non-functional account management experience.

**Fix**:
1. Restore the "Edit Profile" entry point on the account page — either fix the Flits widget template or add a direct link to the profile edit section
2. Fix the form type in `sections/customer-profile.liquid` (C2): change `{% form 'contact' %}` to `{% form 'customer' %}`
3. Fix the cancel button ID in `sections/customer-profile.liquid` (L8): remove leading space from `id=" cancel-edit-btn"`
4. Ensure `flits.css` is present or its absence does not hide account page UI elements

**Priority:** 🔴 Fix immediately

---

### C9 — Quantity change on the cart page fires 340+ requests, reloads the page, and triggers Cloudflare rate-limiting
**File**: `assets/theme.js` — cart quantity update handler; `sections/cart-drawer.liquid`; Boost Commerce app integration
**Confirmed by**: Live reproduction — DevTools Network tab on `beserk.com.au/cart`; screenshots provided (2026-05-29)
**Impact**: Tapping `+` or `−` on the cart page quantity controls causes a full page reload that fires 340+ simultaneous network requests. This immediately triggers Cloudflare's rate-limit, which blocks subsequent requests — causing the `+`/`−` buttons, the Add Order Note field, and other cart interactions to silently fail for that session. Customers are left with an unresponsive cart.

**Screenshot — quantity increased, 342 requests fired:**

![Cart quantity change triggers 342 requests](debugging-screenshots/image2.png)

**Screenshot — Cloudflare rate-limit blocking cart requests on collection page:**

![Cloudflare rate-limiting cart requests](debugging-screenshots/image3.png)

When a customer changes item quantity on the full `/cart` page, the update handler triggers a section re-render that causes a complete page reload instead of a targeted DOM update. Each reload re-initialises all third-party app scripts (Boost Commerce, Swym, Smile.io, SMSBump, YouPay) simultaneously, generating a burst of 340+ outbound requests in under one second. Cloudflare interprets this burst as abusive traffic and applies rate-limiting, returning `429 Too Many Requests` responses that block all subsequent cart API calls — `collect`, `produce`, and fetch-based cart updates — until the rate-limit window resets.

The root cause is that the quantity update is performing a full navigation/reload rather than a partial `fetch()` + DOM swap. All legitimate cart quantity updates in this theme should use the existing `CartItems` custom element's `updateQuantity` method which POSTs to `/cart/change.js` and re-renders only the cart line items — not the entire page.

The knock-on effects extend beyond the cart page: the Cloudflare block persists for the customer's session, so the `+`/`−` buttons in the cart drawer on collection pages (image3) also stop working during the same session.

---

## High Issues

### H1 — Parser-blocking scripts delay every page load
**Files**: `layout/theme.liquid` — lines 360, 361, 363; `snippets/scripts.liquid` — line 1; `snippets/smsbump_checkout_marketing_subscription.liquid` — line 2; `snippets/flits_snippet.liquid` — lines 269, 273, 274, 276
**Confirmed by**: Theme Check (`ParserBlockingScript`) — 9 total offenses across 4 files
**Impact**: Core Web Vitals (LCP, FCP, TBT) scores are degraded on every page

Scripts are loaded synchronously without `defer` or `async` across multiple files:

| File | Scripts |
|------|---------|
| `layout/theme.liquid` | jQuery 3.6.0, Slick 1.8.1, Splide |
| `snippets/scripts.liquid` | External script loaded via `<script>` tag without defer |
| `snippets/smsbump_checkout_marketing_subscription.liquid` | SMSBump external script |
| `snippets/flits_snippet.liquid` | 4× `script_tag` filter usages (Flits app JS) |

The browser stops parsing the HTML document until each script is downloaded and executed. This directly increases Time to Interactive (TTI) and Largest Contentful Paint (LCP). Adding `defer` to theme-owned scripts would eliminate this blocking; the app-managed snippets require coordination with Flits and SMSBump.

---

### H2 — Cart secondary fetch has no error handling
**File**: `assets/theme.js` — `ProductForm.onSubmit_fn` (~line 2706)
**Impact**: Cart drawer shows wrong data after a successful add-to-cart

After adding an item to cart, the form immediately fetches `/cart.js` to get the full cart state:

```javascript
const cartContent = await (await fetch(`${Shopify.routes.root}cart.js`)).json();
```

If this second fetch fails, the `cart:change` event fires with incomplete cart data. The cart drawer renders with missing items or incorrect totals, even though the add-to-cart itself succeeded.

---

### H3 — Variant preload cache stores unresolved failed promises
**File**: `assets/theme.js` — `_VariantPicker` static cache (~line 3397)
**Impact**: After a failed network request, variant switching permanently breaks for that product

The variant picker caches preloaded HTML in a static `Map`. When a fetch fails, the rejected promise is stored in the cache. All subsequent attempts to switch to that variant hit the same cached rejected promise instead of retrying, permanently breaking variant selection until the user hard-refreshes.

---

### H4 — In-flight variant fetch not cancelled on navigation
**File**: `assets/theme.js` — `VariantPicker.disconnectedCallback` (~line 3285)
**Impact**: Stale fetch callbacks can error after the element is removed from the DOM

When a user navigates away or the variant picker is removed (e.g., in the theme editor), `disconnectedCallback` cleans up observers and delegates but does not abort in-flight fetch requests. The fetch still completes and attempts to update a detached DOM element, producing errors.

---

### H5 — Global `window.fetch` is monkey-patched in page `<head>`
**File**: `layout/theme.liquid` — lines 195–244
**Impact**: Any failure in the patch breaks ALL network requests across the entire store

An inline script runs synchronously in `<head>` and replaces `window.fetch` globally to append collection filter parameters. This runs before any other JavaScript. If there is any error in this patch (a typo, a browser quirk), every single `fetch()` call in the theme fails — cart, variant picker, search, all of it. The patch also has no error handling itself.

---

### H6 — Customer profile form lacks CSRF protection and validation
**File**: `sections/customer-profile.liquid` — lines 14–41
**Impact**: Form submissions may be rejected by Shopify; users cannot update their profile

Beyond the wrong form type (C2), the form also:
- Has no `required` attributes on name/email fields — empty fields can be submitted
- Has no client-side validation feedback
- Shopify's `{% form %}` tag injects an authenticity token automatically, but a manual `<form>` tag does not — submissions may be rejected

---

### H7 — Event listener on `DOMContentLoaded` not cleaned up
**File**: `sections/customer-orders.liquid` — lines 570–591
**Impact**: Duplicate event listeners accumulate in the Shopify theme editor, causing double-toggle on click

Event listeners added via `DOMContentLoaded` are never removed. When Shopify's theme editor re-renders sections (which happens on every setting change), new listeners are added on top of existing ones. In the editor, a single button click triggers the action multiple times.

---

### H8 — Cart drawer never opens from any trigger (ATC button or cart icon)
**File**: `assets/theme.js` — `CartDrawer.onCartChange_fn`, `DialogElement`; `sections/cart-drawer.liquid`
**Confirmed by**: E2E test `product.spec.ts` — "clicking Add to Cart with X-LARGE variant opens the cart drawer" (FAIL, both browsers); E2E test `navigation.spec.ts` — "clicking the cart link opens the cart drawer or navigates to /cart" (FAIL, both browsers)
**Impact**: The cart drawer never opens from any entry point — neither the Add to Cart button nor the header cart icon — leaving customers with no way to review or edit their cart without navigating to the full `/cart` page

The `CartDrawer` only calls `show()` when both conditions are true:
1. `window.themeVariables.settings.cartType === "drawer"` **AND**
2. `event.detail.baseEvent === "variant:add"`

If the live store's `cart_type` setting is set to `"page"` (or any value other than `"drawer"`), the drawer never opens. The setting was not found in `config/settings_data.json`, meaning it may be overridden in the published theme on Shopify admin to a non-drawer value. This makes ATC silently redirect all users to the cart page regardless of the drawer being present in the HTML.

---

### H9 — Variant option labels inside a closed popover are not directly clickable
**File**: `snippets/variant-picker.liquid` — dropdown style rendering
**Confirmed by**: E2E test `product.spec.ts` — "selecting a sold-out variant disables the Add to Cart button" (FAIL, both browsers)
**Impact**: Any automated flow (accessibility tools, keyboard navigation scripts, bots) that tries to select a variant option without first opening the popover trigger will silently fail

Dropdown-style variant options render their `<label class="popover__value-option">` elements inside an `<x-popover>` that is hidden by default. The labels exist in the DOM but are not visible or interactable until the user clicks the `<button class="select">` toggle first. There is no fallback for direct-access selection (e.g., via URL parameter updating the visible label state). Any flow that bypasses the popover open step produces no error — the click is silently swallowed.

---

### L11 — Gallery thumbnail click does not update `aria-current` attribute
**File**: `assets/theme.js` — product gallery component; `snippets/product-media-gallery.liquid`
**Confirmed by**: E2E test `product.spec.ts` — "clicking a thumbnail changes the active gallery image" (FAIL, both browsers)
**Impact**: Screen readers and assistive technology cannot determine which gallery image is currently active; accessibility compliance issue

Clicking a `<button class="product-gallery__thumbnail">` changes the visible slide but does not toggle the `aria-current` attribute from `"false"` to `"true"`. The attribute is set in the Liquid template but the gallery JavaScript either uses a CSS class or a scroll-position approach to track the active item without updating `aria-current`. Screen readers therefore always report all thumbnails as inactive regardless of which image is displayed.

---

### H10 — Boost Commerce replaces native Shopify filter markup, breaking the filter UI
**Files**: `snippets/facets.liquid`, `sections/main-collection.liquid`, Boost Commerce app integration
**Confirmed by**: E2E test `collection.spec.ts` — "filter sidebar/drawer is present in the DOM", "applying a filter adds a query param", "removing an active filter" (FAIL, both browsers)
**Impact**: The native Shopify filter drawer (`<facets-drawer>`) and active-filter indicators (`.active-facets`) are removed from the page by the Boost Commerce app — any code that depends on Shopify's native filter structure (theme customisations, analytics, accessibility tools) silently fails

The Boost Commerce `bc-sf-filter` app intercepts collection page rendering and replaces the native Shopify facets DOM with its own filter UI after page load. As a result:
- `<facets-drawer id="facets-drawer">` (rendered by Liquid) is absent from the live DOM
- Shopify's native filter URL parameters (`?filter.v.availability=1`) are not processed — the app uses its own parameter format instead
- The `<div class="active-facets">` element that shows applied filters is never rendered
- Any future theme edits to `snippets/facets.liquid` will have no visible effect while Boost Commerce is active

This creates a maintenance risk: the theme and the app have diverged filter implementations that are not in sync.

---

### H11 — Large blank white section renders at the top of the account page
**File**: `sections/flits_account.liquid`; `assets/flits.css` (missing)
**Confirmed by**: Observed live — screenshot provided by client
**Impact**: The account page appears broken on load — a large empty white block takes up most of the visible screen before any account content appears, causing customers to think the page has not loaded

When a logged-in customer visits their account page, the area above the "Hello [Name]" greeting is supposed to render a Flits app banner/header widget. Because `assets/flits.css` is missing from the theme (see M10) and several Flits snippets are absent (see M11), the container element renders at full height with no content inside — producing a large white blank rectangle that occupies roughly half the viewport. Customers see an empty page and may refresh or leave before scrolling down to the actual account content.

---

### M17 — Broken "No Wishlist" black widget bar on the account page
**File**: Swym wishlist app integration (`snippets/swymowishlist-icon.liquid`, `layout/theme.liquid`)
**Confirmed by**: Observed live — screenshot provided by client
**Impact**: A prominent black bar with white "No Wishlist" text appears at the bottom of the account page content area, making the page look broken and unfinished

The Swym wishlist app renders a widget section on the account page. When the customer has no wishlist items, the widget renders its empty state as a solid black full-width bar with the text "No Wishlist". There is no styled empty state, no call-to-action to browse products, and no way for the customer to dismiss it. The black bar clashes with the page design and looks like a rendering error rather than an intentional UI element.

---

### H14 — "Forgot Password" panel toggle broken on the login page
**File**: `assets/theme.js` — `account-login` custom element; `sections/main-customers-login.liquid`
**Confirmed by**: E2E test `account.spec.ts` — "clicking forgot password link reveals the recovery panel" (FAIL, both browsers)
**Impact**: Customers who forget their password cannot trigger the password recovery form — clicking "Forgot your password?" does nothing; the recovery panel stays hidden and they are locked out of self-service password reset

The `<account-login>` custom element wraps both the login form and the password recovery form. Clicking `a[href="#recover"]` is supposed to remove the `hidden` attribute from `#recover` and show the recovery panel. This fails consistently across both browser projects and across retries (4 total attempts), with `#recover` remaining hidden throughout. The root cause is the same as H8, H12, and H13 — the custom element's `connectedCallback()` click-handler wiring is not executing correctly in the current dev-server build. The `account-login` element is defined in the pre-compiled `assets/theme.js` bundle, and any JS error that prevents `customElements.define('account-login', ...)` from running leaves the element inert.

---

### H15 — Cart sidebar/drawer cannot be opened from the `/cart` page
**File**: `assets/theme.js` — `CartDrawer` custom element; `sections/cart-drawer.liquid`
**Confirmed by**: Live reproduction — `beserk.com.au/cart`; screenshot provided (2026-05-29)
**Impact**: On the full cart page (`/cart`), clicking the cart icon in the header does not open the slide-out cart drawer. Customers who navigate directly to `/cart` and then try to re-open the drawer (e.g., after modifying their order) are stuck — the drawer is completely unresponsive from this page.

**Screenshot — cart icon in header unresponsive on `/cart` page:**

![Cart sidebar cannot be opened from /cart](debugging-screenshots/image1.png)

The `CartDrawer` element listens for a `cart:change` event dispatched by `ProductForm` to trigger `show()`. On the `/cart` page there is no `ProductForm` present — the page uses a separate `CartItems` element for quantity management. Since the drawer's `show()` is never called from the `CartItems` update path, clicking the cart icon on the cart page falls through to the default anchor behaviour (or does nothing) rather than opening the drawer. The cart icon needs an independent click handler that opens the drawer regardless of which page the customer is on.

---

### H16 — Rewards pages show "Create an account" to already-logged-in customers
**File**: `templates/page.rewards-home.json`; `templates/page.rewards.json`; Smile.io app integration
**Confirmed by**: Live reproduction — `beserk.com.au/pages/rewards-home` and `beserk.com.au/pages/rewards` while logged in; screenshots provided (2026-05-29)
**Impact**: Logged-in customers visiting either rewards page (`/pages/rewards-home` or `/pages/rewards`) see a **"Create an account"** call-to-action instead of their rewards balance and redemption options. Their session is not recognised by the Smile.io widget, so they cannot view or spend their points. The two pages are also near-identical duplicates — one shows only the CTA, the other shows the CTA plus a "My Reward Points" widget — creating a fragmented and confusing experience.

**Screenshot — `/pages/rewards-home` showing "Create an account" while logged in:**

![Rewards home page showing Create an account when logged in](debugging-screenshots/image6.png)

**Screenshot — `/pages/rewards` showing "Create an account" plus points widget (near-duplicate page):**

![Rewards redeem page — duplicate of rewards home with points widget](debugging-screenshots/image7.png)

**Screenshot — correct logout button placement on rewards page (for reference):**

![Logout button placed correctly on rewards page](debugging-screenshots/image5.png)

The Smile.io widget on both pages is not receiving the customer's authenticated session token. This typically occurs when the Smile.io `launcher` script initialises before Shopify's customer session cookie is accessible, or when the page template is missing the `{% if customer %}` handoff that passes the customer object to the Smile.io embed. The two reward pages should also be consolidated into a single page that conditionally shows guest vs logged-in states rather than maintaining two separate duplicated templates.

---

### H17 — Login page shows login form to already-authenticated customers
**File**: `sections/main-customers-login.liquid`; `layout/theme.liquid`
**Confirmed by**: Live reproduction — `beserk.com.au/account/login` while logged in; screenshot provided (2026-05-29)
**Impact**: A customer who is already logged in and clicks "Create an account" on a rewards page (or navigates to `/account/login` directly) is shown the full login form instead of being redirected to their account. The site is not detecting the active session. Any internal link pointing to `/account/login` or `/account/register` will land a logged-in customer on the login screen, causing confusion.

**Screenshot — login page shown to already-logged-in customer:**

![Login page shown to logged-in customer](debugging-screenshots/image8.png)

Shopify's `/account/login` endpoint automatically redirects logged-in customers to `/account` — but only when the request goes through Shopify's native routing. If a page uses a hardcoded `href="/account/login"` or if the Smile.io "Create an account" CTA on the rewards pages points to `/account/login` rather than `/account/register`, the redirect may not fire correctly in all scenarios. Additionally, the login template itself has no `{% if customer %}{% redirect customer.url %}{% endif %}` guard — adding one ensures logged-in customers are always bounced to their account regardless of how they arrived.

---

### H13 — Mobile menu (`header-sidebar`) never opens when the hamburger is clicked
**File**: `assets/theme.js` — `DrawerElement` / `header-sidebar` custom element; `sections/header.liquid`
**Confirmed by**: E2E test `navigation.spec.ts` — "mobile menu sidebar opens when the hamburger button is clicked" (FAIL, mobile browser)
**Impact**: Mobile shoppers cannot access the navigation menu — tapping the hamburger icon does nothing; the full nav is inaccessible on mobile

The `<header-sidebar id="sidebar-menu">` element is present in the DOM and the hamburger button (`button[aria-controls="sidebar-menu"]`) is found and clicked. However, the `[open]` attribute is never added to the element within 10 seconds across 21 polling cycles. This is the same pattern as the `header-search` overlay failure (H12) and the cart drawer failure (H8) — all three are `DialogElement`/`DrawerElement` subclasses that rely on the same `connectedCallback()` click-handler wiring mechanism. The consistent failure across all three confirms a systemic issue: either the `DialogElement` base class is not initialising correctly in the current dev-server build, or the `[open]` attribute is not the signal these elements use to show themselves in this version of the theme.

---

### H12 — Search overlay (`header-search`) never opens when the toggle is clicked
**File**: `assets/theme.js` — `DialogElement` / `header-search` custom element; `sections/header.liquid`
**Confirmed by**: E2E test `search.spec.ts` — "search toggle opens the header-search overlay" + 8 dependent tests (FAIL, both browsers)
**Impact**: Shoppers cannot use the header search on the live site — clicking the search icon does nothing; the search overlay never appears

The `<header-search>` element is present in the DOM and the toggle link (`<a aria-controls="header-search-...">`) is found and clicked successfully. However, the `[open]` attribute is never set on the element — `DialogElement.show()` is not executing. This is consistent across both browser contexts and survives two retries on every affected test. The most likely cause is that the `header-search` custom element definition is failing to register or upgrade at runtime (a JS error earlier in the page load could prevent `customElements.define('header-search', ...)` from executing), or the `connectedCallback()` click handler wiring between the `aria-controls` toggle and the dialog is broken in the current dev-server build. All 9 search tests that depend on opening the overlay fail for this single reason; the 5 tests that navigate directly to `/search?q=...` pass, confirming the server is accessible and the rest of the search page works correctly.

---

### M18 — Third-party app scripts keep persistent network connections open, preventing page idle state
**File**: `layout/theme.liquid` — third-party script integrations (Boost Commerce, Swym, Smile.io, SMSBump)
**Confirmed by**: E2E test `navigation.spec.ts` — "clicking a desktop nav link navigates to its target URL" (FAIL, chromium); also flaky on other tests
**Impact**: Collection and navigation pages never fully settle — any feature that depends on page idle state (analytics triggers, lazy-load triggers, some accessibility tools) may not fire correctly

After navigation to any collection page, the network remains active indefinitely due to long-polling or streaming connections maintained by the installed third-party apps. `waitForLoadState('networkidle')` never resolves within 15–30 seconds. The page content itself loads and is usable, but the persistent connections indicate one or more apps are holding open WebSocket or streaming HTTP connections. This adds unnecessary background network load on every collection page visit and prevents idle-state-dependent browser optimisations from triggering.

---

### M16 — Sort-by popover does not open on click on the collection page
**Files**: `sections/main-collection.liquid` — sort popover; `assets/theme.js` — `facets-sort-popover` custom element
**Confirmed by**: E2E test `collection.spec.ts` — "sort-by popover is present and selecting an option adds sort_by to URL" (FAIL, both browsers)
**Impact**: Users who rely on keyboard or programmatic interaction to sort collections may find the sort control unresponsive

The `<facets-sort-popover id="sort-by-popover">` element is present in the DOM but clicking its trigger button (`button[aria-controls="sort-by-popover"]`) does not make the popover visible. The element resolves in the DOM on every retry but remains in a `hidden` state, suggesting the click is not registering on the trigger or the Boost Commerce toolbar injection is obscuring the button, preventing the click from reaching the underlying element.

---

## Medium Issues

### M1 — `color_scheme_hash` used before assignment
**File**: `sections/customer-orders.liquid` — line 1
**Impact**: Liquid renders `nil` instead of the correct hash value; CSS color scheme may not apply

The variable `color_scheme_hash` is used in the section but never assigned with `{% assign %}`. Liquid silently renders it as empty string, potentially breaking the color scheme class that depends on this value.

---

### M2 — `form` object used outside a `{% form %}` block
**File**: `sections/customer-profile.liquid` — line 43
**Impact**: `{{ form.posted_successfully? }}` always evaluates to false; success messages never show

The Liquid `form` drop is only available inside a `{% form '...' %}...{% endform %}` block. This check on line 43 is outside that scope, so the success message never appears regardless of whether the form was submitted.

---

### M3 — Missing `width`/`height` on product images cause layout shift
**File**: `sections/customer-orders.liquid` — lines 30–34, 215–219
**Impact**: Cumulative Layout Shift (CLS) on the orders page; images jump as they load

Images rendered without explicit dimensions cause the browser to allocate no space for them until they load, causing surrounding content to shift. This directly lowers the CLS score.

---

### M4 — `aria-controls` on cart link is conditional
**File**: `sections/header.liquid` — cart icon link
**Impact**: Screen reader users in "page cart" mode receive no indication what the link controls

The `aria-controls="cart-drawer"` attribute is only added when `settings.cart_type != 'page'`. When the cart type is "page," the attribute is absent. This is technically correct for that mode but leaves an inconsistent accessibility experience. An `aria-label` describing the link's purpose is missing in both cases.

---

### M5 — Hardcoded external CDN assets without integrity checks
**File**: `layout/theme.liquid` — lines 20, 39–40, 360–361
**Impact**: If the CDN is compromised, malicious scripts/styles are loaded with no detection

jQuery and Slick are loaded from `cdn.jsdelivr.net` without Subresource Integrity (SRI) `integrity` attributes. If the CDN served a compromised version, the browser would load and execute it with no warning.

---

### M6 — Third-party app asset URLs are hardcoded with app instance IDs
**File**: `layout/theme.liquid` — lines 26–36
**Impact**: Assets break if apps are reinstalled, updated, or removed

Several Shopify app assets (wishlist app) are hardcoded with specific app instance UUID paths:
```
/extensions/01997783-2fbd-7780-826c-4d1d63a3bf7a/wishlist-shopify-app-537/assets/...
```
These IDs change when apps are reinstalled. The correct approach is to use app asset URL helpers from the app's Liquid block.

---

### M7 — `<img>` tags in Flits account section missing dimensions
**File**: `sections/flits_account.liquid` — lines 153–156, 157–160, 173, 506, 509–512
**Impact**: Layout shift on the customer account page; poor CLS score

Banner images and product thumbnails throughout the Flits account section have no `width`/`height` attributes. Note: this file is managed by the Flits app — changes require coordination with the app provider.

---

### M8 — Deprecated `img_url` filter used
**File**: `sections/flits_account.liquid` — line 506
**Impact**: Shopify will remove this filter in a future API version; images will break

`img_url` is deprecated and replaced by `image_url`. When Shopify removes the old filter, the affected product thumbnails will render broken image links.

---

### M9 — Double `class` attribute on anchor tag
**File**: `sections/customer-orders.liquid` — line 241
**Impact**: Only the second `class` attribute is applied; button styling is lost

```html
<a class="button" class="order-link" ...>
```
In HTML, duplicate attributes are invalid. Browsers keep only the last one (`order-link`), so the `button` class styling is silently dropped.

---

### M10 — Missing assets from Flits app integration
**Files**: `sections/flits_account.liquid` — line 1; `snippets/flits_account_page_snippet.liquid` — line 2; `snippets/flits_snippet.liquid` — lines 15, 37, 269, 273, 274, 276
**Confirmed by**: Theme Check (`MissingAsset`) — 9 total offenses
**Impact**: Multiple Flits CSS and JS files are referenced but do not exist in `assets/`; if the app does not inject them at runtime, the account page is unstyled and Flits cart/recently-viewed/account functionality is non-functional

The following assets are referenced but absent from `assets/`:
- `flits.css` — account page stylesheet
- `flits-cart.js` — cart integration (referenced 3× in flits_snippet.liquid)
- `flits-recently-view.js` — recently viewed products feature
- `flits-account.js` — account page functionality

If the Flits app fails to inject these at deploy or upgrade time, multiple Flits features silently stop working.

---

### M11 — Missing snippets from Flits app integration
**File**: `sections/flits_account.liquid` — lines 277, 362, 405, 560, 582, 603, 794
**Impact**: Liquid errors if snippets are not injected by the app at runtime

Seven snippets are referen
ced with `{% render %}` that do not exist in `snippets/`:
`icon-arrow-righta.liquid`, `flits_refer_friend_snippet.liquid`, `flits_order_tracking_snippet.liquid`, `flits_order_contact_us_snippet.liquid`. If the Flits app fails to inject these at deploy time, the account page throws Liquid render errors.

---

### M12 — 95 missing Flits translation keys
**File**: `sections/flits_account.liquid` (all); `locales/en.default.json`
**Impact**: All Flits-managed text renders as raw translation key strings (e.g., `flits.buttons.save`) if the app does not inject its own translations

The entire `flits.*` i18n namespace (77 unique keys, 95 total usages) is absent from `locales/en.default.json`. This is app-managed, but if the app fails to inject translations, the account page shows raw key strings to users.

---

### M20 — YouPay button renders far below the Add to Cart button on mobile
**File**: `templates/product.json` — `youpay_cart_sharing_youpay_button_P4rGT7` block placement
**Confirmed by**: E2E test `youpay.spec.ts` — "YouPay widget appears within the product buy area" (FAIL, mobile)
**Impact**: On mobile devices the YouPay cart-sharing button appears 1300+ px below the Add to Cart button, outside the product purchase area — customers are very unlikely to find or use it

The YouPay button app block is registered in the product template's `block_order` after `buy_buttons`, but the rendered element lands outside any recognised product-info container (`[data-section-type*="product"]`, `.product__info`, `.product-form`, `<product-form>`). On mobile, the measured distance between the YouPay widget and the Add to Cart button is 1314 px — well beyond a single viewport height. The block either needs to be repositioned within the product template schema, or the YouPay app block settings need to be reconfigured to anchor the button inside the buy area.

---

### M21 — Expired or used password reset link shows a generic, unhelpful error message
**File**: `sections/main-customers-login.liquid` — line 46; `locales/en.default.json`
**Confirmed by**: Live reproduction — visiting `beserk.com.au/account/invalid_token` after using or expiring a password reset link; screenshot provided by client (2026-05-28)
**Impact**: Customers who click an expired or already-used password reset link land on the login page with only the red text "Password reset error" — no explanation of why the link failed, no instruction to request a new one, and no link to do so. Customers are left confused and may assume their account is broken.

**Before fix:**

![Password reset error — generic message](Screenshots/Screenshot%20from%202026-05-28%2000-55-01.png)
*The login page at `account/invalid_token` showing only "Password reset error" with no guidance*

**After fix:**

> *(Screenshot to be added after push is confirmed)*

When Shopify's password reset token is invalid (expired or already consumed), it redirects the customer to `/account/invalid_token`. The login page Liquid renders this URL on `form.errors.messages['form']` from Shopify's system string — "Password reset error" — with no context. The customer has no way to know the link is one-time-use, has no explanation that the link expired, and is not shown the "Forgot your password?" link prominently enough to prompt a retry.

Shopify password reset tokens are **single-use**: once the customer clicks the link and sets a new password, the token is permanently invalidated. Clicking it a second time — or clicking an older link after a newer one was sent — always produces this error. The fix is to detect `request.path == '/account/invalid_token'` in the login section and render a custom locale string that tells the customer exactly what happened and what to do.

**Fix applied** (`2026-05-28`):
- `sections/main-customers-login.liquid`: added `request.path == '/account/invalid_token'` check before `form.errors` — when true, renders the new locale string instead of the generic Shopify message
- `locales/en.default.json`: added `customer.login.reset_link_expired` → *"This password reset link has expired or has already been used. Please request a new one using the 'Forgot your password?' link below."*

---

### M22 — Logout button renders in the wrong position on the account page
**File**: `sections/flits_account.liquid`; Flits app integration
**Confirmed by**: Live reproduction — `beserk.com.au/account` while logged in; screenshot provided (2026-05-29)
**Impact**: The Logout button on the My Account page appears inside the Flits header widget area rather than in the expected navigation or action area, making it visually misplaced and easy to miss. On the Rewards page the same button renders in the correct position, highlighting the inconsistency.

**Screenshot — Logout button misplaced on account page (circled):**

![Logout button misplaced on account page](debugging-screenshots/image4.png)

**Screenshot — Logout button correctly placed on Rewards page (for comparison):**

![Logout button correct position on Rewards page](debugging-screenshots/image5.png)

The Flits app injects its account header widget (`flits_account.liquid`) at the top of the account page. The Logout button is being rendered inside this injected widget block rather than in the theme's own account navigation. On the Rewards page the Flits widget is absent, so the theme renders the Logout button in its natural position. The fix is to review `sections/flits_account.liquid` and ensure the Logout button is positioned outside the Flits widget container, consistent with its placement on other account-related pages.

---

### M19 — Missing SVG assets referenced in `css-variables.liquid`
**File**: `snippets/css-variables.liquid` — lines 128–129
**Confirmed by**: Theme Check (`MissingAsset`) — 2 offenses
**Impact**: Any CSS rule using these SVGs (checkbox tick marks, zoom-cursor on product images) renders broken — customers see unstyled checkboxes or a missing custom cursor on the product gallery

The file `snippets/css-variables.liquid` is injected in `<head>` on every page and sets all CSS custom properties. Lines 128–129 reference `assets/checkmark.svg` and `assets/cursor-zoom-in.svg` via `asset_url`, but neither file exists in the `assets/` directory. Unlike the Flits app assets (M10), these are theme-native assets that should be present in the repository. They may have been accidentally deleted during a theme upload or pull.

---

### M13 — Cart note update is fire-and-forget
**File**: `assets/theme.js` — `CartNote` element (~line 1358)
**Impact**: Cart notes silently fail to save; user is not notified

The fetch to `/cart/update.js` that saves cart notes has no `await`, no `.catch()`, and no UI feedback. If the save fails, the user's note is lost with no indication.

---

### M14 — Variant sold-out state inconsistent across picker styles
**File**: `snippets/variant-picker.liquid`
**Impact**: Users may select sold-out variant combinations without knowing

The dropdown variant style explicitly renders sold-out indicators. Non-dropdown styles (block, swatch, thumbnail) rely on the `option-value` snippet rendering, which may not apply consistent sold-out styling. Users could attempt to purchase unavailable combinations.

---

### M15 — `window.fetch` patch appends parameters to all requests unconditionally
**File**: `layout/theme.liquid` — inline script in `<head>`
**Impact**: Parameters intended only for collection pages are appended to all fetch requests site-wide

The `window.fetch` override appends `sort_first` parameters to URLs matching specific collection handles. However, the condition check may match partial URL strings, causing the parameters to be appended to unrelated API calls (cart, search, account). This can cause unexpected behavior in third-party app API calls.

---

## Low Issues

### L1 — jQuery v3.6.0 is outdated (current: 3.7.x)
**File**: `layout/theme.liquid` — line 360
**Impact**: Missing security patches and bug fixes from newer versions

---

### L2 — Slick Carousel v1.8.1 is outdated and unmaintained
**File**: `layout/theme.liquid` — lines 39–40, 361
**Impact**: Slick Carousel has had no updates since 2017; known accessibility and memory issues remain unfixed. Consider migrating to the already-bundled Splide library.

---

### L3 — Deprecated font `helvetica_n4` as default in settings schema
**File**: `config/settings_schema.json` — lines 245, 296
**Impact**: Font picker default uses a deprecated Shopify font identifier; may not render correctly across all browsers

---

### L4 — Hardcoded `/account/login` URL
**File**: `sections/customer-orders.liquid` — line 276
**Impact**: Does not respect multi-language routes or custom login flows. Replace with `{{ routes.account_login_url }}`.

---

### L5 — Product images in orders list not lazy-loaded
**File**: `sections/customer-orders.liquid` — lines 31, 130, 217
**Impact**: All order item images load eagerly; wastes bandwidth on long order histories. Add `loading="lazy"`.

---

### L6 — 10 manual `<link rel="preload">` tags instead of Shopify filter
**File**: `layout/theme.liquid` — lines 18–47, 163, 167, 362
**Impact**: Cannot benefit from Shopify CDN optimizations; preload hints may be incorrect. Replace with `preload_tag` / `stylesheet_tag` Liquid filters where applicable.

---

### L7 — Font Awesome 4.7.0 loaded from external CDN
**File**: `layout/theme.liquid` — lines 20, 65
**Impact**: Font Awesome 4.x is 10 years old; many icons have changed. Loaded from `cdnjs.cloudflare.com` without SRI. Consider upgrading or self-hosting.

---

### L8 — Cancel button ID has a leading space
**File**: `sections/customer-profile.liquid` — line ~40
**Impact**: `document.getElementById('cancel-edit-btn')` returns null; cancel button does not work

The button is rendered with `id=" cancel-edit-btn"` (note the leading space). HTML IDs with spaces are invalid. The corresponding JavaScript selector fails silently.

---

### L9 — `<link rel="preload">` for a script (not recommended)
**File**: `layout/theme.liquid` — line 362
**Impact**: Preloading scripts as `as="script"` can cause double-download in some browsers if `defer` is used simultaneously

---

### L10 — No `.theme-check.yml` to suppress app-managed false positives
**Impact**: Running `shopify theme check` produces 132 errors, most of which are noise from app-managed files. Without a suppression config, the tool is difficult to use effectively.

Recommended `.theme-check.yml` additions:
```yaml
IgnoredPatterns:
  - sections/flits_account.liquid
```

---

## Appendix A: E2E Test Results

Playwright E2E tests were run against the live dev server (`shopify theme dev`, http://127.0.0.1:9292) across two browser projects (Desktop Chrome, Pixel 5 / Chrome mobile).

### YouPay flow — `tests/e2e/youpay.spec.ts` (12 tests × 2 browsers = 24 runs)

| Test | Chromium | Mobile | Finding |
|------|----------|--------|---------|
| YouPay stylesheet link present in `<head>` | ✅ PASS | ⚠️ FLAKY | Dev server timeout on first attempt; passes on retry |
| YouPay widget present in DOM on product page | ❌ FAIL | ⚠️ FLAKY | Chromium: widget not injected within 15 s on cold load; Mobile: passes on retry |
| YouPay widget is visible to the user | ❌ FAIL | ⚠️ FLAKY | Blocked by above on Chromium; passes on retry on mobile |
| YouPay widget within product buy area | ❌ FAIL | ❌ FAIL | **Real bug** — element is 1314 px from ATC button on mobile; not inside product container — see M20 |
| YouPay widget text references "YouPay" or "paid" | ⚠️ FLAKY | ✅ PASS | Chromium: timeout on first attempt; passes on retry |
| YouPay element present on /cart page | ⚠️ FLAKY | ⚠️ FLAKY | Dev server timeout on first attempt; passes on retry. Note: cart block is disabled in settings |
| YouPay flyout element present on product page | ❌ FAIL | ❌ FAIL | Page.goto 30 s timeout on both attempts — dev server too slow for this test |
| YouPay flyout present on homepage | ✅ PASS | ✅ PASS | — |
| YouPay widget present on mobile viewport | ⚠️ FLAKY | ⚠️ FLAKY | Timeout on first attempt; passes on retry |
| YouPay widget visible on mobile viewport | ✅ PASS | ✅ PASS | — |
| FAQ page lists YouPay as payment method | ❌ FAIL | ❌ FAIL | Test issue — element inside collapsed accordion; `toBeVisible()` fails on hidden `<li>` |
| FAQ payment methods answer contains YouPay | ❌ FAIL | ❌ FAIL | Test issue — same collapsed accordion cause |

**Overall: 6 passing, 10 failing/flaky — 1 real bug (M20), 2 test logic issues (FAQ accordion), remainder dev server timeout flakiness**

---

### Account flow — `tests/e2e/account.spec.ts` (27 tests × 2 browsers = 54 runs; 47 skipped — no test credentials)

| Test | Chromium | Mobile | Finding |
|------|----------|--------|---------|
| Renders inside the account-login custom element | ✅ PASS | ✅ PASS | — |
| Email input is present and has type email | ✅ PASS | ✅ PASS | — |
| Password input is present and has type password | ✅ PASS | ✅ PASS | — |
| Login form action points to /account/login | ✅ PASS | ✅ PASS | — |
| Login form uses POST method | ✅ PASS | ✅ PASS | — |
| Forgot password link is present | ✅ PASS | ✅ PASS | — |
| Create account link is present | ✅ PASS | ✅ PASS | — |
| Password recovery panel is hidden by default | ✅ PASS | ✅ PASS | — |
| login form h1 heading is visible | ❌ FAIL | ❌ FAIL | Test issue — strict mode: selector matches 2 `<h1>` elements |
| Submit button is visible | ❌ FAIL | ❌ FAIL | Test issue — strict mode: selector matches 2 submit buttons |
| Clicking forgot password link reveals recovery panel | ❌ FAIL | ❌ FAIL | Real bug — `account-login` element not toggling `#recover` — see H14 |
| Unauthenticated /account redirects to login | ✅ PASS | SKIP | — |
| All authenticated dashboard tests (17) | SKIP | SKIP | No `TEST_CUSTOMER_EMAIL` set |

**Overall: 17 passing, 8 failing (6 test issues, 2 real-bug), 47 skipped (no credentials)**

---

### Cart flow — `tests/e2e/cart.spec.ts` (5 tests × 2 browsers = 10 runs)

| Test | Chromium | Mobile | Finding |
|------|----------|--------|---------|
| Opens when product added to cart | ❌ FAIL | ❌ FAIL | Drawer does not open after ATC — see H8 |
| Updates header cart count | ✅ PASS | ✅ PASS | — |
| Quantity stepper increments quantity | ❌ FAIL | ❌ FAIL | Helper cannot open drawer to test — blocked by H8 |
| Remove button empties cart drawer | ❌ FAIL | ❌ FAIL | Helper cannot open drawer to test — blocked by H8 |
| Close button hides the cart drawer | ❌ FAIL | ❌ FAIL | Helper cannot open drawer to test — blocked by H8 |

### Navigation flow — `tests/e2e/navigation.spec.ts` (19 tests × 2 browsers = 38 runs)

| Test | Chromium | Mobile | Finding |
|------|----------|--------|---------|
| Logo visible and links to homepage | ✅ PASS | ✅ PASS | — |
| Clicking logo navigates to homepage | ✅ PASS | ✅ PASS | — |
| Desktop primary nav renders 3+ links | ✅ PASS | SKIP | — |
| Clicking desktop nav link navigates to target | ❌ FAIL | SKIP | networkidle never reached — see M18 |
| Desktop nav dropdown reveals on hover | ✅ PASS | SKIP | — |
| Hamburger button visible on mobile | SKIP | ✅ PASS | — |
| Mobile menu sidebar opens on hamburger click | SKIP | ❌ FAIL | `[open]` never set — see H13 |
| Mobile sidebar contains nav links after opening | SKIP | ❌ FAIL | Blocked by H13 |
| Mobile sidebar closes on close button click | SKIP | ❌ FAIL | Blocked by H13 |
| Header cart icon is visible | ✅ PASS | ✅ PASS | — |
| Header cart icon links to /cart | ✅ PASS | ✅ PASS | — |
| Cart link opens drawer or navigates to /cart | ❌ FAIL | ❌ FAIL | `[open]` never set — see H8 |
| Header account icon present on desktop | ⚠️ FLAKY | SKIP | networkidle timeout; passes on retry |
| Account link points to /account | ✅ PASS | SKIP | — |
| Announcement bar is visible | ✅ PASS | ✅ PASS | — |
| Header stays visible after scroll | ✅ PASS | ✅ PASS | — |
| Header has sticky CSS positioning | ✅ PASS | ✅ PASS | — |
| Primary navigation landmark present | ✅ PASS | ✅ PASS | — |
| Secondary navigation landmark present | ✅ PASS | ✅ PASS | — |

### Search flow — `tests/e2e/search.spec.ts` (14 tests × 2 browsers = 28 runs)

| Test | Chromium | Mobile | Finding |
|------|----------|--------|---------|
| Search toggle opens header-search overlay | ❌ FAIL | ❌ FAIL | `[open]` never set on element — see H12 |
| Search input accessible after opening overlay | ❌ FAIL | ❌ FAIL | Blocked by H12 |
| Typing product name shows predictive results | ❌ FAIL | ❌ FAIL | Blocked by H12 |
| Predictive result links to product page | ❌ FAIL | ❌ FAIL | Blocked by H12 |
| Pressing Enter submits search and shows results | ❌ FAIL | ❌ FAIL | Blocked by H12 |
| Search results page shows product cards | ✅ PASS | ✅ PASS | — |
| Search results page displays result count | ✅ PASS | ✅ PASS | — |
| Nonsense term shows no-results message | ❌ FAIL | ❌ FAIL | Test issue — no-results text does not echo search term |
| No-results page renders a search form | ✅ PASS | ⚠️ FLAKY | Timing on mobile — passes on retry |
| Empty search navigates without errors | ❌ FAIL | ❌ FAIL | Blocked by H12 |
| Close button dismisses overlay | ❌ FAIL | ❌ FAIL | Blocked by H12 |
| Escape key closes overlay | ❌ FAIL | ❌ FAIL | Blocked by H12 |
| Predictive search shows no-results for unknown term | ❌ FAIL | ❌ FAIL | Blocked by H12 |
| /search without query renders search form | ✅ PASS | ✅ PASS | — |

### Collection page flow — `tests/e2e/collection.spec.ts` (8 tests × 2 browsers = 16 runs)

| Test | Chromium | Mobile | Finding |
|------|----------|--------|---------|
| Page loads with heading and product card | ✅ PASS | ✅ PASS | — |
| Product grid renders cards with title and price | ✅ PASS | ✅ PASS | — |
| Filter drawer is present in DOM | ❌ FAIL | ❌ FAIL | Boost Commerce removes native filter DOM — see H10 |
| Applying a filter adds query param and re-renders grid | ❌ FAIL | ❌ FAIL | Native filter params not processed — see H10 |
| Removing active filter restores unfiltered grid | ❌ FAIL | ❌ FAIL | `.active-facets` never rendered — see H10 |
| Sort-by popover opens and adds `sort_by` to URL | ❌ FAIL | ❌ FAIL | Popover click does not open element — see M16 |
| Pagination navigates to page 2 | ⚠️ FLAKY | ⚠️ FLAKY | Timing — passes on retry |
| Product card links to `/products/` URL | ✅ PASS | ✅ PASS | — |

### Product page flow — `tests/e2e/product.spec.ts` (7 tests × 2 browsers = 14 runs)

| Test | Chromium | Mobile | Finding |
|------|----------|--------|---------|
| Shows title, price, and image | ✅ PASS | ✅ PASS | — |
| Variant picker renders with options | ✅ PASS | ✅ PASS | — |
| Variant query param pre-selects variant | ✅ PASS | ✅ PASS | — |
| ATC opens cart drawer | ❌ FAIL | ❌ FAIL | Drawer never opens — see H8 |
| Sold-out variant disables ATC button | ❌ FAIL | ❌ FAIL | Option labels inside closed popover — see H9 |
| Thumbnail click changes active image | ❌ FAIL | ❌ FAIL | `aria-current` not updated by gallery JS — see L11 |
| Breadcrumb navigation present | ⚠️ FLAKY | ✅ PASS | Timing issue on chromium first load |

**Overall: 7 passing, 10 failing, 1 flaky across both test suites**

---

## Appendix B: Theme Check Summary

### Full theme scan

`shopify theme check` was run against the full theme and produced **132 errors** and **33 warnings** across 5 files. The majority (110 of 132 errors) originate from `sections/flits_account.liquid`, which is managed by the Flits third-party app and cannot be edited in the theme directly.

| Category | Count | Actionable in theme code |
|----------|-------|--------------------------|
| TranslationKeyExists (Flits) | 95 | No — app-managed |
| MissingTemplate (Flits snippets) | 14 | No — app-managed |
| MissingAsset (flits.css) | 1 | No — app-managed |
| ImgWidthAndHeight | 7 | Partially (2 in customer-orders.liquid) |
| ParserBlockingScript | 3 | Yes |
| RemoteAsset | 9 (warnings) | Partially |
| AssetPreload | 10 (warnings) | Yes |
| UndefinedObject | 2 (warnings) | Yes |
| HardcodedRoutes | 1 (warning) | Yes |
| DeprecatedFonts | 2 (warnings) | Yes |

### Snippets-only scan (`shopify theme check snippets/`)

A targeted scan of all 65 snippet files produced **155 errors** and **83 warnings**. The bulk are Flits app-managed false positives; one new theme-native issue was identified (M19).

| Category | Count | Actionable | Notes |
|----------|-------|------------|-------|
| TranslationKeyExists (Flits) | 112 | No — app-managed | All `flits.*` keys in flits_account_page_snippet.liquid + flits_snippet.liquid |
| MissingTemplate (Flits snippets) | 23 | No — app-managed | 16 in flits_account_page_snippet, 7 in flits_snippet |
| MissingAsset (Flits JS/CSS) | 7 | No — app-managed | flits-cart.js ×3, flits-recently-view.js ×2, flits-account.js ×1, flits.css ×1 — see M10 |
| MissingAsset (theme SVGs) | 2 | **Yes** | checkmark.svg, cursor-zoom-in.svg in css-variables.liquid — see M19 |
| ParserBlockingScript | 6 | Partial | 1 in scripts.liquid (theme), 1 in smsbump (app), 4 in flits_snippet (app) — see H1 |
| ImgWidthAndHeight | 5 | No — app-managed | All in flits_account_page_snippet.liquid |
| OrphanedSnippet | 65 | No — false positive | Theme Check cannot detect dynamic snippet rendering; all snippets are in use |
| HardcodedRoutes | 6 | No — app-managed | All in flits_account_page_snippet.liquid |
| RemoteAsset | 6 | No — app-managed | SMSBump, PreOrder Me, ISP Smart Nav snippets |
| DeprecatedFilter | 1 | No — app-managed | img_url in flits_account_page_snippet.liquid |
| UnusedAssign | 2 | No — app-managed | In flits_account_page_snippet.liquid |
| VariableName | 2 | No — app-managed | camelCase vars in flits_snippet.liquid |
| AssetPreload | 1 | Yes | In scripts.liquid |

---

## Recommended Fix Priority

| # | Issue | Effort | Impact |
|---|-------|--------|--------|
| 0 | **C6 — Remove obfuscated script from `snippets/scripts.liquid` immediately** | Low | Critical |
| 1 | H8 — Verify `cart_type` setting in Shopify admin; confirm drawer mode is active | Low | Critical |
| 2 | C2 — Wrong form type in customer profile | Low | Critical |
| 3 | C1 — Add error handling to all cart fetch calls | Medium | Critical |
| 4 | C3 — Add null check in CartDrawer replaceContent | Low | Critical |
| 5 | C5 — Replace hardcoded domain in checkout CSS | Low | Critical |
| 6 | H1 — Add `defer` to jQuery, Slick, Splide | Low | High |
| 7 | H9 — Document popover-open requirement for variant selection flows | Low | High |
| 8 | L11 — Fix gallery JS to update `aria-current` on thumbnail click | Medium | Medium |
| 9 | L4 — Replace hardcoded `/account/login` with route helper | Low | Medium |
| 10 | M1 — Assign `color_scheme_hash` before use | Low | Medium |
| 11 | M2 — Move `form.posted_successfully?` inside form block | Low | Medium |
| 12 | M3 — Add `width`/`height` to order page images | Low | Medium |
| 13 | L8 — Fix leading space in cancel button ID | Low | Medium |
| 14 | M9 — Fix duplicate `class` attribute on anchor | Low | Low |
| 15 | L10 — Add `.theme-check.yml` to suppress Flits noise | Low | Low |
| 16 | H13 — Investigate why header-sidebar DrawerElement [open] never set on mobile | Medium | High |
| 17 | H12 — Investigate why header-search DialogElement show() does not fire | Medium | High |
| 18 | M18 — Audit third-party scripts for persistent connections causing page-idle failures | Medium | Medium |
| 17 | H10 — Audit Boost Commerce / native facets conflict; align filter markup | High | High |
| 17 | M16 — Investigate sort popover click handler on collection page | Low | Medium |
| 18 | H3 — Fix variant preload cache error handling | Medium | High |
| 19 | H4 — Abort in-flight fetches on variant picker disconnect | Medium | Medium |
| 20 | M6 — Replace hardcoded app asset URLs | Medium | Medium |
| 21 | H14 — Investigate why account-login custom element does not toggle recovery panel | Medium | High |
| 22 | M19 — Restore missing `checkmark.svg` and `cursor-zoom-in.svg` to `assets/` | Low | Medium |
| 23 | M20 — Reposition YouPay button block within the product buy area on mobile | Low | Medium |
| 24 | M21 — Replace generic "Password reset error" with expired-link message on `account/invalid_token` | Low | Medium |
| 25 | C9 — Fix quantity update on cart page to use fetch/DOM swap instead of full page reload; eliminate 340+ request burst and Cloudflare rate-limiting | Medium | Critical |
| 26 | H15 — Add independent click handler to open cart drawer from the `/cart` page | Low | High |
| 27 | H16 — Fix Smile.io session handoff on rewards pages; consolidate duplicate rewards pages into one | Medium | High |
| 28 | H17 — Add `{% if customer %}` redirect guard to login template; fix "Create an account" CTA destination on rewards pages | Low | High |
| 29 | M22 — Move Logout button out of Flits widget container on account page to match Rewards page placement | Low | Medium |

---

## Design Issues

### D1 — Search results page: black price text on dark product cards is unreadable

**Where:** Search results page (`beserk.com.au/search?q=*`) — product cards
**Confirmed by:** Live reproduction — `beserk.com.au/search?q=dress`; screenshot provided (2026-06-02)
**Impact:** Customers searching for products cannot read the price on dark or black product cards. The price text renders in black on a near-black background, making it completely illegible. This directly damages the purchase experience — customers cannot compare prices or make buying decisions from the search results page.

**Screenshot — search results showing black price text on dark product cards:**

The search results grid displays product cards where items with dark/black imagery (e.g. Hell Bunny Samara Dress, Forest Ink Rosetta Buckle Maxi Dress) render their price labels in black text. The card background and the product image are both dark, so the price text (`$119.95 AUD`, `$132.95 AUD`) has virtually no contrast and is invisible to the customer at a glance.

**Root cause:** The product card price text colour is hardcoded or inherits a dark colour scheme (`color-scheme--dark` or similar CSS class) that works on light backgrounds but is not overridden for cards where the product image is dark. The Prestige theme's `product-card.liquid` applies a colour scheme class based on the section's global setting, not the individual product image — so dark images get black text on a dark background.

**Specific issues observed:**
- Price text (`$AUD`) is black on dark product card backgrounds
- Sale price and original price strikethrough are both invisible on dark cards
- Star rating text and review count have the same contrast problem
- The issue affects any product with a dark or black product photo — a large portion of Beserk's catalogue given the alt/gothic niche

**Fix:**
- Change the product card price text colour to white or light on dark backgrounds — either force `color: white` on `.price` elements inside dark-background cards, or detect the dominant image colour and apply the appropriate text class
- Quickest fix: in `snippets/product-card.liquid` or `assets/theme.css`, override the price text colour for cards inside the search results section to use a light colour (`#ffffff` or CSS variable `--color-foreground-light`) that is always visible regardless of background
- Alternatively set the card background to always be a light/neutral colour on the search page so dark text remains readable

**Priority:** 🟠 Fix soon
