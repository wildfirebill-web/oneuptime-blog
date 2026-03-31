# How to Use compile_expressions Setting in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, compile_expressions, JIT Compilation, Performance, Query Optimization

Description: Learn how ClickHouse's compile_expressions setting uses JIT compilation to speed up repeated expressions and aggregation functions in analytical queries.

---

ClickHouse includes a just-in-time (JIT) compiler that can convert frequently executed query expressions into native machine code at runtime. The `compile_expressions` setting controls this behavior. When enabled, ClickHouse compiles expressions and aggregation functions after they have been executed a certain number of times, replacing the interpreter with optimized native code.

## How JIT Compilation Works in ClickHouse

ClickHouse uses LLVM to compile query expressions to native code. The compilation happens in the background after a query has run `min_count_to_compile_expression` times (default: 3). Once compiled, subsequent executions use the native code path, which can be significantly faster for CPU-bound workloads.

This is particularly effective for:
- Complex arithmetic expressions evaluated over millions of rows
- Aggregation functions like `sum`, `count`, `avg` with filtering conditions
- Repeated sub-expression evaluation

## Enabling and Configuring

```sql
-- Enable JIT compilation for the session
SET compile_expressions = 1;

-- Set minimum execution count before compilation
SET min_count_to_compile_expression = 3;
```

For server-wide defaults, configure in `users.xml`:

```xml
<clickhouse>
  <profiles>
    <default>
      <compile_expressions>1</compile_expressions>
      <min_count_to_compile_expression>3</min_count_to_compile_expression>
    </default>
  </profiles>
</clickhouse>
```

## Example: Expressions That Benefit from JIT

Complex mathematical expressions see the most improvement:

```sql
SELECT
    sensor_id,
    sum(
        (temperature - 32.0) * 5.0 / 9.0 * calibration_factor + baseline_offset
    ) AS adjusted_temp_sum,
    count() AS readings
FROM sensor_data
WHERE event_date >= today() - 30
GROUP BY sensor_id
SETTINGS compile_expressions = 1;
```

After this query runs 3 times, the inner arithmetic expression gets JIT-compiled. The fourth execution and beyond use native code.

## Monitoring Compilation Activity

```sql
SELECT
    name,
    value
FROM system.events
WHERE name LIKE '%Compile%';
```

```sql
SELECT
    ProfileEvents['CompileFunction'] AS compiled_functions,
    ProfileEvents['CompileExpressionsMicroseconds'] AS compile_time_us,
    query_duration_ms,
    query
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_date = today()
ORDER BY event_time DESC
LIMIT 10;
```

## When JIT Compilation Helps

JIT compilation provides meaningful speedups when:

1. The same expression runs repeatedly across many query executions (dashboards, scheduled reports).
2. Expressions involve non-trivial arithmetic that keeps the CPU busy.
3. Queries scan hundreds of millions of rows per execution.

For one-off queries or simple expressions, the compilation overhead may not be worth it. The `min_count_to_compile_expression` threshold ensures compilation only happens for hot paths.

## Caveats and Considerations

- JIT compilation adds a small warm-up cost during the first few executions while LLVM compiles the code.
- The compiled code cache is per-process and resets on server restart.
- Not all expression types are JIT-compilable; ClickHouse falls back to interpretation for unsupported operations.

```sql
-- Check if JIT is available on your ClickHouse build
SELECT value
FROM system.build_options
WHERE name = 'USE_EMBEDDED_COMPILER';
```

If the value is `0`, JIT compilation is not available in your build.

## Summary

The `compile_expressions` setting enables LLVM-based JIT compilation for frequently executed query expressions in ClickHouse. Enable it for workloads with repeated complex arithmetic or aggregations over large datasets. The `min_count_to_compile_expression` threshold prevents unnecessary compilation overhead for one-off queries.
