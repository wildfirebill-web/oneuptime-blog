# How to Build Collaborative Filtering Features in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Collaborative Filtering, Recommendation, Cosine Similarity, Analytics

Description: Learn how to build user-based and item-based collaborative filtering in ClickHouse using dot products and cosine similarity to power recommendations.

---

Collaborative filtering recommends items to users based on patterns from similar users or similar items. ClickHouse's aggregation engine makes it efficient to compute similarity matrices over millions of interactions.

## Data Model

```sql
CREATE TABLE user_ratings (
    user_id UInt32,
    item_id UInt32,
    rating Float32
) ENGINE = MergeTree() ORDER BY (user_id, item_id);
```

## User-Based Similarity (Cosine)

Compute the cosine similarity between pairs of users:

```sql
SELECT
    a.user_id AS user_a,
    b.user_id AS user_b,
    sum(a.rating * b.rating) /
        (sqrt(sum(a.rating * a.rating)) * sqrt(sum(b.rating * b.rating))) AS cosine_sim
FROM user_ratings a
JOIN user_ratings b ON a.item_id = b.item_id AND a.user_id < b.user_id
GROUP BY user_a, user_b
HAVING cosine_sim > 0.5
ORDER BY cosine_sim DESC;
```

## Finding Top-N Similar Users

```sql
SELECT user_b AS similar_user, cosine_sim
FROM user_similarity
WHERE user_a = {target_user:UInt32}
ORDER BY cosine_sim DESC
LIMIT 10;
```

## Generating Item Recommendations

For a target user, find items rated by similar users but not yet seen:

```sql
WITH similar_users AS (
    SELECT user_b AS similar_user, cosine_sim
    FROM user_similarity
    WHERE user_a = {target_user:UInt32}
    ORDER BY cosine_sim DESC
    LIMIT 20
),
seen AS (
    SELECT item_id FROM user_ratings WHERE user_id = {target_user:UInt32}
)
SELECT
    r.item_id,
    sum(r.rating * s.cosine_sim) AS weighted_score
FROM user_ratings r
JOIN similar_users s ON r.user_id = s.similar_user
WHERE r.item_id NOT IN (SELECT item_id FROM seen)
GROUP BY r.item_id
ORDER BY weighted_score DESC
LIMIT 10;
```

## Item-Based Similarity

Swap user and item dimensions for item-based CF:

```sql
SELECT
    a.item_id AS item_a,
    b.item_id AS item_b,
    sum(a.rating * b.rating) /
        (sqrt(sum(a.rating * a.rating)) * sqrt(sum(b.rating * b.rating))) AS sim
FROM user_ratings a
JOIN user_ratings b ON a.user_id = b.user_id AND a.item_id < b.item_id
GROUP BY item_a, item_b
HAVING sim > 0.4
ORDER BY sim DESC;
```

## Summary

ClickHouse supports collaborative filtering through cosine similarity computed via aggregated dot products over the user-item matrix. Use user-based CF to find similar users and predict unseen ratings, or item-based CF to recommend items similar to ones a user already likes.
