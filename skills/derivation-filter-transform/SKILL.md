---
name: derivation-filter-transform
description: Create an Estuary derivation for stateless filtering, field selection, and per-document field transformation. Use for WHERE clauses, projecting fields, computing new fields from existing ones, or data cleansing. Derivations add complexity and cost — confirm the user wants a derivation before reaching for this (renaming alone is usually a projection, not a derivation). Use when user says "filter-transform derivation", "derivation to filter by status", "transformation with WHERE clause", "derive computed fields", "cleanse data with a derivation", or "stateless derivation".
---

# derivation-filter-transform

Stateless Estuary derivation that filters documents and/or transforms field values. Each source document produces zero or one output document; no state, no cross-document coordination.

**Prereq:** read `derivation-basics` first for concepts, project layout, workflow, and language choice.

## When to use this over alternatives

- **Filtering** (`WHERE` clause): drop documents that don't match criteria
- **Computed fields**: derive new fields from existing ones (`CASE`, arithmetic, string manipulation, lookups)
- **Data cleansing**: normalise formats, trim whitespace, replace nulls
- **Selecting a subset of fields** when combined with the above

For a plain rename or nested-field flattening with no computation, use `schema-projections` — a projection doesn't require a derivation task.

## Canonical Example — SQL

Cleanse a `customers` collection for downstream analytics: drop inactive accounts and those with obviously invalid emails, normalise the email, and compute a `tier` bucket from lifetime spend.

### `schema.yaml`

The `writeSchema` (what the lambda must emit). Enum constraints on `tier` are enforced at write time; the `readSchema` (in `flow.yaml` below) layers in inference for everything else. See `derivation-basics` § Schema setup.

```yaml
type: object
properties:
  customer_id:
    type: string
  email_normalized:
    type: string
  signup_date:
    type: string
    format: date
  tier:
    type: string
    enum: [bronze, silver, gold]
required: [customer_id, email_normalized, tier]
```

### `flow.yaml`

```yaml
collections:
  acmeCo/analytics/active-customers:
    writeSchema: schema.yaml
    readSchema:
      allOf:
        - $ref: flow://write-schema
        - $ref: flow://inferred-schema
    key: [/customer_id]
    derive:
      using:
        sqlite: {}
      transforms:
        - name: cleanseCustomers
          source: acmeCo/production/customers
          shuffle: any
          lambda: |
            SELECT
              $id AS customer_id,
              LOWER(TRIM($email)) AS email_normalized,
              date($created_at) AS signup_date,
              CASE
                WHEN $lifetime_spend >= 10000 THEN 'gold'
                WHEN $lifetime_spend >= 1000  THEN 'silver'
                ELSE 'bronze'
              END AS tier
            WHERE $status = 'active'
              AND $email LIKE '%@%';
```

How it works:

- `$field` binds to the source document's JSON pointer `/field`. Nested fields use `$nested$field`, or `json_extract($obj, '$.path')` for deep paths.
- The `WHERE` clause drops rows that don't match — they produce zero output documents.
- Any per-row SQLite expression works: `CASE`, `LOWER`, `TRIM`, `date()`, arithmetic, string concatenation, `COALESCE`, `CAST`.

Preview and publish via the workflow in `derivation-basics`.

## Canonical Example — TypeScript

Same task in TypeScript. Reach for TS when the logic is too tangled for SQL: nested parsing, `try/catch` around dirty data, lookup maps, or splitting one input into many outputs. Here, TS lets us split `full_name` into `first_name` / `last_name` — awkward in SQLite.

### `schema.yaml`

```yaml
type: object
properties:
  customer_id:
    type: string
  email_normalized:
    type: string
  first_name:
    type: string
  last_name:
    type: string
  signup_date:
    type: string
    format: date
  tier:
    type: string
    enum: [bronze, silver, gold]
required: [customer_id, email_normalized, tier]
```

### `flow.yaml`

```yaml
collections:
  acmeCo/analytics/active-customers:
    writeSchema: schema.yaml
    readSchema:
      allOf:
        - $ref: flow://write-schema
        - $ref: flow://inferred-schema
    key: [/customer_id]
    derive:
      using:
        typescript:
          module: cleanse.ts
      transforms:
        - name: cleanseCustomers
          source: acmeCo/production/customers
          shuffle: any
```

