---
name: materialize-clickhouse-create
description: Create a ClickHouse materialization to stream Estuary collections into ClickHouse tables. Use when setting up ClickHouse as a destination for captured data. Use when user says "send to ClickHouse", "materialize to ClickHouse", "ClickHouse destination", or "load into ClickHouse".
---

# Create ClickHouse Materialization

Create a ClickHouse materialization using flowctl to stream data from Estuary collections into ClickHouse tables.

**Applies to**: materialize-clickhouse (self-hosted and ClickHouse Cloud)

## Step 0: Load Connector Documentation

Before proceeding, fetch the official connector docs for prerequisites, config reference, and setup instructions.

**Load the docs page:**
https://docs.estuary.dev/reference/Connectors/materialization-connectors/ClickHouse/

Use WebFetch to load this page. It covers:

- Prerequisites (user creation, database and system table permissions)
- Full config property reference
- SSL configuration (sslmode: disable, require, verify-full)
- ReplacingMergeTree engine details and FINAL directive usage
- Advanced options (delta updates, hard deletes)

**Search Kapa for tribal knowledge** (if the Estuary MCP is configured):

```
Search kapa ai knowledge sources for "materialize clickhouse common issues"
```

If Kapa MCP is not configured, the user can set it up: https://docs.estuary.dev/features/mcp-integration/

This skill provides the **flowctl workflow** and **troubleshooting** that docs don't cover.

## Step 1: Gather Requirements

Before writing any YAML, ask the user:

1. **ClickHouse deployment?** — Self-hosted or ClickHouse Cloud
2. **Address?** — `host[:port]`. Default port is **9440** with TLS enabled (default) or **9000** with TLS disabled. The connector uses the **Native protocol** — not the HTTP interface on port 8123.
3. **Target database?** — Name of the ClickHouse database to materialize to
4. **Username and password?** — Auth type is `user_password` (the only supported auth method)
5. **SSL mode?** — Defaults to `verify-full`. Options: `disable`, `require`, `verify-full`
6. **Non-default data plane?** — Most users use the default. Ask if they need a non-default data plane.
7. **Source collections?** — Which Estuary collections to materialize
8. **Hard deletes?** — Off by default (soft-delete via `_meta/op` column). Enable to insert tombstone rows (`_is_deleted = 1`) that ClickHouse uses to exclude rows from FINAL queries and eventually remove from storage.
9. **Delta updates?** — Off by default. When enabled per-binding, the table uses the `MergeTree` engine instead of `ReplacingMergeTree`. Every store is appended as-is with no deduplication. Use for event logs or history tables.

**CRITICAL: Use the Native protocol, not HTTP.** Port 8123 (HTTP interface) is **not** supported. Use port 9440 (TLS) or 9000 (no TLS).

**CRITICAL: Query with the FINAL directive.** In standard (non-delta) mode, the connector uses `ReplacingMergeTree` with `flow_published_at` as the version column. Duplicate keys and tombstones are deduplicated in background merges — queries must use `SELECT ... FROM my_table FINAL` to see correctly deduplicated results.

**CRITICAL: ClickHouse `allow_nullable_key` must be enabled.** The connector creates tables with `ORDER BY (`_meta/row_id`)` where `_meta/row_id` is `Nullable(Int64)`. ClickHouse refuses nullable sort-key columns by default and the first CREATE TABLE fails with `Code 44: Sorting key contains nullable columns`. See Step 3 for the fix — ClickHouse Cloud has this set already; self-hosted servers need a config snippet.

### Load pattern note (relevant if you build downstream materialized views)

The connector does **not** use plain `INSERT INTO <table>`. For each batch it stages rows into `flow_temp_store_0_<table>` and then issues `ALTER TABLE flow_temp_store_0_<table> MOVE PARTITION ID 'all' TO TABLE <table>`. `MOVE PARTITION` is a metadata operation — it relinks part files atomically without re-writing data.

Two consequences worth telling the user upfront:

- Batch loads are essentially free at the ClickHouse layer (no merge cost amortized at write time).
- **Classic insert-triggered `MATERIALIZED VIEW` objects attached to the destination table do NOT fire** — MOVE PARTITION is invisible to MV triggers. Any `CREATE MATERIALIZED VIEW … TO <events_feed> AS SELECT … FROM <table>` will freeze at the rows that existed at MV creation time and silently never update. Use a `REFRESHABLE MATERIALIZED VIEW` (ClickHouse 23.12+) instead — it polls on a schedule and atomically swaps the target.

## Step 2: Find the Correct Connector Version

Always use the latest numbered version tag. Query the connector registry:

