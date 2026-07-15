---
name: derivation-join-collections
description: Create an Estuary derivation that joins two or more collections on a shared key, merging their fields into a single enriched collection. Use for data enrichment, denormalisation, building wide tables for BI, and real-time CDC joins. Derivations add complexity and cost — confirm the user wants a derivation before reaching for this (a destination-side SQL join or BI tool view is often simpler). Use when user says "join-collections derivation", "derivation to join two collections", "transformation to enrich with customer data", "derive a wide table", "lookup-join derivation", or "build a join derivation".
---

# derivation-join-collections

Estuary derivation that merges documents from multiple source collections into a single collection, keyed on a shared join field. Fields from each source contribute, and Estuary's reduction engine merges them.

**Prereq:** read `derivation-basics` first for concepts, the stateless-vs-stateful distinction, and reduction-annotation mechanics.

**Docs:**
- https://docs.estuary.dev/guides/howto_join_two_collections_typescript/ — official TypeScript walkthrough
- https://docs.estuary.dev/reference/reduction-strategies/ — reduction strategy reference (critical for joins)

**Canonical examples in the Flow repo** (cover the three join shapes — see "Join types" below):
- [join-outer.flow.yaml](https://github.com/estuary/flow/blob/master/examples/derive-patterns/join-outer.flow.yaml) — full outer (reduction-based, no state)
- [join-inner.flow.yaml](https://github.com/estuary/flow/blob/master/examples/derive-patterns/join-inner.flow.yaml) — inner (SQLite state, both sides must match)
- [join-one-sided.flow.yaml](https://github.com/estuary/flow/blob/master/examples/derive-patterns/join-one-sided.flow.yaml) — left / right (SQLite state, one side drives emission)

## When to use this over alternatives

- **Enrichment**: attach customer profile fields onto every order
- **Denormalisation**: combine dimension + fact tables into a wide analytics table
- **Customer lifetime metrics**: one doc per customer with counts / totals rolled up from orders
- **Pre-joined views** for BI tools that prefer wide tables over joins at query time

Reach for other skills when:
- Only one source collection, aggregating by key → `derivation-aggregate-metrics`
- Need an embedded array of matches, not merged fields → combine with `append` reduction (still this skill, but see Variations)
- You want arbitrary stateful logic beyond reductions (approval flows, balance checks) → `derivation-stateful-logic`

## Join types

Stream joins aren't quite the same as SQL joins. Pick the type before writing the derivation — the pattern differs by type:

| Join type | Approach | When to use |
|---|---|---|
| **Full outer** | **Reduction-based** (this skill's canonical example below). Both sides emit partial documents independently; reductions merge them. Documents appear in the output as soon as either side has data for a key. | Default for enrichment / denormalisation when you want every key joined regardless of match. |
| **Inner** | **Stateful** ([join-inner.flow.yaml](https://github.com/estuary/flow/blob/master/examples/derive-patterns/join-inner.flow.yaml)). Both sides write to a SQLite state table; emission is gated on `lhs IS NOT NULL AND rhs IS NOT NULL` so unmatched keys never appear. | When you only want output for keys present on both sides. |
| **Left / right (one-sided)** | **Stateful** ([join-one-sided.flow.yaml](https://github.com/estuary/flow/blob/master/examples/derive-patterns/join-one-sided.flow.yaml)). Both sides update state, but only one side triggers emission. RHS-side updates accumulate silently; LHS-side events emit the joined record. | When a specific side is the natural "driver" (emit per-customer using the latest accumulated orders, but not vice versa). |

Inner and one-sided joins are stateful — they have a state table and `INSERT ... ON CONFLICT DO UPDATE` style lambdas, and conceptually belong to `derivation-stateful-logic`. They're called out here because they share the "joining two streams" problem framing.

## How it works (outer-join reduction pattern — one paragraph)

For the full-outer reduction pattern, you define **one transform per source collection**. Each transform shuffles on the join key and emits partial documents keyed by that same key. The output collection's schema has `reduce:` annotations on every field — at materialisation time, Estuary combines all partial documents with the same key into a single merged row. The "join" isn't a SQL join; it's reduction over co-keyed documents.

## Canonical Example — SQL (outer join via reductions)

Customers table and orders table merge into a wide `customer-lifetime` collection with customer contact info + aggregate order metrics.

### Project layout

```
my-derivation/
├── flow.yaml
└── schema.yaml
```

### `schema.yaml`

```yaml
type: object
properties:
  customer_id:
    type: string

  # Fields from customers source — keep the latest non-empty value
  customer_name:
    type: string
    default: ""
    reduce: { strategy: maximize }
  customer_email:
    type: string
    default: ""
    reduce: { strategy: maximize }
  tier:
    type: string
    default: ""
    reduce: { strategy: maximize }

  # Fields aggregated from orders source
  lifetime_order_count:
    type: integer
    default: 0
    reduce: { strategy: sum }
  lifetime_revenue:
    type: number
    default: 0
    reduce: { strategy: sum }
  first_order_at:
    type: string
    format: date-time
    reduce: { strategy: minimize }
  last_order_at:
    type: string
    format: date-time
    reduce: { strategy: maximize }

reduce: { strategy: merge }
required: [customer_id]
```

Top-level `reduce: merge` is mandatory. `maximize` on string fields keeps the lexicographically greatest value — that's usually fine for dimension fields where the source updates infrequently, but if you actually need "most recently written," use `lastWriteWins` instead. `sum` on numeric fields accumulates deltas from each order.

### `flow.yaml`

Two transforms, one per source, both shuffled on `customer_id`:

```yaml
collections:
  acmeCo/analytics/customer-lifetime:
    schema: schema.yaml
    key: [/customer_id]
    derive:
      using:
        sqlite: {}
      transforms:
        - name: fromCustomers
          source: acmeCo/production/customers
          shuffle: { key: [/customer_id] }
          lambda: |
            SELECT
              $customer_id  AS customer_id,
              $name         AS customer_name,
              $email        AS customer_email,
              $tier         AS tier;

        - name: fromOrders
          source: acmeCo/production/orders
          shuffle: { key: [/customer_id] }
          lambda: |
            SELECT
              $customer_id  AS customer_id,
              1             AS lifetime_order_count,
              $total        AS lifetime_revenue,
              $created_at   AS first_order_at,
              $created_at   AS last_order_at;
```

Each transform emits only the fields it contributes; the reduction merges them field-by-field.

## Canonical Example — TypeScript

TypeScript shines when you want to **collect orders into an array** rather than summing them. The TS variant uses a different output collection (`customer-with-orders`) since its schema replaces the numeric-sum fields with an `orders` array:

```typescript
import { IDerivation, Document, SourceFromCustomers, SourceFromOrders } from 'flow/acmeCo/analytics/customer-with-orders.ts';

export class Derivation extends IDerivation {
  fromCustomers(_read: { doc: SourceFromCustomers }): Document[] {
    return [{
      customer_id: _read.doc.customer_id,
      customer_name: _read.doc.name,
    }];
  }

  fromOrders(_read: { doc: SourceFromOrders }): Document[] {
    return [{
      customer_id: _read.doc.customer_id,
      orders: [{
        order_id: _read.doc.order_id,
        total: _read.doc.total,
        created_at: _read.doc.created_at,
      }],
    }];
  }
}
```

Schema difference: add an `orders: { type: array, reduce: { strategy: append } }` field. Each `fromOrders` call emits a single-element array that `append` appends to the array already accumulated for that `customer_id`.

## Test

### Local validation — `flowctl preview --fixture`

Like aggregate-metrics, preview may show partially-reduced output. With one customer doc + two order docs all for `c1`, you'll see multiple deltas that the materialisation will merge into the final single row.

`fixture.jsonl`:

```json
["acmeCo/production/customers", {"customer_id": "c1", "name": "Alice", "email": "alice@example.com", "tier": "gold"}]
["acmeCo/production/orders", {"order_id": "o1", "customer_id": "c1", "total": 100.00, "created_at": "2024-01-15T10:00:00Z"}]
["acmeCo/production/orders", {"order_id": "o2", "customer_id": "c1", "total": 250.00, "created_at": "2024-03-20T14:30:00Z"}]
["acmeCo/production/customers", {"customer_id": "c2", "name": "Bob", "email": "bob@example.com", "tier": "silver"}]
{"commit": true}
```

```bash
flowctl preview --source flow.yaml \
  --name acmeCo/analytics/customer-lifetime \
  --fixture fixture.jsonl --timeout 30s
```

Expected mental reduction at materialisation:

- `c1`: name=Alice, email=alice@example.com, tier=gold, lifetime_order_count=2, lifetime_revenue=350.00, first_order_at=2024-01-15T10:00:00Z, last_order_at=2024-03-20T14:30:00Z
- `c2`: name=Bob, email=bob@example.com, tier=silver (no orders → numeric defaults, no timestamps)

### Spec-defined tests — `tests:` block (aspirational)

```yaml
tests:
  acmeCo/tests/customer-lifetime:
    - ingest:
        collection: acmeCo/production/customers
        documents:
          - { customer_id: "c1", name: "Alice", email: "alice@example.com", tier: "gold" }
          - { customer_id: "c2", name: "Bob",   email: "bob@example.com",   tier: "silver" }
    - ingest:
        collection: acmeCo/production/orders
        documents:
          - { order_id: "o1", customer_id: "c1", total: 100.00, created_at: "2024-01-15T10:00:00Z" }
          - { order_id: "o2", customer_id: "c1", total: 250.00, created_at: "2024-03-20T14:30:00Z" }
    - verify:
        collection: acmeCo/analytics/customer-lifetime
        documents:
          - { customer_id: "c1", customer_name: "Alice", customer_email: "alice@example.com", tier: "gold",   lifetime_order_count: 2, lifetime_revenue: 350.00, first_order_at: "2024-01-15T10:00:00Z", last_order_at: "2024-03-20T14:30:00Z" }
          - { customer_id: "c2", customer_name: "Bob",   customer_email: "bob@example.com",   tier: "silver" }
```

## Variations

- **Many-to-many via bridge table:** three transforms — e.g. products + product_categories + categories, each shuffled on the appropriate key. Each transform emits the fields it contributes; reduction merges.
- **Multi-level join (orders → line items → products):** chain derivations: first join orders with line items to produce a wide line-item table, then join that with products.
- **Embedded array of child records:** array field with `reduce: append` on the parent side; each child-source transform emits a single-element array.
- **Dimension with slowly-changing-value (SCD):** use `lastWriteWins` on dimension fields so each dimension update replaces prior values. For full history, track in a separate collection.
- **One-sided filter:** a transform can filter its source (`WHERE $status = 'active'`) and only contribute to the joined output for matching rows.

## Gotchas specific to join-collections

- **This reduction-based join pattern requires standard-updates mode on the destination.** The pattern works by each transform emitting a partial document and the materialization loading the existing row, reducing the incoming partial into it, and writing back the merged result. That only happens in standard-updates mode (the default on every connector). If the destination must use `delta_updates: true` (opt-in on Snowflake / BigQuery / Databricks for cost or latency reasons), this pattern won't work — your destination will show fragmented per-source rows instead of joined records. For delta-updates destinations, use a stateful join via `derivation-stateful-logic`: maintain the join table inside SQLite (or Python) and emit already-merged documents so no downstream reduction is needed. See [Delta updates](https://docs.estuary.dev/concepts/materialization/#delta-updates).
- **Shuffle keys must match across transforms.** Both (or all) transforms must `shuffle: { key: [/customer_id] }` so docs with the same join key land on the same task shard. If one transform shuffles on `/id` and another on `/customer_id`, the join never happens.
- **Shuffle key can be any JSON pointer into the source — it doesn't have to be related to the source's collection key.** The constraint is cross-transform: shuffle key types and arity must align so the same logical key lands on the same shard from every source. `shuffle: any` is wrong for joins (it breaks co-location), but no "prefix of collection key" rule exists.
- **String field reduction needs care.** `reduce: maximize` on a string keeps the lexicographically greatest value seen — not the most-recently-written. For dimension fields (name, email), lex-max is usually fine; if you actually need latest-received, use `lastWriteWins`.
- **Orphan keys won't have aggregated fields in the derivation collection.** If a customer has no orders, fields like `lifetime_order_count` are simply absent from the derived doc (defaults are ignored inside derivations). Set `default: 0` so the **materialization** writes 0 instead of NULL — defaults take effect at materialization time, not in the derivation collection itself.
- **`--uncommitted` and preview show partial docs per source.** Each transform emits its own partial doc; reduction merges at materialisation. You'll see a customer doc, then separate order docs, not a single merged doc, in preview.
- **Timestamp conflicts across sources.** If two sources have `updated_at` and you want the newest, explicitly `maximize` — don't rely on ordering.

## Related

- `derivation-basics` — prerequisite
- `derivation-aggregate-metrics` — when aggregating within a single source, same reduction machinery
- `derivation-stateful-logic` — when reductions aren't enough and you need custom SQLite state
- [Join Two Collections Using TypeScript (docs)](https://docs.estuary.dev/guides/howto_join_two_collections_typescript/)
- [Reduction strategies reference](https://docs.estuary.dev/reference/reduction-strategies/)
