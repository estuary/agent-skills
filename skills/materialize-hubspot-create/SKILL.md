---
name: materialize-hubspot-create
description: Create a HubSpot materialization to write Estuary collections back into HubSpot CRM objects (Contacts, Companies, Deals, Tickets, etc). Use when setting up HubSpot as a destination for captured data. Use when user says "send to HubSpot", "materialize to HubSpot", "write back to HubSpot", or "HubSpot destination".
---

# Create HubSpot Materialization

Create a HubSpot materialization using flowctl to write data from Estuary collections into HubSpot CRM objects.

**Applies to**: materialize-hubspot

This connector is a **write-back** connector — it's for pushing enriched or external data into HubSpot CRM records, not for general-purpose data warehousing. Common use case: enrich HubSpot contacts/companies with data computed elsewhere (product usage, lead scoring, support signals) and sync it back into HubSpot properties.

## Known Limitations

- **Non-unique match properties can create duplicate records.** With a non-unique match property, the connector falls back to a search-based lookup to find existing records; concurrent writes against the same non-unique value can create two records for one logical entity instead of updating one. Treat a unique match property as a hard requirement, not a recommendation.
- **An invalid enumeration value stops the whole materialization task, not just that record.** The connector either writes the data as given or stops — it does not skip or drop an invalid value to keep the task running. All bound object types share one task, so one bad value on any binding halts every binding until it's resolved. Validate enum values upstream (e.g. in a derivation) before they reach this connector.
- **Sensitive/Highly Sensitive HubSpot properties can't be written.** Writing these fields requires OAuth scopes the connector doesn't request. There's no workaround short of not targeting those properties.
- **Associations between CRM objects can't be materialized.** Associations need a separate process (e.g. a HubSpot API call from a derivation, or workflow automation in HubSpot).
- **Deleting a source row never deletes the HubSpot record.** Hard deletes aren't supported. Track deletions via a `meta_op` property instead (see Step 1.4).
- **Long/unusual field names can fail to map to the expected property, silently.** The documented truncation rule for property names over 100 characters doesn't reliably match the resulting property name, and when the computed property name doesn't exist, the value is dropped with no error or warning. Spot-check any field with an unusual (long, digit-leading, special-character) name in HubSpot after publishing rather than trusting a clean task status.
- **Round-trip pipelines (HubSpot capture → derivation → this materialization, writing back into HubSpot) carry a feedback-loop risk.** The connector always runs in delta-update mode, so every document is written as given, and each write bumps the same `updatedAt`/`lastmodifieddate` field the HubSpot capture uses for incremental change detection. The capture can see the materialization's own write as a new change, re-feed it through the derivation, and write again — indefinitely, if the derivation's output is deterministic for unchanged inputs. For this pattern, use a stateful derivation that suppresses re-emission when only previously-written fields have changed: fingerprint the input fields the derivation actually reads, store it, and skip emitting when the fingerprint is unchanged. See `derivation-stateful-logic`.

## Step 0: Load Connector Documentation

Before proceeding, fetch the official connector docs for prerequisites, config reference, and setup instructions.

**Load the docs page:**
https://docs.estuary.dev/reference/Connectors/materialization-connectors/hubspot/

Use WebFetch to load this page. It covers:

- OAuth2 and Service Key (private app token) authentication
- Full config property reference
- Match property / collection key matching behavior
- Property name mapping rules
- Supported CRM objects and their limitations

**Search Kapa for tribal knowledge** (if the Estuary MCP is configured):

```
Search kapa ai knowledge sources for "materialize hubspot common issues"
```

If Kapa MCP is not configured, the user can set it up: https://docs.estuary.dev/features/mcp-integration/

This skill provides the **flowctl workflow** and **troubleshooting** that docs don't cover.

## Step 1: Gather Requirements

Before writing any YAML, ask the user:

