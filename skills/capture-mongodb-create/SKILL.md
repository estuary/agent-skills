---
name: capture-mongodb-create
description: Create a MongoDB CDC capture using flowctl. Use when setting up real-time streaming from MongoDB Atlas, DocumentDB, or self-hosted MongoDB. Use when user says "capture MongoDB", "stream from Mongo", "MongoDB CDC", or "connect MongoDB to Estuary".
---

# Create MongoDB Capture

Create a MongoDB capture using flowctl to stream data from MongoDB collections into Estuary collections using Change Data Capture (CDC).

**Applies to**: source-mongodb, source-amazon-documentdb, source-azure-cosmos-db

## Step 0: Load Connector Documentation

Before proceeding, fetch the official connector docs for prerequisites, config reference, and deployment-specific setup.

**Always load the main page:**
https://docs.estuary.dev/reference/Connectors/capture-connectors/MongoDB/

**Then load the variant subpage based on the user's deployment:**

| Deployment | Docs URL |
|------------|----------|
| MongoDB Atlas | Main page covers this |
| Self-hosted MongoDB | Main page covers this |
| Amazon DocumentDB | https://docs.estuary.dev/reference/Connectors/capture-connectors/MongoDB/amazon-documentdb/ |
| Azure Cosmos DB | https://docs.estuary.dev/reference/Connectors/capture-connectors/MongoDB/azure-cosmosdb/ |

Use WebFetch to load these pages. Together they cover:

- Prerequisites (replica set requirement, user permissions)
- Full config property reference
- Capture modes (Change Stream Incremental, Batch Snapshot, Batch Incremental)
- SSH tunnel configuration
- Network access / IP allowlisting

This skill provides the **flowctl workflow** and **troubleshooting** that docs don't cover.

## Step 1: Gather Requirements

Before writing any YAML, ask the user:

1. **Deployment type?** — MongoDB Atlas, self-hosted replica set, Amazon DocumentDB, or Azure Cosmos DB
2. **Network path?** — Direct connection (Atlas/cloud with IP allowlist), SSH tunnel (private network), Private Link (AWS/Azure/GCP), or ngrok (local dev)
3. **Non-default data plane?** — Most users use the default. Ask if they need a non-default data plane.
4. **Database and collections?** — Which database, all collections or specific subset
5. **Capture mode?** — Change Stream Incremental (default CDC), Batch Snapshot, or Batch Incremental

**Critical check:** MongoDB CDC requires a replica set. Atlas and DocumentDB always are. Self-hosted standalone will NOT work — must be converted to replica set first.

## Step 2: Find the Correct Connector Version

Always use the latest numbered version tag. Query the connector registry to find it:

```bash
flowctl raw get --table connector_tags \
  --query 'documentation_url=ilike.*source-mongodb*' \
  --query 'select=image_tag,documentation_url' \
  --output yaml
```

Choose the connector image:

| Deployment | Connector Image |
|------------|----------------|
| MongoDB Atlas / Self-hosted | `ghcr.io/estuary/source-mongodb` |
| Amazon DocumentDB | `ghcr.io/estuary/source-mongodb` (same connector, different config) |
| Azure Cosmos DB | `ghcr.io/estuary/source-mongodb` (same connector, different config) |

## Step 3: Help User Complete Prerequisites

Walk the user through prerequisites from the docs loaded in Step 0:

1. **Replica set** — `rs.status()` should return replica set info, not an error
2. **User permissions** — needs `read` role on target database and `read` on `local` (for oplog)
3. **Oplog retention** — at least 24 hours recommended to avoid forced re-backfills
4. **Network access** — Estuary IPs allowlisted, or SSH tunnel configured

## Step 4: Create the Capture Spec File

Build `flow.yaml` using the config reference from the docs. Minimal required config:

```yaml
captures:
  <tenant>/<path>/source-mongodb:
    endpoint:
      connector:
        image: ghcr.io/estuary/source-mongodb:<version>
        config:
          address: "<connection_string>"
          database: "<database_name>"
          user: "<username>"
          password: "<password>"
    bindings: []
```

**Important:** The `user` and `password` fields are required even if MongoDB auth is disabled.

