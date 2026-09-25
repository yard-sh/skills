# Custom Landing Pages

Every Yard project has a public landing page. Teams whose plan includes custom pages (check `yard me --json` → `.team_permissions`) can replace the default layout with their own HTML, CSS, JS, images and fonts, kept in the landing-page directory (default `.yard/landing-page/`, set by `landing_page.dir` and selected by `"type": "custom"` in `.yard/settings.json`) and shipped with `yard push`. This file covers what goes **inside** the bundle; the commands are in [cli-commands.md](cli-commands.md#project-files-settingsjson-push-pull-status-ls).

---

## Runtime data: `window.yard.project`

On a custom page the project's public data is a synchronous object: no fetch, no API key.

```js
window.yard.project; // the project, or null if it couldn't be loaded
```

It is the public project JSON (`GET /v1/projects/{username}/{slug}/public`), snake_case as-is. Most useful fields:

| Field | Type | Notes |
| --- | --- | --- |
| `slug`, `title` | `string` | |
| `tagline` | `string?` | Short marketing line |
| `description` / `description_html` | `string?` | Markdown source / rendered HTML |
| `price_cents` | `number` | Default tier price |
| `discounted_price_cents` | `number?` | After any launch-stage discount |
| `launch_stage` | `string` | `draft`, `pre-order`, `early_access` or `published` |
| `stage_discount_percent` | `number?` | Active launch-stage discount |
| `tiers` | `PricingTier[]` | Below |
| `images` | `ProjectImage[]` | Screenshots and icons, each with a `url` |
| `category` | `string?` | |
| `faq` | `{ question, answer }[]` | |
| `metadata` | `{ key, value }[]` | Seller-defined pairs |
| `license_key_enabled` | `boolean` | |
| `latest_release` | `object?` | Newest published release (tag, name, notes, date) |
| `release_count` | `number` | |
| `seller` | `{ username, avatar_url?, … }` | The owning **team** |

Each tier:

```jsonc
{
  "id": "uuid",                  // for data-tier-id / checkout({ tier })
  "name": "Pro",
  "description": "…",            // may be null
  "price_cents": 4900,
  "is_default": true,
  "seat_type": "single",         // "single" | "fixed_pack" | "per_seat"
  "seat_count": null,            // fixed_pack
  "min_seats": null,             // per_seat
  "max_seats": null,
  "pricing_model": "one_time",   // "one_time" | "subscription"
  "yearly_discount_percent": null,
  "features": ["…"],
  "free_trial_enabled": false,   // trials are per tier, never per project
  "free_trial_days": null,
  "trial_requires_card": true,
  "gift_enabled": false,
  "volume_brackets": []
}
```

> **Trials are per tier.** A project offers a trial when some tier has `free_trial_enabled: true` and `free_trial_days > 0`. Gate the trial button on that tier and pass its `id` (see the [worked example](#worked-example)). From the CLI: `yard projects show <slug> --json | jq .tiers`.

`window.yard.project` reflects the **saved** state of the release; save dashboard edits before refreshing a preview.

---

## Zero-JS HTML hooks

### `data-yard`: bind project fields to text

A dotted path into `project` sets the element's `textContent` on load; a missing or `null` field leaves it untouched. Add `data-yard-html` to write `innerHTML` instead.

```html
<h1 data-yard="title">Loading…</h1>
<strong data-yard="tiers.0.name"></strong>
<article data-yard="description_html" data-yard-html></article>
```

After inserting DOM yourself, call `window.yard.refresh()` to bind new nodes.

### `data-action`: checkout and trial buttons

```html
<button data-action="checkout">Buy now</button>                                   <!-- default tier -->
<button data-action="checkout" data-tier-id="…uuid…">Buy Pro</button>
<button data-action="checkout" data-tier-id="…" data-interval="yearly">Subscribe yearly</button>
<button data-action="checkout" data-tier-id="…" data-quantity="5" data-gift>Gift 5 seats</button>
<button data-action="trial">Start free trial</button>                             <!-- default or first trial tier -->
<button data-action="trial" data-tier-id="…uuid…">Start Pro trial</button>
```

| Attribute (`checkout`) | Meaning |
| --- | --- |
| `data-tier-id` (or `data-tier`) | Tier UUID; omit for the default tier |
| `data-interval` | `monthly` or `yearly` (subscription tiers) |
| `data-quantity` | Seats for `fixed_pack` / `per_seat` |
| `data-gift` | Present: start the gift flow |

`data-action="trial"` takes only `data-tier-id`: a trial-enabled tier, or omit it for the default (or first trial-enabled) tier. A trial goes to Yard's trial flow (`/trial/<username>/<slug>`): signed-in visitors start at once, signed-out ones confirm by email.

Clicks are handled by `embed.js` and their default is prevented, so there is no handler to write; the only work is visibility (hide the trial button when no tier has a trial, and set `data-tier-id` when showing it). **The attribute wins over `href`:** an element with `data-action` always starts checkout or a trial, so never repurpose one as a plain link; remove the attribute, or keep two elements and let `data-yard-when` pick one.

### Checkout URL parameters

Only needed when linking to `https://yard.sh/checkout/<username>/<slug>?…` by hand.

| Param | Meaning |
| --- | --- |
| `tier` | Tier UUID or name (case-insensitive); omit for the default |
| `quantity` | Seats for `fixed_pack` / `per_seat` |
| `interval` | `monthly` or `yearly` (`year` / `annual` accepted) |
| `gift` | Gift purchase on a `gift_enabled` one-time tier (`true`, `1` or bare) |
| `ref` | Affiliate code |
| `sandbox` | Sandbox name: simulated checkout, no card charged |

Invalid values are ignored, not rejected. `/trial/<username>/<slug>` takes `tier` and `sandbox` only.

---

## JavaScript API: `window.yard`

```js
window.yard = {
  project,            // public project or null (above)
  checkoutBase,       // e.g. "https://yard.sh"
  checkout(opts),     // redirect to checkout: { tier?, interval?, quantity?, gift? }
  trial(opts),        // redirect to the trial flow: { tier? }
  checkoutURL(opts),  // build the URL without redirecting
  trialURL(opts),
  ownership(),        // Promise<OwnershipState | null>, see Buyer state
  refresh(),          // re-run data-yard binding
};
```

`tier` is a tier UUID (`tierId` also works).

```js
for (const tier of window.yard.project.tiers) {
  const btn = document.createElement("button");
  btn.textContent = `${tier.name}: $${(tier.price_cents / 100).toFixed(2)}`;
  btn.addEventListener("click", () => window.yard.checkout({ tier: tier.id }));
  document.querySelector("#tiers").append(btn);
}
```

---

## Buyer state: `window.yard.ownership()`

`window.yard.project` is the same for everyone; `ownership()` is about this visitor's **yard.sh account**: are they signed in to Yard, and do they own the project? It returns a memoized Promise, which can be `null` (always null-check).

| Field | Type | Notes |
| --- | --- | --- |
| `signed_in` | `boolean` | Signed in to Yard at all |
| `user` | `{ id, username, avatar_url } \| null` | |
| `owned` | `boolean` | Owns the project (any tier; active trials and subscriptions count) |
| `is_trial`, `is_subscription` | `boolean` | Kind of entitlement |
| `transaction_id` | `string \| null` | Opaque entitlement reference |
| `tier_id`, `tier_name` | `string \| null` | Which tier they hold |

It never exposes email, other purchases or payment details. It is read-only UI gating; deeper integrations use the REST API ([api-reference.md](api-reference.md)).

**`data-yard-when`** covers the common case with no JS: `signed_in`, `signed_out`, `owned`, `not_owned`. Such elements stay hidden until the state resolves, so a non-owner never flashes an "Open in Library" link.

```html
<button data-yard-when="not_owned" data-action="checkout">Buy</button>
<a data-yard-when="owned" href="https://yard.sh/library">Open in your library</a>
```

```js
const state = await window.yard.ownership();
if (state?.is_subscription) {
  const { seller, slug } = window.yard.project;
  document.querySelector("#manage-sub").href = `https://yard.sh/library/${seller.username}/${slug}/subscription`;
}
```

On a custom domain, browsers with strict third-party cookie blocking (Safari and some privacy modes) may resolve `null` or `signed_in: false` for a signed-in visitor; default to the Buy button. Pages on `<username>.yard.sh` are not affected.

---

## Signed-in visitors on the landing page

When the project has services, the landing page can use the project's **Yard Auth** session (the one services see as `X-Yard-*` headers), which is separate from `ownership()`'s yard.sh account state. The `__yard/auth/*` endpoints exist at the project root, so relative URLs from the page reach them, locally and hosted (sandbox pages included):

```js
const me = await (await fetch("__yard/auth/me")).json();
// { authenticated: true, user_id, email, entitlement, tier? } or { authenticated: false, entitlement: "none" }
```

```html
<a href="__yard/auth/login?return=/">Sign in</a>      <!-- back to the landing page -->
<a href="__yard/auth/logout?return=/">Sign out</a>
```

`return` is relative to the project root here (`return=/` is this page, `return=/app/` a service). Called under a service, it is relative to that service instead. Send writes to a service with relative `fetch("app/items", …)` calls and `Content-Type: application/json`; see [service-and-database.md](service-and-database.md#identity-yard-auth-never-your-own).

---

## Asset paths

Pages serve under `<username>.yard.sh/<slug>/` (and `/<slug>/@<sandbox>/` in a sandbox), so **use relative URLs**: `href="styles.css"`, `src="app.js"`. A root-absolute `/styles.css` drops the slug and 404s. `yard init --page` scaffolds relative URLs.

---

## Testing a page before users see it

`yard dev` serves the page at `http://localhost:9875/<slug>/` with `embed.js` injected and tab reload on save, so `window.yard.project`, `data-yard` and Buy buttons behave as hosted (live project data when logged in, else a placeholder from settings.json). `ownership()` and `data-yard-when` follow the persona (`yard dev --as user:pro`, or the picker at `/<slug>/__yard/auth/login`). See [local-dev.md](local-dev.md).

Hosted, the project serves at `https://<username>.yard.sh/<slug>/` and each sandbox at `…/<slug>/@<sandbox>/`, with that sandbox's own pricing and copy in `window.yard.project`. Sandbox URLs are team-only (others get a 403, anonymous visitors sign in first) until `yard sandbox visibility public --sandbox <name>`.

A release without a custom page serves the default page (edited in the dashboard), so the URL always resolves. `"landing_page": { "type": "default" }` switches back to it without deleting your files; leaving the block out keeps whatever the release had.

```sh
yard push                                  # into your draft; nothing serves a draft
yard sandbox create preview                # once
yard sandbox pin                           # hold the storefront on what it serves today
yard releases publish v1.0.0
yard sandbox pin v1.0.0 --sandbox preview  # browse …/<slug>/@preview/
yard sandbox unpin                         # the storefront serves v1.0.0
```

---

## Bundle limits

20 files, 1 MB per file, 5 MB total; extensions `.html .css .js .json .svg .png .jpg .jpeg .webp .gif .woff2`; letters, digits and `._-`, at most one subdirectory, no dotfiles; `index.html` required to publish. The project and each sandbox are limited separately. `yard push` rejects violations before uploading.

---

## Worked example

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title data-yard="title">Loading…</title>
    <link rel="stylesheet" href="styles.css" />
  </head>
  <body>
    <h1 data-yard="title"></h1>
    <p data-yard="tagline"></p>
    <article data-yard="description_html" data-yard-html></article>

    <button data-yard-when="not_owned" data-action="checkout">Buy now</button>
    <button data-yard-when="not_owned" data-action="trial" id="trial-btn" hidden>Start free trial</button>
    <a data-yard-when="owned" href="https://yard.sh/library">Open in your library</a>

    <script>
      // Trials are per tier: show the button only when a tier offers one.
      const trialTier = window.yard.project?.tiers.find((t) => t.free_trial_enabled && (t.free_trial_days ?? 0) > 0);
      if (trialTier) {
        const btn = document.querySelector("#trial-btn");
        btn.dataset.tierId = trialTier.id;
        btn.hidden = false;
      }
    </script>
  </body>
</html>
```
