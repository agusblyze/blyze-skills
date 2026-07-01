# GA4 Data Audit Skill

## When to use this skill
Use this skill whenever the user asks to run, perform, conduct, or scope a GA4 data audit against a Snowflake warehouse. Triggers include: "run the GA4 audit", "audit the GA4 data", "check GA4 data quality", "validate GA4 tracking", "start the audit for [client]".

---

## What this skill does
It guides you through a structured, domain-by-domain GA4 data audit against a Snowflake warehouse. You will:
1. Collect the four required connection parameters from the user.
2. Run each check in domain order by generating and executing the appropriate Snowflake SQL.
3. Interpret every result against its pass threshold and classify it as PASS / FAIL-BLOCKER / FAIL-MATERIAL / FAIL-HYGIENE.
4. After all checks, produce a structured audit report following the deliverable format below.

---

## Step 0 — Collect parameters before running anything

Ask the user for these four values. Do not proceed until all four are confirmed.

```
ga4_table               (default: RAW.BIGQUERY.GA4__EVENTS)
shopify_table           (default: RAW.SHOPIFY.ORDERS)
shopify_customers_table (default: RAW.SHOPIFY.CUSTOMERS)
store_tz                (default: America/New_York)
```

Optional — ask only if the user wants refund reconciliation:
```
shopify_refunds_table   (default: RAW.SHOPIFY.ORDER_REFUNDS)
```

Once confirmed, prepend every SQL session with these SET statements:

```sql
SET ga4_table               = '<ga4_table>';
SET shopify_table           = '<shopify_table>';
SET shopify_customers_table = '<shopify_customers_table>';
SET shopify_refunds_table   = '<shopify_refunds_table>';   -- omit if not provided
SET store_tz                = '<store_tz>';
```

All queries use `IDENTIFIER($ga4_table)`, `IDENTIFIER($shopify_table)`, and `$store_tz` — never hardcode table names or timezone strings in the query bodies.

---

## Step 1 — Run domains in order

Run checks in sequence: Domain 0 → 1 → 2 → 3 → 4 → 5 → 6 → 7.

**Domain 0 must complete before Domain 4.** If any Domain 0 check returns FAIL-BLOCKER, surface it immediately and pause before continuing to reconciliation checks.

For each check:
- State the check id and title.
- Run the query.
- Read the result and compare it to the pass threshold.
- Assign a status: **PASS**, **FAIL-BLOCKER**, **FAIL-MATERIAL**, or **FAIL-HYGIENE**.
- Write one sentence stating what the result means for this client.

---

## Schema conventions (apply to every query)

- `EVENT_TIMESTAMP` is microseconds UTC. Store-local day = `TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz, TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)))`. Use this everywhere; never use the raw `EVENT_DATE` string.
- `EVENT_PARAMS` is an array of `{key, value:{string_value, int_value, double_value, float_value}}`. Extract a param with `LATERAL FLATTEN` filtered on `key`, then `COALESCE` the value subtypes.
- Items live in `ECOMMERCE:items` (no top-level ITEMS column). Transaction id: `ECOMMERCE:transaction_id`. Revenue: `ECOMMERCE:purchase_revenue`.
- `__UPDATED_AT` is the connector load timestamp — used for late-arrival checks.
- Shopify columns (Fivetran standard): `ID, CREATED_AT (UTC), TOTAL_PRICE, CURRENCY, FINANCIAL_STATUS, SOURCE_NAME, TEST, CANCELLED_AT`.
- All queries filter to the last 24 months with `DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())`.
- **BFCM**: the rolling-median ±3·MAD anomaly band flags the Thanksgiving-through-Cyber-Monday window as `BFCM_EXPECTED`. Do not treat these as anomalies — note them as expected DTC seasonality.

---

## Domain 0 — Foundations (run first)

### 0.1 Timezone alignment between GA4 and Shopify [BLOCKER]
**Method:** Derive GA4 event date with `TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz, TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)))`. Convert Shopify `CREATED_AT` with `CONVERT_TIMEZONE('UTC', $store_tz, CREATED_AT)`. Count events landing on a different calendar day under the two definitions. Confirm both daily grains share one zone before any reconciliation.
**Pass threshold:** No systematic single-day-shift pattern in the gap series.

```sql
SELECT
    EVENT_DATE                                   AS raw_event_date_string,
    TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
            TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS event_date_et,
    COUNT(*)                                      AS events,
    SUM(CASE WHEN EVENT_DATE <> TO_CHAR(TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
            TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))),'YYYYMMDD') THEN 1 ELSE 0 END) AS boundary_shifted_events
FROM IDENTIFIER($ga4_table) e
WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1,2
ORDER BY 2 DESC
LIMIT 200;
```

