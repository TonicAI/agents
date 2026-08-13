---
name: fabricate-database-profile
description: Profiles a production or staging database locally and produces a Fabricate profile export (.fabricate-profile.json) containing only schema and aggregate statistics, then optionally imports it into a Fabricate project over MCP. Use when the user asks to profile a database for Fabricate, build a statistical model of production data for synthetic data generation, or create/import a Fabricate profile without connecting Fabricate to production.
metadata:
  version: 4.23.0
---

# Fabricate database profiling

Build a statistical model of a database that Fabricate can use to generate realistic synthetic data in lower environments — **without connecting Fabricate to that database**. You connect to the database; Fabricate only ever receives aggregate statistics.

## Trust boundary

```
production database  <->  you (this machine)  ->  .fabricate-profile.json  ->  Fabricate
```

- Credentials never leave this machine and are never written into the profile, generated source files, logs, or chat.
- Raw rows never leave the database client process.
- The profile contains **only** schema metadata and aggregate statistics.

## Workflow

```
- [ ] Step 1: Establish read-only connectivity
- [ ] Step 2: Inventory the schema
- [ ] Step 3: Collect aggregate statistics
- [ ] Step 4: Write findings
- [ ] Step 5: Emit and validate the profile export
- [ ] Step 6: Import into Fabricate (optional)
```

### Step 1: Establish read-only connectivity

There is no bundled profiling tool. Use whatever already works on this machine, in this order:

1. A database-native CLI (`psql`, `mysql`, `sqlcmd`, `sqlplus`, `snowsql`, `bq`, `sqlite3`, …).
2. An existing project dependency, SDK, or configured connection in the repo.
3. A database tool provided by this coding environment (for example an MCP server).
4. A short throwaway program you write, using the database's maintained driver.

Ask the user which database and connection to use if it is not obvious. Infer the platform from the connection string when you can.

Then enforce these before running anything (see [reference.md](reference.md) for per-platform syntax):

- Use a **read-only database role**. Ask the user to provide one if the available credentials are read-write, and warn them explicitly if they decline.
- Set a read-only transaction/session and a **statement timeout** on every session.
- Quote all identifiers you interpolate; never build SQL from unvalidated names.
- Confirm connectivity with a trivial query before profiling.

### Step 2: Inventory the schema

Collect, per table: columns with **native** types, nullability, defaults, primary/foreign/unique constraints, indexes, and an approximate or exact row count. Ask which schemas/tables to include when the database is large, and record the filter you applied.

Preserve platform-specific detail that changes how data must be generated — identity/sequence behavior, schemas and catalogs, partitioning and clustering, semi-structured column types, collation and case sensitivity, and engine-specific default expressions. Record it rather than normalizing it away.

### Step 3: Collect aggregate statistics

Compute these **in the database** with aggregate SQL wherever possible:

| Statistic | Applies to |
|---|---|
| Null ratio, distinct ratio, most-common frequency and its share (without the value) | every column |
| Min, max, mean, standard deviation | numeric and temporal columns |
| Min, max, mean length | text columns |
| Value frequency distribution (top ~30 buckets, including labels) | low-cardinality, non-sensitive categorical columns only |
| Range histogram (~5–30 buckets) | high-cardinality numeric and temporal columns |
| Pearson correlation (report when \|r\| >= 0.3) | numeric and temporal column pairs |

Run that same set against **four different result sets, not just the table**. Statistics over one table at a time describe columns in isolation, and almost everything that makes generated data feel real lives between columns and between tables. See [reference.md](reference.md) for portable SQL for each shape.

1. **The table** — every column, as above.
2. **The child joined to its parent** — profile a foreign key alongside columns from the referenced row. This is how you find rules like "the card charged always belongs to the customer who placed the order".
3. **One row per parent, carrying `COUNT(children)`** — then treat that count as a column and bin it. Report the whole fan-out **distribution** ("1 order 38.5%, 2 orders 33.8%, 3 orders 16.9%, 4 orders 10.8%"), not only the min, max and mean, along with the share of parents that have no children at all.
4. **Derived expressions** — `quantity * unit_price`, `child.created_at - parent.created_at`, `now() - customer.created_at`. A derived value is a column like any other: profile it.

Then look for the relationships a per-column pass cannot see:

- **Conditional statistics.** Average a measure within each category (order value per country, per status, per age band), and cross-tabulate pairs of categorical columns. Report the flat results too — "the refund rate is ~2% within every order status" tells Fabricate to draw the two independently, which is as useful as finding a correlation.
- **Cross-table temporal rules.** For every foreign key, compare the child's timestamps against the parent's and report how many rows violate the expected ordering. This is where both "every order falls after its customer signed up" and "128 payments predate the card they charge" come from.
- **Format rules per group.** When a format depends on another column — card number length and leading digit per brand, identifier prefix per type — profile the pattern within each group rather than across the column as a whole.

Only return category labels when the grouping columns are already known to be low-cardinality and non-sensitive. For names, emails, identifiers, free text, or any uncertain column, return only value-free frequency shapes, derived format buckets, or rank labels such as `top_1`; never select the underlying value into your output or context.

