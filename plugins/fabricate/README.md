# Fabricate skills

Profile a production database from your own machine, then generate matching
synthetic data with [Fabricate](https://fabricate.tonic.ai) — **without giving
Fabricate access to production**.

Your coding agent connects to the database, computes schema and aggregate
statistics locally, and sends Fabricate only the resulting statistical model. No
credentials and no rows leave your machine.

```
production database  <->  your machine  ->  .fabricate-profile.json  ->  Fabricate
```

## What's in the box

- **`fabricate-database-profile` skill** — teaches your agent to profile any
  database Fabricate supports (PostgreSQL, MySQL, SQL Server, Oracle, Redshift,
  Snowflake, BigQuery, SQLite, or anything with an `information_schema`) using
  whatever connectivity you already have, and to emit a PII-free profile export.
- **Fabricate MCP server** — connects your agent to Fabricate so it can import
  the profile and drive data generation.

## Install

Works with Claude Code, Cursor, Codex, OpenCode, Gemini CLI, Copilot, and
[30+ other agents](https://github.com/vercel-labs/skills#supported-agents):

```bash
npx skills add TonicAI/agents
```

Add `-g` to install for every project, or `--agent claude-code` to target one
agent. This installs the skill only — see [Connect Fabricate](#connect-fabricate)
below for the MCP server.

### As a plugin (skill + MCP server in one step)

**Claude Code**

```
/plugin marketplace add TonicAI/agents
/plugin install fabricate@tonic-ai
```

**Cursor** — import `TonicAI/agents` under Dashboard → Settings → Plugins →
Import Marketplace, then install **Fabricate**. Listing it in Cursor's public
Marketplace requires a separate Cursor review.

### Manually

Copy `skills/fabricate-database-profile/` into your skills directory
(`~/.claude/skills/` for personal use, or `.claude/skills/` inside a project).

## Connect Fabricate

The skill profiles your database on its own, but importing the result needs
Fabricate's MCP server. The plugin install above configures it for you; with
`npx skills add` or a manual copy, add it to your MCP client config yourself:

```json
{
  "mcpServers": {
    "fabricate": { "url": "https://fabricate.tonic.ai/api/v1/mcp" }
  }
}
```

**Self-hosted Fabricate:** replace that URL with `https://<your-fabricate-host>/api/v1/mcp`.

Your client walks you through Fabricate's OAuth consent screen on first use — no
keys to copy. You can also skip MCP entirely and upload the generated
`.fabricate-profile.json` through the Profiles sidebar in your Fabricate project.

## Use it

Ask your agent:

> Profile our staging Postgres database for Fabricate and import it into the
> Checkout project.

The agent will establish a read-only connection, inventory the schema, compute
distributions and correlations, write `<name>.fabricate-profile.json`, and
import it. You can also stop at the file and import it later.

## What ends up in a profile

Plain-English statements about schema and statistics, and nothing else:

```json
{
  "content": "orders.status distribution: paid 62%, shipped 24%, pending 9%, cancelled 5%.",
  "finding_type": "distribution",
  "tags": ["table:orders", "column:orders.status"]
}
```

The skill forbids real values from the source data — no names, emails, addresses,
identifiers, or verbatim samples. Columns holding personal data are described by
format and statistics ("emails are 100% unique, mean length 24") rather than
content. The one exception is small non-sensitive lookup tables such as currency
codes, and only when you explicitly approve the snapshot.

See [`skills/fabricate-database-profile/SKILL.md`](skills/fabricate-database-profile/SKILL.md)
for the full methodology and
[`reference.md`](skills/fabricate-database-profile/reference.md) for the export
format and per-platform notes.

## Docs

- [Fabricate MCP server](https://github.com/TonicAI/fabricate/blob/main/docs/mcp.md)
- [Fabricate](https://fabricate.tonic.ai)

---

Generated from the Fabricate repository on each release. Open issues against
[TonicAI/agents](https://github.com/TonicAI/agents).
