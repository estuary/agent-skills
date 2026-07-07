---
name: schema-field-redaction
description: Add redaction annotations to sensitive fields in schemas to block or hash PII. Use when configuring GDPR/HIPAA compliance, protecting credentials, or blocking internal fields from materializations. Use when user says "protect PII", "hide sensitive data", "hash field", "redact field", "GDPR compliance", "don't capture field", "remove field from Estuary", or "prevent Estuary from capturing field".
---

# Protect Sensitive Data with Field Redaction

Add redaction annotations to your collection schema to protect sensitive data at the source. Redacted fields are blocked or hashed before documents are stored or materialized.

## When to Use

- **PII Protection**: Hash emails, SSNs, phone numbers for GDPR/HIPAA/CCPA compliance
- **Credential Security**: Block passwords, API keys, tokens from ever leaving the capture
- **Internal Fields**: Remove internal-only metadata before materialization
- **Data Minimization**: Ensure only necessary data reaches downstream systems

## How Redaction Works

Redaction is applied at write time — sensitive data never reaches Estuary storage, logs, or downstream materializations. This is the earliest point to remove fields, before derivations or materialization field selection.

Redaction only applies to documents captured **after** publishing the annotation. To redact historical data, trigger a dataflow reset backfill on the capture.

## Redaction Strategies

| Strategy | Effect | Use When |
|----------|--------|----------|
| `block` | Removes the field entirely | Field should never exist downstream |
| `sha256` | Replaces value with salted hash | Need to join/match on field (hash is deterministic per task) |

## Prerequisites

- `flowctl` installed and authenticated
- Write access to the collection you want to modify
- Know which fields contain sensitive data

## Key Constraints

1. **Cannot redact key fields** — Would break collection semantics.
2. **`block` on required fields** — Validation fails with `` `block` redact strategy cannot be applied at '/path/to/field' because it must exist`` (the path is a JSON Pointer to the field) UNLESS the read schema references the inferred schema (`flow://inferred-schema`), in which case the validator suppresses that error automatically. For explicit schemas, remove the field from `required` first (see Troubleshooting).
3. **`sha256` on non-string-typed fields** — At runtime the hasher accepts any type. But at publish time, validation fails with ``/path/to/field has 'sha256' redact strategy but cannot be a string (types are "<type>")`` if the field's declared type set has no overlap with `string`. A union like `["string","null"]` or `["string","integer"]` passes — only a field that cannot be a string at all (e.g., pure `integer` or pure `object`) is rejected.

## Procedure

### Step 1: Pull the Collection Spec

```bash
flowctl catalog pull-specs --name <tenant>/<collection-name>
```

This creates a local directory with `flow.yaml` and potentially referenced schema files.

### Step 2: Add Redaction Annotations

Add `redact` annotations to the **writeSchema** (not the readSchema — this is a write-time operation). If the collection uses a simple `schema` (no separate write/read), add annotations there instead.

**Placement depends on how the writeSchema is structured:**

- **Captures using `$ref: flow://connector-schema`** (most SQL captures): add `redact` annotations at the top level of the writeSchema, *alongside* the `$ref` — this ensures they survive auto-discovery updates of the connector-schema.
- **Captures with an inline writeSchema** (HTTP ingest, MongoDB, or older SQL captures): annotate the field directly on its existing property definition. The `redact:` key lives alongside `type:`, `description:`, etc.

**Example - Hash PII, block credentials (connector-schema capture):**

```yaml
collections:
  acmeCo/customers:
    writeSchema:
      $ref: flow://connector-schema
      properties:
        # Hash for pseudonymization (can still join on hashed value)
        email:
          redact: { strategy: sha256 }
        phone:
          redact: { strategy: sha256 }
        ssn:
          redact: { strategy: sha256 }
        # Block entirely (field removed from documents)
        password_hash:
          redact: { strategy: block }
        internal_notes:
          redact: { strategy: block }
```

