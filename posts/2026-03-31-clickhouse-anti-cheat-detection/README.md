# How to Build Anti-Cheat Detection with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Gaming Analytics, Anti-Cheat, Fraud Detection, Anomaly Detection

Description: Use ClickHouse to store player telemetry and detect cheating patterns such as aimbots, speed hacks, and stat anomalies in real time.

---

Cheating ruins online games for honest players and erodes trust in a title. With billions of game events flowing through live service games, you need a system that can flag suspicious behavior quickly. ClickHouse's analytical power makes it an excellent backend for anti-cheat pipelines.

## Telemetry Event Table

```sql
CREATE TABLE game_events (
    event_id        UUID,
    player_id       UInt64,
    game_id         UInt32,
    match_id        UInt64,
    occurred_at     DateTime64(3),
    event_type      LowCardinality(String),
    pos_x           Float32,
    pos_y           Float32,
    pos_z           Float32,
    velocity        Float32,
    accuracy_pct    Float32,
    headshot        UInt8,
    kills_in_match  UInt16,
    ping_ms         UInt16,
    client_version  LowCardinality(String)
) ENGINE = MergeTree()
ORDER BY (game_id, match_id, player_id, occurred_at)
PARTITION BY toYYYYMM(occurred_at);
```

## Detecting Impossible Movement Speed

A player moving faster than the game engine allows is a classic speed-hack indicator.

```sql
SELECT
    player_id,
    match_id,
    max(velocity) AS max_velocity
FROM game_events
WHERE game_id = 42
  AND occurred_at >= now() - INTERVAL 1 HOUR
GROUP BY player_id, match_id
HAVING max_velocity > 25.0   -- units per second, game-specific threshold
ORDER BY max_velocity DESC
LIMIT 50;
```

## Aimbot Detection via Abnormal Accuracy

```sql
SELECT
    player_id,
    count()                              AS shots_fired,
    round(avg(accuracy_pct), 2)          AS avg_accuracy,
    round(avg(headshot), 2)              AS headshot_rate,
    round(quantile(0.95)(accuracy_pct), 2) AS p95_accuracy
FROM game_events
WHERE event_type = 'shot'
  AND game_id = 42
  AND occurred_at >= today() - 1
GROUP BY player_id
HAVING shots_fired > 100
   AND headshot_rate > 0.80   -- 80%+ headshots is statistically improbable
ORDER BY headshot_rate DESC
LIMIT 50;
```

## Kill Rate Outlier Detection

```sql
SELECT
    player_id,
    match_id,
    max(kills_in_match) AS kills
FROM game_events
WHERE game_id = 42
  AND occurred_at >= today() - 7
GROUP BY player_id, match_id
HAVING kills > (
    SELECT quantile(0.9999)(max_kills)
    FROM (
        SELECT match_id, max(kills_in_match) AS max_kills
        FROM game_events
        WHERE game_id = 42
        GROUP BY match_id
    )
)
ORDER BY kills DESC;
```

## Teleportation Detection

A player whose position changes by more than the maximum possible displacement between consecutive events is likely teleporting.

```sql
SELECT
    player_id,
    match_id,
    count() AS teleport_events
FROM (
    SELECT
        player_id,
        match_id,
        occurred_at,
        sqrt(
            pow(pos_x - lagInFrame(pos_x) OVER w, 2) +
            pow(pos_y - lagInFrame(pos_y) OVER w, 2) +
            pow(pos_z - lagInFrame(pos_z) OVER w, 2)
        ) AS displacement
    FROM game_events
    WHERE game_id = 42
    WINDOW w AS (PARTITION BY player_id, match_id ORDER BY occurred_at)
)
WHERE displacement > 50   -- units, game-specific threshold
GROUP BY player_id, match_id
HAVING teleport_events > 3
ORDER BY teleport_events DESC;
```

## Flagging Suspicious Players in Bulk

Combine multiple signals with a scoring approach.

```sql
SELECT
    player_id,
    sumIf(1, signal = 'high_headshot')   AS headshot_flags,
    sumIf(1, signal = 'speed_hack')      AS speed_flags,
    sumIf(1, signal = 'teleport')        AS teleport_flags
FROM suspicious_signals   -- populated by the queries above via INSERT INTO
WHERE flagged_at >= today() - 7
GROUP BY player_id
ORDER BY (headshot_flags + speed_flags + teleport_flags) DESC
LIMIT 100;
```

## Summary

ClickHouse enables anti-cheat detection by providing fast aggregation over billions of game events. Statistical outlier queries using `quantile` functions and window functions like `lagInFrame` let you identify speed hackers, aimbots, and teleporting players without building complex stream processing pipelines. Combine these signals to build a risk score and feed ban decisions back into your moderation workflow.
