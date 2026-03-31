# How to Implement a Ranking System in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Ranking, Window Function, Leaderboard, Query

Description: Build a MySQL ranking system for leaderboards, scores, or ordered results using RANK(), DENSE_RANK(), and user variable techniques for older MySQL versions.

---

## Types of Ranking in MySQL

MySQL supports three standard ranking approaches in version 8.0+:

- `RANK()` - leaves gaps after ties (1, 1, 3, 4)
- `DENSE_RANK()` - no gaps after ties (1, 1, 2, 3)
- `ROW_NUMBER()` - unique numbers regardless of ties (1, 2, 3, 4)

Choosing between them depends on your use case. Sports leaderboards typically use `RANK()`, while ordinal position always uses `ROW_NUMBER()`.

## Sample Data

```sql
CREATE TABLE player_scores (
  player_id INT PRIMARY KEY,
  player_name VARCHAR(100),
  score INT,
  game_mode VARCHAR(30)
);

INSERT INTO player_scores VALUES
(1, 'Alice',  1500, 'ranked'),
(2, 'Bob',    1200, 'ranked'),
(3, 'Carol',  1500, 'ranked'),
(4, 'Dave',   900,  'ranked'),
(5, 'Eve',    1200, 'ranked'),
(6, 'Frank',  1800, 'ranked');
```

## MySQL 8.0 - Window Function Ranking

```sql
SELECT
  player_name,
  score,
  RANK()       OVER (ORDER BY score DESC) AS rank_with_gaps,
  DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rank,
  ROW_NUMBER() OVER (ORDER BY score DESC) AS row_num
FROM player_scores
WHERE game_mode = 'ranked'
ORDER BY score DESC;
```

Result:

```text
+-------------+-------+----------------+------------+---------+
| player_name | score | rank_with_gaps | dense_rank | row_num |
+-------------+-------+----------------+------------+---------+
| Frank       |  1800 |              1 |          1 |       1 |
| Alice       |  1500 |              2 |          2 |       2 |
| Carol       |  1500 |              2 |          2 |       3 |
| Bob         |  1200 |              4 |          3 |       4 |
| Eve         |  1200 |              4 |          3 |       5 |
| Dave        |   900 |              6 |          4 |       6 |
+-------------+-------+----------------+------------+---------+
```

## Ranking Within Groups (PARTITION BY)

Rank players within each game mode:

```sql
SELECT
  player_name,
  game_mode,
  score,
  RANK() OVER (PARTITION BY game_mode ORDER BY score DESC) AS mode_rank
FROM player_scores
ORDER BY game_mode, mode_rank;
```

## MySQL 5.7 - Ranking with User Variables

```sql
SET @rank := 0;
SET @prev_score := NULL;
SET @row_num := 0;

SELECT
  player_name,
  score,
  @row_num := @row_num + 1 AS row_num,
  @rank := IF(@prev_score = score, @rank, @row_num) AS rank_with_gaps,
  @prev_score := score AS _score
FROM player_scores
ORDER BY score DESC;
```

## Percentile Ranking

Assign each player a percentile rank (0-100):

```sql
SELECT
  player_name,
  score,
  ROUND(
    PERCENT_RANK() OVER (ORDER BY score DESC) * 100, 1
  ) AS percentile
FROM player_scores
ORDER BY score DESC;
```

`PERCENT_RANK()` returns 0 for the top rank and 1 for the bottom. The formula `(rank - 1) / (total_rows - 1)` underlies the calculation.

## Finding a Player's Rank

To look up a specific player's rank without returning the full table:

```sql
SELECT rank_position, player_name, score
FROM (
  SELECT
    player_name,
    score,
    RANK() OVER (ORDER BY score DESC) AS rank_position
  FROM player_scores
) ranked
WHERE player_name = 'Alice';
```

## Storing Precomputed Ranks

For high-traffic leaderboards, precompute and store ranks:

```sql
CREATE TABLE leaderboard (
  player_id INT PRIMARY KEY,
  player_name VARCHAR(100),
  score INT,
  rank_position INT,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Refresh with a scheduled event
INSERT INTO leaderboard (player_id, player_name, score, rank_position)
SELECT player_id, player_name, score,
  RANK() OVER (ORDER BY score DESC) AS rank_position
FROM player_scores
ON DUPLICATE KEY UPDATE
  score = VALUES(score),
  rank_position = VALUES(rank_position),
  updated_at = NOW();
```

## Summary

MySQL 8.0's `RANK()`, `DENSE_RANK()`, and `ROW_NUMBER()` window functions cover all standard leaderboard and ranking requirements. `RANK()` is appropriate for competitive rankings with ties, `DENSE_RANK()` for sequential tier assignments, and `ROW_NUMBER()` for unique ordinal positions. For MySQL 5.7, user variables with conditional logic simulate the same behavior. For high-traffic leaderboards, precompute ranks on a schedule and read from a dedicated ranking table.
