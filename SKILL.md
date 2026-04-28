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
  yard init, yard install, yard products, yard releases, yard keys, yard help, installing the yard CLI, pricing, trials,
  device activations, affiliate links, referral codes, update server, file updates, publishing a release, downloading
  updates, creating API keys. Also use this skill when users are working inside a Yard codebase and need to understand
  how Yard works, its CLI commands, API, pricing model or troubleshooting common issues.
---

# Yard

Yard lets developers make their software available for sale in just a few clicks. Yard provides a product page with checkout flow. It also provides a license server, allowing sellers to integrate the Yard REST API into their app to validate a user's ownership. Sellers can also control the number of device activations allowed per account. Buyers manage their purchases through their Yard account. Sellers install the Yard GitHub App on their repo, run `yard init` from the command line, set a price, and start selling. Buyers pay via Yard Checkout page and get instant access to download current and future releases. Yard acts as the Merchant of Record, handling payments, license keys, and file hosting.

**Platform fee:** 4% + $0.40 per transaction.

## API vs CLI — which to use

Yard has two surfaces. Pick by intent:

- **CLI (`yard …`)** — for **managing** a seller's Yard presence: creating/editing products, linking repos, scaffolding and publishing landing pages, viewing products, etc. An LLM/agent working on a seller's codebase should drive all management through the CLI.
- **REST API** — for **integrating** Yard into shipped software: validating a buyer's license at runtime, deactivating a device, fetching the latest release, reading product metadata, managing a buyer's subscription. The API does **not** replace the CLI for catalog management — an agent that wants to "create a product" runs `yard init`, not an HTTP call.

API access uses an **API key with scoped permissions**. Create one from the CLI with `yard keys create` (see [references/releases-and-updates.md](references/releases-and-updates.md)) or from the dashboard at https://yard.sh/dashboard/api-keys?action=create. See [references/api-reference.md](references/api-reference.md) for endpoint details.

For shipped end-user software, use the **license-key update server** rather than API keys — each buyer gets a unique license, so you can revoke or rate-limit per customer. See [references/releases-and-updates.md](references/releases-and-updates.md).

## Onboarding / new product setup — agent workflow

When a user asks to get onboarded to Yard, set up a new product, run `yard init`, publish their software on Yard, or any equivalent request, **do not immediately run commands**. First ask the user which mode they prefer:

