# How to Back Up Specific Tables with mysqldump in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Backup, mysqldump, Table, Recovery

Description: Learn how to use mysqldump to back up specific tables from a MySQL database, including filtering by table name and backing up multiple tables at once.

---

Sometimes you need to back up only specific tables rather than an entire database - for example, when archiving a large log table, migrating specific data between environments, or creating targeted backups before a risky schema change. `mysqldump` supports table-level granularity when specifying the database name followed by individual table names.

## Backing Up a Single Table

```bash
mysqldump -u root -p database_name table_name > table_backup.sql
```

Example - back up the `orders` table from `myapp`:

```bash
mysqldump -u root -p myapp orders > /backup/orders_$(date +%Y%m%d).sql
```

## Backing Up Multiple Tables

List multiple table names after the database name:

```bash
mysqldump -u root -p myapp orders order_items customers \
  > /backup/orders_and_customers_$(date +%Y%m%d).sql
```

## Consistent Table Backup with InnoDB

Use `--single-transaction` for a non-locking consistent backup of InnoDB tables:

```bash
mysqldump -u root -p \
  --single-transaction \
  --quick \
  myapp orders order_items \
  > /backup/orders_consistent_$(date +%Y%m%d).sql
```

## Backing Up Table Structure Only (No Data)

Use `--no-data` when you only need the schema:

```bash
mysqldump -u root -p \
  --no-data \
  myapp orders \
  > /backup/orders_schema_only.sql
```

## Backing Up Data Only (No CREATE TABLE)

Use `--no-create-info` to export only `INSERT` statements:

```bash
mysqldump -u root -p \
  --no-create-info \
  myapp orders \
  > /backup/orders_data_only.sql
```

## Filtering Rows with --where

Back up only a subset of rows matching a condition:

```bash
# Back up orders from the last 30 days only
mysqldump -u root -p \
  --single-transaction \
  --where="created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)" \
  myapp orders \
  > /backup/recent_orders.sql
```

```bash
# Back up a specific customer's data
mysqldump -u root -p \
  --where="customer_id = 1234" \
  myapp orders \
  > /backup/customer_1234_orders.sql
```

## Compressing Table Backups

```bash
mysqldump -u root -p myapp large_log_table | \
  gzip > /backup/large_log_$(date +%Y%m%d).sql.gz
```

## Scripting Multiple Table Backups

```bash
#!/bin/bash
DB="myapp"
BACKUP_DIR="/backup/tables"
DATE=$(date +%Y%m%d_%H%M%S)
TABLES=("orders" "order_items" "customers" "products")

mkdir -p "${BACKUP_DIR}/${DATE}"

for TABLE in "${TABLES[@]}"; do
  mysqldump -u root -p"${MYSQL_PASS}" \
    --single-transaction \
    "${DB}" "${TABLE}" \
    | gzip > "${BACKUP_DIR}/${DATE}/${TABLE}.sql.gz"
  echo "Backed up table: ${TABLE}"
done
```

## Restoring a Single Table Backup

```bash
mysql -u root -p myapp < /backup/orders_20260331.sql
```

If restoring into a different database:

```bash
mysql -u root -p target_db < /backup/orders_20260331.sql
```

## Summary

`mysqldump` backs up specific tables by listing table names after the database name. Use `--where` to filter rows, `--no-data` for schema-only exports, and `--single-transaction` for consistent InnoDB backups without locking. Compress output with `gzip` for storage efficiency, and script multi-table backups to automate routine archival operations.
