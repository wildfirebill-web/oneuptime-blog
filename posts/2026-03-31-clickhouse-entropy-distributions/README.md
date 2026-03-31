# How to Calculate Entropy of Distributions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Entropy, Information Theory, Statistics, Analytics

Description: Learn how to calculate Shannon entropy and distribution diversity in ClickHouse to measure data randomness and detect anomalies in categorical data.

---

Shannon entropy quantifies the randomness or diversity of a distribution. In analytics, it helps detect anomalies, measure feature importance, and compare category distributions over time.

## Shannon Entropy Formula

For a distribution with probabilities p_i:

```text
H = -sum(p_i * log2(p_i))
```

A uniform distribution has maximum entropy; a distribution where one value dominates has low entropy.

## Computing Entropy with ClickHouse's entropy Function

ClickHouse provides a built-in `entropy` aggregate function:

```sql
SELECT entropy(status_code) AS status_entropy
FROM http_requests
WHERE date = today();
```

Higher values indicate more diverse status codes; near-zero values mean one code dominates.

## Entropy Over Time

Track entropy day-by-day to detect distribution shifts:

```sql
SELECT
    toDate(event_time) AS day,
    entropy(country_code) AS geo_entropy,
    entropy(user_agent_family) AS ua_entropy
FROM sessions
GROUP BY day
ORDER BY day;
```

A sudden drop in entropy might indicate bot traffic flooding from one region.

## Entropy per Segment

Compare entropy across product categories:

```sql
SELECT
    category,
    entropy(payment_method) AS payment_entropy
FROM orders
GROUP BY category
ORDER BY payment_entropy ASC;
```

Low entropy means users in that category overwhelmingly prefer one payment method.

## Computing Entropy Manually

If you need to inspect per-value probabilities:

```sql
WITH counts AS (
    SELECT status_code, count() AS cnt
    FROM http_requests
    GROUP BY status_code
),
total AS (SELECT sum(cnt) AS n FROM counts)
SELECT
    -sum((cnt / n) * log2(cnt / n)) AS entropy
FROM counts, total;
```

## Entropy for Anomaly Detection

Alert when entropy drops below a threshold:

```sql
SELECT
    toStartOfHour(event_time) AS hour,
    entropy(source_ip) AS ip_entropy
FROM firewall_logs
GROUP BY hour
HAVING ip_entropy < 1.0
ORDER BY hour;
```

## Summary

ClickHouse's built-in `entropy()` aggregate computes Shannon entropy with a single function call. Use it to measure distribution diversity over time, compare segments, or trigger anomaly alerts when entropy drops below expected levels.
