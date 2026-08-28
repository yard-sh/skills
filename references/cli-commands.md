# Yard CLI Command Reference

## Global Behavior

- The CLI binary name is `yard` (production) or `yard-staging` (staging builds)
- API URL defaults to `https://api.yard.sh` but can be overridden at build time
- Config is stored at `~/.yard/config.json` (or `~/.yard-staging/` for staging)
- All authenticated commands send an `Authorization: Client {token}` header (the token `yard login` saved)

---

## yard login

Authenticate with your GitHub account.

A new account signing up through the CLI is walked through the browser onboarding first — profile, then **team creation** — before the token is handed back, because a seller with no team can't do anything. An existing account goes straight to the handoff. Either way, run `yard team` afterwards to see which team the session acts as.

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

Print the currently logged-in user, the team they act as, and what each is entitled to. Useful for agents deciding which features to suggest before proposing them.

**Usage:** `yard me [--json]`

**Auth:** required. If not logged in, exits with `not logged in. Run 'yard login' first`.

**Human output:**

```
Username:     alice
GitHub:       alice
Email:        alice@example.com
Subscription: Pro
Team:         Acme Corp (@acme, owner)
```

The `GitHub` and `Email` lines are omitted when not set. When the user belongs
to no team, the `Team` line reads `none — create one at https://yard.sh/team to
sell anything`, and every seller command will fail until they do.

**JSON output (`--json`):**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "username": "alice",
  "github_username": "alice",
  "email": "alice@example.com",
  "plan": "Pro",
  "permissions": {
    "create_teams": { "granted": true, "value_type": "boolean" }
  },
  "team": {
    "id": "1f0c8a2e-2a1b-4c3d-9e8f-7a6b5c4d3e2f",
    "name": "Acme Corp",
    "username": "acme",
    "role": "owner"
  },
  "team_permissions": {
    "sell_projects": { "granted": true, "value_type": "boolean" },
    "license_keys": { "granted": true, "value_type": "boolean" },
    "coupons": { "granted": true, "value_type": "boolean" },
    "max_pricing_tiers": {
      "granted": true,
      "limit": 10,
      "value_type": "limit"
    },
    "max_projects": {
      "granted": true,
      "unlimited": true,
      "value_type": "limit"
    }
  }
}
```

`username` is the best available display name (GitHub username → username → email → id), matching `User.DisplayName()`. `plan` is a human label (`Free` / `Basic` / `Pro`) for display only.

There are **two** grant maps, and picking the wrong one gives the wrong answer:

- **`team_permissions` is the source of truth for every seller feature**: projects, tiers, coupons, license keys, custom pages, service, sandboxes, API keys. It is the merged entitlement of the active team's _owners_, which is what the server gates those endpoints on. Read this before proposing any project capability.
- `permissions` is the signed-in user's own entitlement, from their personal billing. It governs account-level things like `create_teams` — not what the team's projects may do.

The two differ routinely: a free user who joins a Pro team can use every Pro feature on that team's projects, and a Pro user acting as a free team cannot. `team` names which team those permissions belong to, and is `null` when the user has no team (in which case `team_permissions` is absent and no seller command will work).

Each entry in either map is a merged grant — booleans carry `granted`; limits also carry `limit` (a number) or `unlimited: true`. Gating is entirely permission-based, so a feature isn't tied to a plan _name_ — a custom role could grant any subset. (The old `is_pro` field was removed; use `team_permissions` or `plan`.)

---

## yard team

Show or switch the team the CLI acts as. **Projects, coupons, affiliate links, payouts and API keys are owned by a team, never by a user**, so every seller command reads and writes the active team's data.

**Usage:** `yard team [--json]` / `yard team use <username>`

**Auth:** required (session auth only — an API key is already pinned to one team).

**Human output:**

```
Team:   Acme Corp (@acme)
Role:   owner
Projects are published at https://yard.sh/@acme/<slug>

Other teams (switch with 'yard team use <username>'):
  @side-project            Side Project
```

With no team, it prints where to create one instead. `--json` emits `{ active_team_id, active_team, teams }`.

### Switching

```bash
yard team use side-project     # the leading @ is optional
```

The active team is stored **on the account, not in `~/.yard/config.json`** — the same setting the dashboard's team switcher writes. So switching in the browser changes what the CLI sees, switching here changes what the dashboard sees, and it follows the user across machines. An agent that has just run `yard team use` should not assume any earlier `yard projects` output is still current.

`use` rejects a username the user isn't a member of and lists the ones they are, rather than silently doing nothing.

### Why a command needs a team

Every seller-scoped endpoint answers `403` with `code: "NO_TEAM"` when the caller belongs to no team. The CLI turns that into instructions to create one — it is **not** a plan problem and upgrading won't fix it. A brand-new account reaches this state if it somehow skipped team creation during signup; the fix is https://yard.sh/team, then `yard team` to confirm.

---

## yard init

Set up a Yard project in the current directory. Interactive flow that links the folder to a Yard project (new or existing) and optionally scaffolds a custom landing page.

**Prerequisites:** none. A Git repo with a GitHub remote enables the "link the repo to a new project" path, but is not required — outside of a git repo, `yard init` still creates projects, they just won't be linked to a GitHub repository.

**Step-by-step flow:**

1. **Update check** — Fetches latest version from `{UpdateURL}/current_version.txt`. If a newer version exists, prompts the user to update. Update is required to continue — if the user declines, `init` aborts.

2. **Login check** — Loads `~/.yard/config.json` and verifies the session is still valid via `GET /v1/me`. If not logged in or session expired, runs the login flow automatically.

3. **Best-effort git context** — All three steps here are non-fatal; on any failure `init` prints a one-line stderr notice and proceeds to create a project without a linked repo.
   - Runs `git rev-parse --show-toplevel` + `git config --get remote.origin.url` and parses owner/repo. SSH and HTTPS GitHub URLs supported.
   - `GET /v1/github/installations` to check the Yard GitHub App. If absent, opens the install page and polls every 2 seconds (5-min timeout).
   - `GET /v1/repos/verify?repo=owner/repo` to confirm access. If the repo is already listed as another project, an informational note is printed but selection is still up to the user.

4. **Project selection prompt** — Calls `GET /v1/projects`. If the user has zero projects, skips straight to the create-new branch. Otherwise prints a numbered list and asks `(n) new project, (1..N) select existing`. Invalid input re-prompts.

5. **Create-new branch** (only when selected):
   - Title prompt (max 60 chars). Defaults to repo name if a git repo was detected.
   - Price prompt in dollars. Minimum $3.00, or $0 for free.
   - `POST /v1/projects`. If git context was fully collected and the repo isn't already listed, the request includes `github_repo_id` + `github_repo_name`; otherwise those fields are omitted (the backend accepts projects with no linked repo).
   - The wizard never blocks a choice by plan. If the assembled project uses something the team's plan doesn't include (e.g. multiple tiers or seat-based pricing), the create call returns `upgrade_required`; the CLI shows the generic upgrade link and then prompts _"Once you've upgraded, press Enter to retry"_ — pressing Enter resubmits the **same** request, so the user upgrades in the browser and retries without re-answering the wizard.

6. **Scaffold `.yard/`** — Writes `./.yard/settings.json` via `EnsureYardProject`. Existing settings are preserved.

7. **Optional landing-page setup** — Prompts `Set up a custom landing page for <slug>? [y/N]`. If yes:
   - Creates `./.yard/landing-page/`.
   - Resolves the project's draft release (same logic as `yard init --page`) and pulls its landing-page files, or scaffolds the hello-world starter when it has none.
   - Uploads the local files into the draft and prints that the page is staged — going live requires publishing the draft (`yard releases publish <tag>`). If the plan doesn't include custom pages, the server returns `upgrade_required`; the CLI prints the `https://yard.sh/pricing` message to stderr and the rest of `init` still succeeds.

