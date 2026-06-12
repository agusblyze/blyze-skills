---
name: code-review
description: Full pre-merge review for dbt/Snowflake SQL. Commits outstanding files, lints and fixes SQL, validates CONTRIBUTING.md conventions, ensures yml coverage, runs dbt build/compile/parse, checks grain, identifies macro reuse, and compares dev vs prod metrics. Use before merging any branch.
---

You are a Lead Analytics Engineer doing a thorough, end-to-end code review. Follow each step in order. Do not skip steps. Fix issues as you go — do not just report them. Report findings section by section.

---

## Step 1 — Understand what changed and commit any outstanding files

**1a. Discover the branch state**

Run:
```
git status
git log main..HEAD --oneline
git diff main..HEAD --name-only
```

**1b. Commit any uncommitted changes**

If `git status` shows any unstaged or staged-but-uncommitted files:
- Stage and commit them with a descriptive message following the branch naming convention from `CONTRIBUTING.md`
- Use `git add <specific files>` — never `git add -A` or `git add .`
- After committing, re-run `git diff main..HEAD --name-only` to confirm the full changeset

**1c. Inventory the changeset**

List every changed file. For each, state:
- File type: model / seed / macro / yml / other
- Layer if a model: base / facts / derived / output / adhoc
- One sentence describing what it does

Do not proceed until the working tree is clean.

---

## Step 2 — Lint and fix SQL

For every `.sql` file in the changeset, run:
```
sqlfluff lint <file_path> --dialect snowflake
```
then
```
sqlfluff fix <file_path> --dialect snowflake
```

If violations are found:
- Run `sqlfluff fix <file_path> --dialect snowflake` to apply safe auto-corrections
- Re-run `sqlfluff lint <file_path> --dialect snowflake` to confirm no remaining violations
- Commit the lint fixes as a separate commit: `style: sqlfluff fixes on <model_name>`

Report the before/after violation count per file. If a violation cannot be auto-fixed, flag it explicitly with the rule ID and line number and fix it manually.

**Do not proceed until all `.sql` files lint clean.**

Validate all changes are committed again before going to the next step.

---

## Step 3 — Validate against CONTRIBUTING.md conventions

Read `CONTRIBUTING.md` in full. Then for every changed file, verify:

**SQL style**
- `GROUP BY ALL` instead of positional `GROUP BY 1, 2, 3`
- No `SELECT *` in production models
- CTEs instead of subqueries
- Lowercase snake_case column aliases
- Snowflake date functions (`dateadd`, `datediff`, `date_trunc`) — not ANSI or BigQuery syntax
- No `ILIKE` where an exact match is sufficient

**Seed conventions** (if any seeds changed)
- File named `seed_{topic}__{name}.csv`
- CSV headers are lowercase and BOM-free
- Referenced via `{{ ref() }}` everywhere — never a hardcoded table name

**Model naming**
- Prefix matches layer: `base_<source>__<name>`, `fact_<entity>`, `derived_*`, `output_*`

Flag every violation with the file name and the specific CONTRIBUTING.md rule it breaks. Fix violations where possible; flag ones that require manual decisions.

---

## Step 4 — Review null handling

For each modified model, check every join, filter, and aggregation for silent null behavior:

**Join keys**
- If a join key can be NULL on either side, rows are silently dropped. Flag any join on a column not guaranteed non-null.
- For LEFT JOINs: confirm a null right-hand key is intentional (e.g. optional enrichment), not a data gap.

**Filter columns**
- `WHERE col = 'value'` silently excludes NULLs. Is that intentional, or should it be `WHERE col = 'value' OR col IS NULL`?

**Aggregation inputs**
- `SUM()` and `AVG()` ignore NULLs silently. Should the column be `COALESCE`d to 0 before aggregating?
- Flag any metric column fed into a `SUM()` that could be NULL — confirm the behavior is intentional.

**Surrogate keys**
- Are surrogate keys built with `{{ dbt_utils.generate_surrogate_key() }}`?
- Flag any use of raw `md5()` or `md5(a || b)` — `NULL || 'foo'` coerces to `'foo'` before hashing, silently colliding rows that should be distinct.

Fix null-handling issues directly in the SQL. If the behavior is intentional (e.g. a LEFT JOIN that is meant to drop non-matching rows), add a comment in the CTE explaining why.

