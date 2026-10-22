---
name: capture-oracle-create
description: Create an Oracle CDC capture (LogMiner) using flowctl. Covers container (CDB/PDB) and non-container databases, Amazon RDS, and the batch connector fallback. Use when user says "capture Oracle", "Oracle CDC", "stream from Oracle", "Oracle LogMiner", "connect Oracle to Estuary", or "Oracle to Snowflake".
---

# Create Oracle Capture

Create an Estuary capture that streams inserts, updates, and deletes from Oracle tables into collections using LogMiner (redo and archived logs), after a one-time backfill of each table.

**Applies to**: source-oracle (CDC via LogMiner, the default), source-oracle-batch (polling fallback, not CDC)

## Step 0: Load Connector Documentation

Before proceeding, fetch the official connector docs for prerequisites and the config reference.

**Always load the main page (CDC):**
https://docs.estuary.dev/reference/Connectors/capture-connectors/OracleDB/

**If the user cannot use LogMiner** (some managed services, views, read replicas without LogMiner, custom queries), load the batch connector page instead:
https://docs.estuary.dev/reference/Connectors/capture-connectors/OracleDB/oracle-batch/

Use WebFetch to load these pages. Together they cover:

- Prerequisites for non-container and container databases (user, grants, watermarks table, supplemental logging)
- Amazon RDS specifics
- Full config property reference (including dictionary mode)
- SSH tunnel and private networking options

This skill provides the **flowctl workflow** and **troubleshooting** that docs don't cover.

## Step 1: Gather Requirements

Before writing any YAML, ask the user:

1. **CDC or batch?** — CDC (LogMiner) is recommended. Use the batch connector only if LogMiner is not available, or for views, read replicas, or ad-hoc queries (it polls, so no real-time deletes).
2. **Container database?** — Oracle multitenant (CDB with pluggable databases) or a non-container database? This changes the user name (`c##` prefix), grants, and the `database` value.
3. **Where is it hosted?** — Self-hosted, Oracle Cloud, or Amazon RDS. RDS multitenant (CDB) setups do not allow access to the root container and are not supported for CDC; use a non-CDB instance or the batch connector.
4. **Network path?** — Direct connection (IP allowlist), SSH tunnel, Private Link (private/BYOC deployments), or ngrok (local dev). Estuary must reach the listener (default port 1521).
5. **Non-default data plane?** — Most users use the default. Ask if they need a non-default data plane.
6. **Tables to capture?** — Which schemas and tables. Names are case-sensitive, and table and column names longer than 30 characters are not supported by LogMiner.
7. **History mode?** — Standard CDC (`false`) or full event history (`true`).

## Step 2: Find the Correct Connector Version

Always use the latest numbered version tag. Query the connector registry:

```bash
# CDC (LogMiner)
flowctl raw get --table connector_tags \
  --query 'documentation_url=eq.https://go.estuary.dev/source-oracle' \
  --query 'select=image_tag,documentation_url' \
  --output yaml

# Batch
flowctl raw get --table connector_tags \
  --query 'documentation_url=eq.https://go.estuary.dev/source-oracle-batch' \
  --query 'select=image_tag,documentation_url' \
  --output yaml
```

| Connector | Image |
|-----------|-------|
| CDC (LogMiner) | `ghcr.io/estuary/source-oracle` |
| Batch (polling) | `ghcr.io/estuary/source-oracle-batch` |

Use the returned `image_tag` (currently `:v1`), never a hardcoded version.

## Step 3: Help User Complete Prerequisites

Walk the user through the prerequisites from the docs loaded in Step 0. Oracle CDC setup has more moving parts than most databases, and the most common failure is configuring the connector first and finding out afterwards that it returns no changes:

1. **Archive logging enabled** (ARCHIVELOG mode) with enough retention. On Amazon RDS, automated backups must be on, and retention is set with `rdsadmin.rdsadmin_util.set_configuration('archivelog retention hours', <hours>)`.
2. **All-column supplemental logging** — `ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;` (on RDS: `BEGIN rdsadmin.rdsadmin_util.alter_supplemental_logging(p_action => 'ADD', p_type => 'ALL'); END;`). Minimal supplemental logging is **not** supported; the capture fails its startup check without it.
3. **A dedicated read-only capture user** with LogMiner and catalog grants (below).
4. **A watermarks table** the connector writes to during backfills (below).
5. **Tablespace quota** for the capture user: `ALTER USER <user> QUOTA UNLIMITED ON USERS;` (otherwise `ORA-01950: no privileges on tablespace 'USERS'`).

### Non-container database

