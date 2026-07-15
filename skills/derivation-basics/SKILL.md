---
name: derivation-basics
description: Foundation for writing Estuary derivations — what they are, language choice (SQL/TypeScript/Python), project layout, flowctl workflow, stateless vs stateful shuffling, and universal gotchas. Read this before any other derivation-* skill. Use when user says "write a derivation", "how do derivations work", "create a derivation", "derivation setup", or before starting any transformation work.
---

# derivation-basics

Foundation for every Estuary derivation. Each `derivation-*` skill assumes you've read this and jumps straight to its specific use case.

**Docs:** https://docs.estuary.dev/concepts/derivations/ — canonical concept page. This skill distills what you actually need to get started.

## What a derivation is

A derivation is a collection produced by continuously transforming one or more source collections. It consists of:

- An **output collection** with its own schema and key
- A **catalog task** running one or more **transforms**. Each transform invokes a **lambda** — the user-written function that maps a source document to zero, one, or many output documents. In SQLite, the lambda is a block of SQL (written inline as `lambda:` or referenced from a `.sql` file). In TypeScript and Python, the lambda is a method on a class and the spec references the file via `module:` instead.
- Optional **internal state**: SQLite tables declared in **migrations** (SQL statements run once at derivation startup to create the state tables), or reduction annotations in the output schema, used for aggregations, joins, and windowing

Derivations run continuously and keep up with source updates in real time. They are not batch jobs.

## When NOT to use a derivation

The biggest footgun with derivations is misfit, not implementation. They are the wrong tool for:

