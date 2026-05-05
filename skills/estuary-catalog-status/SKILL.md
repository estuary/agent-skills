---
name: estuary-catalog-status
description: Run flowctl catalog status to check the control-plane state of an Estuary resource. Use when checking whether a task is running, disabled, or failing, or interpreting status output. Use when user says "check task status", "is my capture running", "catalog status", "flowctl catalog status", "what's wrong with my capture", "task failing", "check catalog status", "collection status", "derivation status", or "why is my task disabled".
---

# flowctl catalog status - Check status of Estuary resources

**Concepts**: The catalog contains all Estuary specs — captures, materializations, collections, derivations, and tests. The `status` command shows the current control-plane state including whether the task is running, its last failure, publication history, and shard health.

**Note**: This command shows *control-plane* status — whether the controller has activated the task, published specs, and received shard health reports. It does **not** directly poll data-plane (runtime) state in real time, so brief transient failures or throughput issues may not be reflected here. For real-time runtime behavior, use `flowctl logs`.

## Prerequisites

```bash
flowctl auth login
```

## Basic Usage

```bash
flowctl catalog status <task> --output json | jq
```

Check a task and everything connected to it (upstream/downstream collections and tasks):
```bash
flowctl catalog status <task> --connected --output json | jq
```

## Interpreting Status

### Status Types

| `status.type` | `status.summary` | What it means |
|---------------|-------------------|---------------|
| `OK` | `Running` | Task is healthy and processing data |
| `OK` | `Ok` | Collection is healthy (collections show `Ok` not `Running`) |
| `WARNING` | varies | Task is running but has recent failures or is retrying |
| `TASK_DISABLED` | `Task shards are disabled` | Task has been intentionally paused |
| `ERROR` | error message | Task has failed repeatedly and is not recovering |

### Key Output Fields

```bash
# Quick health check — just the essentials
flowctl catalog status <task> -o json | jq '{
  status:             .status.type,
  summary:            .status.summary,
  recent_failures:    .status.controller.activation.recentFailureCount,
  last_failure_error: (.status.controller.activation.lastFailure.fields.error // .status.controller.activation.lastFailure.message // null)
}'
```

**Note**: Use `.fields.error` rather than the full `.lastFailure` object — for schema validation failures, the full object includes the entire failing document and produces walls of JSON.

| Field | What it tells you |
|-------|-------------------|
| `.status.type` | Overall health: OK, WARNING, TASK_DISABLED, ERROR |
| `.status.summary` | Human-readable status message |
| `.status.controller.failures` | Cumulative controller failure count |
| `.status.controller.activation.shardStatus.status` | Shard-level health (OK or degraded) |
| `.status.controller.activation.shardStatus.count` | Number of status reports from shards |
| `.status.controller.activation.recentFailureCount` | Recent shard failures (resets on recovery) |
| `.status.controller.activation.lastFailure` | Last failure details: timestamp, message, and error |
| `.status.controller.publications.history` | Recent publication events with results |
| `.status.controller.nextRun` | When the controller will next check the task |
| `.liveSpecUpdatedAt` | When the spec was last published |
| `.userCapability` | Your permission level (admin, write, read) |

## Common jq Recipes

```bash
# Get the last failure message and timestamp
flowctl catalog status <task> -o json | jq '{
  ts: .status.controller.activation.lastFailure.ts,
  message: .status.controller.activation.lastFailure.message,
  error: .status.controller.activation.lastFailure.fields.error
}'

# Check when the spec was last published
flowctl catalog status <task> -o json | jq '.liveSpecUpdatedAt'

# Get publication history
flowctl catalog status <task> -o json | jq '.status.controller.publications.history[] | {id, created, detail, result: .result.type}'

# Check all connected resources at a glance
flowctl catalog status <task> --connected -o json | jq -c '{name: .catalogName, type: .catalogType, status: .status.type, summary: .status.summary}'
```

## Troubleshooting by Status

### Task shows WARNING
The task is running but has recent issues. Check:
```bash
# What failed recently?
flowctl catalog status <task> -o json | jq '.status.controller.activation.lastFailure'
```
If failures are intermittent and `recentFailureCount` is low, the task is likely self-recovering. If failures are climbing, check logs with `flowctl logs --task <task> --since 1d`.

### Task shows TASK_DISABLED
The task was intentionally paused (shards disabled). Re-enable it in the Estuary web app (edit the task and click Save and Publish), or use the **estuary-connector-restart** skill to re-enable via flowctl.

### Task shows ERROR
The task has failed and is not recovering. Steps:
1. Get the error: `flowctl catalog status <task> -o json | jq '.status.controller.activation.lastFailure.fields.error'` — this may be **truncated** in the status output. If the message ends with `[truncated]`, use `flowctl logs` (step 2) for the full error.
2. Check logs: `flowctl logs --task <task> --since 1d`
3. If the error is a configuration issue, fix and re-publish
4. If recovery logs are corrupted, the task may need to be deleted and recreated — this can cause data loss. Recommend the user contact Estuary support before proceeding (Slack: https://go.estuary.dev/slack or email: support@estuary.dev) and include the task name, the error message, and what they've tried so far.

### Status not updating after publish
The controller runs on a schedule. Check when the next run is:
```bash
flowctl catalog status <task> -o json | jq '.status.controller.nextRun'
```
After a publish, the controller typically picks up changes within a few minutes.

## Related Skills

- **estuary-task-health** — Single-task health check combining status, data flow, logs, and history
- **estuary-task-stats** — Check whether data is actually flowing through the task
- **estuary-catalog-history** — View publication history and recent spec changes
- **estuary-logs** — Search and filter task logs for detailed runtime information
- **estuary-connector-restart** — If the task needs to be paused and restarted via flowctl
- **estuary-ssh-tunnels** — If the failure involves SSH tunnel or network connectivity issues

## Tips

- Always use `-o json | jq` for structured output — table mode omits many fields
- `--connected` is useful for checking an entire pipeline's health at once
- `recentFailureCount > 0` with `status.type: OK` means the task recovered on its own
- `.status.controller.activation.lastFailure` persists even after recovery — it shows the most recent failure, not necessarily a current one
