---
name: materialize-generic-create
description: Create a materialization for any Estuary destination connector. Use this generic skill when no dedicated skill exists for the target system. Use when user says "materialize to", "send data to", "destination connector", or "set up materialization".
---

# Create Materialization (Any Connector)

Create a materialization using flowctl for any supported Estuary destination connector. This generic skill handles connectors that don't have a dedicated skill.

## Check for a Dedicated Skill First

Before using this generic workflow, check if a dedicated `materialize-<connector>-create` skill exists for the user's target destination. Dedicated skills have connector-specific guidance and troubleshooting that this generic skill can't provide. Only use this skill if no dedicated one exists.

## Step 0: Find the Connector and Load Documentation

### Find the connector

Query the connector registry to find available materialization connectors:

```bash
# List all materialization connectors
flowctl raw get --table connector_tags \
  --query 'protocol=eq.materialization' \
  --query 'select=image_tag,documentation_url' \
  --output yaml
```

To search for a specific connector by name:

```bash
flowctl raw get --table connector_tags \
  --query 'documentation_url=ilike.*<connector-name>*' \
  --query 'protocol=eq.materialization' \
  --query 'select=image_tag,documentation_url' \
  --output yaml
```

### Load the documentation

Use WebFetch to load the `documentation_url` returned from the query. This is the canonical docs page with prerequisites, config reference, and setup instructions.

**Search Kapa for tribal knowledge** (if the Estuary MCP is configured):

```
Search kapa ai knowledge sources for "materialize <connector-name> common issues"
```

If Kapa MCP is not configured, the user can set it up: https://docs.estuary.dev/features/mcp-integration/

## Step 1: Gather Requirements

Before writing any YAML, ask the user:

1. **Target system details?** — Connection info (host, port, credentials, etc.)
2. **Network path?** — Direct connection, SSH tunnel, or other networking requirements
3. **Non-default data plane?** — Most users use the default. Ask if they need a non-default data plane.
4. **Source collections?** — Which Estuary collections to materialize
5. **Any connector-specific requirements?** — Check the docs loaded in Step 0
6. **Hard deletes?** — Off by default. Without it, deleted rows stay in the destination marked with `_meta/op: 'd'`. Enable to physically remove them.
7. **Delta updates?** — Off by default. Switches from one-row-per-key (standard merge) to append-only. Use for event logs or history tables.
8. **Sync schedule?** — Controls how often batches are written to the destination (default: 30 minutes, `0s` for real-time). Affects latency and destination compute cost.

## Step 2: Find the Correct Connector Version

Use the `image_tag` from the Step 0 query. If you need to re-query for a specific connector:

```bash
flowctl raw get --table connector_tags \
  --query 'documentation_url=eq.<DOCS-URL-FROM-STEP-0>' \
  --query 'select=image_tag,documentation_url' \
  --output yaml
```

Use the returned `image_tag` — never hardcode a version.

## Step 3: Help User Complete Prerequisites

Walk the user through prerequisites from the docs loaded in Step 0. Common prerequisites across materializations:

1. **Destination accessible** — Network connectivity from Estuary cloud
2. **Credentials configured** — User/service account with write permissions
3. **Target location exists** — Database, schema, bucket, etc.

Refer to the docs page for connector-specific prerequisite details.

## Step 4: Create the Spec File

Build `flow.yaml` using the config reference from the docs. General structure:

```yaml
materializations:
  <TENANT>/<PATH>/materialize-<connector>:
    endpoint:
      connector:
        image: <IMAGE>:<VERSION>
        config:
          # Properties from the docs config reference
          # Each connector has different required fields
    bindings:
      - source: <TENANT>/<collection-path>
        resource:
          table: "<TABLE_NAME>"
          # Some connectors use different resource fields
          # (e.g., "topic" for Kafka, "index" for Elasticsearch)
```

Fill in the `config` section using the property reference from the connector's docs page.

For SSH tunnel (if supported by the connector), add `networkTunnel.sshForwarding` block — see docs.

## Step 5: Publish

```bash
flowctl catalog publish --source flow.yaml --auto-approve
```

## Step 6: Verify

```bash
# Check status
flowctl catalog status <TENANT>/<PATH>/materialize-<connector>

# View logs
flowctl logs --task <TENANT>/<PATH>/materialize-<connector> --since 5m | jq -c '{ts, message}'
```

**Status progression:**
1. `PENDING` — Normal for ~30 seconds during shard assignment
2. `BACKFILLING` — Initial data sync from collections
3. `OK` — Running normally with real-time updates

## Troubleshooting

### Materialization stuck in PENDING

Wait 30-60 seconds — this is normal during shard assignment. If still stuck:

```bash
flowctl logs --task <TENANT>/<PATH>/materialize-<connector> --since 5m | jq 'select(.level == "error")'
```

### Config validation errors

**Cause**: Config structure doesn't match the connector's schema

**Fix**: Re-read the docs page from Step 0 and verify all required properties are set with correct types and formatting.

### "collection not found"

**Cause**: Source collection doesn't exist or wrong prefix

**Fix**:
```bash
flowctl catalog list --prefix <TENANT>/ | grep collection
```
Verify the collection name matches exactly.

### "no connector tag found for image"

**Cause**: Wrong image name or tag

**Fix**: Re-query `connector_tags` (Step 0) for the correct image and version.

### Network connectivity failures

**Cause**: Destination not reachable from Estuary cloud

**Fix**:
1. Check firewall rules / security groups
2. Verify Estuary IP addresses are allowlisted (see docs)
3. Consider SSH tunnel for private networks
4. For local testing: use ngrok or similar

### Connector-specific errors

For errors specific to the connector, search Kapa:

```
Search kapa ai knowledge sources for "materialize <connector-name> <error message>"
```

## Related Skills

- `schema-field-selection` — Control which fields are materialized
- `connector-disable-enable` — Pause/restart existing materializations
- `connector-delete-recreate` — Nuclear option for stuck materializations
- `estuary-logs` — Deep log analysis
- `estuary-catalog-status` — Status checking
