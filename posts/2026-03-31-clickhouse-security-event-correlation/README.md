# How to Build Security Event Correlation Rules in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Security, Correlation, SIEM, Threat Detection

Description: Build SQL-based security event correlation rules in ClickHouse to detect multi-step attack patterns across different log sources.

---

Security event correlation identifies attack patterns that span multiple systems. Unlike simple threshold alerts, correlation rules link events across time and source to detect sophisticated threats like reconnaissance followed by exploitation.

## Correlation Rule Definitions

```sql
CREATE TABLE correlation_rules
(
    rule_id UUID DEFAULT generateUUIDv4(),
    rule_name String,
    description String,
    severity LowCardinality(String),
    query String,
    time_window_minutes UInt32,
    is_active UInt8 DEFAULT 1,
    created_at DateTime DEFAULT now()
)
ENGINE = MergeTree()
ORDER BY (severity, rule_id);
```

## Correlation Alerts Table

```sql
CREATE TABLE correlation_alerts
(
    alert_id UUID DEFAULT generateUUIDv4(),
    rule_name String,
    severity LowCardinality(String),
    entity_type LowCardinality(String),
    entity_value String,
    evidence String,
    triggered_at DateTime DEFAULT now(),
    acknowledged_at Nullable(DateTime) DEFAULT NULL
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(triggered_at)
ORDER BY (severity, triggered_at);
```

## Rule 1 - Reconnaissance then Exploitation

Detect port scanning followed by auth failures from the same IP.

```sql
WITH scanners AS (
    SELECT source_ip
    FROM firewall_logs
    WHERE log_time >= now() - INTERVAL 1 HOUR
      AND action = 'deny'
    GROUP BY source_ip
    HAVING countDistinct(dest_port) > 20
),
attackers AS (
    SELECT source_ip
    FROM auth_events
    WHERE event_time >= now() - INTERVAL 1 HOUR
      AND outcome = 'failure'
    GROUP BY source_ip
    HAVING count() > 5
)
SELECT
    s.source_ip AS ip,
    'recon_then_bruteforce' AS pattern
FROM scanners s
JOIN attackers a ON s.source_ip = a.source_ip;
```

## Rule 2 - Account Compromise Pattern

Auth failures then success then data export from same account.

```sql
WITH failed AS (
    SELECT user_name, max(event_time) AS last_failure
    FROM auth_events
    WHERE event_time >= now() - INTERVAL 1 HOUR
      AND outcome = 'failure'
    GROUP BY user_name
    HAVING count() >= 5
),
succeeded AS (
    SELECT user_name, min(event_time) AS success_time
    FROM auth_events
    WHERE event_time >= now() - INTERVAL 1 HOUR
      AND outcome = 'success'
    GROUP BY user_name
),
exported AS (
    SELECT user_name, sum(record_count) AS records
    FROM data_access_logs
    WHERE accessed_at >= now() - INTERVAL 1 HOUR
      AND action = 'export'
    GROUP BY user_name
)
SELECT
    f.user_name,
    'compromise_then_exfil' AS pattern,
    e.records AS exported_records
FROM failed f
JOIN succeeded s ON f.user_name = s.user_name AND s.success_time > f.last_failure
JOIN exported e ON f.user_name = e.user_name;
```

## Rule 3 - Impossible Travel

Same account logs in from two geographically distant IPs within 30 minutes.

```sql
SELECT
    a.user_name,
    a.source_ip AS ip1,
    b.source_ip AS ip2,
    a.event_time AS time1,
    b.event_time AS time2,
    abs(dateDiff('minute', a.event_time, b.event_time)) AS minutes_apart
FROM auth_events a
JOIN auth_events b
    ON a.user_name = b.user_name
    AND a.source_ip != b.source_ip
    AND b.event_time > a.event_time
    AND dateDiff('minute', a.event_time, b.event_time) < 30
WHERE a.outcome = 'success'
  AND b.outcome = 'success'
  AND a.event_time >= now() - INTERVAL 1 HOUR
ORDER BY minutes_apart ASC;
```

## Rule 4 - Beaconing Detection

Regular DNS or HTTP intervals to the same external host (C2 traffic).

```sql
SELECT
    client_ip,
    query_name,
    count() AS queries,
    round(stddevPop(toUnixTimestamp(queried_at)), 0) AS interval_std_dev
FROM dns_queries
WHERE queried_at >= now() - INTERVAL 1 HOUR
GROUP BY client_ip, query_name
HAVING queries >= 20 AND interval_std_dev < 30
ORDER BY interval_std_dev ASC
LIMIT 20;
```

## Alert Backlog Review

```sql
SELECT
    rule_name,
    severity,
    count() AS alert_count,
    countIf(acknowledged_at IS NULL) AS unacknowledged,
    min(triggered_at) AS oldest_alert
FROM correlation_alerts
WHERE triggered_at >= now() - INTERVAL 24 HOUR
GROUP BY rule_name, severity
ORDER BY severity DESC, unacknowledged DESC;
```

## Summary

ClickHouse SQL enables powerful multi-step attack correlation by joining event tables across time windows. Patterns like recon-to-brute-force, account compromise with exfiltration, impossible travel, and C2 beaconing can all be expressed as queries and run on a schedule. The result is a flexible, code-defined SIEM correlation engine without vendor lock-in.
