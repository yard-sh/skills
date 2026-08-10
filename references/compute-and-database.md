# Compute and database on Yard

Yard runs a product's server-side code — with an optional per-environment
database, secrets, and buyer sign-in — using the same CLI you already have.
It powers full web apps, but the same feature hosts any HTTP workload: a
JSON API, a webhook receiver, or the backend an installed product calls
home to. A bundle with no frontend at all (just `_worker.js`) is valid.
Requires the `compute` permission (Pro; check `yard me --json` →
`.permissions` before promising a deploy).

## The runtime model — read this first

**There are no ports.** Never scaffold Express, `app.listen()`, or any
socket-listening server — it cannot run. The backend is a single file,
`_worker.js`, exporting a fetch handler:

```js
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    if (url.pathname.startsWith("/api/")) return handleAPI(request, env, url);
    return env.ASSETS.fetch(request); // static asset fallthrough
  },
};
```

Route by path, not by port. There is no filesystem and no long-lived
process — state belongs in the database. Each request has a CPU budget of
about 50 ms (time spent awaiting fetches or database calls doesn't count
against it).

The app's paths arrive rooted at `/` regardless of where it's mounted
externally. Use **relative URLs** in frontend code (`fetch("api/notes")`,
not `fetch("/api/notes")`; `href="styles.css"`, not `href="/styles.css"`) so
the app works under any mount path. `yard app check` lints the bundle for
root-absolute URLs.

## The bundle contract

A deployable app is one directory (build output or hand-written):

| Path | Meaning |
|---|---|
| `_worker.js` | The backend. One pre-bundled ES module (bundle imports with esbuild if you use dependencies). Required. |
| `migrations/*.sql` | Database schema migrations, applied in filename order at deploy. Never served publicly. |
| everything else | Static assets served via `env.ASSETS` with SPA fallback (unknown paths → `index.html`). |

Limits: 200 files, 5 MB per file, 25 MB total, paths nest up to 8 levels.
Start from a working scaffold with `yard app init` — it records the bundle
directory in `.yard/settings.json` as `app.dir`, so plain `yard push` needs
no `--dir`. The deploy walker skips dotfiles and known local-config files at
the bundle root (e.g. `README.md`, the retired `yard.json` manifest) —
they're reported as skipped, never uploaded.

## App settings (`.yard/settings.json`)

How the app deploys is the `app` block of the project's `.yard/settings.json`
(pushed with everything else as the release's `config` artifact, so a
settings change is an edit plus a `yard push`):

```json
{ "app": { "dir": "app", "access": "authenticated", "database": true } }
```

- `access`: `public` (everyone) · `authenticated` (Yard sign-in required —
  the edge redirects anonymous visitors to login) · `customers` (the edge
  paywall: non-customers are redirected to the product's sales page; only
  buyers/trialers/subscribers get in). Default `public`.
- `database`: `true` provisions a per-environment SQLite database bound as
  `env.DB`.

**The product owner always gets in**, whatever the access mode, with
`X-Yard-Entitlement: owner` — a seller never needs to buy their own product
to use (or test) their own app.

## URLs — where the app serves

Production serves at `https://<handle>.yard.sh/<slug>/app/` (on a custom
domain: `https://<customdomain>/app/`). Never construct these URLs by hand —
`yard app open [--env <slug>]` prints and opens the environment's `url`
(also in `--json`).