For SSH tunnel, add `networkTunnel.sshForwarding` block — see docs for full config.

### Connection String Formats

These are critical and easy to get wrong — not fully covered in docs:

```
# MongoDB Atlas (SRV)
mongodb+srv://cluster0.xxxxx.mongodb.net/?authSource=admin

# MongoDB Atlas (standard)
mongodb://shard-00-00.xxxxx.mongodb.net:27017,.../?ssl=true&replicaSet=atlas-xxxxx&authSource=admin

# Self-hosted (single node replica set)
mongodb://hostname:27017/?authSource=admin&directConnection=true

# Amazon DocumentDB
mongodb://docdb-cluster.xxxxx.us-east-1.docdb.amazonaws.com:27017/?ssl=true&replicaSet=rs0&retryWrites=false

# Via ngrok (local dev)
mongodb://0.tcp.ngrok.io:12345/?authSource=admin&directConnection=true
```

**Key parameters:**
- `authSource=admin` — required when user is defined in admin db
- `directConnection=true` — use for single-node connections
- `ssl=true` — required for Atlas and DocumentDB

## Step 5: Discover and Publish

```bash
# Discover collections
flowctl discover --source flow.yaml

# Review the generated bindings
cat flow.yaml

# Publish the capture
flowctl catalog publish --source flow.yaml --auto-approve
```

## Step 6: Verify

```bash
# Check status (expect PENDING → BACKFILLING → OK: Streaming Change Events)
flowctl catalog status <tenant>/<path>/source-mongodb

# View recent logs
flowctl logs --task <tenant>/<path>/source-mongodb --since 5m | jq -c '{ts, message}'

# Read captured data
flowctl collections read --collection <tenant>/<path>/<database>/<collection> --uncommitted | head -10
```

**Status progression:**
1. `PENDING` — normal for ~30 seconds during shard assignment
2. `BACKFILLING` — initial snapshot of collections
3. `OK: Streaming Change Events` — CDC running normally

## Troubleshooting

### "not a replica set" or "change stream not supported"

**Cause**: MongoDB is standalone, not a replica set

**Fix**: Atlas/DocumentDB are always replica sets. For self-hosted:
```bash
# Add to mongod.conf, restart, then:
mongo --eval "rs.initiate()"
```

### "not authorized" or "Authentication failed"

**Cause**: Invalid credentials or missing permissions

**Fix**:
1. Verify username/password
2. Check `authSource` parameter (usually `admin`)
3. Grant required roles:
```javascript
db.grantRolesToUser("flow_capture", [
  { role: "read", db: "target_database" },
  { role: "read", db: "local" }
])
```

### Missing `authSource=admin` in connection string

**Cause**: User authenticates against admin db but `authSource` not specified

**Fix**: Add `?authSource=admin` to connection string.

### "server selection error" or "no reachable servers"

**Cause**: Incorrect connection string or network issues

**Fix**:
1. Verify connection string format matches deployment type
2. For Atlas SRV records, ensure DNS resolution works
3. Check if SSL/TLS is required (`ssl=true`)

### "resume token not found" or forced re-backfill

**Cause**: Oplog rolled over while connector was paused/stopped

**Impact**: Connector must re-snapshot all data (happens automatically)

**Prevention**: Increase oplog size, keep retention at least 24 hours, don't pause captures for extended periods.

### "user and password are required"

**Cause**: Config missing user/password fields

**Fix**: Always include `user` and `password` even if MongoDB auth is disabled — use placeholder values.

### Understanding MongoDB "update" semantics

In MongoDB, deleting a field from a document appears as an "update" event, not a delete. The captured document reflects the new state without that field.

### Capture stuck in PENDING

Wait 30-60 seconds — this is normal during shard assignment. If still stuck:
```bash
flowctl logs --task <tenant>/<path>/source-mongodb --since 5m | jq 'select(.level == "error" or .level == "warn")'
```

## Related Skills

- `connector-disable-enable` — Pause/restart existing captures
- `connector-delete-recreate` — Nuclear option for stuck captures
- `estuary-logs` — Deep log analysis
- `estuary-catalog-status` — Status checking
