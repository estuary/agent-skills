---
name: estuary-catalog-history
description: View publication history for catalog resources — who changed what and when. Use when investigating timeline of changes, finding who modified a connector, or correlating changes across capture/collection/materialization chains. Use when user says "what changed", "who changed", "publication history", "catalog history", "when was it published", "timeline of changes", or "what happened to my connector".
---

# flowctl catalog history - Publication History for Catalog Resources

**Concepts**: Every change to a catalog resource (capture, materialization, collection) creates a publication. `catalog history` shows the full timeline of publications including who made the change, what changed, and the full spec at each point in time.

## Basic Usage

View publication history for any catalog resource:
```bash
flowctl catalog history --name <tenant/resource-name>
```

Examples:
```bash
# Capture history
flowctl catalog history --name acme/source-postgres

# Materialization history
flowctl catalog history --name acme/materialize-snowflake

# Collection history
flowctl catalog history --name acme/orders
```

## With Full Specs (`--models`)

Include the complete spec at each publication point. Requires json or yaml output:
```bash
flowctl catalog history --name acme/source-postgres --models -o json | jq
```

## Output Format

Output is **newline-delimited JSON** (NDJSON — one JSON object per line). Use `jq -s` to slurp into an array for filtering/sorting.

Fields per entry:
- `catalog_name` — resource name
- `catalog_type` — capture, materialization, collection, etc.
- `publication.detail` — human-readable change description (e.g. "Published via flowctl", "auto-discover", "periodic publication")
- `publication.publicationId` — unique publication ID for this entry
- `publication.publishedAt` — ISO 8601 timestamp
- `publication.userEmail` — who published
- `publication.userFullName` — display name of publisher
- `publication.userId` — UUID of publisher
- `publication.model` — full spec at that point (only with `--models`, otherwise null)

Example JSON output:
```json
{"catalog_name":"acme/materialize-snowflake","catalog_type":"materialization","publication":{"detail":"Published via flowctl","model":null,"publicationId":"119fa3bab9000078","publishedAt":"2025-07-01T15:58:30.115433Z","userEmail":"alice@acme.com","userFullName":"Alice Smith","userId":"3be2166e-28a2-440f-876c-a9c2259e413c"}}
```

## Common Detail Values

The `publication.detail` field can be `null`, empty string `""`, or a string. Details are often **multi-line** (separated by `\n`) combining a primary reason with specific actions taken.

**User-initiated:**
| Detail | Meaning |
|--------|---------|
| `null` or `""` | Manual publication via UI (no description) |
| `Published via flowctl` | User published via CLI |

**System-initiated:**
| Detail pattern | Meaning |
|----------------|---------|
| `auto-discover changes (N added, N modified, N removed)` | Auto-discover updated bindings |
| `auto-discover changes (...), and re-creating N collections` | Auto-discover with collection re-versioning |
| `updating inferred schema` | Inferred schema updated on a collection |
| `periodic publication` | Scheduled periodic publication |
| `in response to publication of one or more depencencies` | Cascade from upstream change (note: "depencencies" is a typo in the system) |
| `in response to change in dependencies, prev hash: ..., new hash: ...` | Dependency hash changed, materialization rebuilt |
| `adding binding(s) to match the sourceCapture: [...]` | sourceCapture auto-added bindings to materialization |
| `system created publication in response to discover: <id>` | System discover response |
| `Data_Flow_Reset : disable capture : <uuid>` | Dataflow reset initiated |

**Action suffixes** (appended after `\n` to the primary reason):
| Suffix | Meaning |
|--------|---------|
| `- updated resource /_meta of N bindings` | Resource metadata update |
| `- updated inferred schema` | Inferred schema was actually changed |
| `- backfilling binding ["schema", "table"] due to incompatible field selection (onIncompatibleSchemaChange: backfill)` | Binding backfilled due to schema incompatibility |
| `- backfilled binding of reset collection <name>` | Binding backfilled because its source collection was reset |
| `and applying onIncompatibleSchemaChange actions` | Schema change actions were applied |

