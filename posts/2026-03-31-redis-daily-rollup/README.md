# How to Implement Daily Rollup with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Analytics, Time Series, Rollup, Counter

Description: Build daily rollup counters in Redis by aggregating hourly data into day-level summaries with automatic expiry and pipeline-optimized reads.

---

Daily totals are the currency of business metrics: daily active users, daily revenue, daily error counts. Redis daily rollups consolidate 24 hourly buckets into a single day key, enabling fast queries across weeks or months of historical data.

## Key Design for Daily Rollups

Encode the date in the key using ISO format for natural sorting:

```python
import redis
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def day_bucket(ts: float = None) -> str:
    ts = ts or time.time()
    return time.strftime("%Y%m%d", time.gmtime(ts))

def hour_bucket(ts: float = None) -> str:
    ts = ts or time.time()
    return time.strftime("%Y%m%d%H", time.gmtime(ts))
```

## Rolling Up Hours to Days

Sum the 24 hourly buckets for a given day:

```python
def rollup_day(metric: str, day_ts: float) -> int:
    day = day_bucket(day_ts)
    day_key = f"daily:{metric}:{day}"

    if r.exists(day_key):
        return int(r.get(day_key) or 0)

    # Snap to midnight of the target day
    day_start = (day_ts // 86400) * 86400
    pipe = r.pipeline()
    for hour in range(24):
        ts = day_start + hour * 3600
        pipe.get(f"hourly:{metric}:{hour_bucket(ts)}")

    results = pipe.execute()
    total = sum(int(v or 0) for v in results)

    r.set(day_key, total)
    r.expire(day_key, 365 * 86400)  # keep 1 year
    return total
```

## Incremental Daily Counters

For metrics that need real-time daily totals (not just rollups), increment the daily key directly alongside minute/hour keys:

```python
def record_event(metric: str, value: int = 1):
    now = time.time()
    pipe = r.pipeline()

    # Minute counter
    mkey = f"counter:{metric}:{time.strftime('%Y%m%d%H%M', time.gmtime(now))}"
    pipe.incrby(mkey, value)
    pipe.expire(mkey, 7200)

    # Daily counter
    dkey = f"daily:{metric}:{day_bucket(now)}"
    pipe.incrby(dkey, value)
    pipe.expire(dkey, 400 * 86400)

    pipe.execute()
```

## Comparing Day-Over-Day

Redis makes day-over-day comparisons trivial:

```python
def day_over_day(metric: str) -> dict:
    now = time.time()
    today = day_bucket(now)
    yesterday_ts = now - 86400
    yesterday = day_bucket(yesterday_ts)

    pipe = r.pipeline()
    pipe.get(f"daily:{metric}:{today}")
    pipe.get(f"daily:{metric}:{yesterday}")
    today_val, yesterday_val = pipe.execute()

    today_count = int(today_val or 0)
    yesterday_count = int(yesterday_val or 0)
    change_pct = ((today_count - yesterday_count) / max(yesterday_count, 1)) * 100

    return {
        "today": today_count,
        "yesterday": yesterday_count,
        "change_pct": round(change_pct, 1),
    }
```

## Reading a Date Range

Fetch daily data for a custom date range:

```python
def get_daily_range(metric: str, days: int = 30) -> list:
    now = time.time()
    pipe = r.pipeline()
    dates = []
    for i in range(days):
        ts = now - i * 86400
        d = day_bucket(ts)
        pipe.get(f"daily:{metric}:{d}")
        dates.append(time.strftime("%Y-%m-%d", time.gmtime(ts)))

    values = pipe.execute()
    return [
        {"date": d, "count": int(v or 0)}
        for d, v in zip(reversed(dates), reversed(values))
    ]
```

## Summary

Daily Redis rollups provide fast access to day-level business metrics without re-reading raw event data. Rolling up 24 hourly buckets at midnight keeps keys compact, while incremental daily counters give real-time running totals. Pipeline reads let you retrieve 30 days of data in a single round trip.
