# Devbox CLI setup: install, log in, update

Prerequisites for everything in [../SKILL.md](../SKILL.md). Read this only when the
preflight in SKILL.md fails - that is, when `devbox` is not on PATH, or
`devbox auth check-login` exits non-zero. During normal operation none of this is needed.

## Install

Hand the user the command for their operating system and let them run it.

**Important (`devbox` is an ambiguous name)** Use only the URL below. Jetify ships an
unrelated Nix-based tool also called `devbox`, installed by piping a script from
`get.jetify.com/devbox` - so a web search for "devbox install" can easily return the
wrong product, and installing it will not help. The Namespace CLI comes from
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
installation. Run `devbox version` to verify the installation.

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

Tell the user to complete the flow in the browser. **The agent cannot complete this
browser step.** Do not ask the user to confirm when they are done; keep waiting for the
`devbox login` process, which returns on its own after authentication completes.

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

**Important (do not improvise credentials)** If authentication is the problem, the fix
is `devbox login` by the user - nothing else. Do not attempt to mint or borrow tokens
from adjacent tooling to work around it; that path does not authenticate `devbox` and
wastes time. In particular, `devbox.so` URLs are gated on a signed-in browser session
and no API or ID token substitutes for it (see "Share web work with teammates"
in SKILL.md).

## Update

```bash
devbox update
devbox version    # confirm what you ended up on
```

Unlike installing, this one you may run yourself. `devbox update` is the CLI's own
self-update subcommand on a binary the user already trusts - not a remote script piped
into a shell - so the reasoning that makes installing the user's call does not apply.

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
found` (macOS/Linux) or `not recognized` (Windows). Either means the CLI is absent, not
that the command was wrong.

Do NOT install it yourself. Give the user the command for their OS from the Install
section above and let them run it - it fetches and executes a remote script on their
machine, which is their call to make, not yours. Once they confirm, verify with
`devbox version` and retry the original command.

## Upstream documentation

- [Devbox CLI installation and update](https://namespace.so/docs/reference/devbox-cli/installation.md)
- [`devbox login`](https://namespace.so/docs/reference/devbox-cli/auth.md)
