# Output Template & Report Writing Guide

The audit report follows the structure established in the Merit Beauty Data Architecture
Audit. Every client report uses this structure — sections may vary in depth but should
all be present.

---

## Report Structure

```
1. Cover / Header
2. Executive Summary
3. Project Overview
   3a. Stack & Environment
   3b. Data Sources
   3c. Repository Inventory
   3d. dbt Project Structure (if dbt is present)
4. Data Architecture Roadmap (summary table)
5. Domain Findings (one section per domain)
   5a. Project Structure & Naming Convention
   5b. Data Validation & Observability
   5c. Materialization & Performance Strategy
   5d. Data Ingestion Process
   5e. BI Layer Health
   5f. AI Readiness
   5g. Project Management & Process
6. Return on Investment
7. Appendix (link to action items spreadsheet)
```

---

## Section Writing Guides

### 1. Cover / Header

```
[Client Logo] + [Blyze Labs Logo]
[Client Name] Data Architecture Audit for AI Era
Scalable data engineering practices for [Client Name]
---
Confidential / Prepared for [Client Name] by Blyze Labs / [Month Year]
Author: Agustin Velazquez — agus@blyzelabs.co — blyzelabs.co
```

---

### 2. Executive Summary

**Length**: 3–5 short paragraphs  
**Tone**: Direct, honest, not alarmist. Lead with what's working, then what needs to change.

**Structure**:
- Para 1: Overall assessment — is the infrastructure solid, fragile, or critical?
- Para 2: The 2–3 most important findings and why they matter to the business
- Para 3: The strategic opportunity — what becomes possible when this is fixed
- Para 4: The 7 strategic objectives (bullet list) that organize the roadmap

**Strategic objectives to frame the audit** (adapt per client):
- AI Enablement
- Faster Time-to-Production
- DTC/Core Reporting Foundation
- Financial Data Integrity
- Data Collection & IP
- M&A Readiness (if relevant)
- Technical Debt Reduction

---

### 3a. Stack & Environment

Short narrative describing the client's modern data stack.
Name each tool and its role. Include a diagram description if possible.

Example:
> [Client] operates a modern data stack centered on Snowflake as the warehouse,
> dbt Cloud for transformation, Looker as the governed BI layer, and Fivetran
> for data ingestion.

---

### 3b. Data Sources

Table listing all source systems by category:

| Category | Sources |
|---|---|
| Commerce | Shopify, Global-E |
| Marketing | Facebook Ads, Google Ads, TikTok, etc. |
| Email/SMS | Klaviyo, Attentive |
| Analytics | GA4 |
| Retail | Sephora, Kohl's |
| CRM/Support | Zendesk, Yotpo |
| Manual | Google Sheets (count them) |

Note total source count. Flag if Google Sheets count is high (>10 is a risk signal).

---

### 3c. Repository Inventory

Table of repos with description and a brief one-line assessment:

| Repository | Description | Assessment |
|---|---|---|
| [client]_dbt | dbt transformation project | Well-organized by source; no source() refs yet |
| [client]_looker | LookML BI layer | Best-documented layer; customer models exemplary |
| [client]_pipelines | Prefect/Airflow orchestration | Adequate; some complex models |

If a layer has NO repo (e.g., no version-controlled BI), note it as a gap.

---

### 3d. dbt Project Structure

Table of layers, model counts, and purpose:

| Layer | Model Count | Purpose |
|---|---|---|
| staging/ | N | Raw-adjacent cleaning, type casting, renaming |
| intermediate/ | N | Cross-source joins, shared business logic |
| marts/ | N | Stakeholder-facing: Looker, syncs, dashboards |

If the client uses non-standard layer names (base/facts/output), document what they have
and what they should migrate to.

---

### 4. Data Architecture Roadmap (Summary Table)

A table giving a one-line summary of each domain finding. This is the "map" before the detail.

| Domain | Summary |
|---|---|
| Project Structure & Naming | [1-2 sentence summary of state and key issue] |
| Data Validation & Observability | [1-2 sentence summary] |
| Materialization & Performance | [1-2 sentence summary] |
| Data Ingestion | [1-2 sentence summary] |
| BI Layer Health | [1-2 sentence summary] |
| AI Readiness | [1-2 sentence summary] |
| Project Management & Process | [1-2 sentence summary] |

---

### 5. Domain Findings

Each domain section follows this repeating pattern:

```
## [Domain Name]
---
### Topic: [Short Name]
**Impact**: High | Mid | Low  |  **Effort**: High | Mid | Low

**Issue**: [What is wrong. Be specific — use file names, model counts, config values.
           1-3 sentences. Answer: what is the problem and why does it matter?]

**Actions**:
1. [Concrete step]
2. [Concrete step]
3. [Concrete step, etc.]
```

**Writing rules for findings:**
- Every issue statement must be specific. No vague statements.
  - ❌ "Tests are missing"
  - ✅ "Of 298 models, only 14 have any YAML declarations. 95% of models have no tests."
- Action items must be actionable by an engineer. Not "improve tests" but "Add unique
  and not_null tests to the primary key column of every output mart."
- If providing a code example (e.g., folder structure, config snippet), use a code block.
- List topics in order of Impact (High first).

---

### 6. Return on Investment

**Length**: 5–8 bullet points  
**Tone**: Business value, not technical. Written for a founder or CFO.

Frame each major workstream in terms of what it unlocks:

- **Financial Data Integrity**: [what risk does it eliminate]
- **Developer Velocity**: [what does it speed up or prevent]
- **Business Applications**: [what decisions become possible]
- **AI Readiness**: [what self-serve capability does it unlock]
- **Margin Protection**: [what financial visibility does it create]
- **Faster Decision Making**: [what does it replace/speed up for business teams]

---

### 7. Appendix

Link to the companion Google Sheet with the full action items list, organized by:
- Domain
- Topic
- Impact
- Effort
- Status (for implementation tracking)

---

## Tone and Style Guide

| Do | Don't |
|---|---|
| Use specific numbers ("298 models", "14 YAML files") | Say "many models" or "some tests" |
| Name actual files when relevant | Be vague about where problems live |
| Write for a technical founder or head of data | Write for a general business audience |
| Be direct about severity | Soften findings to the point of ambiguity |
| Explain why each issue matters to the business | List issues without business context |
| Use short paragraphs and tables | Write dense prose paragraphs |
| End every section with actionable next steps | Leave findings open-ended |

**Confidentiality footer** (every page):
> Confidential / Prepared for [Client Name] by Blyze Labs / [Date]
