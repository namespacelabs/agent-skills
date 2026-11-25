# Collect and interpret Namespace CI evidence

Sources: [Namespace MCP](https://namespace.so/docs/reference/mcp), [nsc instance report](https://namespace.so/docs/reference/cli/instance-report), and [Machine shapes](https://namespace.so/docs/architecture/compute/machine-shapes). Tool sets differ between deployments; check the MCP tool list and `nsc <command> --help` before relying on a tool or flag.

## Aggregates

| Tool | What it returns | Limits |
|---|---|---|
| `github_job_summarize` | Completed jobs by repository, workflow, and job, ranked by execution count | Count is not cost or critical-path time. Up to the last 7 days. |
| `github_job_resource_usage_summarize` | Peak CPU and memory grouped by workload, profile, and allocation | At most 24 hours per call. CPU is `maxSingleVCPUUsedPercent`, the busiest vCPU. For longer baselines, call per day or use a report; averaging daily peaks does not give p90. |
| `github_job_optimization_find` | Jobs that can move to smaller runners because no run used more than 50% of allocated RAM | Last 30 days by default. Candidates, not target shapes or measured savings. CPU risk is reported separately; set `includeLowerCPUP90s` for low-CPU values. Follow `paginationCursor`. An empty list does not prove sizing is optimal; one run with missing metrics or high memory excludes its whole group, so for a busy repository retry with a 24-hour window and cross-check low-RAM groups from `github_job_resource_usage_summarize` before concluding there are no candidates. |
| `github_runner_profile_list`, `github_runner_profile_get` | Current profile configuration | Request `CACHE_VOLUMES`, `BUILDER_MODE`, or `SWAP` with `requestedFields`; an omitted field does not mean disabled. Today's profile may differ from the one past runs used. |

## Individual runs

- `github_job_list` and `github_job_describe`: identity, timestamps, workflow run, runner and instance, and profile. Request the fields you need and follow cursors.
- `github_job_step_list`, `github_job_step_logs_fetch`, `github_job_resource_usage_fetch`: steps, retained logs, and CPU, I/O wait, memory, and storage series. Missing or truncated step logs do not prove a step succeeded or did no work; use `gh run view <run-id> --log-failed` for complete logs.
- `cpu_avg` is the average across vCPUs and `cpu_max` the busiest vCPU, both 0 to 100. `cpu_avg` times the vCPU count approximates how many vCPUs were busy. Busiest-vCPU peaks, including `maxSingleVCPUUsedPercent`, near 100 are normal on runners with several vCPUs and do not show that more vCPUs are needed; use `cpu_avg` or the per-vCPU percentiles below.
- With `CPU_BREAKDOWN` in `metricResources`, `cpuBreakdown` gives each vCPU's p50, p95, and max over the range. A few busy vCPUs with the rest idle points to one thread that the kernel moves between cores; high p50 on every vCPU means the runner is saturated; high p95 with low p50 everywhere means a short parallel phase.
- `github_job_instance_describe` and `instance_oom_events_fetch`: allocation, shutdown reasons, and OOM events. OOM events can be empty, and per-second samples can miss a final memory spike.
- `build_list_by_origin`, `build_describe`, `build_steps_get`, `build_step_logs_fetch`: Docker builds started by a job. Runner metrics do not describe Remote Builder work.

## GitHub

No MCP tool reports job duration or billable minutes in aggregate, and `nsc` may not be installed. `gh api --paginate 'repos/<owner>/<repo>/actions/runs/<run-id>/jobs?per_page=100&filter=all'` returns every job in a run, including earlier attempts, with step timings; use it for durations and the critical path. `started_at` minus `created_at` is queue time.

## Workspace capacity

`workspace_concurrency_get` (CLI: `nsc workspace concurrency -o json`) returns current committed instances, vCPU, and memory against each concurrency pool's limit. It is a point-in-time snapshot of allocated capacity, not utilization, so it explains jobs waiting now but not an earlier queue delay. Pools overlap; report them separately and never sum them. Totals cover GitHub Actions runners, named builders, and Bazel workers without a per-workload breakdown, and exclude Devboxes. An empty list means no concurrency limits are configured.

## CLI and reports

```bash
nsc github job list --repository <owner/repo> --since 168h -o json
nsc github job describe <job-id> -o json
```

`nsc github job list` returns at most 50 jobs by default (`--max_entries`); a time filter does not make it a complete baseline.

For population analysis, export a report filtered to the workload:

```bash
nsc instance report --start <RFC3339> --end <RFC3339> \
  --repository <owner/repo> --out report.csv
```

If `nsc instance report --help` lists `-o, --output` with `json`, prefer `-o json --out report.jsonl`: one entry per line, with `github_job` or `buildkite_job` nested and per-volume `cache_hit` under `volumes`. Keys at their zero or false value are omitted, and 64-bit integers such as `job_id` are strings, so parse it with `jq` or a JSON library rather than grep. Older CLIs only write CSV.

Narrow the range or filters if it times out. A nonzero exit means the file is incomplete. Write to a file: older CLIs mix a status line into stdout.

## Interpretation limits

- Sources select different populations: job summaries by job start, job lists by queue time, and reports and the downsizing finder by instance creation. Equal bounds do not guarantee equal populations.
- Aggregates merge platform, matrix variant, branch, and attempt. Recover comparable groups with report filters such as `--platform`, `--branch`, and `--profile`, or from instance details. The report has no platform column, so record the filters you used.
- Report `resources_cpu_actual_max` is the busiest vCPU's peak as a ratio, not whole-runner utilization. `resources_ram_gb_actual_max_percent` is also a ratio despite its name.
- The finder's `p90CPUUsedPercent` and `cpuBottleneckRisk` values are the highest per-instance p90 of the busiest vCPU, not whole-runner p90 across runs. High values can mean one busy thread.
- CSV `cache_volume_hit` is true if any attached volume hit; JSON reports `cache_hit` per volume. Match volume tags and paths to configuration and logs before claiming a dependency, checkout, or Docker layer cache hit.
- For Linux and Windows, `max(vCPU, RAM GB / 2)` gives base shape units, not billable units. Apply current platform multipliers and rounding from [Machine shapes](https://namespace.so/docs/architecture/compute/machine-shapes), or use metered usage. Apple Silicon uses fixed shapes. Dollar figures are estimates unless reconciled to billing.
