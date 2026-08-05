# Yard CLI Command Reference

## Global Behavior

- The CLI binary name is `yard` (production) or `yard-staging` (staging builds)
- API URL defaults to `https://api.yard.sh` but can be overridden at build time
- Config is stored at `~/.yard/config.json` (or `~/.yard-staging/` for staging)
- All authenticated commands send an `Authorization: Client {token}` header (the token `yard login` saved)

---

## yard login

Authenticate with your GitHub account.

**Flow:**

1. Binds a local HTTP server on **port 9876** (fails immediately if port is in use)
2. Builds the login URL:
   - Standard: `{apiURL}/v1/auth/login?cli=true`
   - In Coder workspaces: uses `VSCODE_PROXY_URI` to construct workspace-aware proxy URLs for both the API and callback
3. Opens the default browser (macOS: `open`, Linux: `xdg-open`, Windows: `rundll32`)
4. User authorizes via GitHub OAuth in the browser
5. Backend creates a session token and redirects the browser to `http://localhost:9876/callback?token={token}`
6. CLI receives the token, calls `GET /v1/me` to fetch user info
7. Saves to `~/.yard/config.json`:
   ```json
   {
     "session_token": "{64-char-hex-token}",
     "user": {
       "id": "{uuid}",
       "github_username": "username",
       "username": "optional-username",
       "email": "user@example.com"
     },
     "api_url": "https://api.yard.sh"
   }
   ```
8. File permissions are set to `0600`

**Timeout:** 5 minutes. If the user does not complete the OAuth flow in time, login fails.

**Coder workspace support:** When `VSCODE_PROXY_URI` is set, the CLI automatically constructs proxy URLs so the OAuth callback works through the Coder workspace proxy.

---

## yard logout

Clear stored credentials.

- Deletes `~/.yard/config.json`
- If already logged out (file doesn't exist), prints "Already logged out" and succeeds
- Does not invalidate the server-side session

---

## yard me

Print the currently logged-in user, their plan, and the permissions their plan grants. Useful for agents deciding which features to suggest before proposing them.

**Usage:** `yard me [--json]`

**Auth:** required. If not logged in, exits with `not logged in. Run 'yard login' first`.

**Human output:**

```
Username:     alice
GitHub:       alice
Email:        alice@example.com
Subscription: Pro
```

The `GitHub` and `Email` lines are omitted when not set.

**JSON output (`--json`):**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "username": "alice",
  "github_username": "alice",
  "email": "alice@example.com",
  "plan": "Pro",
  "permissions": {
    "sell_products": { "granted": true, "value_type": "boolean" },
    "license_keys": { "granted": true, "value_type": "boolean" },
    "coupons": { "granted": true, "value_type": "boolean" },
    "max_pricing_tiers": {
      "granted": true,
      "limit": 10,
      "value_type": "limit"
    },
    "max_products": {
      "granted": true,
      "unlimited": true,
      "value_type": "limit"
    }
  }
}
```

`username` is the best available display name (GitHub username → username → email → id), matching `User.DisplayName()`. `plan` is a human label (`Free` / `Basic` / `Pro`) for display only. **`permissions` is the source of truth** for what the account can do: each entry is a merged grant — booleans carry `granted`; limits also carry `limit` (a number) or `unlimited: true`. Feature gating is entirely permission-based, so a feature isn't tied to a plan _name_ — a custom role could grant any subset. (The old `is_pro` field was removed; use `permissions` or `plan`.)

---

## yard init

Set up a Yard project in the current directory. Interactive flow that links the folder to a Yard product (new or existing) and optionally scaffolds a custom landing page.

**Prerequisites:** none. A Git repo with a GitHub remote enables the "link the repo to a new product" path, but is not required — outside of a git repo, `yard init` still creates products, they just won't be linked to a GitHub repository.

**Step-by-step flow:**

1. **Update check** — Fetches latest version from `{UpdateURL}/current_version.txt`. If a newer version exists, prompts the user to update. Update is required to continue — if the user declines, `init` aborts.

2. **Login check** — Loads `~/.yard/config.json` and verifies the session is still valid via `GET /v1/me`. If not logged in or session expired, runs the login flow automatically.

3. **Best-effort git context** — All three steps here are non-fatal; on any failure `init` prints a one-line stderr notice and proceeds to create a product without a linked repo.
   - Runs `git rev-parse --show-toplevel` + `git config --get remote.origin.url` and parses owner/repo. SSH and HTTPS GitHub URLs supported.
   - `GET /v1/github/installations` to check the Yard GitHub App. If absent, opens the install page and polls every 2 seconds (5-min timeout).
   - `GET /v1/repos/verify?repo=owner/repo` to confirm access. If the repo is already listed as another product, an informational note is printed but selection is still up to the user.

4. **Product selection prompt** — Calls `GET /v1/products`. If the user has zero products, skips straight to the create-new branch. Otherwise prints a numbered list and asks `(n) new product, (1..N) select existing`. Invalid input re-prompts.

5. **Create-new branch** (only when selected):
   - Title prompt (max 60 chars). Defaults to repo name if a git repo was detected.
   - Price prompt in dollars. Minimum $3.00, or $0 for free.
   - `POST /v1/products`. If git context was fully collected and the repo isn't already listed, the request includes `github_repo_id` + `github_repo_name`; otherwise those fields are omitted (the backend accepts products with no linked repo).
   - The wizard never blocks a choice by plan. If the assembled product uses something the account's plan doesn't include (e.g. multiple tiers or seat-based pricing), the create call returns `upgrade_required`; the CLI shows the generic upgrade link and then prompts _"Once you've upgraded, press Enter to retry"_ — pressing Enter resubmits the **same** request, so the user upgrades in the browser and retries without re-answering the wizard.

6. **Scaffold `.yard/`** — Writes `./.yard/settings.json` via `EnsureYardProject`. Existing settings are preserved.

7. **Optional landing-page setup** — Prompts `Set up a custom landing page for <slug>? [y/N]`. If yes:
   - Creates `./.yard/landing-page/`.
   - Runs the same source-pick logic as `yard page init` (the environment → production → starter) and pulls or scaffolds accordingly.
   - Calls `POST /v1/products/{id}/landing-page/publish`. If the plan doesn't include custom pages, the server returns `upgrade_required`; the CLI prints the `https://yard.sh/pricing` message to stderr, keeps the uploaded files, and exits 0. On other errors, fails.