8. **Optional project-settings prompts** — After landing-page setup, the wizard always asks (regardless of plan — the server decides, no client-side gate):
   - "Enable license keys? [y/N]" — toggles `license_key_enabled`.
   - If license keys are on: "Enable device activations? [y/N]" → on yes, "Device activation limit (1-10000) [3]:".

   Changes are applied via `PUT /v1/projects/{id}`. If the team's plan doesn't include a setting, the server returns `upgrade_required` and the CLI shows the generic upgrade link — the rest of `init` still succeeds. Spec mode (`yard init --spec`) accepts the same fields directly in the JSON payload (see SKILL.md schema). (Free trials, `trial_requires_card`, and `gift_enabled` are configured **per tier**, not here — see `yard projects tiers`.)

9. **Success output** — Prints the project display name, slug, and the buy / profile URLs (new projects only). If a landing page was set up, also prints the preview URL — and the live URL when publish succeeded.

**JSON output:** the current implementation is interactive-only; there's no `--json` variant yet.

---

## yard projects

List all your published projects. With no subcommand, shows the same table as before; the `edit` subcommand modifies seller settings.

**Output format:**

```
NAME                                PRICE      RELEASES   SALES
----------------------------------------------------------------------
✓ my-awesome-tool                   $9.99      3          12
✗ private-beta                      $29.00     1          0

Total: 2 project(s)
```

