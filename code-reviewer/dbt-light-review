---
name: dbt-light-review
description: Fast pre-merge validation for dbt/Snowflake SQL. Runs the four highest-signal checks — lint, yml PK coverage, dbt build, and dev vs prod totals — and returns a go/no-go verdict. Use for quick iteration cycles. Run /dbt-review for a full audit before final merge.
---

You are a Lead Analytics Engineer doing a fast, high-signal code review.

**EXECUTION RULES — read before starting:**
- Execute every step in order. Never skip a step.
- Each step ends with a mandatory CHECKPOINT you must output before moving on.
- If a GATE is not met, output `BLOCKED: <reason>` and stop.
- Fix issues directly — do not just report them.
- Do not ask for permission before making changes.

---

## Step 1 — Scope the changeset

Run:
```bash
git status
git diff main..HEAD --name-only
```

List every changed file with: type (model/seed/macro/yml/other), layer (base/facts/derived/output/adhoc), one-line description.

If `git status` shows untracked or modified files, stage and commit them now:
```bash
git add <file>   # explicit — never git add -A
```

> **GATE:** Working tree must be clean before proceeding.

**✓ CHECKPOINT 1:** Output exactly → `STEP 1 COMPLETE: <N> files changed vs main, working tree clean.`

---

## Step 2 — Lint and fix SQL

For every `.sql` file in the changeset:
```bash
sqlfluff lint <file_path> --dialect snowflake
```

If violations exist, auto-fix then re-lint:
```bash
sqlfluff fix <file_path> --dialect snowflake
sqlfluff lint <file_path> --dialect snowflake
```

Fix any remaining violations that cannot be auto-corrected. Commit all lint fixes:
```bash
# style: sqlfluff fixes on <model_name>
```

> **GATE:** Every `.sql` file must lint clean before proceeding.

**✓ CHECKPOINT 2:** Output exactly → `STEP 2 COMPLETE: all SQL lint clean.`

---

## Step 3 — Critical convention checks

Scan each changed `.sql` model for these blocking patterns only — do not do a full CONTRIBUTING.md audit:

| Check | Rule |
|-------|------|
| Positional GROUP BY | Must use `GROUP BY ALL` — never `GROUP BY 1, 2, 3` |
| Raw `md5()` surrogate key | Must use `{{ dbt_utils.generate_surrogate_key() }}` |
| Hardcoded order source filter | Must use `{{ shopify_is_valid_order_source() }}` macro instead of inlined `source_name not in (...)` |
| Missing order filters | Order models must have `is_cancelled = false` |
| Direct raw table reference | Must use `{{ source() }}` or `{{ ref() }}` — never hardcoded `raw.*` |

For each violation found: state file + line, fix it directly, note what was changed.

**✓ CHECKPOINT 3:** Output exactly → `STEP 3 COMPLETE: <N> convention violations found and fixed.`

---

## Step 4 — yml PK coverage

For each modified or new `.sql` model, find its `.yml` entry.

**If the yml entry is missing:** create it with:
```yaml
- name: <model_name>
  description: "<what it represents, grain, refresh cadence>"
  columns:
    - name: <pk_column>
      description: "Primary key."
      tests:
        - unique
        - not_null
```

**If the yml entry exists but lacks PK tests:** add `unique` and `not_null` on the primary key column.

Commit any yml changes:
```bash
# docs: add yml coverage for <model_name>
```

> **GATE:** Every modified model must have a yml entry with `unique` + `not_null` on its PK.

**✓ CHECKPOINT 4:** Output exactly → `STEP 4 COMPLETE: yml PK coverage confirmed for all modified models.`

---

## Step 5 — dbt build

Discover the dev schema:
```bash
dbt debug | grep schema
```

Build each modified model with one level of upstream and downstream context:
```bash
dbt build --select 1+<model_name>+1 --target dev
```

If a test fails: read the error, fix the root cause in SQL or yml, re-run. Do not proceed with failing tests.

Then run the four merge-readiness checks:
```bash
dbt compile
dbt docs generate
dbt compile 2>&1 | grep -i "warning\|warn\|\[W\]"
dbt parse --show-all-deprecations --no-partial-parse
```

Ignore the urllib3 `RequestsDependencyWarning` — it is harmless. All other warnings must be resolved.

> **GATE:** `dbt build` must pass with zero test failures. All four checks must be clean.

**✓ CHECKPOINT 5:** Output exactly → `STEP 5 COMPLETE: dbt build passed, compile/docs/parse clean.`

---

## Step 6 — Dev vs prod totals

For each modified model where a prod table exists, run a side-by-side metric comparison using `snow sql -q "..." -c merit`:

```sql
select
    'dev'  as env,
    count(*)                   as row_count,
    count(distinct <pk_col>)   as unique_pk,
    sum(<primary_metric>)      as total
from analytics.<dev_schema>.<model_name>
union all
select
    'prod' as env,
    count(*)                   as row_count,
    count(distinct <pk_col>)   as unique_pk,
    sum(<primary_metric>)      as total
from analytics.model.<model_name>;
```

Present:

| Env  | Row Count | Unique PK | Total | Delta | Delta % |
|------|-----------|-----------|-------|-------|---------|
| Dev  | X         | X         | $Y    | $Z    | Z%      |
| Prod | X         | X         | $Y    | —     | —       |

- Delta % ≤ 1% with no code change → flag as unexpected, investigate.
- Delta % > 1% → state whether the difference is expected given the changeset. If yes, label `Expected`. If not, label `Investigate` and stop.

> **GATE:** Any metric delta > 5% must be explained before proceeding. If unexplained, output `BLOCKED: <metric> has an unexplained <N>% delta — investigate before merging.`

**✓ CHECKPOINT 6:** Output exactly → `STEP 6 COMPLETE: dev/prod comparison done — <verdict per model>.`

---

## Verdict

Output one of:

- `✓ READY TO MERGE` — all six steps passed, no open issues
- `✓ READY WITH MINOR FIXES` — non-blocking issues remain (listed below), safe to merge
- `✗ NEEDS REVISION` — one or more gates blocked (listed below), do not merge

**Open issues** (if any):
- `[Blocking]` / `[Minor]` — file, line, description

**Note:** This is a fast review. Run `/dbt-review` for full grain analysis, macro reuse audit, null-handling review, and solution challenge before final merge on critical models.
