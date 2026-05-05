---
name: capture-postgres-create
description: Create a new PostgreSQL CDC capture from scratch using flowctl. Covers all Postgres variants (vanilla, Aurora, RDS, Cloud SQL, Supabase, Neon). Use when user says "capture Postgres", "stream from Postgres", "Postgres CDC", "set up replication slot", or "connect Postgres to Estuary".
---

# Create PostgreSQL Capture

Create a PostgreSQL CDC capture that continuously streams changes from Postgres tables into Estuary collections.

**Applies to**: source-postgres, source-amazon-aurora-postgres, source-amazon-rds-postgres, source-google-cloud-sql-postgres, source-supabase-postgres, source-neon-postgres

## Step 0: Load Connector Documentation

Before proceeding, fetch the official connector docs for prerequisites, config reference, and cloud-specific setup.

**Always load the main page:**
https://docs.estuary.dev/reference/Connectors/capture-connectors/PostgreSQL/

**Then load the variant subpage based on the user's Postgres type:**

| Variant | Docs URL |
|---------|----------|
| Self-hosted / Vanilla | Main page covers this |
| Amazon Aurora | Main page covers this |
| Azure Database | Main page covers this |
| Amazon RDS | https://docs.estuary.dev/reference/Connectors/capture-connectors/PostgreSQL/amazon-rds-postgres/ |
| Google Cloud SQL | https://docs.estuary.dev/reference/Connectors/capture-connectors/PostgreSQL/google-cloud-sql-postgres/ |
| Supabase | https://docs.estuary.dev/reference/Connectors/capture-connectors/PostgreSQL/Supabase/ |
| Neon | https://docs.estuary.dev/reference/Connectors/capture-connectors/PostgreSQL/neon-postgres/ |

Use WebFetch to load these pages. Together they cover:

- Prerequisites (WAL level, user creation, publication setup, watermarks table)
- Full config property reference
- Cloud-specific setup instructions
- SSH tunnel configuration
- Network access / IP allowlisting

This skill provides the **flowctl workflow** and **troubleshooting** that docs don't cover.

## Step 1: Gather Requirements

Before writing any YAML, ask the user:

1. **Postgres variant?** — Vanilla, Amazon RDS, Aurora, Google Cloud SQL, Supabase, Neon
2. **Network path?** — Direct connection (cloud-hosted with IP allowlist), SSH tunnel (private network/on-prem), Private Link (AWS/Azure/GCP), or ngrok (local dev)
3. **Non-default data plane?** — Most users use the default. Ask if they need a non-default data plane.
4. **Tables to capture?** — All standard tables in the publication is the default. A specific subset, partitioned tables (must capture child partitions, not the parent), views (require batch connector instead of CDC), or tables not yet in the publication all require additional configuration.
5. **History mode?** — Standard CDC (`false`) or full event history (`true`)

## Step 2: Find the Correct Connector Version

Always use the latest numbered version tag. Query the connector registry to find it:

```bash
# For standard Postgres
flowctl raw get --table connector_tags \
  --query 'documentation_url=eq.https://go.estuary.dev/source-postgres' \
  --query 'select=image_tag,documentation_url' \
  --output yaml
```

To find any Postgres variant version:

```bash
flowctl raw get --table connector_tags \
  --query 'documentation_url=ilike.*<connector-name>*' \
  --query 'select=image_tag,documentation_url' \
  --output yaml
```

Choose the connector image based on the user's Postgres variant:
| Variant | Connector Image |
|---------|----------------|
| Vanilla / self-hosted | `ghcr.io/estuary/source-postgres` |
| Amazon RDS | `ghcr.io/estuary/source-amazon-rds-postgres` |
| Amazon Aurora | `ghcr.io/estuary/source-amazon-aurora-postgres` |
| Google Cloud SQL | `ghcr.io/estuary/source-google-cloud-sql-postgres` |
| Supabase | `ghcr.io/estuary/source-supabase-postgres` |
| Neon | `ghcr.io/estuary/source-neon-postgres` |

## Step 3: Help User Complete Prerequisites

Walk the user through the prerequisites from the docs page loaded in Step 0. Key items:

1. **Logical replication** — `SHOW wal_level;` must return `logical`
2. **WAL retention safety** — Recommend setting `max_slot_wal_keep_size` (50GB is a good starting point, more for high-change-rate databases). Without this, Postgres retains unbounded WAL if the capture ever stalls, which can fill the disk and crash the database. The connector does not check this — it's a safety net for the source database, not a replication prerequisite.
3. **Capture user** — needs REPLICATION permission and SELECT access
4. **Publication** — `CREATE PUBLICATION flow_publication FOR ALL TABLES;`
5. **Watermarks table** — recommended for backfill accuracy

If the user is on a managed service (RDS, Cloud SQL, Supabase), refer to the cloud-specific section in the docs.

## Step 4: Create the Capture Spec File

Build `flow.yaml` using the config reference from the docs. Minimal required config:

```yaml
captures:
  <tenant>/<path>/source-postgres:
    endpoint:
      connector:
        image: ghcr.io/estuary/source-postgres:<version>
        config:
          address: "<host>:<port>"
          database: "<database_name>"
          user: "<username>"
          credentials:
            auth_type: "UserPassword"
            password: "<password>"
          historyMode: false
    bindings: []
```