- **dbt-style multi-step SQL DAGs.** dbt runs against a destination snapshot and expresses arbitrary multi-CTE pipelines. A derivation processes one document at a time as it streams in, with no global view of the dataset. Use dbt for DAGs that run against the materialised destination; reserve derivations for streaming logic that has to happen before materialisation.
- **Massive joins where one side is unbounded.** Each transform maintains per-shard state in RocksDB on the reactor. Joining a large fact stream against an unbounded keyspace can balloon state to tens of GB and cause reactor-wide disk pressure (there's no enforced cap). If both sides are unbounded, do the join at the destination.
- **One-shot backfills or report generation.** A SQL query against the materialised destination is simpler. Derivations earn their complexity by running continuously over a stream.
- **External lookups during processing.** Derivations can't call external APIs (the one exception is `derivation-python` on a private data plane, with caveats), can't query other databases, and can only read source collection documents as they arrive.
- **Remapping nested paths to flat columns at the destination.** Use [projections](https://docs.estuary.dev/concepts/advanced/projections/) to flatten depth into columns ({a:{b:1}} → `a_b` column) or rename a field for a destination. No derivation needed.

For nested-object extraction into new rows (one source doc with `line_items: [...]` or nested children → many derived docs), use `derivation-flatten-array`. If the use case doesn't fit one of the `derivation-*` skills cleanly, a derivation may not be the right tool.

## Language choice

| Language | Best for | Notes |
|----------|----------|-------|
| **SQLite** | Filtering, projections, reductions, stateful logic expressible in SQL | Default choice. `lambda` inline in `flow.yaml` or in a separate `.sql` file. |
| **TypeScript** | Complex logic, nested parsing, lookup tables, multi-output per input | Strongly typed via generated interfaces. Runs on Deno. |
| **Python** | ML pipelines, async operations, Pydantic models | **Private / BYOC data planes only** — not available on shared infrastructure. See https://docs.estuary.dev/private-byoc/. |

## Project layout

```
my-derivation/
├── flow.yaml          # Collection spec + derive stanza
├── schema.yaml        # Output schema (or inline in flow.yaml)
└── transform.ts       # TypeScript transform class (TS only)
    or transform.sql   # Separate lambda file (SQLite, optional — simple lambdas can stay inline)
    or transform.py    # Python transform class (Python only)
```

Filenames are arbitrary path references from `flow.yaml`. Keep `flow.yaml` itself — tooling looks for it by default, though `--source path/to/file.yaml` overrides.

For TypeScript, `flowctl generate` additionally produces a `flow_generated/` directory with typed interfaces and a `deno.json` for editor tooling. These are safe to commit but not required.

## Schema setup

A derivation has two schemas: the source collection's (what your lambda reads) and the derived collection's output (what your lambda emits). Each is set up differently.

### Source schema

The source collection's schema lives in its own spec. If the source came from a capture, its `writeSchema` is typically inferred from observed data and may appear as `$ref: flow://inferred-schema` in pulled specs — that's normal, not a problem to fix. See [schema inference](https://docs.estuary.dev/concepts/schemas/#inference) for the full mechanics.

To see what fields the source actually has before writing your lambda:

```bash
flowctl catalog pull-specs --name acmeCo/production/customers
```

For TypeScript and Python, also run `flowctl generate --source flow.yaml` — it produces typed interfaces from the source schema under `flow_generated/`, giving you autocomplete and type checking on `_read.doc.*` access. For SQLite, lambdas bind by JSON pointer (`$field`, `$nested$field`) with no codegen step; just match field names to what the pulled source spec shows.

### Output schema

**Default to schema inference. Hand-author the output schema only for the specific exceptions listed at the end of this section.** Estuary continuously infers a schema from the documents your derivation actually emits. You reference that inferred schema by name and let it widen as data flows — you never write its fields out yourself. The pattern, inline in `flow.yaml`:

```yaml
collections:
  acmeCo/derived/customers:
    key: [/customer_id]
    writeSchema:
      type: object
      properties:
        customer_id: { type: string }
      required: [customer_id]
    readSchema:
      allOf:
        - $ref: flow://write-schema
        - $ref: flow://inferred-schema
```

How the two schemas divide the work:

- `writeSchema` is what the lambda must satisfy on write. Keep it to the **key plus any hard constraints** (see exceptions below) and nothing else. A minimal `writeSchema` is also what lets delete events — which carry only the key and `_meta` — pass write validation; see [Handling deletes](#handling-deletes) below.
- `readSchema` combines `flow://write-schema` with `flow://inferred-schema`. Both are **references the control plane expands at publish time** — `flow://inferred-schema` resolves to whatever Estuary has inferred from emitted documents. Materializations consume `readSchema`, so they pick up new fields automatically as the lambda emits them.

**Do not hand-write the inferred schema.** `flow://inferred-schema` is a reference, not a placeholder for you to fill in. Spelling the emitted fields out yourself — listing them under `readSchema`, or replacing the `$ref` with an explicit `properties` block — defeats the whole mechanism: the schema stops tracking what the lambda emits and goes stale the moment the lambda changes. Write the key in `writeSchema`, reference `flow://inferred-schema`, and stop there. To *inspect* what got inferred once data is flowing, `flowctl catalog pull-specs` shows the expanded form; that is for reading, never for authoring.

**First-publish caveat:** `flow://inferred-schema` initially expands to a placeholder that rejects all documents until the derivation emits something. Downstream materializations may briefly show a schema error on first publish; it clears as soon as data flows.

**When to hand-author instead** — the narrow exceptions (some skills in this family always hit one):

- The output needs **reduction annotations** (`reduce: { strategy: sum }`, `merge`, `lastWriteWins`, etc.). Inference never adds reduce annotations, so `derivation-aggregate-metrics`, `derivation-join-collections`, and reduction-flavored cases of `derivation-stateful-logic` declare the schema by hand.
- You need a specific **constraint enforced at write time** — an enum, an exact type, a regex — that inference wouldn't produce. Declare only that one field in `writeSchema`; keep referencing `flow://inferred-schema` in `readSchema` for everything else.
- You want a **stable published contract** rather than one that widens organically.

Even in the constraint case you hand-author only the constrained field in `writeSchema` — the rest still comes from inference. Reduction annotations belong on the **output** schema (not the source) and control how documents reduce at materialization time.

## Stateless vs stateful

The distinction is whether the transform remembers anything between documents:

- **Stateless** — filter, project, transform fields, flatten arrays. Each input produces zero or more outputs based only on that input. No memory.
- **Stateful** — aggregate, join, window, balance tracking, deduplication. The transform accumulates across documents, either by writing to SQLite tables (via migrations) or by emitting documents that Estuary reduces using schema annotations.

The `shuffle` setting routes documents to task shards, and must match the work:

- Stateless: `shuffle: any` — docs can go anywhere, no co-location needed.
- Stateful: `shuffle: { key: [/field] }` — docs with the same key land on the same shard so they share the same state.

Common mistakes:
- `shuffle: any` with stateful work: each event may land on a different shard, see empty state, and aggregates stay at zero.
- Keyed shuffle on stateless work: not an error — just unnecessary co-location overhead. Prefer `shuffle: any` for stateless transforms.

## Workflow

Full walkthrough: https://docs.estuary.dev/guides/flowctl/create-derivation/#create-a-derivation-locally. Summary:

```bash
# 1. Pull the source collection locally
flowctl catalog pull-specs --name acmeCo/source/collection

# 2. Add a derive stanza to flow.yaml (or create a new flow.yaml)

# 3. Scaffold stub files for referenced modules/migrations/lambdas (optional)
flowctl generate --source flow.yaml

# 4. Implement the transform

# 5. Preview locally — runs the derivation on real source data, outputs to stdout
flowctl preview --source flow.yaml
# Target a specific derived collection if the file has multiple:
flowctl preview --source flow.yaml --name acmeCo/derived/collection

# 6. Publish when preview looks right
flowctl catalog publish --source flow.yaml
```

`flowctl generate` is a scaffolding convenience, not a mandatory step. It won't overwrite your existing TypeScript module — only the generated interfaces under `flow_generated/`.

## Testing

### Local prerequisite: Docker (for TS and Python only)

`flowctl preview` and `flowctl catalog test` execute derivations locally. SQLite derivations run in-process and need nothing extra. TypeScript and Python derivations run inside containers (`ghcr.io/estuary/derive-typescript:dev` and `ghcr.io/estuary/derive-python:dev`) — so for those languages **you must have a Docker daemon running locally** (or `colima start`, `Docker Desktop`, etc.). Podman works too: set `DOCKER_CLI=podman` in your environment. Capture and materialization previews always need a container runtime since connectors are images.

Full reference: https://docs.estuary.dev/concepts/tests/. Tests are a first-class spec type with `ingest` and `verify` steps:

```yaml
tests:
  acmeCo/tests/cleanse-customers:
    - ingest:
        collection: acmeCo/production/customers
        documents:
          - { id: "c1", status: "active",   email: "a@x.com", lifetime_spend: 12000, full_name: "Ada Lovelace", created_at: "2024-01-15T10:00:00Z" }
          - { id: "c2", status: "inactive", email: "b@x.com", lifetime_spend: 500,   full_name: "Bob",          created_at: "2024-02-01T10:00:00Z" }
    - verify:
        collection: acmeCo/analytics/active-customers
        documents:
          - { customer_id: "c1", email_normalized: "a@x.com", tier: "gold" }
```

Run locally:

```bash
flowctl catalog test --source flow.yaml
```

Key behaviours:

- **Tests run automatically on publish.** Any test that references a derivation in an `ingest` or `verify` step runs as part of `flowctl catalog publish` — publication fails if the test fails.
- **All docs in one `ingest` step land in a single transaction.** Multiple documents with the same key are reduced before being appended — the right way to test `sum`, `merge`, and other reductions.
- **Use YAML anchors (`&name` / `*name`)** to share setup fixtures across tests and avoid duplication.

## Handling deletes

Delete events arrive with `_meta/op` set to `'d'` and usually carry only the primary key and `_meta` — none of the other fields. Two consequences trip up derivations:

**1. A filter on a business field silently drops deletes.** `WHERE $status = 'active'` evaluates false for a delete event (the field is absent, so `$status` binds as NULL), so the delete never reaches the materialization and the row lingers. Decide explicitly:

- To *keep* deletes flowing (the usual intent), exempt them: `WHERE $_meta$op = 'd' OR (<your conditions>)`.
- To *drop* deletes on purpose, say so: `WHERE $_meta$op != 'd' AND <your conditions>`.

**2. Propagating a hard delete needs the "double delete."** Estuary applies a hard delete through a [reduction](https://docs.estuary.dev/concepts/#reductions): the `_meta/op: 'd'` document has to reduce *into* an existing document for that key. If the delete is the first time the derivation has seen the key — common after filtering, or during an imprecise backfill where the delete can arrive before the create — there is nothing to reduce into, so the row materializes as a soft delete instead of being removed. Emitting the delete **twice in the same lambda** guarantees a second document for the reduction to act on (and when the key was genuinely never seen, the two deletes reduce together and create no row):

```sql
-- Normal path: transform every non-delete event.
SELECT $id, JSON($flow_document)
WHERE $_meta$op != 'd';

-- Delete path: emit the same delete twice, in this same lambda block.
SELECT $id, $_meta WHERE $_meta$op = 'd';
SELECT $id, $_meta WHERE $_meta$op = 'd';
```

Both emits must live in one lambda block. Splitting them into separate named transforms gives each its own read progress, so the two deletes can land in different transactions and orphan again. The delete emit carries only `$id` and `$_meta`, which is why the output `writeSchema` must not `require` business fields (see [Output schema](#output-schema) above) — a delete event doesn't have them.

The downstream materialization binding must have **Hard Delete** enabled to physically remove the row; without it the delete stays as a soft delete (`_meta/op: 'd'` on the row). Full reference and the connector support list: https://docs.estuary.dev/reference/deletions/.

**When the double delete applies, per derivation type:**

| Derivation type | Double-delete needed? |
|-----------------|----------------------|
| filter-transform, flatten-array, join-collections, stateful per-record outputs | Yes, if output is 1:1 with source records and a downstream materialization has Hard Delete enabled |
| aggregate-metrics, windowing | No — outputs are summaries/buckets, not individual source rows |

If you instead express the delete branch as a JSON Schema conditional reduction, read [Composing with conditionals](https://docs.estuary.dev/reference/reduction-strategies/composing-with-conditionals/) for `if/then/else` and `oneOf` rules — and the gotcha in the next section.

## Conditional reduction — always pin `required`

`properties: { foo: { const: "X" } }` matches "`foo` equals X **or** `foo` is absent." Customer derivations that route deletes via `if: { properties: { _meta: { properties: { op: { const: "d" } } } } }` without `required` stanzas silently route every document missing the discriminator into the delete branch — and a document without `_meta` satisfies the `if`. The result is silent row loss during backfills or whenever upstream emits documents without the expected discriminator path.

Always pin `required` at every level the conditional mentions:

```yaml
if:
  required: ["_meta"]
  properties:
    _meta:
      required: ["op"]
      properties:
        op: { const: "d" }
then:
  reduce: { strategy: merge, delete: true }
```

The full reference for `if/then/else` and `oneOf` in reductions is [Composing with conditionals](https://docs.estuary.dev/reference/reduction-strategies/composing-with-conditionals/).

## Universal gotchas

- **SQLite type coercion follows source schema `format` annotations.** A source field declared `type: string` with `format: number` (or `format: integer`) binds to `$field` as a numeric value, not a string. Without that annotation, a stringified number like `"49.98"` binds as text. Match output schema types to what the parameter actually binds as, not the raw JSON shape.
- **Missing fields bind as SQL NULL.** A document missing the JSON pointer that backs `$field` produces `$field = NULL`, which makes `WHERE $field = 'x'` evaluate false (not unknown-matching). Use `COALESCE($field, '') = 'x'` or `$field IS NOT NULL AND $field = 'x'` to handle absent fields.
- **TypeScript modules aren't overwritten.** Regenerating stubs after a schema change leaves your `.ts` file alone — only `flow_generated/` updates. Delete and regenerate if you want a fresh stub.
- **Changing a collection's key forces re-creation.** A published collection's key can't be updated in place — Estuary's [schema evolution](https://docs.estuary.dev/guides/schema-evolution/#re-creating-a-collection) feature handles this by creating a new collection with a `_v2` suffix and updating downstream materializations to point at it.

## Which derivation skill to use

| Skill | Purpose |
|-------|---------|
| `derivation-filter-transform` | Stateless filtering, field selection, per-document transforms |
| `derivation-flatten-array` | Emit many documents from one (unnest arrays or extract nested objects into rows) |
| `derivation-aggregate-metrics` | Sum/count/min/max reductions, continuous materialized views |
| `derivation-join-collections` | Combine two or more collections on a shared key |
| `derivation-stateful-logic` | Business logic with SQLite state (balances, workflows, inventory) |
| `derivation-windowing` | Time-bounded sliding windows via `readDelay` |
| `derivation-python` | Python class with async transforms, Pydantic types, state lifecycle (private/BYOC planes only) |

## Related docs

- [Derivations concept](https://docs.estuary.dev/concepts/derivations/)
- [Create a Derivation guide](https://docs.estuary.dev/guides/flowctl/create-derivation/)
- [SQL tutorial](https://docs.estuary.dev/guides/derivation_tutorial_sql/)
- [TypeScript tutorial](https://docs.estuary.dev/guides/transform_data_using_typescript/)
- [Python tutorial](https://docs.estuary.dev/guides/transform_data_using_python/)
- [Reduction strategies reference](https://docs.estuary.dev/reference/reduction-strategies/)
- [AcmeBank tutorial (stateful + windowing)](https://docs.estuary.dev/getting-started/tutorials/derivations_acmebank/)
