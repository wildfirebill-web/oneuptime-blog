# How to Build a Real-Time Bidding Analytics Platform with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Real-Time Bidding, Ad Tech, Analytics, Programmatic Advertising

Description: Build a high-throughput real-time bidding analytics platform with ClickHouse to track bid requests, wins, and revenue at scale.

---

Real-time bidding (RTB) systems process billions of auction events daily. Each auction produces bid requests, responses, wins, and losses - all in milliseconds. ClickHouse is purpose-built for this kind of high-velocity, high-volume analytics.

## Bid Events Table

```sql
CREATE TABLE bid_events (
    event_time      DateTime64(3),
    auction_id      String,
    bidder_id       UInt32,
    campaign_id     UInt32,
    publisher_id    UInt32,
    bid_price       Float64,
    clearing_price  Float64,
    won             UInt8,
    ad_slot         LowCardinality(String),
    geo_country     LowCardinality(String),
    device_type     LowCardinality(String),
    date            Date DEFAULT toDate(event_time)
) ENGINE = MergeTree()
PARTITION BY date
ORDER BY (campaign_id, event_time);
```

## Win Rate by Campaign

```sql
SELECT
    campaign_id,
    count() AS total_bids,
    sum(won) AS wins,
    round(sum(won) / count() * 100, 2) AS win_rate_pct,
    avg(bid_price) AS avg_bid,
    avg(clearing_price) AS avg_clearing
FROM bid_events
WHERE date = today()
GROUP BY campaign_id
ORDER BY win_rate_pct DESC
LIMIT 20;
```

## Spend by Hour

Track how campaign budgets are being consumed across the day:

```sql
SELECT
    toStartOfHour(event_time) AS hour,
    campaign_id,
    sum(clearing_price) AS total_spend,
    sum(won) AS impressions
FROM bid_events
WHERE date = today() AND won = 1
GROUP BY hour, campaign_id
ORDER BY hour, total_spend DESC;
```

## Bid Landscape Analysis

Understand the competitive landscape by examining bid price distributions:

```sql
SELECT
    ad_slot,
    quantile(0.25)(bid_price) AS p25,
    quantile(0.50)(bid_price) AS p50,
    quantile(0.75)(bid_price) AS p75,
    quantile(0.95)(bid_price) AS p95,
    count() AS bid_count
FROM bid_events
WHERE date = today()
GROUP BY ad_slot
ORDER BY bid_count DESC;
```

## Real-Time Budget Pacing

Check whether campaigns are spending at the expected pace given time of day:

```sql
SELECT
    campaign_id,
    sum(clearing_price) AS spend_so_far,
    toHour(now()) / 24.0 AS day_fraction,
    sum(clearing_price) / (toHour(now()) / 24.0) AS projected_daily_spend
FROM bid_events
WHERE date = today() AND won = 1
GROUP BY campaign_id
ORDER BY projected_daily_spend DESC;
```

## Geo Performance Breakdown

```sql
SELECT
    geo_country,
    count() AS bids,
    sum(won) AS wins,
    round(avg(clearing_price), 4) AS avg_cpm,
    round(sum(won) / count() * 100, 2) AS win_rate
FROM bid_events
WHERE date = today()
GROUP BY geo_country
ORDER BY wins DESC
LIMIT 30;
```

## Summary

ClickHouse handles the extreme write throughput of RTB workloads while delivering sub-second analytical queries across billions of auction records. By partitioning by date and ordering by campaign, you get fast campaign-level aggregations that power real-time bidding dashboards and budget pacing systems.
