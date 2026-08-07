# Reference

Companion to [SKILL.md](SKILL.md): finding vocabulary, the export contract, and per-platform notes.

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

Only two namespaces are accepted; anything else is rejected at import:

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
  "Each customer has 0-47 orders, mean 3.2, median 2. 18% of customers have no
   orders."

anomaly      table:orders, column:orders.total
  "0.4% of orders have total = 0.00, clustered in 2023-Q1 - likely comp or test
   orders."
```

Statistics are the payload. "The email column contains email addresses" is worthless; "emails are 100% unique and non-null, mean length 24, 61% on the top 10 consumer domains" is a spec Fabricate can generate against.

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
- Each finding needs `content` (non-empty string), `finding_type` (from the table above), and `tags` (array of accepted tags).
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
