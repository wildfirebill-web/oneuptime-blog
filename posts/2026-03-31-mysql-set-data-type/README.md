# How to Use SET Data Type in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Mysql, Set, Data Types, String Types, Database Design

Description: The MySQL SET data type stores zero or more values from a predefined list of allowed string members, stored internally as a bitmask for efficient queries.

---

## What Is the SET Data Type

The `SET` data type in MySQL is a string type that can hold zero or more values chosen from a predefined list of allowed members defined at column creation time. Unlike `ENUM`, which allows exactly one value, a `SET` column can contain multiple values simultaneously.

`SET` is stored internally as an integer bitmask, where each bit corresponds to one member of the set. This makes storage compact and certain queries efficient.

A `SET` column can have a maximum of 64 members.

## Syntax

```sql
column_name SET('value1', 'value2', 'value3', ...)
```

## Creating a Table with SET

```sql
CREATE TABLE articles (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    tags SET('technology', 'science', 'health', 'sports', 'politics', 'entertainment'),
    published_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Inserting SET Values

When inserting, provide the selected values as a comma-separated string:

```sql
-- Single value
INSERT INTO articles (title, tags)
VALUES ('AI Revolution', 'technology');

-- Multiple values
INSERT INTO articles (title, tags)
VALUES ('Climate and Health', 'science,health');

-- Three values
INSERT INTO articles (title, tags)
VALUES ('Olympics Coverage', 'sports,health,technology');

-- Empty set
INSERT INTO articles (title, tags)
VALUES ('Uncategorized', '');
```

## Querying SET Columns

To find rows where a SET column contains a specific value, use the `FIND_IN_SET()` function:

```sql
-- Find all articles tagged with 'technology'
SELECT id, title, tags
FROM articles
WHERE FIND_IN_SET('technology', tags) > 0;
```

You can also use bitwise operations for efficiency since SET values are stored as integers:

```sql
-- Using LIKE (less efficient but readable)
SELECT title, tags FROM articles
WHERE tags LIKE '%technology%';
```

The most reliable approach is `FIND_IN_SET()`:

```sql
-- Find articles tagged with both 'science' AND 'health'
SELECT title, tags
FROM articles
WHERE FIND_IN_SET('science', tags) > 0
  AND FIND_IN_SET('health', tags) > 0;
```

## Using SET with ENUM Comparison

`SET` values support direct equality comparison for exact matches of the entire set:

```sql
-- Find articles tagged with exactly 'technology' and nothing else
SELECT title, tags FROM articles WHERE tags = 'technology';

-- Find articles with exactly 'science,health' (order matters internally)
SELECT title, tags FROM articles WHERE tags = 'science,health';
```

## Checking the Allowed Members

You can inspect the allowed SET members using `SHOW COLUMNS`:

```sql
SHOW COLUMNS FROM articles LIKE 'tags';
```

Output:

```text
+-------+----------------------------------------------+------+-----+-------------------+---+
| Field | Type                                         | Null | Key | Default           |...
+-------+----------------------------------------------+------+-----+-------------------+---+
| tags  | set('technology','science','health','sports') | YES  |     | NULL              |   |
+-------+----------------------------------------------+------+-----+-------------------+---+
```

## Internal Storage

MySQL stores `SET` values as integers using one to eight bytes depending on the number of members:

| Members | Storage |
|---|---|
| 1-8 | 1 byte |
| 9-16 | 2 bytes |
| 17-24 | 3 bytes |
| 25-32 | 4 bytes |
| 33-64 | 8 bytes |

Each bit in the integer represents whether a particular member is included. The first member in the definition corresponds to bit 1 (value 1), the second to bit 2 (value 2), the third to bit 3 (value 4), and so on.

## Modifying SET Members

To add a new allowed value to a SET column, use `ALTER TABLE`:

```sql
ALTER TABLE articles
MODIFY COLUMN tags SET('technology', 'science', 'health', 'sports', 'politics', 'entertainment', 'business');
```

Be careful when removing members - existing rows that contain the removed value will have that value stripped out.

## SET vs ENUM

| Feature | SET | ENUM |
|---|---|---|
| Values per row | Zero or more | Exactly one |
| Max members | 64 | 65535 |
| Storage | Bitmask integer | Integer |
| Query with | FIND_IN_SET() | Direct comparison |
| Use case | Multi-select tags, flags | Single-select categories |

## Practical Example: User Permissions

`SET` is well suited for storing a small, fixed set of permissions or feature flags:

```sql
CREATE TABLE user_permissions (
    user_id INT UNSIGNED NOT NULL,
    permissions SET('read', 'write', 'delete', 'admin', 'export') DEFAULT 'read',
    PRIMARY KEY (user_id)
);

INSERT INTO user_permissions (user_id, permissions)
VALUES
    (1, 'read'),
    (2, 'read,write'),
    (3, 'read,write,delete'),
    (4, 'read,write,delete,admin,export');

-- Find all users who have 'admin' permission
SELECT user_id, permissions
FROM user_permissions
WHERE FIND_IN_SET('admin', permissions) > 0;
```

## Summary

The MySQL `SET` data type stores zero or more values from a fixed list of members defined at column creation time. It is stored as a compact integer bitmask and is useful for multi-value flag fields, tags, or permission masks. Use `FIND_IN_SET()` for reliable membership queries, and prefer `SET` over comma-separated `VARCHAR` columns when the allowed values are known and finite. For single-value selections, use `ENUM` instead.
