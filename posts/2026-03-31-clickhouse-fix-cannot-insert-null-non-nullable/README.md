# How to Fix 'Cannot insert NULL into non-nullable column' in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, NULL, Error, Data Ingestion, Schema

Description: Fix 'Cannot insert NULL into non-nullable column' errors in ClickHouse by handling NULLs at insert time or altering column nullability.

---

ClickHouse columns are non-nullable by default for performance reasons. When you try to insert a NULL into a column defined as `UInt64`, `String`, or any other non-nullable type, ClickHouse raises: `DB::Exception: Cannot insert NULL value into column`.

## Identify the Column

The error message names the column. Confirm its type:

```sql
SELECT name, type, default_expression
FROM system.columns
WHERE table = 'my_table' AND database = 'my_db'
  AND name = 'the_column';
```

A non-nullable `String` column shows type `String`, while a nullable one shows `Nullable(String)`.

## Option 1 - Make the Column Nullable

If NULLs are semantically valid in your data model:

```sql
ALTER TABLE my_table MODIFY COLUMN user_email Nullable(String);
```

For replicated tables, use ON CLUSTER:

```sql
ALTER TABLE my_table ON CLUSTER my_cluster
MODIFY COLUMN user_email Nullable(String);
```

## Option 2 - Replace NULLs at Insert Time

If you prefer to keep columns non-nullable (better performance), replace NULLs with defaults at insert time:

```sql
INSERT INTO my_table (user_id, user_email)
SELECT
    user_id,
    coalesce(user_email, '') AS user_email
FROM staging_table;
```

## Option 3 - Set Column Default

Define a default value that is used when NULL is inserted:

```sql
ALTER TABLE my_table MODIFY COLUMN user_email String DEFAULT '';
```

Enable automatic default substitution:

```sql
SET input_format_defaults_for_omitted_fields = 1;
```

## Fix in ETL Pipeline

If the source data contains NULLs from a relational database, handle them in the pipeline:

```sql
-- Using ifNull to replace NULLs during SELECT
SELECT
    user_id,
    ifNull(email, 'unknown@example.com') AS email,
    ifNull(age, 0) AS age
FROM mysql_source_table;
```

## Bulk Insert with NULL Handling

For CSV imports, set the NULL representation and a fallback:

```sql
SET format_csv_null_representation = '\\N';
SET input_format_defaults_for_omitted_fields = 1;

INSERT INTO my_table FORMAT CSV;
```

## Summary

"Cannot insert NULL into non-nullable column" is resolved by either making the column Nullable if NULLs are valid, or replacing NULLs with defaults at insert time using `coalesce` or `ifNull`. Keep columns non-nullable where possible - ClickHouse query performance is better without Nullable types since they add an extra byte overhead per value.
