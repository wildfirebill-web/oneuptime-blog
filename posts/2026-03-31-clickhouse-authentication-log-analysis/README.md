# How to Analyze Authentication Logs with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Authentication, Log Analysis, Security, Audit, SIEM

Description: Store and query authentication logs in ClickHouse to detect failed logins, brute-force attempts, and suspicious access patterns at scale.

---

Authentication logs are a primary data source for security monitoring. ClickHouse can ingest login events at high throughput and answer security queries in milliseconds, making it a practical alternative to dedicated SIEM tools for teams already operating ClickHouse clusters.

## Authentication Events Table

```sql
CREATE TABLE auth_events
(
    event_time DateTime,
    user_name LowCardinality(String),
    source_ip IPv4,
    auth_method LowCardinality(String),
    outcome LowCardinality(String),  -- 'success' | 'failure'
    failure_reason LowCardinality(String),
    service LowCardinality(String),
    host LowCardinality(String)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (outcome, user_name, event_time)
TTL event_time + INTERVAL 1 YEAR;
```

## Ingest Logs from a File

```sql
INSERT INTO auth_events
SELECT
    parseDateTimeBestEffort(JSONExtractString(raw, 'timestamp')) AS event_time,
    JSONExtractString(raw, 'user')          AS user_name,
    toIPv4(JSONExtractString(raw, 'src_ip')) AS source_ip,
    JSONExtractString(raw, 'method')        AS auth_method,
    JSONExtractString(raw, 'result')        AS outcome,
    JSONExtractString(raw, 'reason')        AS failure_reason,
    JSONExtractString(raw, 'service')       AS service,
    JSONExtractString(raw, 'host')          AS host
FROM file('/var/log/auth_events.jsonl', 'JSONEachRow', 'raw String');
```

## Failed Login Count per User (Last 24 Hours)

```sql
SELECT
    user_name,
    count() AS failures,
    uniq(source_ip) AS distinct_ips
FROM auth_events
WHERE outcome = 'failure'
  AND event_time >= now() - INTERVAL 24 HOUR
GROUP BY user_name
ORDER BY failures DESC
LIMIT 20;
```

## Brute-Force Detection: IPs with Many Failures

```sql
SELECT
    source_ip,
    count() AS attempts,
    uniq(user_name) AS targeted_users,
    min(event_time) AS first_seen,
    max(event_time) AS last_seen
FROM auth_events
WHERE outcome = 'failure'
  AND event_time >= now() - INTERVAL 1 HOUR
GROUP BY source_ip
HAVING attempts > 50
ORDER BY attempts DESC;
```

## Successful Logins After Multiple Failures (Credential Stuffing)

Detect IPs that failed many times then succeeded within a short window.

```sql
WITH
    failures AS (
        SELECT source_ip, user_name, max(event_time) AS last_fail
        FROM auth_events
        WHERE outcome = 'failure'
          AND event_time >= now() - INTERVAL 1 HOUR
        GROUP BY source_ip, user_name
        HAVING count() >= 5
    ),
    successes AS (
        SELECT source_ip, user_name, min(event_time) AS success_time
        FROM auth_events
        WHERE outcome = 'success'
          AND event_time >= now() - INTERVAL 1 HOUR
        GROUP BY source_ip, user_name
    )
SELECT
    f.source_ip,
    f.user_name,
    f.last_fail,
    s.success_time,
    dateDiff('second', f.last_fail, s.success_time) AS seconds_to_success
FROM failures f
JOIN successes s USING (source_ip, user_name)
WHERE s.success_time > f.last_fail
ORDER BY seconds_to_success;
```

## Hourly Authentication Failure Rate

```sql
SELECT
    toStartOfHour(event_time) AS hour,
    countIf(outcome = 'failure') AS failures,
    countIf(outcome = 'success') AS successes,
    round(countIf(outcome = 'failure') * 100.0 / count(), 2) AS failure_pct
FROM auth_events
WHERE event_time >= now() - INTERVAL 7 DAY
GROUP BY hour
ORDER BY hour;
```

## Summary

ClickHouse is well-suited for authentication log analysis because its columnar storage and vectorized execution make aggregation queries over billions of events fast. By modelling events with low-cardinality fields and partitioning by month, you can retain a year of logs while keeping queries sub-second for security dashboards and incident investigations.