8. **Optional product-settings prompts** — After landing-page setup, the wizard always asks (regardless of plan — the server decides, no client-side gate):
   - "Enable license keys? [y/N]" — toggles `license_key_enabled`.
   - If license keys are on: "Enable device activations? [y/N]" → on yes, "Device activation limit (1-10000) [3]:".

   Changes are applied via `PUT /v1/products/{id}`. If the account's plan doesn't include a setting, the server returns `upgrade_required` and the CLI shows the generic upgrade link — the rest of `init` still succeeds. Spec mode (`yard init --spec`) accepts the same fields directly in the JSON payload (see SKILL.md schema). (Free trials, `trial_requires_card`, and `gift_enabled` are configured **per tier**, not here — see `yard products tiers`.)

9. **Success output** — Prints the product display name, slug, and the buy / profile URLs (new products only). If a landing page was set up, also prints the preview URL — and the live URL when publish succeeded.

**JSON output:** the current implementation is interactive-only; there's no `--json` variant yet.

---

## yard products

List all your published products. With no subcommand, shows the same table as before; the `edit` subcommand modifies seller settings.

**Output format:**

```
NAME                                PRICE      RELEASES   SALES
----------------------------------------------------------------------
✓ my-awesome-tool                   $9.99      3          12
✗ private-beta                      $29.00     1          0

Total: 2 product(s)
```

- `✓` = public visibility, `✗` = not public
- Names truncated to 32 characters with `...` suffix
- Requires login; prompts to run `yard login` if not authenticated
- `--json` emits the underlying `ProductListItem` array (including `license_key_enabled`, `activations_enabled`, `max_activations`). **Tiers are not included** — free trials, `trial_requires_card`, and `gift_enabled` are configured per tier, so use `yard products show <slug> --json` (below) to inspect `tiers[].free_trial_enabled` / `tiers[].trial_requires_card` / `tiers[].gift_enabled`.

**Discovering a product's slug.** The default table shows the _display name_ (title → repo name → slug fallback), not the slug. Every other CLI command that takes `<slug-or-id>` or `--product <slug>` needs the raw slug, which the JSON output carries on `.slug`. Common ways to find it:

```sh
# All slugs the current account owns
yard products --json | jq -r '.[].slug'

# Slug + title, tab-separated (handy when titles aren't unique)
yard products --json | jq -r '.[] | "\(.slug)\t\(.title)"'

# Find a slug by partial title match (case-insensitive)
yard products --json | jq -r '.[] | select(.title | ascii_downcase | contains("simple")) | .slug'

# Slug for the project you're sitting in (set by `yard init` / `yard page init`)
jq -r .product_slug .yard/settings.json
```

If you only have the UUID, `yard products show <uuid> --json | jq -r .slug` resolves it. Slugs and UUIDs are interchangeable everywhere `<slug-or-id>` is accepted.

### yard products show <slug-or-id>

Prints one product's full detail — the same shape the seller dashboard fetches via `GET /v1/products/{id}`, including `tiers[]` with `pricing_model`, `seat_type`, `features`, `volume_brackets`, and per-tier `free_trial_enabled` / `free_trial_days`.

**Usage:**

```sh
yard products show my-awesome-tool             # human-readable summary
yard products show my-awesome-tool --json      # full ProductDetailResponse on stdout
```

**Typical agent flow** — gating a landing-page trial CTA on whether _any_ tier offers a trial:

```sh
yard products show my-awesome-tool --json | jq '.tiers[] | select(.free_trial_enabled) | {id, name, free_trial_days}'
```

If the output is empty, no tier offers a trial — the trial button on the landing page should stay hidden.

**Resolution:** same as `yard products edit` — accepts either a slug or a UUID; resolves slugs through the seller's product list.

### yard products edit [slug-or-id]

Modify seller settings on an existing product: license keys and device activations (and the per-key limit). Whether the account's plan includes these is enforced by the server, not the CLI.

**Resolution:**

- Pass a slug or UUID as the first argument to target a specific product.
- Without an argument, auto-selects when the user has only one product, prompts to pick from a numbered list otherwise.
- In `--spec` mode with multiple products and no argument, errors out with the list of slugs.

**Interactive flow:**

1. Prints `Editing settings for <title> (<slug>)`.
2. Same prompt sequence as `yard init`'s settings step (no plan gate — every prompt is shown). Each prompt's default reflects the _current_ value, so pressing Enter is always a no-op.
3. Calls `PUT /v1/products/{id}` with only the fields that changed.

**When the plan doesn't include a setting:** the CLI still attempts the update; the server returns `upgrade_required` and the CLI prints the server's reason plus the `https://yard.sh/pricing` link. Interactive mode exits 0 (nothing changed); `--spec` mode exits non-zero.

**Spec mode:**

- `--spec <file|->` — read JSON from a file or stdin. The JSON shape is `UpdateProductRequest`:
  ```json
  {
    "license_key_enabled": true,
    "activations_enabled": true,
    "max_activations": 5
  }
  ```
  Unknown fields are rejected. Missing fields are left untouched (the request is sparse). **Free trials, `trial_requires_card`, and `gift_enabled` are per-tier** — they're not product-level settings; use `yard products tiers edit` (below) instead.
- A `tiers` array MAY be included to replace the full tier list in one shot — every existing tier with a matching `id` is updated in place, new tiers (no `id`) are inserted, and tiers omitted from the array are deleted (or marked non-default if they have transactions/active subscriptions). Always read the current tiers first (`yard products show <slug> --json | jq '.tiers'`) and mutate before sending back, otherwise you'll accidentally drop tiers.
- `--json` — emit `{ "product": {...}, "settings": {...} }` on stdout; logs go to stderr.
- The CLI pre-checks the activations-needs-license-keys rule against the _effective_ state, so a spec that flips activations on without restating `license_key_enabled` succeeds when the product already has license keys enabled.
- A 403 with `error_code: "upgrade_required"` is rendered as a clean upgrade message rather than the raw HTTP error.

**Typical agent flow:**

```sh
# Discover what exists.
yard products --json

# Apply product-level settings to a known product.
yard products edit my-awesome-tool --spec - --json <<'EOF'
{
  "license_key_enabled": true,
  "activations_enabled": true,
  "max_activations": 5
}
EOF

# Enable a 14-day card-required free trial on the Base tier (per-tier settings).
yard products tiers edit my-awesome-tool Base --spec - <<'EOF'
{ "free_trial_enabled": true, "free_trial_days": 14, "trial_requires_card": true }
EOF
```

### yard products tiers

