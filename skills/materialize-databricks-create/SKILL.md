---
name: materialize-databricks-create
description: Create a Databricks materialization to stream Estuary collections into Databricks tables via Unity Catalog. Use when setting up Databricks as a destination for captured data. Use when user says "send to Databricks", "materialize to Databricks", "Databricks destination", or "load into Databricks".
---

# Create Databricks Materialization

Create a Databricks materialization using flowctl to stream data from Estuary collections into Databricks Unity Catalog tables.

**Applies to**: materialize-databricks

## Step 0: Load Connector Documentation

Before proceeding, fetch the official connector docs for prerequisites, config reference, and setup instructions.

**Load the docs page:**
https://docs.estuary.dev/reference/Connectors/materialization-connectors/databricks/

Use WebFetch to load this page. It covers:

- Prerequisites (Unity Catalog, SQL Warehouse, credentials)
- Full config property reference
- Authentication setup (PAT or OAuth2 M2M)
- Staging via Unity Catalog Volumes

**Search Kapa for tribal knowledge** (if the Estuary MCP is configured):

```
Search kapa ai knowledge sources for "materialize databricks common issues"
```

If Kapa MCP is not configured, the user can set it up: https://docs.estuary.dev/features/mcp-integration/

This skill provides the **flowctl workflow** and **troubleshooting** that docs don't cover.

## Step 1: Gather Requirements

Before writing any YAML, ask the user:

1. **Workspace URL?** — Databricks workspace hostname (e.g., `dbc-abc123.cloud.databricks.com`)
2. **SQL Warehouse HTTP path?** — Found in SQL Warehouse > Connection details
3. **Catalog and schema?** — Unity Catalog name and target schema
4. **Authentication method?** — Personal Access Token (PAT) or OAuth2 Machine-to-Machine (M2M)
5. **Non-default data plane?** — Most users use the default. Ask if they need a non-default data plane.
6. **Source collections?** — Which Estuary collections to materialize
7. **Hard deletes?** — Off by default. Without it, deleted rows stay in Databricks marked with `_meta/op: 'd'`. Enable to physically remove them.
8. **Delta updates?** — Off by default. Switches from one-row-per-key (standard merge) to append-only. Use for event logs or history tables.
9. **Sync schedule?** — Controls how often batches are written to Databricks (default: 30 minutes, `0s` for real-time). Affects latency and compute cost.

## Step 2: Find the Correct Connector Version

Always use the latest numbered version tag. Query the connector registry:

```bash
flowctl raw get --table connector_tags \
  --query 'documentation_url=eq.https://go.estuary.dev/materialize-databricks' \
  --query 'select=image_tag,documentation_url' \
  --output yaml
```

Use the returned `image_tag` — never hardcode a version.

## Step 3: Help User Complete Prerequisites

Walk the user through prerequisites from the docs loaded in Step 0:

1. **Unity Catalog enabled** — Required for staging data via Volumes
2. **SQL Warehouse running** — Must be active during publish and ongoing operation
3. **Schema exists** — Create the target schema if it doesn't exist
4. **Credentials ready** — PAT token or OAuth2 M2M client ID/secret

Staging: The connector automatically creates a Unity Catalog Volume (`flow_staging`) for staging — no user setup needed for storage.

Refer to the docs page for exact setup steps.

## Step 4: Create the Spec File

Build `flow.yaml` using the config reference from the docs.

**With Personal Access Token (PAT):**

```yaml
materializations:
  <TENANT>/<PATH>/materialize-databricks:
    endpoint:
      connector:
        image: ghcr.io/estuary/materialize-databricks:<VERSION>
        config:
          address: "<WORKSPACE>.cloud.databricks.com"
          http_path: "/sql/1.0/warehouses/<WAREHOUSE-ID>"
          catalog_name: "<CATALOG>"
          schema_name: "<SCHEMA>"
          credentials:
            auth_type: "PAT"
            personal_access_token: "<TOKEN>"
    bindings:
      - source: <TENANT>/<collection-path>
        resource:
          table: "<TABLE_NAME>"
```

