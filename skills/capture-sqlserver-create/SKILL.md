---
name: capture-sqlserver-create
description: Create a SQL Server CDC capture using flowctl. Use when setting up real-time streaming from SQL Server, Azure SQL, Amazon RDS SQL Server, or Google Cloud SQL. Use when user says "capture SQL Server", "stream from MSSQL", "SQL Server CDC", or "connect SQL Server to Estuary".
---

# Create SQL Server Capture

Create an Estuary capture from Microsoft SQL Server using Change Data Capture (CDC).

**Applies to**: source-sqlserver, source-amazon-rds-sqlserver, source-google-cloud-sql-sqlserver, source-azure-sqlserver

## Step 0: Load Connector Documentation

Before proceeding, fetch the official connector docs for prerequisites, config reference, and cloud-specific setup.

**Always load the main page:**
https://docs.estuary.dev/reference/Connectors/capture-connectors/SQLServer/

**Then load the variant subpage based on the user's SQL Server type:**

| Variant | Docs URL |
|---------|----------|
| Self-hosted SQL Server | Main page covers this |
| Azure SQL Database | Main page covers this |
| Amazon RDS SQL Server | https://docs.estuary.dev/reference/Connectors/capture-connectors/SQLServer/amazon-rds-sqlserver/ |
| Google Cloud SQL | https://docs.estuary.dev/reference/Connectors/capture-connectors/SQLServer/google-cloud-sql-sqlserver/ |

Use WebFetch to load these pages. Together they cover:

- Prerequisites (CDC enable at database and table level, user permissions)
- Full config property reference
- Cloud-specific setup instructions
- SSH tunnel configuration
- Network access / IP allowlisting

This skill provides the **flowctl workflow** and **troubleshooting** that docs don't cover.

## Step 1: Gather Requirements

Before writing any YAML, ask the user:

1. **SQL Server variant?** — Self-hosted, Azure SQL Database, Amazon RDS, or Google Cloud SQL
2. **Edition?** — CDC requires Enterprise, Developer, or Evaluation (not Standard/Express). Azure SQL supports all tiers.
3. **Network path?** — Direct connection (cloud with IP allowlist), SSH tunnel (private network), Private Link (AWS/Azure/GCP), or ngrok (local dev)
4. **Non-default data plane?** — Most users use the default. Ask if they need a non-default data plane.
5. **Tables to capture?** — Which specific tables (CDC must be enabled per table)
6. **History mode?** — Standard CDC (`false`) or full event history (`true`)

## Step 2: Find the Correct Connector Version

Always use the latest numbered version tag. Query the connector registry to find it:

```bash
flowctl raw get --table connector_tags \
  --query 'documentation_url=ilike.*source-sqlserver*' \
  --query 'select=image_tag,documentation_url' \
  --output yaml
```

Choose the connector image:

| Variant | Connector Image |
|---------|----------------|
| Self-hosted / Azure SQL | `ghcr.io/estuary/source-sqlserver` |
| Amazon RDS SQL Server | `ghcr.io/estuary/source-amazon-rds-sqlserver` |
| Google Cloud SQL | `ghcr.io/estuary/source-google-cloud-sql-sqlserver` |

## Step 3: Help User Complete Prerequisites

Walk the user through prerequisites from the docs loaded in Step 0. SQL Server CDC setup is more involved than other databases:

1. **SQL Server Agent MUST be running** — The Agent runs CDC capture/cleanup jobs. Without it, CDC data won't be processed. Azure SQL doesn't need this (managed automatically).
2. **Enable CDC on database** — The command varies by platform:
   - Standard: `EXEC sys.sp_cdc_enable_db;`
   - RDS: `EXEC msdb.dbo.rds_cdc_enable_db '<db>';`
   - Cloud SQL: `EXEC msdb.dbo.gcloudsql_cdc_enable_db '<db>';`
3. **Enable CDC on each table** — Must be done individually per table
4. **User permissions** — SELECT on dbo and cdc schemas, VIEW DATABASE STATE

## Step 4: Create the Capture Spec File

Build `flow.yaml` using the config reference from the docs. Minimal required config:

```yaml
captures:
  <tenant>/<path>/source-sqlserver:
    endpoint:
      connector:
        image: ghcr.io/estuary/source-sqlserver:<version>
        config:
          address: "<host>:1433"
          database: "<database>"
          user: "<username>"
          password: "<password>"
          historyMode: false
    bindings: []
```

For SSH tunnel, add `networkTunnel.sshForwarding` block — see docs for full config.

