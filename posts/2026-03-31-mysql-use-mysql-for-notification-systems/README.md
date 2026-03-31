# How to Use MySQL for Notification Systems

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Notification, Queue, Schema, Performance

Description: Learn how to build a MySQL notification system with delivery tracking, read status management, bulk queries, and efficient polling patterns for in-app notifications.

---

In-app notification systems store, deliver, and track notification read status for users. MySQL handles this workload well because notifications are relatively low-volume, require durable storage, and benefit from relational queries to count unread items and load notification feeds.

## Notification Schema

```sql
CREATE TABLE notifications (
  id           BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  recipient_id BIGINT UNSIGNED NOT NULL,
  type         VARCHAR(100) NOT NULL,
  title        VARCHAR(300) NOT NULL,
  body         TEXT,
  action_url   VARCHAR(500),
  metadata     JSON,
  is_read      TINYINT(1) NOT NULL DEFAULT 0,
  read_at      DATETIME,
  created_at   DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  expires_at   DATETIME,
  INDEX idx_recipient_read (recipient_id, is_read, created_at),
  INDEX idx_created_at (created_at)
) ENGINE=InnoDB;
```

The composite index `(recipient_id, is_read, created_at)` serves the most common query: load unread notifications for a user, ordered by creation time.

## Creating Notifications

```sql
INSERT INTO notifications (recipient_id, type, title, body, action_url, metadata)
VALUES
  (42, 'order_shipped', 'Your order has shipped',
   'Order #1001 is on its way. Expected delivery: April 3.',
   '/orders/1001',
   '{"order_id": 1001, "tracking_number": "1Z999AA1"}'),
  (42, 'comment_reply', 'New reply to your comment',
   'Alice replied to your comment on "MySQL Best Practices".',
   '/posts/mysql-best-practices#comment-55',
   '{"post_id": 12, "comment_id": 55}');
```

## Loading the Notification Feed

```sql
SELECT id, type, title, body, action_url, is_read, created_at
FROM notifications
WHERE recipient_id = 42
  AND (expires_at IS NULL OR expires_at > NOW())
ORDER BY created_at DESC
LIMIT 20;
```

## Unread Count

```sql
SELECT COUNT(*) AS unread_count
FROM notifications
WHERE recipient_id = 42
  AND is_read = 0
  AND (expires_at IS NULL OR expires_at > NOW());
```

This query uses the `(recipient_id, is_read, created_at)` index and runs in microseconds.

## Marking Notifications as Read

Mark a single notification:

```sql
UPDATE notifications
SET is_read = 1, read_at = NOW()
WHERE id = 999 AND recipient_id = 42;
```

Mark all as read in bulk:

```sql
UPDATE notifications
SET is_read = 1, read_at = NOW()
WHERE recipient_id = 42 AND is_read = 0;
```

## Bulk Notifications (Fan-Out)

For system-wide announcements, insert one row per recipient:

```sql
INSERT INTO notifications (recipient_id, type, title, body)
SELECT id, 'system_announcement', 'Scheduled maintenance tonight',
       'The system will be offline from 2:00 AM to 3:00 AM UTC on April 1.'
FROM users
WHERE is_active = 1;
```

For very large user bases (millions), process in batches to avoid long lock times:

```sql
INSERT INTO notifications (recipient_id, type, title, body)
SELECT id, 'system_announcement', 'Scheduled maintenance tonight',
       'The system will be offline from 2:00 AM to 3:00 AM UTC on April 1.'
FROM users
WHERE is_active = 1 AND id BETWEEN 1 AND 50000;
```

## Cleaning Up Old Notifications

```sql
DELETE FROM notifications
WHERE created_at < NOW() - INTERVAL 90 DAY
  AND is_read = 1
LIMIT 10000;
```

Run this deletion on a schedule in batches to avoid long table locks.

## Summary

A MySQL notification system stores one row per recipient-notification pair with a composite index on `(recipient_id, is_read, created_at)` for fast unread feed queries. Mark-as-read operations update individual or all rows efficiently. Fan-out for bulk notifications uses INSERT ... SELECT in batches, and a scheduled cleanup job removes old read notifications to keep the table size manageable.
