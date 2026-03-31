# How to Use array() and Range Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Array, Range Functions, SQL, Data Engineering

Description: Learn how to use array() to create arrays and range() functions to generate numeric sequences in ClickHouse queries.

---

## Overview

ClickHouse has first-class support for arrays. The `array()` function constructs arrays inline, while `range()` generates sequential numeric arrays. Together they enable powerful inline data generation, iteration patterns, and unnesting operations.

## Creating Arrays with array()

`array(val1, val2, ...)` constructs an array from the given values:

```sql
SELECT array(1, 2, 3, 4, 5) AS numbers
-- [1, 2, 3, 4, 5]

SELECT array('apple', 'banana', 'cherry') AS fruits
-- ['apple', 'banana', 'cherry']
```

You can also use the `[...]` bracket syntax as a shorthand:

```sql
SELECT [1, 2, 3] AS numbers
SELECT ['a', 'b', 'c'] AS letters
```

## Generating Sequences with range()

`range(n)` generates an array of integers from 0 to n-1:

```sql
SELECT range(5) AS seq
-- [0, 1, 2, 3, 4]
```

`range(start, end)` generates from start (inclusive) to end (exclusive):

```sql
SELECT range(1, 6) AS seq
-- [1, 2, 3, 4, 5]
```

`range(start, end, step)` adds a step increment:

```sql
SELECT range(0, 20, 5) AS seq
-- [0, 5, 10, 15]

SELECT range(10, 0, -2) AS countdown
-- [10, 8, 6, 4, 2]
```

## Using range() for Date Series

A common pattern is generating a sequence of date offsets:

```sql
SELECT arrayMap(x -> today() - toIntervalDay(x), range(7)) AS last_7_days
-- [today, yesterday, 2 days ago, ..., 6 days ago]
```

This is useful for filling gaps in time series data.

## Unnesting Arrays with ARRAY JOIN

Use `ARRAY JOIN` to expand array elements into rows:

```sql
SELECT number
FROM (SELECT range(1, 6) AS numbers)
ARRAY JOIN numbers AS number
```

Output:

```text
number
1
2
3
4
5
```

## Combining array() with arrayMap()

Transform array values using `arrayMap()`:

```sql
SELECT arrayMap(x -> x * x, range(1, 6)) AS squares
-- [1, 4, 9, 16, 25]

SELECT arrayMap(x -> concat('item_', toString(x)), range(1, 4)) AS labels
-- ['item_1', 'item_2', 'item_3']
```

## Practical Example - Generating Test Data

Use `range()` to generate test rows via `numbers()` table function in combination with arrays:

```sql
SELECT
    number + 1 AS id,
    arrayElement(['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun'], (number % 6) + 1) AS month_name
FROM numbers(6)
```

## Storing Arrays in Tables

Declare columns as Array types in your schema:

```sql
CREATE TABLE product_tags
(
    product_id  UInt64,
    tags        Array(String)
)
ENGINE = MergeTree()
ORDER BY product_id;

INSERT INTO product_tags VALUES
(1, ['electronics', 'portable', 'wireless']),
(2, ['clothing', 'summer']),
(3, ['food', 'organic', 'gluten-free']);
```

Query tags:

```sql
SELECT product_id, arrayJoin(tags) AS tag FROM product_tags;
```

## Array Length and Filtering

```sql
SELECT
    product_id,
    length(tags) AS tag_count
FROM product_tags
WHERE length(tags) > 2
```

## Summary

`array()` and `range()` are fundamental building blocks for array operations in ClickHouse. `array()` constructs literal arrays, `range()` generates integer sequences, and both combine naturally with `arrayMap()`, `ARRAY JOIN`, and other array functions to support time series generation, inline data creation, and flexible analytical patterns.