Manage pricing tiers (add, change, remove) without rebuilding the whole tier array yourself. The backend exposes tier mutations only through `PUT /v1/products/{id}` with a full-replace `tiers[]`; these subcommands do the read-modify-write for you so you don't accidentally drop tiers.

For wholesale changes (multiple tiers at once, reorder, etc.) use `yard products edit <slug> --spec -` with the full `tiers[]` array directly.

#### yard products tiers add <product-slug> --spec <file|->

Append a new tier. Spec shape:

```jsonc
{
  "name": "Pro", // required, must contain a letter/digit
  "price_cents": 4900, // required; 0 (free) or 300..1000000
  "description": "All the things", // optional
  "seat_type": "single", // single | fixed_pack | per_seat (seat-based needs the seat_based_pricing permission; server-enforced)
  "seat_count": 5, // required when seat_type=fixed_pack
  "min_seats": 1, // per_seat only
  "max_seats": 25, // per_seat only; null = unlimited
  "features": ["a", "b"],
  "pricing_model": "one_time", // one_time | subscription
  "yearly_discount_percent": 20, // subscription only; 1..100
  "free_trial_enabled": true, // needs the free_trials permission (server-enforced)
  "free_trial_days": 14, // 1..365; required when free_trial_enabled
  "is_default": false, // when true, the current default is demoted
}
```

How many tiers a product may have depends on the account's plan (server-enforced via the `max_pricing_tiers` permission — currently Basic: 2, Pro: 10). The CLI doesn't pre-check; if you exceed the plan's limit the add is rejected with `upgrade_required`. Check the current cap with `yard me --json` → `.permissions.max_pricing_tiers`.

#### yard products tiers edit <product-slug> <tier-id-or-name> --spec <file|->

Apply a partial spec to one tier. Any field present in the spec replaces the current value; absent fields are left untouched. Match by UUID or case-insensitive name (UUID required when two tiers share a name).

```sh
# Enable a 14-day trial on the Base tier
echo '{"free_trial_enabled": true, "free_trial_days": 14}' \
  | yard products tiers edit simple-note Base --spec -

# Drop a tier's price
echo '{"price_cents": 1500}' \
  | yard products tiers edit simple-note Base --spec -
```

#### yard products tiers rm <product-slug> <tier-id-or-name> [--yes] [--promote-default]

Remove a tier. Tiers with paid transactions or active subscriptions cannot be hard-deleted server-side — they are kept but marked non-default so new buyers won't see them. Refuses to remove the only remaining tier.

- `--yes` — skip the confirmation prompt (required in scripts / non-TTY).
- `--promote-default` — when removing the current default tier, auto-promote the first surviving tier to default. Without this flag, the command refuses to leave the product without a default.

All three subcommands accept `--json` to emit the refreshed tier list on stdout.

---

## yard releases

Manage releases for a product. The CLI exposes `publish` and `promote` — list/edit/delete still happen in the dashboard.

### yard releases publish [tag]

Create a new release with optional file assets. The release is created first, then each file is uploaded one at a time so a single failed asset doesn't lose the release.

**Flags:**

- `[tag]` positional — the tag name (e.g. `v1.4.0`). Required in `--spec` mode (read from spec); optional and prompted in interactive mode.
- `--product <slug-or-uuid>` — target product. Required only if you have multiple products.
- `--name <string>` — optional human-readable release name.
- `--notes <string>` — short release notes (markdown).
- `--notes-file <path|->` — read notes from a file or stdin.
- `--file <path>` — file to upload, repeatable (`--file a.zip --file b.zip`).
- `--env <slug>` — target environment. Defaults to **development**, so a release published without this flag is private until promoted. Pass `--env production` to ship straight to customers. (Note this differs from the read endpoints, which default to production.)
- `--spec <path|->` — JSON spec, alternative to flags.
- `--json` — emit a single JSON result on stdout; logs go to stderr.

**Spec JSON shape:**

```jsonc
{
  "product": "my-slug", // optional if user has only one product
  "tag_name": "v1.4.0", // required
  "release_name": "Late April fixes", // optional, ≤255 chars
  "release_notes": "## Highlights\n…", // optional, markdown, ≤125,000 chars
  "environment": "production", // optional; defaults to development
  "files": [
    // optional; absolute or relative paths
    "./dist/yard-darwin-arm64.tar.gz",
    "./dist/yard-linux-amd64.tar.gz",
  ],
}
```

Unknown fields are rejected. Each file path must exist and be a regular file.

**Two-step publish flow:**

1. Each `files[]` entry streams into the environment's draft download set as `multipart/form-data`.
2. The draft is frozen into an immutable product release (`POST /v1/products/{id}/product-releases`), snapshotting whatever the draft holds.
3. The CLI prints `✓ <path>` or `✗ <path>: <error>` per file on stderr, then a summary like `Release "v1.4.0" published. Uploaded 2/3 file(s).`
4. Exit code is non-zero if any uploads failed; if at least one file uploaded the release was still frozen, and missing assets can be added from the dashboard.

**`--json` output:**

```json
{
  "release":  { "id":"…", "tag_name":"v1.4.0", "files":[...], ... },
  "files": [
    {"path":"./dist/a.tar.gz", "status":"uploaded", "size_bytes":12345678},
    {"path":"./dist/b.tar.gz", "status":"failed",   "error":"open …: no such file"}
  ],
  "uploaded": 1,
  "failed":   1
}
```

**Typical agent flow:**

```sh
# Build artifacts, then ship.
go build -o dist/cli-darwin ./cmd/cli
go build -o dist/cli-linux  ./cmd/cli

yard releases publish --spec - --json <<EOF
{
  "tag_name": "v1.4.0",
  "release_name": "Late April fixes",
  "release_notes": $(jq -Rs . < CHANGELOG.md),
  "files": ["./dist/cli-darwin", "./dist/cli-linux"]
}
EOF
```

### yard releases promote <tag>

Copy a release and its files into another environment. The source release is left untouched.

**Flags:**

- `<tag>` positional — the tag to promote (case-insensitive), e.g. `v1.4.0`.
- `--to <slug>` — target environment. Required.
- `--from <slug>` — source environment. Defaults to `development`, which is where `publish` puts a release when no `--env` is given.
- `--product <slug-or-uuid>` — target product.
- `--json` — emit a single JSON result on stdout.

```sh
yard releases publish v1.4.0 --file dist/app.zip   # lands in development
# …verify the download works…
yard releases promote v1.4.0 --to production       # ships it
```

The files are **duplicated** in storage, so a promoted release counts against your storage quota in both environments, and the copy's download counts start at zero. Promoting fails with 409 if the target already has that tag, and with 403 if the copy wouldn't fit in your quota.

