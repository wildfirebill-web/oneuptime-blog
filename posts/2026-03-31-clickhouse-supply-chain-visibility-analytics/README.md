# How to Build Supply Chain Visibility Analytics with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Supply Chain, Visibility, Analytics, Logistics, Tracking

Description: Build end-to-end supply chain visibility in ClickHouse by tracking events across suppliers, warehouses, and carriers to identify bottlenecks and delays.

---

Supply chain visibility requires aggregating events from dozens of systems - supplier confirmations, warehouse scans, carrier updates, and customs clearances. ClickHouse unifies these event streams for real-time tracking and analytics.

## Supply Chain Event Table

```sql
CREATE TABLE supply_chain_events (
    event_id      UUID,
    order_id      String,
    shipment_id   String,
    event_type    LowCardinality(String),  -- ordered, confirmed, picked, shipped, in_transit, delivered
    actor_type    LowCardinality(String),  -- supplier, warehouse, carrier, customs
    actor_id      LowCardinality(String),
    location      String,
    country       LowCardinality(String),
    latitude      Nullable(Float32),
    longitude     Nullable(Float32),
    event_at      DateTime,
    notes         String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_at)
ORDER BY (order_id, event_at);
```

## Order Status - Latest Event Per Order

Track where every order is right now:

```sql
SELECT
    order_id,
    argMax(event_type, event_at) AS current_status,
    argMax(location, event_at) AS current_location,
    argMax(actor_id, event_at) AS current_actor,
    max(event_at) AS last_updated
FROM supply_chain_events
WHERE event_at >= today() - 30
GROUP BY order_id
ORDER BY last_updated DESC;
```

## Lead Time by Supplier

Measure supplier-to-warehouse lead times:

```sql
SELECT
    actor_id AS supplier_id,
    avg(dateDiff('day', min_event, max_event)) AS avg_lead_days,
    quantile(0.9)(dateDiff('day', min_event, max_event)) AS p90_lead_days,
    count() AS orders
FROM (
    SELECT
        order_id,
        actor_id,
        min(event_at) AS min_event,
        max(event_at) AS max_event
    FROM supply_chain_events
    WHERE actor_type = 'supplier'
      AND event_type IN ('confirmed', 'shipped')
      AND event_at >= today() - 90
    GROUP BY order_id, actor_id
)
GROUP BY supplier_id
ORDER BY avg_lead_days DESC;
```

## Bottleneck Detection - Time at Each Stage

Identify which supply chain stage introduces the most delay:

```sql
SELECT
    event_type,
    count() AS transitions,
    avg(hours_in_stage) AS avg_hours,
    quantile(0.95)(hours_in_stage) AS p95_hours
FROM (
    SELECT
        order_id,
        event_type,
        dateDiff('hour', event_at,
            lead(event_at) OVER (PARTITION BY order_id ORDER BY event_at)) AS hours_in_stage
    FROM supply_chain_events
    WHERE event_at >= today() - 60
)
WHERE hours_in_stage > 0
GROUP BY event_type
ORDER BY avg_hours DESC;
```

## On-Time Delivery Rate by Carrier

```sql
SELECT
    actor_id AS carrier_id,
    count() AS deliveries,
    countIf(event_type = 'delivered') AS delivered_count,
    -- assumes promised_delivery_date stored in notes or a separate column
    round(countIf(event_type = 'delivered') / count() * 100, 2) AS delivery_rate_pct
FROM supply_chain_events
WHERE actor_type = 'carrier'
  AND event_at >= today() - 90
GROUP BY carrier_id
ORDER BY delivery_rate_pct;
```

## Geographic Shipment Distribution

Visualize where shipments are concentrated:

```sql
SELECT
    country,
    count() AS event_count,
    countDistinct(order_id) AS orders_in_country
FROM supply_chain_events
WHERE event_at >= today() - 30
  AND latitude IS NOT NULL
GROUP BY country
ORDER BY orders_in_country DESC;
```

## Summary

ClickHouse provides a unified platform for supply chain visibility analytics - ingesting events from every stage, computing lead times, identifying bottlenecks, and measuring carrier performance. Real-time event tracking and fast aggregations make it easy to surface delays before they impact customers.
