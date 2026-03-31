# How to Configure ClickHouse for Maximum Concurrent Users

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Concurrency, User Management, Resource Limit, Configuration, Performance

Description: Configure ClickHouse to serve many concurrent users by tuning connection limits, memory quotas, query priority, and read-only profiles.

---

## Understanding ClickHouse Concurrency Model

ClickHouse uses a thread-pool model where each query gets multiple threads. The default is to use all available CPU cores for a single query. When many users query simultaneously, this causes CPU contention. Proper configuration prevents one heavy query from starving all others.

## Setting Global Concurrent Query Limits

In `config.xml` or via query:

```xml
<max_concurrent_queries>200</max_concurrent_queries>
<max_concurrent_insert_queries>50</max_concurrent_insert_queries>
<max_concurrent_select_queries>150</max_concurrent_select_queries>
```

These are server-wide limits. Requests above the limit receive a "Too many simultaneous queries" error.

## Limiting Threads per Query

Reduce the threads allocated to each query so more queries can run in parallel:

```sql
CREATE SETTINGS PROFILE low_concurrency_profile
SETTINGS
    max_threads = 4,
    max_memory_usage = 2000000000,
    max_execution_time = 30;

ALTER USER dashboard_user SETTINGS PROFILE 'low_concurrency_profile';
```

Fewer threads per query means more queries run simultaneously without CPU saturation.

## Configuring User Quotas

Quotas limit resource use per time interval:

```sql
CREATE QUOTA dashboard_quota
FOR INTERVAL 1 MINUTE MAX queries = 60, MAX read_rows = 100000000
TO dashboard_user;
```

This prevents a single user from issuing too many queries or scanning too much data.

## Read-Only Profiles for Dashboard Users

Dashboard users rarely need to write. Restrict them:

```sql
CREATE SETTINGS PROFILE readonly_profile
SETTINGS readonly = 1;

ALTER USER dashboard_user SETTINGS PROFILE 'readonly_profile';
```

Read-only mode prevents accidental mutations that could lock resources.

## Connection Pool Tuning

For applications using HTTP or native protocol, configure connection pools:

```xml
<max_connections>4096</max_connections>
<keep_alive_timeout>10</keep_alive_timeout>
```

For a ClickHouse cluster behind a load balancer, ensure sticky sessions or stateless authentication is configured so connections distribute evenly.

## Monitoring Active Queries

Track live query activity:

```sql
SELECT
    user,
    query_id,
    elapsed,
    memory_usage,
    read_rows,
    query
FROM system.processes
ORDER BY elapsed DESC;
```

Kill runaway queries:

```sql
KILL QUERY WHERE elapsed > 60 AND user = 'dashboard_user';
```

## Summary

Configure ClickHouse for maximum concurrent users by capping threads per query, applying memory and time quotas, restricting dashboard users to read-only profiles, and setting global concurrent query limits. Monitor `system.processes` to detect and kill long-running queries before they affect other users.
