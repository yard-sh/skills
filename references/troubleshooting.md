# Troubleshooting

## "command not found: yard" after installation

The installer adds the binary directory to your shell profile (`.bashrc`, `.zshrc`, or `config.fish`), but the current terminal session doesn't pick up PATH changes automatically.

**Fix:** Restart your terminal, or source your profile:
```sh
source ~/.bashrc   # or ~/.zshrc
```

If the binary isn't in the expected location, check:
- Linux/macOS: `/usr/local/bin/yard` or `~/.local/bin/yard`
- Windows: `$env:LOCALAPPDATA\yard\bin\yard.exe`

You can also set `YARD_INSTALL_DIR` and re-run the installer to choose a custom location.

---

## Port 9876 already in use during `yard login`

The CLI starts a local callback server on port 9876 to receive the OAuth token. If another process is using that port, login fails immediately.

**Fix:** Find and stop the conflicting process:
```sh
# Linux/macOS
lsof -i :9876
kill <PID>

# Windows
netstat -ano | findstr :9876
taskkill /PID <PID> /F
```

Then re-run `yard login`.

---

## `yard init` in a non-git folder

`yard init` works outside a Git repository — the project will simply be created without a linked GitHub repo. If you *want* the project linked to a repo, make sure you run `yard init` from inside a Git repository that has a GitHub remote named `origin`.

**Fix:**
```sh
# Make sure you're in the repo root
cd /path/to/your-repo

# Check that origin points to GitHub
git remote -v
```

If origin points to a non-GitHub host (GitLab, Bitbucket, etc.), Yard will skip the linking step and print a one-line note. Only GitHub repositories are supported for linking.

---

## Session expired

Sessions last 30 days. When expired, any authenticated CLI command will fail.

**Fix:**
```sh
yard login
```

This runs a fresh OAuth flow and saves a new session token.

---

## "A team is required" / `NO_TEAM` 403

Every seller-side command — `yard init`, `yard projects`, `yard coupons`, `yard keys`, `yard push` — acts on a **team**, because teams own projects. An account that belongs to no team can authenticate fine and still fail all of them with a `403` carrying `code: "NO_TEAM"`.

This is **not** a plan problem. Upgrading changes nothing, and any message suggesting an upgrade here is misleading.

**Fix:** create a team at https://yard.sh/team, then confirm:
```sh
yard team
```

Signup normally creates a team on the way through, so this mostly shows up on accounts that exited onboarding early.

---

## Commands act on the wrong team

Projects or coupons that exist in the dashboard don't show up in the CLI (or land under an unexpected username). The CLI acts as **one** team at a time, and which one is stored on the account — the same setting the dashboard's team switcher writes — so it can change out from under a session.

**Fix:** check and switch:
```sh
yard team                  # who am I acting as?
yard team use acme         # switch (the leading @ is optional)
```

Because the setting is shared, switching in the browser changes what the CLI sees and vice versa. If a public project URL 404s, compare its username against `yard team --json` → `.active_team.username` — a project lives under its owning team's username, never under the seller's username.

---

## GitHub App not installed

If you haven't installed the Yard GitHub App and you want to link a repo during `yard init`, the CLI will:
1. Open your browser to the GitHub App installation page
2. Wait up to 5 minutes for you to complete the installation

If it times out, you close the browser, or the install fails, `yard init` falls back to creating the project without a linked repo. To retry the link later:
1. Go to https://github.com/apps/yard-app-official/installations/new
2. Select the account/org and grant access to the repositories you want to sell
3. Link the repo from the dashboard, or delete the project and re-run `yard init`

---

## "repository is already listed as a project"

Each GitHub repository can only be published as one Yard project. If you've already published it, use the web dashboard to manage the existing project.

---

## Price validation errors

- **"minimum price is $3.00"** — Paid projects must be at least $3.00. Enter `0` for a free project.
- **"price cannot be negative"** — Prices must be zero or positive.
- **"could not parse price"** — Enter a number like `5`, `5.00`, or `9.99`. Don't include the `$` sign.

---

## Update required to init

`yard init` checks for CLI updates before proceeding. If a newer version is available, you must update to continue.

**Fix:**
```sh
yard update
```

Then re-run `yard init`.

---

## Permission denied during install or update

If the installer or `yard update` can't write to the binary location:

**Linux/macOS:**
```sh
# Option 1: Install to a user-writable location
YARD_INSTALL_DIR=~/.local/bin curl -fsSL cli.yard.sh | sh

# Option 2: Use sudo for /usr/local/bin
sudo curl -fsSL cli.yard.sh | sh
```

**Windows:** Run PowerShell as Administrator, or set `$env:YARD_INSTALL_DIR` to a writable location.

---

## Browser doesn't open automatically

If `yard login` or the GitHub App installation can't open your browser:

1. Copy the URL printed in the terminal
2. Paste it into your browser manually
3. Complete the flow in the browser
4. The CLI will detect the callback automatically

This commonly happens in headless environments, SSH sessions, or WSL without browser integration.

---

## Coder workspace connectivity

When running in a Coder workspace, the CLI automatically detects the `VSCODE_PROXY_URI` environment variable and constructs proxy-aware URLs for the OAuth callback. If login fails in a Coder workspace:

1. Verify `VSCODE_PROXY_URI` is set: `echo $VSCODE_PROXY_URI`
2. Ensure the workspace proxy allows traffic on port 9876
3. Try the URL printed in the terminal manually

---

## `.yard/settings.json` uses an old service layout

A service's settings - `name`, `url`, `access`, `database_access` - live on
its entry in the `services` list of `.yard/settings.json`. Three retired
layouts are rejected rather than upgraded, because reading them would have to
guess values the seller chose:

**Services entries without a `"name"` (v5)** - the settings lived in each
directory's own `settings.json`. Run `yard migrate`: it folds every
per-directory settings file onto its entry, deletes those files, and stamps
`"version": 7`. Or move the fields by hand and delete the files.

**A services entry carrying `database` (v6)** - the flag was renamed to
`database_access`, because it only binds `env.DB`; the release's migrations
are what create the database. Run `yard migrate`: it renames the key on every
entry and stamps `"version": 7`. Or rename it by hand and set `"version": 7`.

**A top-level `"service"` (or `"app"`) block (v4 and older)** - convert by
hand:

1. Replace the block with a `services` list entry carrying its old values:
   `"services": [{ "dir": "service", "name": "service", "url": "/service", "access": "public", "database_access": true }]`.
2. Set `"version": 7`.

Or start over with `yard service init <name>`, which records the entry for
you.

## A GitHub tag fails to sync after upgrading

Tag content is immutable. A tag whose `.yard/settings.json` still uses the
retired v5 layout, or the retired v6 `database` key, fails the sync with an
error naming the fix; the release on Yard keeps serving as it was. Run
`yard migrate` in the repo, commit, and publish the next tag - or force-move
the tag and use the dashboard's Re-sync.
