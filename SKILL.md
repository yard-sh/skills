---
name: yard
metadata:
  author: yard.sh
  files:
    - SKILL.md
    - references/cli-commands.md
    - references/pricing-and-licensing.md
    - references/api-reference.md
    - references/landing-pages.md
    - references/releases-and-updates.md
    - references/troubleshooting.md
description: >-
  Yard is the complete platform for digital commerce, compliance, distribution, and growth so you can ship faster.
  Use this skill whenever the user mentions Yard, the Yard CLI, license keys, GitHub release integration, yard login,
  yard init, yard install, yard products, yard releases, yard keys, yard licenses, yard help, installing the yard CLI,
  pricing, trials, device activations, affiliate links, referral codes, update server, file updates, publishing a
  release, downloading updates, creating API keys, testing license-key validation, the test license key, clearing
  test device activations, coupons or discount codes, customers or buyers, transactions, sales, orders, or extending
  and shortening a buyer's free trial. Also use this skill when users are working inside a Yard codebase and need to understand
  how Yard works, its CLI commands, API, pricing model or troubleshooting common issues.
---

# Yard

Yard lets developers make their software available for sale in just a few clicks. Yard provides a product page with checkout flow. It also provides a license server, allowing sellers to integrate the Yard REST API into their app to validate a user's ownership. Sellers can also control the number of device activations allowed per account. Buyers manage their purchases through their Yard account. Sellers install the Yard GitHub App on their repo, run `yard init` from the command line, set a price, and start selling. Buyers pay via Yard Checkout page and get instant access to download current and future releases. Yard acts as the Merchant of Record, handling payments, license keys, and file hosting.

## API vs CLI — which to use

Yard has two surfaces. Pick by intent:

- **CLI (`yard …`)** — for **managing** a seller's Yard presence: creating/editing products, linking repos, scaffolding and publishing landing pages, viewing products, running discount codes, and reading who bought what. An LLM/agent working on a seller's codebase should drive all management through the CLI. This includes the seller's own reporting — `yard customers` and `yard transactions` — which an API key cannot reach.
- **REST API** — for **integrating** Yard into shipped software: validating a buyer's license at runtime, deactivating a device, fetching the latest release, reading product metadata, managing a buyer's subscription. The API does **not** replace the CLI for catalog management — an agent that wants to "create a product" runs `yard init`, not an HTTP call.

API access uses an **API key with scoped permissions**. Create one from the CLI with `yard keys create` (see [references/releases-and-updates.md](references/releases-and-updates.md)) or from the dashboard at https://yard.sh/dashboard/api-keys?action=create. See [references/api-reference.md](references/api-reference.md) for endpoint details.

For shipped end-user software, the API supports two auth approaches and either works — pick by what fits the product:

- **License key** (per-buyer) hitting `GET /v1/updates/latest`. Easiest if the product issues license keys: each buyer's key is unique, so revocation, activation limits, and per-customer rate limits work for free, and there's no shared secret to embed in the binary.
- **Embedded API key** (one shared key) hitting `GET /v1/products/{id}/releases/latest` with the `releases:download` scope. Simpler app UX (no key entry), at the cost of per-buyer revocation — every install carries the same key.

See [references/releases-and-updates.md](references/releases-and-updates.md).

## Onboarding / new product setup — agent workflow

When a user asks to get onboarded to Yard, set up a new product, run `yard init`, publish their software on Yard, or any equivalent request, **do not immediately run commands**. First ask the user which mode they prefer:

1. **Guided (step by step).** The user drives; you explain each step and wait for their input. Start by asking which directory to run `yard init` in, then walk through `yard login` (if needed) and the `yard init` prompts one at a time, pausing for confirmation between commands.
2. **Autopilot (the agent handles it).** You drive the CLI on the user's behalf. In this mode:
   1. Ask the user for a **brief description of the product** (what it is, who it's for, and — if they already have one in mind — a rough price point).
   2. Based on the description, formulate a **pricing recommendation** using the options the CLI actually supports, and note which pieces require **Yard Pro** (check with `yard me --json` → `.permissions`):
      - **One-time purchase, single tier** — simple products, one price, lifetime access. Works on the free plan.
      - **Subscription, single tier** — recurring billing (monthly, with optional yearly discount). Works on the free plan.
      - **Multiple pricing tiers** (e.g., Starter / Pro / Team) — the number allowed depends on the plan and is server-enforced via `max_pricing_tiers` (currently Basic: 2, Pro: 10; check `yard me --json` → `.permissions.max_pricing_tiers`). Good when the product has clearly differentiated feature sets.
      - **Seat-based pricing** (`fixed_pack` packs like "Team 5-Pack", or `per_seat` with min/max and volume discounts) — **requires Yard Pro** (check with `yard me --json` → `.permissions`). Good for B2B / team software.
      - **"Enterprise" / contact-sales** — Yard has no first-class enterprise tier; model it as a high-priced `per_seat` tier (Pro) or a separate high-end tier in a multi-tier product (Pro), and let the seller handle custom contracts off-platform (check with `yard me --json` → `.permissions`).
      - Also flag, where relevant, that **gift purchases, custom landing pages, license keys, device activations, and free trials are Pro-only**, and that **coupons depend on the plan** too — never state a tier from memory, read `yard me --json` → `.permissions` (coupons live under `.permissions.coupons`).
      - If license-key features are appropriate for the product (anything users install locally and where the seller needs to validate ownership at runtime), suggest enabling **license keys**, optionally **device activations** with a per-key limit, and/or a **free trial** of N days. These can be set in the same `yard init --spec` payload (Pro only — check with `yard me --json` → `.permissions`) or configured later via `yard products edit`.
      - Recommend a **launch stage**. Every new product is created in `draft` (not visible to buyers). After setup, the seller advances the stage from the Yard dashboard — stage transitions are **forward-only** (`draft` → `early_access` → `released`) and `released` is final. Two reasonable launch paths:
        - **Straight to `released`** — for finished products with no soft-launch period. Skip `early_access` entirely.
        - **`early_access` first, then `released` later** — for products the seller wants to ship but signal as still being polished. Optionally pair with `early_access_discount_percent` (1–100) so early adopters get a launch discount that disappears when the product moves to `released`. The discount field is set in the dashboard; `yard init`/`yard products edit` don't surface it today.
   3. Present the recommendation as a short plan: title, pricing model, tier(s), seat type, price(s), any Pro requirements (check with `yard me --json` → `.permissions` before suggesting Pro-only items), and the recommended launch stage (and any early-access discount). **If the product is locally-installed software** (desktop app, CLI tool, native binary), the plan must also include (a) running `yard releases publish` for each shipped version and (b) wiring `GET /v1/updates/latest` into the app's update path — otherwise the buy page sells nothing and the installed app has no update channel. See ["Desktop / CLI app integration scope"](#desktop--cli-app-integration-scope) below and [references/releases-and-updates.md](references/releases-and-updates.md). Then ask the user to **accept**, **edit**, or **switch to guided mode**.
   4. On accept: run `yard products --json` first to see what already exists (avoids accidentally creating a duplicate after a failed attempt) — each entry's `.slug` is what the rest of the CLI takes as `<slug-or-id>` or `--product`; see [references/cli-commands.md#yard-products](references/cli-commands.md#yard-products) ("Discovering a product's slug") for the common `jq` recipes. Then run `yard init --spec - --json` with the accepted plan encoded as JSON on stdin. **Do not** pipe answers to the interactive wizard — the CLI ships a non-interactive spec mode specifically for agents. See ["Autopilot: non-interactive `yard init`"](#autopilot-non-interactive-yard-init) below.
   5. On edit: adjust the plan and re-confirm before running anything.

Keep the CLI as the single source of truth for product creation — **never** try to create a product via the REST API (see "API vs CLI" above).

### Autopilot: non-interactive `yard init`

`yard init` supports three modes. Agents should always use one of the first two:

| Mode            | Invocation                           | When to use                                                                                                                       |
| --------------- | ------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| **Spec**        | `yard init --spec <file\|->`         | Creating a new product. Accepts the full pricing shape as JSON.                                                                   |
| **Link**        | `yard init --product <slug-or-uuid>` | Linking the current directory to an existing product.                                                                             |
| **Interactive** | `yard init`                          | Humans only. Trying to drive this from an agent via stdin is a dead end — the prompt order is load-bearing and changes over time. |

Global non-interactive flags:

- `--json` — emit a single JSON object on stdout; logs (including HTTP request lines and progress messages) go to stderr. Safe to pipe into `jq`.
- `--page` / `--no-page` — explicitly opt in or out of landing-page scaffolding without prompting. `--json` defaults to no page unless `--page` is set.
- `--link-repo` / `--no-link-repo` — force GitHub repo linking on or off in spec mode. The default tries to link if (a) the cwd is a git repo with a GitHub remote and (b) the Yard GitHub App is already installed. If the App isn't installed, linking is silently skipped — the product is still created.

If the user isn't authenticated, all non-interactive modes fail with `not logged in. Run 'yard login' first`. Ask the user to run `yard login` in their terminal, then retry.

#### Spec schema

The spec matches `CreateProductRequest` exactly. Only `title` and `tiers` are strictly required; `pricing_model` defaults to `one_time`.

```jsonc
{
  "title": "My Product", // required, 1–60 chars, must contain a letter/digit
  "pricing_model": "one_time", // "one_time" | "subscription"
  "tiers": [
    // one or more tiers; how many depends on the plan (server-enforced via max_pricing_tiers — Basic: 2, Pro: 10)
    {
      "name": "Base", // required; for single-tier just use "Base"
      "price_cents": 1900, // 0 for free, else 300..1000000
      "is_default": true, // exactly one tier must be the default
      "seat_type": "single", // "single" | "fixed_pack" | "per_seat" (seat-based needs seat_based_pricing; server-enforced)
      "seat_count": null, // required for fixed_pack (2..1000)
      "min_seats": null, // per_seat; defaults to 1
      "max_seats": null, // per_seat; optional
      "yearly_discount_percent": null, // subscription only, 1..100
      "volume_brackets": [], // per_seat only; contiguous, increasing discount
      "free_trial_enabled": false, // needs free_trials permission when true — per-tier flag, not product-level
      "free_trial_days": null, // 1..365; required when free_trial_enabled is true
      "trial_requires_card": true, // per-tier; when true, subscription-tier trials collect a card via checkout (omitted = true)
      "gift_enabled": false, // per-tier; whether this tier can be bought as a gift (one-time tiers only)
    },
  ],
  // Optional product-level seller settings.
  // license keys + activations depend on the plan (server-enforced); the CLI sends the
  // request either way and the server returns upgrade_required if not included.
  // Applied via a follow-up PUT /v1/products/{id} after creation.
  "license_key_enabled": false, // needs license_keys permission when true
  "activations_enabled": false, // needs device_activations permission when true; requires license_key_enabled=true
  "max_activations": null, // 1..10000; only meaningful when activations_enabled
}
```

> **Heads up — per-tier trials.** `free_trial_enabled` / `free_trial_days` live on each tier, not on the product. A product "offers a trial" when at least one of its tiers has them set. To enable a trial on an existing product later, use `yard products tiers edit <slug> <tier-id-or-name> --spec -` (see [references/cli-commands.md](references/cli-commands.md#yard-products-tiers)). Putting `free_trial_enabled` at the product level in a spec will be rejected with `unknown field`.

#### Typical agent flow

```sh
# 1. Discover what exists (safe to run on every autopilot turn).
yard products --json

# 2. If the user asked for a new product, create it from a spec.
yard init --spec - --json <<'EOF'
{
  "title": "Simple Note",
  "pricing_model": "one_time",
  "tiers": [{ "name": "Base", "price_cents": 1900, "seat_type": "single", "is_default": true }]
}
EOF

# 3. If the user already has the product, just link this directory to it.
yard init --product simple-note --json
```

`yard init --json` output shape:

```json
{
  "product": {
    "id": "...",
    "slug": "simple-note",
    "title": "Simple Note",
    "buy_url": "https://yard.sh/@alice/simple-note",
    "profile_url": "https://yard.sh/@alice",
    "created": true
  },
  "settings_file": "/abs/path/.yard/settings.json",
  "github_repo_linked": false,
  "landing_page": null
}
```

#### Troubleshooting

- **`yard init` hangs silently.** You're in the interactive wizard. Interrupt, then retry with `--spec -` (for a new product) or `--product <slug>` (for an existing one).
- **`403 not logged in`.** Ask the user to run `yard login` in their terminal — you can't drive the OAuth browser flow.
- **Plan-gated 403s.** The account's plan doesn't include the feature you used (an extra tier, seat-based pricing, license keys, device activations, a free trial, custom pages, coupons, …). Product endpoints tag these `upgrade_required`; permission-gated endpoints like `yard coupons` answer a plain `FORBIDDEN` 403 — either way it's a plan problem, not a bad request. The CLI doesn't gate this client-side; the server decides. Either send a spec the plan supports, or ask the user to upgrade at https://yard.sh/pricing. To see what the plan includes, read `yard me --json` → `.permissions`.
- **Duplicate product after a failed attempt.** Run `yard products --json` first — if the product already exists, link it with `yard init --product <slug>` instead of re-creating.
- **Need to change settings on an existing product.** Use `yard products edit <slug> --spec -` with an `UpdateProductRequest` JSON body for product-level fields (`license_key_enabled`, `activations_enabled`, `max_activations`). For tier mutations — including **enabling a free trial on a specific tier**, its `trial_requires_card` / `gift_enabled` flags, changing a tier's price, or removing a tier — use `yard products tiers add | edit | rm` (see the command table). The legacy product-level `free_trial_enabled` / `free_trial_days` / `trial_requires_card` / `gift_enabled` fields no longer exist; they were all moved per-tier.

### Desktop / CLI app integration scope

If the product is **locally-installed software** — a desktop app, CLI tool, native binary, anything the buyer downloads and runs on their own machine — `yard init` alone is **not** a complete sales surface. The buy page has nothing to download until a release is published, and the installed app has no built-in update path. Whenever you detect this product type during the autopilot flow, the plan you present in step 3 must cover both halves:

1. **Publish releases.** After `yard init`, the seller publishes each shipped version with `yard releases publish` (spec mode is the agent-friendly form). This is what populates the buy page's download. See [references/releases-and-updates.md](references/releases-and-updates.md) — _Publishing a release with the CLI_.
2. **Wire the update endpoint.** The installed app needs an auto-updater that pulls the latest release from Yard. Two auth options — pick whichever fits:
   - **License key** → `GET https://api.yard.sh/v1/updates/latest?license_key=<key>`. One key per buyer (revocable per customer). Easiest if the product already issues license keys.
   - **Embedded API key** → `GET https://api.yard.sh/v1/products/{productId}/releases/latest` with `Authorization: Bearer yard_<key>` and the `releases:download` scope. One key shared across all installs (no per-buyer revocation), but no key-entry UX in the app.

   Tell the user this needs to be wired into their app's auto-updater. See [references/releases-and-updates.md](references/releases-and-updates.md) — _Downloading releases_.

Scope is deliberately narrow: this section covers **publish + updates only**. License validation (refusing to run for non-buyers), device activations, and free trials are separate Pro-only concerns and should not be folded into this step (check with `yard me --json` → `.permissions` before suggesting any of those).

For content products or anything static the buyer does not install locally, skip this section — the default `yard init` + landing-page flow is sufficient. For **SaaS / web apps, Yard hosts the app itself** — see the next section.

### Web app / SaaS scope

If the product **is** the web app — the buyer uses it in the browser rather than downloading anything — Yard hosts frontend, backend, database, and buyer sign-in. Requires the `web_apps` permission (Pro; check `yard me --json` → `.permissions`). Whenever you detect this product type, the plan you present must cover:

1. **Scaffold and build.** `yard app init` writes a zero-dependency working bundle (plain `_worker.js` fetch handler, static frontend, first migration, `yard.json`). Build the user's actual app inside that contract. **No ports, no `listen()`, no Express** — the backend is a fetch handler; route by path; use relative URLs in the frontend. Full contract: [references/webapps.md](references/webapps.md).
2. **Never build auth.** The Yard edge signs buyers in and injects trusted `X-Yard-User-Id` / `X-Yard-Entitlement` headers; `yard.json`'s `access: customers` is a complete paywall with zero app code. Building your own login/OAuth/session layer is a bug.
3. **Deploy → test → promote.** `yard app deploy` targets the development environment; verify against the deployed dev app (and `npx wrangler dev` locally), then `yard env promote development production` to go live. Data and secrets never promote — set production secrets explicitly.
4. **Pricing still applies.** The app is gated by normal Yard pricing (tiers, trials, subscriptions) — configure it as for any product; the product page remains the sales surface and the app lives under `/app/`.

Releases/`yard releases publish` are **not** part of this flow — web apps deploy, they don't ship files.

### Testing license-key validation

Every product with `license_key_enabled: true` has a sandbox **test license key** — a real license key value the seller can pass to `POST /v1/licenses/validate` and have the API treat it like a paid customer's key. Test activations live in a separate `test_activations` table, so they never collide with real customers.

**Auth:** `POST /v1/licenses/validate` is **not unauthenticated**. It requires an API key with the `licenses:validate` scope (`Authorization: Bearer yard_<key>`). The test license key goes in the **request body**, not the header — it's the data being validated, not the credential.

Three CLI commands cover the loop:

```sh
# Print the test key for the current product (or pass --product <slug>).
yard licenses test-key

# See what's currently activated against the test key.
yard licenses test-activations list

# Wipe every test device when you've hit max_activations during testing.
yard licenses test-activations clear --yes
```

Typical agent flow when wiring license validation into a seller's app:

1. `yard keys create --spec - --json <<<'{"name":"local-validate","scopes":["licenses:validate"]}'` to mint an API key (capture `key` from the JSON output).
2. `yard licenses test-key --json` to grab the test license key.
3. From the app under test, `POST /v1/licenses/validate` with the API key in the `Authorization` header and `{"license_key": "<test-key>", "device_id": "<your-choice>"}` in the body. Verify the response shape matches what your app expects.
4. `yard licenses test-activations list` to confirm the device showed up.
5. After enough iterations to hit `max_activations`, `yard licenses test-activations clear` resets the slate.

All three `yard licenses` commands accept `--product <slug>` and fall back to the slug in `.yard/settings.json`. `--json` is supported on each for scripted use.

## Quick Start

### 1. Install the CLI

**macOS / Linux:**

```sh
curl -fsSL https://cli.yard.sh | sh
```

**Windows (PowerShell):**

```powershell
irm https://cli.yard.sh/install.ps1 | iex
```

- Linux/macOS installs to `/usr/local/bin` (if writable) or `~/.local/bin`
- Windows installs to `$env:LOCALAPPDATA\yard\bin`
- Set `YARD_INSTALL_DIR` to customize the install location
- The installer auto-detects OS (linux/darwin/windows) and architecture (amd64/arm64)
- Restart your terminal after installation so the new PATH takes effect

### 2. Log In

```sh
yard login
```

- Opens your default browser to authenticate via GitHub OAuth
- Starts a local callback server on **port 9876** to receive the token
- Saves credentials to `~/.yard/config.json` (file permissions 0600)
- Sessions last **30 days** before requiring re-authentication
- If port 9876 is already in use, login will fail — close the conflicting process first

### 3. Initialise a Project

```sh
cd /path/to/your-project
yard init
```

The interactive flow:

1. Checks for CLI updates (prompts to install if available)
2. Verifies login (runs login flow if needed)
3. **Best-effort** git context: if inside a git repo with a GitHub remote, prompts to install the Yard GitHub App and verifies the repo. Any step failing here is non-fatal — the product will simply be created without a linked repo.
4. Lets you select an existing product or create a new one (prompts for title + price on create)
5. Writes `.yard/settings.json` in the current directory
6. Offers to scaffold a custom landing page. If accepted, pulls or scaffolds starter files into `.yard/landing-page/` and attempts to publish. If the plan doesn't include custom pages, the server returns `upgrade_required`; the CLI shows a friendly upgrade link and keeps the saved draft.

**No git repo required.** `yard init` works inside any directory — it will just create the product without a GitHub link. Only GitHub repositories are supported for linking.

## CLI Commands

| Command                                                                       | Description                                                                                                                                                                                                                                                             |
| ----------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `yard login`                                                                  | Authenticate via GitHub OAuth                                                                                                                                                                                                                                           |
| `yard logout`                                                                 | Clear local credentials (`~/.yard/config.json`)                                                                                                                                                                                                                         |
| `yard me [--json]`                                                            | Show the current user (id, username, GitHub, email), plan, and the `permissions` map. Read `--json` → `.permissions` to see what the account can do; feature limits are server-enforced, so you can also just attempt an action and handle `upgrade_required`.          |
| `yard init`                                                                   | Set up a Yard project in the current directory — create or select a product, scaffold `.yard/`, optional landing-page setup. Supports `--spec <file\|->`, `--product <slug>`, `--json`, `--page`/`--no-page`, `--link-repo`/`--no-link-repo` for non-interactive use.   |
| `yard products [--json]`                                                      | List your published products with stats                                                                                                                                                                                                                                 |
| `yard products show <slug-or-id> [--json]`                                    | Print one product's full detail, including `tiers[]` with per-tier `free_trial_enabled` / `free_trial_days`. Use this (not `yard products`) when you need to check whether a tier offers a trial. This command is used to retrieve any sort of metadata about a product |
| `yard products edit [slug-or-id] [--spec <file\|->] [--json]`                 | Modify product-level seller settings (`license_key_enabled`, `activations_enabled`, `max_activations`) and optionally the full `tiers[]` array (full-replace). No client-side plan gate — the server returns `upgrade_required` if the plan doesn't include a setting.       |
| `yard products tiers add <slug> --spec <file\|->`                             | Append one tier without rebuilding the full tier list.                                                                                                                                                                                                                  |
| `yard products tiers edit <slug> <tier-id-or-name> --spec <file\|->`          | Apply a partial spec to one tier (e.g. enable a free trial on Base: `{"free_trial_enabled": true, "free_trial_days": 14}`).                                                                                                                                             |
| `yard products tiers rm <slug> <tier-id-or-name> [--yes] [--promote-default]` | Remove a tier. Tiers with paid transactions are kept as historical records but marked non-default.                                                                                                                                                                      |
| `yard releases publish [tag] [flags]`                                         | Create a new release with optional file assets. Supports `--spec <file\|->` and `--json` for non-interactive use. See [references/releases-and-updates.md](references/releases-and-updates.md).                                                                         |
| `yard keys list [--json]`                                                     | List your API keys (name, prefix, scopes, last-used, created). The full secret is never shown.                                                                                                                                                                          |
| `yard keys create [name] [flags]`                                             | Mint a new API key. Supports `--spec <file\|->` and `--json`. The full secret is shown only once at creation.                                                                                                                                                           |
| `yard licenses test-key [--product <slug>] [--json]`                          | Print the product's sandbox **test license key** — usable with `POST /v1/licenses/validate` to exercise license-validation logic without buying your own product.                                                                                                       |
| `yard licenses test-activations list [--product <slug>] [--json]`             | List active test device activations attached to the product's test license key.                                                                                                                                                                                         |
| `yard licenses test-activations clear [--product <slug>] [--yes] [--json]`    | Deactivate every test device on the product's test key (real customer activations are untouched).                                                                                                                                                                       |
| `yard coupons [--json]`                                                       | List discount codes with their discount, scope, usage, and derived status (`active`, `scheduled`, `expired`, `used up`, `inactive`).                                                                                                                                    |
| `yard coupons show <code-or-id> [--json]`                                     | One coupon plus its redemption analytics. `--json` emits `{coupon, analytics}`. Accepts the code or the UUID.                                                                                                                                                           |
| `yard coupons create <code> [flags] [--spec <file\|->] [--json]`              | Create a code. `--percent 20` or `--amount 5` (**dollars**; a spec's `discount_value` is **cents** for `fixed_amount`), `--products <csv>`, `--max-uses`, `--expires`, `--valid-from`, `--subscription-duration once\|forever`. Flags override `--spec` field by field.  |
| `yard coupons generate --count N [flags] [--json]`                            | Bulk-generate up to 100 unique codes sharing one discount (`--prefix`, `--length`). Codes are returned **once** — capture them from the output.                                                                                                                        |
| `yard coupons update <code-or-id> [flags] [--spec <file\|->] [--json]`        | Partial update. `--activate`/`--deactivate`, and explicit clearing via `--no-expiry`, `--no-valid-from`, `--unlimited-uses` (or `null` in a spec). Discounts can't change once a coupon has been redeemed.                                                              |
| `yard coupons rm <code-or-id> [--yes]`                                        | Delete an unused coupon. Redeemed coupons can't be deleted — deactivate them instead.                                                                                                                                                                                  |
| `yard coupons transactions <code-or-id> [--json]`                             | The purchases a coupon was redeemed on.                                                                                                                                                                                                                                |
| `yard coupons validate <code> --product <slug> [--json]`                      | Dry-run a code through the checkout-time check and see what the buyer would pay. The product must be public.                                                                                                                                                            |
| `yard customers [--json]`                                                     | List the buyers who completed a purchase, with order count, spend, and activity dates. `--product <slug>` narrows both the rows and the summary to one product's buyers. Amounts are pre-formatted display strings, not cents.                                          |
| `yard customers show <cust-id> [--json]`                                      | One buyer's totals plus their orders (refunded ones included). Takes the opaque `cust_xxxxxxxx` id from the list, not an email.                                                                                                                                         |
| `yard transactions [--json]`                                                  | List sales. `--trials`, `--product <slug>`, `--start`/`--end` narrow the rows and total; the earnings summary stays account-wide.                                                                                                                                       |
| `yard transactions show <order-id> [--json]`                                  | One sale in full — tier, coupon, refund state, trial expiry. Takes the short `order_xxxxxxxx` id or the full UUID.                                                                                                                                                      |
| `yard transactions trial <order-id> --add-days N [--json]`                    | Lengthen (`7`) or shorten (`-3`) a free trial, ±365. Days are added to the **current expiry, not today**; an expired trial whose new expiry is in the future goes back to active (`reactivated: true`). The buyer is emailed. Needs `.permissions.sell_products`. |
| `yard page init`                                                              | Create a `.yard/` project directory linked to a product and scaffold a hello-world landing page                                                                                                                                                                         |
| `yard page status`                                                            | Diff local landing-page files vs the remote draft (no writes)                                                                                                                                                                                                           |
| `yard page ls [--source draft\|published]`                                    | List files in the remote draft or published bundle                                                                                                                                                                                                                      |
| `yard page push [--prune] [--publish]`                                        | Upload changed local files to the remote draft; optionally prune + publish                                                                                                                                                                                              |
| `yard page pull [--source draft\|published] [--dir PATH]`                     | Download remote files to the local landing-page directory                                                                                                                                                                                                               |
| `yard page publish`                                                           | Promote the current draft bundle to live                                                                                                                                                                                                                                |
| `yard page revert [--yes]`                                                    | Discard the draft and restore it from the published bundle                                                                                                                                                                                                              |
| `yard version`                                                                | Show version, commit hash, platform, Go version                                                                                                                                                                                                                         |
| `yard update`                                                                 | Download and install the latest CLI version                                                                                                                                                                                                                             |
| `yard update --check`                                                         | Check for updates without installing                                                                                                                                                                                                                                    |
| `yard uninstall`                                                              | Remove CLI binary and config directory (`--force` to skip confirmation)                                                                                                                                                                                                 |

See [references/cli-commands.md](references/cli-commands.md) for detailed command documentation.

## How It Works

1. **Seller** installs the Yard GitHub App on their repository
2. **Seller** runs `yard init` to create a product with pricing (and optionally a custom landing page)
3. When the seller publishes a **GitHub release**, Yard automatically captures the release assets via webhook and hosts them
4. **Buyers** visit the product page, pay via Stripe, and get instant download access
5. Seller earnings are tracked and paid out by admin

## Custom Landing Pages

Every Yard product has a public landing page. Pro sellers can replace the default layout with their own HTML/CSS/JS via a custom landing page (check with `yard me --json` → `.permissions`). The same editor is available from both the frontend dashboard and the CLI, so the flow can be driven by an LLM-based coding agent.

For everything an agent needs to **author** the page itself — how to read product data at runtime (`window.yard.product`), the `data-yard` / `data-action` attribute conventions, the `window.yard.checkout(...)` / `trial()` helpers, and the full `PublicProduct` field reference — see [references/landing-pages.md](references/landing-pages.md). The remainder of this section covers the **management** flow (scaffolding, pushing, publishing).

**Limits** (enforced server-side; also validated client-side before upload):

- 20 files max per bundle
- 1 MB max per file
- 5 MB max total bundle size
- Allowed extensions: `.html .css .js .json .svg .png .jpg .jpeg .webp .gif .woff2`
- Paths: letters/digits/`._-` only, at most one subdirectory level
- `index.html` is required to publish

### Project Layout

`yard page init` creates a `.yard/` directory at the project root:

```
<project>/
└── .yard/
    ├── settings.json         # product_slug, version, ignore_files
    └── landing-page/
        ├── index.html
        ├── styles.css
        └── ...
```

`.yard/settings.json` schema (v1):

```json
{
  "version": 1,
  "product_slug": "my-product",
  "ignore_files": ["*.bak", "drafts/**"]
}
```

`ignore_files` uses shell-style globs relative to `.yard/landing-page/`; `**` matches any depth. Dotfiles are always ignored.

### Typical Flow

1. `cd <project>` and `yard page init` — scaffolds `.yard/` and a hello-world page
2. Edit files in `.yard/landing-page/` (by hand, or prompt an agent to do it)
3. `yard page status` — preview the diff without writing anything
4. `yard page push` — upload changed files; prints a `Preview:` URL for the draft
5. `yard page push --publish` — upload and go live in one step; prints both `Preview:` and `Live:` URLs

### Driving It From an Agent

All `yard page` mutating commands accept:

- `--product <slug-or-uuid>` — override the product in `.yard/settings.json`
- `--project <path>` — project root override (defaults to walking up from cwd for `.yard/`)
- `--json` — emit a single machine-readable JSON object; logs go to stderr
- `--yes` — skip confirmation prompts (for `revert`, `push --prune`)

Exit codes: `0` = success, `1` = fatal (auth/validation/network), `2` = partial success (`push` only).

Example `push --publish --json` output:

```json
{
  "product": "my-slug",
  "uploaded": ["index.html", "styles.css"],
  "skipped": [],
  "deleted": [],
  "published": true,
  "preview_url": "https://yard.sh/api/v1/products/.../custom-page/preview",
  "live_url": "https://yard.sh/@alice/my-slug",
  "errors": []
}
```

Diff is SHA-256 content-addressed against the server's existing hashes, so repeated pushes with no changes upload nothing.

## Configuration

| Item             | Value                                                           |
| ---------------- | --------------------------------------------------------------- |
| Config file      | `~/.yard/config.json`                                           |
| File permissions | `0600` (owner read/write only)                                  |
| Config directory | `~/.yard/`                                                      |
| Contents         | `session_token`, `user` (id, github_username, email), `api_url` |
| Auth header      | `Authorization: Session {token}`                                |

## Key URLs

| URL                                                           | Purpose                                 |
| ------------------------------------------------------------- | --------------------------------------- |
| `https://yard.sh`                                             | Yard website                            |
| `https://api.yard.sh`                                         | API base URL                            |
| `https://cli.yard.sh`                                         | CLI binary downloads and version checks |
| `https://github.com/apps/yard-app-official/installations/new` | Install the Yard GitHub App             |

## Key Features

- **Pricing tiers** — Multiple tiers per product (how many depends on the plan; Basic: 2, Pro: 10), with single, fixed-pack, or per-seat licensing
- **Volume discounts** — Percentage discounts at quantity thresholds for per-seat tiers
- **Product stages** — Draft → Early Access → Released, forward-only (Early Access supports a launch discount; Released is final)
- **Free trials** — Configurable trial periods (1-365 days)
- **License keys** — Automatic generation with device activation tracking
- **Coupons** — Percentage or fixed-amount discounts, single or bulk-generated, managed with `yard coupons` (plan-gated — check `yard me --json` → `.permissions.coupons`)
- **Gift purchases** — Buy for someone else via recipient email (Pro sellers)
- **Subscriptions** — Recurring billing with optional yearly discounts
- **Webhooks** — Get notified when sales happen
- **API keys** — Programmatic access for _integrating_ Yard into your software (license validation, release metadata, subscriptions) — not for catalog management (use the CLI)

## Reference Files

| Topic                                                                               | File                                                                       |
| ----------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Detailed CLI command reference                                                      | [references/cli-commands.md](references/cli-commands.md)                   |
| Pricing, licensing, coupons, trials                                                 | [references/pricing-and-licensing.md](references/pricing-and-licensing.md) |
| REST API (integration endpoints for license validation, releases, subscriptions)    | [references/api-reference.md](references/api-reference.md)                 |
| Custom landing pages — runtime data, `data-yard` / `data-action`, `window.yard` API | [references/landing-pages.md](references/landing-pages.md)                 |
| Publishing releases, downloading updates, API keys                                  | [references/releases-and-updates.md](references/releases-and-updates.md)   |
| Troubleshooting common issues                                                       | [references/troubleshooting.md](references/troubleshooting.md)             |
