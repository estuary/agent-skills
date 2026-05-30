---
name: capture-hubspot-create
description: Create a HubSpot Real-Time capture using flowctl to stream CRM data into Estuary collections. Use when setting up a HubSpot source for contacts, companies, deals, tickets, or other HubSpot resources. Use when user says "capture HubSpot", "stream from HubSpot", "HubSpot CDC", or "connect HubSpot to Estuary".
---

# Create HubSpot (Real-Time) Capture

Create a HubSpot capture using flowctl to stream CRM data from HubSpot into Estuary collections in real time.

**Applies to**: source-hubspot-native (HubSpot Real-Time connector)

## Step 0: Load Connector Documentation

Before proceeding, fetch the official connector docs for prerequisites, config reference, and OAuth setup.

**Load the docs page:**
https://docs.estuary.dev/reference/Connectors/capture-connectors/HubSpot-real-time/

Use WebFetch to load this page. It covers:

- Supported HubSpot resources (Contacts, Companies, Deals, Tickets, Custom Objects, etc.)
- OAuth2 authentication and scopes
- Full config property reference
- Calculated property refresh behavior
- Custom object naming (`useLegacyNamingForCustomObjects`)

**Search Kapa for tribal knowledge** (if the Estuary MCP is configured):

```
Search kapa ai knowledge sources for "capture hubspot common issues"
```

If Kapa MCP is not configured, the user can set it up: https://docs.estuary.dev/features/mcp-integration/

This skill provides the **flowctl workflow** and **troubleshooting** that docs don't cover.

## Step 1: Gather Requirements

Before writing any YAML, ask the user:

1. **HubSpot account access?** — User must have access to the HubSpot account being captured
2. **OAuth credentials?** — The connector authenticates with HubSpot via OAuth using `client_id` + `client_secret` + `refresh_token`. The easiest path is to complete the OAuth flow in the Estuary web UI (it mints the `refresh_token` for you), then pull the spec to local. Building your own HubSpot OAuth app and exchanging an auth code for a refresh token manually is supported but more involved.

   The connector's JSON schema also exposes a `Private App Credentials` discriminator, but in practice HubSpot Private App tokens have not been observed to authenticate successfully against the connector — **use OAuth**.
3. **Non-default data plane?** — Most users use the default. Ask if they need a non-default data plane.
4. **Which resources to capture?** — All discovered resources (default) or a subset. The connector auto-discovers: Campaigns, Companies, Contact List Memberships, Contact Lists, Contacts, Custom Objects, Deal Pipelines, Deals, Email Events, Engagements, Feedback Submissions, Form Submissions, Forms, Goals, Line Items, Marketing Emails, Marketing Events, Orders, Owners, Products, Properties, Tickets, Workflows.
5. **Capture property history?** — Off by default. Enable to include historical changes to HubSpot object properties in captured documents.
6. **Calculated property refresh schedule?** — Per-binding cron. The web UI pre-fills `55 23 * * *` (daily at 23:55 UTC), but the connector's spec default is `""` (disabled) — set the cron explicitly if you want refreshes. See note below.
7. **Custom objects naming?** — Off by default. The hidden `useLegacyNamingForCustomObjects` field is `false` by default, which prefixes custom object bindings with `custom_` (e.g., `custom_form_submissions`) to avoid collisions with standard objects. **Do not change without contacting Estuary Support** — it affects discovered resource and collection names.

### OAuth scopes — what to tell the user

During the OAuth flow, HubSpot may present scopes that include **create/update/delete permissions**. This is due to how HubSpot groups permissions and exposes some read APIs behind combined read/write scopes.

**The connector is read-only.** It only reads data from HubSpot and never creates, updates, or deletes CRM objects or other resources. Reassure the user before they click "Approve".

### Calculated properties

HubSpot calculated properties are evaluated at query time and do **not** update the record's `updatedAt` timestamp when they change. The connector uses `updatedAt` for incremental change detection, so calculated property updates are missed by the normal real-time path.

