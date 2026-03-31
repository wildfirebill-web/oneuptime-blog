# How to Write Your First ClickHouse UDF

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, UDF, User-Defined Function, SQL, Executable

Description: Learn how to write your first ClickHouse user-defined function using both SQL lambda syntax and executable UDFs backed by an external script.

---

ClickHouse supports two types of user-defined functions: SQL UDFs defined with lambda expressions, and executable UDFs that delegate to an external script. SQL UDFs are the simplest and fastest option for most use cases.

## SQL UDFs

SQL UDFs are defined with `CREATE FUNCTION` and use lambda syntax:

```sql
CREATE FUNCTION celsius_to_fahrenheit AS (c) -> (c * 9.0 / 5.0) + 32;
```

Use it just like a built-in function:

```sql
SELECT
    sensor_id,
    temperature_c,
    celsius_to_fahrenheit(temperature_c) AS temperature_f
FROM sensor_readings
LIMIT 10;
```

You can also define functions with multiple arguments:

```sql
CREATE FUNCTION clamp AS (val, lo, hi) ->
    if(val < lo, lo, if(val > hi, hi, val));
```

```sql
SELECT clamp(sensor_value, 0, 100) AS normalized
FROM readings;
```

## Listing and Dropping UDFs

```sql
-- List all user-defined functions
SELECT name, create_query FROM system.functions WHERE origin = 'SQLUserDefined';

-- Drop a function
DROP FUNCTION celsius_to_fahrenheit;
```

## Executable UDFs

For complex logic that cannot be expressed in SQL, ClickHouse can call an external script over stdin/stdout. Create a Python script at `/var/lib/clickhouse/user_scripts/parse_ua.py`:

```python
#!/usr/bin/env python3
import sys

for line in sys.stdin:
    ua = line.strip()
    if 'Mobile' in ua:
        print('mobile')
    elif 'Tablet' in ua:
        print('tablet')
    else:
        print('desktop')
```

```bash
chmod +x /var/lib/clickhouse/user_scripts/parse_ua.py
```

Register it in `/etc/clickhouse-server/user_defined/parse_ua.xml`:

```xml
<functions>
  <function>
    <type>executable</type>
    <name>parse_ua</name>
    <return_type>String</return_type>
    <argument><type>String</type></argument>
    <format>TabSeparated</format>
    <command>parse_ua.py</command>
  </function>
</functions>
```

Then use it:

```sql
SELECT
    user_agent,
    parse_ua(user_agent) AS device_type
FROM web_logs
LIMIT 5;
```

## Performance Considerations

- SQL UDFs are inlined at query planning time - they have near-zero overhead.
- Executable UDFs have process-spawn overhead and inter-process I/O cost; avoid them for high-frequency calls on large datasets.
- For complex transformations, consider pre-computing values in materialized views rather than applying UDFs at query time.

## Summary

ClickHouse SQL UDFs provide a clean way to encapsulate reusable expressions using lambda syntax and `CREATE FUNCTION`. For logic that requires external processing, executable UDFs call a script over stdin/stdout. SQL UDFs are always preferred for performance, with executable UDFs reserved for logic that cannot be expressed in SQL.
