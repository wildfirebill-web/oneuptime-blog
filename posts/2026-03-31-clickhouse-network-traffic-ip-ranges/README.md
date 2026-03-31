# How to Analyze Network Traffic by IP Ranges in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Network, Analytics, Security, IP Address

Description: Learn how to segment, aggregate, and analyze network traffic by IP ranges and CIDR subnets in ClickHouse using IP functions and efficient schema design patterns.

---

Analyzing network traffic by IP ranges is a common requirement in security operations, capacity planning, and infrastructure monitoring. ClickHouse's native `IPv4` and `IPv6` column types, combined with functions like `isIPAddressInRange()`, `IPv4CIDRToRange()`, and bitwise operations, make subnet-level aggregations fast and expressive. This guide covers practical patterns for segmenting traffic by CIDR block, building per-subnet dashboards, and detecting anomalies.

## Schema Design for Network Log Analysis

Storing IPs as native types (`IPv4`, `IPv6`) instead of strings reduces storage and speeds up range comparisons.

```sql
CREATE TABLE network_flows (
    flow_time       DateTime,
    src_ip          IPv4,
    dst_ip          IPv4,
    src_port        UInt16,
    dst_port        UInt16,
    protocol        LowCardinality(String),
    bytes_sent      UInt64,
    bytes_received  UInt64,
    packets         UInt32,
    direction       LowCardinality(String)  -- 'inbound' | 'outbound' | 'lateral'
) ENGINE = MergeTree
PARTITION BY toYYYYMM(flow_time)
ORDER BY (flow_time, src_ip, dst_ip);
```

## Computing the /24 Subnet for Each IP

`IPv4CIDRToRange()` returns the start and end of a CIDR block. To group by /24 subnet, mask the last octet.

```sql
-- Group traffic by /24 source subnet
SELECT
    IPv4NumToString(bitAnd(toUInt32(src_ip), 0xFFFFFF00)) AS src_subnet_24,
    count()         AS flow_count,
    sum(bytes_sent) AS total_bytes,
    uniq(src_ip)    AS unique_src_ips
FROM network_flows
WHERE flow_time >= now() - INTERVAL 1 HOUR
GROUP BY src_subnet_24
ORDER BY total_bytes DESC
LIMIT 20;
```

## Using IPv4CIDRToRange() to Define Subnets

```sql
-- Show the address range covered by common CIDR blocks
SELECT
    '10.0.0.0/8'     AS cidr,
    IPv4NumToString((IPv4CIDRToRange(toIPv4('10.0.0.0'), 8)).1)  AS range_start,
    IPv4NumToString((IPv4CIDRToRange(toIPv4('10.0.0.0'), 8)).2)  AS range_end

UNION ALL

SELECT
    '172.16.0.0/12',
    IPv4NumToString((IPv4CIDRToRange(toIPv4('172.16.0.0'), 12)).1),
    IPv4NumToString((IPv4CIDRToRange(toIPv4('172.16.0.0'), 12)).2)

UNION ALL

SELECT
    '192.168.0.0/16',
    IPv4NumToString((IPv4CIDRToRange(toIPv4('192.168.0.0'), 16)).1),
    IPv4NumToString((IPv4CIDRToRange(toIPv4('192.168.0.0'), 16)).2);
```

```text
cidr              range_start   range_end
10.0.0.0/8        10.0.0.0      10.255.255.255
172.16.0.0/12     172.16.0.0    172.31.255.255
192.168.0.0/16    192.168.0.0   192.168.255.255
```

## Traffic Volume per Named Subnet

```sql
-- Compare inbound traffic volume across named subnets
SELECT
    CASE
        WHEN isIPAddressInRange(toString(src_ip), '10.1.0.0/16')  THEN 'prod-vpc'
        WHEN isIPAddressInRange(toString(src_ip), '10.2.0.0/16')  THEN 'staging-vpc'
        WHEN isIPAddressInRange(toString(src_ip), '10.3.0.0/16')  THEN 'dev-vpc'
        WHEN isIPAddressInRange(toString(src_ip), '10.0.0.0/8')   THEN 'other-internal'
        ELSE 'external'
    END AS src_zone,
    count()              AS flows,
    sum(bytes_sent)      AS bytes_out,
    sum(bytes_received)  AS bytes_in,
    uniq(src_ip)         AS unique_sources
FROM network_flows
WHERE flow_time >= today()
GROUP BY src_zone
ORDER BY bytes_out DESC;
```

