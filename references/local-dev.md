# Local development: `yard dev`

`yard dev` runs a Yard project on the machine exactly as Yard hosts it: the landing page, every service under its mount path, the `X-Yard-*` identity headers, secrets, and a local database with the project's migrations applied. It is the fast loop; a sandbox remains the check with real commerce before buyers see anything.

Use it whenever you are building or changing a service or a custom landing page. Iterate here until the behavior is right, then `yard push`.

## Running it

```sh
yard dev                       # serve the working directory (walks up to find .yard/settings.json)
yard dev --as customer:pro     # default persona for requests with no identity cookie
yard dev --json                # one JSON event per line, for scripts and agents
yard dev --port 4000           # default 9875
yard dev --root                # serve at / like a custom domain, instead of /<slug>/
yard dev --reset-db            # delete .yard/dev/data.sqlite and re-apply migrations
yard dev --offline             # skip Yard lookups (live project data, remote secret names)
yard dev --no-panel            # disable the control panel
yard dev --allow-local-egress  # let services reach localhost and private networks
```

Requirements: `.yard/settings.json` with at least one `services` entry or a landing page. The first run downloads the Yard local runtime (about 40 MB) into `~/.yard/runtime/`. Linux needs glibc 2.35+, macOS 13.5+. Windows on ARM is not supported; test with a sandbox there.

No login is needed. Logged in, `yard dev` also fetches the project's live public data for the landing page (so `window.yard.project` is real) and warns about secrets set on Yard that have no local value.

Startup output (human mode):

```
Yard local runtime v1.20260903.1
  Landing page   http://localhost:9875/widget/
  Service api    http://localhost:9875/widget/api/   access=customers  database=yes
  Control panel  http://localhost:9875/__yard/dev/
  Persona        anonymous (change with --as or in the control panel)
  Database       .yard/dev/data.sqlite  (2 migrations applied, 0 pending)
  Secrets        .yard/dev/secrets.env  (3 loaded)
  Project data   live from Yard
Watching for changes. Press Ctrl-C to stop.
```

With `--json`, events are `{"event":"ready",...}` (same fields: `landing_page`, `services[]`, `panel`, `persona`, `database`, `migrations_applied`, `migrations_pending`, `secrets_loaded`, `missing_secrets`, `snapshot_source`, `warnings`), then `restart`, `validation_error` (with `errors[]`), `migrations` (with `applied[]`), `log` (with `service`, `level`, `message`), `request`, `runtime_error`, `runtime_download`, `notice`, and `stopped`. Run it in the background and read stdout; `ready` means the URLs answer.

## URLs

Local URLs keep the hosted shape so relative links and `fetch("api/...")` behave identically:

| Hosted | Local |
| --- | --- |
| `https://<team>.yard.sh/<slug>/` | `http://localhost:9875/<slug>/` |
| `https://<team>.yard.sh/<slug>/api/` | `http://localhost:9875/<slug>/api/` |

`/` redirects to `/<slug>/`. Inside `_service.js` the path is rooted at `/` (a visit to `/<slug>/api/notes` arrives as `/notes`) and `request.url` is `http://localhost:9875/notes`. Root-absolute URLs that break when hosted break here too, which is the point.

## Personas instead of sign-in

There is no real sign-in locally. A persona decides which `X-Yard-*` headers the edge stamps and what `__yard/auth/me` returns:

| Persona id | `X-Yard-User-Id` | `X-Yard-Entitlement` | `X-Yard-Tier` | `member` |
| --- | --- | --- | --- | --- |
| `anonymous` | (none) | (none) | | |
| `signed-in` | `dev-user-signed-in` | `none` | | false |
| `trial` | `dev-user-trial` | `trial` | first tier with a free trial | false |
| `customer:<tier-slug>` | `dev-user-customer-<tier-slug>` | `active` | the tier's name | false |
| `member` | `dev-user-member` | `owner` | | true |

