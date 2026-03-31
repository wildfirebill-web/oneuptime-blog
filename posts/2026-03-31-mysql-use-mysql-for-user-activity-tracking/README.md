# How to Use MySQL for User Activity Tracking

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Activity Tracking, Analytics, Partition, Schema

Description: Learn how to design a MySQL user activity tracking system with an append-only events table, time-based partitioning, and aggregation queries for behavioral analytics.

---

User activity tracking records what users do in an application: page views, clicks, searches, purchases, and feature interactions. MySQL handles this workload through an append-only design with time-based partitioning to manage data growth and fast aggregation queries for behavioral analytics.

## Activity Events Schema

```sql
CREATE TABLE user_activities (
  id           BIGINT UNSIGNED AUTO_INCREMENT,
  user_id      BIGINT UNSIGNED,
  session_id   CHAR(36),
  event_name   VARCHAR(100) NOT NULL,
  page         VARCHAR(500),
  referrer     VARCHAR(500),
  properties   JSON,
  ip_address   VARCHAR(45),
  user_agent   TEXT,
  occurred_at  DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id, occurred_at),
  INDEX idx_user_event (user_id, event_name, occurred_at),
  INDEX idx_session (session_id, occurred_at),
  INDEX idx_event_time (event_name, occurred_at)
) ENGINE=InnoDB
PARTITION BY RANGE (TO_DAYS(occurred_at)) (
  PARTITION p2026_01 VALUES LESS THAN (TO_DAYS('2026-02-01')),
  PARTITION p2026_02 VALUES LESS THAN (TO_DAYS('2026-03-01')),
  PARTITION p2026_03 VALUES LESS THAN (TO_DAYS('2026-04-01')),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

## Recording Activity Events

```sql
INSERT INTO user_activities (user_id, session_id, event_name, page, properties)
VALUES
  (42, 'sess-abc123', 'page_view', '/products/widget',
   '{"time_on_page_ms": 3400, "scroll_depth_pct": 75}'),
  (42, 'sess-abc123', 'add_to_cart', '/products/widget',
   '{"product_id": 99, "sku": "WGT-001", "price": 29.99}'),
  (NULL, 'sess-xyz789', 'page_view', '/blog/mysql-tips', NULL);
```

Null `user_id` handles anonymous visitors before login.

## Querying a User's Activity Timeline

```sql
SELECT occurred_at, event_name, page, properties
FROM user_activities
WHERE user_id = 42
  AND occurred_at >= NOW() - INTERVAL 7 DAY
ORDER BY occurred_at DESC
LIMIT 50;
```

## Analyzing Event Funnel Conversion

Track how many users completed each step of a conversion funnel:

```sql
SELECT
  event_name,
  COUNT(DISTINCT user_id) AS unique_users,
  COUNT(*) AS total_events
FROM user_activities
WHERE event_name IN ('product_view', 'add_to_cart', 'checkout_start', 'purchase')
  AND occurred_at >= '2026-03-01'
  AND occurred_at <  '2026-04-01'
  AND user_id IS NOT NULL
GROUP BY event_name
ORDER BY FIELD(event_name, 'product_view', 'add_to_cart', 'checkout_start', 'purchase');
```

## Daily Active Users

```sql
SELECT
  DATE(occurred_at) AS activity_date,
  COUNT(DISTINCT user_id) AS dau
FROM user_activities
WHERE occurred_at >= '2026-03-01'
  AND occurred_at <  '2026-04-01'
  AND user_id IS NOT NULL
GROUP BY activity_date
ORDER BY activity_date;
```

## Most Common User Paths

Find the most frequent two-event sequences per session:

```sql
SELECT
  a1.event_name AS step_1,
  a2.event_name AS step_2,
  COUNT(*) AS transitions
FROM user_activities a1
JOIN user_activities a2
  ON a2.session_id = a1.session_id
  AND a2.occurred_at > a1.occurred_at
  AND a2.occurred_at <= a1.occurred_at + INTERVAL 5 MINUTE
GROUP BY step_1, step_2
ORDER BY transitions DESC
LIMIT 20;
```

## Managing Retention

Drop old partitions to enforce data retention policy:

```sql
ALTER TABLE user_activities DROP PARTITION p2026_01;
```

## Summary

MySQL user activity tracking uses an append-only events table with JSON properties for flexible event metadata, partitioned by month for efficient retention management, and indexed by user, session, and event type for the three common query patterns. Funnel analysis, DAU metrics, and path analysis all run efficiently when queries are scoped to narrow time ranges that benefit from partition pruning.
