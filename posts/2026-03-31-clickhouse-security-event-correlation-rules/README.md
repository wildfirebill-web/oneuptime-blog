# How to Build Security Event Correlation Rules in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Security, SIEM, Event Correlation, Log Analysis

Description: Build powerful security event correlation rules in ClickHouse to detect multi-step attack patterns across log sources.

---

Security Information and Event Management (SIEM) systems traditionally rely on proprietary rule engines that are expensive and hard to scale. ClickHouse offers an alternative: store raw security events at scale and express correlation logic directly in SQL.

## Security Events Table

```sql
CREATE TABLE security_events (
    event_id        UUID DEFAULT generateUUIDv4(),
    timestamp       DateTime,
    source_ip       IPv4,
    dest_ip         IPv4,
    username        String,
    event_type      LowCardinality(String),
    severity        LowCardinality(String),
    host            String,
    raw_message     String,
    date            Date DEFAULT toDate(timestamp)
) ENGINE = MergeTree()
PARTITION BY date
ORDER BY (event_type, source_ip, timestamp);
```

## Brute Force Detection Rule

Detect accounts under brute-force attack by correlating failed login events within a time window:

```sql
SELECT
    source_ip,
    username,
    count() AS failed_attempts,
    min(timestamp) AS first_attempt,
    max(timestamp) AS last_attempt
FROM security_events
WHERE event_type = 'auth_failure'
  AND timestamp >= now() - INTERVAL 5 MINUTE
GROUP BY source_ip, username
HAVING failed_attempts >= 10
ORDER BY failed_attempts DESC;
```

## Lateral Movement Detection

Identify hosts that logged into an unusual number of distinct targets in a short window - a hallmark of lateral movement:

```sql
SELECT
    source_ip,
    uniq(dest_ip) AS unique_targets,
    count() AS total_events
FROM security_events
WHERE event_type IN ('auth_success', 'smb_connect', 'rdp_connect')
  AND timestamp >= now() - INTERVAL 30 MINUTE
GROUP BY source_ip
HAVING unique_targets >= 5
ORDER BY unique_targets DESC;
```

## Privilege Escalation Followed by Data Access

Correlate events across two stages using a self-join approach:

```sql
SELECT
    e1.username,
    e1.source_ip,
    e1.timestamp AS escalation_time,
    e2.timestamp AS data_access_time,
    e2.raw_message
FROM security_events AS e1
JOIN security_events AS e2
    ON e1.username = e2.username
    AND e2.timestamp BETWEEN e1.timestamp AND e1.timestamp + INTERVAL 10 MINUTE
WHERE e1.event_type = 'privilege_escalation'
  AND e2.event_type = 'sensitive_file_access'
  AND e1.date = today();
```

## Aggregating Alerts into Incident Summaries

Use `windowFunnel` to detect sequences of suspicious events from the same source:

```sql
SELECT
    source_ip,
    windowFunnel(3600)(
        timestamp,
        event_type = 'port_scan',
        event_type = 'auth_failure',
        event_type = 'auth_success'
    ) AS attack_stage
FROM security_events
WHERE date = today()
GROUP BY source_ip
HAVING attack_stage >= 2;
```

## Materialized View for Alert Aggregation

Pre-aggregate failed authentication counts to speed up dashboards:

```sql
CREATE MATERIALIZED VIEW auth_failure_summary
ENGINE = SummingMergeTree()
ORDER BY (minute, source_ip, username)
AS
SELECT
    toStartOfMinute(timestamp) AS minute,
    source_ip,
    username,
    count() AS failures
FROM security_events
WHERE event_type = 'auth_failure'
GROUP BY minute, source_ip, username;
```

## Summary

ClickHouse enables SIEM-style correlation logic through SQL, with the performance to handle millions of events per second. By combining time-windowed aggregations, self-joins, and the `windowFunnel` function, you can detect brute force, lateral movement, and multi-stage attacks without a dedicated correlation engine.
