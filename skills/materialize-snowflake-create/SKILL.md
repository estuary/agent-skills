---
name: materialize-snowflake-create
description: Create a Snowflake materialization to stream Estuary collections into Snowflake tables. Use when setting up Snowflake as a destination for captured data. Use when user says "send to Snowflake", "materialize to Snowflake", "Snowflake destination", or "load into Snowflake".
---

# Create Snowflake Materialization

Create a Snowflake materialization using flowctl to stream data from Estuary collections into Snowflake tables.

**Applies to**: materialize-snowflake

## Step 0: Load Connector Documentation

Before proceeding, fetch the official connector docs for prerequisites, config reference, and setup instructions.

**Load the docs page:**
https://docs.estuary.dev/reference/Connectors/materialization-connectors/Snowflake/

Use WebFetch to load this page. It covers:

- Prerequisites (role/user creation, key-pair setup, warehouse)
- Full config property reference
- Authentication setup (JWT key-pair)
- Advanced options (delta updates, Snowpipe Streaming, hard deletes, sync schedule)

**Search Kapa for tribal knowledge** (if the Estuary MCP is configured):

```
Search kapa ai knowledge sources for "materialize snowflake common issues"
```

If Kapa MCP is not configured, the user can set it up: https://docs.estuary.dev/features/mcp-integration/

This skill provides the **flowctl workflow** and **troubleshooting** that docs don't cover.

## Step 1: Gather Requirements

Before writing any YAML, ask the user:

1. **Account URL?** — Snowflake account identifier (e.g., `xy12345.us-east-1.snowflakecomputing.com`)
2. **Database, schema, warehouse?** — Target database, schema, and warehouse (optional — uses user's default if not specified)
3. **Authentication method?** — **Must be JWT/key-pair** (Snowflake deprecated password auth April 2025)
4. **Timestamp type?** — This is required. Help the user choose (see guidance below).
5. **Non-default data plane?** — Most users use the default. Ask if they need a non-default data plane.
6. **Source collections?** — Which Estuary collections to materialize
7. **Hard deletes?** — Off by default. Without it, deleted rows stay in Snowflake marked with `_meta/op: 'd'`. Enable to physically remove them.
8. **Delta updates?** — Off by default. Switches from one-row-per-key (standard merge) to append-only via Snowpipe Streaming. Use for event logs or history tables.
9. **Sync schedule?** — Controls how often batches are written to Snowflake (default: 30 minutes, `0s` for real-time). Affects latency and warehouse cost.

**CRITICAL**: JWT/key-pair authentication is required. Snowflake deprecated username/password authentication in April 2025. If the user asks about password auth, explain they must use key-pair instead.

### Choosing a timestamp type

This setting controls how Estuary creates timestamp columns in Snowflake. It's required and frequently confusing — especially for users migrating from Fivetran (which defaults to NTZ).

| Type | Stores as | Best for | Spec value |
|------|-----------|----------|------------|
| **TIMESTAMP_LTZ** | UTC internally, displays in session timezone | **Most users** — preserves timezone, prevents ambiguity | `TIMESTAMP_LTZ` |
| **TIMESTAMP_TZ** | UTC + offset | When you need to preserve the original timezone offset (note: DST info is lost — only offset is stored) | `TIMESTAMP_TZ` |
| **TIMESTAMP_NTZ (normalize)** | Wall-clock time, no timezone (converted to UTC first) | **New tasks** that need NTZ — e.g., migrating from Fivetran where downstream queries assume NTZ | `TIMESTAMP_NTZ (normalize to UTC)` |
| **TIMESTAMP_NTZ (discard)** | Wall-clock time, no timezone (raw, timezone stripped) | **Existing tasks** already using NTZ that can't change | `TIMESTAMP_NTZ (discard TZ)` |

**Default recommendation**: Use `TIMESTAMP_LTZ` unless you have a specific reason not to. It stores UTC and lets Snowflake handle display conversion per session — this avoids the "my timestamps shifted by N hours" problems that NTZ causes.

**Migrating from Fivetran?** Fivetran defaults to NTZ, so existing dbt models and queries may assume NTZ columns. Use `TIMESTAMP_NTZ (normalize to UTC)` to match that expectation while still normalizing to UTC before storage.

**Important**: If the user has explicitly set `TIMESTAMP_TYPE_MAPPING` in Snowflake (at account/role/user level), the Estuary setting **must match**. If it's left at Snowflake's default (unset), Estuary will use LTZ regardless of Snowflake's default being NTZ — this is intentional.

## Step 2: Find the Correct Connector Version

Always use the latest numbered version tag. Query the connector registry:

```bash
flowctl raw get --table connector_tags \
  --query 'documentation_url=eq.https://go.estuary.dev/materialize-snowflake' \
  --query 'select=image_tag,documentation_url' \
  --output yaml
```

Use the returned `image_tag` — never hardcode a version.

## Step 3: Help User Complete Prerequisites

Walk the user through prerequisites from the docs loaded in Step 0:

1. **Role and user** — Create a dedicated role and service user
2. **Key-pair generation** — Generate PKCS#8 private key and extract public key
3. **Assign public key** — Set RSA_PUBLIC_KEY on the Snowflake user
4. **Database/schema/warehouse** — Create and grant permissions

Refer to the docs page for exact SQL and shell commands.

## Step 4: Create the Spec File

Build `flow.yaml` using the config reference from the docs. Minimal required config:

```yaml
materializations:
  <TENANT>/<PATH>/materialize-snowflake:
    endpoint:
      connector:
        image: ghcr.io/estuary/materialize-snowflake:<VERSION>
        config:
          host: "<ACCOUNT>.snowflakecomputing.com"
          database: "<DATABASE>"
          schema: "<SCHEMA>"
          warehouse: "<WAREHOUSE>"
          timestamp_type: "TIMESTAMP_LTZ"
          role: "<ROLE>"
          credentials:
            auth_type: "jwt"
            user: "<USERNAME>"
            private_key: "<PKCS8-PRIVATE-KEY>"
    bindings:
      - source: <TENANT>/<collection-path>
        resource:
          table: "<TABLE_NAME>"
```

**Important**: `timestamp_type` is required. Use the value from the table in Step 1 (e.g., `TIMESTAMP_LTZ`, `TIMESTAMP_TZ`, `TIMESTAMP_NTZ (normalize to UTC)`, or `TIMESTAMP_NTZ (discard TZ)`).

## Step 5: Publish

```bash
flowctl catalog publish --source flow.yaml --auto-approve
```

## Step 6: Verify

```bash
# Check status
flowctl catalog status <TENANT>/<PATH>/materialize-snowflake

# View logs
flowctl logs --task <TENANT>/<PATH>/materialize-snowflake --since 5m | jq -c '{ts, message}'
```

**Status progression:**
1. `PENDING` — Normal for ~30 seconds during shard assignment
2. `BACKFILLING` — Initial data sync from collection
3. `OK` — Running normally with real-time updates

## Troubleshooting

### "timestamp_type is required"

**Cause**: Missing `timestamp_type` in config

**Fix**: Add `timestamp_type` — see the choosing guide in Step 1. Default: `TIMESTAMP_LTZ`.

### Timestamps shifted by N hours after backfill

**Cause**: Table was re-created with a different timestamp type (usually NTZ → LTZ)

**Fix**:
1. Set `TIMESTAMP_TYPE_MAPPING` explicitly on the Snowflake user to match the Estuary config:
   ```sql
   ALTER USER <user> SET TIMESTAMP_TYPE_MAPPING = 'TIMESTAMP_NTZ';
   ```
   Note: the `level` column in `SHOW PARAMETERS LIKE 'TIMESTAMP_TYPE_MAPPING'` must show `USER` (not blank) for the connector to recognize the setting.
2. Backfill the affected bindings — the fix only takes effect on new or re-backfilled tables.

### Changed timestamp_type but columns still use the old type

**Cause**: Changing `timestamp_type` only affects new or re-backfilled tables — existing columns keep their original type

**Fix**: Backfill the affected bindings after changing the setting. Increment the `backfill` counter on each binding, or use the UI backfill button.

### "JWT token is invalid" or authentication failures

**Cause**: Key-pair mismatch or incorrect format

**Fix**:
1. Verify public key fingerprints match (local vs Snowflake)
2. Ensure private key includes BEGIN/END markers
3. Ensure private key is PKCS#8 format (not PKCS#1)
4. Check user has correct RSA_PUBLIC_KEY set

### "Warehouse not found" or warehouse errors

**Cause**: Warehouse doesn't exist or user lacks USAGE permission

**Fix**: Verify the warehouse exists and the role has USAGE — see docs for SQL.

### "Schema does not exist"

**Cause**: Schema not created or user lacks permission

**Fix**: Create the schema and grant permissions — see docs for SQL.

### Materialization stuck in PENDING

Wait 30-60 seconds — this is normal during shard assignment. If still stuck:

```bash
flowctl logs --task <TENANT>/<PATH>/materialize-snowflake --since 5m | jq 'select(.level == "error")'
```

### "User cannot be used for login" or "Password required"

**Cause**: Using TYPE = SERVICE user with password auth, or deprecated auth method

**Fix**: Service users require JWT authentication. Ensure `auth_type: "jwt"` with valid private key.

## Related Skills

- `connector-disable-enable` — Pause/restart existing materializations
- `connector-delete-recreate` — Nuclear option for stuck materializations
- `estuary-logs` — Deep log analysis
- `estuary-catalog-status` — Status checking