1. **Guided (step by step).** The user drives; you explain each step and wait for their input. Start by asking which directory to run `yard init` in, then walk through `yard login` (if needed) and the `yard init` prompts one at a time, pausing for confirmation between commands.
2. **Autopilot (the agent handles it).** You drive the CLI on the user's behalf. In this mode:
   1. Ask the user for a **brief description of the product** (what it is, who it's for, and — if they already have one in mind — a rough price point).
   2. Based on the description, formulate a **pricing recommendation** using the options the CLI actually supports, and note which pieces require **Yard Pro**:
      - **One-time purchase, single tier** — simple products, one price, lifetime access. Works on the free plan.
      - **Subscription, single tier** — recurring billing (monthly, with optional yearly discount). Works on the free plan.
      - **Multiple pricing tiers** (up to 5 — e.g., Starter / Pro / Team) — **requires Yard Pro**. Good when the product has clearly differentiated feature sets.
      - **Seat-based pricing** (`fixed_pack` packs like "Team 5-Pack", or `per_seat` with min/max and volume discounts) — **requires Yard Pro**. Good for B2B / team software.
      - **"Enterprise" / contact-sales** — Yard has no first-class enterprise tier; model it as a high-priced `per_seat` tier (Pro) or a separate high-end tier in a multi-tier product (Pro), and let the seller handle custom contracts off-platform.
      - Also flag, where relevant, that **coupons, gift purchases, custom landing pages, license keys, device activations, and free trials are Pro-only**.
      - If license-key features are appropriate for the product (anything users install locally and where the seller needs to validate ownership at runtime), suggest enabling **license keys**, optionally **device activations** with a per-key limit, and/or a **free trial** of N days. These can be set in the same `yard init --spec` payload (Pro only) or configured later via `yard products edit`.
   3. Present the recommendation as a short plan (title, pricing model, tier(s), seat type, price(s), any Pro requirements) and ask the user to **accept**, **edit**, or **switch to guided mode**.
   4. On accept: run `yard products --json` first to see what already exists (avoids accidentally creating a duplicate after a failed attempt), then run `yard init --spec - --json` with the accepted plan encoded as JSON on stdin. **Do not** pipe answers to the interactive wizard — the CLI ships a non-interactive spec mode specifically for agents. See ["Autopilot: non-interactive `yard init`"](#autopilot-non-interactive-yard-init) below.
   5. On edit: adjust the plan and re-confirm before running anything.

Keep the CLI as the single source of truth for product creation — **never** try to create a product via the REST API (see "API vs CLI" above).

### Autopilot: non-interactive `yard init`

`yard init` supports three modes. Agents should always use one of the first two:

| Mode | Invocation | When to use |
|---|---|---|
| **Spec** | `yard init --spec <file\|->` | Creating a new product. Accepts the full pricing shape as JSON. |
| **Link** | `yard init --product <slug-or-uuid>` | Linking the current directory to an existing product. |
| **Interactive** | `yard init` | Humans only. Trying to drive this from an agent via stdin is a dead end — the prompt order is load-bearing and changes over time. |

Global non-interactive flags:

- `--json` — emit a single JSON object on stdout; logs (including HTTP request lines and progress messages) go to stderr. Safe to pipe into `jq`.
- `--page` / `--no-page` — explicitly opt in or out of landing-page scaffolding without prompting. `--json` defaults to no page unless `--page` is set.
- `--link-repo` / `--no-link-repo` — force GitHub repo linking on or off in spec mode. The default tries to link if (a) the cwd is a git repo with a GitHub remote and (b) the Yard GitHub App is already installed. If the App isn't installed, linking is silently skipped — the product is still created.

If the user isn't authenticated, all non-interactive modes fail with `not logged in. Run 'yard login' first`. Ask the user to run `yard login` in their terminal, then retry.

#### Spec schema

The spec matches `CreateProductRequest` exactly. Only `title` and `tiers` are strictly required; `pricing_model` defaults to `one_time`.

```jsonc
{
  "title": "My Product",              // required, 1–60 chars, must contain a letter/digit
  "pricing_model": "one_time",        // "one_time" | "subscription"
  "tiers": [                          // 1..10 tiers; multiple tiers require Pro
    {
      "name": "Base",                 // required; for single-tier just use "Base"
      "price_cents": 1900,            // 0 for free, else 300..1000000
      "is_default": true,             // exactly one tier must be the default
      "seat_type": "single",          // "single" | "fixed_pack" (Pro) | "per_seat" (Pro)
      "seat_count": null,             // required for fixed_pack (2..1000)
      "min_seats": null,              // per_seat; defaults to 1
      "max_seats": null,              // per_seat; optional
      "yearly_discount_percent": null,// subscription only, 1..100
      "volume_brackets": []           // per_seat only; contiguous, increasing discount
    }
  ],
  // Optional seller settings — all Pro-only when set to true.
  // Applied via a follow-up PUT /v1/products/{id} after creation.
  "license_key_enabled": false,    // Pro-only when true
  "activations_enabled": false,    // Pro-only when true; requires license_key_enabled=true
  "max_activations": null,         // 1..10000; only meaningful when activations_enabled
  "free_trial_enabled": false,     // Pro-only when true
  "free_trial_days": null          // 1..365; defaults to 7 if trial enabled without value
}
```

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
    "id": "...", "slug": "simple-note", "title": "Simple Note",
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
- **Pro-gated feature error.** The user's account isn't on Pro. Either pick a spec shape that works on the free plan (single-tier, `seat_type=single`, no license/activation/trial settings) or ask the user to upgrade at https://yard.sh/upgrade.
- **License/activation/trial settings rejected with 403 (`pro_required`).** Same root cause: free account. Drop those fields from the spec or upgrade.
- **Duplicate product after a failed attempt.** Run `yard products --json` first — if the product already exists, link it with `yard init --product <slug>` instead of re-creating.
- **Need to change settings on an existing product.** Use `yard products edit <slug> --spec -` with an `UpdateProductRequest` JSON body (the same five settings fields above).

## Quick Start

### 1. Install the CLI

**macOS / Linux:**
```sh
curl -fsSL https://api.yard.sh/yard-cli/install.sh | sh
```

**Windows (PowerShell):**
```powershell
irm https://api.yard.sh/yard-cli/install.ps1 | iex
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
6. Offers to scaffold a custom landing page. If accepted, pulls or scaffolds starter files into `.yard/landing-page/` and attempts to publish. Non-Pro accounts see a friendly upgrade link and keep the saved draft.

**No git repo required.** `yard init` works inside any directory — it will just create the product without a GitHub link. Only GitHub repositories are supported for linking.

## CLI Commands

| Command | Description |
|---|---|
| `yard login` | Authenticate via GitHub OAuth |
| `yard logout` | Clear local credentials (`~/.yard/config.json`) |
| `yard init` | Set up a Yard project in the current directory — create or select a product, scaffold `.yard/`, optional landing-page setup. Supports `--spec <file\|->`, `--product <slug>`, `--json`, `--page`/`--no-page`, `--link-repo`/`--no-link-repo` for non-interactive use. |
| `yard products [--json]` | List your published products with stats |
| `yard releases publish [tag] [flags]` | Create a new release with optional file assets. Supports `--spec <file\|->` and `--json` for non-interactive use. See [references/releases-and-updates.md](references/releases-and-updates.md). |
| `yard keys list [--json]` | List your API keys (name, prefix, scopes, last-used, created). The full secret is never shown. |
| `yard keys create [name] [flags]` | Mint a new API key. Supports `--spec <file\|->` and `--json`. The full secret is shown only once at creation. |
| `yard page init` | Create a `.yard/` project directory linked to a product and scaffold a hello-world landing page |
| `yard page status` | Diff local landing-page files vs the remote draft (no writes) |
| `yard page ls [--source draft\|published]` | List files in the remote draft or published bundle |
| `yard page push [--prune] [--publish]` | Upload changed local files to the remote draft; optionally prune + publish |
| `yard page pull [--source draft\|published] [--dir PATH]` | Download remote files to the local landing-page directory |
| `yard page publish` | Promote the current draft bundle to live |
| `yard page revert [--yes]` | Discard the draft and restore it from the published bundle |
| `yard version` | Show version, commit hash, platform, Go version |
| `yard update` | Download and install the latest CLI version |
| `yard update --check` | Check for updates without installing |
| `yard uninstall` | Remove CLI binary and config directory (`--force` to skip confirmation) |

See [references/cli-commands.md](references/cli-commands.md) for detailed command documentation.

## How It Works

1. **Seller** installs the Yard GitHub App on their repository
2. **Seller** runs `yard init` to create a product with pricing (and optionally a custom landing page)
3. When the seller publishes a **GitHub release**, Yard automatically captures the release assets via webhook and hosts them
4. **Buyers** visit the product page, pay via Stripe, and get instant download access
5. Yard takes a **5% + $0.50 platform fee**; seller earnings are tracked and paid out by admin

## Custom Landing Pages

Every Yard product has a public landing page. Pro sellers can replace the default layout with their own HTML/CSS/JS via a custom landing page. The same editor is available from both the frontend dashboard and the CLI, so the flow can be driven by an LLM-based coding agent.

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
  "preview_url": "https://api.yard.sh/v1/products/.../custom-page/preview",
  "live_url": "https://yard.sh/@alice/my-slug",
  "errors": []
}
```

Diff is SHA-256 content-addressed against the server's existing hashes, so repeated pushes with no changes upload nothing.

## Configuration

| Item | Value |
|---|---|
| Config file | `~/.yard/config.json` |
| File permissions | `0600` (owner read/write only) |
| Config directory | `~/.yard/` |
| Contents | `session_token`, `user` (id, github_username, email), `api_url` |
| Auth header | `Authorization: Session {token}` |

## Key URLs

| URL | Purpose |
|---|---|
| `https://yard.sh` | Yard website |
| `https://api.yard.sh` | API base URL |
| `https://install.yard.sh` | CLI binary downloads and version checks |
| `https://github.com/apps/yard-app-official/installations/new` | Install the Yard GitHub App |

## Key Features

- **Pricing tiers** — Up to 5 tiers per product (Pro), with single, fixed-pack, or per-seat licensing
- **Volume discounts** — Percentage discounts at quantity thresholds for per-seat tiers
- **Product states** — Draft, Pre-order, Early Access, Released (with state-specific discounts)
- **Free trials** — Configurable trial periods (1-365 days)
- **License keys** — Automatic generation with device activation tracking
- **Coupons** — Percentage or fixed-amount discounts (Pro sellers)
- **Gift purchases** — Buy for someone else via recipient email (Pro sellers)
- **Subscriptions** — Recurring billing with optional yearly discounts
- **Webhooks** — Get notified when sales happen
- **API keys** — Programmatic access for *integrating* Yard into your software (license validation, release metadata, subscriptions) — not for catalog management (use the CLI)

## Reference Files

| Topic | File |
|---|---|
| Detailed CLI command reference | [references/cli-commands.md](references/cli-commands.md) |
| Pricing, licensing, coupons, trials | [references/pricing-and-licensing.md](references/pricing-and-licensing.md) |
| REST API (integration endpoints for license validation, releases, subscriptions) | [references/api-reference.md](references/api-reference.md) |
| Custom landing pages — runtime data, `data-yard` / `data-action`, `window.yard` API | [references/landing-pages.md](references/landing-pages.md) |
| Publishing releases, downloading updates, API keys | [references/releases-and-updates.md](references/releases-and-updates.md) |
| Troubleshooting common issues | [references/troubleshooting.md](references/troubleshooting.md) |
