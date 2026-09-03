# Service and database on Yard

A Yard Service runs a project's server-side code, with an optional
database, secrets, and buyer sign-in (the project and each sandbox get their
own), using the same CLI you already have. A release can carry several, each on its own path and each
deployed on its own. It powers full web apps, but the same feature hosts any HTTP
workload: a JSON API, a webhook receiver, or the backend an installed project
calls home to. A bundle with no frontend at all (just `_service.js`) is valid.
Requires the `service` permission (Pro; check `yard me --json` →
`.team_permissions` before promising a deploy).

## The runtime model — read this first

**There are no ports.** Never scaffold Express, `app.listen()`, or any
socket-listening server — it cannot run. The backend is a single file,
`_service.js`, exporting a fetch handler:

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

| Path            | Meaning                                                                                                 |
| --------------- | ------------------------------------------------------------------------------------------------------- |
| `_service.js`   | The backend. One pre-bundled ES module (bundle imports with esbuild if you use dependencies). Required. |
| everything else | Static assets served via `env.ASSETS` with SPA fallback (unknown paths → `index.html`).                 |

Limits are per service: 200 files, 5 MB per file, 25 MB total, paths nest up
to 8 levels. Start from a working scaffold with `yard service init <name>`: it
writes the bundle and records the service on the `services` list in
`.yard/settings.json`, so plain `yard push` finds it. The deploy walker skips
dotfiles and known local-config files at the bundle root (e.g. `README.md`,
the retired `yard.json` manifest and the retired per-service `settings.json`)
— they're reported as skipped, never uploaded.

## Service settings

One file: the project's `.yard/settings.json`. Each entry of its `services`
list is the whole service declaration — the directory the bundle lives in and
how it deploys — so changing how a service deploys is an edit there plus a
`yard push`:

```json
{
  "version": 7,
  "services": [
    {
      "dir": "api",
      "name": "api",
      "url": "/api",
      "access": "authenticated",
      "database_access": true
    },
    { "dir": "jobs", "name": "jobs" }
  ]
}
```

- `dir`: the bundle directory, relative to the working directory. Directories
  must not nest inside one another.
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
- `database_access`: `true` lets the service reach the database as `env.DB`.
  The database itself is created by the release's migrations (see _Database_
  below), so a service flagged before the first migration deploys without
  `env.DB` and is redeployed with it once the database exists. The project and
  each sandbox have their own database, so every service there with access
  shares it.

**The project owner always gets in**, whatever the access mode, with
`X-Yard-Entitlement: owner`. A seller never needs to buy their own project
to use (or test) their own service.

## URLs: where a service serves

The project itself serves each service at
`https://<username>.yard.sh/<slug><url>/` (on a custom domain:
`https://<customdomain><url>/`). A sandbox serves the same paths one segment
down, at `https://<username>.yard.sh/<slug>/@<sandbox><url>/`. Never construct
these URLs by hand: `yard service open [--sandbox <slug>] [--service <name>]`
prints and opens the right `url` (also in `--json`, as
`{"sandbox": "...", "service": "...", "url": "...", "deployed": true}`, where
`""` is the project itself). Omitting `--sandbox` targets the project itself.

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

| Header               | Value                                                                                                                     |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `X-Yard-User-Id`     | Stable buyer id — use as your foreign key                                                                                 |
| `X-Yard-Email`       | Buyer email (may be empty)                                                                                                |
| `X-Yard-Entitlement` | `none` \| `trial` \| `active` \| `owner`                                                                                  |
| `X-Yard-Tier`        | Purchased pricing-tier **name**. Absent when the entitlement carries no named tier — single-price projects never send it. |
| `X-Yard-Sandbox`     | A sandbox name, or empty when the project itself is serving                                                               |

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
)
  .bind(user)
  .all(); // .run() for writes, .first() for a single row