For SSH tunnel, add `networkTunnel.sshForwarding` block — see docs for full config.

For AWS IAM or GCP IAM auth, set `credentials.auth_type` accordingly — see docs for required fields.

## Step 5: Discover Tables

```bash
flowctl discover --source flow.yaml
```

**Note:** The first discover attempt occasionally fails with a generic error. If this happens, simply retry — it typically succeeds on the second attempt.

What this does:

1. Connects to the database
2. Discovers all tables in the publication
3. Generates JSON schemas from table structures
4. Updates `flow.yaml` with bindings
5. Creates SOPS-encrypted config file

Generated file structure:

```
flow.yaml                          # Updated with bindings
source-postgres.config.yaml        # SOPS-encrypted credentials
<tenant>/
  flow.yaml                        # Import file
  <path>/
    flow.yaml                      # Collection definitions
    public/
      <table>.write.schema.yaml    # Write schema per table
      <table>.read.schema.yaml     # Read schema per table
```

## Step 6: Review and Customize Bindings

After discovery, review the generated bindings in `flow.yaml`:

- **Remove unwanted tables** — Delete the binding entry
- **Rename target collections** — Change the `target` field

## Step 7: Publish

```bash
flowctl catalog publish --source flow.yaml --auto-approve
```

The `--auto-approve` flag is required for non-interactive use.

## Step 8: Verify

```bash
# Check status (expect PENDING → Backfilling → OK: Streaming CDC Events)
flowctl catalog status <tenant>/<path>/source-postgres

# View recent logs
flowctl logs --task <tenant>/<path>/source-postgres --since 5m | jq -c '{ts, message}'

# Read captured data
flowctl collections read --collection <tenant>/<path>/public/<table> --uncommitted | head -20
```

**Status progression:**

1. `WARNING: waiting for task shards to be ready` (PENDING) — normal for ~30 seconds
2. `Backfilling Tables (N tables backfilling)` — initial data sync
3. `OK: Streaming CDC Events` — capture is running normally

## Step 9: Test Real-Time CDC

Insert a row in the source database and verify it appears in the collection within seconds:

```bash
flowctl collections read --collection <tenant>/<path>/public/<table> --uncommitted | \
  jq 'select(.<key_field> == "<test_value>")'
```

## Troubleshooting

### "no connector tag found for image"

**Cause**: Wrong image tag (e.g., `:dev`)

**Fix**: Query connector_tags for the correct version (see Step 2).

### "Config failed schema validation"

**Cause**: Config structure is wrong. Common mistakes:

- Password at root level instead of `credentials.password`
- Missing `credentials.auth_type: "UserPassword"`
- Missing `historyMode: false`

### "connection refused" or timeout

**Cause**: Database not reachable from Estuary cloud

**Fix**:

1. Verify firewall allows Estuary IPs (see docs for current list)
2. Use SSH tunnel for private networks
3. For local dev: `ngrok tcp 5432`

### "permission denied for replication"

**Cause**: User lacks REPLICATION permission

**Fix**:

```sql
ALTER USER flow_capture WITH REPLICATION;
GRANT pg_read_all_data TO flow_capture;  -- Postgres 14+
```

### "publication does not exist"

**Cause**: Publication not created or wrong name in advanced config

**Fix**:

```sql
CREATE PUBLICATION flow_publication FOR ALL TABLES;
```

### No tables discovered

**Cause**: Publication exists but has no tables, or the capture user lacks permissions to see them. This is a very common support issue.

**Debug**:

```sql
-- Check publication has tables (run as admin)
SELECT p.pubname, pt.schemaname, pt.tablename
FROM pg_publication p
JOIN pg_publication_tables pt ON p.oid = pt.pubid
ORDER BY p.pubname, pt.schemaname, pt.tablename;

-- Check capture user can see tables (run as capture user)
SELECT table_schema, table_name, table_type
FROM information_schema.tables
WHERE table_schema NOT IN ('pg_catalog', 'pg_internal', 'information_schema');
```

If the publication is empty, add tables or recreate with `FOR ALL TABLES`. If the capture user sees fewer tables than admin, it's a permissions/schema visibility issue.

### Capture stuck in PENDING

Wait 30-60 seconds — this is normal during shard assignment. If still stuck:

```bash
flowctl logs --task <tenant>/<path>/source-postgres --since 5m | jq 'select(.level == "error" or .level == "warn")'
```

### Watermarks table warning

**Warning**: `relation "public.flow_watermarks" does not exist`

**Impact**: Capture still works, but backfill accuracy may be affected.

**Fix**: Create the watermarks table per docs prerequisites.

### Discover fails on first attempt

This is a known intermittent issue. Simply retry `flowctl discover --source flow.yaml` — it typically succeeds on the second attempt.

## Related Skills

- `connector-disable-enable` — Pause/restart existing captures
- `connector-delete-recreate` — Nuclear option for stuck captures
- `estuary-logs` — Deep log analysis
- `estuary-catalog-status` — Status checking
