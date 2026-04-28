# Yard CLI Command Reference

## Global Behavior

- The CLI binary name is `yard` (production) or `yard-staging` (staging builds)
- API URL defaults to `https://api.yard.sh` but can be overridden at build time
- Config is stored at `~/.yard/config.json` (or `~/.yard-staging/` for staging)
- All authenticated commands send `Authorization: Session {token}` header

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

Print the currently logged-in user and their subscription level. Useful for agents that need to gate Pro-only feature suggestions before proposing them.

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
  "is_pro": true
}
```

`username` is the best available display name (GitHub username → username → email → id), matching `User.DisplayName()`. `is_pro` is the authoritative subscription check — it reflects the `pro` role on `GET /v1/me`, which the platform automatically grants/revokes when the user's subscription tier changes.

**Agent usage:** before suggesting any Pro-only feature (license keys, device activations, free trials, multiple pricing tiers, seat-based pricing, coupons, gift purchases, custom landing pages), run `yard me --json` and read `.is_pro`. If false, either pick a free-tier-compatible alternative or surface the upgrade link `https://yard.sh/upgrade`.

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

6. **Scaffold `.yard/`** — Writes `./.yard/settings.json` via `EnsureYardProject`. Existing settings are preserved.

7. **Optional landing-page setup** — Prompts `Set up a custom landing page for <slug>? [y/N]`. If yes:
   - Creates `./.yard/landing-page/`.
   - Runs the same source-pick logic as `yard page init` (draft → published → starter) and pulls or scaffolds accordingly.
   - Calls `POST /v1/products/{id}/custom-page/publish`. On a Pro-required 403, prints the `https://yard.sh/upgrade` message to stderr, keeps the saved draft, and exits 0. On other errors, fails.

8. **Optional product-settings prompts (Pro only — check with `yard me --json` → `.is_pro`)** — After landing-page setup, the wizard asks Pro accounts (in order):
   - "Enable license keys? [y/N]" — toggles `license_key_enabled`.
   - If license keys are on: "Enable device activations? [y/N]" → on yes, "Device activation limit (1-10000) [3]:".
   - "Enable a free trial? [y/N]" → on yes, "Free trial days (1-365) [7]:".

   Changes are applied via `PUT /v1/products/{id}` — server-side enforcement of the Pro requirement remains authoritative. Free accounts see a single block describing these as Pro features with the `https://yard.sh/upgrade` link, and the wizard skips the prompts. Spec mode (`yard init --spec`) accepts the same fields directly in the JSON payload (see SKILL.md schema).

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
- `--json` emits the underlying `ProductListItem` array (now including `license_key_enabled`, `activations_enabled`, `max_activations`, `free_trial_enabled`, `free_trial_days`).

### yard products edit [slug-or-id]

Modify the Pro-only seller settings on an existing product: license keys, device activations (and the per-key limit), and free trials (and the trial duration).

**Resolution:**
- Pass a slug or UUID as the first argument to target a specific product.
- Without an argument, auto-selects when the user has only one product, prompts to pick from a numbered list otherwise.
- In `--spec` mode with multiple products and no argument, errors out with the list of slugs.

**Interactive flow (Pro accounts):**
1. Prints `Editing settings for <title> (<slug>)`.
2. Same prompt sequence as `yard init`'s settings step. Each prompt's default reflects the *current* value, so pressing Enter is always a no-op.
3. Calls `PUT /v1/products/{id}` with only the fields that changed.

**Free-account behavior:** prints the upgrade message and the `https://yard.sh/upgrade` link, then exits cleanly (exit 0 in interactive mode, exit 1 with `pro_required` in `--spec` mode).

**Spec mode:**
- `--spec <file|->` — read JSON from a file or stdin. The JSON shape is `UpdateProductRequest`:
  ```json
  {
    "license_key_enabled": true,
    "activations_enabled": true,
    "max_activations": 5,
    "free_trial_enabled": true,
    "free_trial_days": 14
  }
  ```
  Unknown fields are rejected. Missing fields are left untouched (the request is sparse).
