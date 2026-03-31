# How to Use pt-index-usage for MySQL Index Analysis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Index, Percona, Query, Optimization

Description: Learn how to use pt-index-usage to analyze MySQL slow query logs and identify which indexes are never used, so you can safely drop them.

---

## What is pt-index-usage?

`pt-index-usage` is a Percona Toolkit utility that reads a MySQL slow query log, runs `EXPLAIN` on each query, and tracks which indexes are actually used. At the end of the run it reports which indexes were never referenced, making them candidates for removal. Removing unused indexes reduces storage overhead and speeds up write operations.

## Prerequisites

- A slow query log captured from the MySQL server
- The database accessible with sufficient privileges for `EXPLAIN`
- Percona Toolkit installed

Enable the slow query log if not already active:

```sql
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0;
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
```

Capture queries for a representative period (at least a few hours during peak traffic), then turn off full logging:

```sql
SET GLOBAL long_query_time = 1;
```

## Basic Usage

```bash
pt-index-usage \
  --host=127.0.0.1 \
  --user=root \
  --password=secret \
  /var/log/mysql/slow.log
```

The tool reads the slow log, `EXPLAIN`s each unique query, and prints a report.

## Example Output

```text
# INDEX USAGE REPORT
# Unused indexes for `mydb`:
#   table `orders`
#     idx_legacy_flag
#     idx_archived
#   table `customers`
#     idx_old_segment
```

These indexes were never referenced in any of the analyzed queries.

## Saving Results to a File

```bash
pt-index-usage \
  --host=127.0.0.1 \
  --user=root \
  --password=secret \
  /var/log/mysql/slow.log > index_usage_report.txt
```

## Filtering to a Specific Database

```bash
pt-index-usage \
  --host=127.0.0.1 \
  --user=root \
  --password=secret \
  --databases=mydb \
  /var/log/mysql/slow.log
```

## Understanding the Limitations

The analysis is only as good as the log sample. Consider:

- The slow log must cover all query patterns, including batch jobs and reports that run infrequently
- Indexes used only by ad-hoc queries that were not captured will appear unused
- A 24-hour window capturing peak and off-peak traffic is a minimum baseline

Run the analysis over a larger log file for more accurate results:

```bash
# Combine multiple slow log files
cat /var/log/mysql/slow.log.1 /var/log/mysql/slow.log > combined_slow.log

pt-index-usage \
  --host=127.0.0.1 \
  --user=root \
  --password=secret \
  combined_slow.log
```

## Cross-Referencing with pt-duplicate-key-checker

For a comprehensive index cleanup, run both tools:

```bash
# Find unused indexes
pt-index-usage --host=127.0.0.1 --user=root --password=secret slow.log > unused.txt

# Find duplicate indexes
pt-duplicate-key-checker --host=127.0.0.1 --user=root --password=secret --databases=mydb > dupes.txt
```

Indexes appearing in both reports are the safest candidates for removal.

## Dropping an Unused Index

After confirming an index is genuinely unused, drop it with a schema change tool like pt-online-schema-change to avoid locking:

```bash
pt-online-schema-change \
  --alter="DROP INDEX idx_legacy_flag" \
  D=mydb,t=orders \
  --host=127.0.0.1 --user=root --password=secret \
  --execute
```

## Summary

`pt-index-usage` bridges the gap between your schema and your actual query workload by using real query data to identify dead weight indexes. Capture a representative slow query log, run the analysis, cross-reference with `pt-duplicate-key-checker`, and use `pt-online-schema-change` to drop confirmed unused indexes safely in production.
