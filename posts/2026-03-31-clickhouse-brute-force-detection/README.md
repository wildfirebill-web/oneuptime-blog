# How to Detect Brute Force Attacks with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Brute Force, Security, Threat Detection, Authentication

Description: Detect brute force attacks in real time using ClickHouse by analyzing authentication failure patterns across IPs and user accounts.

---

Brute force attacks generate high volumes of authentication failures. ClickHouse's fast aggregation over recent time windows makes it ideal for detecting these attacks in near real time.

## Prerequisites

Assume the `auth_events` table from the authentication logs post exists.

## IP-Based Brute Force - High Failures from One IP

```sql
SELECT
    source_ip,
    countDistinct(user_name) AS targeted_accounts,
    count() AS total_failures,
    min(event_time) AS attack_start,
    max(event_time) AS last_attempt
FROM auth_events
WHERE outcome = 'failure'
  AND event_time >= now() - INTERVAL 15 MINUTE
GROUP BY source_ip
HAVING total_failures >= 20
ORDER BY total_failures DESC;
```

## Credential Stuffing - Many Accounts, Few Failures Each

```sql
SELECT
    source_ip,
    countDistinct(user_name) AS accounts_tried,
    count() AS total_attempts,
    round(count() * 1.0 / countDistinct(user_name), 2) AS attempts_per_account
FROM auth_events
WHERE outcome = 'failure'
  AND event_time >= now() - INTERVAL 1 HOUR
GROUP BY source_ip
HAVING accounts_tried > 50 AND attempts_per_account < 3
ORDER BY accounts_tried DESC;
```

## Account-Targeted Attack - Many Failures on One Account

```sql
SELECT
    user_name,
    countDistinct(source_ip) AS attacker_ips,
    count() AS total_failures,
    groupArray(5)(CAST(source_ip AS String)) AS sample_ips
FROM auth_events
WHERE outcome = 'failure'
  AND event_time >= now() - INTERVAL 1 HOUR
GROUP BY user_name
HAVING total_failures >= 10
ORDER BY total_failures DESC
LIMIT 20;
```

## Attack Timeline - 1-Minute Buckets

Track the progression of an attack over time.

```sql
SELECT
    toStartOfMinute(event_time) AS minute,
    source_ip,
    count() AS failures
FROM auth_events
WHERE outcome = 'failure'
  AND source_ip = '185.220.101.45'
  AND event_time >= now() - INTERVAL 2 HOUR
GROUP BY minute, source_ip
ORDER BY minute;
```

## Distributed Attack - Same Account from Many IPs

Detect when an account is attacked from a botnet.

```sql
SELECT
    user_name,
    countDistinct(source_ip) AS unique_ips,
    count() AS failures
FROM auth_events
WHERE outcome = 'failure'
  AND event_time >= now() - INTERVAL 1 HOUR
GROUP BY user_name
HAVING unique_ips > 10
ORDER BY unique_ips DESC;
```

## Success After Failures (Successful Compromise)

The most dangerous pattern: failures followed by a successful login.

```sql
WITH fail_users AS (
    SELECT
        user_name,
        source_ip,
        count() AS fail_count
    FROM auth_events
    WHERE outcome = 'failure'
      AND event_time >= now() - INTERVAL 1 HOUR
    GROUP BY user_name, source_ip
    HAVING fail_count >= 5
)
SELECT
    s.user_name,
    s.source_ip,
    s.event_time AS success_time,
    f.fail_count
FROM auth_events s
JOIN fail_users f ON s.user_name = f.user_name AND s.source_ip = f.source_ip
WHERE s.outcome = 'success'
  AND s.event_time >= now() - INTERVAL 1 HOUR
ORDER BY s.event_time DESC;
```

## Blocklist Candidates - IPs to Block

Generate a list of IPs that should be blocked.

```sql
SELECT source_ip, count() AS failures
FROM auth_events
WHERE outcome = 'failure'
  AND event_time >= now() - INTERVAL 24 HOUR
GROUP BY source_ip
HAVING failures > 100
ORDER BY failures DESC
FORMAT TabSeparated;
```

## Summary

ClickHouse enables real-time brute force detection through fast aggregation over sliding time windows. The key patterns to watch are high failure volume per IP, credential stuffing (many accounts per IP with low attempts each), account-targeted attacks, and the critical success-after-failures pattern that signals a compromise. Running these queries every minute provides near-real-time threat detection.
