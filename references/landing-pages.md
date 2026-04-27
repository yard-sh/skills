# Custom Landing Pages

Every Yard product has a public landing page. **Pro** sellers can replace the default layout with a custom page — plain HTML, CSS, JS (and images/fonts) bundled in a `.yard/landing-page/` directory and uploaded with the `yard page …` commands.

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
| `state` | `string` | `draft`, `pre_order`, `early_access`, or `released` |
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
      <button data-action="checkout">Buy now</button>
      <button data-action="trial" id="trial-btn" hidden>Start free trial</button>
    </section>

    <script>
      // Hide the trial button if trials aren't enabled for this product.
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
