# How to Use groupUniqArray() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, GroupUniqArray, Array, Distinct

Description: Learn how to use groupUniqArray() and groupUniqArray(N)() in ClickHouse to collect unique column values into arrays during aggregation with practical examples.

---

`groupUniqArray(column)` is a ClickHouse aggregate function that collects all unique (distinct) values of a column within a group into an array. It is the deduplicated counterpart to `groupArray()` - instead of collecting every value including duplicates, it uses a hash set internally to emit only unique values. This makes it more memory-efficient and convenient than calling `arrayDistinct(groupArray(col))`.

## Basic groupUniqArray()

```sql
CREATE TABLE user_tags
(
    user_id UInt64,
    tag     String,
    added_at DateTime
)
ENGINE = MergeTree()
ORDER BY (added_at, user_id);

INSERT INTO user_tags VALUES
    (1, 'premium',  '2026-01-01 00:00:00'),
    (1, 'active',   '2026-02-01 00:00:00'),
    (1, 'premium',  '2026-03-01 00:00:00'),  -- duplicate
    (2, 'trial',    '2026-01-15 00:00:00'),
    (2, 'active',   '2026-02-15 00:00:00'),
    (2, 'active',   '2026-03-15 00:00:00');  -- duplicate

-- Collect unique tags per user
SELECT
    user_id,
    groupUniqArray(tag) AS unique_tags
FROM user_tags
GROUP BY user_id;
```

Output:
```text
user_id | unique_tags
--------|---------------------
1       | ['premium', 'active']
2       | ['trial', 'active']
```

Duplicates are removed during aggregation - no post-processing needed.

## groupUniqArray(N)(column) - Limit Unique Array Size

`groupUniqArray(N)(column)` collects at most `N` unique values. Once `N` distinct values are accumulated, additional new values are discarded.

```sql
-- Collect up to 10 unique tags per user
SELECT
    user_id,
    groupUniqArray(10)(tag) AS top_unique_tags
FROM user_tags
GROUP BY user_id;
```

This is useful when you need a sample of distinct values for display purposes and want to cap memory usage for high-cardinality groups.

## groupUniqArray vs arrayDistinct(groupArray())

Both produce a unique-value array, but they differ in when deduplication happens:

```sql
-- Option 1: groupUniqArray - deduplicates during aggregation (efficient)
SELECT user_id, groupUniqArray(tag) AS unique_tags
FROM user_tags
GROUP BY user_id;

-- Option 2: groupArray + arrayDistinct - accumulates all values first, then deduplicates
SELECT user_id, arrayDistinct(groupArray(tag)) AS unique_tags
FROM user_tags
GROUP BY user_id;
```

`groupUniqArray` is more memory-efficient because it maintains a hash set and never stores duplicate values. `arrayDistinct(groupArray(...))` first collects all values (including duplicates) and then deduplicates, which uses more intermediate memory.

## Collecting Unique Values with Conditions

You can filter which rows contribute to the unique array using a subquery or `WHERE` clause:

```sql
-- Unique tags added in 2026 per user
SELECT
    user_id,
    groupUniqArray(tag) AS tags_in_2026
FROM user_tags
WHERE toYear(added_at) = 2026
GROUP BY user_id;
```

For conditional collection without a subquery, combine with `if()`:

```sql
-- Collect unique recent tags and unique old tags in one pass
SELECT
    user_id,
    groupUniqArray(if(added_at >= '2026-01-01', tag, '')) AS recent_tags,
    groupUniqArray(if(added_at < '2026-01-01',  tag, '')) AS old_tags
FROM user_tags
GROUP BY user_id;
```

Note: this approach includes an empty string `''` in the array for rows that do not match the condition. Filter it out with `arrayFilter`:

```sql
SELECT
    user_id,
    arrayFilter(x -> x != '', groupUniqArray(if(added_at >= '2026-01-01', tag, ''))) AS recent_tags
FROM user_tags
GROUP BY user_id;
```

## Counting Unique Values vs Collecting Them

When you only need the count of distinct values (not the values themselves), use `uniq()` instead - it is much more memory-efficient:

```sql
-- If you need the count only, use uniq()
SELECT user_id, uniq(tag) AS unique_tag_count
FROM user_tags
GROUP BY user_id;

-- If you need the actual values, use groupUniqArray()
SELECT user_id, groupUniqArray(tag) AS unique_tags
FROM user_tags
GROUP BY user_id;
```

## Practical Use Cases

### Permission and role listing

```sql
CREATE TABLE user_permissions
(
    user_id    UInt64,
    permission String,
    granted_at DateTime
)
ENGINE = MergeTree()
ORDER BY (user_id, permission);

-- All unique permissions per user
SELECT
    user_id,
    groupUniqArray(permission) AS permissions
FROM user_permissions
GROUP BY user_id;
```

### Unique category tagging for products

```sql
CREATE TABLE product_categories
(
    product_id  UInt64,
    category    String,
    assigned_at DateTime
)
ENGINE = MergeTree()
ORDER BY (product_id, category);

-- Distinct categories per product
SELECT
    product_id,
    groupUniqArray(category)    AS categories,
    length(groupUniqArray(category)) AS category_count
FROM product_categories
GROUP BY product_id
HAVING category_count >= 2;
```

### Multi-value filtering with hasAll / hasAny

Collect unique tags and then filter groups by array membership:

```sql
-- Users who have both 'premium' and 'active' tags
SELECT
    user_id,
    groupUniqArray(tag) AS tags
FROM user_tags
GROUP BY user_id
HAVING hasAll(tags, ['premium', 'active']);
```

## Summary

`groupUniqArray(col)` collects distinct values into an array during aggregation, using a hash set to avoid storing duplicates. It is more memory-efficient than `arrayDistinct(groupArray(col))` for duplicate-heavy data. Use `groupUniqArray(N)(col)` to cap the result to `N` unique values. When only the count of unique values is needed, prefer `uniq()` to avoid materializing the array entirely.
