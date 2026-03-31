# How to Use clickhouse-git-import for Repository Analysis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, clickhouse-git-import, Git Analysis, Developer Analytics, Utility

Description: Learn how to use clickhouse-git-import to load Git repository history into ClickHouse for powerful commit analytics, contributor stats, and code churn analysis.

---

`clickhouse-git-import` parses a Git repository and imports its commit history, file changes, and blame data into ClickHouse tables. This enables SQL-based analysis of code evolution, contributor activity, and repository health.

## Installation

Available in the ClickHouse tools package:

```bash
which clickhouse-git-import
# /usr/bin/clickhouse-git-import
```

## Basic Import

Navigate to a Git repository and run the import:

```bash
cd /path/to/your/repo

clickhouse-git-import \
  --host localhost \
  --port 9000 \
  --database git_analytics \
  --skip-paths ".vendor,.git,node_modules"
```

The tool creates and populates several tables automatically.

## Tables Created

```sql
-- See what was created
SHOW TABLES FROM git_analytics;
```

Key tables include:

```text
commits         - one row per commit (hash, author, date, message)
file_changes    - per-file changes per commit (additions, deletions)
line_changes    - per-line blame data (author, sign, line content)
```

## Example Queries

### Top Contributors by Commits

```sql
SELECT
    author,
    count() AS commits,
    sum(lines_added) AS added,
    sum(lines_deleted) AS deleted
FROM git_analytics.file_changes
GROUP BY author
ORDER BY commits DESC
LIMIT 20;
```

### Files with Most Churn

```sql
SELECT
    path,
    count() AS change_count,
    sum(lines_added + lines_deleted) AS total_lines_changed
FROM git_analytics.file_changes
GROUP BY path
ORDER BY change_count DESC
LIMIT 20;
```

### Commit Activity by Day of Week

```sql
SELECT
    toDayOfWeek(time) AS day_of_week,
    count() AS commits
FROM git_analytics.commits
GROUP BY day_of_week
ORDER BY day_of_week;
```

### Recent Commits by Author

```sql
SELECT
    hash,
    author,
    time,
    message
FROM git_analytics.commits
WHERE time >= now() - INTERVAL 30 DAY
ORDER BY time DESC
LIMIT 50;
```

## Analyzing Multiple Repositories

Import multiple repositories into the same database with different table prefixes by running the tool separately and joining on author or timestamp.

## Re-importing After New Commits

The import is a full snapshot. For incremental updates, re-run the import periodically (it truncates and reimports):

```bash
clickhouse-git-import --host localhost --port 9000 --database git_analytics
```

## Summary

`clickhouse-git-import` turns any Git repository into a queryable ClickHouse dataset, enabling deep analytics on code history, contributor patterns, and file-level churn using standard SQL.
