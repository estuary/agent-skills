# Estuary Agent Skills

Set up Estuary data pipelines through your AI assistant. No docs to stitch together, no flowctl commands to remember.

Ask your AI to "capture my Postgres into Snowflake" or "why is this materialization failing?" — these skills give it everything it needs to run the right commands, build the right specs, and explain the gotchas.

Works with [Claude Code, GitHub Copilot, Cursor, OpenAI Codex, Gemini CLI, and 30+ other tools](https://agentskills.io) via the open [SKILL.md](https://agentskills.io) standard.

## What's included

### Captures (sources)

Capture data from databases, APIs, and webhooks into Estuary collections.

| Skill | Description |
|-------|-------------|
| `capture-postgres-create` | PostgreSQL CDC (vanilla, RDS, Aurora, Cloud SQL, Supabase, Neon) |
| `capture-mysql-create` | MySQL CDC via binlog replication (RDS, Aurora, Cloud SQL, Azure) |
| `capture-mongodb-create` | MongoDB CDC (Atlas, DocumentDB, self-hosted) |
| `capture-sqlserver-create` | SQL Server CDC (RDS, Azure SQL, Cloud SQL) |
| `capture-http-ingest-create` | HTTP webhook capture (GitHub, Shopify, Stripe, or any JSON source) |
| `capture-generic-create` | Any of 148+ source connectors via dynamic schema discovery |

### Materializations (destinations)

Stream Estuary collections into downstream databases and warehouses.

| Skill | Description |
|-------|-------------|
| `materialize-postgres-create` | PostgreSQL destination |
| `materialize-snowflake-create` | Snowflake destination (JWT auth) |
| `materialize-bigquery-create` | BigQuery destination (GCS staging) |
| `materialize-redshift-create` | Amazon Redshift destination (S3 staging) |
| `materialize-databricks-create` | Databricks destination (Unity Catalog) |
| `materialize-generic-create` | Any destination connector via dynamic schema discovery |

### Derivations (transformations)

Transform, aggregate, and reshape Estuary collections with streaming SQL, TypeScript, or Python.

| Skill | Description |
|-------|-------------|
| `derivation-basics` | Foundation: what derivations are, language choice, project layout, and workflow — read first |
| `derivation-filter-transform` | Stateless filtering, field selection, and per-document field transformation |
| `derivation-aggregate-metrics` | Daily totals, running counts, min/max, and lifetime metrics via reduction annotations |
| `derivation-join-collections` | Join two or more collections on a shared key into an enriched collection |
| `derivation-flatten-array` | Flatten a nested array into one output document per element |
| `derivation-stateful-logic` | Custom SQLite state for balances, inventory, approval workflows, and deduplication |
| `derivation-windowing` | Sliding time-window state (e.g. last-24h events) via `readDelay` expiration |
| `derivation-python` | Write derivations in Python (async transforms, Pydantic types, `uv` deps) for ML/embeddings/async APIs |

### Operations

Manage and troubleshoot running pipelines.

| Skill | Description |
|-------|-------------|
| `estuary-flowctl-setup` | Install, authenticate, and update the flowctl CLI |
| `estuary-task-health` | End-to-end health check for a task: status, data flow, errors, and history |
| `estuary-catalog-status` | Check control-plane status of a task with `flowctl catalog status` |
| `estuary-task-stats` | Inspect data volume, document counts, and hourly throughput for a task |
| `estuary-catalog-history` | Review publication history and recent spec changes |
| `estuary-logs` | Search and analyze task logs with jq filtering |
| `estuary-connector-restart` | Pause and restart connectors via shard management |
| `estuary-ssh-tunnels` | Diagnose and fix SSH tunnel connection issues |

## Prerequisites

- An [Estuary](https://dashboard.estuary.dev/register) account
- [flowctl](https://docs.estuary.dev/getting-started/installation/) CLI — the `estuary-flowctl-setup` skill walks you through installation and authentication

## Installation

### Skills CLI

Install all skills at once:

```bash
npx skills add estuary/agent-skills
```

### Claude Code

Add the Estuary marketplace:

```bash
/plugin marketplace add estuary/agent-skills
```

Then install by group:

```bash
/plugin install estuary-captures@estuary
/plugin install estuary-materializations@estuary
/plugin install estuary-derivations@estuary
/plugin install estuary-operations@estuary
```

Or run `/plugin` to browse from the Discover tab. Installed skills auto-update when the marketplace refreshes.

### Manual installation

Clone this repo and copy the skill folders into your AI tool's skill directory:

```bash
git clone https://github.com/estuary/agent-skills.git
cp -r agent-skills/skills/* your-project/.claude/skills/
```

Common paths by tool:

| Tool | Path |
|------|------|
| Claude Code | `.claude/skills/` |
| Cursor | `.cursor/skills/` |
| GitHub Copilot / VS Code | `.github/skills/` |
| OpenCode | `.opencode/skills/` |
| Codex | `.codex/skills/` |

## Usage

Once installed, ask your AI assistant in plain English:

> "Capture my Postgres database into Estuary"
>
> "Materialize my collections into Snowflake"
>
> "Capture from MySQL and materialize into Redshift"
>
> "Why is my materialization failing?"

## Resources

- [Estuary documentation](https://docs.estuary.dev)
- [Estuary MCP integration](https://docs.estuary.dev/features/mcp-integration/)
- [flowctl CLI installation](https://docs.estuary.dev/getting-started/installation/)
- [Connector catalog](https://estuary.dev/integrations/)
- [SKILL.md standard](https://agentskills.io)

## Feedback and support

- Open an [issue](https://github.com/estuary/agent-skills/issues) in this repo
- Join the [Estuary Slack community](https://go.estuary.dev/slack)
- Contact [support@estuary.dev](mailto:support@estuary.dev)

## License

Apache 2.0 — see [LICENSE](LICENSE) for details.
