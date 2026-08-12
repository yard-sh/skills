# Yard API Reference

> **Scope of this API.** The Yard REST API is the **integration surface** — it lets a seller's shipped software (or an agent working on that software) validate licenses, read release metadata, and manage buyer subscriptions. It is **not** used to manage a seller's own Yard catalog. Product, release, and coupon management — plus reading the seller's customers and sales — happen through the **Yard CLI** (`yard init`, `yard products`, `yard coupons`, `yard customers`, `yard transactions`, `yard push / yard pull`) — see [cli-commands.md](./cli-commands.md).
>
> Create an API key with the scopes you need at **https://yard.sh/dashboard/api-keys?action=create**.

## Base URL

```
https://api.yard.sh
```

All API paths below are relative to this base URL (e.g., `/v1/licenses/validate` means `https://api.yard.sh/v1/licenses/validate`).

## `{handle}` — how products are addressed

Products are owned by a **team**, and a product is addressed by its owning team's handle plus its slug: `/v1/products/{handle}/{slug}/…`, matching the public URL `https://yard.sh/@{handle}/{slug}` and the subdomain `https://{handle}.yard.sh/{slug}`.

A handle is **not** a user's username. They share one namespace (so neither can collide with the other), but a team handle is what resolves here, and a seller's personal username resolves nothing unless they happen to own a team with the same handle. Read the value from `yard team --json` → `.active_team.handle`, or from `seller.username` on the public product response — never assume it matches the signed-in user.

(Some server-side route definitions still spell this parameter `username`, from before teams existed. The value it takes is the team handle.)

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
| `products:read` | Read a product's metadata |
| `licenses:validate` | Validate a license key |
| `licenses:activate` | Activate or deactivate a device against a license |
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
| `GET` | `/v1/products/{handle}/{slug}/metadata` | `products:read` | Read product metadata (title, stage, tiers, pricing) |

### Licenses

| Method | Path | Scope | Description |
|---|---|---|---|
| `POST` | `/v1/licenses/validate` | `licenses:validate` | Validate a license key (optionally bind to a device) |
| `POST` | `/v1/licenses/deactivate` | `licenses:activate` | Deactivate a device from a license |

### Subscriptions (buyer-facing)

| Method | Path | Scope | Description |
|---|---|---|---|
| `POST` | `/v1/subscription-intent` | `subscriptions:write` | Create a subscription payment intent |
| `GET` | `/v1/products/{handle}/{slug}/subscription` | `subscriptions:read` | Read a buyer's subscription status for a product |
| `POST` | `/v1/products/{handle}/{slug}/subscription/cancel` | `subscriptions:write` | Cancel a buyer's subscription |
| `POST` | `/v1/products/{handle}/{slug}/subscription/reactivate` | `subscriptions:write` | Reactivate a cancelled subscription |

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
| `GET` | `/v1/products/{handle}/{slug}/public` | Get a public product (scoped under the owning team's handle — this is the shape `window.yard.product` exposes) |
| `GET` | `/v1/authors/{handle}` | Get an author profile (a team) and its products |
| `GET` | `/v1/search?q={query}` | Search products |
| `POST` | `/v1/coupons/validate` | Validate a coupon code |

---

## CLI-only operations

The following are **not** exposed over HTTP as integration endpoints — an API key can't reach them, because they manage the seller's own catalog. If an agent needs to do any of these, it must run the CLI, not issue HTTP requests:

- Create / update / delete a product (`yard init`, product edits in the dashboard)
- Create, publish, or promote a release (`yard releases publish`, `yard releases promote`, the GitHub App on release webhook, or the dashboard)
- Create / update / delete / bulk-generate coupons (`yard coupons create`, `yard coupons generate`, `yard coupons update`, `yard coupons rm`)
- Read the seller's customers and sales (`yard customers`, `yard transactions`) — these are reporting on the selling team's own books, not an integration surface, so an API key can't reach them
- Lengthen or shorten a buyer's running free trial (`yard transactions trial <order-id> --add-days N`)
- Stripe Connect onboarding and payout management
- Custom domains, product images / videos, webhook secrets
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
