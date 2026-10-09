# Yard API Reference

> **What this API is for.** Integrating Yard into shipped software: validating licenses, reading release metadata, managing your users' existing subscriptions, reading and fulfilling the [products](products.md) they buy, and signing your users in with [Yard Auth](#yard-auth-for-external-apps). An agent managing the team's own catalog (projects, pricing, products, releases, pages, services, coupons, users, sales) uses the **Yard CLI** instead; see [cli-commands.md](./cli-commands.md).
>
> Create an API key with the scopes you need at **https://dash.yard.sh/configure/api-keys?action=create**.

## Base URL

```
https://api.yard.sh
```

All API paths below are relative to this base URL (e.g., `/v1/licenses/validate` means `https://api.yard.sh/v1/licenses/validate`).

## `{username}`: how projects are addressed

Projects are owned by a **team**, and a project is addressed by its owning team's username plus its slug: `/v1/projects/{username}/{slug}/…`, matching the public URL `https://yard.sh/@{username}/{slug}` and the subdomain `https://{username}.yard.sh/{slug}`.

A team username is **not** the user's username (they share one namespace but routinely differ). Read it from `yard team --json` → `.active_team.username` or `team.username` on the public project; never assume it matches the signed-in user.

---

## The `sandbox` parameter

A project has its own real data plus any number of **sandboxes**, each an optional copy with simulated commerce ([pricing-and-licensing.md](pricing-and-licensing.md#commerce-in-a-sandbox)). Everywhere, an **omitted or empty `sandbox` means the project itself**, the only place money is real; naming a sandbox selects it.

The parameter travels in the query string on `GET`s and on the subscription-management `POST`s:

| Endpoint | Where `sandbox` goes |
|---|---|
| `GET /v1/updates/latest`, `/v1/updates/sandboxes`, `/v1/updates/channels`, `/v1/updates/releases`, and both download paths | query string |
| `GET /v1/projects/{username}/{slug}/public` | query string |
| `GET /v1/projects/{username}/{slug}/subscription` | query string |
| `POST /v1/projects/{username}/{slug}/subscription/cancel` \| `/reactivate` \| `/change` | query string |
| `GET /v1/projects/{id}/users/{userDisplayId}/products`, `/fulfillments/pending`, `POST …/fulfillments/{transactionId}` | query string |
| Your users' download and library endpoints | query string |

An unknown sandbox is a `404` naming it. A **private** sandbox (the default) answers only members of the owning team and gives everyone else the same `404`; a **public** one answers anyone. Responses that resolve a sandbox are never cacheable.

---

## Authentication

### API Key (recommended for integrations)

```
Authorization: Bearer yard_{key}
```

API keys start with `yard_` and belong to a **team** (created with `yard keys create` or at https://dash.yard.sh/configure/api-keys?action=create); a key keeps working when the person who minted it leaves. Send it as `Authorization: Bearer yard_…`.

**Scopes:** a key reaches exactly the endpoints its scopes allow. A listed endpoint whose scope the key lacks answers `403` with `insufficient_scope`; an endpoint outside this reference answers `401` whatever the scopes. Scopes do not imply one another; pick only what you use. `yard keys create` prints the catalog.

Integration scopes are safe to ship inside the app your users run. Every install shares the key and its rate limit, so validate once at launch and keep the answer:

| Scope | What it allows |
|-------|----------------|
| `metadata:read` | Read a project's public metadata and pricing |
| `licenses:validate` | Validate a license key |
| `licenses:activate` | Activate or deactivate a device against a license |

Management scopes act on the team's account, its release files and its users; keep keys holding them on servers the team controls. That includes `releases:read` (downloads every file in a public channel without a purchase), the subscription scopes (act on the subscription of any user named by email) and the product scopes (read and fulfill any user's purchases). An app lists channels, checks for updates and downloads with the user's license key instead ([License-Gated Endpoints](#license-gated-endpoints-no-auth-header)):

| Scope | What it allows |
|-------|----------------|
| `projects:read` | List and read the team's projects, including drafts, pricing history and published tier and product keys |
| `projects:write` | Create, update and delete projects, and change pricing and page content |
| `releases:read` | List public channels and their releases, and download release files |
| `releases:write` | Create, edit, publish and archive releases, manage channels, and read draft contents |
| `sandboxes:read` | List sandboxes and the releases they serve |
| `sandboxes:write` | Create, rename and delete sandboxes, and pin or roll back what they serve |
| `services:read` | List services, deployments, logs and metrics, and the names of secrets |
| `services:write` | Create, update and delete services, their files and their migrations |
| `secrets:write` | Set and delete secret values |
| `db:query` | Run SQL, including writes, against every database of the project (sensitive) |
| `users:read` | List the people who bought your projects, with their license keys and subscriptions |
| `subscriptions:read` | Read a user's project subscription status |
| `subscriptions:write` | Cancel, reactivate or change a user's project subscription |
| `transactions:read` | List and inspect sales |
| `transactions:write` | Change the trial on a sale |
| `products:read` | See which products a user holds, the tiers that let them buy more, and purchases waiting to be fulfilled |
| `products:fulfill` | Mark consumable purchases delivered so they are not refunded |
| `coupons:read` | List coupons, their analytics and the sales they were used on |
| `coupons:write` | Create, update and delete coupons |

### Sessions (CLI and dashboard only)

The CLI and dashboard use a signed-in session, not an API key. Integrations must use an API key.

### Yard Auth access tokens (a user, in an external app)

```
Authorization: Bearer {Yard Auth access token}
```

A token your app obtained for one of its users from the project's own OpenID Connect issuer. It identifies **a user of one project**, never the team, and it reaches exactly three endpoints: `GET /v1/yard-auth/userinfo`, `GET /v1/yard-auth/products` and `POST /v1/yard-auth/fulfillments/{transaction_id}`. See [Yard Auth for external apps](#yard-auth-for-external-apps).

---

## API-Key Endpoints

Everything below takes `Authorization: Bearer yard_…` with the listed scope. The management endpoints (projects, pricing, products, releases, channels, sandboxes, services, secrets, database, users, transactions, coupons) are documented with request and response shapes at https://yard.sh/docs/v1/api; a key with the matching scope can do over HTTP what the CLI does, except manage API keys, the team, or money.

### Projects

| Method | Path | Scope | Description |
|---|---|---|---|
| `GET` | `/v1/projects/{username}/{slug}/metadata` | `metadata:read` | Read project metadata (title, launch stage, tiers with their keys, pricing) |

### Releases (public channel reads)

| Method | Path | Scope | Description |
|---|---|---|---|
| `GET` | `/v1/projects/{id}/channels` | `releases:read` | List the project's public channels |
| `GET` | `/v1/projects/{id}/project-releases` | `releases:read` | List releases, optionally one channel's |
| `GET` | `/v1/projects/{id}/project-releases/by-version/{version}` | `releases:read` | Read a release by version |
| `GET` | `/v1/projects/{id}/project-releases/{releaseId}` | `releases:read` | Read a release by id |
| `GET` | `/v1/projects/{id}/project-releases/{releaseId}/files/{fileId}/download` | `releases:read` | Download a release file (302 to the file) |

A key holding only `releases:read` sees public channels and no drafts or archived releases; a key that also holds `releases:write` sees everything a session sees.

### Licenses

| Method | Path | Scope | Description |
|---|---|---|---|
| `POST` | `/v1/licenses/validate` | `licenses:validate` | Validate a license key for the project in `project_id`, or its sandbox in `sandbox` (optionally bind to a device) |
| `POST` | `/v1/licenses/deactivate` | `licenses:activate` | Deactivate a device from a license of the project in `project_id`, or its sandbox in `sandbox` |

Both look only where the request points: with no `sandbox`, at the project's own keys (a sandbox key answers like an unknown key); with `sandbox`, at that sandbox's keys alone ([pricing-and-licensing.md](pricing-and-licensing.md#commerce-in-a-sandbox)).


### Subscriptions (a user's subscription, from the team's server)

A subscription only starts at Yard's own checkout, when the user subscribes. These endpoints read and manage one that already exists.

| Method | Path | Scope | Description |
|---|---|---|---|
| `GET` | `/v1/projects/{username}/{slug}/subscription` | `subscriptions:read` | Read a user's subscription status for a project |
| `POST` | `/v1/projects/{username}/{slug}/subscription/cancel` | `subscriptions:write` | Cancel a user's subscription |
| `POST` | `/v1/projects/{username}/{slug}/subscription/reactivate` | `subscriptions:write` | Reactivate a cancelled subscription |
| `POST` | `/v1/projects/{username}/{slug}/subscription/change` | `subscriptions:write` | Change a user's tier or billing interval |

### Products (a user's products, from the team's server)

`{id}` is the project UUID; every route takes `?sandbox=<name>` for a sandbox's simulated purchases. Shapes, errors and the delivery loop: [products.md](products.md#delivering-consumables).

| Method | Path | Scope | Description |
|---|---|---|---|
| `GET` | `/v1/projects/{id}/users/{userDisplayId}/products` | `products:read` | One user's `tier_keys` and held `products`. `{userDisplayId}` is `user_` plus the first 8 characters of the user id, or the email; `404` if they never bought from the project, `409` if two buyers share the 8 characters |
| `GET` | `/v1/projects/{id}/fulfillments/pending` | `products:read` | Every paid consumable purchase not yet fulfilled, oldest first. Paged by `after` (the last `transaction_id`) and `limit` (max 100); `has_more` says whether to read on |
| `POST` | `/v1/projects/{id}/fulfillments/{transactionId}` | `products:fulfill` | Mark a consumable purchase delivered; idempotent |

Product setup goes through the CLI ([cli-commands.md](cli-commands.md#yard-projects-products)); the HTTP equivalents are `PUT /v1/projects/{id}/products?release=<release id>` (`projects:write`, the whole set) and `PUT` / `DELETE /v1/projects/{id}/products/{key}/icon?release=<release id>` (`releases:write`).

---

## License-Gated Endpoints (no auth header)

Built-in updaters in your software can reach these directly with just a license key. No `Authorization` header.

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/updates/latest?license_key={key}` | Check for the latest release by license key |
| `GET` | `/v1/updates/latest/download/{filename}?license_key={key}` | Download the latest release file by license key |
| `GET` | `/v1/updates/sandboxes?license_key={key}` | The key's own update stream: the project itself, or the sandbox it was bought in |
| `GET` | `/v1/updates/channels?license_key={key}` | List the release channels the key may see, for a channel picker |
| `GET` | `/v1/updates/releases?license_key={key}` | List the project's, one sandbox's or one channel's releases (GitHub Releases list shape) |
| `GET` | `/v1/updates/releases/{version}/download/{filename}?license_key={key}` | Download a file from a specific release |

All take an optional `sandbox`, and `latest`, `releases` and the by-version download also take `channel`; details and response shapes in [releases-and-updates.md](releases-and-updates.md#downloading-releases-with-a-license-key).

---

## Yard Auth for external apps

Inside a hosted service, Yard Auth is the edge: it signs users in and stamps `X-Yard-*` headers (see [service-and-database.md](service-and-database.md#identity-yard-auth-never-your-own)). Software that runs **outside** the project (a desktop or mobile app, a backend on another host) uses the same Yard Auth as a standard **OpenID Connect** client: every project with Yard Auth is its own issuer, and any OIDC library works against it. Requires the `yard_auth` permission on the owning team (Basic and Pro; check `yard me --json` → `.team_permissions.yard_auth`).

| Item | Value |
|---|---|
| Issuer | `https://yard.sh/auth/application/o/yard-auth-<project id>/` |
| Discovery | `https://yard.sh/auth/application/o/yard-auth-<project id>/.well-known/openid-configuration` |
| Client id | `yard-auth-<project id>` |
| Client secret | From the project's **Auth** page in the dashboard (rotate it there too) |
| Redirect URIs | Managed on the same page: absolute `https` URLs, or `http` on `localhost` while developing; no wildcards or fragments; up to 10, matched exactly |
| Grant | Authorization code (PKCE recommended) |
| Scopes | `openid email profile yard_account offline_access` |
| Token lifetime | Access tokens last one hour; use the refresh token (`offline_access`) to get a new one |

`<project id>` is the project's UUID (`yard projects --json` → `.id`), not its slug. The project has exactly one client: the id above and the secret from the page. Archived projects and teams whose plan lacks `yard_auth` have none; when the plan returns, the client is registered again with a new secret and the redirect URIs must be added again. There is no self-service client registration, so an app is always the team's own app for its own project.

**Claims** in the ID token and from the issuer's own userinfo endpoint:

| Claim | Meaning |
|---|---|
| `sub` | Stable identifier of the person for this issuer |
| `email` | The person's email address |
| `email_verified` | Whether that address has been confirmed |
| `yard_user_id` | The Yard user id, the same value the edge sends a hosted service as `X-Yard-User-Id`; present from the first sign-in |

Purchase status is **not** in the token, because it changes underneath a token's lifetime. Read it from Yard:

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/v1/yard-auth/userinfo` | `Authorization: Bearer <access token>` | The person behind the token and their standing on the project the token was issued for |
| `GET` | `/v1/yard-auth/products` | `Authorization: Bearer <access token>` | Their products: `{ "authenticated": true, "tier_keys": [...], "products": [...] }`, for the project itself (sandbox purchases are reachable only from a sandbox's hosted pages) |
| `POST` | `/v1/yard-auth/fulfillments/{transaction_id}` | `Authorization: Bearer <access token>` | Mark one of their consumable purchases delivered after the app granted it |

```json
{
  "sub": "…",
  "user_id": "<uuid>",
  "email": "a@b.c",
  "email_verified": true,
  "entitlement": "active",
  "tier": "Pro",
  "tier_key": "pro"
}
```

`entitlement` is `none` \| `trial` \| `active` \| `owner`, resolved the same way as the edge header; `tier` (display name) and `tier_key` (stable key; gate features on it) are omitted when the entitlement carries no tier. Call it on every launch rather than caching the verdict for the token's lifetime.

The products and fulfill endpoints answer exactly like the hosted `__yard/products` and `__yard/fulfillments/{transaction_id}` ([products.md](products.md#what-someone-holds)), with the same `error_code`s. Like the update endpoints, all three answer any origin, so an Electron or Tauri app can call them directly.

**Consent and disconnecting.** The first sign-in to a project's app shows the person a consent screen naming the app and what it receives (email address, Yard account, purchase status). The answer is remembered until they disconnect the app on the security page of their Yard account ("Connected apps"), which also revokes the app's refresh tokens; the app's next refresh fails and it has to send the person through sign-in again.

---

## Public Endpoints (no auth)

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/projects/public` | List all public projects |
| `GET` | `/v1/projects/{username}/{slug}/public` | Get a public project (addressed under the owning team's username - this is the shape `window.yard.project` exposes, each tier with its `key` and the `products` on sale) |
| `GET` | `/v1/teams/{id}` | Get a team's public profile and its projects. `{id}` is the team's UUID **or** its username (the subject is always a team, never an individual user) |
| `GET` | `/v1/search?q={query}` | Search projects |
| `POST` | `/v1/coupons/validate` | Validate a coupon code |

---

## Not reachable with an API key

These need a signed-in session (CLI or dashboard) whatever scopes a key holds:

- Minting, listing, editing or deleting API keys (`yard keys …`, the dashboard)
- Team management: members, roles, invites, ownership, switching the active team
- Payout onboarding, payouts, payment methods, the team's own plan
- Custom domains, project images and videos, regenerating a webhook signing secret, the webhook's delivery history and test events
- The GitHub App install, its repo list and the team's linked repos (linking or unlinking one project's repo takes `projects:write`)
- Account, session and security-device management

---

## Error Response Format

All errors return a JSON body:

```json
{
  "error": "Human-readable error message",
  "error_code": "insufficient_scope",
  "error_id": "3f9a1c2e"
}
```

`error_code` is a lowercase code to branch on, present on the errors a client may handle on its own: `unauthorized`, `insufficient_scope`, `upgrade_required`, `no_team`, `rate_limited`, the plan limits such as `project_limit_reached` and `storage_limit_reached`, `not_consumable`, `purchase_not_paid` and `purchase_refunded` from the fulfill endpoints, and `yard_auth_unavailable` from the Yard Auth bearer endpoints. Never match on the `error` text. `error_id` identifies the request for support.

Common HTTP status codes:
- `400`: Bad request (validation error)
- `401`: Unauthorized (missing or invalid token)
- `403`: Forbidden (insufficient scope, or endpoint requires session auth)
- `404`: Not found
- `409`: Conflict (e.g., duplicate resource)
- `429`: Rate limited
- `500`: Internal server error
