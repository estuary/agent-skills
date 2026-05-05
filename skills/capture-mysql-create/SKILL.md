---
name: capture-mysql-create
description: Create a MySQL CDC capture using flowctl with binlog replication. Use when setting up streaming from MySQL, Amazon RDS MySQL, or Aurora MySQL. Use when user says "capture MySQL", "stream from MySQL", "MySQL CDC", "binlog replication", or "connect MySQL to Estuary".
---

# Create MySQL Capture

Create a MySQL capture using flowctl to stream data from MySQL tables into Estuary collections using Change Data Capture (CDC) via binary log (binlog) replication.

**Applies to**: source-mysql, source-amazon-rds-mysql, source-amazon-aurora-mysql, source-google-cloud-sql-mysql, source-azure-mysql

## Step 0: Load Connector Documentation

Before proceeding, fetch the official connector docs for prerequisites, config reference, and cloud-specific setup.

**Always load the main page:**
https://docs.estuary.dev/reference/Connectors/capture-connectors/MySQL/

**Then load the variant subpage based on the user's MySQL type:**

| Variant | Docs URL |
|---------|----------|
| Self-hosted MySQL | Main page covers this |
| Amazon Aurora MySQL | Main page covers this |
| Amazon RDS MySQL | https://docs.estuary.dev/reference/Connectors/capture-connectors/MySQL/amazon-rds-mysql/ |
| Google Cloud SQL MySQL | https://docs.estuary.dev/reference/Connectors/capture-connectors/MySQL/google-cloud-sql-mysql/ |

Use WebFetch to load these pages. Together they cover:

- Prerequisites (binlog format, user permissions, binlog retention)
- Full config property reference
- Cloud-specific setup instructions
- SSH tunnel configuration
- Network access / IP allowlisting
- Troubleshooting common errors

This skill provides the **flowctl workflow** and **decision logic** that docs don't cover.

## Step 1: Gather Requirements

Before writing any YAML, ask the user:

1. **MySQL variant?** — Self-hosted, Amazon RDS, Aurora MySQL, Google Cloud SQL, or Azure
2. **Network path?** — Direct connection (cloud with IP allowlist), SSH tunnel (private network), Private Link (AWS/Azure/GCP), or ngrok (local dev)
3. **Non-default data plane?** — Most users use the default. Ask if they need a non-default data plane.
4. **Tables to capture?** — All tables or specific subset
5. **History mode?** — Standard CDC (`false`) or full event history (`true`)
6. **DATETIME columns?** — If yes, need timezone config (e.g., `America/New_York`)

## Step 2: Find the Correct Connector Version

Always use the latest numbered version tag. Query the connector registry to find it:

```bash
flowctl raw get --table connector_tags \
  --query 'documentation_url=ilike.*source-mysql*' \
  --query 'select=image_tag,documentation_url' \
  --output yaml
```

Choose the connector image based on the user's MySQL variant:

| Variant | Connector Image |
|---------|----------------|
| Self-hosted / Vanilla | `ghcr.io/estuary/source-mysql` |
| Amazon RDS MySQL | `ghcr.io/estuary/source-amazon-rds-mysql` |
| Amazon Aurora MySQL | `ghcr.io/estuary/source-amazon-aurora-mysql` |
| Google Cloud SQL MySQL | `ghcr.io/estuary/source-google-cloud-sql-mysql` |
| Azure Database for MySQL | `ghcr.io/estuary/source-azure-mysql` |

## Step 3: Help User Complete Prerequisites

Walk the user through prerequisites from the docs loaded in Step 0:

1. **Binlog format** — must be ROW: `SHOW VARIABLES LIKE 'binlog_format';`
2. **Binlog row image** — must be FULL: `SHOW VARIABLES LIKE 'binlog_row_image';`
3. **User permissions** — needs SELECT, REPLICATION CLIENT, REPLICATION SLAVE
4. **Binlog retention** — at least 24-72 hours recommended

For RDS: binlog retention is set via `CALL mysql.rds_set_configuration('binlog retention hours', 72);`

## Step 4: Create the Capture Spec File

