# How to Use -If, -Array, -Map Aggregate Combinators in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Aggregate Function, Combinator, SELECT, Performance

Description: Learn how the -If, -Array, and -Map aggregate combinators extend any aggregate function with conditional filtering, array element aggregation, and map value aggregation.

---

ClickHouse aggregate functions are already powerful on their own, but the combinator system makes them dramatically more flexible. A combinator is a suffix you attach to any aggregate function name to modify its behavior - no subqueries or CTEs required. The three most commonly used combinators are `-If`, `-Array`, and `-Map`. `-If` restricts aggregation to rows that satisfy a boolean condition. `-Array` unpacks array columns and applies the aggregate to the individual elements. `-Map` applies the aggregate to the values of a Map column, keyed by map keys. Understanding these three combinators unlocks a whole class of compact, readable queries.

## The -If Combinator

Appending `-If` to any aggregate function name adds a condition parameter. The function only processes rows where the condition evaluates to true (non-zero). The syntax is:

```text
aggFuncIf(column, condition)
```

This is equivalent to wrapping the column in a `CASE WHEN` expression, but more concise and often faster because ClickHouse can push the filter down into the aggregation kernel.

### Setting Up Sample Data

```sql
CREATE TABLE orders
(
    order_id   UInt32,
    customer   String,
    region     String,
    amount     Float64,
    status     String,
    created_at Date
)
ENGINE = MergeTree()
ORDER BY (created_at, customer);

INSERT INTO orders VALUES
    (1, 'alice', 'us-east', 120.0, 'completed', '2026-03-01'),
    (2, 'alice', 'us-east', 45.0,  'refunded',  '2026-03-05'),
    (3, 'bob',   'eu-west', 300.0, 'completed', '2026-03-10'),
    (4, 'bob',   'eu-west', 80.0,  'pending',   '2026-03-15'),
    (5, 'carol', 'us-east', 200.0, 'completed', '2026-03-20'),
    (6, 'carol', 'eu-west', 60.0,  'refunded',  '2026-03-25');
```

### Using sumIf() and countIf()

Compute the total revenue from completed orders and the count of refunded orders in a single pass over the data.

```sql
SELECT
    region,
    sumIf(amount, status = 'completed')  AS completed_revenue,
    countIf(status = 'refunded')         AS refund_count,
    avgIf(amount, status = 'completed')  AS avg_completed_order
FROM orders
GROUP BY region
ORDER BY region;
```

```text
region   completed_revenue  refund_count  avg_completed_order
eu-west  300                1             300
us-east  320                1             160
```

Without `-If` you would need multiple subqueries or a pivot-style CASE expression. With `-If`, everything lives in one aggregation step.

### Combining Multiple -If Conditions

You can use multiple `*If` columns in the same SELECT to compute several conditional aggregations simultaneously.

```sql
SELECT
    customer,
    sumIf(amount, status = 'completed')   AS total_spent,
    sumIf(amount, status = 'refunded')    AS total_refunded,
    countIf(status = 'completed')         AS completed_orders,
    countIf(status = 'refunded')          AS refunded_orders
FROM orders
GROUP BY customer
ORDER BY customer;
```

## The -Array Combinator

Appending `-Array` makes an aggregate function operate on array columns by flattening all array elements across rows and aggregating them together. The syntax is:

```text
aggFuncArray(array_column)
```

### Setting Up Array Sample Data

```sql
CREATE TABLE page_views
(
    session_id  String,
    page        String,
    durations   Array(UInt32)   -- time spent on each element of the page in ms
)
ENGINE = MergeTree()
ORDER BY session_id;

INSERT INTO page_views VALUES
    ('s1', '/home',    [1200, 800, 950]),
    ('s2', '/home',    [2000, 1500]),
    ('s3', '/about',   [300]),
    ('s4', '/about',   [450, 600, 200]);
```

### Using sumArray() and avgArray()

Sum all individual duration values across every row and page, treating each array element as a separate data point.

```sql
SELECT
    page,
    sumArray(durations)              AS total_time_ms,
    avgArray(durations)              AS avg_element_ms,
    countArray(durations)            AS total_elements
FROM page_views
GROUP BY page
ORDER BY page;
```

```text
page    total_time_ms  avg_element_ms  total_elements
/about  1550           387.5           4
/home   6450           1290            5
```

`avgArray` computes the mean over every individual array element across all rows in the group, not the mean of row-level sums.

### Nesting -Array with -If: -ArrayIf

Combinators can be stacked. `-ArrayIf` flattens arrays and applies a per-row condition.

```sql
SELECT
    sumArrayIf(durations, page = '/home') AS home_total_ms
FROM page_views;
```

```text
home_total_ms
6450
```

## The -Map Combinator

Appending `-Map` makes an aggregate function work on `Map(K, V)` columns. It aggregates the values for each key across all rows, returning a Map where each key maps to the aggregated result. The syntax is:

```text
aggFuncMap(map_column)
```

### Setting Up Map Sample Data

```sql
CREATE TABLE feature_usage
(
    user_id  UInt32,
    date     Date,
    clicks   Map(String, UInt32)   -- feature name -> click count
)
ENGINE = MergeTree()
ORDER BY (user_id, date);

INSERT INTO feature_usage VALUES
    (1, '2026-03-01', {'search': 5, 'export': 2}),
    (1, '2026-03-02', {'search': 3, 'import': 1}),
    (2, '2026-03-01', {'export': 4, 'search': 7}),
    (2, '2026-03-02', {'import': 2, 'search': 1});
```

### Using sumMap()

`sumMap()` sums the values for each key across all rows in the group.

```sql
SELECT
    user_id,
    sumMap(clicks) AS total_clicks_by_feature
FROM feature_usage
GROUP BY user_id
ORDER BY user_id;
```

```text
user_id  total_clicks_by_feature
1        {'export':2,'import':1,'search':8}
2        {'export':4,'import':2,'search':8}
```

### Global Feature Totals with sumMap()

Aggregate across all users to see which features are most used overall.

```sql
SELECT
    sumMap(clicks) AS global_feature_clicks
FROM feature_usage;
```

```text
global_feature_clicks
{'export':6,'import':3,'search':16}
```

### minMap() and maxMap()

`minMap()` and `maxMap()` work the same way, returning the minimum or maximum value per key.

```sql
SELECT
    minMap(clicks) AS min_clicks_per_feature,
    maxMap(clicks) AS max_clicks_per_feature
FROM feature_usage;
```

```text
min_clicks_per_feature           max_clicks_per_feature
{'export':2,'import':1,'search':1}  {'export':4,'import':2,'search':7}
```

## Summary

The `-If`, `-Array`, and `-Map` combinators extend every ClickHouse aggregate function without requiring additional subqueries or joins. Use `-If` to conditionally filter rows inside an aggregation (e.g., `sumIf`, `countIf`, `avgIf`). Use `-Array` to flatten array columns and aggregate their elements individually (e.g., `sumArray`, `avgArray`). Use `-Map` to aggregate across map values keyed by their map keys (e.g., `sumMap`, `minMap`, `maxMap`). All three combinators compose with each other and with other combinators like `-State` or `-Distinct`, giving you a highly expressive aggregation vocabulary within a single SELECT statement.
