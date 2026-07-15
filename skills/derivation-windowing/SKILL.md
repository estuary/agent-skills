---
name: derivation-windowing
description: Create an Estuary derivation that maintains a sliding time window of events (e.g. "transfers in the last 24h", "failed logins in the last 5 minutes") using internal SQLite state and readDelay for time-based expiration. Derivations add complexity and cost — confirm the user wants a derivation before reaching for this. Use when user says "windowing derivation", "derivation for last-24h events", "rolling-window derivation", "sliding-window transformation", "fraud-detection derivation", "rate-limit derivation", or "time-window derivation".
---

# derivation-windowing

Estuary derivation that maintains a **sliding time window** of events per key — events enter the window on arrival and leave it after a fixed delay. Typically used for fraud detection, rate limiting, or "recent activity" signals.

**Prereq:** read `derivation-basics` first. This skill is the most complex of the set — it combines internal SQLite state (like `derivation-stateful-logic`) with a second transform using `readDelay` to age events out of the window.

**Docs:**
- https://docs.estuary.dev/getting-started/tutorials/derivations_acmebank/#grouped-windows-of-transfers — the canonical walkthrough for this exact pattern
- https://docs.estuary.dev/concepts/derivations/#internal-state — internal state mechanics

## When to use this over alternatives

- **Sliding / rolling windows**: "last 24h of high-value transfers", "last 5min of failed logins"
- **Time-bounded rate limits**: "more than 10 requests per user in the last minute"
- **Fraud / anomaly signals**: "user did X more than N times within a window"
- **Real-time dashboards** showing "activity in the last hour"

Reach for other skills when:
- Fixed **tumbling** windows (hourly buckets, daily totals, no overlap) → `derivation-aggregate-metrics` with the hour/date as key. Much simpler.
- State that doesn't age out on a time basis → `derivation-stateful-logic`
- Latest N events (not time-bounded) → aggregate with `append` + truncation at query time

## Before you build this — cost caveat

This is the rarest derivation pattern in practice and the most expensive. Every event lives in per-shard SQLite state for the full `readDelay`, so a high-volume real-time stream holds a large working set in RocksDB on the reactor — the state cost scales with event volume times window length. In most cases a simpler tool fits:

- A **destination-side window function** (`OVER (... RANGE ...)`), which most warehouses run natively and cheaply against already-materialized data. This is usually the right answer.
- A **fixed tumbling window** via `derivation-aggregate-metrics` keyed on the hour/date, if you only need non-overlapping buckets.

Reach for a windowing derivation only when you genuinely need a continuously-maintained *sliding* window upstream of the materialization (e.g. a real-time fraud/rate-limit signal that has to act before data lands). If the window is just for reporting, keep it at the destination.

## How it works (one paragraph)

Two transforms read the **same source**. The first (no `readDelay`) processes each event immediately — adding it to an internal SQLite window table and emitting any derived output based on the current window. The second (with `readDelay: 24h` or similar) processes the same events later — removing them from the window table. Net effect: each event spends exactly `readDelay` inside the window. The output collection reflects the state as of "the last event processed" for each key.

## Canonical Example — SQL

Fraud signal: flag users who make more than N high-value transfers within a 24-hour window.

### Project layout

```
my-derivation/
├── flow.yaml
├── schema.yaml
├── add.sql       # check window + add to window state
├── remove.sql    # remove from window after readDelay
└── init.sql      # migration: window_transfers table
```

### `init.sql` (migration)

```sql
CREATE TABLE IF NOT EXISTS window_transfers (
  transfer_id TEXT PRIMARY KEY NOT NULL,
  user_id     TEXT NOT NULL,
  amount      REAL NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_user ON window_transfers (user_id);
```

### `schema.yaml`

```yaml
type: object
properties:
  user_id:
    type: string
  transfer_id:
    type: string
  amount:
    type: number
  timestamp:
    type: string
    format: date-time

  # Window-derived fields:
  window_count:
    description: Number of tracked transfers in the last 24h (including this one)
    type: integer
  window_total:
    description: Sum of amounts in the last 24h (including this one)
    type: number
  is_suspicious:
    type: boolean

required: [user_id, transfer_id]
```

### `flow.yaml`

```yaml
collections:
  acmeCo/fraud/suspicious-transfers:
    writeSchema: schema.yaml
    readSchema:
      allOf:
        - $ref: flow://write-schema
        - $ref: flow://inferred-schema
    key: [/user_id, /transfer_id]
    derive:
      using:
        sqlite:
          migrations:
            - init.sql
      transforms:
        # On arrival: count what's in window + add this event
        - name: addToWindow
          source: acmeCo/production/transfers
          shuffle: { key: [/user_id] }
          lambda: add.sql

        # 24h later: remove this event from window
        - name: removeFromWindow
          source: acmeCo/production/transfers
          shuffle: { key: [/user_id] }
          readDelay: 24h
          lambda: remove.sql
```

