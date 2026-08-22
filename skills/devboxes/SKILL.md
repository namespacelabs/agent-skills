---
name: devboxes
description: Create, navigate, hydrate, and tear down Namespace devboxes - Linux/amd64 or macOS/arm64 remote development machines. Use whenever the user needs an isolated Linux or macOS environment, a remote dev environment, a Linux Docker host, a cloud machine to run code or shell commands, or wants to spin up ephemeral machines to run a test suite (including sharding across multiple devboxes in parallel). Also triggers on the words "devbox", "Namespace devbox", or any ask for a disposable Linux or macOS machine.
---

# Namespace devboxes

Namespace devboxes are remote `linux/amd64` or `macos/arm64` machines. Linux devboxes have Docker available. This skill describes the general building blocks for working with devboxes - creating them, transferring files, hydrating a workspace, installing toolchains, running commands with `devbox exec`, sharing web work with teammates through `devbox.so`, and managing the devbox lifecycle.

Throughout this skill:

- **devbox** - a Namespace machine, identified by its `name`.
- `<name>` - the devbox name passed to all `devbox` subcommands.

The devbox may be long-lived, reused across tasks, or torn down immediately depending on the caller's intent. Pass `--ephemeral` at creation time to mark it as disposable.

For run-once-then-destroy workflows (single-script or test runs, optional sharding), see [references/devboxes-run-tests.md](references/devboxes-run-tests.md).

## 1. Create a devbox

If available, pick an image for the requested platform that already includes the project's toolchain (Go, Node, Xcode, etc.) to avoid reinstalling dependencies on every run. Start by running `devbox image list -o json` to discover existing project images - if one fits, use it. For simpler Linux cases, fall back to `builtin:base` and install dependencies directly.

**Important** Pass the short `name` field from `devbox image list -o json` to `--image` (e.g. `<org>/<image>` or `builtin:base`), not the full `repository` URL - full references typically fail. Ensure the image matches the requested platform.

```bash
devbox create \
  --name <name> \
  [--platform <linux/amd64|macos/arm64>] \
  --image <image> \
  --size <size> \
  [--ephemeral] \
  [--checkout <repo-url>] \
  --purpose "<purpose>"
```

Linux/amd64 is the default when `--platform` is omitted. Pass `--platform macos/arm64` when the task requires macOS or Apple Silicon.

Add `--ephemeral` when the devbox does not need to persist after a task is complete; omit it when persistence is required.

**Important (workspace checkout default)** If `--checkout` is unset, Namespace uses the workspace's configured default repo and clones it automatically. When no repository is needed (e.g. a scratch VM for a one-off command), pass `--no_checkout` to skip cloning entirely.

`--checkout <repo-url>` (e.g. `git@github.com:org/repo.git`) clones the repository into `/workspaces/<repo-name>`. If creation fails because checkout is unavailable for the repo or org:

1. Expire the just-created devbox (`devbox expire <name> --force`) to free capacity.
2. If `gh` is installed locally, recreate the devbox WITHOUT `--checkout`, then run `devbox setup-github <name>` to forward your `gh` token into the devbox. Once that succeeds, run `git clone --depth=1 <repo-url> /workspaces/<repo-name>` inside the devbox via `devbox exec <name> --`. Always use `--depth=1` for large monorepos - full clones can run for minutes with sparse progress output and look identical to a hang. **Important** The forwarded `gh` token authenticates HTTPS only - clone via `https://github.com/<org>/<repo>.git`, not `git@github.com:...`.
3. If `gh` is not installed, or `devbox setup-github` / the in-devbox `git clone` fails, **instruct the user to link their GitHub organization to Namespace** at https://cloud.namespace.so/workspace/workspace/integrations?integration=github - checkout cannot proceed without it.

**Important (case-sensitive org slug)** The Namespace association lookup is case-sensitive on the GitHub org/owner segment of `--checkout`. Use the org/owner casing EXACTLY as it appears in the repo URL the user gave you (or as returned by `git remote -v`). NEVER try alternative casings (e.g. lowercasing the org) as a "fix" when checkout fails.

Linux sizes: `s` (4 vCPU / 8 GB), `m` (8 vCPU / 16 GB), `l` (16 vCPU / 32 GB), `xl` (32 vCPU / 64 GB). macOS sizes: `m` (6 vCPU / 14 GB) and `l` (12 vCPU / 28 GB). Use the smallest size only for narrow commands or small targeted workloads; prefer bigger when the workload has large dependencies or builds multiple packages, to avoid OOM. If a process is killed with `signal: killed`, upgrade to a larger size and retry.

**Overlap creation with local prep** `devbox create` blocks until the machine is ready, which takes tens of seconds. Use that time: run `devbox create` as a background task, then immediately compute any local git diff and write the setup and run scripts locally. Once creation completes, upload and execute. This eliminates idle waiting.

**Note** Devbox creation might fail due to Namespace plan limits on resource tiers or instance counts. If this happens: **PROMPT THE USER BEFORE PROCEEDING** run `devbox list` and ask to expire or reuse any of the available devboxes, then retry; As a last resort, consolidate work across fewer devboxes rather than blocking.

