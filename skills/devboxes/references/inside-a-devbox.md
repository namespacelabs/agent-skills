# Running inside a devbox

Read this when you are running ON a devbox rather than driving one from a user's machine. Two
things change: you are already on the machine, so ordinary commands need no wrapper; and the
tool that works without authentication is `boxctl`.

[../SKILL.md](../SKILL.md) remains the reference for anything that crosses machines - creating
devboxes, moving files, or driving a *different* devbox - and for task markers, which apply
whichever side you are on.

## Am I on a devbox?

```bash
[ -n "${NAMESPACE_DEVBOX_WORKSPACE_DIR:-}" ] && echo "on a devbox"
```

The Devbox agent injects this into every process on the machine, so it survives non-login
shells, plain `sh`, and SSH. `$HOME/.devbox/bin/boxctl` is an equivalent signal if you need one
that does not depend on the environment.

You are on the machine, so run `go test`, `git` and file edits directly. Do not wrap ordinary
work in `boxctl exec` - see below for the one reason to.

### Environment exposed to devbox processes

| Variable | Contents |
| --- | --- |
| `NAMESPACE_DEVBOX_WORKSPACE_DIR` | Workspace directory, where the repo is checked out |
| `NAMESPACE_DEVBOX_TASKS_DIR` | Task marker directory - `boxctl task` uses it for you |
| `DEVBOX_WORKSPACE_DIR` | Compatibility alias for the workspace directory |

Prefer `$NAMESPACE_DEVBOX_WORKSPACE_DIR` over a hardcoded path: the workspace differs by platform
(`/workspaces` on Linux, `/Users/runner/workspaces` on macOS) and between ephemeral and
persistent devboxes.

## Tooling: `boxctl` and the `devbox` CLI

Both ship with every devbox, symlinked into `~/.devbox/bin` and on `PATH`:

```
~/.devbox/bin/boxctl      # acts on THIS devbox, over a local socket
~/.devbox/bin/devbox      # the full CLI, already authenticated with a workload token
~/.devbox/bin/tmux        # the managed tmux client
~/.devbox/metadata.json   # identity and environment metadata
```

**Important (the CLI is already authenticated - do not run `devbox login`)** A devbox carries a
*workload* credential, so the `devbox` CLI works out of the box for most operations. It is not a
user login, and that difference shows up in two ways worth knowing before you hit them.

The first is a trap: `devbox auth check-login` and `devbox auth check` report **"Not logged in"**
even though the CLI is working fine. They describe user/OAuth login state and know nothing about
the workload credential. Treating that as a gate is how an agent ends up in a `devbox login`
browser flow it cannot complete, on a machine that never needed one. Inside a devbox, skip the
auth check and just run the command you wanted.

The second is what that credential *is*. It authenticates as the tenant, not as a person, which
has a sharp consequence: it cannot touch devboxes that are private to their creator - **including
the one it is running on**. Control-plane calls against this machine fail even though you are
sitting on it:

```
$ devbox exec $(boxctl metadata -o json --jq '.devbox.name' | tr -d '"') -- echo hi
rpc error: code = PermissionDenied desc = devbox "..." is private to its creator
```

That is the real reason `boxctl` exists, and why it is not merely a faster path: for a private
devbox it is the *only* way to act on the machine you are on.

| Operation from inside a devbox | Result |
| --- | --- |
| `boxctl <anything>` on this devbox | works, always - local socket, no credential involved |
| `devbox list`, `devbox version` | work |
| `devbox <exec\|url\|logs\|upload\|download\|expire> <workspace-access box>` | work |
| `devbox <exec\|url\|logs\|...> <private box, incl. this one>` | `PermissionDenied ... private to its creator` |
| `devbox create --access_mode workspace` | works |
| `devbox create` (default, private) | fails: `workload credentials don't have an actor` |
| `devbox image list` | fails: `PermissionDenied` |

Fanning work out to sibling devboxes therefore does work from inside one - test shards, say - and
it composes: a box you create from here must be `--access_mode workspace`, which is exactly the
kind the credential can then drive. Since `image list` is denied, pick `builtin:default` or another
image you already know rather than discovering one.

`boxctl` remains the better tool for the devbox you are on: it talks to the local agent over a
unix socket, so there is no control-plane round trip, and it needs no `<name>` because there is no
ambiguity about which machine you mean. It also covers things the CLI cannot do from here at all.

