---
name: capture-http-ingest-create
description: Create an HTTP Ingest (webhook) capture to receive data via HTTP POST requests. Use when setting up webhooks from GitHub, Shopify, Stripe, or any JSON source. Use when user says "set up webhook capture", "HTTP ingest to Estuary", "receive POST requests to Estuary", or "webhook capture".
---

# Create HTTP Ingest (Webhook) Capture

Create an Estuary capture that accepts incoming HTTP requests (webhooks) from external services.

## Step 0: Load Connector Documentation

Before proceeding, fetch the official connector docs for config reference and webhook URL details:

**Primary:** https://docs.estuary.dev/reference/Connectors/capture-connectors/http-ingest/

Use WebFetch to load this page. The docs cover:

- Configuration reference (auth tokens, paths)
- Webhook URL format and retrieval
- Deduplication via `idFromHeader`
- Authentication setup

This skill provides the **flowctl workflow**, **service-specific guidance**, and **troubleshooting** that docs don't cover.

## Step 1: Gather Requirements

Before writing any YAML, ask the user:

1. **What service(s) are sending webhooks?** — GitHub, Shopify, Stripe, Segment, custom, etc.
2. **How many event types/paths?** — One webhook endpoint or multiple (e.g., `/orders`, `/customers`)
3. **Authentication method?** — HTTP Ingest only supports Bearer token auth. If the source requires OAuth callbacks or other auth flows, stop and inform the user this connector won't work for that source.
4. **Non-default data plane?** — Most users use the default. Ask if they need a non-default data plane (affects webhook URL domain).

### Key Difference from Database Captures

HTTP Ingest works in reverse — external services push TO Estuary, rather than Estuary pulling FROM a source:

- No firewall changes needed on the user's end
- Estuary provides the public HTTPS endpoint
- Authentication is via Bearer token
- **The webhook URL is only visible in the web UI after publishing** — it cannot be retrieved via CLI

## Step 2: Create the Capture Spec File

```yaml
captures:
  <tenant>/<path>/source-http-ingest:
    endpoint:
      connector:
        image: ghcr.io/estuary/source-http-ingest:v1
        config:
          require_auth_token: "<secure-random-token>"
          paths:
            - /<path-name>
    bindings: []
```

**Generate a secure token:**

```bash
openssl rand -base64 32
```

## Step 3: Configure Deduplication (Recommended)

After discovery, add `idFromHeader` to bindings for webhook services that provide unique delivery IDs:

| Service | `idFromHeader` Value   | Notes                          |
| ------- | ---------------------- | ------------------------------ |
| GitHub  | `X-Github-Delivery`    | Unique ID per delivery         |
| Shopify | `X-Shopify-Webhook-Id` | Unique ID per webhook          |
| Zendesk | `x-zendesk-webhook-id` | Case-sensitive                 |
| Stripe  | `Stripe-Signature`     | Contains timestamp + signature |
| Segment | `X-Request-Id`         | Standard request ID            |

Example with deduplication:

```yaml
bindings:
  - resource:
      path: /github-events
      stream: github-events
      idFromHeader: X-Github-Delivery
    target: <tenant>/<path>/github-events
```

## Step 4: Discover and Publish

```bash
# Discover (creates collection bindings for each path)
flowctl discover --source flow.yaml

# Review bindings, add idFromHeader if needed
cat flow.yaml

# Publish
flowctl catalog publish --source flow.yaml --auto-approve
```

## Step 5: Get the Webhook URL

**The webhook URL is only available in the Estuary web UI after publishing.**

Generate the dashboard link for the user from the catalog name:

```
https://dashboard.estuary.dev/captures/details/overview?catalogName=<tenant>/<path>/source-http-ingest
```

For example, if the capture is `acmeCo/webhooks/source-http-ingest`:
```
https://dashboard.estuary.dev/captures/details/overview?catalogName=acmeCo/webhooks/source-http-ingest
```

Provide this link to the user and tell them to:
1. Open the link (or click it)
2. Scroll to the **Endpoints** section
3. Copy the base URL (looks like `https://<unique-hostname>/`)
4. Append the configured path (e.g., `/github-events`)

**Full URL format**: `https://<unique-hostname>/<your-path>`

## Step 6: Configure the Webhook Sender

If the user named a specific source system (e.g., Shopify, GitHub, Stripe), use WebSearch to find that service's webhook configuration docs. Help the user with both sides of the setup — the Estuary capture AND the source-side webhook registration.

In the external service:

1. Set webhook URL to the Estuary endpoint from Step 5
2. Set Content-Type to `application/json`
3. Configure authentication header: `Authorization: Bearer <your-token>`

## Step 7: Test and Verify

```bash
# Test with curl
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-token>" \
  -d '{"event": "test", "data": {"key": "value"}}' \
  "https://<your-endpoint-url>/<path>"

# Check status
flowctl catalog status <tenant>/<path>/source-http-ingest

# Read ingested data
flowctl collections read --collection <tenant>/<path>/<stream-name> --uncommitted | head -10
```

## Troubleshooting

### 401 Unauthorized

**Cause**: Token doesn't match `require_auth_token` config

**Fix**: Verify token matches exactly. Check for trailing spaces/newlines. Format must be `Bearer <token>` (with space).

### 404 Not Found

**Cause**: URL path doesn't match configured paths

**Fix**: Check path spelling (case-sensitive), ensure path starts with `/` in config, don't include query parameters.

### 405 Method Not Allowed

**Cause**: Using wrong HTTP method (e.g., GET)

**Fix**: Use POST or PUT.

### "Parse Error: Expected HTTP/"

**Cause**: HTTP client doesn't properly support HTTP/2

**Fix**: Force HTTP/1.1 if possible: `curl --http1.1 -X POST ...`

### 500 Internal Server Error during high volume

**Cause**: Schema inference updates can cause brief connector restarts

**Fix**: Implement retry logic. Most webhook services (GitHub, Stripe, etc.) automatically retry failed deliveries. Once schema stabilizes, this stops happening.

### Webhook service doesn't support Bearer tokens

**Cause**: Some services (like SendGrid) only support OAuth or signature-based auth

**Workaround**: Use middleware (AWS API Gateway, Cloudflare Workers) to translate auth, or check if Estuary has a dedicated connector for that service.

### Data not appearing in collection

**Diagnostic steps:**

1. Check capture status: `flowctl catalog status <task>`
2. Test with curl to verify endpoint works
3. Check logs: `flowctl logs --task <task> --since 10m`
4. Verify webhook sender shows successful delivery

### Capture stuck in PENDING

Wait 30-60 seconds — this is normal during shard assignment. If still stuck:

```bash
flowctl logs --task <tenant>/<path>/source-http-ingest --since 5m | jq 'select(.level == "error")'
```

## Related Skills

- `connector-disable-enable` — Pause/restart existing captures
- `estuary-logs` — Deep log analysis
- `estuary-catalog-status` — Status checking