- `--json` — emit `{ "product": {...}, "settings": {...} }` on stdout; logs go to stderr.
- The CLI pre-checks the activations-needs-license-keys rule against the *effective* state, so a spec that flips activations on without restating `license_key_enabled` succeeds when the product already has license keys enabled.
- A 403 with `error_code: "pro_required"` is rendered as a clean upgrade message rather than the raw HTTP error.

**Typical agent flow:**
```sh
# Discover what exists.
yard products --json

# Apply settings to a known product.
yard products edit my-awesome-tool --spec - --json <<'EOF'
{
  "license_key_enabled": true,
  "activations_enabled": true,
  "max_activations": 5,
  "free_trial_enabled": true,
  "free_trial_days": 14
}
EOF
```

---

## yard releases

Manage releases for a product. Today the CLI exposes `publish` only — list/edit/delete still happen in the dashboard.

### yard releases publish [tag]

Create a new release with optional file assets. The release is created first, then each file is uploaded one at a time so a single failed asset doesn't lose the release.

**Flags:**
- `[tag]` positional — the tag name (e.g. `v1.4.0`). Required in `--spec` mode (read from spec); optional and prompted in interactive mode.
- `--product <slug-or-uuid>` — target product. Required only if you have multiple products.
- `--name <string>` — optional human-readable release name.
- `--notes <string>` — short release notes (markdown).
- `--notes-file <path|->` — read notes from a file or stdin.
- `--file <path>` — file to upload, repeatable (`--file a.zip --file b.zip`).
- `--spec <path|->` — JSON spec, alternative to flags.
- `--json` — emit a single JSON result on stdout; logs go to stderr.

**Spec JSON shape:**
```jsonc
{
  "product":        "my-slug",          // optional if user has only one product
  "tag_name":       "v1.4.0",           // required
  "release_name":   "Late April fixes", // optional, ≤255 chars
  "release_notes":  "## Highlights\n…", // optional, markdown, ≤125,000 chars
  "files": [                             // optional; absolute or relative paths
    "./dist/yard-darwin-arm64.tar.gz",
    "./dist/yard-linux-amd64.tar.gz"
  ]
}
```

Unknown fields are rejected. Each file path must exist and be a regular file.

**Two-step publish flow:**
1. `POST /v1/products/{id}/releases` (JSON body) creates the release with metadata only.
2. For each `files[]` entry, `POST /v1/products/{id}/releases/{releaseId}/files` streams the file as `multipart/form-data`.
3. The CLI prints `✓ <path>` or `✗ <path>: <error>` per file on stderr, then a summary like `Release "v1.4.0" published. Uploaded 2/3 file(s).`
4. Exit code is non-zero if any uploads failed; the release still exists in the dashboard so you can re-upload via the UI.

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
ci-runner                yard_a1b2c3d       releases:read, releases:download         2 hours ago    2026-04-12
local-dev                yard_e5f6789       products:read                            never          2026-03-30

Total: 2 / 100 keys
```

### yard keys create [name]

Mints a new API key. **The full secret is shown only once at creation time.** After that, only the prefix is recoverable.

**Flags:**
- `[name]` positional — key name (e.g. `ci-runner`). Required in `--spec` mode; prompted in interactive mode.
- `--scopes <csv>` — comma-separated scope list (e.g. `releases:read,releases:download`).
- `--spec <path|->` — JSON spec.
- `--json` — emit `APIKeyCreateResponse` (including `key`) on stdout; logs go to stderr.

**Spec JSON shape:**
```jsonc
{
  "name":   "ci-runner",
  "scopes": ["releases:read", "releases:download"]
}
```

**Available scopes:**

| Scope | Description |
|---|---|
| `products:read` | Read product metadata |
| `releases:read` | Read release metadata for owned products |
| `releases:download` | Download release files for owned products |
| `licenses:validate` | Validate license keys (called from your own software) |
| `licenses:activate` | Activate / deactivate license keys |
| `subscriptions:read` | Read product subscription status |
| `subscriptions:write` | Create / cancel / reactivate product subscriptions |

Backend caps each user at 100 API keys; on `403` from the create endpoint the CLI prints the reached-limit message.

**Typical agent flow:**
```sh
# Mint a key for a CI job, capture the secret out of the JSON output.
KEY=$(echo '{"name":"ci-runner","scopes":["releases:read","releases:download"]}' \
  | yard keys create --spec - --json | jq -r .key)
