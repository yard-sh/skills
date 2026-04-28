# Pricing, Licensing, and Monetization

## Table of Contents

- [Pricing Tiers](#pricing-tiers)
- [Seat Types](#seat-types)
- [Volume Brackets](#volume-brackets)
- [Product States and Discounts](#product-states-and-discounts)
- [Coupons](#coupons)
- [Free Trials](#free-trials)
- [Gift Purchases](#gift-purchases)
- [License Keys](#license-keys)
- [Device Activations](#device-activations)
- [Platform Fee](#platform-fee)
- [Checkout Calculation Flow](#checkout-calculation-flow)

---

## Pricing Tiers

Each product has 1 to 5 pricing tiers. Free-tier sellers can create 1 tier per product; Pro sellers can create up to 5.

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

## Product States and Discounts

Products progress through states, each with optional discounts:

| State | Description | Discount field |
|---|---|---|
| `draft` | Not visible to buyers | — |
| `preorder` | Accepting orders before release | `preorder_discount_percent` |
| `early_access` | Limited release | `early_access_discount_percent` |
| `released` | General availability | — |
| `archived` | No longer available | — |

State discounts are applied before coupon discounts during checkout.

---

## Coupons

Coupons are a **Pro-only** feature.

**Coupon types:**
- `percentage` — 1-100% discount off the price
- `fixed_amount` — Fixed amount in cents subtracted from the price

**Coupon scopes:**
- `all_products` — Applies to all of the seller's products (including future ones)
- `specific_products` — Applies only to selected products (via `coupon_products` junction table)

**Coupon fields:**
- `code` — The coupon code string
- `discount_type` — `percentage` or `fixed_amount`
- `discount_value` — Percentage (1-100) or amount in cents
- `max_uses` — Usage limit (nil = unlimited)
- `current_uses` — Current usage count
- `expires_at` — Optional expiration date
- `is_active` — Can be deactivated by seller

**Bulk generation:** Sellers can generate multiple unique coupon codes at once with the same discount settings.

---

## Free Trials

Free trials are a **Pro-only** feature. Configure via `yard init --spec` (at creation) or `yard products edit` (later).

Products can offer free trials:

- `free_trial_enabled` — Toggle on the product
- `free_trial_days` — Duration (1-365 days; defaults to 7 if enabled without an explicit value)
- Creates a purchase with `is_trial = true` and `trial_expires_at` timestamp
- Buyers can activate trials as guests (no account required)
- After trial expires, buyer must purchase to continue access

---

## Gift Purchases

Gift purchasing is a **Pro-only** feature.

- `gift_enabled` — Toggle on the product
- Buyer provides a recipient email at checkout
- Recipient receives activation instructions via email
- Tracked via `gift_activations` table
- Gift purchases create a license key that the recipient activates

---

## License Keys

License keys are a **Pro-only** feature. Configure via `yard init --spec` (at creation) or `yard products edit` (later) — both accept the `license_key_enabled` flag.

Yard automatically generates license keys for each purchase.

**Key characteristics:**
- Generated per transaction based on tier's seat type and quantity
- `single` tier: 1 key per purchase
- `fixed_pack` tier: `seat_count` keys per purchase
- `per_seat` tier: `quantity` keys per purchase (one per seat)

**Validation endpoint:** `POST /v1/licenses/validate`
- Input: license key + optional device ID
- Output: validation result with product/tier info
- Can be called without authentication (designed for use inside the seller's software)

---

## Device Activations

Device activations are a **Pro-only** feature and require license keys to be enabled. Configure via `yard init --spec` (at creation) or `yard products edit` (later) — both accept `activations_enabled` and `max_activations` (1-10000).

License keys can track device activations:

- Each activation records a `device_id` provided by the buyer's software
- Sellers can configure a maximum activation limit per license key
- Activations are tracked in the `license_activations` table
- Buyers can view and manage their activations from the buyer dashboard

---

## Platform Fee

**Formula:** `platform_fee_cents = (amount_cents * fee_percent / 100) + fee_fixed_cents`

**Default:** 5% + $0.50 (50 cents)

**Resolution chain** (checked in order):
1. Per-user fee override (`user_fee_overrides` table)
2. Global platform settings (`platform_settings` table)
3. Hardcoded fallback (5% + $0.50)

**Seller earnings:** `seller_earnings_cents = amount_cents - platform_fee_cents`

Each transaction stores a snapshot of the fee percentage and fixed amount for historical auditability.

---

## Checkout Calculation Flow

The full price calculation during checkout (`CreatePaymentIntent`):

1. **Resolve tier** — Use the specified tier or fall back to the product's default tier
2. **Validate quantity** — Check quantity against the tier's seat type constraints
3. **Calculate base price** — `tier.GetPriceForQuantity(quantity)` (applies volume brackets if applicable)
4. **Apply state discount** — If product is in preorder or early access state, apply the corresponding percentage discount
5. **Apply coupon discount** — If a valid coupon code is provided, apply percentage or fixed-amount discount
6. **Calculate tax** — Via Stripe Tax API based on buyer's location
7. **Compute platform fee** — Using the fee resolution chain
8. **Calculate seller earnings** — `amount - platform_fee`
9. **Create Stripe PaymentIntent** — With metadata for tracking
10. **Create purchase record** — In the database with all pricing details