**With OAuth2 Machine-to-Machine (M2M):**

```yaml
materializations:
  <TENANT>/<PATH>/materialize-databricks:
    endpoint:
      connector:
        image: ghcr.io/estuary/materialize-databricks:<VERSION>
        config:
          address: "<WORKSPACE>.cloud.databricks.com"
          http_path: "/sql/1.0/warehouses/<WAREHOUSE-ID>"
          catalog_name: "<CATALOG>"
          schema_name: "<SCHEMA>"
          credentials:
            auth_type: "OAuth2 M2M"
            client_id: "<CLIENT-ID>"
            client_secret: "<CLIENT-SECRET>"
    bindings:
      - source: <TENANT>/<collection-path>
        resource:
          table: "<TABLE_NAME>"
```

**Important**: The connector is single-threaded — one failing binding blocks all bindings in the materialization. Group critical tables together and isolate experimental ones.

## Step 5: Publish

```bash
flowctl catalog publish --source flow.yaml --auto-approve
```

## Step 6: Verify

```bash
# Check status
flowctl catalog status <TENANT>/<PATH>/materialize-databricks

# View logs
flowctl logs --task <TENANT>/<PATH>/materialize-databricks --since 5m | jq -c '{ts, message}'
```

**Status progression:**
1. `PENDING` — Normal for ~30 seconds during shard assignment
2. `BACKFILLING` — Initial data sync from collections
3. `OK` — Running normally with real-time updates

## Troubleshooting

### "Field selection validation failed"

**Cause**: Usually a UI state issue (stale cache) or schema evolution conflict (required field no longer exists in collection)

**Fix**:
1. Try a browser cache clear and reload
2. Click the "Refresh" button in the Field Selection section
3. Check if any required fields were pruned from the collection's inferred schema (complexity limit of 1000 fields)
4. If a required field disappeared, either un-require it or re-add it to the collection schema

### DELTA_UNSUPPORTED_DROP_COLUMN

**Cause**: Connector tried to drop a column but the table's column mapping mode doesn't support it

**Fix**: Enable column mapping mode on the affected table in Databricks:
```sql
ALTER TABLE <catalog>.<schema>.<table> SET TBLPROPERTIES ('delta.columnMapping.mode' = 'name');
```

### "schema not found" or "catalog not found"

**Cause**: `catalog_name` or `schema_name` is misspelled, or credentials lack access

**Fix**: Verify the exact catalog and schema names in Databricks. Check that the service principal or PAT user has access to both.

### 64 MB gRPC document size limit

**Cause**: Very large documents exceeding the gRPC message size limit

**Fix**: Use field selection to exclude large fields, or create a derivation to truncate oversized documents before materializing.

### "Merge query too large"

**Cause**: Too many bindings or too many fields in a single materialization

**Fix**: Split into multiple materializations with fewer bindings, or use field selection to reduce the number of fields per binding.

### PAT re-entry needed after backfill

**Cause**: Known issue where PAT may need to be re-entered after triggering a backfill

**Fix**: Re-enter the same PAT in the connector configuration. This is a known UI quirk, not a credential expiration.

### SQL Warehouse auto-stopped

**Cause**: SQL Warehouse stopped due to inactivity timeout

**Fix**: Ensure the SQL Warehouse auto-stop timeout is set long enough, or the connector will restart it automatically on the next sync cycle. Set auto-stop to minimum if cost permits.

### Materialization stuck in PENDING

Wait 30-60 seconds — this is normal during shard assignment. If still stuck:

```bash
flowctl logs --task <TENANT>/<PATH>/materialize-databricks --since 5m | jq 'select(.level == "error")'
```

## Related Skills

- `schema-field-selection` — Control which fields are materialized
- `connector-disable-enable` — Pause/restart existing materializations
- `connector-delete-recreate` — Nuclear option for stuck materializations
- `estuary-logs` — Deep log analysis
- `estuary-catalog-status` — Status checking
