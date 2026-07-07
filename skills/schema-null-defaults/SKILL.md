---
name: schema-null-defaults
description: Handle NULL values and set default values for missing fields in collections and materializations. Use when fields need defaults, nullable primary key errors occur, or source data has gaps. Use when user says "NULL values", "default value", "missing field", "nullable primary key error", or "fill in missing data".
---

# Handle NULLs and Default Values

Handle NULL values and set default values for missing fields in materializations.

**Docs:** https://docs.estuary.dev/concepts/schemas/#default-annotations

## When to Use

- Need to provide default values for missing/optional fields
- Handling nullable primary key errors in materializations (`cannot materialize collection with nullable key field unless it has a default value annotation`)
- Ensuring consistent values when source data has gaps

## Prerequisites

- Existing collection to modify
- `flowctl` authenticated

## Key Concepts

- `default` applies only to **missing** fields, not to explicit `null` values. A field present with `null` stays `null`.
- `default` is read by materializations only — captures and derivations ignore it. Add defaults to the readSchema so auto-discover (which only touches writeSchema/connector-schema) can't overwrite them.

## Procedure

### Step 1: Pull Collection Specs

```bash
flowctl catalog pull-specs --name <tenant>/<collection-name>
```

### Step 2: Identify the Read Schema

Find the read schema file (e.g., `collection.read.schema.yaml`).

Typical structure for collections using schema inference:
```yaml
allOf:
  - $ref: flow://relaxed-write-schema
  - $ref: flow://inferred-schema
```

### Step 3: Add Default Values

Add `default` annotations at the top level of the readSchema, alongside `allOf`:

```yaml
allOf:
  - $ref: flow://relaxed-write-schema
  - $ref: flow://inferred-schema
properties:
  status:
    type: string
    default: "pending"
  count:
    type: integer
    default: 0
  is_active:
    type: boolean
    default: true
```

The `properties` block with defaults sits alongside the `allOf`, not inside it.

### Step 4: Publish Changes

```bash
flowctl catalog publish --source flow.yaml
```

## Common Patterns

### Default for Optional String Field

```yaml
allOf:
  - $ref: flow://relaxed-write-schema
  - $ref: flow://inferred-schema
properties:
  category:
    type:
      - string
      - "null"
    default: "uncategorized"
```

Note: Default only applies when field is missing. If value is explicitly `null`, it remains `null`.

### Nullable Primary Key with Default

The most common use case. SQL materializations reject key fields that are nullable unless they have a default. This often happens when the inferred schema doesn't mark the field as `required` (e.g., because it was absent in historical data).

```yaml
allOf:
  - $ref: flow://relaxed-write-schema
  - $ref: flow://inferred-schema
properties:
  year:
    default: 0
```

### Timestamp Default

```yaml
allOf:
  - $ref: flow://relaxed-write-schema
  - $ref: flow://inferred-schema
properties:
  created_at:
    type: string
    format: date-time
    default: "1970-01-01T00:00:00Z"
```

## Handling Explicit NULLs

`default` doesn't replace explicit `null` values, only missing fields. If you can't fix the source data, use a SQL derivation to replace nulls and materialize the derived collection instead:

```yaml
collections:
  tenant/derived/collection:
    schema: schema.yaml      # schema + key for the derived collection
    key: [/id]
    derive:
      using:
        sqlite: {}
      transforms:
        - name: replaceNulls
          source: tenant/source/collection
          shuffle: any
          lambda: SELECT COALESCE($status, 'unknown') AS status, $id AS id;
```

## Troubleshooting

### Default Not Applied

- Check if field is **missing** vs explicitly **null**
- Defaults only apply to missing fields
- Verify default is in **readSchema**, not just writeSchema
- Defaults are only used by materializations — ignored by captures and derivations

### Nullable PK Error in Materialization

```
cannot materialize collection 'X' with nullable key field 'Y' unless it has a default value annotation
```

- Add `default` annotation to the key field in readSchema (alongside `allOf`, not inside it)
- This can happen even if the field looks non-nullable in the current schema — the inferred schema may have seen it as nullable historically
- Alternatives: collection reset (`reset: true`) to re-infer from scratch, or add the field to `required` in the readSchema

### Schema Validation Errors After Adding Default

- Ensure default value matches the field's type
- For typed fields, default must be valid: `default: 0` for integer, not `default: "0"`

### Existing NULL Data Causing Issues

- Defaults don't transform existing null values
- Use a derivation to replace nulls (see "Handling Explicit NULLs" above)
- Or perform a Dataflow reset after fixing source data

## Related Skills

- `schema-field-selection` — Control which nullable fields materialize
- `schema-define-fields` — Define fields in the readSchema (pruned fields, type refinement, etc.). Note: that skill adds new field definitions as a third entry inside `allOf:`, while this skill places `properties:` as a peer of `allOf:`. Both are valid JSON Schema (sibling keywords are AND'd at the same level). The peer-`properties` pattern is convenient for annotation-only changes like `default`; the third-`allOf` pattern is preferred when introducing new field structure.
