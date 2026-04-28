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

8. **Optional product-settings prompts (Pro only)** — After landing-page setup, the wizard asks Pro accounts (in order):
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
