---
name: estuary-task-stats
description: Check task processing statistics to see how much data and how many docs are moving. Use when checking throughput, diagnosing a stalled task, verifying data movement, or reporting on task activity. Use when user says "is data flowing", "how much data", "check throughput", "how many documents", "task not processing", "data not arriving", "stats", "bytes processed", "stalled capture", "no data in destination", "data volume", "task report", "usage report", or "how much data did we process".
---

# flowctl raw stats - Task Processing Statistics

**Concepts**: Stats show document and byte counts per transaction for a task. They answer "is data actually moving, and how much?" — separate from `flowctl catalog status`, which only shows control-plane state. A task can show `OK` status while having zero recent stats (stalled, no new source data, or sync schedule delay).

**IMPORTANT**: The command is `flowctl raw stats`, not `flowctl stats`.

## Prerequisites

`flowctl` must already be authenticated — see the **estuary-flowctl-setup** skill.

## Total Data Over a Period

**Materialization** — bytes read from source collections ("Data Read" in UI):
```bash
flowctl raw stats --task <task> --since 24h | \
  jq -s '{
    total_gb:     ([.[] | select(.materialize != null) | .materialize | to_entries[] | .value.right.bytesTotal // 0] | add // 0 | . / 1073741824 * 10 | round / 10),
    total_docs:   ([.[] | select(.materialize != null) | .materialize | to_entries[] | .value.right.docsTotal  // 0] | add),
    transactions: ([.[] | select(.materialize != null)] | length)
  }'
```

**Capture** — bytes written to Estuary collections ("Data Written" in UI):
```bash
flowctl raw stats --task <task> --since 24h | \
  jq -s '{
    total_gb:     ([.[] | select(.capture != null) | .capture | to_entries[] | .value.out.bytesTotal // 0] | add // 0 | . / 1073741824 * 10 | round / 10),
    total_docs:   ([.[] | select(.capture != null) | .capture | to_entries[] | .value.out.docsTotal  // 0] | add),
    transactions: ([.[] | select(.capture != null)] | length)
  }'
```

## Hourly Breakdown

Useful for spotting gaps, spikes, or batch patterns. Matches the hourly bar chart in the Estuary UI.

**Materialization:**
```bash
flowctl raw stats --task <task> --since 48h | \
  jq -s '[.[] | select(.materialize != null) | {
    hour:  .ts[0:13],
    bytes: ([.materialize | to_entries[] | .value.right.bytesTotal // 0] | add),
    docs:  ([.materialize | to_entries[] | .value.right.docsTotal  // 0] | add)
  }] | group_by(.hour) | map({
    hour: .[0].hour,
    gb:   (([.[].bytes] | add) / 1073741824 * 10 | round / 10),
    docs: ([.[].docs]  | add)
  })'
```

**Capture:**
```bash
flowctl raw stats --task <task> --since 48h | \
  jq -s '[.[] | select(.capture != null) | {
    hour:  .ts[0:13],
    bytes: ([.capture | to_entries[] | .value.out.bytesTotal // 0] | add),
    docs:  ([.capture | to_entries[] | .value.out.docsTotal  // 0] | add)
  }] | group_by(.hour) | map({
    hour: .[0].hour,
    gb:   (([.[].bytes] | add) / 1073741824 * 10 | round / 10),
    docs: ([.[].docs]  | add)
  })'
```

Hours with no entry = no transactions that hour (expected for gaps, quiet sources, or batch sync schedules).

## Stats Structure

Raw entries are nested by task type, then by collection/binding:

```json
{
  "ts": "2026-04-10T00:01:00Z",
  "capture":     { "<collection>": { "out":   { "docsTotal": N, "bytesTotal": N } } },
  "materialize": { "<collection>": { "right": { "docsTotal": N, "bytesTotal": N },
                                     "left":  { "docsTotal": N, "bytesTotal": N },
                                     "out":   { "docsTotal": N, "bytesTotal": N } } },
  "derivation":  { "<transform>":  { "input": { "docsTotal": N }, "out": { "docsTotal": N } } }
}
```

For materializations:
- `right` — docs/bytes read from source Estuary collections ("Data Read" in UI)
- `left` — docs/bytes read from the destination (for merging)
- `out` — docs/bytes written to the destination

The raw output also contains `interval` heartbeat entries and `ack` entries — the `select(.materialize != null)` / `select(.capture != null)` filters exclude these.

## Diagnosing a Stalled Task

If `flowctl catalog status` shows `OK` but data isn't arriving:

```bash
# Check for any recent activity
flowctl raw stats --task <task> --since 6h | jq -s '[.[] | select(.materialize != null)] | length'
# → 0 = no transactions in 6h despite OK status

# Run hourly breakdown to find where the gap starts (substitute .capture for capture tasks)
flowctl raw stats --task <task> --since 48h | \
  jq -s '[.[] | select(.materialize != null) | {hour: .ts[0:13], gb: ([.materialize | to_entries[] | .value.right.bytesTotal // 0] | add // 0 | . / 1073741824 * 10 | round / 10)}] | group_by(.hour) | map({hour: .[0].hour, gb: ([.[].gb] | add)})'
```

**Common causes of stats silence with OK status:**
- Source has no new data (expected for quiet sources)
- Sync schedule delay — materializations batch on a schedule (default: 30 min)
- Connector in retry backoff — check with `flowctl logs --task <task> --since 2h`

## Related Skills

- **estuary-task-health** — Single-task health check combining status, stats, logs, and history
- **estuary-catalog-status** — Check control-plane state (running/failed/disabled)
- **estuary-logs** — Investigate errors if stats show zero activity
