# How to Use JSONExtractBool() and JSONExtractRaw() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, JSON, JSONExtractBool, JSONExtractRaw, Function

Description: Learn how to extract boolean and raw JSON values from JSON strings in ClickHouse using JSONExtractBool() and JSONExtractRaw().

---

JSON payloads often contain boolean flags and nested sub-objects. `JSONExtractBool()` extracts a `true`/`false` field as a ClickHouse `UInt8`, and `JSONExtractRaw()` returns an un-parsed sub-document as a raw string, letting you pass it on to further extraction functions.

## How These Functions Work

- `JSONExtractBool(json, path...)` - returns `1` for `true`, `0` for `false`, and `0` when the key is missing or not boolean.
- `JSONExtractRaw(json, path...)` - returns the raw JSON fragment at the given path as a `String`. This includes objects, arrays, strings (with quotes), numbers, and booleans in their original JSON form.

## Syntax

```sql
JSONExtractBool(json, path_element [, path_element ...])
JSONExtractRaw(json, path_element [, path_element ...])
```

## Difference Between JSONExtractRaw and JSONExtractString

```mermaid
graph LR
    A["JSON: {\"label\": \"hello\"}"] --> B[JSONExtractString]
    A --> C[JSONExtractRaw]
    B --> D["hello  (no quotes)"]
    C --> E["\"hello\"  (with quotes)"]
```

## Examples

### Extracting Boolean Values

```sql
SELECT
    JSONExtractBool('{"active": true,  "deleted": false}', 'active')  AS is_active,
    JSONExtractBool('{"active": true,  "deleted": false}', 'deleted') AS is_deleted;
```

```text
is_active  is_deleted
1          0
```

### Using Boolean in WHERE Clause

```sql
SELECT user_id
FROM (
    SELECT 1 AS user_id, '{"active": true}'  AS profile
    UNION ALL
    SELECT 2 AS user_id, '{"active": false}' AS profile
    UNION ALL
    SELECT 3 AS user_id, '{"active": true}'  AS profile
)
WHERE JSONExtractBool(profile, 'active') = 1;
```

```text
user_id
1
3
```

### Extracting Raw Sub-Objects

`JSONExtractRaw()` returns the sub-document including its container brackets:

```sql
SELECT JSONExtractRaw(
    '{"user": {"id": 42, "name": "Alice"}, "score": 99}',
    'user'
) AS raw_user;
```

```text
raw_user
{"id": 42, "name": "Alice"}
```

### Chaining JSONExtractRaw with Further Extraction

Extract a sub-object and then extract fields from it:

```sql
WITH '{"order": {"id": 1001, "amount": 59.99, "paid": true}}' AS doc
SELECT
    JSONExtractRaw(doc, 'order')                        AS raw_order,
    JSONExtractInt(JSONExtractRaw(doc, 'order'), 'id')  AS order_id,
    JSONExtractBool(JSONExtractRaw(doc, 'order'), 'paid') AS is_paid;
```

```text
raw_order                                    order_id  is_paid
{"id": 1001, "amount": 59.99, "paid": true}  1001      1
```

### Extracting Raw Arrays

`JSONExtractRaw()` returns the array literal when pointing to an array:

```sql
SELECT JSONExtractRaw('{"tags": ["a", "b", "c"]}', 'tags') AS raw_tags;
```

```text
raw_tags
["a", "b", "c"]
```

### Complete Working Example

Process feature flag payloads for A/B testing analysis:

```sql
CREATE TABLE ab_events
(
    event_id UInt64,
    payload  String
) ENGINE = MergeTree()
ORDER BY event_id;

INSERT INTO ab_events VALUES
    (1, '{"user": "u1", "flags": {"new_ui": true,  "beta_search": false}}'),
    (2, '{"user": "u2", "flags": {"new_ui": false, "beta_search": true}}'),
    (3, '{"user": "u3", "flags": {"new_ui": true,  "beta_search": true}}');

SELECT
    JSONExtractString(payload, 'user')                       AS user_id,
    JSONExtractBool(JSONExtractRaw(payload, 'flags'), 'new_ui')      AS new_ui,
    JSONExtractBool(JSONExtractRaw(payload, 'flags'), 'beta_search') AS beta_search
FROM ab_events
ORDER BY event_id;
```

```text
user_id  new_ui  beta_search
u1       1       0
u2       0       1
u3       1       1
```

## Summary

`JSONExtractBool()` converts JSON boolean fields to ClickHouse `UInt8` (1/0), making them usable in WHERE clauses and aggregations. `JSONExtractRaw()` returns the raw JSON fragment at a given path as a string, which is useful for extracting sub-objects that you then pass to other extraction functions. Together, these two functions handle the boolean and structural navigation needs of JSON processing in ClickHouse.
