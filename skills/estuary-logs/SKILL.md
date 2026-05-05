---
name: estuary-logs
description: Search and analyze Estuary task logs with jq filtering. Use when investigating task failures, debugging connector errors, analyzing behavior patterns, or streaming live logs. Use when user says "check logs", "show me logs", "task failing", "why is my capture failing", "why is my materialization failing", "error messages", "why did it fail", "search task logs", "shard failed", "follow logs", "increase log level", "debug logging", "is this log message normal", "what does this message mean", "connection timeout", or "i/o timeout".
---

# flowctl logs - Search and analyze Estuary task logs

**Concepts**: Logs contain structured JSON with fields: `ts` (timestamp), `level`, `message`, and optional `fields` (extra context). Tasks are captures, materializations, or derivations running in Estuary.

## Prerequisites

`flowctl` must already be authenticated, as per estuary-flowctl-setup skill.

**Finding your task name**: If you don't already know the task name, use `flowctl catalog list --prefix <tenant/prefix>` to narrow it down, or copy it from the Captures or Materializations page in the Estuary web app. It looks like `org/prefix/connector-name`, for example:
```
acmeCo/ops/source-postgres
```

## Basic Usage

Quick check — pipe directly to the terminal:
```bash
flowctl logs --task <task> --since 1h | tail | jq -c
```

