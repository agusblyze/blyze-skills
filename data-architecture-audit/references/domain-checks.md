# Domain Checks — Verifiable Audit Checklist

Every check below is written to be **mechanically verifiable** against a client's
repo, warehouse, or BI tool. Where possible, the command or pattern to run is included.

Each finding gets an **Impact** and **Effort** score — see `scoring-rubric.md`.

> **How to use**: Work top to bottom per domain. For each check, record PASS / FAIL /
> N/A and capture the specific evidence (file names, counts, config values). Evidence
> is what makes the report credible — never write a finding without a number or filename.

---

## PRE-AUDIT: Does the project even build?

Run these first. If the project doesn't compile, everything downstream is suspect.

| # | Check | How to verify | If FAIL |
|---|---|---|---|
| P1 | Project compiles | `dbt compile` runs with no errors | Critical — document every compile error before continuing |
| P2 | All models build | `dbt build` (or `dbt run`) completes | List failing models; note error types |
| P3 | All tests pass | `dbt test` — record pass/fail/error/warn counts | List failing tests |
| P4 | No parse errors | `dbt parse` clean | Document YAML/Jinja syntax issues |
| P5 | dbt version | Check `dbt_project.yml` `require-dbt-version` + `dbt --version` | Flag if >2 minor versions behind current stable (e.g. on 1.6 when 1.8 is out) |
| P6 | Packages resolve | `dbt deps` succeeds; review `packages.yml` | Note missing/outdated packages (dbt_utils, dbt_expectations) |
| P7 | Dependencies clean | `dbt ls` runs; check for disabled/orphaned models | List orphaned or disabled models |

---

## Domain 1: Project Structure & Naming Convention

### 1.1 Layer architecture
| # | Check | How to verify |
|---|---|---|
| 1.1.1 | Standard layer names | Folders are `staging/` `intermediate/` `marts/` — NOT `base/` `facts/` `output/` |
| 1.1.2 | Staging = 1 model per source table | Each `stg_` model maps to exactly one raw table |
| 1.1.3 | Intermediate layer exists | Cross-source/shared logic lives in `intermediate/`, not in marts |
| 1.1.4 | Marts organized by business domain | `marts/marketing/`, `marts/customer/`, `marts/finance/`, `marts/core/` |
| 1.1.5 | No ad-hoc / temp / sync folders | Flag `adhoc/`, `sync/`, `temp/`, `scratch/` — relocate or remove |
| 1.1.6 | reverse_etl / outbound isolated | Models that sync OUT to 3rd-party tools live in a dedicated `outbound/` (or `reverse_etl/`) folder |

### 1.1b Outbound / Reverse-ETL layer
Models that push data OUT to third-party tools (Klaviyo lists, ad-platform audiences,
Google Sheets write-back, CRM enrichment, Census/Hightouch syncs) are a distinct concern
from analytical marts and must be isolated.

| # | Check | How to verify | Severity |
|---|---|---|---|
| 1.1b.1 | Dedicated `outbound/` folder | All reverse-ETL models live in one folder, not mixed into marts | **Mid** |
| 1.1b.2 | Naming flags the destination | Models named to show the target tool/sync (e.g. `outbound_klaviyo__active_subscribers.sql`) | **Mid** |
| 1.1b.3 | Built FROM marts, not raw | Outbound models select from governed marts via `ref()`, never raw/staging | **High** |
| 1.1b.4 | Each sync documented | YAML notes destination tool, sync tool (Census/Hightouch/custom), cadence, owner | **Mid** |
| 1.1b.5 | Sync inventory complete | Every active reverse-ETL sync has a corresponding model (no syncs querying ad-hoc SQL) | **Mid** |
| 1.1b.6 | PII handling flagged | Outbound models carrying PII tagged in `meta:` and access-reviewed | **High** |

