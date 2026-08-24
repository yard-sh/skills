# Releases and the Update Server

This reference covers how to publish a release for a Yard project (with file
assets) and how shipped software downloads those releases. Downloads
authenticate with **license keys** (per-buyer, hit `/v1/updates/latest`).
API keys cover first-party automation for licenses and subscriptions; they do
not grant release access.

## Table of Contents

- [What a release is](#what-a-release-is)
- [Publishing a release with the CLI](#publishing-a-release-with-the-cli)
- [Syncing releases from GitHub](#syncing-releases-from-github)
- [Downloading releases — license-key path](#downloading-releases--license-key-path)
- [Creating an API key](#creating-an-api-key)
- [Listing API keys](#listing-api-keys)
- [Troubleshooting](#troubleshooting)

---

## What a release is

A Yard release is a **project-wide snapshot** — landing page, pricing, download
buttons, service bundle, and file assets all live in one release. It consists
of:

- **`tag_name`** (version) — ≤255 chars (e.g. `v1.4.0`). Required to publish; a
  draft may hold a tentative tag or none yet.
- **`release_name`** — optional human title (≤255 chars)
- **`release_notes`** — optional markdown body (≤125,000 chars)
- **`files`** — zero or more file assets uploaded to the seller's storage bucket

Every release starts as a **draft**: an ordinary release that simply has not
been published yet. A draft can never be deployed anywhere, so editing one has
no side effects. Publishing stamps the tag and makes the release deployable —
publishing itself is **one-way**, but the release stays editable afterwards.
Editing a published release nothing serves is still side-effect free; editing
one an environment is serving is live the moment it saves. A project can hold
at most **10 open drafts**.

Releases belong to the **project**, not to any environment. Each environment
holds a **set** of releases and serves the newest member of that set, unless
the seller explicitly pins an older one. **Attaching a release to an
environment is the deploy moment.** Buyers only ever see what **production**
serves — attach a release there (`yard releases promote <tag> --to
production`, or publish with `--env production`) to make it live.

Version tags are **unique per project** across all non-archived releases —
publishing under an existing tag (or tagging a draft with one) is rejected
with `409`. A draft may reserve its tag early.

---

## Publishing a release with the CLI

`yard releases publish` is the canonical command. It supports three usage
patterns: full interactive, flag-driven, and `--spec` JSON for agents/scripts.

`publish` works on a **draft**: it uploads each file into your open draft
(creating one seeded from your newest published release if you have none;
`--release <id|tag>` names one explicitly), then publishes the draft
under the tag and attaches it to the target environment (`--env` /
`environment`, default `production` — live to customers). Anything `yard push`
already staged in that draft, landing page and service bundle alike, ships
with it.

### Spec mode (recommended for agents)

```sh
echo '{
  "project":      "my-slug",
  "tag_name":     "v1.4.0",
  "release_name": "Late April fixes",
  "release_notes": "## Highlights\n- Faster startup\n- Bug fixes",
  "files": [
    "./dist/yard-darwin-arm64.tar.gz",
    "./dist/yard-linux-amd64.tar.gz"
  ]
}' | yard releases publish --spec - --json
```

Spec field rules:

| Field | Type | Required | Notes |
|---|---|---|---|
| `project` | string | Only if you have multiple projects | slug or UUID |
| `tag_name` | string | Yes | ≤255 chars |
| `release_name` | string | No | ≤255 chars |
| `release_notes` | string | No | markdown, ≤125,000 chars |
| `environment` | string | No | environment the release deploys to; defaults to `production` |
| `files` | array of paths | No | absolute or relative; each path must exist and be a regular file |

`--json` prints a single object on stdout (logs go to stderr):

```json
{
  "release": { /* the published release */ },
  "deployed": [
    {"release_id": "…", "version": "v1.4.0", "to": "production", "action": "attach", "artifacts": ["page", "releases"]}
  ],
  "files": [
    {"path": "./dist/yard-darwin-arm64.tar.gz", "status": "uploaded", "size_bytes": 12345678},
    {"path": "./dist/yard-linux-amd64.tar.gz",  "status": "failed",   "error": "open …: no such file"}
  ],
  "uploaded": 1,
  "failed":   1
}
```

### Flag mode

```sh
yard releases publish v1.4.0 \
  --project my-slug \
  --name "Late April fixes" \
  --notes-file ./CHANGELOG.md \
  --file ./dist/yard-darwin-arm64.tar.gz \
  --file ./dist/yard-linux-amd64.tar.gz
```

### Interactive mode

```sh
yard releases publish
# prompts for: project (if multiple), tag, name, notes, files
```

### How upload failures are handled

The CLI runs a **two-step publish**: it streams each file into the draft
release first, then publishes the draft. A failed upload never leaves a
half-described release — the published release is whatever the draft holds at
publish time, including files uploaded on earlier runs. You'll see a per-file
summary on stderr and the JSON output marks each file `uploaded` or `failed`.

Exit codes:

- `0` — all files (if any) uploaded and the release was published.
- non-zero — at least one file failed. If any file uploaded, the draft was
  still published (without the failed files); if every file failed, nothing
  was published and the draft stays open. Re-run or add the missing assets
  from the dashboard.

### Promoting to another environment

`yard releases promote <tag> --to <env>` attaches an already-published release
to another environment's set — as the newest member it starts serving there
(unless the environment is pinned to something else). Nothing is copied:
storage is not consumed twice and download counts carry over. There is no
source environment to name — the tag alone identifies the release.

```sh
yard releases publish v1.4.0 --file dist/app.zip              # live to buyers
```

To check it before buyers do, publish to an environment of your own and promote
when it looks right:

```sh
yard env create preview
yard releases publish v1.4.0 --env preview --file dist/app.zip
# …verify the download works…
yard releases promote v1.4.0 --to production                  # ships it to buyers
```

---

## Syncing releases from GitHub

A project linked to a GitHub repository (Yard GitHub App installed, repo linked
from the dashboard's Releases tab) publishes automatically: when the seller
publishes a GitHub release, Yard receives the webhook and creates a matching
**published** release — tag, title, notes, and every attached asset are copied
into Yard's own storage. Editing the GitHub release re-syncs it; deleting it
**archives** the Yard release (buyers keep their downloads).

### Code and pricing from the repo

If the repository contains `.yard/settings.json` at the released tag — the same
file `yard init` writes and `yard push` uploads — the sync also imports what
that file declares:

| settings.json section | What syncs |
|---|---|
| `service.dir` | The service bundle in that directory (must contain `_worker.js`) becomes the release's service |
| `landing_page.dir` — or files under the default `.yard/landing-page` | Those files become the release's landing page |
| `pricing.tiers` | The release's pricing tiers are replaced to **match the array exactly** — tiers missing from the file are removed |
| `downloads.buttons` | The release's download buttons are replaced to **match the array exactly**; rules missing from the file are removed |

Each section is independent, and **absent means "not managed from GitHub"**:
the release carries that part forward unchanged, and removing it stays a
dashboard/CLI operation. A repo with no `.yard/settings.json` syncs assets,
name, and notes only. A section that IS declared must resolve — a declared dir
with no files at the tag fails the sync (typo protection), as does a service
bundle without `_worker.js`.

The `pricing` section uses the release-document tier shape (note the nested
`free_trial` object — this differs from the flat `free_trial_enabled` fields in
`yard init --spec`). Array order is the display order:

```json
{
  "version": 4,
  "project_slug": "my-project",
  "service": { "dir": "service" },
  "pricing": {
    "tiers": [
      { "name": "Personal", "price_cents": 900, "is_default": true,
        "pricing_model": "one_time", "features": ["Lifetime updates"] },
      { "name": "Team", "price_cents": 4900, "pricing_model": "subscription",
        "seat_type": "per_seat", "min_seats": 2, "yearly_discount_percent": 20,
        "free_trial": { "enabled": true, "days": 14, "requires_card": true } }
    ]
  },
  "downloads": {
    "buttons": [
      { "condition": "ends_with", "value": ".dmg", "label": "Download for Mac" },
      { "condition": "has_extension", "value": "exe", "label": "Download for Windows" }
    ]
  }
}
```

Notes:

- Normal pricing rules apply, validated against the **seller's plan**
  (tier count, seat-based pricing, trials — all plan-gated as usual); existing
  subscribers get the standard 30-day price-change notice when a sync changes
  their tier's price.
- Emptying `tiers` takes the project off sale and applies like any other
  change — a project with no tiers is a landing page that isn't for sale, which
  is a supported state. Existing purchases and subscriptions keep resolving
  against the tiers they were bought on.
- Tag content is immutable, so a settings.json change lands with the **next**
  release (or via Re-sync after force-moving a tag).
- `downloads.buttons` rules match release files by `condition`
  (`contains` | `starts_with` | `ends_with` | `has_extension`, case-insensitive)
  and `value` (1-255 chars), and label the button (`label`, 1-50 chars); max 10
  rules. Rules that match none of the release's files trigger the standard
  unmatched-button warning email on publish or sync.
- `yard push` applies the `pricing` and `downloads` sections the same way — CLI
  and GitHub sync read the same file identically.

### Local edits and Re-sync

A GitHub-sourced release edited from the dashboard is marked **Modified since
sync**, and automatic syncs skip it so local changes are never silently
overwritten. The dashboard's **Re-sync** button (after a confirmation)
restores the GitHub-managed parts, keeps everything settings.json doesn't
declare, and clears the marker.

---

## Downloading releases — license-key path

Use this path for shipped software that downloads updates. It requires the
project to issue a license key per buyer (i.e. `license_key_enabled: true`).
Each buyer's key is unique, so revocation, activation limits, and per-user
analytics work out of the box, and there's no shared secret to embed in the
binary.

### Get latest release metadata

```
GET https://api.yard.sh/v1/updates/latest
Authorization: Bearer <license_key>
```

Or, if header injection is awkward (some embedded HTTP clients), pass it as a
query parameter:

```
GET https://api.yard.sh/v1/updates/latest?license_key=<license_key>
```

**Environments.** Both update endpoints accept an optional `environment`
parameter and default to `production`, so an updater that omits it always gets
the live build:

```
GET https://api.yard.sh/v1/updates/latest?license_key=<license_key>&environment=beta
```

Environment **visibility** decides who can read a channel. A `public`
environment answers any valid license key for the project — that is how open
beta channels work: hand testers the license key they already have and point
their updater at the beta environment. A `private` environment — Production
included — answers only license keys held by a member of the owning team (and
the project's test license key); everyone else gets the same `404` an unknown
slug gets, so a private channel's existence never leaks. Unpublished work is
never reachable: draft releases can't belong to any environment, and each
environment only serves releases attached to it.

**Response** (shape mirrors the GitHub Releases API for easy adoption of
existing tooling):

```json
{
  "tag_name":     "v1.4.0",
  "name":         "Late April fixes",
  "body":         "## Highlights\n…",
  "body_html":    "<h2>Highlights</h2>…",
  "draft":        false,
  "prerelease":   false,
  "immutable":    false,
  "created_at":   "2026-04-28T16:32:11Z",
  "published_at": "2026-04-28T16:32:11Z",
  "assets": [
    {
      "name":               "yard-darwin-arm64.tar.gz",
      "content_type":       "application/gzip",
      "size":               12345678,
      "download_count":     0,
      "created_at":         "2026-04-28T16:32:14Z",
      "browser_download_url": "https://api.yard.sh/v1/updates/latest/download/yard-darwin-arm64.tar.gz?license_key=…"
    }
  ]
}
```

Errors:

- `400 Missing license key` — neither `?license_key=` nor `Authorization: Bearer` was set.
- `400 Invalid license key format` — the key doesn't match the expected format.
- `403 Purchase not completed` — the license exists but the buyer's payment hasn't finalized.
- `403 License has been refunded` — the seller refunded the buyer; access revoked.
- `403 Trial period has expired` — the key came from a trial that has ended.
- `404 No releases found` — the project has no synced, non-archived release yet.
- `404 License key not found` — the key isn't in our system.
- `404 Environment "…" not found` — the slug doesn't exist, or the environment is private and the key's holder is not a member of the owning team (deliberately indistinguishable).

### Download a release file

Hit `browser_download_url` from the response above (or build it manually):

```
GET https://api.yard.sh/v1/updates/latest/download/{filename}?license_key=<license_key>
```

The endpoint returns `302 Found` with a presigned URL pointing to the storage
bucket; follow the redirect (most HTTP clients do this automatically). The
presigned URL expires after 5 minutes.

### List environments (update channels)

```
GET https://api.yard.sh/v1/updates/environments?license_key=<license_key>
```

Lists the environments the key may see, so an app can offer a channel picker.
Private environments are simply omitted for non-members; the list can be empty.

```json
{
  "environments": [
    { "slug": "Production", "visibility": "public", "protected": true,
      "current_version": "v1.4.0", "current_published_at": "2026-04-28T16:32:11Z" },
    { "slug": "beta", "visibility": "public", "protected": false,
      "current_version": "v1.5.0-beta.1", "current_published_at": "2026-05-02T09:00:00Z" }
  ]
}
```

### List releases in an environment

```
GET https://api.yard.sh/v1/updates/releases?license_key=<license_key>&environment=beta
```

Returns a **bare JSON array** of releases attached to the environment, newest
first, each in the same GitHub-Release shape as `/v1/updates/latest`. Archived
releases are excluded (that is also what keeps versions unique in download
URLs). Paginate with `page` (default 1) and `limit` (default 50, max 100).
Every asset's `browser_download_url` already carries the license key, version,
and environment — use it verbatim.

### Download a file from a specific release

```
GET https://api.yard.sh/v1/updates/releases/{version}/download/{filename}?license_key=<license_key>&environment=beta
```

Downloads by version rather than "latest". The release must be attached to the
requested environment, or the endpoint returns `404 Release not found`. Same
`302`-to-presigned-URL behavior as the latest-download endpoint.

---

## Creating an API key

```sh
yard keys create ci-runner --scopes licenses:validate,licenses:activate
```

Or, non-interactively from a spec:

```sh
echo '{"name":"ci-runner","scopes":["licenses:validate","licenses:activate"]}' \
  | yard keys create --spec - --json
```

**The full secret is shown only once** at creation time. Copy it from stdout
(or the `key` field in `--json` output) before closing the terminal — the
backend never returns it again. Subsequent `yard keys list` calls show only
the prefix (`yard_a1b2c3d`) and metadata.

### Available scopes

| Scope | Description |
|---|---|
| `projects:read` | Read project metadata |
| `licenses:validate` | Validate license keys (called from your own software) |
| `licenses:activate` | Activate / deactivate license keys |
| `subscriptions:read` | Read project subscription status |
| `subscriptions:write` | Create / cancel / reactivate project subscriptions |

Validation rules:

- `name` is required and ≤100 chars.
- At least one valid scope is required.
- Each user is capped at 100 API keys (`MaxAPIKeysPerUser`).

---

## Listing API keys

```sh
yard keys list
```

Prints a table with the same columns the dashboard shows (name, prefix,
scopes, last-used, created). The full key is never displayed.

```sh
yard keys list --json
```

Emits the raw `APIKeyListResponse`:

```json
{
  "api_keys": [
    {
      "id": "5f8b…",
      "name": "ci-runner",
      "key_prefix": "yard_a1b2c3d",
      "scopes": ["licenses:validate", "licenses:activate"],
      "last_used_at": "2026-04-28T16:32:11Z",
      "created_at":   "2026-04-12T09:14:02Z"
    }
  ],
  "count": 1,
  "limit": 100
}
```

---

## Troubleshooting

- **`400 Missing license key` on `/v1/updates/latest`** — neither query nor `Authorization: Bearer` header was sent. Most update libraries default to query-param auth; double-check that the key actually got injected.
- **`403 License has been refunded`** — the seller refunded the buyer; the license is permanently revoked. Surface this to the user and invite them to re-purchase.
- **`404 No releases found for this project`** — nothing has shipped to production yet. Run `yard releases publish` and then `yard releases promote <tag> --to production` (or publish with `--env production` in one step).
- **Storage-limit `403` on publish** — the selling team's plan has a storage cap. Either upgrade the plan or delete old release files from the dashboard.
- **API key gone, can't re-read it** — keys are unrecoverable by design. Run `yard keys create` to mint a new one (and update wherever the old one was embedded).
- **Wrong scopes on an existing key** — there's no CLI command to edit scopes today; edit the key from the dashboard or delete + recreate via the CLI.
