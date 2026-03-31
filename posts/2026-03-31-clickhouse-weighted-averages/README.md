# How to Calculate Weighted Averages in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Weighted Average, Aggregate Function, Analytics, Statistics

Description: Learn how to calculate weighted averages in ClickHouse using avgWeighted and manual sum/weight expressions for accurate metric aggregation.

---

A weighted average gives each value a weight proportional to its importance before averaging. ClickHouse provides a dedicated `avgWeighted` function and supports manual weighted calculations for flexible use cases.

## Using avgWeighted

`avgWeighted(value, weight)` computes `sum(value * weight) / sum(weight)`:

```sql
SELECT avgWeighted(price, quantity) AS weighted_avg_price
FROM order_items
WHERE order_date >= today() - 30;
```

This calculates the volume-weighted average price across all items.

## Weighted Average by Group

```sql
SELECT
    product_category,
    avgWeighted(unit_price, units_sold) AS vwap
FROM sales
GROUP BY product_category
ORDER BY vwap DESC;
```

## Manual Calculation for Transparency

When you need to see the numerator and denominator separately:

```sql
SELECT
    region,
    sum(score * response_count) AS weighted_sum,
    sum(response_count) AS total_responses,
    round(sum(score * response_count) / sum(response_count), 2) AS weighted_avg
FROM survey_results
GROUP BY region
ORDER BY weighted_avg DESC;
```

## Time-Weighted Average

For sensor or time-series data, weight by the duration each reading was active:

```sql
SELECT
    sensor_id,
    sum(value * toFloat64(duration_seconds)) / sum(duration_seconds) AS time_weighted_avg
FROM sensor_readings
WHERE recorded_at >= now() - INTERVAL 1 HOUR
GROUP BY sensor_id;
```

## Combining with Window Functions

Compute a rolling weighted average over a sliding time window:

```sql
SELECT
    event_time,
    avgWeighted(price, volume) OVER (
        ORDER BY event_time
        ROWS BETWEEN 9 PRECEDING AND CURRENT ROW
    ) AS rolling_vwap
FROM trades
ORDER BY event_time;
```

## Handling Zero Weights

Guard against division by zero when weights can be zero:

```sql
SELECT
    if(sum(weight) > 0, sum(value * weight) / sum(weight), NULL) AS safe_weighted_avg
FROM measurements;
```

## Summary

ClickHouse's `avgWeighted(value, weight)` function is the simplest way to compute weighted averages. For time-series data, use duration-weighted expressions. Always guard against zero-weight denominators, and combine with window functions for rolling weighted metrics.