**Interpret:** Non-zero `boundary_shifted_events` is the timezone misalignment signal. A sawtooth gap (GA4 high one day / low the next) means Domain 4 counts are unreliable until fixed.

---

### 0.2 Sync freshness and partial-day mechanics [BLOCKER]
**Method:** Per store-local event day, compute the lag between `MAX(EVENT_TIMESTAMP)` and `MAX(__UPDATED_AT)`. Identify trailing days still partially synced and flag them for exclusion in trend analysis.
**Pass threshold:** No recurring drop on the latest 1–3 days.

```sql
SELECT
    TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
            TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)))      AS event_date_et,
    MIN(__UPDATED_AT)                                  AS first_loaded_at,
    MAX(__UPDATED_AT)                                  AS last_loaded_at,
    DATEDIFF('hour',
        TO_TIMESTAMP_NTZ(MAX(EVENT_TIMESTAMP),6),
        MAX(__UPDATED_AT))                             AS load_lag_hours,
    COUNT(*)                                           AS events
FROM IDENTIFIER($ga4_table)
WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1
ORDER BY 1 DESC;
```

**Interpret:** A consistent drop on only the most recent days is sync latency, not tracking failure. Flag those days in all subsequent trend queries.

---

### 0.3 Late-arriving and reprocessed events [MATERIAL]
**Method:** Per `EVENT_NAME`, count the share of events where `__UPDATED_AT` is more than 24h and more than 72h after the event occurred. Focus on the `purchase` row.
**Pass threshold:** Late arrivals settle to <1–2% change after 72h for web.

```sql
SELECT
    EVENT_NAME,
    COUNT(*)                                                          AS total_events,
    SUM(CASE WHEN DATEDIFF('hour',
            TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6), __UPDATED_AT) > 24
        THEN 1 ELSE 0 END)                                           AS arrived_gt_24h,
    SUM(CASE WHEN DATEDIFF('hour',
            TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6), __UPDATED_AT) > 72
        THEN 1 ELSE 0 END)                                           AS arrived_gt_72h,
    ROUND(100 * arrived_gt_72h / NULLIF(total_events,0), 2)          AS pct_gt_72h
FROM IDENTIFIER($ga4_table)
WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1
ORDER BY total_events DESC;
```

**Interpret:** High `pct_gt_72h` on `purchase` means revenue dashboards built on fresh data understate recent performance.

---

### 0.4 Canonical dedup keys defined [BLOCKER]
**Method:** Confirm the event-identity grain `(USER_PSEUDO_ID, ga_session_id, EVENT_NAME, EVENT_TIMESTAMP, EVENT_BUNDLE_SEQUENCE_ID)` is actually unique. Quantify duplicate rows.
**Pass threshold:** Duplicate-row rate near 0.

```sql
WITH base AS (
    SELECT
        USER_PSEUDO_ID,
        (SELECT p.value:value:int_value::number
           FROM LATERAL FLATTEN(input => e.EVENT_PARAMS) p
          WHERE p.value:key::string = 'ga_session_id')      AS ga_session_id,
        EVENT_NAME, EVENT_TIMESTAMP, EVENT_BUNDLE_SEQUENCE_ID
    FROM IDENTIFIER($ga4_table) e
    WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
)
SELECT
    COUNT(*)                                                 AS rows_total,
    COUNT(*) - COUNT(DISTINCT USER_PSEUDO_ID||':'||ga_session_id||':'||EVENT_NAME||':'
              ||EVENT_TIMESTAMP||':'||EVENT_BUNDLE_SEQUENCE_ID) AS duplicate_rows,
    ROUND(100 * duplicate_rows / NULLIF(rows_total,0), 3)    AS pct_duplicate_rows
FROM base;
```

**Interpret:** Any meaningful duplicate rate must be resolved before dedup-dependent checks (2.x, 4.x) can be trusted.

---

## Domain 1 — Volume & completeness

### 1.1 Total event volume by day [MATERIAL]
**Method:** Count events per store-local day. Overlay rolling-median ±3·MAD band. Read `is_bfcm` / `flag` columns — `BFCM_EXPECTED` rows are seasonal and must not be flagged as anomalies.
**Pass threshold:** No unexplained zero days; all non-BFCM days within the ±3·MAD band.

