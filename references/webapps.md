# Web apps (SaaS) on Yard

Yard hosts full web apps — static frontend plus a server-side Worker, with a
per-environment database, secrets, and buyer sign-in — using the same CLI you
already have. The buyer visits `<handle>.yard.sh/<slug>/app/` (or the
product's custom domain under `/app/`).

## The runtime model — read this first

**There are no ports.** Never scaffold Express, `app.listen()`, or any
socket-listening server — it cannot run. The backend is a single file,
`_worker.js`, exporting a fetch handler:

```js
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    if (url.pathname.startsWith("/api/")) return handleAPI(request, env, url);
    return env.ASSETS.fetch(request); // static frontend fallthrough
  },
};
```

Route by path, not by port. The app's paths arrive rooted at `/` regardless of
where it's mounted externally. Use **relative URLs** in frontend code
(`fetch("api/notes")`, not `fetch("/api/notes")`) so the app works under any
mount path.

## The bundle contract

A deployable app is one directory (build output or hand-written):

| Path | Meaning |
|---|---|
| `_worker.js` | The backend. One pre-bundled ES module (bundle imports with esbuild if you use dependencies). Required. |
| `yard.json` | Manifest (below). Optional. |
| `migrations/*.sql` | Database schema migrations, applied in filename order at deploy. Never served publicly. |
| everything else | Static assets served via `env.ASSETS` with SPA fallback (unknown paths → `index.html`). |

Limits: 200 files, 5 MB per file, 25 MB total, paths nest up to 8 levels.
Start from a working scaffold with `yard app init`.

## yard.json

```json
{ "access": "authenticated", "database": true }
```

- `access`: `public` (everyone) · `authenticated` (Yard sign-in required —
  the edge redirects anonymous visitors to login) · `customers` (the edge
  paywall: non-customers are redirected to the product's sales page; only
  buyers/trialers/subscribers get in).
- `database`: `true` provisions a per-environment SQLite (D1) database bound
  as `env.DB`.

## Identity: never build your own auth

The Yard edge authenticates buyers and hands the Worker trusted headers:

| Header | Value |
|---|---|
| `X-Yard-User-Id` | Stable buyer id — use as your foreign key |
| `X-Yard-Email` | Buyer email (may be empty) |
| `X-Yard-Entitlement` | `none` \| `trial` \| `active` |
| `X-Yard-Tier` | Pricing tier name, when known |

Absent headers = anonymous visitor. Clients can never spoof these — the edge
strips inbound `X-Yard-*` before your code runs. Frontend code checks login
state with `fetch("__yard/auth/me")` (relative URL!) and links to
`__yard/auth/login?return=/` and `__yard/auth/logout`. Do not implement
OAuth, sessions, or password storage — with `access: customers` even the
paywall is enforced before your code runs.

## Database

`env.DB` is a standard D1/SQLite binding:

```js
const { results } = await env.DB.prepare(
  "SELECT * FROM notes WHERE user_id = ?1",
).bind(user).all();
```

Schema changes go in `migrations/` as new numbered files
(`0002_add_column.sql`, …) — never edit an applied migration; deploys apply
pending files in order and a failed migration aborts the deploy (the previous
version keeps serving). Each environment has its own database: development
data never touches production. Inspect with
`yard app db query "select ..." --env development --json`.

## Secrets

Third-party API keys go in per-environment secrets, exposed as `env.<NAME>`:

```
yard app secrets set OPENAI_API_KEY=sk-... --env development
yard app secrets set OPENAI_API_KEY=sk-... --env production
```

Values are write-only and take effect on the **next deploy** of that
environment. Never commit keys into the bundle.

## The workflow loop

```
yard app init                              # scaffold (once)
npx wrangler dev                           # local: real runtime, local SQLite
yard app deploy --dir app                  # → development environment
yard app status --env development --json   # verify
yard env promote development production    # go live
```

Deploys target **development** by default; production is only reached by an
explicit promote (which carries the landing page too, if one exists). Local
dev needs no Cloudflare account. Fake identity locally by sending the
`X-Yard-User-Id` header yourself.

## Sales surface

The app does not replace the product page: `<handle>.yard.sh/<slug>` remains
the landing/sales page (custom or built-in) where buyers purchase, and the
app lives under `/app/`. Pricing, trials, subscriptions, coupons, and
checkout are standard Yard — configure them with the CLI as for any product.
