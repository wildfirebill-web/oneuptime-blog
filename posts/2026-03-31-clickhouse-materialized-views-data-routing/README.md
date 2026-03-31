# How to Use Materialized Views for Data Routing in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Materialized View, Data Routing, Event Streaming, Architecture

Description: Learn how to use ClickHouse materialized views to route incoming data to different tables based on content, enabling efficient multi-tenant or event-type partitioning.

---

## Data Routing with Materialized Views

Data routing means directing rows from one source table to different destination tables based on column values. In ClickHouse, you create multiple materialized views on the same source table, each with a WHERE clause to select specific rows.

## Source Table (Landing Zone)

Create a single landing table for all incoming events:

```sql
CREATE TABLE events_landing (
    event_time DateTime,
    tenant_id LowCardinality(String),
    event_type LowCardinality(String),
    payload String
) ENGINE = Null;  -- Null engine discards data after views process it
```

Using the `Null` engine means data is not stored in the landing table, only passed to views.

## Route by Event Type

```sql
-- Destination tables
CREATE TABLE order_events (
    event_time DateTime,
    tenant_id LowCardinality(String),
    order_id String,
    amount Float64
) ENGINE = MergeTree() ORDER BY (tenant_id, event_time);

CREATE TABLE user_events (
    event_time DateTime,
    tenant_id LowCardinality(String),
    user_id String,
    action LowCardinality(String)
) ENGINE = MergeTree() ORDER BY (tenant_id, event_time);

-- Route order events
CREATE MATERIALIZED VIEW mv_route_orders
TO order_events
AS SELECT
    event_time,
    tenant_id,
    JSONExtractString(payload, 'order_id') AS order_id,
    JSONExtractFloat(payload, 'amount') AS amount
FROM events_landing
WHERE event_type = 'order';

-- Route user events
CREATE MATERIALIZED VIEW mv_route_users
TO user_events
AS SELECT
    event_time,
    tenant_id,
    JSONExtractString(payload, 'user_id') AS user_id,
    JSONExtractString(payload, 'action') AS action
FROM events_landing
WHERE event_type = 'user_action';
```

## Route by Tenant (Multi-Tenant Sharding)

```sql
-- Separate table per major tenant
CREATE MATERIALIZED VIEW mv_tenant_acme
TO events_tenant_acme
AS SELECT event_time, event_type, payload
FROM events_landing
WHERE tenant_id = 'acme';

CREATE MATERIALIZED VIEW mv_tenant_other
TO events_tenant_other
AS SELECT event_time, event_type, payload
FROM events_landing
WHERE tenant_id != 'acme';
```

## Route by Priority

```sql
CREATE MATERIALIZED VIEW mv_high_priority_errors
TO critical_errors
AS SELECT
    event_time,
    tenant_id,
    JSONExtractString(payload, 'message') AS message
FROM events_landing
WHERE event_type = 'error'
  AND JSONExtractString(payload, 'severity') = 'critical';
```

## Verify Routing

```sql
-- Insert test record
INSERT INTO events_landing VALUES (now(), 'acme', 'order', '{"order_id":"123","amount":99.99}');

-- Verify routing
SELECT * FROM order_events WHERE tenant_id = 'acme' LIMIT 5;
SELECT * FROM events_tenant_acme LIMIT 5;
```

## Routing with Transformation

Combine routing and transformation in the same view:

```sql
CREATE MATERIALIZED VIEW mv_route_and_enrich_orders
TO enriched_order_events
AS SELECT
    event_time,
    tenant_id,
    JSONExtractString(payload, 'order_id') AS order_id,
    JSONExtractFloat(payload, 'amount') AS amount,
    dictGetString('currency_dict', 'name',
        JSONExtractString(payload, 'currency')) AS currency_name
FROM events_landing
WHERE event_type = 'order';
```

## Summary

Materialized views on a `Null` engine landing table create a powerful and lightweight data routing layer in ClickHouse. You can split event streams by type, tenant, or priority without extra infrastructure, while optionally applying transformations and enrichments at the same time.