```sql
WITH daily AS (
    SELECT TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
                TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS d,
           COUNT(*) AS events
    FROM IDENTIFIER($ga4_table)
    WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
    GROUP BY 1
),
flagged AS (
    SELECT d, events,
           DATEADD('day', (3 - DAYOFWEEKISO(DATE_FROM_PARTS(YEAR(d),11,1)) + 7) % 7 + 21,
                   DATE_FROM_PARTS(YEAR(d),11,1))                       AS thanksgiving,
           MEDIAN(events) OVER (ORDER BY d ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING)  AS roll_median,
           MEDIAN(ABS(events - MEDIAN(events) OVER (ORDER BY d ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING)))
               OVER (ORDER BY d ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING)             AS mad
    FROM daily
)
SELECT d, events, roll_median, mad,
       (d BETWEEN thanksgiving AND DATEADD('day',4,thanksgiving))       AS is_bfcm,
       CASE WHEN d BETWEEN thanksgiving AND DATEADD('day',4,thanksgiving) THEN 'BFCM_EXPECTED'
            WHEN mad > 0 AND ABS(events - roll_median) > 3*mad          THEN 'ANOMALY'
            ELSE 'ok' END                                               AS flag
FROM flagged
ORDER BY d;
```

---

### 1.2 Volume split by stream and platform [MATERIAL]
**Method:** Group counts by `STREAM_ID` and `PLATFORM`. Trend each stream independently.
**Pass threshold:** Each active stream is individually continuous with no silent dropout.

```sql
SELECT TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
            TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS d,
       STREAM_ID, PLATFORM, COUNT(*) AS events
FROM IDENTIFIER($ga4_table)
WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1,2,3
ORDER BY 1 DESC, events DESC;
```

---

### 1.3 page_view completeness and single-fire [MATERIAL]
**Method:** (a) Trend `page_view` per day. (b) Dedup on `(USER_PSEUDO_ID, ga_session_id, page_location)` within 2 seconds and quantify the double-fire rate.
**Pass threshold:** Duplicate `page_view` rate under 1%.

```sql
-- 1.3a: daily page_view trend
SELECT TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
            TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS d,
       COUNT(*) AS page_views
FROM IDENTIFIER($ga4_table)
WHERE EVENT_NAME = 'page_view'
AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1 ORDER BY 1;

-- 1.3b: double-fire detection
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
    COUNT(*)                                                        AS page_views,
    SUM(CASE WHEN micros_since_prev < 2000000 THEN 1 ELSE 0 END)   AS likely_double_fires,
    ROUND(100 * likely_double_fires / NULLIF(COUNT(*),0), 2)        AS pct_double_fire
FROM flagged;
```

---

### 1.4 session_start single-fire and session integrity [MATERIAL]
**Method:** Count `session_start` per `(USER_PSEUDO_ID, ga_session_id)` — must be exactly one.
**Pass threshold:** Zero sessions with `session_start_count > 1`.

```sql
WITH ss AS (
    SELECT USER_PSEUDO_ID,
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

### 1.5 Event naming, reserved collisions, and cardinality [HYGIENE]
**Method:** (a) Flag names not matching lowercase snake_case or exceeding 40 chars. (b) Count distinct event names vs the 500 cap. (c) Confirm error/failure events exist.
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
FROM IDENTIFIER($ga4_table)
WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1
ORDER BY events DESC;

-- 1.5b: cardinality vs 500 cap
SELECT COUNT(DISTINCT EVENT_NAME) AS distinct_events,
       CASE WHEN COUNT(DISTINCT EVENT_NAME) > 450 THEN 'NEAR_CAP' ELSE 'ok' END AS flag
FROM IDENTIFIER($ga4_table)
WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE());

-- 1.5c: error/failure event presence
SELECT EVENT_NAME, COUNT(*) AS events
FROM IDENTIFIER($ga4_table)
WHERE (EVENT_NAME ILIKE '%error%' OR EVENT_NAME ILIKE '%fail%'
    OR EVENT_NAME IN ('form_error','payment_failed'))
AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1 ORDER BY events DESC;
```

---

### 1.6 Parameter coverage and null rates [HYGIENE]
**Method:** Per key event, compute populated rate of `page_location` and `ga_session_id`.
**Pass threshold:** Critical params populated >98% on the events that use them.

```sql
SELECT
    EVENT_NAME,
    COUNT(*) AS events,
    ROUND(100*AVG(CASE WHEN (SELECT p.value:value:string_value::string
        FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
        WHERE p.value:key::string='page_location') IS NOT NULL THEN 1 ELSE 0 END),1) AS pct_has_page_location,
    ROUND(100*AVG(CASE WHEN (SELECT p.value:value:int_value::number
        FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
        WHERE p.value:key::string='ga_session_id') IS NOT NULL THEN 1 ELSE 0 END),1) AS pct_has_session_id
FROM IDENTIFIER($ga4_table) e
WHERE EVENT_NAME IN ('page_view','add_to_cart','begin_checkout','purchase','view_item')
AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1
ORDER BY events DESC;
```

---

### 1.7 Events firing per page [HYGIENE]
**Method:** Group event counts by stripped page path and event name. Compare each template's event set against what it should fire.
**Pass threshold:** No template silently missing `add_to_cart` or `begin_checkout`.

