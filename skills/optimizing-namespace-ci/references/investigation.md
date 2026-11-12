# Collect and interpret Namespace CI evidence

## Connect and select tools

Resolve the workspace connection's fully qualified MCP names and schemas once; the slugs below identify capabilities, not a promise that every deployment exposes them. Use [Namespace MCP](https://namespace.so/docs/reference/mcp) for authentication. The read-only endpoint is `https://mcp.namespace.so/mcp/readonly`; it does not restrict shell or CLI credentials. Use customer authentication, not internal databases, operator credentials, or credentials borrowed from another application.

Start with aggregates:

- `github_job_summarize`: repository/workflow/job groups ranked by execution count, not cost or critical-path latency.
- `github_job_optimization_find`: smaller-runner candidates, not target shapes or measured savings. Request `includeLowerCPUP90s` for numerical low-CPU values; follow pagination.
- `github_job_resource_usage_summarize`: peaks grouped by workload/profile/allocation, over at most 24 hours. For a longer baseline, use separate windows or a bulk report; averaging daily maxima does not produce p90.
- `github_runner_profile_list` / `github_runner_profile_get`: current configuration. Request optional `CACHE_VOLUMES`, `BUILDER_MODE`, `SWAP`, or other relevant fields. Omission does not mean disabled, and today's profile does not establish historical configuration.

Drill into representative runs:

- `github_job_list` / `github_job_describe`: identity, timestamps, workflow run, instance, and profile. Request the fields needed and follow cursors.
- `github_job_step_list`, `github_job_step_logs_fetch`, and `github_job_resource_usage_fetch`: steps, retained logs, and resource time series. Missing/truncated step logs do not prove success or absence of work. Use `gh` or a GitHub integration for PR/commit context, artifacts, and complete/live logs.
- For CPU parallelism, request `CPU` and `CPU_BREAKDOWN` together from `github_job_resource_usage_fetch` or `instance_resource_usage_fetch` if the schema supports them. `cpu_avg` averages vCPUs, `cpu_max` is the busiest vCPU, and `cpu_<n>` is an individual vCPU, all 0–100%. Per-core data is opt-in; if unavailable, leave core-level attribution unknown.
- `github_job_instance_describe` and `instance_oom_events_fetch`: actual allocation, shutdown reasons, and OOM evidence.
- `build_list_by_origin`, `build_describe`, `build_steps_get`, and `build_step_logs_fetch`: Namespace Docker builds associated with a job. Remote builders are separate machines; runner metrics do not describe their work. Build steps may lack timings or cache-hit fields; use logs/dashboard evidence and leave missing attribution unknown.

For jobs waiting now, use `workspace_concurrency_get` if available. It returns committed instance/vCPU/memory allocations and limits, not measured utilization. Pools overlap; report them separately. `fetchedAt` is retrieval time, not a historical measurement. Empty pools mean no configured pools were returned, not proof that no instances are running. This snapshot cannot explain an earlier queue delay.

## CLI fallback and bulk reports

Follow [CLI installation](https://namespace.so/docs/reference/cli/installation) and the customer login flow when needed. Check version and command help before relying on flags.

```bash
nsc github job list --repository <owner/repo> --since 168h -o json
nsc github job describe <job-id> -o json
nsc workspace concurrency --output json
```

The CLI job list defaults to at most 50 jobs; a time filter does not make it a complete baseline. Concurrency has the same overlapping-pool and point-in-time limits as MCP.

For population analysis, export a filtered report:

```bash
nsc instance report \
  --start <start-RFC3339> \
  --end <end-RFC3339> \
  --repository <owner/repo> \
  --out <report.csv>
```

Read [the report contract](https://namespace.so/docs/reference/cli/instance-report). GitHub and Buildkite CSV columns are conditional. If help advertises `--format jsonl`, prefer it with `--out <report.jsonl>`: one API entry per line, snake_case proto names, RFC 3339 timestamps, and string-encoded 64-bit integers. Nested `volumes` retains tag, mount point, size, used space, and cache-hit state for each volume. Omitted scalar fields have protobuf defaults. Older CLIs support CSV only; use a named file because their status line may mix with stdout.

Narrow range/filters if an export times out. A nonzero exit means failure; emitted JSONL may be partial and must not be treated as a complete baseline. For metered consumption beyond these reports, use the [public compute usage SDK](https://namespace.so/docs/reference/typescript-sdk/compute/usage) if available, or state the evidence gap.

For Buildkite, use documented Buildkite report filters instead of `--repository`. Group by pipeline/job/shape and preserve platform from filtered exports or instance evidence. General Buildkite discovery is not covered here.

## Interpretation limits

- Aggregates omit some platform, matrix, branch, and attempt dimensions. Recover comparable cohorts through instance descriptions or separately filtered exports using supported `--platform`, `--branch`, and workload filters. Reports omit platform in both formats, so preserve filters as dataset metadata. A few sampled runs cannot reconstruct a merged group's counts or peaks.
- Sources select different populations: job summaries by job start, job lists by queue time, and reports/downsizing queries by instance creation. Equal bounds do not guarantee equal populations.
- Report `resources_cpu_actual_max` is a ratio of one vCPU's utilization, not total-runner utilization. RAM values are ratios despite `_percent` in the name. MCP peak summaries use percentages and name CPU `maxSingleVCPUUsedPercent`. None is average CPU or p90.
- Finder `p90CPUUsedPercent` and `cpuBottleneckRisk.p90UtilizationPercent` are the maximum across candidate instances of each instance's p90 busiest-vCPU samples, not whole-runner utilization or p90 across runs. High values may mean one busy thread.
- The downsizing finder requires more than five completed linked runs, positive measurements for every linked run, and no run above 50% allocated RAM. CPU risk is separate. This does not establish complete telemetry or coverage of every workflow variant; an empty candidate list does not prove optimal sizing.
- CSV `cache_volume_hit` means any attached volume hit; JSONL `volumes[].cache_hit` identifies a volume. Correlate tag/mount point with configuration and logs before claiming a dependency, checkout, or Docker layer hit.
- For Linux AMD64/ARM64 and Windows, `max(vCPU, RAM GB / 2)` gives base shape units, not full billable units. Use current platform/shape multipliers and runtime rounding/minimums, or already-metered usage; do not multiply metered units again. Windows/high-CPU adjustments differ, and Apple Silicon uses fixed shapes. Check [Machine Shapes](https://namespace.so/docs/architecture/compute/machine-shapes) and [Billing](https://namespace.so/docs/workspaces/billing-and-limits). Dollar figures are estimates unless reconciled to billing.
