# How to Track Player Events and Actions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Gaming, Player Event, Analytics, Game Development

Description: Track player events and in-game actions in ClickHouse to understand player behavior, session patterns, and feature engagement at scale.

---

Player event tracking is the foundation of game analytics. Every action a player takes - moving, shooting, buying, completing a level - is a signal that helps you understand behavior, balance game design, and detect cheating. ClickHouse stores and queries these events efficiently at gaming scale.

## Player Events Table

```sql
CREATE TABLE player_events
(
    event_id UUID DEFAULT generateUUIDv4(),
    player_id UInt64,
    session_id UUID,
    game_id LowCardinality(String),
    event_type LowCardinality(String),
    level UInt16 DEFAULT 0,
    zone LowCardinality(String) DEFAULT '',
    action LowCardinality(String),
    target_id UInt64 DEFAULT 0,
    value Float64 DEFAULT 0,
    x_pos Float32 DEFAULT 0,
    y_pos Float32 DEFAULT 0,
    z_pos Float32 DEFAULT 0,
    client_version LowCardinality(String),
    platform LowCardinality(String),
    event_time DateTime64(3)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(event_time)
ORDER BY (game_id, player_id, event_time);
```

## Top Actions in the Last Hour

```sql
SELECT
    event_type,
    action,
    count() AS occurrences,
    countDistinct(player_id) AS unique_players,
    round(avg(value), 2) AS avg_value
FROM player_events
WHERE event_time >= now() - INTERVAL 1 HOUR
GROUP BY event_type, action
ORDER BY occurrences DESC
LIMIT 30;
```

## Player Session Analysis

```sql
SELECT
    player_id,
    session_id,
    count() AS events_in_session,
    min(event_time) AS session_start,
    max(event_time) AS session_end,
    dateDiff('minute', min(event_time), max(event_time)) AS duration_minutes,
    countDistinct(zone) AS zones_visited
FROM player_events
WHERE event_time >= now() - INTERVAL 24 HOUR
GROUP BY player_id, session_id
ORDER BY duration_minutes DESC
LIMIT 20;
```

## Most Active Zones / Areas in Game

```sql
SELECT
    zone,
    level,
    count() AS events,
    countDistinct(player_id) AS unique_players,
    countIf(event_type = 'death') AS deaths,
    round(countIf(event_type = 'death') * 100.0 / count(), 2) AS death_pct
FROM player_events
WHERE event_time >= now() - INTERVAL 24 HOUR
  AND zone != ''
GROUP BY zone, level
ORDER BY events DESC
LIMIT 20;
```

## First-Time Players vs Returning Players

```sql
WITH first_session AS (
    SELECT player_id, min(toDate(event_time)) AS first_day
    FROM player_events
    GROUP BY player_id
)
SELECT
    multiIf(first_day = today(), 'new', first_day >= today() - 7, 'recent', 'veteran') AS player_type,
    count() AS events,
    countDistinct(p.player_id) AS unique_players
FROM player_events p
JOIN first_session f ON p.player_id = f.player_id
WHERE p.event_time >= now() - INTERVAL 1 HOUR
GROUP BY player_type
ORDER BY unique_players DESC;
```

## Platform and Version Distribution

```sql
SELECT
    platform,
    client_version,
    countDistinct(player_id) AS active_players,
    count() AS events
FROM player_events
WHERE event_time >= now() - INTERVAL 24 HOUR
GROUP BY platform, client_version
ORDER BY active_players DESC
LIMIT 20;
```

## Player Heatmap Data - Event Coordinates

```sql
SELECT
    round(x_pos / 10) * 10 AS x_bucket,
    round(y_pos / 10) * 10 AS y_bucket,
    count() AS events,
    countIf(event_type = 'death') AS deaths
FROM player_events
WHERE zone = 'dungeon_1'
  AND event_time >= now() - INTERVAL 24 HOUR
GROUP BY x_bucket, y_bucket
ORDER BY events DESC
LIMIT 50;
```

## Summary

ClickHouse handles player event tracking at game scale through fast ingest, efficient columnar storage, and aggregate queries across sessions, zones, and event types. Heatmap data, session summaries, and platform breakdowns are all practical at billions of events per day. This foundation supports game designers making data-driven decisions about level design, feature balance, and player experience improvements.