To work around this, the connector can refresh calculated property values on a cron schedule (per-binding `schedule` field). When a refresh fires, it fetches every record's current calculated property values and merges them in.

- **Standard updates materializations**: partial refresh documents are combined with previously captured complete documents — works correctly.
- **Delta updates materializations**: do not fully reduce documents, so partial documents from refreshes land with all non-calculated properties as `null`. Warn the user.

## Step 2: Find the Correct Connector Version

Always use the latest numbered version tag. Query the connector registry:

```bash
flowctl raw get --table connector_tags \
  --query 'documentation_url=eq.https://go.estuary.dev/hubspot-real-time' \
  --query 'select=image_tag,documentation_url' \
  --output yaml
```

Use the returned `image_tag` — never hardcode a version.

## Step 3: Help User Complete Prerequisites

The OAuth Authorization Code flow needs an interactive browser redirect, so it **cannot** be completed inside a headless agent / CLI session. The user must complete it elsewhere first, then bring the resulting credentials back to flowctl.

### Recommended: OAuth via Estuary web UI (then pull to flowctl)

1. In the Estuary web UI, create a new capture and select the **HubSpot ( Real-Time )** connector
2. Click **Authenticate your HubSpot account** to launch the OAuth flow
3. Sign in to HubSpot and approve the scopes (read-only — see note in Step 1)
4. HubSpot redirects back to Estuary with a fresh `refresh_token`
5. To continue in flowctl, pull the published capture spec to local:

```bash
flowctl catalog pull-specs --name <TENANT>/<PATH>/source-hubspot-native
```

### Alternative: bring your own HubSpot OAuth app

If the user needs to manage their own OAuth app (for an air-gapped tenant, scope control, etc.):

1. Register a HubSpot OAuth app via `hs project create` (HubSpot CLI, v7.6.0+) or the HubSpot Developer portal
2. Configure `requiredScopes` / `optionalScopes` and `redirectUrls` on the app
3. Install / authorize the app against the target HubSpot account
4. Exchange the resulting auth code for a `refresh_token` against `https://api.hubapi.com/oauth/v1/token`
5. Plug `client_id`, `client_secret`, and `refresh_token` into the spec

A small local helper (HTTP server on the registered redirect URI + `curl` for the token exchange) is the most reliable way to drive the code-to-refresh-token step from a CLI workflow. `flowctl raw oauth` exists for this purpose but currently has a parsing bug (see Troubleshooting).

## Step 4: Create the Capture Spec File

Build `flow.yaml` using the config reference from the docs:

```yaml
captures:
  <TENANT>/<PATH>/source-hubspot-native:
    endpoint:
      connector:
        image: ghcr.io/estuary/source-hubspot-native:<VERSION>
        config:
          capturePropertyHistory: false
          credentials:
            credentials_title: "OAuth Credentials"
            client_id: "<CLIENT_ID>"
            client_secret: "<CLIENT_SECRET>"
            refresh_token: "<REFRESH_TOKEN>"
          # useLegacyNamingForCustomObjects: false  # hidden — do not change without Estuary Support
    bindings: []
```

**Important**:
- `credentials_title` must be the literal string `"OAuth Credentials"` — it's the discriminator the connector uses to pick the auth schema.
- All three of `client_id`, `client_secret`, and `refresh_token` are required.
- `useLegacyNamingForCustomObjects` is hidden in the dashboard and only editable via flowctl. Leave at default unless instructed by Estuary Support.

### Protect secrets before committing

`client_secret` and `refresh_token` are secrets — don't commit them in plain text. Encrypt them with Estuary's sops-based mechanism:

```bash
# Encrypt only the secret fields in the spec (sops + KMS)
sops --encrypt \
  --input-type yaml --output-type yaml \
  --encrypted-suffix "_token" \
  --encrypted-suffix "_secret" \
  --gcp-kms projects/<PROJECT>/locations/global/keyRings/<RING>/cryptoKeys/<KEY> \
  flow.yaml > flow.encrypted.yaml
mv flow.encrypted.yaml flow.yaml
```

