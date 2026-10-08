# Pricing, Licensing, and Monetization

Every feature here is plan-gated on the **team**: read `yard me --json` → `.team_permissions` rather than assuming. Nothing is gated client-side; the server answers `upgrade_required` (or a `403`) when the plan lacks something.

## Pricing Tiers

A project has one or more tiers; how many is `max_pricing_tiers` (currently 10 on both Basic and Pro). Fields:

- `key`: the tier's stable identity ([Tier keys](#tier-keys)); `name`: display text, unique within a release; `description` (optional); `features` (up to 20 strings); `sort_order` (display order, 0-based)
- `price_cents`: `0` (free) or 300 to 1,000,000 ($3.00 to $10,000.00)
- `is_default`: exactly one tier per project
- `seat_type`: `single`, `fixed_pack` or `per_seat` (seat-based needs `seat_based_pricing`)
- `pricing_model`: `one_time` or `subscription`; `yearly_discount_percent` (1-100) for subscriptions
- Per tier: `free_trial` (`enabled`, `days`, `requires_card`), `gift_enabled` (below)

There is no first-class "enterprise" or contact-sales tier: model it as a high-priced `per_seat` tier or a separate top tier, and handle custom contracts outside Yard.

### Tier keys

A tier's `key` (`pro`, `team-5-pack`) is its identity across releases and pricing revisions; the name is only what people see. Up to 64 lowercase letters, digits, `-` and `_`, starting with a letter or digit, unique within a release.

