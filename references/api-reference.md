# Yard API Reference

> **What this API is for.** Integrating Yard into shipped software: validating licenses, reading release metadata, managing buyer subscriptions, and signing buyers in with [Yard Auth](#yard-auth-for-external-apps). An agent managing the seller's own catalog (projects, releases, pages, services, coupons, buyers, sales) uses the **Yard CLI** instead; see [cli-commands.md](./cli-commands.md).
>
> Create an API key with the scopes you need at **https://dash.yard.sh/team/api-keys?action=create**.

## Base URL

```
https://api.yard.sh
```

All API paths below are relative to this base URL (e.g., `/v1/licenses/validate` means `https://api.yard.sh/v1/licenses/validate`).

## `{username}`: how projects are addressed

Projects are owned by a **team**, and a project is addressed by its owning team's username plus its slug: `/v1/projects/{username}/{slug}/…`, matching the public URL `https://yard.sh/@{username}/{slug}` and the subdomain `https://{username}.yard.sh/{slug}`.

A team username is **not** the user's username (they share one namespace but routinely differ). Read it from `yard team --json` → `.active_team.username` or `seller.username` on the public project; never assume it matches the signed-in user.

---

## The `sandbox` parameter

A project has its own real data plus any number of **sandboxes**, each an optional copy with simulated commerce ([pricing-and-licensing.md](pricing-and-licensing.md#commerce-in-a-sandbox)). Everywhere, an **omitted or empty `sandbox` means the project itself**, the only place money is real; naming a sandbox selects it.

The parameter travels in the query string on `GET`s and on the subscription-management `POST`s, and in the JSON body on the checkout endpoints:

| Endpoint | Where `sandbox` goes |
|---|---|
| `GET /v1/updates/latest`, `/v1/updates/sandboxes`, `/v1/updates/releases`, and both download paths | query string |
| `GET /v1/projects/{username}/{slug}/public` | query string |
| `GET /v1/projects/{username}/{slug}/subscription` | query string |
| `POST /v1/projects/{username}/{slug}/subscription/cancel` \| `/reactivate` \| `/change` | query string |
| `POST /v1/subscription-intent` | JSON body (`"sandbox": "preview"`) |
| The buyer's download and library endpoints | query string |

An unknown sandbox is a `404` naming it. A **private** sandbox (the default) answers only members of the owning team and gives everyone else the same `404`; a **public** one answers anyone. Responses that resolve a sandbox are never cacheable.

---

## Authentication

### API Key (recommended for integrations)

```
Authorization: Bearer yard_{key}
```

API keys start with `yard_` and belong to a **team** (created with `yard keys create` or at https://dash.yard.sh/team/api-keys?action=create); a key keeps working when the person who minted it leaves. Send it as `Authorization: Bearer yard_…`.

**Scopes:** a key reaches exactly the endpoints its scopes allow; anything else answers `401`. Scopes do not imply one another; pick only what you use. `yard keys create` prints the catalog.

Integration scopes are safe to ship inside a buyer's app:

| Scope | What it allows |
|-------|----------------|
| `metadata:read` | Read a project's public metadata and pricing |
| `releases:read` | List public channels and their releases, and download release files |
| `licenses:validate` | Validate a license key |
| `licenses:activate` | Activate or deactivate a device against a license |
| `subscriptions:read` | Read a buyer's project subscription status |
| `subscriptions:write` | Create, cancel, reactivate or change a buyer's project subscription |

Management scopes act on the team's own account; keep keys holding them on servers the team controls:

| Scope | What it allows |
|-------|----------------|
| `projects:read` | List and read the team's projects, including drafts and pricing history |
| `projects:write` | Create, update and delete projects, and change pricing and page content |
| `releases:write` | Create, edit, publish and archive releases, manage channels, and read draft contents |
| `sandboxes:read` | List sandboxes and the releases they serve |
| `sandboxes:write` | Create, rename and delete sandboxes, and pin or roll back what they serve |
| `services:read` | List services, deployments, logs and metrics, and the names of secrets |
| `services:write` | Create, update and delete services, their files and their migrations |
| `secrets:write` | Set and delete secret values |
| `db:query` | Run SQL, including writes, against every database of the project (sensitive) |
| `users:read` | List the people who bought your projects, with their license keys and subscriptions |
| `transactions:read` | List and inspect sales |
| `transactions:write` | Change the trial on a sale |
| `coupons:read` | List coupons, their analytics and the sales they were used on |
| `coupons:write` | Create, update and delete coupons |

### Sessions (CLI and dashboard only)

The CLI and dashboard use a signed-in session, not an API key. Integrations must use an API key.

### Yard Auth access tokens (a buyer, in an external app)

```
Authorization: Bearer {Yard Auth access token}
```

A token a buyer's app obtained from the project's own OpenID Connect issuer. It identifies **a buyer of one project**, never the seller, and it reaches exactly one endpoint, `GET /v1/yard-auth/userinfo`. See [Yard Auth for external apps](#yard-auth-for-external-apps).

---

## API-Key Endpoints

Everything below takes `Authorization: Bearer yard_…` with the listed scope. The management endpoints (projects, releases, channels, sandboxes, services, secrets, database, users, transactions, coupons) are documented with request and response shapes at https://yard.sh/docs/v1/api; a key with the matching scope can do over HTTP what the CLI does, except manage API keys, the team, or money.

### Projects

| Method | Path | Scope | Description |
|---|---|---|---|
| `GET` | `/v1/projects/{username}/{slug}/metadata` | `metadata:read` | Read project metadata (title, launch stage, tiers, pricing) |

### Releases (public reads for a shipped app)

| Method | Path | Scope | Description |
|---|---|---|---|
| `GET` | `/v1/projects/{id}/channels` | `releases:read` | List the project's public channels |
| `GET` | `/v1/projects/{id}/project-releases` | `releases:read` | List releases, optionally one channel's |
| `GET` | `/v1/projects/{id}/project-releases/by-version/{version}` | `releases:read` | Read a release by version |
| `GET` | `/v1/projects/{id}/project-releases/{releaseId}` | `releases:read` | Read a release by id |
| `GET` | `/v1/projects/{id}/project-releases/{releaseId}/files/{fileId}/download` | `releases:read` | Download a release file (302 to the file) |

A key holding only `releases:read` sees public channels and no drafts; a key that also holds `releases:write` sees everything a session sees.

### Licenses

| Method | Path | Scope | Description |
|---|---|---|---|
| `POST` | `/v1/licenses/validate` | `licenses:validate` | Validate a license key (optionally bind to a device) |
| `POST` | `/v1/licenses/deactivate` | `licenses:activate` | Deactivate a device from a license |

`validate` answers `valid: true` for a sandbox key too; check its `sandbox` field before granting anything ([pricing-and-licensing.md](pricing-and-licensing.md#commerce-in-a-sandbox)).


### Subscriptions (buyer-facing)

| Method | Path | Scope | Description |
|---|---|---|---|
| `POST` | `/v1/subscription-intent` | `subscriptions:write` | Create a subscription payment intent |
| `GET` | `/v1/projects/{username}/{slug}/subscription` | `subscriptions:read` | Read a buyer's subscription status for a project |
| `POST` | `/v1/projects/{username}/{slug}/subscription/cancel` | `subscriptions:write` | Cancel a buyer's subscription |
| `POST` | `/v1/projects/{username}/{slug}/subscription/reactivate` | `subscriptions:write` | Reactivate a cancelled subscription |
| `POST` | `/v1/projects/{username}/{slug}/subscription/change` | `subscriptions:write` | Change a buyer's tier or billing interval |

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

All take an optional `sandbox`; details and response shapes in [releases-and-updates.md](releases-and-updates.md#downloading-releases-with-a-license-key).

---

## Yard Auth for external apps

Inside a hosted service, Yard Auth is the edge: it signs buyers in and stamps `X-Yard-*` headers (see [service-and-database.md](service-and-database.md#identity-yard-auth-never-your-own)). Software that runs **outside** the project (a desktop or mobile app, a backend on another host) uses the same Yard Auth as a standard **OpenID Connect** client: every project with Yard Auth is its own issuer, and any OIDC library works against it. Requires the `yard_auth` permission on the owning team (Basic and Pro; check `yard me --json` → `.team_permissions.yard_auth`).

| Item | Value |
|---|---|
| Issuer | `https://yard.sh/auth/application/o/yard-auth-<project id>/` |
| Discovery | `https://yard.sh/auth/application/o/yard-auth-<project id>/.well-known/openid-configuration` |
| Client id | `yard-auth-<project id>` |
| Client secret | From the project's **Auth** page in the dashboard (rotate it there too) |
| Redirect URIs | Managed on the same page: `https` only, or `http` on `localhost` while developing; up to 10, matched exactly |
| Grant | Authorization code (PKCE recommended) |
| Scopes | `openid email profile yard_account offline_access` |
| Token lifetime | Access tokens last one hour; use the refresh token (`offline_access`) to get a new one |

`<project id>` is the project's UUID (`yard projects --json` → `.id`), not its slug. The seller's project has exactly one client: the id above and the secret from the tab. There is no self-service client registration, so an app is always the seller's own app for their own project.

**Claims** in the ID token and from the issuer's own userinfo endpoint:

| Claim | Meaning |
|---|---|
| `sub` | Stable identifier of the person for this issuer |
| `email` | The person's email address |
| `email_verified` | Whether that address has been confirmed |
| `yard_user_id` | The Yard user id, the same value the edge sends a hosted service as `X-Yard-User-Id` |

Purchase status is **not** in the token, because it changes underneath a token's lifetime. Read it from Yard:

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/v1/yard-auth/userinfo` | `Authorization: Bearer <access token>` | The person behind the token and their standing on the project the token was issued for |

```json
{
  "sub": "…",
  "user_id": "<uuid>",
  "email": "a@b.c",
  "email_verified": true,
  "entitlement": "active",
  "tier": "Pro"
}
```

`entitlement` is `none` \| `trial` \| `active` \| `owner`, resolved the same way as the edge header; `tier` is omitted when the entitlement carries no named tier. Call it on every launch rather than caching the verdict for the token's lifetime.

**Consent and disconnecting.** The first sign-in to a project's app shows the person a consent screen naming the app and what it receives (email address, Yard account, purchase status). The answer is remembered until they disconnect the app on the security page of their Yard account ("Connected apps"), which also revokes the app's refresh tokens; the app's next refresh fails and it has to send the person through sign-in again.

---

## Public Endpoints (no auth)

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/projects/public` | List all public projects |
| `GET` | `/v1/projects/{username}/{slug}/public` | Get a public project (addressed under the owning team's username - this is the shape `window.yard.project` exposes) |
| `GET` | `/v1/teams/{id}` | Get a team's public profile and its projects. `{id}` is the team's UUID **or** its username (the subject is always a team, never an individual user) |
| `GET` | `/v1/search?q={query}` | Search projects |
| `POST` | `/v1/coupons/validate` | Validate a coupon code |

---

## Not reachable with an API key

These need a signed-in session (CLI or dashboard) whatever scopes a key holds:

- Minting, listing, editing or deleting API keys (`yard keys …`, the dashboard)
- Team management: members, roles, invites, ownership, switching the active team
- Payout onboarding, payouts, payment methods, the team's own plan
- Custom domains, project images and videos, webhook secrets
- Account, session and security-device management

---

## Error Response Format

All errors return a JSON body:

```json
{
  "error": "Human-readable error message"
}
```

Common HTTP status codes:
- `400`: Bad request (validation error)
- `401`: Unauthorized (missing or invalid token)
- `403`: Forbidden (insufficient scope, or endpoint requires session auth)
- `404`: Not found
- `409`: Conflict (e.g., duplicate resource)
- `429`: Rate limited
- `500`: Internal server error
