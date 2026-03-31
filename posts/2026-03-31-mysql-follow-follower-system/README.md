# How to Implement a Follow/Follower System in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Schema, Pattern, Relationship, Social

Description: Learn how to design a MySQL follow/follower system for social features, supporting mutual follows, follower counts, and activity feeds.

---

## Schema Design

A follow system is a directed graph: user A follows user B does not imply user B follows user A. The core table stores directed follow relationships.

```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    display_name VARCHAR(100),
    follower_count INT NOT NULL DEFAULT 0,
    following_count INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE follows (
    follower_id INT NOT NULL,  -- The user who is following
    followee_id INT NOT NULL,  -- The user being followed
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id),
    INDEX idx_followee (followee_id),
    FOREIGN KEY (follower_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (followee_id) REFERENCES users(id) ON DELETE CASCADE,
    CHECK (follower_id != followee_id)  -- Cannot follow yourself
);
```

## Follow and Unfollow Operations

```sql
DELIMITER $$

CREATE PROCEDURE follow_user(IN p_follower INT, IN p_followee INT)
BEGIN
    -- Insert follow relationship (ignore if already following)
    INSERT IGNORE INTO follows (follower_id, followee_id)
    VALUES (p_follower, p_followee);

    -- Update denormalized counts only if a row was inserted
    IF ROW_COUNT() > 0 THEN
        UPDATE users SET following_count = following_count + 1 WHERE id = p_follower;
        UPDATE users SET follower_count = follower_count + 1 WHERE id = p_followee;
    END IF;
END$$

CREATE PROCEDURE unfollow_user(IN p_follower INT, IN p_followee INT)
BEGIN
    DELETE FROM follows
    WHERE follower_id = p_follower
      AND followee_id = p_followee;

    IF ROW_COUNT() > 0 THEN
        UPDATE users SET following_count = GREATEST(0, following_count - 1) WHERE id = p_follower;
        UPDATE users SET follower_count = GREATEST(0, follower_count - 1) WHERE id = p_followee;
    END IF;
END$$

DELIMITER ;

-- Usage
CALL follow_user(1, 2);    -- User 1 follows User 2
CALL unfollow_user(1, 2);  -- User 1 unfollows User 2
```

## Querying Followers and Following

```sql
-- Get all followers of user 2
SELECT u.id, u.username, u.display_name, f.created_at AS followed_since
FROM follows f
JOIN users u ON f.follower_id = u.id
WHERE f.followee_id = 2
ORDER BY f.created_at DESC
LIMIT 20;

-- Get all users that user 1 is following
SELECT u.id, u.username, u.display_name, f.created_at AS following_since
FROM follows f
JOIN users u ON f.followee_id = u.id
WHERE f.follower_id = 1
ORDER BY f.created_at DESC
LIMIT 20;
```

## Mutual Follows (Friends)

```sql
-- Find mutual follows (people user 1 follows who also follow user 1)
SELECT u.id, u.username, u.display_name
FROM follows f1
JOIN follows f2 ON f1.follower_id = f2.followee_id
                AND f1.followee_id = f2.follower_id
JOIN users u ON f1.followee_id = u.id
WHERE f1.follower_id = 1;
```

## "People You May Know" Suggestions

```sql
-- Suggest users followed by people user 1 follows, but not already followed
SELECT
    u.id,
    u.username,
    COUNT(*) AS mutual_connections
FROM follows f1
JOIN follows f2 ON f1.followee_id = f2.follower_id
JOIN users u ON f2.followee_id = u.id
WHERE f1.follower_id = 1         -- Users that user 1 follows
  AND f2.followee_id != 1        -- Exclude user 1 themselves
  AND NOT EXISTS (
    SELECT 1 FROM follows f3
    WHERE f3.follower_id = 1
      AND f3.followee_id = f2.followee_id
  )
GROUP BY u.id, u.username
ORDER BY mutual_connections DESC
LIMIT 10;
```

## Activity Feed from Followed Users

```sql
-- Get recent posts from users that user 1 follows
SELECT
    p.id,
    p.title,
    p.created_at,
    u.username AS author
FROM posts p
JOIN users u ON p.author_id = u.id
JOIN follows f ON f.followee_id = p.author_id
WHERE f.follower_id = 1
ORDER BY p.created_at DESC
LIMIT 20;
```

## Checking Follow Status

```sql
-- Check if user 1 follows user 2
SELECT EXISTS (
    SELECT 1 FROM follows
    WHERE follower_id = 1 AND followee_id = 2
) AS is_following;
```

## Summary

A MySQL follow/follower system uses a directed relationship table with a composite primary key enforcing uniqueness. Denormalized follower and following counts on the user table enable fast display without aggregate queries. Stored procedures encapsulate the follow/unfollow logic including counter maintenance, keeping application code simple while ensuring data consistency.
