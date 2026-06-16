---
name: capture-alpaca-create
description: Create an Alpaca capture using flowctl to stream stock trade data into Estuary collections. Use when setting up an Alpaca Market Data source for historical and real-time stock trades. Use when user says "capture Alpaca", "stream stock trades", "Alpaca market data", or "connect Alpaca to Estuary".
---

# Create Alpaca Capture

Create an Alpaca capture using flowctl to stream stock trade data from the Alpaca Market Data API into Estuary collections — both a historical backfill and a low-latency real-time stream, in parallel.

**Applies to**: source-alpaca (Alpaca Market Data connector)

## Step 0: Load Connector Documentation

Before proceeding, fetch the official connector docs for prerequisites, config reference, and limitations.

**Load the docs page:**
https://docs.estuary.dev/reference/Connectors/capture-connectors/alpaca/

Use WebFetch to load this page. It covers:

- The dual-API model (Trades REST API for backfill + Data API websocket for real-time)
- Supported symbols (8000+ stocks/ETFs, max 20 per capture)
- Plan requirements (Free/`iex` vs. Unlimited/`sip`)
- Full config property reference
- The historical/real-time overlap and how to reconcile duplicates

**Search Kapa for tribal knowledge** (if the Estuary MCP is configured):

```
Search kapa ai knowledge sources for "capture alpaca common issues"
```

If Kapa MCP is not configured, the user can set it up: https://docs.estuary.dev/features/mcp-integration/

This skill provides the **flowctl workflow** and **troubleshooting** that docs don't cover.

## How this connector works

Alpaca captures trade data using **two APIs at once**:

- **Trades REST API** — performs a historical backfill from your `start_date` forward until it catches up to the present.
- **Data API websocket** — opens a real-time stream from the present moment and runs indefinitely until you stop the capture.

All symbols in a capture land in a **single collection** via one binding (`trades`). Because the backfill catches up to where the live stream began, the two streams **overlap and produce some duplicate trade documents** — this is expected and reconcilable (see Limitations and Troubleshooting).

## Step 1: Gather Requirements

Before writing any YAML, ask the user:

1. **Alpaca account + API credentials?** — They need an Alpaca account and its **API Key ID** and **Secret Key** (generate in the Alpaca dashboard under API Keys). Unlike OAuth connectors, this is plain key auth, so the whole flow can be completed from the CLI.
2. **Which plan — Free or Unlimited?** This drives the `feed` and `is_free_plan` settings:
   - **Free plan** → `feed: iex`, `advanced.is_free_plan: true`. Data is a smaller sample and **delayed 15 minutes**.
   - **Unlimited plan** → `feed: sip` for complete, real-time SIP data.