### `cleanse.ts`

Generated with `flowctl generate --source flow.yaml`, then filled in:

```typescript
import { IDerivation, Document, SourceCleanseCustomers } from 'flow/acmeCo/analytics/active-customers.ts';

export class Derivation extends IDerivation {
  cleanseCustomers(_read: { doc: SourceCleanseCustomers }): Document[] {
    const c = _read.doc;

    // Filter: return [] to drop the document
    if (c.status !== 'active' || !c.email?.includes('@')) {
      return [];
    }

    const [first, ...rest] = (c.full_name ?? '').trim().split(/\s+/);
    const tier =
      c.lifetime_spend >= 10000 ? 'gold' :
      c.lifetime_spend >= 1000  ? 'silver' : 'bronze';

    return [{
      customer_id: c.id,
      email_normalized: c.email.toLowerCase().trim(),
      first_name: first || '',
      last_name: rest.join(' '),
      signup_date: c.created_at.split('T')[0],
      tier,
    }];
  }
}
```

Return `[]` to drop, `[{...}]` to emit one, `[{...}, {...}]` to emit many.

## Test

### Local validation — `flowctl preview --fixture` (works today)

Write newline-delimited JSON fixtures of source documents and run the derivation locally. Output can be diffed against expected:

`fixture.jsonl`:

```json
["acmeCo/production/customers", {"id": "c1", "status": "active",   "email": "Alice@Example.COM", "lifetime_spend": 12000, "full_name": "Alice Smith",  "created_at": "2024-01-15T10:00:00Z"}]
["acmeCo/production/customers", {"id": "c2", "status": "active",   "email": "bob@example.com",   "lifetime_spend": 2500,  "full_name": "Bob Jones",    "created_at": "2024-02-01T10:00:00Z"}]
["acmeCo/production/customers", {"id": "c3", "status": "active",   "email": "carol@example.com", "lifetime_spend": 50,    "full_name": "Carol Nguyen", "created_at": "2024-03-10T10:00:00Z"}]
["acmeCo/production/customers", {"id": "c4", "status": "inactive", "email": "dave@example.com",  "lifetime_spend": 5000,  "full_name": "Dave Kim",     "created_at": "2024-04-05T10:00:00Z"}]
["acmeCo/production/customers", {"id": "c5", "status": "active",   "email": "no-at-sign",        "lifetime_spend": 3000,  "full_name": "Eve Watson",   "created_at": "2024-05-20T10:00:00Z"}]
{"commit": true}
```

```bash
flowctl preview --source flow.yaml \
  --name acmeCo/analytics/active-customers \
  --fixture fixture.jsonl --timeout 30s
```

Expected output (c4 dropped as inactive, c5 dropped for missing `@`):

```json
{"customer_id":"c1","email_normalized":"alice@example.com","signup_date":"2024-01-15","tier":"gold"}
{"customer_id":"c2","email_normalized":"bob@example.com","signup_date":"2024-02-01","tier":"silver"}
{"customer_id":"c3","email_normalized":"carol@example.com","signup_date":"2024-03-10","tier":"bronze"}
```

### Spec-defined tests — `flowctl catalog test`

You can also declare tests in-spec with `tests:`. This is the documented approach and is useful in CI. `flowctl preview --fixture` is still handy for fast local iteration without round-tripping to the control plane.

