# How to Use ClickHouse with Segment for Customer Data

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Segment, Customer Data, Analytics, Event Tracking

Description: Route Segment event data into ClickHouse for a SQL-queryable customer data store that enables cross-channel analytics beyond what Segment provides natively.

---

## Why Use ClickHouse as a Segment Destination

Segment collects events and routes them to destinations. Using ClickHouse as a destination gives you:

- A full SQL-queryable copy of all customer events
- Join capability with your existing business data
- Custom retention policies and data ownership
- Unlimited query flexibility beyond Segment's built-in analytics

## Setting Up the Segment ClickHouse Destination

Segment supports ClickHouse via its Warehouse destination category. Configure it in Segment:

1. Go to Destinations and add a Warehouse
2. Select ClickHouse
3. Enter connection details:

```text
Host: ch.internal
Port: 8443
Database: segment
Username: segment_writer
Password: <secret>
SSL: true
```

Grant appropriate permissions:

```sql
CREATE USER segment_writer
    IDENTIFIED WITH sha256_password BY 'strong_password'
    HOST IP '52.25.130.38', '52.25.153.220';  -- Segment IPs

GRANT CREATE TABLE, INSERT, SELECT ON segment.* TO segment_writer;
```

## Segment Schema in ClickHouse

Segment creates tables per event type with a consistent schema:

```sql
-- segment.order_completed (auto-created)
-- Contains standard Segment fields + your track properties
SELECT
    id,
    user_id,
    anonymous_id,
    received_at,
    sent_at,
    original_timestamp,
    -- Your custom properties
    order_id,
    revenue,
    currency,
    product_category
FROM segment.order_completed
LIMIT 5;
```

## Cross-Event Funnel Analysis

Combine multiple Segment event tables for funnel analysis:

```sql
SELECT
    step,
    count(DISTINCT user_id) AS users
FROM (
    SELECT user_id, 1 AS step FROM segment.product_viewed WHERE received_at >= today() - 7
    UNION ALL
    SELECT user_id, 2 AS step FROM segment.checkout_started WHERE received_at >= today() - 7
    UNION ALL
    SELECT user_id, 3 AS step FROM segment.order_completed WHERE received_at >= today() - 7
) t
GROUP BY step
ORDER BY step;
```

## Joining Segment Events with Your Database

Enrich Segment data with your internal user records:

```sql
SELECT
    oc.user_id,
    u.plan_type,
    u.company_size,
    count() AS orders,
    sum(oc.revenue) AS total_revenue
FROM segment.order_completed oc
JOIN your_db.users u ON u.user_id = oc.user_id
WHERE oc.received_at >= today() - 30
GROUP BY oc.user_id, u.plan_type, u.company_size
ORDER BY total_revenue DESC
LIMIT 20;
```

## Replay and Backfill

Segment supports replaying historical data to new destinations. When you add ClickHouse as a destination, replay your full event history:

```bash
# In Segment: Destinations > ClickHouse > Replay History
# Select date range and source
# Segment re-sends all events to ClickHouse
```

## Handling Schema Evolution

Segment events evolve as your product changes. ClickHouse handles new properties by adding columns via the Segment connector's schema evolution feature. You can also add columns manually:

```sql
ALTER TABLE segment.order_completed
    ADD COLUMN IF NOT EXISTS discount_code String;
```

## Summary

Using ClickHouse as a Segment destination creates a versioned, SQL-queryable copy of all customer event data that can be joined with internal systems, enabling analytics and segmentation capabilities far beyond what Segment's built-in tools provide.
