# Macro Candidate Analysis — Output Format

When auditing for repetitive code (check 1.5), produce a table like the one below.
This is the format that made the Merit audit actionable: for every candidate macro,
name it, list the exact files that should use it, count the call sites, and give the
proposed logic so an engineer can implement it immediately.

## How to find candidates

```bash
# Find a repeated filter pattern (example: valid-order filter)
grep -rn "is_cancelled = false" models/ | wc -l
grep -rln "source_name.*not in" models/

# Find repeated timezone conversions
grep -rln "convert_timezone" models/

# Find repeated cents->dollars or /100 math
grep -rln "/ 100" models/
grep -rln "/100" models/

# Find repeated email cleaning
grep -rln "lower(trim(" models/

# Count call sites for any pattern
grep -rln "<pattern>" models/ | sort | uniq
```

## Output table format

| Macro | Effort / Criticality | Files / Call sites | Proposed logic |
|---|---|---|---|
| `valid_order_filter(alias)` | Mid / **High** | 20 files confirmed using the filter inline (list each) | `coalesce({{ alias }}.source_name,'') not in ('Grin','Draft') and {{ alias }}.is_cancelled = false` |
| `pst_timestamp(col)` / `pst_date(col)` | Low / Mid | 10 files, ~18 call sites (list each) | `convert_timezone('UTC','America/Los_Angeles',{{ col }}::timestamp)` (+ `::date` variant) |
| `customer_scope(alias)` | Low / Mid | 3 files today; all future LTV/repeat/AOV/cohort models | `{{ valid_order_filter(alias) }} and {{ alias }}.country in ('US','CA','UK') and {{ alias }}.created_at >= '2021-01-01' and {{ alias }}.total_sales > 0` |
| `strip_taxes_if_inclusive(price, tax, included)` | Low / Low | tax-inclusive market revenue models | `iff({{ included }} = true, {{ price }} - {{ tax }}, {{ price }})` |
| `clean_email(col)` / `is_internal_email(col)` | Low / Low | customer/order/loyalty models | `lower(trim({{ col }}))` ; internal: `... ilike '%@<companydomain>%'` |
| `{channel}_hierarchy` seed | Low / Mid | channel/retail target marts | Seed columns: sub_channel, channel, region, is_retail, display_order. Replaces inline CASE WHEN. |
| `join_fiscal_calendar(date_col)` | Low / Low | models joining the date spine directly | `left join {{ ref('date_spine') }} cal on {{ date_col }} = cal.day` — **blocked** until any source/prod divergence in the spine is fixed |

## Rules for the analysis

- **Count call sites precisely.** "20 files" is credible; "many models" is not.
- **List the actual file paths** under each candidate so the refactor is mechanical.
- **Flag blockers.** If a candidate depends on fixing something else first (e.g. a broken
  shared model), say so and recommend deferring — don't macro-ize broken logic.
- **Don't over-macro.** If a pattern appears in only 1–2 places, note it as "adoption
  appropriate — no new macro." Recommend deleting dead macros (0 call sites).
- **Tie criticality to data risk.** A filter that defines what counts as valid revenue
  is High criticality; a formatting helper is Low.
