# How to Use toYYYYMMDDhhmmss() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Date Function, Storage Optimization, Analytics, Timestamp

Description: Learn how toYYYYMMDDhhmmss() encodes a DateTime as a sortable UInt64 integer combining date and time, useful for compact keys and timestamp storage.

---

`toYYYYMMDDhhmmss(dt)` converts a `DateTime` value into a `UInt64` integer of the form `YYYYMMDDhhmmss`. For example, `2024-06-15 14:32:07` becomes `20240615143207`. The result is a 14-digit integer that encodes both date and time in a single number. Because larger values always represent later moments in time, the integer is naturally sortable. This makes it useful for compact timestamp keys, human-readable audit trails, and scenarios where you want a single integer column that serves both as a sort key and a human-readable timestamp.

## Basic Usage

```sql
-- Observe the encoding for a sample DateTime
SELECT
    toDateTime('2024-06-15 14:32:07') AS dt,
    toYYYYMMDDhhmmss(dt) AS encoded;
```

```text
dt                      encoded
2024-06-15 14:32:07     20240615143207
```

The 14-digit integer packs year, month, day, hour, minute, and second in order, so lexicographic and numeric sort orders both agree with chronological order.

## Using as a Sortable Timestamp Key

In systems that export data to formats that do not support a native DateTime type (such as CSV exports or certain downstream tools), storing the timestamp as a `UInt64` from `toYYYYMMDDhhmmss` preserves sortability.

```sql
-- Create a table with a compact timestamp key
CREATE TABLE audit_events
(
    event_key   UInt64,  -- stores toYYYYMMDDhhmmss(event_time)
    user_id     UInt64,
    action      String,
    event_time  DateTime
)
ENGINE = MergeTree()
ORDER BY event_key;
```

```sql
-- Insert with the computed key
INSERT INTO audit_events (event_key, user_id, action, event_time)
SELECT
    toYYYYMMDDhhmmss(event_time),
    user_id,
    action,
    event_time
FROM raw_audit_events;
```

## Range Filtering With the Encoded Integer

You can filter rows by constructing the integer boundaries directly from known date-time strings.

```sql
-- Query events between two specific times using integer comparison
SELECT
    event_key,
    user_id,
    action,
    event_time
FROM audit_events
WHERE event_key BETWEEN 20240615000000 AND 20240615235959
ORDER BY event_key;
```

This is equivalent to filtering on the full `event_time` column but avoids parsing a string at query time.

## Compact Timestamp in Exported Reports

For reports exported to CSV or spreadsheets, a single integer column is easier to work with than a formatted timestamp string that may cause timezone confusion.

```sql
-- Export a daily report with compact timestamps
SELECT
    toYYYYMMDDhhmmss(completed_at) AS completed_ts,
    order_id,
    customer_id,
    total_amount
FROM orders
WHERE toYYYYMMDD(completed_at) = 20240615
ORDER BY completed_ts;
```

## Generating a Human-Readable Audit Key

Combining `toYYYYMMDDhhmmss` with a sequence or hash gives you audit keys that are both unique and instantly human-parseable.

```sql
-- Construct a human-readable composite key
SELECT
    concat(
        toString(toYYYYMMDDhhmmss(event_time)),
        '-',
        toString(user_id)
    ) AS audit_key,
    action,
    event_time
FROM audit_events
LIMIT 10;
```

```text
audit_key               action          event_time
20240615143207-1042     login           2024-06-15 14:32:07
20240615143512-2187     update_profile  2024-06-15 14:35:12
```

## Converting Back to DateTime

To go from the integer back to a `DateTime`, extract the components using integer division and modulo, then assemble a string.

```sql
-- Decode a toYYYYMMDDhhmmss integer back to a DateTime string
SELECT
    encoded,
    toDateTime(
        concat(
            toString(intDiv(encoded, 10000000000)),           -- YYYY
            '-',
            lpad(toString(intDiv(encoded MOD 10000000000, 100000000)), 2, '0'), -- MM
            '-',
            lpad(toString(intDiv(encoded MOD 100000000, 1000000)), 2, '0'),     -- DD
            ' ',
            lpad(toString(intDiv(encoded MOD 1000000, 10000)), 2, '0'),         -- hh
            ':',
            lpad(toString(intDiv(encoded MOD 10000, 100)), 2, '0'),             -- mm
            ':',
            lpad(toString(encoded MOD 100), 2, '0')                             -- ss
        )
    ) AS recovered_dt
FROM (SELECT toYYYYMMDDhhmmss(now()) AS encoded);
```

## Comparing toYYYYMMDDhhmmss With Unix Timestamps

Both `toYYYYMMDDhhmmss` and `toUnixTimestamp` produce integers suitable for sort keys, but they differ in intent.

```sql
SELECT
    now() AS dt,
    toYYYYMMDDhhmmss(now()) AS yyyymmddhhmmss,  -- human-readable: 20240615143207
    toUnixTimestamp(now())  AS unix_ts;          -- seconds since epoch: 1718458327
```

Use `toYYYYMMDDhhmmss` when human readability matters (audit logs, exported keys). Use `toUnixTimestamp` when arithmetic over durations is the priority.

## Summary

`toYYYYMMDDhhmmss(dt)` encodes a `DateTime` as a 14-digit `UInt64` integer in `YYYYMMDDhhmmss` format. Because the encoding preserves chronological order numerically, it works as a sort key and enables efficient integer range filtering. It is especially useful in audit logs, CSV exports, and composite keys where a single, human-readable timestamp integer is more convenient than a formatted string or a separate `DateTime` column.
