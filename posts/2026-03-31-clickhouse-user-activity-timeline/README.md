# How to Build a User Activity Timeline in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, User Activity, Timeline, Session, Analytics

Description: Learn how to build a user activity timeline in ClickHouse that reconstructs event sequences, session boundaries, and activity patterns for individual users.

---

## User Activity Timelines

A user activity timeline shows every action a user took, in order, across sessions. It powers customer support investigation, fraud detection, and UX analysis. ClickHouse's window functions and array aggregations make timeline reconstruction fast.

## Basic Event Timeline

Fetch all events for a user sorted by time:

```sql
SELECT
    event_time,
    event_type,
    page_url,
    properties
FROM user_events
WHERE user_id = 12345
  AND event_time >= today() - 30
ORDER BY event_time;
```

## Adding Time Between Events

Calculate the gap since the previous event:

```sql
SELECT
    event_time,
    event_type,
    dateDiff(
        'second',
        lag(event_time) OVER (PARTITION BY user_id ORDER BY event_time),
        event_time
    ) AS seconds_since_prev
FROM user_events
WHERE user_id = 12345
ORDER BY event_time;
```

## Session Segmentation

Assign session IDs by detecting gaps larger than 30 minutes:

```sql
SELECT
    event_time,
    event_type,
    sum(is_new_session) OVER (PARTITION BY user_id ORDER BY event_time) AS session_id
FROM (
    SELECT
        user_id,
        event_time,
        event_type,
        if(
            dateDiff('minute',
                lag(event_time, 1, event_time - INTERVAL 31 MINUTE) OVER (PARTITION BY user_id ORDER BY event_time),
                event_time
            ) > 30,
            1, 0
        ) AS is_new_session
    FROM user_events
    WHERE user_id = 12345
)
ORDER BY event_time;
```

## Activity Summary per Day

Aggregate a user's daily activity pattern:

```sql
SELECT
    toDate(event_time) AS day,
    count() AS events,
    groupArray(event_type) AS event_types,
    min(event_time) AS first_event,
    max(event_time) AS last_event,
    dateDiff('minute', min(event_time), max(event_time)) AS active_minutes
FROM user_events
WHERE user_id = 12345
  AND event_time >= today() - 90
GROUP BY day
ORDER BY day;
```

## Event Sequence as Array

Compress a session's event sequence into an ordered array:

```sql
SELECT
    session_id,
    arrayStringConcat(
        arrayMap(e -> e, groupArray(event_type)),
        ' -> '
    ) AS event_sequence
FROM (
    SELECT
        user_id,
        session_id,
        event_time,
        event_type
    FROM (
        SELECT *, sum(is_new) OVER (PARTITION BY user_id ORDER BY event_time) AS session_id
        FROM (
            SELECT *,
                if(dateDiff('minute', lag(event_time) OVER (PARTITION BY user_id ORDER BY event_time), event_time) > 30, 1, 0) AS is_new
            FROM user_events WHERE user_id = 12345
        )
    )
)
GROUP BY session_id
ORDER BY min(event_time);
```

## Summary

ClickHouse builds user activity timelines using `lag()` for gap calculation, cumulative sums of session-break flags for session assignment, and `groupArray` to compress event sequences. These patterns support customer support tooling, fraud investigation, and UX analysis without round-tripping through application code.
