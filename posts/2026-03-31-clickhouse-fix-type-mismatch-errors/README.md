# How to Fix "Type mismatch" Errors in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Type, Error, Data Type, Troubleshooting

Description: Resolve ClickHouse type mismatch errors by understanding strict type coercion rules and using explicit CAST or conversion functions.

---

ClickHouse has strict type checking. "Type mismatch" errors occur when an operation receives incompatible types - for example, comparing a String to a UInt64, or passing a Nullable(T) where T is expected.

## Identify the Type Mismatch

The error message names the conflicting types:

```text
Code: 386. DB::Exception: Type mismatch in IN section:
  UInt64 cannot be equal to String.
```

Check column types:

```sql
SELECT name, type
FROM system.columns
WHERE table = 'my_table' AND database = 'my_db';
```

## Explicit Type Casting

Use `CAST` or type conversion functions:

```sql
-- Compare a String column to a UInt64 literal
SELECT * FROM events WHERE CAST(user_id_str AS UInt64) = 12345;

-- Or convert the literal to String
SELECT * FROM events WHERE user_id_str = toString(12345);
```

## Fix Nullable Mismatches

Many ClickHouse functions do not accept `Nullable(T)`. Use `assumeNotNull` or `coalesce`:

```sql
-- Fails if column is Nullable(String):
SELECT length(nullable_col) FROM t;

-- Fix:
SELECT length(assumeNotNull(nullable_col)) FROM t;
-- or
SELECT length(coalesce(nullable_col, '')) FROM t;
```

## Fix IN Clause Type Mismatches

The IN set must match the column type exactly:

```sql
-- Column is UInt32, values in IN must also be numeric:
SELECT * FROM events WHERE user_id IN (1, 2, 3);  -- OK
SELECT * FROM events WHERE user_id IN ('1', '2');  -- Type mismatch

-- Fix: cast the column or use numeric literals
SELECT * FROM events WHERE toString(user_id) IN ('1', '2');
```

## Fix JOIN Key Type Mismatches

JOIN keys must be the same type:

```sql
-- Fails if orders.user_id is UInt64 and users.id is String:
SELECT * FROM orders o JOIN users u ON o.user_id = u.id;

-- Fix:
SELECT * FROM orders o
JOIN users u ON toString(o.user_id) = u.id;
```

## Fix DateTime vs Date Mismatches

```sql
-- Comparing Date to DateTime
SELECT * FROM events
WHERE event_date = now();  -- Type mismatch

-- Fix: use toDate() to convert
SELECT * FROM events
WHERE event_date = toDate(now());
```

## Summary

ClickHouse type mismatch errors require explicit conversions using `CAST`, `toString`, `toUInt64`, `toDate`, and similar functions. Pay special attention to `Nullable(T)` vs `T`, String vs numeric types in IN clauses, and Date vs DateTime comparisons. Using `DESCRIBE TABLE` and `system.columns` to confirm types before writing complex queries prevents most type mismatch issues.
