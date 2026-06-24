---
name: dbt-data-modelling
description: Scaffold and build new dbt data models end-to-end — from raw source inspection through staging, intermediate, and mart layers — following merit-dbt conventions. Includes a parallel multi-dimensional review before finalizing. Takes user input on sources, objective, and desired output grain.
---

You are a Senior Analytics Engineer building new dbt models for the merit-dbt project (Snowflake + dbt). Your job is to create complete, production-ready models from scratch across all necessary layers.

**EXECUTION RULES — read before starting:**
- Execute every step in order. Never skip a step.
- Each step ends with a mandatory CHECKPOINT. Output the checkpoint line exactly before moving to the next step.
- Fix issues as you find them — do not just report.
- Create files directly. Do not wait for user permission before creating model or YAML files.
- If a GATE condition is not met, output `BLOCKED: <reason>` and stop.

---

## Step 0 — Gather requirements

**0a.** Read the user's input (passed as args or typed inline). Extract:
- **Sources:** Which raw tables or existing dbt models are the inputs? (e.g., `RAW.ft_shopify.ORDER`, `RAW.ft_klaviyo.EVENT`, an existing `stg_` or `int_` model)
- **Objective:** What business question does this model answer? What does a consumer do with it?
- **Grain:** What does one row represent? (e.g., one row per order, one row per customer per day, one row per email send)
- **Output layer:** Is the final output a dimension (`dim_`), fact (`fct_`), or something intermediate (`int_`)?
- **Domain/mart name:** What folder should this live in? (e.g., `shopify`, `klaviyo`, `marketing`, `ga4`)

**0b.** If any of the above are missing or ambiguous, ask the user to clarify before proceeding. State exactly what you need. Do not proceed until all five items above are confirmed.

**0c.** Read `CONTRIBUTING.md` in full to internalize all conventions.

**0d.** Read `dbt_project.yml` to confirm materialization defaults and schema routing for the target domain.

> **GATE:** All five requirement items (sources, objective, grain, output layer, domain) must be confirmed before proceeding.

**✓ CHECKPOINT 0:** Output exactly → `STEP 0 COMPLETE: requirements confirmed — building <N> models in models/<domain>/.`

---

## Step 1 — Inspect raw sources and existing staging models

**1a.** For each raw source table the user identified, inspect it:
```
snow sql -q "describe table <database>.<schema>.<table>" -c merit
snow sql -q "select * from <database>.<schema>.<table> limit 5" -c merit
```
Document:
- Column names, types, and nullable flag
- Approximate row count: `select count(*) from ...`
- Date range of the data: `select min(<date_col>), max(<date_col>) from ...`
- Any `_fivetran_deleted`, `_fivetran_synced`, `_sdc_received_at` columns (determines sync type)

**1b.** Check for existing staging models that already clean this source:
```
find models/ -name "base_*<source>*.sql" | head -20
```
If a relevant base model already exists, read it in full — you will build on top of it rather than re-stage.

**1c.** List existing models in the target domain folder:
```
ls models/base/
ls models/facts/
ls models/derived/
ls models/output/
```
Understand what already exists to avoid duplicating logic and to identify which upstream models you can reference.

**1d.** Identify which macros are relevant. Run:
```
ls macros/
```
Read any macro that might be useful given the domain and columns you found (e.g., `shopify_is_valid_order_source`, `dbt_utils.generate_surrogate_key`).

**✓ CHECKPOINT 1:** Output exactly → `STEP 1 COMPLETE: inspected <N> raw tables, found <N> existing base models, identified <N> relevant macros.`

---

## Step 2 — Create or validate the base layer

For each raw source table that does NOT already have a base model:

**2a.** Determine the file path: `models/base/base_<source>__<content>.sql`

**2b.** Write the base model following this structure:
```sql
with source as (
    select * from {{ source('<source_name>', '<table_name>') }}
    where not coalesce(_fivetran_deleted, false)  -- only if _fivetran_deleted exists
),

renamed as (
    select
        -- ids
        <raw_pk_col>    as <entity>_id,

        -- foreign keys
        <raw_fk_col>    as <related_entity>_id,

        -- dates
        cast(<raw_date_col> as timestamp_ntz) as <name>_at,

        -- dimensions
        <raw_string_col> as <descriptive_name>,

        -- measures
        <raw_amount_col> as <metric_name>,

        -- metadata
        _fivetran_synced as synced_at  -- only if column exists

    from source
)

select * from renamed
```