```

Schema changes go in `.yard/migrations/` as new numbered files
(`0002_add_column.sql`, ...) - never edit an applied migration; deploys apply
pending files in filename order, before any new services are deployed. The
first migration deployed creates the database, whether or not any service
has `database_access`: a release with migration files and no services still
gets one. The migration stream is project-level: one ordered set of flat
`.sql` files shared by every service (the directory is configurable via
`migrations.dir` in `.yard/settings.json`, and the dashboard's release
editor has a Database Migrations page for writing them). The project and
each sandbox have their own database, shared by every service there with
`database_access`, so two services can read each other's tables
while a sandbox's test data never reaches the storefront. Inspect the
data with `yard db query "select ..." --sandbox preview --json`
(omit `--sandbox` for the project itself; or
`--file schema.sql`, `-` for stdin). Query limits: SQL up to 10 000 bytes,
up to 100 bind params, results truncated past 1000 rows.

### The migration ledger

`yard dev` applies the same files to its local database with the same ledger
(`release_tag = 'local'`), so a file that runs cleanly locally runs cleanly on
deploy.

Applied migrations are recorded in `_yard_migrations (name, applied_at)`
inside the database they ran against, by filename (`0001_init.sql`), so each
file runs once per database. `yard db migrations list` answers "did my
migration run?" (ledger entries merged with the local pending files); rows
whose name contains a `/` are from the retired per-service scheme.
Underscore-prefixed table names are reserved for Yard; don't create your own.

### Failed migrations: read before writing multi-statement files

A failed migration **aborts the deploy before the new version goes live**;
the previous version keeps serving. But migration files are **not
transactional**: statements run one at a time, so a mid-file failure leaves
earlier statements applied and the file unrecorded in the ledger. The next
deploy re-runs the whole file from the top, which then typically fails on
the already-created objects. Recovery: fix the file so it can re-run from
the top (e.g. `CREATE TABLE IF NOT EXISTS`), or mark it applied manually:
`yard db migrations mark-applied 0002_x.sql`.
Best practice: keep each migration file to a single statement, or make every
statement idempotent.

## Secrets

Third-party API keys go in secrets, exposed as `env.<NAME>`; the project
and each sandbox keep their own set:

```
yard service secrets set OPENAI_API_KEY=sk-... --sandbox preview
yard service secrets set OPENAI_API_KEY=sk-...                    # the project itself
```

Values are write-only and take effect there on the **next deploy**. Every
service deployed alongside sees the same set. Secrets never move between
the project and a sandbox. Never commit
keys into a bundle.

## The workflow loop

```
yard service init api                              # scaffold (once per service)
yard dev                                           # run it locally; reloads on save (see local-dev.md)
yard service check                                 # validate every bundle + lint (no network)
yard push                                          # uploads every bundle into your draft release
yard releases publish v1.0.0                       # tag the draft; this is the go-live step
yard sandbox create preview                        # a sandbox of your own (a project starts with none)
yard sandbox pin v1.0.0 --sandbox preview          # serve the same release there
yard service open --sandbox preview --service api  # browse it (your team only)
yard service logs --sandbox preview --service api  # console output + exceptions
```

`yard push` uploads the bundle into your **draft release** and deploys nothing:
nothing serves a draft. What runs is always what the serving release
holds, so going live is always a release operation. Publishing the draft
(`yard releases publish v1.0.0`) adds it to the `Production` channel, which the
project itself follows, so it ships to buyers; `yard sandbox pin
v1.0.0` ships a release by name, and `yard sandbox promote preview`
ships whatever the sandbox is serving. To keep a new release off the
storefront while you try it in a sandbox, run `yard sandbox pin` first
and `yard sandbox unpin` when it looks right. The release carries the
landing page too, if one exists. Sandboxes are created with
`yard sandbox create` and are plan-gated.

Editing a release that is already being served is live: Yard redeploys it
automatically, and `yard status` reports it as stale, then updating, then
up to date while it catches up.

A sandbox isolates more than the database and secrets: it carries its own
customers, transactions, subscriptions, trials and license keys, simulated by
the platform rather than charged through Stripe. So a paywalled service
(`"access": "customers"`) can be exercised end to end in a sandbox: buy it
there, and the buyer reaches the service with a real entitlement header and a
real license key, without any money moving. See
[pricing-and-licensing.md](pricing-and-licensing.md#commerce-in-a-sandbox).

## Testing before customers see it

Start on the machine: `yard dev` serves every service at
`http://localhost:9875/<slug>/<mount>/` with the identity headers (pick a
persona with `--as` or the `yard_dev_identity` cookie), secrets from
`.yard/dev/secrets.env`, and a local database with the migrations applied and
recorded in `_yard_migrations`. Access gating, header stripping and the
`__yard/auth/*` endpoints all behave as hosted. What it cannot do is real
commerce, so a purchase or trial flow is confirmed in a sandbox. Details:
[local-dev.md](local-dev.md).

The project and every sandbox have a real, browsable URL. Opening
`https://<username>.yard.sh/<slug>/@preview/service/` signs the visitor in (if
needed), verifies they **belong to the team that owns the project**, and serves
the `preview` sandbox's service. Drop the `/@preview` segment to go back to
the project itself. Everyone outside the team gets an explanatory 403, so
these URLs are safe to have in scrollback
and, by default, not shareable. `yard sandbox visibility public --sandbox
<name>` opens
a sandbox to anyone with the URL (no sign-in, no purchase); `private` closes
it again.

The sandbox lives in the path rather than in a query parameter, so it applies
to the one request: your service can carry whatever query string it
likes without switching sandboxes, and a URL always says what
it serves. A sandbox with no service deployed says so rather than quietly
serving another's.

A `draft` (or archived) project's `/service/` works the same way **for the
owning team**, as does a private project
(`yard sandbox visibility private`): anonymous visitors are sent
through sign-in, everyone outside the team gets an explanatory 403, and any
team member gets the service.
You can build and verify everything before ever advancing the launch stage
(launch stage changes are one-way).

## Debugging deployed services

`yard service logs [--sandbox <slug>] [--limit n] [--since 2h]` returns the
service's `console.log` output, uncaught exceptions, and abnormal request
outcomes (e.g. CPU limit exceeded), newest ~24h, up to 500 entries. A
brand-new service with no traffic returns an empty list, not an error. Logs
appear a few seconds after the request. `yard db query` against the
live database answers data questions directly.

There is no `window.yard` runtime inside a service; that exists only on
landing pages (see landing-pages.md).
