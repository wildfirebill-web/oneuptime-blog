# How to Fix 'Limit for rows exceeded' in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Row Limit, Error, Query, Troubleshooting

Description: Fix 'Limit for rows exceeded' errors in ClickHouse by adjusting result size limits and understanding max_result_rows vs max_rows_to_read settings.

---

ClickHouse has configurable limits on the number of rows a query can read or return. "Limit for rows exceeded" means a configured limit was hit. Two settings commonly cause this: `max_result_rows` (limits output rows) and `max_rows_to_read` (limits rows scanned from storage).

## Identify Which Limit Was Hit

```text
Code: 396. DB::Exception: Limit for result exceeded,
  max_result_rows = 1000, result_rows = 1001.

Code: 158. DB::Exception: Limit for rows read exceeded,
  max_rows_to_read = 1000000, rows read = 1000001.
```

Check your current limits:

```sql
SELECT name, value
FROM system.settings
WHERE name IN (
    'max_result_rows',
    'max_rows_to_read',
    'result_overflow_mode',
    'read_overflow_mode'
);
```

## Raise max_result_rows

If you need more rows in the output:

```sql
SET max_result_rows = 0;  -- 0 means unlimited
SELECT * FROM large_table WHERE event_date = today();
```

Or set a specific higher limit:

```sql
SET max_result_rows = 10000000;
```

## Raise max_rows_to_read

If the limit is on rows scanned (not returned):

```sql
SET max_rows_to_read = 0;  -- Unlimited
SELECT count() FROM large_table;
```

## Change Overflow Mode to Break Instead of Throw

By default, exceeding limits throws an exception. You can configure ClickHouse to return partial results instead:

```sql
SET result_overflow_mode = 'break';  -- Return what fits, no error
SET read_overflow_mode = 'break';

SELECT * FROM large_table WHERE event_date = today();
```

## Set Limits Per User Profile

In `users.xml`, control limits by user profile:

```xml
<profiles>
  <readonly_user>
    <max_result_rows>100000</max_result_rows>
    <result_overflow_mode>break</result_overflow_mode>
    <max_rows_to_read>1000000000</max_rows_to_read>
    <read_overflow_mode>throw</read_overflow_mode>
  </readonly_user>
  <analyst>
    <max_result_rows>0</max_result_rows>
    <max_rows_to_read>0</max_rows_to_read>
  </analyst>
</profiles>
```

## Use LIMIT to Avoid the Error

The proper way to handle large result sets is pagination with LIMIT/OFFSET:

```sql
SELECT * FROM large_table
WHERE event_date = today()
ORDER BY event_time DESC
LIMIT 10000 OFFSET 0;
```

## Summary

"Limit for rows exceeded" errors in ClickHouse are controlled by `max_result_rows` and `max_rows_to_read`. Set them to 0 for unlimited, raise them for specific use cases, or change `result_overflow_mode` to `break` to return partial results without an error. For production APIs, always use explicit LIMIT clauses rather than relying on server-side row limits.
