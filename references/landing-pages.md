# Custom Landing Pages

Every Yard product has a public landing page. **Pro** sellers can replace the default layout with a custom page — plain HTML, CSS, JS (and images/fonts) bundled in a `.yard/landing-page/` directory and uploaded with the `yard page …` commands (check with `yard me --json` → `.is_pro`).

This document covers what you can put **inside** that bundle: the product data your page can read at runtime, the helper functions for wiring up checkout/trial buttons, and the limits the bundle has to fit within. For the commands that scaffold and publish the bundle, see [cli-commands.md](./cli-commands.md).

---

## Runtime data: `window.yard.product`

When a custom landing page is rendered, Yard makes the product's data available to your code as a synchronous JavaScript object — no `fetch`, no API key, no async wait:

```js
window.yard.product   // the product (or null if data couldn't be loaded)
```

The object matches the same shape returned by `GET /v1/products/{slug}/public`. The most useful fields:

| Field | Type | Notes |
|---|---|---|
| `slug` | `string` | URL-safe product identifier |
| `title` | `string` | Display name |
| `tagline` | `string?` | Short marketing line |
| `description` | `string?` | Markdown source of the long description |
| `description_html` | `string?` | Server-rendered HTML for `description` |
| `readme_html` | `string?` | Server-rendered HTML of the linked repo's README, if any |
| `price_cents` | `number` | Default tier price, in cents |
| `discounted_price_cents` | `number?` | Effective price after any state discount |
| `state` | `string` | `draft`, `early_access`, or `released` |
| `state_discount_percent` | `number?` | Active state-discount percent, if any |
| `tiers` | `PricingTier[]` | All pricing tiers — see below |
| `images` | `ProductImage[]` | Uploaded screenshots/icons; each has a `url` |
| `category` | `string?` | Optional category label |
| `faq` | `{ question, answer }[]` | Seller-defined FAQ entries |
| `metadata` | `{ key, value }[]` | Seller-defined free-form metadata pairs |
| `free_trial_enabled` | `boolean` | Whether free trials are offered |
| `free_trial_days` | `number?` | Trial length in days, if enabled |
| `gift_enabled` | `boolean` | Whether gift purchases are allowed |
| `license_key_enabled` | `boolean` | Whether license keys are issued |
| `latest_release` | `PublicReleaseInfo?` | Most recent published release (tag, name, notes, date) |
| `release_count` | `number` | Total published releases |
| `seller` | `{ username, avatar_url?, … }` | Public seller info |

Each entry in `tiers` exposes:

```jsonc
{
  "id": "uuid",                  // pass to data-tier-id / window.yard.checkout({ tier })
  "name": "Pro",
  "description": "…",            // optional, may be null
  "price_cents": 4900,
  "is_default": true,
  "seat_type": "single",         // "single" | "fixed_pack" | "per_seat"
  "seat_count": null,            // set for fixed_pack
  "min_seats": null,             // set for per_seat
  "max_seats": null,
  "pricing_model": "one_time",   // "one_time" | "subscription"
  "yearly_discount_percent": null,
  "features": ["…", "…"],
  "volume_brackets": []          // per_seat tiers may define quantity discounts
}
```

> **Heads-up:** `window.yard.product` reflects the **saved** product state. While you're editing in the dashboard, the preview iframe won't pick up unsaved edits to product fields — save first, then refresh the preview.

---

## Zero-JS HTML hooks

For most pages you don't need to write any JavaScript. Two attribute conventions cover the common cases:

### `data-yard` — bind product fields to text

Put a dotted path to any field on `product` in a `data-yard` attribute. On page load, the element's `textContent` is set to that value:

```html
<h1 data-yard="title">Loading…</h1>
<p data-yard="tagline"></p>
<span>$<span data-yard="price_cents"></span> (in cents)</span>

<!-- Dotted paths work into arrays and nested objects -->
<strong data-yard="tiers.0.name"></strong>
<em data-yard="seller.username"></em>
```

If the field is missing or `null`, the element is left untouched. Add `data-yard-html` on the same element to write `innerHTML` instead of `textContent` — useful for `description_html` or `readme_html`:

```html
<article data-yard="description_html" data-yard-html></article>
```

After dynamic re-renders (e.g. you injected new DOM yourself), call `window.yard.refresh()` to re-bind any new `data-yard` nodes.

### `data-action` — wire up checkout / trial buttons

Add `data-action="checkout"` (or `"trial"`) to any clickable element and Yard handles the redirect:

```html
<!-- Default tier, no extra options -->
<button data-action="checkout">Buy now</button>

<!-- Specific tier (UUID from product.tiers[i].id) -->
<button data-action="checkout" data-tier-id="…uuid…">Buy Pro</button>

<!-- Subscription tier with billing interval -->
<button data-action="checkout" data-tier-id="…" data-interval="yearly">
  Subscribe yearly
</button>

<!-- Per-seat tier with quantity, sent as a gift -->
<button data-action="checkout" data-tier-id="…" data-quantity="5" data-gift>
  Buy 5 seats as a gift
</button>

<!-- Free trial -->
<button data-action="trial">Start free trial</button>
```

