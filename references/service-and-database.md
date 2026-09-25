# Services and database on Yard

A Yard service runs a project's server-side code, with an optional database, secrets, realtime objects and buyer sign-in through Yard Auth. It hosts any HTTP workload: a full web app, a JSON API, a webhook receiver, or the backend an installed app calls. A release can carry several services, each on its own path; a bundle with only `_service.js` is valid. Requires the `service` permission (check `yard me --json` → `.team_permissions` before promising a deploy).

## The runtime model: read this first

**There are no ports.** Never scaffold Express, `app.listen()` or any listening server; it cannot run. The backend is one file, `_service.js`, exporting a fetch handler:

```js
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    if (url.pathname.startsWith("/api/")) return handleAPI(request, env, url);
    return env.ASSETS.fetch(request); // static files, falling back to index.html
  },
};
```

Route by path. There is no filesystem and no long-lived process: state belongs in the database (or in an object for live shared state such as rooms and presence, see [objects.md](objects.md)). Each request has about 50 ms of CPU; time spent awaiting fetches or the database does not count.

## Paths and static files

A service sees its paths rooted at `/`, whatever its mount: with `"url": "/s"`, a visit to `/<slug>/s/abc` reaches `_service.js` as `/abc`. For each request:

1. A path naming a file in the bundle (`/app.js`) is served directly; `_service.js` does not run.
2. `/` or a folder path (`/docs/`) whose `index.html` is in the bundle is served from that file directly.
3. Everything else reaches `_service.js` with the path unchanged: `/<slug>/s/` arrives as `/`, `/<slug>/s/abc/` as `/abc/`.

`env.ASSETS.fetch(request)` serves a bundle file and falls back to `/index.html` for unknown paths, so client-side routing works when the handler falls through to it. Use **relative URLs** in frontend code (`fetch("api/notes")`, `href="styles.css"`), never root-absolute ones, so the service works under any mount and in sandboxes; `yard service check` lints for them.

## The bundle

One directory per service: `_service.js` (one pre-bundled ES module; bundle dependencies with any bundler, e.g. esbuild) plus any static files. Limits per service: 200 files, 5 MB per file, 25 MB total, 8 levels of nesting. Dotfiles and a bundle-root `README.md` are skipped. `yard service init <name>` writes a working bundle and records it in settings.json.

## Service settings

Each entry of `services` in `.yard/settings.json` is the whole declaration; changing how a service deploys is an edit there plus `yard push`:

```json
{
  "version": 7,
  "services": [
    { "dir": "api", "name": "api", "url": "/api", "access": "authenticated", "database_access": true },
    { "dir": "chat", "name": "chat", "url": "/chat", "access": "users", "objects": [{ "class": "Room", "binding": "ROOMS" }] }
  ]
}
```

- `dir`: the bundle directory, relative to the working directory. Directories must not nest.
- `name`: 1-30 lowercase letters, digits and inner hyphens, unique in the release.
- `url`: where it serves. Default `/<name>`; `/` takes the whole site (the landing page then serves nothing). Unique; `/__yard` and `/@…` are reserved. `/api` and `/api/v2` can both exist; the longer path wins.
- `access`: `public` (default, everyone) · `authenticated` (sign-in required) · `users` (buyers, trialers and subscribers only; others go to the sales page). Anything but `public` needs `yard_auth` (`upgrade_required` otherwise).
- `database_access`: `true` binds the database as `env.DB`. The release's migrations create the database; a service flagged before the first migration deploys without `env.DB` and is redeployed with it once the database exists.
- `objects`: object classes the service exports, each reachable as `env.<binding>`. Needs `service_objects`. See [objects.md](objects.md).

## URLs

The project serves each service at `https://<team>.yard.sh/<slug><url>/` (custom domain: `https://<domain><url>/`), and a sandbox one segment down at `…/<slug>/@<sandbox><url>/`. Don't build these by hand: `yard service open [--sandbox <name>] [--service <name>]` prints the right one. A path no service claims belongs to the landing page. To send a visitor to buy, link to the sales page (relative `../` from a one-segment mount, or the project's `buy_url`).

## Identity: Yard Auth, never your own

The Yard edge signs visitors in and gives your code trusted headers:

| Header | Value |
| --- | --- |
| `X-Yard-User-Id` | Stable user id; use it as your foreign key |
| `X-Yard-Email` | Email (may be empty) |
| `X-Yard-Entitlement` | `none` \| `trial` \| `active` \| `owner` |
| `X-Yard-Tier` | Held tier's **name**; absent when there is none (single-price projects never send it) |
| `X-Yard-Sandbox` | Sandbox name, or empty for the project itself |

