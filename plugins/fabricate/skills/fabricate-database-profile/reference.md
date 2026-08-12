# Reference

Companion to [SKILL.md](SKILL.md): query patterns for Step 3, finding vocabulary, the export contract, and per-platform notes.

## Query patterns

Portable starting points for the statistics Step 3 asks for beyond the per-column pass. All of them return aggregates only. Only patterns explicitly limited to low-cardinality, non-sensitive categories may return category labels. Otherwise, where a pattern derives something from a value — a length, a leading character, a bucket index — group by the derived shape and never select the value itself.

**Fan-out distribution.** Aggregate twice: children per parent, then parents per child count. The inner `LEFT JOIN` is what makes childless parents count as zero rather than vanish.

```sql
SELECT child_count,
       COUNT(*)                                AS parents,
       COUNT(*) * 1.0 / SUM(COUNT(*)) OVER ()  AS share
FROM (
  SELECT p.id, COUNT(c.id) AS child_count
  FROM parent p
  LEFT JOIN child c ON c.parent_id = p.id
  GROUP BY p.id
) counts
GROUP BY child_count
ORDER BY child_count;
```

Report every row when the counts are few. Past ~30 distinct counts, keep the head exact and collapse the tail into ranges.

**Most-common frequency and its share, without the value**, for any column regardless of type. This reveals whether a column is dominated by repeated values without placing a name, email, identifier, or other source value into query output or agent context.

```sql
SELECT MAX(n)             AS most_common_n,
       MAX(n) * 1.0
         / NULLIF(SUM(n), 0) AS most_common_share
FROM (
  SELECT COUNT(*) AS n
  FROM t
  WHERE col IS NOT NULL
  GROUP BY col
) frequencies;
```

**Labeled value-frequency distribution**, only after the column is known to be low-cardinality and non-sensitive (for example, a status enum). The window sums over all groups before the limit, so each share remains a share of non-null rows. Never run this shape against names, emails, identifiers, free text, or an uncertain column.

```sql
SELECT category,
       COUNT(*)                                AS n,
       COUNT(*) * 1.0 / SUM(COUNT(*)) OVER ()  AS share
FROM t
WHERE category IS NOT NULL
GROUP BY category
ORDER BY n DESC
LIMIT 30;
```

**Conditional statistics** — a measure summarized within each low-cardinality, non-sensitive category. Bucket the grouping column first (`FLOOR((x - min) / width)`) when it is numeric or temporal. For a sensitive or uncertain grouping column, replace its label with a rank or derived shape before returning results.

```sql
SELECT category, COUNT(*) AS n, AVG(measure) AS mean, MIN(measure) AS lo, MAX(measure) AS hi
FROM t
GROUP BY category
ORDER BY n DESC;
```

**Crosstab** of two low-cardinality, non-sensitive categorical columns. Compare each cell's share against the product of the two marginal shares: close means independent, and that is worth recording. Do not return identifying values as either axis.

```sql
SELECT a, b, COUNT(*) AS n
FROM t
GROUP BY a, b
ORDER BY n DESC;
```

**Rule check with counts.** The shape behind every quantified assertion — one row giving the population and how much of it satisfies the rule.

```sql
SELECT COUNT(*)                                        AS total,
       SUM(CASE WHEN <rule> THEN 1 ELSE 0 END)         AS satisfying
FROM t;
```

**Cross-table temporal ordering** is that same shape across a join. Interval arithmetic is dialect-specific, so summarize gaps separately with the platform's date-difference function.

```sql
SELECT COUNT(*)                                                   AS total,
       SUM(CASE WHEN c.created_at < p.created_at THEN 1 ELSE 0 END) AS violations
FROM child c
JOIN parent p ON p.id = c.parent_id;
```

**Format rules per group.** Group by the derived shape alongside the column that governs it.

```sql
SELECT brand, LENGTH(number) AS len, SUBSTR(number, 1, 1) AS lead_digit, COUNT(*) AS n
FROM cards
GROUP BY brand, LENGTH(number), SUBSTR(number, 1, 1)
ORDER BY n DESC;
```

## Finding types