Recognised attributes on `data-action="checkout"` elements:

| Attribute | Meaning |
|---|---|
| `data-tier-id` (or `data-tier`) | Tier UUID. Omit to use the default tier. |
| `data-interval` | `monthly` or `yearly` (subscription tiers only). |
| `data-quantity` | Seat count for `fixed_pack` / `per_seat` tiers. |
| `data-gift` | Presence of the attribute opens the gift-purchase flow. |

Clicks on `data-action` elements have their default behaviour prevented automatically — there's no need for the surrounding `<a>` to point anywhere.

---

## JavaScript API: `window.yard`

If the attribute conventions don't fit your page, drive everything from JS:

```js
window.yard = {
  product,                 // PublicProduct | null  (see above)
  checkoutBase,            // string, e.g. "https://yard.sh"

  checkout(opts),          // top-level redirect to checkout
  trial(),                 // top-level redirect to the trial flow

  checkoutURL(opts),       // build the checkout URL without redirecting
  trialURL(),              // build the trial URL without redirecting

  ownership(),             // Promise<OwnershipState | null> — buyer state
                           // (signed-in, owned, tier). See "Buyer state" below.

  refresh(),               // re-run data-yard binding (after dynamic DOM updates)
}
```

`opts` for `checkout` / `checkoutURL`:

```ts
{
  tier?: string,        // tier UUID (or pass tierId, same thing)
  interval?: "monthly" | "yearly",
  quantity?: number,
  gift?: boolean,
}
```

Examples:

```js
// Render tier buttons dynamically — no hardcoded UUIDs in HTML
for (const tier of window.yard.product.tiers) {
  const btn = document.createElement('button');
  btn.textContent = `${tier.name} — $${(tier.price_cents / 100).toFixed(2)}`;
  btn.addEventListener('click', () => window.yard.checkout({ tier: tier.id }));
  document.querySelector('#tiers').append(btn);
}

// Conditionally show a "Start free trial" CTA
if (window.yard.product.free_trial_enabled) {
  document.querySelector('#trial-cta').hidden = false;
}

// Build a shareable checkout URL (e.g. for a copy-to-clipboard button)
const url = window.yard.checkoutURL({ tier: defaultTier.id, quantity: 3 });
```

---

## Buyer state: `window.yard.ownership()`

`window.yard.product` is **product** data — same for every visitor.
`window.yard.ownership()` is **buyer** data — specific to the visitor:
are they signed in to yard, and do they own this product? Useful for
swapping a "Buy" button for "Open in Library", showing the buyer's
avatar, gating gated content, or branching on which tier they hold.

It returns a `Promise` (memoized — call it as many times as you like):

```js
const state = await window.yard.ownership();
// state may be null on a yard.sh-direct page where no bridge is needed,
// or in browsers that block third-party cookies on a custom merchant
// domain. Always null-check.
if (state?.owned) { /* … */ }
```

Resolved shape:

| Field | Type | Notes |
|---|---|---|
| `signed_in` | `boolean` | Is the visitor signed in to yard at all? |
| `user` | `{ id, username, avatar_url } \| null` | Minimal profile; `null` when signed out. |
| `owned` | `boolean` | Does this user own this product (any tier, paid or free)? Active trials and active subscriptions count. |
| `is_trial` | `boolean` | True when the active entitlement is a free trial. |
| `is_subscription` | `boolean` | True when the active entitlement is a subscription. |
| `transaction_id` | `string \| null` | The transaction or subscription ID. Useful as an opaque entitlement reference. |
| `tier_id` | `string \| null` | UUID of the tier they hold. Match against `window.yard.product.tiers[i].id` to know **which** tier. |
| `tier_name` | `string \| null` | Display name of the held tier (e.g. `"Pro"` or `"Monthly"`). Handy for UI copy. |

### Zero-JS shortcuts: `data-yard-when`

