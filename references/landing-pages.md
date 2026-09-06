# Custom Landing Pages

Every Yard project has a public landing page. **Pro** sellers can replace the default layout with a custom page: plain HTML, CSS, JS (and images/fonts) bundled in the project's landing-page directory (default `.yard/landing-page/`, configurable via `landing_page.dir` in `.yard/settings.json`, and selected by `"type": "custom"` on that same block) and uploaded with the `yard push / yard pull` commands (check with `yard me --json` → `.team_permissions`).

This document covers what you can put **inside** that bundle: the project data your page can read at runtime, the helper functions for wiring up checkout/trial buttons, and the limits the bundle has to fit within. For the commands that scaffold and publish the bundle, see [cli-commands.md](./cli-commands.md).

---

## Runtime data: `window.yard.project`

When a custom landing page is rendered, Yard makes the project's data available to your code as a synchronous JavaScript object — no `fetch`, no API key, no async wait:

```js
window.yard.project   // the project (or null if data couldn't be loaded)
```

The object is the JSON returned by `GET /v1/projects/{username}/{slug}/public` — same snake_case field names (no camelization happens between the response and `window.yard.project`). The most useful fields:

| Field | Type | Notes |
|---|---|---|
| `slug` | `string` | URL-safe project identifier |
| `title` | `string` | Display name |
| `tagline` | `string?` | Short marketing line |
| `description` | `string?` | Markdown source of the long description |
| `description_html` | `string?` | Server-rendered HTML for `description` |
| `price_cents` | `number` | Default tier price, in cents |
| `discounted_price_cents` | `number?` | Effective price after any launch stage discount |
| `launch_stage` | `string` | `draft`, `early_access`, or `published` |
| `stage_discount_percent` | `number?` | Active launch-stage discount percent, if any |
| `tiers` | `PricingTier[]` | All pricing tiers — see below |
| `images` | `ProjectImage[]` | Uploaded screenshots/icons; each has a `url` |
| `category` | `string?` | Optional category label |
| `faq` | `{ question, answer }[]` | Seller-defined FAQ entries |
| `metadata` | `{ key, value }[]` | Seller-defined free-form metadata pairs |
| `license_key_enabled` | `boolean` | Whether license keys are issued |
| `latest_release` | `PublicReleaseInfo?` | Most recent published release (tag, name, notes, date) |
| `release_count` | `number` | Total published releases |
| `seller` | `{ username, avatar_url?, … }` | The owning **team**: `username` carries the team's username and `avatar_url` its icon. |

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
  "free_trial_enabled": false,   // free trials are configured PER TIER, not per project
  "free_trial_days": null,       // trial length in days, when free_trial_enabled
  "trial_requires_card": true,   // per-tier: subscription-tier trials collect a card via checkout
  "gift_enabled": false,         // per-tier: whether this tier can be bought as a gift
  "volume_brackets": []          // per_seat tiers may define quantity discounts
}
```

> **Free trials are per-tier.** There is no project-level trial flag — a project
> "offers a trial" when at least one of its `tiers` has `free_trial_enabled: true`
> and `free_trial_days > 0`. Gate your trial CTA on a tier, not on the project (see
> the [worked example](#worked-example)), and pass that tier's `id` to the trial button.
> To inspect this from the CLI before wiring the page, run `yard projects show <slug> --json`
> and read `.tiers[]` — `yard projects --json` only returns project-level fields.

> **Heads-up:** `window.yard.project` reflects the **saved** state of the release you are editing. While you're editing in the dashboard, the preview iframe won't pick up unsaved edits — save first, then refresh the preview.

---

## Zero-JS HTML hooks

For most pages you don't need to write any JavaScript. Two attribute conventions cover the common cases:

### `data-yard` — bind project fields to text

Put a dotted path to any field on `project` in a `data-yard` attribute. On page load, the element's `textContent` is set to that value:

```html
<h1 data-yard="title">Loading…</h1>
<p data-yard="tagline"></p>
<span>$<span data-yard="price_cents"></span> (in cents)</span>

<!-- Dotted paths work into arrays and nested objects -->
<strong data-yard="tiers.0.name"></strong>
<em data-yard="seller.username"></em>
```

If the field is missing or `null`, the element is left untouched. Add `data-yard-html` on the same element to write `innerHTML` instead of `textContent`, useful for `description_html`:

```html
<article data-yard="description_html" data-yard-html></article>
```

After dynamic re-renders (e.g. you injected new DOM yourself), call `window.yard.refresh()` to re-bind any new `data-yard` nodes.

### `data-action` — wire up checkout / trial buttons

Add `data-action="checkout"` (or `"trial"`) to any clickable element and Yard handles the redirect:

```html
<!-- Default tier, no extra options -->
<button data-action="checkout">Buy now</button>

<!-- Specific tier (UUID from project.tiers[i].id) -->
<button data-action="checkout" data-tier-id="…uuid…">Buy Pro</button>