| Capability | `boxctl` (this devbox) | `devbox` CLI (any devbox) |
| --- | --- | --- |
| Identify this devbox | `metadata` | - |
| Keep a devbox awake | `task mark` / `clear-mark` / `list` | - |
| Open a URL in the user's browser | `open` | - |
| Terminal sessions | `session list/create/connect` | `session list/connect` only |
| Run with host-retained output | `exec` | `exec <name>` |
| Manage `devbox.so` URLs | `url access/expose/get/list/unexpose` | same, with `<name>` |
| Stop a devbox | `shutdown` | `shutdown <name>` |
| Read back retained logs | **not available** | `logs <name> <exec-id>` |
| Move files between machines | - | `upload`, `download` |
| Create / expire / list devboxes | - | `create` (see above), `expire`, `list` |
| SSH, port-forward, IDE | - | `ssh`, `configure-ssh`, `port-forward`, `open-ide` |

A full user login is only worth pursuing if you need something the workload credential genuinely
cannot do. That is rare from inside a devbox, and it needs the user at a browser - so ask rather
than starting the flow. `boxctl open` can hand them the URL when the terminal supports it:

```bash
if boxctl open --check; then boxctl open "<login-url>"; fi
```

See [cli-setup.md](cli-setup.md) for the login and expiry story on a user's own machine.

## Task markers

Long-running work outside an active `boxctl exec` or SSH command must be marked or the devbox can
stop underneath it. The concept, and the equivalent commands for driving a devbox from outside,
are in the lifecycle section of [../SKILL.md](../SKILL.md). From inside:

```bash
boxctl task mark <name>         # hold the devbox busy
boxctl task clear-mark <name>   # release it
boxctl task list [-o json]      # what is currently holding it
```

Both `mark` and `clear-mark` are idempotent and exit 0 when the marker already exists or is
already gone, so they are safe to re-run and safe in cleanup paths. Names must be a single path
component - no `/`.

**Important** Always pair a mark with an unwind path. Markers have no TTL: one left behind keeps
the machine awake and billed indefinitely. A trap covers the failure and interrupt paths that a
trailing `clear-mark` does not:

```bash
boxctl task mark my-build
cleanup() {
  status=$?
  trap - EXIT
  boxctl task clear-mark my-build
  exit "$status"
}
trap cleanup EXIT
trap 'exit 130' INT
trap 'exit 143' TERM

make build && make test
```

The signal traps exit with the conventional status and then let the `EXIT` trap clear the marker;
do not put cleanup directly in an `INT` or `TERM` trap and then continue the script.

Before finishing work on a devbox, check you left nothing behind with `boxctl task list`.

## Inspect metadata

```bash
boxctl metadata                                  # summary: Devbox ID and Name only
boxctl metadata -o json                          # everything
boxctl metadata -o json --jq '.devbox.name'
```

```json
{
  "version": "1-dev",
  "devbox": { "id": "...", "name": "...", "access_mode": "owner" },
  "paths": { "workspace": "/workspaces", "boxctl": "~/.devbox/bin/boxctl" }
}
```

`--jq` requires `-o json` and errors otherwise, and emits JSON - a string value arrives quoted,
so strip it before using it as a path (`| tr -d '"'`). There is no raw-output flag. The `name`
here is what to pass to `devbox` subcommands, or to give the user so they can reach this box.

**Important (two different access vocabularies)** `devbox.access_mode` in metadata describes who
can reach the *devbox* and reads `owner` or `tenant`. `devbox.so` URL access is a separate
setting and reads `private` or `workspace`. Do not translate one into the other or infer URL
access from metadata - read it with `boxctl url access`.

## Run commands with retained output

```bash
boxctl exec -- make test
```

The only reason to prefer this over running `make test` directly: output is retained in the
host-side devbox logs, which is what makes a run visible to the user watching from their own
machine. It streams stdout and stderr and exits with the command's own status. It also counts as
activity while it runs, so a foreground `boxctl exec` needs no task marker.

```bash
boxctl exec --detach -o json -- make test        # prints an exec ID, returns immediately
```

**Important (exec does not give you a shell)** Like `devbox exec`, `boxctl exec` runs the command
directly, so `&&`, `;`, redirections, globs and `$VAR` are not interpreted - they arrive as
literal arguments. Ask for a shell when you need them:

```bash
boxctl exec -- bash -lc 'cd "$NAMESPACE_DEVBOX_WORKSPACE_DIR" && go build ./... && go test ./...'
```

Unlike `devbox exec`, the command inherits your current working directory, so a `cd` is often
unnecessary.

**Important (you usually cannot read a detached exec back from inside)** `--detach` returns an
exec ID, but there is no `boxctl logs`, and the retained logs are host-side. `devbox logs <name>
<exec-id>` reads them - but on the workload credential that call fails for a devbox private to
its creator, which is the default and therefore the common case. So treat `--detach` as being for
work the *user* will watch from their own machine. When you need the output yourself, run in the
foreground or redirect to a file you can read directly:

