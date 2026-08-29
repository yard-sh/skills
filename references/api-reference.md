# Yard API Reference

> **What this API covers.** The Yard REST API is the **integration surface** — it lets a seller's shipped software (or an agent working on that software) validate licenses, read release metadata, and manage buyer subscriptions. It is **not** used to manage a seller's own Yard catalog. Project, release, and coupon management — plus reading the seller's customers and sales — happen through the **Yard CLI** (`yard init`, `yard projects`, `yard coupons`, `yard customers`, `yard transactions`, `yard push / yard pull`) — see [cli-commands.md](./cli-commands.md).
>
> Create an API key with the scopes you need at **https://yard.sh/dashboard/api-keys?action=create**.

## Base URL

```
https://api.yard.sh
```

All API paths below are relative to this base URL (e.g., `/v1/licenses/validate` means `https://api.yard.sh/v1/licenses/validate`).

## `{username}` — how projects are addressed

Projects are owned by a **team**, and a project is addressed by its owning team's username plus its slug: `/v1/projects/{username}/{slug}/…`, matching the public URL `https://yard.sh/@{username}/{slug}` and the subdomain `https://{username}.yard.sh/{slug}`.

A team username is **not** a user's username. They share one namespace (so neither can collide with the other), but a team username is what resolves here, and a seller's personal username resolves nothing unless they happen to own a team with the same username. Read the value from `yard team --json` → `.active_team.username`, or from `seller.username` on the public project response — never assume it matches the signed-in user.

---

## The `sandbox` parameter

A project has one set of real data - its own - plus any number of **sandboxes**, each an optional copy of the project's landing page, pricing, service, database and commerce. Endpoints that read or write project state resolve the `sandbox` parameter the same way everywhere: **omitted or empty means the project itself**, which is what buyers reach and the only place where money is real. Naming a sandbox selects it, and its commerce is simulated by the platform (see [pricing-and-licensing.md](pricing-and-licensing.md#commerce-in-a-sandbox)).

The parameter travels in the query string on `GET`s and on the subscription-management `POST`s, and in the JSON body on the checkout endpoints:

| Endpoint | Where `sandbox` goes |
|---|---|
| `GET /v1/updates/latest`, `/v1/updates/sandboxes`, `/v1/updates/releases`, and both download paths | query string |
| `GET /v1/projects/{username}/{slug}/public` | query string |
| `GET /v1/projects/{username}/{slug}/subscription` | query string |
| `POST /v1/projects/{username}/{slug}/subscription/cancel` \| `/reactivate` \| `/change` | query string |
| `POST /v1/subscription-intent` | JSON body (`"sandbox": "preview"`) |
| The buyer's download and library endpoints | query string |

A sandbox that does not exist is a `404` naming it. A **private** sandbox, the default for a new one, answers only callers who belong to the project's owning team, and answers everyone else with `404`, as if it did not exist; a **public** sandbox answers anyone with the URL. Responses that resolve a sandbox are never cacheable, because the same URL can answer differently to a stranger and to a team member, and visibility can flip at any moment.

---

## Authentication

### API Key (recommended for integrations)

```
Authorization: Bearer yard_{key}
```

API keys start with the `yard_` prefix and are issued **per team** in the dashboard at https://yard.sh/dashboard/api-keys?action=create — a key is a team credential pinned to the team that created it, so it stays valid when the person who minted it leaves. Every integration request carries the key in the `Authorization` header with the `Bearer ` prefix.

**Scopes** — each key is limited to the actions it actually needs. Pick only what you use:

| Scope | What it allows |
|-------|----------------|
| `projects:read` | Read a project's metadata |
| `licenses:validate` | Validate a license key |
| `licenses:activate` | Activate or deactivate a device against a license |
| `subscriptions:read` | Read a buyer's project subscription status |
| `subscriptions:write` | Create, cancel, or reactivate a buyer's project subscription |

Catalog management scopes do **not** exist — project create / update / delete are CLI-only.

### Session Token (CLI and dashboard only)

```
Authorization: Session {token}
```

The session token is a 64-character hex string (32 random bytes) issued to the CLI (`yard login`) and to the web dashboard. Sessions expire after 30 days. The CLI stores the token in `~/.yard/config.json`. **Third-party integrations should not use session tokens** — use an API key instead.

---

## API-Key Endpoints

Everything below accepts `Authorization: Bearer yard_...` with the listed scope. These are the endpoints you integrate into your software.

### Projects

| Method | Path | Scope | Description |
|---|---|---|---|
| `GET` | `/v1/projects/{username}/{slug}/metadata` | `projects:read` | Read project metadata (title, stage, tiers, pricing) |

### Licenses

| Method | Path | Scope | Description |
|---|---|---|---|
| `POST` | `/v1/licenses/validate` | `licenses:validate` | Validate a license key (optionally bind to a device) |
| `POST` | `/v1/licenses/deactivate` | `licenses:activate` | Deactivate a device from a license |

