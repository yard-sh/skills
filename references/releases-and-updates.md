# Releases and the Update Server

This reference covers how to publish a release for a Yard product (with file
assets) and how shipped software downloads those releases. Downloads
authenticate with **license keys** (per-buyer, hit `/v1/updates/latest`).
API keys cover first-party automation for licenses and subscriptions; they do
not grant release access.

## Table of Contents

- [What a release is](#what-a-release-is)
- [Publishing a release with the CLI](#publishing-a-release-with-the-cli)
- [Downloading releases — license-key path](#downloading-releases--license-key-path)
- [Creating an API key](#creating-an-api-key)
- [Listing API keys](#listing-api-keys)
- [Troubleshooting](#troubleshooting)

---

## What a release is

A Yard release is a **product-wide snapshot** — landing page, pricing, download
buttons, web-app bundle, and file assets all live in one release. It consists
of:

- **`tag_name`** (version) — ≤255 chars (e.g. `v1.4.0`). Required to publish; a
  draft may hold a tentative tag or none yet.
- **`release_name`** — optional human title (≤255 chars)
- **`release_notes`** — optional markdown body (≤125,000 chars)
- **`files`** — zero or more file assets uploaded to the seller's storage bucket

Every release starts as a **draft**: an ordinary release that simply has not
been published yet. Drafts are the only editable releases and can never be
deployed anywhere, so editing one has no side effects. Publishing stamps the
tag and freezes the release into an immutable snapshot — publishing is
**one-way**. A product can hold at most **10 open drafts**.

Releases belong to the **product**, not to any environment. Each environment
holds a **set** of releases and serves the newest member of that set, unless
the seller explicitly pins an older one. **Attaching a release to an
environment is the deploy moment.** Buyers only ever see what **production**
serves — attach a release there (`yard releases promote <tag> --to
production`, or publish with `--env production`) to make it live.

Version tags are **unique per product** across all non-archived releases —
publishing under an existing tag (or tagging a draft with one) is rejected
with `409`. A draft may reserve its tag early.

---

## Publishing a release with the CLI

`yard releases publish` is the canonical command. It supports three usage
patterns: full interactive, flag-driven, and `--spec` JSON for agents/scripts.

The CLI's working area is a **draft release**: `publish` uploads each file into
your open draft (creating one seeded from what the target environment serves if
you have none; `--release <id|tag>` names one explicitly), then publishes the
draft under the tag and attaches it to the target environment (`--env` /
`environment`, default `development`). Anything `yard page push` or
`yard app deploy` already staged in that draft ships with it.

### Spec mode (recommended for agents)

```sh
echo '{
  "product":      "my-slug",
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
| `product` | string | Only if you have multiple products | slug or UUID |
| `tag_name` | string | Yes | ≤255 chars |
| `release_name` | string | No | ≤255 chars |
| `release_notes` | string | No | markdown, ≤125,000 chars |
| `environment` | string | No | environment the release deploys to; defaults to `development` |
| `files` | array of paths | No | absolute or relative; each path must exist and be a regular file |

`--json` prints a single object on stdout (logs go to stderr):

```json
{
  "release": { /* the published release */ },
  "deployed": [
    {"release_id": "…", "version": "v1.4.0", "to": "development", "action": "attach", "artifacts": ["page", "releases"]}
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
  --product my-slug \
  --name "Late April fixes" \
  --notes-file ./CHANGELOG.md \
  --file ./dist/yard-darwin-arm64.tar.gz \
  --file ./dist/yard-linux-amd64.tar.gz
```

### Interactive mode

```sh
yard releases publish
# prompts for: product (if multiple), tag, name, notes, files
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
yard releases publish v1.4.0 --file dist/app.zip   # publishes + deploys to development
# …verify the download works…
yard releases promote v1.4.0 --to production       # ships it to buyers
```

---

## Downloading releases — license-key path

Use this path for shipped software that downloads updates. It requires the
product to issue a license key per buyer (i.e. `license_key_enabled: true`).
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

Any valid license key for the product can read any of its environments — that is
how beta channels work: hand testers the license key they already have and point
their updater at a beta environment. Unpublished work is never reachable: draft
releases can't belong to any environment, and each environment only serves
releases attached to it. A slug that doesn't exist returns `404`.

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
- `404 No releases found` — the product has no synced, non-archived release yet.
- `404 License key not found` — the key isn't in our system.

### Download a release file

Hit `browser_download_url` from the response above (or build it manually):

```
GET https://api.yard.sh/v1/updates/latest/download/{filename}?license_key=<license_key>
```

The endpoint returns `302 Found` with a presigned URL pointing to the storage
bucket; follow the redirect (most HTTP clients do this automatically). The
presigned URL expires after 5 minutes.

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
| `products:read` | Read product metadata |
| `licenses:validate` | Validate license keys (called from your own software) |
| `licenses:activate` | Activate / deactivate license keys |
| `subscriptions:read` | Read product subscription status |
| `subscriptions:write` | Create / cancel / reactivate product subscriptions |

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
- **`404 No releases found for this product`** — nothing has shipped to production yet. Run `yard releases publish` and then `yard releases promote <tag> --to production` (or publish with `--env production` in one step).
- **Storage-limit `403` on publish** — the seller's plan has a storage cap. Either upgrade the plan or delete old release files from the dashboard.
- **API key gone, can't re-read it** — keys are unrecoverable by design. Run `yard keys create` to mint a new one (and update wherever the old one was embedded).
- **Wrong scopes on an existing key** — there's no CLI command to edit scopes today; edit the key from the dashboard or delete + recreate via the CLI.
