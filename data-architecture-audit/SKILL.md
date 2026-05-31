---
name: data-architecture-audit
description: >
  Conduct a full data architecture audit for a retail or e-commerce brand and produce
  a structured audit report with scored findings and prioritized action items.
  Use this skill whenever a user wants to audit, assess, review, or evaluate a client's
  data stack — including dbt projects, ETL pipelines, BI tools, warehouse configuration,
  or overall data platform maturity. Also triggers for: "data stack review",
  "data infrastructure assessment", "data platform audit", "dbt audit", "Looker audit",
  "data architecture review", "onboard new client data stack". Always use this skill
  before starting any data audit work — even if the user hasn't said "audit" explicitly.
---

# Data Architecture Audit Skill

**Author**: Blyze Labs — Fractional Data Analytics  
**Version**: 1.0  
**Target clients**: Retail & e-commerce brands (DTC, omnichannel, marketplace)

---

## Overview

This skill guides a complete data architecture audit from discovery through deliverable.
The output is a structured audit report — modeled on the Merit Beauty Data Architecture
Audit — with an executive summary, stack overview, scored findings per domain, and a
prioritized action item roadmap.

**The audit has two phases:**

1. **Discovery** — Understand the client's stack, tools, and access before touching repos
2. **Deep Audit** — Scan each layer (ingestion → transformation → serving) and score findings

Read `references/audit-phases.md` for the full phase-by-phase workflow.  
Read `references/scoring-rubric.md` for how to score Impact and Effort.  
Read `references/domain-checks.md` for the detailed, **mechanically verifiable** checklist per domain.  
Read `references/macro-candidates-example.md` for how to analyze and present repeated-code findings.  
Read `references/output-template.md` for the report structure and writing guide.

---

## Quick Start

### Step 1 — Stack Discovery

Before scanning anything, map the client's stack by asking for or reviewing:

| Layer | What to identify |
|---|---|
| **Warehouse** | Snowflake / BigQuery / Redshift / Databricks / other |
| **Transformation** | dbt Core / dbt Cloud / custom SQL / none |
| **Ingestion** | Fivetran / Stitch / Airbyte / custom / mixed |
| **BI** | Looker / Omni / Tableau / Metabase / Power BI / other |
| **Orchestration** | dbt Cloud jobs / Airflow / Prefect / none |
| **Repos** | GitHub / GitLab / Bitbucket — one repo or many |
| **Data sources** | Commerce, marketing, email/SMS, analytics, retail, CRM |

**If a layer is missing entirely — flag it as a gap in the audit.** For example:
- No dbt → transformation is ad-hoc SQL; high-risk finding
- No BI tool → no governed reporting layer; flag for roadmap
- No ingestion tool → manual pipelines or unknown; flag for discovery

### Step 2 — Request Access

Standard access needed (read-only everywhere):

```
□ dbt project repo (GitHub / GitLab)
□ dbt Cloud account (job history, environments, CI config)
□ Warehouse (Snowflake / BigQuery) — read-only query access
□ BI tool (Looker / Omni / Tableau) — viewer or developer access
□ ETL tool (Fivetran / Stitch) — connector list and status
□ Orchestration tool if separate from dbt Cloud
```

If access is partial, note what was unavailable and caveat findings in the report.

### Step 3 — Run the Audit

**Always start by verifying the project builds** (see the PRE-AUDIT section in
`references/domain-checks.md`): does it `compile`, `build`, and `test` cleanly, and what
dbt version is it on? A project that doesn't compile makes every downstream finding
suspect — document build/test failures before going further.

Then work through each domain in `references/domain-checks.md`:

1. Project Structure & Naming
2. Data Validation & Observability
3. Materialization & Performance
4. Data Ingestion
5. BI Layer Health
6. AI Readiness
7. Project Management & Process

For each domain: scan → identify issues → score each finding → document action items.

### Step 4 — Produce the Report

Follow the template in `references/output-template.md`. The report sections are:

1. Executive Summary
2. Stack & Environment overview
3. Data Sources inventory
4. Repository inventory
5. Domain findings (one section per domain)
6. Action Items Scorecard (full table: topic, impact, effort, summary)
7. Return on Investment narrative
8. Appendix

---

## Key Principles

**Be specific, not generic.** Every finding must reference actual file names, model counts,
config values, or query patterns found in the client's repo. Avoid vague statements like
"tests are missing" — write "Of 298 models, only 14 have any YAML declarations."

**Score every action item.** Every finding gets an Impact and Effort score (High/Mid/Low).
This drives prioritization for the client. See `references/scoring-rubric.md`.

**Flag missing layers.** If a client has no dbt, no BI tool, or no ingestion tool,
that IS a finding. Document it and recommend what to add.

**AI Readiness is always a section.** Every audit ends with a roadmap for how the client
can move toward conversational analytics (MCP + Claude + semantic layer). This is a
strategic differentiator for Blyze Labs.

**Tone**: Direct, practical, expert. Written for a technical founder or head of data.
Not academic. Every section should answer: "what is broken and what do we do about it."
