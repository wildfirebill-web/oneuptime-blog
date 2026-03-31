# How to Use LEFT ARRAY JOIN in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, SELECT, ARRAY JOIN, LEFT, Array

Description: Use LEFT ARRAY JOIN in ClickHouse to unnest arrays while preserving rows with empty or NULL arrays, unlike the inner ARRAY JOIN which drops them.

---

The standard `ARRAY JOIN` in ClickHouse behaves like an inner join: rows whose array is empty or `NULL` are silently dropped from the result. `LEFT ARRAY JOIN` changes that behaviour - it preserves every source row, emitting a single row with a `NULL` (or zero/empty-string default depending on the element type) for rows that have an empty array. This mirrors the semantics of a `LEFT JOIN` in standard SQL.

## Inner vs. LEFT ARRAY JOIN

The difference is easiest to see with a small example:

```sql
CREATE TABLE user_tags
(
    user_id UInt32,
    tags    Array(String)
)
ENGINE = MergeTree()
ORDER BY user_id;

INSERT INTO user_tags VALUES
    (1, ['sql', 'olap']),
    (2, []),
    (3, ['clickhouse']);
```

Inner `ARRAY JOIN` - user 2 disappears:

```sql
SELECT user_id, tag
FROM user_tags
ARRAY JOIN tags AS tag;
```

```text
user_id | tag
--------|----------
1       | sql
1       | olap
3       | clickhouse
```

`LEFT ARRAY JOIN` - user 2 is preserved with an empty string:

```sql
SELECT user_id, tag
FROM user_tags
LEFT ARRAY JOIN tags AS tag;
```

```text
user_id | tag
--------|----------
1       | sql
1       | olap
2       |
3       | clickhouse
```

## Practical Use Case - Preserving Users Without Tags

```sql
-- Count events per tag, keeping users who have no tags as a separate group
SELECT
    user_id,
    if(tag = '', '(no tag)', tag) AS tag_label,
    count()                       AS event_count
FROM events
LEFT ARRAY JOIN tags AS tag
GROUP BY user_id, tag_label
ORDER BY user_id, tag_label;
```

## Multiple Arrays with LEFT ARRAY JOIN

Like the inner form, you can expand several arrays simultaneously. All listed arrays are zipped:

```sql
CREATE TABLE scores
(
    player     String,
    rounds     Array(UInt8),
    bonuses    Array(UInt8)
)
ENGINE = MergeTree()
ORDER BY player;

INSERT INTO scores VALUES
    ('Alice', [10, 20, 30], [1, 2, 3]),
    ('Bob',   [],           []);

-- Bob is preserved with NULL/zero values
SELECT
    player,
    round,
    bonus
FROM scores
LEFT ARRAY JOIN
    rounds  AS round,
    bonuses AS bonus;
```

```text
player | round | bonus
-------|-------|------
Alice  | 10    | 1
Alice  | 20    | 2
Alice  | 30    | 3
Bob    | 0     | 0
```

## Using LEFT ARRAY JOIN with arrayEnumerate

```sql
SELECT
    user_id,
    tag,
    pos
FROM user_tags
LEFT ARRAY JOIN
    tags                  AS tag,
    arrayEnumerate(tags)  AS pos;
```

For users with empty arrays, `pos` will be 0.

## Filtering After LEFT ARRAY JOIN

Apply `WHERE` after the join to keep only rows with actual array elements while still distinguishing them from empty-array rows:

```sql
-- Report rows where no tags were found (for data quality checks)
SELECT user_id
FROM user_tags
LEFT ARRAY JOIN tags AS tag
WHERE tag = '';
```

## LEFT ARRAY JOIN with Nested Types

Nested types work the same way - all nested sub-columns are expanded together and an empty Nested produces a single preserved row:

```sql
CREATE TABLE page_events
(
    session_id   String,
    clicks       Nested(ts DateTime, url String)
)
ENGINE = MergeTree()
ORDER BY session_id;

INSERT INTO page_events VALUES
    ('sess-a', [now(), now()+5], ['/', '/about']),
    ('sess-b', [],               []);

SELECT session_id, clicks.ts, clicks.url
FROM page_events
LEFT ARRAY JOIN clicks;
```

## Summary

`LEFT ARRAY JOIN` is the array-unnesting operator to reach for when you must preserve source rows that carry empty or `NULL` arrays. It mirrors the semantics of a SQL `LEFT JOIN`: every input row appears in the output at least once, with default/empty values substituted for missing array elements. Use the plain `ARRAY JOIN` only when you are certain every row has a non-empty array and want the cleaner output.
