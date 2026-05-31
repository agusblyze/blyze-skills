# Scoring Rubric

Every finding in the audit gets two scores: **Impact** and **Effort**.
Both use a three-point scale: **High / Mid / Low**.

---

## Impact — How much does fixing this matter?

| Score | Meaning | Examples |
|---|---|---|
| **High** | Directly affects data accuracy, stakeholder trust, financial reporting, developer velocity, or strategic capabilities | Missing revenue integrity tests, no CI pipeline, no PK tests on output marts, BI connected to raw tables, no source freshness on core sources |
| **Mid** | Meaningful improvement but not immediately breaking anything | Wrong naming conventions, no macro for repeated logic, outdated dbt version, missing documentation on output marts |
| **Low** | Nice to have, quality-of-life, or hygiene | README missing, seeds slightly over-organized, no pre-commit hook, stale dashboards |

**When in doubt, ask**: "If we never fix this, what's the worst case in 12 months?"
- Data errors that cost the business money or erode investor confidence → High
- Slower development or frustrated engineers → Mid
- Mild technical debt → Low

---

## Effort — How hard is this to fix?

| Score | Meaning | Examples |
|---|---|---|
| **High** | Requires significant refactoring, migration, or coordination across systems | Renaming all 161 staging models, migrating GA4 from BigQuery to native connector, incrementalizing 20+ models |
| **Mid** | A few hours to a few days of focused work | Adding CI job, writing revenue integrity test, refactoring a complex model |
| **Low** | < 1-2 hours; mostly mechanical or config-level | Fixing materialization typo, adding README, creating source.yml files, adding freshness thresholds |

---

## Priority Matrix

Use this to guide the client on sequencing:

```
         LOW EFFORT          HIGH EFFORT
HIGH  ┌─────────────────┬─────────────────┐
      │  DO FIRST       │  PLAN CAREFULLY │
      │  (quick wins)   │  (big projects) │
      ├─────────────────┼─────────────────┤
MID   │  DO SOON        │  DEFER OR BATCH │
      │                 │                 │
      ├─────────────────┼─────────────────┤
LOW   │  DO WHEN IDLE   │  PROBABLY SKIP  │
      │                 │                 │
      └─────────────────┴─────────────────┘
```

**Quick wins** (High Impact + Low Effort) always lead the recommendations.
These are the findings that build trust with the client fastest.

---

## Framing Impact in Business Terms

When writing the report, translate technical impact into business language:

| Technical Finding | Business Impact Framing |
|---|---|
| No revenue integrity tests | "Reporting errors could erode investor or board confidence" |
| No CI pipeline | "Any PR can introduce silent bugs to production data" |
| All models full-refresh | "Compute costs grow linearly with data volume; no cost ceiling" |
| No dbt documentation | "New team members take weeks to onboard; AI tools can't query the data" |
| BI connected to raw tables | "Dashboard numbers bypass all business logic and can't be trusted" |
| No freshness monitors | "Stakeholders have no way to know if data is stale" |
| Missing marts for key domains | "Business teams can't self-serve; every question goes through data team" |
