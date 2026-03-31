# How to Use JSONExtractInt() and JSONExtractFloat() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, JSON, JSONExtractInt, JSONExtractFloat, Function

Description: Learn how to extract integer and float values from JSON columns in ClickHouse using JSONExtractInt() and JSONExtractFloat() for analytics queries.

---

When JSON payloads contain numeric fields - counts, prices, scores, timestamps - you need to extract them as actual numbers rather than strings. `JSONExtractInt()` and `JSONExtractFloat()` handle this by returning typed numeric values that can be used directly in calculations, aggregations, and comparisons.

## How These Functions Work

Both functions traverse the JSON document using the provided path and return a typed numeric value:

- `JSONExtractInt()` returns an `Int64` value. It truncates fractional values.
- `JSONExtractFloat()` returns a `Float64` value. It preserves fractional precision.

If the path does not exist or the value cannot be interpreted as a number, both functions return `0`.

## Syntax

```sql
JSONExtractInt(json, path_element [, path_element ...])
JSONExtractFloat(json, path_element [, path_element ...])
```

## Examples

### Basic Extraction

```sql
SELECT
    JSONExtractInt('{"count": 42, "score": 9.87}',   'count') AS count_val,
    JSONExtractFloat('{"count": 42, "score": 9.87}', 'score') AS score_val;
```

```text
count_val  score_val
42         9.87
```

### JSONExtractInt Truncates Floats

When a float value is extracted with `JSONExtractInt()`, it is truncated:

```sql
SELECT
    JSONExtractInt('{"value": 3.99}',   'value') AS truncated,
    JSONExtractFloat('{"value": 3.99}', 'value') AS precise;
```

```text
truncated  precise
3          3.99
```

### Accessing Nested Numeric Fields

```sql
SELECT
    JSONExtractFloat('{"order": {"subtotal": 49.95, "tax": 4.50}}',
        'order', 'subtotal') AS subtotal,
    JSONExtractFloat('{"order": {"subtotal": 49.95, "tax": 4.50}}',
        'order', 'tax') AS tax;
```

```text
subtotal  tax
49.95     4.5
```

### Extracting from Arrays

Use integer indices to access array elements:

```sql
SELECT JSONExtractInt('{"scores": [10, 20, 30]}', 'scores', 2) AS third_score;
```

```text
third_score
30
```

### Using Numeric Extracts in Aggregations

```sql
SELECT
    sum(JSONExtractFloat(payload, 'amount')) AS total_amount,
    avg(JSONExtractFloat(payload, 'amount')) AS avg_amount
FROM (
    SELECT '{"amount": 19.99}' AS payload
    UNION ALL SELECT '{"amount": 49.95}' AS payload
    UNION ALL SELECT '{"amount": 9.00}'  AS payload
);
```

```text
total_amount  avg_amount
78.94         26.313...
```

### Complete Working Example

Analyze sales events stored as JSON:

```sql
CREATE TABLE sales_events
(
    event_id UInt64,
    payload  String
) ENGINE = MergeTree()
ORDER BY event_id;

INSERT INTO sales_events VALUES
    (1, '{"product_id": 101, "quantity": 2,  "unit_price": 29.99}'),
    (2, '{"product_id": 102, "quantity": 1,  "unit_price": 149.00}'),
    (3, '{"product_id": 101, "quantity": 5,  "unit_price": 29.99}'),
    (4, '{"product_id": 103, "quantity": 3,  "unit_price": 9.50}');

SELECT
    JSONExtractInt(payload,   'product_id')                           AS product_id,
    sum(JSONExtractInt(payload, 'quantity'))                          AS total_qty,
    sum(
        JSONExtractInt(payload, 'quantity') *
        JSONExtractFloat(payload, 'unit_price')
    )                                                                 AS total_revenue
FROM sales_events
GROUP BY product_id
ORDER BY total_revenue DESC;
```

```text
product_id  total_qty  total_revenue
102         1          149
101         7          209.93
103         3          28.5
```

## Summary

`JSONExtractInt()` and `JSONExtractFloat()` extract typed numeric values from JSON strings in ClickHouse, enabling aggregations, calculations, and comparisons directly on JSON data. `JSONExtractInt()` truncates decimals and returns `Int64`; `JSONExtractFloat()` preserves fractions and returns `Float64`. Both return `0` when the path is missing. For high-frequency analytic queries, materialize extracted numeric columns to avoid repeated JSON parsing.
