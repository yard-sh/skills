# Project templates

A template is a public GitHub repository that anyone can turn into their own Yard project in one click. The official ones live under `github.com/yard-sh/yard-*`; `yard-sh/yard-hello-world` is the minimal example (one public service, a custom landing page, a free tier).

## The Create in Yard button

Put this in the template's README, with the repository URL URL-encoded in `repo`:

```html
<a href="https://dash.yard.sh/projects?action=create&repo=https%3A%2F%2Fgithub.com%2Fowner%2Frepo"><img src="https://yard.sh/create-in-yard.png" width="200" alt="Create in Yard" /></a>
```

The link opens the dashboard's create dialog on a preview of the repository (services, landing page, migrations, pricing). `repo` must be `https://github.com/<owner>/<repo>` (an optional `.git` or trailing slash is fine); anything else is ignored. The same URL can be pasted into the dialog's "Create from GitHub URL" field.

## What the repository needs

- Public on GitHub, with at least one commit. Git submodules are not included.
- `.yard/settings.json` at the root (at most 16 KB), in the current layout (`"version": 7`). Nothing else is required: services, a landing page, migrations and pricing are all optional.
- Every declared service directory holds its files and a `_service.js`. A `custom` landing page needs files in its directory, and a `migrations` block needs `.sql` files. Files under the default `.yard/landing-page/` and `.yard/migrations/` count even without a block.
- The usual bundle limits (landing page 20 files, 1 MB each, 5 MB total; service 200 files, 5 MB each, 25 MB total; migrations 200 files, 1 MB each, 5 MB total).

Everything is validated before anything is created, so a broken template creates nothing.

## What happens when someone uses it

- A new project owned by the user's active team, in `draft`. Its title comes from the repository name (made unique with ` (2)` and so on) and its description from the repository's; its slug is generated from the title.
- `project_slug` in the template's settings.json is ignored. When the new owner runs `yard init --project <new-slug>` in a fresh directory, the pulled settings.json gets their slug.
- Pricing comes from the `pricing` block, validated against the team's plan. Without one the project has no tiers (a landing page that isn't for sale yet).
- Services, the landing page, migrations and download buttons become release `1.0.0`, published to the `Production` channel and deployed.
- Secrets are not carried (settings.json has none): the owner sets them with `yard service secrets set`, which applies on the next deploy.
- The project is not linked to the template repository.
- Plan gates apply up front: services need `service`, a custom page `custom_project_pages`, objects `service_objects`, and the team must be under its project limit. A service whose `access` is not `public` also needs `yard_auth` to deploy.

## Conventions for a good template

Keep the README short: the Create in Yard button, the URLs the project serves, the directory layout, how the pieces fit together, and the local loop (`yard dev`) plus shipping (`yard push`, `yard releases publish`). Build a template the same way as any project, and test it by creating a project from its URL before sharing the button.