```sql
CREATE USER estuary_flow_user IDENTIFIED BY <password>;
GRANT CREATE SESSION TO estuary_flow_user;
GRANT SELECT ANY TABLE TO estuary_flow_user;          -- or grant SELECT on specific tables
CREATE TABLE estuary_flow_user.FLOW_WATERMARKS(SLOT varchar(1000) PRIMARY KEY, WATERMARK varchar(4000));
GRANT SELECT_CATALOG_ROLE TO estuary_flow_user;
GRANT EXECUTE_CATALOG_ROLE TO estuary_flow_user;
GRANT SELECT ON V$DATABASE TO estuary_flow_user;
GRANT SELECT ON V$LOG TO estuary_flow_user;
GRANT LOGMINING TO estuary_flow_user;
GRANT INSERT, UPDATE ON estuary_flow_user.FLOW_WATERMARKS TO estuary_flow_user;
ALTER USER estuary_flow_user QUOTA UNLIMITED ON USERS;
```

### Container database (CDB with a PDB)

Run as a DBA in the root container. Use a **common user** (`c##` prefix); in the connector config, `database` is the **PDB name** (for example `FREEPDB1` on Oracle Free), not the CDB.

```sql
CREATE USER c##estuary_flow_user IDENTIFIED BY <password> CONTAINER=ALL;
GRANT CREATE SESSION TO c##estuary_flow_user CONTAINER=ALL;
GRANT SELECT ANY TABLE TO c##estuary_flow_user CONTAINER=ALL;
GRANT SELECT_CATALOG_ROLE TO c##estuary_flow_user CONTAINER=ALL;
GRANT EXECUTE_CATALOG_ROLE TO c##estuary_flow_user CONTAINER=ALL;
GRANT LOGMINING TO c##estuary_flow_user CONTAINER=ALL;
GRANT ALTER SESSION TO c##estuary_flow_user CONTAINER=ALL;
GRANT SET CONTAINER TO c##estuary_flow_user CONTAINER=ALL;
ALTER USER c##estuary_flow_user QUOTA UNLIMITED ON USERS CONTAINER=ALL;
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;

-- the watermarks table lives in the PDB
ALTER SESSION SET CONTAINER = <PDB_NAME>;
CREATE TABLE c##estuary_flow_user.FLOW_WATERMARKS(SLOT varchar(1000) PRIMARY KEY, WATERMARK varchar(4000));
GRANT INSERT, UPDATE ON c##estuary_flow_user.FLOW_WATERMARKS TO c##estuary_flow_user;
```

### Check it before you build anything

```sql
SELECT LOG_MODE, SUPPLEMENTAL_LOG_DATA_ALL FROM V$DATABASE;        -- expect ARCHIVELOG, YES
SELECT MIN(FIRST_TIME) FROM V$ARCHIVED_LOG WHERE NAME IS NOT NULL;  -- should be well over 24 hours ago (7+ days recommended)
```

**Retention check:** at startup the capture fails if the oldest archived log is less than 24 hours old, and warns below 7 days (the check is skipped for databases created less than 24 hours ago). If archived logs are deleted aggressively (for example hourly backups that remove them), CDC cannot keep up; fix the backup or deletion policy, or use the batch connector.

## Step 4: Create the Capture Spec File

Build `flow.yaml` using the config reference from the docs. Minimal required config (CDC):

```yaml
captures:
  <tenant>/<path>/source-oracle:
    endpoint:
      connector:
        image: ghcr.io/estuary/source-oracle:<version>
        config:
          address: "<host>:1521"
          user: "estuary_flow_user"          # c##estuary_flow_user for a container database
          password: "<password>"
          database: "<ORCL | PDB name>"      # the PDB name for a container database
          historyMode: false
    bindings: []
```

Notes on the config:

- **Do not set `advanced.discover_schemas` unless you must.** Restricting discovery to your data schema hid the watermarks table and the capture failed with `table missing from latest autodiscovery`. If you do filter, make sure the watermarks table is also discoverable (or leave the filter unset). Schema names are case-sensitive (usually uppercase).
- **`advanced.dictionary_mode`**: `smart` (default) switches between `online` (light, but schema changes can break the capture) and `extract` (handles schema changes, uses more database CPU, RAM, and disk). Leave it on `smart` unless you have a reason.
- **`advanced.watermarksTable`** defaults to `<USER>.FLOW_WATERMARKS`; set it only if you created the table elsewhere (`<schema>.<table>` form).
- For an SSH tunnel, add a `networkTunnel.sshForwarding` block (`sshEndpoint`, `privateKey`); see the docs for the full config.
- Oracle RAC needs no special configuration.

## Step 5: Discover and Publish

```bash
# Discover tables (retry once if the first attempt fails)
flowctl discover --source flow.yaml

# Review the generated bindings; remove tables you do not want
cat flow.yaml

# Publish
flowctl catalog publish --source flow.yaml --auto-approve
```

Collections are named `<tenant>/<path>/<schema>/<table>` (lowercased), for example `acmeCo/oracle/shop/orders`.

## Step 6: Verify