`yard env promote <from> <to>` also carries releases the target is missing, alongside the app, landing page, and pricing.

For full download server schemas (license-key path and API-key path), see [references/releases-and-updates.md](releases-and-updates.md).

---

## yard keys

Manage API keys for programmatic access. Without a subcommand, runs `keys list`.

### yard keys list

Lists your API keys with the same columns the dashboard shows (name, prefix, scopes, last-used, created). The full secret is never displayed — only the prefix `yard_xxxxxxx`.

**Flags:**

- `--json` — emit the raw `APIKeyListResponse` JSON on stdout. The `key` (full secret) is **never** present here.
- `--sort <col>` — `created_at` (default), `name`, or `last_used_at`.
- `--direction <asc|desc>` — sort direction.

**Table output:**

```
NAME                     PREFIX             SCOPES                                   LAST USED      CREATED
--------------------------------------------------------------------------------------------------------------
ci-runner                yard_a1b2c3d       licenses:validate, licenses:activate     2 hours ago    2026-04-12
local-dev                yard_e5f6789       products:read                            never          2026-03-30

Total: 2 / 100 keys
```

### yard keys create [name]

Mints a new API key. **The full secret is shown only once at creation time.** After that, only the prefix is recoverable.

**Flags:**

- `[name]` positional — key name (e.g. `ci-runner`). Required in `--spec` mode; prompted in interactive mode.
- `--scopes <csv>` — comma-separated scope list (e.g. `licenses:validate,licenses:activate`).
- `--spec <path|->` — JSON spec.
- `--json` — emit `APIKeyCreateResponse` (including `key`) on stdout; logs go to stderr.

**Spec JSON shape:**

```jsonc
{
  "name": "ci-runner",
  "scopes": ["licenses:validate", "licenses:activate"],
}
```

**Available scopes:**

| Scope                 | Description                                           |
| --------------------- | ----------------------------------------------------- |
| `products:read`       | Read product metadata                                 |
| `licenses:validate`   | Validate license keys (called from your own software) |
| `licenses:activate`   | Activate / deactivate license keys                    |
| `subscriptions:read`  | Read product subscription status                      |
| `subscriptions:write` | Create / cancel / reactivate product subscriptions    |

Backend caps each user at 100 API keys; on `403` from the create endpoint the CLI prints the reached-limit message.

**Typical agent flow:**

```sh
# Mint a key for a CI job, capture the secret out of the JSON output.
KEY=$(echo '{"name":"ci-runner","scopes":["licenses:validate","licenses:activate"]}' \
  | yard keys create --spec - --json | jq -r .key)
```

For end-user-shipped software, downloads authenticate with license keys against the update server (`/v1/updates/latest`) — see [references/releases-and-updates.md](releases-and-updates.md).

---

## yard licenses

Test license-key validation and inspect test device activations. All three subcommands accept `--product <slug-or-uuid>`; if omitted, the CLI reads the slug from `.yard/settings.json` (walking up from cwd) and falls back to auto-selecting your only product if you have one. All three accept `--json`.

Every product with `license_key_enabled: true` has a sandbox **test license key** — a license key value `POST /v1/licenses/validate` accepts the same way it accepts a real customer's key. The validate endpoint itself still requires an API key with the `licenses:validate` scope (`Authorization: Bearer yard_<key>`); the test key is what goes in the request **body**. Test activations are tracked in a separate `test_activations` table, so they never affect real buyers.

### yard licenses test-key

Print the test license key for a product. Plain output is the bare key (one line, suitable for `$(...)` capture); `--json` emits an object with `product_id`, `product_slug`, and `test_license_key`.

**Errors:**

- Product doesn't have license keys enabled — surfaces a hint to run `yard products edit <slug>` (needs the license_keys permission).
- Product has no test key recorded — rare; usually means license keys were never toggled on.

**Typical use:**

```sh
# Capture the test key for use in a curl/integration test.
KEY=$(yard licenses test-key --product my-app)

# POST /v1/licenses/validate requires an API key with licenses:validate scope —
# the license key being validated goes in the body, the API key goes in the header.
curl -X POST https://api.yard.sh/v1/licenses/validate \
  -H "Authorization: Bearer $YARD_API_KEY" \
  -H 'Content-Type: application/json' \
  -d "{\"license_key\":\"$KEY\",\"device_id\":\"laptop-42\"}"
```

### yard licenses test-activations list

List active test device activations attached to the product's test license key.

**Output (table):**

```
ID                                   DEVICE ID                      DEVICE NAME          ACTIVATED    LAST SEEN
--------------------------------------------------------------------------------------------------------------------
9f3e1c2a-...                         laptop-42                      Alice's MBP          2026-04-28   2026-04-28

2 active / 5 max (3 slots remaining)
```

`--json` emits the raw `ActivationsListResponse` (`{ "activations": [...], "settings": { "enabled", "max_activations", "current_count", "remaining_slots" } }`). When `activations_enabled` is false on the product, the list is empty and the table form prints a one-liner explaining why.

### yard licenses test-activations clear

Deactivate every test device on the product's test license key. **Real customer activations are not touched** — the call only resets `test_activations` rows.

**Flags:**

- `--yes` — skip the confirmation prompt (required when running non-interactively / piped). Implied by `--json`.

**Typical use:**

```sh
# Reset between test runs that hit max_activations.
yard licenses test-activations clear --yes
```

`--json` emits `{ "product_id", "product_slug", "cleared": true }`.

---

## yard coupons

Manage the discount codes buyers redeem at checkout. Without a subcommand, runs `coupons list`.

Coupons need the `coupons` permission on the seller's plan — check `yard me --json` → `.permissions.coupons`. There is no client-side gate: the server answers `403` and the CLI prints what the plan is missing plus the pricing link.

**Two ways to describe a coupon.** Flags cover the common path; `--spec <file|->` takes the API's own JSON shape for full control. When both are given, **flags win** field by field, so a spec can supply the base and a flag can override one value.

**Units differ between the two**, and this is the single most common mistake:

- `--percent 20` → 20% off. `--amount 5` → **dollars** ($5.00 off).
- In a spec, `discount_value` is a percentage for `percentage`, and **cents** for `fixed_amount` (`"discount_value": 500` is $5.00).

Dates accept `YYYY-MM-DD` or full RFC3339. A bare `--expires` date covers the whole day (23:59:59 UTC); a bare `--valid-from` date starts at midnight UTC.