For the most common case ("show this when signed in / owned, hide
otherwise") you don't need to call the API yourself. Add
`data-yard-when` to any element:

```html
<!-- Visible only after the bridge resolves with signed_in: true -->
<a data-yard-when="signed_in" href="https://yard.sh/library">Open library</a>

<!-- Visible only when the visitor is signed out (default until resolved) -->
<a data-yard-when="signed_out" href="https://yard.sh/login">Sign in</a>

<!-- Show "Open in Library" once we know they own it -->
<button data-yard-when="owned" data-yard="tier_name">Open Pro</button>

<!-- Hide the Buy button once we know they own it -->
<button data-yard-when="not_owned" data-action="checkout">Buy</button>
```

Recognized values: `signed_in`, `signed_out`, `owned`, `not_owned`.
Until the bridge resolves, `data-yard-when` elements are hidden by
default — so a non-owner never briefly sees an "Open in Library" CTA
on first paint.

### Common patterns

```js
// Show user avatar in the corner
const state = await window.yard.ownership();
if (state?.signed_in) {
  document.querySelector('#avatar').src = state.user.avatar_url;
  document.querySelector('#username').textContent = state.user.username;
}

// Branch by tier
const state = await window.yard.ownership();
if (state?.owned) {
  const proTier = window.yard.product.tiers.find((t) => t.name === 'Pro');
  if (state.tier_id === proTier?.id) {
    showProFeatures();
  } else {
    showStandardFeatures();
  }
}

// Subscription self-service link
const state = await window.yard.ownership();
if (state?.is_subscription) {
  document.querySelector('#manage-sub').href =
    `https://yard.sh/library/${window.yard.product.slug}/subscription`;
}
```

### What state is **not** included (and why)

The bridge intentionally returns the minimum needed to render
buyer-specific UI. It does **not** expose: the visitor's email,
their other purchases, their roles, payment-method details, or any
other yard account data — that information stays on yard.sh.

If you need a deeper integration (e.g. you want to call yard's API
from your own backend on behalf of the user), use API keys and the
yard REST API — see [api-reference.md](./api-reference.md). The
ownership bridge is for read-only UI gating.

### Caveat: third-party cookies on custom domains

In browsers with strict third-party cookie blocking (Safari ITP and
some privacy modes), the cross-origin lookup from a custom merchant
domain back to yard.sh may not see the visitor's session even when
they're signed in. The bridge will resolve with `null` or
`signed_in: false` in that case — your UI should fall back gracefully
(default to the "Buy" CTA instead of "Open in Library"). Pages on
`<handle>.yard.sh` subdomains aren't affected — they're same-site
with yard.sh.

---

## Asset paths in your HTML

Pages serve under `<handle>.yard.sh/<slug>/`, so any CSS, JS, image, or font your `index.html` references needs to resolve inside that directory. The simplest rule: **use relative URLs**.

```html
<!-- Good — resolves to <handle>.yard.sh/<slug>/styles.css -->
<link rel="stylesheet" href="styles.css" />
<script type="module" src="app.js"></script>
<img src="screenshot.png" alt="" />
```

Avoid bare root-relative paths (`href="/styles.css"`, `src="/app.js"`) — those drop the slug and resolve to `<handle>.yard.sh/styles.css`, which isn't part of your bundle and will 404. If you have to use a leading slash, prefix the slug: `href="/<slug>/styles.css"`. Relative URLs are easier and survive renaming the product, which is what `yard page init` scaffolds.

---

## Bundle constraints (recap)

The same limits apply whether you upload via `yard page push` or the dashboard editor:

| Limit | Value |
|---|---|
| Files per bundle | 20 |
| Max size per file | 1 MB |
| Max total bundle size | 5 MB |
| Allowed extensions | `.html .css .js .json .svg .png .jpg .jpeg .webp .gif .woff2` |
| Path rules | letters/digits/`._-` only, at most one subdirectory level, no dotfiles |
| Required file | `index.html` (must exist before you can publish) |

Anything outside these constraints is rejected client-side by `yard page push` before any upload happens.

---

## Worked example

A complete one-file landing page for a single-tier product:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title data-yard="title">Loading…</title>
    <link rel="stylesheet" href="styles.css" />
  </head>
  <body>
    <header>
      <h1 data-yard="title"></h1>
      <p class="tagline" data-yard="tagline"></p>
    </header>

    <article data-yard="description_html" data-yard-html></article>

    <section class="cta">
      <button data-yard-when="not_owned" data-action="checkout">Buy now</button>
      <button data-yard-when="not_owned" data-action="trial" id="trial-btn" hidden>Start free trial</button>
      <a data-yard-when="owned" href="https://yard.sh/library">Open in your library</a>
    </section>

    <script>
      // Hide the trial button if trials aren't enabled for this product.
      // (Independent of ownership — `data-yard-when="not_owned"` already
      // hides it for buyers.)
      if (window.yard.product?.free_trial_enabled) {
        document.querySelector('#trial-btn').hidden = false;
      }
    </script>
  </body>
</html>
```

---

## Cross-references

- [cli-commands.md → `yard page`](./cli-commands.md#yard-page) — the commands to scaffold, push, publish, and revert the bundle
- [pricing-and-licensing.md](./pricing-and-licensing.md) — what the tier shapes (`one_time` / `subscription`, `single` / `fixed_pack` / `per_seat`, volume brackets) actually mean
- [api-reference.md](./api-reference.md) — the public product endpoint that backs `window.yard.product`
