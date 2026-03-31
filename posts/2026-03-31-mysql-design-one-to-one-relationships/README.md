# How to Design One-to-One Relationships in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Relationship, Schema Design, Foreign Key

Description: Learn how to model one-to-one relationships in MySQL using foreign keys and unique constraints, with practical schema examples.

---

A one-to-one relationship means each row in table A relates to exactly one row in table B and vice versa. MySQL does not enforce this natively, so you implement it through a unique foreign key on the child table.

## When to Use One-to-One

Split a table into two when columns are rarely accessed together, contain sensitive data you want to isolate, or belong to different security domains. Common examples include separating `users` from `user_profiles`, or `orders` from `order_shipping_details`.

## Basic Schema

```sql
CREATE TABLE users (
    id       INT UNSIGNED NOT NULL AUTO_INCREMENT,
    email    VARCHAR(255) NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uq_email (email)
);

CREATE TABLE user_profiles (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT,
    user_id    INT UNSIGNED NOT NULL,
    bio        TEXT,
    avatar_url VARCHAR(512),
    PRIMARY KEY (id),
    UNIQUE KEY uq_user_id (user_id),
    CONSTRAINT fk_profile_user
        FOREIGN KEY (user_id) REFERENCES users (id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);
```

The `UNIQUE KEY` on `user_id` enforces the one-to-one constraint. `ON DELETE CASCADE` removes the profile automatically when the parent user is deleted.

## Shared Primary Key Pattern

An alternative avoids the separate auto-increment and saves a join column:

```sql
CREATE TABLE user_profiles (
    user_id    INT UNSIGNED NOT NULL,
    bio        TEXT,
    avatar_url VARCHAR(512),
    PRIMARY KEY (user_id),
    CONSTRAINT fk_profile_user
        FOREIGN KEY (user_id) REFERENCES users (id)
        ON DELETE CASCADE
);
```

Here `user_id` serves as both the primary key and the foreign key, guaranteeing one-to-one at the storage level.

## Querying the Relationship

```sql
SELECT u.email, p.bio, p.avatar_url
FROM   users u
LEFT JOIN user_profiles p ON p.user_id = u.id
WHERE  u.id = 42;
```

Use `LEFT JOIN` when the child row is optional (nullable relationship) and `INNER JOIN` when the child row must always exist.

## Enforcing Not-Null Existence

If every user must have a profile, consider using a deferred approach: insert both rows in a single transaction to avoid chicken-and-egg constraint violations.

```sql
START TRANSACTION;

INSERT INTO users (email) VALUES ('alice@example.com');
SET @uid = LAST_INSERT_ID();

INSERT INTO user_profiles (user_id, bio)
VALUES (@uid, 'Software engineer');

COMMIT;
```

## Checking for Missing Profiles

```sql
SELECT u.id, u.email
FROM   users u
LEFT JOIN user_profiles p ON p.user_id = u.id
WHERE  p.user_id IS NULL;
```

This query identifies users without a profile row, useful for data integrity audits.

## Summary

Design one-to-one relationships in MySQL by adding a unique foreign key on the child table or by sharing the primary key. Use `ON DELETE CASCADE` to keep data consistent, wrap multi-table inserts in transactions, and use `LEFT JOIN` to detect optional child rows. The shared primary key pattern is the most storage-efficient option when the child row always exists.
