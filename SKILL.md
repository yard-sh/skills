---
name: yard
metadata:
  author: yard.sh
  files:
    - SKILL.md
    - references/cli-commands.md
    - references/pricing-and-licensing.md
    - references/api-reference.md
    - references/landing-pages.md
    - references/releases-and-updates.md
    - references/service-and-database.md
    - references/local-dev.md
    - references/objects.md
    - references/templates.md
    - references/troubleshooting.md
description: >-
  Yard sells, licenses, distributes and hosts software. Use whenever the user mentions Yard or the yard CLI:
  projects, pricing, license keys, releases and updates, sandboxes, hosted services and databases, yard dev,
  landing pages, Yard Auth sign-in, realtime objects, templates, coupons, buyers or sales.
---

# Yard

Yard lets developers sell software: checkout (Yard is the merchant of record), license keys and device activations, release downloads with an update server, custom landing pages, and hosted services with a database, buyer sign-in (Yard Auth) and realtime objects. Sellers manage it with the `yard` CLI; shipped software integrates through the REST API.

Install: `curl -fsSL https://cli.yard.sh | sh` (Windows: `irm https://cli.yard.sh/install.ps1 | iex`). `yard login` prints a one-time code the user confirms in a browser, so ask the user to run it; you cannot finish it for them. `yard <command> --help` is always current.

## Ground rules

- **Manage with the CLI, integrate with the API.** Creating projects, pricing, releases, pages, services, coupons and reading buyers and sales is CLI work. The REST API is for shipped software: license validation, updates, subscriptions, Yard Auth in external apps. Never create a project over HTTP.
- **Non-interactive only.** Use `--json` (result on stdout, logs on stderr) and `--spec <file|->` (JSON input). Never pipe answers into a prompt; bare `yard init` is for humans.
- **Teams own everything.** Projects, coupons, API keys and payouts belong to a team, and the CLI acts as the active team (`yard team --json` → `.active_team`), which is stored on the account and shared with the dashboard. Projects live under the team's username: `https://<team>.yard.sh/<slug>/`.
- **Check entitlements, never assume them.** Read `yard me --json` → `.team_permissions` before proposing a feature (`.permissions` is the user's own and gates nothing on a project). Never quote a plan's features from memory.
- **Read errors by code.** `upgrade_required` or a permission `FORBIDDEN`: the team's plan lacks the feature; send a spec it supports or point to https://yard.sh/pricing. `NO_TEAM`: the user has no team; send them to https://yard.sh/team, upgrading does not help. `NOT_TEAM_OWNER`: payouts and billing are owner-only.
- **Launch stages are one-way** (`draft` → `early_access` → `published`, set in the dashboard). Never advance one to test: a draft project already serves its pages and services to its own team.
- **Sandboxes are for testing.** A sandbox (`/<slug>/@<name>/`) is a private copy of the project with its own data, secrets and simulated commerce: purchases, trials and license keys work, no money moves. A sandbox license key validates as `valid: true` with a `sandbox` field, so shipped software must reject keys whose `sandbox` is set.
- **Look before creating.** Run `yard projects --json` first so a retry never creates a duplicate.

## New project

Ask first: **guided** (the user drives and you explain each step) or **autopilot** (you drive). On autopilot:

1. Get a one-line description and a price idea.
2. Propose title, pricing model (one-time or subscription), tiers, seat type, prices and launch plan, flagging anything the team's plan lacks (license keys, activations, trials, gifts, seat-based pricing, extra tiers, custom page, services). Wait for the user's OK.
3. `yard projects --json`, then `yard init --spec - --json` (schema in [cli-commands.md](references/cli-commands.md#yard-init)), or `yard init --project <slug> --json` to link an existing project.

Then cover what the project type needs:

- **Installed software** (desktop app, CLI, binary): publish each version with `yard releases publish` and point the app's updater at `GET https://api.yard.sh/v1/updates/latest` with the buyer's license key. Otherwise the buy page has nothing to download. See [releases-and-updates.md](references/releases-and-updates.md).
- **Web app, API or backend Yard runs:** see Hosted services below. Check the `service` permission while planning, not at push time.
- **Custom landing page:** `yard init --page`, edit `.yard/landing-page/`, preview with `yard dev`. See [landing-pages.md](references/landing-pages.md).
- **A template others can start from:** see [templates.md](references/templates.md).

## Hosted services

- `yard service init <name>` scaffolds a working bundle and records it under `services` in `.yard/settings.json`. The backend is one file, `_service.js`, exporting a fetch handler. No ports, no `listen()`, no Express: route by path and use relative URLs.
- **Never build auth.** Yard Auth signs visitors in and gives the service trusted `X-Yard-*` headers; `"access": "users"` is a complete paywall with no code. Any access other than `public` needs `.team_permissions.yard_auth`.
- **Loop:** `yard dev` (everything at `http://localhost:9875/<slug>/`) → `yard push` (into a draft release; nothing serves a draft) → `yard releases publish <tag>` (the go-live step: the release lands in the `Production` channel the project follows). To try it first: `yard sandbox pin` (hold the storefront), publish, `yard sandbox pin <tag> --sandbox preview`, check it, then `yard sandbox unpin`.
- **Realtime** (chat rooms, presence, multiplayer) belongs in objects, not in a table: [objects.md](references/objects.md).

## References

| File | Covers |
| --- | --- |
| [cli-commands.md](references/cli-commands.md) | Every command: flags, `--spec` shapes, `--json` output, settings.json, recipes |
| [pricing-and-licensing.md](references/pricing-and-licensing.md) | Tiers, seats, launch stages, coupons, trials, gifts, license keys, activations, sandbox commerce |
| [releases-and-updates.md](references/releases-and-updates.md) | Releases, channels, rollback, GitHub sync, the update server |
| [api-reference.md](references/api-reference.md) | REST API, API key scopes, Yard Auth for external apps |
| [landing-pages.md](references/landing-pages.md) | `window.yard`, `data-yard` / `data-action`, buyer state, signed-in visitors |
| [service-and-database.md](references/service-and-database.md) | Service contract and paths, settings, Yard Auth headers and endpoints, database, secrets |
| [local-dev.md](references/local-dev.md) | `yard dev`: personas, local database, control panel API |
| [objects.md](references/objects.md) | Realtime objects: WebSockets, storage, limits, lifecycle |
| [templates.md](references/templates.md) | Making a template and the Create in Yard button |
| [troubleshooting.md](references/troubleshooting.md) | Install, login, team and settings errors |
