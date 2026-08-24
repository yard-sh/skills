# Service and database on Yard

A Yard Service runs a project's server-side code, with an optional
per-environment database, secrets, and buyer sign-in, using the same CLI you
already have. It powers full web apps, but the same feature hosts any HTTP
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

A deployable service is one directory (build output or hand-written):

| Path | Meaning |
|---|---|
| `_worker.js` | The backend. One pre-bundled ES module (bundle imports with esbuild if you use dependencies). Required. |
| `migrations/*.sql` | Database schema migrations, applied in filename order at deploy. Never served publicly. |
| everything else | Static assets served via `env.ASSETS` with SPA fallback (unknown paths → `index.html`). |

Limits: 200 files, 5 MB per file, 25 MB total, paths nest up to 8 levels.
Start from a working scaffold with `yard service init`: it records the bundle
directory in `.yard/settings.json` as `service.dir`, so plain `yard push`
needs no override. The deploy walker skips dotfiles and known local-config
files at the bundle root (e.g. `README.md`, the retired `yard.json`
manifest) — they're reported as skipped, never uploaded.

## Service settings (`.yard/settings.json`)

How the service deploys is the `service` block of the project's
`.yard/settings.json` (pushed with everything else as the release's `config`
artifact, so a settings change is an edit plus a `yard push`):

```json
{ "version": 4, "service": { "dir": "service", "access": "authenticated", "database": true } }
```

- `access`: `public` (everyone) · `authenticated` (Yard sign-in required —
  the edge redirects anonymous visitors to login) · `customers` (the edge
  paywall: non-customers are redirected to the project's sales page; only
  buyers/trialers/subscribers get in). Default `public`.
- `database`: `true` provisions a per-environment SQLite database bound as
  `env.DB`.

**The project owner always gets in**, whatever the access mode, with
`X-Yard-Entitlement: owner`. A seller never needs to buy their own project
to use (or test) their own service.

## URLs: where the service serves

Production serves at `https://<username>.yard.sh/<slug>/service/` (on a
custom domain: `https://<customdomain>/service/`). Every other environment
serves one path segment down, at
`https://<username>.yard.sh/<slug>/@<env>/service/`. Never construct these
URLs by hand: `yard service open [--env <slug>]` prints and opens the
environment's `url` (also in `--json`).

The service never takes over the project root: `<username>.yard.sh/<slug>`
stays the landing/sales page, and pricing, trials, subscriptions, coupons,
and checkout are standard Yard, configured as for any project. To send a
buyer to purchase or upgrade from inside the service, link to the sales page
(relative `../` from `/service/`, or the project's `buy_url`).

`__yard/` is a reserved path prefix, so don't use it in the service's own
URLs. A path segment beginning with `@` directly after the slug names an
environment and is likewise reserved, though the bundle path rules already
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
| `X-Yard-Environment` | Which environment is serving: `production`, or one of your own |

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
per-project: signing in to one seller's service grants nothing anywhere else.

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
`yard service db query "select ..." --env preview --json` (or
`--file schema.sql`, `-` for stdin). Query limits: SQL up to 10 000 bytes,
up to 100 bind params, results truncated past 1000 rows.

### The migration ledger

Applied migrations are recorded in `_yard_migrations (name, applied_at)`
inside the service's own database; query it to answer "did my migration
run?". Underscore-prefixed table names are reserved for Yard; don't create
your own.

### Failed migrations — read before writing multi-statement files

A failed migration **aborts the deploy before the new version goes live** —
the previous version keeps serving. But migration files are **not
transactional**: statements run one at a time, so a mid-file failure leaves
earlier statements applied and the file unrecorded in the ledger. The next
deploy re-runs the whole file from the top, which then typically fails on
the already-created objects. Recovery: fix the file so it can re-run from
the top (e.g. `CREATE TABLE IF NOT EXISTS`), or mark it applied manually:
`yard service db query "INSERT INTO _yard_migrations (name) VALUES ('0002_x.sql')"`.
Best practice: keep each migration file to a single statement, or make every
statement idempotent.

## Secrets

Third-party API keys go in per-environment secrets, exposed as `env.<NAME>`:

```
yard service secrets set OPENAI_API_KEY=sk-... --env preview
yard service secrets set OPENAI_API_KEY=sk-... --env production
```

Values are write-only and take effect on the **next deploy** of that
environment. Secrets never promote between environments. Never commit keys
into the bundle.

## The workflow loop

```
yard service init                              # scaffold (once)
yard service check                             # validate bundle + lint (no network)
yard push                                      # uploads the bundle into your draft release
yard env create preview                        # once; production is the only built-in environment
yard releases publish v1.0.0 --env preview     # tag the draft and deploy it to your environment
yard service open --env preview                # browse it (your team only)
yard service logs --env preview                # console output + exceptions
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

Every environment has a real, browsable URL. Opening
`https://<username>.yard.sh/<slug>/@preview/service/` signs the visitor in (if
needed), verifies they **belong to the team that owns the project**, and serves
the `preview` environment's service. Drop the `/@preview` segment to go back to
production. Everyone outside the team gets an explanatory 403 — these URLs are
safe to have in scrollback
and, by default, not shareable. `yard env visibility <env> public` opens an
environment to anyone with the URL (no sign-in, no purchase); `private` closes
it again.

The environment lives in the path rather than in a query parameter, so it is
scoped to the one request: your service can carry whatever query string it
likes without switching environments, and a URL always says which environment
it serves. An environment with no service deployed says so rather than quietly
serving production's.

A `draft` (or archived) project's `/service/` works the same way **for the
owning team**, as does a project whose Production environment is private
(`yard env visibility production private`): anonymous visitors are sent
through sign-in, everyone outside the team gets an explanatory 403, and any
team member gets the service.
You can build and verify everything before ever advancing the project stage
(stage changes are one-way).

## Debugging deployed services

`yard service logs [--env <slug>] [--limit n] [--since 2h]` returns the
service's `console.log` output, uncaught exceptions, and abnormal request
outcomes (e.g. CPU limit exceeded), newest ~24h, up to 500 entries. A
brand-new service with no traffic returns an empty list, not an error. Logs
appear a few seconds after the request. `yard service db query` against the
live environment answers data questions directly.

There is no `window.yard` runtime inside a service; that exists only on
landing pages (see landing-pages.md).
