# How to Use toIPv4() and toIPv6() Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, IP Function, Analytics, Network, Type Conversion

Description: Learn how toIPv4() and toIPv6() cast IP address strings to ClickHouse native IPv4 and IPv6 types, enabling type-safe comparisons, CIDR range queries, and cleaner schemas.

---

ClickHouse provides native `IPv4` and `IPv6` column types for storing IP addresses in a compact and semantically meaningful way. `toIPv4(str)` converts a dotted-decimal string to the `IPv4` type (stored as `UInt32`). `toIPv6(str)` converts a colon-hex string to the `IPv6` type (stored as `FixedString(16)`). Both functions throw an exception if the input is not a valid IP address. For nullable inputs use `toIPv4OrNull()` / `toIPv6OrNull()`, or `toIPv4OrZero()` / `toIPv6OrZero()` for a default value on invalid input.

## Basic Usage

```sql
-- Convert string literals to native IP types
SELECT
    toIPv4('192.168.1.1')                     AS ipv4_val,
    toIPv4OrNull('not-an-ip')                 AS ipv4_null,
    toIPv4OrZero('not-an-ip')                 AS ipv4_zero,
    toIPv6('2001:db8::1')                     AS ipv6_val,
    toIPv6OrNull('not-an-ipv6')               AS ipv6_null,
    toIPv6OrZero('bad-input')                 AS ipv6_zero;
```

```text
ipv4_val     ipv4_null  ipv4_zero  ipv6_val    ipv6_null  ipv6_zero
192.168.1.1  \N         0.0.0.0    2001:db8::1  \N         ::
```

## Creating Tables With Native IP Types

```sql
-- Schema using native IPv4 and IPv6 column types
CREATE TABLE connection_events
(
    ts          DateTime,
    src_ip4     IPv4,
    dst_ip4     IPv4,
    src_ip6     IPv6,
    dst_ip6     IPv6,
    src_port    UInt16,
    dst_port    UInt16,
    protocol    LowCardinality(String),
    bytes_sent  UInt64
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(ts)
ORDER BY (ts, src_ip4);
```

```sql
-- Insert using toIPv4() and toIPv6() for safe type conversion
INSERT INTO connection_events VALUES
    (now(), toIPv4('203.0.113.10'), toIPv4('192.168.1.5'),
     toIPv6('::'), toIPv6('::'), 54321, 443, 'TCP', 1500),
    (now(), toIPv4('198.51.100.20'), toIPv4('10.0.0.1'),
     toIPv6('::'), toIPv6('::'), 60001, 80, 'TCP', 3200);
```

## Filtering With Native Type Comparisons

```sql
-- Direct comparison using the IPv4 type (no string parsing at query time)
SELECT
    src_ip4,
    dst_ip4,
    bytes_sent
FROM connection_events
WHERE toDate(ts) = yesterday()
  AND src_ip4 = toIPv4('203.0.113.10')
ORDER BY ts DESC
LIMIT 20;
```

## CIDR Range Query With toIPv4() and IPv4CIDRToRange()

```sql
-- Find all connections from the 203.0.113.0/24 block
SELECT
    src_ip4,
    count()         AS connections,
    sum(bytes_sent) AS total_bytes
FROM connection_events
WHERE toDate(ts) = yesterday()
  AND src_ip4 BETWEEN
      IPv4CIDRToRange(toIPv4('203.0.113.0'), 24).1 AND
      IPv4CIDRToRange(toIPv4('203.0.113.0'), 24).2
GROUP BY src_ip4
ORDER BY connections DESC
LIMIT 20;
```

## Validating IP Strings Before Insert

```sql
-- Use toIPv4OrNull() to detect invalid IPs before inserting
SELECT
    raw_ip,
    toIPv4OrNull(raw_ip)  AS parsed,
    parsed IS NULL         AS is_invalid
FROM (
    SELECT arrayJoin([
        '192.168.1.1',
        '256.0.0.1',
        'not-an-ip',
        '10.0.0.0',
        '0.0.0.0'
    ]) AS raw_ip
);
```

```text
raw_ip       parsed       is_invalid
192.168.1.1  192.168.1.1  0
256.0.0.1    \N           1
not-an-ip    \N           1
10.0.0.0     10.0.0.0     0
0.0.0.0      0.0.0.0      0
```

## Sorting IPs in Correct Numeric Order

```sql
-- IPv4 native type sorts numerically, not lexicographically
SELECT
    src_ip4
FROM connection_events
WHERE toDate(ts) = yesterday()
GROUP BY src_ip4
ORDER BY src_ip4 ASC
LIMIT 20;
```

## IPv6 CIDR Membership With toIPv6()

```sql
-- Connections from within the link-local range fe80::/10
SELECT
    src_ip6,
    count() AS connections
FROM connection_events
WHERE toDate(ts) = yesterday()
  AND src_ip6 BETWEEN
      IPv6CIDRToRange(toIPv6('fe80::'), 10).1 AND
      IPv6CIDRToRange(toIPv6('fe80::'), 10).2
GROUP BY src_ip6
ORDER BY connections DESC
LIMIT 20;
```

## Safe Batch Conversion With OrZero Variants

```sql
-- Convert a column of raw strings, defaulting invalid entries to 0.0.0.0
INSERT INTO clean_ip_table
SELECT
    ts,
    toIPv4OrZero(raw_ip) AS ip4,
    raw_ip
FROM raw_logs
WHERE toDate(ts) = yesterday();

-- Count how many rows had invalid IPs
SELECT
    countIf(toIPv4OrNull(raw_ip) IS NULL) AS invalid_count,
    count()                               AS total
FROM raw_logs
WHERE toDate(ts) = yesterday();
```

## Summary

`toIPv4()` and `toIPv6()` convert string IP addresses to ClickHouse's native `IPv4` and `IPv6` types, which store data compactly (`UInt32` and `FixedString(16)` respectively) and support proper numeric sorting and range comparisons. Use `toIPv4OrNull()` / `toIPv6OrNull()` to handle potentially invalid input safely. Native IP types integrate directly with `IPv4CIDRToRange()` and `IPv6CIDRToRange()` for ergonomic subnet membership queries, making them the preferred approach over working with raw integers or binary strings.
