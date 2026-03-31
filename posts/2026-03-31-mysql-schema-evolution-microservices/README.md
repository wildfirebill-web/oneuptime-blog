# How to Handle Schema Evolution in MySQL for Microservices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Schema Evolution, Microservice, Migration

Description: Learn how to safely evolve MySQL schemas in microservices using backward-compatible migrations, expand-contract pattern, and zero-downtime deployment strategies.

---

## The Challenge of Schema Evolution in Microservices

In microservices, multiple service instances run simultaneously during a rolling deployment. When you change a MySQL schema, old code and new code must both work against the database at the same time. A migration that drops a column or renames a table will break the old version while the new version is still starting up.

## The Expand-Contract Pattern

The safest approach is the expand-contract pattern, which splits a breaking schema change into three phases:

```text
Phase 1 - Expand:   Add the new column/table (old code ignores it, new code writes both)
Phase 2 - Migrate:  Backfill data from old to new structure
Phase 3 - Contract: Remove the old column/table once all instances use the new code
```

## Example: Renaming a Column

Rename `full_name` to `display_name` in a users table without downtime.

### Phase 1 - Expand (backward-compatible)

```sql
-- Add the new column (old code writes to full_name, new code writes to both)
ALTER TABLE users
  ADD COLUMN display_name VARCHAR(200) NOT NULL DEFAULT '';

-- New code reads display_name, old code reads full_name - both work
```

### Phase 2 - Backfill

```sql
-- Copy existing data to the new column in batches
UPDATE users
SET display_name = full_name
WHERE display_name = ''
LIMIT 10000;

-- Repeat until all rows are migrated
-- Check remaining: SELECT COUNT(*) FROM users WHERE display_name = '';
```

After backfill, add a generated column or a trigger to keep both in sync while transitioning:

```sql
-- Temporary trigger to keep columns in sync during rollout
DELIMITER $$
CREATE TRIGGER sync_display_name
BEFORE INSERT ON users
FOR EACH ROW
BEGIN
  IF NEW.display_name = '' AND NEW.full_name != '' THEN
    SET NEW.display_name = NEW.full_name;
  END IF;
END$$
DELIMITER ;
```

### Phase 3 - Contract (after all instances run new code)

```sql
DROP TRIGGER IF EXISTS sync_display_name;
ALTER TABLE users DROP COLUMN full_name;
```

## Adding a NOT NULL Column Safely

Adding a `NOT NULL` column without a default breaks `INSERT` from old code. Use this sequence:

```sql
-- Step 1: Add as nullable
ALTER TABLE orders ADD COLUMN currency VARCHAR(3);

-- Step 2: Backfill existing rows
UPDATE orders SET currency = 'USD' WHERE currency IS NULL;

-- Step 3: After all service instances use the new code, add NOT NULL constraint
ALTER TABLE orders MODIFY COLUMN currency VARCHAR(3) NOT NULL DEFAULT 'USD';
```

## Index Changes Without Locking

In MySQL 8.0, most `ALTER TABLE` operations for indexes use online DDL and do not block reads or writes:

```sql
-- Add an index online (does not block DML)
ALTER TABLE orders
  ADD INDEX idx_status_date (status, created_at),
  ALGORITHM=INPLACE, LOCK=NONE;

-- Drop an index online
ALTER TABLE orders
  DROP INDEX idx_old_index,
  ALGORITHM=INPLACE, LOCK=NONE;
```

## Using Flyway for Service-Specific Migrations

```text
order-service/
  src/main/resources/db/migration/
    V1__create_orders.sql
    V2__add_currency.sql
    V3__rename_full_name.sql
```

```java
// Spring Boot + Flyway: auto-applies pending migrations on startup
@SpringBootApplication
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

```yaml
# application.yml
spring:
  flyway:
    locations: classpath:db/migration
    baseline-on-migrate: true
```

## Tracking Migration State

```sql
-- Flyway creates this table to track applied migrations
SELECT version, description, installed_on, success
FROM order_svc.flyway_schema_history
ORDER BY installed_rank;
```

## Schema Evolution Best Practices

```text
1. Never drop or rename columns in a single deployment
2. Always add columns as nullable or with a default value first
3. Backfill data before adding NOT NULL constraints
4. Use ALGORITHM=INPLACE, LOCK=NONE for index changes
5. Keep migrations small and focused on one change each
6. Test rollback by running old service code against the new schema
7. Each service manages its own migration files independently
```

## Summary

Schema evolution in MySQL microservices requires backward-compatible changes that allow old and new service instances to run simultaneously. The expand-contract pattern - add, backfill, then remove - prevents breaking changes during rolling deployments. Use online DDL for index changes, Flyway or Liquibase per service for migration tracking, and always verify that old code works against the new schema before completing any contract phase.
