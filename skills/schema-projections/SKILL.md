---
name: schema-projections
description: Add projections to collections to rename fields or map nested JSON paths to flat column names. Use when renaming fields for destination conventions, flattening nested paths to columns, or creating field aliases. Use when user says "rename field", "flatten nested JSON", "map JSON path to column", "field alias", or "projection".
---

# Projections

Add projections to collections to rename fields or map nested JSON paths to flat column names.

**Docs:** https://docs.estuary.dev/concepts/advanced/projections/

## When to Use

- Rename fields to match destination naming conventions
- Flatten nested JSON paths to column names (e.g., `/user/name` → `user_name`)
- Extract array elements as named fields

## Procedure

### Step 1: Pull the Collection Spec

```bash
flowctl catalog pull-specs --name <collection-name>
```

### Step 2: Add Projections

Add a `projections` stanza to the collection spec (top-level, alongside `key` and schema):

```yaml
collections:
  tenant/your/collection:
    schema: schema.yaml
    key: [/id]
    projections:
      # Rename a top-level field
      user_id: /id

      # Flatten nested paths
      customer_name: /customer/name
      customer_email: /customer/email

      # Extract array elements
      primary_tag: /tags/0
```

Values are JSON Pointers: `/` separated, array indices numeric (`/tags/0`).

Fields from your schema already show up as columns in materializations automatically. You only need projections when you want to rename a field or expose a nested path as a different column name.

### Step 3: Publish Changes

```bash
flowctl catalog publish --source flow.yaml
```

New field names become available in materialization field selection. Add them to `fields.require` in the materialization binding to include them.

## Notes

- Projections are also used for logical partitions (`partition: true`) — see [docs](https://docs.estuary.dev/concepts/advanced/projections/#logical-partitions) for that use case
- If the underlying field is pruned by the complexity limit, the projection survives but its type may widen (e.g., to JSON) — define the field in the readSchema to pin the type

## Troubleshooting

### Projection Not Appearing in Materialization
- Ensure JSON pointer path exists in the schema
- Add projected field to `fields.require` in materialization binding
- Projections are defined on the collection, not the materialization

### Invalid JSON Pointer
- Uses `/` not `.` for path separation
- Array indices are numeric: `/tags/0` not `/tags[0]`
- Escape `~` as `~0` and `/` as `~1` in field names

## Related Skills

- `schema-field-selection` — Choose which projected fields to materialize
- `schema-custom-types-ddl` — Customize column types for projected fields
- `schema-field-redaction` — Hash or block fields at capture time