```sql
SELECT
    REGEXP_REPLACE(
        (SELECT p.value:value:string_value::string FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
          WHERE p.value:key::string='page_location'),
        '\\?.*$','')                                  AS page_path,
    EVENT_NAME,
    COUNT(*)                                          AS events
FROM IDENTIFIER($ga4_table) e
WHERE EVENT_NAME IN ('page_view','view_item','add_to_cart','begin_checkout','purchase')
AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1,2
ORDER BY events DESC
LIMIT 200;
```

---

### 1.8 Anomaly detection on core metrics [MATERIAL]
**Method:** Compute daily transactions, users, sessions, page_views. Apply rolling-median ±3·MAD to transactions. Treat `BFCM_EXPECTED` rows as seasonal.
**Pass threshold:** All four series within band on non-BFCM days.

```sql
WITH d AS (
    SELECT TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
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
       CASE WHEN dt BETWEEN tg AND DATEADD('day',4,tg) THEN 'BFCM_EXPECTED'
            WHEN ABS(transactions - MEDIAN(transactions) OVER (ORDER BY dt ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING))
                 > 3*NULLIF(MEDIAN(ABS(transactions - MEDIAN(transactions) OVER (ORDER BY dt ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING)))
                   OVER (ORDER BY dt ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING),0) THEN 'TXN_ANOMALY'
            ELSE 'ok' END AS txn_flag
FROM (SELECT d.*,
             DATEADD('day', (3 - DAYOFWEEKISO(DATE_FROM_PARTS(YEAR(dt),11,1)) + 7) % 7 + 21,
                     DATE_FROM_PARTS(YEAR(dt),11,1)) AS tg
      FROM d) d
ORDER BY dt;
```

---

## Domain 2 — Identity & sessions

### 2.1 user_id implementation post-login [MATERIAL]
**Method:** Trend the share of sessions carrying a non-null `USER_ID` per day.
**Pass threshold:** `USER_ID` present on the large majority of post-login sessions, stable trend.

```sql
SELECT
    TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
        TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)))                       AS d,
    COUNT(DISTINCT USER_PSEUDO_ID||':'||
        (SELECT p.value:value:int_value::number FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
          WHERE p.value:key::string='ga_session_id'))               AS sessions,
    COUNT(DISTINCT CASE WHEN USER_ID IS NOT NULL THEN USER_PSEUDO_ID||':'||
        (SELECT p.value:value:int_value::number FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
          WHERE p.value:key::string='ga_session_id') END)           AS sessions_with_user_id,
    ROUND(100*sessions_with_user_id/NULLIF(sessions,0),1)           AS pct_sessions_with_user_id
FROM IDENTIFIER($ga4_table) e
WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1 ORDER BY 1;
```

---

### 2.2 user_id ↔ Shopify customer_id join coverage [BLOCKER for LTV]
**Method:** Join distinct GA4 `USER_ID` to Shopify customer IDs. Confirm the join contract (raw id vs hashed email) before trusting the rate.
**Pass threshold:** Match rate sufficient for cohort analysis (set per client, document the threshold).