- `✓` = publicly visible (the project's global data has visibility `public`), `✗` = not public. Change it with `yard sandbox visibility global <public|private>`
- Names truncated to 32 characters with `...` suffix
- Requires login; prompts to run `yard login` if not authenticated
- `--json` emits the underlying `ProjectListItem` array (including `license_key_enabled`, `activations_enabled`, `max_activations`). **Tiers are not included** — free trials, `trial_requires_card`, and `gift_enabled` are configured per tier, so use `yard projects show <slug> --json` (below) to inspect `tiers[].free_trial_enabled` / `tiers[].trial_requires_card` / `tiers[].gift_enabled`.

**Discovering a project's slug.** The default table shows the _display name_ (title → repo name → slug fallback), not the slug. Every other CLI command that takes `<slug-or-id>` or `--project <slug>` needs the raw slug, which the JSON output carries on `.slug`. Common ways to find it:

```sh
# All slugs the active team owns
yard projects --json | jq -r '.[].slug'

# Slug + title, tab-separated (handy when titles aren't unique)
yard projects --json | jq -r '.[] | "\(.slug)\t\(.title)"'

# Find a slug by partial title match (case-insensitive)
yard projects --json | jq -r '.[] | select(.title | ascii_downcase | contains("simple")) | .slug'

# Slug for the project you're sitting in (set by `yard init` / `yard init --page`)
jq -r .project_slug .yard/settings.json
```

If you only have the UUID, `yard projects show <uuid> --json | jq -r .slug` resolves it. Slugs and UUIDs are interchangeable everywhere `<slug-or-id>` is accepted.

### yard projects show <slug-or-id>

Prints one project's full detail — the same shape the seller dashboard fetches via `GET /v1/projects/{id}`, including `tiers[]` with `pricing_model`, `seat_type`, `features`, `volume_brackets`, and per-tier `free_trial_enabled` / `free_trial_days`.

**Usage:**

```sh
yard projects show my-awesome-tool             # human-readable summary
yard projects show my-awesome-tool --json      # full ProjectDetailResponse on stdout
```

**Typical agent flow** — gating a landing-page trial CTA on whether _any_ tier offers a trial:

```sh
yard projects show my-awesome-tool --json | jq '.tiers[] | select(.free_trial_enabled) | {id, name, free_trial_days}'
```

If the output is empty, no tier offers a trial — the trial button on the landing page should stay hidden.

**Resolution:** same as `yard projects edit` — accepts either a slug or a UUID; resolves slugs through the seller's project list.

### yard projects edit [slug-or-id]

Modify seller settings on an existing project: license keys and device activations (and the per-key limit). Whether the team's plan includes these is enforced by the server, not the CLI.

**Resolution:**

- Pass a slug or UUID as the first argument to target a specific project.
- Without an argument, auto-selects when the user has only one project, prompts to pick from a numbered list otherwise.
- In `--spec` mode with multiple projects and no argument, errors out with the list of slugs.

**Interactive flow:**

1. Prints `Editing settings for <title> (<slug>)`.
2. Same prompt sequence as `yard init`'s settings step (no plan gate — every prompt is shown). Each prompt's default reflects the _current_ value, so pressing Enter is always a no-op.
3. Calls `PUT /v1/projects/{id}` with only the fields that changed.

**When the plan doesn't include a setting:** the CLI still attempts the update; the server returns `upgrade_required` and the CLI prints the server's reason plus the `https://yard.sh/pricing` link. Interactive mode exits 0 (nothing changed); `--spec` mode exits non-zero.

**Spec mode:**

- `--spec <file|->` — read JSON from a file or stdin. The JSON shape is `UpdateProjectRequest`:
  ```json
  {
    "license_key_enabled": true,
    "activations_enabled": true,
    "max_activations": 5
  }
  ```
  Unknown fields are rejected. Missing fields are left untouched (the request is sparse). **Free trials, `trial_requires_card`, and `gift_enabled` are per-tier** — they're not project-level settings; use `yard projects tiers edit` (below) instead.
- A `tiers` array MAY be included to replace the full tier list in one shot — every existing tier with a matching `id` is updated in place, new tiers (no `id`) are inserted, and tiers omitted from the array are deleted (or marked non-default if they have transactions/active subscriptions). Always read the current tiers first (`yard projects show <slug> --json | jq '.tiers'`) and mutate before sending back, otherwise you'll accidentally drop tiers.
- `--json` — emit `{ "project": {...}, "settings": {...} }` on stdout; logs go to stderr.
- The CLI pre-checks the activations-needs-license-keys rule against the _effective_ state, so a spec that flips activations on without restating `license_key_enabled` succeeds when the project already has license keys enabled.
- A 403 with `error_code: "upgrade_required"` is rendered as a clean upgrade message rather than the raw HTTP error.

**Typical agent flow:**

```sh
# Discover what exists.
yard projects --json

# Apply project-level settings to a known project.
yard projects edit my-awesome-tool --spec - --json <<'EOF'
{
  "license_key_enabled": true,
  "activations_enabled": true,
  "max_activations": 5
}
EOF

# Enable a 14-day card-required free trial on the Base tier (per-tier settings).
yard projects tiers edit my-awesome-tool Base --spec - <<'EOF'
{ "free_trial_enabled": true, "free_trial_days": 14, "trial_requires_card": true }
EOF
```

### yard projects tiers

Manage pricing tiers (add, change, remove) without rebuilding the whole tier array yourself. The backend exposes tier mutations only through `PUT /v1/projects/{id}` with a full-replace `tiers[]`; these subcommands do the read-modify-write for you so you don't accidentally drop tiers.

For wholesale changes (multiple tiers at once, reorder, etc.) use `yard projects edit <slug> --spec -` with the full `tiers[]` array directly.

#### yard projects tiers add <project-slug> --spec <file|->

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

How many tiers a project may have depends on the owning team's plan (server-enforced via the `max_pricing_tiers` permission — currently Basic: 2, Pro: 10). The CLI doesn't pre-check; if you exceed the plan's limit the add is rejected with `upgrade_required`. Check the current cap with `yard me --json` → `.team_permissions.max_pricing_tiers`.

#### yard projects tiers edit <project-slug> <tier-id-or-name> --spec <file|->

Apply a partial spec to one tier. Any field present in the spec replaces the current value; absent fields are left untouched. Match by UUID or case-insensitive name (UUID required when two tiers share a name).

```sh
# Enable a 14-day trial on the Base tier
echo '{"free_trial_enabled": true, "free_trial_days": 14}' \
  | yard projects tiers edit simple-note Base --spec -

# Drop a tier's price
echo '{"price_cents": 1500}' \
  | yard projects tiers edit simple-note Base --spec -
```

#### yard projects tiers rm <project-slug> <tier-id-or-name> [--yes] [--promote-default]

Remove a tier. Tiers with paid transactions or active subscriptions cannot be hard-deleted server-side — they are kept but marked non-default so new buyers won't see them. Refuses to remove the only remaining tier.

- `--yes` — skip the confirmation prompt (required in scripts / non-TTY).
- `--promote-default` — when removing the current default tier, auto-promote the first surviving tier to default. Without this flag, the command refuses to leave the project without a default.

All three subcommands accept `--json` to emit the refreshed tier list on stdout.

---

## yard releases

Manage releases for a project. The CLI exposes `publish` and `promote` — list/edit/delete still happen in the dashboard.

**The draft-release model.** A release starts as a **draft**: an ordinary release that has not been published yet and is unreachable by buyers (drafts can never belong to a scope). With no `--release`, every CLI command that writes files (`yard push`, `yard releases publish`) targets the same draft: your open one, or a new draft seeded from your newest published release. Publishing stamps the tag and puts the release in its target channel, and every scope connected to that channel attaches it: attaching is the deploy moment. Every published release belongs to exactly one channel; a draft belongs to none, which is what makes it a draft. A scope holds a **set** of releases and serves the newest member unless pinned.

**Published releases stay editable.** Publishing is one-way, but the release it produces is not frozen: `--release <id|tag>` names any release, draft or published, and `yard push` edits it in place. Editing a release nothing serves has no deploy side effects; editing one a scope is serving is live the moment it saves, which is why `yard push` names the scopes a release belongs to and asks before uploading (`--yes` skips the prompt).

**`--release` takes a tag or a UUID.** `--release v1.4.0` and `--release <release-uuid>` resolve to the same release; the CLI picks the right lookup from the shape of the value.

### yard releases publish [tag]

Publish a draft release under a tag, with optional file assets. Files upload into the draft first, then the draft is published and deployed — so a single failed asset never leaves a half-described release, and anything `yard push` already staged in the draft, landing page and service bundle alike, ships with it.

**Flags:**

- `[tag]` positional — the tag name (e.g. `v1.4.0`). Required in `--spec` mode (read from spec); optional and prompted in interactive mode. Tags are unique per project across non-archived releases — a collision is a `409`.
- `--project <slug-or-uuid>` — target project. Required only if you have multiple projects.
- `--name <string>` — optional human-readable release name.
- `--notes <string>` — short release notes (markdown).
- `--notes-file <path|->` — read notes from a file or stdin.
- `--file <path>` — file to upload, repeatable (`--file a.zip --file b.zip`).
- `--channel <name>`: the one release channel the release lands in. Defaults to **Production**, the protected channel every project has and the one the project's global data follows, so a release published without this flag is live to customers. Every scope connected to that channel attaches the release and deploys it. `yard channels list` shows the project's channels and which scopes follow each one; a name that doesn't exist is rejected before anything is published. Channels are created in the dashboard, not from the CLI, so a custom `--channel` target has to exist first.
- `--release <id|tag>`: the draft to publish. Defaults to your open draft (creating one seeded from your newest published release if none exists); required when multiple drafts are open. Must still be a draft: publishing is one-way, so an already-published release is refused here (edit it with `yard push --release <id|tag>` instead).
- `--spec <path|->` — JSON spec, alternative to flags.
- `--json` — emit a single JSON result on stdout; logs go to stderr.

**Spec JSON shape:**

```jsonc
{
  "project": "my-slug", // optional if user has only one project
  "tag_name": "v1.4.0", // required
  "release_name": "Late April fixes", // optional, ≤255 chars
  "release_notes": "## Highlights\n…", // optional, markdown, ≤125,000 chars
  "channel": "Production", // optional; defaults to "Production", live to customers
  "files": [
    // optional; absolute or relative paths
    "./dist/yard-darwin-arm64.tar.gz",
    "./dist/yard-linux-amd64.tar.gz",
  ],
}
```

Unknown fields are rejected. Each file path must exist and be a regular file.

**Two-step publish flow:**

1. The draft is resolved (`--release` → sole open draft → new draft seeded from your newest published release), and each `files[]` entry streams into it as `multipart/form-data`.
2. The draft is published (`POST /v1/projects/{id}/project-releases/{rid}/publish`): the tag is stamped, the release lands in its target channel, and every scope following that channel attaches and deploys it.
3. The CLI prints `✓ <path>` or `✗ <path>: <error>` per file on stderr, then a summary like `Release "v1.4.0" published. Uploaded 2/3 file(s).`
4. Exit code is non-zero if any uploads failed; if at least one file uploaded the draft was still published (without the failed files), and missing assets can be added from the dashboard. If every file failed, nothing is published and the draft stays open.

**`--json` output:**

```json
{
  "release":  { "id":"…", "version":"v1.4.0", "is_draft":false, "published_at":"…", "files":[...], ... },
  "deployed": [
    {"release_id":"…", "version":"v1.4.0", "to":"", "action":"attach", "artifacts":["page","releases"]}
  ],
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

Move an already-published release into a **release channel**, out of whichever one it is in. Every scope connected to the new channel attaches it and then serves exactly what that release holds: landing page, pricing, download buttons, service bundle, and downloadable files; scopes connected to the old channel detach it and fall back to their next newest release. A channel is a project-wide category of releases, not a scope: `Production` is the protected one every project has, and a new project's global data follows it.

**Flags:**

- `<tag>` positional — the tag to promote, e.g. `v1.4.0`. There is no `--from`: releases belong to the project, so the tag alone identifies one.
- `--to <channel>`: target channel. Required. It has to already exist: channels are created in the dashboard, and the CLI only reads them (`yard channels list`). The one exception is `Production`, which every project has.
- `--project <slug-or-uuid>` — target project.
- `--dir <path>` — directory containing `.yard/`.
- `--json` — emit a single JSON result on stdout.

```sh
yard releases publish v1.4.0 --file dist/app.zip   # publishes into the Production channel
# …verify the download works…
yard releases promote v1.4.0 --to Beta             # move it into Beta, out of Production
```

**Nothing is copied**: a release is one project-wide snapshot, and it belongs to exactly one channel. Promoting moves it, so the channel it came from no longer has it. Each scope connected to the new channel takes it into its own set, where as the newest member it starts serving (unless that scope is pinned to another release). Storage is not consumed twice and download counts carry over. Promoting a release into the channel it is already in is a no-op. Because the scopes now serve the same release, a later edit to it shows up in all of them. To attach a release to one scope by hand, use `yard sandbox attach` or `yard sandbox deploy`.

**`--json` output** (the channel-membership result, plus what it deployed):

```json
{"channel": "Beta", "deployed": [{"release_id": "…", "version": "v1.4.0", "to": "beta", "action": "attach", "artifacts": ["pricing", "identity", "page", "service", "releases"]}]}
```

Each `deployed` entry's `to` is the scope the mirror shipped to: `""` is the project's global data, any other value is a sandbox slug.

For full download server schemas (license-key path and API-key path), see [references/releases-and-updates.md](releases-and-updates.md).

---

## yard channels

Read the project's **release channels**. A channel is a project-wide category of
releases: every published release belongs to exactly one, and a draft belongs to
none, which is what makes it a draft. Scopes connect to a channel and mirror its
membership, so a channel is how a release reaches several scopes at once without
naming any of them.

Every project has a protected `Production` channel, created with the project and
impossible to rename or delete. Publishing and GitHub release sync default to it,
and a new project's global data follows it, which is why `yard releases publish
<tag>` goes live.

**The CLI only reads channels.** Creating, renaming, deleting and reordering them
is done from the dashboard (Releases page). A name that doesn't exist yet is
rejected wherever it is used (`yard releases publish --channel`, `yard releases
promote --to`, `yard sandbox channel`), so create the channel in the dashboard
before scripting against it. The commands that *do* move releases and scopes
between channels are `yard releases promote` and `yard sandbox channel`.

### yard channels list

**Flags:** `--json`, `--project <slug-or-uuid>`, `--dir <path>`.

```
CHANNEL              RELEASES  LATEST                   SCOPES                         PROTECTED
-----------------------------------------------------------------------------------------------
Production           7         v1.4.0                   global                         yes
Beta                 2         v1.5.0-beta.1            preview, staging
```

- `RELEASES` counts every release the channel holds, archived ones included.
- `LATEST` is the narrower question: the newest published, non-archived, healthy
  release, the one a scope connecting to this channel would start serving. `-`
  when the channel has nothing to serve.
- `SCOPES` lists the scopes following the channel, the project's global data
  first as `global`; `-` when nothing follows it.

`--json` emits `{ "project": "<slug>", "channels": [...] }`, each channel with
`id`, `name`, `protected`, `position`, `created_at`, `release_count`,
`latest_version`, and `scopes` (an array of scope names, `""` for the global
scope).

---

## yard keys

Manage API keys for programmatic access. Without a subcommand, runs `keys list`.

Keys belong to the **active team**, not to the person who created them: anyone on the team can use one, and it keeps working after that person leaves. Confirm which team you're minting into with `yard team` first — a key created under the wrong team authenticates as the wrong team.

### yard keys list

Lists the **active team's** API keys with the same columns the dashboard shows (name, prefix, scopes, last-used, created). The full secret is never displayed — only the prefix `yard_xxxxxxx`. Switching teams (`yard team use`) changes what this lists.

**Flags:**

- `--json` — emit the raw `APIKeyListResponse` JSON on stdout. The `key` (full secret) is **never** present here.
- `--sort <col>` — `created_at` (default), `name`, or `last_used_at`.
- `--direction <asc|desc>` — sort direction.

**Table output:**

```
NAME                     PREFIX             SCOPES                                   LAST USED      CREATED
--------------------------------------------------------------------------------------------------------------
ci-runner                yard_a1b2c3d       licenses:validate, licenses:activate     2 hours ago    2026-04-12
local-dev                yard_e5f6789       projects:read                            never          2026-03-30

Total: 2 / 100 keys
```

### yard keys create [name]

Mints a new API key **for the active team**. **The full secret is shown only once at creation time.** After that, only the prefix is recoverable. The key limit is per team, so `403 … maximum` counts the team's keys, not yours.

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
| `projects:read`       | Read project metadata                                 |
| `licenses:validate`   | Validate license keys (called from your own software) |
| `licenses:activate`   | Activate / deactivate license keys                    |
| `subscriptions:read`  | Read project subscription status                      |
| `subscriptions:write` | Create / cancel / reactivate project subscriptions    |

Backend caps each user at 100 API keys; on `403` from the create endpoint the CLI prints the reached-limit message.

**Typical agent flow:**

```sh
# Mint a key for a CI job, capture the secret out of the JSON output.
KEY=$(echo '{"name":"ci-runner","scopes":["licenses:validate","licenses:activate"]}' \
  | yard keys create --spec - --json | jq -r .key)
```

For end-user-shipped software, downloads authenticate with license keys against the update server (`/v1/updates/latest`) — see [references/releases-and-updates.md](releases-and-updates.md).

---

## Testing license keys

There is no `yard licenses` command group: license-key testing happens in a **sandbox** rather than through a per-project test key.

License-key settings are per scope. A sandbox carries its own `license_key_enabled`, `activations_enabled` and `max_activations`, copied from the project's global scope when the sandbox is created and diverging from the next edit. Buying inside a sandbox is simulated - no card, no money - and mints a **real** license key scoped to that sandbox, so validation, device activation and the update server all exercise the same code path a buyer's key does.

```sh
# A scope to rehearse in. It inherits the project's licensing settings.
yard sandbox create staging

# Mint an API key for the validate endpoint (the license key goes in the body,
# this goes in the Authorization header).
yard keys create --spec - --json <<<'{"name":"local-validate","scopes":["licenses:validate"]}'

# Then buy the project inside the sandbox from its checkout page, and validate
# the key it mints. The response's `sandbox` field reads "staging".
curl -X POST https://api.yard.sh/v1/licenses/validate \
  -H "Authorization: Bearer $YARD_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"license_key":"XXXX-XXXX-XXXX-XXXX","device_id":"laptop-42"}'

# A clean slate is a fresh sandbox: deleting one takes its transactions,
# license keys and device activations with it.
yard sandbox delete staging --yes
```

A sandbox's own licensing settings and the keys it has minted are read and edited from that sandbox's License Keys page in the dashboard; `yard projects edit` reaches the **global** scope's settings only.

---

## yard coupons

Manage the discount codes buyers redeem at checkout. Without a subcommand, runs `coupons list`.

Coupons need the `coupons` permission on the selling team's plan — check `yard me --json` → `.team_permissions.coupons`. There is no client-side gate: the server answers `403` and the CLI prints what the plan is missing plus the pricing link.

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
LAUNCH20               20%        all projects       3 / 100          active     2026-12-31
INFL7K2M               15%        2 project(s)       0 / unlimited    scheduled  —

2 coupons (1 active), 3 redemptions, $15.00 saved for buyers
```

`STATUS` is derived, not stored: `inactive` (disabled), `expired`, `scheduled` (`valid_from` in the future), `used up` (hit `max_uses`), or `active`. `--json` emits the raw `CouponListResponse`, where `is_active` is only the on/off toggle — compute the rest from `expires_at`, `valid_from`, and `current_uses` if you need it.

### yard coupons show \<code-or-id\>

Coupon detail plus redemption analytics. `--json` emits one object: `{ "coupon": {...}, "analytics": {...} }`.

### yard coupons create \<code\>

**Flags:** `--percent N` | `--amount D`, `--projects <csv>`, `--max-uses N`, `--expires <date>`, `--valid-from <date>`, `--subscription-duration <once|forever>`, `--spec <file|->`, `--json`.

The code is upper-cased and must be 4-50 alphanumeric characters. `--projects` takes slugs **or** UUIDs and implies `scope=specific_projects`; without it a coupon applies to everything the seller sells, including projects created later.

`--subscription-duration` only matters for subscription projects: `once` discounts the first payment (default), `forever` discounts every renewal.

**Spec shape:**

```jsonc
{
  "discount_type":         "percentage",           // or "fixed_amount"
  "discount_value":        20,                     // percent, or CENTS for fixed_amount
  "scope":                 "all_projects",         // or "specific_projects"
  "project_ids":           ["<uuid>"],             // required for specific_projects
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

Also accepts `--activate` / `--deactivate`, `--scope <all_projects|specific_projects>`, and every `create` flag. `--projects` alone re-scopes a coupon; no need to resend `--scope`.

In a spec, `null` clears the same three fields — `{"expires_at": null}` — while omitting a key leaves it alone.

The server refuses to change `discount_type` / `discount_value` on a coupon that has already been redeemed. Everything else stays editable.

### yard coupons rm \<code-or-id\>

Deletes a coupon. A coupon that has been redeemed **cannot** be deleted (its redemptions are transaction history) — use `--deactivate` instead. Requires `--yes` when stdin isn't a TTY.

### yard coupons transactions \<code-or-id\>

The purchases a coupon was redeemed on. **Flags:** `--json`, `--page N`, `--limit N`.

### yard coupons validate \<code\>

Runs the code through the same check the checkout page performs — active, in date, under its limit, applicable to this project — and reports what the buyer would pay. **Flags:** `--project <slug>`, `--tier <uuid>`, `--team <username>` (the team that owns the project; defaults to your active team), `--json`.

The project is resolved under the team's username, so validating a coupon on another team's public project means naming it: `--team acme`.

The project must be **public** (the visibility of its global data): checkout never sees drafts or private projects, so either answers `PROJECT_NOT_FOUND`. Exit status is 0 whenever the check ran, so read `.valid` for the answer.

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
yard coupons validate LAUNCH20 --project my-tool --json | jq '{valid, final_price_cents}'
```

---

## yard customers

Read-only view of the people who bought the seller's projects. Without a subcommand, runs `customers list`.

A "customer" is a buyer with at least one **completed, unrefunded** purchase, aggregated across the seller's whole catalog — a buyer whose only order was refunded does not appear. With `--project`, the aggregation narrows to that one project: the rows are its buyers, and each customer's order count and spend cover only their orders of it.

Buyers are identified by an opaque id: `cust_` plus the first 8 characters of their account id. That id is what `customers show` takes; there is no email lookup.

Money comes back **pre-formatted** (`"total_spent_display": "$87.00"`). There is no cents field on these responses — use `yard transactions` when you need to do arithmetic.

**Sandbox commerce is not here.** These rows are the seller's real books: simulated purchases made inside a sandbox are excluded from the list and from the summary figures, and there is no `--sandbox` flag to reach them. A sandbox's customers live on that sandbox's pages in the dashboard. See [pricing-and-licensing.md](pricing-and-licensing.md#commerce-in-a-sandbox).


### yard customers list

**Flags:** `--json`, `--sort <col>`, `--direction <asc|desc>`, `--project <slug-or-uuid>`, `--page N`, `--limit N` (max 100).

Sort columns: `lastTransaction` (default), `email`, `username`, `orderCount`, `totalSpent`, `buyerId`.

```
ID               CUSTOMER                       ORDERS   SPENT        FIRST        LAST
--------------------------------------------------------------------------------------
cust_deadbeef    @alice                         3        $87.00       2026-01-04   2026-07-20
cust_c0ffee11    bob@example.com                1        $29.00       2026-07-18   2026-07-18

2 customers (1 new this month), $58.00 avg spend, 50.0% repeat rate
```

The summary line doesn't move with pagination. It is team-wide by default, and project-wide under `--project`.

Unlike `yard transactions --project` — where the filters narrow the rows but the earnings summary stays team-wide — `yard customers --project` narrows the summary too, because "customers of this project" is a different set from "customers", not a filtered view of it.

### yard customers show \<customer-id\>

One buyer's totals plus their orders from this seller. **Flags:** `--json`, `--page N`, `--limit N`.

`--json` emits `{ "customer": {...}, "transactions": [...], "total", "page", "limit" }`. Unlike the list, the transactions here **include refunded orders**.

Two customers can share an 8-character id prefix. The server answers `409` rather than guessing; pass the buyer's full UUID in that case.

**Typical agent flows:**

```sh
# Biggest spenders
yard customers --json | jq -r '.customers | sort_by(.order_count) | reverse | .[:5] | .[] | "\(.email) \(.total_spent_display)"'

# Everything one buyer owns
yard customers show cust_deadbeef --json | jq -r '.transactions[] | "\(.project_name) \(.created_at)"'

# Who bought one specific project, biggest spenders first
yard customers --project my-tool --sort totalSpent --direction desc --json | jq -r '.customers[] | "\(.email) \(.total_spent_display)"'

# Who bought in the last month
yard customers --json | jq -r --arg since "$(date -u -d '30 days ago' +%Y-%m-%d)" \
  '.customers[] | select(.last_transaction >= $since) | .email'
```

---

## yard transactions

The seller's sales, plus the one write in this area: adjusting a buyer's free-trial length. Without a subcommand, runs `transactions list`.

Every sale has a full UUID and a short **order id** — `order_` plus the first 8 characters, which is what the dashboard and receipts show. Both are accepted anywhere a transaction id is taken; an ambiguous short id is answered with `409` rather than a guess.

Refunds are **not** exposed here — issuing one stays in the dashboard.

**Sandbox commerce is not here.** Simulated transactions made inside a sandbox never appear in this list, in the summary figures, or in earnings and payouts, and there is no `--sandbox` flag to reach them. A sandbox's transactions live on that sandbox's pages in the dashboard. See [pricing-and-licensing.md](pricing-and-licensing.md#commerce-in-a-sandbox).


### yard transactions list

**Flags:** `--json`, `--trials`, `--project <slug-or-id>`, `--start <date>`, `--end <date>`, `--sort <col>`, `--direction <asc|desc>`, `--page N`, `--limit N` (max 100).

Sort columns: `date` (default), `amount`, `sellerEarnings`, `projectName`. Dates take `YYYY-MM-DD` or RFC3339; a bare `--end` date covers that whole day.

```
ORDER            DATE         PROJECT                    CUSTOMER              TOTAL       EARNINGS    TYPE           STATUS
-----------------------------------------------------------------------------------------------------------------------------
order_1a2b3c4d   2026-07-20   My Tool                    @alice                $29.99      $26.99      purchase       completed
order_9f8e7d6c   2026-07-18   My Tool                    bob@example.com       $0.00       $0.00       trial          completed

2 of 97 transactions · $2699.00 earned, 97 sales, $27.81 avg order, 3 active trials
```

`--trials` and the date/project filters narrow the rows **and** the total; the summary figures stay team-wide, matching the dashboard. `TYPE` is derived: `gift`, `trial`, `trial upgrade`, `subscription`, or `purchase`. `STATUS` shows the refund state when there is one, because a refunded sale still has `status: "completed"`.

### yard transactions show \<order-id\>

One sale in full: tier, quantity, coupon, refund date, billing period, and — for a trial — when it runs out and how long that is from now. **Flags:** `--json`.

### yard transactions trial \<order-id\> --add-days N

Lengthen or shorten a live free trial. `--add-days 7` gives a week; `--add-days -3` takes three days back. Non-zero, and 365 either way. **Flags:** `--add-days N` (required), `--json`.

Two behaviours to state plainly before running it:

- **Days are added to the trial's current expiry, not to today.** Extending a trial that expired a month ago by 7 days still leaves it in the past — add enough days to land in the future. When the new expiry *is* in the future, an expired trial is set back to `active` and the buyer has access again; the response reports this as `"reactivated": true`. If the trial stays expired, the CLI says why.
- **The buyer is emailed** about the change, same as adjusting it from the dashboard.

A revival is refused in one case: the buyer already has another pending or active trial on that project. The expiry still moves, `reactivated` comes back `false`, and the status stays `expired`.

Only trial transactions have a length to adjust; anything else is rejected before the request is made. This is the *running* trial for one buyer — the trial length offered to **new** buyers is the per-tier `free_trial_days` setting, changed with `yard projects tiers edit`.

Requires a plan that can sell projects (`yard me --json` → `.team_permissions.sell_projects`).

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

# What one project earned this month
yard transactions list --project my-tool --start 2026-07-01 --json \
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

## yard push / pull / status / ls

The project sync commands. One set covers **everything** a project sends to
Yard: the landing page in its directory (`landing_page.dir` in
`.yard/settings.json`, default `.yard/landing-page/`), one bundle per service
listed in `services` (each carrying its own `settings.json`, which is how
deploys read that service's `name`/`url`/`access`/`database`, so a
service-settings change is an edit in that file plus a push), and
`.yard/settings.json` itself (the `config` bundle). A project with only some
of the bundles simply syncs what it has.
The config bundle is **push-only**: `yard pull` never overwrites your local
settings.json, since it is the project's identity.

Common flags:

- `--project <slug-or-uuid>` — identify the project explicitly (otherwise read from `.yard/settings.json`; if that's missing and the user has exactly one project, it's auto-selected)
- `--dir <path>` — directory containing `.yard/` (defaults to walking up from cwd)
- `--release <id|tag>` — which release to act on (defaults to your open draft)
- `--json` — emit a single machine-readable JSON object on stdout; logs go to stderr
- `--yes`, `-y`: skip confirmation prompts (`push --prune`, and pushing into a release attached to a scope)

**Everything targets a release, never a scope.** With no `--release`,
writes go to a draft (your open one, or a new draft seeded from your newest
published release when none exists) and reads (`status`, `ls`, `pull`) default
to that same draft, which keeps `yard status` describing exactly what `yard
push` would change. With multiple drafts open, `--release` is required.
`--release <id|tag>` names any release explicitly, published ones included:
reads inspect it, writes edit it in place. Editing a draft touches no live
scope, since content goes live when the draft is published (`yard releases
publish <tag>`), which attaches it to every scope following a target channel.
Editing a release a scope is already serving is live on save.

**Exit codes:**

- `0` — success
- `1` — fatal error (auth, network, validation, API error)
- `2` — partial success (only produced by `push` when some files uploaded and others failed)

**Project discovery:** every project command walks upward from cwd looking for `.yard/settings.json` (same mechanism as `git`'s `.git` lookup). That directory is the working directory the command runs against. The CLI's own config dir at `~/.yard/` is not matched because it only holds `config.json`, never `settings.json`. `--dir <path>` overrides this.

**Landing-page constraints enforced client-side before any HTTP:**

- Extension must be in `.html .css .js .json .svg .png .jpg .jpeg .webp .gif .woff2`
- Path must match `^[a-zA-Z0-9][a-zA-Z0-9._-]*(/[a-zA-Z0-9][a-zA-Z0-9._-]*)?$` — at most one subdirectory level, no dotfiles, no leading slash, no `..`
- Per-file ≤ 1 MB, bundle ≤ 5 MB total, ≤ 20 files
- `index.html` required when publishing

**Service-bundle constraints:** ≤200 files, ≤5 MB per file, ≤25 MB total, paths nest
≤8 levels, extensions in `.html .css .js .mjs .json .svg .png .jpg .jpeg .webp
.gif .woff2 .woff .ttf .otf .txt .md .ico .map .wasm .webmanifest` (plus
`_worker.js`, `migrations/*.sql`). `_worker.js` is required. Dotfiles and
legacy bundle-root local-dev files (e.g. `README.md`, the retired `yard.json`
manifest) are skipped and reported.

---

### yard init --page

Scaffold `.yard/landing-page/` inside an existing Yard project (run `yard init` first). If the resolved draft release already has landing-page files, they are pulled into `.yard/landing-page/`; otherwise a hello-world `index.html` + `styles.css` is scaffolded.

**Behavior:**

1. Resolves the project (flag → existing settings → sole project → error).
2. Creates the landing-page directory (`landing_page.dir` in settings, default `<project>/.yard/landing-page/`) and writes `<project>/.yard/settings.json` if absent: `{"version": 5, "project_slug": "<slug>", "ignore_files": []}`.
3. Resolves the draft release (your open draft, or a new one seeded from your newest published release — a first init on a fresh project starts from what shipped rather than from a blank page).
4. If the draft has landing-page files, pulls them; else writes the hello-world starter (`index.html` + `styles.css`).
5. Local files that already match the remote SHA-256 are skipped.
6. Prints next steps (edit → `yard push` → `yard releases publish <tag>`) and the preview URL.

**Idempotent:** running `init` twice is safe. Existing files are preserved.

**JSON output (`--json`):**

```json
{
  "project_root": "/home/alice/my-landing",
  "project": "my-slug",
  "source": "draft 9f3e1c2a",
  "release": "9f3e1c2a-…",
  "written": ["index.html", "styles.css"],
  "skipped": [],
  "preview_url": "https://yard.sh/dashboard/projects/my-slug/landing-page",
  "live_url": null
}
```

`source` is `"starter"` when the draft had no page files and the starter was written; otherwise a label for the release it pulled from (its tag, or `draft <short-id>` for an untagged draft). `live_url` is only non-null when the project's global data actually serves a release.

---

### yard status

Print the diff between local files and a release without writing anything, per
bundle — the exact set `yard push` would upload. Defaults to your open draft.

Also lists the scopes serving that release, with each one's deploy
status (`up_to_date`, `stale`, `updating`, `failed`), so an agent can tell
whether a redeploy is still catching up. `global` names the project's global
data; anything else is a sandbox.

**Per-bundle categories:**

- `to_upload` — files changed or missing in the release
- `unchanged` — local and release hashes match
- `remote_only` — files in the release but not locally (deleted only by `push --prune`)

**JSON output:**

```json
{
  "project": "my-slug",
  "release": "9f3e1c2a-…",
  "version": "",
  "draft": true,
  "page": {
    "dir": "/home/alice/proj/.yard/landing-page",
    "to_upload": ["index.html"],
    "unchanged": ["styles.css"],
    "remote_only": ["old.html"]
  },
  "service": {
    "dir": "/home/alice/proj/service",
    "to_upload": [],
    "unchanged": ["_worker.js"],
    "remote_only": []
  },
  "serving": [
    {"scope": "global", "deploy": "stale"}
  ]
}
```

A bundle the project doesn't have is absent from the output.

---

### yard ls

List a release's files, grouped by bundle. Defaults to your open draft;
`--release v1.4.0` lists a specific release.

**JSON output:**

```json
{
  "project": "my-slug",
  "release": "9f3e1c2a-…",
  "version": "",
  "page": [
    {
      "id": "…",
      "artifact": "page",
      "path": "index.html",
      "content_type": "text/html",
      "size_bytes": 412,
      "content_hash": "…"
    }
  ],
  "service": [
    {
      "id": "…",
      "artifact": "service",
      "path": "_worker.js",
      "content_type": "application/javascript; charset=utf-8",
      "size_bytes": 2048,
      "content_hash": "…"
    }
  ]
}
```

---

### yard push

Hash-diff every local bundle against a release and upload what changed. A push
writes to a draft — your open one, or a new draft seeded from your newest
published release.

**Flags:**

- `--prune` — delete release files that are not present locally (opt-in for safety)
- `--release <id|tag>` — release to push into (defaults to your open draft)
- `--yes`, `-y` — skip prune confirmation prompt

**Behavior:**

1. Walk each bundle directory that exists, skipping dotfiles and (for the landing page) `ignore_files` patterns from `settings.json`.
2. Validate every bundle client-side. Validation happens for all bundles before any upload, so a bad file never leaves the release half-updated.
3. Compare each local file's SHA-256 against the release's; upload the ones that differ (`PUT …/custom-page/files/{path}?release=` and `PUT …/services/{name}/files/{path}?release=`).
4. If `--prune`, `DELETE` release files not present locally (one confirmation covering every bundle, unless `--yes` or `--json`).
5. Print `Preview:`. Going live is a separate step: `yard releases publish <tag>`.

Pushing `settings.json` with a `pricing` section also **applies** it: the
draft's pricing tiers are replaced to match `pricing.tiers` exactly (invalid
pricing is a 400 naming the tier/field, before anything uploads). GitHub
release sync reads the same section the same way — see
[releases-and-updates.md](releases-and-updates.md) — _Syncing releases from
GitHub_.

**JSON output:**

```json
{
  "project": "my-slug",
  "release": "9f3e1c2a-…",
  "version": "",
  "page": {
    "dir": "/home/alice/proj/.yard/landing-page",
    "uploaded": ["index.html", "styles.css"],
    "skipped": ["logo.png"],
    "deleted": [],
    "remote_only": []
  },
  "services": {
    "api": {
      "dir": "/home/alice/proj/api",
      "uploaded": ["_worker.js"],
      "skipped": ["settings.json"],
      "deleted": [],
      "remote_only": []
    }
  },
  "preview_url": "https://yard.sh/dashboard/projects/my-slug/releases/9f3e1c2a-…/landing-page",
  "live_url": null,
  "errors": []
}
```

A bundle the project doesn't have is simply absent from the output; every
service is keyed by name under `services`. When
`--prune` is not passed, `remote_only` lists paths in the release that don't
exist locally (informational); with `--prune` it is emitted empty and the
removed paths appear in `deleted`.

**Exit code 2** is returned when at least one file uploaded and at least one failed. Use `errors` in the JSON output to disambiguate.

---

### yard pull

Download a release's files into the project — the landing page into
`.yard/landing-page/` and each service into its recorded bundle dir.
Defaults to your open draft; pass `--release <id|tag>` for a specific release.

**Flags:**

- `--force` — overwrite local files even if their contents match

**Behavior:**

1. Resolves project root (with `.yard/` discovery allowed to be missing — `pull` is useful for bootstrapping a fresh checkout).
2. For each bundle, downloads and writes each file unless the local file already has the same SHA-256 (skipped). Subdirectories are created as needed.
3. The landing-page directory is created on demand; a service directory is not invented, since it belongs to the seller's build.

**JSON output:**

```json
{
  "project": "my-slug",
  "release": "9f3e1c2a-…",
  "version": "",
  "page": {
    "destination": "/home/alice/proj/.yard/landing-page",
    "written": ["index.html"],
    "skipped": ["styles.css"]
  }
}
```

---

There is **no publish flag on `push`**: everything ships inside a release, so
going live is `yard releases publish <tag>` (or `yard sandbox deploy global
<tag>` for a release that is already published). To discard draft
changes, delete the draft from the dashboard and pull afresh
(`yard pull --release <last-published-tag> --force`). To correct something
already shipped, either publish a new release or edit the published one in
place with `yard push --release <tag>`; the latter is live immediately if a
scope is serving it.

---

## yard sandbox

Manage a project's scopes and choose what each one serves.

A **release** is one project-wide snapshot: landing page, pricing, downloads,
and the service bundle. A **scope** is a *set* of releases serving one of
them: the newest member, unless a pointer says otherwise. Attaching a release
to a scope is the deploy moment. Every project has its **global data**, which
is what buyers reach at the project's plain URL; a **sandbox** is an optional
extra scope of your own, private by default, at `/@<sandbox>/`. A project
starts with zero sandboxes. Commands that take a `<scope>` accept the literal
`global` for the project's own data; anything else names a sandbox.


A sandbox is not only a deployment target: it carries its own **customers,
transactions, subscriptions, trials and license keys**, all simulated by the
platform rather than run through Stripe. That commerce is reachable from the
sandbox's pages in the dashboard and from the buyer-facing endpoints with a
`sandbox` parameter; the CLI does not read it, and it is excluded from
`yard transactions`, `yard customers`, earnings and payouts. See
[pricing-and-licensing.md](pricing-and-licensing.md#commerce-in-a-sandbox).

A scope is always in one of three serving states, and the commands below
are how it moves between them:

| State | Reached by | Next attach |
|---|---|---|
| newest-wins | `sandbox unpin`, or `sandbox attach` of the newest release | takes over |
| deployed | `sandbox deploy` | takes over |
| pinned | `sandbox pin` | joins the set, does **not** take over |

Shared flags: `--project <slug-or-uuid>`, `--dir <path>`, `--json`. Every
`<release>` argument accepts a version tag or a release UUID. Sandbox
mutations need the `sandboxes` permission, check `yard me --json` →
`.team_permissions`; writes to the global scope need only the ordinary
project-write permission.

### yard sandbox list

Lists the project's scopes with what each serves and why, its global data
first.

```
SCOPE                SERVING                  RELEASES  CHANNEL        VISIBILITY  DEPLOY
-----------------------------------------------------------------------------------------------
global               1.4.0 (pinned)           3         Production     public
staging              1.3.0 (deployed)         2         -              private     stale
preview              -                        0         -              public
```

`DEPLOY` is blank when the service is up to date; a `failed` scope gets a
warning line after the table with its error. JSON:

```json
{
  "project": "my-slug",
  "global": {
    "slug": "", "visibility": "public",
    "active_release_id": "<uuid>", "pinned": true,
    "created_at": "…", "deploy_status": "up_to_date",
    "channel": "Production",
    "page_url": "https://acme.yard.sh/my-slug/",
    "serving_release": { "id": "<uuid>", "version": "v1.4.0", "published_at": "…" },
    "releases": [
      { "id": "<uuid>", "version": "v1.4.0", "published_at": "…", "is_archived": false, "attached_at": "…" }
    ]
  },
  "sandboxes": [
    {
      "id": "<uuid>", "project_id": "<uuid>", "slug": "staging",
      "visibility": "private", "active_release_id": "<uuid>", "pinned": false,
      "created_at": "…", "deploy_status": "stale",
      "page_url": "https://acme.yard.sh/my-slug/@staging/",
      "serving_release": { "id": "<uuid>", "version": "v1.3.0", "published_at": "…" },
      "releases": [ /* … */ ]
    }
  ]
}
```

The global scope has no id and no name: on the wire it is the empty slug, and
an absent `?sandbox=` parameter names it.

Read the serving state from two fields: `active_release_id` null means
newest-wins; set with `pinned: false` means deployed; set with `pinned: true`
means pinned. `serving_release` is what the scope serves right now (absent when
nothing has been deployed), and `releases` is the membership set, newest first.

`deploy_status` reports how the scope's running service compares to that
release: `up_to_date`, `stale` (the release changed and the redeploy hasn't
finished), `updating`, or `failed` (with `deploy_error` saying why; the
scope keeps serving what it had). `deploy_synced_at` is when the last
redeploy settled as up to date. Yard drives this itself: editing a release a
scope serves triggers exactly one redeploy, however many files changed.

### yard sandbox create \<sandbox\>

Creates a sandbox. Names are 2-60 characters: letters, digits, and hyphens,
starting with a letter, compared case-insensitively. How many sandboxes a
project may have is server-enforced via `max_sandboxes` (currently Basic: 1,
Pro: 10); over the cap the API answers `403` with error code
`sandbox_limit_reached`. A new sandbox is private and holds nothing until you
attach a release.

### yard sandbox rename \<sandbox\> \<new-name\>

Renames in place. Only the name moves: releases, files, secrets and the database
follow it, because the sandbox keeps its id. Its URLs change with the name,
since the `/@<sandbox>/` segment *is* the name, so anything pointing at the old
one stops resolving. An existing name conflicts (409). The project's global data
has no name to change.

### yard sandbox visibility \<scope\> \<public|private\>

Sets who may view the scope's URLs. `private` (the default for a sandbox)
admits every member of the owning team, `owner` and `admin` alike, and nobody
else: the edge sends everyone else through sign-in.
`public` means anyone with the URL can view the scope's landing page and
service, with no sign-in and no purchase. `global` works too, and is how a
project goes private: the visibility of its global data is the storefront's.
The project's stage still trumps, so a draft project serves nothing publicly
regardless. The seller gets an in-app + email notification on every flip.

### yard sandbox delete \<sandbox\> [-y]

Deletes the sandbox and everything scoped to it: its files, its service,
its service database, and its whole simulated commerce chain: transactions,
subscriptions, trials, license keys, device activations, coupon usages, gifts
and affiliate commissions (immediately, with no grace window). Releases are
project-scoped, so they survive, and the project's real books are untouched.
Prompts first; pass `-y`/`--yes` in scripts.

### yard sandbox channel \<scope\> \<channel|none\>

Connects the scope to a release channel, so it mirrors that channel's
membership: releases moved into the channel attach and deploy here
automatically, and releases removed from it detach. Manual attaches are left
alone, and a pin still freezes what serves. `none` disconnects the scope and
leaves its releases attached. A new project's global data follows the
protected `Production` channel, which is why publishing goes live. See
`yard channels list` for the project's channels; a channel other than
`Production` has to be created in the dashboard before a scope can follow it.

### yard sandbox attach \<scope\> \<release\> [--no-serve]

Adds a release to the scope's set. As the newest member it starts serving
there; an older one joins the set without taking over. `--no-serve` holds what
the scope serves today, staging the release for a later `sandbox deploy`.

Attaching the newest release also clears an unpinned pointer left by an earlier
`sandbox deploy`, which is newest-wins resuming. A pinned scope is unmoved.

### yard sandbox detach \<scope\> \<release\>

Removes a release from the set, the "stop serving this" action. The release
survives; it belongs to the project. Refused (409) when it would leave the
scope with nothing to serve.

### yard sandbox deploy \<scope\> \<release\>

Points the scope at a release and checks it out, attaching it first when
it is not already a member. This is how you ship a specific release, however
old. The choice is **not** frozen: the next release attached on top takes over.

### yard sandbox pin \<scope\> [release]

Pins the scope to a release, so later attaches join the set without taking
over. Naming no release pins what it serves right now, which is how a rollback
is held.

### yard sandbox unpin \<scope\>

Clears the pointer, so the newest member serves again and every later attach
takes over.

### yard sandbox promote \<from\> \<to\>

Attaches the release `<from>` currently serves to `<to>`, so the two serve
identical content. **Nothing is copied**: promote adds the release to the
target's set, where as the newest member it starts serving. **Data and secrets
never promote**; each scope keeps its own. `<from>` must serve a release.
Prefer `sandbox deploy` when you can name the release.

`attach`, `detach`, `deploy`, `pin`, `unpin` and `promote` all answer the same
JSON:
`{"release_id": "…", "version": "v1.4.0", "to": "", "action": "attach", "artifacts": [...]}`,
where `to` is the wire slug of the scope acted on and `""` is the project's
global data.
Human output ends with what the scope now serves, plus the live and
`/service/` URLs when the target was the global scope.

### Recipes

```sh
# Try a release in a sandbox, then ship it
yard sandbox create staging
yard sandbox deploy staging v1.4.0
yard sandbox deploy global v1.4.0

# Roll back the storefront and hold it there through later publishes
yard sandbox pin global v1.3.0

# Resume newest-wins once the fix ships
yard sandbox unpin global

# Stage a release without serving it, deploy on your own schedule
yard sandbox attach global v1.5.0 --no-serve
yard sandbox deploy global v1.5.0

# Share a work-in-progress preview with someone who has no Yard account
yard sandbox create preview
yard sandbox deploy preview v1.5.0-rc1
yard sandbox visibility preview public   # anyone with the URL can now view it

# Let a sandbox follow a channel instead of attaching by hand
yard sandbox channel preview Beta

# What is each scope serving, and why?
yard sandbox list --json | jq '[.global] + .sandboxes | .[] | {slug, serving: .serving_release.version, pinned, deploy_status}'
```

---

## yard service

Manage a project's running service: its URL, logs, secrets and database (see
`service-and-database.md` for the runtime model). Service CODE is not managed
here: `yard push` uploads it into a release, and attaching that release to a
scope is what deploys it. Requires the `service` permission (Pro).
Shared flags: `--project <slug-or-uuid>`, `--dir <path>`, `--sandbox <slug>`
(omitted = the project's global data), `--json`.

**Bundle constraints enforced client-side before any HTTP:** ≤200 files,
≤5 MB per file, ≤25 MB total, paths nest ≤8 levels, extensions in
`.html .css .js .mjs .json .svg .png .jpg .jpeg .webp .gif .woff2 .woff
.ttf .otf .txt .md .ico .map .wasm .webmanifest` (plus `_worker.js`,
`migrations/*.sql`). `_worker.js` is required. Dotfiles and legacy
bundle-root local-dev files (e.g. `README.md`, the retired `yard.json`
manifest) are skipped (reported as ignored), so old scaffolds that carried
them inside the bundle still deploy.

### yard service init

Scaffolds a zero-dependency working service (notes API + vanilla frontend +
first migration) into `./<name>` (`--service-dir <dir>` to change the
directory, `--url <path>` the path it serves under, default `/<name>`). The
bundle's own `settings.json` is written alongside it as `{"name": "<name>",
"url": "/<name>", "access": "authenticated", "database": true}` — that file
is what names the service. A `README.md` describing the workflow is written
at the top of the **working directory**, never inside the bundle, and is
write-if-absent (an existing file is skipped and reported). Adds the bundle
dir to `services` in `.yard/settings.json`, bootstrapping an unlinked
settings file first when the project isn't yard-initialized (a note says to
run `yard init`). Run it once per service. JSON: `{"dir": "api",
"service": "api", "url": "/api", "written": [...], "skipped": [...],
"service_dir_recorded": true, "settings_bootstrapped": false}`.

### yard service open

Prints a service's URL for the scope and opens it in a browser.
`--service <name>` picks one when the project declares several; with exactly
one it is inferred. A private sandbox's URL is a team-only preview
(sign-in enforced at the edge). JSON: `{"sandbox": "...", "service":
"api", "url": "...", "deployed": true}`, where `sandbox` is `""` for the
project's global data.

### yard service check

Validates every declared bundle exactly like a deploy would (limits,
extensions, `_worker.js` presence, each bundle's own `settings.json`) and
checks that no two services share a name or a mount path, plus lint warnings
for root-absolute `href`/`src`/`fetch("/…")` URLs — no network, no login.
JSON: `{"services": [{"name": "api", "dir": "api", "mount_path": "/api",
"files": 7, "total_bytes": 5494, "ignored": [...], "warnings": [...],
"access": "authenticated", "database": true}]}`.

### yard service secrets set KEY=VALUE [KEY=VALUE...] / list / rm \<name\>

Per-scope secrets that become `env.<NAME>` bindings on the **next
deploy**, in every service of that scope. Names UPPER_SNAKE (≤32 per
scope, ≤4 KB each; `DB` and `ASSETS` reserved). Write-only: `list` shows
names and timestamps, never values. Secrets don't promote between
scopes.

### yard service db query [sql]

Runs SQL against the scope's database, the one every service there
shares, and prints rows as JSON. SQL from the inline argument, `--file
<path>`, or stdin (`-`). Caps: 10 kB SQL, 1000 rows returned. The
`_yard_migrations` table records applied migrations as `<service>/<file>`.

### yard service logs

Recent output from one service (console lines, uncaught exceptions, abnormal
request outcomes), newest window ≤24 h. `--service <name>` picks one when the
project declares several. `--limit <n>` (cap 500), `--since <dur>` (e.g.
`30m`, `2h`). A service that has never logged returns an empty list, not an
error. Logs appear a few seconds after the request.

---

## yard uninstall

Remove the CLI from your system.

**Flags:**

- `--force`, `-f` — Skip the confirmation prompt

**What gets removed:**

1. Config directory (`~/.yard/` — contains `config.json`)
2. CLI binary (detected via `os.Executable()`)

If removal fails (e.g., permission denied), prints manual cleanup instructions with the exact paths.
