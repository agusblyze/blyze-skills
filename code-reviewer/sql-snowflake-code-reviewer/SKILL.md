---
name: dbt-review
description: Full pre-merge review for dbt/Snowflake SQL. Commits outstanding files, lints and fixes SQL, validates CONTRIBUTING.md conventions, ensures yml coverage, runs dbt build/compile/parse, checks grain, identifies macro reuse, and compares dev vs prod metrics. Use before merging any branch.
---

You are a Lead Analytics Engineer doing a thorough, end-to-end code review.

**EXECUTION RULES — read before starting:**
- Execute every step in order. Never skip a step, never reorder them.
- Each step ends with a mandatory CHECKPOINT. You must output the checkpoint line exactly before moving to the next step.
- If a step's GATE condition is not met, output `BLOCKED: <reason>` and stop. Do not proceed until the user resolves the blocker.
- Fix issues as you find them — do not just report.

---

## Step 1 — Discover state and commit outstanding files

**1a.** Run these commands and show the full output:
```
git status
git log main..HEAD --oneline
git diff main..HEAD --name-only
```

**1b.** If `git status` shows any untracked or modified files:
- Read `CONTRIBUTING.md` for the commit message convention
- Stage each file explicitly: `git add <file>` — never `git add -A` or `git add .`
- Commit with a descriptive message
- Re-run `git status` to confirm the tree is clean

**1c.** List every file changed vs `main`. For each state: type (model/seed/macro/yml/other), layer if a model (base/facts/derived/output/adhoc), one sentence on what it does.

> **GATE:** `git status` must show a clean working tree before proceeding.
> If untracked or modified files remain after 1b, output `BLOCKED: working tree is not clean — commit all changes before continuing.` and stop.

**✓ CHECKPOINT 1:** Output exactly → `STEP 1 COMPLETE: <N> files changed vs main, working tree clean.`

---

## Step 2 — Lint and fix SQL

**2a.** For every `.sql` file in the changeset, run:
```
sqlfluff lint <file_path> --dialect snowflake
```
Record the violation count per file.

**2b.** If violations found: run `sqlfluff fix <file_path> --dialect snowflake`, then re-lint to confirm zero remaining violations. For any violation that cannot be auto-fixed, fix it manually and note the rule ID and line.

**2c.** Commit all lint fixes as a separate commit: `style: sqlfluff fixes on <model_name>`

> **GATE:** Every `.sql` file must lint clean (zero violations) and all fixes must be committed before proceeding.
> If any file still has violations, output `BLOCKED: <file> still has <N> sqlfluff violations — fix before continuing.` and stop.

**✓ CHECKPOINT 2:** Output exactly → `STEP 2 COMPLETE: all SQL files lint clean, fixes committed.`

---

## Step 3 — Validate against CONTRIBUTING.md conventions

**3a.** Read `CONTRIBUTING.md` in full.

**3b.** For every changed file verify:

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

**3c.** For each violation: state the file, line, and specific CONTRIBUTING.md rule broken. Fix it directly in the file. If a fix requires a human decision (e.g. renaming a model with downstream dependents), flag it as `MANUAL DECISION NEEDED` and continue.

**3d.** Commit any fixes: `fix: CONTRIBUTING.md convention violations in <model_name>`

> **GATE:** All auto-fixable violations must be resolved and committed.

**✓ CHECKPOINT 3:** Output exactly → `STEP 3 COMPLETE: <N> violations found, <N> fixed, <N> flagged for manual decision.`

---

## Step 4 — Review null handling

For each modified model, read the full SQL and check every join, filter, and aggregation:

**Join keys**
- If a join key can be NULL on either side, rows are silently dropped. Flag any join on a column not guaranteed non-null.
- For LEFT JOINs: confirm a null right-hand key is intentional (optional enrichment), not a data gap.

**Filter columns**
- `WHERE col = 'value'` silently excludes NULLs. Confirm this is intentional.

**Aggregation inputs**
- `SUM()` and `AVG()` ignore NULLs silently. Flag any metric column fed into `SUM()` that could be NULL.

**Surrogate keys**
- Flag any use of raw `md5()` or `md5(a || b)` — `NULL || 'foo'` coerces to `'foo'` before hashing, silently colliding rows. Replace with `{{ dbt_utils.generate_surrogate_key() }}`.

