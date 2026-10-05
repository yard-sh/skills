# Yard CLI Command Reference

The binary is `yard`. Most commands take `--json` (one JSON object on stdout, logs on stderr) and many take `--spec <file|->` (a JSON body from a file or stdin). `yard <command> --help` lists every flag.

---

## yard login / logout

`yard login` prints a nine-digit one-time code (`123 456 789`), opens `https://yard.sh/login/device`, and waits while the user signs in (or signs up, which walks them through creating a team), enters the code and confirms. Nothing on the machine has to be reachable from the browser, so it works over SSH and in remote workspaces: open the printed link on any device. If no browser opens, the CLI says so and keeps waiting.

- The code is valid for 15 minutes; after that login fails with "the code expired before it was used". Cancelling on the confirm screen leaves the CLI waiting until then.
- The session lasts up to 90 days, is renewed automatically, and is listed under command-line sessions on the account's security page, where it can be revoked.
- It is saved in `~/.yard/config.json` (permissions 0600). The active team is not stored there (see `yard team`).

`yard logout` revokes the session on the server and deletes the local file (it still deletes the file when the server is unreachable).

---

## yard me

The signed-in user, the team they act as, and what each may do. `--json`:

```json
{
  "id": "550e8400-…",
  "username": "alice",
  "github_username": "alice",
  "email": "alice@example.com",
  "plan": "Pro",
  "permissions": { "create_teams": { "granted": true, "value_type": "boolean" } },
  "team": { "id": "1f0c8a2e-…", "name": "Acme Corp", "username": "acme", "role": "owner" },
  "team_permissions": {
    "sell_projects": { "granted": true, "value_type": "boolean" },
    "license_keys": { "granted": true, "value_type": "boolean" },
    "coupons": { "granted": true, "value_type": "boolean" },
    "max_pricing_tiers": { "granted": true, "limit": 10, "value_type": "limit" },
    "max_projects": { "granted": true, "unlimited": true, "value_type": "limit" }
  }
}
```

- **`team_permissions` decides every team feature** (projects, tiers, coupons, license keys, custom pages, services, Yard Auth, sandboxes, API keys). It is the merged entitlement of the active team's owners. A free user in a Pro team gets Pro features on that team's projects; a Pro user acting as a free team does not.
- `permissions` is the user's own entitlement, for account-level things like `create_teams`.
- Booleans carry `granted`; limits also carry `limit` or `unlimited: true`. Gates are permission-based, not tied to a plan name. `plan` is a display label.
- `team` is `null` (and `team_permissions` absent) when the user belongs to no team; every team command then fails with `NO_TEAM`.

---

## yard team

