# Sub-skill: JSON column schema extraction

Companion to [SKILL.md](SKILL.md), invoked from Step 3 whenever the schema inventory turns up semi-structured columns — `JSON`/`JSONB` (PostgreSQL, MySQL), `VARIANT`/`OBJECT`/`ARRAY` (Snowflake), `NVARCHAR(MAX) AS JSON` (SQL Server), or text columns whose contents parse as JSON. A consistent JSON column is a structure Fabricate can generate row by row, but only if the profile says what the structure is. A column of blobs cannot be reproduced faithfully; a column of `{ "event": …, "ts": …, "props": { … } }` documents can.

Everything here follows the skill's trust boundary: keys and value **types** only, never values. The extracted schema is metadata about shape, the same as a `ddl` finding.

## When to apply this sub-skill

Apply it to a column when all of these hold:

1. The column's native type is JSON-like (see the platform notes in [reference.md](reference.md)), or a text column's values parse as JSON.
2. A sample of non-null values (the same bounded sample as Step 3, ~5,000 rows) is at least **95% valid JSON**.
3. The values are **structurally consistent**: at least **90% of sampled values conform to one schema** (see "Conformance" below). A single dominant shape with a small minority of variants is normal (old rows written before a key was added); three unrelated shapes is a polymorphic payload, not a schema.

If the column fails 2 or 3, do not force it. Record a `description` or `anomaly` finding instead — "`events.payload` is a JSON column but only 40% of sampled values parse as JSON; treat as opaque text" — so the generator knows not to invent a structure.

## Extraction procedure

1. **Detect JSON columns** from the schema inventory (Step 2). On text columns where JSON is plausible, test parseability in the database where a function exists (`jsonb_typeof`, `ISJSON`, `TRY_PARSE_JSON`, `JSON_VALID`) and count parse failures with the rule-check shape from [reference.md](reference.md).
2. **Compute the key-shape distribution** in the database. On PostgreSQL, `jsonb_object_keys` / `jsonb_each`; Snowflake `OBJECT_KEYS`; MySQL `JSON_KEYS`; SQL Server `ISJSON` + client-side sampling where there is no key function. For each distinct key set, report the share of sampled rows that have it. Use the top-level key set first, then recurse one shape at a time.
3. **Infer value types per key** — `string`, `number`, `boolean`, `null`, `array`, `object` — from aggregates, never by eyeballing values. For arrays, record the type of the elements when uniform (`array<number>`), otherwise `"array"` with `mixed` elements. For strings that follow an obvious format (ISO timestamp, UUID), record the format rather than any example, using a standard JSON Schema name (`date-time`, `date`, `uuid`, `email`, `uri`, `ipv4`): the Fabricate validator enforces those, so the conformance count covers them. A non-standard name such as `iso8601-timestamp` is ignored by the validator and is documentation only; to enforce a custom shape, add a `pattern`.
4. **Quantify conformance.** Run the rule-check shape ("N of M sampled values satisfy the schema"). Record the counts in the finding — never "all rows have field X" without the number behind it.
5. **Record nesting depth and array fan-out** the way you would a foreign-key fan-out: depth distribution if it varies, and the distribution of array lengths (top ~10 exact, tail collapsed) for repeated keys.

## The `json_schema` finding

The extracted schema is persisted as a `json_schema` finding — the only finding type other than `reference_rows` allowed to carry a `data` payload, and unlike `reference_rows` it needs no user approval because it contains no source values.

- `content` — a self-contained sentence: the column, the dominant shape, conformance numbers, sample size, nesting depth, null ratio. Example: "`events.payload` is a JSONB column; 98.6% of 5,000 sampled values conform to one schema with keys `event` (string), `ts` (ISO-8601 timestamp string), `user_id` (number), `props` (object, 0–8 keys, mean 3.2); 1.4% parse failures. Depth ≤ 3."
- `finding_type` — `json_schema`.
- `tags` — `table:<table>` and `column:<table>.<column>`, schema-qualified per the tag rules.
- `data` — an object with exactly these keys:

```json
{
  "schema": "public",
  "table": "events",
  "column": "payload",
  "json_schema": {
    "type": "object",
    "properties": {
      "event": { "type": "string" },
      "ts": { "type": "string", "format": "date-time" },
      "user_id": { "type": "number" },
      "props": {
        "type": "object",
        "additionalProperties": { "type": "string" },
        "minProperties": 0,
        "maxProperties": 8
      }
    },
    "required": ["event", "ts"],
    "additionalProperties": false
  },
  "stats": {
    "sample_size": 5000,
    "null_share": 0.0,
    "parse_failure_share": 0.014,
    "conforming_share": 0.986,
    "distinct_key_sets": 3,
    "max_depth": 3
  }
}
```

`json_schema` must be a real JSON Schema document: it needs at least one of `type`, `properties`, `items`, `$ref`, `oneOf`, `anyOf` or `allOf` at its top level. Unknown keywords are ignored by the validator, so a map like `{ "event": "string" }` would be stored and then accept every value. `schema` is optional (only on multi-schema engines, non-default schemas). `stats` is optional but expected; its values must be finite numbers. Use JSON-Schema vocabulary the reader can act on: `type`, `properties`, `required`, `items`, `additionalProperties`. Never use `enum`, `const`, `default` or `examples`: each embeds source values, and this finding type needs no approval only because it carries keys and types. For a low-cardinality key such as an event name or status, record how many distinct values it has in `stats` (for example `distinct_values_event`), and if the values themselves are worth keeping, capture them with the `reference_rows` approval flow instead.

## Validation contract

The importer rejects the profile — naming the offending finding — when:

- `data` appears on any type other than `reference_rows` or `json_schema`.
- A `json_schema` finding has no `data`, or its `data` is not an object with a non-empty `table` string, a non-empty `column` string, and a non-empty `json_schema` object that uses at least one structural keyword (see above).
- `schema` is present but not a string, or `stats` is present but not an object of finite numbers.
- The `json_schema` object uses `enum`, `const`, `default` or `examples` anywhere a sub-schema can appear.
- The `json_schema` object is not a valid JSON Schema document (for example an unknown `type` name, a `required` that is not an array, a `pattern` that is not a valid regex) or has a `$ref` that points outside the document.
- The `data` payload exceeds 64 KB — a distilled key/type schema is small; anything past that was never distilled.

Fix a rejected `json_schema` finding by distilling the schema further (collapse wide `properties` into `additionalProperties` with the key-count distribution in `stats`), never by deleting the finding silently.

## Verifying with the Fabricate engine

The extracted schema can be checked by the same engine that will consume it. Fetch the finding with `read_findings`, copy its `data.json_schema` member into the script as a literal, and apply it natively — no third-party validator:

```javascript
const schema = { type: "object", required: ["event", "ts"], properties: { /* ...copied from data.json_schema... */ } };
const result = validateJsonSchema(schema, candidateValue);
// → { valid: boolean, errors: [{ instancePath, message }], truncated }
```

The importer already rejects a document that is not a valid JSON Schema (see the validation contract above), so a broken schema fails the import rather than a later generation run; what a script adds after import is the check that a sampled value you extracted the schema from comes back valid, which tells you whether the schema describes the column. `validateJsonSchema` runs in Rust inside the sandbox and compiles the schema once per run, so a per-row loop is cheap. A schema document that does not compile returns `schemaErrors` instead of throwing — fix the schema, that is a bug in the extraction. Because profiling conformance counts and generation-time verification use the same validator, the `conforming_share` you record in `stats` is reproducible by the engine. In a local profiling context (no Fabricate runtime), the in-database conformance counts from the Extraction procedure above remain the source of truth; `validateJsonSchema` is the counterpart once the findings are imported.

## Hard stops specific to this sub-skill

- No source values in the schema: no example strings, no captured array contents, and no `enum`, `const`, `default` or `examples` keywords at all (the importer rejects them).
- No free-text key values (user notes, message bodies, search queries) surfaced as anything other than `"type": "string"` with, at most, a length distribution.
- If a key's values are themselves sensitive (emails, names), record only the type — do not profile the inner format unless it is non-identifying.
