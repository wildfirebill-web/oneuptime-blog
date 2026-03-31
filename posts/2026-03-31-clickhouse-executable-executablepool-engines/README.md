# How to Use Executable and ExecutablePool Table Engines in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Executable Engine, ExecutablePool Engine, Custom Script, Data Integration

Description: Learn how to use the Executable and ExecutablePool table engines in ClickHouse to run external scripts as data sources inside SQL queries.

---

ClickHouse's Executable and ExecutablePool table engines let you use any external script or binary as a data source. ClickHouse pipes data to the script's stdin (for INSERT) or reads from its stdout (for SELECT). This opens the door to arbitrary data generation, API polling, or custom transformations written in Python, Bash, or any other language.

## Executable Engine

The Executable engine runs a script fresh on every query. It is simple but carries startup overhead per execution.

```sql
CREATE TABLE python_source (
    ts DateTime,
    metric String,
    value Float64
)
ENGINE = Executable(
    'my_script.py',
    TabSeparated,
    []
);
```

The script path is relative to `user_scripts_path` defined in your ClickHouse config (default: `/var/lib/clickhouse/user_scripts/`).

## Writing the Script

Your script outputs rows in the specified format on stdout:

```python
#!/usr/bin/env python3
import sys
from datetime import datetime

rows = [
    (datetime.utcnow().strftime('%Y-%m-%d %H:%M:%S'), 'cpu', 72.4),
    (datetime.utcnow().strftime('%Y-%m-%d %H:%M:%S'), 'mem', 45.1),
]
for ts, metric, value in rows:
    print(f"{ts}\t{metric}\t{value}")
```

Make the script executable:

```bash
chmod +x /var/lib/clickhouse/user_scripts/my_script.py
```

## ExecutablePool Engine

ExecutablePool keeps a pool of persistent script processes to avoid repeated startup cost. It is better for high-frequency queries:

```sql
CREATE TABLE pooled_source (
    id UInt64,
    data String
)
ENGINE = ExecutablePool(
    'pooled_script.py',
    TabSeparated,
    [],
    pool_size = 4,
    max_command_execution_time = 10
);
```

`pool_size` controls how many script processes are kept running. Each query picks a worker from the pool, sends arguments via stdin, and reads results from stdout.

## Passing Arguments from Query

You can pass arguments to scripts using the third parameter:

```sql
CREATE TABLE filtered_source (
    id UInt64,
    name String
)
ENGINE = Executable(
    'fetch.py',
    TabSeparated,
    ['--limit', '1000']
);
```

Inside your script, read arguments with `sys.argv`:

```python
#!/usr/bin/env python3
import sys

limit = int(sys.argv[sys.argv.index('--limit') + 1])
for i in range(limit):
    print(f"{i}\titem_{i}")
```

## INSERT Support

The Executable engine also accepts INSERT to pipe data into the script:

```sql
INSERT INTO python_source VALUES
(now(), 'disk', 88.2),
(now(), 'net', 12.5);
```

The script receives rows on stdin and can forward them to a downstream system, database, or file.

## Security Considerations

Restrict which scripts can be used by configuring `user_scripts_path` carefully. Grant minimal OS-level permissions to the scripts. Avoid running untrusted code as the ClickHouse process user.

```bash
# Recommended: dedicated user_scripts directory with tight permissions
chown clickhouse:clickhouse /var/lib/clickhouse/user_scripts/
chmod 700 /var/lib/clickhouse/user_scripts/
```

## Summary

The Executable and ExecutablePool engines provide a flexible escape hatch for integrating custom logic or external data sources directly into ClickHouse SQL. Use Executable for simple or infrequent queries and ExecutablePool for performance-sensitive workloads where script startup cost matters. Always audit scripts for security and resource usage in production environments.
