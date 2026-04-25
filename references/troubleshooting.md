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

`yard init` works outside a Git repository — the product will simply be created without a linked GitHub repo. If you *want* the product linked to a repo, make sure you run `yard init` from inside a Git repository that has a GitHub remote named `origin`.

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

## GitHub App not installed

If you haven't installed the Yard GitHub App and you want to link a repo during `yard init`, the CLI will:
1. Open your browser to the GitHub App installation page
2. Wait up to 5 minutes for you to complete the installation

If it times out, you close the browser, or the install fails, `yard init` falls back to creating the product without a linked repo. To retry the link later:
1. Go to https://github.com/apps/yard-app-official/installations/new
2. Select the account/org and grant access to the repositories you want to sell
3. Link the repo from the dashboard, or delete the product and re-run `yard init`

---

## "repository is already listed as a product"

Each GitHub repository can only be published as one Yard product. If you've already published it, use the web dashboard to manage the existing product.

---

## Price validation errors

- **"minimum price is $3.00"** — Paid products must be at least $3.00. Enter `0` for a free product.
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
YARD_INSTALL_DIR=~/.local/bin curl -fsSL install.yard.sh | sh

# Option 2: Use sudo for /usr/local/bin
sudo curl -fsSL install.yard.sh | sh
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