## Prefix Queries

Use a catalog name prefix to get history across multiple related tasks at once:
```bash
# All resources sharing a prefix (capture + its collections)
flowctl catalog history --name acme/source-postgres
# Returns history for acme/source-postgres, acme/source-postgres/table1, etc.
```

**Note**: The prefix must be a valid catalog name prefix (no trailing slash). It matches any resource whose name starts with the given string.

## jq Recipes

> **Important**: Output is NDJSON (one object per line). For per-line transforms, jq works directly. For array operations (filtering, sorting), use `jq -s` to slurp into an array first.

### Filter by date range
```bash
flowctl catalog history --name acme/source-postgres -o json | \
  jq -s '[.[] | select(.publication.publishedAt >= "2025-01-01" and .publication.publishedAt <= "2025-02-01")]'
```

### Find who changed a task
```bash
flowctl catalog history --name acme/source-postgres -o json | \
  jq '{published: .publication.publishedAt, user: .publication.userEmail, detail: .publication.detail}'
```

### Extract publication details only
```bash
flowctl catalog history --name acme/source-postgres -o json | \
  jq '{name: .catalog_name, id: .publication.publicationId, at: .publication.publishedAt, detail: .publication.detail}'
```

### Find schema/config changes between publications
```bash
# Save specs from two most recent publications and diff them
flowctl catalog history --name acme/source-postgres --models -o json | \
  jq -s '.[0].publication.model' > /tmp/spec-new.json
flowctl catalog history --name acme/source-postgres --models -o json | \
  jq -s '.[1].publication.model' > /tmp/spec-old.json
diff /tmp/spec-old.json /tmp/spec-new.json
```

### Find auto-discover vs manual publications
```bash
# Auto-discover publications (system-initiated)
# Use length > 0 guard before test() to handle empty detail strings safely
flowctl catalog history --name acme/source-postgres -o json | \
  jq -s '[.[] | select(.publication.detail | length > 0 and test("auto-discover|evolved|periodic"))]'

# Manual publications (user-initiated)
flowctl catalog history --name acme/source-postgres -o json | \
  jq -s '[.[] | select(.publication.detail | length == 0 or test("auto-discover|evolved|periodic") | not)]'
```

## Real-World Investigation Patterns

### 1. Unexpected Backfill Investigation
A materialization backfilled unexpectedly — find the triggering change:
```bash
# Find backfill-related publications on the materialization
flowctl catalog history --name acme/materialize-snowflake -o json | \
  jq 'select(.publication.detail | length > 0 and test("backfill|incompatible")) | {at: .publication.publishedAt, detail: .publication.detail}'

# Check the source collection for schema changes around that timestamp
flowctl catalog history --name acme/orders -o json | \
  jq -s '[.[] | select(.publication.publishedAt >= "2025-01-15" and .publication.publishedAt <= "2025-01-16")]'

# Check the capture for auto-discover that triggered the schema change
flowctl catalog history --name acme/source-postgres -o json | \
  jq 'select(.publication.detail | length > 0 and test("auto-discover")) | {at: .publication.publishedAt, detail: .publication.detail}'
```
Common root causes: auto-discover detected type change (VARIANT → TEXT), field type widened (integer → string), manual destination table changes.

### 2. Auto-Discover Changed Something Unexpectedly
Auto-discover added/removed bindings or changed config without user intent:
```bash
# Find all auto-discover publications
flowctl catalog history --name acme/source-postgres -o json | \
  jq 'select(.publication.detail | length > 0 and test("auto-discover"))'

# Use --models to diff what changed between the auto-discover and previous publication
flowctl catalog history --name acme/source-postgres --models -o json | \
  jq -s '.[0].publication.model' > /tmp/after.json && \
flowctl catalog history --name acme/source-postgres --models -o json | \
  jq -s '.[1].publication.model' > /tmp/before.json && \
diff /tmp/before.json /tmp/after.json
```
Common root causes: connector returned incomplete discover response, resource paths changed, access revoked causing bindings to be removed.