Build `flow.yaml` using the config reference from the docs. Minimal required config:

```yaml
captures:
  <tenant>/<path>/source-mysql:
    endpoint:
      connector:
        image: ghcr.io/estuary/source-mysql:<version>
        config:
          address: "<host>:<port>"
          user: "<username>"
          password: "<password>"
          historyMode: false
    bindings: []
```

**Important fields not in minimal config but commonly needed:**
- `timezone: "America/New_York"` — required if tables have DATETIME columns
- `advanced.dbname: "your_app_db"` — required if user can't access the `mysql` system database

For SSH tunnel, add `networkTunnel.sshForwarding` block — see docs for full config.

## Step 5: Discover and Publish

```bash
# Discover tables
flowctl discover --source flow.yaml

# Review the generated bindings
cat flow.yaml

# Publish the capture
flowctl catalog publish --source flow.yaml --auto-approve
```

## Step 6: Verify

```bash
# Check status (expect PENDING → BACKFILLING → OK: Streaming Binlog Events)
flowctl catalog status <tenant>/<path>/source-mysql

# View recent logs
flowctl logs --task <tenant>/<path>/source-mysql --since 5m | jq -c '{ts, message}'

# Read captured data
flowctl collections read --collection <tenant>/<path>/<schema>/<table> --uncommitted | head -10
```

**Status progression:**
1. `PENDING` — normal for ~30 seconds during shard assignment
2. `BACKFILLING` — initial table snapshots
3. `OK: Streaming Binlog Events` — CDC running normally

## Troubleshooting

### "historyMode is required"

**Cause**: Missing `historyMode` field in config

**Fix**: Add `historyMode: false` (or `true` for full event history).

### "Access denied to database 'mysql'"

**Cause**: Capture user can't access the `mysql` system database

**Fix**: Specify an alternative database:
```yaml
config:
  advanced:
    dbname: "your_application_db"
```

### "unsupported DML query" or statement-based binlog error

**Cause**: `binlog_format` is not ROW

**Fix**: `SET GLOBAL binlog_format = 'ROW';` — for RDS/Cloud SQL, update the parameter group/flags.

### "could not find first log file name in binary log index file"

**Cause**: Binlog files purged; connector can't find its last position. Must re-backfill.

**Prevention**: Increase retention — `SET GLOBAL binlog_expire_logs_seconds = 259200;` (72 hours). For RDS: `CALL mysql.rds_set_configuration('binlog retention hours', 72);`

### "log event entry exceeded max_allowed_packet"

**Cause**: A single row/transaction exceeds MySQL's `max_allowed_packet`

**Fix**: `SET GLOBAL max_allowed_packet = 1073741824;` (1GB). For RDS: update via parameter group.

### DATETIME values not permitted or incorrect timestamps

**Cause**: Tables have DATETIME columns but timezone not configured

**Fix**: Add `timezone: "America/New_York"` (or appropriate IANA timezone) to config.

### "Access denied; you need REPLICATION SLAVE privilege"

**Cause**: User lacks replication permissions

**Fix**:
```sql
GRANT REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'flow_capture'@'%';
FLUSH PRIVILEGES;
```

### Capture halts after ALTER TABLE

**Cause**: Certain schema changes (beyond ADD/DROP COLUMN) stop the connector. DROP TABLE or TRUNCATE TABLE will also halt.

**Fix**: Check logs for the specific error. May need to remove the binding or re-create the capture.

### Capture appears "stuck" for hours

**Cause**: Processing a very large transaction — the capture must process all changes before checkpointing.

**Fix**: Wait for completion. Check logs for progress. For future large operations, batch into smaller transactions.

### Capture stuck in PENDING

Wait 30-60 seconds — this is normal during shard assignment. If still stuck:
```bash
flowctl logs --task <tenant>/<path>/source-mysql --since 5m | jq 'select(.level == "error" or .level == "warn")'
```

## Related Skills

- `connector-disable-enable` — Pause/restart existing captures
- `connector-delete-recreate` — Nuclear option for stuck captures
- `estuary-logs` — Deep log analysis
- `estuary-catalog-status` — Status checking