**Quantify every rule you assert.** Never write "every", "always" or "never" without the count behind it: say "N of M rows satisfy this" and give the number of exceptions. A rule holding for 995 of 1,000 rows is a different generation instruction from one holding for all 1,000, and asserting the absolute without counting it is how a profile ends up simply wrong.

Sample rather than scanning whole tables: draw a bounded random sample (Fabricate's own profiler defaults to 5,000 rows) per table and note the sample size in the findings. Prefer platform-native sampling (`TABLESAMPLE`, `SAMPLE`, `ORDER BY RANDOM()`) over full scans on large tables.

If you must compute a statistic outside the database, keep raw rows inside your program, write only aggregates to a temporary file, never print raw rows to stdout or read them into context, and delete any temporary program and data file once the profile is written.

### Step 4: Write findings

A profile is a list of **findings**: self-contained, plain-English statements about the schema and data. See [reference.md](reference.md) for the finding types, tag rules, and worked examples.

Every finding needs:

- `content` — a complete sentence that makes sense on its own, including the concrete numbers.
- `finding_type` — one of `ddl`, `description`, `distribution`, `correlation`, `constraints`, `cardinality`, `anomaly`, or `reference_rows`.
- `tags` — use only `table:<table_name>` and `column:<table_name>.<column_name>`. These are the controlled namespaces Fabricate's agents understand; do not invent others.

`reference_rows` is the only type that carries real values, and the only type that may carry a `data` payload. It requires explicit user approval and a small, non-sensitive lookup table — see the `reference_rows` section in [reference.md](reference.md). Every other type describes the data and must omit `data` entirely.

**Separate what the data does from what it should do.** A source database — especially one already seeded or generated — carries artifacts nobody wants reproduced: every parent having at least one child, a column perfectly unique where real data would repeat heavily, a hard cap truncating a distribution, statuses drawn independently of the dates around them. Record these as `anomaly` findings that name the artifact and say what real data would look like instead. Writing one up as a `constraints` finding instead turns a flaw in the source into a requirement, and Fabricate will faithfully reproduce it.

### Step 5: Emit and validate the profile export

First check coverage, while a thin profile is still cheap to fill in:

- Every table has a `ddl` finding and a `description` finding carrying its row count.
- Every foreign key has a `cardinality` finding carrying a fan-out distribution, not just a mean.
- Every column appears in at least one finding beyond its table's `ddl`.
- Every rule phrased as an absolute carries the row counts behind it.
- The arithmetic reconciles. Bucket counts sum to the population they describe, shares sum to 100%, and any total you state matches the per-table counts you recorded. Add the numbers up rather than trusting a figure you carried over from an earlier step.

Then write `<profile-name>.fabricate-profile.json` in the format documented in [reference.md](reference.md). Before handing it over, verify: the export marker is present, every `finding_type` is valid, every tag uses an accepted namespace, the profile name does not start with `db:`, all numbers are finite, and `data` appears only on `reference_rows` findings the user approved in Step 4.

No real values from the source data may appear anywhere else. Do not silently drop a finding to satisfy this check — an approved `reference_rows` snapshot belongs in the export, and anything else that fails validation is a bug to fix and report, not to delete.

Then summarize for the user: tables profiled, finding count by type, and anything you deliberately skipped.

### Step 6: Import into Fabricate (optional)

If a Fabricate MCP server is connected, call `import_profile` with the target `project_id` and the parsed JSON object as `profile_export`. Use `list_workspaces` and `list_projects` to find the project.

Importing appends by default, so a retry would leave two copies of every finding. Pass `replace: true` if you are retrying a call whose outcome you are unsure of, or re-profiling a database this profile name already covers.

If it is not connected, do not stop at the file — offer to connect it, since installing this skill on its own does not add the MCP server. Add this to the user's MCP client config and let it run Fabricate's OAuth flow on first use:

```json
{
  "mcpServers": {
    "fabricate": { "url": "https://fabricate.tonic.ai/api/v1/mcp" }
  }
}
```

Self-hosted Fabricate uses `https://<their-fabricate-host>/api/v1/mcp`. If the user would rather not connect MCP at all, they can upload the `.fabricate-profile.json` file through the Profiles sidebar in their Fabricate project.

## Hard stops

Never do any of the following, even if asked:

- Copy real values from the source data into findings — no names, emails, phone numbers, addresses, dates of birth, government identifiers, payment details, IP addresses, free-text notes, or verbatim sample values. Describe **format and statistics** instead: "emails follow `firstname.lastname@<domain>`, 100% unique, mean length 24".
- Include high-cardinality identifying values as distribution buckets. Report a frequency shape ("top value covers 3% of rows") rather than the values themselves.
- Put credentials, connection strings, or tokens in generated files, the profile, logs, or chat.
- Write to the source database, or run DDL/DML of any kind.
- Leave temporary programs, dumps, or extracts on disk after finishing.

Values from small, non-sensitive, well-known reference tables (currencies, country codes, statuses) are the one exception, and only with explicit user approval — see the `reference_rows` section in [reference.md](reference.md).
