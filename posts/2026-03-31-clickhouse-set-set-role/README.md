# How to Use SET and SET ROLE in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Set, SET ROLE, Settings, Security

Description: Learn how to configure session-scoped settings with SET and activate roles with SET ROLE in ClickHouse, including how to reset settings and manage role context.

---

ClickHouse provides two related but distinct statements for runtime configuration: SET, which adjusts query and session settings, and SET ROLE, which activates or deactivates roles for the current session. Both changes are scoped to the current session and do not persist after the connection is closed. This post covers both statements with practical examples.

## SET - Configuring Session Settings

The SET statement changes the value of a ClickHouse setting for the duration of the current session. All subsequent queries in the session use the new value.

```sql
SET setting_name = value;
```

### Common Examples

```sql
-- Allow more memory for heavy analytical queries
SET max_memory_usage = 10000000000;

-- Increase the number of threads for a large query
SET max_threads = 16;

-- Allow approximate GROUP BY when memory is low
SET group_by_overflow_mode = 'any';

-- Set the format for query output
SET output_format_pretty_color = 0;

-- Enable or disable PREWHERE auto-optimization
SET optimize_move_to_prewhere = 1;
```

### Verifying Current Setting Values

```sql
-- Check the current value of a setting
SELECT value
FROM system.settings
WHERE name = 'max_memory_usage';

-- Show all non-default settings for the session
SELECT name, value
FROM system.settings
WHERE changed;
```

### Applying Settings Per Query

Instead of a session-level SET, you can apply settings to a single query using the SETTINGS clause:

```sql
SELECT count()
FROM analytics.events
WHERE event_type = 'buy'
SETTINGS max_threads = 8, max_memory_usage = 5000000000;
```

Per-query settings override session settings for that query only.

## Resetting a Setting to Its Default

To restore a setting to its default value, use the DEFAULT keyword:

```sql
SET max_memory_usage = DEFAULT;
```

Or explicitly set the server default value:

```sql
SET max_threads = 0;  -- 0 means "use all available CPU threads"
```

## SET ROLE - Activating Roles

SET ROLE activates one or more roles for the current session. Only roles that have been granted to the user (via GRANT role TO user) can be activated.

```sql
SET ROLE role_name;
```

### Activate a Single Role

```sql
SET ROLE analyst;
```

After this statement, the user gains all privileges associated with the `analyst` role for the remainder of the session.

### Activate Multiple Roles

```sql
SET ROLE analyst, data_engineer;
```

### Activate All Assigned Roles

```sql
SET ROLE ALL;
```

This activates every role that has been granted to the current user.

### Deactivate All Roles

```sql
SET ROLE NONE;
```

This resets the session to no active roles, even if the user has a DEFAULT ROLE configured. Useful for testing the least-privilege state.

### Activate All Roles Except One

```sql
SET ROLE ALL EXCEPT sensitive_data_reader;
```

## DEFAULT ROLE vs SET ROLE

When a user connects, ClickHouse activates their DEFAULT ROLE automatically. SET ROLE overrides this for the current session without changing the user's DEFAULT ROLE definition.

```sql
-- Change a user's default role permanently (requires admin privilege)
ALTER USER alice DEFAULT ROLE analyst;

-- Temporarily override the active role for this session only
SET ROLE data_engineer;
```

## Practical Example - Switching Roles in a Pipeline

```sql
-- Start with minimal privileges, activate elevated role only when needed
SET ROLE NONE;

-- Validate read access works
SELECT count() FROM analytics.events;

-- Activate write role for the insert step
SET ROLE data_engineer;

INSERT INTO staging.events_raw
SELECT * FROM input_data;

-- Drop back to read-only after the write step
SET ROLE analyst;
```

### Checking the Active Role

```sql
-- Show currently active roles for the session
SELECT currentRoles();

-- Show all roles granted to the current user
SELECT *
FROM system.role_grants
WHERE user_name = currentUser();
```

## Summary

SET adjusts session-scoped ClickHouse settings such as memory limits and thread counts, with changes lasting only for the current connection. SET ROLE activates one or more granted roles for the session, and SET ROLE NONE reverts to no active roles. Together these two statements let you tune query behavior and manage privilege context without making permanent configuration changes.