3. **Which symbols?** — A comma-separated list, **maximum 20 per capture**. To track more than 20, set up multiple captures and split symbols between them.
4. **Start date for the backfill?** — Must be **no earlier than `2016-01-01T00:00:00Z`** (Alpaca's earliest available data). Note this has **no effect if changed after the capture has started**.
5. **Stop date?** (optional) — Set `advanced.stop_date` to halt the historical backfill at a fixed date (e.g. for a bounded one-time load).
6. **Backfill-only or real-time-only?** (optional, advanced) — By default the connector runs both. `advanced.disable_real_time: true` gives a backfill-only load; `advanced.disable_backfill: true` gives a stream-only capture.
7. **Non-default data plane?** — Most users use the default. Ask if they need a non-default data plane.

## Step 2: Find the Correct Connector Version

Always use the latest numbered version tag. Query the connector registry to find it:

```bash
flowctl raw get --table connector_tags \
  --query 'documentation_url=eq.https://go.estuary.dev/source-alpaca' \
  --query 'select=image_tag,documentation_url' \
  --output yaml
```

Use the returned `image_tag` — never hardcode a version. (The docs sample shows `:v1`, but confirm against the registry.)

## Step 3: Help User Complete Prerequisites

1. **Create / locate Alpaca API keys** — In the Alpaca dashboard, generate an **API Key ID** and **Secret Key**. The secret key is shown only once at creation; regenerate if lost.
2. **Confirm the plan ↔ feed pairing** (see Step 1). Using `feed: sip` without an Unlimited subscription will fail with a subscription/permission error.
3. **Verify symbols are supported** — Alpaca supports 8000+ stocks and ETFs. To check a symbol, use the Alpaca Broker API. Keep the list to **≤ 20**.

No browser redirect is required — once the keys are in hand, everything else runs in flowctl.

## Step 4: Create the Capture Spec File

Build `flow.yaml` using the config reference from the docs:

```yaml
captures:
  <TENANT>/<PATH>/source-alpaca:
    endpoint:
      connector:
        image: ghcr.io/estuary/source-alpaca:<VERSION>
        config:
          api_key_id: "<ALPACA_API_KEY_ID>"
          api_secret_key: "<ALPACA_API_SECRET_KEY>"
          feed: iex                       # iex (free) or sip (Unlimited plan)
          start_date: "2022-11-01T00:00:00Z"   # must be >= 2016-01-01T00:00:00Z
          symbols: "AAPL,MSFT,AMZN,TSLA,GOOGL"  # max 20, comma-separated
          advanced:
            is_free_plan: true            # set true on a Free plan (delays data 15 min)
            # disable_backfill: false     # set true for a stream-only capture
            # disable_real_time: false    # set true for a backfill-only load
            # stop_date: "2023-01-01T00:00:00Z"
            # max_backfill_interval: "24h"  # smaller helps when tracking many symbols
            # min_backfill_interval: "1m"
    bindings:
      - resource:
          name: trades
        target: <TENANT>/<PATH>/trades
```

**Important**:
- `feed` and `is_free_plan` must match the account's plan. Free plan → `feed: iex` **and** `is_free_plan: true`.
- `start_date` must be `>= 2016-01-01T00:00:00Z`, and is effectively immutable once the capture starts.
- `symbols` is a single comma-separated **string**, not a YAML list, and must contain **≤ 20** symbols.
- `max_backfill_interval` / `min_backfill_interval` are **Go duration strings** (e.g. `24h`, `1m`), not numbers.
- All symbols capture into the **one** `trades` collection.

### Protect secrets before committing

`api_secret_key` is a secret — don't commit it in plain text. Encrypt it with Estuary's sops-based mechanism:

```bash
sops --encrypt \
  --input-type yaml --output-type yaml \
  --encrypted-suffix "_key" \
  --gcp-kms projects/<PROJECT>/locations/global/keyRings/<RING>/cryptoKeys/<KEY> \
  flow.yaml > flow.encrypted.yaml
mv flow.encrypted.yaml flow.yaml
```

> Note: `--encrypted-suffix "_key"` matches both `api_key_id` and `api_secret_key`. The key ID isn't strictly secret, but encrypting both is harmless. flowctl decrypts sops-encrypted specs at publish time. See https://docs.estuary.dev/concepts/connectors/#protecting-secrets for AWS KMS / Azure Key Vault / age options.

## Step 5: Discover and Publish

```bash
# Validate the spec and confirm the trades binding
flowctl discover --source flow.yaml

# Review the spec
cat flow.yaml

# Publish the capture
flowctl catalog publish --source flow.yaml --auto-approve
```

The connector exposes a single `trades` resource, so discovery is mostly a validation step here — you can also author the binding by hand as shown in Step 4.

## Step 6: Verify

```bash
# Check status
flowctl catalog status <TENANT>/<PATH>/source-alpaca

# View logs
flowctl logs --task <TENANT>/<PATH>/source-alpaca --since 5m | jq -c '{ts, message}'

# Read captured trade data
flowctl collections read --collection <TENANT>/<PATH>/trades --uncommitted | head -10
```

**Status progression:**
1. `PENDING` — Normal for ~30 seconds during shard assignment
2. `BACKFILLING` — Historical backfill via the Trades REST API (runs in parallel with the live stream)
3. `OK` — Backfill caught up and/or real-time websocket stream running

**What to expect in the data:**
- On a **Free plan**, real-time trades arrive ~15 minutes delayed and are an `iex`-only sample.
- You'll see **some duplicate trades** where the backfill overlaps the live stream — expected (see Limitations).
- During U.S. market closed hours, the real-time stream is quiet; the backfill still runs.

## Limitations

### Maximum 20 symbols per capture
Capturing more than 20 symbols in a single capture can cause API errors. To track more, **split symbols across multiple captures** (each writing to its own collection).

### Duplicate trade documents from overlapping streams
The historical backfill and the real-time stream overlap, producing duplicate trade documents with identical Alpaca properties but different Estuary metadata. Resolve them downstream:

- **Standard (non-delta) updates materialization** — Estuary deduplicates during materialization automatically. Most materialization connectors default to standard updates.
- **Delta-updates-only materialization** — ensure the destination supports the equivalent of `lastWriteWins` reductions.

## Troubleshooting

### `401 Unauthorized` / `403 Forbidden` / "access key verification failed"

**Cause**: Wrong `api_key_id` / `api_secret_key`, a regenerated/revoked key, or keys from the wrong Alpaca environment.

**Fix**:
1. Re-copy the **API Key ID** and **Secret Key** from the Alpaca dashboard (the secret is shown only once — regenerate if you don't have it).
2. Confirm there's no stray whitespace/newline in the values.
3. Republish after updating the spec.

### "subscription does not permit querying recent SIP data" / feed permission errors

**Cause**: `feed: sip` was set without an Alpaca **Unlimited** subscription. The full SIP feed requires a paid plan.

**Fix**: Either upgrade the Alpaca plan, or switch to the free feed: set `feed: iex` and `advanced.is_free_plan: true`. Free-plan data is an `iex` sample delayed 15 minutes.

### Real-time data is ~15 minutes behind

**Cause**: Free plan. `is_free_plan: true` intentionally delays data by 15 minutes, and `iex` is a partial sample of the market.

**Fix**: This is expected on the Free plan. For complete, low-latency data, upgrade to Unlimited and use `feed: sip` (and remove/disable `is_free_plan`).

### "websocket connection limit exceeded" / stream repeatedly disconnects

**Cause**: Alpaca permits only **one** concurrent market-data websocket connection per account. Running a second Alpaca capture, or using the same keys for another websocket client (a trading bot, a notebook, etc.), contends for that single connection.

**Fix**:
1. Ensure only one consumer uses these API keys' websocket at a time.
2. If you need multiple Alpaca captures (e.g. to exceed 20 symbols), use a **separate Alpaca account / API keys per capture**, since each account allows one stream.
3. As a workaround for a single account, set `advanced.disable_real_time: true` on all but one capture (the others become backfill-only).

### "start_date must be no earlier than 2016-01-01" / config schema validation on start_date

**Cause**: `start_date` is before `2016-01-01T00:00:00Z` (Alpaca's earliest data), or not an RFC 3339 timestamp.

**Fix**: Set `start_date` to `2016-01-01T00:00:00Z` or later, in full timestamp form (e.g. `2022-11-01T00:00:00Z`).

### Changing `start_date` did nothing

**Cause**: `start_date` only takes effect at capture creation — it has **no effect if changed after the capture has started**.

**Fix**: To re-backfill from an earlier date, bump the binding's `backfill` counter (re-reads from the configured `start_date`) or delete and recreate the capture with the new `start_date`.

```yaml
bindings:
  - resource: { name: trades }
    target: <TENANT>/<PATH>/trades
    backfill: 1   # increment to force a fresh backfill
```

### API errors when capturing many symbols

**Cause**: More than 20 symbols in one capture, or large backfill windows straining the REST API.

**Fix**:
1. Keep `symbols` to **≤ 20**; split across multiple captures otherwise.
2. Lower `advanced.max_backfill_interval` (e.g. `"6h"` or `"1h"`) so each backfill request covers a smaller window — useful when tracking many symbols.

### Duplicate trades in the destination

**Cause**: Expected — the backfill overlaps the real-time stream (see Limitations).

**Fix**: Materialize with **standard updates** (Estuary deduplicates automatically), or for delta-updates-only destinations ensure `lastWriteWins`-equivalent reduction support.

### Backfill never finishes / "Config failed schema validation"

**Cause**: Common config mistakes:
- `symbols` written as a YAML list instead of a comma-separated string
- `max_backfill_interval` / `min_backfill_interval` given as a number instead of a Go duration string (`"24h"`, `"1m"`)
- `is_free_plan` not set while using `feed: iex`, or `feed` missing entirely

**Fix**: Match the spec in Step 4. `symbols` is one string; durations are quoted Go-duration strings; `feed` is required.

### Capture stuck in PENDING

Wait 30-60 seconds — this is normal during shard assignment. If still stuck:

```bash
flowctl logs --task <TENANT>/<PATH>/source-alpaca --since 5m | jq 'select(.level == "error")'
```

### No real-time data appearing

**Cause**: U.S. equity markets are closed (no live trades to stream), or `advanced.disable_real_time: true` is set, or the Free plan's 15-minute delay hasn't elapsed yet.

**Fix**:
1. Confirm market hours — the live stream is quiet when the market is closed; the backfill still runs.
2. Check `disable_real_time` is not set.
3. On a Free plan, allow for the 15-minute delay.

## Related Skills

- `materialize-clickhouse-create` / `materialize-snowflake-create` / `materialize-bigquery-create` — Send captured trades to a warehouse (use standard updates to dedupe)
- `estuary-connector-restart` — Pause/restart the capture
- `estuary-logs` — Deep log analysis
- `estuary-catalog-status` — Status checking
- `estuary-task-stats` — Confirm trade throughput
