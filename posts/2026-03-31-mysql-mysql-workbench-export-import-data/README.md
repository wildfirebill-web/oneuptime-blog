# How to Export and Import Data with MySQL Workbench

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Workbench, Export, Import, Backup

Description: Learn how to export and import MySQL databases using MySQL Workbench's Data Export and Data Import wizards for backups and migrations.

---

## Introduction

MySQL Workbench provides built-in Data Export and Data Import tools that wrap `mysqldump` and `mysql` in a graphical interface. These tools allow you to backup databases, export specific tables, and restore from SQL dump files without using the command line. They are suitable for small-to-medium datasets and development workflows.

## Exporting Data

Navigate to:

```text
Server > Data Export
```

### Selecting Schemas and Tables

The left panel lists all schemas. Check the schemas to export. Below each schema, you can select individual tables:

```text
[x] ecommerce
    [x] customers
    [x] orders
    [ ] audit_log
```

### Export Options

```text
Export to Dump Project Folder:  /backups/workbench_exports/
Export to Self-Contained File:  /backups/ecommerce_2026_03_31.sql

[x] Include Create Schema
[x] Dump Stored Procedures and Functions
[x] Dump Events
[x] Dump Triggers
```

Choose **Dump Project Folder** to get one file per table, or **Self-Contained File** for a single combined SQL file.

### Starting the Export

Click **Start Export**. Workbench runs:

```bash
mysqldump --defaults-file=/tmp/tmpXXX --host=127.0.0.1 --port=3306 \
  --user=root --protocol=tcp --single-transaction \
  --routines --events --triggers \
  ecommerce customers orders > /backups/ecommerce_2026_03_31.sql
```

Progress appears in the output log. On completion:

```text
Export completed successfully
Duration: 00:00:12
```

## Importing Data

Navigate to:

```text
Server > Data Import
```

### Import from a Self-Contained File

Select a single `.sql` file:

```text
Import from Self-Contained File: /backups/ecommerce_2026_03_31.sql
Default Target Schema: ecommerce
```

### Import from a Dump Project Folder

If you exported as a project folder:

```text
Import from Dump Project Folder: /backups/workbench_exports/
```

Select which schemas and tables to import from the folder.

### Starting the Import

Click **Start Import**. Workbench runs:

```bash
mysql --defaults-file=/tmp/tmpXXX --host=127.0.0.1 --port=3306 \
  --user=root --protocol=tcp < /backups/ecommerce_2026_03_31.sql
```

Monitor the output log for errors. Common issues:

```text
ERROR 1046 (3D000): No database selected
```

Fix by specifying the target schema or including `USE schema_name;` at the top of the dump file.

## Exporting Query Results

From the SQL editor, after running a query, export the result set:

```sql
SELECT * FROM orders WHERE created_at > '2026-01-01';
```

Click the **Export** button in the result grid toolbar and choose:

```text
Export as CSV
Export as JSON
Export as XML
Export as HTML
```

## Limitations vs. MySQL Shell

Workbench Data Export uses `mysqldump` which is single-threaded. For large databases, prefer MySQL Shell utilities:

```javascript
// Much faster for large datasets
util.dumpSchemas(["ecommerce"], "/backups/ecommerce_dump", {threads: 8})
util.loadDump("/backups/ecommerce_dump", {threads: 8})
```

## Summary

MySQL Workbench Data Export and Import provide a graphical interface for `mysqldump`-based backups and restores. They are ideal for small datasets, development exports, and quick backups. For production backups of large databases, use MySQL Shell's parallel dump utilities (`util.dumpInstance()`, `util.loadDump()`) for significantly better performance.
