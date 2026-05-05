---
name: materialize-redshift-create
description: Create a Redshift materialization to stream Estuary collections into Amazon Redshift tables. Use when setting up Redshift as a destination for captured data. Use when user says "send to Redshift", "materialize to Redshift", "Redshift destination", or "load into Redshift".
---

# Create Redshift Materialization

Create a Redshift materialization using flowctl to stream data from Estuary collections into Amazon Redshift tables.

**Applies to**: materialize-redshift (Redshift Provisioned and Serverless)

## Step 0: Load Connector Documentation

Before proceeding, fetch the official connector docs for prerequisites, config reference, and setup instructions.

**Load the docs page:**
https://docs.estuary.dev/reference/Connectors/materialization-connectors/amazon-redshift/

Use WebFetch to load this page. It covers:

- Prerequisites (Redshift cluster, S3 bucket, IAM credentials)
- Full config property reference
- SSH tunnel configuration
- Network access and IP allowlisting

**Search Kapa for tribal knowledge** (if the Estuary MCP is configured):

```
Search kapa ai knowledge sources for "materialize redshift common issues"
```

If Kapa MCP is not configured, the user can set it up: https://docs.estuary.dev/features/mcp-integration/

This skill provides the **flowctl workflow** and **troubleshooting** that docs don't cover.

## Step 1: Gather Requirements

Before writing any YAML, ask the user:

1. **Redshift endpoint?** — Cluster hostname and port (default: 5439)
2. **Database and schema?** — Target database name, schema (default: `public`)
3. **S3 staging bucket?** — Bucket name and region (same region as Redshift cluster is recommended for best performance)
4. **AWS credentials?** — Access key ID and secret access key with S3 read/write
5. **Network path?** — Direct connection (with IP allowlist) or SSH tunnel
6. **Non-default data plane?** — Most users use the default. Ask if they need a non-default data plane.
7. **Source collections?** — Which Estuary collections to materialize
8. **Hard deletes?** — Off by default. Without it, deleted rows stay in Redshift marked with `_meta/op: 'd'`. Enable to physically remove them.
9. **Delta updates?** — Off by default. Switches from one-row-per-key (standard merge) to append-only. Use for event logs or history tables.
10. **Sync schedule?** — Controls how often batches are written to Redshift (default: 30 minutes, `0s` for real-time). Affects latency and compute cost.

**Key constraints to communicate upfront:**
- Best practice: **one materialization per schema** (multiple bindings, not multiple materializations)
- All identifiers are **lowercase** unless `enable_case_sensitive_identifier` is enabled on the cluster
- Maximum document size is **4 MB** per document

## Step 2: Find the Correct Connector Version

Always use the latest numbered version tag. Query the connector registry:

```bash
flowctl raw get --table connector_tags \
  --query 'documentation_url=eq.https://go.estuary.dev/materialize-redshift' \
  --query 'select=image_tag,documentation_url' \
  --output yaml
```

Use the returned `image_tag` — never hardcode a version.

## Step 3: Help User Complete Prerequisites

Walk the user through prerequisites from the docs loaded in Step 0:

1. **Redshift accessible** — Cluster publicly accessible or SSH tunnel configured
2. **S3 bucket** — Same region as Redshift cluster, for staging data
3. **IAM user** — With read/write access to the S3 bucket
4. **Redshift user** — With CREATE TABLE permissions on the target schema

Refer to the docs page for exact setup steps.

## Step 4: Create the Spec File

Build `flow.yaml` using the config reference from the docs. Minimal required config:

```yaml
materializations:
  <TENANT>/<PATH>/materialize-redshift:
    endpoint:
      connector:
        image: ghcr.io/estuary/materialize-redshift:<VERSION>
        config:
          address: "<CLUSTER-ENDPOINT>:5439"
          user: "<USERNAME>"
          password: "<PASSWORD>"
          database: "<DATABASE>"
          schema: "public"
          awsAccessKeyId: "<AWS-ACCESS-KEY-ID>"
          awsSecretAccessKey: "<AWS-SECRET-ACCESS-KEY>"
          bucket: "<S3-BUCKET-NAME>"
          region: "<AWS-REGION>"
          bucketPath: "estuary-staging"
    bindings:
      - source: <TENANT>/<collection-path>
        resource:
          table: "<table_name>"
```

**Important**: Table names should be lowercase. The `bucket` field is just the bucket name — no `s3://` prefix.

For SSH tunnel, add `networkTunnel.sshForwarding` block — see docs for full config.

## Step 5: Publish

```bash
flowctl catalog publish --source flow.yaml --auto-approve
```

## Step 6: Verify

```bash
# Check status
flowctl catalog status <TENANT>/<PATH>/materialize-redshift

# View logs
flowctl logs --task <TENANT>/<PATH>/materialize-redshift --since 5m | jq -c '{ts, message}'
```

**Status progression:**
1. `PENDING` — Normal for ~30 seconds during shard assignment
2. `BACKFILLING` — Initial data sync from collections
3. `OK` — Running normally with real-time updates

## Troubleshooting

### "column does not exist" (SQLSTATE 42703)

**Cause**: Column was dropped or renamed directly in Redshift outside of Estuary

**Fix**: Avoid altering connector-created tables directly. If the schema drifted, you may need to drop the table and let the connector recreate it (trigger a backfill by incrementing `backfill` counter on the binding).

### "cannot drop sort key column"

**Cause**: The connector tried to alter a table but can't drop a sort key column

**Fix**: If using `always_drop_tables_on_backfill`, disable it. Otherwise, manually drop and recreate the table.

### OOM on Serverless

**Cause**: Query ran out of memory on Redshift Serverless with low RPU

**Fix**: Increase RPU (Redshift Processing Units) on the Serverless workgroup, or reduce the number of fields being materialized using field selection.

### S3 timeout errors

**Cause**: Network or region mismatch between S3 bucket and Redshift cluster

**Fix**:
1. Verify S3 bucket is in the same AWS region as Redshift
2. Check S3 bucket permissions (IAM user needs read/write)
3. Verify Redshift cluster can reach S3 (VPC endpoints or internet access)

### Slow type migrations on large tables

**Cause**: Redshift performs full table rewrite for type changes

**Fix**: This is expected behavior. Do not interrupt — let it complete. For very large tables, consider scheduling during low-traffic periods.

### Materialization stuck in PENDING

Wait 30-60 seconds — this is normal during shard assignment. If still stuck:

```bash
flowctl logs --task <TENANT>/<PATH>/materialize-redshift --since 5m | jq 'select(.level == "error")'
```

### Documents exceeding 4 MB limit

**Cause**: Individual documents are larger than Redshift's 4 MB per-document limit

**Fix**: Use field selection to exclude large fields (e.g., binary blobs, nested arrays), or create a derivation to truncate/filter before materializing.

## Related Skills

- `schema-field-selection` — Reduce materialized fields to avoid size limits
- `connector-disable-enable` — Pause/restart existing materializations
- `connector-delete-recreate` — Nuclear option for stuck materializations
- `estuary-logs` — Deep log analysis
- `estuary-catalog-status` — Status checking