The app never takes over the product root: `<handle>.yard.sh/<slug>` stays
the landing/sales page, and pricing, trials, subscriptions, coupons, and
checkout are standard Yard — configure them as for any product. To send a
buyer to purchase or upgrade from inside the app, link to the sales page
(relative `../` from `/app/`, or the product's `buy_url`).

`yard_env` is a reserved query parameter and `__yard/` a reserved path
prefix — don't use them in the app's own URLs.

## Identity: never build your own auth

The Yard edge signs buyers in and hands your code trusted headers:

| Header | Value |
|---|---|
| `X-Yard-User-Id` | Stable buyer id — use as your foreign key |
| `X-Yard-Email` | Buyer email (may be empty) |
| `X-Yard-Entitlement` | `none` \| `trial` \| `active` \| `owner` |
| `X-Yard-Tier` | Purchased pricing-tier **name**. Absent when the entitlement carries no named tier — single-price products never send it. |
| `X-Yard-Environment` | Which environment is serving: `production`, or one of your own |

Absent identity headers = anonymous visitor (`public` apps only — gated
modes never reach your code anonymously). Clients can never spoof these —
the edge strips inbound `X-Yard-*` before your code runs. Gate features on
`entitlement !== "none"`, and treat `"owner"` as full access (optionally
with extra admin/debug UI for the seller). Entitlement resolves as: owner →
active subscription → most recent completed purchase (trials count while
unexpired) → `none`.

**Freshness:** verdicts are cached at the edge for up to **60 seconds** per
location. An upgrade, refund, or trial expiry reaches the app within about a
minute; there is no push signal — a long-running UI that must react promptly
should poll `__yard/auth/me`.

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

`env.DB` is a SQLite database with a prepared-statement API:

```js
const { results } = await env.DB.prepare(
  "SELECT * FROM notes WHERE user_id = ?1",
).bind(user).all(); // .run() for writes, .first() for a single row
```

Schema changes go in `migrations/` as new numbered files
(`0002_add_column.sql`, …) — never edit an applied migration; deploys apply
pending files in order. Each environment has its own database, so test data
never touches production. Inspect any environment's data with
`yard app db query "select ..." --env preview --json` (or
`--file schema.sql`, `-` for stdin). Query limits: SQL up to 10 000 bytes,
up to 100 bind params, results truncated past 1000 rows.

### The migration ledger

Applied migrations are recorded in `_yard_migrations (name, applied_at)`
inside the app's own database — query it to answer "did my migration run?".
Underscore-prefixed table names are reserved for Yard; don't create your own.

### Failed migrations — read before writing multi-statement files

A failed migration **aborts the deploy before the new version goes live** —
the previous version keeps serving. But migration files are **not
transactional**: statements run one at a time, so a mid-file failure leaves
earlier statements applied and the file unrecorded in the ledger. The next
deploy re-runs the whole file from the top, which then typically fails on
the already-created objects. Recovery: fix the file so it can re-run from
the top (e.g. `CREATE TABLE IF NOT EXISTS`), or mark it applied manually:
`yard app db query "INSERT INTO _yard_migrations (name) VALUES ('0002_x.sql')"`.
Best practice: keep each migration file to a single statement, or make every
statement idempotent.

## Secrets

Third-party API keys go in per-environment secrets, exposed as `env.<NAME>`:

```
yard app secrets set OPENAI_API_KEY=sk-... --env preview
yard app secrets set OPENAI_API_KEY=sk-... --env production
```

Values are write-only and take effect on the **next deploy** of that
environment. Secrets never promote between environments. Never commit keys
into the bundle.

## The workflow loop

```
yard app init                                  # scaffold (once)
yard app check                                 # validate bundle + lint (no network)
yard push                                      # uploads the bundle into your draft release
yard env create preview                        # once — production is the only built-in environment
yard releases publish v1.0.0 --env preview     # tag the draft and deploy it to your environment
yard app open --env preview                    # browse it (owner-only)
yard app logs --env preview                    # console output + exceptions
yard releases promote v1.0.0 --to production   # go live
```

`yard push` uploads the bundle into your **draft release** and deploys nothing:
nothing serves a draft. An environment only ever runs what its serving release
holds, so going live is always a release operation — publish the draft
(`yard releases publish v1.0.0`, which defaults to production), or publish it
to an environment of your own first and promote when it looks right
(`yard env promote preview production` also works). The release carries the
landing page too, if one exists. Environments beyond the built-in
`production` are created with `yard env create` and are plan-gated.

Editing a release an environment already serves is live: Yard redeploys that
environment automatically, and `yard status` reports it as stale → updating →
up to date while it catches up.

## Testing before customers see it

Every deployed environment has a real, browsable URL. Opening
`…/app/?yard_env=preview` signs the visitor in (if needed), verifies they
**own the product**, sets a host-locked cookie, and serves the `preview`
environment at the same `/app/` path from then on. `?yard_env=` (empty or
`production`) switches back. Non-owners get an explanatory 403 — these URLs
are safe to have in scrollback but not shareable.

A `draft` (or archived/private) product's `/app/` works the same way **for
the owner**: anonymous visitors are sent through sign-in, non-owners get an
explanatory 403, the owner gets the app. You can build and verify everything
before ever advancing the product stage (stage changes are one-way).

## Debugging deployed apps

`yard app logs [--env <slug>] [--limit n] [--since 2h]` returns the app's
`console.log` output, uncaught exceptions, and abnormal request outcomes
(e.g. CPU limit exceeded), newest ~24h, up to 500 entries. A brand-new app
with no traffic returns an empty list, not an error. Logs appear a few
seconds after the request. `yard app db query` against the live environment
answers data questions directly.

There is no in-app `window.yard` runtime — that exists only on landing pages
(see landing-pages.md).