```sql
WITH ga AS (
    SELECT DISTINCT USER_ID FROM IDENTIFIER($ga4_table)
    WHERE USER_ID IS NOT NULL
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
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

### 2.3 user_pseudo_id stability and inflation [MATERIAL]
**Method:** Count distinct `USER_PSEUDO_ID` per logged-in `USER_ID`. Report average, median, and worst case.
**Pass threshold:** Pseudo-id inflation within expected range for the consent/browser mix.

```sql
WITH m AS (
    SELECT USER_ID, COUNT(DISTINCT USER_PSEUDO_ID) AS pseudo_ids
    FROM IDENTIFIER($ga4_table)
    WHERE USER_ID IS NOT NULL
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
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

---

## Domain 3 — Ecommerce funnel & item-level

### 3.1 Funnel event presence and healthy ratios [MATERIAL]
**Method:** Count each funnel event and express each as a % of `view_item`. Confirm every step exists and step-to-step ratios are plausible.
**Pass threshold:** All steps present, ratios plausible, no step at effectively zero.

```sql
WITH f AS (
    SELECT EVENT_NAME, COUNT(*) AS events
    FROM IDENTIFIER($ga4_table)
    WHERE EVENT_NAME IN ('view_item_list','view_item','add_to_cart','begin_checkout',
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

### 3.2 Item-detail completeness across ecommerce events [MATERIAL]
**Method:** Flatten `ECOMMERCE:items` on key events. Compute populated rate of `item_id`, `item_name`, `item_category`, `item_brand`, `price`, `quantity`.
**Pass threshold:** Core item fields populated >98% on every ecommerce event.

```sql
WITH items AS (
    SELECT e.EVENT_NAME, i.value AS item
    FROM IDENTIFIER($ga4_table) e,
         LATERAL FLATTEN(input => e.ECOMMERCE:items) i
    WHERE e.EVENT_NAME IN ('view_item','add_to_cart','begin_checkout','purchase')
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
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

---

### 3.3 Purchase events contain item-level data [MATERIAL]
**Method:** Count purchases where `ECOMMERCE:items` is null or empty, as a share of all purchases.
**Pass threshold:** Effectively 0% purchases with an empty items array.

```sql
SELECT
    COUNT(*)                                                            AS purchases,
    SUM(CASE WHEN ECOMMERCE:items IS NULL
              OR ARRAY_SIZE(ECOMMERCE:items) = 0 THEN 1 ELSE 0 END)   AS purchases_no_items,
    ROUND(100*purchases_no_items/NULLIF(COUNT(*),0),2)                 AS pct_no_items
FROM IDENTIFIER($ga4_table)
WHERE EVENT_NAME='purchase'
AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE());
```

---

### 3.4 Item value reconciliation [HYGIENE]
**Method:** Per purchase, compare `SUM(item price × quantity)` to `ECOMMERCE:purchase_revenue`. Quantify mismatches beyond $0.01.
**Pass threshold:** Item totals reconcile to event value within the documented tax/shipping rule.

```sql
WITH p AS (
    SELECT
        e.ECOMMERCE:transaction_id::string                                   AS transaction_id,
        e.ECOMMERCE:purchase_revenue::float                                  AS event_value,
        (SELECT SUM(i.value:price::float * COALESCE(i.value:quantity::float,1))
           FROM LATERAL FLATTEN(input => e.ECOMMERCE:items) i)               AS item_value
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
> **Requires Domain 0 to be complete.** Do not run this domain if check 0.1 returned FAIL-BLOCKER.

### 4.1 Purchase event volume and anomalies [MATERIAL]
**Method:** Trend daily GA4 purchases with rolling-median ±3·MAD band. Treat `BFCM_EXPECTED` rows as seasonal.
**Pass threshold:** All non-BFCM days within band.

```sql
WITH d AS (
    SELECT TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
                TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS dt, COUNT(*) AS purchases
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
       CASE WHEN dt BETWEEN tg AND DATEADD('day',4,tg) THEN 'BFCM_EXPECTED'
            WHEN ABS(purchases - MEDIAN(purchases) OVER (ORDER BY dt ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING)) >
                 3*NULLIF(MEDIAN(ABS(purchases - MEDIAN(purchases) OVER (ORDER BY dt ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING)))
                   OVER (ORDER BY dt ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING),0) THEN 'ANOMALY'
            ELSE 'ok' END AS flag
FROM b ORDER BY dt;
```

---

### 4.2 Purchase / transaction duplication [MATERIAL]
**Method:** Group by `ECOMMERCE:transaction_id`. Flag ids firing more than once; count nulls/empty strings.
**Pass threshold:** Duplicate rate <0.5%; null/empty rate near 0.

```sql
WITH tx AS (
    SELECT ECOMMERCE:transaction_id::string AS transaction_id, COUNT(*) AS fires
    FROM IDENTIFIER($ga4_table)
    WHERE EVENT_NAME='purchase'
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
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

### 4.3 Daily transaction gap vs Shopify [MATERIAL]
**Method:** Join GA4 distinct purchase transaction_ids to Shopify order counts per store-local day (exclude test and cancelled). Trend gap and capture %.
**Pass threshold:** GA4 captures 85–95% of Shopify orders; gap stable, not drifting.

```sql
WITH ga AS (
    SELECT TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
                TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS dt,
           COUNT(DISTINCT ECOMMERCE:transaction_id::string) AS ga_orders
    FROM IDENTIFIER($ga4_table)
    WHERE EVENT_NAME='purchase'
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
    GROUP BY 1
),
shop AS (
    SELECT TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz, TO_TIMESTAMP(CREATED_AT))) AS dt,
           COUNT(*) AS shopify_orders
    FROM IDENTIFIER($shopify_table)
    WHERE COALESCE(TEST,FALSE)=FALSE AND CANCELLED_AT IS NULL
    AND TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz, TO_TIMESTAMP(CREATED_AT))) >= DATEADD('months', -24, CURRENT_DATE())
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

---

### 4.4 Gap directionality and classification [MATERIAL]
**Method:** Left-join Shopify orders to GA4 purchases on order id. Bucket misses by `SOURCE_NAME`. For subscription brands, isolate rebills (expected to be absent from GA4).
**Pass threshold:** Residual unexplained gap small after removing structural buckets.

```sql
WITH shop AS (
    SELECT TO_VARCHAR(ID) AS order_id,
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
    s.SOURCE_NAME, s.bucket,
    COUNT(*)                                                    AS shopify_orders,
    SUM(CASE WHEN ga.order_id IS NULL THEN 1 ELSE 0 END)        AS missing_from_ga,
    ROUND(100*missing_from_ga/NULLIF(COUNT(*),0),1)             AS pct_missing
FROM shop s LEFT JOIN ga ON s.order_id = ga.order_id
GROUP BY 1,2
ORDER BY shopify_orders DESC;
```

---

### 4.5 Revenue reconciliation [MATERIAL]
**Method:** Compare daily `SUM(ECOMMERCE:purchase_revenue)` vs Shopify `SUM(TOTAL_PRICE)`. Confirm gross/subtotal/discount handling before judging the gap.
**Pass threshold:** Revenue reconciles within a documented tolerance attributable to tax/shipping rules.

```sql
WITH ga AS (
    SELECT TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
                TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS dt,
           SUM(ECOMMERCE:purchase_revenue::float) AS ga_revenue
    FROM IDENTIFIER($ga4_table)
    WHERE EVENT_NAME='purchase'
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
    GROUP BY 1
),
shop AS (
    SELECT TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz, CREATED_AT)) AS dt,
           SUM(TOTAL_PRICE::float) AS shopify_revenue
    FROM IDENTIFIER($shopify_table)
    WHERE COALESCE(TEST,FALSE)=FALSE AND CANCELLED_AT IS NULL
    AND TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz, TO_TIMESTAMP(CREATED_AT))) >= DATEADD('months', -24, CURRENT_DATE())
    GROUP BY 1
)
SELECT COALESCE(ga.dt,shop.dt) AS dt, ga_revenue, shopify_revenue,
       shopify_revenue - ga_revenue                              AS revenue_gap,
       ROUND(100*ga_revenue/NULLIF(shopify_revenue,0),1)         AS ga_revenue_capture_pct
FROM ga FULL OUTER JOIN shop ON ga.dt=shop.dt
ORDER BY dt;
```

---

### 4.6 Currency matching per order [HYGIENE]
**Method:** Join GA4 purchases to Shopify orders on order id. Compare GA4 currency param to Shopify `CURRENCY`.
**Pass threshold:** Currency matches per order; no mixed-currency summation.

```sql
WITH ga AS (
    SELECT ECOMMERCE:transaction_id::string AS order_id,
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
    COUNT(*)                                                       AS matched_orders,
    SUM(CASE WHEN ga.ga_currency <> shop.shop_currency THEN 1 ELSE 0 END) AS currency_mismatches,
    ROUND(100*currency_mismatches/NULLIF(COUNT(*),0),2)            AS pct_mismatch
FROM ga JOIN shop ON ga.order_id=shop.order_id
WHERE ga.ga_currency IS NOT NULL;
```

---

### 4.7 Refund tracking and reconciliation [MATERIAL]
**Method:** Trend GA4 `refund` events per day. If `$shopify_refunds_table` is set, reconcile counts and value.
**Pass threshold:** Refund events present and reconciling to Shopify refunds within tolerance.

```sql
-- GA4 side
SELECT TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
            TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS dt,
       COUNT(*) AS ga_refund_events
FROM IDENTIFIER($ga4_table)
WHERE EVENT_NAME='refund'
AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1 ORDER BY 1;

-- Shopify side (only if $shopify_refunds_table is set)
-- SELECT TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz, CREATED_AT)) AS dt,
--        COUNT(*) AS shopify_refunds
-- FROM IDENTIFIER($shopify_refunds_table)
-- WHERE DATE(CONVERT_TIMEZONE('UTC', $store_tz, CREATED_AT)) >= DATEADD('months', -24, CURRENT_DATE())
-- GROUP BY 1 ORDER BY 1;
```

---

### 4.8 Consistency across countries [HYGIENE]
**Method:** Break GA4 purchase counts down by `GEO:country`. Join to Shopify country if needed.
**Pass threshold:** Capture rate consistent across major markets.

```sql
WITH ga AS (
    SELECT GEO:country::string AS country,
           COUNT(DISTINCT ECOMMERCE:transaction_id::string) AS ga_orders
    FROM IDENTIFIER($ga4_table)
    WHERE EVENT_NAME='purchase'
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
    GROUP BY 1
)
SELECT country, ga_orders
FROM ga ORDER BY ga_orders DESC;
```

---

## Domain 5 — Traffic quality & filtering

### 5.1 Bot and invalid traffic proportion [MATERIAL]
**Method:** Build per-session event counts and time spans. Flag sessions with >50 events in <10 seconds and single-event sessions.
**Pass threshold:** Suspected-bot share low and stable.

```sql
WITH s AS (
    SELECT USER_PSEUDO_ID,
        (SELECT p.value:value:int_value::number FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
          WHERE p.value:key::string='ga_session_id') AS ga_session_id,
        COUNT(*) AS events,
        (MAX(EVENT_TIMESTAMP)-MIN(EVENT_TIMESTAMP))/1e6 AS span_seconds
    FROM IDENTIFIER($ga4_table) e
    WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
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

### 5.2 Internal and employee traffic [MATERIAL]
**Method:** Break events by `traffic_type` param. Where no filter exists, identify internal activity via known patterns.
**Pass threshold:** Internal traffic filtered or flagged; near-zero internal test purchases in production.

```sql
SELECT
    COALESCE((SELECT p.value:value:string_value::string FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
        WHERE p.value:key::string='traffic_type'),'(none)') AS traffic_type,
    COUNT(*) AS events
FROM IDENTIFIER($ga4_table) e
WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1 ORDER BY events DESC;
```

---

### 5.4 Self-referrals and referral exclusions [HYGIENE]
**Method:** List top `page_referrer` values on `session_start`. Scan for first-party domains and payment/subscription subdomains.
**Pass threshold:** No first-party domains appearing as acquisition referrers.

```sql
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

### 6.1 Channel composition over time [MATERIAL]
**Method:** Derive each session's channel from `COLLECTED_TRAFFIC_SOURCE` (fallback to `TRAFFIC_SOURCE`). Map to channel groups. Trend session counts.
**Pass threshold:** Stable, explainable channel mix.

```sql
WITH s AS (
    SELECT
        TO_DATE(CONVERT_TIMEZONE('UTC', $store_tz,
            TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6))) AS dt,
        LOWER(COALESCE(e.COLLECTED_TRAFFIC_SOURCE:manual_source::string,
                       e.TRAFFIC_SOURCE:source::string,'(direct)'))  AS source,
        LOWER(COALESCE(e.COLLECTED_TRAFFIC_SOURCE:manual_medium::string,
                       e.TRAFFIC_SOURCE:medium::string,'(none)'))    AS medium
    FROM IDENTIFIER($ga4_table) e
    WHERE EVENT_NAME='session_start'
    AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
)
SELECT dt,
    CASE
        WHEN medium IN ('cpc','ppc','paid','paidsearch') THEN 'Paid Search'
        WHEN medium IN ('email')                          THEN 'Email'
        WHEN medium IN ('social','paid_social','cpc_social') THEN 'Social'
        WHEN medium IN ('referral')                       THEN 'Referral'
        WHEN medium IN ('organic')                        THEN 'Organic Search'
        WHEN source='(direct)' OR medium IN ('(none)','(not set)') THEN 'Direct'
        ELSE 'Other/Unassigned' END                       AS channel,
    COUNT(*)                                              AS sessions
