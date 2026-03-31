# How to Use argMin() and argMax() Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, argMin, argMax

Description: Learn how to use argMin() and argMax() in ClickHouse to retrieve values associated with minimum or maximum keys, with timestamp-based examples.

---

`argMin(val, key)` and `argMax(val, key)` are two of the most useful aggregate functions in ClickHouse. They return the value of `val` from the row where `key` is at its minimum or maximum. This solves a common problem in SQL - "give me the value of another column from the row with the highest timestamp" - without a self-join, window function, or subquery.

## Syntax and Basic Usage

```sql
argMin(value_column, key_column)
argMax(value_column, key_column)
```

- `value_column` - the column whose value you want to retrieve
- `key_column` - the column to find the min or max of

```sql
CREATE TABLE device_events
(
    device_id   String,
    status      String,
    temperature Float64,
    event_time  DateTime
)
ENGINE = MergeTree()
ORDER BY (event_time, device_id);

INSERT INTO device_events VALUES
    ('dev_001', 'online',  22.1, '2026-03-31 08:00:00'),
    ('dev_001', 'warning', 87.4, '2026-03-31 12:00:00'),
    ('dev_001', 'online',  23.0, '2026-03-31 18:00:00'),
    ('dev_002', 'offline',  0.0, '2026-03-31 09:00:00'),
    ('dev_002', 'online',  20.5, '2026-03-31 15:00:00');

-- Get the status at the earliest and latest event per device
SELECT
    device_id,
    argMin(status, event_time)  AS first_status,
    argMax(status, event_time)  AS latest_status,
    min(event_time)              AS first_seen,
    max(event_time)              AS last_seen
FROM device_events
GROUP BY device_id;
```

## Getting Values at the Maximum Timestamp

The most common use case for `argMax` is retrieving the "current" or "latest" value of a column for each entity - the equivalent of a "last write wins" lookup.

```sql
CREATE TABLE user_profiles
(
    user_id    UInt64,
    name       String,
    email      String,
    updated_at DateTime
)
ENGINE = MergeTree()
ORDER BY (updated_at, user_id);

INSERT INTO user_profiles VALUES
    (1, 'Alice', 'alice@old.com', '2026-01-01 00:00:00'),
    (1, 'Alice', 'alice@new.com', '2026-03-01 00:00:00'),
    (2, 'Bob',   'bob@example.com', '2026-02-15 00:00:00');

-- Latest email per user
SELECT
    user_id,
    argMax(name,       updated_at) AS current_name,
    argMax(email,      updated_at) AS current_email,
    max(updated_at)                AS last_updated
FROM user_profiles
GROUP BY user_id;
```

This pattern is the ClickHouse-idiomatic alternative to `DISTINCT ON` in PostgreSQL or a correlated subquery.

## Getting Values at the Minimum Timestamp

`argMin` retrieves the value from the row with the earliest key - useful for "first event" queries.

```sql
-- What was each device's first recorded status?
SELECT
    device_id,
    argMin(status,     event_time) AS first_status,
    argMin(temperature, event_time) AS temp_at_first_event,
    min(event_time)                 AS first_event_time
FROM device_events
GROUP BY device_id;
```

## Multiple argMin/argMax in One Query

You can call `argMin` and `argMax` multiple times in the same `SELECT` to retrieve several columns from the min-row and max-row simultaneously, all in a single scan.

```sql
-- For each device, get full context of both the first and last events
SELECT
    device_id,
    -- First event details
    argMin(status,      event_time) AS first_status,
    argMin(temperature, event_time) AS first_temp,
    min(event_time)                  AS first_time,
    -- Last event details
    argMax(status,      event_time) AS latest_status,
    argMax(temperature, event_time) AS latest_temp,
    max(event_time)                  AS latest_time
FROM device_events
GROUP BY device_id;
```

## Using argMax with Non-Timestamp Keys

The key column does not have to be a timestamp - any comparable type works.

```sql
CREATE TABLE product_prices
(
    product_id  UInt64,
    store_id    UInt32,
    price       Decimal(10, 2),
    stock       UInt32
)
ENGINE = MergeTree()
ORDER BY (product_id, store_id);

INSERT INTO product_prices VALUES
    (1, 10, 9.99,  100),
    (1, 20, 8.49,   50),
    (1, 30, 10.50, 200),
    (2, 10, 24.99,  10),
    (2, 20, 22.00,   5);

-- For each product, which store has the cheapest price and its stock?
SELECT
    product_id,
    argMin(store_id, price) AS cheapest_store,
    argMin(stock,    price) AS stock_at_cheapest,
    min(price)               AS min_price
FROM product_prices
GROUP BY product_id;
```

## Tie-Breaking Behavior

When multiple rows share the same minimum or maximum key value, `argMin`/`argMax` return the value from one of those tied rows. The specific row chosen is nondeterministic (depends on storage order and parallelism). If tie-breaking matters, add a secondary sort key by composing a tuple:

```sql
-- Break ties on event_time by using (event_time, device_id) as the key
SELECT
    argMax(status, (event_time, device_id)) AS latest_status
FROM device_events
GROUP BY device_id;
```

## argMin/argMax vs Window Functions

For small to medium datasets, a window function approach is familiar from standard SQL. In ClickHouse, `argMax` is generally faster because it avoids materializing a full window frame.

```sql
-- Window function approach (works but less efficient in ClickHouse)
SELECT DISTINCT
    device_id,
    last_value(status) OVER (PARTITION BY device_id ORDER BY event_time
                              ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS latest_status
FROM device_events;

-- ClickHouse-idiomatic approach (faster)
SELECT
    device_id,
    argMax(status, event_time) AS latest_status
FROM device_events
GROUP BY device_id;
```

## Summary

`argMin(val, key)` and `argMax(val, key)` return the value of `val` from the row where `key` is at its minimum or maximum - solving "fetch the sibling column from the extreme row" without joins or subqueries. They are the standard ClickHouse pattern for "latest state per entity" queries using timestamps as keys. For multiple columns, call the function multiple times in the same query to retrieve all needed fields in a single scan.