1. **Which HubSpot CRM object(s)?** — One binding per object type. Supported: Contacts, Companies, Deals, Calls, Emails, Line Items, Meetings, Postal Mail, Products, Quotes, Tasks, Tickets.
2. **Authentication method?**
   - **OAuth2** (recommended) — same OAuth flow as the HubSpot capture connector; easiest to complete via the Estuary web UI, which mints the `refresh_token`.
   - **Service Key** — a HubSpot private app access token. Unlike the HubSpot *capture* connector (where private app tokens don't work), the **materialization connector does support private app tokens** via `ServiceKey`. This is a useful path for local/CLI testing without the web UI OAuth dance.
3. **Match property already created in HubSpot?** — HubSpot assigns record IDs automatically; the connector can't set them. It matches incoming documents to existing records using the **collection key**, compared against a property in HubSpot. Confirm the user has created a property on the target object for this — and require it to be **unique** (check "require unique values for this property" in the property's Property Rules). Putting it in a dedicated property group is recommended but not required. See Known Limitations for why non-unique is a hard no.
4. **Tracking deletes?** — Hard deletes are **not supported**. If the user needs to track deletions, they must create a `meta_op` property on the object in HubSpot, **and** declare `/_meta/op` in the collection's schema with a `projections: { meta_op: /_meta/op }` entry and `fields: { require: { meta_op: {} } }` on the binding (see Troubleshooting → "Deleted source rows still present in HubSpot" for the full YAML). Creating the property alone isn't enough — the default field-selection mode excludes `/_meta/op`. Otherwise, deleted source rows are simply never removed from HubSpot.
5. **Enum-valued HubSpot properties?** — If any target property is a HubSpot enumeration/dropdown property, only pre-defined values are accepted. See Known Limitations: an invalid value doesn't just fail that write, it stops the whole task. Confirm with the user that upstream data is validated against the property's allowed values before this connector ever sees it.
6. **Associations?** — Not supported. If the user needs to materialize relationships between CRM objects (e.g. deal-to-contact associations), this connector can't do it — flag this early.
7. **Sensitive fields?** — Properties marked Sensitive or Highly Sensitive in HubSpot cannot be written to (see Known Limitations). Confirm none of the target properties are so classified. Note: Sensitive Data Properties is also a HubSpot plan-tier feature — free/developer test portals may reject creating such properties at all (`PORTAL_NOT_ENABLED_FOR_SENSITIVE_DATA`).
8. **Round-trip pipeline (HubSpot → derivation → back to HubSpot)?** — If the user is capturing from HubSpot, enriching in a derivation, and materializing back into the *same* HubSpot objects, read the feedback-loop entry in Known Limitations before building anything — it needs a stateful derivation to avoid an infinite capture/write loop.
9. **Non-default data plane?** — Most users use the default. Ask if they need a non-default data plane.
10. **Source collections?** — Which Estuary collections map to which HubSpot object bindings.

### Delta updates — always on

Unlike most materializations, this connector **always** uses delta updates (`delta_updates: true`) — it's not a configurable choice. There's no merge-then-read-back cycle; writes are one-way. Don't offer this as a toggle to the user.

### Updates are partial patches, not full-record replacements