## Step 5: Discover and Publish

```bash
# Discover CDC-enabled tables
flowctl discover --source flow.yaml

# Review the generated bindings
cat flow.yaml

# Publish the capture
flowctl catalog publish --source flow.yaml --auto-approve
```

**Note:** Only tables with CDC enabled will appear in discovery results. If a table is missing, it needs CDC enabled first.

## Step 6: Verify

```bash
# Check status (expect PENDING → BACKFILLING → OK: Streaming CDC Events)
flowctl catalog status <tenant>/<path>/source-sqlserver

# View recent logs
flowctl logs --task <tenant>/<path>/source-sqlserver --since 5m | jq -c '{ts, message}'

# Read captured data
flowctl collections read --collection <tenant>/<path>/dbo/<table> --uncommitted | head -10
```

**Status progression:**
1. `PENDING` — normal for ~30 seconds during shard assignment
2. `BACKFILLING` — initial table snapshots
3. `OK: Streaming CDC Events` — CDC running normally

## Troubleshooting

### "CDC is not enabled for the database"

**Cause**: Database-level CDC not enabled

**Fix**: Run the appropriate enable command for your platform (see Step 3).

### "table has no capture instances"

**Cause**: CDC not enabled on the specific table

**Fix**:
```sql
EXEC sys.sp_cdc_enable_table
    @source_schema = 'dbo',
    @source_name = 'your_table',
    @role_name = 'flow_capture',
    @supports_net_changes = 1;
```

### "no valid capture instances for stream"

**Cause**: Table was dropped and recreated, CDC capture instance has newer start LSN

**Fix**: Increment `backfill` counter on the binding:
```yaml
bindings:
  - resource:
      namespace: dbo
      stream: your_table
      backfill: 1  # Increment this
    target: tenant/path/dbo/your_table
```

### "maximum CDC LSN is currently unset"

**Cause**: SQL Server Agent is not running, CDC jobs aren't processing

**Fix**: Start SQL Server Agent service. Verify jobs exist: `EXEC sys.sp_cdc_help_jobs;`

### "CDC worker may not be running" or fence position errors

**Cause**: CDC capture job is missing, stopped, or broken

**Diagnostic:**
```sql
SELECT name, is_cdc_enabled FROM sys.databases WHERE name = DB_NAME();
EXEC sys.sp_cdc_help_jobs;
SELECT * FROM sys.dm_cdc_log_scan_sessions ORDER BY start_time DESC;
```

**Fix**: If jobs missing, recreate:
```sql
EXEC sys.sp_cdc_add_job @job_type = 'capture';
EXEC sys.sp_cdc_add_job @job_type = 'cleanup';
```

### "invalid '__$start_lsn' value"

**Cause**: CDC role_name mismatch — user not in the correct role

**Fix**: Check role and add user:
```sql
SELECT capture_instance, role_name FROM cdc.change_tables;
EXEC sp_addrolemember '<role_name>', 'flow_capture';
```

### RoleName:'null' issue

**Cause**: Literal string `"null"` used as role_name instead of SQL NULL

**Fix**: Re-enable CDC with NULL role:
```sql
EXEC sys.sp_cdc_disable_table @source_schema = 'dbo', @source_name = 'table', @capture_instance = 'all';
EXEC sys.sp_cdc_enable_table @source_schema = 'dbo', @source_name = 'table', @role_name = NULL, @supports_net_changes = 1;
```

### CDC partially enabled (broken state)

**Symptoms**: CDC commands fail with errors like "user cdc already exists"

**Fix**: Disable and re-enable CDC at database level:
```sql
EXEC sys.sp_cdc_disable_db;
EXEC sys.sp_cdc_enable_db;
-- Then re-enable on each table
```

### Permission denied on dbo or cdc schema

**Fix**:
```sql
GRANT SELECT ON SCHEMA::dbo TO flow_capture;
GRANT SELECT ON SCHEMA::cdc TO flow_capture;
GRANT VIEW DATABASE STATE TO flow_capture;
```

### Capture stuck in PENDING

Wait 30-60 seconds — this is normal during shard assignment. If still stuck:
```bash
flowctl logs --task <tenant>/<path>/source-sqlserver --since 5m | jq 'select(.level == "error" or .level == "warn")'
```

## Related Skills

- `connector-disable-enable` — Pause/restart existing captures
- `connector-delete-recreate` — Nuclear option for stuck captures
- `estuary-logs` — Deep log analysis
- `estuary-catalog-status` — Status checking