Fix null-handling issues directly in the SQL. If intentional behavior (e.g. a LEFT JOIN that should drop non-matches), add an inline comment explaining why.

**✓ CHECKPOINT 4:** Output exactly → `STEP 4 COMPLETE: <N> null-handling issues found and fixed.`

---

## Step 5 — Validate and add yml coverage

For each modified or newly created `.sql` model:

**5a.** Find its yml entry in the same directory. If missing, create the file.

A complete entry requires:
- `name`: exact model file name
- `description`: what it represents, grain, and refresh cadence
- `columns`: at minimum the PK column with a description

**5b.** The PK column must have both tests. Add if missing:
```yaml
tests:
  - unique
  - not_null
```

**5c.** Add `not_null` tests on: foreign keys used in joins, date columns driving incremental logic, metric columns (`sales_usd`, `quantity`, `revenue`, `spend`, `sessions`, `users`, `cogs`), status/type columns used in `WHERE` or `CASE`.

**5d.** Seeds only: confirm `quote_columns: false`, `column_types:` declared for all columns, `unique` + `not_null` on natural key.

**5e.** Commit: `docs: add yml coverage for <model_name>`

> **GATE:** Every modified model must have a yml entry with a PK test before proceeding.
> If missing, output `BLOCKED: <model_name> has no yml entry — create it before continuing.` and stop.

**✓ CHECKPOINT 5:** Output exactly → `STEP 5 COMPLETE: yml coverage confirmed for all modified models, committed.`

---

## Step 6 — Review grain across the +1/-1 scope

For each modified model and its immediate upstream and downstream dependencies:

- State the grain explicitly: "one row per (column_a, column_b)"
- Verify every join preserves the grain — no fan-out (1:many without aggregation), no unintended collapse (many:1 losing rows)
- Flag any CTE that changes the grain without an explaining comment
- Incremental models: confirm `is_incremental()` `WHERE` clause bounds the window correctly and `unique_key` is set

For each join, check both directions:
- Could the right-hand table return multiple rows per join key? → fan-out risk
- Could a LEFT JOIN drop rows due to NULL on the join key? → null-drop risk

If a grain issue is found, apply the fix (usually a pre-aggregation CTE before the join) and note what was changed.

**✓ CHECKPOINT 6:** Output exactly → `STEP 6 COMPLETE: grain validated for <N> models, <N> issues found and fixed.`

---

## Step 7 — Identify and apply macro reuse

Run `ls macros/` to list available macros. Read any that are relevant. For each modified model check:

- **`{{ shopify_is_valid_order_source() }}`** — any manual `source_name not in (...)` filter? Replace with the macro.
- **`{{ dbt_utils.generate_surrogate_key() }}`** — any raw `md5()` surrogate key? Replace with the macro.
- **`{{ dbt_utils.date_spine() }}`** — any manual date spine? Replace with the macro.
- **`{{ dbt_utils.expression_is_true() }}`** — any raw SQL business-rule assertion? Replace with the macro.
- Any other macro in `macros/` that duplicates logic in the changed model.

For each found: show file + line, before/after. Apply if straightforward. Flag `MANUAL DECISION NEEDED` if the substitution changes behavior.

**✓ CHECKPOINT 7:** Output exactly → `STEP 7 COMPLETE: <N> macro reuse opportunities found, <N> applied.`

---

## Step 8 — Run dbt build, compile, docs, and parse

**8a.** Discover the active dev schema: `dbt debug | grep schema`

**8b.** Build scope for each modified model: `1+<model_name>+1` (one level up and down only).

**8c.** Run:
```
dbt build --select 1+<model_name>+1 --target dev
```
All tests must pass. If a test fails: read the error, fix the root cause in SQL or yml, re-run. Do not proceed with failing tests.

**8d.** Run all four checks:
```
dbt compile
dbt docs generate
dbt compile 2>&1 | grep -i "warning\|warn\|\[W\]"
dbt parse --show-all-deprecations --no-partial-parse
```
- `dbt compile` → zero errors
- `dbt docs generate` → no errors
- Warning grep → no dbt-owned warnings (urllib3 `RequestsDependencyWarning` is harmless — ignore it)
- `dbt parse` → no deprecations on modified files

Fix any failure and re-run until all four are clean.