flowctl recognizes sops-encrypted specs and decrypts them at publish time. See https://docs.estuary.dev/concepts/connectors/#protecting-secrets for full options (AWS KMS / Azure Key Vault / age / etc.).

## Step 5: Discover and Publish

```bash
# Discover available HubSpot resources (auto-generates bindings)
flowctl discover --source flow.yaml

# Review the generated bindings — remove any resources the user doesn't want
cat flow.yaml

# Optionally adjust the per-binding calculated property refresh schedule
# (default is "55 23 * * *", set to "" to disable)

# Publish the capture
flowctl catalog publish --source flow.yaml --auto-approve
```

Example binding with explicit refresh schedule:

```yaml
bindings:
  - resource:
      name: companies
      schedule: "55 23 * * *"   # daily at 23:55 UTC
    target: <TENANT>/<PATH>/companies
  - resource:
      name: contacts
      schedule: ""              # disable calculated property refresh (connector default)
    target: <TENANT>/<PATH>/contacts
```

## Step 6: Verify

```bash
# Check status
flowctl catalog status <TENANT>/<PATH>/source-hubspot-native

# View logs
flowctl logs --task <TENANT>/<PATH>/source-hubspot-native --since 5m | jq -c '{ts, message}'

# Read captured data
flowctl collections read --collection <TENANT>/<PATH>/<resource> --uncommitted | head -10
```

**Status progression:**
1. `PENDING` — Normal for ~30 seconds during shard assignment
2. `BACKFILLING` — Initial backfill of each discovered resource
3. `OK` — Running normally with real-time updates and scheduled calculated-property refreshes

## Troubleshooting

### "invalid_grant" or "refresh token is invalid"

**Cause**: Refresh token was revoked, expired, or copied incorrectly. HubSpot refresh tokens can be revoked if the connected app is uninstalled or the user's HubSpot session is invalidated.

**Fix**:
1. Re-run the OAuth flow in the Estuary web UI to mint a fresh `refresh_token`
2. Update the capture spec with the new token and republish
3. Verify the HubSpot user account still has access to the portal

### `BAD_CLIENT_ID` / "missing or unknown client id" (HTTP 400 from `https://api.hubapi.com/oauth/v1/token`)

**Cause**: The `client_id` (or `client_secret`) under `credentials` is wrong, a placeholder, or for a HubSpot OAuth app that has been deleted.

**Fix**:
1. Verify `client_id` and `client_secret` match a current HubSpot OAuth app.
2. If the spec came from the Estuary web UI, re-publish from there to refresh credentials.

### `EXPIRED_AUTHENTICATION` with `expire time: 1970-01-01T00:00:00Z`

**Cause**: A non-OAuth bearer token was supplied (e.g. a HubSpot Private App access token, or a CMS API key) under what the connector treats as an OAuth grant. HubSpot recognizes the token shape but cannot decode an expiry from it, so it falls back to the Unix epoch — making the error message say "expired 20602 day(s) ago." Despite the JSON schema exposing a `Private App Credentials` discriminator, in practice only `OAuth Credentials` reliably authenticates.

**Fix**: Use `credentials_title: "OAuth Credentials"` with a real `client_id` / `client_secret` / `refresh_token` (see Step 3).

### Scopes warning during OAuth — write permissions in the consent screen

**Cause**: HubSpot groups some read APIs behind combined read/write scopes. This is HubSpot's permission model, not the connector requesting write access.

**Fix**: This is expected. The connector is read-only — it never creates, updates, or deletes data. Approve the scopes to proceed.

### Custom object binding has the same name as a standard resource

**Cause**: `useLegacyNamingForCustomObjects: true` was set, allowing custom objects to shadow standard objects (e.g., a custom `form_submissions` object replaces the standard Form Submissions resource).

**Fix**: Set `useLegacyNamingForCustomObjects: false` (default). Custom objects will be prefixed with `custom_` (e.g., `custom_form_submissions`). **Contact Estuary Support before flipping this** — it changes discovered resource names.

