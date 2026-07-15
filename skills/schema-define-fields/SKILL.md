---
name: schema-define-fields
description: Add or define specific fields in a collection's readSchema. Use when fields are missing from materialization field selection, pruned by complexity limit, need type refinement, need to be protected from future inference changes, or need to be required for downstream consumers. Use when user says "missing fields", "pruned fields", "field not showing up", "complexity limit", "define field in read schema", "restrict field type", or "protect field from inference".
---

# Define Fields in the Read Schema

Add or refine specific field definitions in a collection's readSchema. The readSchema controls what materializations and derivations see when reading from a collection.

**Docs:** https://docs.estuary.dev/guides/customize-materialization-fields/#pruned-fields

## When to Use

- **Pruned fields** — Fields missing from materialization field selection because schema inference hit `x-complexity-limit` (1,000 fields default, 10,000 per binding for captures that emit a `SourcedSchema` — most SQL captures, plus Salesforce-native, HubSpot-native, Kinesis, and other CDK connectors that opt in)
- **Type refinement** — Inference saw bad data and widened a field's type (e.g., `[string, null, array]`); you want to narrow it to `[string, null]` for downstream consumers
- **Define specific fields** — Ensure certain fields are always present and typed when materializations read the collection
- **Protect fields from inference changes** — Proactively define fields so future inference updates can't prune or retype them. Also prevents dataflow reset failures: a reset clears the inferred schema, and if a downstream materialization has `required` fields that only existed in inference, the publish will fail with `required projection X does not exist in collection Y`
- **Derivation shuffle key** — Change a field's effective read-side type so it works as a shuffle key

## Prerequisites

- Know the exact JSON path of the field(s)
- Know the field's data type
- `flowctl` authenticated

## Key Concepts

- The readSchema is what materializations and derivations see. Captures write to the writeSchema (use `schema-field-redaction` for write-side changes).
- Don't edit `flow://connector-schema` or `flow://inferred-schema` directly — auto-discover and inference will overwrite your changes. Add a custom entry alongside them using `allOf`.
- `allOf` is an intersection (AND). You can narrow types and add `required`, but you can't widen what inference already narrowed.

## Procedure

### Step 1: Pull Collection Specs

```bash
flowctl catalog pull-specs --name <tenant>/<collection-name>
```

### Step 2: Identify the Read Schema

Find the read schema file (e.g., `collection.read.schema.yaml`).

Typical structure:
```yaml
allOf:
  - $ref: flow://relaxed-write-schema
  - $ref: flow://inferred-schema
```

### Step 3: Add Custom Schema Entry

Add a third entry to `allOf` with your field definitions. You can include `properties:`, `required:`, or any other JSON Schema keyword inside this entry — all are AND'd with the inferred schema:

```yaml
allOf:
  - $ref: flow://relaxed-write-schema
  - $ref: flow://inferred-schema
  # Custom field definitions - survives inference updates
  - type: object
    properties:
      your_field:
        type: string
    required: [your_field]  # Optional — mark the field as required for downstream consumers
```

### Step 4: Publish Changes

```bash
flowctl catalog publish --source flow.yaml
```

## Use Case Examples

### Re-add a Pruned Field

Field was pruned by `x-complexity-limit` and isn't available in field selection:

```yaml
allOf:
  - $ref: flow://relaxed-write-schema
  - $ref: flow://inferred-schema
  - type: object
    properties:
      custom_field:
        type: string
```

### Pruned Nested Field (e.g., Jira project ID)

```yaml
allOf:
  - $ref: flow://relaxed-write-schema
  - $ref: flow://inferred-schema
  - type: object
    properties:
      fields:
        type: object
        properties:
          project:
            type: object
            properties:
              id:
                type: string
                format: integer
                minLength: 1
                maxLength: 16
```

### Restrict a Field's Type (Bad Data Cleanup)

Inference widened `last_payment_date` to `[string, null, array]` due to bad source data. Narrow it back:

```yaml
allOf:
  - $ref: flow://relaxed-write-schema
  - $ref: flow://inferred-schema
  - type: object
    properties:
      last_payment_date:
        type: ["string", "null"]
```

