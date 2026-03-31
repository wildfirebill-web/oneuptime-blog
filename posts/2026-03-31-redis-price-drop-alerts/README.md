# How to Implement Price Drop Alerts with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, E-commerce, Sorted Set, Alert

Description: Build a price drop alert system with Redis - track user price thresholds per product, detect drops when prices update, and queue notifications for triggered alerts.

---

Price drop alerts let customers set a target price for a product and get notified when it drops to that level. Redis Sorted Sets store per-product alert thresholds for efficient range queries.

## Data Model

```text
alerts:{productId}           -> Sorted Set: userId -> target price
user:alerts:{userId}         -> Set of productIds the user is watching
product:price:{productId}    -> String: current price
alert:notifications          -> List: pending notification payloads
```

## Registering a Price Alert

```python
import redis
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def set_price_alert(user_id, product_id, target_price):
    pipe = r.pipeline()
    # Store target price as score in sorted set
    pipe.zadd(f"alerts:{product_id}", {user_id: target_price})
    # Track which products this user watches
    pipe.sadd(f"user:alerts:{user_id}", product_id)
    pipe.execute()

def remove_price_alert(user_id, product_id):
    pipe = r.pipeline()
    pipe.zrem(f"alerts:{product_id}", user_id)
    pipe.srem(f"user:alerts:{user_id}", product_id)
    pipe.execute()

def get_user_alerts(user_id):
    product_ids = r.smembers(f"user:alerts:{user_id}")
    alerts = {}
    for pid in product_ids:
        score = r.zscore(f"alerts:{pid}", user_id)
        if score is not None:
            alerts[pid] = score
    return alerts
```

## Detecting and Triggering Alerts on Price Update

When a product price changes, query the Sorted Set for users whose target price is now met or exceeded:

```python
def update_product_price(product_id, new_price):
    old_price = r.get(f"product:price:{product_id}")
    r.set(f"product:price:{product_id}", str(new_price))

    if old_price and float(old_price) > new_price:
        # Price dropped - find users whose target is >= new_price
        triggered_users = r.zrangebyscore(
            f"alerts:{product_id}", new_price, float('inf')
        )
        for user_id in triggered_users:
            target = r.zscore(f"alerts:{product_id}", user_id)
            notification = json.dumps({
                "user_id": user_id,
                "product_id": product_id,
                "new_price": new_price,
                "target_price": target,
            })
            r.rpush("alert:notifications", notification)
            # Remove the alert after triggering
            remove_price_alert(user_id, product_id)
```

## Processing Notification Queue

```python
import time

def process_notifications(send_fn, batch_size=50):
    while True:
        items = []
        for _ in range(batch_size):
            item = r.lpop("alert:notifications")
            if not item:
                break
            items.append(json.loads(item))
        if not items:
            time.sleep(1)
            continue
        for notif in items:
            send_fn(notif)
```

## Checking Current Alert Count Per Product

```python
def get_alert_count(product_id):
    return r.zcard(f"alerts:{product_id}")

def get_alerts_above_price(product_id, current_price):
    # Users who would be notified if price drops to current_price
    return r.zrangebyscore(f"alerts:{product_id}", current_price, "+inf", withscores=True)
```

## Example Usage

```bash
# User alice sets alert for product 101 at $29.99
ZADD alerts:prod:101 29.99 user:alice
SADD user:alerts:user:alice prod:101

# Current price
SET product:price:prod:101 35.00

# Price drops to $28 - find triggered alerts
ZRANGEBYSCORE alerts:prod:101 28 +inf   # Returns user:alice
```

## Summary

Redis Sorted Sets make price alert queries trivial - ZRANGEBYSCORE finds all users with a target price above the new price in O(log N + M) time. A notification List acts as a work queue for delivering alerts asynchronously. Removing alerts after triggering prevents duplicate notifications for subsequent price fluctuations.