Both transforms shuffle on `/user_id` so the window for each user stays on one shard. Only the remove transform sets `readDelay`.

### `add.sql`

Using `JSON_OBJECT` is important here so that `is_suspicious` becomes a proper JSON boolean — SQLite has no native boolean type, so `(expr >= 5)` returns 0 or 1. The special "single column named `json*` becomes the document" rule lets `JSON_OBJECT(...)` plus `JSON('true')` / `JSON('false')` produce real booleans in the output document.

```sql
-- 1. Snapshot the current window for this user BEFORE adding this event
WITH stats AS (
  SELECT
    COUNT(*)                   AS cnt,
    COALESCE(SUM(amount), 0.0) AS total
  FROM window_transfers
  WHERE user_id = $user_id
)
-- 2. Emit the enriched event (only for high-value transfers; skip tiny ones)
SELECT json_object(
  'user_id',       $user_id,
  'transfer_id',   $transfer_id,
  'amount',        $amount,
  'timestamp',     $timestamp,
  'window_count',  stats.cnt + 1,
  'window_total',  stats.total + $amount,
  'is_suspicious', json(CASE WHEN stats.cnt + 1 >= 5 THEN 'true' ELSE 'false' END)
) AS json_result
FROM stats
WHERE $amount > 500;

-- 3. Always add to window (including low-value — simpler; tweak per use case)
INSERT OR IGNORE INTO window_transfers (transfer_id, user_id, amount)
VALUES ($transfer_id, $user_id, $amount);
```

Critical: **query before inserting**. Otherwise the freshly-inserted row shows up in its own `stats.cnt`, which double-counts it.

### `remove.sql`

```sql
DELETE FROM window_transfers WHERE transfer_id = $transfer_id;
```

A bare `DELETE` is a valid lambda — no trailing `SELECT` needed.

## Canonical Example — TypeScript

TypeScript is rarely the right choice for windowing — the pattern is fundamentally SQLite-shaped (table + insert/delete/select). If you're thinking of TypeScript, reconsider whether a tumbling window via `derivation-aggregate-metrics` would do.

## Test

### Local validation — `flowctl preview --fixture` (partial)

Windowing has a fundamental problem for local validation: `readDelay: 24h` requires 24 hours of wall-clock time to pass for the remove transform to process the events. `flowctl preview` doesn't simulate the passage of time — its fixture timestamps are the document's content, not runtime clock.

What this means:
- **The "add" side can be validated**: feed events, see how `window_count` evolves as events accumulate.
- **The "remove" side cannot easily be validated locally**: events won't expire from the window during a preview run.

To fully test expiration, publish to a real data plane and wait, or run a custom harness (TypeScript test script that simulates time).

`fixture.jsonl`:

```json
["acmeCo/production/transfers", {"transfer_id": "t1", "user_id": "alice", "amount": 600, "timestamp": "2024-06-01T10:00:00Z"}]
["acmeCo/production/transfers", {"transfer_id": "t2", "user_id": "alice", "amount": 700, "timestamp": "2024-06-01T10:05:00Z"}]
["acmeCo/production/transfers", {"transfer_id": "t3", "user_id": "alice", "amount": 550, "timestamp": "2024-06-01T10:10:00Z"}]
["acmeCo/production/transfers", {"transfer_id": "t4", "user_id": "alice", "amount": 900, "timestamp": "2024-06-01T10:15:00Z"}]
["acmeCo/production/transfers", {"transfer_id": "t5", "user_id": "alice", "amount": 1200, "timestamp": "2024-06-01T10:20:00Z"}]
["acmeCo/production/transfers", {"transfer_id": "t6", "user_id": "bob", "amount": 800, "timestamp": "2024-06-01T10:25:00Z"}]
{"commit": true}
```

