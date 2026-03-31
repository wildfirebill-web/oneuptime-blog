# How to Calculate Standard Deviation Bands in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Standard Deviation, Bollinger Band, Analytics, Statistics

Description: Learn how to calculate standard deviation bands in ClickHouse for time-series data to build Bollinger Bands and statistical control charts.

---

Standard deviation bands - also called Bollinger Bands in finance - visualize the expected range of a metric by drawing upper and lower envelopes around a moving average. ClickHouse computes them efficiently with window functions.

## Computing Standard Deviation Bands

```sql
SELECT
    event_date,
    avg_value,
    std_value,
    avg_value - 2 * std_value AS lower_band,
    avg_value + 2 * std_value AS upper_band
FROM (
    SELECT
        event_date,
        avg(metric_value) OVER (
            ORDER BY event_date
            ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
        ) AS avg_value,
        stddevPop(metric_value) OVER (
            ORDER BY event_date
            ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
        ) AS std_value
    FROM daily_metrics
)
ORDER BY event_date;
```

The 2-sigma band covers approximately 95% of values for a normal distribution.

## Bollinger Bands for Financial Data

```sql
SELECT
    trade_date,
    close_price,
    avg(close_price) OVER w AS sma_20,
    stddevPop(close_price) OVER w AS std_20,
    avg(close_price) OVER w + 2 * stddevPop(close_price) OVER w AS upper_band,
    avg(close_price) OVER w - 2 * stddevPop(close_price) OVER w AS lower_band
FROM stock_prices
WHERE symbol = 'AAPL'
WINDOW w AS (ORDER BY trade_date ROWS BETWEEN 19 PRECEDING AND CURRENT ROW)
ORDER BY trade_date;
```

## Detecting Breakouts

Flag when a value breaks outside the bands:

```sql
WITH bands AS (
    SELECT
        ts,
        value,
        avg(value) OVER (ORDER BY ts ROWS BETWEEN 19 PRECEDING AND CURRENT ROW) AS mid,
        stddevPop(value) OVER (ORDER BY ts ROWS BETWEEN 19 PRECEDING AND CURRENT ROW) AS std
    FROM metric_series
)
SELECT ts, value, mid, std,
    CASE
        WHEN value > mid + 2 * std THEN 'above_upper'
        WHEN value < mid - 2 * std THEN 'below_lower'
        ELSE 'normal'
    END AS band_position
FROM bands
WHERE band_position != 'normal'
ORDER BY ts;
```

## Bandwidth Metric

The bandwidth measures how wide the bands are relative to the moving average:

```sql
SELECT
    ts,
    (upper_band - lower_band) / mid AS bandwidth
FROM bands_table
ORDER BY ts;
```

Expanding bandwidth signals increasing volatility.

## Summary

ClickHouse computes standard deviation bands using `avg` and `stddevPop` window functions over a fixed-row window. Use the `WINDOW` clause to avoid repeating the frame definition. Combine with a CASE statement to flag values that break outside the expected range.
