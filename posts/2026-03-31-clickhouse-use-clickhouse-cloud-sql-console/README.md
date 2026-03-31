# How to Use ClickHouse Cloud SQL Console

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Cloud, SQL Console, Query, Dashboard, Analytics

Description: A practical guide to using the ClickHouse Cloud SQL Console for writing queries, exploring schemas, and managing your ClickHouse Cloud databases.

---

The ClickHouse Cloud SQL Console is a browser-based query interface that lets you explore data, run SQL queries, and manage databases without installing any client tools. It provides schema browsing, query history, saved queries, and basic data visualization.

## Accessing the SQL Console

Log in to `console.clickhouse.cloud`, select your service, and click **Open SQL Console**. The console opens in a new tab connected directly to your service.

## Interface Overview

The SQL Console is divided into three main areas:

- **Left sidebar** - Schema browser showing databases, tables, and columns
- **Query editor** - Where you write and execute SQL
- **Results panel** - Displays query output with column headers and data types

## Writing and Running Queries

Click in the query editor and type your SQL. Use `Ctrl+Enter` (or `Cmd+Enter` on macOS) to run a query:

```sql
SELECT
    database,
    table,
    formatReadableSize(sum(bytes)) AS size,
    sum(rows) AS rows
FROM system.parts
WHERE active = 1
GROUP BY database, table
ORDER BY sum(bytes) DESC
LIMIT 20;
```

## Using the Schema Browser

Expand the left sidebar to explore your databases and tables. Clicking a table name auto-populates a `SELECT * FROM table LIMIT 100` query so you can preview data quickly.

To see column definitions for a table:

```sql
DESCRIBE TABLE my_database.my_table;
```

## Multiple Query Tabs

You can open multiple query tabs by clicking the **+** icon near the tab bar. Each tab maintains its own query history and session state, making it convenient to work on multiple queries simultaneously.

## Saving and Sharing Queries

To save a query, click the **Save** button and provide a name. Saved queries appear in the **Saved Queries** panel and can be shared with team members who have access to the same ClickHouse Cloud organization.

## Query History

Access recent queries by clicking **History** in the toolbar. The history shows timestamps, execution duration, and rows returned, making it easy to re-run previous queries:

```sql
-- Example of a common analytics query you might save
SELECT
    toStartOfHour(event_time) AS hour,
    count() AS events,
    uniq(user_id) AS unique_users
FROM events
WHERE event_date = today()
GROUP BY hour
ORDER BY hour;
```

## Exporting Results

After running a query, click **Download** in the results panel to export data as CSV or JSON. This is useful for sharing data with stakeholders or importing into other tools.

## Query Settings

Click the **Settings** gear icon in the editor toolbar to adjust per-query settings:

```text
max_execution_time = 60
max_rows_to_read = 1000000000
```

These settings override service-level defaults for the current query session.

## Tips for Effective Use

- Use the `EXPLAIN` statement before heavy queries to understand the execution plan
- Leverage the `FORMAT` clause to see results in different formats like `Pretty` or `JSON`
- Use `WITH` (CTE) syntax for complex queries to improve readability
- Monitor the **Query Duration** indicator in the results panel to catch slow queries early

## Summary

The ClickHouse Cloud SQL Console provides a convenient, zero-install environment for querying and managing your ClickHouse databases. With features like schema browsing, saved queries, and export options, it serves as an effective tool for both ad-hoc analysis and day-to-day database management.
