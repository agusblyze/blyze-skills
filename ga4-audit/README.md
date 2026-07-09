# GA4 Data Audit — Methodology & Checklist

> Reusable audit playbook for GA4 export landed in Snowflake, reconciled against Shopify.
> Table names and timezone are parametrized — change the `SET` values, never the query bodies.

**Default sources**
- GA4: `RAW.BIGQUERY.GA4__EVENTS` (Fivetran/BigQuery export)
- Shopify orders: `RAW.SHOPIFY.ORDERS` · customers: `RAW.SHOPIFY.CUSTOMERS` (Fivetran)
- Daily grain: parametrized store timezone (default `America/New_York`)
- Every check runs against a one-time flattened staging table (`ga4_flattened_table`), not the raw GA4 table directly

---

## Table of contents

- [How to use this document](#how-to-use-this-document)
- [Prerequisites — Snowflake CLI](#prerequisites--snowflake-cli)
- [Parameters](#parameters)
- [Building the flattened staging table](#building-the-flattened-staging-table)
- [Schema conventions](#schema-conventions)
- [Run order](#run-order)
- [Domain 0 — Foundations](#domain-0--foundations)
- [Domain 1 — Volume & completeness](#domain-1--volume--completeness)
- [Domain 2 — Identity & sessions](#domain-2--identity--sessions)
- [Domain 3 — Ecommerce funnel & item-level](#domain-3--ecommerce-funnel--item-level)
- [Domain 4 — Purchase reconciliation vs Shopify](#domain-4--purchase-reconciliation-vs-shopify)
- [Domain 5 — Traffic quality & filtering](#domain-5--traffic-quality--filtering)
- [Domain 6 — Attribution & channels](#domain-6--attribution--channels)
- [Domain 7 — Consent, privacy & PII](#domain-7--consent-privacy--pii)
- [Deliverable structure](#deliverable-structure)
- [Behavioral rules](#behavioral-rules)

---

## How to use this document

Each check carries three fixed fields:

| Field | Description |
|---|---|
| **Method** | Direct, step-by-step instructions for running the check in Snowflake — which query to run, what to read, and the relevant schema specifics. Each check includes a runnable query. |
| **Why it matters** | The business/trust reason the check exists and what a failure implies. |
| **Pass threshold** | A default tolerance. Tune per client and document the tuned value in the engagement. |

*Checks are grouped into domains and severity-tagged. Severity drives the order of the executive summary, not the order of execution.*

### Severity tags

| Tag | Meaning |
|---|---|
| `[BLOCKER]` | Invalidates downstream analytics or reconciliation if unaddressed. Fix before anything else. |
| `[MATERIAL]` | Meaningfully distorts a metric the client reports on or spends against. |
| `[HYGIENE]` | Quality/maintainability issue; low immediate impact but compounds over time. |

---

## Prerequisites — Snowflake CLI

This methodology requires the Snowflake CLI (`snow`) to run queries directly against the warehouse.

Check availability first:
```bash
which snow
```

If not installed, install it via Homebrew:
```bash
brew tap snowflakedb/snowflake-cli
brew trust snowflakedb/snowflake-cli
brew install snowflake-cli
```

Run queries with `snow sql -q "<query>" -c <connection_name>`.

**Fallback (if `snow` cannot be installed — no Homebrew, no permissions):** use `dbt show --inline "<query>" --limit -1 --quiet` against the project's configured dbt Snowflake connection instead. Two things to know about this fallback:
- Snowflake session variables set with `SET` do **not** persist across separate `dbt show` invocations — each call opens a fresh session. Prepend the full `SET ga4_table = ...; SET shopify_table = ...; ...` block to *every* `dbt show --inline` call, not just the first one.
- Always pass `--limit -1`. dbt's default result-limiting wraps the query in an outer `SELECT ... LIMIT N`, which breaks on multi-statement SQL containing `SET` statements.

**Session scoping matters.** Each CLI invocation is typically its own fresh Snowflake session, so the flattened staging table (below) must be a regular, persistent table — not a session-scoped temporary one — or it disappears before the next check can read it. Build it once with `CREATE OR REPLACE TABLE` and keep it in place across the whole audit (and future re-runs); never drop it at the end.

---

## Parameters

Set these once per engagement. Every query references them via `IDENTIFIER()` and `$store_tz` — never hardcode table names or timezones in query bodies.

```sql
SET ga4_table               = 'RAW.BIGQUERY.GA4__EVENTS';
SET shopify_table           = 'RAW.SHOPIFY.ORDERS';
SET shopify_customers_table = 'RAW.SHOPIFY.CUSTOMERS';
SET shopify_refunds_table   = 'RAW.SHOPIFY.ORDER_REFUNDS'; -- optional
SET store_tz                = 'America/New_York';
SET ga4_flattened_table     = '<same database as ga4_table>.SCRATCH.GA4_AUDIT_FLATTENED';
```

Confirm CREATE TABLE privileges on the flattened table's target schema before starting — if unavailable, use the no-staging-table fallback pattern described under [Schema conventions](#schema-conventions).

---

## Building the flattened staging table

**Why:** every domain check needs one or more `EVENT_PARAMS` values (`ga_session_id`, `page_location`, `page_referrer`, `traffic_type`, `currency`, `source`, `medium`). Flattening once up front, into a persistent table, avoids re-flattening `EVENT_PARAMS` per query — a large performance win at volume.

**First, resize the warehouse** to speed up build and audit:
```sql
ALTER WAREHOUSE ANALYTICS SET WAREHOUSE_SIZE = 'MEDIUM';
```

**Second, confirm the actual schema** with `DESCRIBE TABLE IDENTIFIER($ga4_table)`, then pick the matching template:

- **Template A** — typed top-level columns (`EVENT_TIMESTAMP`, `EVENT_NAME`, `EVENT_PARAMS`, `DEVICE`, `GEO`, `ECOMMERCE`, `TRAFFIC_SOURCE`, `COLLECTED_TRAFFIC_SOURCE`, `PRIVACY_INFO`, `_ID`, etc.).
- **Template B** — a single `RAW` VARIANT blob per row (`RAW:"event_timestamp"`, `RAW:"event_name"`, etc.).

Both templates must produce the same output column contract, so every domain query below works unchanged regardless of which was used: `event_date_local`, `ga_session_id`, `page_location`, `page_referrer`, `traffic_type`, `ga_currency`, `utm_source`, `utm_medium`, `session_key`, `geo_country`, `geo_region`, `ecommerce_transaction_id`, `ecommerce_purchase_revenue`, `ecommerce_items`, `traffic_source_source`, `traffic_source_medium`, `collected_manual_source`, `collected_manual_medium`, `shopify_customer_id_user_prop`, plus the original `EVENT_NAME`, `EVENT_TIMESTAMP`, `USER_PSEUDO_ID`, `USER_ID`, `__UPDATED_AT`, `STREAM_ID`, `PLATFORM`, `EVENT_DATE`, `EVENT_BUNDLE_SEQUENCE_ID`.

### Template A — typed top-level columns

```sql
CREATE OR REPLACE TABLE IDENTIFIER($ga4_flattened_table) AS
WITH base AS (
    SELECT *
    FROM IDENTIFIER($ga4_table)
    WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
),
-- one flatten pass, scoped to only the param keys every check actually needs
params AS (
    SELECT
        b._ID,
        MAX(CASE WHEN p.value:key::string = 'ga_session_id' THEN p.value:value:int_value::number    END) AS ga_session_id,
        MAX(CASE WHEN p.value:key::string = 'page_location' THEN p.value:value:string_value::string  END) AS page_location,
        MAX(CASE WHEN p.value:key::string = 'page_referrer' THEN p.value:value:string_value::string  END) AS page_referrer,
        MAX(CASE WHEN p.value:key::string = 'traffic_type'  THEN p.value:value:string_value::string  END) AS traffic_type,
        MAX(CASE WHEN p.value:key::string = 'currency'      THEN p.value:value:string_value::string  END) AS ga_currency,
        MAX(CASE WHEN p.value:key::string = 'source'        THEN p.value:value:string_value::string  END) AS utm_source,
        MAX(CASE WHEN p.value:key::string = 'medium'        THEN p.value:value:string_value::string  END) AS utm_medium
    FROM base b,
    LATERAL FLATTEN(input => b.EVENT_PARAMS) p
    WHERE p.value:key::string IN ('ga_session_id','page_location','page_referrer','traffic_type','currency','source','medium')
    GROUP BY 1
),
-- separate flatten pass over USER_PROPERTIES looking for a Shopify customer_id
-- pushed as a user property under any of these common key names
user_props AS (
    SELECT
        b._ID,
        MAX(CASE WHEN LOWER(up.value:key::string) IN ('customer_id','shopify_customer_id','customerid','shopify_customerid')
            THEN COALESCE(up.value:value:string_value::string, up.value:value:int_value::string) END) AS shopify_customer_id_user_prop
    FROM base b,
    LATERAL FLATTEN(input => b.USER_PROPERTIES) up
    WHERE LOWER(up.value:key::string) IN ('customer_id','shopify_customer_id','customerid','shopify_customerid')
    GROUP BY 1
)
SELECT
    b.*,
    TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz, TO_TIMESTAMP_NTZ(b.EVENT_TIMESTAMP,6))) AS event_date_local,
    pr.ga_session_id, pr.page_location, pr.page_referrer, pr.traffic_type, pr.ga_currency,
    pr.utm_source, pr.utm_medium,
    b.USER_PSEUDO_ID || ':' || pr.ga_session_id             AS session_key,
    b.PRIVACY_INFO:analytics_storage::string                AS privacy_analytics_storage,
    b.PRIVACY_INFO:ads_storage::string                       AS privacy_ads_storage,
    b.GEO:country::string                                    AS geo_country,
    b.GEO:region::string                                     AS geo_region,
    b.ECOMMERCE:transaction_id::string                       AS ecommerce_transaction_id,
    b.ECOMMERCE:purchase_revenue::float                      AS ecommerce_purchase_revenue,
    b.ECOMMERCE:items                                        AS ecommerce_items,
    b.TRAFFIC_SOURCE:source::string                          AS traffic_source_source,
    b.TRAFFIC_SOURCE:medium::string                          AS traffic_source_medium,
    b.COLLECTED_TRAFFIC_SOURCE:manual_source::string         AS collected_manual_source,
    b.COLLECTED_TRAFFIC_SOURCE:manual_medium::string         AS collected_manual_medium,
    up.shopify_customer_id_user_prop
FROM base b
LEFT JOIN params pr ON pr._ID = b._ID
LEFT JOIN user_props up ON up._ID = b._ID;
```

### Template B — single RAW VARIANT blob per row

Uses `ARRAYS_TO_OBJECT` + `TRANSFORM` to flatten the entire `event_params` array into one keyed object in a single pass (no `LATERAL FLATTEN` join needed). These are newer Snowflake array functions — if the account/edition doesn't support them (`Unknown function` error), fall back to Template A's `LATERAL FLATTEN` + pivot approach adapted to `RAW:"..."` paths.

```sql
CREATE OR REPLACE TABLE IDENTIFIER($ga4_flattened_table) AS
WITH final_events AS (
    SELECT *
    FROM IDENTIFIER($ga4_table)
    -- keep only if an INGESTION_COMPLETE column exists; drop this filter otherwise
    WHERE INGESTION_COMPLETE = TRUE
    AND DATE(TO_TIMESTAMP_NTZ(RAW:"event_timestamp"::int,6)) >= DATEADD('months', -24, CURRENT_DATE())
),
snowflake_events AS (
    SELECT
        RAW:"event_timestamp"::int                              AS EVENT_TIMESTAMP,
        RAW:"event_date"::string                                AS EVENT_DATE,
        RAW:"event_name"::string                                AS EVENT_NAME,
        RAW:"event_params"                                      AS EVENT_PARAMS,
        RAW:"event_bundle_sequence_id"::int                     AS EVENT_BUNDLE_SEQUENCE_ID,
        RAW:"user_id"::string                                   AS USER_ID,
        RAW:"user_pseudo_id"::string                            AS USER_PSEUDO_ID,
        RAW:"stream_id"::string                                 AS STREAM_ID,
        RAW:"platform"::string                                  AS PLATFORM,
        RAW:"__updated_at"::timestamp_ntz                       AS __UPDATED_AT,
        RAW:"privacy_info"."analytics_storage"::string          AS privacy_analytics_storage,
        RAW:"privacy_info"."ads_storage"::string                AS privacy_ads_storage,
        RAW:"geo"."country"::string                             AS geo_country,
        RAW:"geo"."region"::string                               AS geo_region,
        RAW:"ecommerce"."transaction_id"::string                AS ecommerce_transaction_id,
        RAW:"ecommerce"."purchase_revenue"::float                AS ecommerce_purchase_revenue,
        RAW:"items"                                              AS ecommerce_items,
        RAW:"traffic_source"."source"::string                    AS traffic_source_source,
        RAW:"traffic_source"."medium"::string                    AS traffic_source_medium,
        RAW:"collected_traffic_source"."manual_source"::string   AS collected_manual_source,
        RAW:"collected_traffic_source"."manual_medium"::string   AS collected_manual_medium,
        ARRAYS_TO_OBJECT(
            TRANSFORM(RAW:"event_params", entry -> entry:key::string),
            TRANSFORM(RAW:"event_params", entry ->
                CASE
                    WHEN NOT IS_NULL_VALUE(entry:value.double_value) THEN entry:value.double_value
                    WHEN NOT IS_NULL_VALUE(entry:value.float_value)  THEN entry:value.float_value
                    WHEN NOT IS_NULL_VALUE(entry:value.int_value)    THEN entry:value.int_value
                    WHEN NOT IS_NULL_VALUE(entry:value.string_value) THEN entry:value.string_value
                    ELSE NULL
                END
            )
        ) AS event_params__flattened,
        ARRAYS_TO_OBJECT(
            TRANSFORM(RAW:"user_properties", entry -> LOWER(entry:key::string)),
            TRANSFORM(RAW:"user_properties", entry ->
                CASE
                    WHEN NOT IS_NULL_VALUE(entry:value.int_value)    THEN entry:value.int_value
                    WHEN NOT IS_NULL_VALUE(entry:value.string_value) THEN entry:value.string_value
                    ELSE NULL
                END
            )
        ) AS user_properties__flattened
    FROM final_events
)
SELECT
    *,
    TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz, TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS event_date_local,
    event_params__flattened:ga_session_id::number AS ga_session_id,
    event_params__flattened:page_location::string AS page_location,
    event_params__flattened:page_referrer::string AS page_referrer,
    event_params__flattened:traffic_type::string  AS traffic_type,
    event_params__flattened:currency::string      AS ga_currency,
    event_params__flattened:source::string        AS utm_source,
    event_params__flattened:medium::string        AS utm_medium,
    USER_PSEUDO_ID || ':' || event_params__flattened:ga_session_id::string AS session_key,
    COALESCE(
        user_properties__flattened:customer_id::string,
        user_properties__flattened:shopify_customer_id::string,
        user_properties__flattened:customerid::string,
        user_properties__flattened:shopify_customerid::string
    ) AS shopify_customer_id_user_prop
FROM snowflake_events;
```

**This is additive, not deduplicating** — both templates preserve the exact row count and grain of `IDENTIFIER($ga4_table)` (scoped to the 24-month window). Check 0.4 (dedup key uniqueness) still runs correctly against this table; it is not pre-deduped by the flatten step.

**Verify before proceeding:**
```sql
SELECT COUNT(*) AS flattened_rows FROM IDENTIFIER($ga4_flattened_table);
SELECT COUNT(*) AS raw_rows FROM IDENTIFIER($ga4_table) WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE());
```
These two counts must match exactly. If they don't, the `LEFT JOIN` fan-out somewhere (most likely a `_ID` collision, Template A) or the `WHERE` scoping differs (Template B) — stop and investigate before running any domain checks, since every check downstream inherits this table.

**If `CREATE TABLE` privileges aren't available on any writable schema**, fall back to the per-query `LATERAL FLATTEN` pattern in [Schema conventions](#schema-conventions) — every check still runs correctly, just slower.

---

## Schema conventions

- **Timezone** — `EVENT_TIMESTAMP` is microseconds UTC. Store-local day is precomputed as `event_date_local`; use it instead of re-deriving `TO_DATE(CONVERT_TIMEZONE(...))` per query. Never use the raw `EVENT_DATE` string for anything except the 0.1 boundary-shift comparison, which needs it explicitly.
- **Event params** — `EVENT_PARAMS` is an array of `{key, value:{string_value, int_value, double_value, float_value}}`. After the flattening step, read the precomputed columns (`ga_session_id`, `page_location`, etc.) directly instead of re-flattening per query. If a check needs a param not in the precomputed set, add it to the flatten step's `params` CTE / `event_params__flattened` object, or use the fallback pattern below for a one-off.
- **User properties** — `USER_PROPERTIES` is a *different* array from `EVENT_PARAMS`, same `{key, value:{...}}` shape, representing user-scoped properties set once via `gtag('set','user_properties',...)` and echoed on every subsequent event. GA4 itself writes a `prevenue_28d` (predicted 28-day revenue) property automatically — don't mistake that for a custom property. A Shopify `customer_id` is sometimes pushed here under a key like `customer_id` or `shopify_customer_id` as an identity signal alternative to `USER_ID` (precomputed as `shopify_customer_id_user_prop`; see check 2.4). Shopify customer IDs are large plain integers (e.g. `8940668747962`, 10-15 digits) — never a UUID, hash, or email; validate any candidate with `RLIKE '^[0-9]{9,15}$'` before trusting it.
- **Items** — live in `ECOMMERCE:items` / `ecommerce_items` in the flattened table (no top-level `ITEMS` column). Transaction id: `ecommerce_transaction_id`. Revenue: `ecommerce_purchase_revenue`. Some GA4 implementations never populate `items` at all — if checks 3.2/3.3/3.4 all return 100% empty, treat it as a genuine site-side tagging gap, not a query bug; confirm by inspecting a raw `ECOMMERCE` payload before concluding that.
- **Freshness** — `__UPDATED_AT` is the connector load/reprocess timestamp, used for late-arrival checks.
- **Shopify columns** (Fivetran standard) — `ID, CREATED_AT, TOTAL_PRICE, CURRENCY, PRESENTMENT_CURRENCY, FINANCIAL_STATUS, SOURCE_NAME, TEST, CANCELLED_AT`. `CREATED_AT` is `TIMESTAMP_TZ` (already zone-aware) — convert with the 2-argument form `CONVERT_TIMEZONE($store_tz, CREATED_AT)`, not `TO_TIMESTAMP(CREATED_AT)` first (that cast is for GA4's UTC epoch column and errors or silently mis-converts on an already-zoned value). Verify with `DESCRIBE TABLE` before assuming.
- **Currency** — `CURRENCY` is the shop's base currency; `PRESENTMENT_CURRENCY` is what the customer actually saw at checkout. GA4's `currency` param reflects the presented currency, so always compare against `PRESENTMENT_CURRENCY` (check 4.6) — comparing against `CURRENCY` produces false-positive mismatches on every international order.
- **24-month window** — the flattened table is already scoped to the last 24 months; no need to repeat the date filter downstream. If running against the raw table directly (fallback path), keep the filter.
- **BFCM** — the rolling-median ±3·MAD anomaly band will legitimately flag the Thanksgiving-through-Cyber-Monday window as `BFCM_EXPECTED`. Treat it as seasonality, not a data anomaly.

### Snowflake-specific gotchas

- **No correlated subqueries against `EVENT_PARAMS`.** This environment rejects `(SELECT ... FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) ...)` inside a `SELECT` list (`002031: Unsupported subquery type cannot be evaluated`). If working against the raw table (no staging table available), extract params via an explicit `LATERAL FLATTEN` **join** in the `FROM` clause instead:

  ```sql
  -- WRONG — correlated subquery, will fail:
  SELECT e.USER_PSEUDO_ID,
      (SELECT p.value:value:int_value::number FROM LATERAL FLATTEN(input => e.EVENT_PARAMS) p
        WHERE p.value:key::string = 'ga_session_id') AS ga_session_id
  FROM IDENTIFIER($ga4_table) e

  -- RIGHT — LATERAL JOIN in FROM, one flatten per distinct param key needed:
  SELECT e.USER_PSEUDO_ID, p.value:value:int_value::number AS ga_session_id
  FROM IDENTIFIER($ga4_table) e,
  LATERAL FLATTEN(input => e.EVENT_PARAMS) p
  WHERE p.value:key::string = 'ga_session_id'
  ```

  A `LATERAL FLATTEN` join is an inner join — an event with no matching key (e.g. `traffic_type` never implemented) disappears from the result instead of appearing as NULL. If a check's pass criterion needs "how many events have zero of this param," compute the complement explicitly rather than reading an empty result set as "0%, PASS."

- **No sliding-window `MEDIAN()`.** This environment rejects `MEDIAN(...) OVER (ORDER BY ... ROWS BETWEEN N PRECEDING AND M PRECEDING)` (`002303: Sliding window frame unsupported for function MEDIAN`). Every check needing a trailing 14-day rolling median/MAD (1.1, 1.8, 4.1) instead self-joins the daily aggregate to its own trailing 14-day window, then computes `roll_median` and `MAD` as plain aggregate `MEDIAN()` calls after a `GROUP BY` on the anchor day. See check 1.1 for the canonical rewrite; 1.8 and 4.1 reuse the same shape.

---

## Run order

Several checks are prerequisites for others. **Do not skip this sequence** or later reconciliations will produce phantom findings.

1. Build the flattened staging table *(one-time, before any domain check)*
2. Domain 0 — Foundations *(calibrates every count and every Shopify gap)*
3. Domain 1 — Volume & completeness
4. Domain 2 — Identity & sessions
5. Domain 3 — Ecommerce funnel & item-level
6. Domain 4 — Purchase reconciliation vs Shopify *(depends on Domain 0)*
7. Domain 5 — Traffic quality & filtering
8. Domain 6 — Attribution & channels
9. Domain 7 — Consent, privacy & PII

---

## Domain 0 — Foundations

> Run this domain first. These checks silently break every count and every Shopify reconciliation downstream if skipped.

---

### 0.1 Timezone alignment between GA4 and Shopify `[BLOCKER]`

**Method:** Compare the raw `EVENT_DATE` string to the precomputed `event_date_local`. Count events landing on a different calendar day under the two definitions.

**Why it matters:** GA4 and Shopify timestamps are stored in UTC but the two systems report on different calendar boundaries. A 3–5 hour offset reshuffles 5–15% of orders across the day boundary and manufactures a daily GA4-vs-Shopify gap that does not exist — every Domain 4 reconciliation is invalid until both sides are pinned to one zone.

**Pass threshold:** No systematic single-day-shift pattern in the gap series.

```sql
SELECT
    EVENT_DATE          AS raw_event_date_string,
    event_date_local     AS event_date_et,
    COUNT(*)             AS events,
    SUM(CASE WHEN EVENT_DATE <> TO_CHAR(event_date_local,'YYYYMMDD') THEN 1 ELSE 0 END) AS boundary_shifted_events
FROM IDENTIFIER($ga4_flattened_table)
GROUP BY 1,2
ORDER BY 2 DESC
LIMIT 200;
```

**A consistent, stable percentage across every day (e.g. ~7% every single day) is not a misalignment bug** — it's the expected artifact of a fixed UTC→local offset (evening events land on the next UTC calendar day). Only flag this as a genuine problem if the shift pattern is erratic or grows/shrinks over time.

---

### 0.2 Sync freshness and partial-day mechanics `[BLOCKER]`

**Method:** Per store-local event day, compute the lag between `MAX(EVENT_TIMESTAMP)` and `MAX(__UPDATED_AT)`.

**Why it matters:** The connector loads events on a schedule, so the most recent day is usually partially loaded at query time. Reading trailing days without accounting for sync lag produces phantom drops that look like tracking failure.

**Pass threshold:** No recurring drop on the latest 1–3 days.

```sql
SELECT
    event_date_local                                   AS event_date_et,
    MIN(__UPDATED_AT)                                  AS first_loaded_at,
    MAX(__UPDATED_AT)                                  AS last_loaded_at,
    DATEDIFF('hour', TO_TIMESTAMP_NTZ(MAX(EVENT_TIMESTAMP),6), MAX(__UPDATED_AT)) AS load_lag_hours,
    COUNT(*)                                           AS events
FROM IDENTIFIER($ga4_flattened_table)
GROUP BY 1
ORDER BY 1 DESC;
```

**A consistent drop on only the most recent days is sync latency, not tracking failure.** Flag those days in all subsequent trend queries.

---

### 0.3 Late-arriving and reprocessed events `[MATERIAL]`

**Method:** Per `EVENT_NAME`, count the share of events where `__UPDATED_AT` is more than 24h and more than 72h after the event occurred. Focus on `purchase`.

**Why it matters:** Events can arrive up to 72h late, so a snapshot taken too early undercounts recent days. Large late deltas on `purchase` specifically mean revenue dashboards built on fresh data understate recent performance.

**Pass threshold:** Late arrivals settle to under 1–2% change after 72h for web.

```sql
SELECT
    EVENT_NAME,
    COUNT(*)                                                          AS total_events,
    SUM(CASE WHEN DATEDIFF('hour', TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6), __UPDATED_AT) > 24 THEN 1 ELSE 0 END) AS arrived_gt_24h,
    SUM(CASE WHEN DATEDIFF('hour', TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6), __UPDATED_AT) > 72 THEN 1 ELSE 0 END) AS arrived_gt_72h,
    ROUND(100 * arrived_gt_72h / NULLIF(total_events,0), 2)          AS pct_gt_72h
FROM IDENTIFIER($ga4_flattened_table)
GROUP BY 1
ORDER BY total_events DESC;
```

**If `arrived_gt_24h` is unusually high (e.g. >50%) and near-uniform across every event type — not just `purchase`** — that pattern looks less like genuine per-row late arrival and more like periodic batch reprocessing of `__UPDATED_AT` by the ingestion connector. Note this as a caveat on the metric's interpretation rather than treating the raw percentage as pure tracking lag.

---

### 0.4 Canonical dedup keys defined `[BLOCKER]`

**Method:** Adopt these exact grains and reuse them in every dedup check:
- **Event identity** — `(USER_PSEUDO_ID, ga_session_id, EVENT_NAME, EVENT_TIMESTAMP, EVENT_BUNDLE_SEQUENCE_ID)`
- **Session identity** — `(USER_PSEUDO_ID, ga_session_id)`
- **Transaction identity** — `ecommerce_transaction_id`

**Why it matters:** The GA4 export has no row-level primary key, so dedup logic must be explicit and identical everywhere. Inconsistent keys cause two analysts to report different duplication rates on the same data.

**Pass threshold:** The event-identity grain is unique (duplicate-row rate near 0).

```sql
SELECT
    COUNT(*)                                                 AS rows_total,
    COUNT(*) - COUNT(DISTINCT COALESCE(USER_PSEUDO_ID,'') ||':'|| COALESCE(ga_session_id::string,'') ||':'
              || COALESCE(EVENT_NAME,'') ||':'|| COALESCE(EVENT_TIMESTAMP::string,'') ||':'|| COALESCE(EVENT_BUNDLE_SEQUENCE_ID::string,'')) AS duplicate_rows,
    ROUND(100 * duplicate_rows / NULLIF(rows_total,0), 3)    AS pct_duplicate_rows
FROM IDENTIFIER($ga4_flattened_table);
```

**The `COALESCE` on every key part is required, not optional.** `'a'||NULL||'b' = NULL`, so `COUNT(DISTINCT expr)` silently excludes every row whose key contains any NULL part from the distinct count, collapsing them into one uncounted bucket. In a live run, `ga_session_id` was NULL on ~1% of rows (events that never fire the param), and without `COALESCE` the apparent duplicate rate read materially higher than the genuine number — an artifact large enough to change the headline finding. Confirm with `SELECT COUNT(*), SUM(CASE WHEN ga_session_id IS NULL THEN 1 ELSE 0 END), SUM(CASE WHEN EVENT_BUNDLE_SEQUENCE_ID IS NULL THEN 1 ELSE 0 END) FROM IDENTIFIER($ga4_flattened_table)` — if those null counts are non-trivial, the `COALESCE`d query is the one to trust.

---

## Domain 1 — Volume & completeness

> Volume, completeness, naming, parameter coverage, and anomaly detection across all event collection.

---

### 1.1 Total event volume by day `[MATERIAL]`

**Method:** Count events per store-local day. Self-join each day to its trailing 14-day window to compute rolling median/MAD (sliding-window `MEDIAN()` is unsupported — see [Snowflake-specific gotchas](#snowflake-specific-gotchas)). Read the `is_bfcm` / `flag` columns — `BFCM_EXPECTED` rows are seasonal.

**Why it matters:** Daily event volume detects collection outages, tag deployments, and seasonality, and is the foundation for every other trend.

**Pass threshold:** No unexplained zero days; all non-BFCM days within the ±3·MAD band.

```sql
WITH daily AS (
    SELECT event_date_local AS d, COUNT(*) AS events
    FROM IDENTIFIER($ga4_flattened_table)
    GROUP BY 1
),
joined AS (
    SELECT d1.d AS d, d1.events AS events, d2.events AS past_events
    FROM daily d1 JOIN daily d2 ON d2.d BETWEEN DATEADD('day',-14,d1.d) AND DATEADD('day',-1,d1.d)
),
stats AS (
    SELECT d, MAX(events) AS events, MEDIAN(past_events) AS roll_median
    FROM joined GROUP BY d
),
mad_joined AS (
    SELECT s.d, s.events, s.roll_median, ABS(j.past_events - s.roll_median) AS abs_dev
    FROM stats s JOIN joined j ON j.d = s.d
),
mad_calc AS (
    SELECT d, MAX(events) AS events, MAX(roll_median) AS roll_median, MEDIAN(abs_dev) AS mad
    FROM mad_joined GROUP BY d
),
flagged AS (
    SELECT d, events, roll_median, mad,
           DATEADD('day', (3 - DAYOFWEEKISO(DATE_FROM_PARTS(YEAR(d),11,1)) + 7) % 7 + 21,
                   DATE_FROM_PARTS(YEAR(d),11,1))                       AS thanksgiving
    FROM mad_calc
)
SELECT d, events, roll_median, mad,
       (d BETWEEN thanksgiving AND DATEADD('day',4,thanksgiving))       AS is_bfcm,
       CASE WHEN d BETWEEN thanksgiving AND DATEADD('day',4,thanksgiving) THEN 'BFCM_EXPECTED'
            WHEN mad > 0 AND ABS(events - roll_median) > 3*mad          THEN 'ANOMALY'
            ELSE 'ok' END                                               AS flag
FROM flagged
ORDER BY d;
```

**Missing days matter as much as the MAD band.** A day absent from the `daily` CTE entirely (not low-count — genuinely missing) never appears in this result set and never trips the `ANOMALY` flag, silently understating a real tracking gap. Always cross-check against a full calendar-day spine before declaring "no unexplained zero days":

```sql
WITH daily AS (... as above ...),
spine AS (
    SELECT DATEADD('day', seq4(), (SELECT MIN(d) FROM daily)) AS d
    FROM TABLE(GENERATOR(ROWCOUNT => 800))
    WHERE d <= (SELECT MAX(d) FROM daily)
)
SELECT spine.d AS missing_day
FROM spine LEFT JOIN daily ON spine.d = daily.d
WHERE daily.d IS NULL
ORDER BY 1;
```

If missing days form a long contiguous run, treat it as a BLOCKER-level tracking outage regardless of the check's default MATERIAL tier — check whether it overlaps BFCM or another critical revenue window before finalizing severity.

**Method-noise caveat:** this naive 14-day trailing MAD band doesn't adjust for weekday/weekend seasonality. On a typical ecommerce site it's common to see 20-25% of days flagged `ANOMALY` purely from that noise. Cross-reference against the missing-day spine check (the real signal) before treating the raw anomaly-day count as material on its own.

---

### 1.2 Volume split by stream and platform `[MATERIAL]`

**Method:** Group counts by `STREAM_ID` and `PLATFORM`. Trend each stream independently.

**Why it matters:** A drop in one stream (web vs app vs server-side) nets out at the total and stays hidden. A server-side stream going dark while web continues is a classic GTM server-container failure that only a per-stream view surfaces.

**Pass threshold:** Each active stream is individually continuous with no silent single-stream dropout.

```sql
SELECT event_date_local AS d,
       STREAM_ID, PLATFORM, COUNT(*) AS events
FROM IDENTIFIER($ga4_flattened_table)
GROUP BY 1,2,3
ORDER BY 1 DESC, events DESC;
```

---

### 1.3 page_view completeness and single-fire `[MATERIAL]`

**Method:** (a) Trend `page_view` per day. (b) Dedup on `(USER_PSEUDO_ID, ga_session_id, page_location)` within 2 seconds and quantify the double-fire rate.

**Why it matters:** Double-fires inflate sessions and pageviews and dilute every conversion rate. A consistent ~2x pageview-to-session ratio versus benchmark signals systematic double-tagging (GTM plus hardcoded gtag both present, or an SPA firing a manual and an automatic `page_view`).

**Pass threshold:** Duplicate `page_view` rate under 1%.

```sql
-- 1.3a: daily page_view trend
SELECT event_date_local AS d, COUNT(*) AS page_views
FROM IDENTIFIER($ga4_flattened_table)
WHERE EVENT_NAME = 'page_view'
GROUP BY 1 ORDER BY 1;

-- 1.3b: double-fire detection
WITH pv AS (
    SELECT USER_PSEUDO_ID, ga_session_id, page_location, EVENT_TIMESTAMP
    FROM IDENTIFIER($ga4_flattened_table)
    WHERE EVENT_NAME='page_view'
),
flagged AS (
    SELECT *,
        EVENT_TIMESTAMP - LAG(EVENT_TIMESTAMP) OVER (
            PARTITION BY USER_PSEUDO_ID, ga_session_id, page_location
            ORDER BY EVENT_TIMESTAMP) AS micros_since_prev
    FROM pv
)
SELECT
    COUNT(*)                                                        AS page_views,
    SUM(CASE WHEN micros_since_prev < 2000000 THEN 1 ELSE 0 END)   AS likely_double_fires,
    ROUND(100 * likely_double_fires / NULLIF(COUNT(*),0), 2)        AS pct_double_fire
FROM flagged;
```

---

### 1.4 session_start single-fire and session integrity `[MATERIAL]`

**Method:** Count `session_start` per `(USER_PSEUDO_ID, ga_session_id)` — must be exactly one.

**Why it matters:** Fragmented or duplicated sessions corrupt every per-session metric and channel attribution. A new session minted at the payment or subscription subdomain shows up as a self-referral and misattributes the conversion.

**Pass threshold:** Zero sessions with `session_start_count > 1`.

```sql
WITH ss AS (
    SELECT USER_PSEUDO_ID, ga_session_id, COUNT(*) AS session_start_count
    FROM IDENTIFIER($ga4_flattened_table)
    WHERE EVENT_NAME='session_start'
    GROUP BY 1,2
)
SELECT
    COUNT(*)                                                  AS sessions_with_start,
    SUM(CASE WHEN session_start_count > 1 THEN 1 ELSE 0 END)  AS sessions_multi_start,
    ROUND(100 * sessions_multi_start / NULLIF(COUNT(*),0),3)  AS pct_multi_start,
    MAX(session_start_count)                                  AS worst_case
FROM ss;
```

---

### 1.5 Event naming, reserved collisions, and cardinality `[HYGIENE]`

**Method:** (a) Flag names not matching lowercase snake_case or exceeding 40 chars, and classify reserved vs custom. (b) Count distinct event names vs the 500 cap. (c) Confirm error/failure events exist.

**Why it matters:** Bad names and reserved-name collisions corrupt both the GA4 UI and the export, and high-cardinality event sets hit the 500-distinct-event cap after which new events are silently discarded. Missing error events mean failure states are invisible.

**Pass threshold:** Naming adherence >95%; distinct events well under 500; error events present.

```sql
-- 1.5a: naming and classification
SELECT
    EVENT_NAME,
    COUNT(*) AS events,
    CASE WHEN EVENT_NAME = LOWER(EVENT_NAME)
              AND EVENT_NAME NOT LIKE '% %'
              AND LENGTH(EVENT_NAME) <= 40 THEN 'ok' ELSE 'NAMING_ISSUE' END AS naming_flag,
    CASE WHEN EVENT_NAME IN ('first_open','first_visit','session_start','page_view',
        'user_engagement','scroll','click','view_search_results','purchase','refund',
        'add_to_cart','begin_checkout','view_item','view_item_list','select_item',
        'add_payment_info','add_shipping_info')
        THEN 'standard/reserved' ELSE 'custom' END AS event_class
FROM IDENTIFIER($ga4_flattened_table)
GROUP BY 1
ORDER BY events DESC;

-- 1.5b: cardinality vs 500 cap
SELECT COUNT(DISTINCT EVENT_NAME) AS distinct_events,
       CASE WHEN COUNT(DISTINCT EVENT_NAME) > 450 THEN 'NEAR_CAP' ELSE 'ok' END AS flag
FROM IDENTIFIER($ga4_flattened_table);

-- 1.5c: error/failure event presence
SELECT EVENT_NAME, COUNT(*) AS events
FROM IDENTIFIER($ga4_flattened_table)
WHERE (EVENT_NAME ILIKE '%error%' OR EVENT_NAME ILIKE '%fail%'
    OR EVENT_NAME IN ('form_error','payment_failed'))
GROUP BY 1 ORDER BY events DESC;
```

---

### 1.6 Parameter coverage and null rates `[HYGIENE]`

**Method:** Per key event, compute populated rate of `page_location` and `ga_session_id`.

**Why it matters:** An event can fire while the parameters the client actually reports on are null, leaving the event present but useless. A high null rate on a reported parameter is invisible in event counts yet quietly breaks the dashboard built on it.

**Pass threshold:** Critical params populated >98% on the events that use them.

```sql
SELECT
    EVENT_NAME,
    COUNT(*) AS events,
    ROUND(100*AVG(CASE WHEN page_location IS NOT NULL THEN 1 ELSE 0 END),1) AS pct_has_page_location,
    ROUND(100*AVG(CASE WHEN ga_session_id IS NOT NULL THEN 1 ELSE 0 END),1) AS pct_has_session_id
FROM IDENTIFIER($ga4_flattened_table)
WHERE EVENT_NAME IN ('page_view','add_to_cart','begin_checkout','purchase','view_item')
GROUP BY 1
ORDER BY events DESC;
```

---

### 1.7 Events firing per page `[HYGIENE]`

**Method:** (a) Group event counts by stripped page path and event name. (b) Quantify pages that fire `view_item` but never `add_to_cart` — the actual pass-threshold check.

**Why it matters:** This surfaces templates that are over- or under-instrumented. A PDP template with no `view_item` points to a tag scoped to the wrong page pattern.

**Pass threshold:** No high-traffic template silently missing `add_to_cart` or `begin_checkout`.

```sql
-- 1.7a: raw per-page/event breakdown
SELECT
    REGEXP_REPLACE(page_location, '\\?.*$','') AS page_path,
    EVENT_NAME,
    COUNT(*)                                          AS events
FROM IDENTIFIER($ga4_flattened_table)
WHERE EVENT_NAME IN ('page_view','view_item','add_to_cart','begin_checkout','purchase')
GROUP BY 1,2
ORDER BY events DESC
LIMIT 200;

-- 1.7b: pages with view_item but zero add_to_cart
WITH per_page AS (
    SELECT REGEXP_REPLACE(page_location, '\\?.*$','') AS page_path,
        SUM(CASE WHEN EVENT_NAME='view_item' THEN 1 ELSE 0 END) AS view_item_events,
        SUM(CASE WHEN EVENT_NAME='add_to_cart' THEN 1 ELSE 0 END) AS add_to_cart_events
    FROM IDENTIFIER($ga4_flattened_table)
    WHERE EVENT_NAME IN ('view_item','add_to_cart')
    GROUP BY 1
)
SELECT
    COUNT(*) AS pages_with_view_item,
    SUM(CASE WHEN view_item_events > 0 AND add_to_cart_events = 0 THEN 1 ELSE 0 END) AS pages_missing_add_to_cart,
    SUM(CASE WHEN view_item_events > 100 AND add_to_cart_events = 0 THEN 1 ELSE 0 END) AS high_traffic_pages_missing_atc
FROM per_page
WHERE view_item_events > 0;
```

**A nonzero `pages_missing_add_to_cart` is expected at the long tail** (low-traffic URL variants, UTM-tagged duplicates). Weight severity by `high_traffic_pages_missing_atc` — a handful of pages with >100 `view_item` events and zero `add_to_cart` is a real mid-funnel tracking gap worth a template-level drill-down, not just noise.

---

### 1.8 Anomaly detection on core metrics `[MATERIAL]`

**Method:** Compute daily transactions, users, sessions, page_views. Apply the same self-join rolling-median ±3·MAD pattern as 1.1 to transactions.

**Why it matters:** Systematic outlier detection across the metrics that drive decisions catches problems a single-metric view misses. Correlated anomalies across all four on the same non-BFCM date point to a collection or sync issue rather than real behavior.

**Pass threshold:** All four series within band on non-BFCM days.

```sql
WITH d AS (
    SELECT event_date_local AS dt,
           SUM(CASE WHEN EVENT_NAME='purchase'  THEN 1 ELSE 0 END) AS transactions,
           COUNT(DISTINCT USER_PSEUDO_ID)                          AS users,
           COUNT(DISTINCT session_key)                             AS sessions,
           SUM(CASE WHEN EVENT_NAME='page_view' THEN 1 ELSE 0 END) AS page_views
    FROM IDENTIFIER($ga4_flattened_table)
    GROUP BY 1
),
selfjoin AS (
    SELECT d1.dt, d1.transactions, d2.transactions AS past_txn
    FROM d d1 JOIN d d2 ON d2.dt BETWEEN DATEADD('day',-14,d1.dt) AND DATEADD('day',-1,d1.dt)
),
stats AS (
    SELECT dt, MEDIAN(past_txn) AS roll_median FROM selfjoin GROUP BY dt
),
mad_join AS (
    SELECT s.dt, s.roll_median, ABS(sj.past_txn - s.roll_median) AS dev
    FROM stats s JOIN selfjoin sj ON sj.dt = s.dt
),
mad AS (
    SELECT dt, MAX(roll_median) AS roll_median, MEDIAN(dev) AS mad FROM mad_join GROUP BY dt
),
final AS (
    SELECT d.dt, d.transactions, d.users, d.sessions, d.page_views, m.roll_median, m.mad,
        DATEADD('day', (3 - DAYOFWEEKISO(DATE_FROM_PARTS(YEAR(d.dt),11,1)) + 7) % 7 + 21,
                DATE_FROM_PARTS(YEAR(d.dt),11,1)) AS thanksgiving
    FROM d LEFT JOIN mad m ON d.dt = m.dt
)
SELECT dt, transactions, users, sessions, page_views,
       (dt BETWEEN thanksgiving AND DATEADD('day',4,thanksgiving)) AS is_bfcm,
       CASE WHEN dt BETWEEN thanksgiving AND DATEADD('day',4,thanksgiving) THEN 'BFCM_EXPECTED'
            WHEN mad > 0 AND ABS(transactions - roll_median) > 3*mad THEN 'TXN_ANOMALY'
            ELSE 'ok' END AS txn_flag
FROM final
ORDER BY dt;
```

---

## Domain 2 — Identity & sessions

> The linchpin for all downstream LTV, retention, and cohort work — central to the subscription-DTC use case.

---

### 2.1 user_id implementation post-login `[MATERIAL]`

**Method:** Trend the share of sessions carrying a non-null `USER_ID` per day.

**Why it matters:** Without a reliable `USER_ID`, cross-device stitching and logged-in LTV are impossible. A drop coincides with a login/auth or tagging change.

**Pass threshold:** `USER_ID` present on the large majority of post-login sessions, stable trend.

```sql
SELECT
    event_date_local                                                                              AS d,
    COUNT(DISTINCT session_key)                                                                    AS sessions,
    COUNT(DISTINCT CASE WHEN USER_ID IS NOT NULL THEN session_key END)                             AS sessions_with_user_id,
    ROUND(100*sessions_with_user_id/NULLIF(sessions,0),1)                                           AS pct_sessions_with_user_id
FROM IDENTIFIER($ga4_flattened_table)
GROUP BY 1 ORDER BY 1;
```

**Before trusting a 0% result:** confirm directly with `SELECT COUNT(*), COUNT(USER_ID), COUNT(DISTINCT USER_ID) FROM IDENTIFIER($ga4_flattened_table)` — a genuine 0% means `USER_ID` was never implemented site-wide (a single `gtag('set','user_id',...)` fix unblocks this check and 2.2/2.3 simultaneously), not a query artifact.

---

### 2.2 user_id ↔ Shopify customer_id join coverage `[BLOCKER for LTV]`

**Method:** Join distinct GA4 `USER_ID` to Shopify customer IDs. Confirm the join contract (raw id vs hashed email) before trusting the rate.

**Why it matters:** Without a reliable GA4-to-Shopify join, behavioral data cannot be tied to revenue, retention, or churn cohorts.

**Pass threshold:** Match rate sufficient for cohort analysis (set per client, document the threshold).

```sql
WITH ga AS (
    SELECT DISTINCT USER_ID FROM IDENTIFIER($ga4_flattened_table)
    WHERE USER_ID IS NOT NULL
),
shop AS (
    SELECT DISTINCT TO_VARCHAR(ID) AS customer_id FROM IDENTIFIER($shopify_customers_table)
)
SELECT
    (SELECT COUNT(*) FROM ga)                                              AS ga_distinct_user_ids,
    (SELECT COUNT(*) FROM shop)                                            AS shopify_customers,
    COUNT(*)                                                               AS matched,
    ROUND(100*COUNT(*)/NULLIF((SELECT COUNT(*) FROM shop),0),1)            AS pct_shopify_matched_to_ga
FROM ga JOIN shop ON ga.USER_ID = shop.customer_id;
```

---

### 2.3 user_pseudo_id stability and inflation `[MATERIAL]`

**Method:** Count distinct `USER_PSEUDO_ID` per logged-in `USER_ID`. Report average, median, and worst case.

**Why it matters:** Cookie and consent resets mint new pseudo-ids, inflating user counts and fragmenting journeys.

**Pass threshold:** Pseudo-id inflation within expected range for the consent/browser mix.

```sql
WITH m AS (
    SELECT USER_ID, COUNT(DISTINCT USER_PSEUDO_ID) AS pseudo_ids
    FROM IDENTIFIER($ga4_flattened_table)
    WHERE USER_ID IS NOT NULL
    GROUP BY 1
)
SELECT
    COUNT(*)                                       AS logged_in_users,
    ROUND(AVG(pseudo_ids),2)                       AS avg_pseudo_ids_per_user,
    MEDIAN(pseudo_ids)                             AS median_pseudo_ids_per_user,
    MAX(pseudo_ids)                                AS worst_case,
    SUM(CASE WHEN pseudo_ids > 3 THEN 1 ELSE 0 END) AS users_gt_3_pseudo_ids
FROM m;
```

**If 2.1 returned 0% (`USER_ID` never populated), this check has nothing to measure** (`logged_in_users = 0`). Report it as N/A / blocked-upstream rather than PASS or FAIL.

---

### 2.4 Sessions with an identifiable Shopify customer ID `[BLOCKER for LTV]`

**Method:** `USER_ID` alone (check 2.1) may miss identity signals the site pushes through other channels. Check three possible locations for a Shopify customer ID: (1) `USER_ID` directly, (2) `USER_PROPERTIES` under a key like `customer_id` or `shopify_customer_id` (precomputed as `shopify_customer_id_user_prop`), (3) `USER_PSEUDO_ID` itself, in case a misconfiguration writes the customer ID there instead of a proper GA client ID. Validate every candidate against the real Shopify customer ID shape — a large plain integer like `8940668747962` (10-15 digits) — before trusting it.

**Why it matters:** A single failed check (2.1's `USER_ID`) can mask a working identity path through `USER_PROPERTIES` or a misconfiguration — this check is the exhaustive version before concluding no identity path exists at all.

**Pass threshold:** A meaningful share of sessions (especially post-login and post-purchase) carry an identifiable, validated Shopify customer ID from at least one of the three sources.

```sql
WITH candidates AS (
    SELECT
        session_key,
        CASE WHEN USER_ID RLIKE '^[0-9]{9,15}$' THEN USER_ID END                                       AS from_user_id,
        CASE WHEN shopify_customer_id_user_prop RLIKE '^[0-9]{9,15}$' THEN shopify_customer_id_user_prop END AS from_user_properties,
        CASE WHEN USER_PSEUDO_ID RLIKE '^[0-9]{9,15}$' THEN USER_PSEUDO_ID END                          AS from_pseudo_id
    FROM IDENTIFIER($ga4_flattened_table)
),
scored AS (
    SELECT
        session_key,
        COALESCE(from_user_id, from_user_properties, from_pseudo_id) AS identified_shopify_customer_id,
        from_user_id, from_user_properties, from_pseudo_id
    FROM candidates
)
SELECT
    COUNT(DISTINCT session_key)                                                                     AS total_sessions,
    COUNT(DISTINCT CASE WHEN identified_shopify_customer_id IS NOT NULL THEN session_key END)        AS sessions_with_customer_id,
    ROUND(100*sessions_with_customer_id/NULLIF(total_sessions,0),2)                                   AS pct_sessions_identified,
    COUNT(DISTINCT CASE WHEN from_user_id IS NOT NULL THEN session_key END)                           AS sessions_via_user_id,
    COUNT(DISTINCT CASE WHEN from_user_properties IS NOT NULL THEN session_key END)                   AS sessions_via_user_properties,
    COUNT(DISTINCT CASE WHEN from_pseudo_id IS NOT NULL THEN session_key END)                          AS sessions_via_pseudo_id_misconfig
FROM scored;
```

**Before trusting a non-zero `pct_sessions_identified`, validate the candidates are real customers, not coincidental numbers:**

```sql
WITH candidates AS (
    SELECT DISTINCT COALESCE(
        CASE WHEN USER_ID RLIKE '^[0-9]{9,15}$' THEN USER_ID END,
        CASE WHEN shopify_customer_id_user_prop RLIKE '^[0-9]{9,15}$' THEN shopify_customer_id_user_prop END,
        CASE WHEN USER_PSEUDO_ID RLIKE '^[0-9]{9,15}$' THEN USER_PSEUDO_ID END
    ) AS candidate_id
    FROM IDENTIFIER($ga4_flattened_table)
    WHERE candidate_id IS NOT NULL
)
SELECT
    COUNT(*)                                                          AS distinct_candidates,
    COUNT(shop.customer_id)                                           AS matched_real_shopify_customers,
    ROUND(100*matched_real_shopify_customers/NULLIF(COUNT(*),0),1)    AS pct_validated
FROM candidates c
LEFT JOIN (SELECT DISTINCT TO_VARCHAR(ID) AS customer_id FROM IDENTIFIER($shopify_customers_table)) shop
    ON c.candidate_id = shop.customer_id;
```

**Interpret:** If `pct_sessions_identified` is 0% (or `pct_validated` is far below 100%, meaning most candidates are coincidental numeric strings), the finding is the same root cause as 2.1/2.2 — no reliable path exists from a GA4 session to a Shopify customer through *any* of the three channels checked. Report `sessions_via_user_properties` and `sessions_via_pseudo_id_misconfig` separately even when both are 0 — a future re-run after a `USER_PROPERTIES`-based fix should show movement specifically in that column. **Do not assume `USER_PROPERTIES` uses exactly these four key spellings** — if this check returns 0% and 2.1/2.2 also show 0%, run `SELECT LOWER(p.value:key::string) AS k, COUNT(*) FROM IDENTIFIER($ga4_table), LATERAL FLATTEN(input=>USER_PROPERTIES) p GROUP BY 1 ORDER BY 2 DESC` once to confirm no customer-id-shaped key exists under a different name before concluding the signal is genuinely absent.

---

## Domain 3 — Ecommerce funnel & item-level

> Ecommerce funnel event presence, healthy step ratios, and item-level data completeness.

---

### 3.1 Funnel event presence and healthy ratios `[MATERIAL]`

**Method:** Count each funnel event and express each as a % of `view_item`. Confirm every step exists and ratios are plausible.

**Why it matters:** Missing or malformed funnel events break conversion-rate analysis end to end. A near-zero `begin_checkout` alongside healthy `add_to_cart` points to a checkout-domain tagging gap, common on Shopify checkout.

**Pass threshold:** All steps present, ratios plausible, no step at effectively zero.

```sql
WITH f AS (
    SELECT EVENT_NAME, COUNT(*) AS events
    FROM IDENTIFIER($ga4_flattened_table)
    WHERE EVENT_NAME IN ('view_item_list','view_item','add_to_cart','begin_checkout',
                         'add_shipping_info','add_payment_info','purchase')
    GROUP BY 1
)
SELECT EVENT_NAME, events,
       ROUND(100*events / NULLIF(MAX(CASE WHEN EVENT_NAME='view_item' THEN events END) OVER (),0),1) AS pct_of_view_item
FROM f
ORDER BY events DESC;
```

**A step missing from the result set entirely (not a low count — genuinely absent) means that funnel event never fires.** Cross-check `SELECT COUNT(*) FROM IDENTIFIER($ga4_flattened_table) WHERE EVENT_NAME = '<missing_step>'` before concluding it's a total gap — confirm there's no near-miss naming variant capturing the same intent under a custom/vendor event name.

---

### 3.2 Item-detail completeness across ecommerce events `[MATERIAL]`

**Method:** Flatten `ecommerce_items` on key events. Compute populated rate of `item_id`, `item_name`, `item_category`, `item_brand`, `price`, `quantity`.

**Why it matters:** Sparse item arrays break product, category, and merchandising analysis. `item_id` present but `item_category` null means product-level analysis works while category roll-ups silently fail.

**Pass threshold:** Core item fields populated >98% on every ecommerce event.

```sql
WITH items AS (
    SELECT e.EVENT_NAME, i.value AS item
    FROM IDENTIFIER($ga4_flattened_table) e,
         LATERAL FLATTEN(input => e.ecommerce_items) i
    WHERE e.EVENT_NAME IN ('view_item','add_to_cart','begin_checkout','purchase')
)
SELECT EVENT_NAME,
    COUNT(*)                                                                          AS item_rows,
    ROUND(100*AVG(CASE WHEN item:item_id::string      IS NOT NULL THEN 1 ELSE 0 END),1) AS pct_item_id,
    ROUND(100*AVG(CASE WHEN item:item_name::string    IS NOT NULL THEN 1 ELSE 0 END),1) AS pct_item_name,
    ROUND(100*AVG(CASE WHEN item:item_category::string IS NOT NULL THEN 1 ELSE 0 END),1) AS pct_item_category,
    ROUND(100*AVG(CASE WHEN item:item_brand::string   IS NOT NULL THEN 1 ELSE 0 END),1) AS pct_item_brand,
    ROUND(100*AVG(CASE WHEN item:price                IS NOT NULL THEN 1 ELSE 0 END),1) AS pct_price,
    ROUND(100*AVG(CASE WHEN item:quantity             IS NOT NULL THEN 1 ELSE 0 END),1) AS pct_quantity
FROM items
GROUP BY 1
ORDER BY item_rows DESC;
```

**If this returns zero rows for every event type**, do not conclude "0% coverage, FAIL" from an empty result — an empty result here means the `LATERAL FLATTEN` join found nothing to flatten (`ecommerce_items` is null/empty on every row), which is exactly what check 3.3 tests explicitly. Run 3.3 to confirm before writing up 3.2.

---

### 3.3 Purchase events contain item-level data `[MATERIAL]`

**Method:** Count purchases where `ecommerce_items` is null or empty, as a share of all purchases.

**Why it matters:** A purchase with an empty items array breaks revenue-by-product and AOV-by-item. Item-less purchases usually mean the purchase tag fires before the dataLayer ecommerce object is populated.

**Pass threshold:** Effectively 0% of purchases with an empty items array.

```sql
SELECT
    COUNT(*)                                                            AS purchases,
    SUM(CASE WHEN ecommerce_items IS NULL
              OR ARRAY_SIZE(ecommerce_items) = 0 THEN 1 ELSE 0 END)   AS purchases_no_items,
    ROUND(100*purchases_no_items/NULLIF(COUNT(*),0),2)                 AS pct_no_items
FROM IDENTIFIER($ga4_flattened_table)
WHERE EVENT_NAME='purchase';
```

**If `pct_no_items` is 100%, verify it's a genuine site-side tagging gap before writing it up:** inspect one raw payload with `SELECT ECOMMERCE FROM IDENTIFIER($ga4_flattened_table) WHERE EVENT_NAME='purchase' LIMIT 1` and confirm the `items` key is absent from the JSON entirely (not present-but-null). Also check `add_to_cart` the same way — if `items` is absent there too, this is a GTM/gtag implementation gap affecting the whole ecommerce data layer, not a purchase-specific bug. If confirmed, 3.2 and 3.4 are moot — report them as N/A rather than running redundant queries.

---

### 3.4 Item value reconciliation `[HYGIENE]`

**Method:** Per purchase, compare `SUM(item price × quantity)` to `ecommerce_purchase_revenue`. Quantify mismatches beyond $0.01.

**Why it matters:** The sum of item revenue should reconcile to the event-level value. A systematic mismatch reveals inconsistent gross/net/discount handling in the tag.

**Pass threshold:** Item totals reconcile to event value within the documented tax/shipping rule.

```sql
WITH item_totals AS (
    SELECT e.ecommerce_transaction_id AS transaction_id,
           SUM(i.value:price::float * COALESCE(i.value:quantity::float,1)) AS item_value
    FROM IDENTIFIER($ga4_flattened_table) e,
    LATERAL FLATTEN(input => e.ecommerce_items) i
    WHERE e.EVENT_NAME='purchase'
    GROUP BY 1
),
p AS (
    SELECT
        e.ecommerce_transaction_id  AS transaction_id,
        e.ecommerce_purchase_revenue AS event_value,
        it.item_value
    FROM IDENTIFIER($ga4_flattened_table) e
    LEFT JOIN item_totals it ON it.transaction_id = e.ecommerce_transaction_id
    WHERE e.EVENT_NAME='purchase'
)
SELECT
    COUNT(*)                                                        AS purchases,
    SUM(CASE WHEN ABS(COALESCE(event_value,0)-COALESCE(item_value,0)) > 0.01
        THEN 1 ELSE 0 END)                                         AS mismatched,
    ROUND(100*mismatched/NULLIF(COUNT(*),0),2)                     AS pct_mismatched
FROM p;
```

---

## Domain 4 — Purchase reconciliation vs Shopify

> A full reconciliation of GA4 purchases against Shopify as the system of record.
> **Requires Domain 0 to be complete.** Do not run this domain if check 0.1 returned FAIL-BLOCKER.

---

### 4.1 Purchase event volume and anomalies `[MATERIAL]`

**Method:** Trend daily GA4 purchases with the same self-join rolling-median ±3·MAD band used in 1.1/1.8. Treat `BFCM_EXPECTED` rows as seasonal.

**Why it matters:** This is the baseline before reconciliation. A drop here with stable `add_to_cart` isolates the failure to the purchase tag.

**Pass threshold:** All non-BFCM days within band.

```sql
WITH d AS (
    SELECT event_date_local AS dt, COUNT(*) AS purchases
    FROM IDENTIFIER($ga4_flattened_table)
    WHERE EVENT_NAME='purchase'
    GROUP BY 1
),
joined AS (
    SELECT d1.dt, d1.purchases, d2.purchases AS past_p
    FROM d d1 JOIN d d2 ON d2.dt BETWEEN DATEADD('day',-14,d1.dt) AND DATEADD('day',-1,d1.dt)
),
stats AS (SELECT dt, MEDIAN(past_p) AS roll_median FROM joined GROUP BY dt),
mad_j AS (SELECT s.dt, s.roll_median, ABS(j.past_p - s.roll_median) AS dev FROM stats s JOIN joined j ON j.dt=s.dt),
mad AS (SELECT dt, MAX(roll_median) AS roll_median, MEDIAN(dev) AS mad FROM mad_j GROUP BY dt),
final AS (
    SELECT d.dt, d.purchases, m.roll_median, m.mad,
        DATEADD('day', (3 - DAYOFWEEKISO(DATE_FROM_PARTS(YEAR(d.dt),11,1)) + 7) % 7 + 21,
                DATE_FROM_PARTS(YEAR(d.dt),11,1)) AS thanksgiving
    FROM d LEFT JOIN mad m ON d.dt=m.dt
)
SELECT dt, purchases, roll_median, mad,
       (dt BETWEEN thanksgiving AND DATEADD('day',4,thanksgiving)) AS is_bfcm,
       CASE WHEN dt BETWEEN thanksgiving AND DATEADD('day',4,thanksgiving) THEN 'BFCM_EXPECTED'
            WHEN mad > 0 AND ABS(purchases - roll_median) > 3*mad THEN 'ANOMALY'
            ELSE 'ok' END AS flag
FROM final ORDER BY dt;
```

---

### 4.2 Purchase / transaction duplication `[MATERIAL]`

**Method:** Group purchases by `ecommerce_transaction_id`. Flag transaction_ids that fire more than once; count null/empty separately.

**Why it matters:** Duplicate purchases overstate revenue and conversion. The classic pattern is a thank-you page that re-fires on refresh.

**Pass threshold:** Duplicate `transaction_id` rate under 0.5% and null/empty rate near 0.

```sql
WITH tx AS (
    SELECT ecommerce_transaction_id AS transaction_id, COUNT(*) AS fires
    FROM IDENTIFIER($ga4_flattened_table)
    WHERE EVENT_NAME='purchase'
    GROUP BY 1
)
SELECT
    COUNT(*)                                                         AS distinct_txn_ids,
    SUM(CASE WHEN fires > 1 THEN 1 ELSE 0 END)                      AS duplicated_txn_ids,
    SUM(CASE WHEN transaction_id IS NULL OR transaction_id='' THEN fires ELSE 0 END) AS null_or_empty_txn_events,
    ROUND(100*duplicated_txn_ids/NULLIF(COUNT(*),0),2)              AS pct_duplicated
FROM tx;
```

---

### 4.3 Daily transaction gap vs Shopify `[MATERIAL]`

**Method:** After Domain 0 alignment, join distinct GA4 purchase transaction_ids to Shopify order counts per store-local day (excluding test and cancelled orders). Trend the gap and GA4 capture percentage over time.

**Why it matters:** This is the headline reconciliation: it quantifies how much GA4 under- or over-counts versus the system of record. A widening gap signals tag decay or rising consent/ad-block loss.

**Pass threshold:** GA4 captures a stable, explainable share of Shopify orders (commonly 85–95%) and the gap is steady rather than drifting.

```sql
WITH ga AS (
    SELECT event_date_local AS dt,
           COUNT(DISTINCT ecommerce_transaction_id) AS ga_orders
    FROM IDENTIFIER($ga4_flattened_table)
    WHERE EVENT_NAME='purchase'
    GROUP BY 1
),
shop AS (
    SELECT TO_DATE(CONVERT_TIMEZONE($store_tz, CREATED_AT)) AS dt,
           COUNT(*) AS shopify_orders
    FROM IDENTIFIER($shopify_table)
    WHERE COALESCE(TEST,FALSE)=FALSE AND CANCELLED_AT IS NULL
    AND TO_DATE(CONVERT_TIMEZONE($store_tz, CREATED_AT)) >= DATEADD('months', -24, CURRENT_DATE())
    GROUP BY 1
)
SELECT
    COALESCE(ga.dt, shop.dt)                                  AS dt,
    ga_orders, shopify_orders,
    shopify_orders - ga_orders                                AS gap,
    ROUND(100*ga_orders/NULLIF(shopify_orders,0),1)           AS ga_capture_pct
FROM ga FULL OUTER JOIN shop ON ga.dt = shop.dt
ORDER BY dt;
```

**Read the volume-weighted total, not just the simple daily average.** `AVG(ga_capture_pct)` across days is skewed upward by low-volume day noise (a day with 1 Shopify order and 1 GA4 order reads as "100% capture" and pulls the average up). Always also compute `SUM(ga_orders)/SUM(shopify_orders)` across the whole window — that number is often meaningfully lower than the naive daily average, and it's the one that matters for revenue/attribution impact.

---

### 4.4 Gap directionality and classification `[MATERIAL]`

**Method:** Left-join Shopify orders to GA4 purchases on order id and bucket the misses by `SOURCE_NAME`. Classify Shopify-only orders (POS, draft, phone, subscription rebills that never touch the storefront) versus GA4-only fires. For subscription brands, isolate rebills explicitly — expected to be absent from GA4.

**Why it matters:** The gap is only actionable once decomposed into GA4-only versus Shopify-only orders. For subscription DTC, recurring rebills are the dominant Shopify-only bucket and must be removed before judging tracking health.

**Pass threshold:** The residual unexplained gap is small after removing the known structural buckets.

```sql
WITH shop AS (
    SELECT TO_VARCHAR(ID) AS order_id,
           SOURCE_NAME,
           CASE WHEN SOURCE_NAME NOT IN ('web','checkout_next') THEN 'shopify_only_structural'
                ELSE 'web_expected_in_ga' END AS bucket
    FROM IDENTIFIER($shopify_table)
    WHERE COALESCE(TEST,FALSE)=FALSE AND CANCELLED_AT IS NULL
    AND TO_DATE(CONVERT_TIMEZONE($store_tz, CREATED_AT)) >= DATEADD('months', -24, CURRENT_DATE())
),
ga AS (
    SELECT DISTINCT ecommerce_transaction_id AS order_id
    FROM IDENTIFIER($ga4_flattened_table)
    WHERE EVENT_NAME='purchase'
)
SELECT
    s.SOURCE_NAME, s.bucket,
    COUNT(*)                                                    AS shopify_orders,
    SUM(CASE WHEN ga.order_id IS NULL THEN 1 ELSE 0 END)        AS missing_from_ga,
    ROUND(100*missing_from_ga/NULLIF(COUNT(*),0),1)             AS pct_missing
FROM shop s LEFT JOIN ga ON s.order_id = ga.order_id
GROUP BY 1,2
ORDER BY shopify_orders DESC;
```

**This is the check that tells you whether 4.3's gap is structural or a real tracking failure.** Sum `missing_from_ga` separately for `bucket = 'shopify_only_structural'` (expected — non-web channels never fire GA4 events) vs `bucket = 'web_expected_in_ga'` (unexpected — these should have fired a `purchase` event and didn't). If the `web_expected_in_ga` share of the total gap is large (e.g. >50%), that's the headline finding, not the structural exclusions — don't let a long tail of small non-web `SOURCE_NAME` values dilute the real signal in the writeup.

---

### 4.5 Revenue reconciliation, not just counts `[MATERIAL]`

**Method:** Compare daily `SUM(ecommerce_purchase_revenue)` against Shopify `SUM(TOTAL_PRICE)`. Confirm explicitly whether GA4 sends gross, subtotal, or post-discount, and how tax and shipping are treated, before judging the gap.

**Why it matters:** Matching order counts while revenue diverges still breaks every margin and ROAS calculation. GA4 sending subtotal while Shopify reports gross inflates apparent discount and deflates ROAS.

**Pass threshold:** Daily revenue reconciles within a small, documented tolerance attributable to the tax/shipping rule.

```sql
WITH ga AS (
    SELECT event_date_local AS dt,
           SUM(ecommerce_purchase_revenue) AS ga_revenue
    FROM IDENTIFIER($ga4_flattened_table)
    WHERE EVENT_NAME='purchase'
    GROUP BY 1
),
shop AS (
    SELECT TO_DATE(CONVERT_TIMEZONE($store_tz, CREATED_AT)) AS dt,
           SUM(TOTAL_PRICE::float) AS shopify_revenue
    FROM IDENTIFIER($shopify_table)
    WHERE COALESCE(TEST,FALSE)=FALSE AND CANCELLED_AT IS NULL
    AND TO_DATE(CONVERT_TIMEZONE($store_tz, CREATED_AT)) >= DATEADD('months', -24, CURRENT_DATE())
    GROUP BY 1
)
SELECT COALESCE(ga.dt,shop.dt) AS dt, ga_revenue, shopify_revenue,
       shopify_revenue - ga_revenue                              AS revenue_gap,
       ROUND(100*ga_revenue/NULLIF(shopify_revenue,0),1)         AS ga_revenue_capture_pct
FROM ga FULL OUTER JOIN shop ON ga.dt=shop.dt
ORDER BY dt;
```

**As with 4.3, report the volume-weighted total capture** (`SUM(ga_revenue)/SUM(shopify_revenue)` across the whole window), not the simple daily average. Also check whether the revenue-capture percentage tracks the order-capture percentage from 4.3 closely — if so, the missing orders aren't concentrated in a particular AOV band, useful context for the writeup. Pull a monthly rollup (`DATE_TRUNC('month', ...)`) alongside the daily view to establish whether the gap is chronic or a one-time incident — this materially changes remediation urgency.

---

### 4.6 Currency matching per order `[HYGIENE]`

**Method:** Join GA4 purchases to Shopify orders on order id. Compare the GA4 currency parameter to Shopify `PRESENTMENT_CURRENCY` — the currency the customer actually saw at checkout, **not** `CURRENCY`, which is the shop's base currency and produces false-positive mismatches on every international order.

**Why it matters:** Multi-currency stores can mismatch transaction currency and corrupt revenue. Summing mixed currencies as if single-currency silently overstates or understates revenue.

**Pass threshold:** Currency matches per order; no mixed-currency summation.

```sql
WITH ga AS (
    SELECT ecommerce_transaction_id AS order_id, ga_currency
    FROM IDENTIFIER($ga4_flattened_table)
    WHERE EVENT_NAME='purchase'
    AND ga_currency IS NOT NULL
),
shop AS (
    SELECT TO_VARCHAR(ID) AS order_id, PRESENTMENT_CURRENCY AS shop_currency
    FROM IDENTIFIER($shopify_table)
    WHERE COALESCE(TEST,FALSE)=FALSE
)
SELECT
    COUNT(*)                                                       AS matched_orders,
    SUM(CASE WHEN ga.ga_currency <> shop.shop_currency THEN 1 ELSE 0 END) AS currency_mismatches,
    ROUND(100*currency_mismatches/NULLIF(COUNT(*),0),2)            AS pct_mismatch
FROM ga JOIN shop ON ga.order_id=shop.order_id;
```

**If `PRESENTMENT_CURRENCY` doesn't exist on the connector version in use**, fall back to `CURRENCY` but treat any nonzero mismatch rate as a methodology artifact first (not a genuine data-quality finding) — international DTC brands routinely show 15-20%+ "mismatch" against base currency purely from EU/UK/CA/AU orders. Confirm with `DESCRIBE TABLE` which currency column is available before writing up this check.

---

### 4.7 Refund tracking and reconciliation `[MATERIAL]`

**Method:** Trend GA4 `refund` events per day. If `$shopify_refunds_table` is set, reconcile counts and value.

**Why it matters:** Without refund events, GA4 revenue is gross-only and overstates net. Missing refunds mean every net-revenue and contribution-margin view built on GA4 is overstated.

**Pass threshold:** Refund events present and reconciling to Shopify refunds within tolerance.

```sql
-- GA4 side
SELECT event_date_local AS dt, COUNT(*) AS ga_refund_events
FROM IDENTIFIER($ga4_flattened_table)
WHERE EVENT_NAME='refund'
GROUP BY 1 ORDER BY 1;

-- Shopify side (only if $shopify_refunds_table is set)
-- SELECT TO_DATE(CONVERT_TIMEZONE($store_tz, CREATED_AT)) AS dt,
--        COUNT(*) AS shopify_refunds
-- FROM IDENTIFIER($shopify_refunds_table)
-- WHERE TO_DATE(CONVERT_TIMEZONE($store_tz, CREATED_AT)) >= DATEADD('months', -24, CURRENT_DATE())
-- GROUP BY 1 ORDER BY 1;
```

---

### 4.8 Consistency across countries `[HYGIENE]`

**Method:** Break GA4 purchase counts down by `geo_country`. Join to Shopify country if a true per-country capture rate is needed.

**Why it matters:** Geo-level tracking gaps hide behind healthy totals. A single market with low capture often maps to a region-specific consent banner or CMP behavior.

**Pass threshold:** Capture rate consistent across major markets.

```sql
SELECT geo_country AS country,
       COUNT(DISTINCT ecommerce_transaction_id) AS ga_orders
FROM IDENTIFIER($ga4_flattened_table)
WHERE EVENT_NAME='purchase'
GROUP BY 1
ORDER BY ga_orders DESC;
```

---

## Domain 5 — Traffic quality & filtering

> Filtering bots, internal traffic, and self-referrals — the contamination sources that distort every rate.

---

### 5.1 Bot and invalid traffic proportion `[MATERIAL]`

**Method:** Build per-session event counts and time spans. Flag sessions with >50 events in <10 seconds and single-event sessions.

**Why it matters:** Bots inflate sessions and dilute every rate. A spike often coincides with a scraping campaign or a referral-spam wave.

**Pass threshold:** Suspected-bot share low and stable.

```sql
WITH s AS (
    SELECT USER_PSEUDO_ID, ga_session_id,
        COUNT(*) AS events,
        (MAX(EVENT_TIMESTAMP)-MIN(EVENT_TIMESTAMP))/1e6 AS span_seconds
    FROM IDENTIFIER($ga4_flattened_table)
    GROUP BY 1,2
)
SELECT
    COUNT(*)                                                          AS sessions,
    SUM(CASE WHEN events > 50 AND span_seconds < 10 THEN 1 ELSE 0 END) AS suspected_bot_velocity,
    SUM(CASE WHEN events = 1 THEN 1 ELSE 0 END)                       AS single_event_sessions,
    ROUND(100*suspected_bot_velocity/NULLIF(COUNT(*),0),2)            AS pct_suspected_bot
FROM s;
```

---

### 5.2 Internal and employee traffic `[MATERIAL]`

**Method:** Break events by the `traffic_type` parameter (GA4 stamps `'internal'` when an internal-traffic filter is configured). Where no filter exists, identify internal activity via known patterns.

**Why it matters:** Office, employee, and QA traffic inflates engagement and can fire test purchases. Unfiltered internal traffic both inflates engagement and pollutes the GA4-only purchase bucket in check 4.4.

**Pass threshold:** Internal traffic filtered or flagged; near-zero internal test purchases in production.

```sql
SELECT COALESCE(traffic_type,'(none)') AS traffic_type, COUNT(*) AS events
FROM IDENTIFIER($ga4_flattened_table)
GROUP BY 1 ORDER BY events DESC;
```

**Because `traffic_type` was precomputed via a `LEFT JOIN` (not an inner join), a row genuinely lacking the param shows up as `(none)` here rather than disappearing** — if `(none)` accounts for the entire row count, the internal-traffic-exclusion feature was never configured in GA4 Admin at all. That's a MATERIAL finding regardless of how clean the rest of the breakdown looks.

---

### 5.4 Self-referrals and referral exclusions `[HYGIENE]`

**Method:** List top `page_referrer` values on `session_start`. Scan for first-party domains and payment/subscription subdomains.

**Why it matters:** Payment and subscription subdomains appearing as referrers fragment sessions and steal attribution. A first-party domain in the referrer list is the cross-domain breakage from check 1.4 surfacing as misattribution.

**Pass threshold:** No first-party domains appearing as acquisition referrers.

```sql
SELECT COALESCE(page_referrer,'(none)') AS referrer, COUNT(*) AS events
FROM IDENTIFIER($ga4_flattened_table)
WHERE EVENT_NAME='session_start'
GROUP BY 1
ORDER BY events DESC
LIMIT 100;
```

---

## Domain 6 — Attribution & channels

> The most CFO-legible domain — where *"attribution is unknown, margins shrinking despite spend"* gets proven with numbers.

---

### 6.1 Channel composition over time `[MATERIAL]`

**Method:** Derive each session's channel from `collected_manual_source`/`collected_manual_medium` (fallback to `traffic_source_source`/`traffic_source_medium`). Map to channel groups. Trend session counts.

**Why it matters:** This is the baseline of where traffic and revenue are attributed. An unexplained mix shift often follows a UTM or redirect change rather than a real demand shift.

**Pass threshold:** Stable, explainable channel mix where shifts map to known campaign changes.

```sql
WITH s AS (
    SELECT
        event_date_local AS dt,
        LOWER(COALESCE(collected_manual_source, traffic_source_source,'(direct)')) AS source,
        LOWER(COALESCE(collected_manual_medium, traffic_source_medium,'(none)'))   AS medium
    FROM IDENTIFIER($ga4_flattened_table)
    WHERE EVENT_NAME='session_start'
)
SELECT dt,
    CASE
        WHEN medium IN ('cpc','ppc','paid','paidsearch') THEN 'Paid Search'
        WHEN medium IN ('email')                          THEN 'Email'
        WHEN medium IN ('social','paid_social','cpc_social','paid-social') THEN 'Social'
        WHEN medium IN ('referral')                       THEN 'Referral'
        WHEN medium IN ('organic')                        THEN 'Organic Search'
        WHEN medium IN ('sms')                             THEN 'SMS'
        WHEN medium IN ('affiliate')                       THEN 'Affiliate'
        WHEN source='(direct)' OR medium IN ('(none)','(not set)') THEN 'Direct'
        ELSE 'Other/Unassigned' END                       AS channel,
    COUNT(*)                                              AS sessions
FROM s
GROUP BY 1,2
ORDER BY dt, sessions DESC;
```

`paid-social`, `sms`, and `affiliate` are common mediums on DTC sites that otherwise fall into `Other/Unassigned` and inflate that bucket without indicating a real problem. Extend the `CASE` list for any client-specific mediums observed in check 6.3's raw breakdown before finalizing this view.

---

### 6.2 Direct and unassigned traffic share `[MATERIAL]`

**Method:** Trend % of sessions classified Direct or unassigned over time, split by `geo_country`.

**Why it matters:** This is the headline attribution finding: inflated Direct/unassigned equals wasted or misattributed spend the CFO is paying for blind. Direct above roughly 20–25% usually means broken UTMs, missing tagging on email or paid, or app-to-web handoff loss.

**Pass threshold:** Direct share under ~20% for ecommerce; low unassigned.

```sql
WITH s AS (
    SELECT
        event_date_local AS dt,
        geo_country AS country,
        LOWER(COALESCE(collected_manual_source, traffic_source_source,'(direct)')) AS source,
        LOWER(COALESCE(collected_manual_medium, traffic_source_medium,'(none)'))   AS medium
    FROM IDENTIFIER($ga4_flattened_table)
    WHERE EVENT_NAME='session_start'
)
SELECT dt, country,
    COUNT(*)                                                                         AS sessions,
    ROUND(100*AVG(CASE WHEN source='(direct)' OR medium IN ('(none)','(not set)')
                       THEN 1 ELSE 0 END),1)                                         AS pct_direct_unassigned
FROM s
GROUP BY 1,2
ORDER BY dt, sessions DESC;
```

---

### 6.3 UTM hygiene `[MATERIAL]`

**Method:** Group `session_start` by `utm_source`/`utm_medium` (the raw UTM params captured on the session, distinct from GA4's session-level attributed `traffic_source_source`/`medium`). Inspect for casing inconsistency, missing medium, and paid links arriving with gclid/fbclid but medium `(none)`.

**Why it matters:** Inconsistent UTMs are the usual root cause of inflated Direct and broken channel grouping. Mixed casing fragments one channel into several and feeds the Direct/unassigned bucket.

**Pass threshold:** UTM taxonomy consistent; paid and email traffic reliably tagged.

```sql
SELECT utm_source, utm_medium, COUNT(*) AS events
FROM IDENTIFIER($ga4_flattened_table)
WHERE EVENT_NAME='session_start'
GROUP BY 1,2 ORDER BY events DESC;
```

**Deliberately keep the raw casing in this query** (don't `LOWER()` here) — casing inconsistency (`Google`/`google`, `CPC`/`cpc`, `Influencer`/`influencer`) is exactly what this check exists to surface, and normalizing it away in the query would hide the finding. If found, flag it as FAIL and recommend normalizing at the source (ad platform / link-builder templates) or with a `LOWER()` transform in the semantic layer downstream — not by fixing the audit query.

---

### 6.4 Channel grouping re-derivation `[HYGIENE]`

**Method:** Compare `collected_manual_medium` (manual UTMs) to `traffic_source_medium` (GA4-attributed). Quantify reclassification volume.

**Why it matters:** The export's default attribution can disagree with the client's mental model of their channels. Large reclassification means dashboards using the default grouping tell a different story than the client believes.

**Pass threshold:** Re-derived grouping matches default within tolerance, or differences are understood.

```sql
SELECT
    LOWER(COALESCE(collected_manual_medium,'(none)')) AS collected_medium,
    LOWER(COALESCE(traffic_source_medium,'(none)'))    AS attributed_medium,
    COUNT(*) AS events
FROM IDENTIFIER($ga4_flattened_table)
WHERE EVENT_NAME='session_start'
GROUP BY 1,2
ORDER BY events DESC;
```

---

## Domain 7 — Consent, privacy & PII

> Weighted light for mostly-US/CA traffic (CCPA/CPRA and Global Privacy Control rather than full Consent Mode v2 modeling), but the PII check is non-negotiable everywhere — it is a compliance liability, not just a data-quality issue.

---

### 7.1 PII in event parameters `[BLOCKER]`

**Method:** Scan `page_location` for email and phone patterns. Any non-zero result is a finding to remediate immediately — not merely report.

**Why it matters:** Email, phone, or names in `page_location`, `page_title`, or custom params is a direct privacy and compliance liability and can violate Google's terms, risking data deletion. It typically comes from forms submitting via GET or account pages passing identifiers in the query string.

**Pass threshold:** Zero PII detected in any parameter.

```sql
SELECT
    COUNT(*)                                                                AS rows_checked,
    SUM(CASE WHEN page_location RLIKE '.*[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}.*' THEN 1 ELSE 0 END) AS email_in_url,
    SUM(CASE WHEN page_location RLIKE '.*(phone|tel)=.*[0-9]{7,}.*' THEN 1 ELSE 0 END)                         AS phone_in_url
FROM IDENTIFIER($ga4_flattened_table);
```

**Even a trivially small non-zero count (e.g. a few hundred rows out of 200M+) is still a FAIL-BLOCKER per the zero-tolerance policy** — do not downgrade severity because the volume is tiny. Pull one or two example URLs directly for the engineering team (likely a pre-filled form or account-recovery page appending `?email=` to the URL) without exposing the actual PII value in the audit report itself.

---

### 7.2 Consent signal coverage (US/CA) `[MATERIAL]`

**Method:** Break events by `privacy_analytics_storage` and `privacy_ads_storage` values. Quantify denied/opted-out share.

**Why it matters:** For US/CA the relevant signal is CCPA/CPRA opt-out and Global Privacy Control rather than EU Consent Mode, but it still affects how many events are collected. A rising opt-out share partly explains a widening GA4-vs-Shopify gap in check 4.3.

**Pass threshold:** Consent signal present and behaving; opt-out share within expected range.

```sql
SELECT
    privacy_analytics_storage AS analytics_storage,
    privacy_ads_storage       AS ads_storage,
    COUNT(*) AS events
FROM IDENTIFIER($ga4_flattened_table)
GROUP BY 1,2
ORDER BY events DESC;
```

---

### 7.3 Geo-resolution completeness `[HYGIENE]`

**Method:** Measure populated rate of `geo_country` and `geo_region` across all events.

**Why it matters:** Missing geo breaks the country splits used throughout Domains 4 and 6. High null geo undermines every country-split reconciliation elsewhere in this audit.

**Pass threshold:** Geo populated on the large majority of sessions.

```sql
SELECT
    ROUND(100*AVG(CASE WHEN geo_country IS NOT NULL AND geo_country<>'' THEN 1 ELSE 0 END),1) AS pct_has_country,
    ROUND(100*AVG(CASE WHEN geo_region  IS NOT NULL AND geo_region<>''  THEN 1 ELSE 0 END),1) AS pct_has_region
FROM IDENTIFIER($ga4_flattened_table);
```

---

## Deliverable structure

When this methodology is run for a client, the output document follows this structure, formatted for Google Docs:

1. **Executive summary** — open with the overall data trust verdict: **TRUSTED / CONDITIONALLY TRUSTED / UNRELIABLE**. List all FAIL-BLOCKER findings first (check id, actual value, threshold, business impact), then FAIL-MATERIAL. Close with the top-3 profit-leak or misattribution findings from Domains 4 and 6, quantified in $ or % wherever possible.
2. **Scope & data sources** — GA4 table, Shopify table, timezone, date range covered, and any Domain 0 findings that affect interpretation of later results.
3. **Domain-by-domain findings** — for each domain, which checks passed/failed, the actual values for every FAIL, and a one-line remediation recommendation.
4. **Reconciliation appendix** — the GA4-vs-Shopify daily gap table (from 4.3) summarized as average capture %, min/max day, trend direction; the revenue gap summary (from 4.5).
5. **Prioritized remediation roadmap** — a ranked table (Priority / Check / Finding / Owner / Effort / Expected impact), ordered BLOCKER → MATERIAL → HYGIENE, each estimating downstream analytics impact if left unaddressed.
6. **Scoring rollup** — a per-domain table (Domain / Checks / PASS / FAIL-BLOCKER / FAIL-MATERIAL / FAIL-HYGIENE / Domain score) and an overall trust score:

   `Overall trust score = 100 − (20 × BLOCKER count) − (5 × MATERIAL count) − (1 × HYGIENE count)`, capped at 0, max 100.

7. **Resize the warehouse back down** to the default size (or X-SMALL) once the audit is complete:
   ```sql
   ALTER WAREHOUSE {SNOWFLAKE_WAREHOUSE} SET WAREHOUSE_SIZE = 'X-SMALL';
   ```
   Keep the flattened staging table in place — do not drop it — so future re-audits skip the rebuild.

---

## Behavioral rules

- **Build the flattened staging table first**, before running any domain check. This is a one-time cost that makes every subsequent check dramatically faster — don't skip it to save a step, and don't re-flatten `EVENT_PARAMS` per query once it exists. Use a regular (non-temporary) table, since each CLI invocation is typically a fresh Snowflake session and a temporary table would not survive to the next check.
- **Run the actual SQL.** Do not summarize or skip queries — every check requires query execution and result interpretation.
- **Stop on Domain 0 BLOCKERs.** If 0.1 or 0.4 fail, surface the finding, explain the impact on downstream domains, and confirm with the client whether to continue.
- **Be quantitative.** Every finding must cite the actual number returned (e.g. "boundary_shifted_events = 4,231, representing 6.2% of events"). Never describe a result vaguely.
- **BFCM.** Never flag a BFCM-window day as an anomaly — note it as expected seasonality.
- **Subscription rebills.** For DTC subscription clients, isolate non-web `SOURCE_NAME` orders in check 4.4 before reporting the tracking gap. Rebills absent from GA4 are not a tracking failure.
- **Never hardcode table names or timezones** in query bodies — always use `IDENTIFIER($ga4_flattened_table)` and `$store_tz`.
- **One finding, one sentence.** In the report body, each check result gets exactly one interpretive sentence stating what it means for this client — not a generic definition.
- **Never use a correlated subquery to extract `EVENT_PARAMS`.** This Snowflake environment rejects it; use the `LATERAL JOIN` pattern from [Snowflake-specific gotchas](#snowflake-specific-gotchas) instead.
- **Never use a sliding-window `MEDIAN() OVER (... ROWS BETWEEN N PRECEDING ...)`.** This Snowflake environment rejects it; always use the self-join + `GROUP BY` pattern from checks 1.1/1.8/4.1.
- **Never wrap Shopify `CREATED_AT` in `TO_TIMESTAMP(...)`** before converting timezone — it is already `TIMESTAMP_TZ`. Use the 2-argument `CONVERT_TIMEZONE($store_tz, CREATED_AT)` form directly.
- **Always compare GA4 currency against Shopify `PRESENTMENT_CURRENCY`, never `CURRENCY`**, in check 4.6 — comparing against the shop's base currency produces false-positive mismatches on every international order.