| Type | Records |
|---|---|
| `ddl` | Schema structure: columns, native types, nullability, defaults, primary/foreign/unique constraints, indexes. |
| `description` | Table purpose, row counts, business domain context. |
| `distribution` | Value distributions, histograms, time-series shape, seasonality, growth trends. |
| `correlation` | Statistical relationships between columns or tables, with the coefficient. |
| `constraints` | Value ranges, syntax patterns, naming conventions, deterministic formulas, ordering rules. |
| `cardinality` | Parent-child fan-out ratios and orphan/childless shares. Critical for realistic child-row volumes. |
| `anomaly` | Outliers and data-quality quirks worth reproducing or avoiding. |
| `reference_rows` | The one type that carries real values. See [Reference rows](#reference-rows). |

## Tags

Profiles produced by this skill use only two controlled namespaces:

- `table:<table_name>`
- `column:<table_name>.<column_name>`

A finding spanning several columns carries one tag per affected column plus one per affected table.

## Writing good findings

Each finding must stand alone — a reader with no other context should be able to act on it. Include the numbers.

```
ddl          table:orders, column:orders.status
  "orders.status is VARCHAR(20) NOT NULL with default 'pending' and a CHECK
   constraint limiting it to pending, paid, shipped, cancelled."

distribution table:orders, column:orders.status
  "orders.status distribution over a 5,000-row sample: paid 62%, shipped 24%,
   pending 9%, cancelled 5%."

distribution table:orders, column:orders.created_at
  "orders.created_at spans 2022-01-03 to 2026-07-30. Monthly volume grew from
   ~410/mo in 2022 to ~1,180/mo in 2026, with a 38% lift in November-December."

constraints  table:invoices, column:invoices.number
  "invoices.number matches INV-<YYYY>-<6-digit zero-padded sequence>, 100%
   unique, non-null, length always 15."

correlation  table:orders, column:orders.total, column:orders.item_count
  "orders.total correlates strongly with orders.item_count (Pearson r = 0.87)."

cardinality  table:customers, table:orders
  "Orders per customer, over all 2,000 customers: 0 orders 18.0%, 1 order
   31.6%, 2 orders 24.1%, 3 orders 13.2%, 4-6 orders 9.8%, 7+ orders 3.3%.
   Mean 2.1, median 2, max 47."

correlation  table:orders, column:orders.total, column:orders.channel
  "Mean order total is flat across channels - web $84.10, ios $86.40,
   android $82.90, phone $85.20 - so channel and total can be drawn
   independently."

constraints  table:customers, table:orders, column:orders.created_at,
             column:customers.created_at
  "Every one of the 4,214 orders was created at or after its customer's
   created_at; 0 violations. Median gap 31 days, p90 402 days."

constraints  table:cards, column:cards.brand, column:cards.number_last4
  "Card number length and leading digit are determined by brand: visa always
   16 digits starting 4 (1,204 of 1,204), amex always 15 starting 34 or 37
   (188 of 188), mastercard always 16 starting 51-55 (903 of 903)."

anomaly      table:orders, column:orders.total
  "0.4% of orders have total = 0.00, clustered in 2023-Q1 - likely comp or test
   orders."

anomaly      table:payments, column:payments.updated_at
  "1,551 of 2,000 payments share the single updated_at value
   2026-03-14T02:11:07Z - a bulk backfill, not organic edit activity.
   Realistic generated data should spread updated_at across the window."
```

Statistics are the payload. "The email column contains email addresses" is worthless; "emails are 100% unique and non-null, mean length 24, 61% on the top 10 consumer domains" is a spec Fabricate can generate against.

## Artifacts versus realism

Source databases carry properties that are true of the data yet wrong to reproduce, and this is especially common when the source was itself seeded or generated. Watch for:

- Every parent having at least one child, or no dormant, cancelled or empty records anywhere.
- Perfect uniqueness in a column where real populations repeat heavily — postcodes, city names, employers, device models.
- A distribution that stops dead at a round number, which usually means a cap or a default rather than a real ceiling.
- Many rows sharing one timestamp, which is a migration or backfill rather than user activity.
- Categorical columns drawn independently where reality couples them — a status unrelated to the dates around it, a country unrelated to currency.

Record each as an `anomaly` that names the artifact and says what real data would look like instead. The distinction matters because Fabricate treats a `constraints` finding as a rule to satisfy: written up that way, "100% of customers have placed an order" stops being an observation and becomes a requirement that guarantees the generated data inherits the flaw.

## Export format

Write UTF-8 JSON named `<profile-name>.fabricate-profile.json`:

```json
{
  "fabricate_profile_export": 1,
  "exported_at": "2026-08-07T12:00:00Z",
  "profile": "prod-ecommerce",
  "findings": [
    {
      "content": "orders.status distribution over a 5,000-row sample: paid 62%, shipped 24%, pending 9%, cancelled 5%.",
      "finding_type": "distribution",
      "tags": ["table:orders", "column:orders.status"]
    }
  ]
}
```

Contract:

- `fabricate_profile_export` must be exactly the integer `1`. Imports without it are rejected.
- `exported_at` is ISO 8601 UTC.
- `profile` is the profile name. It must not start with `db:` — that prefix is reserved for profiles that belong to a Fabricate database.
- Each finding needs `content` (non-empty string), `finding_type` (from the table above), and `tags` (an array using the controlled namespaces above).
- `data` is only permitted on `reference_rows` findings and must be omitted everywhere else.
- All numbers must be finite. Write out `NaN`/`Infinity` as prose or omit the statistic.

Write the file atomically (temp file, then rename) so a partial write is never mistaken for a profile.

## Reference rows

`reference_rows` is the only finding type allowed to contain real values, and it exists for small lookup tables that must be identical across environments — currencies, country codes, statuses, category taxonomies.

Both conditions must hold, and the user must explicitly approve:

1. **Not sensitive.** No PII, PHI, financial account data, credentials, or anything identifying a real person or company. When in doubt, skip it.
2. **Reference data.** At most 500 rows, rarely changes, a foreign-key parent with no outgoing FKs of its own, and a well-known enumerable domain.

Caps enforced on import: 500 rows and 256 KB serialized per finding.

```json
{
  "content": "currencies is a 153-row ISO 4217 reference table captured verbatim.",
  "finding_type": "reference_rows",
  "tags": ["table:currencies"],
  "data": {
    "table": "currencies",
    "columns": ["code", "name", "minor_unit"],
    "rows": [{ "code": "USD", "name": "US Dollar", "minor_unit": 2 }]
  }
}
```

`data` must be an object with a non-empty `table` string, a non-empty `columns` array of strings, and a non-empty `rows` array of objects. Anything else is rejected.

## Platform notes

Cover the same ground on every platform; only the syntax changes. Record vendor-specific behavior as `ddl` or `constraints` findings — Fabricate needs it to generate data the target engine will accept.

### PostgreSQL (and Redshift)

- Session guards: `SET TRANSACTION READ ONLY;` and `SET statement_timeout = '60s';` (Redshift uses `statement_timeout` too, but has no `TABLESAMPLE`).
- Metadata: `information_schema.columns` / `.table_constraints` / `.key_column_usage`, plus `pg_catalog` for index definitions and `pg_class.reltuples` for approximate row counts.
- Sampling: `TABLESAMPLE SYSTEM (n)` on PostgreSQL; `ORDER BY RANDOM() LIMIT n` on Redshift.
- Capture: identity vs. `serial` sequences and their current values, `ENUM` types and their labels, arrays, `jsonb` key shapes (keys and value types only — never values), partitioning, and `CHECK` constraint expressions. On Redshift also capture `DISTKEY`, `SORTKEY`, and compression encodings.
- Quote identifiers with double quotes; identifiers fold to lowercase unless quoted.

### MySQL / MariaDB

- Session guards: `SET SESSION TRANSACTION READ ONLY;` and `SET SESSION max_execution_time = 60000;`.
- Metadata: `information_schema.COLUMNS`, `.STATISTICS`, `.KEY_COLUMN_USAGE`, `.TABLES` (`TABLE_ROWS` is approximate for InnoDB — use `COUNT(*)` when exactness matters).
- Sampling: `ORDER BY RAND() LIMIT n` — expensive on large tables; prefer a bounded key-range scan.
- Capture: storage engine, `AUTO_INCREMENT` position, `ENUM`/`SET` members, unsigned integer types, `ON UPDATE CURRENT_TIMESTAMP` defaults, and the column collation (it decides case sensitivity of uniqueness).
- Quote identifiers with backticks.

### Microsoft SQL Server

- Session guards: `SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;` plus a client-side command timeout (`LOCK_TIMEOUT` bounds lock waits only).
- Metadata: `sys.columns`, `sys.types`, `sys.indexes`, `sys.foreign_keys`, `sys.check_constraints`; approximate row counts from `sys.dm_db_partition_stats`.
- Sampling: `TABLESAMPLE SYSTEM (n PERCENT)` or `TOP n ... ORDER BY NEWID()`.
- Capture: `IDENTITY` seed/increment, computed columns and their definitions, `UNIQUEIDENTIFIER` defaults, `datetime2` precision, column collation, and filtered indexes.
- Quote identifiers with square brackets.

### Oracle

- Session guards: `SET TRANSACTION READ ONLY;` and a resource-manager or client-side timeout.
- Metadata: `ALL_TAB_COLUMNS`, `ALL_CONSTRAINTS`, `ALL_CONS_COLUMNS`, `ALL_INDEXES`; `ALL_TABLES.NUM_ROWS` is stats-dependent and may be stale.
- Sampling: `SELECT ... FROM t SAMPLE (n)`.
- Capture: `NUMBER(p,s)` precision and scale, `VARCHAR2` byte-vs-char semantics, sequences and identity columns, and that Oracle treats the empty string as `NULL` — that materially changes generation.
- Quote identifiers with double quotes; unquoted identifiers fold to uppercase.

### Snowflake

- Session guards: `ALTER SESSION SET STATEMENT_TIMEOUT_IN_SECONDS = 60;` and use a read-only role.
- Metadata: `INFORMATION_SCHEMA.COLUMNS` / `.TABLE_CONSTRAINTS`, or `SHOW` commands. Note that Snowflake does not enforce most constraints — record declared foreign keys as intent, then verify them with a join.
- Sampling: `SELECT ... FROM t SAMPLE (n ROWS)`.
- Capture: `VARIANT`/`OBJECT`/`ARRAY` key shapes (keys and types only), clustering keys, and time-travel-irrelevant columns. Unquoted identifiers fold to uppercase.

### BigQuery

- Session guards: run with a read-only service account, and set a maximum-bytes-billed limit on every job.
- Metadata: `INFORMATION_SCHEMA.COLUMNS`, `.COLUMN_FIELD_PATHS`, `.TABLE_OPTIONS`; `__TABLES__` for row counts.
- Sampling: `TABLESAMPLE SYSTEM (n PERCENT)`, or `WHERE RAND() < p`.
- Capture: partitioning and clustering columns (they dominate query patterns), `STRUCT`/`ARRAY` field paths, `NUMERIC`/`BIGNUMERIC` precision, and the absence of enforced primary and foreign keys.
- Quote identifiers with backticks; there are no cross-table constraints to read.

### SQLite

- Session guards: open the file read-only (`file:db.sqlite?mode=ro`) and set a busy timeout.
- Metadata: `PRAGMA table_info`, `PRAGMA foreign_key_list`, `PRAGMA index_list`, and `sqlite_master.sql` for the original DDL.
- Sampling: `ORDER BY RANDOM() LIMIT n`.
- Capture: type affinity rather than declared type, `WITHOUT ROWID` tables, and `AUTOINCREMENT` usage.

### Anything else

Fall back to ANSI `information_schema` views (`columns`, `tables`, `table_constraints`, `key_column_usage`, `referential_constraints`) and portable aggregate SQL: `COUNT(*)`, `COUNT(col)`, `COUNT(DISTINCT col)`, `MIN`, `MAX`, `AVG`, `STDDEV`, `GROUP BY` for frequencies, and `CORR` where available (otherwise compute Pearson from `SUM(x)`, `SUM(y)`, `SUM(x*y)`, `SUM(x*x)`, `SUM(y*y)`, `COUNT(*)`). Record the platform and version in a `description` finding so the profile's provenance is clear.

## Importing into Fabricate

With the Fabricate MCP server connected:

```
import_profile(project_id: "<uuid>", profile_export: <parsed JSON object>)
```

`profile_export` takes the parsed object, not a string. Pass `profile: "<name>"` to override the name inside the export. For large profiles, or when your client can send an authenticated `PUT`, use `create_upload` to get an upload URL, `PUT` the file, then call `import_profile(project_id:, upload_id:)`.

The call returns the profile name, how many findings were imported, and the count per finding type. The import is atomic: either every finding lands or none does.

Without MCP, the user uploads the `.fabricate-profile.json` file to a Fabricate conversation, or imports it from the Profiles sidebar in the project.