FROM s
GROUP BY 1,2
ORDER BY dt, sessions DESC;
```

---

### 6.2 Direct and unassigned traffic share [MATERIAL]
**Method:** Trend % of sessions classified Direct or unassigned over time, split by `GEO:country`.
**Pass threshold:** Direct share under ~20% for ecommerce; low unassigned.

```sql
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
    COUNT(*)                                                                         AS sessions,
    ROUND(100*AVG(CASE WHEN source='(direct)' OR medium IN ('(none)','(not set)')
                       THEN 1 ELSE 0 END),1)                                         AS pct_direct_unassigned
FROM s
GROUP BY 1,2
ORDER BY dt, sessions DESC;
```

---

### 6.3 UTM hygiene [MATERIAL]
**Method:** Group `session_start` by source and medium params. Inspect for casing inconsistency, missing medium, and paid links arriving with gclid/fbclid but medium `(none)`.
**Pass threshold:** UTM taxonomy consistent; paid and email traffic reliably tagged.

```sql
SELECT
    (SELECT p.value:value:string_value::string FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
        WHERE p.value:key::string='source')  AS utm_source,
    (SELECT p.value:value:string_value::string FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
        WHERE p.value:key::string='medium')  AS utm_medium,
    COUNT(*) AS events
FROM IDENTIFIER($ga4_table) e
WHERE EVENT_NAME='session_start'
AND DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1,2
ORDER BY events DESC;
```

---

### 6.4 Channel grouping re-derivation [HYGIENE]
**Method:** Compare medium from `COLLECTED_TRAFFIC_SOURCE` (manual UTMs) to `TRAFFIC_SOURCE` (GA4-attributed). Quantify reclassification volume.
**Pass threshold:** Re-derived grouping matches default within tolerance, or differences are understood.

```sql
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

