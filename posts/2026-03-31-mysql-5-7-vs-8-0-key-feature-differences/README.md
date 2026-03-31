# MySQL 5.7 vs MySQL 8.0: Key Feature Differences

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Upgrade, Version

Description: Explore the most important changes between MySQL 5.7 and 8.0 including window functions, roles, JSON, invisible indexes, and the removed query cache.

---

MySQL 8.0 introduced more new features than any previous major release. If you are still running MySQL 5.7 (which reached end of life in October 2023), understanding what 8.0 offers helps justify the upgrade and prepares you for breaking changes.

## Window Functions

MySQL 8.0 added native window functions. In 5.7, you needed complex self-joins or user variables to achieve the same result.

```sql
-- MySQL 8.0: rank users by order total using window function
SELECT
  user_id,
  total,
  RANK() OVER (ORDER BY total DESC) AS rank_pos,
  LAG(total) OVER (ORDER BY total DESC) AS prev_total
FROM orders;

-- MySQL 5.7: complex workaround needed
SET @rank := 0;
SELECT user_id, total, @rank := @rank + 1 AS rank_pos
FROM orders ORDER BY total DESC;
```

## CTEs (Common Table Expressions)

MySQL 8.0 supports `WITH` clauses including recursive CTEs. In 5.7, you had to use derived tables or temporary tables.

```sql
-- MySQL 8.0: recursive CTE for hierarchical data
WITH RECURSIVE category_tree AS (
  SELECT id, name, parent_id, 0 AS depth
  FROM categories WHERE parent_id IS NULL
  UNION ALL
  SELECT c.id, c.name, c.parent_id, ct.depth + 1
  FROM categories c
  JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY depth, name;
```

## Roles

MySQL 8.0 introduced user roles for centralized privilege management.

```sql
-- MySQL 8.0: create and assign a role
CREATE ROLE 'app_read';
GRANT SELECT ON myapp.* TO 'app_read';
GRANT 'app_read' TO 'alice'@'%';
-- User activates role
SET ROLE 'app_read';
```

MySQL 5.7 required granting privileges directly to each user.

## Invisible Indexes

MySQL 8.0 allows marking an index as invisible so the optimizer ignores it without dropping it - useful for testing whether removing an index would hurt performance.

```sql
-- MySQL 8.0: make an index invisible for testing
ALTER TABLE orders ALTER INDEX idx_customer INVISIBLE;
-- Test queries without the index
ALTER TABLE orders ALTER INDEX idx_customer VISIBLE;
```

## Descending Indexes

MySQL 8.0 supports true descending indexes. In 5.7, `DESC` in an index definition was parsed but ignored.

```sql
-- MySQL 8.0: index optimizes ORDER BY created_at DESC
CREATE INDEX idx_created_desc ON events (created_at DESC);
```

## JSON Improvements

Both versions support JSON columns, but 8.0 adds JSON table functions, `JSON_OVERLAPS()`, `JSON_VALUE()`, and JSON partial update optimization.

```sql
-- MySQL 8.0: JSON_TABLE to convert JSON array to rows
SELECT jt.*
FROM config, JSON_TABLE(config.features, '$[*]'
  COLUMNS (name VARCHAR(50) PATH '$.name')) AS jt;
```

## Removed Features

MySQL 8.0 removed the query cache, which was deprecated in 5.7. It also changed default authentication from `mysql_native_password` to `caching_sha2_password`.

```sql
-- MySQL 8.0: create user with legacy auth for old clients
CREATE USER 'legacy_app'@'%'
IDENTIFIED WITH mysql_native_password BY 'password';
```

## Summary

MySQL 8.0 is a substantial improvement over 5.7. Window functions, CTEs, roles, invisible indexes, and descending indexes are all significant additions that simplify complex queries and administration. The removed query cache and changed default authentication require attention during migration. All production systems should be on MySQL 8.0 now that 5.7 is past end of life.
