# How to Use MySQL Workbench for Query Editing and Execution

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Workbench, Query, Editor, Execution

Description: Learn how to use MySQL Workbench's SQL editor to write, execute, and analyze queries with syntax highlighting, autocomplete, and result export features.

---

## Introduction

MySQL Workbench includes a full-featured SQL editor that supports syntax highlighting, code completion, query history, and result set management. It is a powerful tool for ad-hoc query development, data analysis, and running database maintenance scripts. This guide covers the key features of the Workbench query editor.

## Opening the SQL Editor

To open the SQL editor, connect to a MySQL server from the Workbench home screen by clicking on a stored connection. The editor opens with a new query tab.

You can also open additional query tabs with:

```text
File > New Query Tab  (Ctrl+T / Cmd+T)
```

## Writing and Executing Queries

Type SQL directly in the editor pane. To execute:

- **Execute All** - runs all statements (lightning bolt icon, Ctrl+Shift+Enter)
- **Execute Current Statement** - runs the statement where the cursor is (Ctrl+Enter)

Example query:

```sql
SELECT
  o.id,
  c.email,
  o.total,
  o.status,
  o.created_at
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'pending'
ORDER BY o.created_at DESC
LIMIT 100;
```

Results appear in the **Result Grid** below the editor.

## Code Completion and Syntax Help

MySQL Workbench provides context-aware autocomplete. Press **Ctrl+Space** to trigger suggestions for:

- Schema names
- Table names
- Column names
- SQL keywords and functions

## Query History

All executed queries are stored in the query history panel. Access it via:

```text
View > Query History Panel
```

You can search history, copy past queries, or re-run them directly.

## Explain Plan

To analyze query performance, click the **Explain** button (magnifying glass with lightning bolt) or use:

```text
Query > Explain Current Statement
```

This shows the query execution plan:

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42 AND status = 'pending';
```

The result includes columns like `type`, `key`, `rows`, and `Extra` to help identify missing indexes or full table scans.

## Using the Visual EXPLAIN

Workbench also offers a Visual EXPLAIN that renders the query plan as a flowchart, making it easier to spot bottlenecks such as full table scans, index range scans, and join types.

## Working with Result Sets

The result grid supports:

- Inline editing of cell values
- Sorting by column (click column headers)
- Filtering rows
- Exporting to CSV, JSON, or XML via the **Export** button

To export results:

```text
Result Grid > Export > CSV
```

## Snippets and Bookmarks

Save frequently used queries as snippets:

```text
Snippets Panel > Add New Snippet
```

Name the snippet and paste the SQL. Snippets are accessible from the left sidebar for quick access.

## Running Multiple Statements

Separate statements with semicolons. Workbench executes them in sequence:

```sql
START TRANSACTION;

UPDATE accounts SET balance = balance - 500 WHERE id = 1;
UPDATE accounts SET balance = balance + 500 WHERE id = 2;

COMMIT;
```

## Formatting SQL

Use the auto-format feature to clean up messy SQL:

```text
Edit > Format > Beautify Query  (Ctrl+B)
```

## Summary

MySQL Workbench's SQL editor provides everything needed for productive query development - autocomplete, explain plans, query history, result export, and snippet management. Use Execute Current Statement for iterative query development and Visual EXPLAIN to quickly identify performance issues without reading raw explain output.