```bash
flowctl raw get --table connector_tags \
  --query 'documentation_url=eq.https://go.estuary.dev/materialize-clickhouse' \
  --query 'select=image_tag,documentation_url' \
  --output yaml
```

Use the returned `image_tag` — never hardcode a version.

## Step 3: Help User Complete Prerequisites

Walk the user through prerequisites from the docs loaded in Step 0:

1. **ClickHouse database and user** — Create a user with a password
2. **Database permissions** — Grant on the target database:
   ```sql
   GRANT ALL ON <database>.* TO <user>;
   ```
3. **System table permissions** — These are **NOT** covered by the database grant above. The connector needs them for metadata discovery and partition management:
   ```sql
   GRANT SELECT ON system.columns TO <user>;
   GRANT SELECT ON system.parts TO <user>;
   GRANT SELECT ON system.tables TO <user>;
   ```
4. **Optional: row-level policies** — To restrict the user's system table access to only the target database:
   ```sql
   CREATE ROW POLICY estuary_tables ON system.tables FOR SELECT USING database = '<database>' TO <user>;
   CREATE ROW POLICY estuary_columns ON system.columns FOR SELECT USING database = '<database>' TO <user>;
   CREATE ROW POLICY estuary_parts ON system.parts FOR SELECT USING database = '<database>' TO <user>;
   ```
5. **`allow_nullable_key` enabled** — Required at the **MergeTree level** (not user profile) because the connector ORDERs BY a `Nullable(Int64)` column. ClickHouse Cloud has this on already.

   For self-hosted servers, drop a snippet into `/etc/clickhouse-server/config.d/`:

   ```xml
   <!-- /etc/clickhouse-server/config.d/allow_nullable_key.xml -->
   <clickhouse>
     <merge_tree>
       <allow_nullable_key>1</allow_nullable_key>
     </merge_tree>
   </clickhouse>
   ```

   Then restart the server. Putting this under `<profiles>` in `users.d/` **will not work** and causes startup to fail with `Setting allow_nullable_key is neither a builtin setting nor started with the prefix 'SQL_'`.

6. **Network access** — Ensure the ClickHouse server is reachable on the Native protocol port (9440 or 9000) from Estuary

Refer to the docs page for exact SQL commands.

## Step 4: Create the Spec File

Build `flow.yaml` using the config reference from the docs. Minimal required config:

```yaml
materializations:
  <TENANT>/<PATH>/materialize-clickhouse:
    endpoint:
      connector:
        image: ghcr.io/estuary/materialize-clickhouse:<VERSION>
        config:
          address: "<HOST>:9440"
          credentials:
            auth_type: "user_password"
            username: "<USERNAME>"
            password: "<PASSWORD>"
          database: "<DATABASE>"
          # hardDelete: false           # default — soft-delete via _meta/op
          # advanced:
          #   sslmode: "verify-full"    # default; or "require" / "disable"
          #   no_flow_document: false   # exclude root document column from standard updates
    bindings:
      - source: <TENANT>/<collection-path>
        resource:
          table: "<TABLE_NAME>"
          # delta_updates: false        # default; true to use MergeTree append-only
```

**Important**:
- `address` must include the port — use `9440` for TLS (default) or `9000` for plaintext.
- `auth_type` must be `user_password`.
- Setting `sslmode: "disable"` requires using port 9000.

## Step 5: Publish

```bash
flowctl catalog publish --source flow.yaml --auto-approve
```

## Step 6: Verify

```bash
# Check status
flowctl catalog status <TENANT>/<PATH>/materialize-clickhouse

# View logs
flowctl logs --task <TENANT>/<PATH>/materialize-clickhouse --since 5m | jq -c '{ts, message}'
```

**Status progression:**
1. `PENDING` — Normal for ~30 seconds during shard assignment
2. `BACKFILLING` — Initial data sync from collections
3. `OK` — Running normally with real-time updates

**Verify rows in ClickHouse using FINAL:**

```sql
SELECT * FROM <database>.<table> FINAL LIMIT 10;
```

Without `FINAL`, you may see duplicate rows or tombstoned rows that haven't been merged yet.

## Troubleshooting

### "connection refused" or timeout

**Cause**: ClickHouse not reachable on the Native protocol port

**Fix**:
1. Verify port `9440` (TLS) or `9000` (no TLS) is open — **not** `8123` (HTTP interface, unsupported)
2. For ClickHouse Cloud, verify the IP allowlist includes Estuary's IPs (see docs)
3. For self-hosted, check the server's `listen_host` and `tcp_port_secure` / `tcp_port` settings

### "authentication failed" or "Wrong password"