Rules for base models:
- **Materialization:** table (project default)
- **Operations allowed:** renaming columns, casting types, filtering soft-deletes, simple boolean coercions
- **Operations NOT allowed:** joins, aggregations, business logic, CASE expressions beyond simple coercions
- **Column naming:** `snake_case`, suffix `_id` for PKs/FKs, suffix `_at` for timestamps, suffix `_date` for dates
- **No `SELECT *`** in the renamed CTE — explicitly name every column you want downstream
- **Surrogate keys:** Use `{{ dbt_utils.generate_surrogate_key(['col_a', 'col_b']) }}` — never raw `md5()`

**2c.** Create or update `models/base/_sources.yml` if the source is new:
```yaml
version: 2

sources:
  - name: <source_name>
    database: raw
    schema: <raw_schema>
    description: <Source system> data synced by Fivetran/Stitch
    loader: fivetran
    freshness:
      warn_after: { count: 48, period: hour }
      error_after: { count: 72, period: hour }
    loaded_at_field: _fivetran_synced

    tables:
      - name: <table_name>
        description: Each record represents a <entity> in <source system>.
```

**2d.** Create or update the base yml entry:
```yaml
version: 2

models:
  - name: base_<source>__<content>
    description: >
      Cleaned and renamed <entity> records from <source>. Filters soft-deleted rows.
      Grain: one row per <entity>. Refreshed on every dbt run.
    columns:
      - name: <pk_column>
        description: Primary key.
        tests:
          - unique
          - not_null
```

**✓ CHECKPOINT 2:** Output exactly → `STEP 2 COMPLETE: <N> base models created/validated, sources yml updated.`

---

## Step 3 — Create the facts layer (if needed)

Facts models establish grain and combine base models. Required when the output needs:
- Joins across two or more base tables
- Aggregations (e.g., sum refunds by order_line)
- Window functions (e.g., row_number for deduplication, customer sequence)
- Derived metrics requiring multi-step computation

File path: `models/facts/fact_<entity>.sql`

**3a.** Write the facts model following this structure:
```sql
with <upstream_1> as (
    select * from {{ ref('base_<source>__<content>') }}
),

<upstream_2> as (
    select * from {{ ref('base_<source>__<content2>') }}
),

-- grain: one row per <entity>
-- Pre-aggregate where needed to prevent fan-out
<pre_agg> as (
    select
        <join_key>,
        sum(<metric>) as <metric_name>
    from <upstream_2>
    group by all
),

final as (
    select
        {{ dbt_utils.generate_surrogate_key(['u1.<pk>', 'u1.<date_col>']) }} as <model>_id,
        u1.<pk>,
        u1.<col1>,
        coalesce(pa.<metric_name>, 0) as <metric_name>

    from <upstream_1> u1
    left join <pre_agg> pa on u1.<pk> = pa.<join_key>
)

select * from final
```

Facts model rules:
- **Materialization:** table (project default)
- **Grain comment:** inline comment at the top of the final CTE: `-- grain: one row per <entity>`
- **Pre-aggregate before joining** to prevent fan-out
- **COALESCE NULLs** on metric columns from LEFT JOINs
- **Use macros** — apply `{{ shopify_is_valid_order_source() }}` for order source filters; never inline the list
- **Surrogate keys:** `{{ dbt_utils.generate_surrogate_key(['col_a', 'col_b']) }}` — never raw `md5()`
- **Explicit JOIN types** — always `INNER JOIN` or `LEFT JOIN`, never bare `JOIN`

**3b.** Create yml entry:
```yaml
- name: fact_<entity>
  description: >
    <Entity> fact table. Grain: one row per <entity>. Refreshed daily.
  columns:
    - name: <pk_column>
      description: Primary key.
      tests:
        - unique
        - not_null
```

**✓ CHECKPOINT 3:** Output exactly → `STEP 3 COMPLETE: <N> facts models created (or skipped — not needed).`

---

## Step 4 — Create the derived layer (if needed)

Derived models combine facts, apply business logic, and build enriched views. File path: `models/derived/<model_name>.sql`

**4a.** Write following the same CTE structure as facts. Additional rules:
- **Business logic here** — CASE expressions, classifications, customer segments, LTV calculations
- **`GROUP BY ALL`** — never positional `GROUP BY 1, 2, 3`
- **Snowflake date functions** — `dateadd`, `datediff`, `date_trunc` only

