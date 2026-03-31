# How to Use IPv4NumToString() and IPv4StringToNum() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, IP Function, Analytics, Network, Security

Description: Learn how IPv4NumToString() converts a 32-bit integer to a dotted-decimal IPv4 string and IPv4StringToNum() does the reverse in ClickHouse for efficient IP address storage and analysis.

---

ClickHouse stores IPv4 addresses most efficiently as 32-bit unsigned integers (`UInt32`). `IPv4NumToString(num)` converts a `UInt32` value into a human-readable dotted-decimal string like `192.168.1.1`. `IPv4StringToNum(str)` converts a dotted-decimal string back into its `UInt32` representation. These two functions are the foundation for IP address manipulation in ClickHouse - used for storage optimisation, range queries, CIDR membership checks, and display formatting.

## Basic Usage

```sql
-- Convert between numeric and string representations
SELECT
    3232235777                             AS ip_num,
    IPv4NumToString(3232235777)            AS ip_string,
    IPv4StringToNum('192.168.1.1')         AS back_to_num,
    IPv4NumToString(IPv4StringToNum('10.0.0.1')) AS round_trip;
```

```text
ip_num       ip_string    back_to_num   round_trip
3232235777   192.168.1.1  167772161     10.0.0.1
```

## Converting a Column of IP Strings to Integers

```sql
-- Show numeric equivalents of IP strings (useful for range queries)
SELECT
    ip_string,
    IPv4StringToNum(ip_string) AS ip_num
FROM (
    SELECT arrayJoin([
        '0.0.0.0',
        '10.0.0.1',
        '172.16.0.1',
        '192.168.1.100',
        '255.255.255.255'
    ]) AS ip_string
);
```

```text
ip_string        ip_num
0.0.0.0          0
10.0.0.1         167772161
172.16.0.1       2886729729
192.168.1.100    3232235876
255.255.255.255  4294967295
```

## Storing IPs as UInt32 for Efficient Range Queries

```sql
-- Create a table that stores IPs numerically for fast CIDR lookups
CREATE TABLE access_logs_v2
(
    ts              DateTime,
    client_ip_num   UInt32,  -- stored as integer
    url             String,
    status_code     UInt16
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(ts)
ORDER BY (ts, client_ip_num);

-- Insert with conversion from string
INSERT INTO access_logs_v2
SELECT
    ts,
    IPv4StringToNum(client_ip) AS client_ip_num,
    url,
    status_code
FROM raw_access_logs
WHERE toDate(ts) = yesterday();
```

## CIDR Range Query Using Numeric Comparison

```sql
-- Find all requests from the 192.168.1.0/24 subnet
SELECT
    IPv4NumToString(client_ip_num) AS client_ip,
    count()                        AS requests
FROM access_logs_v2
WHERE toDate(ts) = yesterday()
  AND client_ip_num BETWEEN IPv4StringToNum('192.168.1.0')
                        AND IPv4StringToNum('192.168.1.255')
GROUP BY client_ip
ORDER BY requests DESC
LIMIT 20;
```

## Private IP Detection

```sql
-- Flag requests from RFC-1918 private address space
SELECT
    IPv4NumToString(client_ip_num) AS client_ip,
    count()                        AS requests,
    -- True if IP falls in any RFC-1918 range
    (
        client_ip_num BETWEEN IPv4StringToNum('10.0.0.0')    AND IPv4StringToNum('10.255.255.255')
        OR client_ip_num BETWEEN IPv4StringToNum('172.16.0.0') AND IPv4StringToNum('172.31.255.255')
        OR client_ip_num BETWEEN IPv4StringToNum('192.168.0.0') AND IPv4StringToNum('192.168.255.255')
    ) AS is_private
FROM access_logs_v2
WHERE toDate(ts) = yesterday()
GROUP BY client_ip, is_private
ORDER BY requests DESC
LIMIT 30;
```

## Top Client IPs With Display Formatting

```sql
-- Top 20 IPs by request volume, displayed in dotted-decimal
SELECT
    IPv4NumToString(client_ip_num) AS client_ip,
    count()                        AS requests,
    countIf(status_code >= 400)    AS error_requests,
    round(countIf(status_code >= 400) * 100.0 / count(), 2) AS error_rate_pct
FROM access_logs_v2
WHERE toDate(ts) = yesterday()
GROUP BY client_ip_num
ORDER BY requests DESC
LIMIT 20;
```

## Sorting IPs in Correct Numeric Order

```sql
-- IPs sorted numerically (not lexicographically) using IPv4StringToNum
SELECT
    client_ip,
    count() AS requests
FROM access_logs
WHERE toDate(ts) = yesterday()
GROUP BY client_ip
ORDER BY IPv4StringToNum(client_ip) ASC
LIMIT 30;
```

## Round-Trip Verification

```sql
-- Confirm that conversion is lossless
SELECT
    ip,
    IPv4NumToString(IPv4StringToNum(ip)) AS converted,
    ip = IPv4NumToString(IPv4StringToNum(ip)) AS lossless
FROM (
    SELECT arrayJoin(['0.0.0.0','127.0.0.1','192.168.0.1','255.255.255.255']) AS ip
);
```

```text
ip               converted        lossless
0.0.0.0          0.0.0.0          1
127.0.0.1        127.0.0.1        1
192.168.0.1      192.168.0.1      1
255.255.255.255  255.255.255.255  1
```

## Summary

`IPv4NumToString()` and `IPv4StringToNum()` provide the bridge between human-readable dotted-decimal notation and the compact `UInt32` representation that ClickHouse uses for efficient IP address storage and arithmetic. Store IPs as `UInt32` in production tables for better compression and faster range scans, and use these functions at the boundary when ingesting string-format IPs or displaying results to end users. For even cleaner semantics, use the dedicated `IPv4` column type together with `toIPv4()`.
