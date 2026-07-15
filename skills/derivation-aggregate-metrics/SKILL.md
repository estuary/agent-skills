---
name: derivation-aggregate-metrics
description: Create an Estuary derivation that continuously aggregates metrics (daily totals, running counts, min/max, lifetime revenue) using reduction annotations. Use for real-time dashboards, continuous materialised views, and replacing batch ETL aggregations with streaming. Derivations add complexity and cost — confirm the user wants a derivation before reaching for this. Use when user says "aggregation derivation", "derivation to sum daily totals", "running-count derivation", "transformation for min/max per group", "derive lifetime metrics", "aggregate-by-date derivation", or "real-time-dashboard derivation".
---

# derivation-aggregate-metrics

Stateless Estuary derivation that continuously aggregates source documents into metrics, using schema-level reduction annotations to combine documents with the same key.

**Prereq:** read `derivation-basics` first for concepts, project layout, workflow, and the stateless-vs-stateful distinction.

**Docs:**
- https://docs.estuary.dev/getting-started/tutorials/continuous-materialized-view/ — end-to-end tutorial with reductions
- https://docs.estuary.dev/reference/reduction-strategies/ — reference for `sum`, `merge`, `minimize`, `maximize`, `append`, `firstWriteWins`, `lastWriteWins`

## When to use this over alternatives

- **Daily / hourly / per-customer aggregations**: sum orders by day, count events per user, etc.
- **Running totals**: lifetime revenue per customer that updates in real time
- **Statistical metrics per group**: min / max / sum / count per sensor, per region
- **Continuous materialised views**: stream-native replacement for scheduled aggregation queries

Reach for other skills when:
- Output is 1:1 with input (no grouping) → `derivation-filter-transform`
- Output is many-per-input (not grouping) → `derivation-flatten-array`
- Combining across multiple source collections on a shared key → `derivation-join-collections` (which uses the same reduction mechanics)
- Custom state you update procedurally (not pure reduction) → `derivation-stateful-logic`

## How it works (one paragraph)

You don't write aggregation logic yourself. Your lambda emits one "delta" document per source input (`1 AS order_count`, `$total AS revenue`). The derived collection's **schema** carries `reduce:` annotations telling Estuary's runtime how to merge documents with the same key (`sum` adds numeric values, `maximize` keeps the latest timestamp, etc.). At materialisation time, the runtime reduces all deltas with the same key into a single final document. This is why `shuffle: any` works even though the effect is aggregation — the lambda itself is stateless.

## Canonical Example — SQL

Aggregate orders into daily metrics: order count, total revenue, total items, smallest/largest order.

### Project layout

```
my-derivation/
├── flow.yaml
└── schema.yaml
```

### `schema.yaml`

Top-level `reduce: merge` is mandatory — it tells Estuary to merge documents with the same key rather than overwrite. Each aggregated field carries its own strategy:

```yaml
type: object
properties:
  date:
    type: string
    format: date

  # sum-of-ones counts
  order_count:
    type: integer
    default: 0
    reduce: { strategy: sum }

  # sum of per-order values
  total_revenue:
    type: number
    default: 0
    reduce: { strategy: sum }
  total_items:
    type: integer
    default: 0
    reduce: { strategy: sum }

  # min / max tracking
  smallest_order:
    type: number
    reduce: { strategy: minimize }
  largest_order:
    type: number
    reduce: { strategy: maximize }

reduce: { strategy: merge }
required: [date]
```

`default: 0` on numeric sum fields is optional for the reduction itself (sum implicitly treats a missing LHS as 0). Keep it if downstream consumers expect a non-null field on the first delta — otherwise it can be dropped.

### `flow.yaml`

