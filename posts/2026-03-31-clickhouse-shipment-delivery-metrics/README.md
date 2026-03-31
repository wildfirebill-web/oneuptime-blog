# How to Track Shipment and Delivery Metrics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Shipment, Delivery, Logistics, Analytics, On-Time

Description: Track shipment and delivery performance metrics in ClickHouse including on-time delivery rates, exception rates, carrier SLAs, and regional performance.

---

Shipment and delivery analytics help logistics teams monitor carrier performance, identify late deliveries, and improve customer experience. ClickHouse efficiently processes millions of shipment records.

## Shipments Table

```sql
CREATE TABLE shipments (
    shipment_id      String,
    order_id         String,
    carrier          LowCardinality(String),
    service_level    LowCardinality(String),  -- overnight, 2day, ground
    origin_country   LowCardinality(String),
    dest_country     LowCardinality(String),
    dest_zip         String,
    weight_kg        Float32,
    created_at       DateTime,
    promised_date    Date,
    actual_delivery  Nullable(Date),
    status           LowCardinality(String),  -- in_transit, delivered, exception, returned
    exception_reason Nullable(LowCardinality(String))
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (carrier, created_at, shipment_id);
```

## On-Time Delivery Rate

```sql
SELECT
    carrier,
    service_level,
    count() AS total_shipments,
    countIf(actual_delivery <= promised_date) AS on_time,
    countIf(actual_delivery > promised_date) AS late,
    round(countIf(actual_delivery <= promised_date) / countIf(actual_delivery IS NOT NULL) * 100, 2) AS otd_pct
FROM shipments
WHERE status = 'delivered'
  AND created_at >= today() - 30
GROUP BY carrier, service_level
ORDER BY otd_pct;
```

## Average Delivery Days

Compare actual vs. promised transit times:

```sql
SELECT
    carrier,
    service_level,
    avg(dateDiff('day', toDate(created_at), actual_delivery)) AS avg_actual_days,
    avg(dateDiff('day', toDate(created_at), promised_date)) AS avg_promised_days,
    avg(dateDiff('day', toDate(created_at), actual_delivery)) -
        avg(dateDiff('day', toDate(created_at), promised_date)) AS avg_delay_days
FROM shipments
WHERE status = 'delivered'
  AND actual_delivery IS NOT NULL
  AND created_at >= today() - 60
GROUP BY carrier, service_level
ORDER BY avg_delay_days DESC;
```

## Exception Rate by Carrier

```sql
SELECT
    carrier,
    exception_reason,
    count() AS exception_count,
    round(count() / sum(count()) OVER (PARTITION BY carrier) * 100, 2) AS reason_share_pct
FROM shipments
WHERE status = 'exception'
  AND created_at >= today() - 30
GROUP BY carrier, exception_reason
ORDER BY carrier, exception_count DESC;
```

## Regional Delivery Performance

Identify zip codes or regions with chronic delivery issues:

```sql
SELECT
    dest_country,
    dest_zip,
    count() AS shipments,
    round(countIf(actual_delivery > promised_date) / countIf(actual_delivery IS NOT NULL) * 100, 2) AS late_pct,
    avg(dateDiff('day', promised_date, actual_delivery)) AS avg_days_late
FROM shipments
WHERE status = 'delivered'
  AND created_at >= today() - 90
GROUP BY dest_country, dest_zip
HAVING late_pct > 20 AND shipments > 10
ORDER BY late_pct DESC
LIMIT 50;
```

## In-Transit Age - Stale Shipments

Flag shipments that have been in transit longer than the SLA:

```sql
SELECT
    shipment_id,
    carrier,
    service_level,
    toDate(created_at) AS ship_date,
    promised_date,
    today() - promised_date AS days_overdue
FROM shipments
WHERE status = 'in_transit'
  AND today() > promised_date
ORDER BY days_overdue DESC
LIMIT 200;
```

## Summary

ClickHouse makes it straightforward to track shipment KPIs - on-time delivery rates, exception analysis, transit time comparisons, and regional performance are all computed efficiently over millions of shipment records. Carriers can be benchmarked, late delivery patterns identified, and stale in-transit shipments surfaced for proactive customer communication.
