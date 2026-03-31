# How to Use isIPv4String() and isIPv6String() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Network, IP Address, Validation, Analytics

Description: Learn how isIPv4String() and isIPv6String() validate IP address strings in ClickHouse, enabling safe parsing, data quality checks, and network log analysis.

---

ClickHouse provides two lightweight validation functions for IP address strings: `isIPv4String(str)` returns `1` if the input is a valid dotted-decimal IPv4 address, and `isIPv6String(str)` returns `1` if the input is a valid IPv6 address in any standard notation. Both return `UInt8` (0 or 1). They are useful for data quality checks before calling conversion functions like `toIPv4()` or `toIPv6()`, which throw exceptions on invalid input.

## Basic Usage

```sql
-- Validate individual IP strings
SELECT
    isIPv4String('192.168.1.1')    AS valid_ipv4,
    isIPv4String('999.0.0.1')      AS invalid_ipv4,
    isIPv4String('not-an-ip')      AS not_ip,
    isIPv6String('::1')            AS valid_ipv6_loopback,
    isIPv6String('2001:db8::1')    AS valid_ipv6,
    isIPv6String('192.168.1.1')    AS ipv4_as_ipv6;
```

```text
valid_ipv4  invalid_ipv4  not_ip  valid_ipv6_loopback  valid_ipv6  ipv4_as_ipv6
1           0             0       1                    1           0
```

An IPv4 address is not considered a valid IPv6 string and vice versa - the functions are exclusive.

## Classifying IP Addresses in a Table

```sql
-- Classify raw log strings into IPv4, IPv6, or invalid
SELECT
    raw_ip,
    CASE
        WHEN isIPv4String(raw_ip) THEN 'IPv4'
        WHEN isIPv6String(raw_ip) THEN 'IPv6'
        ELSE 'invalid'
    END AS ip_version
FROM (
    SELECT arrayJoin([
        '10.0.0.1',
        '2001:db8:85a3::8a2e:370:7334',
        '::ffff:192.0.2.1',
        'garbage',
        '256.1.1.1',
        '172.16.0.0'
    ]) AS raw_ip
);
```

```text
raw_ip                         ip_version
10.0.0.1                       IPv4
2001:db8:85a3::8a2e:370:7334   IPv6
::ffff:192.0.2.1               IPv6
garbage                        invalid
256.1.1.1                      invalid
172.16.0.0                     IPv4
```

## Data Quality Check on a Log Table

Use the functions to measure how many rows have malformed IP fields before ingesting or transforming data.

```sql
-- Audit IP field quality in a web access log table
SELECT
    count()                                         AS total_rows,
    countIf(isIPv4String(client_ip))                AS valid_ipv4_count,
    countIf(isIPv6String(client_ip))                AS valid_ipv6_count,
    countIf(NOT isIPv4String(client_ip)
            AND NOT isIPv6String(client_ip))         AS invalid_count,
    round(100.0 * countIf(NOT isIPv4String(client_ip)
            AND NOT isIPv6String(client_ip))
            / count(), 2)                           AS invalid_pct
FROM access_logs;
```

## Safe Conversion Using isIPv4String()

`toIPv4()` raises an exception on bad input. Guard calls with `isIPv4String()` to produce NULLs instead.

```sql
-- Safe conversion: only parse valid IPv4 strings
SELECT
    client_ip,
    if(isIPv4String(client_ip), toIPv4(client_ip), NULL) AS parsed_ipv4
FROM access_logs
LIMIT 10;
```

For ClickHouse 22.3+, you can also use `toIPv4OrNull()` / `toIPv6OrNull()`, but `isIPv4String` remains useful when you need to branch on the address family before any conversion.

## Filtering Only Valid IPv4 Addresses

```sql
-- Aggregate traffic only from syntactically valid IPv4 sources
SELECT
    toIPv4(client_ip)        AS ip,
    count()                  AS requests,
    sum(response_bytes)      AS bytes_sent
FROM access_logs
WHERE isIPv4String(client_ip)
GROUP BY ip
ORDER BY requests DESC
LIMIT 20;
```

## Detecting IPv4-Mapped IPv6 Addresses

IPv4-mapped IPv6 addresses (e.g. `::ffff:192.0.2.1`) pass `isIPv6String()` but not `isIPv4String()`.

```sql
-- Find IPv4-mapped IPv6 addresses in the log
SELECT
    client_ip,
    isIPv4String(client_ip) AS is_v4,
    isIPv6String(client_ip) AS is_v6,
    client_ip LIKE '::ffff:%' AS is_v4_mapped
FROM access_logs
WHERE isIPv6String(client_ip)
  AND client_ip LIKE '::ffff:%'
LIMIT 10;
```

## Routing Dual-Stack Events to Separate Tables with a Materialized View

```sql
-- Source table that accepts raw events
CREATE TABLE raw_events (
    event_time DateTime,
    client_ip  String,
    event_type String
) ENGINE = MergeTree ORDER BY event_time;

-- Separate tables for each address family
CREATE TABLE events_ipv4 (
    event_time DateTime,
    client_ip  IPv4,
    event_type String
) ENGINE = MergeTree ORDER BY event_time;

CREATE TABLE events_ipv6 (
    event_time DateTime,
    client_ip  IPv6,
    event_type String
) ENGINE = MergeTree ORDER BY event_time;

-- Materialized view routes valid IPv4 rows
CREATE MATERIALIZED VIEW mv_ipv4 TO events_ipv4 AS
SELECT
    event_time,
    toIPv4(client_ip) AS client_ip,
    event_type
FROM raw_events
WHERE isIPv4String(client_ip);

-- Materialized view routes valid IPv6 rows
CREATE MATERIALIZED VIEW mv_ipv6 TO events_ipv6 AS
SELECT
    event_time,
    toIPv6(client_ip) AS client_ip,
    event_type
FROM raw_events
WHERE isIPv6String(client_ip);
```

## Counting Versions Over Time

```sql
-- Track the ratio of IPv4 vs IPv6 clients per day
SELECT
    toDate(event_time)                     AS day,
    countIf(isIPv4String(client_ip))       AS ipv4_requests,
    countIf(isIPv6String(client_ip))       AS ipv6_requests,
    round(100.0 * countIf(isIPv6String(client_ip)) / count(), 2) AS ipv6_pct
FROM access_logs
GROUP BY day
ORDER BY day DESC
LIMIT 30;
```

## Summary

`isIPv4String()` and `isIPv6String()` are zero-overhead string validators that return `UInt8`. Use them to classify address families, guard against exceptions before calling conversion functions, build routing materialized views, and audit data quality in network log pipelines. They are mutually exclusive: an IPv4 address string will never pass `isIPv6String()` and vice versa.
