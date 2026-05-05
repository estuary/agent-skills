---
name: capture-generic-create
description: Create a capture for ANY Estuary connector using dynamic schema discovery. Use when the user wants to capture from a source that doesn't have a dedicated skill (e.g., Kafka, Salesforce, HubSpot, Stripe, S3, GCS, Kinesis, or any of the 148+ connectors). Use when user says "capture from <source>", "stream from <source>", "connect <source> to Estuary", or "set up <source> capture" and no connector-specific skill exists.
---

# Create Capture (Any Connector)

Create a capture for any Estuary source connector by dynamically discovering the connector's config schema, documentation, and version from the connector registry. This skill works for all 148+ capture connectors.

**Check for a dedicated skill first:** Before using this generic workflow, check if a dedicated `capture-<connector>-create` skill exists for the user's data source. Dedicated skills have connector-specific prerequisite walkthroughs and troubleshooting that this generic skill can't provide. Only use this skill if no dedicated one exists.

## Step 0: Identify the Connector

Ask the user what data source they want to capture from, then search the connector registry:

```bash
flowctl raw get --table connector_tags \
  --query 'image_name=ilike.*<search-term>*' \
  --query 'protocol=eq.capture' \
  --query 'select=image_name,image_tag,documentation_url,endpoint_spec_schema' \
  --query 'order=image_tag.desc' \
  --query 'limit=5'
```

Replace `<search-term>` with a keyword from the user's request (e.g., `kafka`, `salesforce`, `hubspot`, `stripe`).

**If multiple matches are returned:**
- Show the user the options (image name + latest tag) and ask which one they want
- Some connectors have cloud-specific variants (e.g., `source-kafka` vs `source-amazon-msk`)

**If no matches are returned:**
- Try broader search terms (e.g., `s3` instead of `aws-s3`)
- Try searching by the service name without `source-` prefix

**Extract from the result:**
- `image_name` + `image_tag` — the connector image and latest version
- `documentation_url` — link to connector docs
- `endpoint_spec_schema` — JSON Schema for the connector's config (this drives Steps 2-3)

**Version selection:** Use the highest numbered tag (e.g., `:v5` over `:v3`). The `order=image_tag.desc` with `limit=5` gives you the latest versions. Pick the highest `:vN` tag — ignore `:dev` or `:local` tags.

## Step 1: Load Documentation + Tribal Knowledge

### 1a. Fetch connector documentation

Use WebFetch on the `documentation_url` from Step 0:

```
WebFetch(<documentation_url>)
```

This page covers:
- Prerequisites (what the user needs to set up on the source side)
- Config property reference
- Cloud-specific variants and setup
- Network access requirements

**Note on legacy batch connectors:** ~36 batch connectors have opaque `go.estuary.dev/<hash>` documentation URLs. These still redirect to valid docs pages — WebFetch will follow the redirect. If the page content is unhelpful, check the `endpoint_spec_schema.title` field to identify the connector.

### 1b. Search Kapa for tribal knowledge (if available)

If the Estuary MCP is configured, search for real-world setup issues and pitfalls:

```
Search kapa ai knowledge sources for "<connector-name> capture setup"
Search kapa ai knowledge sources for "<connector-name> errors troubleshooting"
```

Kapa results supplement docs with:
- Common configuration mistakes other users have made
- Workarounds for known issues
- Tips not covered in official documentation

If the Estuary MCP is not configured, the user (or you) can set it up by following the Estuary MCP integration guide: https://docs.estuary.dev/features/mcp-integration/ — Set up the Estuary MCP to enable knowledge base search.

## Step 2: Gather Requirements

### Standard questions (all captures):

1. **Network path?** — Direct connection, SSH tunnel, Private Link, or ngrok (local dev)
2. **Non-default data plane?** — Most users use the default. Only ask if they mention it.
3. **Tenant prefix and capture name?** — e.g., `acmeCo/production/source-kafka`

### Connector-specific questions (driven by `endpoint_spec_schema`):

Parse the `endpoint_spec_schema` JSON Schema to identify what the connector needs:

1. **Required fields** — Look at `required` array and each property's `description` to form clear questions
2. **Authentication variants** — Look for `oneOf` or `anyOf` in credential/auth fields. These indicate multiple auth methods (e.g., password vs. IAM vs. OAuth). Ask which method the user wants.
3. **Enum fields** — Fields with `enum` values indicate a fixed set of options. Present these as choices.
4. **Cloud variant config** — Some connectors behave differently for managed vs. self-hosted (e.g., RDS vs. on-prem). The docs from Step 1 will clarify these.

**Important:** Don't just dump the schema at the user. Translate it into clear, conversational questions using the field descriptions and docs context.

## Step 3: Build Config from Schema

Construct the connector config using the collected values:

1. Map user answers to the schema's required fields
2. Include optional fields only when the user provided values or the docs indicate they're commonly needed
3. For nested objects (common in auth config), build the correct structure from the schema

### SSH Tunnel Config (if needed)

If the user needs an SSH tunnel, add the standard `networkTunnel.sshForwarding` block:

```yaml
networkTunnel:
  sshForwarding:
    sshEndpoint: "<bastion-host>:<port>"
    privateKey: "<ssh-private-key>"
    forwardHost: "<database-host>"
    forwardPort: <database-port>
    user: "<ssh-user>"
```

## Step 4: Create flow.yaml

Build the spec with collected values + discovered image tag:

```yaml
captures:
  <tenant>/<path>/<capture-name>:
    endpoint:
      connector:
        image: <image_name>:<image_tag>
        config:
          # ... fields from Step 3, structured per endpoint_spec_schema
    bindings: []
```

Write this to `flow.yaml` in the working directory.

## Step 5: Discover, Customize, and Publish

### 5a. Discover available bindings

```bash
flowctl discover --source flow.yaml
```

**Note:** The first discover attempt occasionally fails with a generic error. If this happens, retry — it typically succeeds on the second attempt.

This connects to the source and populates `flow.yaml` with bindings (one per discovered table/stream/object).

### 5b. Review and customize bindings

After discovery, review the generated bindings in `flow.yaml`:

- **Remove unwanted bindings** — Delete binding entries for tables/streams the user doesn't need
- **Rename target collections** — Change the `target` field if needed
- **Trigger re-backfill** — Add `backfill: 1` to a binding's resource to force a fresh backfill

### 5c. Publish

```bash
flowctl catalog publish --source flow.yaml --auto-approve
```

The `--auto-approve` flag is required for non-interactive use.

## Step 6: Verify

```bash
# Check capture status
flowctl catalog status <tenant>/<path>/<capture-name>

# View recent logs
flowctl logs --task <tenant>/<path>/<capture-name> --since 5m | jq -c '{ts, message}'

# Read captured data (pick one collection from the bindings)
flowctl collections read --collection <tenant>/<path>/<collection> --uncommitted | head -5
```

**Status progression:**
1. `PENDING` — normal for ~30-60 seconds during shard assignment
2. `BACKFILLING` — initial data sync (for connectors that support backfill)
3. `OK` / streaming status — capture is running normally

**Note on batch connectors:** Connectors with `disable_backfill: true` in their schema don't follow the CDC "backfill then stream" pattern. They poll on a schedule instead. Their steady-state status may differ.

## Troubleshooting

### Generic Issues (common across all captures)

| Issue | Cause | Fix |
|-------|-------|-----|
| Capture stuck in PENDING | Storage mapping / shard assignment delay | Wait 60s, then check logs for errors |
| Connection refused / timeout | Source not reachable from Estuary cloud | Check firewall rules, Estuary IP allowlist, or use SSH tunnel |
| Authentication failed | Wrong credentials or insufficient permissions | Verify credentials, check user grants match docs prerequisites |
| SSL/TLS errors | Certificate mismatch or wrong SSL mode | Try `require` mode first, check connector docs for SSL config |
| Backfill taking long time | Large initial dataset | Normal for first sync; monitor progress via logs |
| Schema drift errors | Source schema changed after capture creation | Re-discover to pick up new schema, may need backfill increment |
| "connector {} does not exist" | Wrong image name or tag | Re-query `connector_tags` API for correct image (Step 0) |
| Recovery log errors | Corrupted state from previous failed attempt | Delete and recreate the capture |
| "Config failed schema validation" | Config structure doesn't match schema | Compare config against `endpoint_spec_schema` from Step 0 |
| Discover fails on first attempt | Intermittent connection issue | Retry `flowctl discover --source flow.yaml` — usually works on second try |

### Diagnosing with logs

```bash
# Show errors and warnings only
flowctl logs --task <tenant>/<path>/<capture-name> --since 10m | jq 'select(.level == "error" or .level == "warn")'

# Show all logs with timestamps
flowctl logs --task <tenant>/<path>/<capture-name> --since 5m | jq -c '{ts, level, message}'
```

### Connector-Specific Troubleshooting

If the generic table doesn't resolve the issue:

1. **If Kapa MCP is available**, search for the specific error:
   ```
   Search kapa ai knowledge sources for "<connector-name> <error-message>"
   ```
2. **Re-read the connector docs** loaded in Step 1 — check for known limitations or requirements you may have missed
3. **Check the connector's GitHub issues** for known bugs:
   ```bash
   gh search issues "<connector-name>" --repo estuary/connectors --state open
   ```

## Related Skills

- `connector-disable-enable` — Pause/restart existing captures
- `connector-delete-recreate` — Nuclear option for stuck captures
- `estuary-logs` — Deep log analysis
- `estuary-catalog-status` — Status checking
