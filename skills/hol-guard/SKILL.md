---
name: hol-guard
description: Protect agent-driven Namespace cloud and devbox mutations with HOL Guard on a supported local coding-agent harness. Use before creating, expiring, exposing, reconfiguring, or otherwise changing Namespace resources from an agent session.
---

# HOL Guard for Namespace workflows

Use HOL Guard to protect the supported local coding-agent harness before the agent performs state-changing Namespace work.

HOL Guard runs at the local agent boundary. It does not run inside Namespace and does not replace Namespace authentication, workspace permissions, plan limits, access controls, resource targeting, or the safeguards in the `devboxes` skill.

## When to use

Use this skill before agent-driven Namespace operations that can create, mutate, expose, or destroy resources, including:

- creating or expiring devboxes
- changing devbox URL access or exposing/unexposing ports
- uploading files or running commands that mutate a remote workspace
- changing remote infrastructure or deployment state from a devbox
- agent workflows that turn generated shell commands into real Namespace-side effects

Keep the existing `devboxes` skill as the source of truth for Namespace CLI syntax and resource lifecycle guidance.

## Install and protect the local agent

Install HOL Guard in the environment that launches the coding agent:

```bash
pipx install hol-guard
```

Detect the supported local harness:

```bash
hol-guard status
hol-guard detect --json
```

Use the harness identifier returned by detection. Do not guess adapter names.

Bootstrap Guard, install protection for that harness, and verify the protected launch path before state-changing Namespace work:

```bash
hol-guard bootstrap
hol-guard install <detected-harness>
hol-guard run <detected-harness> --dry-run
hol-guard doctor <detected-harness> --json
```

Start the coding-agent session through HOL Guard only after those checks succeed:

```bash
hol-guard run <detected-harness>
```

Run Namespace workflows from that protected session.

## Protected Namespace workflow

1. Read the `devboxes` skill for the exact Namespace command, target, lifecycle, and access semantics.
2. Confirm the intended Namespace workspace, devbox, URL audience, checkout, and resource impact before mutation.
3. Require a healthy HOL Guard setup and launch the agent through `hol-guard run <detected-harness>`.
4. Prefer Namespace-native inspection before mutation, such as listing devboxes, checking versions, or inspecting URL access.
5. If HOL Guard denies, requests review, errors, or is unavailable, do not bypass it by launching an unprotected agent or shell for the same mutation.
6. Preserve Namespace-native prompts, access checks, plan constraints, and any user-confirmation requirements. Guard approval is additive, not a substitute.
7. Verify the resulting state with Namespace-native commands after the operation.

## High-impact actions

Apply extra care to actions such as:

```bash
devbox expire <name> --force
devbox url access <name> --mode workspace
devbox url expose <name> --port <port> --access workspace -o json
```

These examples illustrate operations with destructive or access-expanding consequences. They do not claim HOL Guard has dedicated built-in classification for every Namespace command.

Before executing them, confirm the exact target and intended effect in the Namespace workflow, then keep the action inside the protected agent session.

## Fail closed

Do not silently fall back to unprotected execution for a state-changing Namespace action.

Stop and repair protection when:

- `hol-guard detect --json` cannot identify a supported harness
- `hol-guard bootstrap` fails
- `hol-guard install <detected-harness>` fails
- the Guard dry-run fails
- `hol-guard doctor <detected-harness> --json` reports an unhealthy required protection state
- a Guard decision denies or requires review
- the protected harness cannot start or exits unexpectedly

Use Namespace's own recovery and lifecycle procedures for Namespace-side failures.

## Inspection is not enforcement

`hol-guard command test --json` is side-effect-free command inspection. It can help inspect a command, but inspection alone is not the same as running the coding agent through HOL Guard's protected harness boundary.

For state-changing Namespace work, use the protected harness workflow above.

## Boundaries

Keep these layers separate:

- **HOL Guard** protects the supported local coding-agent execution boundary.
- **Namespace** remains authoritative for authentication, workspace permissions, devbox lifecycle, URL access, plan constraints, and cloud resource behavior.
- **Workload-specific controls** remain authoritative inside workloads run on devboxes.

Do not claim that HOL Guard is embedded in Namespace or intercepts Namespace server-side traffic.