### 1.2 Naming convention (institutionalize a standard)
| # | Check | How to verify |
|---|---|---|
| 1.2.1 | Staging follows `stg_{source}__{object}.sql` | grep filenames; flag deviations (e.g. source-prefix without double underscore) |
| 1.2.2 | Intermediate follows `int_{domain}__{verb}.sql` | grep filenames |
| 1.2.3 | Marts follow `fct_`/`dim_{domain}__{noun}.sql` | grep filenames |
| 1.2.4 | No models mislabeled as marts | Flag raw-style names in marts (e.g. `bigquery_ga4_raw_events_daily_mart.sql`) |
| 1.2.5 | No per-report duplicate marts | Flag marts that duplicate logic per report (e.g. a `/base/fct_marketing_order.sql` nested inside a tracker) |
| 1.2.6 | Consistent prefix coverage | Count models with NO type prefix at all |
| 1.2.7 | Adopt dimensional + medallion + OBT vocabulary | Confirm the project documents which modeling concepts it uses |

### 1.3 Sources — every source declared with freshness
| # | Check | How to verify | Severity |
|---|---|---|---|
| 1.3.1 | `_{source}__sources.yml` exists per staging folder | Every source folder has a sources file | **High / Low effort** |
| 1.3.2 | NO model reads from `raw.*` directly | grep for hardcoded `raw.` / fully-qualified table refs in SQL — every raw read must go through `{{ source() }}` | **High** |
| 1.3.3 | Every source has freshness thresholds | Each source table has `loaded_at_field` + `freshness:` block | **High** |
| 1.3.4 | Freshness thresholds match SLA by source type | See table below | **High** |

