# How to Use IPv6NumToString() and IPv6StringToNum() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, IP Function, Analytics, Network, Security

Description: Learn how IPv6NumToString() converts a 16-byte binary IPv6 address to a colon-hex string and IPv6StringToNum() does the reverse in ClickHouse for IPv6 log analysis.

---

IPv6 addresses are 128-bit values. ClickHouse represents them as a 16-byte `FixedString(16)` internally. `IPv6NumToString(bin)` converts this binary representation to a standard colon-separated hexadecimal string in compressed notation (for example, `2001:db8::1`). `IPv6StringToNum(str)` converts a colon-hex string back to the 16-byte binary form. Both functions also handle IPv4-mapped IPv6 addresses (e.g., `::ffff:192.168.1.1`).

## Basic Usage

```sql
-- Convert IPv6 strings to binary and back
SELECT
    ip_str,
    IPv6StringToNum(ip_str)              AS binary_repr,
    IPv6NumToString(IPv6StringToNum(ip_str)) AS round_trip
FROM (
    SELECT arrayJoin([
        '2001:0db8:85a3:0000:0000:8a2e:0370:7334',
        '2001:db8::1',
        '::1',
        '::ffff:192.168.1.1',
        'fe80::1'
    ]) AS ip_str
);
```

```text
ip_str                                   round_trip
2001:0db8:85a3:0000:0000:8a2e:0370:7334  2001:db8:85a3::8a2e:370:7334
2001:db8::1                              2001:db8::1
::1                                      ::1
::ffff:192.168.1.1                       ::ffff:192.168.1.1
fe80::1                                  fe80::1
```

Note that `IPv6NumToString()` always returns the compressed canonical form, normalising addresses like `2001:0db8:85a3:0000:0000:8a2e:0370:7334` to `2001:db8:85a3::8a2e:370:7334`.

## Normalising IPv6 Addresses in Logs

```sql
-- Normalise raw IPv6 strings to their canonical compressed form
SELECT
    raw_ip,
    IPv6NumToString(IPv6StringToNum(raw_ip)) AS canonical_ip,
    count()                                  AS requests
FROM ipv6_access_logs
WHERE toDate(ts) = yesterday()
GROUP BY raw_ip, canonical_ip
ORDER BY requests DESC
LIMIT 20;
```

## Detecting IPv4-Mapped IPv6 Addresses

```sql
-- Find IPv4-mapped IPv6 addresses (::ffff:x.x.x.x) and extract the IPv4 part
SELECT
    IPv6NumToString(IPv6StringToNum(client_ip)) AS canonical,
    -- Check if it is an IPv4-mapped address
    startsWith(canonical, '::ffff:')            AS is_ipv4_mapped,
    -- Extract the embedded IPv4 address
    if(
        startsWith(canonical, '::ffff:'),
        substr(canonical, 8),
        ''
    )                                           AS embedded_ipv4,
    count()                                     AS requests
FROM ipv6_access_logs
WHERE toDate(ts) = yesterday()
GROUP BY canonical, is_ipv4_mapped, embedded_ipv4
ORDER BY requests DESC
LIMIT 30;
```

## Checking Loopback and Link-Local Addresses

```sql
-- Categorise IPv6 traffic by address type
SELECT
    client_ip,
    IPv6NumToString(IPv6StringToNum(client_ip)) AS canonical,
    CASE
        WHEN canonical = '::1'
            THEN 'loopback'
        WHEN startsWith(canonical, 'fe80:')
            THEN 'link-local'
        WHEN startsWith(canonical, 'fc') OR startsWith(canonical, 'fd')
            THEN 'unique-local'
        WHEN startsWith(canonical, '2001:db8:')
            THEN 'documentation'
        ELSE 'global-unicast'
    END AS address_type,
    count() AS requests
FROM ipv6_access_logs
WHERE toDate(ts) = yesterday()
GROUP BY client_ip, canonical, address_type
ORDER BY requests DESC
LIMIT 30;
```

## Subnet Range Query Using Binary Comparison

```sql
-- Find all requests from the 2001:db8::/32 prefix
SELECT
    IPv6NumToString(client_ip_bin) AS client_ip,
    count()                        AS requests
FROM ipv6_logs_bin
WHERE toDate(ts) = yesterday()
  AND client_ip_bin >= IPv6StringToNum('2001:0db8::')
  AND client_ip_bin <= IPv6StringToNum('2001:0db8:ffff:ffff:ffff:ffff:ffff:ffff')
GROUP BY client_ip
ORDER BY requests DESC
LIMIT 20;
```

## Storing IPv6 Addresses Efficiently

```sql
-- Table using FixedString(16) for IPv6 binary storage
CREATE TABLE ipv6_access_logs_bin
(
    ts              DateTime,
    client_ip_bin   FixedString(16),   -- raw 16-byte binary IPv6
    url             String,
    status_code     UInt16
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(ts)
ORDER BY (ts, client_ip_bin);

-- Insert with conversion
INSERT INTO ipv6_access_logs_bin
SELECT
    ts,
    IPv6StringToNum(client_ip) AS client_ip_bin,
    url,
    status_code
FROM raw_ipv6_logs
WHERE toDate(ts) = yesterday();
```

## Top IPv6 Clients With Readable Display

```sql
-- Top 20 IPv6 clients by request count
SELECT
    IPv6NumToString(client_ip_bin) AS client_ip,
    count()                        AS requests,
    countIf(status_code >= 400)    AS errors
FROM ipv6_access_logs_bin
WHERE toDate(ts) = yesterday()
GROUP BY client_ip_bin
ORDER BY requests DESC
LIMIT 20;
```

## Summary

`IPv6NumToString()` and `IPv6StringToNum()` handle the conversion between the 16-byte binary form and human-readable colon-hex notation. A key benefit of `IPv6NumToString()` is that it always outputs canonical compressed form, so it normalises inconsistently formatted input strings. Store IPv6 addresses as `FixedString(16)` using `IPv6StringToNum()` at insert time for compact storage and fast binary range comparisons. Use `IPv6NumToString()` at query time to display results. For a more ergonomic API, consider using the native `IPv6` column type with `toIPv6()`.
