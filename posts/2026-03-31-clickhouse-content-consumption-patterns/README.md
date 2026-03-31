# How to Track Content Consumption Patterns in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Content Consumption, Streaming, Analytics, Engagement, Recommendation

Description: Track content consumption patterns in ClickHouse to analyze viewing behavior, completion rates, binge-watching trends, and content performance for recommendations.

---

Understanding how users consume content enables personalized recommendations, content investment decisions, and churn prevention. ClickHouse aggregates viewing events to reveal consumption patterns at scale.

## Content Viewing Events Table

```sql
CREATE TABLE viewing_events (
    event_id     UUID,
    user_id      UInt64,
    content_id   String,
    content_title String,
    genre        LowCardinality(String),
    content_type LowCardinality(String),  -- movie, series, episode, short
    series_id    Nullable(String),
    season_num   Nullable(UInt8),
    episode_num  Nullable(UInt8),
    event_type   LowCardinality(String),  -- start, progress, complete, skip, resume
    watch_position_s UInt32,
    content_duration_s UInt32,
    device_type  LowCardinality(String),
    recorded_at  DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(recorded_at)
ORDER BY (user_id, recorded_at);
```

## Content Completion Rate

```sql
SELECT
    content_id,
    content_title,
    genre,
    count() AS starts,
    countIf(event_type = 'complete') AS completions,
    round(countIf(event_type = 'complete') / count() * 100, 2) AS completion_rate_pct,
    avg(watch_position_s) AS avg_watch_time_s,
    avg(content_duration_s) AS content_duration_s
FROM viewing_events
WHERE event_type IN ('start', 'complete')
  AND recorded_at >= today() - 7
GROUP BY content_id, content_title, genre
ORDER BY starts DESC
LIMIT 50;
```

## Viewing Drop-Off Analysis

Find where in the content users stop watching:

```sql
SELECT
    content_id,
    content_title,
    round(watch_position_s / content_duration_s * 100 / 10) * 10 AS watch_pct_bucket,
    count() AS events_at_position
FROM viewing_events
WHERE event_type = 'progress'
  AND recorded_at >= today() - 7
  AND content_duration_s > 0
GROUP BY content_id, content_title, watch_pct_bucket
ORDER BY content_id, watch_pct_bucket;
```

## Binge-Watching Detection

Identify users who watch multiple episodes in a single session:

```sql
SELECT
    user_id,
    series_id,
    toDate(recorded_at) AS watch_date,
    countDistinct(episode_num) AS episodes_watched,
    sum(watch_position_s) AS total_watch_seconds
FROM viewing_events
WHERE event_type = 'complete'
  AND content_type = 'episode'
  AND recorded_at >= today() - 30
GROUP BY user_id, series_id, watch_date
HAVING episodes_watched >= 3
ORDER BY episodes_watched DESC
LIMIT 100;
```

## Genre Consumption by Day of Week

Understand what genre users prefer on which days:

```sql
SELECT
    genre,
    toDayOfWeek(recorded_at) AS day_of_week,
    toDayOfWeekName(recorded_at) AS day_name,
    count() AS views,
    sum(watch_position_s) AS total_watch_seconds
FROM viewing_events
WHERE event_type IN ('start', 'progress')
  AND recorded_at >= today() - 30
GROUP BY genre, day_of_week, day_name
ORDER BY genre, day_of_week;
```

## Content Discovery Funnel

Track how many users start vs. finish content to find engagement drop-offs:

```sql
SELECT
    content_id,
    content_title,
    countDistinctIf(user_id, event_type = 'start') AS started,
    countDistinctIf(user_id, watch_position_s >= content_duration_s * 0.25) AS reached_25pct,
    countDistinctIf(user_id, watch_position_s >= content_duration_s * 0.5) AS reached_50pct,
    countDistinctIf(user_id, watch_position_s >= content_duration_s * 0.75) AS reached_75pct,
    countDistinctIf(user_id, event_type = 'complete') AS completed
FROM viewing_events
WHERE recorded_at >= today() - 7
GROUP BY content_id, content_title
ORDER BY started DESC
LIMIT 20;
```

## User Engagement Score

Rank users by engagement for churn modeling:

```sql
SELECT
    user_id,
    count() AS sessions_30d,
    sum(watch_position_s) / 3600 AS total_watch_hours_30d,
    countDistinct(content_id) AS unique_titles_watched,
    countIf(event_type = 'complete') AS completions_30d,
    round(countIf(event_type = 'complete') / nullIf(countIf(event_type = 'start'), 0) * 100, 2) AS completion_rate_pct
FROM viewing_events
WHERE recorded_at >= today() - 30
GROUP BY user_id
ORDER BY total_watch_hours_30d DESC
LIMIT 1000;
```

## Summary

ClickHouse provides the analytics infrastructure for content consumption intelligence - completion rates, drop-off analysis, binge-watching detection, and engagement scoring all run efficiently over billions of viewing events. These insights power recommendation engines, content acquisition decisions, and churn prevention programs.
