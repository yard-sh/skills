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

## `yard login` says the code expired

The one-time code is valid for 15 minutes. If it wasn't entered and confirmed in the browser in time, the CLI stops with "the code expired before it was used".

**Fix:** Run `yard login` again and use the new code. If you cancelled on the authorize screen, the CLI keeps waiting until the code expires; press Ctrl-C and start over.

---

## `yard dev`: port 9875 is in use

Another process (often a previous `yard dev`) holds the port. Pass `--port <n>`, or stop the other process (`lsof -i :9875` on Linux/macOS, `netstat -ano | findstr :9875` on Windows).

---

## `yard dev`: the runtime download did not verify

The CLI only runs a runtime whose checksum matches the one it expects; a mismatch means an incomplete or tampered download. Run `yard update`, then `yard dev` again. If it persists, delete `~/.yard/runtime/` and retry.

---

## `yard dev is not available on this platform yet`

There is no local runtime build for Windows on ARM. Test with a sandbox instead: `yard push`, publish, `yard sandbox pin <tag> --sandbox preview`, `yard service open --sandbox preview`. On Linux the runtime needs glibc 2.35 or newer (`ldd --version`); on macOS, 13.5 or newer.

---

## `yard init` in a non-git folder

`yard init` works outside a Git repository; the project is simply created without a linked GitHub repo. If you _want_ the project linked to a repo, make sure you run `yard init` from inside a Git repository that has a GitHub remote named `origin`.

**Fix:**

```sh
# Make sure you're in the repo root
cd /path/to/your-repo

# Check that origin points to GitHub
git remote -v
```

If origin points to a non-GitHub host (GitLab, Bitbucket, etc.), Yard will skip the linking step and print a one-line note. Only GitHub repositories are supported for linking.

---

## Session expired or revoked

Sessions last up to 90 days; in between, the server renews the short-lived access token transparently. A session also ends when it is revoked from the security page's command-line sessions, when the password changes, or after `yard logout`. Once it has ended, any authenticated CLI command fails with a 401.

**Fix:**

```sh
yard login
```

This runs the device sign-in again and saves a new session.

---

## "A team is required" / `NO_TEAM` 403

Every seller-side command (`yard init`, `yard projects`, `yard coupons`, `yard keys`, `yard push`) acts on a **team**, because teams own projects. An account that belongs to no team can authenticate fine and still fail all of them with a `403` carrying `code: "NO_TEAM"`.

This is **not** a plan problem. Upgrading changes nothing, and any message suggesting an upgrade here is misleading.

**Fix:** create a team at https://yard.sh/team, then confirm:

```sh
yard team
```

Signup normally creates a team on the way through, so this mostly shows up on accounts that exited onboarding early.

---

## Commands act on the wrong team

Projects or coupons that exist in the dashboard don't show up in the CLI (or land under an unexpected username). The CLI acts as **one** team at a time, and which one is stored on the account (the same setting the dashboard's team switcher writes), so it can change out from under a session.

**Fix:** check and switch:

```sh
yard team                  # who am I acting as?
yard team use acme         # switch (the leading @ is optional)
```

Because the setting is shared, switching in the browser changes what the CLI sees and vice versa. If a public project URL 404s, compare its username against `yard team --json` → `.active_team.username`: a project lives under its owning team's username, never under the seller's username.

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

- **"minimum price is $3.00"**: Paid projects must be at least $3.00. Enter `0` for a free project.
- **"price cannot be negative"**: Prices must be zero or positive.
- **"could not parse price"**: Enter a number like `5`, `5.00`, or `9.99`. Don't include the `$` sign.

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
2. Paste it into a browser on any device
3. Complete the flow there (for `yard login`, enter the printed one-time code and confirm)
4. The CLI picks up the result on its own

This is normal in headless environments, SSH sessions, remote workspaces and WSL: nothing on the machine has to be reachable from your browser.

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
