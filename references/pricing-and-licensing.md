# Pricing, Licensing, and Monetization

## Table of Contents

- [Pricing Tiers](#pricing-tiers)
- [Seat Types](#seat-types)
- [Volume Brackets](#volume-brackets)
- [Project Stages and Discounts](#project-stages-and-discounts)
- [Coupons](#coupons)
- [Free Trials](#free-trials)
- [Gift Purchases](#gift-purchases)
- [Commerce in a Sandbox](#commerce-in-a-sandbox)
- [License Keys](#license-keys)
- [Device Activations](#device-activations)
- [Checkout Calculation Flow](#checkout-calculation-flow)

---

## Pricing Tiers

Each project has one or more pricing tiers. How many a seller may create depends on their plan and is enforced server-side via the `max_pricing_tiers` permission (currently Basic: 2, Pro: 10). Nothing is gated client-side — an over-limit change is rejected with `upgrade_required`. Check the current cap with `yard me --json` → `.team_permissions.max_pricing_tiers`.

**Tier fields:**
- `name` — Display name for the tier
- `price_cents` — Base price in cents ($0 for free, or $3.00-$10,000.00)
- `description` — Optional description
- `sort_order` — Display ordering (0-based)
- `is_default` — Exactly one tier per project must be the default
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

## Project Stages and Discounts

Every project has a stage that controls availability and pricing. New projects always start in `draft` and progress through stages **forward-only**: once advanced, a project can never go back. Stage is not the same thing as visibility. Whether strangers may view the storefront at all is the project's **own visibility** (`yard sandbox visibility <public|private>`), and stage gates on top of it: a draft serves nothing publicly however visibility is set.

| Stage | Description | Discount field |
|---|---|---|
| `draft` | Initial stage. Not visible to buyers. Use this while configuring tiers, copy, and the landing page. | — |
| `early_access` | Public, purchasable, but the seller signals the project is still being polished. Buyers see an "Early Access" indicator. Optional launch discount via `early_access_discount_percent`. | `early_access_discount_percent` |
| `published` | General availability. Final stage. | — |
| `archived` | No longer available for new purchases. (Existing buyers retain access.) | — |

**Transition rules** (enforced server-side):

- Order: `draft` → `early_access` → `published`. Going backwards is rejected with `Cannot move project stage backward`.
- `published` is terminal — once there, the project cannot be moved again. The API returns `Project is already in the final 'published' stage and cannot be changed.`
- Skipping `early_access` is allowed — `draft` → `published` directly is valid.
- Transitioning out of `draft` requires an active Stripe Connect seller account. Backend rejects the change otherwise.

Stage discounts are applied before coupon discounts during checkout.

**How to advance a stage:** the CLI does not currently surface stage transitions — `yard projects edit` accepts settings (license keys, activations, trials) but not `stage`. Sellers advance project stage from the Yard dashboard at `https://dash.yard.sh/projects`. Direct REST `PUT /v1/projects/{id}` with `{"stage": "early_access"}` works server-side, but is not exposed as a public seller-API surface.

---

## Coupons

Coupons depend on the plan's `coupons` permission — check with `yard me --json` → `.team_permissions.coupons` rather than assuming a tier. There is no client-side gate: the server answers `403` if the plan doesn't include them.

**Manage them from the CLI** — `yard coupons` covers list / show / create / generate / update / rm / transactions / validate, all with `--json` and `--spec`. See [cli-commands.md](./cli-commands.md#yard-coupons) for the full surface.

**Coupon types:**
- `percentage` — 1-100% discount off the price
- `fixed_amount` — Fixed amount in cents subtracted from the price

**Coupon scopes:**
- `all_projects` — Applies to all of the seller's projects (including future ones)
- `specific_projects` — Applies only to selected projects (via `coupon_projects` junction table)

**Coupon fields:**
- `code` — The coupon code string (upper-cased, 4-50 alphanumeric characters)
- `discount_type` — `percentage` or `fixed_amount`
- `discount_value` — Percentage (1-100) or amount in **cents**
- `max_uses` — Usage limit across all buyers (null = unlimited). There is no per-buyer limit.
- `current_uses` — Current usage count
- `valid_from` — Optional start date; before it, the code is rejected as not yet valid
- `expires_at` — Optional expiration date
- `subscription_duration` — For subscription projects: `once` (first payment only, the default) or `forever` (every renewal). Ignored for one-time purchases.
- `is_active` — Seller's on/off toggle

A coupon is only usable when it is active, started, unexpired, and under its limit — `is_active: true` alone doesn't mean redeemable.

**Bulk generation:** Sellers can generate up to 100 unique codes at once with the same discount settings (`yard coupons generate --count N`). Generated codes are returned exactly once, at creation.

**Editing limits:** the discount can't be changed once a coupon has been redeemed, and a redeemed coupon can't be deleted — deactivate it instead. Sending `null` for `max_uses`, `expires_at`, or `valid_from` clears that field; omitting the key leaves it unchanged.

---

## Free Trials

Free trials are a **Pro-only** feature (check with `yard me --json` → `.team_permissions`) configured **per tier**, not on the project. A project "offers a trial" when at least one of its tiers has `free_trial_enabled: true`. Configure inside each tier object via `yard init --spec` (at creation) or with the dedicated subcommand `yard projects tiers edit <slug> <tier-name> --spec -` (later).

Tier-level trial fields:

- `free_trial_enabled` (boolean) — Toggle on a specific tier
- `free_trial_days` (1-365) — Duration in days; required when `free_trial_enabled` is true
- Creates a purchase with `is_trial = true` and `trial_expires_at` timestamp
- Buyers can activate trials as guests (no account required) for one-time tiers; subscription tiers may also require a card up-front via the per-tier `trial_requires_card` setting
- After trial expires, buyer must purchase the tier to continue access

There is no project-level trial setting anymore — putting `free_trial_enabled` (or `trial_requires_card`) outside a tier in a spec is rejected with `unknown field`. Card collection is also configured per tier:

- `trial_requires_card` (per-tier, defaults to true) — When true, trials on that subscription tier route through Stripe Checkout so the buyer enters a payment method up-front. When false, trials on the tier start without a card and convert silently when the trial ends. Has no effect on one-time tiers. Toggle via `yard projects tiers edit <slug> <tier> --spec -`.

**Adjusting a trial that's already running.** The tier setting only governs new trials. To change the time left for one buyer, use `yard transactions trial <order-id> --add-days N` (±365; `-3` shortens). Find the ids with `yard transactions list --trials`. Two things the command does that aren't obvious: the days are added to the trial's **current expiry rather than to today** — so extending an already-expired trial can still land in the past and restore nothing — and the **buyer is emailed** about the change. An expired trial whose new expiry lands in the future goes back to `active` and the buyer regains access (`"reactivated": true` in the response), unless they have since started another trial on that project. See [cli-commands.md](./cli-commands.md#yard-transactions).

---

## Gift Purchases

Gift purchasing is a **Pro-only** feature (check with `yard me --json` → `.team_permissions`).

- `gift_enabled` — Toggle on each pricing tier (one-time tiers only; checkout hides gifting for subscriptions)
- Buyer provides a recipient email at checkout
- Recipient receives activation instructions via email
- Tracked via `gift_activations` table
- Gift purchases create a license key that the recipient activates

---

## Commerce in a Sandbox

The project and each of its sandboxes carry their own commerce. The **project's own** is the real one: its checkouts go through Stripe, its money reaches the seller's payouts, and it alone appears in the seller's books. A **sandbox** carries a parallel set of customers, transactions, subscriptions, trials, license keys, coupon redemptions and gifts that the platform **simulates**: no Stripe object is created, no card is charged, and no money moves. The amounts are still computed and recorded exactly as a real sale would compute them, so a simulated purchase is a faithful rehearsal of the real one.

This is how a seller exercises the whole buying flow - checkout, entitlement, license validation, subscription renewal - without buying their own project and without a test card. Delete the sandbox and the whole rehearsal goes with it.

### What a simulated purchase does

A checkout that names a sandbox runs the same post-purchase machinery a paid one does:

- Writes a **completed transaction** carrying the tier, quantity, base price, volume and stage discounts, and coupon discount it would have charged. No `stripe_payment_intent_id` is set.
- **Mints license keys** on the project's normal rules (seat type and quantity), indistinguishable from real keys except for the sandbox they belong to.
- **Starts subscriptions** with no Stripe subscription behind them. They renew on a platform worker rather than on invoice webhooks, writing the `subscription_renewal` transactions Stripe would have written. The worker follows the app clock, so a dev-clock advance drives sandbox renewals.
- **Starts and converts free trials.** The one-pending-or-active-trial-per-buyer rule applies to the project and to each sandbox separately, so a buyer's trial on the storefront does not block starting one in a sandbox.
- **Records coupon redemptions and gifts.** A sandbox redemption records the usage row so it shows up on the transaction, but deliberately does **not** consume the coupon's real `current_uses` counter - simulating a redemption can never exhaust a live coupon.

Readiness gates that exist only to protect real money (Stripe Connect onboarding, payout setup) do not apply in a sandbox: a sandbox has to work before the seller has ever taken a payment.

### Simulated sales stay out of the books

Every seller-wide report reads the project itself only: the earnings summary, the transactions list, the customers list, payouts, subscribers and MRR. So `yard transactions list` and `yard customers list` never show simulated rows, and a sandbox can never inflate what a seller is owed.

The flip side is that **the CLI cannot read a sandbox's commerce at all** - neither command takes a `--sandbox` flag. A sandbox's transactions, subscriptions, customers and license keys live on that sandbox's pages in the dashboard. Do not tell a user to look for them in the CLI.

### Telling a sandbox key apart at runtime

`POST /v1/licenses/validate` answers `valid: true` for a key minted by a simulated purchase, exactly as it does for a real one. The response carries a **`sandbox`** field saying where the key's purchase lives:

| `sandbox` value | What the key is |
|---|---|
| absent / empty | a real purchase on the project itself |
| a sandbox name | a simulated purchase inside that sandbox |

Software that grants entitlement on a successful validation **must check this field**, or a simulated purchase entitles someone for real. The safe default in shipped software is to accept only an absent `sandbox` unless the build is a test build.

### Deleting a sandbox

Deleting a sandbox deletes its commerce with it, immediately and without a confirmation beyond the command's own prompt: its transactions, subscriptions, trials, license keys, device activations, coupon usages, gifts and affiliate commissions all go. Nothing cascades out of the sandbox - the project's own books are untouched.

---

## License Keys

License keys are a **Pro-only** feature (check with `yard me --json` → `.team_permissions`). Configure via `yard init --spec` (at creation) or `yard projects edit` (later) — both accept the `license_key_enabled` flag.

Yard automatically generates license keys for each purchase.

**Key characteristics:**
- Generated per transaction based on tier's seat type and quantity
- `single` tier: 1 key per purchase
- `fixed_pack` tier: `seat_count` keys per purchase
- `per_seat` tier: `quantity` keys per purchase (one per seat)

**Validation endpoint:** `POST /v1/licenses/validate`
- Input: license key + optional device ID
- Output: validation result with project/tier info
- **Requires an API key** with the `licenses:validate` scope (`Authorization: Bearer yard_<key>`). Embed it in the seller's software the same way you would for releases — see [api-reference.md](api-reference.md) for the endpoint definition and [releases-and-updates.md](releases-and-updates.md) for the embedded-API-key tradeoffs.
- The response carries a **`sandbox`** field: absent or empty for a real purchase on the project itself, a sandbox name for a key minted by a simulated purchase. Entitlement logic has to check it; see [Commerce in a Sandbox](#commerce-in-a-sandbox).

**License keys are configured on the project and on each sandbox separately.** `license_key_enabled`, `activations_enabled` and `max_activations` exist once for the project itself and again for each sandbox. A new sandbox starts with a copy of the project's three and diverges from the next edit.

That is what makes a sandbox the way to test. Enable license keys there, buy the project inside it - simulated commerce, no card, no money - and the purchase mints a real key belonging to that sandbox, exercising the same code path a buyer's key does rather than a parallel one. The project's real buyers are untouched by anything the sandbox does, and deleting the sandbox takes its keys and activations with it. `yard projects edit` still edits the **project's own** settings; a sandbox's are set from its License Keys page in the dashboard.

---

## Device Activations

Device activations are a **Pro-only** feature (check with `yard me --json` → `.team_permissions`) and require license keys to be enabled. Configure via `yard init --spec` (at creation) or `yard projects edit` (later) — both accept `activations_enabled` and `max_activations` (1-10000).

License keys can track device activations:

- Each activation records a `device_id` provided by the buyer's software
- Sellers can configure a maximum activation limit per license key
- Activations are tracked in the `license_activations` table
- Buyers can view and manage their activations from the buyer dashboard
- Activations belong to their key, and a key belongs to where its purchase lives, so a sandbox's device activations count against that sandbox's `max_activations` and never against the project's own

---

## Checkout Calculation Flow

The full price calculation during checkout (`CreatePaymentIntent`):

1. **Resolve tier** — Use the specified tier or fall back to the project's default tier
2. **Validate quantity** — Check quantity against the tier's seat type constraints
3. **Calculate base price** — `tier.GetPriceForQuantity(quantity)` (applies volume brackets if applicable)
4. **Apply stage discount** — If project is in the early access stage, apply `early_access_discount_percent`
5. **Apply coupon discount** — If a valid coupon code is provided, apply percentage or fixed-amount discount
6. **Calculate tax** — Via Stripe Tax API based on buyer's location
7. **Calculate seller earnings**
8. **Create Stripe PaymentIntent** — With metadata for tracking
9. **Create purchase record** — In the database with all pricing details
