# How to Analyze Ad Viewability Metrics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Ad Viewability, Ad Tech, Analytics, MRC Standard

Description: Analyze ad viewability metrics in ClickHouse to measure how much of your ad inventory was actually seen by users.

---

Ad viewability measures whether an ad had the opportunity to be seen. The MRC standard defines a display ad as viewable when at least 50% of pixels are in-view for at least 1 continuous second. ClickHouse makes it easy to aggregate and analyze viewability data at scale.

## Viewability Events Table

```sql
CREATE TABLE viewability_events (
    event_time          DateTime64(3),
    impression_id       String,
    ad_id               UInt32,
    campaign_id         UInt32,
    publisher_id        UInt32,
    placement           LowCardinality(String),
    in_view_pct         Float32,
    in_view_duration_ms UInt32,
    is_viewable         UInt8,
    device_type         LowCardinality(String),
    date                Date DEFAULT toDate(event_time)
) ENGINE = MergeTree()
PARTITION BY date
ORDER BY (campaign_id, publisher_id, event_time);
```

## Overall Viewability Rate

```sql
SELECT
    date,
    count() AS total_impressions,
    sum(is_viewable) AS viewable,
    round(sum(is_viewable) / count() * 100, 2) AS viewability_rate_pct
FROM viewability_events
WHERE date >= today() - 7
GROUP BY date
ORDER BY date;
```

## Viewability by Placement

Identify which ad placements perform best for viewability:

```sql
SELECT
    placement,
    count() AS impressions,
    sum(is_viewable) AS viewable,
    round(sum(is_viewable) / count() * 100, 2) AS viewability_rate_pct,
    avg(in_view_pct) AS avg_in_view_pct,
    avg(in_view_duration_ms) AS avg_duration_ms
FROM viewability_events
WHERE date = today()
GROUP BY placement
ORDER BY viewability_rate_pct DESC;
```

## Publisher Viewability Scorecard

```sql
SELECT
    publisher_id,
    count() AS impressions,
    round(sum(is_viewable) / count() * 100, 2) AS viewability_rate_pct,
    avg(in_view_pct) AS avg_pixel_pct,
    avg(in_view_duration_ms) AS avg_duration_ms
FROM viewability_events
WHERE date >= today() - 30
GROUP BY publisher_id
HAVING impressions > 10000
ORDER BY viewability_rate_pct DESC;
```

## Viewability Distribution

Understand the distribution of in-view percentages, not just the binary viewable flag:

```sql
SELECT
    multiIf(
        in_view_pct < 25, '0-25%',
        in_view_pct < 50, '25-50%',
        in_view_pct < 75, '50-75%',
        '75-100%'
    ) AS visibility_bucket,
    count() AS count,
    round(count() / sum(count()) OVER () * 100, 2) AS share_pct
FROM viewability_events
WHERE date = today()
GROUP BY visibility_bucket
ORDER BY visibility_bucket;
```

## Campaign Viewability vs CTR Correlation

```sql
SELECT
    v.campaign_id,
    round(sum(v.is_viewable) / count(v.impression_id) * 100, 2) AS viewability_rate,
    round(count(c.click_id) / count(v.impression_id) * 100, 4) AS ctr_pct
FROM viewability_events AS v
LEFT JOIN ad_clicks AS c ON v.impression_id = c.impression_id AND v.date = c.date
WHERE v.date >= today() - 7
GROUP BY v.campaign_id
ORDER BY viewability_rate DESC;
```

## Summary

ClickHouse enables fast viewability reporting across millions of impression records. By tracking both the binary viewable flag and continuous metrics like in-view percentage and duration, you can surface publisher scorecards, placement benchmarks, and correlations between viewability and downstream engagement.
