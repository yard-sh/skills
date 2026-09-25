# Pricing, Licensing, and Monetization

Every feature here is plan-gated on the **team**: read `yard me --json` → `.team_permissions` rather than assuming. Nothing is gated client-side; the server answers `upgrade_required` (or a `403`) when the plan lacks something.

## Pricing Tiers

A project has one or more tiers; how many is `max_pricing_tiers` (currently Basic 2, Pro 10). Fields:

- `name`; `description` (optional); `features` (up to 10 strings); `sort_order` (display order, 0-based)
- `price_cents`: `0` (free) or 300 to 1,000,000 ($3.00 to $10,000.00)
- `is_default`: exactly one tier per project
- `seat_type`: `single`, `fixed_pack` or `per_seat` (seat-based needs `seat_based_pricing`)
- `pricing_model`: `one_time` or `subscription`; `yearly_discount_percent` (1-100) for subscriptions
- Per tier: `free_trial_enabled`, `free_trial_days`, `trial_requires_card`, `gift_enabled` (below)

There is no first-class "enterprise" or contact-sales tier: model it as a high-priced `per_seat` tier or a separate top tier, and handle custom contracts outside Yard.

## Seat Types

- **`single`**: one license, quantity 1, one license key per purchase.
- **`fixed_pack`**: a fixed bundle ("Team 5-Pack") bought as one unit; `seat_count` (2-1000) keys per purchase.
- **`per_seat`**: the buyer picks a quantity within `min_seats` / `max_seats` (null = unlimited); one key per seat; optional volume brackets.

## Volume Brackets

`per_seat` only: percentage discounts at quantity thresholds. Brackets are contiguous (no gaps or overlaps), ordered by `min_quantity`; the last may have `max_quantity: null`; `discount_percent` is 1-99. Price per seat in a bracket is `base_price * (1 - discount_percent/100)`.

| Min | Max | Discount | At $10/seat |
| --- | --- | --- | --- |
| 1 | 10 | 0% | $10.00 |
| 11 | 50 | 15% | $8.50 |
| 51 | unlimited | 25% | $7.50 |

## Launch Stages and Discounts

New projects start in `draft` and move **forward only**:

| Stage | Meaning |
| --- | --- |
| `draft` | Not visible to buyers; the owning team can still use every page and service. |
| `early_access` | Public and purchasable, marked "Early Access". Optional launch discount `early_access_discount_percent` (1-100), which ends at `published`. |
| `published` | General availability. Final. |
| `archived` | No new purchases; existing buyers keep access. |

