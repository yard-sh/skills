# Releases and the Update Server

This reference covers how to publish a release for a Yard product (with file
assets) and how shipped software downloads those releases. Two auth approaches
work for downloading: **license keys** (per-buyer, hit `/v1/updates/latest`)
and **API keys** (per-seller, hit `/v1/products/{id}/releases/...`). Either
fits shipped software — pick by what suits the product. API keys are also the
only option for first-party automation (CI, internal tooling).

## Table of Contents

- [What a release is](#what-a-release-is)
- [Publishing a release with the CLI](#publishing-a-release-with-the-cli)
- [Downloading releases — license-key path](#downloading-releases--license-key-path)
- [Downloading releases — API-key path](#downloading-releases--api-key-path)
- [Creating an API key](#creating-an-api-key)
- [Listing API keys](#listing-api-keys)
- [Troubleshooting](#troubleshooting)

---

## What a release is

A Yard release belongs to a single product and consists of:

- **`tag_name`** — required, ≤255 chars (e.g. `v1.4.0`)
- **`release_name`** — optional human title (≤255 chars)
- **`release_notes`** — optional markdown body (≤125,000 chars)
- **`files`** — zero or more file assets uploaded to the seller's storage bucket

There is **no draft state** — releases go live immediately on creation. The
newest non-archived, non-stale, "synced" release for a product is automatically
the "latest" release.

Duplicate `tag_name` values are *allowed* on the same product (no unique
constraint), but it's poor practice — pick distinct tags.

---

## Publishing a release with the CLI

`yard releases publish` is the canonical command. It supports three usage
patterns: full interactive, flag-driven, and `--spec` JSON for agents/scripts.

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
| `files` | array of paths | No | absolute or relative; each path must exist and be a regular file |

`--json` prints a single object on stdout (logs go to stderr):

```json
{
  "release": { /* ReleaseResponse */ },
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

The CLI runs a **two-step publish**: it creates the release first (JSON), then
streams each file to `POST /v1/products/{id}/releases/{releaseId}/files` one at
a time. A single failed upload does **not** delete the release — you'll see a
per-file summary on stderr and the JSON output marks each file `uploaded` or
`failed`.

Exit codes:

- `0` — release created and all files (if any) uploaded.
- non-zero — release was created but at least one file failed; the release
  exists in the dashboard and you can re-upload missing assets there.

---

## Downloading releases — license-key path

Use this path when your software issues a license key per buyer (i.e.
`license_key_enabled: true` on the product). It's the easiest auth for shipped
software — each buyer's key is unique, so revocation, activation limits, and
per-user analytics work out of the box, and there's no shared secret to embed
in the binary. If the product doesn't issue license keys, or you'd rather skip
the per-buyer-key UX, the API-key path below works equally well — it just
hits a different endpoint.

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

## Downloading releases — API-key path

API keys work in two scenarios: **first-party automation** (CI scripts,
internal tooling, integration tests) and **shipped software** where you'd
rather embed a single key than have each buyer authenticate with a license
key. Tradeoff for shipped software: every install carries the same key, so
you can't revoke or rate-limit per customer the way you can with license
keys — pick this path when that tradeoff is acceptable, or when the product
doesn't issue license keys at all.

All endpoints are under the seller's product:

| Method | Path | Required scope |
|---|---|---|
| `GET` | `/v1/products/{productId}/releases` | `releases:read` |
| `GET` | `/v1/products/{productId}/releases/{releaseId}` | `releases:read` |
| `GET` | `/v1/products/{productId}/releases/latest` | `releases:download` |
| `GET` | `/v1/products/{productId}/releases/{releaseId}/files/{fileId}/download` | `releases:download` |
| `GET` | `/v1/products/{productId}/releases/latest/files/{fileId}/download` | `releases:download` |

Auth header on every request:

```
Authorization: Bearer yard_<full-key>
```

Download endpoints behave the same as the license-key path — they redirect
(`302`) to a 5-minute presigned URL.

---

## Creating an API key

```sh
yard keys create ci-runner --scopes releases:read,releases:download
```

Or, non-interactively from a spec:

```sh
echo '{"name":"ci-runner","scopes":["releases:read","releases:download"]}' \
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
| `releases:read` | Read release metadata for owned products |
| `releases:download` | Download release files for owned products |
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
      "scopes": ["releases:read", "releases:download"],
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
- **`404 No releases found for this product`** — the product has zero non-archived, synced releases. For new products, run `yard releases publish` to ship the first one.
- **Storage-limit `403` on publish** — the seller's plan has a storage cap. Either upgrade the plan or delete old release files from the dashboard.
- **API key gone, can't re-read it** — keys are unrecoverable by design. Run `yard keys create` to mint a new one (and update wherever the old one was embedded).
- **Wrong scopes on an existing key** — there's no CLI command to edit scopes today; edit the key from the dashboard or delete + recreate via the CLI.