### Calculated property values look stale or never update

**Cause**: HubSpot calculated properties don't update `updatedAt`, so the incremental path misses them. They only refresh on the per-binding cron schedule.

**Fix**:
1. Check the binding's `schedule` — empty string disables refresh entirely
2. Default `55 23 * * *` fires once a day at 23:55 UTC. Tighten the cron if you need fresher values (mindful of HubSpot rate limits)
3. Refreshes only fire **after the initial backfill completes** — wait for backfill to finish

### Calculated properties appear as `null` in delta updates destinations

**Cause**: Calculated property refreshes emit partial documents (only the key and calculated properties). Standard materializations merge these with the prior complete document; delta-update materializations don't reduce, so non-calculated fields land as `null`.

**Fix**: Use a standard (non-delta) materialization for tables that depend on calculated property values, or filter out the partial refresh documents in a derivation before the delta-update materialization.

### "429 Too Many Requests" / HubSpot rate limit errors

**Cause**: HubSpot enforces API rate limits per portal. Tight calculated-property refresh schedules or very large portals can hit them.

**Fix**:
1. Loosen the per-binding `schedule` (don't refresh every minute on a large portal)
2. Reduce the number of bindings if practical
3. Estuary automatically retries with backoff — transient 429s usually self-heal

### Discovery returns no bindings or fewer than expected

**Cause**: The OAuth token's HubSpot user doesn't have access to the missing resources, or the HubSpot account doesn't have the feature enabled (e.g., custom objects require certain HubSpot tiers).

**Fix**:
1. Verify the HubSpot user has access to the missing resources (check their HubSpot permissions)
2. For custom objects, verify they exist in the HubSpot portal and the user can see them
3. Re-run `flowctl discover --source flow.yaml`

### Newly-created HubSpot records don't appear for ~1 hour

**Cause**: The connector runs *two* incremental subtasks per resource — a `realtime` cursor (minutes-fresh) and a `delayed` cursor lagging by ~1 hour to pick up rows HubSpot's API may emit late. For some resources or bulk-write scenarios, fresh rows can sit on the delayed cursor side until the lag elapses.

**Fix for testing or interactive demos**: Bump the `backfill` counter on the affected bindings and republish — Estuary re-reads via the realtime path and downstream materializations see the new state in seconds.

```yaml
bindings:
  - resource: { name: contacts }
    target: <TENANT>/<PATH>/contacts
    backfill: 1   # increment each time you want a re-pull
```

```bash
flowctl catalog publish --source flow.yaml --auto-approve
```

### `flowctl raw oauth` fails with "Type mismatch: expected a object"

**Cause**: A current parsing bug in `flowctl raw oauth` — the `--endpoint-config` argument fails JSON-Schema validation for any input (`{}`, JSON, YAML all fail with the same error). The command can't currently drive a HubSpot OAuth flow locally.

**Fix**:
- Complete the OAuth flow once in the Estuary web UI (it'll mint the `refresh_token` for you), then pull the published spec down to local with `flowctl catalog pull-specs`.
- If you need to run the flow locally (e.g. for an OAuth app you own), write a small helper that listens on the registered redirect URI and exchanges the code for tokens via `curl`. Use `curl` rather than Python's `urllib.request` — the python.org macOS Python distribution doesn't link the system trust store and `urlopen` fails the TLS handshake.

### Capture stuck in PENDING

Wait 30-60 seconds — this is normal during shard assignment. If still stuck:

```bash
flowctl logs --task <TENANT>/<PATH>/source-hubspot-native --since 5m | jq 'select(.level == "error")'
```

### Property history not appearing in documents

**Cause**: `capturePropertyHistory` is `false` by default.

**Fix**: Set `capturePropertyHistory: true` in the endpoint config and republish. Note this increases document size and capture volume.

## Related Skills

- `connector-disable-enable` — Pause/restart existing captures
- `connector-delete-recreate` — Nuclear option for stuck captures
- `estuary-logs` — Deep log analysis
- `estuary-catalog-status` — Status checking
