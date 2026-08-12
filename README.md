# Tonic AI skills

Reusable [Agent Skills](https://agentskills.io/) for working with Tonic AI
products from coding agents such as Claude Code, Cursor, Codex, GitHub Copilot,
Gemini CLI, and OpenCode.

Skills are portable instruction packages: each one teaches an agent how to
complete a specialized workflow safely and consistently. They contain no
credentials and do not run background services merely because they are
installed.

## Install

Install skills with the open-source
[`skills`](https://github.com/vercel-labs/skills) CLI:

```bash
npx skills add TonicAI/skills
```

The installer discovers the coding agents on your machine and lets you choose
which skills and agents to configure.

Useful variations:

```bash
# See the available skills without installing anything
npx skills add TonicAI/skills --list

# Install one skill
npx skills add TonicAI/skills --skill fabricate-database-profile

# Install globally, so the skill is available in every project
npx skills add TonicAI/skills --skill fabricate-database-profile --global

# Install for a specific agent without interactive prompts
npx skills add TonicAI/skills \
  --skill fabricate-database-profile \
  --agent claude-code \
  --global \
  --yes
```

Run `npx skills add TonicAI/skills --help` for all supported agents and options.

## Available skills

### `fabricate-database-profile`

Profiles a production or staging database locally and creates a PII-safe
statistical model for [Fabricate](https://fabricate.tonic.ai). Fabricate can
then generate realistic synthetic data for development, testing, and staging
without receiving production database credentials or raw rows.

The skill supports PostgreSQL, MySQL, Microsoft SQL Server, Oracle, Redshift,
Snowflake, BigQuery, SQLite, and databases exposing standard information-schema
metadata.

Install it directly:

```bash
npx skills add TonicAI/skills --skill fabricate-database-profile
```

To import the resulting profile automatically, connect your agent to the
Fabricate MCP server:

```json
{
  "mcpServers": {
    "fabricate": {
      "url": "https://fabricate.tonic.ai/api/v1/mcp"
    }
  }
}
```

Your MCP client starts Fabricate's OAuth flow the first time it connects.
Self-hosted users should replace the hostname with their Fabricate deployment.
MCP is optional: the generated `.fabricate-profile.json` file can also be
uploaded through the Profiles sidebar in Fabricate.

## Repository layout

```text
skills/
  <skill-name>/
    SKILL.md
    supporting files
```

Each subdirectory under `skills/` is independently installable. Product release
workflows publish only the skill namespaces they own, allowing this repository
to serve skills for multiple Tonic AI products without one product overwriting
another.

## Updates

Run the install command again, or use the CLI's update command:

```bash
npx skills update
```

## Feedback

Open an issue in this repository for installation or skill-content problems.
For product support, visit [Tonic AI](https://www.tonic.ai/).

<!-- fabricate-plugin:start -->
### Fabricate

Profile a production database locally, then import its PII-free statistical model
into Fabricate without giving Fabricate access to production.

Install only the portable skill:

```bash
npx skills add TonicAI/agents --skill fabricate-database-profile
```

Or install the skill and Fabricate MCP server together in Claude Code:

```text
/plugin marketplace add TonicAI/agents
/plugin install fabricate@tonic-ai
```

Cursor teams can import `TonicAI/agents` as a marketplace and install the
**Fabricate** plugin. See [`plugins/fabricate/README.md`](plugins/fabricate/README.md)
for usage and manual MCP configuration.
<!-- fabricate-plugin:end -->
