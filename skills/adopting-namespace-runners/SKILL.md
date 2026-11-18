---
name: adopting-namespace-runners
description: "Moves GitHub Actions workflows onto Namespace runners and turns on Namespace caching: runner profiles or nscloud machine labels, Remote Builders for Docker, cache volumes, nscloud-cache-action, nscloud-checkout-action, and toolchain and container image caches. Use when setting up Namespace runners for a repository, converting runs-on labels, replacing actions/cache or docker/setup-buildx-action, or enabling Namespace caching in workflows."
---

# Adopt Namespace runners

Make the smallest change first, confirm the jobs run, then adopt caching one step at a time. Deliver workflow edits for review in a pull request. Do not merge, change runner profiles, or change workspace or GitHub settings unless the user asks.

The [Namespace MCP server](https://namespace.so/docs/reference/mcp) shows runner profiles and the jobs that ran on them. Without it, ask the user for their runner profiles and recent Namespace jobs from the dashboard.

## 1. Check the starting point

1. Read every workflow under `.github/workflows/` that the user wants to move, including reusable workflows and composite actions they call. List each job's `runs-on`, matrix, container, services, caching steps, and Docker build steps.
2. Check that the repository's GitHub organization is connected to Namespace: `github_job_list` with `requestedFields: ["REPOSITORY"]` returns jobs from other repositories in the same organization. A repository that has never run on Namespace has no jobs of its own. If the organization has none, the user connects it in the [dashboard](https://cloud.namespace.so/workspace/ghrunners); if it has some, the user checks that the Namespace GitHub App can access this repository. Jobs pointed at Namespace labels before that stay queued.
3. Reuse an existing profile when one fits the job's OS and architecture. Read its configuration with `github_runner_profile_get` with `requestedFields: ["CACHE_VOLUMES", "BUILDER_MODE"]` before relying on it. If no profile fits, use machine labels or ask the user to create a profile.

## 2. Choose how runners are selected

Use exactly one of these per job:

- **Runner profile** (recommended for teams): `runs-on: namespace-profile-<name>`. Shape, caching, and builder settings live in the profile, which people edit in the [dashboard](https://cloud.namespace.so/workspace/actions/profiles). Changing a profile affects every job that uses it, so treat it as shared configuration.
- **Machine labels** (configuration in code): `runs-on: nscloud-{os}-{arch}-{shape}`, such as `nscloud-ubuntu-24.04-amd64-4x8` or `nscloud-macos-tahoe-arm64-6x14`. Only one `nscloud-` machine label is allowed per job; Namespace does not schedule a job with more than one or an invalid one. Read [references/runner-labels.md](references/runner-labels.md) before writing labels.

Keep the job's OS version and architecture. Start with a shape close to the current runner and right-size after a few runs with the `optimizing-namespace-ci` skill.

Change one workflow first, on a branch, before converting the rest.

## 3. Docker builds

Namespace runners send `docker build` and `docker/build-push-action` to Remote Builders by default. In jobs that build images:

- Remove `docker/setup-buildx-action`, or replace it with `namespacelabs/nscloud-setup-buildx-action@v0`; `docker/setup-buildx-action` replaces the Remote Builder configuration.
- Remove `docker/setup-qemu-action` when it only exists for buildx.
- Remove `cache-from` and `cache-to` registry or GitHub cache settings; Remote Builders keep layer caches.
- Images can be pushed to the workspace registry through `${{ env.NSC_CONTAINER_REGISTRY }}` without extra login steps.

See [Optimizing Docker build performance](https://namespace.so/docs/solutions/github-actions/docker-builds) for in-runner builders and cache mounts.

## 4. Caching

Caches live on cache volumes attached to the runner. Turn them on in the profile (caching options), or with labels for label-based jobs. Read [references/caching.md](references/caching.md) for the exact steps and limits, then adopt in this order, measuring after each:

1. **Toolchains, container images, and action downloads:** transparent once enabled; no workflow steps. Action download caching is a profile setting only; no label enables it.
2. **Package managers and build outputs:** `namespacelabs/nscloud-cache-action@v1` with `cache:` modes, placed after checkout and after the toolchain setup step. Turn off the setup action's own GitHub cache: `cache: false` on `actions/setup-go`; on `actions/setup-node`, remove `cache` and, on v5 and later, set `package-manager-cache: false`.
3. **`actions/cache`:** replace with `nscloud-cache-action` and the same `path:` list. Keys do not carry over: every branch writes to the same volume.
4. **Git checkouts**, only for very large repositories, long history, many submodules, or Git LFS: `namespacelabs/nscloud-checkout-action@v9` in place of `actions/checkout`, with Git repository checkout caching enabled.

## 5. Validate

- Open the change as a pull request. Confirm with `github_job_list` and `github_job_describe`, requesting the `PROFILE` and `RUNNER` fields, that the jobs ran on Namespace, on the expected profile or shape, and passed.
- Expect cache misses on the first runs and on branches before the change reaches the default branch. Compare durations only after a few warm runs on the default branch.
- If a job fails only on Namespace, debug it from evidence (job steps, logs, resource usage, OOM events) before changing shapes or settings.

## 6. Report

List the converted jobs with old and new `runs-on`, the caching and Docker changes, jobs left on other runners and why, dashboard settings the user must change, and how to verify. Label speedups as expected until measured.
