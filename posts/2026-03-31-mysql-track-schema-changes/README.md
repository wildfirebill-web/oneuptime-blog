# How to Track Schema Changes in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Schema, Migration

Description: Learn how to track MySQL schema changes over time using schema snapshots, diff tools, migration frameworks, and version-controlled DDL scripts.

---

Tracking schema changes means knowing what changed, when it changed, and who changed it. Unlike DDL event logging (which captures when a statement ran), schema change tracking focuses on the structural difference - which columns were added, removed, or modified, and how the schema evolved over the history of the database.

## Approach 1: Version-Controlled Migration Scripts

The most reliable method is to never apply schema changes manually. Use migration files tracked in version control:

```text
db/migrations/
  001_create_users.sql
  002_add_email_verified.sql
  003_create_orders.sql
  004_alter_orders_add_status.sql
```

Each migration has an up and (optionally) a down script:

```sql
-- 004_alter_orders_add_status.sql (up)
ALTER TABLE orders ADD COLUMN status ENUM('pending','completed','cancelled') NOT NULL DEFAULT 'pending';
CREATE INDEX idx_orders_status ON orders(status);
```

Track applied migrations in a schema versions table:

```sql
CREATE TABLE schema_migrations (
  version    VARCHAR(20) PRIMARY KEY,
  applied_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  applied_by VARCHAR(100)
);
```

## Approach 2: Schema Snapshots with mysqldump

Dump the schema (no data) at regular intervals to capture the state at each point in time:

```bash
#!/bin/bash
DATE=$(date +%F)
SNAPSHOT_DIR="/var/schema-snapshots"
mkdir -p "$SNAPSHOT_DIR"

mysqldump \
  --no-data \
  --routines \
  --triggers \
  --events \
  myapp > "$SNAPSHOT_DIR/schema-$DATE.sql"

# Commit to git for full diff history
cd "$SNAPSHOT_DIR"
git add "schema-$DATE.sql"
git commit -m "Schema snapshot $DATE"
```

## Approach 3: Comparing Schema with mysqldiff or skeema

Use schema diff tools to compare two schema states:

```bash
# Using mysqldiff (part of MySQL Utilities)
mysqldiff \
  --server1=user:pass@host1 \
  --server2=user:pass@host2 \
  myapp:myapp

# Example output
# ALTER TABLE `orders` ADD COLUMN `shipped_at` DATETIME NULL AFTER `status`;
```

Or use Skeema for Git-native schema management:

```bash
# Pull current schema from database
skeema pull -h 127.0.0.1 -u root -p myapp

# Compare working directory schema to database
skeema diff

# Push schema changes
skeema push
```

## Approach 4: INFORMATION_SCHEMA Polling

Periodically snapshot `INFORMATION_SCHEMA.COLUMNS` to a tracking table and compare:

```sql
CREATE TABLE schema_column_snapshots (
  snapshot_date DATE NOT NULL,
  table_schema  VARCHAR(64),
  table_name    VARCHAR(64),
  column_name   VARCHAR(64),
  data_type     VARCHAR(64),
  is_nullable   VARCHAR(3),
  column_default TEXT,
  PRIMARY KEY (snapshot_date, table_schema, table_name, column_name)
);

-- Insert today's snapshot
INSERT INTO schema_column_snapshots
SELECT CURDATE(), table_schema, table_name, column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_schema = 'myapp';
```

Find columns added since yesterday:

```sql
SELECT t.table_name, t.column_name, t.data_type, 'ADDED' AS change_type
FROM schema_column_snapshots t
LEFT JOIN schema_column_snapshots y
  ON t.table_name = y.table_name AND t.column_name = y.column_name
  AND y.snapshot_date = CURDATE() - INTERVAL 1 DAY
WHERE t.snapshot_date = CURDATE()
  AND y.column_name IS NULL;
```

## Recommended Workflow

1. Store migration scripts in Git.
2. Apply migrations via a deployment pipeline.
3. Record each migration in the `schema_migrations` table with timestamp and deployer identity.
4. Take schema snapshots nightly for audit purposes.

```bash
# CI/CD pipeline step
mysql myapp < db/migrations/005_add_product_sku.sql
mysql myapp -e "INSERT INTO schema_migrations (version, applied_by) VALUES ('005', '$CI_USER')"
```

## Summary

Tracking MySQL schema changes effectively requires version-controlled migration files as the authoritative record, supplemented by schema snapshots and `INFORMATION_SCHEMA` polling for auditing. Tools like Skeema provide Git-native workflows for schema management, making it easy to review and approve schema changes through pull requests before applying them.