## Top Talkers per Subnet

```sql
-- Top 5 source IPs per /16 subnet, ranked by bytes sent
SELECT
    src_subnet_16,
    src_ip,
    bytes_rank,
    total_bytes
FROM (
    SELECT
        IPv4NumToString(bitAnd(toUInt32(src_ip), 0xFFFF0000)) AS src_subnet_16,
        toString(src_ip)   AS src_ip,
        sum(bytes_sent)    AS total_bytes,
        row_number() OVER (
            PARTITION BY src_subnet_16
            ORDER BY sum(bytes_sent) DESC
        ) AS bytes_rank
    FROM network_flows
    WHERE flow_time >= now() - INTERVAL 24 HOUR
    GROUP BY src_subnet_16, src_ip
)
WHERE bytes_rank <= 5
ORDER BY src_subnet_16, bytes_rank;
```

## Detecting Port Scans from a Subnet

```sql
-- Source /24 subnets contacting an unusually large number of unique destination ports
SELECT
    IPv4NumToString(bitAnd(toUInt32(src_ip), 0xFFFFFF00)) AS src_subnet,
    uniq(dst_port)   AS unique_dst_ports,
    uniq(dst_ip)     AS unique_dst_ips,
    count()          AS total_flows
FROM network_flows
WHERE
    flow_time >= now() - INTERVAL 10 MINUTE
    AND direction = 'outbound'
GROUP BY src_subnet
HAVING unique_dst_ports > 100
ORDER BY unique_dst_ports DESC;
```

## Lateral Movement Detection

Lateral movement often appears as unusual internal-to-internal traffic.

```sql
-- Internal source IPs communicating with many unique internal destinations
SELECT
    toString(src_ip)  AS source,
    uniq(dst_ip)      AS unique_internal_targets,
    count()           AS flow_count
FROM network_flows
WHERE
    flow_time >= now() - INTERVAL 1 HOUR
    AND isIPAddressInRange(toString(src_ip), '10.0.0.0/8')
    AND isIPAddressInRange(toString(dst_ip), '10.0.0.0/8')
GROUP BY source
HAVING unique_internal_targets > 20
ORDER BY unique_internal_targets DESC;
```

## Bandwidth Trending per Subnet by Hour

```sql
-- Hourly bandwidth usage per /16 subnet over the past 7 days
SELECT
    toStartOfHour(flow_time)                                       AS hour,
    IPv4NumToString(bitAnd(toUInt32(src_ip), 0xFFFF0000))          AS subnet_16,
    sum(bytes_sent + bytes_received)                               AS total_bytes,
    round(sum(bytes_sent + bytes_received) / 1e9, 2)              AS total_gb
FROM network_flows
WHERE flow_time >= now() - INTERVAL 7 DAY
GROUP BY hour, subnet_16
ORDER BY hour DESC, total_bytes DESC;
```

## Building a Subnet Summary Dictionary

For repeated subnet lookups, a dictionary avoids per-row CASE expressions.

```sql
CREATE TABLE subnet_registry (
    cidr        String,
    name        String,
    team        String,
    environment LowCardinality(String)
) ENGINE = MergeTree ORDER BY cidr;

INSERT INTO subnet_registry VALUES
    ('10.1.0.0/16',  'prod-east',    'platform',  'production'),
    ('10.2.0.0/16',  'prod-west',    'platform',  'production'),
    ('10.10.0.0/16', 'staging',      'platform',  'staging'),
    ('10.20.0.0/14', 'dev',          'engineering','development');

-- Join flows with subnet registry
SELECT
    r.name        AS subnet_name,
    r.environment AS env,
    count()       AS flows,
    sum(f.bytes_sent) AS bytes
FROM network_flows f
JOIN subnet_registry r
    ON isIPAddressInRange(toString(f.src_ip), r.cidr)
WHERE f.flow_time >= today()
GROUP BY subnet_name, env
ORDER BY bytes DESC;
```

## Summary

Analyzing network traffic by IP ranges in ClickHouse combines the native `IPv4`/`IPv6` column types, bitwise masking for subnet grouping, and `isIPAddressInRange()` for CIDR membership tests. Store IPs as native types to reduce storage and enable fast integer comparisons. Use bitwise AND with subnet masks for /8, /16, and /24 groupings, and reserve `isIPAddressInRange()` for named-CIDR zone tagging. For repeated zone lookups, a subnet registry table joined via `isIPAddressInRange()` in the ON clause keeps queries clean and easy to maintain.
