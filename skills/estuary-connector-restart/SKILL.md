---
name: estuary-connector-restart
description: Pause and restart Estuary connectors by disabling and re-enabling shards via flowctl. Use when temporarily stopping a connector, performing source/destination maintenance, or forcing a fresh restart to clear transient errors or load updated configs immediately. Use when user says "pause connector", "stop connector", "restart connector", "disable connector", "enable connector", "re-enable task", "disable shards", "stop my capture", "stop my materialization", "disable my materialization", "force restart connector", "connector maintenance", "temporarily stop processing", "shards disable true", "enable shards", "resume connector", or "unpause connector".
---

# Restart a Connector — Disable and Re-enable via flowctl

Temporarily pause a capture or materialization by disabling its shards, then re-enable it. The connector resumes from its last checkpoint — no data is lost. Works for both captures and materializations.

## Prerequisites

```bash
flowctl auth login
```

## Step 1: Pull the current spec

```bash
flowctl catalog pull-specs --name <task-name>
```

This creates a `flow.yaml` and a directory with the task's YAML spec.

## Step 2: Disable the connector

Edit the task's YAML file (under `<task-name>/flow.yaml`) and add a `shards` block at the same level as `endpoint` and `bindings`:

```yaml
captures:
  acmeCo/source-postgres:
    endpoint:
      connector:
        image: ghcr.io/estuary/source-postgres:v1
        config: ...
    bindings:
      - resource: ...
        target: ...
    # Add this block to disable:
    shards:
      disable: true
    # To re-enable: remove the shards block and any expectPubId line
```

If a `shards` section already exists, just add `disable: true` to it.

Publish to apply:
```bash
flowctl catalog publish --source flow.yaml --auto-approve
```

Confirm it's disabled:
```bash
flowctl catalog status <task-name> --output json | jq '.status.type'
# Expected: "TASK_DISABLED"
```

## Step 3: Re-enable the connector

Edit the same file and:
1. Remove `disable: true` (or the entire `shards` block if it only contained that) — do **not** set it to `false` (an explicit `false` is treated as a setting and may cause unexpected behavior)
2. Remove the `expectPubId` line if present — it's stale after the disable publish and will cause errors

Publish again:
```bash
flowctl catalog publish --source flow.yaml --auto-approve
```

Verify it's running:
```bash
flowctl catalog status <task-name> --output json | jq '.status.type'
# Expected: "WARNING" initially (shards starting), then "OK" after 30-120 seconds
```

Activation timing varies: small connectors take 30-60 seconds, large connectors (100+ bindings) take 60-120 seconds.

Optionally remove the pulled spec files when done: `rm -rf flow.yaml <tenant>/`

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `expected publication ID ... was not matched` | Remove the `expectPubId` line from the spec and re-publish |
| Status stuck on `WARNING` after re-enable | Wait 30-120 seconds. If it persists, check logs with `flowctl logs --task <task-name> --since 1h` |
| `dns error: failed to lookup address` | Transient — retry the publish command |
| Status still `OK` right after disable | Wait a few seconds and re-check — status updates may take a moment |
| Can't re-enable | Ensure you removed `disable: true` entirely (not set to `false`), removed the empty `shards` block, and removed `expectPubId` |

## Tips

- Disabling is safer than deleting — all configuration and state are preserved
- The connector resumes from its checkpoint when re-enabled
- To force an immediate restart when a connector is in retry backoff (up to 15 min between retries), a disable/enable cycle bypasses the backoff entirely

## Related Skills

- **estuary-flowctl-setup** — Install and authenticate flowctl before using this skill
- **estuary-catalog-status** — Check task health before and after restarting
- **estuary-logs** — Investigate errors if the connector doesn't recover after restart
