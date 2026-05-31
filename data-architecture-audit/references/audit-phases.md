# Audit Phases — Detailed Workflow

## Phase 1: Discovery (Before Repo Access)

**Goal**: Understand the client's stack at a high level before doing any deep scanning.
This phase often happens in a kick-off call or intake questionnaire.

### Discovery Questions

**Stack inventory:**
- What tools are in your data stack? (warehouse, ETL, transformation, BI, orchestration)
- How long has the current stack been in place?
- How many people are on the data team? What are their roles?
- Is there a dbt project? Is it in version control?

**Pain points (these become audit priorities):**
- What data questions can't you answer today?
- What takes too long or breaks too often?
- Are there data trust issues with stakeholders?
- Any upcoming events that depend on data? (fundraising, M&A, major campaigns)

**Business context:**
- DTC only, or omnichannel (retail, wholesale, marketplace)?
- Key commerce platform (Shopify, Salesforce Commerce, custom)?
- Which marketing channels are active?
- Any international operations? Multi-currency?

### Red Flags to Listen For in Discovery

| Signal | Likely Finding |
|---|---|
| "We have a lot of Google Sheets" | Seeds overuse, manual ingestion risk |
| "Numbers don't always match" | Missing tests, materialization bugs, duplicate logic |
| "It takes a few days to get a report" | No self-serve BI, output marts missing |
| "We're planning to raise / sell" | M&A readiness = high priority framing |
| "We just hired a data person" | No README, no contributing guide, no onboarding |
| "We use BigQuery for GA4" | Multi-warehouse complexity, consolidation opportunity |
| "We're not sure what's in Snowflake" | Orphaned tables, schema sprawl |

---

## Phase 2: Repo & Tool Access Setup

**Goal**: Get read-only access to all layers and do a first-pass inventory.

### dbt Repo — First Pass Checklist

```
□ Clone repo locally or review via GitHub UI
□ Count total models by layer (staging/intermediate/marts or base/facts/output)
□ Check dbt_project.yml — version, materialization defaults, schema config
□ Check packages.yml — what packages are installed
□ Check for README.md and CONTRIBUTING.md
□ Check for .sqlfluff or .pre-commit-config.yaml
□ Count YAML files (.yml) vs SQL files — coverage ratio
□ Check for source() declarations in staging layer
□ Check for CI job config (.github/workflows/ or dbt Cloud CI setup)
□ Note any obvious naming inconsistencies in file names
```

### Warehouse — First Pass Checklist

```
□ List all schemas and databases
□ Identify which schemas map to which dbt layers
□ Check for orphaned schemas not referenced by dbt
□ Query information_schema for table last_altered dates (find stale tables)
□ Check for multiple schemas for the same source (GA4 sprawl, Sheets sprawl)
□ Review warehouse size and compute setup
□ Check for cost monitoring / billing alerts
```

### BI Tool — First Pass Checklist (Looker / Omni / Tableau)

```
□ How many dashboards / explores / workbooks exist?
□ Are they organized by team/domain or flat list?
□ How many were accessed in last 30/90 days? (active vs stale)
□ Is the BI layer connected to marts or directly to raw/staging?
□ Are there LookML model files in a repo (Looker)? Is it version controlled?
□ Are there broken / erroring dashboards?
□ Who maintains it — data team or embedded analysts?
```

### ETL / Ingestion — First Pass Checklist

```
□ What tool(s) are used? (Fivetran, Stitch, Airbyte, custom)
□ List all active connectors and their destination schemas
□ Check for duplicate connectors for the same source
□ Check sync frequency per connector
□ Any connectors in error state?
□ Any manual ingestion (Google Sheets, CSV uploads, scripts)?
□ Is ingestion documented anywhere?
```

---

## Phase 3: Deep Audit by Domain

Work through `references/domain-checks.md` domain by domain.
For each finding, capture:

```
Topic: [short name]
Impact: High | Mid | Low
Effort: High | Mid | Low
Issue: [what is wrong and why it matters — be specific with numbers/files]
Actions: [numbered list of concrete steps to fix it]
```

---

## Phase 4: Report Writing

Follow `references/output-template.md` exactly.

**Writing order** (easier than document order):
1. Write all domain findings first — this is where the detail lives
2. Build the Action Items Scorecard table from your findings
3. Write the Stack & Environment and Data Sources sections
4. Write Executive Summary last — it synthesizes everything above
5. Write the ROI section — frame in business terms, not technical terms

---

## Phase 5: Delivery

- Export as a Google Doc or Word document
- Include the Action Items as a linked Google Sheet (sortable by Impact/Effort)
- Walk through findings in a 60-minute review call
- Offer implementation support as a follow-on engagement
