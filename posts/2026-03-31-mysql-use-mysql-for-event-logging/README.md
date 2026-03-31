# How to Use MySQL for Event Logging

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Event, Logging, Audit, Partition

Description: Learn how to implement a structured event log in MySQL with append-only writes, range partitioning for retention management, and efficient query patterns for audit and analytics.

---

MySQL is a practical event log store for applications that need structured, queryable audit trails, user activity history, or system event records. The key design principles are append-only inserts, partitioning by time for efficient retention, and indexed queries by entity and event type.

## Event Log Schema

Design the table to be append-only: no updates or deletes, only inserts:

```sql
CREATE TABLE event_log (
  id           BIGINT UNSIGNED AUTO_INCREMENT,
  event_type   VARCHAR(100) NOT NULL,
  entity_type  VARCHAR(100) NOT NULL,
  entity_id    BIGINT UNSIGNED NOT NULL,
  actor_id     BIGINT UNSIGNED,
  actor_type   VARCHAR(50) NOT NULL DEFAULT 'user',
  payload      JSON,
  ip_address   VARCHAR(45),
  occurred_at  DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id, occurred_at),
  INDEX idx_entity (entity_type, entity_id, occurred_at),
  INDEX idx_actor  (actor_id, occurred_at),
  INDEX idx_event_type (event_type, occurred_at)
) ENGINE=InnoDB
PARTITION BY RANGE (TO_DAYS(occurred_at)) (
  PARTITION p2026_q1 VALUES LESS THAN (TO_DAYS('2026-04-01')),
  PARTITION p2026_q2 VALUES LESS THAN (TO_DAYS('2026-07-01')),
  PARTITION p2026_q3 VALUES LESS THAN (TO_DAYS('2026-10-01')),
  PARTITION p2026_q4 VALUES LESS THAN (TO_DAYS('2027-01-01')),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

## Writing Events

Insert events without ever updating or deleting rows:

```sql
INSERT INTO event_log (event_type, entity_type, entity_id, actor_id, payload, ip_address)
VALUES
  ('order.placed',   'order', 1001, 42, '{"total": 150.00, "items": 3}',  '203.0.113.1'),
  ('order.shipped',  'order', 1001, 99, '{"tracking": "1Z999AA1012345678"}', NULL),
  ('user.login',     'user',    42, 42, '{"method": "password"}',          '203.0.113.1');
```

## Querying the Audit Trail for an Entity

```sql
SELECT occurred_at, event_type, actor_id, payload
FROM event_log
WHERE entity_type = 'order'
  AND entity_id = 1001
ORDER BY occurred_at ASC;
```

## Querying Recent Activity by Actor

```sql
SELECT occurred_at, event_type, entity_type, entity_id
FROM event_log
WHERE actor_id = 42
  AND occurred_at >= NOW() - INTERVAL 7 DAY
ORDER BY occurred_at DESC
LIMIT 50;
```

## Aggregating Event Counts for Analytics

```sql
SELECT
  event_type,
  DATE(occurred_at) AS event_date,
  COUNT(*) AS event_count
FROM event_log
WHERE occurred_at >= '2026-03-01'
  AND occurred_at <  '2026-04-01'
GROUP BY event_type, event_date
ORDER BY event_date, event_count DESC;
```

## Managing Retention with Partitions

Drop old quarters of data instantly without row-by-row deletes:

```sql
-- Drop data older than 1 year
ALTER TABLE event_log DROP PARTITION p2025_q1;

-- Add a new future partition
ALTER TABLE event_log REORGANIZE PARTITION p_future INTO (
  PARTITION p2027_q1 VALUES LESS THAN (TO_DAYS('2027-04-01')),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

## Summary

A MySQL event log uses append-only inserts, a `DATETIME(3)` timestamp for millisecond precision, and range partitioning for fast data retention management. Indexes on entity, actor, and event type support the three most common query patterns: audit trails for a specific entity, activity history for a user, and aggregate counts for analytics. Partition drops replace slow DELETE operations for retention enforcement.
