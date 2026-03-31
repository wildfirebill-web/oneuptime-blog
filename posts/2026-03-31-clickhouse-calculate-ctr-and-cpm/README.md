# How to Calculate CTR and CPM in Real-Time with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, CTR, CPM, AdTech, Analytics

Description: Calculate click-through rate and cost per mille in real time with ClickHouse using efficient aggregation queries over ad event streams.

---

CTR (Click-Through Rate) and CPM (Cost Per Mille) are the two most fundamental metrics in digital advertising. ClickHouse computes them in real time over high-volume impression and click event streams.

## Ad Events Table

```sql
CREATE TABLE ad_events
(
    event_id UUID DEFAULT generateUUIDv4(),
    event_type LowCardinality(String),
    impression_id String,
    ad_id UInt32,
    campaign_id UInt32,
    advertiser_id UInt32,
    placement_id UInt32,
    cost Decimal(10, 6) DEFAULT 0,
    device_type LowCardinality(String),
    country LowCardinality(String),
    event_time DateTime64(3)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(event_time)
ORDER BY (campaign_id, ad_id, event_time);
```

## Real-Time CTR per Campaign

```sql
SELECT
    campaign_id,
    countIf(event_type = 'impression') AS impressions,
    countIf(event_type = 'click') AS clicks,
    round(
        countIf(event_type = 'click') * 100.0 /
        nullIf(countIf(event_type = 'impression'), 0),
        4
    ) AS ctr_pct
FROM ad_events
WHERE event_time >= now() - INTERVAL 1 HOUR
GROUP BY campaign_id
ORDER BY ctr_pct DESC
LIMIT 20;
```

## Real-Time CPM per Campaign

```sql
SELECT
    campaign_id,
    countIf(event_type = 'impression') AS impressions,
    sum(cost) AS total_spend,
    round(
        sum(cost) * 1000.0 /
        nullIf(countIf(event_type = 'impression'), 0),
        4
    ) AS cpm
FROM ad_events
WHERE event_time >= now() - INTERVAL 1 HOUR
GROUP BY campaign_id
ORDER BY cpm DESC
LIMIT 20;
```

## CTR and CPM by Ad Creative

```sql
SELECT
    ad_id,
    campaign_id,
    countIf(event_type = 'impression') AS impressions,
    countIf(event_type = 'click') AS clicks,
    sum(cost) AS spend,
    round(
        countIf(event_type = 'click') * 100.0 /
        nullIf(countIf(event_type = 'impression'), 0),
        4
    ) AS ctr_pct,
    round(
        sum(cost) * 1000.0 /
        nullIf(countIf(event_type = 'impression'), 0),
        4
    ) AS cpm
FROM ad_events
WHERE event_time >= now() - INTERVAL 24 HOUR
GROUP BY ad_id, campaign_id
HAVING impressions > 1000
ORDER BY ctr_pct DESC
LIMIT 30;
```

## Hourly CTR Trend

```sql
SELECT
    toStartOfHour(event_time) AS hour,
    campaign_id,
    countIf(event_type = 'impression') AS impressions,
    countIf(event_type = 'click') AS clicks,
    round(
        countIf(event_type = 'click') * 100.0 /
        nullIf(countIf(event_type = 'impression'), 0),
        4
    ) AS ctr_pct
FROM ad_events
WHERE campaign_id = 42
  AND event_time >= now() - INTERVAL 24 HOUR
GROUP BY hour, campaign_id
ORDER BY hour;
```

## Country-Level CTR

```sql
SELECT
    country,
    device_type,
    countIf(event_type = 'impression') AS impressions,
    countIf(event_type = 'click') AS clicks,
    round(
        countIf(event_type = 'click') * 100.0 /
        nullIf(countIf(event_type = 'impression'), 0),
        4
    ) AS ctr_pct
FROM ad_events
WHERE event_time >= now() - INTERVAL 24 HOUR
GROUP BY country, device_type
HAVING impressions > 5000
ORDER BY ctr_pct DESC
LIMIT 20;
```

## Materialized View for Sub-Second Dashboards

```sql
CREATE MATERIALIZED VIEW ad_metrics_5min
ENGINE = SummingMergeTree()
ORDER BY (campaign_id, ad_id, bucket)
AS
SELECT
    campaign_id,
    ad_id,
    toStartOfFiveMinutes(event_time) AS bucket,
    countIf(event_type = 'impression') AS impressions,
    countIf(event_type = 'click') AS clicks,
    sum(cost) AS spend
FROM ad_events
GROUP BY campaign_id, ad_id, bucket;
```

Query the materialized view for fast dashboard reads:

```sql
SELECT
    campaign_id,
    sum(impressions) AS total_impressions,
    sum(clicks) AS total_clicks,
    round(sum(clicks) * 100.0 / nullIf(sum(impressions), 0), 4) AS ctr_pct,
    round(sum(spend) * 1000.0 / nullIf(sum(impressions), 0), 4) AS cpm
FROM ad_metrics_5min
WHERE bucket >= now() - INTERVAL 1 HOUR
GROUP BY campaign_id
ORDER BY ctr_pct DESC;
```

## Summary

ClickHouse computes CTR and CPM in real time using `countIf` for impression and click counting within a single scan. `SummingMergeTree` materialized views with 5-minute buckets reduce dashboard query time to milliseconds. This approach scales to billions of ad events per day while keeping advertiser dashboards responsive.