One `customer:*` persona exists per tier in `pricing.tiers` (`Pro` becomes `customer:pro`); with no tiers there is a single `customer`. `X-Yard-Sandbox` is always empty (the project's own scope). Client-sent `X-Yard-*` headers are stripped, so forged identity does not work locally either.

Ways to choose the persona:

- `--as <id>` sets the default for requests without a cookie.
- Send the cookie directly: `curl -H 'Cookie: yard_dev_identity=customer:pro' http://localhost:9875/widget/api/notes`.
- `POST /__yard/dev/api/persona` with `{"persona":"member","default":true}` (JSON, from the same origin) changes the default for everyone.
- In a browser, `/<slug>/<service>/__yard/auth/login` shows the picker; `__yard/auth/logout` clears it.

Access gating applies exactly as hosted: `authenticated` redirects anonymous visitors to the picker, `customers` sends `entitlement: none` visitors to the landing page, and `member` passes every gate.

## Secrets

`.yard/dev/secrets.env` holds `KEY=value` lines and is read into `env.KEY`. The directory ignores itself in git and nothing in it is ever pushed. Names follow the `yard service secrets set` grammar (`^[A-Z][A-Z0-9_]{0,63}$`, not `DB` or `ASSETS`); an invalid line is reported as a validation error and the previous values stay in effect. The startup line `Warning: secrets set on Yard but missing locally: ...` names remote secrets with no local value.

## Database and migrations

Services with `database_access: true` get `env.DB` backed by `.yard/dev/data.sqlite`. All of `.yard/migrations/*.sql` is applied in filename order and recorded in `_yard_migrations` with `release_tag = 'local'`, exactly like a deploy: a file never runs twice, and a failing file leaves its earlier statements applied and prints the same recovery message. New files are applied the moment they are saved. `--reset-db` (or `POST /__yard/dev/api/db/reset`) starts from an empty database.

Query the local database from the panel or with `POST /__yard/dev/api/db/query` `{"sql":"select * from notes","params":[]}`; the response is `{columns, rows, meta}` or `{error}`. `yard db query` still targets the hosted database, not this one.

## Control panel and its API

`http://localhost:9875/__yard/dev/` is a small page over these JSON endpoints, all loopback-only:

| Endpoint | Purpose |
| --- | --- |
| `GET /__yard/dev/api/state` | Everything at a glance: URLs, services (mount, access, database), personas and the current default, runtime state and validation errors, migrations applied and pending, local and remote secret names |
| `GET /__yard/dev/api/requests?since=<id>&limit=<n>` | Recent requests: method, path, target service, status, duration, persona. `requests/stream` is the same as Server-Sent Events |
| `GET /__yard/dev/api/logs?service=<name>&since=<id>` | Console output and exceptions from your services. `logs/stream` for SSE |
| `POST /__yard/dev/api/persona` | `{"persona": "<id>", "default": true|false}` |
| `POST /__yard/dev/api/db/query` | `{"sql": "...", "params": []}` against the local database |
| `POST /__yard/dev/api/migrations/apply` | Apply pending migrations now |
| `POST /__yard/dev/api/db/reset` | Empty the local database and re-apply |

POST requests must send `Content-Type: application/json`. For an automated check, start `yard dev --json`, wait for `ready`, exercise the URLs with curl (choosing personas by cookie), read `/__yard/dev/api/requests` and `/api/logs` for what happened, then stop the process.

## Reload and validation

Every service directory, the landing page directory, `.yard/migrations`, `.yard/settings.json` and the secrets file are watched. A save re-validates with the same rules as `yard push` (file limits, extensions, `_service.js` present, mount and name rules). A validation error is printed (`validation_error` in JSON mode) and the last good version keeps serving. Changes to code, assets, settings or secrets restart the runtime in well under a second; static files are served fresh from disk.

## What differs from hosted Yard

- The 50 ms CPU budget per request is not enforced locally.
- Outbound requests to private networks and localhost are blocked as hosted, by address class only (`--allow-local-egress` lifts it).
- Personas replace sign-in; nothing touches the Yard account.
- No sandboxes, draft gating, dashboard metrics or `yard service logs` for local runs; use the panel's logs.
- `request.url` is `http://localhost:<port>/...`.
- The Cache API is unavailable.

When behavior must be confirmed with real commerce (a purchase, a trial, a license key), move to a sandbox: `yard push`, publish, `yard sandbox pin <tag> --sandbox preview`.
