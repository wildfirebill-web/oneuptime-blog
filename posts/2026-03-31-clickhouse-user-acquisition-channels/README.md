# How to Track User Acquisition Channels in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Acquisition, UTM, Attribution, Marketing Analytics

Description: Learn how to track user acquisition channels in ClickHouse by parsing UTM parameters, attributing conversions, and comparing channel performance metrics.

---

## Acquisition Channel Analytics

Understanding where users come from is fundamental to marketing efficiency. ClickHouse stores UTM parameters from landing page events and attributes conversions to acquisition sources using first-touch and last-touch models.

## Storing UTM Parameters

Capture UTM parameters on signup or first session:

```sql
CREATE TABLE user_acquisition
(
    user_id         UInt64,
    signup_time     DateTime,
    utm_source      LowCardinality(String) DEFAULT '',
    utm_medium      LowCardinality(String) DEFAULT '',
    utm_campaign    String DEFAULT '',
    utm_content     String DEFAULT '',
    landing_page    String DEFAULT '',
    country         LowCardinality(String) DEFAULT ''
)
ENGINE = MergeTree()
ORDER BY (signup_time, utm_source);
```

## Parsing UTM from URL

Extract UTM parameters from raw landing page URLs:

```sql
SELECT
    user_id,
    signup_time,
    extractURLParameter(landing_url, 'utm_source') AS utm_source,
    extractURLParameter(landing_url, 'utm_medium') AS utm_medium,
    extractURLParameter(landing_url, 'utm_campaign') AS utm_campaign
FROM raw_signups
WHERE landing_url LIKE '%utm_source%';
```

## Channel Performance Dashboard

Compare key metrics by acquisition channel:

```sql
SELECT
    a.utm_source AS channel,
    count(DISTINCT a.user_id) AS signups,
    countDistinctIf(o.customer_id, o.status = 'completed') AS paying_customers,
    round(countDistinctIf(o.customer_id, o.status = 'completed') /
        count(DISTINCT a.user_id) * 100, 2) AS conversion_rate,
    round(sum(o.order_value) / count(DISTINCT a.user_id), 2) AS revenue_per_signup
FROM user_acquisition a
LEFT JOIN orders o ON a.user_id = o.customer_id
WHERE a.signup_time >= today() - 90
GROUP BY channel
ORDER BY paying_customers DESC;
```

## First-Touch Attribution

Assign all conversion value to the first channel:

```sql
SELECT
    first_channel,
    count(DISTINCT customer_id) AS converted_users,
    sum(total_revenue) AS attributed_revenue
FROM (
    SELECT
        o.customer_id,
        a.utm_source AS first_channel,
        sum(o.order_value) AS total_revenue
    FROM orders o
    JOIN user_acquisition a ON o.customer_id = a.user_id
    WHERE o.status = 'completed'
    GROUP BY o.customer_id, a.utm_source
)
GROUP BY first_channel
ORDER BY attributed_revenue DESC;
```

## Time to Convert by Channel

Measure how long it takes users from each channel to make their first purchase:

```sql
SELECT
    a.utm_source AS channel,
    quantile(0.50)(dateDiff('day', a.signup_time, o.first_order)) AS median_days_to_convert,
    quantile(0.90)(dateDiff('day', a.signup_time, o.first_order)) AS p90_days_to_convert,
    count() AS converted_users
FROM user_acquisition a
JOIN (
    SELECT customer_id, min(order_time) AS first_order
    FROM orders WHERE status = 'completed'
    GROUP BY customer_id
) o ON a.user_id = o.customer_id
GROUP BY channel
ORDER BY median_days_to_convert;
```

## Weekly Channel Trend

Track channel mix over time:

```sql
SELECT
    toMonday(signup_time) AS week,
    utm_source,
    count() AS signups
FROM user_acquisition
WHERE signup_time >= today() - 90
GROUP BY week, utm_source
ORDER BY week, signups DESC;
```

## Summary

ClickHouse acquisition analytics ingests UTM parameters at signup, attributes conversions using first-touch joins, and compares channel performance across conversion rate, revenue per signup, and time-to-convert. Weekly channel mix trends reveal shifts in marketing efficiency, all in sub-second queries over millions of users.
