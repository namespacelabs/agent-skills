# Running tests on Namespace devboxes

Test-specific additions to the general devbox building blocks in [../SKILL.md](../SKILL.md). Read SKILL.md first - this file only covers additional topics that apply when the workload is a test suite.

## Workflow

Test runs are fire-once-and-tear-down: each devbox exists to execute a single suite and is destroyed afterwards.

1. Fire `devbox create --ephemeral` (one per shard) in the background.
2. While the devboxes initialize: compute local diffs, write setup and run scripts.
3. Once ready: apply uncommitted changes, upload scripts, install dependencies, run tests.
4. Always expire every devbox at the end of the task - including failure paths - unless the user asks to keep one for debugging.

## Sharding across multiple devboxes

**Important** Default to splitting the test suite across multiple devboxes created simultaneously. Before deciding on the devbox count, count work units per planned shard (test files, packages, or equivalent for the language) and verify they are roughly equal - **not lines of code**, which are a poor proxy. Justify. Only use a single devbox if the repo is clearly small and single-purpose. Reason the chosen count of devboxes.

Run `devbox exec` invocations against different devboxes in parallel - do NOT serialize independent shards.

## Test runner notes

**Note** When using a bare `.` as a test target, confirm the directory actually contains testable source files first - running a test runner against an empty or non-package directory typically exits non-zero.

**Note** Capture the test process's exit code (`status=0; go test ./... || status=$?`) before printing the summary so a failing suite still reports cleanly. Use `devbox download` only for devboxes whose summary reported a non-zero exit, unless specified otherwise.

## Hydrate-and-test recipe

Concrete end-to-end example for a Go suite on Linux/amd64. Adapt the platform, image, toolchain install, and test command for other environments.

```bash
NAME=test-$(date +%s)

devbox create \
  --name "$NAME" \
  --platform linux/amd64 \
  --image builtin:default \
  --size <size> \
  --ephemeral \
  --checkout <repo-url> \
  --purpose "<purpose>"

# Check if the repo was auto-cloned by Namespace (configured repos land at /workspaces/<repo-name>)
devbox exec "$NAME" -- ls /workspaces/

# Check for the toolchain BEFORE installing - builtin:default already ships Go (and other
# common languages), so this is usually a no-op that saves a download and an untar.
devbox exec "$NAME" -- bash -lc 'go version || echo NEEDS_GO'

# ONLY if the check reported NEEDS_GO:
# devbox exec "$NAME" -- curl -fsSL -o /tmp/go.tgz https://go.dev/dl/go<version>.linux-amd64.tar.gz
# devbox exec "$NAME" -- tar -xzf /tmp/go.tgz -C /usr/local

cat > /tmp/run.sh <<'EOF'
#!/bin/bash
export PATH=/usr/local/go/bin:$PATH
command -v go >/dev/null || { echo "MISSING_TOOL: go"; exit 127; }
set -euo pipefail
mkdir -p /workspaces/tmp
export TMPDIR=/workspaces/tmp
LOG=/workspaces/run.log
cd <repo_dir>
status=0
go test ./... >"$LOG" 2>&1 || status=$?
echo "exit=$status log=$LOG bytes=$(wc -c <"$LOG")"
exit "$status"
EOF
devbox upload "$NAME" /tmp/run.sh /tmp/run.sh
devbox exec   "$NAME" -- chmod +x /tmp/run.sh
devbox exec   "$NAME" -- bash /tmp/run.sh

# Only download the full log if the summary above shows a non-zero exit.
# devbox download "$NAME" /workspaces/run.log /tmp/run.log

devbox expire "$NAME" --force
```

## After tests

After the final step, print a result summary as an ASCII table followed by a short comment. If any devbox creation was skipped or constrained due to plan limits or size restrictions, note it explicitly.
