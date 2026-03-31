# How to Build a Real-Time Bidding Analytics Platform with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Real-Time Bidding, AdTech, Analytics, Programmatic

Description: Build a real-time bidding analytics platform with ClickHouse to analyze bid requests, win rates, and spend optimization at millisecond scale.

---

Real-time bidding (RTB) generates millions of auction events per second. ClickHouse's high ingest rate and fast aggregation make it the right choice for a bidding analytics platform that needs to track win rates, CPM trends, and budget pacing in near real time.

## Bid Events Table

```sql
CREATE TABLE bid_events
(
    bid_id UUID DEFAULT generateUUIDv4(),
    auction_id String,
    advertiser_id UInt32,
    campaign_id UInt32,
    ad_unit_id UInt32,
    publisher_id UInt32,
    bid_price Decimal(10, 6),
    clear_price Decimal(10, 6) DEFAULT 0,
    outcome LowCardinality(String),
    win UInt8 DEFAULT 0,
    ad_format LowCardinality(String),
    device_type LowCardinality(String),
    country LowCardinality(String),
    bid_time DateTime64(3)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(bid_time)
ORDER BY (advertiser_id, campaign_id, bid_time);
```

## Win Rate and Average CPM by Campaign

```sql
SELECT
    campaign_id,
    advertiser_id,
    count() AS total_bids,
    countIf(win = 1) AS wins,
    round(countIf(win = 1) * 100.0 / count(), 3) AS win_rate_pct,
    round(avg(bid_price) * 1000, 4) AS avg_bid_cpm,
    round(avg(clear_price) * 1000, 4) AS avg_clear_cpm,
    sum(clear_price) AS total_spend
FROM bid_events
WHERE bid_time >= now() - INTERVAL 1 HOUR
  AND win = 1
GROUP BY campaign_id, advertiser_id
ORDER BY win_rate_pct DESC
LIMIT 20;
```

## Bid Landscape - Distribution of Bid Prices

```sql
SELECT
    campaign_id,
    round(bid_price * 1000, 2) AS bid_cpm_rounded,
    count() AS bids,
    countIf(win = 1) AS wins
FROM bid_events
WHERE bid_time >= now() - INTERVAL 1 HOUR
  AND campaign_id = 12345
GROUP BY campaign_id, bid_cpm_rounded
ORDER BY bid_cpm_rounded;
```

## Budget Pacing - Spend per Hour

```sql
SELECT
    toStartOfHour(bid_time) AS hour,
    campaign_id,
    sum(clear_price) AS spend,
    countIf(win = 1) AS impressions,
    round(sum(clear_price) * 1000 / nullIf(countIf(win = 1), 0), 4) AS effective_cpm
FROM bid_events
WHERE campaign_id = 12345
  AND bid_time >= now() - INTERVAL 24 HOUR
GROUP BY hour, campaign_id
ORDER BY hour;
```

## Publisher Performance

```sql
SELECT
    publisher_id,
    ad_format,
    count() AS bid_requests,
    countIf(win = 1) AS wins,
    round(countIf(win = 1) * 100.0 / count(), 3) AS win_rate_pct,
    round(avg(clear_price) * 1000, 4) AS avg_cpm,
    sum(clear_price) AS total_revenue
FROM bid_events
WHERE bid_time >= now() - INTERVAL 24 HOUR
GROUP BY publisher_id, ad_format
ORDER BY total_revenue DESC
LIMIT 20;
```

## Device Type Performance Breakdown

```sql
SELECT
    device_type,
    country,
    count() AS bids,
    countIf(win = 1) AS wins,
    round(countIf(win = 1) * 100.0 / count(), 2) AS win_rate_pct,
    round(avg(clear_price) * 1000, 4) AS avg_cpm
FROM bid_events
WHERE bid_time >= now() - INTERVAL 6 HOUR
GROUP BY device_type, country
ORDER BY wins DESC
LIMIT 30;
```

## Materialized View for Real-Time Campaign Dashboard

```sql
CREATE MATERIALIZED VIEW campaign_hourly_stats
ENGINE = SummingMergeTree()
ORDER BY (advertiser_id, campaign_id, hour)
AS
SELECT
    advertiser_id,
    campaign_id,
    toStartOfHour(bid_time) AS hour,
    count() AS total_bids,
    sum(win) AS wins,
    sum(clear_price) AS spend
FROM bid_events
GROUP BY advertiser_id, campaign_id, hour;
```

## Summary

ClickHouse handles RTB analytics by combining high-write throughput for bid event ingestion with fast aggregation for win rate, CPM, and spend queries. `SummingMergeTree` materialized views enable sub-second dashboard updates even at millions of bids per minute. This architecture supports the tight feedback loops that programmatic advertising optimization requires.
