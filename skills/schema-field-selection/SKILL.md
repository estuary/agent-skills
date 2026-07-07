---
name: schema-field-selection
description: Control which fields are included or excluded in materializations using depth modes and field overrides. Use when reducing table width, including nested fields, excluding PII, or optimizing performance. Use when user says "exclude fields", "include nested field", "too many columns", "select specific fields", or "field depth".
---

# Field Selection for Materializations

Control which fields are included or excluded in materializations using depth modes and field overrides.

**Docs:** https://docs.estuary.dev/guides/customize-materialization-fields/

## When to Use

- Reduce table width by excluding unnecessary fields
- Include deeply nested fields that aren't selected by default
- Exclude sensitive fields from materializations (note: data still exists in the collection — for true PII protection, use redaction on the writeSchema instead)
- Override connector's default field recommendations

## Procedure

### Step 1: Pull the Materialization Spec

```bash
flowctl catalog pull-specs --name <materialization-name>
```

### Step 2: Configure Field Selection

Add or modify the `fields` stanza in the binding:

```yaml
bindings:
  - source: tenant/your/collection
    resource: { table: your_table }
    fields:
      recommended: true    # Depth mode (see below)
      require:             # Object — explicitly include fields
        deeply/nested/field: {}
        another_field: {}
      exclude:             # Array — explicitly exclude fields
        - sensitive_data
        - debug_info
```

**Note:** `require` is an object (`field: {}`), `exclude` is an array (`- field`).

**Note:** Each connector marks certain fields as required — typically the root document (`flow_document`) and the collection-key components, and some connectors require additional fields (e.g., CDC-style materializations that depend on `_meta/op`). Attempting to exclude a connector-required field returns a `LOCATION_REQUIRED` / `FIELD_REQUIRED` constraint and the publish fails. The exact list comes from each connector's `Validate` response.

### Depth Modes

The `recommended` setting controls how deeply nested fields are automatically selected. UI labels are in parentheses:

| Value | UI Label | Description |
|-------|----------|-------------|
| `0` | Required Only | Selects only fields the connector marks required (root document + key components, plus any connector-specific required fields) |
| `1` | Depth 1 (default) | Top-level scalar fields |
| `2` | Depth 2 | Two levels of nesting |
| `true` | Unlimited Depth | All scalar fields at any depth |

The boolean `recommended: false` is also accepted and behaves like `0` in practice, but the explicit numeric values are documented and preferred. Use `recommended: 0` with `require:` to hand-pick a small set of fields.

### Step 3: Publish Changes

```bash
flowctl catalog publish --source flow.yaml
```

### What happens to the destination on publish

Most field selection changes apply non-destructively, no backfill needed:

- **Adding to `require` (or raising depth)** → connector emits `ALTER TABLE ADD COLUMN` automatically. Existing data is preserved; historical rows show `NULL` for the new column unless you also backfill.
- **Adding to `exclude` (or lowering depth) on a column that already exists** → connector drops the `NOT NULL` constraint, stops writing to that column, and **leaves the column and its existing data in place**. This is by design: per the [schema evolution docs](https://docs.estuary.dev/guides/schema-evolution/#a-field-was-removed), "the existing column will not be dropped, and will just be ignored by the materialization going forward."

**If the endpoint has `always_drop_tables_on_backfill` enabled** (an opt-in feature flag, off by default), this snapshot-preservation behavior is overridden whenever you bump `backfill` on *any* binding — the affected table is dropped and recreated from scratch, and excluded columns are gone. If you have the flag enabled and want to keep an old column around, avoid bumping `backfill` on that binding.

For the standard case (flag not set), an excluded column staying in place is usually the right outcome — downstream queries can keep reading the snapshot value, and you avoid any destructive operation. Just publish:

```bash
flowctl catalog publish --source flow.yaml
```

### Removing an excluded column from the destination

The default behavior leaves an excluded column in place. If you specifically need to physically remove it — for PII purging, schema cleanup, or freeing storage — pick one of the options below.

**Warning**: Each option below alters or destroys data already in the destination. Confirm with the user before any of them, and check whether downstream consumers (BI dashboards, reverse-ETL, replicated tables) reference the column.

**Option A — Manually drop the column (preferred for large tables):**

```sql
-- In your destination, after confirming with the user:
ALTER TABLE your_schema.your_table DROP COLUMN sensitive_data;
```

Then publish your spec with the field in `exclude`. The connector keeps writing the remaining columns; nothing else is touched. This is what support typically recommends for large tables where a full rebuild would be expensive.

**Option B — Force a full table rebuild via `always_drop_tables_on_backfill`:**

If you need multiple columns dropped at once or want the destination schema to exactly match the current field selection, enable the endpoint feature flag and bump `backfill` on the binding. The connector will issue a fresh `CREATE TABLE` honoring only the selected fields.

```yaml
endpoint:
  connector:
    config:
      # ...other endpoint config...
      advanced:
        feature_flags: always_drop_tables_on_backfill
```

**Caveat**: this flag is endpoint-wide. Any binding you bump `backfill` on while it's enabled will drop+recreate its table. If you only want one binding affected, prefer Option A or do a one-off manual `DROP TABLE` of that specific table before bumping `backfill`.

Connectors that always drop+recreate on backfill (no flag or manual drop needed): MongoDB, DynamoDB, Elasticsearch, and both Iceberg variants.

## Troubleshooting

### Field Not Appearing in Destination
- Verify the field exists in the collection schema and isn't in `exclude`
- Try adding it to `require` explicitly, or raise the depth mode
- Field names are case-sensitive; nested fields use `/` (e.g., `_meta/op`)

## Related Skills

- `schema-custom-types-ddl` — Customize column types for included fields
- `schema-projections` — Rename fields before materialization
- `schema-field-redaction` — Hash or block sensitive fields at capture time (more secure than excluding)