<!-- Subscription tier with billing interval -->
<button data-action="checkout" data-tier-id="…" data-interval="yearly">
  Subscribe yearly
</button>

<!-- Per-seat tier with quantity, sent as a gift -->
<button data-action="checkout" data-tier-id="…" data-quantity="5" data-gift>
  Buy 5 seats as a gift
</button>

<!-- Free trial (default / first trial-enabled tier) -->
<button data-action="trial">Start free trial</button>

<!-- Free trial for a specific tier -->
<button data-action="trial" data-tier-id="…uuid…">Start Pro trial</button>
```

Recognised attributes on `data-action="checkout"` elements:

| Attribute | Meaning |
|---|---|
| `data-tier-id` (or `data-tier`) | Tier UUID. Omit to use the default tier. |
| `data-interval` | `monthly` or `yearly` (subscription tiers only). |
| `data-quantity` | Seat count for `fixed_pack` / `per_seat` tiers. |
| `data-gift` | Presence of the attribute opens the gift-purchase flow. |

`data-action="trial"` accepts one attribute:

| Attribute | Meaning |
|---|---|
| `data-tier-id` (or `data-tier`) | Tier UUID to trial. Since trials are per-tier, set this to the id of a tier whose `free_trial_enabled` is true. Omit it to let yard use the default tier (or the first trial-enabled tier if the default has no trial). |

A trial click redirects to yard's hosted trial flow (`/trial/<username>/<slug>`): a signed-in
visitor's trial starts immediately, while a signed-out visitor gets an email-confirmation step.
Only show the trial button when a tier actually offers a trial — see the worked example.

**Hooking up an existing scaffold:** if your HTML already has `data-action="trial"` (or `"checkout"`) on a button, the click is already wired by `embed.js` — there is no JS handler to attach. The only work left is **visibility**: hide the trial button when no tier has `free_trial_enabled: true`, and (when revealing it) set `data-tier-id` to the trial-enabled tier's id so the redirect targets the right tier.

Clicks on `data-action` elements have their default behaviour prevented automatically — there's no need for the surrounding `<a>` to point anywhere.

---

## JavaScript API: `window.yard`

If the attribute conventions don't fit your page, drive everything from JS:

```js
window.yard = {
  project,                 // PublicProject | null  (see above)
  checkoutBase,            // string, e.g. "https://yard.sh"

  checkout(opts),          // top-level redirect to checkout
  trial(opts),             // top-level redirect to the trial flow

  checkoutURL(opts),       // build the checkout URL without redirecting
  trialURL(opts),          // build the trial URL without redirecting

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

`opts` for `trial` / `trialURL`:

```ts
{
  tier?: string,        // UUID of a trial-enabled tier (or pass tierId).
                        // Omit to use the default / first trial-enabled tier.
}
```

Examples:

```js
// Render tier buttons dynamically — no hardcoded UUIDs in HTML
for (const tier of window.yard.project.tiers) {
  const btn = document.createElement('button');
  btn.textContent = `${tier.name} — $${(tier.price_cents / 100).toFixed(2)}`;
  btn.addEventListener('click', () => window.yard.checkout({ tier: tier.id }));
  document.querySelector('#tiers').append(btn);
}

// Conditionally show a "Start free trial" CTA — trials are per-tier
const trialTier = window.yard.project.tiers.find(
  (t) => t.free_trial_enabled && (t.free_trial_days ?? 0) > 0,
);
if (trialTier) {
  const cta = document.querySelector('#trial-cta');
  cta.dataset.tierId = trialTier.id; // so data-action="trial" trials the right tier
  cta.hidden = false;
}

// Build a shareable checkout URL (e.g. for a copy-to-clipboard button)
const url = window.yard.checkoutURL({ tier: defaultTier.id, quantity: 3 });
```

---

## Buyer state: `window.yard.ownership()`

`window.yard.project` is **project** data — same for every visitor.
`window.yard.ownership()` is **buyer** data — specific to the visitor:
are they signed in to yard, and do they own this project? Useful for
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
| `owned` | `boolean` | Does this user own this project (any tier, paid or free)? Active trials and active subscriptions count. |
| `is_trial` | `boolean` | True when the active entitlement is a free trial. |
| `is_subscription` | `boolean` | True when the active entitlement is a subscription. |
| `transaction_id` | `string \| null` | The transaction or subscription ID. Useful as an opaque entitlement reference. |
| `tier_id` | `string \| null` | UUID of the tier they hold. Match against `window.yard.project.tiers[i].id` to know **which** tier. |
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
  const proTier = window.yard.project.tiers.find((t) => t.name === 'Pro');
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
    `https://yard.sh/library/${window.yard.project.slug}/subscription`;
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
`<username>.yard.sh` subdomains aren't affected — they're same-site
with yard.sh.

---

## Asset paths in your HTML

Pages serve under `<username>.yard.sh/<slug>/`, so any CSS, JS, image, or font your `index.html` references needs to resolve inside that directory. The simplest rule: **use relative URLs**.

```html
<!-- Good — resolves to <username>.yard.sh/<slug>/styles.css -->
<link rel="stylesheet" href="styles.css" />
<script type="module" src="app.js"></script>
<img src="screenshot.png" alt="" />
```

Avoid bare root-relative paths (`href="/styles.css"`, `src="/app.js"`) — those drop the slug and resolve to `<username>.yard.sh/styles.css`, which isn't part of your bundle and will 404. If you have to use a leading slash, prefix the slug: `href="/<slug>/styles.css"`. Relative URLs are easier and survive renaming the project, which is what `yard init --page` scaffolds.

Relative URLs matter more than usual because the same bundle serves under more than one prefix: a sandbox adds a path segment (below). Root-relative paths break there too; relative ones just work.

---

## Testing a page before users see it

Locally, `yard dev` serves the page at `http://localhost:9875/<slug>/` with the
`__yard__` snapshot and `embed.js` injected, so `window.yard.project`,
`data-yard` bindings and Buy buttons behave as hosted (with live project data
when logged in, otherwise a placeholder built from `.yard/settings.json`). See
[local-dev.md](local-dev.md). Once it looks right, push and check it hosted:

The project and each sandbox serve their own landing page. A sandbox serves at `https://<username>.yard.sh/<slug>/@<sandbox>/`; the project itself, which is what buyers reach, stays at `https://<username>.yard.sh/<slug>/`.

```
https://alice.yard.sh/widget/            the project itself
https://alice.yard.sh/widget/@preview/   the preview sandbox
```

Sandbox URLs are **team-only by default**: the Yard edge verifies you belong to the team that owns the project before serving, everyone else gets an explanatory 403, and anonymous visitors are sent through sign-in first. Safe to have in scrollback, and shareable only once you opt in with `yard sandbox visibility public --sandbox <name>`, which lets anyone with the URL view that sandbox.

What you get is decided by the path, so it cannot be switched by a query parameter your page happens to carry, and a URL always says what it serves. `window.yard.project` reflects **that sandbox's** state (or the project's own, with no `/@…/` segment), its own pricing, copy, and gallery, so a preview page shows preview prices, not the storefront's.

