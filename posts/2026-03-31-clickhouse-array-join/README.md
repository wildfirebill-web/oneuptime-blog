# How to Use ARRAY JOIN in ClickHouse to Unnest Arrays

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, SELECT, ARRAY JOIN, Array, UNNEST

Description: Use ARRAY JOIN in ClickHouse to flatten array columns into individual rows, similar to UNNEST in other databases, with support for multiple arrays and expressions.

---

ClickHouse stores arrays as first-class column types, and `ARRAY JOIN` is the mechanism for expanding them - turning one row with an array of N elements into N rows, one per element. It is conceptually similar to `UNNEST` in PostgreSQL or `LATERAL VIEW explode()` in Hive, but tightly integrated into ClickHouse's query syntax.

## Basic Syntax

`ARRAY JOIN` appears after the `FROM` clause and before `WHERE`:

```sql
SELECT
    user_id,
    tag
FROM user_profiles
ARRAY JOIN tags AS tag;
```

If the table has a row `(user_id=1, tags=['sports','tech','music'])`, the query returns three rows - one for each tag.

## Simple Array Join Example

```sql
-- Create a sample table
CREATE TABLE orders
(
    order_id  UInt32,
    customer  String,
    items     Array(String)
)
ENGINE = MergeTree()
ORDER BY order_id;

INSERT INTO orders VALUES
    (1, 'Alice', ['laptop', 'mouse', 'keyboard']),
    (2, 'Bob',   ['monitor']),
    (3, 'Carol', ['desk', 'chair']);

-- Unnest the items array
SELECT
    order_id,
    customer,
    item
FROM orders
ARRAY JOIN items AS item;
```

Result:

```text
order_id | customer | item
---------|----------|----------
1        | Alice    | laptop
1        | Alice    | mouse
1        | Alice    | keyboard
2        | Bob      | monitor
3        | Carol    | desk
3        | Carol    | chair
```

## Array Join with Index

Use `arrayEnumerate` to also retrieve the 1-based position of each element:

```sql
SELECT
    order_id,
    item,
    item_index
FROM orders
ARRAY JOIN
    items     AS item,
    arrayEnumerate(items) AS item_index;
```

## Joining Multiple Arrays Simultaneously

When you list multiple arrays in a single `ARRAY JOIN`, they are zipped together (not cross-joined). Both arrays must have the same length per row:

```sql
CREATE TABLE product_scores
(
    product_id UInt32,
    reviewers  Array(String),
    scores     Array(UInt8)
)
ENGINE = MergeTree()
ORDER BY product_id;

INSERT INTO product_scores VALUES
    (42, ['alice', 'bob', 'carol'], [5, 3, 4]);

-- Zip reviewers and scores side by side
SELECT
    product_id,
    reviewer,
    score
FROM product_scores
ARRAY JOIN
    reviewers AS reviewer,
    scores    AS score;
```

Result:

```text
product_id | reviewer | score
-----------|----------|------
42         | alice    | 5
42         | bob      | 3
42         | carol    | 4
```

## Array Join with Expressions

You can apply array functions inside `ARRAY JOIN` rather than referencing a stored column directly:

```sql
SELECT
    event_id,
    upper(tag) AS tag_upper
FROM events
ARRAY JOIN arrayDistinct(tags) AS tag;

-- Split a comma-delimited string into rows
SELECT
    id,
    token
FROM raw_data
ARRAY JOIN splitByChar(',', csv_field) AS token;
```

## Nested Array Join

For `Nested` data types, `ARRAY JOIN` expands all nested columns in sync (they share the same implicit array length):

```sql
CREATE TABLE metrics
(
    host         String,
    ts_values    Nested(ts DateTime, val Float64)
)
ENGINE = MergeTree()
ORDER BY host;

INSERT INTO metrics VALUES
    ('srv-1', [now(), now() + 60], [12.5, 14.0]);

-- Nested columns are expanded together
SELECT
    host,
    ts,
    val
FROM metrics
ARRAY JOIN ts_values;
```

## Array Join Combined with WHERE and GROUP BY

```sql
-- Count how many orders contain each item category
SELECT
    item,
    count() AS order_count
FROM orders
ARRAY JOIN items AS item
WHERE item != ''
GROUP BY item
ORDER BY order_count DESC;
```

## Summary

`ARRAY JOIN` is ClickHouse's native array-unnesting operator. A single `ARRAY JOIN` clause can expand one or more arrays in parallel (zipped, not cross-joined) and supports arbitrary array expressions on the right-hand side. For rows with empty arrays, use `LEFT ARRAY JOIN` (covered in a separate post) to preserve those rows rather than dropping them.
