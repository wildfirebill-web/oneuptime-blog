# How to Fix ERROR 1175 Safe Update Mode Prevents UPDATE Without WHERE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Safe Update, Error, SQL Mode, Update

Description: Fix MySQL ERROR 1175 by disabling safe update mode for the session or by rewriting queries to include a WHERE clause on a key column.

---

MySQL ERROR 1175 occurs when MySQL Workbench or a client with safe-updates enabled prevents you from running an UPDATE or DELETE statement that lacks a WHERE clause on a key column. The error reads: `Error Code: 1175. You are using safe update mode and you tried to update a table without a WHERE that uses a KEY column`.

## Why This Error Exists

The `sql_safe_updates` setting protects against accidental bulk updates or deletes. When enabled, MySQL requires that UPDATE and DELETE statements include a WHERE clause that references a key column (primary key or indexed column), or a LIMIT clause.

## Check the Current Setting

```sql
SHOW VARIABLES LIKE 'sql_safe_updates';
```

## Fix 1: Disable Safe Updates for the Session

The safest approach is to disable the setting only for the current session, not globally:

```sql
SET SESSION sql_safe_updates = 0;

-- Now run your update
UPDATE users SET last_login = NOW();

-- Re-enable when done (optional - it resets at session end)
SET SESSION sql_safe_updates = 1;
```

## Fix 2: Add a WHERE Clause on a Key Column

The better practice is to write queries that safe update mode accepts:

```sql
-- This fails under safe updates (no WHERE on a key)
UPDATE products SET price = price * 1.10;

-- This works - WHERE uses the primary key range
UPDATE products SET price = price * 1.10
WHERE id > 0;

-- Even better - be explicit about what you are updating
UPDATE products SET price = price * 1.10
WHERE category_id IN (1, 2, 3);
```

## Fix 3: Add a LIMIT Clause

A LIMIT clause also satisfies safe update mode requirements:

```sql
UPDATE audit_log SET reviewed = 1
WHERE reviewed = 0
LIMIT 10000;
```

## Fix 4: In MySQL Workbench

MySQL Workbench enables safe updates by default. To disable it in the IDE:

1. Go to Edit > Preferences
2. Select SQL Editor
3. Uncheck "Safe Updates (rejects UPDATEs and DELETEs with no restrictions)"
4. Reconnect to the server for the change to take effect

## Fix 5: Disable Globally in my.cnf

For a development environment where this is always getting in the way, disable it globally. Do not do this in production:

```text
[mysqld]
sql_safe_updates = OFF
```

Or at startup:

```bash
mysqld --sql-safe-updates=OFF
```

## Using a LIMIT with DELETE

For DELETE statements that fail the same check:

```sql
-- Add ORDER BY + LIMIT to satisfy safe update mode
DELETE FROM temp_logs
ORDER BY created_at ASC
LIMIT 5000;
```

## Summary

ERROR 1175 is a safety guard, not a real error. The cleanest fix is to add a WHERE clause on a key column to your UPDATE or DELETE statement, which is best practice anyway. If you need to run a full-table update, disable `sql_safe_updates` for the session only, perform the update, then let it reset when the session ends.