```yaml
collections:
  acmeCo/analytics/daily-order-metrics:
    schema: schema.yaml
    key: [/date]
    derive:
      using:
        sqlite: {}
      transforms:
        - name: perOrder
          source: acmeCo/production/orders
          shuffle: any
          lambda: |
            SELECT
              date($created_at) AS date,
              1                 AS order_count,
              $total            AS total_revenue,
              $item_count       AS total_items,
              $total            AS smallest_order,
              $total            AS largest_order;
```

Each source order becomes one tiny "delta" doc. The runtime reduces all deltas with the same `date` into the final aggregate. `shuffle: any` is correct — the lambda is per-document with no state; reduction happens downstream.

## Canonical Example — TypeScript

TypeScript is rarely needed here — the lambda logic is trivial — but it helps when you need conditional emissions or computed buckets:

```typescript
import { IDerivation, Document, SourcePerOrder } from 'flow/acmeCo/analytics/daily-order-metrics.ts';

export class Derivation extends IDerivation {
  perOrder(_read: { doc: SourcePerOrder }): Document[] {
    const order = _read.doc;

    return [{
      date: order.created_at.split('T')[0],
      order_count: 1,
      total_revenue: order.total,
      total_items: order.item_count,
      smallest_order: order.total,
      largest_order: order.total,
      // Example of computed bucket only TS can do cleanly:
      // high_value_count: order.total > 1000 ? 1 : 0,
    }];
  }
}
```

Output schema and `flow.yaml` are the same — just swap `sqlite: {}` for the TypeScript module.

## Test

### Local validation — `flowctl preview --fixture`

Reductions behave a bit oddly under preview. Preview may show partially-reduced output — Estuary reduces within a transaction boundary but not across, so with 3 same-day orders you might see 1, 2, or 3 docs for that date depending on how the events were batched. The full reduction happens at **materialisation time**, not at emit time. Don't assert on exact preview doc counts; assert on totals-per-key after materialisation.

`fixture.jsonl`:

```json
["acmeCo/production/orders", {"order_id": "o1", "created_at": "2024-06-01T09:00:00Z", "total": 49.98,  "item_count": 2}]
["acmeCo/production/orders", {"order_id": "o2", "created_at": "2024-06-01T13:30:00Z", "total": 250.00, "item_count": 5}]
["acmeCo/production/orders", {"order_id": "o3", "created_at": "2024-06-01T18:15:00Z", "total": 12.50,  "item_count": 1}]
["acmeCo/production/orders", {"order_id": "o4", "created_at": "2024-06-02T10:00:00Z", "total": 99.00,  "item_count": 3}]
{"commit": true}
```

```bash
flowctl preview --source flow.yaml \
  --name acmeCo/analytics/daily-order-metrics \
  --fixture fixture.jsonl --timeout 30s
```

With this single-transaction fixture (all 4 orders before one `{"commit": true}`), the runtime reduces every same-key doc within the transaction, producing one fully-reduced doc per date:

```json
{"date":"2024-06-01","order_count":3,"total_revenue":312.48,"total_items":8,"smallest_order":12.50,"largest_order":250.00}
{"date":"2024-06-02","order_count":1,"total_revenue":99.00,"total_items":3,"smallest_order":99.00,"largest_order":99.00}
```

To see partial / pre-reduction deltas (what the materialization receives in production when events arrive in separate transactions), add intermediate `{"commit": true}` markers between orders in the fixture. Each transaction's deltas are then emitted separately.

### Spec-defined tests — `tests:` block

Note that `verify` compares against the **reduced** result — so the test asserts totals, not per-delta docs:

