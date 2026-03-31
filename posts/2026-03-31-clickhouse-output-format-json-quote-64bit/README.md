# How to Use output_format_json_quote_64bit_integers in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Database, Configuration, JSON, output_format_json_quote_64bit_integers, Format

Description: Learn how output_format_json_quote_64bit_integers controls JSON serialization of large integers in ClickHouse to ensure compatibility with JavaScript clients.

---

JavaScript's `Number` type can only safely represent integers up to 2^53 - 1 (approximately 9 quadrillion). ClickHouse's UInt64 and Int64 types can hold values up to 2^64 - 1, far exceeding this limit. The `output_format_json_quote_64bit_integers` setting controls whether ClickHouse serializes these large integers as JSON strings (quoted) to prevent precision loss in JavaScript clients.

## The Problem with Large Integers in JSON

Consider a ClickHouse UInt64 column storing a timestamp in nanoseconds or a large snowflake ID:

```sql
SELECT toUInt64(1711900000000000000) AS nanosecond_ts;
```

When returned as JSON with the setting disabled, the output is:

```json
{"nanosecond_ts": 1711900000000000000}
```

A JavaScript client parsing this will silently lose precision because the number exceeds `Number.MAX_SAFE_INTEGER` (9007199254740991). The value gets rounded to the nearest representable float.

## Enabling Quoted Output

With `output_format_json_quote_64bit_integers = 1` (the default), 64-bit integers are returned as strings:

```json
{"nanosecond_ts": "1711900000000000000"}
```

Your application can then parse the string using a BigInt library or store it as a string.

## Checking the Default

```sql
SELECT value, changed
FROM system.settings
WHERE name = 'output_format_json_quote_64bit_integers';
```

The default is `1` (quoting enabled).

## Querying with the Setting

Disable quoting when your client can handle 64-bit integers natively (e.g., Python, Java, Go):

```sql
SELECT
    user_id,
    sum(pageviews) AS total_pageviews,
    max(last_visit_ts) AS last_visit
FROM user_stats
GROUP BY user_id
FORMAT JSON
SETTINGS output_format_json_quote_64bit_integers = 0;
```

Enable it explicitly when using HTTP API with a JavaScript frontend:

```sql
SELECT
    event_id,
    user_id,
    event_time_ns
FROM events
WHERE event_date = today()
FORMAT JSON
SETTINGS output_format_json_quote_64bit_integers = 1;
```

## Setting It via HTTP Interface

When calling ClickHouse over HTTP from a JavaScript application:

```bash
curl "http://localhost:8123/?query=SELECT+user_id,+sum(amount)+AS+total+FROM+orders+GROUP+BY+user_id+FORMAT+JSON&output_format_json_quote_64bit_integers=1"
```

## Configuring in users.xml

For a JavaScript-facing service account, enable quoting by default:

```xml
<profiles>
  <javascript_service>
    <output_format_json_quote_64bit_integers>1</output_format_json_quote_64bit_integers>
  </javascript_service>
  <backend_service>
    <output_format_json_quote_64bit_integers>0</output_format_json_quote_64bit_integers>
  </backend_service>
</profiles>
```

## Related Settings

`output_format_json_quote_denormals` controls how NaN and Infinity values are serialized:

```sql
SELECT
    user_id,
    sum(amount) / nullIf(count(), 0) AS avg_amount
FROM orders
GROUP BY user_id
FORMAT JSON
SETTINGS
    output_format_json_quote_64bit_integers = 1,
    output_format_json_quote_denormals = 1;
```

`output_format_json_quote_decimals` controls Decimal type quoting:

```sql
SELECT
    product_id,
    toDecimal64(sum(price), 2) AS total_price
FROM order_items
GROUP BY product_id
FORMAT JSON
SETTINGS
    output_format_json_quote_64bit_integers = 1,
    output_format_json_quote_decimals = 1;
```

## Handling Quoted Integers in Client Code

In JavaScript:

```text
// With quoting enabled, parse BigInt from string
const row = { nanosecond_ts: "1711900000000000000" };
const ts = BigInt(row.nanosecond_ts);
```

In Python (where native 64-bit integers are safe):

```text
# Quoting not needed, but harmless
import json
data = json.loads(response_text)
ts = int(data['data'][0]['nanosecond_ts'])
```

## Summary

`output_format_json_quote_64bit_integers` is an important setting for applications that consume ClickHouse JSON output in JavaScript or other environments that cannot safely represent 64-bit integers. Keeping it enabled (the default) prevents silent precision loss for large integers like snowflake IDs, nanosecond timestamps, and large counters. Disable it only when your backend client language handles 64-bit integers natively.