- Headers arrive **whenever the visitor is signed in, whatever the access mode**, `public` included. No identity headers means an anonymous visitor (possible only on `public` services).
- Clients cannot forge them: the edge strips incoming `X-Yard-*` headers.
- Every member of the owning team gets in everywhere with `owner`, so a seller never buys their own project. Entitlement resolves: owner → active subscription → latest completed purchase (unexpired trials count) → `none`. Verdicts are cached up to 60 seconds; there is no push signal, so a long-lived UI polls `__yard/auth/me`.
- Never implement OAuth, sessions or password storage. Apps running outside the project use Yard Auth as an OpenID Connect client: [api-reference.md](api-reference.md#yard-auth-for-external-apps).

### `__yard/auth/*` endpoints

They exist at the project root (`/<slug>/__yard/auth/…`, which is what a landing page reaches) and under every service (`/<slug>/<service>/__yard/auth/…`). Call them with relative URLs.

- `login?return=<path>` signs the visitor in (an existing Yard session passes silently) and sends them to `return`. **`return` is a path relative to where you called login**: under a service, `return=/` is the service's root; at the project root, `return=/` is the landing page. It must start with `/`; anything else, a full URL included, falls back to `/`. The first sign-in to a project shows a consent screen (email, Yard account, purchase status), remembered until the person disconnects the app under "Connected apps" on their Yard security page.
- `logout?return=<path>` ends the project session (the person stays signed in to Yard) and redirects with the same rule; without `return` it goes to `/` of where it was called.
- `me` always answers 200: `{"authenticated": true, "user_id": "…", "email": "a@b.c", "entitlement": "active", "tier": "Pro"}` when signed in (`email` may be `""`, `tier` is omitted when empty), exactly `{"authenticated": false, "entitlement": "none"}` otherwise. `authenticated: true` with `entitlement: "none"` is a signed-in non-buyer.

A session covers every service of one project and nothing else. The session cookie is HttpOnly, Secure and SameSite=Lax. Never change state on GET, and require `Content-Type: application/json` on writes: every project under `yard.sh` counts as the same site as yours, so SameSite alone does not stop a form on another project's page from posting, while a cross-origin JSON request cannot be sent without your consent.

**Recipe: anyone reads, signed-in users write** (a public service, e.g. comments or a link shortener):

```js
if (request.method === "GET") return listItems(env);
const user = request.headers.get("X-Yard-User-Id");
if (!user) return Response.json({ error: "sign in", login: "__yard/auth/login?return=/" }, { status: 401 });
if (!request.headers.get("Content-Type")?.startsWith("application/json")) return new Response(null, { status: 415 });
return createItem(env, user, await request.json());
```

## Database

`env.DB` is SQLite (its dialect, `RETURNING` and `INSERT … ON CONFLICT … DO UPDATE` upserts included) behind a prepared-statement API:

```js
const stmt = env.DB.prepare("SELECT * FROM notes WHERE user_id = ?1").bind(user);
const { results } = await stmt.all();   // rows
const row = await stmt.first();         // first row, or null
const { meta } = await env.DB.prepare("DELETE FROM notes WHERE id = ?1").bind(id).run();
// run() → { success, meta: { changes, last_row_id } }; meta.changes === 0 means nothing matched
```

The project and each sandbox have their own database, shared by every service there with `database_access`. Inspect it with `yard db query "select …" [--sandbox <name>] --json` (10 000 bytes of SQL, 100 bind params, 1000 rows).

**Migrations** are flat numbered files in `.yard/migrations/` (`0001_init.sql`, `0002_add_column.sql`; the directory is `migrations.dir`), one ordered set for the whole project. Never edit an applied one; add a new file. A deploy applies pending files in filename order before new services go live, and the first migration creates the database, services or not. Each file runs once per database: applied files are recorded by filename in the `_yard_migrations` table (tables starting with `_` are reserved for Yard). `yard db migrations list` answers "did my migration run?".

**Failed migrations:** a failure aborts the deploy and the previous version keeps serving. Files are **not transactional**: statements before the failing one stay applied and the file is not recorded, so the next deploy re-runs it from the top. Make it re-runnable (`CREATE TABLE IF NOT EXISTS`) or run `yard db migrations mark-applied <file>`. Best: one statement per file, or only idempotent statements.

## Secrets

Third-party keys go in secrets, exposed as `env.<NAME>` to every service of the project or of one sandbox; the two never share values. Write-only; applied on the **next deploy**. Never commit keys into a bundle.

```sh
yard service secrets set OPENAI_API_KEY=sk-...                     # the project itself
yard service secrets set OPENAI_API_KEY=sk-... --sandbox preview   # one sandbox
```

## The workflow loop

```sh
yard service init api                              # scaffold (once per service)
yard dev                                           # run locally; restarts on save (local-dev.md)
yard service check                                 # validate bundles + lint (offline)
yard push                                          # into your draft release; nothing serves a draft
yard sandbox create preview                        # once; a project starts with none
yard sandbox pin                                   # hold the storefront on what it serves now
yard releases publish v1.0.0                       # tag the draft into Production
yard sandbox pin v1.0.0 --sandbox preview          # serve it in the sandbox
yard service open --sandbox preview --service api  # browse it (team only)
yard service logs --sandbox preview --service api  # console output + exceptions
yard sandbox unpin                                 # ship it to buyers
```

What runs is always what the serving release holds, so going live is a release operation. Editing a release that is already served redeploys it; `yard status` shows stale, updating, then up to date. A sandbox also has its own simulated commerce, so a `users` service can be bought and used end to end there without money moving ([pricing-and-licensing.md](pricing-and-licensing.md#commerce-in-a-sandbox)).

## Testing before users see it

Start with `yard dev` ([local-dev.md](local-dev.md)): personas, access gating, header stripping and `__yard/auth/*` all behave as hosted. Real purchases and trials need a sandbox.

Sandbox URLs (`…/<slug>/@preview/<service>/`) sign the visitor in and serve only members of the owning team; everyone else gets an explanatory 403. `yard sandbox visibility public --sandbox <name>` opens one to anyone with the URL. A `draft` or private project's services work the same way for the owning team, so everything can be verified before the launch stage moves (it only moves forward).

## Debugging

`yard service logs [--sandbox <name>] [--limit n] [--since 2h]` returns `console.log` output, uncaught exceptions and abnormal outcomes (e.g. CPU limit exceeded) from the last ~24 h, up to 500 entries, a few seconds behind. `yard db query` answers data questions. There is no `window.yard` inside a service; that exists only on landing pages.