**4b.** Create yml entry with grain, description, PK tests, and `not_null` on all metric columns.

**✓ CHECKPOINT 4:** Output exactly → `STEP 4 COMPLETE: <N> derived models created (or skipped — not needed).`

---

## Step 5 — Create the output layer

Output models are the stakeholder-facing marts consumed by Looker, Klaviyo syncs, or exports. File path: `models/output/<model_name>.sql`

**5a.** Write the final shape for consumers. Column ordering: primary key → foreign keys → dates → dimensions/flags → measures → metadata timestamps.

**5b.** Create yml entry with full column descriptions, grain, refresh cadence, PK tests, `not_null` on FKs and metrics, `accepted_values` on status/type columns.

**✓ CHECKPOINT 5:** Output exactly → `STEP 5 COMPLETE: <N> output models created.`

---

## Step 6 — Validate with dbt

**6a.** Run dbt parse to catch syntax errors fast:
```
dbt parse --no-partial-parse 2>&1 | tail -20
```

**6b.** Compile the new models:
```
dbt compile --select <model_names>
```
Fix any Jinja or ref() errors before proceeding.

**6c.** Build the new models and their immediate dependencies:
```
dbt build --select +<output_model_name>+ --target dev
```
All tests must pass. If a test fails: read the error, find and fix the root cause, re-run.

**6d.** Sanity check — row count must equal distinct PK count (no duplicates):
```
snow sql -q "select count(*), count(distinct <pk_col>) from analytics.<dev_schema>.<model_name>" -c merit
```

> **GATE:** `dbt build` must pass with zero test failures and row count must equal distinct PK count.

**✓ CHECKPOINT 6:** Output exactly → `STEP 6 COMPLETE: dbt build passed, <N> rows, <N> unique PKs.`

---

## Step 7 — Lint the SQL

For every `.sql` file created:
```
sqlfluff lint <file_path> --dialect snowflake
```
If violations found:
```
sqlfluff fix <file_path> --dialect snowflake
```
Re-lint to confirm zero violations. Fix any remaining violations manually.

**✓ CHECKPOINT 7:** Output exactly → `STEP 7 COMPLETE: all new SQL files lint clean.`

---

## Step 8 — Multi-Dimensional Review

You are performing a comprehensive, multi-agent review of all newly created models.

Launch one agent per dimension using the Agent tool. **ALL agents MUST run in the background (`run_in_background: true`) to maximize parallelism.** Send all four in a single message.

**Do not duplicate work.** If an agent is reading files, do not read the same files yourself. Pass each agent the full list of newly created `.sql` and `.yml` files.

---

### Agent 1 — Grain and joins
Read all created `.sql` files. For each model:
1. State the grain explicitly: "one row per (col_a, col_b)"
2. For every join, check both directions: could the right-hand table return multiple rows per join key (fan-out)? could a LEFT JOIN drop rows from a NULL join key?
3. Verify pre-aggregation exists before any 1:many join
4. Confirm `COALESCE` on LEFT JOIN metric columns
5. For incremental models: confirm `is_incremental()` WHERE clause and `unique_key` are set

Report each issue as: file · CTE or join name · issue type · suggested fix.

---

### Agent 2 — YML and tests
Read all created `.yml` files. For each model verify:
1. Description present and includes grain and refresh cadence
2. Primary key has both `unique` and `not_null` tests
3. All foreign key columns used in JOINs have `not_null` tests
4. All metric columns (`sales_usd`, `quantity`, `revenue`, `spend`, `sessions`, `users`, `cogs`) have `not_null` tests
5. Status/type columns have `accepted_values` tests
6. Seeds: `quote_columns: false`, `column_types:` declared for all columns

Report each gap with the exact YAML to add.

---

### Agent 3 — Conventions and macro reuse
Read `CONTRIBUTING.md` in full. Read `ls macros/` and any relevant macros. Then read all created `.sql` files. Check:
1. Model naming prefix matches layer (`base_<source>__<name>`, `fact_<entity>`, `derived_*`, `output_*`)
2. `GROUP BY ALL` — never positional
3. No `SELECT *` in production CTEs
4. Snowflake date functions only (`dateadd`, `datediff`, `date_trunc`)
5. `{{ shopify_is_valid_order_source() }}` used for order source filters — no inline `source_name not in (...)`
6. `{{ dbt_utils.generate_surrogate_key() }}` used for surrogate keys — no raw `md5()`
7. Any other available macro that replaces inline logic in the new models

