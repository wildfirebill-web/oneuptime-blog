# How to Use MySQL Shell in SQL Mode

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, MySQL Shell, SQL, Query, Tool

Description: Learn how to use MySQL Shell in SQL mode to run queries, switch databases, execute scripts, and use built-in features like auto-complete and result formatting.

---

MySQL Shell's SQL mode provides an enhanced SQL prompt that supports multi-line queries, auto-complete, result formatting, and seamless switching to JavaScript or Python modes within the same session.

## Start MySQL Shell in SQL Mode

```bash
# Start in SQL mode directly
mysqlsh root@localhost --sql

# Or start without mode and switch to SQL
mysqlsh root@localhost
# Then at the prompt:
\sql
```

The prompt changes to indicate the active mode:

```text
MySQL  localhost:3306 ssl  SQL >
```

## Running Basic SQL Queries

SQL mode behaves like the classic `mysql` client:

```sql
-- Show databases
SHOW DATABASES;

-- Select a database
USE mydb;

-- Run queries
SELECT * FROM users LIMIT 10;

-- Multi-line queries are supported
SELECT u.id, u.email,
       COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.email
ORDER BY order_count DESC
LIMIT 5;
```

## Switch Between Databases

```sql
USE mydb;
SELECT DATABASE();
```

## Execute a SQL Script File

```bash
# From the command line
mysqlsh root@localhost --sql < migration.sql

# From within the shell
\source /path/to/script.sql
# Or
source /path/to/script.sql
```

## Output Formatting

MySQL Shell provides multiple output formats:

```bash
# Table format (default)
mysqlsh root@localhost --sql --result-format=table

# JSON output
mysqlsh root@localhost --sql --result-format=json -e "SELECT * FROM users LIMIT 3;"

# Vertical format (like \G in classic client)
mysqlsh root@localhost --sql --result-format=vertical -e "SHOW ENGINE INNODB STATUS;"
```

Or within the shell session:

```text
\option resultFormat=json
SELECT * FROM users LIMIT 2;
```

## Use Auto-Complete

MySQL Shell has built-in auto-complete for SQL keywords, table names, and column names. Press `Tab` to trigger completion:

```text
MySQL  localhost:3306 ssl  mydb  SQL > SELECT * FROM ord[TAB]
```

This completes to `orders` if that table exists in the current schema.

## Run a Single Statement and Exit

```bash
mysqlsh root@localhost/mydb --sql -e "SELECT COUNT(*) FROM users;"
```

## Switch to Other Modes

From SQL mode, switch to JavaScript or Python without reconnecting:

```text
-- In SQL mode
\js
// Now in JavaScript mode
session.runSql("SELECT 1").fetchAll()

\py
# Now in Python mode
session.run_sql("SELECT 1").fetch_all()

\sql
-- Back to SQL mode
```

## Import and Export Data

```sql
-- Export to a file
SELECT * FROM users INTO OUTFILE '/tmp/users.csv'
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n';

-- Load from a file
LOAD DATA INFILE '/tmp/users.csv'
INTO TABLE users_backup
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n';
```

For bulk import using the Shell's `util` module, switch to JavaScript mode:

```text
\js
util.importTable('/tmp/users.csv', {table: 'users', dialect: 'csv'})
```

## Summary

MySQL Shell's SQL mode provides a feature-rich interactive SQL client with auto-complete, multiple output formats, and direct access to file-based script execution. The `\sql`, `\js`, and `\py` commands let you switch modes within the same session, making it easy to combine SQL operations with scripted logic using the DevAPI or AdminAPI.