For test or script runs that benefit from sharding across multiple devboxes, see [references/devboxes-run-tests.md](references/devboxes-run-tests.md).

## 2. Hydrate the workspace and install toolchains

### Upload and download files

```bash
devbox upload <name> <local-path>  <remote-path> [--mkdir]
devbox download <name> <remote-path> <local-path>
```

- `<remote-path>` MUST be a full file path. A trailing `/` fails.
- `--mkdir` creates missing parent directories; it does NOT make the target a directory.
- The executable bit is NOT preserved. Run `chmod +x` with `devbox exec` after uploading scripts.

### Hydrate the workspace

Namespace devboxes configured with a repo in the UI auto-clone it to `/workspaces/<repo-name>` on startup. Always check there first before syncing:

```bash
devbox exec <name> -- ls /workspaces/
```

If the repo is already present, use that path directly. The auto-cloned repo reflects the default branch. When a repository is checked out (auto-cloned or via `--checkout`), align it to the same commit as your local working tree and apply any uncommitted local changes on top.

**Important (local uncommitted changes)** Devbox clones the default branch - diff and checkout must use the same commit or unrelated files get clobbered.

```bash
BASE=$(git merge-base origin/main HEAD)
git diff "$BASE" > /tmp/changes.patch   # add --binary if the diff touches generated/binary files

# Run each command separately - do NOT chain with && via `bash -c`.
# For git, use `git -C <path>` instead of `cd <path> && git ...`.
devbox exec   <name> -- git -C /workspaces/<repo-name> fetch --depth=1 origin "$BASE"
devbox exec   <name> -- git -C /workspaces/<repo-name> checkout "$BASE"
devbox upload <name> /tmp/changes.patch /tmp/changes.patch
devbox exec   <name> -- git -C /workspaces/<repo-name> apply /tmp/changes.patch
devbox exec   <name> -- git -C /workspaces/<repo-name> status --short

# When multiple steps must share state that cannot be expressed as a single command
# (e.g. exported env vars, multi-line logic), write a script, upload it, then run it:
#   devbox upload <name> /tmp/apply.sh /tmp/apply.sh
#   devbox exec   <name> -- chmod +x /tmp/apply.sh
#   devbox exec   <name> -- bash /tmp/apply.sh
```

**Important** `git diff` only covers tracked files. For untracked files, upload them directly with devbox upload, bundle a few into a tarball, or rsync them if there are many.

```bash
# for rsync - configure native ssh
devbox configure-ssh <name>
# After running this, you can connect with:
ssh <name>.devbox.namespace
# Then list untracked files and feed them to rsync
git ls-files --others --exclude-standard | \
  rsync -av --files-from=- . \
  <name>.devbox.namespace:/workspaces/<repo-name>/
```

### Install toolchains

If a tool is missing, install only what the workload needs.

**Important** Check `--version` or `command -v` for each required tool first - it may already be present. On Linux, `builtin:base` provides some languages (e.g. Go) via an on-demand shim that installs on first use; try the command before manually installing.

**Important** For every CLI tool the script uses, check if it exists first and install only if missing. Use platform-appropriate installers and artifacts. For example, on Linux/amd64:

```bash
devbox exec <name> -- apt-get install -y --no-install-recommends \
    git ca-certificates curl tar build-essential
devbox exec <name> -- curl -fsSL -o /tmp/go.tgz https://go.dev/dl/go<version>.linux-amd64.tar.gz
devbox exec <name> -- tar -xzf /tmp/go.tgz -C /usr/local
```

## 3. Run commands with `devbox exec`
**Important** Run `devbox exec` invocations against different devboxes in parallel. Do NOT serialize independent work.

```bash
devbox exec <name> -- <cmd> <args...>
```

`devbox ssh <name>` remains available for interactive shells and TTY-dependent workflows. Prefer `devbox exec` for non-interactive commands.

**Alternative: native ssh** If a workflow uses a tool, which relies on standard SSH tooling (`scp`, `rsync`, etc.), run `devbox configure-ssh <name>` once to write an entry into your `~/.ssh/config`. After that, the devbox is reachable as `<name>.devbox.namespace`. Prefer `devbox exec` for commands; reach for native ssh when you specifically need standard tooling.


**Important** `devbox exec` runs a command directly without a shell. Run one command per invocation; shell operators such as `&&`, redirections, and variable expansion are not interpreted. For anything beyond a single command, write the script to a file, `devbox upload` it, then run `devbox exec <name> -- bash /tmp/script.sh`.

### Detached commands and retained logs

Use `devbox exec -d` for long-running commands. It starts the command, returns immediately, and prints an exec ID. Use that ID to stream retained and live output with `devbox logs`. `devbox logs list` shows previous executions and their status.

```bash
devbox exec -d <name> -- <cmd> <args...>
# exec_<id>

devbox logs <name> <exec-id>
devbox logs list <name>
```

**Important** Preflight every required CLI before `set -e` so a missing tool exits with a clear message rather than an opaque failure: `command -v <tool> >/dev/null || { echo "MISSING_TOOL: <tool>"; exit 127; }`. On exit 127 or a `MISSING_TOOL` marker, install the dependency and re-run - do NOT report the run as failed.

