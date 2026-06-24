---
name: dbt-review
description: Full pre-merge review for dbt/Snowflake SQL. Commits outstanding files, lints and fixes SQL, runs a parallel multi-dimensional review (conventions, null handling, yml coverage, grain/joins, macro reuse), runs dbt build/compile/parse, and compares dev vs prod metrics. Use before merging any branch.
---

You are a Lead Analytics Engineer doing a thorough, end-to-end code review.

**EXECUTION RULES — read before starting:**
- Execute every step in order. Never skip a step, never reorder them.
- Each step ends with a mandatory CHECKPOINT. You must output the checkpoint line exactly before moving to the next step.
- If a step's GATE condition is not met, output `BLOCKED: <reason>` and stop. Do not proceed until the user resolves the blocker.
- Fix issues as you find them — do not just report.
- Do not wait for user permission to make changes, apply them directly.

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

## Step 3 — Multi-Dimensional Review

You are performing a comprehensive, multi-agent code review.

Launch one agent per dimension using the Agent tool. **ALL agents MUST run in the background (`run_in_background: true`) to maximize parallelism.** Send all five in a single message.

**Do not duplicate work.** If an agent is reading files, do not read the same files yourself. Pass each agent the full list of changed `.sql` and `.yml` files identified in Step 1c.

---

### Agent 1 — Conventions
Read `CONTRIBUTING.md` in full. Then read each changed file in the scope. For every `.sql` file report violations of:
1. `GROUP BY ALL` vs positional `GROUP BY 1, 2, 3`
2. No `SELECT *` in production models
3. CTEs over subqueries
4. Lowercase snake_case column aliases
5. Snowflake date functions (`dateadd`, `datediff`, `date_trunc`) — not ANSI or BigQuery syntax
6. No `ILIKE` where an exact match is sufficient
7. Model naming prefix matches layer (`base_<source>__<name>`, `fact_<entity>`, `derived_*`, `output_*`)
8. Seeds named `seed_{topic}__{name}.csv`, CSV headers lowercase and BOM-free, referenced via `{{ ref() }}` everywhere

Report each violation as: file · line · rule broken · suggested fix.

---

### Agent 2 — Null handling
Read each changed `.sql` file in the scope. For every file check:
1. Join keys that can be NULL on either side — rows are silently dropped on INNER JOINs
2. LEFT JOINs where a NULL right-side key may not be intentional (data gap vs optional enrichment)
3. `WHERE col = 'value'` silently excluding NULLs — confirm intentional
4. `SUM()` or `AVG()` on a column that could be NULL
5. Raw `md5()` or `md5(a || b)` surrogate key — `NULL || 'foo'` coerces before hashing, silently colliding rows; replace with `{{ dbt_utils.generate_surrogate_key() }}`

Report each issue as: file · line · issue type · severity (Blocking / Major / Minor) · suggested fix.

---

### Agent 3 — YML coverage
Read each changed `.sql` file and its corresponding `.yml` file in the scope. For each model verify:
1. A yml entry exists with a `description` that includes grain and refresh cadence
2. Primary key column has both `unique` and `not_null` tests
3. Foreign key columns used in JOINs have `not_null` tests
4. Metric columns (`sales_usd`, `quantity`, `revenue`, `spend`, `sessions`, `users`, `cogs`) have `not_null` tests
5. Status/type columns used in `WHERE` or `CASE` have `accepted_values` tests
6. Seeds: `quote_columns: false`, `column_types:` declared for all columns, `unique` + `not_null` on natural key

Report each gap with the exact YAML to add.

---

### Agent 4 — Grain and joins
Read each changed `.sql` file in the scope. Also read the direct upstream and downstream `.sql` files (one level in each direction via `ref()` calls). For each model:
1. State the grain explicitly: "one row per (col_a, col_b)"
2. For every join, check both directions: could the right-hand table return multiple rows per join key (fan-out)? could a LEFT JOIN drop rows from a NULL join key?
3. Flag any CTE that changes the grain without a comment
4. For incremental models: confirm the `is_incremental()` WHERE clause correctly bounds the window and `unique_key` is set

Report each issue as: file · CTE or join name · issue type · suggested fix (usually a pre-aggregation CTE before the join).

---

### Agent 5 — Macro reuse
Run `ls macros/` and read any macro that is relevant to the scope. Then read each changed `.sql` file. Check for:
1. Manual `source_name not in (...)` filter → replace with `{{ shopify_is_valid_order_source() }}`
2. Raw `md5()` surrogate key → replace with `{{ dbt_utils.generate_surrogate_key() }}`
3. Manual date spine generation → replace with `{{ dbt_utils.date_spine() }}`
4. Raw SQL business-rule assertion → replace with `{{ dbt_utils.expression_is_true() }}`
5. Any other logic in `macros/` that duplicates inline logic in the changed model

Report each opportunity as: file · line · current code snippet · macro to use · before/after example.

---

Wait for all five agents to complete before proceeding to Step 4.