**Example - Inline writeSchema (annotate field directly):**

```yaml
collections:
  acmeCo/users:
    writeSchema:
      type: object
      required: [id]
      properties:
        id:
          type: string
        email:
          type: string
          redact: { strategy: sha256 }
        api_key:
          type: string
          redact: { strategy: block }
```

**Example - Nested fields:**

```yaml
properties:
  user:
    type: object
    properties:
      profile:
        type: object
        properties:
          email:
            type: string
            redact: { strategy: sha256 }
          private_key:
            redact: { strategy: block }
```

### Step 3: Publish the Changes

```bash
flowctl catalog publish --source flow.yaml --auto-approve
```

### Step 4: Verify Redaction

After publishing, new documents will have redaction applied. Check by reading the collection:

```bash
flowctl collections read --collection <collection> --uncommitted | head -5
```

**Expected output:**
- `sha256` fields show hashed values: `"email": "sha256:a3f2b8c9d4e5..."`
- `block` fields are absent from documents entirely

## Troubleshooting

### "Cannot redact key field"

**Cause**: Attempted to redact a field that's part of the collection key.

**Solution**: Key fields cannot be redacted. Consider:
1. Using a different field as the key
2. Creating a derivation that re-keys documents before redaction

### "`block` redact strategy cannot be applied at '/path/to/field' because it must exist"

**Cause**: `block` removes the field, but the schema requires the field to be present (it's listed in `required:`, or otherwise constrained to exist).

**When this errors and when it doesn't**:
- **Read schemas using `flow://inferred-schema`**: the validator suppresses this error automatically. Publish proceeds and `block` simply makes the field absent in captured documents — schema inference will eventually drop the field from required.
- **Explicit schemas (no inferred-schema reference)**: validation fails. Remove the field from `required` before publishing:

```yaml
required: [id]  # Remove the field you're blocking
properties:
  id: { type: string }
  sensitive_field:
    type: string
    redact: { strategy: block }
```

### "/path/to/field has 'sha256' redact strategy but cannot be a string (types are \"<type>\")"

**Cause**: The field's declared type set excludes `string` entirely (e.g., `type: integer` or `type: object`). At publish time, validation rejects `sha256` on a field that can never be a string.

**Note**: At runtime the hasher itself accepts any type (it hashes bool/int/float bytes, JSON-serializes arrays/objects before hashing). The error is a publish-time schema-shape check, not a runtime restriction. A union like `["string", "null"]` or `["string", "integer"]` passes — only a field declared to never be a string fails.

**Solutions**:
- Add `string` to the field's type union: `type: ["string", "null"]`
- Or convert to a string in a derivation first and apply `sha256` on the derived field
- For non-string fields you don't need to look up by, use `block` instead

### Redaction not appearing on existing documents

**Cause**: Redaction only applies to new documents captured after publishing.

**Solution**: If you need to redact historical data:
1. Increment the `backfill` counter on the capture binding
2. This re-captures all data with redaction applied

### Hash values differ between tasks

**Cause**: Each capture/derivation has its own salt for `sha256` hashing.

**Explanation**: This is intentional for security. The same email hashed in different captures will produce different hashes.

**If you need consistent hashes across tasks**: Estuary auto-generates a per-task salt, but you can override it by setting the top-level `redactSalt` field on the capture spec or under a derivation's `derive:` block. The value is a base64-encoded byte string (e.g., `"c29tZS1zZWNyZXQtc2VjcmV0LWFiYw=="` — use a high-entropy secret in production). Specifying the same `redactSalt` across tasks produces matching hashes for matching inputs. See [docs/features/redaction](https://docs.estuary.dev/features/redaction/).

Hash output format: `sha256:` + 64 lowercase hex characters (71 chars total).

## Related Skills

- `schema-field-selection` — Exclude fields at materialization (less secure, field still in collection)
- `schema-define-fields` — Define fields in the readSchema alongside connector-schema reference