**Caveat:** If existing documents in the collection still have the invalid type (e.g., arrays), materialization will fail on those documents. Use `notBefore` on the materialization to skip the period with bad data if needed.

### Define Specific Fields

Ensure `id` and `status` are always present when materializations read:

```yaml
allOf:
  - $ref: flow://relaxed-write-schema
  - $ref: flow://inferred-schema
  - type: object
    properties:
      id:
        type: string
      status:
        type: string
    required: [id, status]
```

### Protect Fields from Inference Changes / Dataflow Reset

A dataflow reset clears the inferred schema. If a downstream materialization has `required` fields that only existed in inference, the publish fails with `required projection X does not exist in collection Y`. Defining those fields in the readSchema upfront means they survive the reset.

## Understanding x-complexity-limit

The complexity limit controls how many unique field locations schema inference tracks:

| Per-binding condition | Default Limit |
|-----------------------|---------------|
| Connector does not emit `SourcedSchema` for the binding (MongoDB, HTTP ingest, etc.) | 1,000 fields |
| Connector emits `SourcedSchema` for the binding (most SQL captures, Salesforce-native, HubSpot-native, Kinesis, opt-in CDK connectors) | 10,000 fields |

The limit is per-binding: only bindings whose capture emitted a `SourcedSchema` get 10,000. When reached, additional fields are pruned from the inferred schema — the data still exists in documents, but the fields aren't available for field selection. The limit itself is runtime-managed and cannot be raised by setting it in the read schema. Adding fields manually to the readSchema is the workaround.

## Verifying Field is Available

After publishing, check if the field appears in materialization:

1. Edit the materialization in UI
2. Select the binding
3. Go to Field Selection
4. Search for your added field

Or via flowctl:
```bash
flowctl catalog pull-specs --name <tenant>/<materialization-name>
# Check if field appears in fields.require options
```

## Troubleshooting

### "Cannot materialize a field with no types" / field forbidden

The type you declared has an empty intersection with what inference allows for that location. Two causes:

- **Declared type is disjoint from the data.** Your custom entry says `string` but inference saw `integer` — the AND of the two is empty. Declare a type that overlaps the actual data (a union like `["string","null"]` is fine).
- **The field has never appeared in any document.** This approach refines or re-surfaces fields that exist in the data (pruned fields, type narrowing). It can't conjure a field inference has never observed — when nothing extra has been seen, the inferred schema's `additionalProperties` is `false` and the intersection is empty. Capture at least one document containing the field first.

### Field Still Not Appearing

- Check JSON path is correct (use `/` for nesting: `/fields/project/id`)
- Verify the type matches actual data
- Ensure you edited the **readSchema**, not writeSchema
- Re-pull specs to confirm changes were published

### Type Mismatch Errors After Narrowing

- If narrowing a type, existing documents with the old type will fail validation
- Use `notBefore` on the materialization to skip documents from before the data was corrected
- For `format` constraints (e.g., `date-time`): `allOf` intersection means you can't remove a format that inference added — you'd need to copy the full inferred schema and edit it

### Changes Overwritten

- You likely edited the inferred schema directly
- Changes must be in the **third allOf entry**, not within `flow://inferred-schema`
- The inferred schema reference must remain untouched

### Field Path Unknown

To find the exact path:
1. Read raw documents from collection: `flowctl collections read --collection <name> --uncommitted | head -5`
2. Look at document structure to identify the JSON pointer path
3. Use `/` separator for nesting: `/top/nested/deep`

## Alternative Approaches

- **Materialize `flow_document`** — When many fields are pruned, materialize the top-level document and parse in the destination.
- **Reshape via derivation** — Emit only needed fields into a narrower collection, then materialize that.

## Related Skills

- `schema-null-defaults` — Add `default` annotations to fields. Note: that skill places `properties:` as a peer of `allOf:`, whereas this skill uses a third `allOf:` entry. Both are valid JSON Schema (sibling keywords are AND'd at the same level). The third-`allOf` pattern is preferred when defining new field structure; the peer-`properties` pattern is convenient for annotation-only changes like `default`.
- `schema-projections` — Rename fields or add projections
- `schema-field-selection` — Select specific fields in materializations