Every subcommand takes a coupon **code or UUID** — codes are resolved server-side, so `yard coupons show LAUNCH20` is one request, not a scan.

### yard coupons list

**Flags:** `--json`, `--sort <col>`, `--direction <asc|desc>`, `--page N`, `--limit N` (max 100).

Sort columns: `createdAt` (default), `lastModified`, `code`, `discountValue`, `scopeDisplay`, `currentUses`, `status`, `savingsCents`.

```
CODE                   DISCOUNT   SCOPE              USES             STATUS     EXPIRES
--------------------------------------------------------------------------------------------
LAUNCH20               20%        all products       3 / 100          active     2026-12-31
INFL7K2M               15%        2 product(s)       0 / unlimited    scheduled  —

2 coupons (1 active), 3 redemptions, $15.00 saved for buyers
```

`STATUS` is derived, not stored: `inactive` (disabled), `expired`, `scheduled` (`valid_from` in the future), `used up` (hit `max_uses`), or `active`. `--json` emits the raw `CouponListResponse`, where `is_active` is only the on/off toggle — compute the rest from `expires_at`, `valid_from`, and `current_uses` if you need it.

### yard coupons show \<code-or-id\>

Coupon detail plus redemption analytics. `--json` emits one object: `{ "coupon": {...}, "analytics": {...} }`.

### yard coupons create \<code\>

**Flags:** `--percent N` | `--amount D`, `--products <csv>`, `--max-uses N`, `--expires <date>`, `--valid-from <date>`, `--subscription-duration <once|forever>`, `--spec <file|->`, `--json`.

The code is upper-cased and must be 4-50 alphanumeric characters. `--products` takes slugs **or** UUIDs and implies `scope=specific_products`; without it a coupon applies to everything the seller sells, including products created later.

`--subscription-duration` only matters for subscription products: `once` discounts the first payment (default), `forever` discounts every renewal.

**Spec shape:**

```jsonc
{
  "discount_type":         "percentage",           // or "fixed_amount"
  "discount_value":        20,                     // percent, or CENTS for fixed_amount
  "scope":                 "all_products",         // or "specific_products"
  "product_ids":           ["<uuid>"],             // required for specific_products
  "max_uses":              100,                    // omit for unlimited
  "expires_at":            "2026-12-31T23:59:59Z",
  "valid_from":            "2026-12-01T00:00:00Z",
  "subscription_duration": "once"                  // or "forever"
}
```

### yard coupons generate

Bulk-generate up to 100 unique codes sharing one set of settings — one code per influencer, reviewer, or beta user.

**Flags:** `--count N` (required, 1-100), `--prefix P`, `--length N` (random part, default 8), plus every `create` flag, `--spec`, `--json`.

Generated codes avoid look-alike characters (no `0`/`O`/`1`/`I`/`L`). **They are returned once** — plain output prints them one per line on stdout (the summary goes to stderr, so a bare redirect captures only codes); `--json` emits the full `CouponListResponse`.

### yard coupons update \<code-or-id\>

Partial update: only the fields passed change. Clearing is explicit, because an omitted flag means "leave it":

- `--no-expiry` — remove the expiry date
- `--no-valid-from` — remove the start date
- `--unlimited-uses` — remove the redemption limit

Also accepts `--activate` / `--deactivate`, `--scope <all_products|specific_products>`, and every `create` flag. `--products` alone re-scopes a coupon; no need to resend `--scope`.

In a spec, `null` clears the same three fields — `{"expires_at": null}` — while omitting a key leaves it alone.

The server refuses to change `discount_type` / `discount_value` on a coupon that has already been redeemed. Everything else stays editable.

### yard coupons rm \<code-or-id\>

Deletes a coupon. A coupon that has been redeemed **cannot** be deleted (its redemptions are transaction history) — use `--deactivate` instead. Requires `--yes` when stdin isn't a TTY.

### yard coupons transactions \<code-or-id\>

The purchases a coupon was redeemed on. **Flags:** `--json`, `--page N`, `--limit N`.

### yard coupons validate \<code\>

Runs the code through the same check the checkout page performs — active, in date, under its limit, applicable to this product — and reports what the buyer would pay. **Flags:** `--product <slug>`, `--tier <uuid>`, `--seller <username>`, `--json`.

The product must be **public**: checkout never sees drafts, so a draft product answers `PRODUCT_NOT_FOUND`. Exit status is 0 whenever the check ran — read `.valid` for the answer.

**Typical agent flows:**

```sh
# Launch discount, capturing the new coupon's id
yard coupons create LAUNCH20 --percent 20 --max-uses 100 --expires 2026-12-31 --json | jq -r .id

# 50 influencer codes, saved for a spreadsheet
yard coupons generate --count 50 --prefix INFL --percent 15 --json | jq -r '.coupons[].code' > codes.txt

# Retire every code that has hit its usage cap
yard coupons list --json \
  | jq -r '.coupons[] | select(.max_uses != null and .current_uses >= .max_uses) | .code' \
  | xargs -r -n1 -I{} yard coupons update {} --deactivate

# Read-modify-write via specs
echo '{"max_uses": 250, "expires_at": null}' | yard coupons update LAUNCH20 --spec - --json

# Confirm a code works before sending it to customers
yard coupons validate LAUNCH20 --product my-tool --json | jq '{valid, final_price_cents}'
```

---

## yard customers

Read-only view of the people who bought the seller's products. Without a subcommand, runs `customers list`.

A "customer" is a buyer with at least one **completed, unrefunded** purchase, aggregated across the seller's whole catalog — a buyer whose only order was refunded does not appear. With `--product`, the aggregation narrows to that one product: the rows are its buyers, and each customer's order count and spend cover only their orders of it.

Buyers are identified by an opaque id: `cust_` plus the first 8 characters of their account id. That id is what `customers show` takes; there is no email lookup.

Money comes back **pre-formatted** (`"total_spent_display": "$87.00"`). There is no cents field on these responses — use `yard transactions` when you need to do arithmetic.

### yard customers list

**Flags:** `--json`, `--sort <col>`, `--direction <asc|desc>`, `--product <slug-or-uuid>`, `--page N`, `--limit N` (max 100).

Sort columns: `lastTransaction` (default), `email`, `username`, `orderCount`, `totalSpent`, `buyerId`.

```
ID               CUSTOMER                       ORDERS   SPENT        FIRST        LAST
--------------------------------------------------------------------------------------
cust_deadbeef    @alice                         3        $87.00       2026-01-04   2026-07-20
cust_c0ffee11    bob@example.com                1        $29.00       2026-07-18   2026-07-18

2 customers (1 new this month), $58.00 avg spend, 50.0% repeat rate
```

