# How to Analyze Returns and Reverse Logistics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Return, Reverse Logistics, E-Commerce, Analytics, Refund

Description: Analyze product returns and reverse logistics in ClickHouse - return rates by category, reason codes, refund cycle times, and restocking efficiency.

---

Returns analytics is critical for e-commerce profitability. Every return has a cost - return shipping, inspection, restocking or disposal. ClickHouse helps identify high-return SKUs, measure processing times, and optimize reverse logistics.

## Returns Table

```sql
CREATE TABLE returns (
    return_id        UUID,
    order_id         String,
    customer_id      UInt64,
    sku              String,
    product_name     String,
    category         LowCardinality(String),
    return_reason    LowCardinality(String),  -- defective, wrong_item, not_needed, size_issue
    return_condition LowCardinality(String),  -- new, damaged, opened
    refund_type      LowCardinality(String),  -- full, partial, exchange
    order_amount     Decimal64(2),
    refund_amount    Decimal64(2),
    requested_at     DateTime,
    received_at      Nullable(DateTime),
    refunded_at      Nullable(DateTime),
    restocked_at     Nullable(DateTime),
    warehouse_id     LowCardinality(String)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(requested_at)
ORDER BY (sku, requested_at);
```

## Return Rate by Category

```sql
SELECT
    r.category,
    count() AS total_returns,
    sum(r.refund_amount) AS total_refund_value,
    -- assumes orders table for denominator
    round(count() * 100.0 / 1000, 2) AS return_rate_pct  -- replace 1000 with actual order count
FROM returns r
WHERE requested_at >= today() - 30
GROUP BY category
ORDER BY total_returns DESC;
```

## Top Return Reasons

```sql
SELECT
    return_reason,
    category,
    count() AS return_count,
    round(count() / sum(count()) OVER (PARTITION BY return_reason) * 100, 2) AS reason_share_pct,
    avg(refund_amount) AS avg_refund_value
FROM returns
WHERE requested_at >= today() - 30
GROUP BY return_reason, category
ORDER BY return_count DESC;
```

## Refund Cycle Time

Measure time from return request to refund issued:

```sql
SELECT
    warehouse_id,
    refund_type,
    avg(dateDiff('hour', requested_at, refunded_at)) AS avg_refund_hours,
    quantile(0.9)(dateDiff('hour', requested_at, refunded_at)) AS p90_refund_hours,
    countIf(dateDiff('hour', requested_at, refunded_at) > 72) AS sla_breach_count
FROM returns
WHERE refunded_at IS NOT NULL
  AND requested_at >= today() - 30
GROUP BY warehouse_id, refund_type
ORDER BY avg_refund_hours DESC;
```

## High-Return SKUs

Identify SKUs that drive the most returns - candidates for listing improvement or discontinuation:

```sql
SELECT
    sku,
    product_name,
    category,
    count() AS return_count,
    sum(refund_amount) AS total_refund_value,
    countIf(return_condition = 'damaged') AS damaged_returns,
    round(countIf(return_condition = 'damaged') / count() * 100, 2) AS damage_rate_pct
FROM returns
WHERE requested_at >= today() - 90
GROUP BY sku, product_name, category
ORDER BY return_count DESC
LIMIT 50;
```

## Restocking Efficiency

Measure how quickly returned items are restocked for resale:

```sql
SELECT
    warehouse_id,
    return_condition,
    count() AS returns_received,
    countIf(restocked_at IS NOT NULL) AS restocked_count,
    avg(dateDiff('hour', received_at, restocked_at)) AS avg_restock_hours
FROM returns
WHERE received_at IS NOT NULL
  AND received_at >= today() - 30
GROUP BY warehouse_id, return_condition
ORDER BY avg_restock_hours DESC;
```

## Summary

ClickHouse makes returns analytics straightforward - aggregating return reasons, measuring refund cycle times, flagging high-return SKUs, and benchmarking restocking efficiency. These insights help reduce return rates through better product listings and optimize reverse logistics for lower processing costs.