Show or switch the team the CLI acts as: `yard team [--json]`, `yard team use <username>` (the leading `@` is optional; an unknown team is rejected with the list of the user's teams).

```
Team:   Acme Corp (@acme)
Role:   owner
Projects are published at https://yard.sh/@acme/<slug>
```

`--json` emits `{ active_team_id, active_team, teams }`. The active team lives on the account, shared with the dashboard's team switcher, so it can change between commands; re-check it rather than trusting an earlier answer. Membership is `owner` or `admin`: both run the whole project surface, but payouts and billing (reads included) are owner-only and answer `403 NOT_TEAM_OWNER` to an admin.

A team command run without a team answers `403` with `code: "NO_TEAM"` ("A team is required"). It is not a plan problem: set one up at https://yard.sh/finish, then run `yard team`.

---

## yard init

Links the current directory to a project and writes `.yard/settings.json`. No git repo is needed. Inside a clone of a GitHub repo the Yard GitHub App can access, every mode links that repo to the project it creates or picks when neither is linked elsewhere; otherwise the wizard asks, and the non-interactive modes print the [`yard github link`](#yard-github) command to run.

| Mode | Invocation | Use |
| --- | --- | --- |
| Spec | `yard init --spec <file\|-> --json` | Create a new project from JSON. Agents use this. |
| Link | `yard init --project <slug-or-uuid> --json` | Link to an existing project. A fresh directory also pulls the latest Production release whole (settings, landing page, service bundles); `--no-pull` skips that. |
| Interactive | `yard init` | Humans only. Driving it through stdin is a dead end. |

Flags: `--json`; `--page` / `--no-page` (scaffold a landing page or not; `--json` defaults to no page); `--link-repo` / `--no-link-repo` (default: link when possible, otherwise note why and carry on; `--link-repo` makes that an error; `--no-link-repo` skips linking). Not logged in, the non-interactive modes fail with `not logged in. Run 'yard login' first`.

**Spec schema.** Only `title` and `tiers` are required.

```jsonc
{
  "title": "My Project",        // 1-60 chars, must contain a letter or digit
  "pricing_model": "one_time",  // "one_time" (default) | "subscription"
  "tiers": [
    {
      "name": "Base",               // required
      "price_cents": 1900,          // 0 for free, else 300..1000000
      "is_default": true,           // exactly one default
      "seat_type": "single",        // "single" | "fixed_pack" | "per_seat" (seat-based is plan-gated)
      "seat_count": null,           // fixed_pack: 2..1000
      "min_seats": null,            // per_seat; default 1
      "max_seats": null,            // per_seat; optional
      "yearly_discount_percent": null, // subscription only, 1..100
      "volume_brackets": [],        // per_seat only
      "free_trial": {               // per tier, plan-gated
        "enabled": false,
        "days": null,               // 7..365; a trial without days runs 7
        "requires_card": false      // subscription tiers: collect a card for the trial; omitted means false
      },
      "gift_enabled": false         // one-time tiers only
    }
  ],
  "license_key_enabled": false,  // plan-gated
  "activations_enabled": false,  // plan-gated; needs license keys
  "max_activations": null        // 1..10000
}
```

How many tiers a plan allows is `max_pricing_tiers`. `free_trial` and `gift_enabled` exist only per tier; a project-level trial field is rejected with `unknown field`. The launch stage and an early-access discount are set in the dashboard.

`--json` output:

```json
{
  "project": { "id": "…", "slug": "simple-note", "title": "Simple Note",
    "buy_url": "https://alice.yard.sh/simple-note", "profile_url": "https://alice.yard.sh/", "created": true },
  "settings_file": "/abs/path/.yard/settings.json",
  "github_repo_linked": false,
  "landing_page": null
}
```

`github_repo_linked` is whether the project has a linked repo once init is done.

**Interactive flow**, for explaining it to a human: an update check (a newer CLI must be installed first), sign-in if needed, the optional GitHub App install and repo check, pick an existing project or create one (title, then a price of $3.00 or more, or $0), write `.yard/settings.json`, link the repo (asking before moving or replacing a link), pull the latest Production release into a fresh directory, offer a custom landing page, then offer license keys and device activations. The wizard never blocks a choice by plan: the server answers `upgrade_required`, the CLI shows the upgrade link and offers to retry the same request once the user has upgraded.

**Troubleshooting:** a hanging `yard init` is the interactive wizard (interrupt, retry with `--spec -` or `--project`). After a failed attempt, check `yard projects --json` and link rather than re-create.

---

## yard projects

`yard projects [--json]` lists the active team's projects:

```
NAME                                PRICE      RELEASES   SALES
✓ my-awesome-tool                   $9.99      3          12
✗ private-beta                      $29.00     1          0
```

`✓` is public, `✗` is not (change it with `yard sandbox visibility`). The table shows display names; every other command wants the slug, which `--json` carries on `.slug` (tiers are not included there; use `projects show`).

```sh
yard projects --json | jq -r '.[].slug'                                  # all slugs
yard projects --json | jq -r '.[] | select(.title | ascii_downcase | contains("simple")) | .slug'
jq -r .project_slug .yard/settings.json                                  # the project in this directory
```

Slugs and UUIDs are interchangeable wherever `<slug-or-id>` is accepted.

### yard projects show \<slug-or-id\>

One project in full, including `tiers[]` with `pricing_model`, `seat_type`, `features`, `volume_brackets` and the per-tier trial and gift fields. Use it to answer "does any tier offer a trial?":

```sh
yard projects show my-tool --json | jq '.tiers[] | select(.free_trial.enabled) | {id, name, days: .free_trial.days}'
```

### yard projects edit [slug-or-id]

Project-level settings: `license_key_enabled`, `activations_enabled`, `max_activations`. With no argument it picks the only project or prompts (in `--spec` mode it errors with the list of slugs). Interactive mode asks each question with the current value as default. `--spec` takes a sparse JSON object (unknown fields rejected, missing fields untouched); `--json` emits `{ "project": {...}, "settings": {...} }`. Enabling activations without restating `license_key_enabled` works when the project already has license keys.

Tiers are not project settings: a `tiers` key is rejected with an error naming `projects tiers add`, `edit` and `rm`. Pricing belongs to a release; change it with `projects tiers` or the `pricing` block in settings.json.

The server enforces the plan: a missing feature is `upgrade_required`, printed with the pricing link (interactive exits 0, `--spec` exits non-zero).

### yard projects tiers

Add, change or remove one tier of a release's pricing without resending the list. With no `--release` they edit your open draft (or a new draft seeded from your newest published release), the same release `yard push` writes to, and the new pricing goes live with `yard releases publish`. `--release <tag|id>` edits a published release instead, live at once wherever it is served; on the release the project serves, subscribers on a subscription tier whose price changes get notice before it applies. Tiers match by name, case-insensitively (pricing has no tier ids). All accept `--json` (the release's tier list after the save). To set every tier at once, use the settings.json `pricing` block and `yard push`.

- `yard projects tiers add <slug> --spec <file|->`: the spec is one entry of settings.json `pricing.tiers` (`name`, `price_cents`, `description`, `features` (max 20), `pricing_model`, `seat_type`, `seat_count`, `min_seats`, `max_seats`, `volume_brackets`, `yearly_discount_percent`, `free_trial`, `gift_enabled`, `is_default`); unknown fields are rejected. `is_default: true` demotes the current default, and a release's first tier is its default. Over `max_pricing_tiers` is `upgrade_required`.
- `yard projects tiers edit <slug> <tier-name> --spec <file|->`: a partial spec; present fields replace, absent ones stay.
  ```sh
  echo '{"free_trial": {"enabled": true, "days": 14}}' | yard projects tiers edit simple-note Base --spec -
  ```
- `yard projects tiers rm <slug> <tier-name> [--yes] [--promote-default]`: drops the tier from the release. Existing purchases are unaffected and its subscribers keep their price. Refuses to remove the last tier, or the default without `--promote-default`. `--yes` is required without a TTY.

---

## yard releases

A release is a project-wide snapshot (landing page, pricing, download buttons, services, files). Concepts: [releases-and-updates.md](releases-and-updates.md). With no `--release`, commands that write target your open draft, or a new draft seeded from your newest published release; `--release` takes a tag or UUID.

### yard releases publish [tag]

Uploads any `--file` assets into the draft, then publishes it under the tag into a channel (default `Production`, which the project follows, so it is live). Anything `yard push` already staged in the draft ships with it.

Flags: `--project`, `--dir`, `--name`, `--notes`, `--notes-file <path|->`, `--file <path>` (repeatable), `--channel <name>` (must exist; channels are created in the dashboard, up to `max_channels` per project with Production included: Basic 1, Pro 10), `--release <id|tag>` (must still be a draft), `--spec <file|->`, `--json`. Tags are unique per project (`409` on reuse).

```jsonc
{
  "project": "my-slug",                 // optional in the project's folder or with one project; --project wins
  "tag_name": "v1.4.0",                 // required, ≤255 chars
  "release_name": "Late April fixes",   // ≤255 chars
  "release_notes": "## Highlights\n…",  // markdown, ≤125,000 chars
  "channel": "Production",
  "files": ["./dist/app-darwin-arm64.tar.gz", "./dist/app-linux-amd64.tar.gz"]
}
```

Per-file `✓`/`✗` lines go to stderr. If some files fail, the draft is still published without them and the exit code is non-zero; if every file fails, nothing is published. `--json`:

```json
{
  "release": { "id": "…", "version": "v1.4.0", "is_draft": false, "files": [] },
  "deployed": [{ "release_id": "…", "version": "v1.4.0", "to": "", "action": "attach", "artifacts": ["page", "releases"] }],
  "files": [{ "path": "./dist/a.tar.gz", "status": "uploaded", "size_bytes": 12345678 }],
  "uploaded": 1,
  "failed": 0
}
```

In `deployed`, `to` is `""` for the project itself, else a sandbox name.

### yard releases promote \<tag\> --to \<channel\>

Moves a published release into another channel (out of the one it was in). A channel serves its most recently published release, so followers of the new channel serve it only if it is the newest published there (use `yard sandbox rollback` or `pin` for an older one); followers of the old one fall back to its next newest release. Nothing is copied. An archived release is refused: unarchive it instead. Flags: `--to` (required, must exist), `--project`, `--dir`, `--json` (`{ "channel": "Beta", "deployed": [...] }`).

### yard releases archive \<tag|id\>

Takes a published release out of its channel, frees its tag and deletes its download files (everything else is kept). Followers of that channel fall back to its next newest release. Refused with `409` while the project or a sandbox is pinned or rolled back to it (unpin, or roll back to a newer release, first). Drafts are refused (delete them in the dashboard). Flags: `--project`, `--dir`, `--json` (`{ "from_channel": "Beta", "deployed": [...] }`).

### yard releases unarchive \<tag|id\> [--to \<channel\>]

Returns an archived release to the channel it was archived from (`Production` if that channel is gone), or to `--to`, without its download files. A tag names the newest archived release carrying it. The release takes its tag back (`409` if another release holds it). Flags: `--to`, `--project`, `--dir`, `--json` (`{ "channel": "Beta", "deployed": [...] }`).

---

## yard channels list

The project's release channels (read-only; create, rename, delete and reorder them in the dashboard). Flags: `--json`, `--project`, `--dir`.

```
CHANNEL      RELEASES  LATEST         FOLLOWED BY        VISIBILITY  PROTECTED
Production   7         v1.4.0         my-project         public      yes
Beta         2         v1.5.0-beta.1  preview, staging   private
```

`LATEST` is the release a new follower would serve (`-` when none). `--json`: `{ "project", "channels": [{ id, name, protected, visibility, created_at, release_count, latest_version, sandboxes }] }`, where `""` in `sandboxes` is the project itself.

---

## yard github

Links GitHub repos to the team's projects; releases published on a linked repo sync to its project ([releases-and-updates.md](releases-and-updates.md#syncing-releases-from-github)). A repo is linked to one project at a time and must be one the Yard GitHub App can access.

- `yard github list [--json]`: every linked repo, its project and sync channel. `--json`: `{ "links": [{ repo_id, repo_name, project: { id, slug, title, visibility }, sync_channel, channels }] }`.
- `yard github link [owner/repo] [--channel <name>] [--move]`: links the repo (default: the directory's `origin`), replacing the project's own. A repo linked to another project fails until `--move`, which leaves that project unlinked. `--channel` sets the sync channel; with no repo argument on a project that already has one, only the channel changes. `--json`: `{ project, repo_id, repo_name, sync_channel, moved_from? }`.
- `yard github unlink`: stops syncing; synced releases and the sync channel stay. `--json`: `{ project, repo_name, unlinked }`.

`link` and `unlink` take `--project`, `--dir` and `--json`. A missing app install or repo access fails with the link to fix it ([installing the app](releases-and-updates.md#installing-the-yard-github-app)).

---

## yard keys

API keys belong to the active team, not to their creator: anyone on the team can use one and it survives its creator leaving. Check `yard team` before minting. Scopes and what they allow: [api-reference.md](api-reference.md#authentication).

- `yard keys list [--json] [--sort created_at|name|last_used_at] [--direction asc|desc]`: name, prefix (`yard_xxxxxxx`), scopes, last used, created. The secret is never shown again.
- `yard keys create [name] [--scopes <csv>] [--spec <file|->] [--json]`: the secret is printed once (`key` in `--json`). Without `--scopes` it prints the scope catalog. Up to 100 keys per team.

```sh
KEY=$(echo '{"name":"ci-runner","scopes":["licenses:validate"]}' | yard keys create --spec - --json | jq -r .key)
```

Testing license-key validation end to end: [pricing-and-licensing.md](pricing-and-licensing.md#license-keys).

---

## yard coupons

Discount codes (needs `.team_permissions.coupons`; the server answers `403` without it). Every subcommand takes a code or UUID. With both flags and `--spec`, flags win field by field.

**Units:** `--percent 20` is 20% off; `--amount 5` is **$5.00**. In a spec, `discount_value` is a percent for `percentage` and **cents** for `fixed_amount`. Dates are `YYYY-MM-DD` or RFC 3339; a bare `--expires` date covers the whole day (UTC), a bare `--valid-from` starts at midnight UTC.

- `list [--json] [--sort <col>] [--direction] [--page] [--limit ≤100]`. Sort by `createdAt`, `lastModified`, `code`, `discountValue`, `scopeDisplay`, `currentUses`, `status`, `savingsCents`. `STATUS` is derived: `inactive`, `expired`, `scheduled`, `used up`, `active`; in JSON, `is_active` is only the on/off switch.
- `show <code>`: `{ "coupon": {...}, "analytics": {...} }`.
- `create <code>`: `--percent N | --amount D`, `--projects <csv>` (slugs or UUIDs; without it the coupon covers every project, including future ones), `--max-uses N`, `--expires`, `--valid-from`, `--subscription-duration once|forever` (first payment, the default, or every renewal). Codes are upper-cased with spaces removed, then 4-50 letters, digits, `-` or `_`.
  ```jsonc
  { "discount_type": "percentage", "discount_value": 20, "scope": "all_projects", "project_ids": [],
    "max_uses": 100, "expires_at": "2026-12-31T23:59:59Z", "valid_from": null, "subscription_duration": "once" }
  ```
- `generate --count N` (1-100) `[--prefix P] [--length N]` plus any create flag: unique codes without look-alike characters, **shown once** (one per line on stdout; `--json` for the full list).
- `update <code>`: partial. Clear with `--no-expiry`, `--no-valid-from`, `--unlimited-uses` (or `null` in a spec); `--activate` / `--deactivate`; `--projects` alone re-scopes. The discount cannot change after the first redemption.
- `rm <code> [--yes]`: only unused coupons; deactivate redeemed ones instead.
- `transactions <code>`: the sales it was used on.
- `validate <code> --project <slug> [--tier <uuid>] [--team <username>]`: runs the checkout check and reports the price; the project must be public. Exit 0 whenever the check ran; read `.valid`.

```sh
yard coupons generate --count 50 --prefix INFL --percent 15 --json | jq -r '.coupons[].code' > codes.txt
yard coupons list --json | jq -r '.coupons[] | select(.max_uses != null and .current_uses >= .max_uses) | .code' \
  | xargs -r -I{} yard coupons update {} --deactivate
```

---

## yard users

Read-only list of users with at least one completed, unrefunded purchase, across the team's projects. Money is pre-formatted text (`"$87.00"`); use `yard transactions` for arithmetic. Sandbox (simulated) users never appear.

- `list [--json] [--project <slug>] [--sort lastTransaction|firstTransaction|email|username|orderCount|totalSpent|userDisplayId] [--direction] [--page] [--limit]`. `--project` narrows both the rows and the summary line.
- `show <user-id-or-email> [--page] [--limit]`: that user's totals and their orders (refunded, pending, failed and trial ones included). Ids look like `user_deadbeef` and are truncated account ids, so two users can share one: that is a `409` ("Two users share this id; pass their email instead"), and the email resolves it.

```sh
yard users --project my-tool --sort totalSpent --direction desc --json | jq -r '.users[] | "\(.email) \(.total_spent_display)"'
```

---

## yard transactions

The team's sales. Ids are `order_xxxxxxxx` or the full UUID. Teams can't issue refunds from the CLI or the dashboard yet; Yard support (support@yard.sh) issues them. Sandbox (simulated) sales never appear here, in earnings or in payouts.

- `list [--json] [--project <slug>] [--start <date>] [--end <date>] [--search <text>] [--type purchase|subscription|renewal|trial|trial_upgrade|gift] [--status completed|pending|failed|converted|refunded|refund_pending] [--sort date|amount|teamEarnings|projectName] [--direction] [--page] [--limit]`. With `--project`, `--sort` takes `date|amount|platformFee|teamEarnings|quantity|tierName|launchStage|userDisplayId|id`. The summary covers every sale in the date range (and only that project's with `--project`); `--search`, `--type` and `--status` narrow the rows and the total only. `TYPE` is `gift`, `trial`, `trial upgrade`, `renewal`, `subscription started` or `purchase`.
- `show <order-id>`: tier, quantity, coupon, refund date, billing period, trial expiry.
- `trial <order-id> --add-days N`: lengthen (`7`) or shorten (`-3`) a running trial, up to 365 either way. Days are added to the **current expiry, not today**. An expired trial whose new expiry is in the future becomes active again (`"reactivated": true`), unless its user has since started another trial on that project. A card-required trial's first charge moves with it: taking it to today or earlier ends the trial and charges the card now, and one already charged or with a plan change scheduled is a `409`. **The user on the trial is notified** (email and in-app) unless they turned off Transaction confirmations. Needs `.team_permissions.sell_projects.granted`. The trial length offered to new users is the tier's `free_trial.days`.

```sh
yard transactions list --type trial --json | jq -r '.transactions[] | "\(.id) \(.user_email) \(.trial_expires_at)"'
yard transactions list --project my-tool --start 2026-07-01 --json | jq '[.transactions[].team_earnings_cents] | add'
```

---

## Project files: settings.json, push, pull, status, ls

Every project command walks up from the cwd to the directory holding `.yard/settings.json` (`--dir` overrides). Layout:

```
<project>/
├── .yard/
│   ├── settings.json
│   ├── migrations/0001_init.sql   # default migrations.dir
│   └── landing-page/index.html    # default landing_page.dir
└── api/_service.js                # one directory per service
```

`.yard/settings.json` (version 8; every block is optional):

```json
{
  "version": 8,
  "project_slug": "my-project",
  "ignore_files": ["*.bak", "drafts/**"],
  "services": [{ "dir": "api", "name": "api", "url": "/api", "access": "authenticated", "database_access": true }],
  "landing_page": { "type": "custom", "dir": ".yard/landing-page" },
  "migrations": { "dir": ".yard/migrations" },
  "pricing": { "tiers": [{ "name": "Base", "price_cents": 1900, "is_default": true, "pricing_model": "one_time" }] },
  "downloads": { "buttons": [{ "condition": "ends_with", "value": ".dmg", "label": "Download for Mac" }] }
}
```

- `project_slug`: the project this directory belongs to.
- `ignore_files`: globs relative to the landing-page directory (`**` matches any depth); dotfiles are always ignored.
- `services`: see [service-and-database.md](service-and-database.md#service-settings).
- `landing_page`: `type` `custom` serves the files in `dir`; `default` serves the dashboard-edited default page (files still upload, unserved). A block without `type` means `custom`; no block keeps whatever the release already has. A `custom` block with no files fails the push; custom pages are plan-gated.
- `migrations.dir`: flat numbered `.sql` files, not inside a service directory. See [service-and-database.md](service-and-database.md#database).
- `pricing.tiers` / `downloads.buttons`: when present, a push replaces the release's tiers or download buttons to match exactly. Shapes: [releases-and-updates.md](releases-and-updates.md#syncing-releases-from-github).

A push uploads `settings.json` itself, which is how deploys learn each service's settings, so changing one is an edit plus a push. The server keeps each release's copy in step with dashboard edits, so `yard status` can show a config diff you did not make; `yard pull` brings it down (your `project_slug` is kept) and `yard pull --force` discards local changes. Older layouts are rejected; see [troubleshooting.md](troubleshooting.md#yardsettingsjson-uses-an-old-service-layout).

**Common flags:** `--project <slug-or-uuid>`, `--dir <path>`, `--release <id|tag>` (default: your open draft, or a new draft seeded from the newest published release; required when several drafts are open; a published release is edited in place and is live on save if something serves it), `--json`, `--yes` (skip prompts: `push --prune`, pushing into a served release).

**Exit codes:** `0` success, `1` fatal (auth, network, validation), `2` partial (`push` only: some files uploaded, some failed; see `errors`).

**Limits, checked before any upload.** File counts and sizes come from the team's plan (`team_permissions` `page_max_*`, `service_max_*`, `migrations_max_*` in GET /v1/me); the numbers below are Basic's and Pro's.

- Landing page: ≤60 files, ≤25 MiB each, ≤100 MiB total; `.html .css .js .json .svg .png .jpg .jpeg .webp .gif .woff2 .mp4 .webm .vtt`; letters, digits and `._-`, at most one subdirectory, no dotfiles; `index.html` required to publish.
- Service: ≤600 files, ≤15 MiB each, ≤75 MiB total, `_service.js` ≤10 MiB on every plan, ≤7 nested folders, paths ≤200 characters; the landing page types except `.mp4 .webm .vtt`, plus `.mjs .woff .ttf .otf .txt .md .ico .map .wasm .webmanifest`; `_service.js` required; `.sql` rejected (migrations are project-level); dot-files, dot-folders and `node_modules/` (any depth) and bundle-root `README.md`, `wrangler.toml`, `yard.json` and `settings.json` are skipped; `ignore_files` doesn't apply.
- Migrations: any number of files, ≤3 MiB each, ≤15 MiB total, no subdirectories.

### yard init --page

Scaffolds the landing-page directory in an existing project: pulls the draft's page files, or writes a hello-world `index.html` + `styles.css`, and records `"landing_page": {"type": "custom"}` so the next push serves it. Idempotent; existing files are kept. `--json`: `{ project_root, project, source, release, written, skipped, preview_url, live_url }` (`source` is `"starter"` or the release it pulled from).

### yard status

What `yard push` would change, per bundle, without writing: `to_upload`, `unchanged`, `remote_only` (removed only by `push --prune`). It also lists who serves that release and each one's deploy status. Nothing serves a draft, so after a push `serving` is empty; `yard sandbox list` covers every place regardless of release.

```json
{ "project": "my-slug", "release": "9f3e…", "version": "1.2.0", "draft": false,
  "page": { "dir": "…/.yard/landing-page", "to_upload": ["index.html"], "unchanged": ["styles.css"], "remote_only": [] },
  "serving": [{ "sandbox": "", "deploy": "stale" }] }
```

### yard ls

A release's files grouped by bundle (`page`, `service`, …), each with `path`, `content_type` and `size_bytes`.

### yard push

Uploads every changed local file (landing page, each service, migrations, settings.json) into the draft; unchanged files are skipped. Every bundle is validated before anything uploads. `--prune` deletes release files missing locally (one confirmation unless `--yes` or `--json`). Prints a `Review:` URL; going live is `yard releases publish <tag>`. A `pricing` block is applied (invalid pricing is a 400 naming the field, before any upload). When the live deployment has a room class the local settings no longer declare, push warns `class Old will be deleted with all its data on deploy`.

```json
{
  "project": "my-slug", "release": "9f3e…", "version": "",
  "page": { "dir": "…", "uploaded": ["index.html"], "skipped": [], "deleted": [], "remote_only": [] },
  "services": { "api": { "dir": "…/api", "uploaded": ["_service.js"], "skipped": [], "deleted": [], "remote_only": [] } },
  "config": { "dir": "…/.yard", "uploaded": ["settings.json"], "skipped": [], "deleted": [], "remote_only": [] },
  "review_url": "https://dash.yard.sh/projects/my-slug/release?release=9f3e…",
  "live_url": null,
  "errors": []
}
```

A bundle the project lacks is absent. `live_url` is set once the project itself serves a release.

### yard pull

Downloads a release (default: your draft) into the project: settings.json, the landing page, and each service into its directory (a missing service directory is not created; `yard init` in a fresh directory is the flow that does). Files already identical are skipped; `--force` overwrites. `--json`: per bundle `{ destination, written, skipped }`.

There is no publish flag on `push`. To discard draft changes, delete the draft in the dashboard and `yard pull --release <last-tag> --force`.

---

## yard sandbox

A sandbox is an optional private copy of the project at `/<slug>/@<sandbox>/`, with its own files, services, database, secrets and simulated commerce ([pricing-and-licensing.md](pricing-and-licensing.md#commerce-in-a-sandbox)). A project starts with none; sandboxes are a Pro feature and `max_sandboxes` caps how many (10; over it is `403 sandbox_limit_reached`). `visibility`, `channel`, `rollback`, `pin` and `unpin` act on the project itself unless `--sandbox <name>` names one; `create`, `rename` and `delete` take the sandbox's name, and `promote` names its target with `--to`. Shared flags: `--project`, `--dir`, `--json`; every `<release>` is a tag or UUID.

What serves:

| State | Reached by | Serves |
| --- | --- | --- |
| following a channel | `sandbox channel <name>`, `sandbox unpin` | the channel's newest release; new releases take over |
| rolled back | `sandbox rollback <release>` | that release until the next one lands in the channel |
| pinned | `sandbox pin [release]`, `sandbox promote` | that release until `unpin` |

A pin outranks a rollback, which outranks the channel. A bad release is live → `rollback` (the fix takes over when published). Hold one release through later publishes → `pin`.

- `list`: the project, then each sandbox, with what serves and why (`1.3.0 (pinned)`, `(rolled back)`, `(<channel>)`, `(no channel)`, `-`), release count, channel, visibility, deploy status (blank when up to date). `stale`: the release changed and a redeploy starts shortly. `updating`: deploying. `failed`: still serving the previous version, with the error; it is retried automatically, but a failure that persists is in the bundle, so fix it and push. JSON: `{ "project": {...}, "sandboxes": [...] }`, each with `slug`, `visibility`, `channel`, `pinned_release_id` (when pinned), `serving_release`, `releases`, `deploy_status` (`up_to_date`, `stale`, `updating`, `failed`), `deploy_error` (when failed), `page_url`.
- `create <name>`: 2-60 letters, digits and hyphens, starting with a letter. Private, serving nothing until it follows a channel or is pinned.
- `rename <name> <new-name>`: keeps releases, files, secrets and database; the URL changes.
- `visibility <public|private>`: `private` (a sandbox's default) admits only the owning team; `public` lets anyone with the URL in. Without `--sandbox` it sets the project itself (that is how a project goes private). A draft project still serves nothing publicly.
- `delete <name> [-y]`: removes its files, services, database and all of its simulated commerce, immediately. Releases belong to the project and survive.
- `channel <channel|none>`: follow a channel (new projects follow `Production`) or none.
- `rollback <release>`: serve an earlier release now; refused with `409` while pinned (unpin, or move the pin).
- `pin [release]`: hold on a release; no release pins what serves now (makes a rollback permanent).
- `unpin`: back to the channel's newest release (nothing serves with no channel).
- `promote <from-sandbox> [--to <sandbox>]`: pin the target to what the source serves; no `--to` means the project itself, taking it live. Data and secrets never move.

`rollback`, `pin`, `unpin` and `promote` answer `{ "release_id", "version", "to", "action", "artifacts" }` (`to` is `""` for the project). Elsewhere in JSON output, `""` in a `sandbox` field also means the project itself.

```sh
# Try a release in a sandbox, then ship it
yard sandbox create staging && yard sandbox pin v1.4.0 --sandbox staging && yard sandbox pin v1.4.0
# Share a preview with someone who has no Yard account
yard sandbox visibility public --sandbox preview
# What serves where, and why
yard sandbox list --json | jq '[.project] + .sandboxes | .[] | {slug, serving: .serving_release.version, pinned: (.pinned_release_id != null), channel, deploy_status}'
```

---

## yard service, yard db, yard migrate

A service's code ships inside a release (`yard push`, then publish); these commands cover everything around it. Contract: [service-and-database.md](service-and-database.md). Commands that reach the project (`service open`, `service logs`, `service secrets`, `db`) take `--project`, `--dir`, `--sandbox <name>` (omitted = the project itself) and `--json`; `service init`, `service check` and `migrate` work on the local directory and take `--dir` and `--json`.

- `yard service init <name> [--service-dir DIR] [--url PATH] [--realtime]`: scaffolds a working service (notes API, vanilla frontend, a first migration in `.yard/migrations/` when none exists) and records `{"dir", "name", "url": "/<name>", "access": "authenticated", "database_access": true}` under `services`. `--realtime` scaffolds a broadcast `Room` class with a WebSocket client instead (no migration) and records `"rooms": [{"class": "Room", "binding": "ROOMS"}]`. A workflow `README.md` is written at the top of the working directory if absent. Run it once per service.
- `yard service open [--service NAME]`: prints and opens the service URL (`{ sandbox, service, url, deployed }`). A private sandbox's URL is team-only.
- `yard service check`: validates every bundle like a deploy would (offline; plan size limits are also checked when logged in), lints root-absolute `href`/`src`/`fetch("/…")` URLs, and warns when a declared room class is not exported.
- `yard service secrets set KEY=VALUE… | list | rm <name>`: `env.<NAME>` values for the project or one sandbox, shared by every service there; each change **redeploys those services** right away. Names are UPPER_SNAKE (not `DB` or `ASSETS`), ≤32 per target, ≤4 KiB each. Write-only: `list` shows names and times.
- `yard service logs [--service NAME] [--limit ≤500] [--since 2h]`: console output, uncaught exceptions and abnormal outcomes from the last 24 h, a few seconds behind. A fresh service returns an empty list.
- `yard db query [sql] [--file PATH]` (`-` for stdin): SQL against the project's or a sandbox's database, rows as JSON. Up to 10 kB of SQL, 1000 rows.
- `yard db migrations list`: applied migrations merged with local files still pending (`{ sandbox, database, migrations: [{ name, applied, applied_at, local }] }`). With no database yet, every file is pending.
- `yard db migrations mark-applied <file>`: records a migration as applied without running it; the repair step after fixing a file whose earlier statements already ran.
- `yard migrate [--dir PATH] [--json]`: upgrades an older `.yard/settings.json` layout (folds per-directory service settings files onto their entries, renames `database` to `database_access`, stamps `"version": 8`). A services entry still carrying `objects` stops it with the parser's error; rename that key to `rooms` by hand. Idempotent. Unrelated to database migrations.

---

## yard dev

Serves the project locally the way Yard hosts it: the landing page at `http://localhost:9875/<slug>/`, each service under its path, identity headers from a chosen persona, secrets from `.yard/dev/secrets.env`, a local database with migrations applied, and a control panel at `/__yard/dev/`. No login needed. Flags: `--port`, `--dir`, `--project`, `--as <persona>`, `--root` (serve at `/`), `--open`, `--secrets-file`, `--reset-db`, `--reset-rooms`, `--allow-local-egress`, `--no-panel`, `--offline`, `--json` (one event per line). Full guide: [local-dev.md](local-dev.md).

---

## Maintenance

- `yard version`: prints `yard v2026.09.24-abc1234` followed by build details.
- `yard update [--check]`: installs the latest CLI (`--check` only reports it). An interactive `yard init` refuses to run on an outdated CLI. Also updates a skill the CLI installed.
- `yard skill install | update [--check] | uninstall [--force]`: the Yard agent skill, in `~/.agents/skills/yard/` plus `~/.claude/skills/yard/` and `~/.codex/skills/yard/` when those directories exist. Downloaded from `cli.yard.sh` and checksum-verified. `npx skills add yard-sh/skills` is the alternative, but the CLI does not manage that copy (`yard update` reports it as not installed; `yard skill install` replaces it).
- `yard uninstall [--force]`: ends the session on the server, then removes `~/.yard/`, the skill copies, the `yard-cli://` handler and the binary; prints the paths to delete by hand if it cannot. Without a terminal it refuses unless `--force` is passed (so does `yard skill uninstall`).