- `draft` → `early_access` → `published`, or straight from `draft` to `published`. Going back is rejected ("Cannot move launch stage backward"); `published` can never change.
- Leaving `draft` requires the team to have finished payout setup.
- Stages and the early-access discount are set in the dashboard (https://dash.yard.sh/projects); the CLI does not change them.
- The launch stage is not visibility: whether strangers may view at all is `yard sandbox visibility`, and a draft serves nothing publicly either way.

## Coupons

Needs `.team_permissions.coupons`. Managed with `yard coupons` ([cli-commands.md](cli-commands.md#yard-coupons)).

- `discount_type`: `percentage` (1-100) or `fixed_amount` (`discount_value` in **cents**).
- `scope`: `all_projects` (every project, including future ones) or `specific_projects` (`project_ids`).
- `code`: upper-cased, 4-50 alphanumeric. `max_uses` counts across all buyers (null = unlimited; no per-buyer limit). `valid_from` / `expires_at` are optional. `subscription_duration`: `once` (first payment, default) or `forever` (every renewal); ignored for one-time purchases.
- A coupon is usable only when active, started, unexpired and under its limit; `is_active` is just the on/off switch.
- Up to 100 codes can be generated at once, returned only at creation. After the first redemption the discount cannot change and the coupon cannot be deleted (deactivate it). `null` clears `max_uses`, `expires_at` or `valid_from`; an omitted key is unchanged.

## Free Trials

Plan-gated and configured **per tier** (set at creation in `yard init --spec`, later with `yard projects tiers edit <slug> <tier> --spec -`). A project offers a trial when any tier has `free_trial_enabled: true`; a project-level trial field is rejected with `unknown field`.

- `free_trial_days`: 1-365, required with a trial.
- `trial_requires_card` (default true): a subscription tier's trial collects a card at checkout and converts when it ends; `false` starts without a card. No effect on one-time tiers.
- One-time tiers can be trialed as a guest (email confirmation). After expiry the buyer must purchase to keep access.
- To change a running trial for one buyer: `yard transactions trial <order-id> --add-days N` (added to the current expiry, not today; the buyer is emailed). See [cli-commands.md](cli-commands.md#yard-transactions).

## Gift Purchases

Plan-gated; `gift_enabled` per tier, one-time tiers only. The buyer enters a recipient email at checkout (from the Gift button or `?gift=true`); the recipient gets activation instructions. The license key is minted on activation. A gift unactivated after 90 days expires and is refunded automatically.

## Commerce in a Sandbox

The project's own commerce is real: checkouts charge cards, money reaches the seller's payouts, and only it appears in the seller's books. Each **sandbox** has a parallel, **simulated** set of users, transactions, subscriptions, trials, license keys, coupon redemptions and gifts: no card is charged and no money moves, but amounts are computed exactly as a real sale would. That is how a seller rehearses checkout, entitlement, license validation and renewals without buying their own project.

A simulated purchase:

- records a completed transaction with the tier, quantity, discounts and amounts it would have charged;
- mints license keys by the usual rules, identical to real ones except for the sandbox they belong to;
- starts subscriptions that renew on schedule and trials that convert (the one-trial-per-buyer rule applies per sandbox);
- records coupon redemptions and gifts, without using up the real coupon's `current_uses`.

Payout setup is not required in a sandbox.

**Out of the books.** Earnings, payouts, subscribers, `yard transactions` and `yard users` read the project itself only, and take no `--sandbox` flag. A sandbox's commerce is on that sandbox's pages in the dashboard; don't send users to the CLI for it.

**Telling a sandbox key apart.** `POST /v1/licenses/validate` answers `valid: true` for a sandbox key too, with a **`sandbox`** field: absent or empty for a real purchase on the project, a sandbox name for a simulated one. Software that grants entitlement **must check it**; the safe default in shipped builds is to accept only an absent `sandbox`.

**Deleting a sandbox** deletes all of its commerce immediately (transactions, subscriptions, trials, license keys, device activations, coupon usages, gifts, affiliate commissions). The project's own books are untouched.

## License Keys

Plan-gated. Enable with `license_key_enabled` in `yard init --spec` or `yard projects edit`. A purchase mints one key for `single`, `seat_count` keys for `fixed_pack`, and one per seat for `per_seat`.

Validate with `POST /v1/licenses/validate` (license key and optional `device_id` in the body), which needs an API key with `licenses:validate` in the `Authorization` header ([api-reference.md](api-reference.md#licenses)). Check the response's `sandbox` field (above).

License-key settings (`license_key_enabled`, `activations_enabled`, `max_activations`) exist on the project and again on each sandbox; a new sandbox copies the project's and diverges on the next edit. `yard projects edit` changes the project's own; a sandbox's are set from its License Keys page in the dashboard.

**Testing validation end to end:**

```sh
yard sandbox create staging                       # inherits the project's license-key settings
yard keys create --spec - --json <<<'{"name":"local-validate","scopes":["licenses:validate"]}'   # capture .key
# Buy the project inside the sandbox from its checkout page (simulated, no card), then:
curl -X POST https://api.yard.sh/v1/licenses/validate \
  -H "Authorization: Bearer $YARD_API_KEY" -H 'Content-Type: application/json' \
  -d '{"license_key":"XXXX-XXXX-XXXX-XXXX","device_id":"laptop-42"}'   # response has "sandbox": "staging"
yard sandbox delete staging --yes                 # a clean slate: keys and activations go with it
```

## Device Activations

Plan-gated and requires license keys. `activations_enabled` and `max_activations` (1-10000 per key), set like license keys. Each activation records the `device_id` the buyer's software sends; buyers manage their devices from their Yard library. Activations belong to their key, so a sandbox's count against that sandbox's limit only.

## How checkout computes the price

1. The tier (given, or the default) and a quantity valid for its seat type.
2. Base price, with any volume bracket.
3. Launch-stage discount (early access).
4. Coupon discount.
5. Tax, by the buyer's location.
