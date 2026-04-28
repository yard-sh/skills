# Yard Skill

The agent skill for [Yard](https://yard.sh) — knowledge files that teach AI coding agents (Claude Code and other skill-aware tools) how to work with the Yard CLI and REST API.

When this skill is installed, an agent gains awareness of:

- The `yard` CLI (login, init, products, releases, keys, page, etc.)
- The `yard init --spec` and `yard releases publish --spec` JSON shapes for non-interactive use
- The REST API endpoints for license validation, release downloads, subscriptions, and the license-key update server
- Pricing model details (tiers, seat types, volume brackets, Pro-only features)
- Custom landing-page authoring (`window.yard`, `data-yard` / `data-action`)
- Common troubleshooting steps

## Layout

- `SKILL.md` — entry point loaded into the agent's context
- `references/` — deeper docs the agent loads on demand:
  - `cli-commands.md` — full CLI reference
  - `pricing-and-licensing.md` — pricing model, license keys, trials, coupons
  - `api-reference.md` — REST API endpoints
  - `landing-pages.md` — custom landing-page runtime and conventions
  - `releases-and-updates.md` — publishing releases and downloading updates
  - `troubleshooting.md` — common issues

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
