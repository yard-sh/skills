# Pricing, Licensing, and Monetization

## Table of Contents

- [Pricing Tiers](#pricing-tiers)
- [Seat Types](#seat-types)
- [Volume Brackets](#volume-brackets)
- [Product Stages and Discounts](#product-stages-and-discounts)
- [Coupons](#coupons)
- [Free Trials](#free-trials)
- [Gift Purchases](#gift-purchases)
- [License Keys](#license-keys)
- [Device Activations](#device-activations)
- [Checkout Calculation Flow](#checkout-calculation-flow)

---

## Pricing Tiers

Each product has one or more pricing tiers. How many a seller may create depends on their plan and is enforced server-side via the `max_pricing_tiers` permission (currently Basic: 2, Pro: 10). Nothing is gated client-side — an over-limit change is rejected with `upgrade_required`. Check the current cap with `yard me --json` → `.permissions.max_pricing_tiers`.

**Tier fields:**
- `name` — Display name for the tier
- `price_cents` — Base price in cents ($0 for free, or $3.00-$10,000.00)
- `description` — Optional description
- `sort_order` — Display ordering (0-based)
- `is_default` — Exactly one tier per product must be the default
- `seat_type` — `single`, `fixed_pack`, or `per_seat`
- `features` — List of feature strings (max 10 per tier)
- `pricing_model` — `one_time` or `subscription`
- `yearly_discount_percent` — Optional discount for yearly subscription billing

**Price constraints:**
- Minimum: $3.00 (300 cents) for paid tiers
- Maximum: $10,000.00 (1,000,000 cents)
- $0 (free) tiers are allowed
- Exactly one tier must have `is_default = true`

---

## Seat Types

### single
Individual license. Quantity is always 1. One license key per purchase.

### fixed_pack
Fixed seat count (e.g., "Team 5-Pack"). Purchased as a single unit — the price is for the whole pack. Generates `seat_count` license keys per purchase.

**Fields:**
- `seat_count` — Number of seats in the pack (required)

### per_seat
Buyer selects a quantity via a quantity selector. Supports min/max seat limits and volume bracket discounts.

**Fields:**
- `min_seats` — Minimum quantity (optional)
- `max_seats` — Maximum quantity (optional, nil = unlimited)
- `volume_brackets` — Optional volume discount brackets

**Price calculation:**
- No matching bracket: `base_price * quantity`
- With bracket: `(base_price - base_price * discount_percent / 100) * quantity`

---

## Volume Brackets

Volume brackets apply only to `per_seat` tiers. They define percentage discounts at quantity thresholds.

**Rules:**
- Brackets must be contiguous (no gaps or overlaps between min/max ranges)
- The last bracket can have unlimited max (`max_quantity = null`)
- `discount_percent` range: 1-99
- Brackets are ordered by `min_quantity`

**Example:**
| Min | Max | Discount | Effective price at $10/seat |
|-----|-----|----------|-----------------------------|
| 1 | 10 | 0% | $10.00/seat |
| 11 | 50 | 15% | $8.50/seat |
| 51 | unlimited | 25% | $7.50/seat |

---

## Product Stages and Discounts

Every product has a stage that controls visibility and pricing. New products always start in `draft` and progress through stages **forward-only** — once advanced, a product can never go back.

| Stage | Description | Discount field |
|---|---|---|
| `draft` | Initial stage. Not visible to buyers. Use this while configuring tiers, copy, and the landing page. | — |
| `early_access` | Public, purchasable, but the seller signals the product is still being polished. Buyers see an "Early Access" indicator. Optional launch discount via `early_access_discount_percent`. | `early_access_discount_percent` |
| `released` | General availability. Final stage. | — |
| `archived` | No longer available for new purchases. (Existing buyers retain access.) | — |

**Transition rules** (enforced server-side):

- Order: `draft` → `early_access` → `released`. Going backwards is rejected with `Cannot move product stage backward`.
- `released` is terminal — once there, the product cannot be moved again. The API returns `Product is already in the final 'released' stage and cannot be changed.`
- Skipping `early_access` is allowed — `draft` → `released` directly is valid.
- Transitioning out of `draft` requires an active Stripe Connect seller account. Backend rejects the change otherwise.

Stage discounts are applied before coupon discounts during checkout.

**How to advance a stage:** the CLI does not currently surface stage transitions — `yard products edit` accepts settings (license keys, activations, trials) but not `stage`. Sellers advance product stage from the Yard dashboard at `https://yard.sh/dashboard/products`. Direct REST `PUT /v1/products/{id}` with `{"stage": "early_access"}` works server-side, but is not exposed as a public seller-API surface.

---

## Coupons

Coupons depend on the plan's `coupons` permission — check with `yard me --json` → `.permissions.coupons` rather than assuming a tier. There is no client-side gate: the server answers `403` if the plan doesn't include them.

**Manage them from the CLI** — `yard coupons` covers list / show / create / generate / update / rm / transactions / validate, all with `--json` and `--spec`. See [cli-commands.md](./cli-commands.md#yard-coupons) for the full surface.

**Coupon types:**
- `percentage` — 1-100% discount off the price
- `fixed_amount` — Fixed amount in cents subtracted from the price

**Coupon scopes:**
- `all_products` — Applies to all of the seller's products (including future ones)
- `specific_products` — Applies only to selected products (via `coupon_products` junction table)

**Coupon fields:**
- `code` — The coupon code string (upper-cased, 4-50 alphanumeric characters)
- `discount_type` — `percentage` or `fixed_amount`
- `discount_value` — Percentage (1-100) or amount in **cents**
- `max_uses` — Usage limit across all buyers (null = unlimited). There is no per-buyer limit.
- `current_uses` — Current usage count
- `valid_from` — Optional start date; before it, the code is rejected as not yet valid
- `expires_at` — Optional expiration date
- `subscription_duration` — For subscription products: `once` (first payment only, the default) or `forever` (every renewal). Ignored for one-time purchases.
- `is_active` — Seller's on/off toggle

A coupon is only usable when it is active, started, unexpired, and under its limit — `is_active: true` alone doesn't mean redeemable.

**Bulk generation:** Sellers can generate up to 100 unique codes at once with the same discount settings (`yard coupons generate --count N`). Generated codes are returned exactly once, at creation.

**Editing limits:** the discount can't be changed once a coupon has been redeemed, and a redeemed coupon can't be deleted — deactivate it instead. Sending `null` for `max_uses`, `expires_at`, or `valid_from` clears that field; omitting the key leaves it unchanged.

---

## Free Trials

Free trials are a **Pro-only** feature (check with `yard me --json` → `.permissions`) configured **per tier**, not on the product. A product "offers a trial" when at least one of its tiers has `free_trial_enabled: true`. Configure inside each tier object via `yard init --spec` (at creation) or with the dedicated subcommand `yard products tiers edit <slug> <tier-name> --spec -` (later).

Tier-level trial fields:

- `free_trial_enabled` (boolean) — Toggle on a specific tier
- `free_trial_days` (1-365) — Duration in days; required when `free_trial_enabled` is true
- Creates a purchase with `is_trial = true` and `trial_expires_at` timestamp
- Buyers can activate trials as guests (no account required) for one-time tiers; subscription tiers may also require a card up-front via the per-tier `trial_requires_card` setting
- After trial expires, buyer must purchase the tier to continue access

There is no product-level trial setting anymore — putting `free_trial_enabled` (or `trial_requires_card`) outside a tier in a spec is rejected with `unknown field`. Card collection is also configured per tier:

- `trial_requires_card` (per-tier, defaults to true) — When true, trials on that subscription tier route through Stripe Checkout so the buyer enters a payment method up-front. When false, trials on the tier start without a card and convert silently when the trial ends. Has no effect on one-time tiers. Toggle via `yard products tiers edit <slug> <tier> --spec -`.

**Adjusting a trial that's already running.** The tier setting only governs new trials. To change the time left for one buyer, use `yard transactions trial <order-id> --add-days N` (±365; `-3` shortens). Find the ids with `yard transactions list --trials`. Two things the command does that aren't obvious: the days are added to the trial's **current expiry rather than to today** — so extending an already-expired trial can still land in the past and restore nothing — and the **buyer is emailed** about the change. An expired trial whose new expiry lands in the future goes back to `active` and the buyer regains access (`"reactivated": true` in the response), unless they have since started another trial on that product. See [cli-commands.md](./cli-commands.md#yard-transactions).

---

## Gift Purchases

Gift purchasing is a **Pro-only** feature (check with `yard me --json` → `.permissions`).

- `gift_enabled` — Toggle on each pricing tier (one-time tiers only; checkout hides gifting for subscriptions)
- Buyer provides a recipient email at checkout
- Recipient receives activation instructions via email
- Tracked via `gift_activations` table
- Gift purchases create a license key that the recipient activates

---

## License Keys

License keys are a **Pro-only** feature (check with `yard me --json` → `.permissions`). Configure via `yard init --spec` (at creation) or `yard products edit` (later) — both accept the `license_key_enabled` flag.

Yard automatically generates license keys for each purchase.

**Key characteristics:**
- Generated per transaction based on tier's seat type and quantity
- `single` tier: 1 key per purchase
- `fixed_pack` tier: `seat_count` keys per purchase
- `per_seat` tier: `quantity` keys per purchase (one per seat)

**Validation endpoint:** `POST /v1/licenses/validate`
- Input: license key + optional device ID
- Output: validation result with product/tier info
- **Requires an API key** with the `licenses:validate` scope (`Authorization: Bearer yard_<key>`). Embed it in the seller's software the same way you would for releases — see [api-reference.md](api-reference.md) for the endpoint definition and [releases-and-updates.md](releases-and-updates.md) for the embedded-API-key tradeoffs.

**Test license key:** Every product with `license_key_enabled: true` has a sandbox key the seller can use to exercise validation/activation logic without buying their own product. The test key behaves identically to a real one against `POST /v1/licenses/validate`, but its activations live in a separate `test_activations` table and never collide with real buyers. Retrieve it with `yard licenses test-key`; manage its activations with `yard licenses test-activations list` and `yard licenses test-activations clear`. See [cli-commands.md](cli-commands.md#yard-licenses) for full flag reference.

---

## Device Activations

Device activations are a **Pro-only** feature (check with `yard me --json` → `.permissions`) and require license keys to be enabled. Configure via `yard init --spec` (at creation) or `yard products edit` (later) — both accept `activations_enabled` and `max_activations` (1-10000).

License keys can track device activations:

- Each activation records a `device_id` provided by the buyer's software
- Sellers can configure a maximum activation limit per license key
- Activations are tracked in the `license_activations` table
- Buyers can view and manage their activations from the buyer dashboard
- Test activations (created via the product's test license key) are isolated in a parallel `test_activations` table, count against `max_activations` independently, and can be wiped with `yard licenses test-activations clear`

---

## Checkout Calculation Flow

The full price calculation during checkout (`CreatePaymentIntent`):

1. **Resolve tier** — Use the specified tier or fall back to the product's default tier
2. **Validate quantity** — Check quantity against the tier's seat type constraints
3. **Calculate base price** — `tier.GetPriceForQuantity(quantity)` (applies volume brackets if applicable)
4. **Apply stage discount** — If product is in the early access stage, apply `early_access_discount_percent`
5. **Apply coupon discount** — If a valid coupon code is provided, apply percentage or fixed-amount discount
6. **Calculate tax** — Via Stripe Tax API based on buyer's location
7. **Calculate seller earnings**
8. **Create Stripe PaymentIntent** — With metadata for tracking
9. **Create purchase record** — In the database with all pricing details
