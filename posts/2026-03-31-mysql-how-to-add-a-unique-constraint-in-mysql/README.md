# How to Add a Unique Constraint in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, DDL, Constraint, Schema Design

Description: Learn how to add unique constraints in MySQL to enforce column uniqueness using ALTER TABLE, including multi-column unique indexes and NULL handling.

---

## What Is a Unique Constraint?

A unique constraint ensures that no two rows in a table have the same value in the constrained column(s). In MySQL, a unique constraint is implemented as a unique index. Unlike a primary key, a unique column can contain NULL values (and multiple NULLs are allowed).

## Adding a Unique Constraint with ALTER TABLE

```sql
ALTER TABLE table_name
ADD CONSTRAINT constraint_name UNIQUE (column_name);
```

Or without a named constraint:

```sql
ALTER TABLE table_name
ADD UNIQUE (column_name);
```

## Simple Example

```sql
ALTER TABLE users
ADD CONSTRAINT uq_users_email UNIQUE (email);
```

Now no two users can share the same email address.

## Composite Unique Constraint

A unique constraint on multiple columns ensures the combination is unique:

```sql
ALTER TABLE team_memberships
ADD CONSTRAINT uq_team_member UNIQUE (team_id, user_id);
```

Each user can appear in the same team only once, but the same user can be in different teams.

## Adding a Unique Index (Equivalent)

In MySQL, `ADD UNIQUE INDEX` and `ADD UNIQUE KEY` are synonyms for `ADD UNIQUE`:

```sql
ALTER TABLE users
ADD UNIQUE INDEX idx_uq_username (username);

-- Same as
ALTER TABLE users
ADD UNIQUE KEY idx_uq_username (username);
```

## Defining Unique Constraints at Table Creation

```sql
CREATE TABLE users (
    id       INT          NOT NULL AUTO_INCREMENT,
    username VARCHAR(50)  NOT NULL,
    email    VARCHAR(100) NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uq_username (username),
    UNIQUE KEY uq_email (email)
);
```

## Checking for Duplicate Values Before Adding

Before adding a unique constraint, verify no duplicates exist:

```sql
SELECT email, COUNT(*) AS cnt
FROM users
GROUP BY email
HAVING cnt > 1;
```

If duplicates exist, resolve them first. MySQL will reject the `ALTER TABLE` if duplicates are found.

## NULL Handling in Unique Constraints

MySQL allows multiple NULL values in a unique column - NULLs are not considered equal:

```sql
-- Both inserts succeed even with unique constraint on phone
INSERT INTO users (username, phone) VALUES ('Alice', NULL);
INSERT INTO users (username, phone) VALUES ('Bob', NULL);

-- This fails - duplicate non-NULL value
INSERT INTO users (username, phone) VALUES ('Carol', '555-1234');
INSERT INTO users (username, phone) VALUES ('Dave', '555-1234');  -- Error
```

## Dropping a Unique Constraint

```sql
ALTER TABLE users
DROP INDEX uq_users_email;
```

Or by constraint name if named differently:

```sql
ALTER TABLE users
DROP KEY uq_email;
```

## Checking Existing Unique Constraints

```sql
SHOW INDEX FROM users WHERE Non_unique = 0;

-- Or via INFORMATION_SCHEMA
SELECT INDEX_NAME, COLUMN_NAME
FROM INFORMATION_SCHEMA.STATISTICS
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'users'
  AND NON_UNIQUE = 0
  AND INDEX_NAME != 'PRIMARY';
```

## Online DDL

Adding a unique constraint in MySQL 8.0 with InnoDB can be done online:

```sql
ALTER TABLE users
ADD UNIQUE KEY uq_phone (phone),
ALGORITHM=INPLACE, LOCK=NONE;
```

MySQL validates uniqueness across all existing rows during the operation.

## Summary

Unique constraints in MySQL prevent duplicate non-NULL values in one or more columns. Add them with `ALTER TABLE ... ADD UNIQUE (column)` or with a named constraint. Multiple NULL values are allowed since NULLs are treated as distinct. Check for duplicate values before adding the constraint. Use composite unique constraints to enforce uniqueness on combinations of columns, and `ALGORITHM=INPLACE, LOCK=NONE` for online addition on large InnoDB tables.
