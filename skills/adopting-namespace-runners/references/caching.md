# Caching on Namespace runners

Read this before changing caching in [../SKILL.md](../SKILL.md). Sources: [Caching in GitHub Actions](https://namespace.so/docs/solutions/github-actions/caching), [Cache Volumes](https://namespace.so/docs/architecture/storage/cache-volumes), and [Runner Labels](https://namespace.so/docs/reference/github-actions/runner-configuration).

## How cache volumes behave

- A job gets the most recently used volume with its cache tag, mounted at `/cache`. Jobs with the same tag share one volume, across repositories too.
- There are no cache keys. Main and branch runs write to the same volume unless writes are restricted (below).
- The volume is 20 GB by default; the maximum depends on the plan. Size it so the working set fits, or hit rates drop.
- Containerized jobs (`container:`) must mount the volume into the container; see [Running jobs in containers](https://namespace.so/docs/solutions/github-actions/runner-controls/containerized-jobs).

## Profile-based jobs

Enable caching options on the runner profile in the [dashboard](https://cloud.namespace.so/workspace/actions/profiles): container images, toolchain downloads, action downloads, and Git repository checkouts. Changing a profile affects every job that uses it; the user makes that change.

## Label-based jobs

| Cache | Labels |
|---|---|
| Cache volume | `-with-cache` suffix on the machine label, `nscloud-cache-tag-<tag>` (`[a-z0-9.-]+`), optional `nscloud-cache-size-<n>gb` |
| Git mirrors | `nscloud-git-mirror-<n>gb`, then use `nscloud-checkout-action` |
| Toolchains (`actions/setup-*` downloads) | `nscloud-runner-tool-cache-<n>gb` |
| Container image pulls | `nscloud-container-image-cache` together with a cache volume |

## Workflow steps

Checkout:

```yaml
- uses: namespacelabs/nscloud-checkout-action@v8
  with:
    path: my-repo
```

Package managers and toolchains, after checkout and after the toolchain is installed:

```yaml
- uses: actions/setup-go@v6
  with:
    cache: false
- uses: namespacelabs/nscloud-cache-action@v1
  with:
    cache: go
```

Supported `cache:` modes include `apt`, `brew`, `bun`, `cocoapods`, `composer`, `deno`, `go`, `golangci-lint`, `gradle`, `kotlin-native`, `maven`, `mise`, `nix`, `npm`, `playwright`, `pnpm`, `poetry`, `python`, `ruby`, `rust`, `swiftpm`, `uv`, and `xcode`. Check the [action reference](https://namespace.so/docs/reference/github-actions/nscloud-cache-action) for each mode's prerequisites.

Arbitrary paths, replacing `actions/cache`:

```yaml
- uses: namespacelabs/nscloud-cache-action@v1
  with:
    path: |
      .next/cache
```

Running the cache action before checkout loses the cache, because checkout wipes the workspace.

## Protect caches

Let only trusted branches write, so pull requests from bots, agents, and forks cannot change what main reads:

```yaml
runs-on:
  - nscloud-ubuntu-24.04-amd64-4x8-with-cache
  - nscloud-cache-tag-npm
  - nscloud-cache-allow-commit-from-main
```

Profiles have an equivalent setting. Recommend protection whenever a cache is shared between the default branch and untrusted branches.

## Check that caching works

A volume hit is not a framework hit. Confirm in step logs that the tool found its cache (for example fewer downloads), and compare warm runs on the default branch, not the first runs after the change.