### 3. Who Changed What and When (Audit Trail)
Need to understand who modified a connector and when:
```bash
# Full audit trail with user attribution
flowctl catalog history --name acme/source-postgres -o json | \
  jq '{at: .publication.publishedAt, who: .publication.userFullName, email: .publication.userEmail, detail: .publication.detail}'

# Find all changes by a specific user
flowctl catalog history --name acme/source-postgres -o json | \
  jq -s '[.[] | select(.publication.userEmail == "alice@acme.com")]'

# Filter OUT system changes to see only human-initiated publications
# Note: use "== ... | not" instead of "!=" — zsh escapes ! in double quotes
flowctl catalog history --name acme/source-postgres -o json | \
  jq -s '[.[] | select(.publication.userEmail == "support@estuary.dev" | not)]'
```
Note: `publication.userFullName` can be `null` for some users — always show `userEmail` as a fallback.

### 4. Endpoint Config / Credential Changes
Server address, credentials, or database settings changed:
```bash
# Use --models to compare config across publications
flowctl catalog history --name acme/source-postgres --models -o json | \
  jq -s '.[] | {at: .publication.publishedAt, who: .publication.userEmail, detail: .publication.detail}'
# Then diff individual models to find the specific config change
```
Common root causes: server address changed (prod → dev), credentials rotated, database name changed.

### 5. Binding State Changes (Disable/Enable/Remove)
Bindings were disabled, removed, or re-enabled unexpectedly:
```bash
# Find publications mentioning binding changes
flowctl catalog history --name acme/materialize-snowflake -o json | \
  jq 'select(.publication.detail | length > 0 and test("disable|enable|remove|binding"))'

# Compare binding counts across publications with --models
flowctl catalog history --name acme/source-postgres --models -o json | \
  jq '{at: .publication.publishedAt, detail: .publication.detail, bindings: (.publication.model.bindings | length)}'
```
Common root causes: `onIncompatibleSchemaChange: disableTask` auto-disabling, auto-discover removing bindings when connector returned incomplete response, user lost database access.

## Cross-Task Timeline Reconstruction

### Example workflow: "Why did my materialization backfill?"
1. `catalog history` on materialization → find backfill publication timestamp and `publicationId`
2. `catalog history` on source collection → find schema change around same timestamp
3. `catalog history` on capture → find auto-discover that triggered the schema change
4. Correlate timestamps: auto-discover → collection schema update → materialization backfill

## When to Use

- Investigating what changed and when for a catalog resource
- Finding who made a specific change (audit trail)
- Correlating changes across capture → collection → materialization chains
- Debugging unexpected backfills by tracing the publication timeline
- Comparing specs between publications to see exactly what changed
- Auditing auto-discover vs manual changes
- Investigating binding state changes (disable/enable/remove)
- Understanding endpoint config or credential changes

## Related Skills

- **estuary-catalog-status** — Check current status and runtime state (also has `controller_status.publications.history` in its JSON output)

## Tips

- Output is **NDJSON** (one JSON object per line) — use `jq -s` to slurp into an array for filtering/sorting
- Without `-s`, jq processes each line individually (useful for simple transforms like `jq '{at: .publication.publishedAt, detail: .publication.detail}'`)
- Default output format is YAML; use `-o json` for scripting
- `publication.detail` can be `null`, `""`, or a string — always guard `test()` with `length > 0` to handle both safely
- **zsh gotcha**: `!=` in jq gets escaped by zsh (`\!=`). Use `== "value" | not` instead
- Prefix queries are powerful for seeing the timeline across related resources
- Combine with `flowctl catalog status` to see both current state and change history
