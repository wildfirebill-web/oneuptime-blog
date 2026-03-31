# How to Use ASOF JOIN in ClickHouse for Time-Series Data

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, SELECT, ASOF JOIN, Time Series, Closest Match

Description: Learn how to use ASOF JOIN in ClickHouse to match each row with the nearest previous row in another table, ideal for time-series and telemetry data.

---

`ASOF JOIN` is a ClickHouse-specific join type that matches each row from the left table with the closest - but not exceeding - row from the right table based on an inequality condition. It is designed for time-series scenarios where you need to look up the most recent state or measurement that was recorded before or at a given timestamp. Standard joins require exact matches, while `ASOF JOIN` finds the nearest prior row automatically.

## ASOF JOIN Syntax

```sql
SELECT ...
FROM left_table AS l
ASOF JOIN right_table AS r
ON l.key_column = r.key_column
AND l.time_column >= r.time_column;
```

The `ON` clause must contain:
1. One or more exact-match equality conditions (the join key).
2. Exactly one inequality condition (the "asof" column), typically `>=` or `<=`.

The right table must be sorted by the asof column for each key.

## Matching Telemetry to Configuration Versions

A typical use case: you have a stream of metric readings and a table of configuration changes. For each metric reading, find the configuration that was active at that moment.

```sql
-- Config changes table: records when each service changed config
CREATE TABLE config_versions
(
    service_id  UInt32,
    changed_at  DateTime,
    config_name String,
    threshold   Float64
)
ENGINE = MergeTree
ORDER BY (service_id, changed_at);

-- Metric readings table
CREATE TABLE metric_readings
(
    service_id   UInt32,
    recorded_at  DateTime,
    metric_value Float64
)
ENGINE = MergeTree
ORDER BY (service_id, recorded_at);

-- ASOF JOIN: for each metric reading, find the most recent config
SELECT
    m.service_id,
    m.recorded_at,
    m.metric_value,
    c.config_name,
    c.threshold,
    m.metric_value > c.threshold AS exceeds_threshold
FROM metric_readings AS m
ASOF JOIN config_versions AS c
    ON m.service_id = c.service_id
    AND m.recorded_at >= c.changed_at
ORDER BY m.service_id, m.recorded_at;
```

## ASOF JOIN with Financial Price Data

Another common case: enriching trade records with the last known bid/ask price before each trade.

```sql
CREATE TABLE stock_prices
(
    symbol     String,
    price_time DateTime64(3),
    bid        Float64,
    ask        Float64
)
ENGINE = MergeTree
ORDER BY (symbol, price_time);

CREATE TABLE trades
(
    trade_id   UInt64,
    symbol     String,
    trade_time DateTime64(3),
    quantity   UInt32,
    trade_price Float64
)
ENGINE = MergeTree
ORDER BY (symbol, trade_time);

SELECT
    t.trade_id,
    t.symbol,
    t.trade_time,
    t.trade_price,
    p.bid,
    p.ask,
    (p.bid + p.ask) / 2 AS midpoint,
    t.trade_price - midpoint AS slippage
FROM trades AS t
ASOF JOIN stock_prices AS p
    ON t.symbol = p.symbol
    AND t.trade_time >= p.price_time;
```

## Supported Inequality Operators

ClickHouse supports both directions of the inequality:

```sql
-- Match the greatest value in right table that is <= left table value (most common)
ASOF JOIN t ON l.key = t.key AND l.ts >= t.ts

-- Match the smallest value in right table that is >= left table value (forward lookup)
ASOF JOIN t ON l.key = t.key AND l.ts <= t.ts
```

## ORDER BY Requirement for the Right Table

The right table must be sorted (or physically stored) by the join key and the asof column. Using a `MergeTree` with `ORDER BY (key, time_column)` satisfies this requirement automatically.

```sql
-- Right table must be ordered by the asof column within each key
CREATE TABLE reference_rates
(
    currency_pair String,
    rate_time     DateTime,
    rate          Float64
)
ENGINE = MergeTree
ORDER BY (currency_pair, rate_time);
-- ASOF JOIN on (currency_pair, transaction_time >= rate_time) works correctly
```

## Left ASOF JOIN

Use `LEFT ASOF JOIN` to keep rows from the left table even when no matching row exists in the right table (the asof columns will be NULL or default).

```sql
SELECT
    m.event_id,
    m.event_time,
    m.value,
    r.threshold  -- NULL if no config existed before this event
FROM events AS m
LEFT ASOF JOIN config_thresholds AS r
    ON m.service_id = r.service_id
    AND m.event_time >= r.effective_from;
```

## Summary

`ASOF JOIN` solves the "last known value" problem in time-series data by matching each left-table row with the nearest prior (or nearest next) row in the right table based on an inequality condition. The right table must be sorted by the join key and asof column. It is ideal for telemetry enrichment, financial data analysis, and any scenario requiring the most recently valid reference value at a given point in time.