The summary line doesn't move with pagination. It is account-wide by default, and product-wide under `--product`.

Unlike `yard transactions --product` — where the filters narrow the rows but the earnings summary stays account-wide — `yard customers --product` narrows the summary too, because "customers of this product" is a different set from "customers", not a filtered view of it.

### yard customers show \<customer-id\>

One buyer's totals plus their orders from this seller. **Flags:** `--json`, `--page N`, `--limit N`.

`--json` emits `{ "customer": {...}, "transactions": [...], "total", "page", "limit" }`. Unlike the list, the transactions here **include refunded orders**.

Two customers can share an 8-character id prefix. The server answers `409` rather than guessing; pass the buyer's full UUID in that case.

**Typical agent flows:**

```sh
# Biggest spenders
yard customers --json | jq -r '.customers | sort_by(.order_count) | reverse | .[:5] | .[] | "\(.email) \(.total_spent_display)"'

# Everything one buyer owns
yard customers show cust_deadbeef --json | jq -r '.transactions[] | "\(.product_name) \(.created_at)"'

# Who bought one specific product, biggest spenders first
yard customers --product my-tool --sort totalSpent --direction desc --json | jq -r '.customers[] | "\(.email) \(.total_spent_display)"'

# Who bought in the last month
yard customers --json | jq -r --arg since "$(date -u -d '30 days ago' +%Y-%m-%d)" \
  '.customers[] | select(.last_transaction >= $since) | .email'
```

---

## yard transactions

The seller's sales, plus the one write in this area: adjusting a buyer's free-trial length. Without a subcommand, runs `transactions list`.

Every sale has a full UUID and a short **order id** — `order_` plus the first 8 characters, which is what the dashboard and receipts show. Both are accepted anywhere a transaction id is taken; an ambiguous short id is answered with `409` rather than a guess.

Refunds are **not** exposed here — issuing one stays in the dashboard.

### yard transactions list

**Flags:** `--json`, `--trials`, `--product <slug-or-id>`, `--start <date>`, `--end <date>`, `--sort <col>`, `--direction <asc|desc>`, `--page N`, `--limit N` (max 100).

Sort columns: `date` (default), `amount`, `sellerEarnings`, `productName`. Dates take `YYYY-MM-DD` or RFC3339; a bare `--end` date covers that whole day.

```
ORDER            DATE         PRODUCT                    CUSTOMER              TOTAL       EARNINGS    TYPE           STATUS
-----------------------------------------------------------------------------------------------------------------------------
order_1a2b3c4d   2026-07-20   My Tool                    @alice                $29.99      $26.99      purchase       completed
order_9f8e7d6c   2026-07-18   My Tool                    bob@example.com       $0.00       $0.00       trial          completed

2 of 97 transactions · $2699.00 earned, 97 sales, $27.81 avg order, 3 active trials
```

`--trials` and the date/product filters narrow the rows **and** the total; the summary figures stay account-wide, matching the dashboard. `TYPE` is derived: `gift`, `trial`, `trial upgrade`, `subscription`, or `purchase`. `STATUS` shows the refund state when there is one, because a refunded sale still has `status: "completed"`.

### yard transactions show \<order-id\>

One sale in full: tier, quantity, coupon, refund date, billing period, and — for a trial — when it runs out and how long that is from now. **Flags:** `--json`.

### yard transactions trial \<order-id\> --add-days N

Lengthen or shorten a live free trial. `--add-days 7` gives a week; `--add-days -3` takes three days back. Non-zero, and 365 either way. **Flags:** `--add-days N` (required), `--json`.

Two behaviours to state plainly before running it:

- **Days are added to the trial's current expiry, not to today.** Extending a trial that expired a month ago by 7 days still leaves it in the past — add enough days to land in the future. When the new expiry *is* in the future, an expired trial is set back to `active` and the buyer has access again; the response reports this as `"reactivated": true`. If the trial stays expired, the CLI says why.
- **The buyer is emailed** about the change, same as adjusting it from the dashboard.

A revival is refused in one case: the buyer already has another pending or active trial on that product. The expiry still moves, `reactivated` comes back `false`, and the status stays `expired`.

Only trial transactions have a length to adjust; anything else is rejected before the request is made. This is the *running* trial for one buyer — the trial length offered to **new** buyers is the per-tier `free_trial_days` setting, changed with `yard products tiers edit`.

Requires a plan that can sell products (`yard me --json` → `.permissions.sell_products`).

**Typical agent flows:**

```sh
# Trials, and when each one runs out
yard transactions list --trials --json \
  | jq -r '.transactions[] | "\(.id) \(.customer_email) \(.trial_expires_at)"'

# Give one buyer another week, and read back the new expiry
yard transactions trial order_1a2b3c4d --add-days 7 --json | jq -r .trial_expires_at

# Add 3 days to every trial ending in the next 48 hours
yard transactions list --trials --json \
  | jq -r --arg cutoff "$(date -u -d '+2 days' +%Y-%m-%dT%H:%M:%SZ)" \
      '.transactions[] | select(.trial_expires_at != null and .trial_expires_at <= $cutoff) | .id' \
  | xargs -r -n1 -I{} yard transactions trial {} --add-days 3

# What one product earned this month
yard transactions list --product my-tool --start 2026-07-01 --json \
  | jq '[.transactions[].seller_earnings_cents] | add'

# Which sales used a coupon
yard transactions list --json | jq -r '.transactions[] | select(.coupon_code) | "\(.coupon_code) \(.id)"'
```

---

## yard version

Display version and build information.

**Output format:**

```
yard v0.1.0+abc1234
  Commit:    abc1234
  Built:     2025-01-15T10:30:00Z
  Platform:  linux/amd64
  Go:        go1.23.0
```

---

## yard update

Update the CLI to the latest version.

**Flags:**

- `--check`, `-c` — Only check for updates without installing

**How it works:**

1. Fetches remote version from `{UpdateURL}/current_version.txt`
2. Compares with local version
3. If `--check`, prints the available version and exits
4. Otherwise, downloads the binary: `{UpdateURL}/{version}/yard-{os}-{arch}-{version}{.exe}`
5. Replaces the current binary in-place:
   - Unix: downloads to temp file, sets 0755 permissions, renames over current binary
   - Windows: renames current binary to `.old`, renames new binary into place, deletes `.old`

**Update URL:** `https://cli.yard.sh` (production)

---

## yard page

Manage a product's custom landing page. All subcommands accept the following common flags unless noted otherwise:

