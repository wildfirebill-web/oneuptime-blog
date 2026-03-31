# How to Design a Schema for a Social Network in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Social Network, Schema Design, Graph

Description: Learn how to design a social network schema in MySQL covering user profiles, follower graphs, posts, likes, and activity feeds.

---

A social network schema must handle user profiles, bidirectional or directed relationships (follows/friends), posts, reactions, and activity feed generation. Each component has specific performance requirements.

## User Profiles

```sql
CREATE TABLE users (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT,
    username   VARCHAR(50)  NOT NULL,
    email      VARCHAR(255) NOT NULL,
    bio        TEXT         NULL,
    avatar_url VARCHAR(512) NULL,
    created_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uq_username (username),
    UNIQUE KEY uq_email    (email)
);
```

## Follow Relationships (Directed Graph)

For a Twitter-style follow model (asymmetric), each row means "follower_id follows followee_id":

```sql
CREATE TABLE follows (
    follower_id INT UNSIGNED NOT NULL,
    followee_id INT UNSIGNED NOT NULL,
    created_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id),
    KEY idx_followee (followee_id, follower_id),
    CONSTRAINT fk_follow_follower FOREIGN KEY (follower_id) REFERENCES users (id) ON DELETE CASCADE,
    CONSTRAINT fk_follow_followee FOREIGN KEY (followee_id) REFERENCES users (id) ON DELETE CASCADE
);
```

## Posts

```sql
CREATE TABLE posts (
    id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    user_id    INT UNSIGNED    NOT NULL,
    body       TEXT            NOT NULL,
    like_count INT UNSIGNED    NOT NULL DEFAULT 0,
    created_at DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    KEY idx_user_time (user_id, created_at),
    CONSTRAINT fk_post_user FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE
);
```

`like_count` is a denormalized counter updated via trigger or application code to avoid expensive COUNT queries on every feed load.

## Likes

```sql
CREATE TABLE post_likes (
    post_id    BIGINT UNSIGNED NOT NULL,
    user_id    INT UNSIGNED    NOT NULL,
    created_at DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (post_id, user_id),
    KEY idx_user_likes (user_id, post_id),
    CONSTRAINT fk_like_post FOREIGN KEY (post_id) REFERENCES posts (id) ON DELETE CASCADE,
    CONSTRAINT fk_like_user FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE
);
```

## Activity Feed Query

The simplest feed: posts from people the current user follows.

```sql
SELECT p.id, p.body, p.like_count, p.created_at,
       u.username, u.avatar_url
FROM   posts p
JOIN   users u ON u.id = p.user_id
WHERE  p.user_id IN (
    SELECT followee_id FROM follows WHERE follower_id = 7
)
ORDER BY p.created_at DESC
LIMIT 20;
```

For large follower counts, pre-compute feeds in a `user_feeds` table on post creation.

## Mutual Followers (Friends)

```sql
SELECT f1.followee_id AS user_id
FROM   follows f1
JOIN   follows f2
    ON f2.follower_id = f1.followee_id
   AND f2.followee_id = f1.follower_id
WHERE  f1.follower_id = 7;
```

## Follower / Following Counts

```sql
SELECT
    u.username,
    (SELECT COUNT(*) FROM follows WHERE follower_id = u.id) AS following,
    (SELECT COUNT(*) FROM follows WHERE followee_id = u.id) AS followers
FROM users u WHERE u.id = 7;
```

## Summary

A social network schema uses a directed `follows` table with composite indexes supporting both directions. Denormalize counters like `like_count` to avoid aggregation queries on hot paths. For small to medium scale, the feed query with a subquery over follows works well. At scale, pre-compute feeds in a dedicated feed table populated on post creation.
