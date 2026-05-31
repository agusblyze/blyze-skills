# Data Architecture Audit

A Claude skill that runs a complete data-stack audit for a retail or e-commerce brand and
produces a structured, scored audit report with a prioritized action-item roadmap.

Modeled on the Blyze Labs methodology first delivered in the Merit Beauty Data
Architecture Audit.

**Author:** Blyze Labs · **Version:** 1.0 · **Target clients:** Retail & e-commerce (DTC, omnichannel, marketplace)

---

## What it does

Given read-only access to a client's stack, the skill walks Claude through a two-phase
audit — **discovery** then **deep audit** — across seven domains plus a business-applications
gap analysis. Every finding is scored by Impact and Effort, and the output is a report in
the Blyze Labs format: executive summary, stack overview, domain findings, action-item
scorecard, and an ROI narrative.

It covers any combination of:
- **dbt** (Core or Cloud) — structure, naming, tests, docs, materialization, performance
- **ETL / ingestion** (Fivetran, Stitch, Airbyte, custom) — consolidation, health, manual sources
- **BI** (Looker, Omni, Tableau, Power BI) — version control, connection integrity, coverage
- **Warehouse** (Snowflake, BigQuery, etc.) — schema hygiene, orphaned tables, cost visibility
- **AI readiness** — metadata, semantic layer, MCP connectivity, skills & pre-built queries
- **Process** — ticketing, CI, Slack/Linear/GitHub integration

If a layer is missing (e.g. no dbt, no governed BI), that itself becomes a finding.

---

## When it triggers

Claude loads this skill whenever a task involves auditing, assessing, reviewing, or
evaluating a client's data stack — including phrases like "data stack review", "dbt audit",
"Looker audit", "data platform assessment", or "onboard a new client's data stack."

---

## Folder structure

```
data-architecture-audit/
├── SKILL.md                          ← entrypoint: quick-start workflow + principles
├── references/
│   ├── audit-phases.md               ← Phase 1–5 workflow: discovery → delivery
│   ├── domain-checks.md              ← the core checklist — ~180 verifiable checks across 7 domains
│   ├── scoring-rubric.md             ← Impact/Effort scoring + priority matrix + business framing
│   ├── macro-candidates-example.md   ← how to find & present repeated-code findings
│   └── output-template.md            ← report structure + section-by-section writing guide
└── assets/
    └── (add your gold-standard example audit .docx here)
```

### How Claude reads it
1. **`SKILL.md`** loads first — the workflow and operating principles.
2. **`references/*`** load on demand as Claude reaches each stage. `domain-checks.md` is the
   workhorse: every item has a "how to verify" column and a severity score.
3. **`assets/`** holds the output template and example deliverable Claude mirrors.

---

## The audit at a glance

**Phases** (see `references/audit-phases.md`)
1. Discovery — map the stack and pain points before touching repos
2. Access setup — read-only access + first-pass inventory
3. Deep audit — work through the domain checklist
4. Report writing — synthesize findings into the deliverable
5. Delivery — review call + linked action-item sheet

**Domains** (see `references/domain-checks.md`)
0. Pre-audit build gate (compile / build / test / dbt version)
1. Project Structure & Naming (incl. outbound/reverse-ETL, query simplification, inconsistent-logic reconciliation)
2. Data Validation & Observability (tests, docs, freshness, CI, revenue integrity)
3. Materialization & Performance (incremental candidates, heavy models, cost)
4. Data Ingestion (tooling, consolidation, manual sources)
5. BI Layer Health (version control, connection integrity, coverage)
6. AI Readiness (metadata → semantic layer → MCP → skills)
7. Project Management & Process
- Plus a **Business Applications gap analysis** tying architecture to decisions.

**Scoring** (see `references/scoring-rubric.md`)
Every finding gets Impact (High/Mid/Low) and Effort (High/Mid/Low). Quick wins
(High Impact + Low Effort) always lead the recommendations.

---

## Using it

Open a session with this skill available and start with discovery:

```
"I'm starting a data architecture audit for [client]. They run [Snowflake + dbt Cloud +
 Looker]. Here's read-only access to the dbt repo and warehouse — let's begin."
```

Claude will verify the project builds, work the domain checklist gathering specific
evidence (file names, model counts, config values), score each finding, and draft the
report in the Blyze Labs structure.

> **Tip:** Drop the anonymized Merit Beauty audit (or your latest best audit) into
> `assets/`. It's the strongest single input — Claude mirrors its structure, tone, and
> level of specificity.

---

## Customizing per engagement

- **Different BI tool?** The BI checks cover Looker/Omni/Tableau/Power BI generically;
  add tool-specific checks to `references/domain-checks.md` (Domain 5) if needed.
- **Different freshness SLAs?** Edit the threshold table in `domain-checks.md` (check 1.3.4).
- **Different report sections?** Adjust `references/output-template.md`.

---

## Changelog

- **v1.0** — Initial release. Seven domains + pre-audit build gate, business-applications
  gap analysis, macro-candidate and inconsistent-logic analyses. Based on the Merit Beauty
  audit methodology.

---

*© Blyze Labs. Confidential methodology.*
