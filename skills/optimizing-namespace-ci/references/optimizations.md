# Match the change to the bottleneck

Read the applicable guide before choosing exact action inputs, runner syntax, or settings. Preserve workflow semantics and pinned-action/security policies rather than copying example versions blindly.

## Runner sizing or CPU parallelism

Use [Optimizing Usage](https://namespace.so/docs/solutions/github-actions/optimizing-usage) and [Machine Shapes](https://namespace.so/docs/architecture/compute/machine-shapes). Compare CPU behavior, memory peaks, OOMs, and runtime across comparable runs. Choose a supported shape with headroom, and inspect worker-count settings. Consider a smaller shape for spend or a larger shape for demonstrated parallel CPU/memory pressure; do not reduce coverage to fit a shape.

## Checkout, dependency installs, or cache transfers

Use [Caching](https://namespace.so/docs/solutions/github-actions/caching), [Checkout](https://namespace.so/docs/reference/github-actions/nscloud-checkout-action), [Cache action](https://namespace.so/docs/reference/github-actions/nscloud-cache-action), and [Build System Integrations](https://namespace.so/docs/solutions/github-actions/caching/build-systems) as applicable.

Verify profile caching and the slow step first. Consider cached checkout, framework presets, or the build system's remote cache. Remove redundant upload/download caching only after an equivalent Namespace cache works. Preserve checkout history, submodules/LFS, paths, and authentication. Cache Volumes are not a drop-in keyed partitioning contract for `actions/cache`; inspect key-dependent conditions and outputs. Keep artifacts needed by downstream jobs. Account for warm-up and write restrictions; do not broaden sharing or delete caches automatically.

## Repeated toolchain or OS-package setup

Use [Custom Base Images](https://namespace.so/docs/solutions/github-actions/custom-base-images). Inspect the standard image first. Move stable repeated setup into profile APT packages or a Dockerfile when worthwhile; keep repository-dependent setup in the workflow.

The runner Dockerfile's final stage must derive from `NAMESPACE_BASE_IMAGE_REF` and end as user `runner`. Never bake secrets. Include image build/distribution time and maintenance, and wait for readiness before measuring. A job's `container:` image is not the runner's pre-distributed base image.

## Docker builds

Use [Optimizing Docker Build Performance](https://namespace.so/docs/solutions/github-actions/docker-builds). Check profile builder mode and the builder actually used. In remote-builder workflows, `docker/setup-buildx-action` can replace Namespace's configured builder. Remove/replace it or remove QEMU only when the workflow uses Namespace's native remote path. Preserve explicit builder, registry, platform, and output requirements. Remove `cache-from`/`cache-to` only after confirming equivalent caching. For large transfer/export costs, assess the guide's locally cached builder rather than resizing the runner blindly.

## Queue delays, failures, or container access

Use [Debugging](https://namespace.so/docs/solutions/github-actions/debugging), [Queue Time](https://namespace.so/docs/solutions/github-actions/debugging/queue-time), and [Runner Controls](https://namespace.so/docs/solutions/github-actions/runner-controls).

Check dependencies, GitHub concurrency groups, profile limits, workspace capacity, and provisioning before blaming runner capacity. A limit increase is a proposal, not an automatic remedy. Inspect concrete OOM/crash events and logs for failures. Breakpoints hold billed runners open and require approval with constrained duration/access.

Swap can prevent OOM but slow execution. Privileged mode, broader API/network access, and priority changes are not generic performance fixes. Profile features use semicolon settings on the profile name, not a separate `namespace-features:` label. For container jobs, read [Running jobs in containers](https://namespace.so/docs/solutions/github-actions/runner-controls/containerized-jobs). A cache goal does not authorize `SYS_ADMIN`, runtime sockets, credential mounts, or other expanded access.

## Test/shard imbalance or unnecessary work

Balance using measured test/shard durations, not source lines or file counts; inspect the slowest shard and critical path. If timings are absent, propose collecting them. Preserve required checks and coverage when changing shards, dependencies, or path filters; more shards may improve latency while increasing compute.

Test timings and flaky statuses alone do not establish which tests are safe to skip. Impact-based selection needs revision-scoped test-to-code/dependency evidence and a conservative fallback to full coverage when that mapping is incomplete.
