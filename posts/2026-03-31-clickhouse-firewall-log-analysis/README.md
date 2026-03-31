# How to Analyze Firewall Logs with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Firewall, Log Analysis, Security, Network, SIEM, Traffic Analysis

Description: Ingest and analyze firewall logs in ClickHouse to detect blocked traffic patterns, top talkers, port scans, and policy violations at scale.

---

Firewall logs record every allowed and denied network flow, making them invaluable for security investigations and capacity planning. ClickHouse can ingest millions of firewall events per second and answer analytical queries in milliseconds, making it a practical log analysis backend without the complexity of a full SIEM.

## Firewall Logs Table

```sql
CREATE TABLE firewall_logs
(
    event_time  DateTime,
    action      LowCardinality(String),  -- 'allow' | 'deny' | 'drop'
    protocol    LowCardinality(String),  -- 'TCP' | 'UDP' | 'ICMP'
    src_ip      IPv4,
    src_port    UInt16,
    dst_ip      IPv4,
    dst_port    UInt16,
    bytes       UInt32,
    packets     UInt32,
    rule_name   LowCardinality(String),
    interface   LowCardinality(String),
    direction   LowCardinality(String)   -- 'inbound' | 'outbound'
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (action, event_time, src_ip)
TTL event_time + INTERVAL 90 DAY;
```

## Ingest Logs from Syslog (CSV format)

```sql
INSERT INTO firewall_logs
SELECT
    parseDateTimeBestEffort(c1),
    c2, c3,
    toIPv4(c4), toUInt16(c5),
    toIPv4(c6), toUInt16(c7),
    toUInt32(c8), toUInt32(c9),
    c10, c11, c12
FROM file('/var/log/firewall/fw_*.csv', 'CSV',
    'c1 String, c2 String, c3 String, c4 String, c5 String,
     c6 String, c7 String, c8 String, c9 String, c10 String,
     c11 String, c12 String');
```

## Top Blocked Source IPs (Last 24 Hours)

```sql
SELECT
    src_ip,
    count()                                AS blocked_attempts,
    uniq(dst_ip)                           AS distinct_targets,
    uniq(dst_port)                         AS distinct_ports,
    round(sum(bytes) / 1048576, 2)         AS total_mb
FROM firewall_logs
WHERE action = 'deny'
  AND event_time >= now() - INTERVAL 24 HOUR
GROUP BY src_ip
ORDER BY blocked_attempts DESC
LIMIT 20;
```

## Port Scan Detection: Single Source Hitting Many Ports

```sql
SELECT
    src_ip,
    count()              AS connection_attempts,
    uniq(dst_port)       AS unique_ports_targeted,
    min(event_time)      AS first_seen,
    max(event_time)      AS last_seen
FROM firewall_logs
WHERE event_time >= now() - INTERVAL 1 HOUR
GROUP BY src_ip
HAVING unique_ports_targeted > 100
ORDER BY unique_ports_targeted DESC;
```

## Most Denied Destination Ports

```sql
SELECT
    dst_port,
    dictGetOrDefault('port_names', 'name', dst_port, 'unknown') AS service,
    count()     AS deny_count,
    uniq(src_ip) AS unique_sources
FROM firewall_logs
WHERE action = 'deny'
  AND event_time >= now() - INTERVAL 7 DAY
GROUP BY dst_port
ORDER BY deny_count DESC
LIMIT 15;
```

## Traffic Volume by Rule (Last 7 Days)

```sql
SELECT
    rule_name,
    action,
    count()                          AS flow_count,
    round(sum(bytes) / 1073741824, 2) AS total_gb
FROM firewall_logs
WHERE event_time >= now() - INTERVAL 7 DAY
GROUP BY rule_name, action
ORDER BY total_gb DESC;
```

## Hourly Deny Rate Trend

```sql
SELECT
    toStartOfHour(event_time)                              AS hour,
    countIf(action = 'deny')                               AS denies,
    countIf(action = 'allow')                              AS allows,
    round(countIf(action = 'deny') * 100.0 / count(), 2)  AS deny_pct
FROM firewall_logs
WHERE event_time >= now() - INTERVAL 7 DAY
GROUP BY hour
ORDER BY hour;
```

## Summary

ClickHouse is an excellent backend for firewall log analysis. Its columnar storage compresses repetitive network fields efficiently, and its vectorized query engine evaluates IP range conditions and `GROUP BY` aggregations quickly. By partitioning on month and applying a TTL, you can retain 90 days of firewall logs without manual cleanup.