`POST /v1/licenses/validate` answers `valid: true` for any live key, including one minted by a **simulated purchase inside a sandbox**. The response carries a `sandbox` field: absent or empty for a real purchase on the project itself, a sandbox name for a simulated purchase inside that sandbox. Software that grants entitlement has to check it, or a simulated purchase entitles someone for real. See [pricing-and-licensing.md](pricing-and-licensing.md#commerce-in-a-sandbox).


### Subscriptions (buyer-facing)

| Method | Path | Scope | Description |
|---|---|---|---|
| `POST` | `/v1/subscription-intent` | `subscriptions:write` | Create a subscription payment intent |
| `GET` | `/v1/projects/{username}/{slug}/subscription` | `subscriptions:read` | Read a buyer's subscription status for a project |
| `POST` | `/v1/projects/{username}/{slug}/subscription/cancel` | `subscriptions:write` | Cancel a buyer's subscription |
| `POST` | `/v1/projects/{username}/{slug}/subscription/reactivate` | `subscriptions:write` | Reactivate a cancelled subscription |

---

## License-Gated Endpoints (no auth header)

Built-in updaters in the seller's software can reach these directly with just a license key. No `Authorization` header.

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/updates/latest?license_key={key}` | Check for the latest release by license key |
| `GET` | `/v1/updates/latest/download/{filename}?license_key={key}` | Download the latest release file by license key |
| `GET` | `/v1/updates/sandboxes?license_key={key}` | List the update streams (the project itself plus each sandbox) the key may see |
| `GET` | `/v1/updates/releases?license_key={key}` | List the project's or one sandbox's releases (GitHub Releases list shape) |
| `GET` | `/v1/updates/releases/{version}/download/{filename}?license_key={key}` | Download a file from a specific release |

All of these accept an optional `sandbox` parameter (see [The `sandbox` parameter](#the-sandbox-parameter)). Omitting it reads the project itself, which is what buyers get. Anything private - a private project included - answers only license keys held by a member of the project's owning team; everyone else gets a 404, as if it doesn't exist. A key also reaches only where its own purchase lives, in both directions: a sandbox key cannot pull the project's own artifacts, and a real buyer's key cannot pull a sandbox's.

---

## Public Endpoints (no auth)

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Health check |
| `GET` | `/ready` | Readiness check |
| `GET` | `/version` | API version info |
| `GET` | `/v1/projects/public` | List all public projects |
| `GET` | `/v1/projects/{username}/{slug}/public` | Get a public project (addressed under the owning team's username - this is the shape `window.yard.project` exposes) |
| `GET` | `/v1/teams/{id}` | Get a team's public profile and its projects. `{id}` is the team's UUID **or** its username (the subject is always a team, never an individual user) |
| `GET` | `/v1/search?q={query}` | Search projects |
| `POST` | `/v1/coupons/validate` | Validate a coupon code |

---

## CLI-only operations

The following are **not** exposed over HTTP as integration endpoints — an API key can't reach them, because they manage the seller's own catalog. If an agent needs to do any of these, it must run the CLI, not issue HTTP requests:

- Create / update / delete a project (`yard init`, project edits in the dashboard)
- Create, publish, or promote a release (`yard releases publish`, `yard releases promote`, the GitHub App on release webhook, or the dashboard)
- Create, rename, delete, or reorder a **release channel**: dashboard-only, and not even in the CLI, which can only list them (`yard channels list`). The endpoints behind the dashboard (`POST`/`PATCH`/`DELETE /v1/projects/{id}/channels…`) are session-authenticated and gated on the `sandboxes` permission; an API key cannot reach them
- Create, rename, delete, or configure a **sandbox**, and choose what the project or a sandbox serves (`yard sandbox …`, or the dashboard). Sandbox writes need the `sandboxes` permission and are capped by `max_sandboxes`; writes to the project itself need only ordinary project-write permission
- Create / update / delete / bulk-generate coupons (`yard coupons create`, `yard coupons generate`, `yard coupons update`, `yard coupons rm`)
- Read the seller's customers and sales (`yard customers`, `yard transactions`) — these are reporting on the selling team's own books, not an integration surface, so an API key can't reach them
- Lengthen or shorten a buyer's running free trial (`yard transactions trial <order-id> --add-days N`)
- Stripe Connect onboarding and payout management
- Custom domains, project images / videos, webhook secrets
- Custom landing page editing (`yard init --page`, `yard push`, …)

See [cli-commands.md](./cli-commands.md) for the full CLI surface.

---

## Error Response Format

All errors return a JSON body:

```json
{
  "error": "Human-readable error message"
}
```

Common HTTP status codes:
- `400` — Bad request (validation error)
- `401` — Unauthorized (missing or invalid token)
- `403` — Forbidden (insufficient scope, or endpoint requires session auth)
- `404` — Not found
- `409` — Conflict (e.g., duplicate resource)
- `429` — Rate limited
- `500` — Internal server error
