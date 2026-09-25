# Releases and the Update Server

How a project's releases work, how they reach buyers, how GitHub releases sync in, and how shipped software downloads updates with a buyer's license key. Command flags and JSON shapes are in [cli-commands.md](cli-commands.md#yard-releases).

## What a release is

A release is a **project-wide snapshot**: landing page, pricing, download buttons, services and file assets together. It has a `tag_name` (≤255 chars, unique per project across non-archived releases; reuse is a `409`), an optional `release_name` (≤255) and `release_notes` (markdown, ≤125,000), and zero or more files stored by Yard.

- Every release starts as a **draft**. Nothing serves a draft, so editing one has no side effects. A project can hold 10 open drafts.
- **Publishing** stamps the tag and puts the release in one **channel**. Publishing is one-way, but the release stays editable: editing one that nothing serves is harmless, editing one that is served is live on save.
- Releases belong to the project, never to a sandbox. The project and each sandbox **follow one channel** and serve its newest release, unless **rolled back** (until the next release lands in the channel) or **pinned** (until unpinned). Landing in a followed channel is the deploy moment.
- Every project has a protected `Production` channel that it follows, so `yard releases publish <tag>` (which defaults to `Production`) is the go-live step. Other channels are created, renamed, deleted and reordered in the dashboard's Releases tab; deleting one reassigns its releases (to `Production` by default).

## Publishing, promoting, rolling back

```sh
yard releases publish v1.4.0 --file dist/app.zip   # live to buyers
yard releases promote v1.4.0 --to Beta             # move it to Beta (nothing is copied; download counts carry over)
```

`publish` uploads the files into your draft first, then publishes it; a failed upload never leaves a half-described release. Try a release before buyers see it:

```sh
yard sandbox create preview
yard sandbox pin                              # the storefront stays on what it serves
yard releases publish v1.4.0 --file dist/app.zip
yard sandbox pin v1.4.0 --sandbox preview     # …verify…
yard sandbox unpin                            # ships it
```

A bad release is live: `yard sandbox rollback v1.3.0` puts the earlier one back now, and the fix takes over by itself when you publish it. `yard sandbox pin <tag>` instead holds a release through later publishes (`pin` with no tag makes a rollback permanent). A rollback is refused while pinned.

## Syncing releases from GitHub

With the Yard GitHub App installed and the repo linked (dashboard, Releases tab), publishing a GitHub release creates a matching **published** Yard release: tag, title, notes and every asset are copied. Editing the GitHub release re-syncs it; deleting it archives the Yard release (buyers keep their downloads). Synced releases land in the project's **GitHub sync channel** (set on the Releases tab, default `Production`, so publishing on GitHub ships to buyers).

If the repo has `.yard/settings.json` at the tag, the sync also imports what it declares:

| Section | What syncs |
| --- | --- |
| `services[]` | Each entry becomes a service built from its directory (which needs `_service.js`). The list is the whole set: a service the tag drops is taken down on deploy. |
| `landing_page` | `type` and the files in `dir` (or under the default `.yard/landing-page/`), as for `yard push`. |
| `pricing.tiers` | Replaces the release's tiers to **match exactly**. |
| `downloads.buttons` | Replaces the release's download buttons to **match exactly**. |

Each section is independent, and **absent means not managed from GitHub**: the release keeps that part as it was. A repo with no settings.json syncs assets, name and notes only. A declared section must resolve: an empty declared directory, a service without `_service.js` or `name`, or two services with the same name or path fails the sync; a `custom` page without the plan feature records an upgrade-required sync error. `yard push` applies `pricing` and `downloads` the same way.

The `pricing` section uses the release tier shape: a nested `free_trial` object instead of `yard init`'s flat trial fields. Array order is display order.

```json
{
  "version": 7,
  "project_slug": "my-project",
  "pricing": {
    "tiers": [
      { "name": "Personal", "price_cents": 900, "is_default": true, "pricing_model": "one_time", "features": ["Lifetime updates"] },
      { "name": "Team", "price_cents": 4900, "pricing_model": "subscription", "seat_type": "per_seat", "min_seats": 2,
        "yearly_discount_percent": 20, "free_trial": { "enabled": true, "days": 14, "requires_card": true } }
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

- Pricing is validated against the team's plan; subscribers get the standard 30-day notice when a sync changes their tier's price. An empty `tiers` takes the project off sale (existing purchases keep resolving).
- `downloads.buttons`: `condition` is `contains`, `starts_with`, `ends_with` or `has_extension` (case-insensitive), `value` 1-255 chars, `label` 1-50 chars, at most 10. A rule matching no file triggers a warning email.
- Tags are immutable, so a settings.json change lands with the **next** release (or a Re-sync after moving the tag). A tag with a retired settings layout fails with an error naming the fix (`yard migrate`, commit, tag again).
- A synced release edited in the dashboard is marked **Modified since sync** and skipped by automatic syncs; **Re-sync** restores the GitHub-managed parts.
- Dashboard edits to pricing, download buttons or services regenerate every release's settings.json, so `yard status` shows them as a config diff and `yard pull` retrieves them.

## Downloading releases with a license key

For shipped software that updates itself. The project must issue license keys (`license_key_enabled`); each buyer's key is their credential, so revocation, activation limits and per-user limits come for free and no shared secret ships in the binary.

```
GET https://api.yard.sh/v1/updates/latest
Authorization: Bearer <license_key>
```

or `GET …/v1/updates/latest?license_key=<license_key>` when headers are awkward. The response mirrors GitHub's Releases API:

```json
{
  "tag_name": "v1.4.0", "name": "Late April fixes", "body": "## Highlights\n…", "body_html": "<h2>…",
  "draft": false, "prerelease": false, "created_at": "2026-04-28T16:32:11Z", "published_at": "2026-04-28T16:32:11Z",
  "assets": [{ "name": "app-darwin-arm64.tar.gz", "content_type": "application/gzip", "size": 12345678,
    "download_count": 0, "browser_download_url": "https://api.yard.sh/v1/updates/latest/download/app-darwin-arm64.tar.gz?license_key=…" }]
}
```

Download with `browser_download_url` (or `GET /v1/updates/latest/download/{filename}?license_key=…`): a `302` to a short-lived URL (5 minutes) that HTTP clients follow automatically.

**Sandboxes.** Every update endpoint takes an optional `sandbox` parameter; omitted, it reads the project itself (the live build). A `public` sandbox answers any key entitled to it (an open beta stream: mint testers keys in the beta sandbox and point their updater at it). Anything `private`, a private project included, answers only keys held by members of the owning team, and everyone else gets the same `404` as an unknown name. A key reaches only where its purchase lives, both ways: a sandbox key cannot read the project's builds and a real key cannot read a sandbox's. To test an updater against a sandbox, buy the project inside it (free, simulated). Drafts are never reachable.

Other endpoints (all take `license_key` and optional `sandbox`):

- `GET /v1/updates/sandboxes`: the streams the key may see, for a stream picker. `{"global": {"visibility", "current_version", "current_published_at"}, "sandboxes": [{"slug", …}]}`, where `global` is the project itself; private sandboxes are omitted for non-members.
- `GET /v1/updates/releases`: a bare array of that stream's releases, newest first, same shape; archived releases excluded; `page` (default 1) and `limit` (default 50, max 100). Use each `browser_download_url` verbatim.
- `GET /v1/updates/releases/{version}/download/{filename}`: a file from a specific release (`404 Release not found` if the stream doesn't hold it).

Errors: `400 Missing license key` / `Invalid license key format`; `403 Purchase not completed` / `License has been refunded` (revoked for good; invite a re-purchase) / `Trial period has expired`; `404 No releases found` (nothing published to that stream yet: `yard releases publish`), `License key not found`, `Sandbox "…" not found` (unknown, or private and the holder isn't on the team).

## Troubleshooting

- **`400 Missing license key`:** most update libraries default to the query parameter; check the key was actually added.
- **Storage-limit `403` on publish:** the team's plan caps storage; upgrade or delete old release files in the dashboard.
- **Lost API key secret:** unrecoverable by design; `yard keys create` a new one and replace it where it was embedded. Scopes are edited in the dashboard (or delete and recreate).
