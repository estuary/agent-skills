---
name: estuary-ssh-tunnels
description: Diagnose and fix SSH tunnel connection issues for Estuary captures and materializations. Use when seeing tunnel failures, connection timeouts, or permission denied errors. Use when user says "SSH tunnel failed", "connection timeout via tunnel", "permission denied SSH", "tunnel not working", "network-tunnel failed", "no pubkey loaded", "bastion host not connecting", or "RSA key format".
---

# SSH Tunnel Troubleshooting for Estuary

## Quick Checklist

Before deep debugging, verify these common issues:

1. **SSH Endpoint format**: Must be `ssh://user@hostname:port`
   - Check for typos, spaces, missing hyphens
   - AWS regions use hyphens: `us-west-2` not `us-west 2`

2. **Private key format**: Must be RSA PEM format
   - Must start with `-----BEGIN RSA PRIVATE KEY-----`
   - NOT `-----BEGIN OPENSSH PRIVATE KEY-----` (OpenSSH format)
   - NOT `-----BEGIN PRIVATE KEY-----` (PKCS#8 format)
   - Ed25519 and ECDSA keys are **not supported** — only RSA

3. **IP allowlisting**: Estuary data plane IPs must be allowed on:
   - Bastion/SSH server security group (port 22)
   - Database security group (from bastion's private IP)
   - Find the IPs for your data plane at Admin → Settings → Data Planes in the Estuary UI
   - Note: data plane IPs can change after infrastructure scaling — re-verify allowlists if a previously working tunnel stops connecting

4. **Data plane selection**: The user should verify the correct data plane is selected in the Estuary UI when editing the connector. Each data plane has different egress IPs — using the wrong one means the wrong IPs are connecting.

## Testing SSH Connectivity

If the quick checklist doesn't resolve the issue, instruct the user to test SSH connectivity from a machine that can reach the bastion host (e.g., their local machine or another server in the same network):

### Test SSH connection
```bash
ssh -i /path/to/private_key -o StrictHostKeyChecking=no -v user@bastion-host -p 22
```
A successful connection confirms the key, username, and network path are correct.

### Test the tunnel
```bash
ssh -i /path/to/private_key -o StrictHostKeyChecking=no \
    -L 3306:database-endpoint:3306 \
    user@bastion-host -N -v &
```
Then test the database connection through the tunnel (substitute the client and port for the relevant database type — MySQL 3306, PostgreSQL 5432, SQL Server 1433):

**Important:** The database address used here (and in the connector's `address` config field) must be the address as seen **from the bastion host** — not from the user's local machine. If the database is on a private subnet visible to the bastion (e.g., `db.internal:5432`), use that address.

```bash
mysql -h 127.0.0.1 -P 3306 -u dbuser -p
```

If this works locally but Estuary's connector fails, the issue is likely IP allowlisting — the user's machine can reach the bastion but Estuary's data plane IPs cannot.

**Note:** If the **Test Connection** button fails but the configuration looks correct, try **Save and Publish** anyway — test connectivity is sometimes stricter than the runtime connection.

## Error Reference

| Error Message | Cause | Solution |
|---------------|-------|----------|
| `network-tunnel failed` | Generic tunnel failure | Check SSH endpoint format, key format, firewall rules |
| `network-tunnel ready` but connector still failing | Known false positive — this log can appear even when the tunnel failed | Check log entries *before* "tunnel ready" for the real error |
| `Permission denied (publickey)` | Auth failure | Check key format (must be RSA PEM), username, key content — also check for trailing whitespace or extra newlines in the pasted key |
| `no pubkey loaded` | Key format issue | Convert key to RSA PEM format (see below) |

## Key Format Conversion

If the private key starts with `-----BEGIN OPENSSH PRIVATE KEY-----`, it needs to be converted to RSA PEM format:

```bash
ssh-keygen -p -N "" -m pem -f /path/to/key
```

To generate a new compatible key from scratch:
```bash
ssh-keygen -m PEM -t rsa -b 4096 -f mykey
```

Then add the public key (`mykey.pub`) to `~/.ssh/authorized_keys` on the bastion host.

## Common Endpoint Format Mistakes

```
# WRONG - space in region
ssh://estuary@ec2-35-87-112-173.us-west 2.compute.amazonaws.com:22

# WRONG - missing hyphen
ssh://estuary@ec2-35-87-112-173.uswest2.compute.amazonaws.com:22

# CORRECT
ssh://estuary@ec2-35-87-112-173.us-west-2.compute.amazonaws.com:22
```

## Related Skills

- **estuary-logs** — Start here to pull task logs and find the exact tunnel error message and timestamps
- **estuary-catalog-status** — Check whether the task is running, failed, or disabled before debugging the tunnel
