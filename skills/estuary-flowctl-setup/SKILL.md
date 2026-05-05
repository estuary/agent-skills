---
name: estuary-flowctl-setup
description: Install and authenticate the flowctl CLI for Estuary. Use when setting up flowctl for the first time, troubleshooting authentication, upgrading to the latest version, or configuring programmatic access for CI/CD. Use when user says "install flowctl", "set up flowctl", "flowctl auth", "how to authenticate", "token expired", "not authenticated", "FLOW_AUTH_TOKEN", "CI/CD setup", "programmatic access", "upgrade flowctl", "flowctl not found", "command not found flowctl", "Exec format error", or "flowctl on Windows".
---

# flowctl Setup — Install, Authenticate, and Update

**flowctl** is Estuary's CLI for managing captures, materializations, collections, and derivations from the command line.

## Install

### Mac (Homebrew — recommended)

```bash
brew tap estuary/flowctl
brew install flowctl
```

### Mac (direct download)

```bash
sudo curl -o /usr/local/bin/flowctl -L 'https://github.com/estuary/flow/releases/latest/download/flowctl-multiarch-macos' && sudo chmod +x /usr/local/bin/flowctl
```

### Linux (x86-64)

```bash
sudo curl -o /usr/local/bin/flowctl -L 'https://github.com/estuary/flow/releases/latest/download/flowctl-x86_64-linux' && sudo chmod +x /usr/local/bin/flowctl
```

### Windows

No native Windows build exists. Use one of:

1. **WSL (recommended)** — Install [Windows Subsystem for Linux](https://learn.microsoft.com/en-us/windows/wsl/), then use the Linux install command above inside your WSL shell.
2. **Remote dev environment** — Run flowctl on a cloud Linux VM and connect via your IDE's remote development features.

**Note:** On ARM-based Windows machines (e.g., Snapdragon/aarch64 under WSL), the x86-64 binary won't work (`Exec format error`). There is no official ARM build yet — use a remote x86-64 environment or build from source.

## Authenticate

### Interactive login (local development)

```bash
flowctl auth login
```

This opens your browser to the Estuary dashboard's CLI-API tab. Copy the access token and paste it into the terminal.

**Note:** Access tokens are short-lived (~1 hour). You'll need to repeat this when the token expires.

### Programmatic access (CI/CD and automation)

For non-interactive environments, use a long-lived refresh token:

1. Go to https://dashboard.estuary.dev/admin/api and generate a **refresh token**
2. Set the environment variable:
   ```bash
   export FLOW_AUTH_TOKEN=<your-refresh-token>
   ```
3. Run flowctl commands normally — authentication is handled automatically

**Important:** If a config file exists for the current profile, flowctl ignores `FLOW_AUTH_TOKEN`. Use a separate profile to avoid conflicts:
```bash
flowctl --profile ci catalog list
```

The refresh token expires after 90 days of inactivity — each use resets the clock.

**Common CI mistake:** Don't run `flowctl auth token --token $FLOW_AUTH_TOKEN` in your scripts — that command is for interactive token exchange only. Just set the env var and run flowctl commands directly.

## Verify Installation

```bash
flowctl --version
```

Quick test that authentication works:
```bash
flowctl catalog list --output json | head -5
```

## Update

### Homebrew

```bash
brew update && brew upgrade flowctl
```

### Direct download

Re-run the install command for your platform — it overwrites the existing binary.

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `command not found: flowctl` | Re-run the install command, or check that `/usr/local/bin` is in your `PATH` |
| `You are not authenticated` | Run `flowctl auth login` — your token likely expired |
| `Exec format error` | Wrong binary for your architecture — ARM machines need an x86-64 environment (see Windows section) |
| `FLOW_AUTH_TOKEN` ignored | An existing profile config takes precedence — use `flowctl --profile ci <command>` |
| `Header of type authorization was missing` | Token didn't save correctly — re-run `flowctl auth login` and paste carefully |
| `Failed to locate sops` | Only needed for local secret encryption — install [sops](https://github.com/getsops/sops/releases) if using encrypted configs |

Check for the latest releases and changelogs: https://github.com/estuary/flow/releases

## Related Skills

- **estuary-logs** — Search and analyze task logs with flowctl
- **estuary-catalog-status** — Check whether tasks are running, disabled, or failed
- **estuary-connector-restart** — Pause and restart connectors via flowctl