```bash
# Check status (expect: Catching up on CDC history -> Streaming CDC Events; it can flap through a warning once while autodiscovery settles)
flowctl catalog status <tenant>/<path>/source-oracle

# View recent logs
flowctl logs --task <tenant>/<path>/source-oracle --since 5m | jq -c '{ts, level, message}'

# Read captured data
flowctl collections read --collection <tenant>/<path>/<schema>/<table> | head -5
```

**What the documents look like:** each change is one document with the row's columns plus a `_meta` block: `_meta.op` is `c` (insert), `u` (update), or `d` (delete); updates include `_meta.before`; `_meta.source` carries the schema, table, SCN, and timestamp. Oracle `NUMBER` values arrive as strings and `DATE`/`TIMESTAMP` as ISO text; LOB, `LONG`, `XMLTYPE`, `BINARY_FLOAT`, and `BINARY_DOUBLE` columns are left out of the discovered schema. Destinations decide how to type these (for example Snowflake lands dates as `TEXT`).

## Local test with Docker (optional)

A free Oracle instance is enough to try the connector end to end. On Apple Silicon use a multi-arch image such as `gvenzl/oracle-free:23-slim-faststart` (it runs init scripts from `/container-entrypoint-initdb.d`; Oracle's own image does not run them). In the init script, switch to ARCHIVELOG (`SHUTDOWN IMMEDIATE`, `STARTUP MOUNT`, `ALTER DATABASE ARCHIVELOG`, `ALTER DATABASE OPEN`, open the PDBs), enable all-column supplemental logging, and run the container-database SQL from Step 3. Expose port 1521 with `ngrok tcp 1521` and use the ngrok `host:port` as `address`, user `c##estuary_flow_user`, and database `FREEPDB1`. The image's database is pre-created, so the 24-hour retention check passes on its build-time archived-log entry; keep the container volume between runs. A complete example (compose file, init scripts, traffic generator) lives in the `estuary/examples` repo.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `supplemental logging not enabled` (startup check) | Not enabled, or only minimal logging | `ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;` (RDS: `rdsadmin_util.alter_supplemental_logging`). Minimal logging is not supported. Table-level logging weaker than ALL COLUMNS only warns. |
| `archived log retention is less than 24 hours` | Archived logs are deleted too quickly (backups, FRA cleanup) | Raise retention (RDS: `set_configuration('archivelog retention hours', ...)`), fix the delete policy, check `SELECT MIN(FIRST_TIME) FROM V$ARCHIVED_LOG WHERE NAME IS NOT NULL`. Use the batch connector if retention cannot be guaranteed. |
| `table missing from latest autodiscovery` / watermarks table error | `discover_schemas` filter excludes the watermarks table, or the table does not exist in the PDB | Remove the filter (or include the watermarks table's schema); verify `FLOW_WATERMARKS` exists in the PDB and the user has INSERT, UPDATE on it. |
| `ORA-01950: no privileges on tablespace 'USERS'` | No quota for the capture user | `ALTER USER <user> QUOTA UNLIMITED ON USERS;` (add `CONTAINER=ALL` for a common user) |
| Login fails / `ORA-01017` | Wrong user form or password | Container database: user must be `c##...` and `database` the PDB name |
| No tables discovered | User lacks SELECT on the tables, or schema filter is wrong (names are case-sensitive) | Grant SELECT (or `SELECT ANY TABLE`); check schema case |
| Long table or column name fails the startup check | LogMiner does not support names over 30 characters | Rename, or exclude that table |
| `redo log gap` / `redo log SCN gap` errors | The capture fell behind and the logs it needs were deleted | Fix retention, then backfill the affected bindings (increment `backfill`) |
| `PGA memory used by the instance exceeds PGA_AGGREGATE_LIMIT` | Large mining iterations on a busy database | Lower the SCN range mined per iteration (`advanced.incremental_scn_range`; check the connector docs for the current name), or give the instance more PGA |
| Capture on RDS multitenant (CDB) fails | RDS does not allow access to the root container | Use a non-CDB RDS instance, or the batch connector |
| Schema changes break the capture | `dictionary_mode: online` | Use `smart` (default) or `extract` |
| Capture stuck in PENDING | Shard assignment | Wait 30-60 seconds, then check logs for errors |
| Status says "waiting on connector status" for minutes | Status label can lag | Check the logs and read the collection; judge by data flowing |

### Diagnosing with logs

```bash
# Errors and warnings only
flowctl logs --task <tenant>/<path>/source-oracle --since 10m | jq 'select(.level == "error" or .level == "warn")'
```

## Related Skills

- `estuary-connector-restart` — Pause and restart existing captures
- `estuary-logs` — Deep log analysis
- `estuary-catalog-status` — Status checking
- `estuary-task-health` — End-to-end health check for a task
- `estuary-ssh-tunnels` — Diagnose and fix SSH tunnel connection issues
- `materialize-snowflake-create` — A common destination for Oracle data
- `capture-generic-create` — Use for any other connector
