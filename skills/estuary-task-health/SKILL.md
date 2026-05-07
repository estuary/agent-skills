---
name: estuary-task-health
description: Run a comprehensive health check on a single Estuary task by combining catalog status, data flow stats, recent errors, and publication history. Use when diagnosing a task that seems broken, slow, or stuck. Use when user says "is my task healthy", "something seems wrong", "check my capture", "check my materialization", "why is data not flowing", "task health check", "diagnose my task", or "what's wrong with my pipeline".
---

# estuary-task-health - Single Task Health Check

Run four checks against a single task and synthesize a verdict. Covers control-plane state, data flow, recent errors, and recent spec changes.

**Applies to**: captures, materializations, derivations

## Step 0: Get the Task Name

If the user hasn't provided a full task name, find it:
```bash
flowctl catalog list --prefix <tenant>/ | grep -E "capture|materialize|derivation"
```

## Step 1: Control-Plane Status

Is the task running at all?

```bash
flowctl catalog status <task> -o json | jq '{
  status:             .status.type,
  summary:            .status.summary,
  recent_failures:    .status.controller.activation.recentFailureCount,
  last_failure_error: (.status.controller.activation.lastFailure.fields.error // .status.controller.activation.lastFailure.message // null)
}'
```

**Note**: `last_failure_error` shows the raw error string, which is much more readable than the full `lastFailure` object. For schema validation failures the full object includes the entire failing document — use `.fields.error` to avoid walls of JSON.

| Result | Meaning |
|--------|---------|
| `status: "OK"` | Controller considers task healthy — continue to Step 2 |
| `status: "WARNING"` | Running but has recent failures — check last_failure_error and Step 3 |
| `status: "TASK_DISABLED"` | Intentionally paused — use **estuary-connector-restart** to re-enable |
| `status: "ERROR"` | Not recovering — check last_failure_error, then Step 3 for full error |

For full status field reference, see the **estuary-catalog-status** skill.

## Step 2: Data Flow

Is data actually moving? (A task can show OK while stalled.)

**For materializations:**
```bash
flowctl raw stats --task <task> --since 24h | \
  jq -s '{
    total_gb:     ([.[] | select(.materialize != null) | .materialize | to_entries[] | .value.right.bytesTotal // 0] | add // 0 | . / 1073741824 * 10 | round / 10),
    total_docs:   ([.[] | select(.materialize != null) | .materialize | to_entries[] | .value.right.docsTotal  // 0] | add // 0),
    transactions: ([.[] | select(.materialize != null)] | length)
  }'
```

**For captures:**
```bash
flowctl raw stats --task <task> --since 24h | \
  jq -s '{
    total_gb:     ([.[] | select(.capture != null) | .capture | to_entries[] | .value.out.bytesTotal // 0] | add // 0 | . / 1073741824 * 10 | round / 10),
    total_docs:   ([.[] | select(.capture != null) | .capture | to_entries[] | .value.out.docsTotal  // 0] | add // 0),
    transactions: ([.[] | select(.capture != null)] | length)
  }'
```

`transactions: 0` with `OK` status = task is running but no data is moving. Common causes: quiet source, sync schedule delay (default 30 min for materializations), connector in retry backoff.

**Note**: Push-mode connectors (Kafka, webhooks) produce one transaction per message — high transaction counts (100k+/day) are normal and expected.

For hourly breakdown (spot gaps and batch patterns), see the **estuary-task-stats** skill.

## Step 3: Recent Errors and Warnings

Any error patterns in the last 24h?

```bash
flowctl logs --task <task> --since 24h | jq -c 'select(.level == "error" or .level == "warn") | {ts, level, message, error: .fields.error}'
```

Look for repeated errors — a single error followed by recovery is normal. Errors repeating every few minutes indicate the task is stuck. Warnings without errors often signal transient issues.

**If errors are found**, search Kapa for tribal knowledge on the specific error message (if the Estuary MCP is configured):
Search kapa ai knowledge sources for `"<error message>"`

Not configured? Set it up: https://docs.estuary.dev/features/mcp-integration/

Also check the connector's documentation URL for known issues — find it with:
```bash
flowctl raw get --table connector_tags --query 'documentation_url=ilike.*<connector-name>*' --query 'select=documentation_url' --output yaml
```

**Benign warnings to ignore:**
- `"could not fetch queued overload time"` — Snowflake internal metric, harmless
- `"Collection not available"` / `"Collection binding exists but has no journals available"` — dekaf/push connector startup, harmless

For full log analysis patterns, see the **estuary-logs** skill.

## Step 4: Recent Spec Changes