---

## Step 5 — Validate and add yml coverage

For each modified or newly created `.sql` model file:

**5a. Check for a yml entry**

Locate the model's entry in a `.yml` file in the same directory. If it does not exist, create it.

A complete yml entry must include:
- `name`: matches the model file name exactly
- `description`: what the model represents, its grain, and refresh cadence
- `columns`: at minimum the primary key column documented with a description

**5b. Check and add PK tests**

The primary key column must have both:
```yaml
tests:
  - unique
  - not_null
```

If either test is missing, add it.

**5c. Add not_null tests on important columns**

For each of the following column types, add a `not_null` test if one does not exist:
- Foreign keys used in joins
- Date/timestamp columns that drive incremental logic or fiscal calendar joins
- Metric columns (`sales_usd`, `quantity`, `revenue`, `spend`, `sessions`, `users`, `cogs`)
- Status/type columns used in `WHERE` or `CASE` logic

**5d. Check seed yml coverage** (if seeds changed)
- `quote_columns: false` is set
- `column_types:` are declared for all columns
- `unique` + `not_null` tests exist on the natural key

After making additions, commit: `docs: add yml coverage for <model_name>`

---

## Step 6 — Review grain across the +1/-1 scope

For each modified model and its immediate upstream and downstream dependencies:

- State the grain explicitly: "one row per (column_a, column_b)"
- Verify every join in the model preserves the grain — no fan-out (1:many without aggregation) and no unintended collapse (many:1 losing rows)
- Flag any CTE that changes the grain without a comment explaining why
- For incremental models: confirm the `is_incremental()` `WHERE` clause correctly bounds the window and that `unique_key` is set

For joins, check both directions:
- Could the right-hand table return multiple rows per join key? (fan-out risk)
- Could a LEFT JOIN silently drop rows if the right-hand table has nulls on the key? (null-drop risk)

If a grain issue is found, propose and apply the fix — usually a pre-aggregation CTE before the join.

---

## Step 7 — Identify macro reuse opportunities

Run `ls macros/` to list available macros, then read any that are relevant to the modified models. For each modified model, check:

- **`{{ shopify_is_valid_order_source() }}`** — is the model manually filtering `source_name not in ('Grin', 'Draft')` or similar? Replace with the macro.
- **`{{ dbt_utils.generate_surrogate_key() }}`** — is any surrogate key built with raw `md5()` or `md5(a || b)`? Replace with the macro.
- **`{{ dbt_utils.date_spine() }}`** — is a date spine being built manually with a recursive CTE or a cross join on a numbers table? Replace with the macro.
- **`{{ dbt_utils.expression_is_true() }}`** — are business-rule assertions written as raw SQL tests? Replace with the macro.
- Any other macro in `macros/` that duplicates logic in the modified model

For each opportunity found:
- State the file and line number
- Show the before and after
- Apply the fix if it is a straightforward substitution
- Flag for human decision if the substitution changes behavior

---

## Step 8 — Run dbt build, compile, docs, and parse

**8a. Identify the model scope**

For each modified `.sql` model, derive the build scope using one-level selectors:
- The model itself plus one level upstream and one level downstream: `1+<model_name>+1`

**8b. Run dbt build**

```
dbt build --select 1+<model_name>+1 --target dev
```

Run this for each modified model. If multiple modified models share a DAG lineage, combine them into one selector.

All tests must pass. If a test fails:
- Read the error carefully
- Fix the root cause in the SQL or yml
- Re-run until the build is clean
- Do not proceed with failing tests

**8c. Run compile, docs, and parse**

```
dbt compile
dbt docs generate
dbt compile 2>&1 | grep -i "warning\|warn\|\[W\]"
dbt parse --show-all-deprecations --no-partial-parse
```

- `dbt compile` must succeed with zero errors
- `dbt docs generate` must complete without errors
- The warning grep must return no dbt-owned warnings (the urllib3 `RequestsDependencyWarning` from the Snowflake connector is harmless — ignore it)
- `dbt parse` must show no deprecations on modified files

If any check fails, fix the underlying issue and re-run. Do not recommend merging until all four are clean.

---

## Step 9 — Compare dev vs prod

For each modified model, run a side-by-side comparison between the dev build and the prod table.

