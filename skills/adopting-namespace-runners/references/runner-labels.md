# Runner labels

Source: [Runner Labels](https://namespace.so/docs/reference/github-actions/runner-configuration), which lists every supported OS version and shape.

## Machine label

`nscloud-{os}-{arch}-{shape}`. Omitting `{arch}` or `{shape}` selects a default, so spell out all three to keep the configuration explicit.

| OS | `{os}` values | `{arch}` | Standard `{shape}` (vCPU x GB) |
|---|---|---|---|
| Ubuntu | `ubuntu-22.04`, `ubuntu-24.04`, `ubuntu-26.04` | `amd64`, `arm64` | `2x4`, `4x8`, `8x16`, `16x32`, `32x64` |
| Windows | `windows-2022` | `amd64` | `2x4`, `4x8`, `8x16`, `16x32`, `32x64` |
| macOS | `macos-sequoia`, `macos-tahoe`, and others listed in the docs | `arm64` | `6x14`, `12x28`, `12x56` |

Larger and odd-sized shapes exist; see [Machine Shapes](https://namespace.so/docs/architecture/compute/machine-shapes).

Rules:

- One `nscloud-` machine label per job. Two, or an unrecognized one, means the job is never scheduled.
- `runs-on` can be a list. Companion labels (cache, features, builders) are separate entries.
- With a matrix, build the label from matrix values, for example `runs-on: nscloud-ubuntu-24.04-${{ matrix.arch }}-8x16`.

## Suffixes

Append to the machine label to enable capabilities without changing the shape. Order does not matter.

| Suffix | Effect |
|---|---|
| `-with-cache` | Attach a cache volume, configured by the `nscloud-cache-*` labels. |
| `-with-features` | Apply settings from a `namespace-features:` label (see [Runner Controls](https://namespace.so/docs/solutions/github-actions/runner-controls)). |
| `-with-builders` | Apply Remote Builder labels such as `nscloud-no-remote-builders` or `nscloud-in-runner-builder`. |

Example: a runner with a cache volume and privileged containers.

```yaml
runs-on:
  - nscloud-ubuntu-24.04-amd64-8x16-with-cache-with-features
  - nscloud-cache-tag-my-repo
  - namespace-features:container.privileged=true
```

## Converting GitHub-hosted labels

Map each job to the same OS version and architecture, then pick a starting shape near what the job had. Confirm the current runner's size in GitHub's documentation for the user's plan rather than assuming it.

| GitHub-hosted | Namespace starting point |
|---|---|
| `ubuntu-26.04` | `nscloud-ubuntu-26.04-amd64-4x8` |
| `ubuntu-24.04` | `nscloud-ubuntu-24.04-amd64-4x8` |
| `ubuntu-22.04` | `nscloud-ubuntu-22.04-amd64-4x8` |
| `ubuntu-26.04-arm`, `ubuntu-24.04-arm` | `nscloud-ubuntu-<version>-arm64-4x8` |
| `windows-2022` | `nscloud-windows-2022-amd64-4x8` |
| `macos-*` on Apple silicon | `nscloud-macos-<matching version>-arm64-6x14` |

`ubuntu-latest` and `windows-latest` move as GitHub updates them; map them to the version GitHub currently resolves, from the job's "Set up job" log. Namespace has no Windows Server 2025 or Intel macOS runners: map `windows-2025` to `windows-2022` and tell the user, and leave Intel macOS jobs where they are.

Images differ from GitHub's: preinstalled tool versions may not match. When a step relied on a preinstalled tool, add an explicit setup step instead of assuming it is present.
