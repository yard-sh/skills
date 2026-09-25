# Yard Skill

The agent skill for [Yard](https://yard.sh): knowledge files that teach AI coding agents (Claude Code and other skill-aware tools) how to work with the Yard CLI and REST API.

When this skill is installed, an agent gains awareness of:

- The `yard` CLI (login, init, projects, releases, sandboxes, channels, service, dev, keys, etc.)
- The `yard init --spec` and `yard releases publish --spec` JSON shapes for non-interactive use
- The REST API endpoints for license validation, release downloads, subscriptions, the license-key update server, and Yard Auth for external apps
- Pricing model details (tiers, seat types, volume brackets, Pro-only features)
- Sandboxes: optional copies of the project, release channels, and the simulated commerce a sandbox carries
- Custom landing-page authoring (`window.yard`, `data-yard` / `data-action`)
- Objects: realtime rooms and WebSocket connections inside a hosted service, their storage, limits and lifecycle
- Project templates and the Create in Yard button
- Common troubleshooting steps

## Layout

- `SKILL.md`: entry point loaded into the agent's context (kept short; details live in references)
- `references/`: deeper docs the agent loads on demand:
  - `cli-commands.md`: full CLI reference, including settings.json
  - `pricing-and-licensing.md`: pricing model, license keys, trials, coupons, sandbox commerce
  - `api-reference.md`: REST API endpoints, API key scopes, Yard Auth for external apps
  - `landing-pages.md`: custom landing-page runtime and conventions
  - `service-and-database.md`: hosted service and database runtime contract and workflow
  - `local-dev.md`: `yard dev`, personas, the control panel API
  - `objects.md`: realtime objects: declaring classes, the class contract, connections, storage, limits, lifecycle
  - `releases-and-updates.md`: releases, channels, GitHub sync, the update server
  - `templates.md`: making a project template and the Create in Yard button
  - `troubleshooting.md`: common issues

## Install

The Yard CLI ships an installer:

```sh
yard skill install
```

This pulls the latest release into `~/.agents/skills/yard/`. To update later:

```sh
yard skill update
```

## Contributing

This repo is consumed by anyone with the skill installed, so changes propagate through the parent [yard-sh/yard](https://github.com/yard-sh) repo's submodule pointer. Open an issue or PR here for content fixes; coordinate larger restructures with the team first.
