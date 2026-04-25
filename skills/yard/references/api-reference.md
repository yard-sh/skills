# Yard API Reference

> **Scope of this API.** The Yard REST API is the **integration surface** — it lets a seller's shipped software (or an agent working on that software) validate licenses, read release metadata, and manage buyer subscriptions. It is **not** used to manage a seller's own Yard catalog. Product, release, and coupon management happen through the **Yard CLI** (`yard init`, `yard products`, `yard page …`) — see [cli-commands.md](./cli-commands.md).
>
> Create an API key with the scopes you need at **https://yard.sh/dashboard/api-keys?action=create**.

## Base URL

```
https://api.yard.sh
```

All API paths below are relative to this base URL (e.g., `/v1/licenses/validate` means `https://api.yard.sh/v1/licenses/validate`).

---

## Authentication

### API Key (recommended for integrations)

```
Authorization: Bearer yard_{key}
```

API keys start with the `yard_` prefix and are issued per seller in the dashboard at https://yard.sh/dashboard/api-keys?action=create. Every integration request carries the key in the `Authorization` header with the `Bearer ` prefix.

**Scopes** — each key is limited to the actions it actually needs. Pick only what you use:

| Scope | What it allows |
|-------|----------------|
| `products:read` | Read a product's metadata |
| `licenses:validate` | Validate a license key |
| `licenses:activate` | Activate or deactivate a device against a license |
| `releases:read` | List / read release metadata for the seller's products |
| `releases:download` | Download release files (latest or by id) |
| `subscriptions:read` | Read a buyer's product subscription status |
| `subscriptions:write` | Create, cancel, or reactivate a buyer's product subscription |

Catalog management scopes do **not** exist — product create / update / delete are CLI-only.

### Session Token (CLI and dashboard only)

```
Authorization: Session {token}
```

The session token is a 64-character hex string (32 random bytes) issued to the CLI (`yard login`) and to the web dashboard. Sessions expire after 30 days. The CLI stores the token in `~/.yard/config.json`. **Third-party integrations should not use session tokens** — use an API key instead.

---

## API-Key Endpoints

Everything below accepts `Authorization: Bearer yard_...` with the listed scope. These are the endpoints you integrate into your software.

### Products

| Method | Path | Scope | Description |
|---|---|---|---|
| `GET` | `/v1/products/{slug}/metadata` | `products:read` | Read product metadata (title, state, tiers, pricing) |

### Licenses

| Method | Path | Scope | Description |
|---|---|---|---|
| `POST` | `/v1/licenses/validate` | `licenses:validate` | Validate a license key (optionally bind to a device) |
| `POST` | `/v1/licenses/deactivate` | `licenses:activate` | Deactivate a device from a license |

### Releases

| Method | Path | Scope | Description |
|---|---|---|---|
| `GET` | `/v1/products/{id}/releases` | `releases:read` | List releases for a product |
| `GET` | `/v1/products/{id}/releases/{releaseId}` | `releases:read` | Get a single release |
| `GET` | `/v1/products/{id}/releases/latest` | `releases:download` | Fetch metadata for the latest release |
| `GET` | `/v1/products/{id}/releases/{releaseId}/files/{fileId}/download` | `releases:download` | Download a file from a specific release |
| `GET` | `/v1/products/{id}/releases/latest/files/{fileId}/download` | `releases:download` | Download a file from the latest release |

### Subscriptions (buyer-facing)

| Method | Path | Scope | Description |
|---|---|---|---|
| `POST` | `/v1/subscription-intent` | `subscriptions:write` | Create a subscription payment intent |
| `GET` | `/v1/products/{slug}/subscription` | `subscriptions:read` | Read a buyer's subscription status for a product |
| `POST` | `/v1/products/{slug}/subscription/cancel` | `subscriptions:write` | Cancel a buyer's subscription |
| `POST` | `/v1/products/{slug}/subscription/reactivate` | `subscriptions:write` | Reactivate a cancelled subscription |

---

## License-Gated Endpoints (no auth header)

Built-in updaters in the seller's software can reach these directly with just a license key. No `Authorization` header.

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/updates/latest?license_key={key}` | Check for the latest release by license key |
| `GET` | `/v1/updates/latest/download/{filename}?license_key={key}` | Download the latest release file by license key |

---

## Public Endpoints (no auth)

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Health check |
| `GET` | `/ready` | Readiness check |
| `GET` | `/version` | API version info |
| `GET` | `/v1/products/public` | List all public products |
| `GET` | `/v1/products/{slug}/public` | Get a public product |
| `GET` | `/v1/authors/{username}` | Get author profile and products |
| `GET` | `/v1/search?q={query}` | Search products |
| `POST` | `/v1/coupons/validate` | Validate a coupon code |

---

## CLI-only operations

The following are **not** exposed over HTTP as integration endpoints — they live on the CLI (session auth) because they manage the seller's own catalog. If an agent needs to do any of these, it must run the CLI, not issue HTTP requests:

- Create / update / delete a product (`yard init`, product edits in the dashboard)
- Create / update / delete / sync / archive a release (handled automatically by the GitHub App on release webhook, or via the dashboard)
- Create / update / delete / bulk-generate coupons
- Stripe Connect onboarding and payout management
- Custom domains, product images / videos, webhook secrets
- Custom landing page editing (`yard page init`, `yard page push`, `yard page publish`, …)

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
