# How to Fix 'Cannot execute query in readonly mode' in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Clickhouse, Permissions, Configuration, Troubleshooting

Description: Fix ClickHouse 'Cannot execute query in readonly mode' by identifying the readonly setting source and adjusting user profiles or connection settings.

---

## Understanding the Error

ClickHouse has a `readonly` setting that restricts what a session or user can execute. When you try to run a write operation in a read-only session, you see:

```text
Code: 164. DB::Exception: Cannot execute query in readonly mode.
```

The `readonly` setting has three levels:
- `0` - no restrictions (read and write allowed)
- `1` - only read queries allowed; settings cannot be changed
- `2` - read queries allowed; some settings can be changed but not the readonly setting itself

## Common Causes

- User profile configured with `readonly = 1` or `readonly = 2`
- Connection made with the `readonly` HTTP parameter set
- ClickHouse started with a corrupted or read-only data directory
- Replica is in a state where writes are temporarily blocked
- Connecting via a read replica endpoint

## Diagnosing the Problem

Check the current readonly setting in your session:

```sql
SELECT value
FROM system.settings
WHERE name = 'readonly';
```

Check the user's profile configuration:

```sql
SELECT
    name,
    storage,
    auth_type
FROM system.users
WHERE name = currentUser();
```

Check profile settings:

```sql
SELECT
    profile,
    name,
    value
FROM system.settings_profile_elements
WHERE name = 'readonly';
```

## Fix 1 - Update the User Profile in users.xml

If the user's profile has `readonly` set, update it:

```xml
<!-- /etc/clickhouse-server/users.d/users.xml -->
<clickhouse>
  <profiles>
    <my_profile>
      <!-- Change from 1 or 2 to 0 to allow writes -->
      <readonly>0</readonly>
    </my_profile>
  </profiles>
</clickhouse>
```

Reload the configuration:

```bash
sudo systemctl reload clickhouse-server
# Or send SIGHUP:
sudo kill -HUP $(pidof clickhouse-server)
```

## Fix 2 - Use SQL to Alter the Profile

In ClickHouse 22.x+ with SQL-driven access control:

```sql
-- Check current profile settings
SHOW CREATE SETTINGS PROFILE my_profile;

-- Alter the profile to remove readonly restriction
ALTER SETTINGS PROFILE my_profile
SETTINGS readonly = 0;
```

## Fix 3 - Fix HTTP Connection Readonly Parameter

If connecting via HTTP and passing `readonly=1`:

```bash
# Wrong: readonly mode
curl "http://localhost:8123/?query=INSERT+INTO+...&readonly=1"

# Correct: no readonly restriction
curl "http://localhost:8123/?query=INSERT+INTO+..."
```

In application code:

```python
import clickhouse_connect

# Don't pass readonly=True for write operations
client = clickhouse_connect.get_client(
    host='localhost',
    user='default',
    password='secret'
    # Do NOT add: settings={'readonly': 1}
)
```

## Fix 4 - Fix Read Replica Connections

If connecting to a replica that only allows reads:

```sql
-- Check if this server is a replica
SELECT is_leader FROM system.replicas LIMIT 1;

-- Check replication status
SELECT
    database,
    table,
    is_leader,
    is_readonly,
    total_replicas,
    active_replicas
FROM system.replicas;
```

Connect to the primary/leader shard for write operations. In your config:

```yaml
# Application connection config
write_host: clickhouse-primary.example.com
read_host: clickhouse-replica.example.com
```

## Fix 5 - Fix Data Directory Permissions

If the data directory is read-only at the OS level:

```bash
# Check permissions
ls -la /var/lib/clickhouse/

# Fix permissions
sudo chown -R clickhouse:clickhouse /var/lib/clickhouse/
sudo chmod -R 755 /var/lib/clickhouse/

# Restart ClickHouse
sudo systemctl restart clickhouse-server
```

## Fix 6 - Allow Setting Changes in readonly = 2 Mode

If `readonly = 2` is intentional and you just need to change a specific setting:

```sql
-- In readonly=2, you can change most settings except readonly itself
-- This works:
SET max_execution_time = 300;

-- But writes still fail in readonly mode
-- You must change the readonly setting at the profile level
```

## Verifying the Fix

```sql
-- After fixing, verify the setting
SELECT value FROM system.settings WHERE name = 'readonly';
-- Should return: 0

-- Test a write operation
CREATE TABLE test_write (id UInt32) ENGINE = Memory;
INSERT INTO test_write VALUES (1);
SELECT * FROM test_write;
DROP TABLE test_write;
```

## Summary

"Cannot execute query in readonly mode" in ClickHouse is controlled by the `readonly` session setting, which can be set at the user profile level, via HTTP parameters, or inherited from a replica configuration. Fix it by setting `readonly = 0` in the user profile XML or via SQL `ALTER SETTINGS PROFILE`, checking HTTP connection parameters, and ensuring you are connecting to a primary node for write operations.