### 7.1 PII in event parameters [BLOCKER]
**Method:** Scan `page_location` for email and phone patterns. Any non-zero result is a finding to remediate immediately — not merely report.
**Pass threshold:** Zero PII detected in any parameter.

```sql
WITH locs AS (
    SELECT (SELECT p.value:value:string_value::string FROM LATERAL FLATTEN(input=>e.EVENT_PARAMS) p
              WHERE p.value:key::string='page_location') AS page_location
    FROM IDENTIFIER($ga4_table) e
    WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
)
SELECT
    COUNT(*)                                                                AS rows_checked,
    SUM(CASE WHEN page_location RLIKE '.*[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}.*' THEN 1 ELSE 0 END) AS email_in_url,
    SUM(CASE WHEN page_location RLIKE '.*(phone|tel)=.*[0-9]{7,}.*' THEN 1 ELSE 0 END)                         AS phone_in_url
FROM locs;
```

---

### 7.2 Consent signal coverage (US/CA) [MATERIAL]
**Method:** Break events by `PRIVACY_INFO` `analytics_storage` and `ads_storage` values. Quantify denied/opted-out share.
**Pass threshold:** Consent signal present and behaving; opt-out share within expected range.

```sql
SELECT
    PRIVACY_INFO:analytics_storage::string AS analytics_storage,
    PRIVACY_INFO:ads_storage::string       AS ads_storage,
    PRIVACY_INFO:uses_transient_token::string AS transient_token,
    COUNT(*) AS events
FROM IDENTIFIER($ga4_table)
WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE())
GROUP BY 1,2,3
ORDER BY events DESC;
```

