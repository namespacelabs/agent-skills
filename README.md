# Namespace plugin and agent skills

Skills and the hosted [Namespace MCP server](https://namespace.so/docs/reference/mcp) for Claude, Codex, Cursor, Gemini, Antigravity, and other agents. Investigate CI and builds, adopt Namespace runners and caching, optimize CI speed and cost, debug failed jobs, analyze compute usage, and manage Devboxes.

The plugin connects to `https://mcp.namespace.so/mcp`. On first use, your agent opens Namespace in the browser to authorize one workspace. For read-only access, skip the plugin and add `https://mcp.namespace.so/mcp/readonly` as described in the [MCP docs](https://namespace.so/docs/reference/mcp).

## Install

### Claude Code and Claude Cowork

```sh
claude plugin marketplace add namespacelabs/agent-skills
claude plugin install namespace@namespace
```

Then run `/mcp`, select `plugin:namespace:namespace`, and choose **Authenticate**. In Claude Cowork, open **Settings** > **Plugins**, click **Add** > **Add marketplace**, enter `namespacelabs/agent-skills`, then install **Namespace** from **Browse**.

### Codex

```sh
codex plugin marketplace add namespacelabs/agent-skills
```

Then run `/plugins` in Codex, select **Namespace**, and install it.

### Cursor

Install **Namespace** from the [Cursor Marketplace](https://cursor.com/marketplace) once it is listed. Until then, use the one-click install in the [MCP docs](https://namespace.so/docs/reference/mcp).

### Gemini CLI

```sh
gemini extensions install https://github.com/namespacelabs/agent-skills
```

### Antigravity

```sh
agy plugin install https://github.com/namespacelabs/agent-skills
```

### Skills only

Without the MCP server, for any agent that supports [Agent Skills](https://agentskills.io):

```sh
npx skills add namespacelabs/agent-skills
npx skills add namespacelabs/agent-skills --skill devboxes
```

## Skill catalog

| Skill | Use case |
|---|---|
| `adopting-namespace-runners` | Move GitHub Actions workflows onto Namespace runners and turn on Remote Builders and caching. References: `runner-labels.md` (label grammar, converting GitHub-hosted labels), `caching.md` (cache volumes, cache action, protecting caches) |
| `debugging-github-actions-jobs` | Find why one Namespace-run job failed, timed out, ran out of memory, ran slowly, or waited to start |
| `optimizing-namespace-ci` | Make CI on Namespace faster, cheaper, or more reliable from measured evidence: runner sizing, caching, repeated setup, Docker builds, queue delays, sharding. References: `evidence.md` (MCP tools, reports, interpretation limits) |
| `devboxes` | Spin up devboxes to run commands or workloads. References: `cli-setup.md` (install the CLI, log in), `run-tests.md` (test suites, sharding), `inside-a-devbox.md` (running inside a devbox: `boxctl`, task markers) |

## Data and permissions

- **What is sent:** each MCP tool call sends its arguments to `https://mcp.namespace.so/mcp` over HTTPS. Nothing is sent anywhere else, and the plugin collects no telemetry.
- **What the server reads:** data in the one Namespace workspace you authorize: GitHub Actions jobs and their steps, logs, and resource usage; builds; runner profiles; instances; and Devboxes.
- **What the server changes:** only Devboxes. It can create, start, stop, and delete them, run commands in them, and expose their URLs. All other tools are read-only. The read-only endpoint, `https://mcp.namespace.so/mcp/readonly`, leaves out the Devbox tools that make changes.
- **What runs on your machine:** the plugin bundles no executables, hooks, or local MCP server. When you ask for it, the `devboxes` skill can have your agent install the Namespace `devbox` CLI from `get.namespace.so` and run `devbox` commands. The CI skills can have it read your workflow files and run `gh` commands.
- **Revoking access:** in the [Namespace dashboard](https://cloud.namespace.so), open **Settings** > **Applications** and select **Revoke** next to the agent. Access stops immediately. Namespace handles workspace data under its [privacy policy](https://namespace.so/privacy).

## Repository layout

One plugin, `namespace`, at the repository root. Each agent reads its own manifest; all share `skills/` and the MCP server.

| Agent | Files |
|---|---|
| Agent Plugins standard (Codex, ChatGPT) | `plugin.json`, `mcp.json` |
| Claude | `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, `.mcp.json` |
| Codex repo marketplace | `.agents/plugins/marketplace.json` |
| Cursor | `.cursor-plugin/plugin.json` |
| Antigravity | `.antigravity-plugin/plugin.json`, `mcp_config.json` |
| Gemini CLI | `gemini-extension.json`, `GEMINI.md` |
| MCP Registry | `server.json` |

CI (`.github/workflows/validate.yml`) checks that the manifests agree on name, version, description, and MCP URL, that each skill is well formed, validates the manifests against the published schemas, and runs `claude plugin validate`. Before a release, also run `gemini extensions validate .`, `agy plugin validate .`, and `mcp-publisher validate server.json`. The repository root is the plugin folder, so it holds no scripts: Anthropic's directory accepts those only from Verified organizations. To release, bump `version` in every manifest and `server.json`.