For investigations, save to a file first (see [Effective Log Analysis Workflow](#effective-log-analysis-workflow)).

### Live streaming (follow mode)

If a task is currently running and you want logs as they appear:
```bash
flowctl logs --task <task> --follow
```

Combine with `--since` to get recent history plus live tailing:
```bash
flowctl logs --task <task> --since 1h --follow
```

### Choosing --since Values

| Situation | Recommended | Why |
|-----------|-------------|-----|
| Recent issue (today/yesterday) | `--since 24h` or `--since 48h` | Focused on recent events |
| Issue shown in 48h dashboard graph | `--since 72h` | Graph period + buffer |
| Pattern analysis or issue from ~a week ago | `--since 7d` | Broader context |
| Finding DDL (CREATE/ALTER TABLE) | omit `--since` | DDL may be weeks/months old |

**Note on `--since` accuracy**: The actual start time snaps back to the previous log fragment boundary, so you may get somewhat more history than requested (e.g. `--since 1h` may return up to several hours). This is expected, so filter downstream to find specific time ranges

**Always use `--since`** unless you need logs since task creation (e.g. for initial DDL statements). Log history can span 1+ years for long-running tasks and will be very slow to fetch without a time bound.

## Effective Log Analysis Workflow

**Recommended approach for investigating issues:**

```bash
# 1. Save relevant logs to file (allows re-processing without re-fetching)
flowctl logs --task <task> --since 2d > task.logs

# 2. Extract useful fields, removing noise (.shard, ._meta clutter the output)
jq -c 'del(.shard, ._meta) | {ts, message} + .' task.logs

# 3. Filter out repeated irrelevant messages
jq -c 'del(.shard, ._meta) | {ts, message} + .' task.logs | grep -v "materialization progress"
```

**Why this pattern:**
- `del(.shard, ._meta)` removes verbose internal metadata that rarely helps debugging
- `{ts, message} + .` puts timestamp and message first while preserving other fields
- Saving to file lets you iterate on filters without re-fetching logs
- `grep -v` removes noisy repeated messages (e.g., periodic progress lines)

## Quick Timeline (Minimal Output)

When you just need timestamps and messages:
```bash
flowctl logs --task <task> --since 1d | jq -c '{ts, message}'
```

## Filtering for Issues

Filter for errors and warnings only:
```bash
flowctl logs --task <task> --since 1d | jq 'select(.level == "error" or .level == "warn")'
```

Get timeline with error detail:
```bash
flowctl logs --task <task> --since 1d | jq -c '{ts, message, level, error: .fields.error}'
```

## Quick Grep Patterns

```bash
# Errors and failures
flowctl logs --task <task> --since 72h | jq -c 'del(.shard, ._meta) | {ts, message} + .' | grep -iE "shard failed|error|fail|warning"

# Task restarts (each new connector run)
flowctl logs --task <task> --since 72h | jq -c 'del(.shard, ._meta) | {ts, message} + .' | grep -iE "initialized catalog task term|started connector container"

# Transaction timing (Snowflake sync schedule)
flowctl logs --task <task> --since 72h | jq -c 'del(.shard, ._meta) | {ts, message} + .' | grep -E "nextScheduledAck|delaying|finished waiting"

# Retry storms (BigQuery DML quota issues)
flowctl logs --task <task> --since 72h | jq -c 'del(.shard, ._meta) | {ts, message} + .' | grep "previously submitted job failed, will retry"
```

## Analyzing Repeated Failures

**1. Count failure frequency by day:**
```bash
flowctl logs --task <task> --since 7d | \
  jq -r 'select(.message == "shard failed") | .ts' | \
  awk -F'T' '{print $1}' | sort | uniq -c
```

**2. Get root cause from shard failures:**
```bash
flowctl logs --task <task> --since 1d | \
  jq -r 'select(.message == "shard failed") | {ts, error: .fields.error} | @json'
```

**3. Find surrounding log entries around failures:**
```bash
flowctl logs --task <task> --since 1d | \
  jq -c '{ts, message, level}' | grep -B2 "shard failed" | head -20
```

## Understanding Common Log Messages

These messages are frequently misread as problems — here's what they actually mean:

| Message | What it means | Action needed? |
|---------|---------------|----------------|
| `initialized catalog task term` + `started connector container` | Normal connector startup or restart | No — expected at start and after any spec change |
| `applying updated task specification` | Connector restarting due to a spec or schema update | No — expected after publishing changes |
| `connector exited with unconsumed input stream remainder` | Normal for batch connectors; check nearby logs if unexpected | No |
| `runTransactions: readMessage: drained` | Data reading intentionally stopped; task will restart | No — not a failure; task should recover on its own |
| `document failed validation against its collection JSON Schema` | A document didn't match the collection schema, often during schema evolution | **Wait 5–10 min first** — often auto-resolves when the task restarts and picks up the new schema; if it persists, the schema needs manual update |

## Common Error Patterns

### Materialization Transaction Failures

**S3 Chunk Fetch Timeouts:**
```
transactor.Load: querying Loads: chunk fetch: copying chunk stream:
decoding row: read tcp [IP]:[PORT]: read: connection timed out
```
- Occurs during transaction finalization when reading staged Load responses from S3
- Common with large materializations (100GB+ commits)

**Snowflake PUT Operation Timeouts:**
```
transactor.StartCommit: flush putFile: putWorker PUT to stage:
Post "https://[snowflake-host]:443/...": dial tcp: i/o timeout
```
- Occurs when uploading staged data to Snowflake
- Check for retry attempts: look for `putWorker retrying PUT to stage` messages

## Getting More Verbose Logs

If default (`info`) logs don't show enough detail, you can increase the log level to `debug`. Add a `shards` stanza to your existing task spec alongside `endpoint`, then re-publish:

```yaml
# Add this at the same level as your existing 'endpoint' block:
shards:
  logLevel: debug
```

Available levels (least → most verbose): `error` → `warn` → `info` (default) → `debug`

Remember to remove the `logLevel` override when done — `debug` produces a lot of output.

## Related Skills

- **estuary-catalog-status** — Check whether a task is running, disabled, or failed before diving into logs
- **estuary-connector-restart** — If the connector needs to be paused and restarted via flowctl
- **estuary-ssh-tunnels** — If logs show SSH tunnel failures, connection timeouts, or permission denied errors

## Tips

- `flowctl logs --help` lists all available flags
- Use `jq -c` for compact, line-by-line output (easier to grep)
- Look for `shard failed` messages — they include `.fields.error` with the root cause