```yaml
tests:
  acmeCo/tests/daily-order-metrics:
    - ingest:
        collection: acmeCo/production/orders
        documents:
          - { order_id: "o1", created_at: "2024-06-01T09:00:00Z", total: 49.98,  item_count: 2 }
          - { order_id: "o2", created_at: "2024-06-01T13:30:00Z", total: 250.00, item_count: 5 }
          - { order_id: "o3", created_at: "2024-06-01T18:15:00Z", total: 12.50,  item_count: 1 }
          - { order_id: "o4", created_at: "2024-06-02T10:00:00Z", total: 99.00,  item_count: 3 }
    - verify:
        collection: acmeCo/analytics/daily-order-metrics
        documents:
          - { date: "2024-06-01", order_count: 3, total_revenue: 312.48, total_items: 8, smallest_order: 12.50, largest_order: 250.00 }
          - { date: "2024-06-02", order_count: 1, total_revenue: 99.00,  total_items: 3, smallest_order: 99.00,  largest_order: 99.00 }
```

## Variations

- **Multi-dimensional grouping:** `key: [/date, /region, /category]`. Extend the lambda to `SELECT date($created_at), $region, $category, ...`. Works the same — reduction happens per composite key.
- **Running totals (lifetime per customer):** `key: [/customer_id]`, no date bucket. The reduce strategies stay the same.
- **Hourly buckets:** `strftime('%Y-%m-%dT%H:00:00Z', $timestamp)` instead of `date()`. The key becomes the hourly bucket.
- **Deriving averages:** store `sum` and `count` separately, divide at query time. Averages can't be reduced (averaging two averages isn't equivalent).
- **"Latest value" for string fields:** use `reduce: { strategy: lastWriteWins }`. (`maximize` keeps the *lexicographically* greatest value, not the most-recently-written one — `"zzz"` arriving before `"aaa"` would still win.)
- **Collecting into arrays:** `reduce: { strategy: append }`. Use sparingly — arrays grow unbounded if the key cardinality × events-per-key is high.

## Gotchas specific to aggregate-metrics

- **This reduction-based aggregation pattern requires standard-updates mode on the destination.** Reduction happens at materialization time when Estuary loads the existing row, reduces incoming deltas into it, and writes back the reduced result — that only happens in standard-updates mode (the default on every connector). If the destination must use `delta_updates: true` (opt-in on Snowflake / BigQuery / Databricks for cost or latency reasons), Estuary appends each per-transaction delta as-is and your table will show fragments, not running totals. Either reduce downstream in the query layer, or use a stateful aggregation via `derivation-stateful-logic` — maintain the running total in SQLite and emit the current total per input so no downstream reduction is needed. See [Delta updates](https://docs.estuary.dev/concepts/materialization/#delta-updates) and the worked `sum` example in [materialization-protocol](https://docs.estuary.dev/reference/Connectors/materialization-protocol/#push-only-endpoints--delta-updates).
- **Preview shows pre-reduction deltas.** This is not a bug — the runtime reduces at materialisation time. If you want to see final reduced state locally, materialise to a local Postgres (via the `flowctl preview` materialization path or `docker-compose` Postgres in `projects/customer-llm-skills/`).
- **`reduce: merge` on the top level is mandatory.** Without it, Estuary overwrites documents with the same key instead of merging field-by-field. Result: only the last delta survives, no summing.
- **Floating-point drift on big sums.** For financial data, store amounts as integer cents (`total_cents: integer`) and format at query time. Summing millions of floats accumulates imprecision.
- **Key cardinality blows up state.** Don't group by things with unbounded cardinality (e.g., event IDs). Pick grouping dimensions that bound the number of keys.
- **Schema changes to reductions may require reset.** Changing a field's `reduce: strategy` after data exists is not always compatible — you may need to publish a new collection and backfill.

## Related

- `derivation-basics` — prerequisite
- `derivation-join-collections` — uses the same reduction mechanics to merge across multiple sources
- `derivation-stateful-logic` — when you need procedural state (balances, approval counts), not pure reduction
- `derivation-windowing` — time-bounded aggregation (last 24h), when simple per-day buckets aren't enough
- [Continuous materialised view tutorial](https://docs.estuary.dev/getting-started/tutorials/continuous-materialized-view/)
- [Reduction strategies reference](https://docs.estuary.dev/reference/reduction-strategies/)
