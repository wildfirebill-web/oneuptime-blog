# How to Build a Sales Leaderboard Dashboard with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Sales, Leaderboard, Dashboard, Real-Time

Description: Build a real-time sales leaderboard dashboard with Redis that tracks rep performance, quotas, and rankings with instant updates.

---

Sales teams thrive on real-time visibility into their rankings. A Redis-backed sales leaderboard updates instantly on each deal close, motivating reps and giving managers live pipeline visibility.

## Recording Sales

```python
import redis
import time
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def record_sale(rep_id: str, amount: float, deal_id: str,
                product_line: str = "general"):
    now = time.time()
    today = time.strftime("%Y-%m-%d")
    month = time.strftime("%Y-%m")

    pipe = r.pipeline()
    # Daily and monthly revenue leaderboards
    pipe.zincrby(f"sales:daily:{today}", amount, rep_id)
    pipe.zincrby(f"sales:monthly:{month}", amount, rep_id)
    pipe.zincrby(f"sales:product:{product_line}:{month}", amount, rep_id)

    # Per-rep stats hash
    pipe.hincrbyfloat(f"rep:stats:{rep_id}", "total_revenue", amount)
    pipe.hincrby(f"rep:stats:{rep_id}", "deal_count", 1)

    # Deal log
    deal = {"rep": rep_id, "amount": amount, "id": deal_id, "ts": now}
    pipe.lpush("sales:recent_deals", json.dumps(deal))
    pipe.ltrim("sales:recent_deals", 0, 99)

    # TTLs
    pipe.expire(f"sales:daily:{today}", 86400 * 7)
    pipe.expire(f"sales:monthly:{month}", 86400 * 90)
    pipe.execute()
```

## Dashboard Data

```python
def get_monthly_leaderboard(n: int = 20) -> list:
    month = time.strftime("%Y-%m")
    entries = r.zrevrange(f"sales:monthly:{month}", 0, n - 1, withscores=True)
    return [
        {"rank": i + 1, "rep": rep, "revenue": round(rev, 2)}
        for i, (rep, rev) in enumerate(entries)
    ]

def get_rep_dashboard(rep_id: str) -> dict:
    month = time.strftime("%Y-%m")
    today = time.strftime("%Y-%m-%d")

    pipe = r.pipeline()
    pipe.hgetall(f"rep:stats:{rep_id}")
    pipe.zscore(f"sales:monthly:{month}", rep_id)
    pipe.zrevrank(f"sales:monthly:{month}", rep_id)
    pipe.zscore(f"sales:daily:{today}", rep_id)
    stats, monthly_rev, monthly_rank, daily_rev = pipe.execute()

    return {
        "rep": rep_id,
        "monthly_revenue": float(monthly_rev or 0),
        "monthly_rank": (monthly_rank + 1) if monthly_rank is not None else None,
        "today_revenue": float(daily_rev or 0),
        "total_deals": int(stats.get("deal_count", 0)),
    }
```

## Quota Tracking

```python
def set_quota(rep_id: str, monthly_quota: float):
    month = time.strftime("%Y-%m")
    r.hset(f"quota:{rep_id}", month, monthly_quota)

def get_quota_attainment(rep_id: str) -> dict:
    month = time.strftime("%Y-%m")
    quota = float(r.hget(f"quota:{rep_id}", month) or 0)
    current = float(r.zscore(f"sales:monthly:{month}", rep_id) or 0)
    return {
        "quota": quota,
        "achieved": current,
        "attainment_pct": round(current / quota * 100, 1) if quota > 0 else 0,
    }
```

## Recent Deals Feed

```python
def get_recent_deals(n: int = 10) -> list:
    raw = r.lrange("sales:recent_deals", 0, n - 1)
    return [json.loads(d) for d in raw]
```

## Monitoring

Use [OneUptime](https://oneuptime.com) to monitor your CRM integration webhook - if deal-close events stop arriving, the leaderboard goes stale and reps lose motivation.

## Summary

Redis Sorted Sets with daily and monthly keys provide instant revenue rankings that update on each deal. Per-rep Hash stats track cumulative totals, deal counts, and enable quota attainment calculations. A recent deals List creates a live activity feed for the dashboard's news ticker section.
