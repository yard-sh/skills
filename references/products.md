# Products

Tiers decide who has the project ([pricing-and-licensing.md](pricing-and-licensing.md#pricing-tiers)). **Products** are what someone holding a tier buys on top of it: DLC or an expansion, a pack of gems, an add-on subscription for a SaaS. A product never grants access to the project and never changes the buyer's tier: `X-Yard-Entitlement`, `X-Yard-Tier`, `X-Yard-Tier-Key`, `owned` and `"access": "users"` all look at tiers only. Buyers need a Yard account (no guest checkout), because a product belongs to the account that holds the tier.

Plan gate: `yard me --json` → `.team_permissions.max_products` (100 per project on Basic and Pro). Over it the server answers "your plan supports up to N products"; without it, "products aren't included in your plan".

---

## Types and keys

| `type` | The buyer pays | They get |
| --- | --- | --- |
| `one_time` | Once | The product, for as long as they hold a tier it requires. Bought once. |
| `consumable` | Once per purchase, 1-99 at a time | Whatever your app delivers. Bought again and again; each purchase waits for your app to [fulfill it](#delivering-consumables). |
| `subscription` | `price_cents` monthly, or yearly with optional `yearly_discount_percent` | The product while the subscription runs, renewing on its own schedule, separate from any tier subscription. |

Every product has a **key** (`gems`, `cloud-sync`): checkout, holdings and webhooks name it, so a product can be renamed freely. Same grammar as tier keys: up to 64 lowercase letters, digits, `-` and `_`, starting with a letter or digit, unique within a release. Treat it as permanent: a different key is a different product, and the dashboard locks a published product's key. Once a release carrying a key is published, the key keeps that type for good: giving it another type, even in a later release, is refused ("has been published as a one-time product and cannot become a consumable product").

---

## Requirements

`requires` lists tier keys; holding any one is enough. Empty or omitted means any tier. A release with products needs at least one tier ("products are sold on top of a pricing tier, so add a tier first"), and every key in `requires` must be a tier of that release.

Holding a tier means the buyer bought it, claimed it free, subscribes to it, or was granted it by the team. A free trial does not count (a subscription still in its free trial included), and neither does a past-due subscription.

When a buyer stops holding every required tier (downgrade, refund, canceled tier subscription, failed tier payment):

| Type | What happens |
| --- | --- |
| `one_time` | Still owned, reported `active: false` and `requirement_met: false` until they hold a qualifying tier again. |
| `subscription` | Set to end at the close of its period (`cancel_at_period_end: true`, `cancel_reason: "requirement_lost"`) and the buyer is emailed. It keeps running if they hold a qualifying tier again before then, or once a failed tier payment goes through. |
| `consumable` | Nothing: the requirement applies only when buying. |

A purchase is judged by the requirements the product had when it was bought, so editing `requires` later never changes what existing buyers hold. While a product requires a tier, that tier can't be removed from the release: `yard projects tiers rm`, a push and the dashboard all refuse with an error naming both. Drop the requirement first.

---

## Setting up products

Products belong to a release, like pricing. Saved into a published release they reach new buyers at once; each change becomes a new product revision, so a past purchase always resolves to what it bought. Removing a product stops new sales; buyers keep it. Three ways to edit:

- **Dashboard:** the release's **Products** page, next to **Pricing** (it crops and resizes icon uploads).
- **settings.json** `products` block, applied by `yard push` and [GitHub sync](releases-and-updates.md#syncing-releases-from-github).
- **CLI:** `yard projects products` for one product at a time.

### The `products` block

```json
"products": [
  { "key": "gems", "name": "500 Gems", "type": "consumable", "price_cents": 499, "requires": ["player"], "icon": ".yard/products/gems.png" },
  { "key": "cloud-sync", "name": "Cloud Sync", "description": "Keep your saves on every device.",
    "type": "subscription", "price_cents": 300, "yearly_discount_percent": 20, "requires": ["pro"] }
]
```

| Field | Notes |
| --- | --- |
| `key` | Required |
| `name` | 1-100 characters, unique within the release (case-insensitive) |
| `description` | Optional, up to 2,000 characters |
| `type` | `one_time`, `consumable` or `subscription` |
| `price_cents` | 99 to 1,000,000; a subscription's monthly price |
| `yearly_discount_percent` | Subscriptions only (dropped for other types): 1-100% off twelve monthly payments |
| `requires` | Tier keys; omit or `[]` for any tier |
| `icon` | Optional path, relative to the directory holding `.yard` and inside it; `.png`, `.jpg`, `.jpeg` or `.webp` |

- No block: products stay managed from the dashboard. `"products": []` removes them all. Present: `yard push` and GitHub sync make the release's products **match exactly**, array order is display order, and the requirements are checked against the tiers the same push leaves.
- An icon must be exactly 256x256 PNG, JPEG or WebP, at most 1 MiB (only the dashboard resizes). A push uploads an icon when its bytes changed, and a product declared without `icon` loses its icon. A declared icon file missing locally fails the push before anything uploads; `yard status` lists it under `product_icons.missing`.
- Dashboard product edits regenerate the release's settings.json; an icon keeps the path the file gave it, else `.yard/products/<key>.<ext>`, which is where `yard pull` and `yard init --project` write it.

### `yard projects products`

```sh
yard projects products my-game --json                         # list (open draft, else newest published)
echo '{"key":"gems","name":"500 Gems","type":"consumable","price_cents":499,"requires":["player"]}' \
  | yard projects products add my-game --spec - --icon art/gems.png
echo '{"price_cents": 599}' | yard projects products edit my-game gems --spec -
yard projects products edit my-game gems --remove-icon
yard projects products rm my-game gems --yes
yard releases publish v1.3.0                                   # the draft goes live
```

- `add`, `edit` and `rm` edit your open draft (or a new draft seeded from the newest published release), the release `yard push` writes to; `--release <tag|id>` edits a published release, live at once wherever it is served. Every subcommand takes `--release` and `--json` (the release's product list after the save).
- `add <slug> --spec <file|-> [--icon <path>]`: one settings.json entry (unknown fields rejected; `icon` in the spec works like `--icon`). `key` may be omitted (derived from the name), but write it.
- `edit <slug> <product-key> [--spec <file|->] [--icon <path> | --remove-icon]`: a partial spec; present fields replace, absent ones stay.
- `rm <slug> <product-key> [--yes]`: buyers keep what they bought. `--yes` is required without a TTY.

Over HTTP: `PUT /v1/projects/{id}/products?release=<release id>` replaces the whole set (`projects:write`; send back an existing `icon` object to keep it), and `PUT` / `DELETE /v1/projects/{id}/products/{key}/icon?release=` upload (multipart field `image`) or remove one icon (`releases:write`). Icon changes record no product revision.

---

## Selling

- **The project page** has a **Products** tab, where signed-in holders of a qualifying tier buy each product.
- **A custom landing page** reads `window.yard.project.products` and starts checkout with `data-action="checkout" data-product="<key>"` (plus `data-quantity` for a consumable, `data-interval` for a subscription) or `window.yard.checkout({ product, quantity, interval })`. `data-return-to` or `returnTo` brings the buyer back to the page afterwards. See [landing-pages.md](landing-pages.md#data-action-checkout-and-trial-buttons).
- **Anything else** (an app, a service's own page, an email) links to checkout.

Each entry of `window.yard.project.products` (the public project JSON; empty when nothing is on sale): `key`, `name`, `description?`, `type`, `price_cents`, `yearly_discount_percent?`, `yearly_price_cents?` (subscriptions), `requires`, `icon_url?`.

### Checkout links

`https://pay.yard.sh/<team>/<slug>?product=<key>`, with:

| Param | Meaning |
| --- | --- |
| `product` | The product's key (instead of `tier`) |
| `quantity` | Consumables: 1-99. Everything else buys 1 |
| `interval` | Subscription products: `monthly` or `yearly` (`month`, `year`, `annual`, `annually` accepted) |
| `return_to` | Absolute URL to send the buyer back to once the purchase settles or they cancel |
| `for` | The Yard user id the purchase is for (`X-Yard-User-Id`, or `user_id` from Yard Auth); anyone else is asked to switch accounts instead of paying |
| `sandbox` | Sandbox name: simulated checkout |

`return_to` must be on the project's own sites, or checkout shows **Invalid Checkout Link**: `https://<team>.yard.sh/<slug>/…`, an active custom domain over `https`, or the origin of a Yard Auth redirect URI (such as `http://localhost:3000`). On the way back checkout adds `yard_product` (the key), `yard_purchase` (the `transaction_id`; omitted on cancel) and `yard_status` (`succeeded`, `processing` (still confirming; check again before delivering) or `canceled`). They are a hint to refresh, never proof of payment: read the holdings or wait for `product.purchased` before delivering.

On a custom landing page, `embed.js` sets both: `data-return-to` (bare: this page) or `window.yard.checkout({ product, returnTo, for })`, with `returnTo` resolved against the page. The page the buyer comes back to finds the outcome in `window.yard.purchase` (`{ product, status, purchase_id }`, taken off the URL), and after `succeeded`, `ownership()` waits up to 10 s for the holding ([landing-pages.md](landing-pages.md#javascript-api-windowyard)):

```js
window.yard.checkout({ product: "gems", quantity: 2, returnTo: location.href });
```

Under `yard dev` the page is on `http://localhost:<port>`, which is refused as a `return_to` unless it is a Yard Auth redirect URI origin.

Checkout refuses a buyer who doesn't hold a required tier, a `one_time` product they already own, a `subscription` product they already subscribe to, and any product while the project is in pre-order. Products are always sold at full price plus tax: coupons, gifts, free trials, affiliate credit, seats and launch-stage discounts apply to tiers only.

---

## What someone holds

Every way of reading it returns the same shape:

```json
{
  "tier_keys": ["pro"],
  "products": [
    { "key": "map-pack", "name": "Map Pack", "type": "one_time", "requires": ["pro"], "transaction_id": "0b7c1f2e-…",
      "active": true, "status": "owned", "acquired_at": "2026-03-02T10:15:00Z", "requirement_met": true },
    { "key": "cloud-sync", "name": "Cloud Sync", "type": "subscription", "requires": [], "subscription_id": "6d5c4b3a-…",
      "active": true, "status": "active", "current_period_end": "2026-04-02T10:15:00Z", "acquired_at": "2026-03-02T10:15:00Z", "requirement_met": true },
    { "key": "gems", "name": "500 Gems", "type": "consumable", "requires": ["pro"], "transaction_id": "4a3b2c1d-…",
      "active": true, "status": "unfulfilled", "quantity": 2, "acquired_at": "2026-03-05T09:30:00Z", "requirement_met": true }
  ]
}
```

- `tier_keys`: the tiers that let them buy products (trials and past-due subscriptions don't count). Compare with each product's `requires` to show a Buy button or what they need first.
- Use a product while `active` is `true`. `status` is `owned` (one-time), `unfulfilled` (consumable) or the subscription's status. A subscription is listed while it runs, with `cancel_at_period_end` and `cancel_reason` once set to end. Each unfulfilled consumable purchase is its own entry, listed only until it is fulfilled.

| Where the code runs | How it reads them |
| --- | --- |
| A hosted service's pages or the landing page (browser) | `GET __yard/products` (relative URL, like `__yard/auth/me`). Always 200: `{"authenticated": true, "tier_keys": [...], "products": [...]}`, or `{"authenticated": false, "tier_keys": [], "products": []}` |
| A custom landing page's script | `window.yard.ownership()` → `.tier_keys`, `.products` (empty when signed out) |
| An app outside the project, with a Yard Auth access token | `GET https://api.yard.sh/v1/yard-auth/products` → `{tier_keys, products}`, for the project itself only ([api-reference.md](api-reference.md#yard-auth-for-external-apps)) |
| A server: the team's, or a service's `_service.js` | `GET https://api.yard.sh/v1/projects/{id}/users/{user}/products[?sandbox=<name>]` with an API key holding `products:read`. `{user}` is `user_` plus the first 8 characters of the user id, or their email; adds a `user` object |

The `__yard/*` endpoints are answered by the edge for the visitor's browser session, before your code runs; the edge strips that session before a request reaches `_service.js`, so server code reads holdings through the API. On the team API, `404` means the user never bought from the project, and `409` means two buyers share those 8 characters (pass the email).

---

## Delivering consumables

Yard can't hand over gems inside your game, so each consumable purchase waits, unfulfilled, until your app delivers it and says so:

1. **Find what's waiting:** holdings with `status: "unfulfilled"`, the `product.purchased` webhook, or `GET /v1/projects/{id}/purchases/unfulfilled[?sandbox=<name>]` (`products:read`; every paid unfulfilled purchase, oldest first, up to 1000, each with `transaction_id`, `product_key`, `product_name`, `quantity`, `purchased_at`, `user`).
2. **Grant** `quantity` times the product, keyed on `transaction_id` so a retry never grants twice.
3. **Fulfill** it by `transaction_id`:
   - from a project page, as the signed-in visitor: `POST __yard/purchases/{transaction_id}/fulfill`
   - with a Yard Auth access token: `POST https://api.yard.sh/v1/yard-auth/purchases/{transaction_id}/fulfill`
   - from a server: `POST https://api.yard.sh/v1/projects/{id}/purchases/{transaction_id}/fulfill[?sandbox=<name>]` with `products:fulfill`

Fulfill only after a grant your own trusted code made: a page calling `__yard/purchases/…` suits a game whose state lives in the player's client; when a service's database holds the balance, verify and fulfill from `_service.js` ([worked example](#worked-example-gem-packs-in-a-hosted-game)).

`200` answers the purchase; fulfilling it again returns it unchanged (original `fulfilled_at`), so retrying is safe:

```json
{ "transaction_id": "4a3b2c1d-…", "product_key": "gems", "product_name": "500 Gems", "quantity": 2,
  "purchased_at": "2026-03-05T09:30:00Z", "fulfilled": true, "fulfilled_at": "2026-03-05T09:30:04Z" }
```

Errors are `{"error": "…", "error_code": "…"}`:

| Status | Meaning |
| --- | --- |
| `401` | Not signed in (page and token routes), or a missing or invalid API key |
| `404` | No such purchase (page and token routes: none of the signed-in user's) |
| `400` `not_consumable` | Only consumables are fulfilled |
| `409` `purchase_not_paid` | The payment hasn't completed yet; try again later |
| `409` `purchase_refunded` | Refunded first: don't deliver it (take back a grant you already made) |

A purchase still unfulfilled **3 days** after it was bought is refunded automatically, sandbox purchases included: the buyer and the team are emailed, it leaves the lists, and `product.refunded` fires with `refund_reason: "unfulfilled"`. When a purchase you already fulfilled is refunded, `product.refunded` fires without that reason, and taking back what you delivered is up to you.

---

## Subscription products

- Renew on their own schedule, separate from the tier subscription, and appear in the buyer's Library alongside their tier. Buyers cancel and resume them on the project's **Products** tab in their Library.
- Each paid renewal fires `product.renewed`; `product.canceled` fires when one ends.
- A price change on the release the project serves works like a tier's: current subscribers keep their price for 30 days, are emailed the date, then move to the new price.
- Losing the required tier sets one to end at period end ([Requirements](#requirements)).

---

## Webhooks

Webhooks are a Pro feature: one per project (dashboard **Configure** > **Webhooks**), receiving every event, each delivery signed in `X-Yard-Signature`. Product events never fire the tier events (`sale.completed`, `sale.refunded`, `subscription.canceled`); those carry the tier's `tier_key`.

| Event | Fires |
| --- | --- |
| `product.purchased` | A one-time product, every consumable purchase, a subscription product's first payment |
| `product.renewed` | Each subscription product renewal whose payment goes through |
| `product.refunded` | Once per transaction, at its first refund (full or partial). `refund_amount_cents` is the total refunded so far, tax included; `refund_reason` is `unfulfilled` for the 3-day automatic refund |
| `product.canceled` | A subscription product ends: its period ran out after a cancel or a lost requirement, it was canceled immediately, or payment retries ran out. Not when it is merely set to end, or resumed |

Fields: `event_id` (the same on every retry; deduplicate on it), `event`, `transaction_id` (not on `canceled`), `project_id`, `project_slug`, `sandbox` (simulated purchases only), `user_email`, `user_username`, `user_display_id` (the account UUID, the same value as `X-Yard-User-Id`), `product_id` (the product as sold, which changes between releases; tell products apart by `product_key`), `product_key`, `product_name`, `product_type`, `subscription_id` (subscriptions), `timestamp`. `purchased` and `renewed` add `quantity`, `amount_cents` (tax included) and `currency` (`usd`), plus `is_subscription: true` for a subscription; `refunded` adds `refund_amount_cents`, `refund_reason` and `currency`.

---

## Sandboxes and local development

- **Sandbox:** product checkout is simulated like the rest of a sandbox's commerce ([pricing-and-licensing.md](pricing-and-licensing.md#commerce-in-a-sandbox)). `__yard/products` and `__yard/purchases/…/fulfill` on a sandbox's pages read and fulfill that sandbox's purchases; the API-key routes take `?sandbox=<name>` (a service reads the name from `X-Yard-Sandbox`); the Yard Auth token routes see the project itself only; webhooks carry `sandbox`.
- **`yard dev`:** one `buyer:<product key>` persona per product some tier qualifies for, holding that product and the first tier that lets it be bought (`active`, with that tier's `X-Yard-Tier` and `X-Yard-Tier-Key`); `user:<tier key>` personas hold their tier and no products. A consumable's buyer has one unfulfilled purchase (quantity 1); fulfilling it locally answers like hosted and keeps it fulfilled until `yard dev` restarts. No webhooks fire and nothing is refunded after 3 days. Offline or logged out, products come from the settings.json block, without icons. See [local-dev.md](local-dev.md#personas-instead-of-sign-in).

---

## Limits

| Rule | Value |
| --- | --- |
| Products per project | 100 (`max_products`) |
| Price | $0.99 to $10,000.00 (`price_cents` 99 to 1,000,000) |
| Consumable quantity per purchase | 1 to 99 |
| Name | 1 to 100 characters |
| Description | Up to 2,000 characters |
| Yearly discount | 1 to 100% |
| Icon | 256x256 PNG, JPEG or WebP, at most 1 MiB |
| Unfulfilled consumable | Refunded after 3 days |

---

## Worked example: gem packs in a hosted game

A game sold as one `player` tier, with a `game` service (`"access": "users"`, `"database_access": true`) that keeps each player's gems in the database.

```json
"pricing": { "tiers": [{ "key": "player", "name": "Player", "price_cents": 900, "is_default": true }] },
"products": [
  { "key": "gems", "name": "500 Gems", "type": "consumable", "price_cents": 499, "requires": ["player"], "icon": ".yard/products/gems.png" }
]
```

```sql
-- .yard/migrations/0002_gem_grants.sql
CREATE TABLE IF NOT EXISTS gem_grants (transaction_id TEXT PRIMARY KEY, user_id TEXT NOT NULL, gems INTEGER NOT NULL, granted_at INTEGER NOT NULL);
```

```sh
yard keys create --spec - --json <<<'{"name":"my-game-gems","scopes":["products:read","products:fulfill"]}' | jq -r .key
yard service secrets set YARD_API_KEY=yard_…    # the key printed above
```

In `_service.js`, `POST api/redeem` credits every gem pack the player bought that nobody delivered yet:

```js
const PROJECT_ID = "…"; // yard projects show my-game --json | jq -r .id
const GEMS_PER_PACK = 500;

function yardAPI(env, method, path, sandbox) {
  const url = new URL(`https://api.yard.sh/v1/projects/${PROJECT_ID}${path}`);
  if (sandbox) url.searchParams.set("sandbox", sandbox);
  return fetch(url, { method, headers: { Authorization: `Bearer ${env.YARD_API_KEY}` } });
}

async function redeem(request, env) {
  const userId = request.headers.get("X-Yard-User-Id");
  const sandbox = request.headers.get("X-Yard-Sandbox") ?? "";
  const res = await yardAPI(env, "GET", `/users/user_${userId.slice(0, 8)}/products`, sandbox);
  if (res.status === 404) return Response.json({ credited: 0 });
  if (!res.ok) return Response.json({ error: "try again" }, { status: 502 });
  let credited = 0;
  for (const p of (await res.json()).products) {
    if (p.key !== "gems" || p.status !== "unfulfilled") continue;
    const gems = p.quantity * GEMS_PER_PACK;
    const { meta } = await env.DB.prepare(
      "INSERT INTO gem_grants (transaction_id, user_id, gems, granted_at) VALUES (?1, ?2, ?3, ?4) ON CONFLICT DO NOTHING",
    ).bind(p.transaction_id, userId, gems, Date.now()).run();
    const done = await yardAPI(env, "POST", `/purchases/${p.transaction_id}/fulfill`, sandbox);
    if (done.status === 409 && (await done.json()).error_code === "purchase_refunded") {
      await env.DB.prepare("DELETE FROM gem_grants WHERE transaction_id = ?1").bind(p.transaction_id).run();
      continue;
    }
    if (meta.changes) credited += gems;
  }
  return Response.json({ credited });
}
```

The balance is `SELECT SUM(gems) FROM gem_grants WHERE user_id = ?1` minus what was spent. A failed fulfill leaves the purchase unfulfilled, so the next call retries it without crediting twice. The game's page links to checkout with `return_to` pointing back at itself and `for` set to `user_id` from `__yard/auth/me` (in a sandbox, add `sandbox=<name>`), and calls `api/redeem` on every load, which also picks up a purchase that came back `processing`:

```js
const me = await (await fetch("__yard/auth/me")).json();
const buy = new URL("https://pay.yard.sh/acme/my-game");
buy.search = new URLSearchParams({ product: "gems", quantity: "1", return_to: location.origin + location.pathname, for: me.user_id });
document.querySelector("#buy-gems").href = buy.href;
const { credited } = await (await fetch("api/redeem", { method: "POST", headers: { "Content-Type": "application/json" } })).json();
```

`yard dev` simulates the `__yard/*` endpoints but not the team API, so test this flow in a sandbox, where checkout is simulated and the API calls carry `?sandbox=`.
