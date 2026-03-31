# How to Use perror to Look Up MySQL Error Codes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Error, Command Line, Troubleshooting

Description: Learn how to use the perror utility to translate MySQL and operating system error codes into human-readable descriptions for faster troubleshooting.

---

## What Is perror?

`perror` is a MySQL command-line utility that decodes numeric error codes into descriptive error messages. When MySQL logs show error numbers like `1045` or OS errors like `ERRNO 28`, `perror` lets you quickly find out what they mean without searching the documentation.

Note: In MySQL 8.0.16+, `perror` was replaced by `mysql --verbose --help` and the `perror` functionality was merged into the `mysqld` binary. On those versions, use `mysql --verbose --help | grep 'ERROR '` or consult the MySQL error message reference directly.

## Basic Syntax

```bash
perror [options] error_code [error_code ...]
```

## Looking Up MySQL Error Codes

```bash
# Look up error 1045 (Access denied)
perror 1045
```

```text
MySQL error code MY-001045 (ER_ACCESS_DENIED_ERROR):
Access denied for user '%s'@'%s' (using password: %s)
```

```bash
# Look up multiple errors at once
perror 1045 1064 1213
```

```text
MySQL error code MY-001045 (ER_ACCESS_DENIED_ERROR):
Access denied for user '%s'@'%s' (using password: %s)

MySQL error code MY-001064 (ER_PARSE_ERROR):
You have an error in your SQL syntax...

MySQL error code MY-001213 (ER_LOCK_DEADLOCK):
Deadlock found when trying to get lock; try restarting transaction
```

## Looking Up OS Error Codes

```bash
# Look up OS error 28 (No space left on device)
perror --ndb 28
```

```text
OS error code  28: No space left on device
```

Common OS errors encountered in MySQL:

```bash
perror 13   # Permission denied
perror 24   # Too many open files
perror 28   # No space left on device
perror 32   # Broken pipe
perror 111  # Connection refused
```

## MySQL 8.0+ Alternative

On MySQL 8.0.16+, use the mysqld binary:

```bash
# Look up a MySQL error code
mysqld --verbose --help 2>/dev/null | grep 'ER_ACCESS'

# Or use the MySQL documentation directly
# Error code reference: https://dev.mysql.com/doc/mysql-errors/8.0/en/
```

## Looking Up Errors from the mysql Client

Within a MySQL session, you can retrieve the last error:

```sql
-- Show the last error number and message
SHOW ERRORS;

-- Show warnings alongside errors
SHOW WARNINGS;

-- Get the error message for a specific code via a stored procedure
SELECT * FROM performance_schema.events_errors_summary_global_by_error
WHERE ERROR_NUMBER = 1213;
```

## Common MySQL Error Codes Reference

```text
1005  - Can't create table (check foreign key constraints)
1045  - Access denied (wrong user/password)
1046  - No database selected
1054  - Unknown column
1062  - Duplicate entry (unique key violation)
1064  - SQL syntax error
1146  - Table does not exist
1213  - Deadlock detected
1215  - Cannot add foreign key constraint
1292  - Incorrect datetime value
1366  - Incorrect integer value
2003  - Cannot connect to MySQL server
2006  - MySQL server has gone away
```

## Using perror in Scripts

```bash
#!/bin/bash
# Decode error codes from a MySQL error log
grep "Got error" /var/log/mysql/error.log | \
  grep -oP '\d+' | \
  sort -u | \
  while read code; do
    echo "Error $code: $(perror $code 2>/dev/null | tail -1)"
  done
```

## Summary

`perror` is a simple but useful tool for translating numeric MySQL and OS error codes into human-readable messages. When you encounter an unfamiliar error number in logs or application output, run `perror <code>` to understand what MySQL is reporting. On MySQL 8.0.16 and later, the same information is accessible via the MySQL error message documentation or `mysqld --verbose --help`.