```bash
boxctl exec -- bash -lc 'make test >/tmp/run.log 2>&1; echo "exit=$?"'
```

`-o json` requires `--detach`; a foreground `boxctl exec -o json` is rejected. A detached exec
counts as activity for its whole runtime, even when nobody is watching it, so it needs no task
marker. Mark only work that continues after the agent-managed exec itself has exited.

## Manage exposed URLs

Same devbox-wide access semantics as the `devbox url` commands in SKILL.md, minus the `<name>`:

```bash
boxctl url access -o json                                  # inspect
boxctl url access --mode workspace -o json                 # private | workspace
boxctl url expose --port 3000 --name web -o json
boxctl url expose --port 8080 --port 8081 -o json          # repeatable or comma-separated
boxctl url get --port 3000 -o json
boxctl url list -o json
boxctl url unexpose --port 8080,8081
```

Expose and unexpose are idempotent. `expose` also accepts `--access <private|workspace>`, which
changes the setting for **every** exposed URL on the devbox, not just the new one - per-URL
access does not exist. If the user has not asked to share, omit `--access` and leave the mode
alone.

**Important** Bind servers to `0.0.0.0`, not `127.0.0.1`, or the `devbox.so` proxy cannot reach
them. A successful `expose` that returns a URL IS the confirmation - do not `curl` the
`devbox.so` URL to prove it works. It is gated on a signed-in browser session, so it returns
`401` regardless, and chasing a `200` is the most expensive dead end in this skill. To check the
server itself, hit it locally:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:3000/
```

**Note** Requests to an exposed URL count as activity, so a service someone is actively using
keeps the devbox alive on its own. Traffic that stops - overnight, say - stops holding it.

## Persistent terminal sessions

Sessions are tmux sessions owned by the Devbox agent. They survive disconnects and agent
restarts, which makes them the right home for a long-running dev server or watcher. Creating or
connecting to one counts as recent activity only for a bounded period: output or a running process
inside a detached session does not continuously keep the devbox busy. Add a task marker for work
that must remain alive unattended.

```bash
boxctl session list -o json
boxctl session create work --cmd 'make dev' -o json
```

`create` is idempotent, and `--cmd` runs only when the session is genuinely new - it does not
rerun against an existing session, so it cannot be used to send a second command.

To drive a session non-interactively, create it with `boxctl`, then use the managed tmux client
against the agent's socket:

```bash
boxctl session create work
~/.devbox/bin/tmux -S /tmp/tmux.namespace.sock send-keys -t work 'make test' Enter
~/.devbox/bin/tmux -S /tmp/tmux.namespace.sock capture-pane -p -t work
```

`capture-pane -p` prints the pane's current contents - poll it to follow progress.

**Important (never run `boxctl session connect`)** It attaches the session to the current
terminal by replacing the running process, so calling it from an agent shell hijacks that shell
and never returns. It exists for humans at an interactive prompt. Use `session create` plus
`send-keys` / `capture-pane` instead.

**Note** Session names cannot contain `.` or `:`.

## Open a URL on the user's machine

Works only when the process is attached to a compatible devbox terminal, so check first:

```bash
if boxctl open --check; then
  boxctl open https://example.com
fi
```

Loopback URLs (`localhost`, `127.0.0.1`, `[::1]`) are forwarded to a matching listener inside the
devbox. That forward is deliberately short-lived: it closes roughly 30 seconds after its first
request, and after five minutes at most. It is built for auth callbacks and one-shot pages -
re-run `boxctl open` if the page stops responding, and use an exposed `devbox.so` URL or
`devbox port-forward` for anything long-running.

## Shut this devbox down

```bash
boxctl shutdown --force
```

**Important** `--force` is required non-interactively. Without it the command prompts on stderr
and reads stdin, so from an agent shell with no TTY it fails rather than shutting down. Treat
this as you would any teardown: stopping the machine ends everything running on it, including
the session you are in, so confirm with the user unless they have already asked for it.

`shutdown` stops the devbox; it does not destroy it. Destroying needs `devbox expire <name>
--force`, which the workload credential can do to workspace-access devboxes but not to one that
is private to its creator - so expiring *this* machine from inside generally fails, while
expiring a sibling you created here succeeds.

## Getting files in and out

`boxctl` has no `upload` or `download`. The bundled `devbox` CLI has both, and they work from
here against *workspace-access* devboxes - so moving files to a sibling box is fine, while a box
private to its creator is not reachable.

What that does not give you is the user's own machine, which the control plane cannot reach. To
get a file to them, either serve it over an exposed port, or have them run
`devbox download <name> <remote> <local>` on their laptop - `boxctl metadata` tells you the name
to quote. For code, `git` is usually the better answer than moving files at all.
