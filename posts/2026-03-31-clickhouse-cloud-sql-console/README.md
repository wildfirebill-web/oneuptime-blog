# How to Use ClickHouse Cloud SQL Console

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ClickHouse Cloud, SQL Console, Query Editor, Data Exploration, UI

Description: Learn how to use the ClickHouse Cloud SQL Console to run queries, explore schemas, visualize results, and share queries with your team.

---

The ClickHouse Cloud SQL Console is a browser-based query editor built directly into the ClickHouse Cloud portal. It provides a convenient way to run ad-hoc queries, explore your schema, and share results without installing any client tools.

## Accessing the SQL Console

1. Log in to [clickhouse.cloud](https://clickhouse.cloud)
2. Open your service
3. Click "SQL Console" in the left navigation

The console connects to your service using your current ClickHouse Cloud credentials - no separate password needed.

## Running Your First Query

```sql
SELECT
    database,
    name,
    engine,
    total_rows,
    formatReadableSize(total_bytes) AS size
FROM system.tables
WHERE database NOT IN ('system', 'information_schema')
ORDER BY total_bytes DESC;
```

Type the query in the editor and press Ctrl+Enter (or Cmd+Enter on Mac) to execute.

## Exploring the Schema Browser

On the left panel of the SQL Console, you can:
- Browse all databases and tables
- Click a table name to see its columns and types
- Preview sample rows by clicking the table preview icon

## Writing Multi-Statement Queries

Use semicolons to run multiple statements:

```sql
USE analytics;

SELECT count() FROM events WHERE event_date = today();

SELECT
    event_type,
    count() AS cnt
FROM events
WHERE event_date = today()
GROUP BY event_type
ORDER BY cnt DESC;
```

Select specific statements with your cursor to run only those.

## Visualizing Results

After running a query, click the "Chart" tab above the results to create:
- Bar charts
- Line charts
- Area charts
- Scatter plots

Configure axes from the result column names.

## Saving and Sharing Queries

Click "Save" to store a query with a name. Saved queries appear in the "Saved Queries" panel and can be shared with team members who have access to the same service.

## Downloading Results

Click the download icon in the results panel to export data as:
- CSV
- JSON
- Parquet

## Keyboard Shortcuts

```text
Ctrl+Enter   - Run query
Ctrl+/       - Comment/uncomment line
Ctrl+Space   - Trigger autocomplete
F1           - Open command palette
```

## Summary

The ClickHouse Cloud SQL Console provides a fully-featured query editor in your browser with schema browsing, multi-statement support, result visualization, and query sharing. It is ideal for ad-hoc analysis, debugging, and exploring data without needing a local ClickHouse client installation.
