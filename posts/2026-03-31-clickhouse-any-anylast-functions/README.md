# How to Use any() and anyLast() Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, Any, AnyLast

Description: Learn how to use any() and anyLast() in ClickHouse to retrieve first and last seen values, including anyHeavy(), nondeterminism, and practical use cases.

---

`any()` and `anyLast()` are ClickHouse-specific aggregate functions that return an arbitrary value from a group - `any()` picks the first value encountered during processing, and `anyLast()` picks the last. Because ClickHouse processes data in parallel across threads and parts, the exact row returned is not guaranteed to be stable across query runs. Knowing when that nondeterminism is acceptable - and when it is not - is the key to using these functions correctly.

## any() - First Seen Value

`any(column)` returns the first value from the column that ClickHouse processes for each group. It is faster than `min()` or `max()` because it stops looking once it finds one value.

```sql
CREATE TABLE user_sessions
(
    session_id  String,
    user_id     UInt64,
    page        String,
    country     String,
    started_at  DateTime
)
ENGINE = MergeTree()
ORDER BY (started_at, user_id);

-- Get any country for each user (e.g., for denormalization)
SELECT
    user_id,
    any(country)    AS sample_country,
    count()          AS session_count
FROM user_sessions
GROUP BY user_id;
```

`any()` is appropriate when you know all values in the group are the same (e.g., a foreign key attribute that does not change per user), or when you just need one example value for a report and exact row selection does not matter.

## anyLast() - Last Seen Value

`anyLast(column)` returns the last value processed in the group. For MergeTree tables ordered by time, this often (but not always) corresponds to the most recent row - however, this is not guaranteed without an explicit `ORDER BY` in the query.

```sql
-- Get the most recently processed page per session
-- Note: "last" here is last in storage order, not necessarily by started_at
SELECT
    user_id,
    anyLast(page)       AS last_page_seen,
    anyLast(country)    AS last_country_seen
FROM user_sessions
GROUP BY user_id;
```

To reliably get the value from the latest row by time, use `argMax()` instead:

```sql
-- Reliable: get page from the row with the maximum started_at
SELECT
    user_id,
    argMax(page, started_at) AS latest_page
FROM user_sessions
GROUP BY user_id;
```

## Nondeterminism

Because ClickHouse reads data in parallel across parts and threads, the value returned by `any()` and `anyLast()` can change between query executions or after a merge operation. Do not rely on them for reproducible, exact results.

Safe uses:
- Picking one representative value when all rows in the group have the same value
- Sampling or exploratory queries where any example is acceptable
- Optimizing GROUP BY queries where only one non-key column is needed and the exact value does not matter

Unsafe uses:
- "Latest record" logic (use `argMax` instead)
- "First event" logic (use `argMin` instead)
- Auditable or financial reporting where exact row selection must be deterministic

## anyHeavy() - Most Frequent Value (Approximate)

`anyHeavy(column)` returns a value that appears in more than half of all rows - the "heavy hitter". It uses a streaming algorithm and is approximate: if no value exceeds the 50% threshold, the result is undefined.

```sql
-- Find the dominant country (if one country has more than 50% of sessions)
SELECT anyHeavy(country) AS dominant_country
FROM user_sessions;
```

This is useful as a fast approximation of mode (most common value) without a full `GROUP BY` + `ORDER BY` + `LIMIT 1`.

```sql
-- Compare anyHeavy vs exact mode
SELECT
    anyHeavy(page)                 AS approx_most_common_page,
    topK(1)(page)[1]               AS exact_most_common_page
FROM user_sessions;
```

## Practical Use Cases

### Denormalization without JOIN

When a dimension attribute is the same for all rows in a group, `any()` lets you select it without a JOIN:

```sql
CREATE TABLE events
(
    event_id    UInt64,
    user_id     UInt64,
    user_email  String,  -- same for all rows of a given user_id
    action      String,
    event_time  DateTime
)
ENGINE = MergeTree()
ORDER BY (event_time, user_id);

-- No need to JOIN a users table - email is consistent per user_id
SELECT
    user_id,
    any(user_email)  AS email,
    count()           AS event_count
FROM events
GROUP BY user_id;
```

### Fast "any example" sampling

```sql
-- Pick one example session per country for debugging
SELECT
    country,
    any(session_id)   AS example_session,
    count()            AS total_sessions
FROM user_sessions
GROUP BY country;
```

### anyLast() for approximate "current state"

In insert-ordered tables where newer rows are appended at the end, `anyLast()` can approximate the latest known state cheaply:

```sql
-- Approximate latest status per user (no guarantee on ordering)
SELECT
    user_id,
    anyLast(country) AS approx_latest_country
FROM user_sessions
GROUP BY user_id;
```

## Summary

`any()` returns the first value encountered for a group and `anyLast()` returns the last - both are nondeterministic across query runs due to parallel execution. Use them when all group values are identical or when any example suffices. For truly first or last values by a defined key, use `argMin(val, key)` or `argMax(val, key)`. `anyHeavy()` provides a fast approximate mode when you need the most dominant value in a distribution.
