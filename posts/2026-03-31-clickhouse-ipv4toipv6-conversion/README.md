# How to Use IPv4ToIPv6() for IP Address Conversion in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, IP Function, Analytics, Network, Type Conversion

Description: Learn how IPv4ToIPv6() maps an IPv4 address to its IPv4-mapped IPv6 representation in ClickHouse, simplifying dual-stack log analysis and unified IP address storage.

---

`IPv4ToIPv6(ipv4)` converts an `IPv4` (or `UInt32`) value into its IPv4-mapped IPv6 representation - the standard `::ffff:x.x.x.x` notation defined in RFC 4291. The result is an `IPv6` value. This function is essential when you run a dual-stack infrastructure and want to store all IP addresses in a single `IPv6` column, or when you need to compare IPv4 and IPv6 addresses in the same query.

## Basic Usage

```sql
-- Convert IPv4 addresses to their IPv4-mapped IPv6 form
SELECT
    ip_str,
    toIPv4(ip_str)             AS ipv4_native,
    IPv4ToIPv6(toIPv4(ip_str)) AS ipv6_mapped
FROM (
    SELECT arrayJoin([
        '192.168.1.1',
        '10.0.0.1',
        '8.8.8.8',
        '0.0.0.0',
        '255.255.255.255'
    ]) AS ip_str
);
```

```text
ip_str           ipv4_native      ipv6_mapped
192.168.1.1      192.168.1.1      ::ffff:192.168.1.1
10.0.0.1         10.0.0.1         ::ffff:10.0.0.1
8.8.8.8          8.8.8.8          ::ffff:8.8.8.8
0.0.0.0          0.0.0.0          ::ffff:0.0.0.0
255.255.255.255  255.255.255.255  ::ffff:255.255.255.255
```

## Unified Dual-Stack Log Table

```sql
-- Store both IPv4 and IPv6 clients in a single IPv6 column
CREATE TABLE dual_stack_access_logs
(
    ts          DateTime,
    client_ip   IPv6,   -- IPv4 clients stored as ::ffff:x.x.x.x
    url         String,
    status_code UInt16,
    bytes_sent  UInt64
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(ts)
ORDER BY (ts, client_ip);

-- Insert IPv4 clients via IPv4ToIPv6() conversion
INSERT INTO dual_stack_access_logs
SELECT
    ts,
    IPv4ToIPv6(toIPv4(client_ip)) AS client_ip,
    url,
    status_code,
    bytes_sent
FROM ipv4_access_logs
WHERE toDate(ts) = yesterday();

-- Insert native IPv6 clients as-is
INSERT INTO dual_stack_access_logs
SELECT
    ts,
    toIPv6(client_ip) AS client_ip,
    url,
    status_code,
    bytes_sent
FROM ipv6_access_logs
WHERE toDate(ts) = yesterday();
```

## Identifying IPv4-Mapped Entries in a Unified Column

```sql
-- Detect which rows in the unified table were originally IPv4
SELECT
    client_ip,
    startsWith(IPv6NumToString(client_ip), '::ffff:') AS is_ipv4_mapped,
    -- Extract the embedded IPv4 address
    if(
        startsWith(IPv6NumToString(client_ip), '::ffff:'),
        toIPv4(substr(IPv6NumToString(client_ip), 8)),
        toIPv4('0.0.0.0')
    ) AS embedded_ipv4,
    count() AS requests
FROM dual_stack_access_logs
WHERE toDate(ts) = yesterday()
GROUP BY client_ip, is_ipv4_mapped, embedded_ipv4
ORDER BY requests DESC
LIMIT 30;
```

## Protocol Distribution in a Unified Table

```sql
-- Split traffic by IP version
SELECT
    startsWith(IPv6NumToString(client_ip), '::ffff:') AS is_ipv4,
    count()          AS requests,
    uniq(client_ip)  AS unique_clients
FROM dual_stack_access_logs
WHERE toDate(ts) = yesterday()
GROUP BY is_ipv4
ORDER BY is_ipv4;
```

## Cross-Version Comparison: Same Logical Host

```sql
-- Check whether the same host appears in both IPv4 and IPv6 form
-- (e.g., a client that connects via both stacks)
SELECT
    extracted_ipv4,
    countIf(is_ipv4_mapped = 1) AS ipv4_requests,
    countIf(is_ipv4_mapped = 0) AS native_ipv6_requests
FROM (
    SELECT
        client_ip,
        startsWith(IPv6NumToString(client_ip), '::ffff:') AS is_ipv4_mapped,
        if(
            startsWith(IPv6NumToString(client_ip), '::ffff:'),
            IPv4ToIPv6(toIPv4(substr(IPv6NumToString(client_ip), 8))),
            client_ip
        ) AS canonical_ipv6,
        if(
            startsWith(IPv6NumToString(client_ip), '::ffff:'),
            substr(IPv6NumToString(client_ip), 8),
            ''
        ) AS extracted_ipv4
    FROM dual_stack_access_logs
    WHERE toDate(ts) = yesterday()
)
WHERE extracted_ipv4 != ''
GROUP BY extracted_ipv4
HAVING ipv4_requests > 0
ORDER BY ipv4_requests DESC
LIMIT 30;
```

## CIDR Range Query on a Unified Column

```sql
-- Find all requests from 192.168.1.0/24 in the unified table
-- (IPv4 clients are stored as ::ffff:192.168.1.x)
SELECT
    IPv6NumToString(client_ip) AS ip,
    count()                    AS requests
FROM dual_stack_access_logs
WHERE toDate(ts) = yesterday()
  AND client_ip BETWEEN
      IPv4ToIPv6(IPv4CIDRToRange(toIPv4('192.168.1.0'), 24).1) AND
      IPv4ToIPv6(IPv4CIDRToRange(toIPv4('192.168.1.0'), 24).2)
GROUP BY client_ip
ORDER BY requests DESC
LIMIT 20;
```

## Round-Trip Verification

```sql
-- Verify that converting IPv4 to IPv6 and extracting back gives the same IP
SELECT
    ip_str,
    IPv4ToIPv6(toIPv4(ip_str))                                    AS ipv6_mapped,
    toIPv4(substr(IPv6NumToString(IPv4ToIPv6(toIPv4(ip_str))), 8)) AS recovered_ipv4,
    ip_str = toString(recovered_ipv4)                              AS lossless
FROM (
    SELECT arrayJoin(['10.0.0.1','172.16.0.1','192.168.1.100','8.8.8.8']) AS ip_str
);
```

```text
ip_str         ipv6_mapped          recovered_ipv4  lossless
10.0.0.1       ::ffff:10.0.0.1      10.0.0.1        1
172.16.0.1     ::ffff:172.16.0.1    172.16.0.1      1
192.168.1.100  ::ffff:192.168.1.100  192.168.1.100  1
8.8.8.8        ::ffff:8.8.8.8       8.8.8.8         1
```

## Summary

`IPv4ToIPv6()` maps an IPv4 address to the standard `::ffff:x.x.x.x` IPv4-mapped IPv6 representation, making it the key function for building unified dual-stack IP columns. Use it when inserting IPv4-sourced rows into a table with an `IPv6` column so all addresses share a single type. To recover the original IPv4 address, check for the `::ffff:` prefix and extract the last four octets. Combine `IPv4ToIPv6()` with `IPv4CIDRToRange()` to perform CIDR subnet queries against the unified column without separating IPv4 and IPv6 traffic into different tables.
