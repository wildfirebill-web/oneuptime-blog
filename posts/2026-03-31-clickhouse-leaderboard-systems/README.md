# How to Build Leaderboard Systems with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Leaderboard, Gaming Analytics, Ranking, Score Aggregation

Description: Build fast and accurate leaderboard systems with ClickHouse to rank players by score across daily, weekly, and all-time periods.

---

Leaderboards require ranking players by aggregate scores across configurable time windows. While Redis is often used for real-time sorted sets, ClickHouse provides richer analytics - historical leaderboards, score distribution analysis, and demographic breakdowns - all from the same event data.

## Scores Table

```sql
CREATE TABLE player_scores (
    event_time      DateTime,
    player_id       UInt64,
    game_id         UInt32,
    game_mode       LowCardinality(String),
    score           Int64,
    kills           UInt16,
    deaths          UInt16,
    assists         UInt16,
    date            Date DEFAULT toDate(event_time)
) ENGINE = MergeTree()
PARTITION BY date
ORDER BY (game_id, game_mode, player_id, event_time);
```

## Daily Leaderboard

```sql
SELECT
    player_id,
    sum(score) AS total_score,
    rank() OVER (ORDER BY sum(score) DESC) AS rank
FROM player_scores
WHERE date = today()
  AND game_id = 1
  AND game_mode = 'ranked'
GROUP BY player_id
ORDER BY rank
LIMIT 100;
```

## Weekly Top Players

```sql
SELECT
    player_id,
    sum(score) AS weekly_score,
    count() AS games_played,
    round(sum(score) / count(), 0) AS avg_score_per_game
FROM player_scores
WHERE date >= today() - 7
  AND game_id = 1
GROUP BY player_id
ORDER BY weekly_score DESC
LIMIT 50;
```

## All-Time Records by Game Mode

```sql
SELECT
    game_mode,
    player_id,
    max(score) AS highest_score,
    argMax(event_time, score) AS record_set_at
FROM player_scores
WHERE game_id = 1
GROUP BY game_mode, player_id
ORDER BY game_mode, highest_score DESC;
```

## Player Rank History

Track how a player's rank has changed over time:

```sql
SELECT
    date,
    rank_value
FROM (
    SELECT
        date,
        player_id,
        rank() OVER (PARTITION BY date ORDER BY daily_score DESC) AS rank_value
    FROM (
        SELECT
            date,
            player_id,
            sum(score) AS daily_score
        FROM player_scores
        WHERE game_id = 1
          AND date >= today() - 30
        GROUP BY date, player_id
    )
)
WHERE player_id = 12345
ORDER BY date;
```

## Score Distribution

Understand score distribution to set tier boundaries:

```sql
SELECT
    quantile(0.50)(total_score) AS p50,
    quantile(0.75)(total_score) AS p75,
    quantile(0.90)(total_score) AS p90,
    quantile(0.95)(total_score) AS p95,
    quantile(0.99)(total_score) AS p99
FROM (
    SELECT
        player_id,
        sum(score) AS total_score
    FROM player_scores
    WHERE date >= today() - 7
    GROUP BY player_id
);
```

## Summary

ClickHouse window functions like `rank()` and `dense_rank()` make leaderboard computation straightforward over any time window. Unlike Redis sorted sets, ClickHouse lets you enrich rankings with win rates, average scores, and historical rank trajectories from a single SQL query.
