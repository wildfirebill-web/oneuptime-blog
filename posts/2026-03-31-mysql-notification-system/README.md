# How to Implement a Notification System in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Schema, Pattern, Notification, Index

Description: Learn how to design a MySQL notification system that supports multiple notification types, read tracking, and efficient unread count queries.

---

## Notification System Schema

A notification system must store what happened, who it's for, whether it has been read, and enough context to render the notification message without additional queries.

```sql
CREATE TABLE notifications (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    recipient_id INT NOT NULL,
    actor_id INT NOT NULL,           -- Who triggered the notification
    type VARCHAR(50) NOT NULL,       -- 'like', 'comment', 'follow', 'mention'
    entity_type VARCHAR(50),         -- 'post', 'comment', 'user'
    entity_id INT,                   -- ID of the related entity
    data JSON,                       -- Extra context for rendering
    is_read TINYINT(1) NOT NULL DEFAULT 0,
    read_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_recipient_read (recipient_id, is_read, created_at),
    INDEX idx_recipient_created (recipient_id, created_at)
);
```

## Creating Notifications

```sql
-- Notify user 2 that user 1 liked their post
INSERT INTO notifications
    (recipient_id, actor_id, type, entity_type, entity_id, data)
VALUES
    (2, 1, 'like', 'post', 42,
     JSON_OBJECT('post_title', 'How to Use MySQL', 'actor_name', 'Alice'));

-- Notify user 3 that user 1 started following them
INSERT INTO notifications
    (recipient_id, actor_id, type, entity_type, entity_id, data)
VALUES
    (3, 1, 'follow', 'user', 1,
     JSON_OBJECT('actor_name', 'Alice', 'actor_username', 'alice'));

-- Notify user 2 about a new comment on their post
INSERT INTO notifications
    (recipient_id, actor_id, type, entity_type, entity_id, data)
VALUES
    (2, 4, 'comment', 'post', 42,
     JSON_OBJECT('post_title', 'How to Use MySQL', 'comment_excerpt', 'Great post!'));
```

## Fetching Notifications for a User

```sql
-- Get paginated notifications for user 2
SELECT
    n.id,
    n.type,
    n.entity_type,
    n.entity_id,
    n.data,
    n.is_read,
    n.created_at,
    u.username AS actor_username,
    u.display_name AS actor_name
FROM notifications n
JOIN users u ON n.actor_id = u.id
WHERE n.recipient_id = 2
ORDER BY n.created_at DESC
LIMIT 20 OFFSET 0;
```

## Unread Count (Notification Badge)

```sql
-- Fast unread count using the indexed (recipient_id, is_read) columns
SELECT COUNT(*) AS unread_count
FROM notifications
WHERE recipient_id = 2
  AND is_read = 0;
```

## Marking Notifications as Read

```sql
-- Mark a single notification as read
UPDATE notifications
SET is_read = 1, read_at = CURRENT_TIMESTAMP
WHERE id = 101 AND recipient_id = 2;

-- Mark all notifications as read for a user
UPDATE notifications
SET is_read = 1, read_at = CURRENT_TIMESTAMP
WHERE recipient_id = 2
  AND is_read = 0;

-- Mark notifications of a specific type as read
UPDATE notifications
SET is_read = 1, read_at = CURRENT_TIMESTAMP
WHERE recipient_id = 2
  AND type = 'like'
  AND is_read = 0;
```

## Notification Deduplication

Prevent duplicate notifications for the same event (e.g., multiple likes on the same post from the same user):

```sql
-- Add unique constraint to prevent duplicate notifications
ALTER TABLE notifications
    ADD UNIQUE KEY uq_notification
        (recipient_id, actor_id, type, entity_type, entity_id);

-- Use INSERT IGNORE or ON DUPLICATE KEY for idempotent inserts
INSERT IGNORE INTO notifications
    (recipient_id, actor_id, type, entity_type, entity_id, data)
VALUES
    (2, 1, 'like', 'post', 42,
     JSON_OBJECT('post_title', 'How to Use MySQL'));
```

## Notification Cleanup

```sql
-- Archive or delete notifications older than 90 days
DELETE FROM notifications
WHERE recipient_id = 2
  AND created_at < NOW() - INTERVAL 90 DAY
  AND is_read = 1;

-- Scheduled cleanup event
CREATE EVENT cleanup_old_notifications
ON SCHEDULE EVERY 1 DAY
DO
    DELETE FROM notifications
    WHERE created_at < NOW() - INTERVAL 90 DAY
      AND is_read = 1;
```

## Summary

A MySQL notification system centers on a flexible `notifications` table with a JSON `data` column for rendering context and a composite index on `(recipient_id, is_read, created_at)` for fast unread queries. Storing all notification context in the row avoids N+1 queries when rendering notification lists, while deduplication constraints and scheduled cleanup events keep the table manageable at scale.