Expected progression when previewing **just the add transform** (remove isn't delayed properly in preview — see gotchas):

```json
{"user_id":"alice","transfer_id":"t1","window_count":1,"window_total":600,"is_suspicious":false,...}
{"user_id":"alice","transfer_id":"t2","window_count":2,"window_total":1300,"is_suspicious":false,...}
{"user_id":"alice","transfer_id":"t3","window_count":3,"window_total":1850,"is_suspicious":false,...}
{"user_id":"alice","transfer_id":"t4","window_count":4,"window_total":2750,"is_suspicious":false,...}
{"user_id":"alice","transfer_id":"t5","window_count":5,"window_total":3950,"is_suspicious":true,...}
{"user_id":"bob","transfer_id":"t6","window_count":1,"window_total":800,"is_suspicious":false,...}
```

If you include the remove-transform in preview, you'll see `window_count=1` on every event — preview processes the remove immediately instead of waiting 24h. That's a preview limitation; the live pipeline honours `readDelay` correctly.

### Spec-defined tests — `tests:` block

```yaml
tests:
  acmeCo/tests/suspicious-transfers:
    - ingest:
        collection: acmeCo/production/transfers
        documents:
          - { transfer_id: "t1", user_id: "alice", amount: 600,  timestamp: "2024-06-01T10:00:00Z" }
          - { transfer_id: "t2", user_id: "alice", amount: 700,  timestamp: "2024-06-01T10:05:00Z" }
          - { transfer_id: "t3", user_id: "alice", amount: 550,  timestamp: "2024-06-01T10:10:00Z" }
          - { transfer_id: "t4", user_id: "alice", amount: 900,  timestamp: "2024-06-01T10:15:00Z" }
          - { transfer_id: "t5", user_id: "alice", amount: 1200, timestamp: "2024-06-01T10:20:00Z" }
    - verify:
        collection: acmeCo/fraud/suspicious-transfers
        documents:
          - { user_id: "alice", transfer_id: "t1", amount: 600,  timestamp: "2024-06-01T10:00:00Z", window_count: 1, window_total: 600,  is_suspicious: false }
          - { user_id: "alice", transfer_id: "t2", amount: 700,  timestamp: "2024-06-01T10:05:00Z", window_count: 2, window_total: 1300, is_suspicious: false }
          - { user_id: "alice", transfer_id: "t3", amount: 550,  timestamp: "2024-06-01T10:10:00Z", window_count: 3, window_total: 1850, is_suspicious: false }
          - { user_id: "alice", transfer_id: "t4", amount: 900,  timestamp: "2024-06-01T10:15:00Z", window_count: 4, window_total: 2750, is_suspicious: false }
          - { user_id: "alice", transfer_id: "t5", amount: 1200, timestamp: "2024-06-01T10:20:00Z", window_count: 5, window_total: 3950, is_suspicious: true  }
```

## Variations

- **Different window sizes:** `readDelay: 1h` / `24h` / `168h` (7d) / `30d`. The duration uses humantime format — `Ns`, `Nm`, `Nh`, `Nd`, `Nw` all work; seconds are convenient for testing.
- **Multiple windows on the same stream:** one derivation per window size. Each has its own state table and remove-transform.
- **Windowed aggregations only (no per-event output):** have only a single transform that updates a `user_stats` table and emits the user's current metrics. No readDelay removal needed if the stats are cumulative.
- **Conditional tracking:** only insert into the window if the event matches (e.g. high-value). Matching filter and tracking filter must agree.
- **Alert-only output:** `WHERE stats.cnt + 1 >= 5` so only suspicious events are emitted (the derived collection holds only alerts, not every event).

## Gotchas specific to windowing

- **SELECT before INSERT**, always. Query the window state first; then add the current event. Inserting first causes the current event to count itself, inflating `window_count` by 1.
- **`readDelay` uses humantime duration format** (`24h`, `168h`, `30m`, `45s`, `7d`, `1w`), not ISO 8601. No quoting needed.
- **Initial activation after publish takes time** because migrations and indexes are created. Subsequent restarts skip migrations that already ran.
- **Window state is unbounded if the remove transform lags or fails.** Watch shard lag. If stuck, events accumulate indefinitely, eventually affecting memory and query performance.
- **Use `INSERT OR IGNORE` when duplicates are possible** (e.g. replaying events). Without it, duplicates in the source fail with `UNIQUE constraint`.
- **Preview can't simulate `readDelay` expiration.** Local testing validates only the "event enters window" side. Full validation requires a live data plane.
- **Remove transform must use the same shuffle key** as the add transform. Otherwise the delete runs on a shard that has no record of the event.
- **Count doesn't include the current event until you add it.** The `stats.cnt + 1` in the add lambda compensates: you're about to insert, so count what's in window + 1.
- **`readDelay` does not re-order events.** It delays the processing of each event by the specified duration relative to its publish time. For strict "time = now − 24h" semantics you'd need the source to carry event-time timestamps and the lambda to filter by them explicitly.

## Related

- `derivation-basics` — prerequisite, especially the stateful section
- `derivation-stateful-logic` — same SQLite state mechanics, but without time-based expiration
- `derivation-aggregate-metrics` — simpler tumbling-window alternative when fixed buckets suffice
- [AcmeBank grouped windows of transfers](https://docs.estuary.dev/getting-started/tutorials/derivations_acmebank/#grouped-windows-of-transfers)
- [Internal task state](https://docs.estuary.dev/concepts/derivations/#internal-state)
