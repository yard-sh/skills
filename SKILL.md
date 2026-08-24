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
    - references/service-and-database.md
    - references/troubleshooting.md
description: >-
  Yard is the complete platform for digital commerce, compliance, distribution, and growth so you can ship faster.
  Use this skill whenever the user mentions Yard, the Yard CLI, license keys, GitHub release integration, yard login,
  yard init, yard install, yard projects, yard releases, yard keys, yard licenses, yard help, installing the yard CLI,
  pricing, trials, device activations, affiliate links, referral codes, update server, file updates, publishing a
  release, downloading updates, creating API keys, testing license-key validation, the test license key, clearing
  test device activations, coupons or discount codes, customers or buyers, transactions, sales, orders, or extending
  and shortening a buyer's free trial. Also use this skill when users are working inside a Yard codebase and need to understand
  how Yard works, its CLI commands, API, pricing model or troubleshooting common issues.
---

# Yard

Yard lets developers make their software available for sale in just a few clicks. Yard provides a project page with checkout flow. It also provides a license server, allowing sellers to integrate the Yard REST API into their app to validate a user's ownership. Sellers can also control the number of device activations allowed per account. Buyers manage their purchases through their Yard account. Sellers install the Yard GitHub App on their repo, run `yard init` from the command line, set a price, and start selling. Buyers pay via Yard Checkout page and get instant access to download current and future releases. Yard acts as the Merchant of Record, handling payments, license keys, and file hosting.

## Teams own everything a seller has

**A project belongs to a team, not to a user.** So do coupons, affiliate links, API keys, payouts, GitHub installations, and the Stripe account the money lands in. A user belongs to any number of teams and acts as exactly one at a time — the **active team** — and that is what every seller-side command and endpoint reads and writes.

Five consequences worth holding onto:

1. **Projects are addressed by the team's username**, not the seller's username: `https://yard.sh/@{username}/{slug}`, `https://{username}.yard.sh/{slug}`, and `/v1/projects/{username}/{slug}/…`. Team and user usernames share one namespace, so they can't collide — but they are not the same thing and routinely differ. Get the team's from `yard team --json` → `.active_team.username`; never assume it from the user's login.

2. **Entitlements come from the team, not from the person typing.** The server gates seller features on the merged plans of the team's _owners_. A free user in a Pro team gets Pro features on that team's projects; a Pro user acting as a free team does not. Read `yard me --json` → `.team_permissions` for anything seller-side. `.permissions` is the user's own account-level entitlement (things like `create_teams`) and is the wrong map for deciding what a project may do.

3. **The active team is server-side state**, shared with the dashboard's team switcher — not something in `~/.yard/config.json`. It can change between commands if the user switches in the browser. When it matters which team you're operating on, check `yard team` rather than remembering an earlier answer; switch with `yard team use <username>`.

4. **No team means nothing works.** Seller endpoints answer `403` with `code: "NO_TEAM"` and the message "A team is required". That is not a plan problem and upgrading does not fix it — the user must create a team at https://yard.sh/team. Signup normally does this, so if you hit it, say so plainly instead of suggesting an upgrade.

5. **The team's money belongs to its owner.** Membership is `owner` or `admin`; both can run the whole project surface, and they part company where accountability sits. **Payouts and billing are owner-only, reads included** — `/v1/payouts/…`, `/v1/billing/invoices`, the payout bank accounts, and the Payouts and Billing pages of the dashboard. An admin gets `403` with `code: "NOT_TEAM_OWNER"`; no upgrade and no permission change clears it, only the owner acting instead. Handing the team over is the owner's alone too. `yard team` prints the acting user's role, so check it before pointing someone at those pages.

API keys follow the same rule: a key is a **team credential** pinned to one team, usable by anyone on that team, and it keeps working when whoever created it leaves. `yard keys list` shows the active team's keys, not a personal set.

## API vs CLI — which to use

Yard has two surfaces. Pick by intent:

- **CLI (`yard …`)** — for **managing** a seller's Yard presence: creating/editing projects, linking repos, scaffolding and publishing landing pages, viewing projects, running discount codes, and reading who bought what. An LLM/agent working on a seller's codebase should drive all management through the CLI. This includes the seller's own reporting — `yard customers` and `yard transactions` — which an API key cannot reach.
- **REST API** — for **integrating** Yard into shipped software: validating a buyer's license at runtime, deactivating a device, fetching the latest release, reading project metadata, managing a buyer's subscription. The API does **not** replace the CLI for catalog management — an agent that wants to "create a project" runs `yard init`, not an HTTP call.

API access uses an **API key with scoped permissions**. Create one from the CLI with `yard keys create` (see [references/releases-and-updates.md](references/releases-and-updates.md)) or from the dashboard at https://yard.sh/dashboard/api-keys?action=create. See [references/api-reference.md](references/api-reference.md) for endpoint details.

For shipped end-user software, downloads authenticate with a **license key** (per-buyer) hitting `GET /v1/updates/latest`: each buyer's key is unique, so revocation, activation limits, and per-customer rate limits work for free, and there's no shared secret to embed in the binary.

