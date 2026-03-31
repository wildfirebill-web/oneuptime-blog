# What Is the MySQL Audit Plugin

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Audit Plugin, Security, Compliance

Description: The MySQL Audit Plugin records user activity and database events to a log file, supporting compliance requirements and forensic investigation of database access.

---

## Overview

The MySQL Audit Plugin logs database activity - connections, query executions, schema changes, and privilege operations - to an audit log file. It is essential for meeting compliance requirements (PCI-DSS, HIPAA, SOX, GDPR) that mandate tracking who accessed or modified data and when.

MySQL Enterprise Edition includes a commercial audit plugin. For community edition users, the open-source `audit_log` plugin (included in MySQL 8.0) or Percona's Audit Log Plugin are available alternatives.

## Installing the MySQL Audit Log Plugin

The audit plugin is included in MySQL 8.0:

```text
[mysqld]
plugin-load-add = audit_log.so
audit_log_format = JSON
audit_log_file = /var/log/mysql/audit.log
audit_log_policy = ALL
```

Or install at runtime:

```sql
INSTALL PLUGIN audit_log SONAME 'audit_log.so';
```

## Verifying the Plugin Is Loaded

```sql
SELECT plugin_name, plugin_status
FROM information_schema.plugins
WHERE plugin_name = 'audit_log';

SHOW VARIABLES LIKE 'audit_log%';
```

## Audit Log Formats

| Format | Description |
|--------|-------------|
| `OLD` | XML format (legacy) |
| `NEW` | Updated XML format |
| `JSON` | JSON format (recommended) |
| `CSV` | Comma-separated values |

## Audit Log Policies

```sql
-- Log everything
SET GLOBAL audit_log_policy = 'ALL';

-- Log only queries
SET GLOBAL audit_log_policy = 'QUERIES';

-- Log only logins
SET GLOBAL audit_log_policy = 'LOGINS';

-- Log nothing (temporarily disable)
SET GLOBAL audit_log_policy = 'NONE';
```

## Sample JSON Audit Log Entry

```json
{
  "timestamp": "2026-03-31T10:00:00.000000Z",
  "id": 1,
  "class": "general",
  "event": "log",
  "connection_id": 42,
  "account": {
    "user": "app_user",
    "host": "10.0.0.5"
  },
  "login": {
    "user": "app_user",
    "os": "10.0.0.5",
    "ip": "10.0.0.5",
    "proxy": ""
  },
  "general_data": {
    "command": "Query",
    "sql_command": "select",
    "query": "SELECT * FROM customers WHERE id = 123",
    "status": 0
  }
}
```

## Filtering with Audit Log Filter

MySQL 8.0 supports fine-grained filtering:

```sql
-- Create a filter that logs only specific users
SELECT audit_log_filter_set_filter('log_app_user',
  '{"filter": {"user": {"name": "app_user"}}}');

-- Assign filter to a user
SELECT audit_log_filter_set_user('app_user@%', 'log_app_user');

-- Create a filter for specific events
SELECT audit_log_filter_set_filter('log_ddl',
  '{"filter": {"class": {"name": "table_ddl_access"}}}');

-- Remove filter assignment
SELECT audit_log_filter_remove_user('app_user@%');
```

## Excluding Users from Audit

```sql
-- Create a filter that logs nothing (for service/monitoring users)
SELECT audit_log_filter_set_filter('exclude_all', '{"filter": {"log": false}}');
SELECT audit_log_filter_set_user('monitoring_user@localhost', 'exclude_all');
```

## Rotating the Audit Log

```sql
-- Rotate the audit log file
SELECT audit_log_rotate();
```

Or via configuration:

```text
[mysqld]
audit_log_rotate_on_size = 1073741824  -- 1 GB
```

## Audit Log with Percona (Community Edition)

Percona's audit plugin offers similar functionality for community MySQL:

```bash
apt-get install percona-server-server
```

```text
[mysqld]
plugin-load-add = audit_log.so
audit_log_handler = FILE
audit_log_file = /var/log/mysql/audit.log
```

## Summary

The MySQL Audit Plugin records database activity to a structured log file, providing the audit trail required for security compliance. It supports JSON output for easy ingestion into SIEM systems, fine-grained per-user filtering to reduce log volume, and log rotation for file management. Always configure the audit log as part of your security baseline for production MySQL instances.
