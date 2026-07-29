# Web apps (SaaS) on Yard

Yard hosts full web apps — static frontend plus a server-side Worker, with a
per-environment database, secrets, and buyer sign-in — using the same CLI you
already have. Requires the `compute` permission (Pro; check
`yard me --json` → `.permissions` before promising a deploy).

## URLs — where the app serves

| Form | Status |
|---|---|
| `https://<handle>.yard.sh/<slug>/app/` | **Canonical.** Production app. |
| `https://<customdomain>/app/` | Same app on the product's custom domain. |
| `https://<handle>.yard.sh/<slug>/app/?yard_env=development` | Owner-only environment preview (below). |
| `https://yard.sh/@<handle>/<slug>/app` | Not served — 308-redirects to the canonical form. |

`/app` without the trailing slash 308s to `/app/`. The app never takes over
the product root: `<handle>.yard.sh/<slug>` stays the sales page.

`yard app deploy` and `yard app status` return the environment's `url`
(also in `--json`), and `yard app open [--env <slug>]` prints and opens it —
never construct these URLs by hand.

### Environment previews (`?yard_env`)

Every deploy has a real, browsable URL — including development:

```
yard app deploy                            # → development
yard app open                              # opens the development preview
```

Opening `…/app/?yard_env=development` signs the visitor in (if needed),
verifies they **own the product**, sets a host-locked cookie, and serves the
`development` Worker at the same `/app/` path from then on. `?yard_env=` (empty
or `production`) switches back. Non-owners get an explanatory 403 — preview
URLs are safe to have in scrollback but not shareable. `yard_env` is a
reserved query parameter: don't use it in the app's own URLs.

### Pre-release products (draft stage)

A `draft` (or archived/private) product's `/app/` still works **for the
owner**: anonymous visitors are sent through sign-in, non-owners get an
explanatory 403, the owner gets the app. You can build and verify the entire
app before ever advancing the product stage (stage changes are one-way).

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
(`fetch("api/notes")`, not `fetch("/api/notes")`; `href="styles.css"`, not
`href="/styles.css"`) so the app works under any mount path.
`yard app check` lints the bundle for root-absolute URLs.

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

The deploy walker skips dotfiles and, at the bundle root, `wrangler.toml` and
`README.md` — local-dev artifacts are reported as skipped, never uploaded.
`yard app init` scaffolds those at the project root anyway (outside the
bundle) and records the bundle directory in `.yard/settings.json` as
`app_dir`, so plain `yard app deploy` needs no `--dir`.

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

**The product owner always gets in**, whatever the access mode, with
`X-Yard-Entitlement: owner` — a seller never needs to buy their own product
to use (or test) their own app.

## Identity: never build your own auth

The Yard edge authenticates buyers and hands the Worker trusted headers:

| Header | Value |
|---|---|
| `X-Yard-User-Id` | Stable buyer id — use as your foreign key |
| `X-Yard-Email` | Buyer email (may be empty) |
| `X-Yard-Entitlement` | `none` \| `trial` \| `active` \| `owner` |
| `X-Yard-Tier` | Purchased pricing-tier **name**. Absent when the entitlement carries no named tier — single-price products never send it. |
| `X-Yard-Environment` | Which environment is serving: `production`, `development`, … |

Absent identity headers = anonymous visitor (`public` apps only — gated
modes never reach your code anonymously). Clients can never spoof these —
the edge strips inbound `X-Yard-*` before your code runs. Gate features on
`entitlement !== "none"`, and treat `"owner"` as full access (optionally
with extra admin/debug UI for the seller).

**Entitlement is resolved in this order:** product owner → active
subscription → most recent completed transaction (a trial-type transaction
counts only while the trial is unexpired) → standalone active trial → `none`.

**Freshness:** verdicts are cached at the edge for up to **60 seconds** per
location (15s for signed-out verdicts). An upgrade, refund, or trial expiry
reaches the app within about a minute; there is no push signal — a
long-running SPA that must react promptly should poll `__yard/auth/me`.

### `__yard/auth/*` — the session endpoints

Relative to the app base (`…/app/__yard/auth/…`). Frontend code checks login
state with `fetch("__yard/auth/me")` (relative URL!) and links to
`__yard/auth/login?return=/` and `__yard/auth/logout`.

`GET __yard/auth/me` **always returns 200** with JSON:

```jsonc
// signed in
{ "authenticated": true, "user_id": "<uuid>", "email": "a@b.c",
  "entitlement": "active", "tier": "Pro" }
// anonymous — exactly these two fields
{ "authenticated": false, "entitlement": "none" }
```

`email` can be `""`; `tier` is **omitted** (not null) when empty. Note
`authenticated: true` with `entitlement: "none"` is a real state (signed-in
non-customer on a `public` or `authenticated` app). Sessions are per-product:
signing in to one seller's app grants nothing anywhere else.

Do not implement OAuth, sessions, or password storage — with
`access: customers` even the paywall is enforced before your code runs.

## Database

`env.DB` is a standard D1/SQLite binding:

```js
const { results } = await env.DB.prepare(
  "SELECT * FROM notes WHERE user_id = ?1",
).bind(user).all();
```

Schema changes go in `migrations/` as new numbered files
(`0002_add_column.sql`, …) — never edit an applied migration; deploys apply
pending files in order. Each environment has its own database: development
data never touches production. Inspect with
`yard app db query "select ..." --env development --json` (or
`--file schema.sql`, `-` for stdin).

### The migration ledger

Applied migrations are recorded in `_yard_migrations (name, applied_at)`
inside the app's own database — query it to answer "did my migration run?".
Underscore-prefixed table names are reserved for Yard; don't create your own.

### Failed migrations — read before writing multi-statement files

A failed migration **aborts the deploy before the new Worker uploads** — the
previous version keeps serving. But migration files are **not transactional**:
statements run one at a time, so a mid-file failure leaves earlier statements
applied and the file unrecorded in the ledger. The next deploy re-runs the
whole file from the top, which then typically fails on the already-created
objects. Recovery: fix the file so it can re-run from the top (e.g.
`CREATE TABLE IF NOT EXISTS`), or mark it applied manually:
`yard app db query "INSERT INTO _yard_migrations (name) VALUES ('0002_x.sql')"`.
Best practice: keep each migration file to a single statement, or make every
statement idempotent.

## Secrets

Third-party API keys go in per-environment secrets, exposed as `env.<NAME>`:

```
yard app secrets set OPENAI_API_KEY=sk-... --env development
yard app secrets set OPENAI_API_KEY=sk-... --env production
```

Values are write-only and take effect on the **next deploy** of that
environment. Secrets never promote between environments. Never commit keys
into the bundle. Locally, `wrangler dev` reads `.dev.vars` instead (the
scaffold writes `.dev.vars.example`).

## The workflow loop

```
yard app init                              # scaffold (once)
npx wrangler dev                           # local: real runtime, local SQLite
yard app check                             # validate bundle + lint (no network)
yard app deploy                            # → development, prints the preview URL
yard app open                              # browse the development preview
yard app logs --env development            # console output + exceptions
yard env promote development production    # go live
```

Deploys target **development** by default; production is only reached by an
explicit promote (which carries the landing page too, if one exists).

### Local dev differs from hosted Yard

Three things behave differently under `npx wrangler dev` — none indicate a
bug in the app:

1. **No edge, so identity headers are not verified.** Anything you send as
   `X-Yard-User-Id` is believed. Useful for testing
   (`curl -H 'X-Yard-User-Id: dev-user' localhost:8787/api/notes`),
   worthless for validating auth.
2. **`__yard/auth/*` does not exist locally.** UI code must degrade
   gracefully when `fetch("__yard/auth/me")` fails (the scaffold does).
3. **No `_yard_migrations` ledger.** Apply migrations by hand:
   `npx wrangler d1 execute dev --local --file app/migrations/0001_init.sql`.

## Debugging deployed apps

`yard app logs [--env <slug>] [--limit n] [--since 2h]` returns the Worker's
`console.log` output, uncaught exceptions, and abnormal request outcomes
(e.g. CPU limit exceeded), newest ~24h. A brand-new app with no traffic
returns an empty list, not an error. Logs appear a few seconds after the
request. `yard app db query` against the live environment answers data
questions directly.

## Sales surface

The app does not replace the product page: `<handle>.yard.sh/<slug>` remains
the landing/sales page (custom or built-in) where buyers purchase, and the
app lives under `/app/`. Pricing, trials, subscriptions, coupons, and
checkout are standard Yard — configure them with the CLI as for any product.
To send a buyer to purchase or upgrade from inside the app, link to the sales
page (relative `../` from `/app/`, or the product's `buy_url`). There is no
in-app `window.yard` runtime yet — that exists only on landing pages (see
landing-pages.md).
