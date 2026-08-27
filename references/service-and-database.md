# Service and database on Yard

A Yard Service runs a project's server-side code, with an optional
per-scope database, secrets, and buyer sign-in, using the same CLI you
already have. A release can carry several, each on its own path and each
deployed on its own. It powers full web apps, but the same feature hosts any HTTP
workload: a JSON API, a webhook receiver, or the backend an installed project
calls home to. A bundle with no frontend at all (just `_worker.js`) is valid.
Requires the `service` permission (Pro; check `yard me --json` →
`.team_permissions` before promising a deploy).

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

The service's paths arrive rooted at `/` regardless of where it's mounted
externally. Use **relative URLs** in frontend code (`fetch("api/notes")`,
not `fetch("/api/notes")`; `href="styles.css"`, not `href="/styles.css"`) so
the service works under any mount path. `yard service check` lints the bundle
for root-absolute URLs.

## The bundle contract

A deployable service is one directory (build output or hand-written). A
release can carry several, each its own directory and its own deployment:

| Path | Meaning |
|---|---|
| `_worker.js` | The backend. One pre-bundled ES module (bundle imports with esbuild if you use dependencies). Required. |
| `settings.json` | This service's name, URL, access and database settings. Required; deploy input, never served. |
| `migrations/*.sql` | Database schema migrations, applied in filename order at deploy. Never served publicly. |
| everything else | Static assets served via `env.ASSETS` with SPA fallback (unknown paths → `index.html`). |

Limits are per service: 200 files, 5 MB per file, 25 MB total, paths nest up
to 8 levels. Start from a working scaffold with `yard service init <name>`: it
writes the bundle plus its `settings.json` and adds the directory to
`services` in `.yard/settings.json`, so plain `yard push` finds it. The deploy
walker skips dotfiles and known local-config files at the bundle root (e.g.
`README.md`, the retired `yard.json` manifest) — they're reported as skipped,
never uploaded.

## Service settings

Two files, at two levels. The project's `.yard/settings.json` lists which
directories are services:

```json
{ "version": 5, "services": [{ "dir": "api" }, { "dir": "jobs" }] }
```

Each of those directories carries its own `settings.json` at its root, which
travels with the bundle — so changing how a service deploys is an edit in that
file plus a `yard push`:

```json
{ "name": "api", "url": "/api", "access": "authenticated", "database": true }
```

- `name`: 1-30 lowercase letters, digits and inner hyphens. Unique within the
  release; it also names the service everywhere the CLI and dashboard show it.
- `url`: the path the service serves under. Default `/<name>`. `/` gives the
  service the whole site (the landing page then serves nothing). Unique within
  the release; `/__yard` and `/@…` are reserved. Nesting is allowed —
  `/api` and `/api/v2` can be two services, and the longer path wins.
- `access`: `public` (everyone) · `authenticated` (Yard sign-in required —
  the edge redirects anonymous visitors to login) · `customers` (the edge
  paywall: non-customers are redirected to the project's sales page; only
  buyers/trialers/subscribers get in). Default `public`. Per service, so one
  release can put a paywalled app next to a public API.
- `database`: `true` binds the scope's SQLite database as `env.DB`. The
  database belongs to the scope, so every service that asks for one
  shares it.

**The project owner always gets in**, whatever the access mode, with
`X-Yard-Entitlement: owner`. A seller never needs to buy their own project
to use (or test) their own service.

## URLs: where a service serves

The project's global data serves each service at
`https://<username>.yard.sh/<slug><url>/` (on a custom domain:
`https://<customdomain><url>/`). A sandbox serves the same paths one segment
down, at `https://<username>.yard.sh/<slug>/@<sandbox><url>/`. Never construct
these URLs by hand: `yard service open [--sandbox <slug>] [--service <name>]`
prints and opens the scope's `url` (also in `--json`, as
`{"sandbox": "...", "service": "...", "url": "...", "deployed": true}`).
Omitting `--sandbox` targets the project's global data.

A path no service claims is the landing page's, so
`<username>.yard.sh/<slug>` stays the sales page unless a service declares
`"url": "/"`. Pricing, trials, subscriptions, coupons, and checkout are
standard Yard, configured as for any project. To send a buyer to purchase or
upgrade from inside a service, link to the sales page (relative `../` from a
one-segment mount, or the project's `buy_url`).

`__yard/` is a reserved path prefix, so don't use it in a service's own
URLs. A path segment beginning with `@` directly after the slug names a
sandbox and is likewise reserved, though the bundle path rules already
prevent a collision: every segment of a file you ship must start with a
letter or digit.

## Identity: never build your own auth

The Yard edge signs buyers in and hands your code trusted headers:

| Header | Value |
|---|---|
| `X-Yard-User-Id` | Stable buyer id — use as your foreign key |
| `X-Yard-Email` | Buyer email (may be empty) |
| `X-Yard-Entitlement` | `none` \| `trial` \| `active` \| `owner` |
| `X-Yard-Tier` | Purchased pricing-tier **name**. Absent when the entitlement carries no named tier — single-price projects never send it. |
| `X-Yard-Sandbox` | Which scope is serving: a sandbox name, or empty for the project's global data |

Absent identity headers = anonymous visitor (`public` services only; gated
modes never reach your code anonymously). Clients can never spoof these —
the edge strips inbound `X-Yard-*` before your code runs. Gate features on
`entitlement !== "none"`, and treat `"owner"` as full access (optionally
with extra admin/debug UI for the seller). Entitlement resolves as: owner →
active subscription → most recent completed purchase (trials count while
unexpired) → `none`.

**Freshness:** verdicts are cached at the edge for up to **60 seconds** per
location. An upgrade, refund, or trial expiry reaches the service within
about a minute; there is no push signal, so a long-running UI that must
react promptly should poll `__yard/auth/me`.

### `__yard/auth/*` — the session endpoints

Relative to the service base (`…/service/__yard/auth/…`). Frontend code
checks login state with `fetch("__yard/auth/me")` (relative URL!) and links
to `__yard/auth/login?return=/` and `__yard/auth/logout`.

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
non-customer on a `public` or `authenticated` service). Sessions are
per-project: signing in to one seller's project grants nothing anywhere else,
and covers every service of that project.

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
pending files in order. Each scope has its own database, shared by every
service there that asks for one, so two services can read each other's tables
while a sandbox's test data never reaches the storefront. Inspect any scope's
data with `yard service db query "select ..." --sandbox preview --json` (or
`--file schema.sql`, `-` for stdin). Query limits: SQL up to 10 000 bytes,
up to 100 bind params, results truncated past 1000 rows.

