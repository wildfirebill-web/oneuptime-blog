# How to Store and Analyze Firewall Logs in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Firewall, Log Analysis, Security, Network

Description: Store and analyze firewall logs in ClickHouse to detect blocked traffic patterns, identify top talkers, and monitor network security posture.

---

Firewall logs are high-volume data that traditional log management systems struggle with. ClickHouse's fast ingest and columnar compression make it an ideal store for firewall log analysis at scale.

## Firewall Log Table

```sql
CREATE TABLE firewall_logs
(
    log_id UUID DEFAULT generateUUIDv4(),
    action LowCardinality(String),
    protocol LowCardinality(String),
    source_ip IPv4,
    dest_ip IPv4,
    source_port UInt16,
    dest_port UInt16,
    bytes_sent UInt64 DEFAULT 0,
    bytes_received UInt64 DEFAULT 0,
    rule_name LowCardinality(String),
    interface LowCardinality(String),
    log_time DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(log_time)
ORDER BY (action, source_ip, log_time)
TTL toDate(log_time) + INTERVAL 180 DAY;
```

## Traffic Summary by Action

```sql
SELECT
    action,
    count() AS connections,
    sum(bytes_sent + bytes_received) AS total_bytes,
    countDistinct(source_ip) AS unique_sources
FROM firewall_logs
WHERE log_time >= now() - INTERVAL 1 HOUR
GROUP BY action
ORDER BY connections DESC;
```

## Top Blocked Source IPs

```sql
SELECT
    source_ip,
    count() AS blocked_attempts,
    groupArray(5)(DISTINCT CAST(dest_port AS String)) AS targeted_ports,
    countDistinct(dest_ip) AS distinct_targets
FROM firewall_logs
WHERE action = 'deny'
  AND log_time >= now() - INTERVAL 24 HOUR
GROUP BY source_ip
ORDER BY blocked_attempts DESC
LIMIT 20;
```

## Most Targeted Destination Ports

```sql
SELECT
    dest_port,
    protocol,
    count() AS connection_attempts,
    countIf(action = 'deny') AS denied,
    round(countIf(action = 'deny') * 100.0 / count(), 2) AS deny_pct
FROM firewall_logs
WHERE log_time >= now() - INTERVAL 24 HOUR
GROUP BY dest_port, protocol
ORDER BY connection_attempts DESC
LIMIT 20;
```

## Bandwidth Top Talkers

```sql
SELECT
    source_ip,
    dest_ip,
    sum(bytes_sent) AS bytes_out,
    sum(bytes_received) AS bytes_in,
    sum(bytes_sent + bytes_received) AS total_bytes,
    count() AS connections
FROM firewall_logs
WHERE action = 'allow'
  AND log_time >= now() - INTERVAL 1 HOUR
GROUP BY source_ip, dest_ip
ORDER BY total_bytes DESC
LIMIT 20;
```

## Port Scan Detection

```sql
SELECT
    source_ip,
    countDistinct(dest_port) AS ports_probed,
    count() AS total_attempts
FROM firewall_logs
WHERE log_time >= now() - INTERVAL 15 MINUTE
  AND action = 'deny'
GROUP BY source_ip
HAVING ports_probed > 20
ORDER BY ports_probed DESC;
```

## Denied Traffic Trend by Hour

```sql
SELECT
    toStartOfHour(log_time) AS hour,
    action,
    count() AS events,
    countDistinct(source_ip) AS unique_sources
FROM firewall_logs
WHERE log_time >= now() - INTERVAL 24 HOUR
GROUP BY hour, action
ORDER BY hour, action;
```

## Rule Effectiveness Report

```sql
SELECT
    rule_name,
    countIf(action = 'allow') AS allowed,
    countIf(action = 'deny') AS denied,
    round(countIf(action = 'deny') * 100.0 / count(), 2) AS deny_rate_pct,
    sum(bytes_sent + bytes_received) AS total_bytes
FROM firewall_logs
WHERE log_time >= now() - INTERVAL 7 DAY
GROUP BY rule_name
ORDER BY denied DESC
LIMIT 20;
```

## Summary

ClickHouse stores firewall logs efficiently with IPv4 column types, fast aggregation for traffic summaries, and TTL-based automatic cleanup. Queries for top blocked sources, port scan detection, bandwidth top talkers, and rule effectiveness give security teams comprehensive visibility into network security posture without the cost of commercial SIEM solutions.
