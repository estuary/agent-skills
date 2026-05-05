---
name: materialize-postgres-create
description: Create a PostgreSQL materialization to stream Estuary collections into PostgreSQL tables. Use when setting up PostgreSQL as a destination for captured data. Use when user says "send to Postgres", "materialize to PostgreSQL", "Postgres destination", or "load into Postgres".
---

# Create PostgreSQL Materialization

Create a PostgreSQL materialization using flowctl to stream data from Estuary collections into PostgreSQL tables.

**Applies to**: materialize-postgres (all variants: vanilla, RDS, Aurora, Cloud SQL, Supabase, AlloyDB, Neon)

## Step 0: Load Connector Documentation

Before proceeding, fetch the official connector docs for prerequisites, config reference, and setup instructions.

**Load the docs page:**
https://docs.estuary.dev/reference/Connectors/materialization-connectors/PostgreSQL/

Use WebFetch to load this page. It covers:

- Prerequisites (user creation, permissions, schema setup)
- Full config property reference
- SSH tunnel configuration
- SSL configuration
- Advanced options (delta updates, hard deletes, sync schedule)

**Search Kapa for tribal knowledge** (if the Estuary MCP is configured):

```
Search kapa ai knowledge sources for "materialize postgres common issues"
```

If Kapa MCP is not configured, the user can set it up: https://docs.estuary.dev/features/mcp-integration/

This skill provides the **flowctl workflow** and **troubleshooting** that docs don't cover.

## Step 1: Gather Requirements

Before writing any YAML, ask the user:

1. **Postgres variant?** — Vanilla, RDS, Aurora, Cloud SQL, Supabase, AlloyDB
2. **Network path?** — Direct connection (cloud-hosted with IP allowlist), SSH tunnel (private network/on-prem), or ngrok (local dev).
Connection poolers are **NOT** supported. If the user mentions Supabase, Neon, PgBouncer, or PgCat, warn them immediately:
   - The connector uses session-level temporary tables and prepared statements that break with pooled connections.
   - **Supabase**: Must use the Direct connection (`db.xxx.supabase.co:5432`), not the pooler (`xxx.pooler.supabase.com:6543`).
   - **Neon**: Must use the non-pooler endpoint — remove `-pooler` from the hostname.
3. **Non-default data plane?** — Most users use the default. Ask if they need a non-default data plane.
4. **Target database and schema?** — Database name, schema (default: `public`)
5. **Source collections?** — Which Estuary collections to materialize
6. **SSL required?** — Some cloud providers require SSL connections
7. **Hard deletes?** — Off by default. Without it, deleted rows stay in the destination marked with `_meta/op: 'd'`. Enable to physically remove them.
8. **Delta updates?** — Off by default. Switches from one-row-per-key (standard merge) to append-only. Use for event logs or history tables.
9. **Sync schedule?** — Controls how often batches are written to the destination (default: 30 minutes, `0s` for real-time). Affects latency and destination compute cost.

## Step 2: Find the Correct Connector Version

Always use the latest numbered version tag. Query the connector registry:

```bash
flowctl raw get --table connector_tags \
  --query 'documentation_url=eq.https://go.estuary.dev/materialize-postgres' \
  --query 'select=image_tag,documentation_url' \
  --output yaml
```

Use the returned `image_tag` — never hardcode a version.

## Step 3: Help User Complete Prerequisites

Walk the user through prerequisites from the docs loaded in Step 0:

1. **Database user** — Create a user with write permissions on the target schema
2. **Schema permissions** — USAGE and CREATE on target schema
3. **Table permissions** — SELECT, INSERT, UPDATE, DELETE if materializing to existing tables
4. **Network access** — Firewall rules or SSH tunnel configured

Refer to the docs page for exact SQL commands and cloud-specific instructions.

## Step 4: Create the Spec File

Build `flow.yaml` using the config reference from the docs. Minimal required config:

```yaml
materializations:
  <TENANT>/<PATH>/materialize-postgres:
    endpoint:
      connector:
        image: ghcr.io/estuary/materialize-postgres:<VERSION>
        config:
          address: "<HOST>:5432"
          database: "<DATABASE>"
          user: "<USERNAME>"
          password: "<PASSWORD>"
          schema: "<SCHEMA>"
    bindings:
      - source: <TENANT>/<collection-path>
        resource:
          table: "<TABLE_NAME>"
```

For SSH tunnel or SSL, add the appropriate config block — see docs for full reference.

## Step 5: Publish

```bash
flowctl catalog publish --source flow.yaml --auto-approve
```

## Step 6: Verify

```bash
# Check status
flowctl catalog status <TENANT>/<PATH>/materialize-postgres

# View logs
flowctl logs --task <TENANT>/<PATH>/materialize-postgres --since 5m | jq -c '{ts, message}'
```

**Status progression:**
1. `PENDING` — Normal for ~30 seconds during shard assignment
2. `BACKFILLING` — Initial data sync from collection
3. `OK` — Running normally with real-time updates

## Troubleshooting

### "connection refused" or timeout

**Cause**: Database not reachable from Estuary cloud

**Fix**:
1. Verify firewall allows Estuary IP addresses (see docs)
2. Check security groups (AWS) or firewall rules (GCP)
3. Use SSH tunnel for private databases
4. For local testing: `ngrok tcp 5432`

### "password authentication failed"

**Cause**: Incorrect credentials or user doesn't exist

**Fix**:
1. Verify username and password
2. Check user exists: `SELECT * FROM pg_user WHERE usename = '<user>';`
3. Verify user can connect from external hosts (check `pg_hba.conf`)

### "permission denied for schema"

**Cause**: User lacks permissions on target schema

**Fix**: Grant USAGE and CREATE on the target schema — see docs for SQL.

### "relation already exists" or schema conflicts

**Cause**: Table already exists with incompatible schema

**Fix**:
1. Drop and recreate the table
2. Use a different table name
3. Ensure collection schema matches existing table

### Materialization stuck in PENDING

Wait 30-60 seconds — this is normal during shard assignment. If still stuck:

```bash
flowctl logs --task <TENANT>/<PATH>/materialize-postgres --since 5m | jq 'select(.level == "error")'
```

### SSL connection errors

**Cause**: SSL mode mismatch or certificate issues

**Fix**:
1. Try `sslmode: "require"` first
2. For AWS RDS, SSL is usually required — use `verify-ca` or `verify-full`
3. For self-signed certs, use `require` mode
4. See docs for full SSL configuration options

### SSH tunnel connection failures

**Cause**: SSH configuration issues

**Fix**:
1. Verify bastion is reachable: `ssh -p 22 user@bastion`
2. Check private key format (must be OpenSSH format)
3. Verify bastion can reach database: `nc -zv db-host 5432`
4. See SSH tunnel troubleshooting docs

## Related Skills

- `connector-disable-enable` — Pause/restart existing materializations
- `connector-delete-recreate` — Nuclear option for stuck materializations
- `estuary-logs` — Deep log analysis
- `estuary-catalog-status` — Status checking
