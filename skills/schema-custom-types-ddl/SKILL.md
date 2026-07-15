---
name: schema-custom-types-ddl
description: Override default column types in materializations using castToString or custom DDL. Use when column type needs different precision (e.g., NUMERIC(22,6)), VARCHAR length is wrong, need JSONB instead of JSON, integers lose precision, or need to force a string representation. Use when user says "change column type", "wrong precision", "VARCHAR too short", "integer truncated", "cast to string", or "custom DDL".
---

# Custom Column Types

Override default column types in materializations using `castToString` or custom `DDL`.

**Docs:** https://docs.estuary.dev/guides/advanced-usage/custom-column-types/

## When to Use

- Integer overflow or `value out of range` on large IDs → `castToString` or `DDL: "NUMERIC(...)"`.
- Monetary values lose precision as `DOUBLE` → `DDL: "NUMERIC(22,6)"`.
- Multi-type unions (`[integer, null, string]`) mapped to JSON when you want a string → `castToString`.
- Need a specific destination type (e.g., `JSONB` instead of `JSON`, custom VARCHAR length).

## Two Options

### castToString — Convert to String (Simpler)

Converts the value to its string representation before writing — numbers become `"3"`, booleans `"true"`, objects stringified JSON. More robust than `DDL: "STRING"` because it actually transforms the data.

```yaml
fields:
  recommended: true
  require:
    large_id: { castToString: true }
```

> Field configs go under `fields.require`. `include` is accepted as a legacy alias but is normalized to `require` on publish — if you write `include`, your next `pull-specs` will show `require`. Use `require` to avoid confusion.

### DDL — Custom Column Definition (Advanced)

Specifies the exact SQL column type for the destination. The DDL string is passed verbatim to the database.

**DDL only changes the column definition — it does NOT transform the data.** The burden is on you to ensure the field's data is compatible with the specified type. When DDL is set, the connector disables its normal type validation for that field.

```yaml
fields:
  recommended: true
  require:
    amount: { DDL: "NUMERIC(22,6)" }
    json_payload: { DDL: "JSONB" }
```

You can combine both: `{ castToString: true, DDL: "VARCHAR(100)" }` — value is converted to string, then stored in the custom column type.

## Prerequisites

- Existing materialization to modify
- For DDL: knowledge of destination database SQL types
- `flowctl` authenticated

## Procedure

### Step 1: Pull the Materialization Spec

```bash
flowctl catalog pull-specs --name <materialization-name>
```

### Step 2: Add Field Configuration

Find the binding and add `castToString` or `DDL` under `require` in the `fields` stanza:

```yaml
bindings:
  - resource:
      table: your_table
    source: tenant/your/collection
    fields:
      recommended: true
      require:
        # Cast to string (robust, transforms data)
        large_id:
          castToString: true

        # Custom DDL (advanced, column type only)
        amount:
          DDL: "NUMERIC(22,6)"
        big_integer:
          DDL: "NUMERIC(22,0)"
        json_payload:
          DDL: "JSONB"
```

