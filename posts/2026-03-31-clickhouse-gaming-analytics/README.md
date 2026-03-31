# How to Build Gaming Analytics with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Gaming Analytics, Player Behavior, Game Telemetry, Analytics

Description: Build a comprehensive gaming analytics platform with ClickHouse to track player behavior, retention, and monetization at scale.

---

Gaming analytics requires processing billions of player events daily - sessions, level completions, purchases, and social interactions. ClickHouse is the ideal engine for this workload: high ingestion throughput, columnar compression for repetitive event data, and fast aggregations for live dashboards.

## Core Events Table

```sql
CREATE TABLE game_events (
    event_time      DateTime64(3),
    player_id       UInt64,
    session_id      String,
    game_id         UInt32,
    event_type      LowCardinality(String),
    level           UInt16,
    score           Int64,
    metadata        String,
    platform        LowCardinality(String),
    region          LowCardinality(String),
    date            Date DEFAULT toDate(event_time)
) ENGINE = MergeTree()
PARTITION BY date
ORDER BY (game_id, player_id, event_time);
```

## Daily Active Users

```sql
SELECT
    date,
    uniq(player_id) AS dau
FROM game_events
WHERE date >= today() - 30
GROUP BY date
ORDER BY date;
```

## Average Session Length

```sql
SELECT
    game_id,
    platform,
    round(avg(session_duration_s) / 60, 2) AS avg_session_min
FROM (
    SELECT
        game_id,
        platform,
        session_id,
        dateDiff('second', min(event_time), max(event_time)) AS session_duration_s
    FROM game_events
    WHERE date = today()
    GROUP BY game_id, platform, session_id
    HAVING session_duration_s > 0
)
GROUP BY game_id, platform
ORDER BY avg_session_min DESC;
```

## Level Completion Funnel

Track how many players complete each level to identify where players drop off:

```sql
SELECT
    level,
    uniq(player_id) AS players_reached,
    uniqIf(player_id, event_type = 'level_complete') AS players_completed,
    round(uniqIf(player_id, event_type = 'level_complete') / uniq(player_id) * 100, 2) AS completion_rate_pct
FROM game_events
WHERE date >= today() - 7
  AND event_type IN ('level_start', 'level_complete')
GROUP BY level
ORDER BY level;
```

## New vs Returning Players

```sql
SELECT
    date,
    uniqIf(player_id, is_new_player = 1) AS new_players,
    uniqIf(player_id, is_new_player = 0) AS returning_players
FROM (
    SELECT
        date,
        player_id,
        if(min(date) OVER (PARTITION BY player_id) = date, 1, 0) AS is_new_player
    FROM game_events
    WHERE date >= today() - 30
)
GROUP BY date
ORDER BY date;
```

## Top Events by Volume

```sql
SELECT
    event_type,
    count() AS count,
    uniq(player_id) AS unique_players
FROM game_events
WHERE date = today()
GROUP BY event_type
ORDER BY count DESC
LIMIT 20;
```

## Summary

ClickHouse powers gaming analytics at any scale, from indie studios to large publishers. By capturing all events in a single partitioned table and combining funnel queries, DAU tracking, and session analysis, game teams get the full picture of player engagement without the cost of managed data warehouses.