Sending a document that carries only a subset of fields (e.g. just the match key + one changed property) updates *only those fields* on the matched HubSpot record — properties omitted from that document are left untouched, not cleared or nulled out. This makes the connector well-suited for incremental enrichment (e.g. writing a single computed field without needing the rest of the contact's data on hand). It does **not** behave like a full-document overwrite/replace.

### Property name mapping

Collection field names auto-map to HubSpot internal property names: lowercased, leading underscores/special characters stripped, prefixed with `n` if the name starts with a digit, truncated to 100 characters. If the user wants specific HubSpot property names, they should create matching properties ahead of time or use a `schema-projections` rename so the mapped name lines up with an existing HubSpot property.

The digit-prefix (`8ball_answer` → `n8ball_answer`) and lowercasing (`CamelCaseField` → `camelcasefield`) rules map as documented. The >100-character truncation rule is less reliable — see Known Limitations. Don't assume a missing column means nothing was sent; for any field with an unusual/long name, verify the actual value landed in HubSpot after publishing rather than trusting a clean `flowctl catalog status`.

Also note: because this connector uses `delta_updates: true`, adding a field to `fields.require` does **not** retroactively backfill it onto previously-materialized records — a record's `lastmodifieddate` doesn't change just from a field-selection-only republish. The source document needs to be re-emitted (or the binding's `backfill` counter bumped) before the new field populates on existing rows.

## Step 2: Find the Correct Connector Version

Always use the latest numbered version tag. Query the connector registry:

```bash
flowctl raw get --table connector_tags \
  --query 'documentation_url=eq.https://go.estuary.dev/materialize-hubspot' \
  --query 'select=image_tag,documentation_url' \
  --output yaml
```

Use the returned `image_tag` — never hardcode a version.

## Step 3: Help User Complete Prerequisites

1. **HubSpot properties exist** — For each target object, the match property (Step 1.3) and every property the collection will write to must already exist in HubSpot. The connector does not create properties.
2. **Credentials**:
   - **OAuth2**: Complete the flow in the Estuary web UI (mints `refresh_token`), then pull the spec to local — same pattern as `capture-hubspot-create`:
     ```bash
     flowctl catalog pull-specs --name <TENANT>/<PATH>/materialize-hubspot
     ```
   - **Service Key**: In HubSpot, create a private app (Settings → Integrations → Private Apps) and grant it write scopes for the target objects and properties. Copy the generated access token.
3. **Deletion tracking (optional)** — If hard-delete tracking matters, create the `meta_op` property on each target object before publishing.

## Step 4: Create the Spec File

Build `flow.yaml` using the config reference from the docs.

**OAuth2:**

```yaml
materializations:
  <TENANT>/<PATH>/materialize-hubspot:
    endpoint:
      connector:
        image: ghcr.io/estuary/materialize-hubspot:<VERSION>
        config:
          credentials:
            auth_type: OAuth2
            refresh_token: "<REFRESH_TOKEN>"
    bindings:
      - source: <TENANT>/<PATH>/contacts
        resource:
          object: Contacts
      - source: <TENANT>/<PATH>/companies
        resource:
          object: Companies
```

**Service Key (private app token):**

```yaml
materializations:
  <TENANT>/<PATH>/materialize-hubspot:
    endpoint:
      connector:
        image: ghcr.io/estuary/materialize-hubspot:<VERSION>
        config:
          credentials:
            auth_type: ServiceKey
            service_key: "<PRIVATE_APP_ACCESS_TOKEN>"
    bindings:
      - source: <TENANT>/<PATH>/contacts
        resource:
          object: Contacts
```

**Advanced rate-limit tuning** (only if the user hits 429s or needs to go faster within HubSpot's account limits):

```yaml
config:
  credentials: { ... }
  advanced:
    limit: 10 # standard API requests/sec (default 10)
    burst: 100 # standard API burst allowance (default 100)
    search_limit: 5 # search API requests/sec (default 5)
    search_burst: 5 # search API burst allowance (default 5)
```

### Protect secrets before committing

`refresh_token` / `service_key` are secrets — don't commit them in plain text. Encrypt with Estuary's sops-based mechanism:

```bash
sops --encrypt \
  --input-type yaml --output-type yaml \
  --encrypted-suffix "_token" \
  --encrypted-suffix "_key" \
  --gcp-kms projects/<PROJECT>/locations/global/keyRings/<RING>/cryptoKeys/<KEY> \
  flow.yaml > flow.encrypted.yaml
mv flow.encrypted.yaml flow.yaml
```

flowctl recognizes sops-encrypted specs and decrypts them at publish time. See https://docs.estuary.dev/concepts/connectors/#protecting-secrets for full options.

## Step 5: Publish

```bash
flowctl catalog publish --source flow.yaml --auto-approve
```

## Step 6: Verify

```bash
# Check status
flowctl catalog status <TENANT>/<PATH>/materialize-hubspot

# View logs
flowctl logs --task <TENANT>/<PATH>/materialize-hubspot --since 5m | jq -c '{ts, message}'
```

**Status progression:**

1. `PENDING` — Normal for ~30 seconds during shard assignment
2. `BACKFILLING` — Initial write of existing collection data into HubSpot
3. `OK` — Running normally with real-time updates

Then confirm in the HubSpot UI that records are appearing/updating on the target object, and that the match property is populated as expected (not creating duplicates).

## Troubleshooting

### Duplicate records being created instead of updated

**Cause**: The match property isn't unique, or the collection key doesn't consistently map to the same value. With a non-unique match property, the connector falls back to a search-based lookup that isn't race-safe — concurrent writes carrying the same new match-key value can create separate HubSpot records instead of one. See Known Limitations.

**Fix**:

1. Make the match property unique in HubSpot: `Settings / Data Management / Properties` → the property → Property Rules → "require unique values for this property"
2. Verify the collection key is stable and actually unique per logical record
3. Consider using the HubSpot native object ID as the collection key if available from the source

### Deleted source rows still present in HubSpot / `meta_op` stays empty

**Cause**: Hard deletes aren't supported by this connector — expected behavior, not a bug. But if `meta_op` exists in HubSpot and still comes back empty/`null`, the field isn't actually being selected for materialization — the property existing is necessary but not sufficient.

**Fix**: To track deletions instead of physically removing records:

1. Create a `meta_op` property on the target object in HubSpot
2. Declare `/_meta/op` in the collection schema, add a projection, and explicitly require the field on the binding:
   ```yaml
   collections:
     <TENANT>/<PATH>/contacts:
       schema:
         properties:
           _meta:
             type: object
             properties:
               op: { type: string }
         # ...rest of schema
       projections:
         meta_op: /_meta/op

   materializations:
     <TENANT>/<PATH>/materialize-hubspot:
       bindings:
         - source: <TENANT>/<PATH>/contacts
           resource: { object: Contacts }
           fields:
             recommended: true
             require:
               meta_op: {}
   ```
3. Filter/alert on `meta_op = 'd'` downstream in HubSpot (e.g. via a workflow) rather than expecting removal

### "PROPERTY_DOESNT_EXIST" or similar property errors — or a field is just silently empty

**Cause**: The auto-mapped property name (lowercased, stripped, truncated) doesn't match any existing HubSpot property — the connector doesn't create properties. For the **match-key** field specifically, this fails loudly at publish time (`field is forbidden by the connector (A property for this field does not exist)`). For an ordinary (non-key) field, a name-mapping mismatch instead fails **silently** — no error, no log line, the property just never gets a value while everything else reports healthy. See Known Limitations.

**Fix**:

1. Create the property in HubSpot first, matching the mapped name exactly (see "Property name mapping" above)
2. Or add a `schema-projections` rename on the source collection so the field maps to the existing property name
3. Don't rely on the absence of errors as confirmation a field materialized — spot-check the actual HubSpot record for fields with unusual (long, special-character, digit-leading) names

### Enumeration / dropdown property errors stop the whole task

**Cause**: The value being written isn't one of the property's pre-defined options. The connector either writes the data as given or stops the task — it does not skip or drop an invalid value to keep running. A single invalid enum value fails the connector process (`transactor.Store: unexpected response: 400 Bad Request: <value> was not one of the allowed options`), and the task repeats that same failure on every automatic restart (the transaction never committed), with retry backoff stretching to several minutes between attempts, until the underlying data problem is resolved. All bound object types share one task, so this blocks every other binding too (e.g. a bad Contacts value also stops Companies updates).

**Fix**:

1. Check the property's allowed values in HubSpot (`Settings / Data Management / Properties`) and ensure upstream data only produces those values — filter or map invalid values in a derivation before materializing. This is a required guardrail given the connector's fail-stop behavior, not optional hardening.
2. If already stuck: either add the missing value as a valid option on the HubSpot property (fastest unblock, if the value is legitimate), or fix the upstream data and re-emit it so the transaction can advance past the bad document.
3. After fixing, don't just wait for the backoff timer — force an immediate retry with a no-op republish: `flowctl catalog publish --source flow.yaml --auto-approve`.
4. Verify recovery with `flowctl catalog status <task>` (expect `OK: Running`) and check logs for a clean `Running` message with no follow-up `shard failed`.

### "insufficient scopes" or write failures on specific properties

**Cause**: The property is marked Sensitive or Highly Sensitive in HubSpot. Writing these needs OAuth scopes the connector doesn't request — this can't be fixed by editing token/app scopes, since the connector's own OAuth flow doesn't ask for the sensitive-data scopes at all. Alternatively, for Service Key auth, the private app may simply be missing the required write scope for that object (a normal, fixable scope issue, distinct from the sensitive-field gap).

**Fix**:

1. Check the property's sensitivity classification in HubSpot — if Sensitive/Highly Sensitive, this connector cannot write it; remove it from the target
2. For Service Key auth and a non-sensitive property that's still failing, edit the private app's scopes to include write access for the object type, then republish with the refreshed token if it changed

### Associations not appearing in HubSpot

**Cause**: This connector doesn't support materializing associations between CRM objects — expected limitation, not an error.

**Fix**: None via this connector. If associations are required, they need to be created through a separate process (HubSpot API call from a derivation, workflow automation, etc).

### "429 Too Many Requests" / HubSpot rate limit errors

**Cause**: Default limits (10 req/s standard, 5 req/s search) can still be tight for very large backfills, and non-unique match properties multiply search API calls.

**Fix**:

1. Make the match property unique to avoid falling back to search-heavy matching (biggest lever)
2. Tune `advanced.limit` / `advanced.burst` / `advanced.search_limit` / `advanced.search_burst` within your HubSpot account's actual API limits
3. Estuary automatically retries with backoff — transient 429s usually self-heal

### Materialization stuck in PENDING

Wait 30-60 seconds — this is normal during shard assignment. If still stuck:

```bash
flowctl logs --task <TENANT>/<PATH>/materialize-hubspot --since 5m | jq 'select(.level == "error")'
```

## Related Skills

- `capture-hubspot-create` — Capture data *from* HubSpot (the read counterpart to this connector)
- `schema-projections` — Rename collection fields to match existing HubSpot property names
- `schema-field-selection` — Control which fields get written to HubSpot
- `estuary-connector-restart` — Pause/restart existing materializations
- `estuary-logs` — Deep log analysis
- `estuary-catalog-status` — Status checking
