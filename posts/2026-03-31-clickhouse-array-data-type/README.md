# How to Use Array Data Type in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Data Type, Array

Description: Learn how to store and query arrays in ClickHouse using Array(T), built-in array functions, and ARRAY JOIN for row-level expansion.

---

The `Array(T)` type stores a variable-length list of elements of type `T` within a single column value. Arrays are useful for representing tags, event sequences, multi-valued attributes, and other one-to-many relationships without requiring a separate join table. ClickHouse provides a rich set of array functions and the `ARRAY JOIN` clause for working with array data efficiently.

## Creating Array Columns

Declare an array column using `Array(T)` where `T` is any ClickHouse type. Literal arrays use square bracket syntax.

```sql
CREATE TABLE articles (
    article_id UInt64,
    title      String,
    tags       Array(String),
    scores     Array(Float32),
    published_at DateTime
) ENGINE = MergeTree()
ORDER BY article_id;

INSERT INTO articles VALUES
    (1, 'ClickHouse Basics',  ['clickhouse', 'sql', 'database'], [4.5, 4.7, 4.2], '2026-01-10 09:00:00'),
    (2, 'Array Deep Dive',    ['clickhouse', 'arrays'],           [4.9, 4.8],       '2026-02-14 10:00:00'),
    (3, 'Performance Tuning', ['clickhouse', 'performance'],      [4.6],            '2026-03-01 11:00:00');
```

## Common Array Functions

ClickHouse includes dozens of built-in array functions. The most frequently used are shown below.

```sql
-- arrayLength: number of elements
SELECT title, arrayLength(tags) AS tag_count FROM articles;

-- has: check if array contains a value
SELECT title FROM articles WHERE has(tags, 'performance');

-- hasAny: check if array contains any of the given values
SELECT title FROM articles WHERE hasAny(tags, ['sql', 'arrays']);

-- arrayElement / subscript: access by 1-based index
SELECT title, tags[1] AS first_tag FROM articles;

-- indexOf: find position of element (0 if not found)
SELECT title, indexOf(tags, 'clickhouse') AS pos FROM articles;

-- arraySort and arrayReverseSort
SELECT arraySort(tags) AS sorted_tags FROM articles WHERE article_id = 1;

-- arrayUniq: count distinct elements
SELECT arrayUniq(tags) AS distinct_tag_count FROM articles WHERE article_id = 1;
```

## Transforming Arrays with arrayMap and arrayFilter

Higher-order array functions accept lambda expressions for element-wise transformations and filtering.

```sql
-- arrayMap: apply a function to each element
SELECT arrayMap(x -> upper(x), tags) AS upper_tags FROM articles;

-- arrayFilter: keep only elements matching a predicate
SELECT arrayFilter(x -> length(x) > 5, tags) AS long_tags FROM articles;

-- arrayReduce: apply an aggregate function over array elements
SELECT arrayReduce('avg', scores) AS avg_score FROM articles;

-- Combine: count tags with length > 5
SELECT
    title,
    length(arrayFilter(x -> length(x) > 5, tags)) AS long_tag_count
FROM articles;
```

## ARRAY JOIN - Expanding Arrays into Rows

`ARRAY JOIN` flattens an array column so that each element becomes its own row, equivalent to an UNNEST in other databases.

```sql
-- Expand tags into individual rows
SELECT article_id, title, tag
FROM articles
ARRAY JOIN tags AS tag;

-- ARRAY JOIN with filtering
SELECT article_id, title, tag
FROM articles
ARRAY JOIN tags AS tag
WHERE tag LIKE 'click%';

-- LEFT ARRAY JOIN preserves rows with empty arrays
SELECT article_id, title, tag
FROM articles
LEFT ARRAY JOIN tags AS tag;
```

## Aggregating into Arrays

Use `groupArray` and related aggregate functions to build arrays from grouped rows.

```sql
-- Collect all tags used per publication month
SELECT
    toYYYYMM(published_at) AS month,
    groupArray(title) AS article_titles,
    arrayFlatten(groupArray(tags)) AS all_tags
FROM articles
GROUP BY month
ORDER BY month;

-- Top 3 tags overall
SELECT tag, count() AS usage
FROM articles
ARRAY JOIN tags AS tag
GROUP BY tag
ORDER BY usage DESC
LIMIT 3;
```

## Nested Arrays

Arrays can contain other arrays, creating `Array(Array(T))` structures.

```sql
CREATE TABLE matrix_data (
    id     UInt32,
    matrix Array(Array(Int32))
) ENGINE = Memory;

INSERT INTO matrix_data VALUES
    (1, [[1, 2, 3], [4, 5, 6], [7, 8, 9]]);

-- Access a nested element (row 2, col 3)
SELECT matrix[2][3] AS element FROM matrix_data WHERE id = 1;

-- Flatten the nested array
SELECT arrayFlatten(matrix) AS flat FROM matrix_data WHERE id = 1;
```

## Summary

`Array(T)` in ClickHouse provides a flexible way to store multi-valued data within a single column. Functions like `has`, `arrayMap`, `arrayFilter`, and `arrayReduce` enable powerful in-place transformations, while `ARRAY JOIN` expands arrays into rows for row-level analysis. Use `groupArray` to aggregate result sets back into arrays when building reports or materialized summaries.
