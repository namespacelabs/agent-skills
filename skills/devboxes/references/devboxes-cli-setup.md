# Devbox CLI setup: install, log in, update

Prerequisites for everything in [../SKILL.md](../SKILL.md). Read this only when the
preflight in SKILL.md fails - that is, when `devbox` is not on PATH, or
`devbox auth check-login` exits non-zero. During normal operation none of this is needed.

## Resolve the executable

Do not assume the Devbox CLI is absent merely because `devbox` is not on the current
shell's PATH. Check PATH first, then its default install location:

- macOS/Linux: `~/.local/bin/devbox`
- Windows: `%LOCALAPPDATA%\Programs\devbox\devbox.exe`

Resolve the executable once on macOS/Linux:

```bash
if command -v devbox >/dev/null 2>&1; then
  DEVBOX=$(command -v devbox)
elif [ -x "$HOME/.local/bin/devbox" ]; then
  DEVBOX="$HOME/.local/bin/devbox"
fi
```

Or in Windows PowerShell:

```powershell
$DevboxCommand = Get-Command devbox -ErrorAction SilentlyContinue
$Devbox = if ($DevboxCommand) {
  $DevboxCommand.Source
} else {
  Join-Path $env:LOCALAPPDATA "Programs\devbox\devbox.exe"
}
```

Confirm the resolved path exists. If it does, use it for every subsequent command
(`"$DEVBOX" version` in macOS/Linux or `& $Devbox version` in PowerShell). A newly
installed binary may not be visible to the agent's already-running shell even though
the installer updated the user's PATH.

## Install

If the executable is absent from both PATH and the platform's default install location,
run the official installer for the user's operating system yourself.

**Important (`devbox` is an ambiguous name)** Use only the URL below. Jetify ships an
unrelated Nix-based tool also called `devbox`, installed by piping a script from
`get.jetify.com/devbox` - so a web search for "devbox install" can easily return the
wrong product, and installing it will not help. The Devbox CLI comes from
`get.namespace.so/devbox/...` and nowhere else.

**macOS or Linux:**

```bash
curl -fsSL get.namespace.so/devbox/install.sh | bash
```

**Windows (PowerShell):**

```powershell
irm https://get.namespace.so/devbox/install.ps1 | iex
```

On Windows, restart the terminal if `devbox` is not available on `PATH` after
installation. The agent does not need to restart: it can execute
`& "$env:LOCALAPPDATA\Programs\devbox\devbox.exe" version` in PowerShell. Run the
installed binary's `version` command to verify the installation.

## Log in

Run `devbox login` as a long-running command. It opens the login URL in the user's
default browser and prints the URL and verification code:

```bash
devbox login
```

Example output:

```text
Please complete the login flow in your browser.

https://cloud.namespace.so/login/workspace?id=9l2drnfnt50chd7c119gtb2o48&code=A1B2-C3D4
Verification code: A1B2-C3D4
```

The CLI opens the user's default browser. Tell the user to complete authentication in
that browser. **The agent cannot complete this browser step.** Do not ask the user to
confirm when they are done; keep waiting for the `devbox login` process, which returns
on its own after authentication completes.

Then verify that the credentials are valid:

```bash
devbox auth check-login
```

`check-login` returns a non-zero exit status when credentials are missing or invalid,
which makes it the reliable check for agent workflows.

**Important (sessions expire)** Authentication is not permanent. `devbox auth check`
prints the expiry alongside the tenant:

```text
Logged in.
  Tenant: tenant_<id>
  Expires: 2026-10-15T12:20:36+01:00
```

So a machine that worked last week can fail today with nothing else changed. When
`devbox` commands start failing in confusing ways, check this before assuming the
failure is in the workload.

**Important (do not improvise credentials)** If authentication is the problem, run
`devbox login` and have the user complete the browser flow - nothing else. Do not
attempt to mint or borrow tokens from adjacent tooling to work around it; that path
does not authenticate `devbox` and wastes time. In particular, `devbox.so` URLs are
gated on a signed-in browser session and no API or ID token substitutes for it (see
"Share web work with teammates" in SKILL.md).

## Update

```bash
devbox update
devbox version    # confirm what you ended up on
```

Run updates yourself when needed. `devbox update` is the CLI's own self-update
subcommand on an installed binary.

Run it when a documented minimum version is actually blocking the task (`devbox url`
needs v0.0.182 or newer, noted at the point of use in SKILL.md). Do NOT update just
because the CLI printed an "a new version is available" banner: that banner is appended
to ordinary command output, is not an error, and chasing it mid-task changes tooling
underneath a run for no benefit.

## Switch Namespace workspaces

Run `devbox login` to switch the Devbox CLI to a different Namespace workspace. Let the
user complete the browser flow, wait for the command to return on its own, then verify
the new login with `devbox auth check-login`.

## Recognising a missing CLI

The shell reports it rather than `devbox` doing so, so the wording varies: `command not
found` (macOS/Linux) or `not recognized` (Windows). This only means the current shell
could not resolve it. Check the platform's default install location before deciding the
CLI is absent. If it is genuinely absent, install it using the official command above,
verify with the installed binary's `version` command, and retry the original command.

## Upstream documentation

- [Devbox CLI installation and update](https://namespace.so/docs/reference/devbox-cli/installation.md)
- [`devbox login`](https://namespace.so/docs/reference/devbox-cli/auth.md)
