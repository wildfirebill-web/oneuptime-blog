# How to Implement Insert If Not Exists in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Insert, Unique, InnoDB, Concurrency

Description: Learn how to implement insert-if-not-exists in MySQL using INSERT IGNORE, ON DUPLICATE KEY UPDATE, and SELECT-based guards with proper concurrency handling.

---

## The Problem

You want to insert a row only when it does not already exist, without raising a duplicate-key error. MySQL provides several approaches with different trade-offs.

## Option 1: INSERT IGNORE

`INSERT IGNORE` silently discards the row if it would violate a `UNIQUE` or `PRIMARY KEY` constraint:

```sql
INSERT IGNORE INTO tags (slug, name)
VALUES ('javascript', 'JavaScript');
```

The row is inserted only if `slug` is not already present. If it exists, MySQL ignores the statement and returns 0 affected rows with no error. Use `ROW_COUNT()` to detect whether insertion happened:

```sql
SELECT ROW_COUNT() AS inserted;
-- 1 = inserted, 0 = already existed
```

Warning: `INSERT IGNORE` also suppresses other errors such as data-type mismatches and NULL violations. Use it only when you are confident the unique constraint is the only possible conflict.

## Option 2: INSERT ... ON DUPLICATE KEY UPDATE (No-Op)

To avoid the broad suppression of `INSERT IGNORE`, use `ON DUPLICATE KEY UPDATE` with a no-op assignment:

```sql
INSERT INTO tags (slug, name)
VALUES ('javascript', 'JavaScript')
ON DUPLICATE KEY UPDATE slug = slug;
```

This updates `slug` to its own value when a duplicate is found, which is a no-op. The row is never changed on conflict, but other errors still surface normally.

## Option 3: INSERT ... SELECT with NOT EXISTS

This pattern checks for existence before inserting:

```sql
INSERT INTO subscriptions (user_id, newsletter_id)
SELECT 42, 7
WHERE NOT EXISTS (
  SELECT 1 FROM subscriptions
  WHERE user_id = 42 AND newsletter_id = 7
);
```

It reads cleanly but is subject to a race condition under concurrent inserts. Two concurrent sessions may both pass the `NOT EXISTS` check before either commits. Use a `UNIQUE` constraint as a safety net to catch the resulting duplicate-key error.

## Option 4: Using a UNIQUE Constraint as the Guard

The safest approach combines a `UNIQUE` constraint with `INSERT IGNORE` or `ON DUPLICATE KEY`:

```sql
ALTER TABLE subscriptions
  ADD UNIQUE KEY uq_user_newsletter (user_id, newsletter_id);

INSERT IGNORE INTO subscriptions (user_id, newsletter_id)
VALUES (42, 7);
```

Even under concurrent load, only one insert succeeds. The constraint enforces correctness at the storage layer.

## Checking the Result in Application Code

```javascript
const [result] = await db.execute(
  'INSERT IGNORE INTO tags (slug, name) VALUES (?, ?)',
  ['javascript', 'JavaScript']
);

if (result.affectedRows === 1) {
  console.log('Tag inserted');
} else {
  console.log('Tag already exists');
}
```

## Batch Insert If Not Exists

For bulk operations, multi-row `INSERT IGNORE` is efficient:

```sql
INSERT IGNORE INTO tags (slug, name) VALUES
  ('javascript', 'JavaScript'),
  ('python',     'Python'),
  ('golang',     'Go');
```

Existing rows are skipped; new rows are inserted - all in one round trip.

## Summary

The recommended approach for insert-if-not-exists in MySQL is to define a `UNIQUE` constraint on the relevant columns and use `INSERT IGNORE` or `INSERT ... ON DUPLICATE KEY UPDATE slug = slug` to handle duplicates gracefully. Always rely on the constraint for correctness rather than application-level `SELECT` checks, which are vulnerable to race conditions under concurrent traffic.