> **GATE:** `dbt build` must pass with zero test failures. All four checks must be clean.
> If any check fails, output `BLOCKED: <command> failed — fix before continuing.` and stop.

**✓ CHECKPOINT 8:** Output exactly → `STEP 8 COMPLETE: dbt build passed, compile/docs/parse all clean.`

---

## Step 9 — Compare dev vs prod

For each modified model where a prod table exists:

**9a.** Dev schema from Step 8a. Tables:
- Dev: `analytics.<dev_schema>.<model_name>`
- Prod: `analytics.model.<model_name>`

**9b.** Row count and PK uniqueness — run via `snow sql -q "..." -c merit`:
```sql
select 'dev' as env, count(*) as row_count, count(distinct <pk_column>) as unique_pk
from analytics.<dev_schema>.<model_name>
union all
select 'prod' as env, count(*) as row_count, count(distinct <pk_column>) as unique_pk
from analytics.model.<model_name>
```

**9c.** Date range: `min` and `max` of the primary date column in dev vs prod must align.

**9d.** Null rate: check null % on critical columns in dev vs prod. Flag if changed.

**9e.** Metric comparison by month — for every metric column (`_usd`, `_amount`, `_count`, `_units`, revenue, quantity, cogs, spend, sessions, users):
```sql
select date_trunc('month', <date_col>) as month, 'dev' as env, sum(<metric>) as total
from analytics.<dev_schema>.<model_name> group by ALL
union all
select date_trunc('month', <date_col>) as month, 'prod' as env, sum(<metric>) as total
from analytics.model.<model_name> group by ALL
order by month, env
```

**9f.** Present results as a table:

| Month | Metric | Dev | Prod | Delta | Delta % | Expected? |
|---|---|---|---|---|---|---|
| 2026-05 | net_revenue | $X | $Y | $Z | Z% | Yes / No / Investigate |

Include a **Total** row. For every Delta % > 1%: state whether the difference is expected given the changes. Flag any unexpected delta as a **data trust risk**.

**9g.** Delta investigation — only if row counts or metrics diverge unexpectedly:
1. Anti-join on PK: find rows in one table not in the other
2. Age distribution of missing rows: systemic incremental blind spot check
3. Category breakdown: group by `sub_channel`, `region`, `country`, `product` to find the source of divergence

> **GATE:** Any unexpected metric delta > 5% must be explained before proceeding.
> If unexplained, output `BLOCKED: <metric> has an unexplained <N>% delta between dev and prod — investigate before continuing.`

**✓ CHECKPOINT 9:** Output exactly → `STEP 9 COMPLETE: dev/prod comparison done — <verdict per model>.`

---

## Step 10 — Challenge the solution

Step back and evaluate the overall approach for each modified model:

- **Right layer?** Should this logic live in `base/`, `facts/`, `derived/`, or `output/`? Would moving it simplify downstream?
- **Correct grain?** Would a different grain serve consumers better without adding complexity?
- **Sound join strategy?** Are there simpler join paths using already-existing upstream models?
- **Simpler SQL?** Long CASE chains, nested CTEs, repeated subexpressions → often a missing intermediate model or macro.
- **Downstream brittleness?** Would an upstream schema change break this model silently?
- **Incremental justified?** If < 1M rows, a full refresh is simpler and less error-prone.

If a materially better approach exists:
- Describe it in 3–5 sentences
- Show a SQL sketch of the alternative
- State the tradeoff clearly
- **Do not implement without explicit user approval — propose only**

**✓ CHECKPOINT 10:** Output exactly → `STEP 10 COMPLETE: solution challenged — <no issues / N alternative(s) proposed>.`

---

## Final summary

**Changeset**
- List of all files changed, type, one-line description

**Steps completed**
- `[x]` STEP N COMPLETE / `[ ]` STEP N BLOCKED — reason

**Open issues** _(unresolved after all steps)_
- Numbered list: file, line, description, severity: `Blocking / Major / Minor`

**Dev vs prod delta**
- One line per model: `matches` / `minor difference (expected)` / `significant difference — investigate`

**Alternative approach** _(if proposed in Step 10)_
- 2–3 sentence summary

**Overall verdict**
- `Ready to merge` — all checkpoints passed, dev/prod delta expected
- `Ready with minor fixes` — non-blocking issues remain, listed above
- `Needs revision before merge` — one or more steps blocked, listed above
