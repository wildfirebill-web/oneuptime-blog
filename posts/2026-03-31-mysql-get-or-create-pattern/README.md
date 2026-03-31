# How to Implement Get or Create Pattern in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Insert, SELECT, Concurrency, Pattern

Description: Learn how to implement the get-or-create pattern in MySQL to atomically retrieve an existing row or insert a new one without race conditions.

---

## What Is Get or Create?

Get or create (also called find or create) is a common application pattern: look up a row by a natural key, return it if it exists, or insert it and return the new row. This pattern is frequently needed for lookups like finding or creating a user by email, a tag by name, or a session by token.

## Naive Approach and Its Race Condition

A simple SELECT followed by a conditional INSERT has a race condition:

```sql
-- Session A and Session B can both run this SELECT simultaneously
SELECT id FROM tags WHERE slug = 'mysql';
-- Both see 0 rows
-- Both then INSERT - one succeeds, one gets a duplicate-key error
INSERT INTO tags (slug, name) VALUES ('mysql', 'MySQL');
```

Under concurrent load this fails without extra protection.

## Approach 1: INSERT IGNORE + SELECT

Use `INSERT IGNORE` to attempt the insert, then always SELECT to retrieve the row:

```sql
INSERT IGNORE INTO tags (slug, name)
VALUES ('mysql', 'MySQL');

SELECT id, slug, name
FROM tags
WHERE slug = 'mysql';
```

This works because even if the insert was a no-op (duplicate), the SELECT returns the existing row. Requires a `UNIQUE` index on `slug`.

```sql
ALTER TABLE tags ADD UNIQUE KEY uq_slug (slug);
```

## Approach 2: ON DUPLICATE KEY UPDATE + LAST_INSERT_ID()

A trick to retrieve the ID after upsert without a second query:

```sql
INSERT INTO tags (slug, name)
VALUES ('mysql', 'MySQL')
ON DUPLICATE KEY UPDATE id = LAST_INSERT_ID(id);

SELECT LAST_INSERT_ID() AS id;
```

When the row is inserted, `LAST_INSERT_ID()` returns the new auto-increment ID. When a duplicate is found, the update sets `LAST_INSERT_ID(id)` to the existing ID, so `LAST_INSERT_ID()` returns the existing row's ID in both cases.

## Approach 3: Stored Procedure for Atomicity

Wrap the logic in a stored procedure for clean reuse:

```sql
DELIMITER $$
CREATE PROCEDURE get_or_create_tag(
  IN p_slug VARCHAR(100),
  IN p_name VARCHAR(255),
  OUT p_id  INT UNSIGNED
)
BEGIN
  INSERT IGNORE INTO tags (slug, name) VALUES (p_slug, p_name);
  SELECT id INTO p_id FROM tags WHERE slug = p_slug;
END$$
DELIMITER ;
```

Call it from application code:

```sql
CALL get_or_create_tag('mysql', 'MySQL', @tag_id);
SELECT @tag_id;
```

## Approach 4: Application-Level with Error Handling

In application code, attempt the insert and handle the duplicate-key error to fall back to a select:

```javascript
async function getOrCreateTag(slug, name) {
  try {
    const [result] = await db.execute(
      'INSERT INTO tags (slug, name) VALUES (?, ?)',
      [slug, name]
    );
    return { id: result.insertId, slug, name, created: true };
  } catch (err) {
    if (err.code === 'ER_DUP_ENTRY') {
      const [rows] = await db.execute(
        'SELECT id, slug, name FROM tags WHERE slug = ?',
        [slug]
      );
      return { ...rows[0], created: false };
    }
    throw err;
  }
}
```

This is robust under concurrent traffic because the `UNIQUE` constraint guarantees only one insert succeeds.

## Summary

The get-or-create pattern in MySQL is best implemented with a `UNIQUE` constraint plus either `INSERT IGNORE` followed by a `SELECT`, or the `ON DUPLICATE KEY UPDATE id = LAST_INSERT_ID(id)` trick to retrieve the ID in a single round trip. Avoid SELECT-then-INSERT patterns without a constraint as they fail under concurrency. The `ER_DUP_ENTRY` catch-and-select approach in application code is equally safe and idiomatic.