- Renaming a tier keeps its key, and with it its holders and subscribers. A different key is a different tier: holders of the old key keep it under that key, and the new tier starts with none.
- `key` is optional on input (settings.json `pricing.tiers[]`, `--spec`, the API): a tier without one takes the key that name was last published with, unless another tier in the same save uses that key; otherwise one derived from its name (lowercased, anything but letters and digits turned into `-`, `-2` and so on added so it never reuses a key another tier has had). So a tier renamed in settings.json without a `key` becomes a new tier: always write `key`. The dashboard and `yard projects tiers edit` refuse to change a published tier's key (`GET /v1/projects/{id}/pricing/keys` lists them).
- A price change moves the subscribers of the same key to the new price (after the 30-day notice); `subscription.tier_changed` fires only when a subscriber's key changes.
- Tier ids change with every pricing revision, so code matches on keys: `X-Yard-Tier-Key` in a service ([service-and-database.md](service-and-database.md#identity-yard-auth-never-your-own)), `tier_key` from `__yard/auth/me`, Yard Auth userinfo and `window.yard.ownership()`, `tier_key` on the `sale.completed`, `sale.refunded`, `subscription.renewed`, `subscription.ending`, `subscription.resumed`, `subscription.past_due`, `subscription.canceled`, `trial.started`, `trial.ended` and `gift.activated` webhooks, and `old_tier_key` / `new_tier_key` on `subscription.tier_changed`. Checkout's `?tier=` takes an id or a key, never a name.
- Products name the tiers they require by key, and a tier a product requires can't be removed.

## Seat Types

- **`single`**: one license, quantity 1, one license key per purchase.
- **`fixed_pack`**: a fixed bundle ("Team 5-Pack") bought as one unit; `seat_count` (2-1000) keys per purchase.
- **`per_seat`**: the user picks a quantity at checkout, within `min_seats` / `max_seats` (null = unlimited); one key per seat; optional volume brackets.

## Volume Brackets

`per_seat` only: percentage discounts at quantity thresholds. Brackets are contiguous (no gaps or overlaps), ordered by `min_quantity`; the last may have `max_quantity: null`; `discount_percent` is 1-99. Price per seat in a bracket is `base_price * (1 - discount_percent/100)`.

| Min | Max | Discount | At $10/seat |
| --- | --- | --- | --- |
| 1 | 10 | 0% | $10.00 |
| 11 | 50 | 15% | $8.50 |
| 51 | unlimited | 25% | $7.50 |

## Products

Products are sold on top of a tier, never instead of one: one-time add-ons and DLC, consumables (gem packs, credits) and add-on subscriptions. A release needs at least one tier before it has products, the buyer must be signed in and hold a tier the product `requires`, and buying one never grants access or changes the buyer's tier. Gated by `max_products`. Everything about them: [products.md](products.md).

## Launch Stages and Discounts

New projects start in `draft` and move **forward only**:

| Stage | Meaning |
| --- | --- |
| `draft` | Not visible to users; the owning team can still use every page and service. |
| `early_access` | Public and purchasable, marked "Early Access". Optional launch discount `early_access_discount_percent` (1-100), which ends at `published`. |
| `published` | General availability. Final. |
| `archived` | No new purchases; existing users keep access. |

- `draft` → `early_access` → `published`, or straight from `draft` to `published`. Going back is rejected ("Cannot move launch stage backward"); `published` can never change.
- Leaving `draft` requires the team to have finished payout setup.
- Stages and the early-access discount are set in the dashboard (https://dash.yard.sh/projects); the CLI does not change them.
- The launch stage is not visibility: whether strangers may view at all is `yard sandbox visibility`, and a draft serves nothing publicly either way.

## Coupons

Needs `.team_permissions.coupons`. Managed with `yard coupons` ([cli-commands.md](cli-commands.md#yard-coupons)).

- `discount_type`: `percentage` (1-100) or `fixed_amount` (`discount_value` in **cents**).
- `scope`: `all_projects` (every project, including future ones) or `specific_projects` (`project_ids`).
- `code`: upper-cased with spaces removed, then 4-50 letters, digits, `-` or `_`. `max_uses` counts across all users (null = unlimited; no per-user limit). `valid_from` / `expires_at` are optional. `subscription_duration`: `once` (first payment, default) or `forever` (every renewal); ignored for one-time purchases.
- A coupon is usable only when active, started, unexpired and under its limit; `is_active` is just the on/off switch.
- Coupons discount tiers only: products always sell at full price.
- Up to 100 codes can be generated at once, returned only at creation. After the first redemption the discount cannot change and the coupon cannot be deleted (deactivate it). `null` clears `max_uses`, `expires_at` or `valid_from`; an omitted key is unchanged.

## Free Trials

Plan-gated and configured **per tier** (set at creation in `yard init --spec`, later in a release's pricing with `yard projects tiers edit <slug> <tier> --spec -` or the settings.json `pricing` block). A project offers a trial when any tier has `free_trial.enabled: true`; an omitted `free_trial` means no trial, and a project-level trial field is rejected with `unknown field`.

- `free_trial.days`: 7-365, optional; a trial enabled without days runs 7.
- `free_trial.requires_card` (default false): a subscription tier's trial collects a card at checkout and converts when it ends; `false` starts without a card. No effect on one-time tiers.
- One-time tiers can be trialed as a guest (email confirmation). After expiry the trial user must purchase to keep access.
- Trials are for tiers only. A trial (or a subscription still in its trial) doesn't count as holding the tier, so it never lets anyone buy a product.
- To change one user's running trial: `yard transactions trial <order-id> --add-days N` (added to the current expiry, not today; a card-required trial's first charge moves with it; the user on the trial is emailed). See [cli-commands.md](cli-commands.md#yard-transactions).

## Gift Purchases

Plan-gated; `gift_enabled` per tier, one-time tiers only (products can't be gifted). The user enters a recipient email at checkout (from the Gift button or `?gift=true`); the recipient gets activation instructions. The license key is minted on activation. A gift unactivated after 90 days expires and is refunded automatically. Refunding a gift takes it back: an unactivated link stops working, and an activated gift leaves the recipient's library and its license keys stop validating.

## Commerce in a Sandbox

The project's own commerce is real: checkouts charge cards, money reaches the team's payouts, and only it appears in the team's books. Each **sandbox** has a parallel, **simulated** set of users, transactions, subscriptions, trials, license keys, coupon redemptions and gifts: no card is charged and no money moves, but amounts are computed exactly as a real sale would. That is how a developer rehearses checkout, entitlement, license validation and renewals without buying their own project.

A simulated purchase:

- records a completed transaction with the tier, quantity, discounts and amounts it would have charged;
- mints license keys by the usual rules, identical to real ones except for the sandbox they belong to;
- starts subscriptions that renew on schedule and trials that convert (the one-trial-per-user rule applies per sandbox);
- records coupon redemptions and gifts, without using up the real coupon's `current_uses`;
- buys products too: a consumable bought in a sandbox is listed and fulfilled there (`__yard/products` and `__yard/fulfillments/…` on the sandbox's pages, or the API with `?sandbox=<name>`), and refunded after 3 days unfulfilled like a real one ([products.md](products.md#delivering-consumables)).

Payout setup is not required in a sandbox.

**Out of the books.** Earnings, payouts, subscribers, `yard transactions` and `yard users` read the project itself only, and take no `--sandbox` flag. A sandbox's commerce is on that sandbox's pages in the dashboard; don't send users to the CLI for it.

**Sandbox keys are opt-in.** `POST /v1/licenses/validate` and `/v1/licenses/deactivate` find a sandbox key only when the request's **`sandbox`** field names that sandbox; without it they find only the project's own keys, and a sandbox key answers exactly like an unknown key (`License key not found`). Shipped builds never send `sandbox`, so a simulated purchase can't entitle anyone; a test build sends the sandbox's name. The response's `sandbox` echoes the sandbox the key was found in.

**Deleting a sandbox** deletes all of its commerce immediately (transactions, subscriptions, trials, license keys, device activations, coupon usages, gifts, affiliate commissions). The project's own books are untouched.

## License Keys

Plan-gated. Enable with `license_key_enabled` in `yard init --spec` or `yard projects edit`. A purchase mints one key for `single`, `seat_count` keys for `fixed_pack`, and one per seat for `per_seat`.

Validate with `POST /v1/licenses/validate` (`project_id`, the license key and optional `sandbox` and `device_id` in the body), which needs an API key with `licenses:validate` in the `Authorization` header, plus `licenses:activate` when it sends `device_id` ([api-reference.md](api-reference.md#licenses)). Letter case and surrounding spaces in the key are ignored. Only a `200` answers for the key: `401`/`402`/`403` mean the team's API key or plan, `429` and `5xx` are transient, so shipped software keeps its last answer rather than locking the user out. Validation is per project: an API key covers the whole team, so `project_id` (from `yard projects show <slug> --json | jq -r .id`) is what keeps a key bought for one project from validating in another; such a key answers `valid: false` with `License key is not for this project`. A sandbox key validates only with `sandbox` set (above).

License-key settings (`license_key_enabled`, `activations_enabled`, `max_activations`) exist on the project and again on each sandbox; a new sandbox copies the project's and diverges on the next edit. `yard projects edit` changes the project's own; a sandbox's are set from its License Keys page in the dashboard.

**Testing validation end to end:**

```sh
yard sandbox create staging                       # inherits the project's license-key settings
yard keys create --spec - --json <<<'{"name":"local-validate","scopes":["licenses:validate","licenses:activate"]}'   # capture .key; device_id needs activate
PROJECT_ID=$(yard projects show <slug> --json | jq -r .id)
# Buy the project inside the sandbox from its checkout page (simulated, no card), then:
curl -X POST https://api.yard.sh/v1/licenses/validate \
  -H "Authorization: Bearer $YARD_API_KEY" -H 'Content-Type: application/json' \
  -d '{"project_id":"'"$PROJECT_ID"'","sandbox":"staging","license_key":"XXXX-XXXX-XXXX-XXXX","device_id":"laptop-42"}'   # without "sandbox" the key is not found
yard sandbox delete staging --yes                 # a clean slate: keys and activations go with it
```

## Device Activations

Plan-gated and requires license keys. `activations_enabled` and `max_activations` (1-10000 per key), set like license keys. Each activation records the `device_id` the software sends; users manage their devices from their Yard library. Only a validate that sends `device_id` enforces the limit: a new device is refused once every slot is used, an activated one keeps validating, and a validate without `device_id` checks the key alone, so send it on every check. Activations belong to their key, so a sandbox's count against that sandbox's limit only.

## How checkout computes the price

For a tier:

1. The tier (given by id or key, or the default) and a quantity valid for its seat type.
2. Base price, with any volume bracket.
3. Launch-stage discount (early access).
4. Coupon discount.
5. Tax, by the user's location.

A product is its price (times `quantity` for a consumable; the yearly price, after `yearly_discount_percent`, for a yearly subscription) plus tax, with no launch-stage or coupon discount.
