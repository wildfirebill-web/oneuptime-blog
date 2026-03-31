# How to Implement Frequency Capping Analytics with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Frequency Capping, Ad Tech, User Analytics, Impression Tracking

Description: Use ClickHouse to analyze and enforce frequency capping by tracking how many times a user sees each ad over configurable time windows.

---

Frequency capping prevents ad fatigue by limiting how many times a user sees the same ad within a given window. ClickHouse enables both post-hoc analysis of frequency distribution and real-time cap enforcement through fast per-user aggregations.

## Impression Log Table

```sql
CREATE TABLE ad_impressions (
    event_time      DateTime64(3),
    user_id         UInt64,
    ad_id           UInt32,
    campaign_id     UInt32,
    date            Date DEFAULT toDate(event_time)
) ENGINE = MergeTree()
PARTITION BY date
ORDER BY (user_id, ad_id, event_time);
```

## Frequency Distribution Analysis

Understand how often users see each ad to set appropriate caps:

```sql
SELECT
    ad_id,
    impressions_per_user,
    count() AS user_count
FROM (
    SELECT
        ad_id,
        user_id,
        count() AS impressions_per_user
    FROM ad_impressions
    WHERE date >= today() - 7
    GROUP BY ad_id, user_id
)
GROUP BY ad_id, impressions_per_user
ORDER BY ad_id, impressions_per_user;
```

## Users Who Have Exceeded a Cap

Find users who have seen an ad more than the defined cap (e.g., 5 times per day):

```sql
SELECT
    user_id,
    ad_id,
    count() AS frequency
FROM ad_impressions
WHERE date = today()
GROUP BY user_id, ad_id
HAVING frequency > 5
ORDER BY frequency DESC
LIMIT 100;
```

## Cap Compliance Rate by Campaign

Measure what percentage of user-ad pairs stayed within a 3-per-day cap:

```sql
SELECT
    campaign_id,
    count() AS total_pairs,
    countIf(freq <= 3) AS within_cap,
    round(countIf(freq <= 3) / count() * 100, 1) AS compliance_pct
FROM (
    SELECT
        campaign_id,
        user_id,
        count() AS freq
    FROM ad_impressions
    WHERE date = today()
    GROUP BY campaign_id, user_id
)
GROUP BY campaign_id
ORDER BY compliance_pct ASC;
```

## Hourly Frequency Tracking

Track frequency across rolling hourly windows to analyze within-day exposure:

```sql
SELECT
    toStartOfHour(event_time) AS hour,
    ad_id,
    avg(freq) AS avg_frequency_per_user
FROM (
    SELECT
        toStartOfHour(event_time) AS hour,
        ad_id,
        user_id,
        count() AS freq
    FROM ad_impressions
    WHERE date = today()
    GROUP BY hour, ad_id, user_id
)
GROUP BY hour, ad_id
ORDER BY hour, ad_id;
```

## Users Ready for Re-Targeting

Find users who saw an ad in the past but not in the last 3 days - prime re-targeting candidates:

```sql
SELECT DISTINCT user_id
FROM ad_impressions
WHERE campaign_id = 42
  AND date < today() - 3
  AND user_id NOT IN (
      SELECT DISTINCT user_id
      FROM ad_impressions
      WHERE campaign_id = 42
        AND date >= today() - 3
  );
```

## Summary

ClickHouse enables granular frequency analytics by efficiently grouping billions of impressions by user-ad pairs. Frequency distribution reports help marketers set data-driven caps, while real-time compliance queries let operations teams verify that delivery systems are honoring those limits.
