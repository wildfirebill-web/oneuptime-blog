# How to Create a Quota in ClickHouse for Resource Limits

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Security, Quota, Resource Management

Description: Learn how to use CREATE QUOTA in ClickHouse to enforce resource limits per user or IP, controlling queries, read rows, and result rows.

---

Resource quotas in ClickHouse allow administrators to cap the number of queries, rows read, and data transferred within a given time window. This is essential for multi-tenant environments where one user's heavy workload should not starve others.

## CREATE QUOTA Syntax

The basic syntax for creating a quota is:

```sql
CREATE QUOTA [IF NOT EXISTS | OR REPLACE] name
    [KEYED BY {user_name | ip_address | client_key | client_key,user_name | client_key,ip_address}]
    [FOR [RANDOMIZED] INTERVAL number {second | minute | hour | day | week | month | quarter | year}
        {MAX {queries = N, query_selects = N, query_inserts = N,
              errors = N, result_rows = N, result_bytes = N,
              read_rows = N, read_bytes = N, execution_time = N} |
         NO LIMITS | TRACKING ONLY} [,...]]
    [TO {role [,...] | ALL | ALL EXCEPT role [,...]}]
```

## KEYED BY Options

The `KEYED BY` clause determines how quota usage is tracked and shared:

```sql
-- Each user gets their own independent quota counter
CREATE QUOTA per_user_quota KEYED BY user_name
    FOR INTERVAL 1 HOUR MAX queries = 1000
    TO ALL;

-- Each IP address gets its own independent quota counter
CREATE QUOTA per_ip_quota KEYED BY ip_address
    FOR INTERVAL 1 HOUR MAX queries = 500;

-- All users share one quota pool (default when KEYED BY is omitted)
CREATE QUOTA shared_quota
    FOR INTERVAL 1 HOUR MAX queries = 10000
    TO ALL;
```

## Setting Time Interval Constraints

Multiple time intervals can be stacked in a single quota definition:

```sql
CREATE QUOTA analyst_quota KEYED BY user_name
    FOR INTERVAL 1 MINUTE MAX queries = 20,
    FOR INTERVAL 1 HOUR   MAX queries = 500, read_rows = 100000000,
    FOR INTERVAL 1 DAY    MAX queries = 5000, read_bytes = 10000000000
    TO analyst_role;
```

The constraints available in `MAX` include:

| Constraint | Description |
|---|---|
| `queries` | Total number of queries |
| `query_selects` | SELECT queries only |
| `query_inserts` | INSERT queries only |
| `errors` | Queries that returned an error |
| `result_rows` | Rows returned to the client |
| `result_bytes` | Bytes returned to the client |
| `read_rows` | Rows read from storage |
| `read_bytes` | Bytes read from storage |
| `execution_time` | Total execution time in seconds |

## ASSIGN TO Users and Roles

Use the `TO` clause to assign a quota to specific users or roles:

```sql
-- Assign to a specific user
CREATE QUOTA heavy_user_quota KEYED BY user_name
    FOR INTERVAL 1 HOUR MAX read_bytes = 5000000000
    TO john;

-- Assign to a role
CREATE QUOTA api_quota KEYED BY client_key
    FOR INTERVAL 1 MINUTE MAX queries = 60
    TO api_role;

-- Assign to all users except admins
CREATE QUOTA default_quota KEYED BY user_name
    FOR INTERVAL 1 DAY MAX queries = 10000, read_rows = 1000000000
    TO ALL EXCEPT admin_role;
```

## TRACKING ONLY Mode

To observe usage without enforcing limits, use `TRACKING ONLY`:

```sql
CREATE QUOTA tracking_quota KEYED BY user_name
    FOR INTERVAL 1 HOUR TRACKING ONLY
    TO ALL;
```

Query the `system.quota_usage` table to review accumulated usage:

```sql
SELECT
    quota_name,
    quota_key,
    duration,
    queries,
    read_rows,
    read_bytes,
    result_rows
FROM system.quota_usage
ORDER BY queries DESC;
```

## Complete Practical Example

The following example sets up tiered quotas for three different user classes:

```sql
-- Free tier: low limits
CREATE QUOTA free_tier_quota KEYED BY user_name
    FOR INTERVAL 1 HOUR  MAX queries = 100, read_rows = 10000000,
    FOR INTERVAL 24 HOUR MAX queries = 500, read_bytes = 1000000000
    TO free_role;

-- Pro tier: higher limits
CREATE QUOTA pro_tier_quota KEYED BY user_name
    FOR INTERVAL 1 HOUR  MAX queries = 1000, read_rows = 500000000,
    FOR INTERVAL 24 HOUR MAX queries = 10000, read_bytes = 50000000000
    TO pro_role;

-- Admin: no limits enforced, but track usage
CREATE QUOTA admin_tracking KEYED BY user_name
    FOR INTERVAL 1 DAY TRACKING ONLY
    TO admin_role;
```

## Altering and Dropping Quotas

```sql
-- Modify an existing quota
ALTER QUOTA free_tier_quota
    FOR INTERVAL 1 HOUR MAX queries = 200, read_rows = 20000000
    TO free_role;

-- Remove a quota
DROP QUOTA IF EXISTS free_tier_quota;

-- View all defined quotas
SHOW QUOTAS;

-- View quota details
SHOW CREATE QUOTA free_tier_quota;
```

## Summary

ClickHouse quotas provide time-windowed resource controls per user, IP, or client key. The `KEYED BY` clause determines isolation, `FOR INTERVAL` windows stack to cover short and long-term limits, and the `TO` clause assigns quotas to users or roles. Use `TRACKING ONLY` to gather usage data before enforcing hard limits.