```

For end-user-shipped software, the license-key update server (`/v1/updates/latest`) and an embedded API key (`/v1/products/{id}/releases/latest`) are both valid auth paths — see [references/releases-and-updates.md](releases-and-updates.md) for the tradeoffs.

---

## yard licenses

Test license-key validation and inspect test device activations. All three subcommands accept `--product <slug-or-uuid>`; if omitted, the CLI reads the slug from `.yard/settings.json` (walking up from cwd) and falls back to auto-selecting your only product if you have one. All three accept `--json`.

Every product with `license_key_enabled: true` has a sandbox **test license key** — a license key value `POST /v1/licenses/validate` accepts the same way it accepts a real customer's key. The validate endpoint itself still requires an API key with the `licenses:validate` scope (`Authorization: Bearer yard_<key>`); the test key is what goes in the request **body**. Test activations are tracked in a separate `test_activations` table, so they never affect real buyers.

### yard licenses test-key

Print the test license key for a product. Plain output is the bare key (one line, suitable for `$(...)` capture); `--json` emits an object with `product_id`, `product_slug`, and `test_license_key`.

**Errors:**
- Product doesn't have license keys enabled — surfaces a hint to run `yard products edit <slug>` (Pro only).
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

**Update URL:** `https://install.yard.sh` (production)

---

## yard page

Manage a product's custom landing page. All subcommands accept the following common flags unless noted otherwise:

- `--product <slug-or-uuid>` — identify the product explicitly (otherwise read from `.yard/settings.json`; if that's missing and the user has exactly one product, it's auto-selected)
- `--project <path>` — project root override (defaults to walking up from cwd for a `.yard/` directory)
- `--json` — emit a single machine-readable JSON object on stdout; logs go to stderr
- `--yes`, `-y` — skip confirmation prompts (only applies to destructive commands)

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
4. Fetches remote metadata. If `draft_files` is non-empty, pulls those. Else if `published_files` is non-empty, pulls those. Else writes the hello-world starter (`index.html` + `styles.css`).
5. Local files that already match the remote SHA-256 are skipped; files never overwrite an existing local file with different contents.
6. Prints the next steps and the preview URL.

**Idempotent:** running `init` twice is safe. Existing files are preserved.

**JSON output (`--json`):**
```json
{
  "project_root": "/home/alice/my-landing",
  "product": "my-slug",
  "source": "draft",
  "written": ["index.html", "styles.css"],
  "skipped": [],
  "preview_url": "https://api.yard.sh/v1/products/.../custom-page/preview",
  "live_url": null
}
```

`source` is `"starter"` when the remote bundle was empty and the starter was written; `"draft"` or `"published"` otherwise.

---

### yard page status

Print the diff between local files and the remote draft without writing anything.

**Output categories:**
- `to_upload` — files changed or missing on remote
- `unchanged` — local and remote hashes match
- `remote_only` — files on remote draft but not locally
- `to_delete` — always empty (prune is only done via `push --prune`)

**JSON output:**
```json
{
  "product": "my-slug",
  "to_upload": ["index.html"],
  "to_delete": [],
  "unchanged": ["styles.css"],
  "remote_only": ["old.html"]
}
```

---

### yard page ls

List files in the remote bundle.

**Flags:**
- `--source draft|published` — which bundle to list (default: `draft`)

