# How to Use IPv4CIDRToRange() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, IP Function, Analytics, Network, Security

Description: Learn how IPv4CIDRToRange() returns the first and last IP of an IPv4 CIDR block in ClickHouse, enabling subnet membership checks, network segmentation, and geo-IP lookups.

---

`IPv4CIDRToRange(ip, prefix_length)` takes an IPv4 address (as a `UInt32` or `IPv4` value) and a prefix length (0-32) and returns a `Tuple(IPv4, IPv4)` containing the network address (first IP) and the broadcast address (last IP) of the corresponding CIDR block. This makes it straightforward to perform subnet membership tests and to build CIDR-based lookup tables.

## Basic Usage

```sql
-- Get the range for common CIDR blocks
SELECT
    ip_str,
    prefix,
    IPv4CIDRToRange(toIPv4(ip_str), prefix).1 AS network_addr,
    IPv4CIDRToRange(toIPv4(ip_str), prefix).2 AS broadcast_addr
FROM (
    SELECT * FROM (
        SELECT arrayJoin(['192.168.1.100','10.0.0.50','172.16.5.200','8.8.8.8']) AS ip_str,
               arrayJoin([24, 8, 12, 32]) AS prefix
    )
    WHERE (ip_str = '192.168.1.100' AND prefix = 24)
       OR (ip_str = '10.0.0.50'     AND prefix = 8)
       OR (ip_str = '172.16.5.200'  AND prefix = 12)
       OR (ip_str = '8.8.8.8'       AND prefix = 32)
);
```

```text
ip_str          prefix  network_addr   broadcast_addr
192.168.1.100   24      192.168.1.0    192.168.1.255
10.0.0.50       8       10.0.0.0       10.255.255.255
172.16.5.200    12      172.16.0.0     172.31.255.255
8.8.8.8         32      8.8.8.8        8.8.8.8
```

## Subnet Membership Test

```sql
-- Check whether each client IP falls within the 192.168.1.0/24 subnet
SELECT
    client_ip,
    toIPv4(client_ip)                                                         AS ip_val,
    IPv4CIDRToRange(toIPv4('192.168.1.0'), 24).1                              AS net_start,
    IPv4CIDRToRange(toIPv4('192.168.1.0'), 24).2                              AS net_end,
    ip_val BETWEEN toIPv4('192.168.1.0') AND toIPv4('192.168.1.255')          AS in_subnet
FROM access_logs
WHERE toDate(ts) = yesterday()
LIMIT 30;
```

## Classifying Traffic by Network Segment

```sql
-- Tag each request with the network segment it came from
SELECT
    client_ip,
    CASE
        WHEN toIPv4(client_ip) BETWEEN
             IPv4CIDRToRange(toIPv4('10.0.0.0'), 8).1 AND
             IPv4CIDRToRange(toIPv4('10.0.0.0'), 8).2
            THEN 'RFC1918-10/8'
        WHEN toIPv4(client_ip) BETWEEN
             IPv4CIDRToRange(toIPv4('172.16.0.0'), 12).1 AND
             IPv4CIDRToRange(toIPv4('172.16.0.0'), 12).2
            THEN 'RFC1918-172.16/12'
        WHEN toIPv4(client_ip) BETWEEN
             IPv4CIDRToRange(toIPv4('192.168.0.0'), 16).1 AND
             IPv4CIDRToRange(toIPv4('192.168.0.0'), 16).2
            THEN 'RFC1918-192.168/16'
        ELSE 'Public'
    END AS segment,
    count() AS requests
FROM access_logs
WHERE toDate(ts) = yesterday()
GROUP BY client_ip, segment
ORDER BY requests DESC
LIMIT 30;
```

## Building a CIDR Lookup Table

```sql
-- A reference table mapping CIDR blocks to owner names
CREATE TABLE cidr_owners
(
    cidr_network IPv4,
    prefix_len   UInt8,
    owner_name   String,
    region       String
)
ENGINE = MergeTree()
ORDER BY (cidr_network, prefix_len);

INSERT INTO cidr_owners VALUES
    (toIPv4('203.0.113.0'), 24, 'ExampleCorp-DMZ',  'us-east'),
    (toIPv4('198.51.100.0'), 24, 'PartnerNet',       'eu-west'),
    (toIPv4('192.0.2.0'),   24, 'TestRange',         'global');
```

```sql
-- Match incoming IPs against the CIDR owner table
SELECT
    al.client_ip,
    co.owner_name,
    co.region,
    count() AS requests
FROM access_logs al
JOIN cidr_owners co
  ON toIPv4(al.client_ip) BETWEEN
       IPv4CIDRToRange(co.cidr_network, co.prefix_len).1 AND
       IPv4CIDRToRange(co.cidr_network, co.prefix_len).2
WHERE toDate(al.ts) = yesterday()
GROUP BY al.client_ip, co.owner_name, co.region
ORDER BY requests DESC
LIMIT 30;
```

## Generating All Host IPs in a /24 Subnet

```sql
-- Generate the full host list for 192.168.1.0/24
SELECT
    IPv4NumToString(
        IPv4StringToNum(toString(IPv4CIDRToRange(toIPv4('192.168.1.0'), 24).1)) + number
    ) AS host_ip
FROM numbers(256)
WHERE host_ip != '192.168.1.0'       -- skip network address
  AND host_ip != '192.168.1.255';    -- skip broadcast address
```

## Subnet Size Calculation

```sql
-- Calculate the number of usable host addresses for various prefix lengths
SELECT
    prefix_len,
    IPv4CIDRToRange(toIPv4('10.0.0.0'), prefix_len).1 AS first_ip,
    IPv4CIDRToRange(toIPv4('10.0.0.0'), prefix_len).2 AS last_ip,
    IPv4StringToNum(toString(IPv4CIDRToRange(toIPv4('10.0.0.0'), prefix_len).2))
    - IPv4StringToNum(toString(IPv4CIDRToRange(toIPv4('10.0.0.0'), prefix_len).1))
    + 1                                               AS total_addresses
FROM (
    SELECT arrayJoin([8, 16, 24, 28, 30, 32]) AS prefix_len
);
```

```text
prefix_len  first_ip     last_ip          total_addresses
8           10.0.0.0     10.255.255.255   16777216
16          10.0.0.0     10.0.255.255     65536
24          10.0.0.0     10.0.0.255       256
28          10.0.0.0     10.0.0.15        16
30          10.0.0.0     10.0.0.3         4
32          10.0.0.0     10.0.0.0         1
```

## Summary

`IPv4CIDRToRange()` returns a `(first, last)` tuple for any CIDR block, making subnet membership tests as simple as a `BETWEEN` predicate on `IPv4` values. Use it for classifying traffic by network segment, performing CIDR-based geo or ownership lookups, generating host lists from a subnet, or building firewall rule audit queries. Combine it with `toIPv4()` for clean, type-safe comparisons, and store reference CIDR data in a lookup table for JOIN-based classification.
