# How to Use JSONExtractRaw() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, JSON, Analytics, Query

Description: Learn how JSONExtractRaw() returns a raw JSON fragment from a field in ClickHouse, preserving nested structure as a String for further processing or forwarding.

---

`JSONExtractRaw` extracts a value from a JSON string and returns it as a raw, unparsed JSON fragment still wrapped in its original JSON syntax. Unlike `JSONExtractString`, which strips quotes and escape sequences from scalar strings, `JSONExtractRaw` preserves the exact JSON representation including surrounding quotes for strings, braces for objects, and brackets for arrays. This makes it the right tool when you need to pass a sub-document onward, store it in another column, or feed it into a subsequent JSON function.

## Basic Usage

```sql
-- Return the raw JSON value for each key
SELECT
    JSONExtractRaw('{"id": 1, "tags": ["a","b"], "meta": {"k": "v"}}', 'id')   AS raw_id,
    JSONExtractRaw('{"id": 1, "tags": ["a","b"], "meta": {"k": "v"}}', 'tags') AS raw_tags,
    JSONExtractRaw('{"id": 1, "tags": ["a","b"], "meta": {"k": "v"}}', 'meta') AS raw_meta;
```

```text
raw_id  raw_tags        raw_meta
1       ["a","b"]       {"k":"v"}
```

Notice that `raw_id` has no quotes because `1` is a JSON number. A JSON string like `"hello"` would appear as `"hello"` including the double quotes.

## Extracting a Sub-Document

```sql
-- Preserve the nested address object for downstream use
SELECT
    user_id,
    JSONExtractRaw(profile, 'address') AS raw_address
FROM users
WHERE JSONHas(profile, 'address') = 1
LIMIT 10;
```

The `raw_address` column is still a `String`, but it contains valid JSON that can be parsed again.

## Chaining JSONExtractRaw with Other Functions

Because the result is a JSON string, it can be passed into another JSON function.

```sql
-- Extract a field from a nested sub-document by chaining
SELECT
    user_id,
    JSONExtractString(
        JSONExtractRaw(profile, 'address'),
        'city'
    ) AS city
FROM users
LIMIT 10;
```

For deeper nesting, use the variadic path form of `JSONExtractString` directly instead of chaining, but chaining is useful when the sub-document is re-used in multiple expressions.

## Storing a Sub-Document in Another Column

```sql
-- Insert the raw nested object into a separate column for archival
INSERT INTO user_addresses (user_id, raw_address_json)
SELECT
    user_id,
    JSONExtractRaw(profile, 'address')
FROM users;
```

## Comparing Raw JSON Fragments

```sql
-- Find rows where the config object matches a known value
SELECT count()
FROM settings
WHERE JSONExtractRaw(config_json, 'theme') = '"dark"';
```

String values in raw JSON include surrounding double quotes, so the comparison string must include them.

## Using JSONExtractRaw in an Array Context

When a key holds a JSON array, `JSONExtractRaw` returns the full array string.

```sql
SELECT
    order_id,
    JSONExtractRaw(order_json, 'items') AS items_json_array
FROM orders
LIMIT 5;
```

You can then apply `JSONExtractArrayRaw` on the result to iterate over elements.

## Summary

`JSONExtractRaw` returns a JSON field as an unparsed fragment, preserving the full JSON syntax including quotes around strings and braces around objects. Use it when you need to forward a sub-document, chain it into another JSON function, or compare against a known JSON literal. For scalar extraction where you want ClickHouse to perform type conversion, prefer `JSONExtractString`, `JSONExtractInt`, or `JSONExtractFloat` instead.