### The migration ledger

Applied migrations are recorded in `_yard_migrations (name, applied_at)`
inside the scope's database; query it to answer "did my migration
run?". Names are recorded as `<service>/<file>`, so two services are free to
both ship a `0001_init.sql`. Underscore-prefixed table names are reserved for
Yard; don't create your own.

### Failed migrations — read before writing multi-statement files

A failed migration **aborts the deploy before the new version goes live** —
the previous version keeps serving. But migration files are **not
transactional**: statements run one at a time, so a mid-file failure leaves
earlier statements applied and the file unrecorded in the ledger. The next
deploy re-runs the whole file from the top, which then typically fails on
the already-created objects. Recovery: fix the file so it can re-run from
the top (e.g. `CREATE TABLE IF NOT EXISTS`), or mark it applied manually:
`yard service db query "INSERT INTO _yard_migrations (name) VALUES ('api/0002_x.sql')"`.
Best practice: keep each migration file to a single statement, or make every
statement idempotent.

## Secrets

Third-party API keys go in per-scope secrets, exposed as `env.<NAME>`:

```
yard service secrets set OPENAI_API_KEY=sk-... --sandbox preview
yard service secrets set OPENAI_API_KEY=sk-...                    # the global scope
```

Values are write-only and take effect on the **next deploy** of that
scope. They belong to the scope, so every service deployed there
sees the same set. Secrets never promote between scopes. Never commit
keys into a bundle.

## The workflow loop

```
yard service init api                              # scaffold (once per service)
yard service check                                 # validate every bundle + lint (no network)
yard push                                          # uploads every bundle into your draft release
yard releases publish v1.0.0                       # tag the draft; this is the go-live step
yard sandbox create preview                        # a scope of your own (a project starts with none)
yard sandbox deploy preview v1.0.0                 # serve the same release there
yard service open --sandbox preview --service api  # browse it (your team only)
yard service logs --sandbox preview --service api  # console output + exceptions
```

`yard push` uploads the bundle into your **draft release** and deploys nothing:
nothing serves a draft. A scope only ever runs what its serving release
holds, so going live is always a release operation. Publishing the draft
(`yard releases publish v1.0.0`) adds it to the `Production` channel, which the
project's global data follows, so it ships to buyers; `yard sandbox deploy
global v1.0.0` ships a release by name, and `yard sandbox promote preview
global` ships whatever the sandbox is serving. To keep a new release off the
storefront while you try it in a sandbox, run `yard sandbox pin global` first
and `yard sandbox unpin global` when it looks right. The release carries the
landing page too, if one exists. Sandboxes are created with
`yard sandbox create` and are plan-gated.

Editing a release a scope already serves is live: Yard redeploys that
scope automatically, and `yard status` reports it as stale, then updating, then
up to date while it catches up.

A sandbox isolates more than the database and secrets: it carries its own
customers, transactions, subscriptions, trials and license keys, simulated by
the platform rather than charged through Stripe. So a paywalled service
(`"access": "customers"`) can be exercised end to end in a sandbox: buy it
there, and the buyer reaches the service with a real entitlement header and a
real license key, without any money moving. See
[pricing-and-licensing.md](pricing-and-licensing.md#commerce-in-a-sandbox).

## Testing before customers see it

Every scope has a real, browsable URL. Opening
`https://<username>.yard.sh/<slug>/@preview/service/` signs the visitor in (if
needed), verifies they **belong to the team that owns the project**, and serves
the `preview` sandbox's service. Drop the `/@preview` segment to go back to the
project's global data. Everyone outside the team gets an explanatory 403, so
these URLs are safe to have in scrollback
and, by default, not shareable. `yard sandbox visibility <sandbox> public` opens
a sandbox to anyone with the URL (no sign-in, no purchase); `private` closes
it again.

The sandbox lives in the path rather than in a query parameter, so it is
scoped to the one request: your service can carry whatever query string it
likes without switching scopes, and a URL always says which scope
it serves. A scope with no service deployed says so rather than quietly
serving another scope's.

A `draft` (or archived) project's `/service/` works the same way **for the
owning team**, as does a project whose global data is private
(`yard sandbox visibility global private`): anonymous visitors are sent
through sign-in, everyone outside the team gets an explanatory 403, and any
team member gets the service.
You can build and verify everything before ever advancing the project stage
(stage changes are one-way).

## Debugging deployed services

`yard service logs [--sandbox <slug>] [--limit n] [--since 2h]` returns the
service's `console.log` output, uncaught exceptions, and abnormal request
outcomes (e.g. CPU limit exceeded), newest ~24h, up to 500 entries. A
brand-new service with no traffic returns an empty list, not an error. Logs
appear a few seconds after the request. `yard service db query` against the
live scope answers data questions directly.

There is no `window.yard` runtime inside a service; that exists only on
landing pages (see landing-pages.md).