See [references/releases-and-updates.md](references/releases-and-updates.md).

## Onboarding / new project setup — agent workflow

When a user asks to get onboarded to Yard, set up a new project, run `yard init`, publish their software on Yard, or any equivalent request, **do not immediately run commands**. First ask the user which mode they prefer:

1. **Guided (step by step).** The user drives; you explain each step and wait for their input. Start by asking which directory to run `yard init` in, then walk through `yard login` (if needed) and the `yard init` prompts one at a time, pausing for confirmation between commands.
2. **Autopilot (the agent handles it).** You drive the CLI on the user's behalf. In this mode:
   1. Ask the user for a **brief description of the project** (what it is, who it's for, and — if they already have one in mind — a rough price point).
   2. Based on the description, formulate a **pricing recommendation** using the options the CLI actually supports, and note which pieces require **Yard Pro** (check with `yard me --json` → `.team_permissions`):
      - **One-time purchase, single tier** — simple projects, one price, lifetime access. Works on the free plan.
      - **Subscription, single tier** — recurring billing (monthly, with optional yearly discount). Works on the free plan.
      - **Multiple pricing tiers** (e.g., Starter / Pro / Team) — the number allowed depends on the plan and is server-enforced via `max_pricing_tiers` (currently Basic: 2, Pro: 10; check `yard me --json` → `.team_permissions.max_pricing_tiers`). Good when the project has clearly differentiated feature sets.
      - **Seat-based pricing** (`fixed_pack` packs like "Team 5-Pack", or `per_seat` with min/max and volume discounts) — **requires Yard Pro** (check with `yard me --json` → `.team_permissions`). Good for B2B / team software.
      - **"Enterprise" / contact-sales** — Yard has no first-class enterprise tier; model it as a high-priced `per_seat` tier (Pro) or a separate high-end tier in a multi-tier project (Pro), and let the seller handle custom contracts off-platform (check with `yard me --json` → `.team_permissions`).
      - Also flag, where relevant, that **gift purchases, custom landing pages, license keys, device activations, and free trials are Pro-only**, and that **coupons depend on the plan** too — never state a tier from memory, read `yard me --json` → `.team_permissions` (coupons live under `.team_permissions.coupons`).
      - If license-key features are appropriate for the project (anything users install locally and where the seller needs to validate ownership at runtime), suggest enabling **license keys**, optionally **device activations** with a per-key limit, and/or a **free trial** of N days. These can be set in the same `yard init --spec` payload (Pro only — check with `yard me --json` → `.team_permissions`) or configured later via `yard projects edit`.
      - Recommend a **launch stage**. Every new project is created in `draft` (not visible to buyers). After setup, the seller advances the stage from the Yard dashboard — stage transitions are **forward-only** (`draft` → `early_access` → `published`) and `published` is final. Two reasonable launch paths:
        - **Straight to `published`** — for finished projects with no soft-launch period. Skip `early_access` entirely.
        - **`early_access` first, then `published` later** — for projects the seller wants to ship but signal as still being polished. Optionally pair with `early_access_discount_percent` (1–100) so early adopters get a launch discount that disappears when the project moves to `published`. The discount field is set in the dashboard; `yard init`/`yard projects edit` don't surface it today.
   3. Present the recommendation as a short plan: title, pricing model, tier(s), seat type, price(s), any Pro requirements (check with `yard me --json` → `.team_permissions` before suggesting Pro-only items), and the recommended launch stage (and any early-access discount). **If the project is locally-installed software** (desktop app, CLI tool, native binary), the plan must also include (a) running `yard releases publish` for each shipped version and (b) wiring `GET /v1/updates/latest` into the app's update path — otherwise the buy page sells nothing and the installed app has no update channel. See ["Desktop / CLI app integration scope"](#desktop--cli-app-integration-scope) below and [references/releases-and-updates.md](references/releases-and-updates.md). **If the project is a web app / SaaS** (the buyer uses it in the browser), verify the `service` permission in `yard me --json` → `.team_permissions` **now, at planning time**: hosted services are Pro-only and failing later at `yard push` wastes the whole build; the plan must follow ["Hosted service scope"](#hosted-service-scope-service--database) below. Then ask the user to **accept**, **edit**, or **switch to guided mode**.
   4. On accept: run `yard projects --json` first to see what already exists (avoids accidentally creating a duplicate after a failed attempt) — each entry's `.slug` is what the rest of the CLI takes as `<slug-or-id>` or `--project`; see [references/cli-commands.md#yard-projects](references/cli-commands.md#yard-projects) ("Discovering a project's slug") for the common `jq` recipes. Then run `yard init --spec - --json` with the accepted plan encoded as JSON on stdin. **Do not** pipe answers to the interactive wizard — the CLI ships a non-interactive spec mode specifically for agents. See ["Autopilot: non-interactive `yard init`"](#autopilot-non-interactive-yard-init) below.
   5. On edit: adjust the plan and re-confirm before running anything.

Keep the CLI as the single source of truth for project creation — **never** try to create a project via the REST API (see "API vs CLI" above).

### Autopilot: non-interactive `yard init`

`yard init` supports three modes. Agents should always use one of the first two:

| Mode            | Invocation                           | When to use                                                                                                                       |
| --------------- | ------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| **Spec**        | `yard init --spec <file\|->`         | Creating a new project. Accepts the full pricing shape as JSON.                                                                   |
| **Link**        | `yard init --project <slug-or-uuid>` | Linking the current directory to an existing project.                                                                             |
| **Interactive** | `yard init`                          | Humans only. Trying to drive this from an agent via stdin is a dead end — the prompt order is load-bearing and changes over time. |

Global non-interactive flags:

- `--json` — emit a single JSON object on stdout; logs (including HTTP request lines and progress messages) go to stderr. Safe to pipe into `jq`.
- `--page` / `--no-page` — explicitly opt in or out of landing-page scaffolding without prompting. `--json` defaults to no page unless `--page` is set.
- `--link-repo` / `--no-link-repo` — force GitHub repo linking on or off in spec mode. The default tries to link if (a) the cwd is a git repo with a GitHub remote and (b) the Yard GitHub App is already installed. If the App isn't installed, linking is silently skipped — the project is still created.

If the user isn't authenticated, all non-interactive modes fail with `not logged in. Run 'yard login' first`. Ask the user to run `yard login` in their terminal, then retry.

#### Spec schema

The spec matches `CreateProjectRequest` exactly. Only `title` and `tiers` are strictly required; `pricing_model` defaults to `one_time`.

```jsonc
{
  "title": "My Project", // required, 1–60 chars, must contain a letter/digit
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
      "free_trial_enabled": false, // needs free_trials permission when true — per-tier flag, not project-level
      "free_trial_days": null, // 1..365; required when free_trial_enabled is true
      "trial_requires_card": true, // per-tier; when true, subscription-tier trials collect a card via checkout (omitted = true)
      "gift_enabled": false, // per-tier; whether this tier can be bought as a gift (one-time tiers only)
    },
  ],
  // Optional project-level seller settings.
  // license keys + activations depend on the plan (server-enforced); the CLI sends the
  // request either way and the server returns upgrade_required if not included.
  // Applied via a follow-up PUT /v1/projects/{id} after creation.
  "license_key_enabled": false, // needs license_keys permission when true
  "activations_enabled": false, // needs device_activations permission when true; requires license_key_enabled=true
  "max_activations": null, // 1..10000; only meaningful when activations_enabled
}
```

> **Heads up — per-tier trials.** `free_trial_enabled` / `free_trial_days` live on each tier, not on the project. A project "offers a trial" when at least one of its tiers has them set. To enable a trial on an existing project later, use `yard projects tiers edit <slug> <tier-id-or-name> --spec -` (see [references/cli-commands.md](references/cli-commands.md#yard-projects-tiers)). Putting `free_trial_enabled` at the project level in a spec will be rejected with `unknown field`.

#### Typical agent flow

```sh
# 1. Discover what exists (safe to run on every autopilot turn).
yard projects --json

# 2. If the user asked for a new project, create it from a spec.
yard init --spec - --json <<'EOF'
{
  "title": "Simple Note",
  "pricing_model": "one_time",
  "tiers": [{ "name": "Base", "price_cents": 1900, "seat_type": "single", "is_default": true }]
}
EOF

# 3. If the user already has the project, just link this directory to it.
yard init --project simple-note --json
```

`yard init --json` output shape:

```json
{
  "project": {
    "id": "...",
    "slug": "simple-note",
    "title": "Simple Note",
    "buy_url": "https://alice.yard.sh/simple-note",
    "profile_url": "https://alice.yard.sh/",
    "created": true
  },
  "settings_file": "/abs/path/.yard/settings.json",
  "github_repo_linked": false,
  "landing_page": null
}
```

#### Troubleshooting

- **`yard init` hangs silently.** You're in the interactive wizard. Interrupt, then retry with `--spec -` (for a new project) or `--project <slug>` (for an existing one).
- **`403 not logged in`.** Ask the user to run `yard login` in their terminal — you can't drive the OAuth browser flow.
- **Plan-gated 403s.** The **team's** plan doesn't include the feature you used (an extra tier, seat-based pricing, license keys, device activations, a free trial, custom pages, coupons, …). Project endpoints tag these `upgrade_required`; permission-gated endpoints like `yard coupons` answer a plain `FORBIDDEN` 403 — either way it's a plan problem, not a bad request. The CLI doesn't gate this client-side; the server decides. Either send a spec the plan supports, or ask the user to upgrade at https://yard.sh/pricing. To see what the plan includes, read `yard me --json` → `.team_permissions` (the team's map — `.permissions` is the user's own and does not gate this).
- **`NO_TEAM` 403s.** A different failure that looks the same: `code: "NO_TEAM"`, message "A team is required". The caller belongs to no team, so there is nothing to own the project. Upgrading does **not** fix it — send them to https://yard.sh/team, then `yard team` to confirm. Never render this as an upsell.
- **Duplicate project after a failed attempt.** Run `yard projects --json` first — if the project already exists, link it with `yard init --project <slug>` instead of re-creating.
- **Need to change settings on an existing project.** Use `yard projects edit <slug> --spec -` with an `UpdateProjectRequest` JSON body for project-level fields (`license_key_enabled`, `activations_enabled`, `max_activations`). For tier mutations — including **enabling a free trial on a specific tier**, its `trial_requires_card` / `gift_enabled` flags, changing a tier's price, or removing a tier — use `yard projects tiers add | edit | rm` (see the command table). The legacy project-level `free_trial_enabled` / `free_trial_days` / `trial_requires_card` / `gift_enabled` fields no longer exist; they were all moved per-tier.

### Desktop / CLI app integration scope

If the project is **locally-installed software** — a desktop app, CLI tool, native binary, anything the buyer downloads and runs on their own machine — `yard init` alone is **not** a complete sales surface. The buy page has nothing to download until a release is published, and the installed app has no built-in update path. Whenever you detect this project type during the autopilot flow, the plan you present in step 3 must cover both halves:

1. **Publish releases.** After `yard init`, the seller publishes each shipped version with `yard releases publish` (spec mode is the agent-friendly form). This is what populates the buy page's download. See [references/releases-and-updates.md](references/releases-and-updates.md) — _Publishing a release with the CLI_.
2. **Wire the update endpoint.** The installed app needs an auto-updater that pulls the latest release from Yard, authenticating with the buyer's **license key** → `GET https://api.yard.sh/v1/updates/latest?license_key=<key>`. One key per buyer (revocable per customer); requires the project to issue license keys.

   Tell the user this needs to be wired into their app's auto-updater. See [references/releases-and-updates.md](references/releases-and-updates.md) — _Downloading releases_.

Scope is deliberately narrow: this section covers **publish + updates only**. License validation (refusing to run for non-buyers), device activations, and free trials are separate Pro-only concerns and should not be folded into this step (check with `yard me --json` → `.team_permissions` before suggesting any of those).

For content projects or anything static the buyer does not install locally, skip this section — the default `yard init` + landing-page flow is sufficient. For **anything Yard runs itself — a SaaS, a web app, an API or hosted backend** — see the next section.

### Hosted service scope (service + database)

If the project runs on Yard — the buyer uses it in the browser, or an installed project calls its API, rather than running the code themselves — Yard hosts the backend, database, buyer sign-in, and any static frontend (a bundle with no frontend at all is valid). Requires the `service` permission (Pro; check `yard me --json` → `.team_permissions`). Whenever you detect this project type, the plan you present must cover:

1. **Scaffold and build.** `yard service init` writes a zero-dependency working bundle (plain `_worker.js` fetch handler, static frontend, first migration) and records the service's deploy settings in the `service` block of `.yard/settings.json`. Build the user's actual service inside that contract. **No ports, no `listen()`, no Express**: the backend is a fetch handler; route by path; use relative URLs in the frontend. Full contract: [references/service-and-database.md](references/service-and-database.md).
2. **Never build auth.** The Yard edge signs buyers in and injects trusted `X-Yard-User-Id` / `X-Yard-Entitlement` headers; `service.access: "customers"` in `.yard/settings.json` is a complete paywall with zero service code. Building your own login/OAuth/session layer is a bug.
3. **Push → test → publish.** `yard push` uploads the bundle into a **draft release**. Nothing serves a draft, so to test it before customers see it, create an environment of your own (`yard env create preview`) and publish there first (`yard releases publish <tag> --env preview`, then `yard service open --env preview`, visible to your team only by default; `yard env visibility preview public` makes the URL shareable with testers). Go live with `yard releases publish <tag>`, which defaults to production. Data and secrets never move between environments, so set production secrets explicitly.
4. **Draft projects serve the service to the owning team only.** You can deploy, promote, and fully verify `/service/` while the project is still `draft`: any member of the team signs in and gets through; everyone else sees an explanatory 403. Never advance the project stage just to test (stage changes are one-way).
5. **Pricing still applies.** The service is gated by normal Yard pricing (tiers, trials, subscriptions), configured as for any project; the project page remains the sales surface and the service lives under `/service/`. Every member of the owning team passes the paywall with `X-Yard-Entitlement: owner`.

A hosted-service release carries the service bundle and landing page, not downloadable files. There is nothing to wire into `GET /v1/updates/latest` here; publishing/promoting the release **is** the deploy.

### Testing license-key validation

Every project with `license_key_enabled: true` has a sandbox **test license key** — a real license key value the seller can pass to `POST /v1/licenses/validate` and have the API treat it like a paid customer's key. Test activations live in a separate `test_activations` table, so they never collide with real customers.

**Auth:** `POST /v1/licenses/validate` is **not unauthenticated**. It requires an API key with the `licenses:validate` scope (`Authorization: Bearer yard_<key>`). The test license key goes in the **request body**, not the header — it's the data being validated, not the credential.

Three CLI commands cover the loop:

```sh
# Print the test key for the current project (or pass --project <slug>).
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

All three `yard licenses` commands accept `--project <slug>` and fall back to the slug in `.yard/settings.json`. `--json` is supported on each for scripted use.

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
3. **Best-effort** git context: if inside a git repo with a GitHub remote, prompts to install the Yard GitHub App and verifies the repo. Any step failing here is non-fatal — the project will simply be created without a linked repo.
4. Lets you select an existing project or create a new one (prompts for title + price on create)
5. Writes `.yard/settings.json` in the current directory
6. Offers to scaffold a custom landing page. If accepted, pulls or scaffolds starter files into `.yard/landing-page/` and uploads them into your draft release (going live requires publishing the draft with `yard releases publish`). If the plan doesn't include custom pages, the server returns `upgrade_required`; the CLI shows a friendly upgrade link and keeps the draft.

**No git repo required.** `yard init` works inside any directory — it will just create the project without a GitHub link. Only GitHub repositories are supported for linking.

## CLI Commands

| Command                                                                       | Description                                                                                                                                                                                                                                                             |
| ----------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `yard login`                                                                  | Authenticate via GitHub OAuth                                                                                                                                                                                                                                           |
| `yard logout`                                                                 | Clear local credentials (`~/.yard/config.json`)                                                                                                                                                                                                                         |
| `yard me [--json]`                                                            | Show the current user (id, username, GitHub, email), plan, and the `permissions` map. Read `--json` → `.team_permissions` to see what the active team can do (`.permissions` is the user's own, and is not what seller features are gated on); feature limits are server-enforced, so you can also just attempt an action and handle `upgrade_required`.          |
| `yard team [--json]` / `yard team use <username>`                              | Show or switch the team the CLI acts as. Projects, coupons and API keys belong to a team, so this decides what every other command reads and writes. The active team is stored on the account (shared with the dashboard), not in the local config.                       |
| `yard init`                                                                   | Set up a Yard project in the current directory — create or select a project, scaffold `.yard/`, optional landing-page setup. Supports `--spec <file\|->`, `--project <slug>`, `--json`, `--page`/`--no-page`, `--link-repo`/`--no-link-repo` for non-interactive use.   |
| `yard projects [--json]`                                                      | List your published projects with stats                                                                                                                                                                                                                                 |
| `yard projects show <slug-or-id> [--json]`                                    | Print one project's full detail, including `tiers[]` with per-tier `free_trial_enabled` / `free_trial_days`. Use this (not `yard projects`) when you need to check whether a tier offers a trial. This command is used to retrieve any sort of metadata about a project |
| `yard projects edit [slug-or-id] [--spec <file\|->] [--json]`                 | Modify project-level seller settings (`license_key_enabled`, `activations_enabled`, `max_activations`) and optionally the full `tiers[]` array (full-replace). No client-side plan gate — the server returns `upgrade_required` if the plan doesn't include a setting.       |
| `yard projects tiers add <slug> --spec <file\|->`                             | Append one tier without rebuilding the full tier list.                                                                                                                                                                                                                  |
| `yard projects tiers edit <slug> <tier-id-or-name> --spec <file\|->`          | Apply a partial spec to one tier (e.g. enable a free trial on Base: `{"free_trial_enabled": true, "free_trial_days": 14}`).                                                                                                                                             |
| `yard projects tiers rm <slug> <tier-id-or-name> [--yes] [--promote-default]` | Remove a tier. Tiers with paid transactions are kept as historical records but marked non-default.                                                                                                                                                                      |
| `yard releases publish [tag] [flags]`                                         | Publish a draft release under a tag, with optional file assets; deploys it to `--env` (default `production` — live to customers). Supports `--spec <file\|->` and `--json` for non-interactive use. See [references/releases-and-updates.md](references/releases-and-updates.md).          |
| `yard releases promote <tag> --to <env>`                                      | Attach an already-published release to another environment (e.g. `--to production` to go live). Nothing is copied — the environment serves the release from its set.                                                                                                   |
| `yard keys list [--json]`                                                     | List the active team's API keys (name, prefix, scopes, last-used, created). Keys are team credentials, not personal ones. The full secret is never shown.                                                                                                               |
| `yard keys create [name] [flags]`                                             | Mint a new API key. Supports `--spec <file\|->` and `--json`. The full secret is shown only once at creation.                                                                                                                                                           |
| `yard licenses test-key [--project <slug>] [--json]`                          | Print the project's sandbox **test license key** — usable with `POST /v1/licenses/validate` to exercise license-validation logic without buying your own project.                                                                                                       |
| `yard licenses test-activations list [--project <slug>] [--json]`             | List active test device activations attached to the project's test license key.                                                                                                                                                                                         |
| `yard licenses test-activations clear [--project <slug>] [--yes] [--json]`    | Deactivate every test device on the project's test key (real customer activations are untouched).                                                                                                                                                                       |
| `yard coupons [--json]`                                                       | List discount codes with their discount, scope, usage, and derived status (`active`, `scheduled`, `expired`, `used up`, `inactive`).                                                                                                                                    |
| `yard coupons show <code-or-id> [--json]`                                     | One coupon plus its redemption analytics. `--json` emits `{coupon, analytics}`. Accepts the code or the UUID.                                                                                                                                                           |
| `yard coupons create <code> [flags] [--spec <file\|->] [--json]`              | Create a code. `--percent 20` or `--amount 5` (**dollars**; a spec's `discount_value` is **cents** for `fixed_amount`), `--projects <csv>`, `--max-uses`, `--expires`, `--valid-from`, `--subscription-duration once\|forever`. Flags override `--spec` field by field.  |
| `yard coupons generate --count N [flags] [--json]`                            | Bulk-generate up to 100 unique codes sharing one discount (`--prefix`, `--length`). Codes are returned **once** — capture them from the output.                                                                                                                        |
| `yard coupons update <code-or-id> [flags] [--spec <file\|->] [--json]`        | Partial update. `--activate`/`--deactivate`, and explicit clearing via `--no-expiry`, `--no-valid-from`, `--unlimited-uses` (or `null` in a spec). Discounts can't change once a coupon has been redeemed.                                                              |
| `yard coupons rm <code-or-id> [--yes]`                                        | Delete an unused coupon. Redeemed coupons can't be deleted — deactivate them instead.                                                                                                                                                                                  |
| `yard coupons transactions <code-or-id> [--json]`                             | The purchases a coupon was redeemed on.                                                                                                                                                                                                                                |
| `yard coupons validate <code> --project <slug> [--json]`                      | Dry-run a code through the checkout-time check and see what the buyer would pay. The project must be public.                                                                                                                                                            |
| `yard customers [--json]`                                                     | List the buyers who completed a purchase, with order count, spend, and activity dates. `--project <slug>` narrows both the rows and the summary to one project's buyers. Amounts are pre-formatted display strings, not cents.                                          |
| `yard customers show <cust-id> [--json]`                                      | One buyer's totals plus their orders (refunded ones included). Takes the opaque `cust_xxxxxxxx` id from the list, not an email.                                                                                                                                         |
| `yard transactions [--json]`                                                  | List sales. `--trials`, `--project <slug>`, `--start`/`--end` narrow the rows and total; the earnings summary stays team-wide.                                                                                                                                       |
| `yard transactions show <order-id> [--json]`                                  | One sale in full — tier, coupon, refund state, trial expiry. Takes the short `order_xxxxxxxx` id or the full UUID.                                                                                                                                                      |
| `yard transactions trial <order-id> --add-days N [--json]`                    | Lengthen (`7`) or shorten (`-3`) a free trial, ±365. Days are added to the **current expiry, not today**; an expired trial whose new expiry is in the future goes back to active (`reactivated: true`). The buyer is emailed. Needs `.team_permissions.sell_projects`. |
| `yard init --page`                                                             | Scaffold `.yard/landing-page/` inside a Yard project, pulling the draft release's page files (or a hello-world starter) |
| `yard status`                                                                  | Diff every local bundle (landing page + service) against your draft release — what `yard push` would change (no writes) |
| `yard ls [--release <id\|tag>]`                                                | List a release's files, grouped by bundle (defaults to your open draft) |
| `yard push [--prune] [--release <id\|tag>]`                                    | Upload every changed local file — landing page and service — into your draft release; go live with `yard releases publish <tag>`. `--release` can name a published release, which is edited in place |
| `yard pull [--release <id\|tag>]`                                              | Download a release's files into the project |
| `yard env list [--json]`                                                      | List the environments with what each serves and **why** — `(newest)`, `(deployed)` or `(pinned)` — plus its release set and whether its running service is up to date (only `production` always exists, and only it is protected)                                            |
| `yard env create <env>` / `yard env rename <env> <new-name>` / `yard env delete <env> [-y]` | Add / rename / remove a custom environment (plan-gated via `max_environments` — check `yard me --json` → `.team_permissions`). Renaming keeps its releases, files, secrets and database but changes its `/@<env>/` URL. Deleting removes its files, service, and service database immediately, and prompts unless `-y`. |
| `yard env visibility <env> <public\|private>`                                  | Set who may view the environment's URLs. `private` (custom-env default) is the owning team only — every member, `owner` and `admin` alike; `public` lets anyone with the URL view its page and service. Works on `production` — that is how a project goes private (there is no project-level visibility setting any more). Stage still trumps: drafts serve nothing publicly. |
| `yard env deploy <env> <release>`                                             | **Serve this release here now**, attaching it first if needed. Works with any release, however old — this is the rollback and the ship command. Not frozen: the next release attached on top takes over.                                                                |
| `yard env pin <env> [release]` / `yard env unpin <env>`                       | Freeze what the environment serves so later attaches join its set without taking over (no release named = pin what it serves now); `unpin` hands the choice back to the newest member.                                                                                  |
| `yard env attach <env> <release> [--no-serve]` / `yard env detach <env> <release>` | Add a release to the environment's set — as the newest member it starts serving, so attaching is the deploy moment; `--no-serve` stages it instead. `detach` stops serving it, and is refused if nothing would be left to serve.                                    |
| `yard env promote <from> <to>`                                                | Attach the release `<from>` currently serves to `<to>`; promoting to `production` takes it live. Nothing is copied; data and secrets never promote. Prefer `env deploy` when you can name the release.                                                                  |
| `yard service init [--service-dir NAME]`                                          | Scaffold a zero-dependency service bundle (backend + frontend + migration); local-dev files land at the top of the working directory and the bundle dir + deploy settings are recorded in the `service` block of `.yard/settings.json`                                                       |
| `yard service open [--env SLUG]`                                              | Print and open the environment's service URL                                                                                                                                                                                                                            |
| `yard service check`                                                 | Validate the bundle offline (limits, extensions, `_worker.js`) + lint root-absolute URLs                                                                                                                                                                                |
| `yard service secrets set/list/rm [--env SLUG]`                               | Per-environment `env.<NAME>` bindings; write-only; apply on the next deploy                                                                                                                                                                                             |
| `yard service db query [sql] [--file PATH] [--env SLUG]`                      | Run SQL against the environment's service database (`-` for stdin; `_yard_migrations` records applied migrations)                                                                                                                                                      |
| `yard service logs [--env SLUG] [--limit N] [--since 2h]`                     | Recent service console output + exceptions (empty list for a fresh service, not an error)                                                                                                                                                                               |
| `yard version`                                                                | Show version, commit hash, platform, Go version                                                                                                                                                                                                                         |
| `yard update`                                                                 | Download and install the latest CLI version                                                                                                                                                                                                                             |
| `yard update --check`                                                         | Check for updates without installing                                                                                                                                                                                                                                    |
| `yard uninstall`                                                              | Remove CLI binary and config directory (`--force` to skip confirmation)                                                                                                                                                                                                 |

See [references/cli-commands.md](references/cli-commands.md) for detailed command documentation.

## How It Works

1. **Seller** installs the Yard GitHub App on their repository
2. **Seller** runs `yard init` to create a project with pricing (and optionally a custom landing page)
3. When the seller publishes a **GitHub release**, Yard automatically captures it via webhook — the release assets always, plus the service bundle, landing page, and pricing tiers when the repo has a `.yard/settings.json` at the tag (see [references/releases-and-updates.md](references/releases-and-updates.md) — _Syncing releases from GitHub_)
4. **Buyers** visit the project page, pay via Stripe, and get instant download access
5. Seller earnings are tracked and paid out by admin

## Custom Landing Pages

Every Yard project has a public landing page. Pro sellers can replace the default layout with their own HTML/CSS/JS via a custom landing page (check with `yard me --json` → `.team_permissions`). The same editor is available from both the frontend dashboard and the CLI, so the flow can be driven by an LLM-based coding agent.

For everything an agent needs to **author** the page itself — how to read project data at runtime (`window.yard.project`), the `data-yard` / `data-action` attribute conventions, the `window.yard.checkout(...)` / `trial()` helpers, and the full `PublicProject` field reference — see [references/landing-pages.md](references/landing-pages.md). The remainder of this section covers the **management** flow (scaffolding, pushing, publishing).

**Limits** (enforced server-side; also validated client-side before upload):

- 20 files max per bundle
- 1 MB max per file
- 5 MB max total bundle size
- Allowed extensions: `.html .css .js .json .svg .png .jpg .jpeg .webp .gif .woff2`
- Paths: letters/digits/`._-` only, at most one subdirectory level
- `index.html` is required to publish

### Project Layout

`.yard/` at the top of a working directory is the hub of everything Yard in a project. `yard init` creates it; `yard init --page` adds the landing-page directory:

```
<project>/
├── .yard/
│   ├── settings.json         # all project settings (below)
│   └── landing-page/         # landing page (default location, configurable)
│       ├── index.html
│       └── ...
└── service/                  # service bundle, wherever service.dir points
```

`.yard/settings.json` schema (v4):

```json
{
  "version": 4,
  "project_slug": "my-project",
  "ignore_files": ["*.bak", "drafts/**"],
  "service": { "dir": "service", "access": "authenticated", "database": true },
  "landing_page": { "dir": ".yard/landing-page" },
  "pricing": { "tiers": [{ "name": "Base", "price_cents": 1900, "is_default": true, "pricing_model": "one_time" }] },
  "downloads": { "buttons": [{ "condition": "ends_with", "value": ".dmg", "label": "Download for Mac" }] }
}
```

- `project_slug` — which project this working directory belongs to.
- `ignore_files` — shell-style globs relative to the landing-page directory; `**` matches any depth. Dotfiles are always ignored.
- `service.dir` — service bundle directory relative to the working directory (recorded by `yard service init`; default `service`).
- `service.access` — who can reach the deployed service: `public` | `authenticated` | `customers` (default `public`).
- `service.database` — `true` provisions a per-environment SQLite database bound as `env.DB`.
- `landing_page.dir` — landing-page directory relative to the working directory (default `.yard/landing-page`).
- `pricing.tiers` — optional; when present, `yard push` and GitHub release sync replace the release's pricing tiers to match the array exactly (tiers missing from the file are removed). Absent = pricing is managed from the dashboard as usual. Full shape and rules: [references/releases-and-updates.md](references/releases-and-updates.md) — _Syncing releases from GitHub_.
- `downloads.buttons` — optional; when present, `yard push` and GitHub release sync replace the release's download buttons to match the array exactly. Each rule matches release files by `condition` (`contains` | `starts_with` | `ends_with` | `has_extension`) and `value` (1-255 chars) and labels the button (`label`, 1-50 chars); max 10 rules. Absent = download buttons are managed from the dashboard as usual.

All blocks are optional. `yard push` uploads `settings.json` itself as the release's `config` artifact — that is how deploys read the service settings, so a service-settings change is a settings edit plus a push. A leftover `yard.json` (the retired bundle manifest) in the bundle is skipped. Settings files below `"version": 3` are rejected, not upgraded: they name the entity `product_slug`, which no longer binds to anything — rename the key to `project_slug` and set `"version": 4`, or re-run `yard init`.

### Typical Flow

1. `cd <project>` and `yard init --page` — scaffolds `.yard/landing-page/`, pulling your draft release's page files (or a hello-world starter)
2. Edit files in `.yard/landing-page/` (by hand, or prompt an agent to do it)
3. `yard status` — preview the diff without writing anything
4. `yard push` — upload changed files into your draft release; prints a `Preview:` URL
5. `yard releases publish <tag>` — publish the draft and go live (or publish to an environment of your own first, then `yard releases promote <tag> --to production` when ready)

### Driving It From an Agent

All project sync commands (`push`, `pull`, `status`, `ls`) accept:

- `--project <slug-or-uuid>` — override the project in `.yard/settings.json`
- `--dir <path>` — directory containing `.yard/` (defaults to walking up from cwd)
- `--release <id|tag>` — target a specific release, by tag or UUID (defaults to your open draft, else a new one seeded from your newest published release; required when several drafts are open). Published releases are valid targets: `push` edits one in place, which is live on save if an environment is serving it.
- `--json` — emit a single machine-readable JSON object; logs go to stderr
- `--yes` — skip confirmation prompts (`push --prune`, and pushing into a release attached to an environment)

Exit codes: `0` = success, `1` = fatal (auth/validation/network), `2` = partial success (`push` only).

Example `push --json` output (one object per bundle the project has — `page`, `service`, `config`):

```json
{
  "project": "my-slug",
  "release": "9f3e1c2a-…",
  "version": "",
  "page": {
    "dir": "/abs/path/.yard/landing-page",
    "uploaded": ["index.html", "styles.css"],
    "skipped": [],
    "deleted": [],
    "remote_only": []
  },
  "config": {
    "dir": "/abs/path/.yard",
    "uploaded": ["settings.json"],
    "skipped": [],
    "deleted": [],
    "remote_only": []
  },
  "preview_url": "https://yard.sh/dashboard/projects/my-slug/landing-page",
  "live_url": null,
  "errors": []
}
```

`live_url` is only non-null once production serves a release.

Diff is SHA-256 content-addressed against the server's existing hashes, so repeated pushes with no changes upload nothing.

## Configuration

| Item             | Value                                                           |
| ---------------- | --------------------------------------------------------------- |
| Config file      | `~/.yard/config.json`                                           |
| File permissions | `0600` (owner read/write only)                                  |
| Config directory | `~/.yard/`                                                      |
| Contents         | `session_token`, `user` (id, github_username, email), `api_url` |
| Not stored       | The active team. It lives on the account (see `yard team`), so the CLI and dashboard always agree and it survives a re-login. |
| Auth header      | `Authorization: Session {token}`                                |

## Key URLs

| URL                                                           | Purpose                                 |
| ------------------------------------------------------------- | --------------------------------------- |
| `https://yard.sh`                                             | Yard website                            |
| `https://api.yard.sh`                                         | API base URL                            |
| `https://cli.yard.sh`                                         | CLI binary downloads and version checks |
| `https://github.com/apps/yard-app-official/installations/new` | Install the Yard GitHub App             |

## Key Features

- **Pricing tiers** — Multiple tiers per project (how many depends on the plan; Basic: 2, Pro: 10), with single, fixed-pack, or per-seat licensing
- **Volume discounts** — Percentage discounts at quantity thresholds for per-seat tiers
- **Project stages** — Draft → Early Access → Published, forward-only (Early Access supports a launch discount; Published is final)
- **Free trials** — Configurable trial periods (1-365 days)
- **License keys** — Automatic generation with device activation tracking
- **Coupons** — Percentage or fixed-amount discounts, single or bulk-generated, managed with `yard coupons` (plan-gated — check `yard me --json` → `.team_permissions.coupons`)
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
| Service & database - runtime contract, `yard service` workflow, auth headers, database | [references/service-and-database.md](references/service-and-database.md)   |
| Publishing releases, downloading updates, API keys                                  | [references/releases-and-updates.md](references/releases-and-updates.md)   |
| Troubleshooting common issues                                                       | [references/troubleshooting.md](references/troubleshooting.md)             |
