# How to Use FIRST_VALUE() and LAST_VALUE() Window Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Window Function, SQL, Analytics, Time Series

Description: Learn how FIRST_VALUE() and LAST_VALUE() retrieve boundary values of a window frame in ClickHouse, with examples for first-touch attribution, last known price, and session analysis.

---

`FIRST_VALUE()` and `LAST_VALUE()` are window functions that return the value from the first or last row of the current window frame, respectively. They are different from `MIN()` and `MAX()` - they return the value at a positional boundary determined by the `ORDER BY` clause, not the mathematically smallest or largest value. This distinction becomes important with non-numeric columns or when you need the value that appeared first or last in time. ClickHouse supports both functions with full frame specification.

## Syntax

```text
FIRST_VALUE(expr) OVER (
    [PARTITION BY partition_expr [, ...]]
    ORDER BY sort_expr [ASC | DESC] [, ...]
    [ROWS BETWEEN frame_start AND frame_end]
)

LAST_VALUE(expr) OVER (
    [PARTITION BY partition_expr [, ...]]
    ORDER BY sort_expr [ASC | DESC] [, ...]
    [ROWS BETWEEN frame_start AND frame_end]
)
```

Common frame boundaries:

```text
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW    -- default in most engines
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING  -- entire partition
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW            -- rolling 3-row window
```

## Why Frame Specification Matters for LAST_VALUE()

`FIRST_VALUE()` is straightforward - it always returns the first value in the partition regardless of the frame, as long as the frame starts at `UNBOUNDED PRECEDING`. `LAST_VALUE()` is trickier: with the default frame (`UNBOUNDED PRECEDING AND CURRENT ROW`), it returns the current row's value for the current row - not the last value of the partition.

To get the truly last value of the partition, you must expand the frame:

```sql
-- Returns the current row's value for each row - probably NOT what you want
SELECT
    event_id,
    event_time,
    LAST_VALUE(event_time) OVER (
        PARTITION BY session_id
        ORDER BY event_time ASC
        -- default frame: UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS last_event_time_wrong,

    -- Returns the partition's last value for every row - correct
    LAST_VALUE(event_time) OVER (
        PARTITION BY session_id
        ORDER BY event_time ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_event_time_correct
FROM session_events
ORDER BY session_id, event_time;
```

## First and Last Event Per Session

A common use case is tagging every event in a session with the session's first and last event details:

```sql
SELECT
    session_id,
    event_time,
    event_type,
    page_url,
    FIRST_VALUE(page_url) OVER (
        PARTITION BY session_id
        ORDER BY event_time ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS entry_page,
    LAST_VALUE(page_url) OVER (
        PARTITION BY session_id
        ORDER BY event_time ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS exit_page
FROM session_events
ORDER BY session_id, event_time;
```

`entry_page` and `exit_page` are the same for every row within a session, making it easy to group and count entry/exit page combinations.

## First-Touch Attribution

Marketing attribution often requires knowing which channel first brought a user to the site. `FIRST_VALUE()` with `PARTITION BY user_id` retrieves the first-touch channel for every session of a returning user:

```sql
SELECT
    user_id,
    session_id,
    session_start,
    channel,
    FIRST_VALUE(channel) OVER (
        PARTITION BY user_id
        ORDER BY session_start ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS first_touch_channel
FROM user_sessions
ORDER BY user_id, session_start;
```

Every session row for a user carries the same `first_touch_channel` - the channel from the user's very first session.

## Last Known Price (Carry-Forward Pattern)

In financial or inventory data, prices are recorded only when they change. `LAST_VALUE()` can carry the most recent known price forward across all rows up to the current one. Use the default frame (current row and all preceding):

```sql
SELECT
    product_id,
    event_date,
    price_change,
    LAST_VALUE(price_change) OVER (
        PARTITION BY product_id
        ORDER BY event_date ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS current_price
FROM product_price_events
ORDER BY product_id, event_date;
```

Each row sees the most recently recorded price change up to and including that date, effectively implementing a last-observation-carried-forward (LOCF) fill.

## Rolling First and Last in a Moving Window

Specify a bounded frame to get the first and last values within a sliding window. This finds the opening and closing price within a rolling 5-row window:

```sql
SELECT
    trade_date,
    closing_price,
    FIRST_VALUE(closing_price) OVER (
        ORDER BY trade_date ASC
        ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
    ) AS window_open,
    LAST_VALUE(closing_price) OVER (
        ORDER BY trade_date ASC
        ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
    ) AS window_close,
    MAX(closing_price) OVER (
        ORDER BY trade_date ASC
        ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
    ) AS window_high,
    MIN(closing_price) OVER (
        ORDER BY trade_date ASC
        ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
    ) AS window_low
FROM stock_prices
WHERE ticker = 'ACME'
ORDER BY trade_date;
```

This produces OHLC (Open-High-Low-Close) data for a 5-day rolling window without any self-joins.

## Comparing First and Last to Compute Change

A concise way to compute the total change across a partition is to subtract `FIRST_VALUE` from `LAST_VALUE`:

```sql
SELECT
    user_id,
    measurement_date,
    metric_value,
    FIRST_VALUE(metric_value) OVER (
        PARTITION BY user_id
        ORDER BY measurement_date ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS baseline_value,
    LAST_VALUE(metric_value) OVER (
        PARTITION BY user_id
        ORDER BY measurement_date ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS latest_value,
    LAST_VALUE(metric_value) OVER (
        PARTITION BY user_id
        ORDER BY measurement_date ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) - FIRST_VALUE(metric_value) OVER (
        PARTITION BY user_id
        ORDER BY measurement_date ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS total_change
FROM user_health_metrics
ORDER BY user_id, measurement_date;
```

## Using the WINDOW Clause to Avoid Repetition

When the same window specification is used multiple times in the same query, the `WINDOW` clause avoids repetition:

```sql
SELECT
    session_id,
    event_time,
    event_type,
    FIRST_VALUE(event_type) OVER w AS first_event,
    LAST_VALUE(event_type)  OVER w AS last_event
FROM session_events
WINDOW w AS (
    PARTITION BY session_id
    ORDER BY event_time ASC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
ORDER BY session_id, event_time;
```

## Summary

`FIRST_VALUE()` and `LAST_VALUE()` return the values at the positional boundaries of a window frame. The most important practical detail is frame specification: `LAST_VALUE()` with the default frame returns the current row's own value, not the partition's last value. Always use `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` when you want the true last row of the partition. These functions are especially powerful for session analysis (entry and exit pages), attribution modeling (first-touch channel), carry-forward fills (last known price), and rolling OHLC-style aggregations.
