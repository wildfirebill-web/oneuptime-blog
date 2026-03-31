# How to Calculate Conversion Rate in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Conversion Rate, Funnel, Analytics, SQL

Description: Calculate funnel conversion rates in ClickHouse using countIf and windowFunnel to measure step-by-step drop-off in user journeys.

---

## Simple Conversion Rate

Conversion rate = users who completed the goal / total users who started * 100.

```sql
SELECT
    uniq(user_id)                               AS visitors,
    uniqIf(user_id, event = 'purchase')         AS purchasers,
    round(
        uniqIf(user_id, event = 'purchase')
        * 100.0 / uniq(user_id),
        2
    )                                           AS conversion_rate_pct
FROM events
WHERE ts >= today() - 30;
```

## Multi-Step Funnel with windowFunnel

ClickHouse's `windowFunnel` function counts how many steps each user completed within a time window.

```sql
SELECT
    level,
    count()  AS users,
    round(count() * 100.0 / max(count()) OVER (), 2) AS pct_of_top
FROM (
    SELECT
        user_id,
        windowFunnel(86400)(  -- 24-hour window
            ts,
            event = 'page_view',
            event = 'add_to_cart',
            event = 'checkout',
            event = 'purchase'
        ) AS level
    FROM events
    WHERE ts >= today() - 30
    GROUP BY user_id
)
GROUP BY level
ORDER BY level DESC;
```

## Step-by-Step Drop-Off

```sql
WITH funnel AS (
    SELECT
        user_id,
        countIf(event = 'page_view')    AS viewed,
        countIf(event = 'add_to_cart')  AS carted,
        countIf(event = 'checkout')     AS checked_out,
        countIf(event = 'purchase')     AS purchased
    FROM events
    WHERE ts >= today() - 30
    GROUP BY user_id
)
SELECT
    countIf(viewed > 0)         AS step1_views,
    countIf(carted > 0)         AS step2_cart,
    countIf(checked_out > 0)    AS step3_checkout,
    countIf(purchased > 0)      AS step4_purchase,
    round(countIf(carted > 0)       * 100.0 / countIf(viewed > 0),      2) AS view_to_cart_pct,
    round(countIf(checked_out > 0)  * 100.0 / countIf(carted > 0),      2) AS cart_to_checkout_pct,
    round(countIf(purchased > 0)    * 100.0 / countIf(checked_out > 0), 2) AS checkout_to_purchase_pct
FROM funnel;
```

## Conversion by Traffic Source

```sql
SELECT
    utm_source,
    uniq(user_id)                               AS visitors,
    uniqIf(user_id, event = 'purchase')         AS buyers,
    round(
        uniqIf(user_id, event = 'purchase')
        * 100.0 / uniq(user_id),
        2
    )                                           AS cvr_pct
FROM events
WHERE ts >= today() - 30
GROUP BY utm_source
HAVING visitors >= 100
ORDER BY cvr_pct DESC;
```

## Conversion Trend by Day

```sql
SELECT
    toDate(ts)                                     AS day,
    uniq(user_id)                                  AS visitors,
    uniqIf(user_id, event = 'purchase')            AS buyers,
    round(uniqIf(user_id, event = 'purchase')
          * 100.0 / uniq(user_id), 2)              AS cvr_pct
FROM events
WHERE ts >= today() - 30
GROUP BY day
ORDER BY day;
```

## Summary

ClickHouse calculates conversion rates efficiently with `uniqIf` for goal completion and `windowFunnel` for multi-step funnels. Break down conversion by traffic source, device, or landing page using GROUP BY, and add HAVING clauses to filter segments with insufficient sample sizes.