---

### 7.3 Geo-resolution completeness [HYGIENE]
**Method:** Measure populated rate of `GEO:country` and `GEO:region` across all events.
**Pass threshold:** Geo populated on the large majority of sessions.

```sql
SELECT
    ROUND(100*AVG(CASE WHEN GEO:country::string IS NOT NULL AND GEO:country::string<>'' THEN 1 ELSE 0 END),1) AS pct_has_country,
    ROUND(100*AVG(CASE WHEN GEO:region::string  IS NOT NULL AND GEO:region::string<>''  THEN 1 ELSE 0 END),1) AS pct_has_region
FROM IDENTIFIER($ga4_table)
WHERE DATE(TO_TIMESTAMP_NTZ(EVENT_TIMESTAMP,6)) >= DATEADD('months', -24, CURRENT_DATE());
```

---

## Step 2 — Produce the audit report

After all checks are complete, produce a structured report in this exact order:

### 1. Executive summary
- Open with the overall data trust verdict: **TRUSTED / CONDITIONALLY TRUSTED / UNRELIABLE**.
- List all FAIL-BLOCKER findings first, each with its check id, the actual value returned, the threshold, and a one-line business impact statement.
- List FAIL-MATERIAL findings next in the same format.
- Close with the top-3 profit-leak or misattribution findings from Domains 4 and 6, quantified in $ or % wherever the data allows.

### 2. Scope & data sources
- GA4 table, Shopify table, timezone, date range covered.
- Any Domain 0 findings that affect interpretation of later results.

### 3. Domain-by-domain findings
For each domain, one paragraph covering:
- Which checks passed and which failed.
- The actual values returned for every FAIL.
- A one-line remediation recommendation for each FAIL.

### 4. Reconciliation appendix
- GA4-vs-Shopify daily gap table (from 4.3), summarized as: average capture %, min/max day, trend direction.
- Revenue gap summary (from 4.5).

### 5. Prioritized remediation roadmap
A ranked table:

| Priority | Check | Finding | Owner | Effort | Expected impact |
|----------|-------|---------|-------|--------|-----------------|

Order: BLOCKER → MATERIAL → HYGIENE. For each, estimate the downstream analytics impact of leaving it unaddressed.

### 6. Scoring rollup
| Domain | Checks | PASS | FAIL-BLOCKER | FAIL-MATERIAL | FAIL-HYGIENE | Domain score |
|--------|--------|------|--------------|---------------|--------------|--------------|

Overall trust score = 100 − (20 × BLOCKER count) − (5 × MATERIAL count) − (1 × HYGIENE count). Cap at 0, max 100.

---

## Behavioral rules

- **Run the actual SQL.** Do not summarize or skip queries. Every check requires query execution and result interpretation.
- **Stop on Domain 0 BLOCKERs.** If 0.1 or 0.4 fail, surface the finding, explain the impact on downstream domains, and ask the user whether to continue.
- **Be quantitative.** Every finding must cite the actual number returned (e.g. "boundary_shifted_events = 4,231, representing 6.2% of events"). Never describe a result vaguely.
- **BFCM.** Never flag a BFCM-window day as an anomaly. Note it as expected seasonality.
- **Subscription rebills.** For DTC subscription clients, isolate non-web `SOURCE_NAME` orders in check 4.4 before reporting the tracking gap. Rebills absent from GA4 are not a tracking failure.
- **Do not hardcode table names or timezones** in query bodies. Always use `IDENTIFIER($ga4_table)` and `$store_tz`.
- **One finding, one sentence.** In the report body, each check result gets exactly one interpretive sentence stating what it means for this client — not a generic definition.
