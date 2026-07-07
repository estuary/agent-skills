---
name: derivation-stateful-logic
description: Create an Estuary derivation with custom SQLite state (internal tables) to make per-document decisions based on accumulated history — account balances, inventory levels, approval workflows, deduplication. Derivations add complexity and cost — confirm the user wants a derivation before reaching for this. Use when user says "stateful derivation", "derivation to track balance", "transformation to approve based on state", "derive an approval workflow", "deduplication derivation", or "build a stateful derivation".
---

# derivation-stateful-logic

Estuary derivation that maintains internal SQLite tables to record state, and whose lambdas query and update that state to decide what to emit.

**Prereq:** read `derivation-basics` first — in particular the stateless-vs-stateful distinction and the role of migrations. This skill is the "stateful via SQLite tables" path.

**Docs:**
- https://docs.estuary.dev/getting-started/tutorials/derivations_acmebank/ — the canonical stateful tutorial (deposits, withdrawals, transfer approval) with the exact pattern this skill uses
- https://docs.estuary.dev/concepts/derivations/#internal-state — the concept page on internal task state

## When to use this over alternatives

- **Balance / account state**: debit sender if funds available, reject otherwise
- **Inventory**: deduct stock, mark backordered if insufficient
- **Approval workflows**: accumulate approvers, emit "approved" once threshold met
- **Deduplication with history**: filter events by ID seen-before (beyond what collection-key uniqueness gives)
- **Any business logic that depends on prior events for the same key**

Reach for other skills when:
- Pure aggregations (sum/count/min/max) that reductions can express → `derivation-aggregate-metrics`
- Merging fields across sources → `derivation-join-collections`
- Time-bounded sliding state (last 24h) → `derivation-windowing` (which also uses SQLite internal state, but with `readDelay`)
- Stateless per-doc transforms → `derivation-filter-transform`

## How it works (one paragraph)

You declare SQLite **migrations** in the `derive.using.sqlite.migrations` list — SQL that runs once at startup to create your state tables. Your **lambda** then queries and updates those tables as it processes each source document, and can `SELECT` a derived output row or not, depending on state. Each task shard owns its own SQLite database; events are routed to shards via a **keyed shuffle**, so all events for the same entity (e.g. the same account) reach the same shard and see the same state.

## Canonical Example — SQL

Transfer approval system: track account balances, approve transfers with sufficient funds, reject overdrafts.

### Project layout

```
my-derivation/
├── flow.yaml
├── schema.yaml
├── init.sql           # migration: create balances table
├── debit-sender.sql   # lambda for sender-side (decide + debit)
└── credit-recipient.sql  # lambda for recipient-side (silent credit)
```

### `schema.yaml`

```yaml
type: object
properties:
  transfer_id:
    type: string
  sender:
    type: string
  recipient:
    type: string
  amount:
    type: number
  outcome:
    type: string
    enum: [approved, rejected]
  reason:
    type: string
  sender_balance_after:
    type: number
required: [transfer_id, sender, outcome]
```

### `init.sql` (migration)

```sql
CREATE TABLE IF NOT EXISTS balances (
  account TEXT PRIMARY KEY NOT NULL,
  balance REAL NOT NULL DEFAULT 0
);
```

### `flow.yaml`

