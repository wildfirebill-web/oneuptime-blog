# How to Use avgWeighted() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, avgWeighted, Weighted Average

Description: Learn how to compute a weighted arithmetic mean in ClickHouse using avgWeighted(value, weight) for pricing, ratings, and metric aggregation.

---

`avgWeighted()` is a ClickHouse aggregate function that computes the weighted arithmetic mean of a value column using a separate weight column. The result equals the sum of `(value * weight)` divided by the sum of all weights. This is the correct aggregation to use when rows represent different-sized populations - such as average product price weighted by units sold, average rating weighted by review count, or average latency weighted by request volume.

## Syntax

```sql
avgWeighted(value, weight)
```

Both `value` and `weight` must be numeric. The function returns `Float64`. If the sum of all weights is zero, the result is `nan`.

## Basic Example - Revenue-Weighted Average Price

Using plain `avg(price)` treats each row equally regardless of how many units were sold. `avgWeighted` accounts for volume.

```sql
SELECT
    category,
    avg(unit_price)                        AS simple_avg_price,
    avgWeighted(unit_price, units_sold)    AS weighted_avg_price
FROM product_sales
WHERE sale_date >= today() - 30
GROUP BY category
ORDER BY category;
```

If one SKU sold 10,000 units at $2 and another sold 5 units at $100, `simple_avg_price` is $51 while `weighted_avg_price` is close to $2 - the economically meaningful result.

## Use Case - Volume-Weighted Average Price (VWAP)

VWAP is a standard financial metric computed as total trade value divided by total volume.

```sql
SELECT
    symbol,
    toDate(trade_time)                     AS trade_date,
    avgWeighted(price, volume)             AS vwap
FROM trades
WHERE trade_time >= today() - 7
GROUP BY symbol, trade_date
ORDER BY symbol, trade_date;
```

## Use Case - Weighted Star Rating

When aggregating ratings from multiple sources that have different review counts, weight by the number of reviews.

```sql
SELECT
    product_id,
    sum(review_count)                          AS total_reviews,
    avgWeighted(avg_star_rating, review_count) AS weighted_rating
FROM product_review_summaries
GROUP BY product_id
HAVING total_reviews >= 10
ORDER BY weighted_rating DESC
LIMIT 20;
```

## Use Case - SLA-Weighted Latency

Compute a latency average weighted by the number of requests handled by each service instance.

```sql
SELECT
    service_name,
    avgWeighted(p99_latency_ms, request_count) AS weighted_p99
FROM service_stats
WHERE window_start >= now() - INTERVAL 1 HOUR
GROUP BY service_name
ORDER BY weighted_p99 DESC;
```

## Comparing avg vs avgWeighted

```sql
SELECT
    region,
    count()                                    AS data_points,
    avg(conversion_rate)                       AS simple_avg,
    avgWeighted(conversion_rate, session_count) AS weighted_avg
FROM daily_conversion_stats
WHERE stat_date >= today() - 14
GROUP BY region
ORDER BY region;
```

The two values diverge most when high-volume data points have very different rates than low-volume ones. The weighted average tells you the effective rate experienced by users, not the effective rate per reporting segment.

## Handling Zero Weights

If any group has all-zero weights, `avgWeighted` returns `nan`. Use `isNaN` to handle this in downstream logic.

```sql
SELECT
    category,
    if(
        isNaN(avgWeighted(unit_price, units_sold)),
        0.0,
        avgWeighted(unit_price, units_sold)
    ) AS safe_weighted_price
FROM product_sales
GROUP BY category;
```

Alternatively, filter out zero-weight rows before aggregating.

```sql
SELECT
    category,
    avgWeighted(unit_price, units_sold) AS weighted_price
FROM product_sales
WHERE units_sold > 0
GROUP BY category;
```

## Using avgWeighted with Pre-aggregated Data

`avgWeighted` composes correctly with pre-grouped data as long as you carry forward the sum of weights.

```sql
-- Already-grouped hourly data rolled up to daily
SELECT
    event_date,
    avgWeighted(hourly_avg_latency, total_requests) AS daily_weighted_latency
FROM hourly_latency_stats
WHERE event_date >= today() - 30
GROUP BY event_date
ORDER BY event_date;
```

This works because `avgWeighted(avg, count)` over hourly buckets equals `sum(avg * count) / sum(count)`, which is the correct daily weighted average.

## Summary

`avgWeighted(value, weight)` computes the statistically correct mean when rows represent populations of different sizes, replacing the misleading simple average that treats every row equally. It is the right function for VWAP calculations, review aggregations, latency rollups, and any scenario where a count or volume column should influence how much each value contributes to the final average. Remember to handle all-zero-weight groups with `isNaN` checks to avoid propagating `nan` values into dashboards or downstream queries.