Did something change recently that might explain the issue?

```bash
flowctl catalog history --name <task> -o json | \
  jq -s '[.[:5][] | {at: .publication.publishedAt, who: .publication.userEmail, detail: .publication.detail}]'
```

Look for recent manual publishes, auto-discover events, or dependency changes that correlate with when the issue started.

For full publication history analysis, see the **estuary-catalog-history** skill.

## Synthesis

After running all four checks, summarize:

| Check | Result | What it means |
|-------|--------|---------------|
| Status | OK / WARNING / ERROR / TASK_DISABLED | Control-plane view |
| Stats | N GB, N docs in 24h | Whether data is moving |
| Errors | N errors, pattern | What's breaking |
| History | Last change at X by Y | Whether a recent change caused it |

**Verdict:**
- **Healthy** — OK status, data flowing, no errors, no recent changes
- **Stalled** — OK status, zero stats, no errors → sync schedule delay or quiet source
- **Degraded** — WARNING status, intermittent errors, recovering on its own
- **Failing** — ERROR status or repeated errors not recovering → needs intervention
- **Misconfigured** — errors correlate with a recent spec change

**Common error patterns and fixes:**

| Error signature | Cause | Fix |
|----------------|-------|-----|
| `document failed validation ... Type mismatch` | Source field changed type (e.g. string → null) | Update collection schema or use field selection |
| `document failed validation ... Array has too many items` | Array grew past inferred `maxItems` | Update inferred schema to allow larger arrays |
| `additionalProperties ... not allowed` | New field appeared in source data | Update capture schema to include the new field |
| `replication slot "flow_slot" does not exist` | Slot was dropped on the database | Recreate the replication slot on the source DB |
| `replication slot ... is active for PID` | Another capture is holding the slot | Find and stop the duplicate capture process |
| `replication became idle (in read-only mode)` | DB is a read replica or failed over | Check if source DB was promoted or failed over |
| `NoSuchUpload` (S3) | Stale multipart upload ID | Restart the connector to clear the stale upload |
| `panic: ... unhandled` | Connector bug | Contact Estuary support — this is not user error |

**Next steps by verdict:**
- Stalled: check sync schedule, verify source has new data, check logs for retry backoff
- Degraded: monitor `recentFailureCount`; if climbing, treat as Failing
- Failing: search Kapa for the error, fix config and re-publish, or use **estuary-connector-restart** to reset; contact support if logs show a connector panic
- Misconfigured: use **estuary-catalog-history** to diff the spec change; re-publish with fix

## Optional: Check Connected Tasks

After completing the health check, offer to check the rest of the pipeline. A single capture may feed many materializations, and a single materialization may read from many captures.

**If the checked task is a materialization**, `--connected` finds upstream captures directly:
```bash
flowctl catalog status <materialization> --connected -o json | \
  jq -c 'select(.catalogType == "capture" or .catalogType == "materialization") | {name: .catalogName, type: .catalogType, status: .status.type}'
```

**If the checked task is a capture**, use a two-hop approach via a collection — `--connected` on a collection finds both directions:
```bash
# Step 1: get any collection produced by the capture
COLLECTION=$(flowctl catalog status <capture> --connected -o json 2>/dev/null | \
  jq -r 'select(.catalogType == "collection") | .catalogName' | head -1)

# Step 2: --connected on the collection shows both the capture and downstream materializations
flowctl catalog status "$COLLECTION" --connected -o json | \
  jq -c 'select(.catalogType == "capture" or .catalogType == "materialization") | {name: .catalogName, type: .catalogType, status: .status.type}'
```

**Important**: `head -1` only checks one collection. Different collections from the same capture can feed different materializations. Show the user what was found and ask:
1. Do these look like the right connected tasks, or should all collections be checked for additional materializations?
2. Which connected tasks would they like to health-check?

To check all collections for completeness (may be slow for large captures):
```bash
flowctl catalog status <capture> --connected -o json 2>/dev/null | \
  jq -r 'select(.catalogType == "collection") | .catalogName' | \
  while read col; do
    flowctl catalog status "$col" --connected -o json 2>/dev/null | \
      jq -c 'select(.catalogType == "materialization") | .catalogName'
  done | sort -u
```

Ask the user if they'd like to run the same health check on any of the connected tasks.

## Related Skills

- **estuary-catalog-status** — Deep dive into control-plane status fields
- **estuary-task-stats** — Hourly breakdown and per-collection stats
- **estuary-logs** — Full log analysis with filtering patterns
- **estuary-catalog-history** — Publication timeline and spec diffs
- **estuary-connector-restart** — Pause and restart a task via flowctl