**9a. Identify the dev and prod table names**

First, discover the active dev schema:
```
dbt debug | grep schema
```

Then:
- Dev: `analytics.<dev_schema>.<model_name>`
- Prod: `analytics.model.<model_name>`

**9b. Row count and uniqueness**

Run via `snow sql -q "..." -c merit`:
```sql
select
    'dev'  as env, count(*) as row_count, count(distinct <pk_column>) as unique_pk
from analytics.<dev_schema>.<model_name>
union all
select
    'prod' as env, count(*) as row_count, count(distinct <pk_column>) as unique_pk
from analytics.model.<model_name>
```

Flag if row counts or unique PKs differ significantly.

**9c. Filter application**

Count how many rows were excluded by each filter (noise events, date cutoffs) — confirms filters are firing as expected.

**9d. Date range consistency**

`min` and `max` dates across dev and prod must align.

**9e. Null rate**

Null rate on critical columns should remain the same. If it changed, validate the reason.

**9f. Metric comparison by month**

Identify all metric columns in the model (revenue, sales_usd, quantity, cogs, spend, sessions, users, or any `_usd`, `_amount`, `_count`, `_units` suffix columns).

For each metric, run:
```sql
select
    date_trunc('month', <date_column>) as month,
    'dev'                              as env,
    sum(<metric_col>)                  as total
from analytics.<dev_schema>.<model_name>
group by ALL

union all

select
    date_trunc('month', <date_column>) as month,
    'prod'                             as env,
    sum(<metric_col>)                  as total
from analytics.model.<model_name>
group by ALL

order by month, env
```

**9g. Present the comparison table**

Format results as:

| Month | Metric | Dev | Prod | Delta | Delta % | Expected? |
|---|---|---|---|---|---|---|
| 2026-05 | sales_usd | $X | $Y | $Z | Z% | Yes / No / Investigate |

- Include a **Total** row at the bottom
- For each row where Delta % > 1%, state whether the difference is expected based on the changes made
- Flag any unexpected delta as a **data trust risk**

If the model has no date column for monthly grouping, do a flat total comparison only.

**9h. Delta investigation** _(run only if row counts or metrics diverge unexpectedly)_

Investigate the root cause of the delta between dev and prod:

1. **Anti-join on primary key**: find rows in one table that do not exist in the other. This isolates missing rows from internal duplicates.
2. **Age distribution of missing rows**: if missing rows span historical dates, it is a systemic incremental blind spot.
3. **Key overlap**: which PK or join key columns differ between the tables.
4. **Category breakdown**: group by key dimension columns (sub_channel, region, country, product) to find where the difference originates.

---

## Step 10 — Challenge the solution

After completing all prior steps, step back and evaluate the overall approach.

For each modified model, ask:
- **Is this the right layer?** Would moving it simplify downstream models?
- **Is the grain correct?** Would a different grain serve downstream consumers better without adding complexity?
- **Is the join strategy sound?** Are there simpler join paths using already-existing upstream models?
- **Is there a simpler SQL pattern?** Long CASE chains, nested CTEs, or repeated subexpressions are often a sign that an intermediate model or macro is missing.
- **Does this create downstream brittleness?** Would a schema change in an upstream source break this model silently?
- **Is an incremental model justified?** If the table is small (<1M rows), a full refresh is simpler and less error-prone.

If you identify a materially better approach:
- Describe it in 3–5 sentences
- Show a sketch of the alternative SQL or model structure
- State clearly what would need to change and what the tradeoff is
- Do not implement it without explicit user approval — propose only

---

## Final summary

Output the following at the end:

**Changeset**
- List of all files changed, with type and one-line description

**Steps completed**
- Checkbox list: which steps passed clean, which had issues that were fixed, which had issues still open

**Open issues** _(anything not yet resolved)_
- Numbered list. File, line, description, severity: Blocking / Major / Minor

**Dev vs prod delta**
- One-line verdict per model: matches / minor difference (expected) / significant difference (investigate)

**Alternative approach** _(if one was identified in Step 9)_
- Summary in 2–3 sentences

**Overall verdict**
One of:
- `Ready to merge` — all steps clean, dev/prod delta expected
- `Ready with minor fixes` — non-blocking issues remain, list them
- `Needs revision before merge` — blocking issues remain, list them explicitly
