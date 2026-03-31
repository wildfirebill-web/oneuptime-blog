# How to Build Real-Time Price Monitoring with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Price Monitoring, Real-Time Analytics, E-commerce, Analytics

Description: Use ClickHouse to ingest product price changes in real time, detect unusual price movements, and alert on threshold breaches within seconds.

---

Price monitoring requires storing large volumes of price ticks, querying historical ranges instantly, and alerting when prices move outside expected bounds. ClickHouse handles all three with minimal infrastructure.

## Price Tick Table

```sql
CREATE TABLE price_ticks
(
    product_id  UInt64,
    source      LowCardinality(String),
    price       Decimal(12, 4),
    currency    LowCardinality(String),
    ts          DateTime64(3) DEFAULT now64()
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(ts)
ORDER BY (product_id, ts);
```

## Latest Price per Product

```sql
SELECT
    product_id,
    argMax(price, ts) AS latest_price,
    argMax(source, ts) AS source,
    max(ts) AS last_seen
FROM price_ticks
GROUP BY product_id;
```

## Price Change Detection

Find products where the latest price differs from the previous price by more than 5%:

```sql
WITH ranked AS (
    SELECT
        product_id,
        price,
        ts,
        lagInFrame(price) OVER (
            PARTITION BY product_id
            ORDER BY ts
        ) AS prev_price
    FROM price_ticks
    WHERE ts >= now() - INTERVAL 1 HOUR
)
SELECT
    product_id,
    prev_price,
    price AS current_price,
    round((price - prev_price) / prev_price * 100, 2) AS change_pct,
    ts
FROM ranked
WHERE abs(change_pct) > 5
ORDER BY abs(change_pct) DESC;
```

## Materialized View for Min/Max Tracking

```sql
CREATE MATERIALIZED VIEW price_range_mv
ENGINE = AggregatingMergeTree()
ORDER BY (product_id, day)
AS
SELECT
    product_id,
    toDate(ts) AS day,
    minState(price) AS min_state,
    maxState(price) AS max_state,
    avgState(price) AS avg_state
FROM price_ticks
GROUP BY product_id, day;
```

```sql
SELECT
    product_id,
    minMerge(min_state) AS day_low,
    maxMerge(max_state) AS day_high,
    avgMerge(avg_state) AS day_avg
FROM price_range_mv
WHERE day = today()
GROUP BY product_id;
```

## Alerting Integration

Run the change detection query every minute from [OneUptime](https://oneuptime.com) as a scheduled check. When the result set is non-empty, fire a webhook to your pricing team's Slack channel.

## Summary

ClickHouse gives you a compact, fast price monitoring store. Window functions handle change detection, materialized views maintain daily OHLC-style statistics, and the HTTP endpoint makes it easy to integrate with alerting pipelines.