**Cause**: Incorrect credentials or auth type mismatch

**Fix**:
1. Verify the username and password
2. Ensure `auth_type` is `user_password` (the only supported value)
3. Check the user exists: `SELECT name FROM system.users WHERE name = '<user>';`

### "Not enough privileges" on system.columns / system.parts / system.tables

**Cause**: System table grants were skipped — `GRANT ALL ON <db>.*` does **not** cover system tables

**Fix**: Explicitly grant SELECT on the system tables:
```sql
GRANT SELECT ON system.columns TO <user>;
GRANT SELECT ON system.parts TO <user>;
GRANT SELECT ON system.tables TO <user>;
```

### TLS / SSL handshake errors

**Cause**: SSL mode mismatch with the server's TLS configuration

**Fix**:
1. ClickHouse Cloud requires TLS — use `sslmode: "verify-full"` (default) with port 9440
2. For self-hosted without TLS, use `sslmode: "disable"` with port 9000
3. For self-signed certs, use `sslmode: "require"` to skip CA verification
4. Check that the address port matches the SSL mode

### Queries return duplicate rows or unexpected deleted rows

**Cause**: Query is missing the `FINAL` directive

**Fix**: ClickHouse merges duplicates in the background — queries must use `FINAL` to deduplicate at query time:
```sql
SELECT * FROM my_table FINAL;
```

### Deletions don't remove rows from the destination

**Cause**: Soft-delete mode is on by default — deleted rows are kept with `_meta/op = 'd'`

**Fix**: Set `hardDelete: true` in the endpoint config. This makes deletes insert a tombstone row (`_is_deleted = 1`) that ClickHouse uses to exclude rows from FINAL queries and eventually remove from persistent storage.

### "Sorting key contains nullable columns" / Code 44 on CREATE TABLE

**Cause**: ClickHouse refuses `Nullable(...)` columns as sort keys by default. The connector ORDERs BY `_meta/row_id` (`Nullable(Int64)`).

**Fix**: Enable `allow_nullable_key` at the MergeTree level (not user profile). On self-hosted servers, add `/etc/clickhouse-server/config.d/allow_nullable_key.xml`:

```xml
<clickhouse>
  <merge_tree>
    <allow_nullable_key>1</allow_nullable_key>
  </merge_tree>
</clickhouse>
```

Restart the server. ClickHouse Cloud has this set already. **Do NOT put this under `<profiles>` in `users.d/`** — it's a MergeTree-level setting and the wrong placement makes the server fail to start with `Setting allow_nullable_key is neither a builtin setting nor started with the prefix 'SQL_'`.

### A `MATERIALIZED VIEW` over a connector-managed table freezes / doesn't update

**Cause**: The connector loads data via `ALTER TABLE … MOVE PARTITION ID 'all' TO TABLE <table>`. MOVE PARTITION is a metadata-only operation and does **not** fire materialized views subscribed to the destination. Your MV will capture rows that existed at MV-creation time (or any direct INSERTs you make), and silently stop seeing new connector batches.

**Fix**: Use a `REFRESHABLE MATERIALIZED VIEW` (ClickHouse 23.12+) instead, which polls on a cron and atomically swaps the target:

```sql
CREATE MATERIALIZED VIEW my_agg
REFRESH EVERY 5 SECOND
ENGINE = MergeTree ORDER BY (key)
AS SELECT ... FROM <connector_table> FINAL GROUP BY key;
```

Inspect status via `system.view_refreshes`. For sub-second freshness, use a plain `VIEW` with `FINAL` and let the query do the work each time.

### "Table already exists" or schema conflicts

**Cause**: Table exists with an incompatible engine or schema

**Fix**:
1. Drop and recreate the table — let the connector create it with `ReplacingMergeTree` (or `MergeTree` for delta updates)
2. Use a different table name
3. Ensure the collection schema is compatible with the existing table

### Materialization stuck in PENDING

Wait 30-60 seconds — this is normal during shard assignment. If still stuck:

```bash
flowctl logs --task <TENANT>/<PATH>/materialize-clickhouse --since 5m | jq 'select(.level == "error")'
```

### Delta updates: rows accumulating without deduplication

**Cause**: This is the expected behavior of delta updates — the binding uses `MergeTree` and appends every store as-is

**Fix**: This is by design. If you need deduplication, disable `delta_updates` on the binding and let the connector recreate the table with `ReplacingMergeTree`.

## Related Skills

- `connector-disable-enable` — Pause/restart existing materializations
- `connector-delete-recreate` — Nuclear option for stuck materializations
- `estuary-logs` — Deep log analysis
- `estuary-catalog-status` — Status checking