- `--product <slug-or-uuid>` — identify the product explicitly (otherwise read from `.yard/settings.json`; if that's missing and the user has exactly one product, it's auto-selected)
- `--project <path>` — project root override (defaults to walking up from cwd for a `.yard/` directory)
- `--env <slug>` — which environment to act on (default: `development`). `publish` uses `--from` instead
- `--json` — emit a single machine-readable JSON object on stdout; logs go to stderr
- `--yes`, `-y` — skip confirmation prompts (only applies to destructive commands)

**Environments:** every command edits exactly one environment and none of them
touch the live page. `production` is what visitors see, and content reaches it
only through `yard page publish`. There is no separate draft: the environment
IS the saved state.

**Exit codes:**

- `0` — success
- `1` — fatal error (auth, network, validation, API error)
- `2` — partial success (only produced by `push` when some files uploaded and others failed)

**Project discovery:** any `yard page` subcommand walks upward from cwd looking for `.yard/settings.json` (same mechanism as `git`'s `.git` lookup). The directory containing it is the project root. The CLI's own config dir at `~/.yard/` is not matched because it only holds `config.json`, never `settings.json`. `--project <path>` overrides this.

**Constraints enforced client-side before any HTTP:**

- Extension must be in `.html .css .js .json .svg .png .jpg .jpeg .webp .gif .woff2`
- Path must match `^[a-zA-Z0-9][a-zA-Z0-9._-]*(/[a-zA-Z0-9][a-zA-Z0-9._-]*)?$` — at most one subdirectory level, no dotfiles, no leading slash, no `..`
- Per-file ≤ 1 MB
- Bundle ≤ 5 MB total
- File count ≤ 20
- `index.html` required when publishing

---

### yard page init

Create a `.yard/` project directory linked to a product. If the product already has remote files, they are pulled into `.yard/landing-page/`; otherwise a hello-world `index.html` + `styles.css` is scaffolded.

**Behavior:**

1. Resolves the product (flag → existing settings → sole product → error).
2. Creates `<project>/.yard/` with perm 0750.
3. Writes `<project>/.yard/settings.json` if absent: `{"version": 1, "product_slug": "<slug>", "ignore_files": []}`.
4. Fetches remote metadata for `--env`. If that environment has files, pulls those. Else if production has files, pulls those. Else writes the hello-world starter (`index.html` + `styles.css`).
5. Local files that already match the remote SHA-256 are skipped; files never overwrite an existing local file with different contents.
6. Prints the next steps and the preview URL.

**Idempotent:** running `init` twice is safe. Existing files are preserved.

**JSON output (`--json`):**

```json
{
  "project_root": "/home/alice/my-landing",
  "product": "my-slug",
  "source": "development",
  "written": ["index.html", "styles.css"],
  "skipped": [],
  "preview_url": "https://yard.sh/api/v1/products/.../custom-page/preview",
  "live_url": null
}
```

`source` is `"starter"` when both bundles were empty and the starter was written; otherwise the environment slug it pulled from (`"production"` when the requested environment was empty).

---

### yard page status

Print the diff between local files and one environment without writing anything.

**Output categories:**

- `to_upload` — files changed or missing on remote
- `unchanged` — local and remote hashes match
- `remote_only` — files in the environment but not locally
- `to_delete` — always empty (prune is only done via `push --prune`)

**JSON output:**

```json
{
  "product": "my-slug",
  "environment": "development",
  "to_upload": ["index.html"],
  "to_delete": [],
  "unchanged": ["styles.css"],
  "remote_only": ["old.html"]
}
```

---

### yard page ls

List files in one environment's bundle. Pass `--env production` to list what
is currently live.

**JSON output:**

```json
{
  "product": "my-slug",
  "environment": "development",
  "files": [
    {
      "path": "index.html",
      "content_type": "text/html",
      "size_bytes": 412,
      "content_hash": "…",
      "updated_at": "2026-04-20T10:00:00Z"
    }
  ]
}
```

---

### yard page push

Hash-diff the local landing-page directory against one environment and upload changed files.

**Flags:**

- `--prune` — delete remote files that are not present locally (opt-in for safety)
- `--publish` — publish the environment to production after uploading (requires `index.html`)
- `--yes`, `-y` — skip prune confirmation prompt

**Behavior:**

1. Walk `<project>/.yard/landing-page/`, skip dotfiles and files matching `ignore_files` patterns from `settings.json`.
2. Validate extensions, paths, per-file size, bundle size, and file count client-side.
3. `GET /v1/products/{id}/custom-page?environment=<env>` to fetch remote hashes.
4. For each local file whose SHA-256 doesn't match the remote hash, `PUT /v1/products/{id}/custom-page/files/{path}?environment=<env>` with the raw bytes.
5. If `--prune`, `DELETE` remote files that are not present locally (prompts unless `--yes` or `--json`).
6. If `--publish`, `POST /v1/products/{id}/landing-page/publish` with `{"environment": "<env>"}`.
7. Re-fetch metadata and print `Preview:` (and `Live:` if published).

**JSON output:**

```json
{
  "product": "my-slug",
  "environment": "development",
  "uploaded": ["index.html", "styles.css"],
  "skipped": ["logo.png"],
  "deleted": [],
  "remote_only": [],
  "published": false,
  "preview_url": "https://yard.sh/api/v1/products/.../custom-page/preview",
  "live_url": null,
  "errors": []
}
```

When `--prune` is not passed, `remote_only` lists paths on the server that don't exist locally (informational). When `--prune` is passed, `remote_only` is emitted as an empty array and the removed paths appear in `deleted` instead.

**Exit code 2** is returned when at least one file uploaded and at least one failed. Use `errors` in the JSON output to disambiguate.

---

### yard page pull

Download one environment's bundle files to the local landing-page directory.

**Flags:**

- `--force` — overwrite local files even if their hash matches the remote
- `--dir <path>` — output directory override (defaults to `<project>/.yard/landing-page/`)

**Behavior:**

1. Resolves project root (with `.yard/` discovery allowed to be missing — `pull` is useful for bootstrapping a fresh checkout).
2. Walks the chosen environment's bundle; for each file, downloads and writes to disk unless the local file already has the same SHA-256 (skipped).
3. Subdirectories are created as needed.

**JSON output:**

```json
{
  "product": "my-slug",
  "environment": "development",
  "destination": "/home/alice/my-landing/.yard/landing-page",
  "written": ["index.html"],
  "skipped": ["styles.css"]
}
```

---

### yard page publish

Overwrite production's landing page with an environment's and make it live. The
whole page moves together: pre-built content, gallery, and the custom bundle.

**Flags:**

- `--from <slug>` — environment to publish from (default: `development`). Pass `--from production` to redeploy production as it already stands, after editing it directly.

**Behavior:**

1. `POST /v1/products/{id}/landing-page/publish` with `{"environment": "<from>"}` — the server validates that `index.html` exists when the page type is custom.
2. Re-fetches metadata and prints `Live:` and `Preview:` URLs.

**JSON output:**

```json
{
  "product": "my-slug",
  "environment": "development",
  "published": true,
  "preview_url": "https://yard.sh/api/v1/products/.../custom-page/preview",
  "live_url": "https://yard.sh/@alice/my-slug"
}
```

There is no `yard page revert`: with no draft layer there is nothing to revert
*to*. Use `yard page pull --env production` to overwrite your working copy with
what is live.

---

## yard env

Manage a product's environments. Every product has two protected
environments, `development` and `production`; landing-page pushes and app
deploys default to `development`, and publishing is what goes live. Shared
flags: `--product <slug-or-uuid>`, `--project <path>`, `--json` (same
semantics as `yard page`).

### yard env list

Lists the product's environments. JSON: array of
`{"id": "<uuid>", "slug": "development", "protected": true}`.

### yard env create \<slug\>

Creates a custom environment (Pro; cap of 10 total per product, counting
the two protected defaults). Slugs are 2-60 characters: lowercase letters,
digits, and hyphens, starting with a letter. `development` and `production`
always exist and cannot be recreated.

### yard env delete \<slug\>

Deletes a custom environment and everything in it: its files, its app
Worker, and its app database (immediately — no grace window for custom
environments). `development`/`production` are protected and refuse.

### yard env promote \<from\> \<to\>

Copies the product's artifacts — landing page and web-app bundle — from one
environment to another. Promoting to `production` deploys the result live.
Code and assets only: **data and secrets never promote**; each environment
keeps its own. JSON: `{"from": "...", "to": "...", "artifacts": ["page", "app"]}`
(`artifacts` names what actually moved). Human output prints the live URL,
plus the `/app/` URL when the app artifact was promoted.

---

## yard app

Deploy and manage a product's web app (see `webapps.md` for the runtime
model). Requires the `compute` permission (Pro). Shared flags:
`--product <slug-or-uuid>`, `--project <path>`, `--env <slug>` (default
`development`), `--json`.

