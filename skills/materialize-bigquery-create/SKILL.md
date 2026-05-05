---
name: materialize-bigquery-create
description: Create a BigQuery materialization to stream Estuary collections into BigQuery tables. Use when setting up BigQuery as a destination for captured data. Use when user says "send to BigQuery", "load into BigQuery", "materialize to BigQuery", or "BigQuery destination".
---

# Create BigQuery Materialization

Create a BigQuery materialization using flowctl to stream data from Estuary collections into BigQuery tables.

**Applies to**: materialize-bigquery

## Step 0: Load Connector Documentation

Before proceeding, fetch the official connector docs for prerequisites, config reference, and setup instructions.

**Load the docs page:**
https://docs.estuary.dev/reference/Connectors/materialization-connectors/BigQuery/

Use WebFetch to load this page. It covers:

- Prerequisites (GCS bucket, service account, IAM roles, dataset)
- Full config property reference
- gcloud CLI commands for setup
- Advanced options (delta updates, hard deletes, sync schedule)

**Search Kapa for tribal knowledge** (if the Estuary MCP is configured):

```
Search kapa ai knowledge sources for "materialize bigquery common issues"
```

If Kapa MCP is not configured, the user can set it up: https://docs.estuary.dev/features/mcp-integration/

This skill provides the **flowctl workflow** and **troubleshooting** that docs don't cover.

## Step 1: Gather Requirements

Before writing any YAML, ask the user:

1. **GCP project ID?** — The Google Cloud project
2. **BigQuery dataset and region?** — Target dataset name and GCP region
3. **GCS staging bucket?** — Bucket name for staging data (**must be same region as dataset!**)
4. **Authentication method?** — Service account key (most common) or GCP IAM (workload identity federation)
5. **Non-default data plane?** — Most users use the default. Ask if they need a non-default data plane.
6. **Source collections?** — Which Estuary collections to materialize
7. **Hard deletes?** — Off by default. Without it, deleted rows stay in BigQuery marked with `_meta/op: 'd'`. Enable to physically remove them.
8. **Delta updates?** — Off by default. Switches from one-row-per-key (standard merge) to append-only. Use for event logs or history tables.
9. **Sync schedule?** — Controls how often batches are written to BigQuery (default: 30 minutes, `0s` for real-time). Affects latency and compute cost.

**CRITICAL**: The GCS staging bucket and BigQuery dataset must be in the same GCP region. Region mismatch is the most common setup error.

## Step 2: Find the Correct Connector Version

Always use the latest numbered version tag. Query the connector registry:

```bash
flowctl raw get --table connector_tags \
  --query 'documentation_url=eq.https://go.estuary.dev/materialize-bigquery' \
  --query 'select=image_tag,documentation_url' \
  --output yaml
```

Use the returned `image_tag` — never hardcode a version.

## Step 3: Help User Complete Prerequisites

Walk the user through prerequisites from the docs loaded in Step 0:

1. **GCS staging bucket** — Create in the same region as the BigQuery dataset
2. **Service account** — Create with required IAM roles:
   - `bigquery.dataEditor` — create/update tables
   - `bigquery.jobUser` — run BigQuery jobs
   - `bigquery.readSessionUser` — read session access
   - `storage.objectAdmin` — read/write staging files in GCS bucket
3. **BigQuery dataset** — Create if it doesn't exist
4. **Service account key** — Generate JSON key file

Refer to the docs page for exact gcloud commands.

## Step 4: Create the Spec File

Build `flow.yaml` using the config reference from the docs. Minimal required config:

```yaml
materializations:
  <TENANT>/<PATH>/materialize-bigquery:
    endpoint:
      connector:
        image: ghcr.io/estuary/materialize-bigquery:<VERSION>
        config:
          project_id: "<GCP-PROJECT-ID>"
          dataset: "<BIGQUERY-DATASET>"
          bucket: "<GCS-BUCKET-NAME>"
          bucket_path: "estuary-staging"
          region: "<REGION>"
          # Option A: Service account key (most common)
          credentials_json: |
            {
              ...full service account key JSON...
            }
          # Option B: GCP IAM (workload identity federation)
          # credentials:
          #   auth_type: "GCPIAM"
          #   gcp_service_account_to_impersonate: "<SERVICE-ACCOUNT-EMAIL>"
          #   gcp_workload_identity_pool_audience: "<WORKLOAD-IDENTITY-POOL-AUDIENCE>"
    bindings:
      - source: <TENANT>/<collection-path>
        resource:
          table: "<TABLE_NAME>"
          dataset: "<OPTIONAL-DATASET-OVERRIDE>"
```

**Important**: `bucket` is just the bucket name — no `gs://` prefix. `region` must match both bucket and dataset.

## Step 5: Publish

```bash
flowctl catalog publish --source flow.yaml --auto-approve
```

## Step 6: Verify

```bash
# Check status
flowctl catalog status <TENANT>/<PATH>/materialize-bigquery

# View logs
flowctl logs --task <TENANT>/<PATH>/materialize-bigquery --since 5m | jq -c '{ts, message}'
```

**Status progression:**
1. `PENDING` — Normal for ~30 seconds during shard assignment
2. `BACKFILLING` — Initial data sync from collections
3. `OK` — Running normally with real-time updates

## Troubleshooting

### "no such bucket" or "bucket does not exist"

**Cause**: GCS bucket doesn't exist or name is incorrect

**Fix**: Create the bucket first — see docs for gcloud command. Ensure no `gs://` prefix in config.

### "not authorized to write to bucket" or storage access denied

**Cause**: Service account lacks permissions on the GCS bucket

**Fix**: Grant `roles/storage.objectAdmin` on the bucket — see docs for gcloud command.

### "Error 409: Already Exists: Dataset"

**Cause**: Service account can't see the existing dataset due to missing permissions, so it tries to create it

**Fix**: This paradoxical error means the dataset exists but the service account can't list it. Grant `bigquery.dataEditor` and `bigquery.jobUser` at the project level.

### "accessDenied" or "403 Forbidden" on BigQuery

**Cause**: Missing BigQuery IAM roles

**Fix**: Ensure all three roles are granted: `bigquery.dataEditor`, `bigquery.jobUser`, `bigquery.readSessionUser`. See docs for gcloud commands.

### "Invalid credentials" or authentication failures

**Cause**: Service account key is invalid, incomplete, or malformed

**Fix**:
1. Regenerate the key
2. Ensure entire JSON content is included (including `{` and `}`)
3. Use YAML `|` for multiline credentials_json

### "BucketRegionError" or 301 status

**Cause**: GCS bucket is in a different region than the BigQuery dataset

**Fix**: Both must be in the same region. Check with gcloud — see docs. Recreate whichever is easier to move.

### Staging files accumulating in bucket

**Cause**: Materialization failed during "store" phase before cleanup

**Fix**: The connector normally cleans up staging files. If files accumulate, check logs for errors, fix the underlying issue, then manually delete old staging files.

### "jobInternalError" or "Error 400" from BigQuery

**Cause**: Transient BigQuery backend error

**Fix**: Estuary automatically retries these. If persistent, check BigQuery service status and logs for specific error details.

### Materialization stuck in PENDING

Wait 30-60 seconds — this is normal during shard assignment. If still stuck:

```bash
flowctl logs --task <TENANT>/<PATH>/materialize-bigquery --since 5m | jq 'select(.level == "error")'
```

## Related Skills

- `connector-disable-enable` — Pause/restart existing materializations
- `connector-delete-recreate` — Nuclear option for stuck materializations
- `estuary-logs` — Deep log analysis
- `estuary-catalog-status` — Status checking