Listing a field under `require` selects it (even if `recommended` wouldn't) and attaches the config. `recommended` still controls automatic selection of everything else.

### Step 3: Apply the Type Change

`castToString` and `DDL` only affect column definitions written when the destination column is **created**. They don't retroactively alter existing columns.

#### Adding a new field with custom DDL

Custom `DDL` is only applied when a column is written by `CREATE TABLE`. Behavior depends on whether the destination table already exists:

- **Table doesn't exist yet** (first publish of the binding): the new column is created with your DDL. Non-destructive, no backfill needed.
- **Table already exists**: the connector adds the column via `ALTER TABLE ADD COLUMN` using its **default type mapping** — a raw `DDL` override is *ignored* (e.g. a number lands as `double precision`, not `NUMERIC(8,2)`). (`castToString` changes the field's logical type to string rather than overriding column text, so it should still take effect on an added column — but verify the result in the destination if you rely on it.)

To get a custom DDL on a new column of an existing table, add the field first (default type), then apply the DDL using one of the paths in "Changing the DDL on an existing column" below.

```bash
flowctl catalog publish --source flow.yaml
```

#### Changing the DDL on an existing column

The standard support-recommended path is:

1. Add the new DDL/castToString annotation
2. Increment `backfill` on the affected binding
3. Publish

When the connector treats the type change as incompatible, it drops the destination table and recreates it under the same name with the new DDL. The existing rows are wiped and repopulated from the source collection.

**Warning**: Bumping `backfill` is destructive — rows in the affected table are cleared and rewritten, and downstream consumers (BI, reverse-ETL, replicated tables) will see either a gap or stale data during repopulation. Confirm with the user before incrementing `backfill`.

```yaml
bindings:
  - resource:
      table: your_table
    source: tenant/your/collection
    backfill: 1              # Increment to trigger
    fields:
      require:
        amount: { DDL: "NUMERIC(22,6)" }
```

**Caveat for same-base-type precision/width changes**: changes that stay within the same base type — e.g. `NUMERIC(22,6)` → `NUMERIC(14,4)`, or `VARCHAR(120)` → `VARCHAR(60)` — were observed in testing to leave the destination column type **unchanged** after a `backfill` bump: the connector `TRUNCATE`s and repopulates without altering the column. There's no error to flag the silent no-op. Always confirm the column type actually changed in the destination after a DDL backfill; if it didn't, fall back to one of the options below.

**If the endpoint already has `always_drop_tables_on_backfill` enabled** (see "Escape hatch" below), this precision-change pitfall does not apply — every backfill bump is a DROP+CREATE, so the new DDL takes effect immediately.

#### Fallback: manually alter the destination column

If the backfill above doesn't apply the new type (common for precision/width changes within the same base type), the customer can `ALTER COLUMN` directly in the destination database and then add the DDL annotation to keep Estuary in sync going forward.

```sql
-- In your destination, after confirming with the user:
ALTER TABLE your_schema.your_table ALTER COLUMN amount TYPE NUMERIC(22,6);
```

Caveat: if a future backfill is triggered (e.g., dataflow reset on the capture), the materialization may clobber your manual definition and reset to the connector's default. Keep the DDL annotation in the spec so it re-applies on rebuild.

#### Escape hatch: force a clean rebuild

If neither of the above works (e.g., the connector consistently leaves the column unchanged and a manual ALTER isn't acceptable), force a fresh table creation. **This is the heaviest option — confirm with the user first.** Either:

- **Manually `DROP TABLE` in the destination** and then bump `backfill` on the binding. The connector issues a fresh `CREATE TABLE` honoring the current DDL/castToString config. Affects only the dropped table.
- **Enable `always_drop_tables_on_backfill`** on the endpoint config and bump `backfill`:

  ```yaml
  endpoint:
    connector:
      config:
        # ...other endpoint config...
        advanced:
          feature_flags: always_drop_tables_on_backfill
  ```

  Caveat: this flag is endpoint-wide. Any binding you bump `backfill` on while it's enabled will drop+recreate.

These rebuilds clear the destination table. If the existing data can't be lost, reach out to support before proceeding.

#### Optional safety: `onIncompatibleSchemaChange` per binding

You can control what happens when Estuary detects an incompatible schema change on a specific binding. Set `onIncompatibleSchemaChange` per binding:

| Value | Behavior |
|-------|----------|
| `backfill` (default) | Automatically trigger a backfill (drop+recreate) |
| `abort` | Block the publication; nothing happens until the user resolves it |
| `disableBinding` | Disable just this binding; other bindings keep materializing |
| `disableTask` | Disable the whole materialization |

Common uses: prevent an auto-rebackfill loop on a binding the user wants to handle manually (`abort`), or isolate one binding's failures so they don't trip the whole task (`disableBinding`).

```yaml
bindings:
  - resource:
      table: your_table
    source: tenant/your/collection
    onIncompatibleSchemaChange: abort
    fields:
      # ...
```

### Step 4: Verify

Check the destination table to confirm the new column types. Use your destination's schema inspection commands (e.g., `\d table` in Postgres, `DESCRIBE TABLE` in Snowflake).

## Connector Support

**DDL** is supported by all SQL and warehouse materialization connectors.

**Iceberg connectors** do not support `DDL`. There are two variants and they take different overrides:
- `materialize-iceberg` (newer) — use `castToString: true`
- `materialize-s3-iceberg` (older) — use `ignoreStringFormat: true`

**castToString** is supported by most SQL and warehouse connectors. Not supported by Elasticsearch, MongoDB, and DynamoDB.

The DDL string is passed verbatim to the destination — use syntax valid for your specific database. See the [docs](https://docs.estuary.dev/guides/advanced-usage/custom-column-types/#ddl) for examples by destination.

## Troubleshooting

### DDL change not taking effect on an existing column
See "Changing the DDL on an existing column" in Step 3 — most common cause is the precision/width caveat where the connector TRUNCATEs without applying the new type. Fall back to manual `ALTER COLUMN`, manual `DROP TABLE` + backfill, or the `always_drop_tables_on_backfill` flag.

### Type Mismatch Errors with DDL
- DDL does not transform data, only the column definition
- Ensure the field's actual data is compatible with the specified type
- If you just need a string, use `castToString` instead — it's safer

### Materialization crashloops after publish with "cannot find encode plan"
A `DDL` that doesn't match the data's actual type passes publish-time validation (DDL disables that field's type check) but then fails at **commit time** on every transaction — the task goes into ERROR and retries forever. Example: `failed to encode args[N]: unable to encode 9007199254740993 into text format for varchar: cannot find encode plan` — an integer field given `DDL: "VARCHAR(...)"` with no value transformation.

Fix: combine `castToString` with the DDL so the value is converted to a string before it's written:
```yaml
big_id:
  castToString: true
  DDL: "VARCHAR(40)"
```

### Backfill Fails
- Check for data that doesn't fit the new DDL (e.g., string too long for new VARCHAR)
- Review materialization logs for specific error messages

## Related Skills

- `schema-field-selection` — Choose which fields to include/exclude in materializations
- `schema-projections` — Rename or restructure fields before materialization
- `schema-define-fields` — Define fields in the collection readSchema
