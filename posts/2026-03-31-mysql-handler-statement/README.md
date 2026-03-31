# How to Use HANDLER Statement in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Handler, Index, Performance, Query

Description: Learn how to use the MySQL HANDLER statement to open a table cursor and read rows directly, bypassing the SQL query optimizer for low-level access.

---

## Overview

The `HANDLER` statement provides a low-level interface to read rows from a table directly, bypassing the SQL query optimizer. It is useful for high-performance sequential scans or direct index traversals where the overhead of full SQL parsing and optimization is not needed. `HANDLER` works with InnoDB and MyISAM tables.

## HANDLER Statements Overview

The HANDLER interface has three main statements:
- `HANDLER table_name OPEN` - opens the table for cursor access
- `HANDLER table_name READ` - reads rows from the open table
- `HANDLER table_name CLOSE` - closes the open handler

## Opening a Handler

```sql
HANDLER users OPEN;

-- You can also alias it
HANDLER users OPEN AS u;
```

The table remains open until you explicitly close it or the session ends.

## Reading Rows

### Reading the First Row

```sql
HANDLER users READ FIRST;
```

### Reading the Next Row (Sequential Scan)

```sql
HANDLER users READ NEXT;
```

Repeat `READ NEXT` to advance through the table row by row.

### Reading by Index

You can read rows using a specific index:

```sql
-- Read the first row with id >= 100 using the primary key
HANDLER users READ `PRIMARY` >= (100);

-- Read the next row in index order
HANDLER users READ `PRIMARY` NEXT;
```

For named indexes:

```sql
HANDLER users READ idx_email = ('alice@example.com');
```

### Reading the Last Row

```sql
HANDLER users READ LAST;
```

### Reading with a WHERE Filter

You can add a `WHERE` clause to filter rows:

```sql
HANDLER users READ NEXT WHERE active = 1;
```

Note: the `WHERE` filter is applied after the row is fetched from the index, not as an index condition.

## Limiting Results

Add `LIMIT` to control how many rows are returned per read:

```sql
HANDLER users READ NEXT LIMIT 10;
```

## Closing the Handler

Always close the handler when done to release resources:

```sql
HANDLER users CLOSE;
```

## Full Example

```sql
-- Open the handler
HANDLER products OPEN;

-- Read the first row
HANDLER products READ FIRST;

-- Iterate through rows
HANDLER products READ NEXT;
HANDLER products READ NEXT;

-- Read using an index
HANDLER products READ idx_category = ('electronics') LIMIT 5;

-- Read the next matching rows
HANDLER products READ idx_category NEXT LIMIT 5;

-- Close when done
HANDLER products CLOSE;
```

## When to Use HANDLER

`HANDLER` is appropriate when:
- You need very fast sequential table scans without optimizer overhead
- You are building application-level cursor logic
- You want to traverse an index manually in a specific direction

It is not a replacement for regular SQL queries in most cases. Use it for specialized low-level data access patterns.

## Limitations

- `HANDLER` does not work with views or temporary tables
- Transactions and `HANDLER` cursors interact - DDL changes may invalidate an open handler
- It is a MySQL-specific extension with no SQL standard equivalent

## Summary

The `HANDLER` statement offers direct, cursor-style access to table rows and indexes, bypassing the query optimizer. Use `HANDLER OPEN` to start, `HANDLER READ` to advance through rows by index or sequentially, and `HANDLER CLOSE` to release the cursor. It is a specialized tool for performance-sensitive code paths that require fine-grained control over row access.