**Important** Write all script output to a remote log file and return only a tiny summary by default.

```bash
cat > /tmp/run.sh <<'EOF'
#!/bin/bash
export PATH=/usr/local/go/bin:$PATH
command -v <tool> >/dev/null || { echo "MISSING_TOOL: <tool>"; exit 127; }
set -euo pipefail
mkdir -p /workspaces/tmp
export TMPDIR=/workspaces/tmp
LOG=/workspaces/run.log

cd /workspaces/<repo-name>
status=0
<workload-cmd> >"$LOG" 2>&1 || status=$?

echo "exit=$status log=$LOG bytes=$(wc -c <"$LOG")"
exit "$status"
EOF
devbox upload <name> /tmp/run.sh /tmp/run.sh
devbox exec   <name> -- chmod +x /tmp/run.sh
devbox exec   <name> -- bash /tmp/run.sh

# Pull the full log only if the summary shows a non-zero exit.
# devbox download <name> /workspaces/run.log /tmp/run.log
```

**Note** Capture the workload's exit code (`status=0; cmd || status=$?`) before printing the summary. Use `devbox download` only for devboxes whose summary reported a non-zero exit, unless specified otherwise.

For test-runner-specific guidance (sharding, parallel runs, the hydrate-and-test recipe, result reporting), see [references/devboxes-run-tests.md](references/devboxes-run-tests.md).

## 4. Lifecycle

```bash
devbox create --name <name> [--platform <linux/amd64|macos/arm64>] --image <image> --size <size> [--ephemeral] [--checkout <repo-url>] --purpose "<purpose>"
devbox exec <name> -- <cmd>
devbox exec -d <name> -- <cmd>                                     # run in the background and print an exec ID
devbox logs <name> <exec-id>                                       # stream retained and live output
devbox logs list <name>                                            # inspect retained executions and status
devbox ssh <name>                                                  # open an interactive shell; prefer exec for commands
devbox list [-o json]                                              # inspect running devboxes
devbox upload <name> <local-path> <remote-path>                    # transfer files to devbox
devbox download <name> <remote-path> <local-path>                  # retrieve files from devbox
devbox configure-ssh <name>                                        # add to ~/.ssh/config for native ssh/scp/rsync
devbox url access <name> [--mode <private|workspace>] -o json      # inspect or change devbox.so access
devbox url expose <name> --port <port> [--name <purpose>] -o json  # create a persistent devbox.so URL
devbox url get <name> (--port <port> | --name <purpose>) -o json   # retrieve an exposed URL
devbox url list <name> -o json                                     # list exposed URLs
devbox url unexpose <name> (--port <port> | --name <purpose>)      # remove an exposed URL
devbox port-forward <name> --ports <local:remote,...>              # forward devbox ports to localhost (e.g. a dev server or DB)
devbox expire <name> --force                                       # tear down a devbox
```

**Important** These `devbox url` commands require devbox CLI v0.0.182 or newer. Check with `devbox version`; if the installed version is older, run `devbox update` before using them.

**Choose who can open `devbox.so` URLs** Each devbox has one access mode shared by all of its exposed URLs:

- `private` - only the devbox owner.
- `workspace` - members of the same Namespace workspace.

The URL mode initially follows the general devbox access mode, but is configured separately afterward. Changing URL access does not grant or remove SSH, exec, or file access to the devbox.

Inspect or change the Devbox-wide URL mode independently:

```bash
devbox url access my-box -o json
devbox url access my-box --mode workspace -o json
```

**Important** `devbox url expose ... --access <mode>` changes this same Devbox-wide setting while exposing a port. It affects every existing and future exposed URL on that devbox; it does not set access only for the new URL. Per-URL access modes are not supported. If URLs need different audiences, ask before changing access or use a separate devbox.

**Share web work with teammates** When the user intends work running in a devbox to be reviewable by teammates, expose its port with `--access workspace` and return the resulting `devbox.so` URL. The URL remains available after the command exits. Pass `--name` when the port's purpose is known so the URL is easier to retrieve and remove later. Prefer JSON output for agent workflows.

```bash
devbox url expose my-box --port 3000 --name web --access workspace -o json
```

If the user has not asked to share, omit `--access` and preserve the current mode. When sharing intent is unclear, inspect the mode and do not broaden access. If the user explicitly requires every exposed URL on the devbox to be owner-only, set `private` before exposing the new port:

```bash
devbox url access my-box --mode private -o json
devbox url expose my-box --port 3000 --name web -o json
```

For local-only access or non-web services, use `devbox port-forward`. Example: `devbox port-forward my-box --ports 3000:3000,5432:5432` maps ports to `localhost:3000` and `localhost:5432`. The command blocks the terminal until Ctrl+C, so run it in a dedicated terminal or as a background process.

Whether and when to call `devbox expire` depends on the caller's intent - the test-suite workflow in [references/devboxes-run-tests.md](references/devboxes-run-tests.md) tears down at the end; other use cases may keep the devbox alive.

**Important** On failure caused by missing dependencies, install them and retry the failing command. Do the same for any uploaded scripts.
