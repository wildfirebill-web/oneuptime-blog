# How to Debug UDF Execution in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, UDF, Debug, Troubleshooting, Logging

Description: Learn how to debug UDF execution issues in ClickHouse, from inspecting SQL UDF definitions to troubleshooting executable script errors and logging output.

---

Debugging UDFs in ClickHouse requires different approaches depending on whether the UDF is SQL-based or backed by an executable script. This post covers the main techniques for diagnosing and fixing UDF problems.

## Debugging SQL UDFs

### View the UDF Definition

```sql
SELECT name, create_query
FROM system.functions
WHERE origin = 'SQLUserDefined' AND name = 'myFunction';
```

### Test with Simple Inputs

```sql
SELECT myFunction(100, 5) AS result;
```

### Trace Type Issues

```sql
SELECT toTypeName(myFunction(100, 5)) AS return_type;
```

### Check for NULL Propagation

```sql
SELECT myFunction(NULL, 5) AS null_test;
```

If the result is unexpected, add explicit NULL handling:

```sql
CREATE OR REPLACE FUNCTION myFunction AS (a, b) ->
    if(isNull(a) OR isNull(b), 0, a / b);
```

## Debugging Executable UDFs

### Check ClickHouse Server Logs

```bash
sudo tail -f /var/log/clickhouse-server/clickhouse-server.err.log | grep -i "udf\|user_defined\|executable"
```

### Test the Script Directly

Run the script manually with sample input:

```bash
echo -e "hello world\nbad experience" | python3 /var/lib/clickhouse/user_scripts/sentiment.py
```

Expected output:

```text
neutral
negative
```

### Enable Verbose UDF Logging

Add to `config.xml`:

```text
<user_defined_executable_functions_config>/etc/clickhouse-server/user_defined/*.xml</user_defined_executable_functions_config>
```

Check the log for load errors:

```bash
grep "user_defined" /var/log/clickhouse-server/clickhouse-server.log
```

### Check Script Permissions

```bash
ls -la /var/lib/clickhouse/user_scripts/
# The clickhouse user must have read and execute permission
sudo chmod +x /var/lib/clickhouse/user_scripts/sentiment.py
sudo chown clickhouse:clickhouse /var/lib/clickhouse/user_scripts/sentiment.py
```

### Verify Python Dependencies

```bash
sudo -u clickhouse python3 -c "import sys; print(sys.version)"
sudo -u clickhouse python3 /var/lib/clickhouse/user_scripts/sentiment.py
```

## Using Query Tracing

```sql
SET send_logs_level = 'debug';
SELECT myUDF(text_column) FROM my_table LIMIT 5;
```

The debug log includes UDF invocation details.

## Handling Script Crashes

If the executable script exits with a non-zero code, ClickHouse logs an error and the query fails. Add a try/except in Python scripts:

```text
#!/usr/bin/env python3
import sys

for line in sys.stdin:
    try:
        result = process(line.strip())
        print(result)
        sys.stdout.flush()
    except Exception as e:
        print('error', file=sys.stderr)
        print('unknown')
        sys.stdout.flush()
```

## Listing All Registered UDFs

```sql
SELECT name, origin
FROM system.functions
WHERE origin IN ('SQLUserDefined', 'ExecutableUserDefined')
ORDER BY name;
```

## Summary

Debugging ClickHouse UDFs involves checking `system.functions` for definitions, testing SQL UDFs with isolated inputs, and running executable scripts manually to verify stdin/stdout behavior. For persistent issues with executable UDFs, check the server error log, verify script permissions, and ensure the `clickhouse` user can execute the script and all its dependencies.
