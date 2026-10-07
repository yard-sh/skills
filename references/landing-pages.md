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
| `products` | `Product[]` | Products on sale on top of a tier: `key`, `name`, `description?`, `type` (`one_time`, `consumable`, `subscription`), `price_cents`, `yearly_discount_percent?`, `yearly_price_cents?` (subscriptions), `requires` (tier keys), `icon_url?`. Empty when none. See [products.md](products.md) |
| `images` | `ProjectImage[]` | Screenshots and icons, each with a `url` |
| `category` | `string?` | |
| `faq` | `{ question, answer }[]` | |
| `metadata` | `{ key, value }[]` | Developer-defined pairs |
| `license_key_enabled` | `boolean` | |
| `latest_release` | `object?` | Newest published release (tag, name, notes, date) |
| `release_count` | `number` | |
| `team` | `{ username, avatar_url?, … }` | The owning **team** |

Each tier:

```jsonc
{
  "id": "uuid",                  // changes with every pricing revision
  "key": "pro",                  // stable identity: match on it, and pass it as data-tier-id / checkout({ tier })
  "name": "Pro",                 // display only
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
  "free_trial": {                // trials are per tier, never per project
    "enabled": false,            // when true, "days" (7..365) is present too
    "requires_card": false
  },
  "gift_enabled": false,
  "volume_brackets": []
}
```

