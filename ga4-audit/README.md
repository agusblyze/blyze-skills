# GA4 Data Audit — Methodology & Checklist

> Reusable audit playbook for GA4 export landed in Snowflake, reconciled against Shopify.
> Table names and timezone are parametrized — change the `SET` values, never the query bodies.

**Default sources**
- GA4: `RAW.BIGQUERY.GA4__EVENTS` (flattened Fivetran export)
- Shopify: `RAW.SHOPIFY.ORDERS` (Fivetran)
- Daily grain: parametrized store timezone (default `America/New_York`)

---

## Table of contents

- [How to use this document](#how-to-use-this-document)
- [Domain 0 — Foundations](#domain-0--foundations)
- [Domain 1 — Volume & completeness](#domain-1--volume--completeness)
- [Domain 2 — Identity & sessions](#domain-2--identity--sessions)
- [Domain 3 — Ecommerce funnel & item-level](#domain-3--ecommerce-funnel--item-level)
- [Domain 4 — Purchase reconciliation vs Shopify](#domain-4--purchase-reconciliation-vs-shopify)
- [Domain 5 — Traffic quality & filtering](#domain-5--traffic-quality--filtering)
- [Domain 6 — Attribution & channels](#domain-6--attribution--channels)
- [Domain 7 — Consent, privacy & PII](#domain-7--consent-privacy--pii)
- [Deliverable structure](#deliverable-structure)

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

### Parameters

Set these once per engagement. Every query references them via `IDENTIFIER()` and `$store_tz` — never hardcode table names or timezones in query bodies.

```sql
SET ga4_table               = 'RAW.BIGQUERY.GA4__EVENTS';
SET shopify_table           = 'RAW.SHOPIFY.ORDERS';
SET shopify_customers_table = 'RAW.SHOPIFY.CUSTOMERS';
SET shopify_refunds_table   = 'RAW.SHOPIFY.ORDER_REFUNDS'; -- optional
SET store_tz                = 'America/New_York';
```

### Schema conventions

- **Timezone** — `EVENT_TIMESTAMP` is microseconds UTC. Store-local day:
  `TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz, TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)))`.
  Use this everywhere; never use the raw `EVENT_DATE` string.
- **Event params** — `EVENT_PARAMS` is an array of `{key, value:{string_value, int_value, double_value, float_value}}`. Extract a param with `LATERAL FLATTEN` filtered on `key`, then `COALESCE` the value subtypes.
- **Items** — live in `ECOMMERCE:items` (no top-level `ITEMS` column). Transaction id: `ECOMMERCE:transaction_id`. Revenue: `ECOMMERCE:purchase_revenue`. Confirm these paths on your sync before trusting Domains 3–4.
- **Freshness** — `__UPDATED_AT` is the connector load/reprocess timestamp, used for late-arrival checks. There is no `events_intraday` concept in this landing.
- **Shopify columns** (Fivetran standard) — `ID`, `CREATED_AT` (UTC), `TOTAL_PRICE`, `CURRENCY`, `FINANCIAL_STATUS`, `SOURCE_NAME`, `TEST`, `CANCELLED_AT`. Remap if your sync differs.
- **BFCM** — the rolling-median ±3·MAD anomaly band will legitimately flag the late-November BFCM window (Thanksgiving through Cyber Monday). Queries tag this window as `BFCM_EXPECTED`. Treat it as seasonality, not a data anomaly.

> **Performance note** — the per-event param subqueries are correct but scan-heavy at volume. For repeated runs, build a flattened `STG_GA4_EVENTS` view that pivots the common params out once and point these queries at it.

### Run order

Several checks are prerequisites for others. **Do not skip this sequence** or later reconciliations will produce phantom findings.

1. Domain 0 — Foundations *(calibrates every count and every Shopify gap)*
2. Domain 1 — Volume & completeness
3. Domain 2 — Identity & sessions
4. Domain 3 — Ecommerce funnel & item-level
5. Domain 4 — Purchase reconciliation vs Shopify *(depends on Domain 0)*
6. Domain 5 — Traffic quality & filtering
7. Domain 6 — Attribution & channels
8. Domain 7 — Consent, privacy & PII

---

## Domain 0 — Foundations

> Run this domain first. These checks silently break every count and every Shopify reconciliation downstream if skipped.

---

### 0.1 Timezone alignment between GA4 and Shopify `[BLOCKER]`

**Method:** Derive the GA4 event date in store-local time with `TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz, TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)))`; do not use the raw `EVENT_DATE` string. Convert Shopify `CREATED_AT` to the same zone with `CONVERT_TIMEZONE('UTC', $store_tz, CREATED_AT)`. Run the query below to count how many events land on a different calendar day under the two definitions, then confirm both daily grains share that one zone before any reconciliation.

**Why it matters:** GA4 and Shopify timestamps are stored in UTC but the two systems report on different calendar boundaries. A 3–5 hour offset reshuffles 5–15% of orders across the day boundary and manufactures a daily GA4-vs-Shopify gap that does not exist — every Domain 4 reconciliation is invalid until both sides are pinned to one zone. A sawtooth gap (high one day, low the next) or a constant ~N-order lag is the timezone signature, not real data loss.

**Pass threshold:** Under a single store-local zone, GA4 and Shopify daily counts align to the same calendar boundary with no systematic single-day-shift pattern in the gap series.

```sql
-- 0.1  Timezone alignment: compare the connector's EVENT_DATE string vs ET-derived date.
--      A non-zero mismatch share is the timezone signature you must resolve before Domain 4.
SELECT
    EVENT_DATE                                    AS raw_event_date_string,
    TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
            TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS event_date_et,
    COUNT(*)                                      AS events,
    SUM(CASE WHEN EVENT_DATE <> TO_CHAR(TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
            TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))),'YYYYMMDD') THEN 1 ELSE 0 END) AS boundary_shifted_events
FROM IDENTIFIER($ga4_table) e
GROUP BY 1,2
ORDER BY 2 DESC
LIMIT 200;
```

---

### 0.2 Sync freshness and partial-day mechanics `[BLOCKER]`

**Method:** Run the query below to compute, per store-local event day, the lag between the latest `EVENT_TIMESTAMP` and `MAX(__UPDATED_AT)`. Identify the trailing days that are still partially synced and exclude or annotate them in every trend; do not read the most recent 1–3 days as final.

**Why it matters:** The connector loads events on a schedule, so the most recent day is usually partially loaded at query time. Reading the trailing days without accounting for sync lag produces phantom drops that look like tracking failure; a reliable drop on only the latest 1–3 days is almost always sync latency, not lost events.

**Pass threshold:** Completeness trends exclude or clearly annotate the trailing partially-synced days; no recurring drop on the latest 1–3 days.

```sql
-- 0.2  Sync freshness: event day vs when it was last loaded.
--      Large lag on the most recent days = connector latency, not lost tracking.
SELECT
    TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
            TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)))  AS event_date_et,
    MIN(__UPDATED_AT)                              AS first_loaded_at,
    MAX(__UPDATED_AT)                              AS last_loaded_at,
    DATEDIFF('hour',
        TO_TIMESTAMP_NTZ(MAX(EVENT_TIMESTAMP),6),
        MAX(__UPDATED_AT))                         AS load_lag_hours,
    COUNT(*)                                       AS events
FROM IDENTIFIER($ga4_table)
WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1
ORDER BY 1 DESC;
```

---

### 0.3 Late-arriving and reprocessed events `[MATERIAL]`

**Method:** Run the query below to count, per `EVENT_NAME`, the share of events whose `__UPDATED_AT` is more than 24h and more than 72h after the event occurred. Pay particular attention to the `purchase` row. Re-capture a fixed past day a few days apart and confirm the totals have stopped moving.

**Why it matters:** Events can arrive up to 72h late (offline app events, delayed beacons), so a snapshot taken too early undercounts recent days and re-running later silently changes historical totals. Large late deltas on `purchase` specifically mean revenue dashboards built on fresh data understate recent performance.

**Pass threshold:** Late arrivals settle to under 1–2% change after 72h for web; app behavior is documented separately.

```sql
-- 0.3  Late-arriving events: quantify by event_name to surface if purchase arrives late.
SELECT
    EVENT_NAME,
    COUNT(*)                                                         AS total_events,
    SUM(CASE WHEN DATEDIFF('hour',
            TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6), __UPDATED_AT) > 24
        THEN 1 ELSE 0 END)                                          AS arrived_gt_24h,
    SUM(CASE WHEN DATEDIFF('hour',
            TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6), __UPDATED_AT) > 72
        THEN 1 ELSE 0 END)                                          AS arrived_gt_72h,
    ROUND(100 * arrived_gt_72h / NULLIF(total_events,0), 2)         AS pct_gt_72h
FROM IDENTIFIER($ga4_table)
WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1
ORDER BY total_events DESC;
```

---

### 0.4 Canonical dedup keys defined `[BLOCKER]`

**Method:** Adopt these exact grains and reuse them in every dedup check:
- **Event identity** — `(USER_PSEUDO_ID, ga_session_id, EVENT_NAME, EVENT_TIMESTAMP, EVENT_BUNDLE_SEQUENCE_ID)`
- **Session identity** — `(USER_PSEUDO_ID, ga_session_id)`
- **Transaction identity** — `ECOMMERCE:transaction_id`

Run the query below to confirm the event-identity grain is actually unique and quantify any duplicate rows.

**Why it matters:** The GA4 export has no row-level primary key, so dedup logic must be explicit and identical everywhere. Inconsistent keys cause two analysts to report different duplication rates on the same data.

**Pass threshold:** The event-identity grain is unique (duplicate-row rate near 0) and the keys are applied uniformly across checks 2.x and 4.x.

```sql
-- 0.4  Canonical dedup-key integrity: are the event-identity keys actually unique?
--      Grain = (user_pseudo_id, ga_session_id, event_name, event_timestamp, event_bundle_sequence_id)
WITH base AS (
    SELECT
        USER_PSEUDO_ID,
        (SELECT p.value:value:int_value::number
           FROM LATERAL FLATTEN(input => e.EVENT_PARAMS) p
          WHERE p.value:key::string = 'ga_session_id')     AS ga_session_id,
        EVENT_NAME, EVENT_TIMESTAMP, EVENT_BUNDLE_SEQUENCE_ID
    FROM IDENTIFIER($ga4_table) e
    WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
)
SELECT
    COUNT(*)                                                AS rows_total,
    COUNT(*) - COUNT(DISTINCT USER_PSEUDO_ID||':'||ga_session_id||':'||EVENT_NAME||':'
              ||EVENT_TIMESTAMP||':'||EVENT_BUNDLE_SEQUENCE_ID) AS duplicate_rows,
    ROUND(100 * duplicate_rows / NULLIF(rows_total,0), 3)   AS pct_duplicate_rows
FROM base;
```

---

## Domain 1 — Volume & completeness

> Volume, completeness, naming, parameter coverage, and anomaly detection across all event collection.

---

### 1.1 Total event volume by day `[MATERIAL]`

**Method:** Run the query below to count events per store-local day and overlay the rolling-median ±3·MAD band. Read the `is_bfcm` / `flag` columns: days inside the BFCM window (Thanksgiving through Cyber Monday) return `BFCM_EXPECTED` and must not be treated as anomalies. Investigate only the remaining `ANOMALY` rows and annotate them against known tag deploys.

**Why it matters:** Daily event volume detects collection outages, tag deployments, and seasonality, and is the foundation for every other trend. A step-change usually maps to a GTM/tag release; a single zero day to a collection or sync outage. The late-November BFCM spike is legitimate DTC seasonality, not a data issue.

**Pass threshold:** No unexplained zero days; all non-BFCM days fall within the ±3·MAD band except where a deploy explains the move.

```sql
-- 1.1  Total event volume by store-local day with rolling-median / MAD anomaly band.
--      BFCM (Thanksgiving -> Cyber Monday) is flagged separately so the legitimate
--      DTC seasonal spike is not mistaken for a data anomaly.
WITH daily AS (
    SELECT
        TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
                TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS d,
        COUNT(*) AS events
    FROM IDENTIFIER($ga4_table)
    WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
    GROUP BY 1
),
flagged AS (
    SELECT d, events,
        -- 4th Thursday of November for this row's year:
        DATEADD('day', (3 - DAYOFWEEKISO(DATE_FROM_PARTS(YEAR(d),11,1)) + 7) % 7 + 21,
                DATE_FROM_PARTS(YEAR(d),11,1))                      AS thanksgiving,
        MEDIAN(events) OVER (ORDER BY d ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING) AS roll_median,
        MEDIAN(ABS(events - MEDIAN(events) OVER (ORDER BY d ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING)))
            OVER (ORDER BY d ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING)            AS mad
    FROM daily
)
SELECT d, events, roll_median, mad,
    (d BETWEEN thanksgiving AND DATEADD('day',4,thanksgiving))      AS is_bfcm,
    CASE
        WHEN d BETWEEN thanksgiving AND DATEADD('day',4,thanksgiving) THEN 'BFCM_EXPECTED'
        WHEN mad > 0 AND ABS(events - roll_median) > 3*mad          THEN 'ANOMALY'
        ELSE 'ok'
    END AS flag
FROM flagged
ORDER BY d;
```

---

### 1.2 Volume split by stream and platform `[MATERIAL]`

**Method:** Run the query below to trend event counts grouped by `STREAM_ID` and `PLATFORM`, one series per stream. Check each active stream independently rather than the total.

**Why it matters:** A drop in one stream (web vs app vs server-side) nets out at the total and stays hidden. A server-side stream going dark while web continues is a classic GTM server-container failure that only a per-stream view surfaces.

**Pass threshold:** Each active stream is individually continuous with no silent single-stream dropout.

```sql
-- 1.2  Volume split by stream and platform (single-stream dropouts hide in the total).
SELECT
    TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
            TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS d,
    STREAM_ID,
    PLATFORM,
    COUNT(*) AS events
FROM IDENTIFIER($ga4_table)
WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1,2,3
ORDER BY 1 DESC, events DESC;
```

---

### 1.3 page_view completeness and single-fire `[MATERIAL]`

**Method:** Run query 1.3a to trend `page_view` per day, then 1.3b to dedup on `(USER_PSEUDO_ID, ga_session_id, page_location)` within a 2-second window and quantify the double-fire rate. When the rate is high, inspect for GTM plus hardcoded gtag both present, or SPA route changes firing a manual and an automatic `page_view`.

**Why it matters:** Double-fires inflate sessions and pageviews and dilute every conversion rate, while under-firing breaks landing-page analysis. A consistent ~2x pageview-to-session ratio versus benchmark signals systematic double-tagging.

**Pass threshold:** Duplicate `page_view` rate under 1% with a stable daily trend.

```sql
-- 1.3a  page_view completeness by day.
SELECT
    TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
            TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS d,
    COUNT(*) AS page_views
FROM IDENTIFIER($ga4_table)
WHERE EVENT_NAME = 'page_view'
AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1
ORDER BY 1;

-- 1.3b  page_view double-fire: same user/session/location within 2 seconds.
WITH pv AS (
    SELECT
        USER_PSEUDO_ID,
        (SELECT p.value:value:int_value::number FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
          WHERE p.value:key::string='ga_session_id')    AS ga_session_id,
        (SELECT p.value:value:string_value::string FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
          WHERE p.value:key::string='page_location')    AS page_location,
        EVENT_TIMESTAMP
    FROM IDENTIFIER($ga4_table) e
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
    COUNT(*)                                                       AS page_views,
    SUM(CASE WHEN micros_since_prev < 2000000 THEN 1 ELSE 0 END)  AS likely_double_fires,
    ROUND(100 * likely_double_fires / NULLIF(COUNT(*),0), 2)       AS pct_double_fire
FROM flagged;
```

---

### 1.4 session_start single-fire and session integrity `[MATERIAL]`

**Method:** Run the query below to count `session_start` per `(USER_PSEUDO_ID, ga_session_id)` — the result must be exactly one. Separately flag sessions split at midnight or at timezone resets, sessions with zero events, and cross-domain breakage where the checkout or subscription subdomain mints a new session.

**Why it matters:** Fragmented or duplicated sessions corrupt every per-session metric and all channel attribution. A new session created at the payment or subscription subdomain shows up as a self-referral and misattributes the conversion.

**Pass threshold:** Exactly one `session_start` per session id, and cross-domain handoff preserves `ga_session_id`.

```sql
-- 1.4  session_start single-fire: each (user_pseudo_id, ga_session_id) should have exactly one.
WITH ss AS (
    SELECT
        USER_PSEUDO_ID,
        (SELECT p.value:value:int_value::number FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
          WHERE p.value:key::string='ga_session_id') AS ga_session_id,
        COUNT(*) AS session_start_count
    FROM IDENTIFIER($ga4_table) e
    WHERE EVENT_NAME='session_start'
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
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

**Method:** Run 1.5a to flag event names that are not lowercase `snake_case` or exceed 40 characters and to classify reserved versus custom names; run 1.5b to count distinct event names against the 500 cap; run 1.5c to confirm error and failure events (`form_error`, `payment_failed`) exist.

**Why it matters:** Bad names and reserved-name collisions corrupt both the GA4 UI and the export, and high-cardinality event sets hit the 500-distinct-event cap after which new events are silently discarded. Missing error events mean failure states are invisible.

**Pass threshold:** Naming-convention adherence above 95%, distinct events well under 500, and error/failure events present.

```sql
-- 1.5a  Event naming convention + reserved/junk + cardinality vs 500 cap.
SELECT
    EVENT_NAME,
    COUNT(*) AS events,
    CASE WHEN EVENT_NAME = LOWER(EVENT_NAME)
              AND EVENT_NAME NOT LIKE '% %'
              AND LENGTH(EVENT_NAME) <= 40 THEN 'ok' ELSE 'NAMING_ISSUE' END AS naming_flag,
    CASE WHEN EVENT_NAME IN (
        'first_open','first_visit','session_start','page_view','user_engagement',
        'scroll','click','view_search_results','purchase','refund','add_to_cart',
        'begin_checkout','view_item','view_item_list','select_item',
        'add_payment_info','add_shipping_info')
        THEN 'standard/reserved' ELSE 'custom' END AS event_class
FROM IDENTIFIER($ga4_table)
WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1
ORDER BY events DESC;

-- 1.5b  Distinct event cardinality vs the 500 cap.
SELECT
    COUNT(DISTINCT EVENT_NAME) AS distinct_events,
    CASE WHEN COUNT(DISTINCT EVENT_NAME) > 450 THEN 'NEAR_CAP' ELSE 'ok' END AS flag
FROM IDENTIFIER($ga4_table)
WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE());

-- 1.5c  Error/failure event presence (should exist for a healthy implementation).
SELECT EVENT_NAME, COUNT(*) AS events
FROM IDENTIFIER($ga4_table)
WHERE (EVENT_NAME ILIKE '%error%' OR EVENT_NAME ILIKE '%fail%'
    OR EVENT_NAME IN ('form_error','payment_failed'))
AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1
ORDER BY events DESC;
```

---

### 1.6 Parameter coverage and null rates `[HYGIENE]`

**Method:** Run the query below to compute, per key event, the populated rate of the parameters that feed client reports (`page_location`, `ga_session_id`, and any custom dimensions in scope) by flattening `EVENT_PARAMS` and measuring non-null shares.

**Why it matters:** An event can fire while the parameters the client actually reports on are null, leaving the event present but useless. A high null rate on a reported parameter is invisible in event counts yet quietly breaks the dashboard built on it.

**Pass threshold:** Critical parameters populated above 98% on the events that use them.

```sql
-- 1.6  Parameter null/coverage on key params (page_location, ga_session_id).
SELECT
    EVENT_NAME,
    COUNT(*) AS events,
    ROUND(100*AVG(CASE WHEN (
        SELECT p.value:value:string_value::string FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
        WHERE p.value:key::string='page_location') IS NOT NULL THEN 1 ELSE 0 END),1) AS pct_has_page_location,
    ROUND(100*AVG(CASE WHEN (
        SELECT p.value:value:int_value::number FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
        WHERE p.value:key::string='ga_session_id') IS NOT NULL THEN 1 ELSE 0 END),1) AS pct_has_session_id
FROM IDENTIFIER($ga4_table) e
WHERE EVENT_NAME IN ('page_view','add_to_cart','begin_checkout','purchase','view_item')
AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1
ORDER BY events DESC;
```

---

### 1.7 Events firing per page `[HYGIENE]`

**Method:** Run the query below to group event counts by page path (`page_location` with the query string stripped) and event name, then compare the event set each template fires (PDP, PLP, cart, checkout) against what it should fire.

**Why it matters:** This surfaces templates that are over- or under-instrumented. A PDP template with no `view_item` points to a tag scoped to the wrong page pattern.

**Pass threshold:** Each key template fires its expected event set, with no template silently missing `add_to_cart` or `begin_checkout`.

```sql
-- 1.7  Events firing per page (template instrumentation coverage).
SELECT
    REGEXP_REPLACE(
        (SELECT p.value:value:string_value::string FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
          WHERE p.value:key::string='page_location'),
        '\\?.*$','')   AS page_path,
    EVENT_NAME,
    COUNT(*)           AS events
FROM IDENTIFIER($ga4_table) e
WHERE EVENT_NAME IN ('page_view','view_item','add_to_cart','begin_checkout','purchase')
AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1,2
ORDER BY events DESC
LIMIT 200;
```

---

### 1.8 Anomaly detection on core metrics `[MATERIAL]`

**Method:** Run the query below to compute daily transactions, users, sessions, and page_views and apply the rolling-median ±3·MAD test to transactions. Read the `is_bfcm` / `txn_flag` columns and treat `BFCM_EXPECTED` rows as seasonal, investigating only `TXN_ANOMALY` rows. Extend the same band to the other three metrics as needed.

**Why it matters:** Systematic outlier detection across the metrics that drive decisions catches problems a single-metric view misses. Correlated anomalies across all four on the same non-BFCM date point to a collection or sync issue rather than real behavior.

**Pass threshold:** All four series fall within band on non-BFCM days except where an explained event accounts for the move.

```sql
-- 1.8  Anomaly detection on four core metrics (transactions, users, sessions, page_views).
WITH d AS (
    SELECT
        TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
                TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS dt,
        SUM(CASE WHEN EVENT_NAME='purchase'  THEN 1 ELSE 0 END) AS transactions,
        COUNT(DISTINCT USER_PSEUDO_ID)                          AS users,
        COUNT(DISTINCT USER_PSEUDO_ID||':'||
            (SELECT p.value:value:int_value::number FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
              WHERE p.value:key::string='ga_session_id'))       AS sessions,
        SUM(CASE WHEN EVENT_NAME='page_view' THEN 1 ELSE 0 END) AS page_views
    FROM IDENTIFIER($ga4_table) e
    WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
    GROUP BY 1
)
SELECT dt, transactions, users, sessions, page_views,
    (dt BETWEEN tg AND DATEADD('day',4,tg)) AS is_bfcm,
    CASE
        WHEN dt BETWEEN tg AND DATEADD('day',4,tg) THEN 'BFCM_EXPECTED'
        WHEN ABS(transactions - MEDIAN(transactions) OVER (ORDER BY dt ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING))
             > 3*NULLIF(MEDIAN(ABS(transactions - MEDIAN(transactions) OVER (ORDER BY dt ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING)))
               OVER (ORDER BY dt ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING),0) THEN 'TXN_ANOMALY'
        ELSE 'ok'
    END AS txn_flag
FROM (
    SELECT d.*,
           DATEADD('day', (3 - DAYOFWEEKISO(DATE_FROM_PARTS(YEAR(dt),11,1)) + 7) % 7 + 21,
                   DATE_FROM_PARTS(YEAR(dt),11,1)) AS tg
    FROM d
) d
ORDER BY dt;
```

---

## Domain 2 — Identity & sessions

> The linchpin for all downstream LTV, retention, and cohort work — central to the subscription-DTC use case.

---

### 2.1 user_id implementation post-login `[MATERIAL]`

**Method:** Run the query below to trend, per store-local day, the share of sessions carrying a non-null `USER_ID`. Confirm `USER_ID` begins populating at the point of authentication rather than remaining null across logged-in sessions.

**Why it matters:** Without a reliable `USER_ID`, cross-device stitching and logged-in LTV are impossible. A drop coincides with a login/auth or tagging change, and sustained low coverage caps any identity-resolution work downstream.

**Pass threshold:** `USER_ID` present on the large majority of post-login sessions with a stable trend.

```sql
-- 2.1  user_id coverage over time (post-login proxy via user_id presence).
SELECT
    TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
        TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)))                    AS d,
    COUNT(DISTINCT USER_PSEUDO_ID||':'||
        (SELECT p.value:value:int_value::number FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
          WHERE p.value:key::string='ga_session_id'))             AS sessions,
    COUNT(DISTINCT CASE WHEN USER_ID IS NOT NULL THEN USER_PSEUDO_ID||':'||
        (SELECT p.value:value:int_value::number FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
          WHERE p.value:key::string='ga_session_id') END)         AS sessions_with_user_id,
    ROUND(100*sessions_with_user_id/NULLIF(sessions,0),1)         AS pct_sessions_with_user_id
FROM IDENTIFIER($ga4_table) e
WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1
ORDER BY 1;
```

---

### 2.2 user_id ↔ Shopify customer_id join coverage `[BLOCKER for LTV]`

**Method:** Run the query below to join distinct GA4 `USER_ID` values to Shopify customer IDs and measure the match rate. First confirm the join contract: inspect whether `USER_ID` holds the raw Shopify `customer_id`, a hashed email, or another identifier, and remap the customers table or key accordingly before trusting the rate.

**Why it matters:** Without a reliable GA4-to-Shopify join, behavioral data cannot be tied to revenue, retention, or churn cohorts. A low match rate means every LTV/retention deliverable is built on a fraction of customers.

**Pass threshold:** Match rate high enough to support cohort analysis (set and document the threshold per client).

```sql
-- 2.2  user_id <-> Shopify customer join coverage.
--      Adjust the join key: GA4 USER_ID is often the Shopify customer_id or a hash of email.
WITH ga AS (
    SELECT DISTINCT USER_ID
    FROM IDENTIFIER($ga4_table)
    WHERE USER_ID IS NOT NULL
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
),
shop AS (
    SELECT DISTINCT TO_VARCHAR(ID) AS customer_id
    FROM IDENTIFIER($shopify_customers_table)
)
SELECT
    (SELECT COUNT(*) FROM ga)                                   AS ga_distinct_user_ids,
    (SELECT COUNT(*) FROM shop)                                 AS shopify_customers,
    COUNT(*)                                                    AS matched,
    ROUND(100*COUNT(*)/NULLIF((SELECT COUNT(*) FROM shop),0),1) AS pct_shopify_matched_to_ga
FROM ga JOIN shop ON ga.USER_ID = shop.customer_id;
```

---

### 2.3 user_pseudo_id stability and inflation `[MATERIAL]`

**Method:** Run the query below to count distinct `USER_PSEUDO_ID` values per logged-in `USER_ID` and report the average, median, and worst case. Inspect users with an unusually high pseudo-id count for rapid cookie turnover.

**Why it matters:** Cookie and consent resets mint new pseudo-ids, inflating user counts and fragmenting journeys. High inflation overstates new users and understates returning behavior, distorting acquisition reporting.

**Pass threshold:** Pseudo-id-per-user inflation within the expected range for the consent and browser mix.

```sql
-- 2.3  user_pseudo_id inflation: pseudo-ids per logged-in user_id (cookie/consent churn).
WITH m AS (
    SELECT USER_ID, COUNT(DISTINCT USER_PSEUDO_ID) AS pseudo_ids
    FROM IDENTIFIER($ga4_table)
    WHERE USER_ID IS NOT NULL
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
    GROUP BY 1
)
SELECT
    COUNT(*)                                        AS logged_in_users,
    ROUND(AVG(pseudo_ids),2)                        AS avg_pseudo_ids_per_user,
    MEDIAN(pseudo_ids)                              AS median_pseudo_ids_per_user,
    MAX(pseudo_ids)                                 AS worst_case,
    SUM(CASE WHEN pseudo_ids > 3 THEN 1 ELSE 0 END) AS users_gt_3_pseudo_ids
FROM m;
```

---

## Domain 3 — Ecommerce funnel & item-level

> Ecommerce funnel event presence, healthy step ratios, and item-level data completeness.

---

### 3.1 Funnel event presence and healthy ratios `[MATERIAL]`

**Method:** Run the query below to count each funnel event (`view_item_list`, `view_item`, `add_to_cart`, `begin_checkout`, `add_shipping_info`, `add_payment_info`, `purchase`) and express each as a percentage of `view_item`. Confirm every step exists and the step-to-step ratios sit in plausible ecommerce ranges.

**Why it matters:** Missing or malformed funnel events break conversion-rate analysis end to end. A near-zero `begin_checkout` alongside a healthy `add_to_cart` points to a checkout-domain tagging gap, which is common on Shopify checkout.

**Pass threshold:** All steps present, ratios within plausible ranges, and no step at effectively zero.

```sql
-- 3.1  Funnel event presence + step-to-step ratios.
WITH f AS (
    SELECT EVENT_NAME, COUNT(*) AS events
    FROM IDENTIFIER($ga4_table)
    WHERE EVENT_NAME IN (
        'view_item_list','view_item','add_to_cart','begin_checkout',
        'add_shipping_info','add_payment_info','purchase')
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
    GROUP BY 1
)
SELECT EVENT_NAME, events,
    ROUND(100*events / NULLIF(MAX(CASE WHEN EVENT_NAME='view_item' THEN events END) OVER (),0),1) AS pct_of_view_item
FROM f
ORDER BY events DESC;
```

---

### 3.2 Item-detail completeness across ecommerce events `[MATERIAL]`

**Method:** Run the query below to flatten `ECOMMERCE:items` on `view_item`, `add_to_cart`, `begin_checkout`, and `purchase`, and compute the populated rate of `item_id`, `item_name`, `item_category`, `item_brand`, `price`, and `quantity` per event.

**Why it matters:** Sparse item arrays break product, category, and merchandising analysis. `item_id` present but `item_category` null means product-level analysis works while category roll-ups silently fail.

**Pass threshold:** Core item fields populated above 98% on every ecommerce event.

```sql
-- 3.2  Item-detail completeness across ecommerce events (null rates on item fields).
WITH items AS (
    SELECT e.EVENT_NAME, i.value AS item
    FROM IDENTIFIER($ga4_table) e,
         LATERAL FLATTEN(input => e.ECOMMERCE:items) i
    WHERE e.EVENT_NAME IN ('view_item','add_to_cart','begin_checkout','purchase')
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
)
SELECT EVENT_NAME,
    COUNT(*)                                                                           AS item_rows,
    ROUND(100*AVG(CASE WHEN item:item_id::string       IS NOT NULL THEN 1 ELSE 0 END),1) AS pct_item_id,
    ROUND(100*AVG(CASE WHEN item:item_name::string     IS NOT NULL THEN 1 ELSE 0 END),1) AS pct_item_name,
    ROUND(100*AVG(CASE WHEN item:item_category::string IS NOT NULL THEN 1 ELSE 0 END),1) AS pct_item_category,
    ROUND(100*AVG(CASE WHEN item:item_brand::string    IS NOT NULL THEN 1 ELSE 0 END),1) AS pct_item_brand,
    ROUND(100*AVG(CASE WHEN item:price                 IS NOT NULL THEN 1 ELSE 0 END),1) AS pct_price,
    ROUND(100*AVG(CASE WHEN item:quantity              IS NOT NULL THEN 1 ELSE 0 END),1) AS pct_quantity
FROM items
GROUP BY 1
ORDER BY item_rows DESC;
```

---

### 3.3 Purchase events contain item-level data `[MATERIAL]`

**Method:** Run the query below to count purchase events where `ECOMMERCE:items` is null or has zero rows and express it as a share of all purchases.

**Why it matters:** A purchase with an empty items array breaks revenue-by-product and AOV-by-item. Item-less purchases usually mean the purchase tag fires before the dataLayer ecommerce object is populated.

**Pass threshold:** Effectively 0% of purchases with an empty items array.

```sql
-- 3.3  Purchases with an empty items array (breaks revenue-by-product).
SELECT
    COUNT(*)                                                           AS purchases,
    SUM(CASE WHEN ECOMMERCE:items IS NULL
              OR ARRAY_SIZE(ECOMMERCE:items) = 0 THEN 1 ELSE 0 END)  AS purchases_no_items,
    ROUND(100*purchases_no_items/NULLIF(COUNT(*),0),2)                AS pct_no_items
FROM IDENTIFIER($ga4_table)
WHERE EVENT_NAME='purchase'
AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE());
```

---

### 3.4 Item value reconciliation `[HYGIENE]`

**Method:** Run the query below to compute, per purchase, `SUM(item price × quantity)` from `ECOMMERCE:items` and compare it to `ECOMMERCE:purchase_revenue`, then quantify the share of purchases where the two disagree beyond a cent (allowing for the documented tax/shipping/discount rule).

**Why it matters:** The sum of item revenue should reconcile to the event-level value. A systematic mismatch reveals inconsistent gross/net/discount handling in the tag.

**Pass threshold:** Item totals reconcile to event value within the documented tax/shipping rule.

```sql
-- 3.4  Item value reconciliation: sum(item price*qty) vs event-level ecommerce value.
WITH p AS (
    SELECT
        e.ECOMMERCE:transaction_id::string                                  AS transaction_id,
        e.ECOMMERCE:purchase_revenue::float                                 AS event_value,
        (SELECT SUM(i.value:price::float * COALESCE(i.value:quantity::float,1))
           FROM LATERAL FLATTEN(input => e.ECOMMERCE:items) i)              AS item_value
    FROM IDENTIFIER($ga4_table) e
    WHERE e.EVENT_NAME='purchase'
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
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
> **Requires Domain 0 to be complete.** Do not run daily gap analysis before timezone alignment.

---

### 4.1 Purchase event volume and anomalies `[MATERIAL]`

**Method:** Run the query below to trend daily GA4 purchase events with the rolling-median ±3·MAD band. Read the `is_bfcm` / `flag` columns and treat `BFCM_EXPECTED` rows as seasonal; investigate the remaining `ANOMALY` rows and annotate promos and launches.

**Why it matters:** This is the baseline before reconciliation. A drop here with stable `add_to_cart` isolates the failure to the purchase tag; the BFCM spike is expected DTC seasonality and is excluded from the anomaly logic.

**Pass threshold:** All non-BFCM days within band except where an explained event accounts for the move.

```sql
-- 4.1  GA4 purchase volume per store-local day with anomaly band.
--      BFCM is flagged separately: a purchase spike there is expected DTC seasonality.
WITH d AS (
    SELECT
        TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
                TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS dt,
        COUNT(*) AS purchases
    FROM IDENTIFIER($ga4_table)
    WHERE EVENT_NAME='purchase'
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
    GROUP BY 1
),
b AS (
    SELECT d.*,
        DATEADD('day', (3 - DAYOFWEEKISO(DATE_FROM_PARTS(YEAR(dt),11,1)) + 7) % 7 + 21,
                DATE_FROM_PARTS(YEAR(dt),11,1)) AS tg
    FROM d
)
SELECT dt, purchases,
    MEDIAN(purchases) OVER (ORDER BY dt ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING) AS roll_median,
    (dt BETWEEN tg AND DATEADD('day',4,tg)) AS is_bfcm,
    CASE
        WHEN dt BETWEEN tg AND DATEADD('day',4,tg) THEN 'BFCM_EXPECTED'
        WHEN ABS(purchases - MEDIAN(purchases) OVER (ORDER BY dt ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING)) >
             3*NULLIF(MEDIAN(ABS(purchases - MEDIAN(purchases) OVER (ORDER BY dt ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING)))
               OVER (ORDER BY dt ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING),0) THEN 'ANOMALY'
        ELSE 'ok'
    END AS flag
FROM b
ORDER BY dt;
```

---

### 4.2 Purchase / transaction duplication `[MATERIAL]`

**Method:** Run the query below to group purchases by `ECOMMERCE:transaction_id`, flag transaction_ids that fire more than once across sessions or days, and separately count null and empty-string transaction_ids.

**Why it matters:** Duplicate purchases overstate revenue and conversion. The classic pattern is a thank-you page that re-fires on refresh; the fix is id-based dedup or a once-per-transaction guard.

**Pass threshold:** Duplicate `transaction_id` rate under 0.5% and null/empty rate near 0.

```sql
-- 4.2  Purchase duplication + null/empty transaction_id.
WITH tx AS (
    SELECT ECOMMERCE:transaction_id::string AS transaction_id, COUNT(*) AS fires
    FROM IDENTIFIER($ga4_table)
    WHERE EVENT_NAME='purchase'
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
    GROUP BY 1
)
SELECT
    COUNT(*)                                                        AS distinct_txn_ids,
    SUM(CASE WHEN fires > 1 THEN 1 ELSE 0 END)                     AS duplicated_txn_ids,
    SUM(CASE WHEN transaction_id IS NULL OR transaction_id=''
        THEN fires ELSE 0 END)                                     AS null_or_empty_txn_events,
    ROUND(100*duplicated_txn_ids/NULLIF(COUNT(*),0),2)             AS pct_duplicated
FROM tx;
```

---

### 4.3 Daily transaction gap vs Shopify `[MATERIAL]`

**Method:** After Domain 0 alignment, run the query below to join distinct GA4 purchase transaction_ids to Shopify order counts per store-local day (excluding test and cancelled orders) and trend the gap and GA4 capture percentage over time.

**Why it matters:** This is the headline reconciliation: it quantifies how much GA4 under- or over-counts versus the system of record. A widening gap signals tag decay or rising consent/ad-block loss.

**Pass threshold:** GA4 captures a stable, explainable share of Shopify orders (commonly 85–95%) and the gap is steady rather than drifting.

```sql
-- 4.3  Daily transaction gap vs Shopify. Excludes test and cancelled orders.
WITH ga AS (
    SELECT
        TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
                TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS dt,
        COUNT(DISTINCT ECOMMERCE:transaction_id::string) AS ga_orders
    FROM IDENTIFIER($ga4_table)
    WHERE EVENT_NAME='purchase'
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
    GROUP BY 1
),
shop AS (
    SELECT
        TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz, TO_TIMESTAMP(CREATED_AT))) AS dt,
        COUNT(*) AS shopify_orders
    FROM IDENTIFIER($shopify_table)
    WHERE COALESCE(TEST,FALSE)=FALSE AND CANCELLED_AT IS NULL
    AND TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz, TO_TIMESTAMP(CREATED_AT))) >= DATEADD('months', -24, CURRENT_DATE())
    GROUP BY 1
)
SELECT
    COALESCE(ga.dt, shop.dt)                        AS dt,
    ga_orders,
    shopify_orders,
    shopify_orders - ga_orders                      AS gap,
    ROUND(100*ga_orders/NULLIF(shopify_orders,0),1) AS ga_capture_pct
FROM ga FULL OUTER JOIN shop ON ga.dt = shop.dt
ORDER BY dt;
```

---

### 4.4 Gap directionality and classification `[MATERIAL]`

**Method:** Run the query below to left-join Shopify orders to GA4 purchases on order id and bucket the misses by `SOURCE_NAME`. Classify Shopify-only orders (POS, draft, phone, and subscription rebills that never touch the storefront) versus GA4-only fires. For subscription brands, isolate rebills explicitly — they are expected to be absent from GA4.

**Why it matters:** The gap is only actionable once decomposed into GA4-only versus Shopify-only orders. For subscription DTC, recurring rebills are the dominant Shopify-only bucket and must be removed before judging tracking health.

**Pass threshold:** The residual unexplained gap is small after removing the known structural buckets.

```sql
-- 4.4  Gap directionality: classify Shopify-only orders (rebills/POS/draft) vs GA4-only.
--      For subscription brands, SOURCE_NAME identifies channel; rebills usually not 'web'.
WITH shop AS (
    SELECT
        TO_VARCHAR(ID) AS order_id,
        SOURCE_NAME,
        CASE WHEN SOURCE_NAME NOT IN ('web','checkout_next') THEN 'shopify_only_structural'
             ELSE 'web_expected_in_ga' END AS bucket
    FROM IDENTIFIER($shopify_table)
    WHERE COALESCE(TEST,FALSE)=FALSE AND CANCELLED_AT IS NULL
    AND TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz, TO_TIMESTAMP(CREATED_AT))) >= DATEADD('months', -24, CURRENT_DATE())
),
ga AS (
    SELECT DISTINCT ECOMMERCE:transaction_id::string AS order_id
    FROM IDENTIFIER($ga4_table)
    WHERE EVENT_NAME='purchase'
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
)
SELECT
    s.SOURCE_NAME,
    s.bucket,
    COUNT(*)                                                  AS shopify_orders,
    SUM(CASE WHEN ga.order_id IS NULL THEN 1 ELSE 0 END)      AS missing_from_ga,
    ROUND(100*missing_from_ga/NULLIF(COUNT(*),0),1)           AS pct_missing
FROM shop s LEFT JOIN ga ON s.order_id = ga.order_id
GROUP BY 1,2
ORDER BY shopify_orders DESC;
```

---

### 4.5 Revenue reconciliation, not just counts `[MATERIAL]`

**Method:** Run the query below to compare daily `SUM(ECOMMERCE:purchase_revenue)` against Shopify `SUM(TOTAL_PRICE)` in the same store-local zone. Confirm explicitly whether GA4 sends gross, subtotal, or post-discount, and how tax and shipping are treated, before judging the gap.

**Why it matters:** Matching order counts while revenue diverges still breaks every margin and ROAS calculation. GA4 sending subtotal while Shopify reports gross inflates apparent discount and deflates ROAS.

**Pass threshold:** Daily revenue reconciles within a small, documented tolerance attributable to the tax/shipping rule.

```sql
-- 4.5  Revenue reconciliation per store-local day (not just counts).
WITH ga AS (
    SELECT
        TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
                TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS dt,
        SUM(ECOMMERCE:purchase_revenue::float) AS ga_revenue
    FROM IDENTIFIER($ga4_table)
    WHERE EVENT_NAME='purchase'
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
    GROUP BY 1
),
shop AS (
    SELECT
        TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz, CREATED_AT)) AS dt,
        SUM(TOTAL_PRICE::float) AS shopify_revenue
    FROM IDENTIFIER($shopify_table)
    WHERE COALESCE(TEST,FALSE)=FALSE AND CANCELLED_AT IS NULL
    AND TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz, TO_TIMESTAMP(CREATED_AT))) >= DATEADD('months', -24, CURRENT_DATE())
    GROUP BY 1
)
SELECT
    COALESCE(ga.dt,shop.dt) AS dt,
    ga_revenue,
    shopify_revenue,
    shopify_revenue - ga_revenue                    AS revenue_gap,
    ROUND(100*ga_revenue/NULLIF(shopify_revenue,0),1) AS ga_revenue_capture_pct
FROM ga FULL OUTER JOIN shop ON ga.dt=shop.dt
ORDER BY dt;
```

---

### 4.6 Currency matching per order `[HYGIENE]`

**Method:** Run the query below to join GA4 purchases to Shopify orders on order id and compare the GA4 currency parameter to Shopify `CURRENCY` per order, quantifying mismatches.

**Why it matters:** Multi-currency stores can mismatch transaction currency and corrupt revenue. Summing mixed currencies as if single-currency silently overstates or understates revenue.

**Pass threshold:** Currency matches per order with no mixed-currency summation.

```sql
-- 4.6  Currency matching per order.
WITH ga AS (
    SELECT
        ECOMMERCE:transaction_id::string AS order_id,
        (SELECT p.value:value:string_value::string FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
          WHERE p.value:key::string='currency') AS ga_currency
    FROM IDENTIFIER($ga4_table) e
    WHERE EVENT_NAME='purchase'
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
),
shop AS (
    SELECT TO_VARCHAR(ID) AS order_id, CURRENCY AS shop_currency
    FROM IDENTIFIER($shopify_table)
    WHERE COALESCE(TEST,FALSE)=FALSE
)
SELECT
    COUNT(*)                                                           AS matched_orders,
    SUM(CASE WHEN ga.ga_currency <> shop.shop_currency THEN 1 ELSE 0 END) AS currency_mismatches,
    ROUND(100*currency_mismatches/NULLIF(COUNT(*),0),2)               AS pct_mismatch
FROM ga JOIN shop ON ga.order_id=shop.order_id
WHERE ga.ga_currency IS NOT NULL;
```

---

### 4.7 Refund tracking and reconciliation `[MATERIAL]`

**Method:** Run the GA4 side of the query below to trend `refund` events per day; then uncomment and run the Shopify side (set `$shopify_refunds_table`) and reconcile counts and value.

**Why it matters:** Without refund events, GA4 revenue is gross-only and overstates net. Missing refunds mean every net-revenue and contribution-margin view built on GA4 is overstated.

**Pass threshold:** Refund events present and reconciling to Shopify refunds within tolerance.

```sql
-- 4.7  Refund tracking + reconciliation vs Shopify refunds.

-- GA4 side:
SELECT
    TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
            TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS dt,
    COUNT(*) AS ga_refund_events
FROM IDENTIFIER($ga4_table)
WHERE EVENT_NAME='refund'
GROUP BY 1
ORDER BY 1;

-- Shopify side (uncomment when $shopify_refunds_table is set):
-- SELECT TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz, CREATED_AT)) AS dt,
--        COUNT(*) AS shopify_refunds
-- FROM IDENTIFIER($shopify_refunds_table)
-- GROUP BY 1 ORDER BY 1;
```

---

### 4.8 Consistency across countries `[HYGIENE]`

**Method:** Run the query below to break GA4 purchase counts down by `GEO:country`, then join to Shopify shipping or billing country where a true per-country capture rate is needed, and flag geographies with outlier capture.

**Why it matters:** Geo-level tracking gaps hide behind healthy totals. A single market with low capture often maps to a region-specific consent banner or CMP behavior.

**Pass threshold:** Capture rate consistent across major markets.

```sql
-- 4.8  Capture consistency across countries (GEO is a VARIANT).
WITH ga AS (
    SELECT
        GEO:country::string AS country,
        COUNT(DISTINCT ECOMMERCE:transaction_id::string) AS ga_orders
    FROM IDENTIFIER($ga4_table)
    WHERE EVENT_NAME='purchase'
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
    GROUP BY 1
)
SELECT country, ga_orders
FROM ga
ORDER BY ga_orders DESC;
-- Join to Shopify shipping/billing country if you need a true per-country capture rate.
```

---

## Domain 5 — Traffic quality & filtering

> Filtering bots, internal traffic, and self-referrals — the contamination sources that distort every rate.

---

### 5.1 Bot and invalid traffic proportion `[MATERIAL]`

**Method:** Run the query below to build per-session event counts and time spans, then flag impossible-velocity sessions (many events in a few seconds) and single-event sessions, reporting the suspected-bot share over time.

**Why it matters:** Bots inflate sessions and dilute every rate. A spike often coincides with a scraping campaign or a referral-spam wave.

**Pass threshold:** Suspected-bot share low and stable.

```sql
-- 5.1  Bot / invalid traffic proxy: sessions with impossible event velocity or no engagement.
WITH s AS (
    SELECT
        USER_PSEUDO_ID,
        (SELECT p.value:value:int_value::number FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
          WHERE p.value:key::string='ga_session_id') AS ga_session_id,
        COUNT(*) AS events,
        (MAX(EVENT_TIMESTAMP)-MIN(EVENT_TIMESTAMP))/1e6 AS span_seconds
    FROM IDENTIFIER($ga4_table) e
    WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
    GROUP BY 1,2
)
SELECT
    COUNT(*)                                                           AS sessions,
    SUM(CASE WHEN events > 50 AND span_seconds < 10 THEN 1 ELSE 0 END) AS suspected_bot_velocity,
    SUM(CASE WHEN events = 1 THEN 1 ELSE 0 END)                        AS single_event_sessions,
    ROUND(100*suspected_bot_velocity/NULLIF(COUNT(*),0),2)             AS pct_suspected_bot
FROM s;
```

---

### 5.2 Internal and employee traffic `[MATERIAL]`

**Method:** Run the query below to break events down by the `traffic_type` parameter (GA4 stamps `'internal'` when an internal-traffic filter is configured) and quantify the internal share. Where no filter exists, identify internal activity via known office IPs or staging UTM patterns.

**Why it matters:** Office, employee, and QA traffic inflates engagement and can fire test purchases. Unfiltered internal traffic both inflates engagement and pollutes the GA4-only purchase bucket in check 4.4.

**Pass threshold:** Internal traffic filtered or flagged, with near-zero internal test purchases in production data.

```sql
-- 5.2  Internal traffic (GA4 sets traffic_type='internal' via filter, surfaced in EVENT_PARAMS).
SELECT
    COALESCE((SELECT p.value:value:string_value::string FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
        WHERE p.value:key::string='traffic_type'),'(none)') AS traffic_type,
    COUNT(*) AS events
FROM IDENTIFIER($ga4_table) e
WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1
ORDER BY events DESC;
```

---

### 5.4 Self-referrals and referral exclusions `[HYGIENE]`

**Method:** Run the query below to list the top `page_referrer` values on `session_start` and scan for the store's own domains and any payment, subscription, or CMP subdomains. Confirm those domains are covered by referral exclusions.

**Why it matters:** Payment and subscription subdomains appearing as referrers fragment sessions and steal attribution. A first-party domain in the referrer list is the cross-domain breakage from check 1.4 surfacing as misattribution.

**Pass threshold:** No first-party domains appearing as acquisition referrers.

```sql
-- 5.4  Self-referrals: own domains / payment / subscription subdomains appearing as referrer.
SELECT
    COALESCE((SELECT p.value:value:string_value::string FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
        WHERE p.value:key::string='page_referrer'),'(none)') AS referrer,
    COUNT(*) AS events
FROM IDENTIFIER($ga4_table) e
WHERE EVENT_NAME='session_start'
AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1
ORDER BY events DESC
LIMIT 100;
```

---

## Domain 6 — Attribution & channels

> The most CFO-legible domain — where *"attribution is unknown, margins shrinking despite spend"* gets proven with numbers.

---

### 6.1 Channel composition over time `[MATERIAL]`

**Method:** Run the query below to derive each session's channel from `COLLECTED_TRAFFIC_SOURCE` (falling back to `TRAFFIC_SOURCE`) source/medium, map it into channel groups, and trend session counts per channel over time. Watch for unexplained mix shifts.

**Why it matters:** This is the baseline of where traffic and revenue are attributed. An unexplained mix shift often follows a UTM or redirect change rather than a real demand shift.

**Pass threshold:** Stable, explainable channel mix where shifts map to known campaign changes.

```sql
-- 6.1  Channel composition over time.
WITH s AS (
    SELECT
        TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
                TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS dt,
        LOWER(COALESCE(e.COLLECTED_TRAFFIC_SOURCE:manual_source::string,
                       e.TRAFFIC_SOURCE:source::string,'(direct)')) AS source,
        LOWER(COALESCE(e.COLLECTED_TRAFFIC_SOURCE:manual_medium::string,
                       e.TRAFFIC_SOURCE:medium::string,'(none)'))   AS medium
    FROM IDENTIFIER($ga4_table) e
    WHERE EVENT_NAME='session_start'
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
)
SELECT dt,
    CASE
        WHEN medium IN ('cpc','ppc','paid','paidsearch')          THEN 'Paid Search'
        WHEN medium IN ('email')                                   THEN 'Email'
        WHEN medium IN ('social','paid_social','cpc_social')       THEN 'Social'
        WHEN medium IN ('referral')                                THEN 'Referral'
        WHEN medium IN ('organic')                                 THEN 'Organic Search'
        WHEN source='(direct)' OR medium IN ('(none)','(not set)') THEN 'Direct'
        ELSE 'Other/Unassigned'
    END AS channel,
    COUNT(*) AS sessions
FROM s
GROUP BY 1,2
ORDER BY dt, sessions DESC;
```

---

### 6.2 Direct and unassigned traffic share `[MATERIAL]`

**Method:** Run the query below to trend the percentage of sessions classified as Direct or unassigned over time, split by `GEO:country`, and cross-check against expected paid and email volume.

**Why it matters:** This is the headline attribution finding: inflated Direct/unassigned equals wasted or misattributed spend the CFO is paying for blind. Direct above roughly 20–25% usually means broken UTMs, missing tagging on email or paid, or app-to-web handoff loss — paid traffic landing in Direct is spend whose return is invisible.

**Pass threshold:** Direct share within benchmark (commonly under ~20% for ecommerce) with low unassigned.

```sql
-- 6.2  Direct + unassigned share over time, split by country.
WITH s AS (
    SELECT
        TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
                TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS dt,
        e.GEO:country::string AS country,
        LOWER(COALESCE(e.COLLECTED_TRAFFIC_SOURCE:manual_source::string,
                       e.TRAFFIC_SOURCE:source::string,'(direct)')) AS source,
        LOWER(COALESCE(e.COLLECTED_TRAFFIC_SOURCE:manual_medium::string,
                       e.TRAFFIC_SOURCE:medium::string,'(none)'))   AS medium
    FROM IDENTIFIER($ga4_table) e
    WHERE EVENT_NAME='session_start'
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
)
SELECT dt, country,
    COUNT(*) AS sessions,
    ROUND(100*AVG(CASE WHEN source='(direct)' OR medium IN ('(none)','(not set)')
                       THEN 1 ELSE 0 END),1) AS pct_direct_unassigned
FROM s
GROUP BY 1,2
ORDER BY dt, sessions DESC;
```

---

### 6.3 UTM hygiene `[MATERIAL]`

**Method:** Run the query below to group `session_start` by the source and medium parameters and inspect for casing inconsistency (`facebook` vs `Facebook` vs `FB`), missing medium, free-text sprawl, and paid links arriving with a gclid/fbclid but medium `(none)`. Quantify untagged paid and email sessions.

**Why it matters:** Inconsistent UTMs are the usual root cause of inflated Direct and broken channel grouping. Mixed casing fragments one channel into several and feeds the Direct/unassigned bucket.

**Pass threshold:** UTM taxonomy consistent, with paid and email traffic reliably tagged.

```sql
-- 6.3  UTM hygiene: casing inconsistency + paid traffic missing medium.
SELECT
    (SELECT p.value:value:string_value::string FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
        WHERE p.value:key::string='source') AS utm_source,
    (SELECT p.value:value:string_value::string FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
        WHERE p.value:key::string='medium') AS utm_medium,
    COUNT(*) AS events
FROM IDENTIFIER($ga4_table) e
WHERE EVENT_NAME='session_start'
AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1,2
ORDER BY events DESC;
-- Inspect for facebook vs Facebook vs FB, and gclid/fbclid landing with medium '(none)'.
```

---

### 6.4 Channel grouping re-derivation `[HYGIENE]`

**Method:** Run the query below to compare the medium from `COLLECTED_TRAFFIC_SOURCE` (manual UTMs) against the GA4-attributed medium in `TRAFFIC_SOURCE` and quantify how many sessions would be reclassified between the two.

**Why it matters:** The export's default attribution can disagree with the client's mental model of their channels. Large reclassification means dashboards using the default grouping tell a different story than the client believes.

**Pass threshold:** Re-derived grouping matches the default within tolerance, or the differences are understood and documented.

```sql
-- 6.4  Channel re-derivation vs collected source (reclassification volume).
SELECT
    LOWER(COALESCE(COLLECTED_TRAFFIC_SOURCE:manual_medium::string,'(none)')) AS collected_medium,
    LOWER(COALESCE(TRAFFIC_SOURCE:medium::string,'(none)'))                  AS attributed_medium,
    COUNT(*) AS events
FROM IDENTIFIER($ga4_table)
WHERE EVENT_NAME='session_start'
AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1,2
ORDER BY events DESC;
```

---

## Domain 7 — Consent, privacy & PII

> Weighted light for mostly-US/CA traffic (CCPA/CPRA and Global Privacy Control rather than full Consent Mode v2 modeling), but the PII check is non-negotiable everywhere — it is a compliance liability, not just a data-quality issue.

---

### 7.1 PII in event parameters `[BLOCKER]`

**Method:** Run the query below to scan `page_location` for email-pattern and phone-pattern matches and count affected events. Treat any non-zero result as a finding to remediate, not merely report.

**Why it matters:** Email, phone, or names in `page_location`, `page_title`, or custom params is a direct privacy and compliance liability and can violate Google's terms, risking data deletion. It typically comes from forms submitting via GET or account pages passing identifiers in the query string.

**Pass threshold:** Zero PII detected in any parameter.

```sql
-- 7.1  PII in parameters: email/phone patterns in page_location (BLOCKER if any).
WITH locs AS (
    SELECT (SELECT p.value:value:string_value::string FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
              WHERE p.value:key::string='page_location') AS page_location
    FROM IDENTIFIER($ga4_table) e
    WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
)
SELECT
    COUNT(*)                                                                AS rows_checked,
    SUM(CASE WHEN page_location RLIKE '.*[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}.*'
        THEN 1 ELSE 0 END)                                                 AS email_in_url,
    SUM(CASE WHEN page_location RLIKE '.*(phone|tel)=.*[0-9]{7,}.*'
        THEN 1 ELSE 0 END)                                                 AS phone_in_url
FROM locs;
```

---

### 7.2 Consent signal coverage (US/CA) `[MATERIAL]`

**Method:** Run the query below to break events down by `PRIVACY_INFO` `analytics_storage` and `ads_storage` values and quantify the denied/opted-out share. Confirm the CMP is wired to gate collection where required.

**Why it matters:** For US/CA the relevant signal is CCPA/CPRA opt-out and Global Privacy Control rather than EU Consent Mode, but it still affects how many events are collected. A rising opt-out share partly explains a widening GA4-vs-Shopify gap in check 4.3.

**Pass threshold:** Consent signal present and behaving, with opt-out share within the expected range.

```sql
-- 7.2  Consent signal coverage (PRIVACY_INFO VARIANT; US opt-out / analytics_storage).
SELECT
    PRIVACY_INFO:analytics_storage::string    AS analytics_storage,
    PRIVACY_INFO:ads_storage::string          AS ads_storage,
    PRIVACY_INFO:uses_transient_token::string AS transient_token,
    COUNT(*) AS events
FROM IDENTIFIER($ga4_table)
WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1,2,3
ORDER BY events DESC;
```

---

### 7.3 Geo-resolution completeness `[HYGIENE]`

**Method:** Run the query below to measure the populated rate of `GEO:country` and `GEO:region` across all events.

**Why it matters:** Missing geo breaks the country splits used throughout Domains 4 and 6. High null geo undermines every country-split reconciliation elsewhere in this audit.

**Pass threshold:** Geo populated on the large majority of sessions.

```sql
-- 7.3  Geo-resolution completeness.
SELECT
    ROUND(100*AVG(CASE WHEN GEO:country::string IS NOT NULL AND GEO:country::string<>'' THEN 1 ELSE 0 END),1) AS pct_has_country,
    ROUND(100*AVG(CASE WHEN GEO:region::string  IS NOT NULL AND GEO:region::string<>''  THEN 1 ELSE 0 END),1) AS pct_has_region
FROM IDENTIFIER($ga4_table)
WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE());
```

---

## Deliverable structure

When this methodology is run for a client, the output document follows this structure:

1. **Executive summary** — findings ordered by severity and estimated $ impact, not by domain. Lead with marketing-efficiency / profit-leak findings (Domains 4 and 6).
2. **Scope & data sources** — properties, date range, tables, timezone decisions from Domain 0.
3. **Domain-by-domain findings** — each check with its computed value, pass/fail vs threshold, and a one-line interpretation.
4. **Reconciliation appendix** — the GA4-vs-Shopify numbers in full.
5. **Prioritized remediation roadmap** — blockers first, then material, then hygiene; each with owner, effort, and expected impact.

### Scoring rollup (optional)

Weight findings by severity to produce a comparable trust score across clients and re-audits:

| Severity | Weight |
|---|---|
| `[BLOCKER]` fail | −20 pts each |
| `[MATERIAL]` fail | −5 pts each |
| `[HYGIENE]` fail | −1 pt each |

**Overall trust score** = 100 − (20 × BLOCKER count) − (5 × MATERIAL count) − (1 × HYGIENE count), capped at 0, max 100. Report a per-domain score and an overall score so progress is measurable on re-audit.