**Bundle constraints enforced client-side before any HTTP:** ≤200 files,
≤5 MB per file, ≤25 MB total, paths nest ≤8 levels, extensions in
`.html .css .js .mjs .json .svg .png .jpg .jpeg .webp .gif .woff2 .woff
.ttf .otf .txt .md .ico .map .wasm .webmanifest` (plus `_worker.js`,
`yard.json`, `migrations/*.sql`). `_worker.js` is required. Dotfiles and
bundle-root `wrangler.toml`/`README.md` are skipped (reported as ignored),
so old scaffolds that carried them inside the bundle still deploy.

### yard app init

Scaffolds a zero-dependency working app (notes API + vanilla frontend +
first migration + `yard.json`) into `./app` (`--dir <name>` to change).
Local-dev files — `wrangler.toml`, `.dev.vars.example`, `README.md` — are
written at the **project root**, never inside the bundle, and are
write-if-absent (existing files are skipped and reported). Records the
bundle dir as `app_dir` in `.yard/settings.json` when the project is
yard-initialized. JSON:
`{"dir": "app", "written": [...], "skipped": [...], "app_dir_recorded": true}`.

### yard app deploy

Uploads the bundle (content-addressed — only changed files transfer;
remote files missing locally are removed) and deploys it to the target
environment. Bundle dir resolution: `--dir` flag → `app_dir` from
`.yard/settings.json` → `./dist`. Prints root-absolute-URL lint warnings.
JSON:

```json
{
  "product": "my-slug",
  "environment": "development",
  "script_name": "ya-<uuid>-<hex8>",
  "etag": "…",
  "access": "authenticated",
  "url": "https://alice.yard.sh/my-slug/app/?yard_env=development",
  "uploaded": ["_worker.js"],
  "deleted": [],
  "ignored": [],
  "warnings": []
}
```

`url` is the browsable address for the deployed environment (owner-only
`?yard_env` preview for non-production). A failed migration aborts the
deploy with a 422 naming the file and the recovery path; the previous
version keeps serving.

### yard app status

Shows an environment's bundle (files, bytes, worker presence), last
deployment, and `url`. `--diff` (with optional `--dir`) compares the local
bundle against the remote one; JSON gains
`"diff": {"changed": [...], "remote_only": [...]}`.

### yard app open

Prints the environment's URL and opens it in a browser. Non-production URLs
are owner-only previews (sign-in enforced at the edge). JSON:
`{"environment": "...", "url": "...", "deployed": true}`.

### yard app check

Validates the local bundle exactly like a deploy would (limits, extensions,
`_worker.js` presence) plus lint warnings for root-absolute `href`/`src`/
`fetch("/…")` URLs — no network, no login. JSON:
`{"dir": "...", "files": 7, "total_bytes": 5494, "ignored": [...], "warnings": [...]}`.

### yard app secrets set KEY=VALUE [KEY=VALUE...] / list / rm \<name\>

Per-environment secrets that become `env.<NAME>` bindings on the **next
deploy**. Names UPPER_SNAKE (≤32 per environment, ≤4 KB each; `DB` and
`ASSETS` reserved). Write-only: `list` shows names and timestamps, never
values. Secrets don't promote between environments.

### yard app db query [sql]

Runs SQL against the environment's database and prints rows as JSON. SQL
from the inline argument, `--file <path>`, or stdin (`-`). Caps: 10 kB SQL,
1000 rows returned. The `_yard_migrations` table records applied
migrations.

### yard app logs

Recent Worker output (console lines, uncaught exceptions, abnormal request
outcomes), newest window ≤24 h. `--limit <n>` (cap 500), `--since <dur>`
(e.g. `30m`, `2h`). An app that has never logged returns an empty list, not
an error. Logs appear a few seconds after the request.

---

## yard uninstall

Remove the CLI from your system.

**Flags:**

- `--force`, `-f` — Skip the confirmation prompt

**What gets removed:**

1. Config directory (`~/.yard/` — contains `config.json`)
2. CLI binary (detected via `os.Executable()`)

If removal fails (e.g., permission denied), prints manual cleanup instructions with the exact paths.
