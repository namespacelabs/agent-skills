# Namespace agent skills

Skills for interacting with [Namespace](https://cloud.namespace.so) using agents like Claude, Codex or Cursor - provision devboxes, run workloads, and manage cloud resources with agents.

## Install

```sh
npx skills add namespacelabs/agent-skills
```

Add a specific skill with `--skill`:

```sh
npx skills add namespacelabs/agent-skills --skill devboxes
```

## Skill catalog

| Skill | Use case |
|---|---|
| `devboxes` | Spin up devboxes to run commands or workloads (incl. test suites - see `references/devboxes-run-tests.md`) |
| `hol-guard` | Protect state-changing Namespace workflows from a supported local coding-agent harness with HOL Guard |