```yaml
tests:
  acmeCo/tests/cleanse-customers:
    - ingest:
        description: Five customers covering each tier, plus an inactive account and an invalid email.
        collection: acmeCo/production/customers
        documents:
          - { id: "c1", status: "active",   email: "Alice@Example.COM", lifetime_spend: 12000, full_name: "Alice Smith",  created_at: "2024-01-15T10:00:00Z" }
          - { id: "c2", status: "active",   email: "bob@example.com",   lifetime_spend: 2500,  full_name: "Bob Jones",    created_at: "2024-02-01T10:00:00Z" }
          - { id: "c3", status: "active",   email: "carol@example.com", lifetime_spend: 50,    full_name: "Carol Nguyen", created_at: "2024-03-10T10:00:00Z" }
          - { id: "c4", status: "inactive", email: "dave@example.com",  lifetime_spend: 5000,  full_name: "Dave Kim",     created_at: "2024-04-05T10:00:00Z" }
          - { id: "c5", status: "active",   email: "no-at-sign",        lifetime_spend: 3000,  full_name: "Eve Watson",   created_at: "2024-05-20T10:00:00Z" }
    - verify:
        description: c4 dropped (inactive) and c5 dropped (no @ in email); remaining three land in their respective tiers with a normalised email.
        collection: acmeCo/analytics/active-customers
        documents:
          - { customer_id: "c1", email_normalized: "alice@example.com", signup_date: "2024-01-15", tier: "gold" }
          - { customer_id: "c2", email_normalized: "bob@example.com",   signup_date: "2024-02-01", tier: "silver" }
          - { customer_id: "c3", email_normalized: "carol@example.com", signup_date: "2024-03-10", tier: "bronze" }
```

For the TypeScript version, add `first_name` and `last_name` to each verify document (e.g. `first_name: "Alice", last_name: "Smith"`).

## Variations

- **Pass through whole document (filter-only):** `SELECT JSON($flow_document) WHERE $status = 'active';` — emits each matching source document verbatim at the top level, without listing fields. Estuary's equivalent of `SELECT *` (which SQLite derivations don't support).
- **Preserve deletes through a filter:** a `WHERE` on a business field drops delete events, because a delete carries only the key and `_meta` (the tested field binds as NULL). To keep hard deletes propagating downstream, exempt deletes from the filter and double-emit them in the same lambda:
  ```sql
  SELECT JSON($flow_document) WHERE $_meta$op != 'd' AND $status = 'active';
  SELECT $id, $_meta WHERE $_meta$op = 'd';
  SELECT $id, $_meta WHERE $_meta$op = 'd';
  ```
  The delete is emitted twice so the reduction has a prior document to act on; see `derivation-basics` § Handling deletes for the full mechanism.
- **Pass through + add/modify a field:** `SELECT json_set($flow_document, '$.newField', <expr>);` — `json_set`, `json_insert`, and `json_replace` each handle existing/new values differently (https://sqlite.org/json1.html#jins). Needed because `SELECT JSON($flow_document), extra_col` re-nests the document.
- **Filter only, explicit columns:** `SELECT $id, $name, $email WHERE $status = 'active';`
- **Extract nested values:** `$nested$field` in SQL, `doc.nested?.field` in TS. For deep paths in SQL: `json_extract($obj, '$.a.b.c')`.
- **Lookup/mapping:** in TS, define a `Record<string, string>` constant and look up values per document.
- **NULL handling:** `COALESCE($field, 'default')` in SQL, `doc.field ?? 'default'` in TS.
- **Proper boolean output:** SQLite stores booleans as 0/1. Use `json('true')` / `json('false')` (often inside a `CASE`) to emit a real JSON boolean.
- **Multi-output per input:** both languages support it. TS returns multiple documents from the array; SQL emits one document per output row, so `UNION ALL`, `json_each`, or recursive CTEs all work. (For array unnesting specifically, see `derivation-flatten-array`.)

## Gotchas specific to filter/transform

- **`WHERE` is evaluated per document, not per collection.** If source documents are updated in place, the filter re-runs on the update. A document that once matched may stop matching — the derivation will emit the new version (which may be empty), not delete the old. Downstream materialisations see the last emitted state.
- **Prefer `shuffle: any` for stateless transforms.** Everything in this skill is stateless, so any-shard routing is simplest and avoids unnecessary co-location overhead. Keyed shuffle is not an error here — just wasteful.
- **`JSON($flow_document)` vs `$flow_document` matters.** The `JSON()` wrapper makes the whole document become the output doc at the top level. Without it, the doc ends up nested under a `flow_document` property and schema validation fails with `"Missing required property: id"` — because `id` is now inside the nested object, not at the root.

## Related

- `derivation-basics` — prerequisite reading
- `schema-projections` — for plain renames without computation, no derivation needed
- `derivation-flatten-array` — when one input should produce many outputs (array unnesting)
- `derivation-aggregate-metrics` — when you need reductions (sum, min, max) across documents
- `derivation-join-collections` — when combining data across collections
