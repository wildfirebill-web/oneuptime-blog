# How to Export a MySQL Table to a SQL File

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Export, mysqldump, SQL, Backup

Description: Learn how to export a single MySQL table to a SQL file using mysqldump, including structure, data, and options for targeted table backups.

---

Exporting a single table to a SQL file is useful for targeted backups before schema changes, migrating individual tables between databases or servers, or sharing specific data with colleagues. `mysqldump` is the standard tool for this and supports many options to control exactly what gets exported.

## Basic Table Export

```bash
mysqldump -u root -p database_name table_name > table_export.sql
```

Example:

```bash
mysqldump -u root -p myapp orders > /backup/orders_$(date +%Y%m%d).sql
```

The output file contains:

```sql
-- Table structure
DROP TABLE IF EXISTS `orders`;
CREATE TABLE `orders` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `user_id` int NOT NULL,
  `total` decimal(12,2) NOT NULL,
  `status` enum('pending','paid','shipped') NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

-- Table data
INSERT INTO `orders` VALUES (1, 42, 99.99, 'paid'), ...;
```

## Consistent Export for InnoDB Tables

Use `--single-transaction` to avoid locking:

```bash
mysqldump -u root -p \
  --single-transaction \
  --quick \
  myapp orders \
  > /backup/orders_consistent.sql
```

## Exporting Only the Table Structure (No Data)

```bash
mysqldump -u root -p \
  --no-data \
  myapp orders \
  > /backup/orders_schema.sql
```

## Exporting Only Data (No CREATE TABLE)

```bash
mysqldump -u root -p \
  --no-create-info \
  myapp orders \
  > /backup/orders_data_only.sql
```

## Exporting with DROP TABLE Statement

Include a `DROP TABLE IF EXISTS` before the `CREATE TABLE` for clean re-import:

```bash
mysqldump -u root -p \
  --add-drop-table \
  myapp orders \
  > /backup/orders_drop_first.sql
```

This is the default behavior, so this flag is usually not necessary unless `--skip-add-drop-table` was previously used.

## Exporting with Compressed Output

```bash
mysqldump -u root -p \
  --single-transaction \
  myapp large_table \
  | gzip > /backup/large_table_$(date +%Y%m%d).sql.gz
```

## Exporting to a Specific Character Set

If your table uses a non-default character set:

```bash
mysqldump -u root -p \
  --default-character-set=utf8mb4 \
  myapp users \
  > /backup/users_utf8mb4.sql
```

## Exporting a Subset of Rows

```bash
# Export only orders from the last 90 days
mysqldump -u root -p \
  --where="created_at >= DATE_SUB(NOW(), INTERVAL 90 DAY)" \
  myapp orders \
  > /backup/recent_orders.sql
```

## Viewing the Export Before Saving

Preview the first 50 lines without writing to disk:

```bash
mysqldump -u root -p \
  --single-transaction \
  myapp orders \
  | head -50
```

## Importing the Table SQL File

```bash
# Import into same database
mysql -u root -p myapp < /backup/orders_20260331.sql

# Import into a different database
mysql -u root -p other_db < /backup/orders_20260331.sql
```

## Verifying the Import

```sql
SELECT COUNT(*) FROM orders;
SELECT * FROM orders ORDER BY id DESC LIMIT 5;
```

## Summary

Exporting a MySQL table to a SQL file with `mysqldump` is straightforward - specify the database and table name after authentication. Add `--single-transaction` for consistent InnoDB exports, `--no-data` for schema-only, and `--where` for partial row exports. Compress with `gzip` for large tables, and always verify the import with a row count check after restoration.
