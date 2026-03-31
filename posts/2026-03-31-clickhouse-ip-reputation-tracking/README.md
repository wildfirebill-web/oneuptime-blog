# How to Build IP Reputation Tracking with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, IP Reputation, Security, Threat Intelligence, Blocklist

Description: Build an IP reputation system in ClickHouse to score IPs based on behavior, track malicious actors, and feed blocklists dynamically.

---

IP reputation tracking aggregates behavioral signals across all your data sources to assign trust scores to IP addresses. ClickHouse handles the aggregation efficiently, enabling dynamic blocklist generation and reputation queries at scale.

## IP Reputation Table

```sql
CREATE TABLE ip_reputation
(
    ip IPv4,
    reputation_score Int16,
    category LowCardinality(String),
    reason String,
    first_seen DateTime,
    last_seen DateTime,
    total_events UInt64,
    is_blocked UInt8 DEFAULT 0,
    updated_at DateTime DEFAULT now()
)
ENGINE = ReplacingMergeTree(updated_at)
ORDER BY ip;
```

## IP Behavior Events Table

```sql
CREATE TABLE ip_events
(
    ip IPv4,
    event_type LowCardinality(String),
    outcome LowCardinality(String),
    target_service LowCardinality(String),
    event_time DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(event_time)
ORDER BY (ip, event_time)
TTL toDate(event_time) + INTERVAL 90 DAY;
```

## Compute Reputation Score from Behavior

```sql
SELECT
    ip,
    count() AS total_events,
    countIf(event_type = 'auth_failure') AS auth_failures,
    countIf(event_type = 'port_scan') AS port_scans,
    countIf(event_type = 'rate_limit_exceeded') AS rate_limits,
    countIf(outcome = 'blocked') AS blocked_events,
    greatest(
        0,
        100
        - countIf(event_type = 'auth_failure') * 2
        - countIf(event_type = 'port_scan') * 10
        - countIf(event_type = 'rate_limit_exceeded') * 3
        - countIf(outcome = 'blocked') * 5
    ) AS reputation_score
FROM ip_events
WHERE event_time >= now() - INTERVAL 24 HOUR
GROUP BY ip
ORDER BY reputation_score ASC
LIMIT 50;
```

## Top Worst Reputation IPs

```sql
SELECT
    ip,
    reputation_score,
    category,
    reason,
    last_seen,
    total_events
FROM ip_reputation FINAL
WHERE is_blocked = 0
ORDER BY reputation_score ASC
LIMIT 30;
```

## IPs Appearing in Multiple Attack Types

```sql
SELECT
    ip,
    groupArray(DISTINCT event_type) AS attack_types,
    count() AS total_events
FROM ip_events
WHERE event_time >= now() - INTERVAL 24 HOUR
GROUP BY ip
HAVING length(attack_types) >= 3
ORDER BY total_events DESC;
```

## Dynamic Blocklist Generation

```sql
SELECT DISTINCT ip
FROM (
    SELECT
        ip,
        countIf(event_type = 'auth_failure') AS af,
        countIf(event_type = 'port_scan') AS ps,
        count() AS total
    FROM ip_events
    WHERE event_time >= now() - INTERVAL 1 HOUR
    GROUP BY ip
    HAVING af > 50 OR ps > 5 OR total > 200
)
FORMAT TabSeparated;
```

## IP Activity Trend - 6-Hour Window

```sql
SELECT
    toStartOfHour(event_time) AS hour,
    ip,
    countIf(event_type = 'auth_failure') AS failures,
    countIf(event_type = 'rate_limit_exceeded') AS throttled
FROM ip_events
WHERE ip = '185.220.101.45'
  AND event_time >= now() - INTERVAL 6 HOUR
GROUP BY hour, ip
ORDER BY hour;
```

## Compare Against Known Threat Feed

```sql
CREATE TABLE threat_feed_ips
(
    ip IPv4,
    threat_type LowCardinality(String),
    confidence UInt8,
    added_at DateTime
)
ENGINE = ReplacingMergeTree(added_at)
ORDER BY ip;

SELECT
    e.ip,
    t.threat_type,
    t.confidence,
    count() AS local_events
FROM ip_events e
JOIN threat_feed_ips t ON e.ip = t.ip
WHERE e.event_time >= now() - INTERVAL 24 HOUR
GROUP BY e.ip, t.threat_type, t.confidence
ORDER BY local_events DESC;
```

## Summary

ClickHouse builds IP reputation systems efficiently using `ReplacingMergeTree` for deduplicated reputation records and fast aggregation queries for behavioral scoring. Combining local behavioral signals with external threat feeds gives a complete picture of each IP's trustworthiness. The dynamic blocklist export query can feed firewall rules or WAF configurations in near real time.