Where the serving release has no custom page there is still a landing page: the default one, rendered from the project's or sandbox's own content and edited in the dashboard. So the URL always resolves, whether or not you have shipped a bundle there. Setting `"landing_page": { "type": "default" }` switches a release back to it without deleting your files: they still upload, they just stop being served. Leaving the `landing_page` block out entirely changes neither the type nor the files, so a release you build from an earlier one keeps whatever page it had.

The usual loop:

```
yard push                                  # into your draft release; nothing serves a draft
yard sandbox create preview                # once
yard sandbox pin                           # hold the storefront on the release it serves today
yard releases publish v1.0.0               # tag the draft; the pin keeps it off the storefront
yard sandbox pin v1.0.0 --sandbox preview  # serve it in the sandbox, then browse …/widget/@preview/
yard sandbox unpin                         # let the storefront serve v1.0.0
```

Editing a release that is already being served is live: Yard redeploys it and `yard status` reports stale, then updating, then up to date while it catches up.

---

## Bundle constraints (recap)

The same limits apply whether you upload via `yard push` or the dashboard editor:

| Limit | Value |
|---|---|
| Files per bundle | 20 |
| Max size per file | 1 MB |
| Max total bundle size | 5 MB |
| Allowed extensions | `.html .css .js .json .svg .png .jpg .jpeg .webp .gif .woff2` |
| Path rules | letters/digits/`._-` only, at most one subdirectory level, no dotfiles |
| Required file | `index.html` (must exist before you can publish) |
| Applies | To the project's and each sandbox's bundle separately |

Anything outside these constraints is rejected client-side by `yard push` before any upload happens.

---

## Worked example

A complete one-file landing page for a single-tier project:

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
      // Trials are per-tier: reveal the button only when some tier offers one,
      // and point it at that tier. (Independent of ownership — the
      // `data-yard-when="not_owned"` already hides it for buyers.)
      const trialTier = window.yard.project?.tiers.find(
        (t) => t.free_trial_enabled && (t.free_trial_days ?? 0) > 0,
      );
      if (trialTier) {
        const btn = document.querySelector('#trial-btn');
        btn.dataset.tierId = trialTier.id;
        btn.hidden = false;
      }
    </script>
  </body>
</html>
```

---

## Cross-references

- [cli-commands.md → project sync](./cli-commands.md#yard-push--pull--status--ls) — the commands to scaffold, push, and publish the bundle
- [pricing-and-licensing.md](./pricing-and-licensing.md) — what the tier shapes (`one_time` / `subscription`, `single` / `fixed_pack` / `per_seat`, volume brackets) actually mean
- [api-reference.md](./api-reference.md) — the public project endpoint that backs `window.yard.project`