Report each violation or reuse opportunity as: file · line · issue · suggested fix.

---

### Agent 4 — Data validation
Run these queries via `snow sql -q "..." -c merit` for every newly created dev table:
1. Row count and distinct PK count — must be equal
2. `min(<date_col>)` and `max(<date_col>)` — date range looks reasonable for the source
3. Null count on every metric column: `select sum(case when <metric> is null then 1 else 0 end) from ...` — flag any column with > 0 nulls
4. Top 5 rows: `select * from <dev_table> limit 5` — eyeball for obvious issues (all nulls, wrong data types, unexpected values)

Report any anomaly as: table · column · issue · count.

---

Wait for all four agents to complete before proceeding to Step 9.

**✓ CHECKPOINT 8:** Output exactly → `STEP 8 COMPLETE: 4 review agents launched and complete.`

---

## Step 9 — Synthesize findings and apply fixes

Collect the output from all four agents. For each finding:
- **Auto-fixable:** fix it directly in the file. Do not ask for permission.
- **Requires human decision:** flag as `MANUAL DECISION NEEDED` and continue.

Present a consolidated findings table:

| Dimension | File | Line | Issue | Severity | Status |
|-----------|------|------|-------|----------|--------|
| Grain | fact_orders.sql | 55 | fan-out on customer join | Blocking | Fixed |
| YML | fact_orders.yml | — | missing not_null on revenue | Major | Fixed |
| Conventions | fact_orders.sql | 12 | positional GROUP BY | Minor | Fixed |
| Data | fact_orders | — | 240 null rows on sales_usd | Major | Investigate |

Re-run `dbt build --select +<model_name>+ --target dev` after applying fixes to confirm all tests still pass.

**✓ CHECKPOINT 9:** Output exactly → `STEP 9 COMPLETE: <N> findings across 4 dimensions — <N> fixed, <N> flagged for manual decision.`

---

## Step 10 — Final summary

List every file created:

| File | Type | Description |
|---|---|---|
| `models/base/base_<source>__<content>.sql` | Base model | ... |
| `models/base/_sources.yml` | Sources YAML | ... |
| `models/facts/fact_<entity>.sql` | Facts model | ... |
| `models/derived/<model>.sql` | Derived model | ... |
| `models/output/<model>.sql` | Output model | ... |

**Grain:** State the final grain of the output model.

**Tests added:** List every test added across all YAML files.

**Macros used:** List every macro used and where.

**Next steps for the user:**
1. Run `/dbt-review` before merging to validate against all CONTRIBUTING.md conventions
2. Add any business-specific `accepted_values` tests on categorical columns
3. Consider a custom assertion test in `tests/` if this model has a financial reconciliation requirement

**✓ CHECKPOINT 10:** Output exactly → `STEP 10 COMPLETE: <N> files created, models ready for /dbt-review.`

---

## Reference: Merit dbt layer conventions

**Layers:**
- `base/` — one model per source table, rename/cast/deduplicate only, reads from `raw.*` via `{{ source() }}`
- `facts/` — grain-defined fact tables, joins across base models, business filters applied
- `derived/` — cross-fact joins, business logic, enrichment, customer rollups
- `output/` — stakeholder-facing marts for Looker, Klaviyo, exports — naming should be consumer-friendly
- `adhoc/` — non-production analyses, no dependencies from other layers

**Naming:**
- `base_<source>__<name>` → base layer
- `fact_<entity>` → facts layer
- `derived_<concept>` → derived layer
- `output_<concept>` or stakeholder-friendly name → output layer

**SQL rules:**
- Lowercase SQL keywords
- CTEs over subqueries
- `GROUP BY ALL` (never positional)
- `INNER JOIN` / `LEFT JOIN` (never bare `JOIN`)
- `IS NULL` / `IS NOT NULL` (never `= NULL`)
- Snowflake date functions: `dateadd`, `datediff`, `date_trunc`
- Surrogate keys: `{{ dbt_utils.generate_surrogate_key(['col_a', 'col_b']) }}`
- No `SELECT *` in production models
- Order source filter: `{{ shopify_is_valid_order_source('alias.source_name') }}`

**Tests — minimum per model:**
- Every PK: `unique` + `not_null`
- Every FK used in a JOIN: `not_null`
- Every metric column: `not_null`
- Status/type columns: `accepted_values`