Two transforms: one evaluates transfers against the sender's balance; the second credits the recipient's account on approved transfers (reading from the derivation's own output — a self-reference).

```yaml
collections:
  acmeCo/processed/transfer-outcomes:
    writeSchema: schema.yaml
    readSchema:
      allOf:
        - $ref: flow://write-schema
        - $ref: flow://inferred-schema
    key: [/transfer_id]
    projections:
      outcome:
        location: /outcome
        partition: true
    derive:
      using:
        sqlite:
          migrations:
            - init.sql
      transforms:
        - name: debitSender
          source: acmeCo/production/transfers
          shuffle: { key: [/sender] }
          lambda: debit-sender.sql

        - name: creditRecipient
          source:
            name: acmeCo/processed/transfer-outcomes   # self-reference!
            partitions:
              include:
                outcome: [approved]   # only approved transfers
          shuffle: { key: [/recipient] }
          lambda: credit-recipient.sql
```

Keyed shuffle is critical: `/sender` for debit, `/recipient` for credit. Each account's events go to one shard, which owns that account's balance.

### `debit-sender.sql`

```sql
-- 1. Read current balance (default 0 if new account)
WITH current AS (
  SELECT COALESCE((SELECT balance FROM balances WHERE account = $sender), 0) AS balance
)

-- 2. Emit the outcome document
SELECT
  $transfer_id AS transfer_id,
  $sender      AS sender,
  $recipient   AS recipient,
  $amount      AS amount,
  CASE WHEN current.balance >= $amount THEN 'approved' ELSE 'rejected' END AS outcome,
  CASE WHEN current.balance >= $amount
    THEN 'Sufficient funds'
    ELSE 'Insufficient funds: balance ' || current.balance || ' < ' || $amount
  END AS reason,
  CASE WHEN current.balance >= $amount
    THEN current.balance - $amount
    ELSE current.balance
  END AS sender_balance_after
FROM current;

-- 3. Deduct from the sender's balance only if approved
INSERT INTO balances (account, balance)
SELECT $sender, -$amount
WHERE (SELECT COALESCE((SELECT balance FROM balances WHERE account = $sender), 0)) >= $amount
ON CONFLICT(account) DO UPDATE SET balance = balance - $amount;
```

### `credit-recipient.sql`

```sql
-- Silently credit the recipient; this transform emits nothing.
INSERT INTO balances (account, balance)
VALUES ($recipient, $amount)
ON CONFLICT(account) DO UPDATE SET balance = balance + $amount;

-- Produce no output rows (so nothing gets published from this transform):
SELECT NULL WHERE 0;
```

The `SELECT NULL WHERE 0` idiom is how you write a transform that updates state but emits no document. Needed because every SQL lambda must be a syntactically valid query.

## Canonical Example — TypeScript

TypeScript derivations support stateful work via an in-memory state object, but **the idiomatic stateful path in Estuary is SQLite**. Use TypeScript for stateful work only if you have a specific reason (complex Deno-side libraries, WASM, etc.) — otherwise SQLite is more robust, survives shard splits cleanly via the recovery log, and has a clearer operational story.

## Test

### Local validation — `flowctl preview --fixture`

Stateful derivations build state within a preview run. Ingest an ordered sequence of transfers and watch the outcomes evolve. Note: `flowctl preview` uses an in-process runtime (no temp-data-plane subprocess), which also means shard-splitting semantics won't be exactly like production — but for single-shard stateful logic, preview is a good reflection for the primary transform.

**Important caveat for self-referential derivations under `--fixture`:** `flowctl preview --fixture` does NOT simulate transforms that source from the derivation's own output. The fixture file is the only input the preview reader sees, so a `creditRecipient`-style self-read sees no documents — the recipient side won't be credited. (Without `--fixture`, journal-backed preview can read historical self-output, but this requires a previously-published live collection.) For self-referential patterns under development, validate in a real data plane: ingest via HTTP or a capture and read back from the derived collection.

`fixture.jsonl`:

```json
["acmeCo/production/transfers", {"transfer_id": "t1", "sender": "alice", "recipient": "bob", "amount": 50}]
["acmeCo/production/transfers", {"transfer_id": "t2", "sender": "bob", "recipient": "alice", "amount": 20}]
["acmeCo/production/transfers", {"transfer_id": "t3", "sender": "alice", "recipient": "bob", "amount": 1000}]
{"commit": true}
```

Starting balances are all 0. Expected outcomes:

- `t1` — alice has 0, tries to send 50 → **rejected** (sender_balance_after: 0)
- `t2` — bob has 0, tries to send 20 → **rejected** (sender_balance_after: 0)
- `t3` — alice still has 0 → **rejected**

To see approvals you need to seed balances first. For a richer test, add a "deposit" transform that credits accounts without checking — see the AcmeBank tutorial for the full flow.

```bash
flowctl preview --source flow.yaml \
  --name acmeCo/processed/transfer-outcomes \
  --fixture fixture.jsonl --timeout 30s
```

### Spec-defined tests — `tests:` block (aspirational)

```yaml
tests:
  acmeCo/tests/transfer-outcomes:
    - ingest:
        description: Three transfers with zero starting balances — all should reject.
        collection: acmeCo/production/transfers
        documents:
          - { transfer_id: "t1", sender: "alice", recipient: "bob",   amount: 50 }
          - { transfer_id: "t2", sender: "bob",   recipient: "alice", amount: 20 }
          - { transfer_id: "t3", sender: "alice", recipient: "bob",   amount: 1000 }
    - verify:
        collection: acmeCo/processed/transfer-outcomes
        documents:
          - { transfer_id: "t1", sender: "alice", outcome: "rejected", sender_balance_after: 0 }
          - { transfer_id: "t2", sender: "bob",   outcome: "rejected", sender_balance_after: 0 }
          - { transfer_id: "t3", sender: "alice", outcome: "rejected", sender_balance_after: 0 }
```

## Variations

- **Inventory tracking:** `CREATE TABLE inventory (product_id TEXT PRIMARY KEY, stock INTEGER)`. Lambda deducts on orders, marks `backordered` if insufficient. Separate state-only transform for replenishments — just `INSERT ... ON CONFLICT DO UPDATE`, no SELECT needed.
- **Approval workflow:** migration creates an `approval_state` table with approvers JSON array. One transform initialises requests, another adds approvers. Emit `approved` when `json_array_length(approvers) >= required`.
- **Deduplication:** `CREATE TABLE seen (event_id TEXT PRIMARY KEY)`. Lambda `SELECT ... WHERE NOT EXISTS (SELECT 1 FROM seen WHERE event_id = $event_id)` then `INSERT OR IGNORE INTO seen`. Collection key uniqueness handles simple cases; use this when source has duplicates at different key granularity.
- **Session tracking:** state per user_id, open session on login, close on logout, emit session summary on close.
- **Rate limits:** count events per user in state; reject once threshold hit. Combine with `derivation-windowing` for time-bounded rate limits.

## Gotchas specific to stateful-logic

- **`shuffle: any` silently breaks state.** With unkeyed shuffle, each event may land on a different shard and see empty state. Use `shuffle: { key: [/entity_id] }` wherever you read/write SQLite tables. This is the #1 cause of "my balances don't accumulate" bugs.
- **Initial activation after publish takes time.** Migrations run once at first shard startup; subsequent restarts only apply new migrations. Transient errors right after publish typically resolve on their own as shards finish initializing.
- **State-only transforms can use bare DML.** When a transform only updates state and emits nothing, plain `INSERT`, `UPDATE`, or `DELETE` statements are valid — no trailing SELECT needed. (The canonical AcmeBank `creditRecipient` lambda is just `INSERT ... ON CONFLICT DO UPDATE`.)
- **Self-referential transforms** (reading from your own output) are powerful but subtle. Use `partitions.include` to filter which self-events the transform processes, or you can create infinite loops.
- **Migrations are append-only.** You cannot change a published migration — you must add a new one. State is destroyed only on shard delete/recreate (disable/enable preserves it). For critical state, plan how you'd rebuild from source-of-truth events if a recreate is ever needed.
- **Shard splits fork the recovery log.** Both new shards initially hold identical SQLite state; subsequently each processes a disjoint key range under keyed shuffle. Stale rows for the other shard's range remain in each DB but are harmless because no reads target them.
- **Don't use `shuffle: any` with `ON CONFLICT`.** You might think `ON CONFLICT` makes it idempotent, but without keyed shuffle, concurrent updates to the same row across shards will not converge.
- **Order matters within a lambda.** Query state *before* updating it if you want to read pre-update values. `SELECT balance FROM balances ...` before `INSERT ... ON CONFLICT UPDATE`.

## Related

- `derivation-basics` — prerequisite (stateless-vs-stateful, shuffle mechanics)
- `derivation-windowing` — time-bounded state via `readDelay`
- `derivation-aggregate-metrics` — when reductions can express the logic (no procedural state needed)
- [AcmeBank derivations tutorial](https://docs.estuary.dev/getting-started/tutorials/derivations_acmebank/) — the canonical walk-through for this pattern
- [Internal task state](https://docs.estuary.dev/concepts/derivations/#internal-state)
