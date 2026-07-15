---
name: derivation-flatten-array
description: Create an Estuary derivation that flattens a nested array into one output document per array element. Use for normalising nested JSON, exploding line items, unpacking tags, or producing one-row-per-item tables from arrays. Derivations add complexity and cost — confirm the user wants a derivation before reaching for this (some destinations can `UNNEST` server-side without one). Use when user says "flatten-array derivation", "derivation to explode an array", "transformation to unnest", "derive one row per item", or "flatten nested data with a derivation".
---

# derivation-flatten-array

Stateless Estuary derivation that emits many output documents from a single source document by iterating over a nested array.

**Prereq:** read `derivation-basics` first for concepts, project layout, workflow, and language choice.

**Docs:** https://docs.estuary.dev/guides/flatten-array/ — the official TypeScript walkthrough for this exact pattern. Read it alongside this skill.

## When to use this over alternatives

- **Normalising nested JSON:** `{order_id, line_items: [...]}` → one doc per line item
- **Exploding tags/categories:** `{article_id, tags: ["a", "b", "c"]}` → one doc per tag
- **Unpacking array columns** from source databases for analytics
- **Preparing data for systems that prefer normalised 1-row-per-item tables**

If the array isn't in the source document itself (e.g., you need to join across collections to assemble it), use `derivation-join-collections`. If you just need to keep the array intact but change how it's materialised (e.g., MongoDB `flow_document`), you don't need a derivation at all.

## Canonical Example — TypeScript (recommended)

Orders arrive with an embedded array of line items. We emit one document per line item, carrying order context (customer, date) onto each row.

### Project layout

```
my-derivation/
├── flow.yaml
├── schema.yaml
└── flatten.ts
```

### `schema.yaml`

The output schema describes **one** line item — not the array:

```yaml
type: object
properties:
  order_id:
    type: string
  line_id:
    type: string
  customer_id:
    type: string
  order_date:
    type: string
    format: date-time
  product_id:
    type: string
  product_name:
    type: string
  quantity:
    type: integer
  unit_price:
    type: number
  line_total:
    type: number
required: [order_id, line_id, product_id]
```

### `flow.yaml`

```yaml
collections:
  acmeCo/normalized/order-line-items:
    writeSchema: schema.yaml
    readSchema:
      allOf:
        - $ref: flow://write-schema
        - $ref: flow://inferred-schema
    key: [/order_id, /line_id]
    derive:
      using:
        typescript:
          module: flatten.ts
      transforms:
        - name: flattenLineItems
          source: acmeCo/production/orders
          shuffle: any
```

Key fields: the collection key is composite (`order_id` + `line_id`) — the array element id keeps output rows unique. `shuffle: any` because flattening is stateless. The `writeSchema` declares what the lambda emits; `readSchema` lets the inferred schema widen automatically as the lambda evolves. See `derivation-basics` § Schema setup.

### `flatten.ts`

Generate the stub with `flowctl generate --source flow.yaml`, then fill in:

```typescript
import { IDerivation, Document, SourceFlattenLineItems } from 'flow/acmeCo/normalized/order-line-items.ts';

export class Derivation extends IDerivation {
  flattenLineItems(_read: { doc: SourceFlattenLineItems }): Document[] {
    const order = _read.doc;

    // Missing or empty array → emit nothing
    if (!order.line_items || order.line_items.length === 0) {
      return [];
    }

    return order.line_items.map(item => ({
      order_id: order.order_id,
      line_id: item.line_id,
      customer_id: order.customer_id,
      order_date: order.order_date,
      product_id: item.product_id,
      product_name: item.product_name,
      quantity: item.quantity,
      unit_price: item.unit_price,
      line_total: item.quantity * item.unit_price,
    }));
  }
}
```

Return `[]` to drop the document entirely; return one document per array element to flatten.

## Canonical Example — SQL

SQLite can flatten via `json_each`, which produces one row per array element. Works well for shallow arrays of primitives or simple objects; gets verbose for deeply nested structures (TypeScript is usually clearer then).

### Tags array (simple case)

```yaml
collections:
  acmeCo/normalized/article-tags:
    schema:
      type: object
      properties:
        article_id: { type: string }
        tag: { type: string }
      required: [article_id, tag]
    key: [/article_id, /tag]
    derive:
      using:
        sqlite: {}
      transforms:
        - name: flattenTags
          source: acmeCo/articles
          shuffle: any
          lambda: |
            SELECT
              $article_id AS article_id,
              json_each.value AS tag
            FROM json_each($tags);
```

### Object array (more verbose)

```yaml
lambda: |
  SELECT
    $order_id AS order_id,
    json_extract(json_each.value, '$.line_id')      AS line_id,
    json_extract(json_each.value, '$.product_id')   AS product_id,
    json_extract(json_each.value, '$.product_name') AS product_name,
    CAST(json_extract(json_each.value, '$.quantity') AS INTEGER)   AS quantity,
    json_extract(json_each.value, '$.unit_price')   AS unit_price
  FROM json_each($line_items);
```

For anything beyond simple fields, prefer the TypeScript form.

## Test

### Local validation — `flowctl preview --fixture` (works today)

`fixture.jsonl`:

```json
["acmeCo/production/orders", {"order_id": "o1", "customer_id": "c1", "order_date": "2024-06-01T10:00:00Z", "line_items": [{"line_id": "li_1", "product_id": "p_a", "product_name": "Widget", "quantity": 2, "unit_price": 10.00}, {"line_id": "li_2", "product_id": "p_b", "product_name": "Gadget", "quantity": 1, "unit_price": 25.00}, {"line_id": "li_3", "product_id": "p_c", "product_name": "Gizmo", "quantity": 3, "unit_price": 5.50}]}]
["acmeCo/production/orders", {"order_id": "o2", "customer_id": "c2", "order_date": "2024-06-02T10:00:00Z", "line_items": []}]
{"commit": true}
```

```bash
flowctl preview --source flow.yaml \
  --name acmeCo/normalized/order-line-items \
  --fixture fixture.jsonl --timeout 30s
```

Expected output (3 docs from o1, nothing from o2's empty array):

```json
{"order_id":"o1","line_id":"li_1","customer_id":"c1","order_date":"2024-06-01T10:00:00Z","product_id":"p_a","product_name":"Widget","quantity":2,"unit_price":10.00,"line_total":20.00}
{"order_id":"o1","line_id":"li_2","customer_id":"c1","order_date":"2024-06-01T10:00:00Z","product_id":"p_b","product_name":"Gadget","quantity":1,"unit_price":25.00,"line_total":25.00}
{"order_id":"o1","line_id":"li_3","customer_id":"c1","order_date":"2024-06-01T10:00:00Z","product_id":"p_c","product_name":"Gizmo","quantity":3,"unit_price":16.50}
```

### Spec-defined tests — `tests:` block

```yaml
tests:
  acmeCo/tests/flatten-line-items:
    - ingest:
        collection: acmeCo/production/orders
        documents:
          - order_id: "o1"
            customer_id: "c1"
            order_date: "2024-06-01T10:00:00Z"
            line_items:
              - { line_id: "li_1", product_id: "p_a", product_name: "Widget", quantity: 2, unit_price: 10.00 }
              - { line_id: "li_2", product_id: "p_b", product_name: "Gadget", quantity: 1, unit_price: 25.00 }
              - { line_id: "li_3", product_id: "p_c", product_name: "Gizmo",  quantity: 3, unit_price: 5.50 }
          - order_id: "o2"
            customer_id: "c2"
            order_date: "2024-06-02T10:00:00Z"
            line_items: []
    - verify:
        collection: acmeCo/normalized/order-line-items
        documents:
          - { order_id: "o1", line_id: "li_1", customer_id: "c1", order_date: "2024-06-01T10:00:00Z", product_id: "p_a", product_name: "Widget", quantity: 2, unit_price: 10.00, line_total: 20.00 }
          - { order_id: "o1", line_id: "li_2", customer_id: "c1", order_date: "2024-06-01T10:00:00Z", product_id: "p_b", product_name: "Gadget", quantity: 1, unit_price: 25.00, line_total: 25.00 }
          - { order_id: "o1", line_id: "li_3", customer_id: "c1", order_date: "2024-06-01T10:00:00Z", product_id: "p_c", product_name: "Gizmo",  quantity: 3, unit_price: 5.50,  line_total: 16.50 }
```

## Variations

- **Simple string array (tags, prefs):** `json_each($tags)` in SQL, `_read.doc.tags.map(tag => ({...}))` in TS.
- **JSON-string-encoded array:** wrap `JSON.parse` in try/catch for TS; `json_each(json($field))` in SQL where `$field` is a string.
- **Deeply nested arrays (categories→products):** nested `for...of` loops in TS. SQL gets painful fast — prefer TS.
- **Conditional flattening:** filter array elements before emitting (`_read.doc.addresses.filter(a => a.is_primary).map(...)`).
- **Include array index as a field:** TS has `.map((item, i) => ...)`; SQL uses `json_each.key` as the index.
- **Partial re-emit on upstream update:** if the source document's array shrinks on update, stale output documents remain. See the gotcha below.

## Gotchas specific to flatten-array

- **Key uniqueness is on you.** The derived collection key must uniquely identify a flattened row. If the array elements have stable IDs (`line_id`), use them. If they don't, include the array index (`item_index`) in the key — otherwise multiple elements collide and reduce into one.
- **Source updates don't remove stale flattened rows.** If an upstream order initially had 3 line items and the array later shrinks to 2, the derivation re-runs on the source update and emits 2 new rows — but the 3rd earlier row stays in the derived collection. If you need strict 1:1 sync with the source array, model deletions explicitly (e.g., source emits full array each update and you compare; or include a `deleted` flag per line item).
- **Empty/missing array is not an error.** `return []` in TS or `FROM json_each(NULL)` in SQL both produce zero output docs, which is the right behaviour — but surprises people expecting a per-source-doc output. The source document itself is not reflected in the derived collection.
- **`json_each` behavior on non-array input.** Only `json_each(NULL)` and `json_each('[]')` silently yield zero rows. A JSON object yields one row per member, a scalar (`json_each(42)`) yields a single row, and a non-JSON string raises a `malformed JSON` error that surfaces as a transform failure. If you hit "no output", inspect the source with `flowctl preview --source flow.yaml --name <source-collection>` to see what the field actually contains.

## Related

- `derivation-basics` — prerequisite
- `derivation-filter-transform` — when you want zero-or-one output per input, not many
- `derivation-join-collections` — when the array you want to flatten is spread across multiple source collections
- [How to Flatten an Array Using TypeScript (docs walkthrough)](https://docs.estuary.dev/guides/flatten-array/)
- [Derivations concept](https://docs.estuary.dev/concepts/derivations/)