**Freshness threshold standard (adapt per client SLA):**
| Source type | warn_after | error_after |
|---|---|---|
| Commerce (Shopify orders/customers/products) | 4h | 8h |
| Email/SMS (Klaviyo, Attentive) | 6h | 12h |
| Analytics (GA4 events) | 12h | 24h |
| Paid media (Facebook/Google/TikTok Ads) | 24h | 36h |
| Manual / retail feeds (Sheets, Sephora, Kohl's) | — | 7 days |

### 1.4 Manual / hardcoded references → must use `ref()` and `source()`
| # | Check | How to verify | Severity |
|---|---|---|---|
| 1.4.1 | No hardcoded table names in `from`/`join` | grep SQL for `from <db>.<schema>.<table>` patterns not wrapped in `ref()`/`source()` | **Mid** |
| 1.4.2 | Manual input tables referenced via `ref()` | List every model hitting a manual/Sheet table without `ref()` — record file + hit count | **Mid** |
| 1.4.3 | Cross-model joins use `ref()` | No mart selecting from another mart by raw schema path | **Mid** |

> Output a table like: `| File | Hardcoded hits | Layer |` so the client can fix systematically.

### 1.5 Repetitive code → encapsulate in macros
| # | Check | How to verify | Severity |
|---|---|---|---|
| 1.5.1 | Identify repeated inline filters | grep for the same WHERE clause across many models (e.g. valid-order filter: `source_name not in (...) and is_cancelled = false`). Count call sites. | **High** if widespread |
| 1.5.2 | Identify repeated transforms | Timezone conversions (`convert_timezone(...)`), cents→dollars, email cleaning, tax stripping — count occurrences | **Mid** |
| 1.5.3 | Identify repeated joins | Same dimension/calendar join repeated across N models → candidate macro | **Mid** |
| 1.5.4 | Macros organized by topic | `macros/utils/`, `macros/{source}/`, `macros/{domain}/` | **Mid / Low effort** |
| 1.5.5 | Dead macros removed | Find macros with 0 call sites; flag for deletion | **Low** |
| 1.5.6 | Repeated reference data → seed | Channel mappings, code lookups done via inline CASE WHEN → move to a seed joined by key | **Mid** |

> For each macro candidate, output: proposed macro name, files where it should be used,
> call-site count, and proposed logic. (See `macro-candidates-example.md` for format.)

### 1.5b Query complexity → simplify
Beyond extracting macros, flag individual queries that are more complex than they need
to be. Complexity hides bugs and slows onboarding.

| # | Check | How to verify | Severity |
|---|---|---|---|
| 1.5b.1 | Deeply nested subqueries | Flag models with 3+ levels of nested subqueries → refactor into CTEs or intermediate models | **Mid** |
| 1.5b.2 | Overly long CTE chains | Models with 10+ CTEs doing unrelated work → split into intermediate models | **Mid** |
| 1.5b.3 | Redundant joins | Same table joined multiple times where one join would do; joins whose columns are never selected | **Mid** |
| 1.5b.4 | Window functions that could be aggregates | Over-engineered `row_number()`/`partition` logic replaceable by `group by` | **Low** |
| 1.5b.5 | Repeated CASE WHEN that should be a seed/macro | Large inline mapping blocks → seed lookup or macro | **Mid** |
| 1.5b.6 | `select *` in marts | Explicit column lists in stakeholder-facing models | **Low** |
| 1.5b.7 | Dead columns / unused CTEs | CTEs defined but never referenced; columns selected but never consumed downstream | **Low** |

> For each flagged model, note WHY it's complex and the proposed simplification
> (e.g. "303-line orders mart → extract customer-attribution and refund logic into two
> intermediate models; replace 4 nested subqueries with CTEs").

### 1.5c Inconsistent logic → reconcile to one definition
The same business concept computed differently in different places is a top source of
"the numbers don't match." Hunt these down explicitly.

| # | Check | How to verify | Severity |
|---|---|---|---|
| 1.5c.1 | Conflicting metric definitions | Compare how revenue / orders / customers / sessions are calculated across models. Flag any divergence (e.g. one model includes tax, another strips it). | **High** |
| 1.5c.2 | Conflicting filters for the same concept | "Valid order" / "active customer" / "paid session" defined differently across models | **High** |
| 1.5c.3 | Conflicting date logic | Some models use order-created date, others fulfilled/paid date, for the same metric | **High** |
| 1.5c.4 | Conflicting timezone handling | Some models convert to local TZ, others stay UTC, mixing in the same report | **High** |
| 1.5c.5 | Duplicate models, same purpose | Two+ models producing the same metric with different results → pick a source of truth, deprecate the rest | **High** |
| 1.5c.6 | Inconsistent grain | Models claiming the same grain but actually at different grains (daily vs order-level) | **Mid** |
| 1.5c.7 | Inconsistent NULL/zero handling | Divisions, coalesces, and empty-state logic handled differently for the same calc | **Mid** |

> Deliverable: a "single source of truth" table — for each contested concept, list every
> place it's defined, show the divergence, and recommend the canonical definition (ideally
> centralized in a macro or upstream intermediate model so it can't drift again).

### 1.6 Seeds
| # | Check | How to verify | Severity |
|---|---|---|---|
| 1.6.1 | No oversized seeds in git | Flag any seed > ~1MB or with frequent changes — move to ETL tool | **Low** |
| 1.6.2 | No unused seeds | Cross-check each seed against `ref()` usage; list zero-reference seeds | **Low** |
| 1.6.3 | No hardcoded seed references | Seeds referenced via `{{ ref() }}`, not hardcoded | **Low** |
| 1.6.4 | Naming `seed_{source}__{topic}.csv` | grep seed filenames | **Low** |
| 1.6.5 | Seeds organized in folders by source | Check `seeds/` subfolder structure | **Low** |
| 1.6.6 | `_seeds__schema.yml` has descriptions + column types | Open seeds schema file | **Low** |

### 1.7 Orphaned warehouse tables
| # | Check | How to verify | Severity |
|---|---|---|---|
| 1.7.1 | No orphaned tables in warehouse | Compare warehouse `information_schema.tables` against dbt manifest models; list tables with no owning model | **Low** |
| 1.7.2 | Drop candidates documented | Produce a drop list with last_altered dates | **Low** |

### 1.8 SQL style & consistency
| # | Check | How to verify | Severity |
|---|---|---|---|
| 1.8.1 | `.sqlfluff` config present | File exists at root with a defined dialect + rules | **Low / High effort** |
| 1.8.2 | `.pre-commit-config.yaml` runs sqlfluff | Pre-commit hook configured | **Low** |
| 1.8.3 | Style rules documented | Rules referenced in CONTRIBUTING.md / CLAUDE.md | **Low** |

### 1.9 Onboarding docs
| # | Check | How to verify | Severity |
|---|---|---|---|
| 1.9.1 | `README.md` at root | Onboarding, branch convention, project overview | **Low** |
| 1.9.2 | `CONTRIBUTING.md` | Naming rules + PR checklist | **Low** |
| 1.9.3 | `CLAUDE.md` | Present and reflects current project structure (if client uses Claude) | **Low** |

---

## Domain 2: Data Validation & Observability

### 2.1 YAML + test coverage (every table, every column)
| # | Check | How to verify | Severity |
|---|---|---|---|
| 2.1.1 | Every model has a YAML entry | Count models vs models present in any `.yml`. Report ratio (e.g. "14 of 298"). | **High** |
| 2.1.2 | Every model has a description | Count models with non-empty `description:` | **Mid** |
| 2.1.3 | Every column documented | For each model YAML, count columns with descriptions vs total columns | **Mid** |
| 2.1.4 | Every model has a PK test | Each model declares a primary key with `unique` + `not_null` (or `dbt_utils.unique_combination_of_columns` for composite) | **High** |
| 2.1.5 | Composite PKs where needed | Grain-defined facts with multi-column keys have composite uniqueness tests | **High** |
| 2.1.6 | FK relationship tests | Foreign keys use `relationships` test to parent | **Mid** |
| 2.1.7 | accepted_values on reused categoricals | Status/channel/type columns reused downstream have `accepted_values` | **Mid** |
| 2.1.8 | Total test count recorded | `dbt ls --resource-type test` count; break down by type | **High** |

> Headline metric for the report: **"X% of models have zero tests; Y% have no YAML entry at all."**

### 2.2 Source freshness (cross-ref with 1.3)
| # | Check | How to verify | Severity |
|---|---|---|---|
| 2.2.1 | `dbt source freshness` runs | Command executes against all sources | **High** |
| 2.2.2 | Freshness actually scheduled | A dbt Cloud job runs freshness on cadence | **High** |

### 2.3 CI pipeline
| # | Check | How to verify | Severity |
|---|---|---|---|
| 2.3.1 | CI job exists | dbt Cloud CI job triggered on GitHub PR | **High / Low effort** |
| 2.3.2 | CI uses slim/state-based selection | Job runs `dbt build --select state:modified+ --defer --state <artifacts>` | **High** |
| 2.3.3 | Branch protection enabled | PR cannot merge to main unless CI passes | **High** |
| 2.3.4 | Pre-commit hook present | `.pre-commit-config.yaml` runs validations before commit | **Low** |
| 2.3.5 | AI code reviewer integrated | Optional: AI-based PR review configured | **Mid / Low effort** |

### 2.4 Metric / revenue integrity
| # | Check | How to verify | Severity |
|---|---|---|---|
| 2.4.1 | Revenue integrity test exists | A singular test asserts historical revenue stays consistent across builds | **High** |
| 2.4.2 | Core KPI guard tests | not_null/range tests on revenue, order count, customer count | **High** |
| 2.4.3 | Warehouse-vs-source reconciliation | Spot-check warehouse totals against source-of-truth (e.g. Shopify admin) | **High** |
| 2.4.4 | Region-specific revenue logic correct | Verify tax-inclusive markets (e.g. UK VAT) handled in ALL revenue models — audit each, apply a `strip_taxes_if_inclusive`-style macro | **High** |
| 2.4.5 | No future-dated / negative records | Singular tests: `assert_revenue_non_negative`, `assert_no_future_dated_orders`, `assert_orders_have_customers` | **Mid** |

### 2.5 Documentation for AI consumption
| # | Check | How to verify | Severity |
|---|---|---|---|
| 2.5.1 | dbt docs generate | `dbt docs generate` succeeds | **Mid** |
| 2.5.2 | Output marts fully column-documented | 100% column descriptions on stakeholder-facing marts | **Mid** |
| 2.5.3 | Macros documented inline | Each macro has a docstring/description | **Low** |

---

## Domain 3: Materialization & Performance

### 3.1 Materialization config
| # | Check | How to verify | Severity |
|---|---|---|---|
| 3.1.1 | Valid project-level default | `dbt_project.yml` has a VALID `+materialized:` (watch for typos like `materialized: materialized`) | **Low** but breaks builds |
| 3.1.2 | Materialization controlled at project level | Defaults set per-layer in `dbt_project.yml`, not duplicated in every model config block | **Mid** |
| 3.1.3 | Schema controlled at project level | Schema assignment configured in `dbt_project.yml` | **Mid** |

### 3.2 Materialization strategy by model type
| # | Check | How to verify | Severity |
|---|---|---|---|
| 3.2.1 | Full-refresh table count | Count models materialized as `table`. Flag if nearly all (e.g. "292 of 298"). | **High** |
| 3.2.2 | Lightweight sources → views | Google Sheets / CSV-upload staging models should be `view` | **High** |
| 3.2.3 | Large append-only facts → incremental | Orders/events/sessions should be `incremental` with `unique_key` + strategy | **High** |
| 3.2.4 | Ephemeral used where appropriate | Trivial passthroughs as `ephemeral` | **Mid** |
| 3.2.5 | Incremental config correct | Each incremental model has `unique_key`, `incremental_strategy`, and an `is_incremental()` filter | **High** |

### 3.3 Heavy / underperforming models
| # | Check | How to verify | Severity |
|---|---|---|---|
| 3.3.1 | No monster models | Flag models > ~250 lines or with > ~10 joins (e.g. a 303-line, 18-join orders mart) | **High** |
| 3.3.2 | Subqueries decomposed | Repeated subqueries pushed into intermediate models | **High** |
| 3.3.3 | Run-time outliers identified | From dbt Cloud run history / `run_results.json` / warehouse query history, rank models by execution time. List the slowest 10–15 with their durations. | **Mid** |
| 3.3.4 | Slow models scored as incremental candidates | For each slow model, assess incremental fit (see table below) and recommend a strategy | **High** |

**Incremental candidate scoring** — for each slow / large model, evaluate:

| Signal | Favors incremental? |
|---|---|
| Append-mostly (orders, events, sessions, ad spend by day) | Yes — strong candidate |
| Has a reliable updated_at / loaded_at / event timestamp | Yes — needed for the `is_incremental()` filter |
| Large row count + long full-refresh time | Yes — biggest payoff |
| Late-arriving or back-dated records common | Use a lookback window (e.g. `where event_at >= (select max(event_at) from {{ this }}) - interval '3 days'`) |
| Heavy dimensional model that fully rebuilds cheaply | No — keep as table |
| Source is a small Google Sheet / lookup | No — make it a `view` instead |
| Frequent schema changes / full historical restatements | Caution — incremental adds maintenance cost; weigh it |

For each recommended conversion, specify: `unique_key`, `incremental_strategy`
(`merge` / `delete+insert` / `append`), the `is_incremental()` predicate, and whether an
`on_schema_change` policy is needed. Note any model where an append-only pattern (insert
new orders only) is simpler and safer than full incremental.

### 3.4 Warehouse schema hygiene
| # | Check | How to verify | Severity |
|---|---|---|---|
| 3.4.1 | Schemas aligned to dbt layers | Warehouse schemas map cleanly to staging/intermediate/marts | **High / Low effort** |
| 3.4.2 | No source schema sprawl | Flag multiple schemas for one source (e.g. several GA4 / Sheets / Facebook schemas) — consolidate RAW | **Mid** |
| 3.4.3 | No dev schemas in prod | Dev/scratch schemas moved out of production DB | **Mid** |
| 3.4.4 | ANALYTICS/prod DB clean | No stale or duplicate production schemas | **Mid** |

### 3.5 Cost visibility
| # | Check | How to verify | Severity |
|---|---|---|---|
| 3.5.1 | Billing / credit report exists | Warehouse spend report built | **High / Low effort** |
| 3.5.2 | Report delivered to Slack | Weekly/monthly billing summary scheduled | **High** |
| 3.5.3 | Job efficiency reviewed | Production jobs not running unnecessary full refreshes | **Mid** |

---

## Domain 4: Data Ingestion

### 4.1 Tool inventory & consolidation
| # | Check | How to verify | Severity |
|---|---|---|---|
| 4.1.1 | Ingestion tools inventoried | List every tool (Fivetran, Stitch, Airbyte, Prefect, custom) | **Mid** |
| 4.1.2 | Consolidate to ≤2 tools | Flag if 3+ overlapping ingestion methods; propose consolidation | **Mid / High effort** |
| 4.1.3 | No duplicate connectors | Flag multiple connectors for the same source (e.g. several Shopify or GA4 connections) | **Mid** |
| 4.1.4 | Landing schema convention | RAW landing schemas follow a naming/schema convention | **Mid** |

### 4.2 Documentation & health
| # | Check | How to verify | Severity |
|---|---|---|---|
| 4.2.1 | Each connector documented | Source → connector → destination schema documented | **Mid** |
| 4.2.2 | No connectors in error state | Review connector status dashboards | **Mid** |
| 4.2.3 | Sync frequency appropriate | Frequency matches the freshness SLA in 1.3.4 | **Mid** |
| 4.2.4 | Plan within limits | Tool not exceeding plan (e.g. Stitch row caps, Fivetran MAR) | **Low** |

### 4.3 Manual ingestion risk
| # | Check | How to verify | Severity |
|---|---|---|---|
| 4.3.1 | Count manual sources | Number of Google Sheets / CSV uploads used as sources. Flag if > ~10. | **Mid** |
| 4.3.2 | Automation candidates | Identify manual feeds that should move to a managed connector | **Mid** |

### 4.4 Multi-warehouse complexity
| # | Check | How to verify | Severity |
|---|---|---|---|
| 4.4.1 | No unnecessary hops | Flag GA4 routed via BigQuery → warehouse when a native connector exists | **Mid / High effort** |
| 4.4.2 | No redundant data paths | Same data ingested via two paths | **Mid** |

---

## Domain 5: BI Layer Health

### 5.1 Version control & structure (Looker / Omni / Tableau / Power BI)
| # | Check | How to verify | Severity |
|---|---|---|---|
| 5.1.1 | BI definitions version-controlled | LookML / Omni model files in a git repo | **Mid** |
| 5.1.2 | Organized by domain | Explores/workbooks grouped by team/domain, not a flat list | **Low** |
| 5.1.3 | Dev → prod workflow | A controlled deploy process exists (e.g. Looker dev mode → production) | **Mid** |

### 5.2 Connection integrity
| # | Check | How to verify | Severity |
|---|---|---|---|
| 5.2.1 | BI reads from marts, not raw | No explore/workbook querying raw or staging directly | **High** |
| 5.2.2 | No broken explores/dashboards | List erroring content | **Mid** |
| 5.2.3 | Field-level descriptions present | Dimensions/measures documented | **Low** |

### 5.3 Usage & coverage
| # | Check | How to verify | Severity |
|---|---|---|---|
| 5.3.1 | Stale content identified | Dashboards with 0 views in 90 days | **Low** |
| 5.3.2 | Coverage for key domains | A governed dashboard exists for each business domain the client needs | **High** |
| 5.3.3 | Access/change management | Edit access controlled; change process defined | **Mid** |

---

## Domain 6: AI Readiness

Score the client on the readiness ladder (Stage 0–5) — see `scoring-rubric.md`.

### 6.1 Metadata foundation
| # | Check | How to verify | Severity |
|---|---|---|---|
| 6.1.1 | Marts column-documented | Prereq for AI: 100% column descriptions on marts | **High** |
| 6.1.2 | Meta blocks with tags | YAML `meta:` blocks tag grain, business_domain, PII, unit | **Low** |
| 6.1.3 | CLAUDE.md current | Reflects project structure + conventions | **Low** |

### 6.2 Semantic layer
| # | Check | How to verify | Severity |
|---|---|---|---|
| 6.2.1 | Metrics defined | dbt semantic layer or BI semantic layer defines core metrics | **Mid** |
| 6.2.2 | Relationships documented | Join paths/entity relationships declared | **Mid** |

### 6.3 Connectivity & access
| # | Check | How to verify | Severity |
|---|---|---|---|
| 6.3.1 | Read-only AI role | Dedicated warehouse role with read-only grants | **High** |
| 6.3.2 | Dedicated AI warehouse | Separate compute for AI workloads | **High** |
| 6.3.3 | Service account + query tags | Service account with query tagging for attribution + a query-log write process | **High** |
| 6.3.4 | MCP connection deployed | Snowflake (or equivalent) MCP connection live | **High / High effort** |

### 6.4 Skills & queries
| # | Check | How to verify | Severity |
|---|---|---|---|
| 6.4.1 | Domain skill exists | A Claude skill scoped to the client's data model + business context | **High** |
| 6.4.2 | Pre-built parameterized queries | Common business questions encoded as reusable queries | **High** |
| 6.4.3 | Team Claude Project | Project with auto-load skill, guardrails, MCP, links to docs/dashboards | **High** |

---

## Domain 7: Project Management & Process

| # | Check | How to verify | Severity |
|---|---|---|---|
| 7.1 | Ticketing system | Linear/Jira/Asana in use | **Low** |
| 7.2 | Linear ↔ GitHub integration | PRs auto-sync with tickets; branch naming includes ticket code | **Low** |
| 7.3 | `#ask-data` Slack channel | Company-wide intake → triage in ticketing tool | **Low** |
| 7.4 | `#data-team` channel | Internal team channel | **Low** |
| 7.5 | `#data-alerts` channel | All alerts (freshness, job failures, CI) unified | **Mid** |
| 7.6 | GitHub PR notifications | PR activity surfaced to the team | **Low** |
| 7.7 | Onboarding doc | Step-by-step for new joiners | **Low** |
| 7.8 | PR review required | No direct merges to main without review | **Mid** |

---

## Business Applications Coverage & Gap Analysis

Architecture exists to serve business decisions. Assess not just whether the plumbing is
clean, but whether the client can actually answer the questions a retail/e-commerce
business needs to answer. For each capability below, score it and recommend next steps.

### How to assess each capability

For every application, determine and record:

| Dimension | Question |
|---|---|
| **Exists?** | Is there a model + dashboard for this today? (Yes / Partial / No) |
| **Trustworthy?** | Does it pass validation — tested, documented, reconciled to source of truth? |
| **Consistent?** | Does it agree with related metrics, or contradict them? (link to 1.5c findings) |
| **Self-serve?** | Can the business team get it without asking the data team? |
| **Gap / improvement** | What's missing, wrong, or could be better? |

### Coverage status scale
- **✅ Solid** — exists, trustworthy, self-serve
- **🟡 Partial** — exists but untrusted, incomplete, manual, or inconsistent
- **🔴 Missing** — no reliable version exists → roadmap item
- **➕ Improvement** — exists and works, but there's a clear upgrade

### Standard retail / e-commerce capability set

| Capability | What "good" looks like | Common gaps to flag |
|---|---|---|
| Commercial pacing | Actuals vs goals/forecast, refreshed daily, by channel & region | Goals in a Sheet, no automated pacing, stale |
| Marketing attribution | Channel attribution model documented & consistent across reports | Multiple conflicting attribution logics (→ 1.5c) |
| CAC by channel / product | Spend ÷ new customers, consistent spend + customer definitions | Spend sources inconsistent; "new customer" defined differently |
| Contribution margin | Revenue − COGS − variable costs at SKU & channel grain | COGS missing or manual; margin only at top line |
| LTV:CAC ratio | Cohort LTV ÷ CAC, consistent customer scope | LTV horizon undefined; customer scope drifts |
| Promo effectiveness | Incremental lift vs baseline, discount cost captured | No baseline; discount leakage not measured |
| Customer segmentation | RFM / cohort / behavioral segments usable in BI + reverse-ETL | One-off SQL, not modeled; not synced to Klaviyo |
| Inventory weeks-on-hand & projected availability | On-hand ÷ run-rate, projected stockout dates | No inventory feed; manual spreadsheet |
| Demand forecasting | Forecast by SKU/category with accuracy tracking | None, or a black-box Sheet |
| Merchandising performance | Sell-through, returns, margin by product/category | Returns not joined; category mapping inconsistent |
| Product sentiment | Composite score from returns + reviews + CS tickets | Sources not unified; no score model |
| Conversion funnel | Session → add-to-cart → checkout → purchase, by channel | GA4 not modeled; funnel steps inconsistent |
| A/B test results | Test readout with significance, tied to revenue | Manual analysis; no framework |

### Output for the report

Produce a **Business Applications scorecard**: one row per capability with status
(✅/🟡/🔴/➕), a one-line current state, and a recommended next step. Then:

- **Missing (🔴)** → roadmap items, scored High Impact with realistic Effort
- **Partial (🟡)** → tie to the specific architecture finding blocking it (e.g. "Contribution
  margin is partial because COGS is a manual Sheet — see check 4.3.1")
- **Inconsistent** → cross-reference the 1.5c single-source-of-truth findings
- **Improvements (➕)** → quick upgrades that increase decision speed

This connects the technical audit to business value and naturally seeds the next
engagement: building the missing capabilities.
