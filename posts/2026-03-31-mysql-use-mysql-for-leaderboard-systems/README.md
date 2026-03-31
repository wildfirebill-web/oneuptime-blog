# How to Use MySQL for Leaderboard Systems

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Leaderboard, Ranking, Window Function, Performance

Description: Learn how to implement efficient leaderboard systems in MySQL using indexed score columns, window functions for rank calculation, and pagination for large player bases.

---

Leaderboards rank players, users, or entities by a numeric score. MySQL handles leaderboard workloads well when scores are indexed, rankings are computed with window functions, and pagination uses keyset rather than offset queries.

## Schema Design

Store scores in a dedicated table with an index on the score column:

```sql
CREATE TABLE leaderboard_scores (
  id           BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  game_id      INT UNSIGNED NOT NULL,
  user_id      BIGINT UNSIGNED NOT NULL,
  score        BIGINT NOT NULL DEFAULT 0,
  updated_at   DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uq_game_user (game_id, user_id),
  INDEX idx_game_score (game_id, score DESC)
) ENGINE=InnoDB;
```

The composite index `(game_id, score DESC)` allows the database to read the top scores for a specific game in index order without sorting.

## Updating Scores

Use `INSERT ... ON DUPLICATE KEY UPDATE` to atomically upsert a score, taking the maximum:

```sql
INSERT INTO leaderboard_scores (game_id, user_id, score)
VALUES (1, 42, 9500)
ON DUPLICATE KEY UPDATE
  score = GREATEST(score, VALUES(score)),
  updated_at = NOW();
```

For cumulative scoring (total points across multiple rounds):

```sql
INSERT INTO leaderboard_scores (game_id, user_id, score)
VALUES (1, 42, 500)
ON DUPLICATE KEY UPDATE
  score = score + VALUES(score),
  updated_at = NOW();
```

## Querying the Top N Players

```sql
SELECT
  RANK() OVER (ORDER BY score DESC) AS `rank`,
  u.username,
  ls.score,
  ls.updated_at
FROM leaderboard_scores ls
JOIN users u ON u.id = ls.user_id
WHERE ls.game_id = 1
ORDER BY ls.score DESC
LIMIT 100;
```

The `RANK()` window function assigns the same rank to tied scores and skips the next rank (e.g., 1, 2, 2, 4). Use `DENSE_RANK()` to avoid gaps.

## Finding a Specific Player's Rank

Computing exact rank for a single player without fetching the entire leaderboard:

```sql
SELECT COUNT(*) + 1 AS player_rank
FROM leaderboard_scores
WHERE game_id = 1
  AND score > (
    SELECT score FROM leaderboard_scores WHERE game_id = 1 AND user_id = 42
  );
```

This query counts how many players scored higher than the target player, adding 1 for the rank position.

## Paginating with Keyset

For scrollable leaderboards, keyset pagination avoids the performance cost of large offsets:

```sql
-- First page
SELECT user_id, score
FROM leaderboard_scores
WHERE game_id = 1
ORDER BY score DESC, user_id ASC
LIMIT 20;

-- Next page (after last row: score=8800, user_id=150)
SELECT user_id, score
FROM leaderboard_scores
WHERE game_id = 1
  AND (score < 8800 OR (score = 8800 AND user_id > 150))
ORDER BY score DESC, user_id ASC
LIMIT 20;
```

The `user_id` tie-break ensures a stable ordering when multiple players share the same score.

## Time-Windowed Leaderboards

For weekly or monthly leaderboards, use a `period` column to scope scores:

```sql
ALTER TABLE leaderboard_scores ADD COLUMN period CHAR(7) NOT NULL DEFAULT '2026-03';
ALTER TABLE leaderboard_scores DROP INDEX uq_game_user;
ALTER TABLE leaderboard_scores ADD UNIQUE KEY uq_game_user_period (game_id, user_id, period);
ALTER TABLE leaderboard_scores ADD INDEX idx_game_period_score (game_id, period, score DESC);
```

## Summary

MySQL leaderboards require an indexed score column, window functions for rank assignment, and keyset pagination for large player bases. The `GREATEST()` function handles high-score upserts atomically, while counting rows above a score efficiently finds a player's rank without scanning the full table. For millions of concurrent real-time updates, Redis Sorted Sets provide lower write latency, but MySQL handles thousands of updates per second comfortably.
