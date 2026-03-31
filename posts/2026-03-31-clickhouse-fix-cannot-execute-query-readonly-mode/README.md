# How to Fix "Cannot execute query in readonly mode" in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Readonly, Permission, Error, Troubleshooting

Description: Fix "Cannot execute query in readonly mode" errors in ClickHouse by adjusting user readonly settings or fixing the underlying readonly cause.

---

ClickHouse has a `readonly` setting that restricts what a user can do. When set to 1 or 2, write operations and settings changes are blocked. This is different from the table-level readonly mode - it applies to the user's session or profile.

## Understand the readonly Levels

```text
readonly = 0  - No restrictions (default for admin)
readonly = 1  - Only SELECT allowed, no settings changes
readonly = 2  - SELECT + SET allowed, but no writes
```

Check your current setting:

```sql
SELECT getSetting('readonly');
```

## Fix for Connection String

If you are connecting with a readonly flag explicitly:

```bash
# Remove the readonly flag
clickhouse-client --host localhost --user my_user --password secret
# NOT: clickhouse-client --readonly 1
```

## Fix in User Profile

Check if the user's profile has readonly set:

```sql
SELECT name, readonly
FROM system.users
WHERE name = 'my_user';
```

Update the profile to allow writes:

```sql
ALTER USER my_user SETTINGS readonly = 0;
```

In `users.xml`:

```xml
<users>
  <my_user>
    <profile>write_profile</profile>
  </my_user>
</users>

<profiles>
  <write_profile>
    <readonly>0</readonly>
  </write_profile>
  <read_only_profile>
    <readonly>1</readonly>
  </read_only_profile>
</profiles>
```

## Fix Temporary Session Readonly

If a query set readonly for the session:

```sql
-- This locks the session to readonly:
SET readonly = 1;

-- You cannot unset it in readonly mode - reconnect instead
```

Reconnect to get a fresh session.

## Fix for HTTP Interface

When using the HTTP API, the `readonly` query parameter overrides profile settings:

```bash
# This sets readonly=1 for this request - remove the param:
curl "http://localhost:8123/?query=INSERT+INTO+t+VALUES+(1)&readonly=0"
```

## Grant Write Permissions

Ensure the user has the correct grants for the target table:

```sql
GRANT INSERT ON my_database.my_table TO my_user;
GRANT CREATE TABLE ON my_database.* TO my_user;
```

## Summary

"Cannot execute query in readonly mode" in ClickHouse is caused by the `readonly` user setting being 1 or 2. Fix it by updating the user's profile in `users.xml` or with `ALTER USER ... SETTINGS readonly = 0`, removing `--readonly` flags from connection strings, and ensuring proper GRANT permissions. Use `readonly = 1` intentionally for dashboard users who should never write data.