**JSON output:**
```json
{
  "product": "my-slug",
  "source": "draft",
  "files": [
    {"path": "index.html", "content_type": "text/html", "size_bytes": 412, "content_hash": "…", "updated_at": "2026-04-20T10:00:00Z"}
  ]
}
```

---

### yard page push

Hash-diff the local landing-page directory against the remote draft and upload changed files.

**Flags:**
- `--prune` — delete remote files that are not present locally (opt-in for safety)
- `--publish` — promote the draft to live after uploading (requires `index.html`)
- `--yes`, `-y` — skip prune confirmation prompt

**Behavior:**
1. Walk `<project>/.yard/landing-page/`, skip dotfiles and files matching `ignore_files` patterns from `settings.json`.
2. Validate extensions, paths, per-file size, bundle size, and file count client-side.
3. `GET /v1/products/{id}/custom-page` to fetch remote hashes.
4. For each local file whose SHA-256 doesn't match the remote hash, `PUT /v1/products/{id}/custom-page/files/{path}` with the raw bytes.
5. If `--prune`, `DELETE` remote files that are not present locally (prompts unless `--yes` or `--json`).
6. If `--publish`, `POST .../publish`.
7. Re-fetch metadata and print `Preview:` (and `Live:` if published).

**JSON output:**
```json
{
  "product": "my-slug",
  "uploaded": ["index.html", "styles.css"],
  "skipped": ["logo.png"],
  "deleted": [],
  "remote_only": [],
  "published": false,
  "preview_url": "https://api.yard.sh/v1/products/.../custom-page/preview",
  "live_url": null,
  "errors": []
}
```

When `--prune` is not passed, `remote_only` lists paths on the server that don't exist locally (informational). When `--prune` is passed, `remote_only` is emitted as an empty array and the removed paths appear in `deleted` instead.

**Exit code 2** is returned when at least one file uploaded and at least one failed. Use `errors` in the JSON output to disambiguate.

---

### yard page pull

Download remote bundle files to the local landing-page directory.

**Flags:**
- `--source draft|published` — which bundle to pull (default: `draft`)
- `--force` — overwrite local files even if their hash matches the remote
- `--dir <path>` — output directory override (defaults to `<project>/.yard/landing-page/`)

**Behavior:**
1. Resolves project root (with `.yard/` discovery allowed to be missing — `pull` is useful for bootstrapping a fresh checkout).
2. Walks the chosen remote bundle; for each file, downloads and writes to disk unless the local file already has the same SHA-256 (skipped).
3. Subdirectories are created as needed.

**JSON output:**
```json
{
  "product": "my-slug",
  "source": "draft",
  "destination": "/home/alice/my-landing/.yard/landing-page",
  "written": ["index.html"],
  "skipped": ["styles.css"]
}
```

---

### yard page publish

Promote the current draft bundle to live.

**Behavior:**
1. `POST /v1/products/{id}/custom-page/publish` — server validates that `index.html` exists.
2. Re-fetches metadata and prints `Live:` and `Preview:` URLs.

**JSON output:**
```json
{
  "product": "my-slug",
  "published": true,
  "preview_url": "https://api.yard.sh/v1/products/.../custom-page/preview",
  "live_url": "https://yard.sh/@alice/my-slug"
}
```

---

### yard page revert

Discard the current draft and restore it from the published bundle.

**Flags:**
- `--yes`, `-y` — skip confirmation prompt

**Behavior:**
1. Confirms with the user unless `--yes` or `--json` is passed.
2. `POST /v1/products/{id}/custom-page/revert`.

**JSON output:**
```json
{
  "product": "my-slug",
  "reverted": true,
  "preview_url": "https://api.yard.sh/v1/products/.../custom-page/preview"
}
```

---

## yard uninstall

Remove the CLI from your system.

**Flags:**
- `--force`, `-f` — Skip the confirmation prompt

**What gets removed:**
1. Config directory (`~/.yard/` — contains `config.json`)
2. CLI binary (detected via `os.Executable()`)

If removal fails (e.g., permission denied), prints manual cleanup instructions with the exact paths.