> **Trials are per tier.** A project offers a trial when some tier has `free_trial.enabled: true`. Gate the trial button on that tier and pass its `key` (see the [worked example](#worked-example)). From the CLI: `yard projects show <slug> --json | jq .tiers`.

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
<button data-action="checkout" data-tier-id="pro">Buy Pro</button>
<button data-action="checkout" data-tier-id="pro" data-interval="yearly">Subscribe yearly</button>
<button data-action="checkout" data-tier-id="team" data-quantity="5" data-gift>Gift 5 seats</button>
<button data-action="trial">Start free trial</button>                             <!-- default or first trial tier -->
<button data-action="trial" data-tier-id="pro">Start Pro trial</button>
<button data-action="checkout" data-product="gems" data-quantity="3">Buy 3 gem packs</button>
```

| Attribute (`checkout`) | Meaning |
| --- | --- |
| `data-tier-id` | Tier key (or id, which changes with every pricing revision); omit for the default tier |
| `data-interval` | `monthly` or `yearly` (subscription tiers and subscription products) |
| `data-quantity` | Seats for `fixed_pack` / `per_seat`; 1-99 of a consumable product |
| `data-gift` | Present: start the gift flow |
| `data-product` | A product key: buy that product instead of a tier (`data-tier-id` and `data-gift` are ignored). The buyer must be signed in and hold a required tier ([products.md](products.md#selling)) |

`data-action="trial"` takes only `data-tier-id`: a trial-enabled tier, or omit it for the default (or first trial-enabled) tier. A trial goes to Yard's trial flow (`https://yard.sh/trial/<username>/<slug>`): signed-in visitors start at once, signed-out ones confirm by email.

Clicks are handled by `embed.js` and their default is prevented, so there is no handler to write; the only work is visibility (hide the trial button when no tier has a trial, and set `data-tier-id` when showing it). **The attribute wins over `href`:** an element with `data-action` always starts checkout or a trial, so never repurpose one as a plain link; remove the attribute, or keep two elements and let `data-yard-when` pick one.

### Checkout URL parameters

Only needed when linking to `https://pay.yard.sh/<username>/<slug>?…` by hand.

| Param | Meaning |
| --- | --- |
| `tier` | Tier id or key (never a name); omit for the default |
| `product` | A product key, instead of `tier` |
| `quantity` | Seats for `fixed_pack` / `per_seat`; 1-99 of a consumable product |
| `interval` | `monthly` or `yearly` (`year` / `annual` accepted), for subscription tiers and products |
| `gift` | Gift purchase on a `gift_enabled` one-time tier (`true`, `1` or bare) |
| `ref` | Affiliate code |
| `sandbox` | Sandbox name: simulated checkout, no card charged |
| `return_to`, `for` | With `product` only: where to send the buyer afterwards, and the Yard user id the purchase is for ([products.md](products.md#checkout-links)) |

Invalid values are ignored, not rejected. `/trial/<username>/<slug>` takes `tier` and `sandbox` only.

---

## JavaScript API: `window.yard`

```js
window.yard = {
  project,            // public project or null (above)
  checkoutBase,       // e.g. "https://pay.yard.sh"
  checkout(opts),     // redirect to checkout: { tier?, interval?, quantity?, gift? } or { product, quantity?, interval? }
  trial(opts),        // redirect to the trial flow: { tier? }
  checkoutURL(opts),  // build the URL without redirecting
  trialURL(opts),
  ownership(),        // Promise<OwnershipState | null>, see User state
  refresh(),          // re-run data-yard binding
};
```

`tier` is a tier key or id; `product` is a product key. Neither `checkout()` nor `data-action` sends `return_to` or `for`: to bring a product buyer back, add them to `checkoutURL({ product })` yourself ([products.md](products.md#checkout-links)).

```js
for (const tier of window.yard.project.tiers) {
  const btn = document.createElement("button");
  btn.textContent = `${tier.name}: $${(tier.price_cents / 100).toFixed(2)}`;
  btn.addEventListener("click", () => window.yard.checkout({ tier: tier.key }));
  document.querySelector("#tiers").append(btn);
}
```

---

## User state: `window.yard.ownership()`

`window.yard.project` is the same for everyone; `ownership()` is about this visitor's **yard.sh account**: are they signed in to Yard, and do they own the project? It returns a memoized Promise, which can be `null` (always null-check).

| Field | Type | Notes |
| --- | --- | --- |
| `signed_in` | `boolean` | Signed in to Yard at all |
| `user` | `{ id, username, avatar_url } \| null` | |
| `owned` | `boolean` | Owns the project (any tier; active trials and subscriptions count) |
| `is_trial`, `is_subscription` | `boolean` | Kind of entitlement; a subscription still in its free trial is `is_trial` |
| `transaction_id` | `string \| null` | Opaque entitlement reference |
| `tier_key`, `tier_name`, `tier_id` | `string \| null` | The tier they hold. Match `tier_key` against `project.tiers[i].key`: a holder from an earlier pricing revision has a `tier_id` the page no longer lists |
| `tier_keys` | `string[]` | Tiers that let them buy products (trials don't count); compare with each product's `requires`. Empty when signed out |
| `products` | `object[]` | Products they hold on this project ([shape](products.md#what-someone-holds)); use one while its `active` is `true`. Empty when signed out |

It never exposes email, purchases of other projects or payment details. Products never change `owned`, which is about tiers. It is read-only UI gating; deeper integrations use the REST API ([api-reference.md](api-reference.md)).

**`data-yard-when`** covers the common case with no JS: `signed_in`, `signed_out`, `owned`, `not_owned`. Such elements stay hidden until the state resolves, by a style rule page CSS can't override (elements added later follow it too), so a non-owner never flashes an "Open in Library" link. `embed.js` never touches the `hidden` attribute, which stays yours.

```html
<button data-yard-when="not_owned" data-action="checkout">Buy</button>
<a data-yard-when="owned" href="https://yard.sh/library">Open in your library</a>
```

```js
const state = await window.yard.ownership();
if (state?.is_subscription) {
  const { team, slug } = window.yard.project;
  document.querySelector("#manage-sub").href = `https://yard.sh/library/${team.username}/${slug}/subscription`;
}
```

On a custom domain, browsers with strict third-party cookie blocking (Safari and some privacy modes) resolve `signed_in: false` for a signed-in visitor; default to the Buy button. Pages on `<username>.yard.sh` are not affected. `null` means Yard couldn't answer (no reply within 8 s, or a draft or private project viewed by its team); treat it as signed out.

---

## Signed-in visitors on the landing page

When the project has services, the landing page can use the project's **Yard Auth** session (the one services see as `X-Yard-*` headers), which is separate from `ownership()`'s yard.sh account state. The `__yard/auth/*` endpoints exist at the project root, so relative URLs reach them from `index.html` and other root pages (from a subfolder page use `../__yard/auth/…`), locally and hosted (sandbox pages included):

```js
const me = await (await fetch("__yard/auth/me")).json();
// { authenticated: true, user_id, email, entitlement, tier?, tier_key? } or { authenticated: false, entitlement: "none" }
```

```html
<a href="__yard/auth/login?return=/">Sign in</a>      <!-- back to the landing page -->
<a href="__yard/auth/logout?return=/">Sign out</a>
```

`__yard/products` sits next to them and answers `{ authenticated, tier_keys, products }` for the same session; `POST __yard/purchases/{transaction_id}/fulfill` marks a consumable delivered ([products.md](products.md#delivering-consumables)).

`return` is relative to the project root here (`return=/` is this page, `return=/app/` a service). Called under a service, it is relative to that service instead. Send writes to a service with relative `fetch("app/items", …)` calls and `Content-Type: application/json`; see [service-and-database.md](service-and-database.md#identity-yard-auth-never-your-own).

---

## Asset paths

Pages serve under `<username>.yard.sh/<slug>/` (and `/<slug>/@<sandbox>/` in a sandbox), so **use relative URLs**: `href="styles.css"`, `src="app.js"`. A root-absolute `/styles.css` drops the slug and 404s. `yard init --page` scaffolds relative URLs.

---

## Testing a page before users see it

`yard dev` serves the page at `http://localhost:9875/<slug>/` with `embed.js` injected and tab reload on save, so `window.yard.project`, `data-yard` and Buy buttons behave as hosted (live project data when logged in, else a placeholder from settings.json). `ownership()`, `data-yard-when` and `__yard/products` follow the persona (`yard dev --as user:<tier key>` or `--as buyer:<product key>`, or the picker at `/<slug>/__yard/auth/login`). See [local-dev.md](local-dev.md).

Hosted, the project serves at `https://<username>.yard.sh/<slug>/` and each sandbox at `…/<slug>/@<sandbox>/`, with that sandbox's own pricing and copy in `window.yard.project`. Sandbox URLs are team-only (others get a 403, anonymous visitors sign in first) until `yard sandbox visibility public --sandbox <name>`, which opens one only while the project itself is public and past draft.

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

File count, per-file size and total size come from the team's plan (Basic and Pro: 60 files, 25 MiB per file, 100 MiB total; read `team_permissions` `page_max_*` from `yard me --json` or GET /v1/me for the exact values). Extensions `.html .css .js .json .svg .png .jpg .jpeg .webp .gif .woff2 .mp4 .webm .vtt`; letters, digits and `._-`, at most one subdirectory, no dotfiles; `index.html` required to publish. Video: H.264/AAC `.mp4`, `muted playsinline` to autoplay, `preload="metadata"` with a `poster`, captions as `.vtt`. Each release's bundle is limited separately. `yard push` rejects violations before uploading.

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
      const trialTier = window.yard.project?.tiers.find((t) => t.free_trial?.enabled);
      if (trialTier) {
        const btn = document.querySelector("#trial-btn");
        btn.dataset.tierId = trialTier.key;
        btn.hidden = false;
      }
    </script>
  </body>
</html>
```