**✓ CHECKPOINT 3:** Output exactly → `STEP 3 COMPLETE: 5 review agents launched and complete.`

---

## Step 4 — Synthesize findings and apply fixes

Collect the output from all five agents. For each finding:
- **Auto-fixable:** fix it directly in the file. Do not ask for permission.
- **Requires human decision** (e.g. renaming a model with downstream dependents, a macro substitution that changes behavior): flag as `MANUAL DECISION NEEDED` and continue.

Commit fixes grouped by type:
```bash
fix: CONTRIBUTING.md convention violations in <model_name>
fix: null handling issues in <model_name>
docs: yml coverage for <model_name>
fix: grain/join issues in <model_name>
fix: replace inline logic with macros in <model_name>
```

Present a consolidated findings table:

| Dimension | File | Line | Issue | Severity | Status |
|-----------|------|------|-------|----------|--------|
| Conventions | model.sql | 42 | positional GROUP BY | Minor | Fixed |
| Null handling | model.sql | 78 | SUM() on nullable column | Major | Fixed |
| YML coverage | model.yml | — | missing not_null on sales_usd | Major | Fixed |
| Grain | model.sql | 55 | fan-out risk on customer join | Blocking | Fixed |
| Macro reuse | model.sql | 31 | inline source_name filter | Minor | Fixed |

> **GATE:** All Blocking and Major findings must be resolved or escalated before proceeding.

**✓ CHECKPOINT 4:** Output exactly → `STEP 4 COMPLETE: <N> findings across 5 dimensions — <N> fixed, <N> flagged for manual decision.`

---

## Step 5 — Run dbt build, compile, docs, and parse

**5a.** Discover the active dev schema: `dbt debug | grep schema`

**5b.** Build scope for each modified model: `1+<model_name>+1` (one level up and down only).

**5c.** Run:
```
dbt build --select 1+<model_name>+1 --target dev
```
All tests must pass. If a test fails: read the error, fix the root cause in SQL or yml, re-run. Do not proceed with failing tests.

**5d.** Run all four checks:
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

**✓ CHECKPOINT 5:** Output exactly → `STEP 5 COMPLETE: dbt build passed, compile/docs/parse all clean.`

---

## Step 6 — Compare dev vs prod

For each modified model where a prod table exists:

**6a.** Dev schema from Step 5a. Tables:
- Dev: `analytics.<dev_schema>.<model_name>`
- Prod: `analytics.model.<model_name>`

**6b.** Row count and PK uniqueness — run via `snow sql -q "..." -c merit`:
```sql
select 'dev' as env, count(*) as row_count, count(distinct <pk_column>) as unique_pk
from analytics.<dev_schema>.<model_name>
union all
select 'prod' as env, count(*) as row_count, count(distinct <pk_column>) as unique_pk
from analytics.model.<model_name>
```

**6c.** Date range: `min` and `max` of the primary date column in dev vs prod must align.

**6d.** Null rate: check null % on critical columns in dev vs prod. Flag if changed.

**6e.** Metric comparison by month — for every metric column (`_usd`, `_amount`, `_count`, `_units`, revenue, quantity, cogs, spend, sessions, users):
```sql
select date_trunc('month', <date_col>) as month, 'dev' as env, sum(<metric>) as total
from analytics.<dev_schema>.<model_name> group by ALL
union all
select date_trunc('month', <date_col>) as month, 'prod' as env, sum(<metric>) as total
from analytics.model.<model_name> group by ALL
order by month, env
```

**6f.** Present results as a table:

| Month | Metric | Dev | Prod | Delta | Delta % | Expected? |
|---|---|---|---|---|---|---|
| 2026-05 | net_revenue | $X | $Y | $Z | Z% | Yes / No / Investigate |

Include a **Total** row. For every Delta % > 1%: state whether the difference is expected given the changes. Flag any unexpected delta as a **data trust risk**.

**6g.** Delta investigation — only if row counts or metrics diverge unexpectedly:
1. Anti-join on PK: find rows in one table not in the other
2. Age distribution of missing rows: systemic incremental blind spot check
3. Category breakdown: group by `sub_channel`, `region`, `country`, `product` to find the source of divergence

> **GATE:** Any unexpected metric delta > 5% must be explained before proceeding.
> If unexplained, output `BLOCKED: <metric> has an unexplained <N>% delta between dev and prod — investigate before continuing.`

**✓ CHECKPOINT 6:** Output exactly → `STEP 6 COMPLETE: dev/prod comparison done — <verdict per model>.`

---

## Step 7 — Challenge the solution

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

**✓ CHECKPOINT 7:** Output exactly → `STEP 7 COMPLETE: solution challenged — <no issues / N alternative(s) proposed>.`

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

**Alternative approach** _(if proposed in Step 7)_
- 2–3 sentence summary

**Overall verdict**
- `Ready to merge` — all checkpoints passed, dev/prod delta expected
- `Ready with minor fixes` — non-blocking issues remain, listed above
- `Needs revision before merge` — one or more steps blocked, listed above
