# How to Use system.quotas and system.quota_usage in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, system.quotas, system.quota_usage, Access Control, Resource Limit, Setting

Description: Use system.quotas to inspect defined quota policies and system.quota_usage to monitor how much of each quota a user or role has consumed in ClickHouse.

---

ClickHouse quotas limit how many queries, rows, or bytes a user or role can consume within a time interval. The `system.quotas` table describes the configured policies, while `system.quota_usage` shows real-time consumption against those limits.

## system.quotas

`system.quotas` lists all quota policies defined on the server.

```sql
SELECT
    name,
    id,
    keys,
    durations,
    max_queries,
    max_errors,
    max_result_rows,
    max_read_rows,
    max_execution_time
FROM system.quotas
ORDER BY name;
```

Key fields:
- `name` - quota policy name
- `keys` - what the quota is keyed on (`user_name`, `ip_address`, `client_key`, or `initial_query`)
- `durations` - time intervals (in seconds) for each limit tier
- `max_queries` - queries allowed per interval
- `max_read_rows` - rows allowed to be read per interval
- `max_execution_time` - total execution time allowed per interval

## Creating a Quota

```sql
CREATE QUOTA analyst_quota
    FOR INTERVAL 1 HOUR MAX queries = 1000,
                         MAX read_rows = 100000000,
                         MAX execution_time = 3600
    TO analyst;
```

## system.quota_usage

`system.quota_usage` shows how much of each quota has been consumed in the current interval for the user running the query.

```sql
SELECT
    quota_name,
    quota_key,
    duration,
    queries,
    max_queries,
    errors,
    read_rows,
    max_read_rows,
    execution_time,
    max_execution_time,
    quota_start_time,
    quota_end_time
FROM system.quota_usage;
```

## Monitoring All Users' Quota Usage

Admins can query `system.quota_limits` and `system.quotas_usage` for all users:

```sql
SELECT
    quota_name,
    quota_key,
    queries,
    max_queries,
    read_rows,
    max_read_rows,
    round(queries / max_queries * 100, 1)    AS query_pct_used,
    round(read_rows / max_read_rows * 100, 1) AS rows_pct_used
FROM system.quota_limits
WHERE max_queries > 0
ORDER BY query_pct_used DESC;
```

## Finding Users Near Their Limit

```sql
SELECT
    quota_name,
    quota_key,
    queries,
    max_queries,
    queries / max_queries AS fraction_used
FROM system.quota_limits
WHERE max_queries > 0
  AND queries / max_queries > 0.8
ORDER BY fraction_used DESC;
```

## Resetting Quota Counters

Quotas reset automatically at the end of each interval. To reset immediately for a specific user, you can recreate or modify the quota, but there's no direct reset command.

## Associating Quotas with Users

```sql
-- Apply quota to a user
ALTER USER analyst DEFAULT QUOTA analyst_quota;

-- Apply quota to a role
ALTER ROLE data_analyst SETTINGS PROFILE 'restricted';
```

## Viewing Quota-to-User Mapping

```sql
SELECT user_name, quota_name
FROM system.user_directories
INNER JOIN system.quotas ON true  -- join through grants
LIMIT 10;
```

Check `SHOW CREATE USER analyst` for the active quota assignment.

## Summary

`system.quotas` and `system.quota_usage` enable full quota lifecycle management in ClickHouse. Define quotas with time-window limits on queries and read volume, monitor real-time consumption through `system.quota_usage`, and alert when users approach their limits to proactively manage resource contention.
