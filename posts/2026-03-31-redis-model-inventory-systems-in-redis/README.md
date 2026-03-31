# How to Model Inventory Systems in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Inventory, E-Commerce, Atomic Operation, Data Modeling, Stock Management

Description: Model inventory systems in Redis using atomic INCR/DECR, hashes, and Lua scripts to prevent overselling, track stock levels, and handle concurrent reservations.

---

## Why Redis for Inventory

Inventory systems face a classic concurrency problem: multiple users attempting to purchase the last item simultaneously. Redis solves this with atomic commands like INCR/DECR and Lua scripts that execute as a single indivisible operation, preventing race conditions without complex locking.

## Basic Stock Tracking

Store stock counts as simple strings (integers):

```bash
# Set initial stock
SET stock:product:1001 100
SET stock:product:1002 50
SET stock:product:1003 0

# Check stock
GET stock:product:1001

# Decrement on purchase (atomic)
DECR stock:product:1001

# Increment on restock
INCRBY stock:product:1001 25
```

## Hash-Based Product Inventory

Store richer inventory data as hashes:

```bash
HSET inventory:product:1001 total 100 reserved 15 available 85 warehouse_a 60 warehouse_b 40
HSET inventory:product:1002 total 50 reserved 5 available 45 warehouse_a 30 warehouse_b 20
```

Update available count:

```bash
HINCRBY inventory:product:1001 available -1
HINCRBY inventory:product:1001 reserved 1
```

## Atomic Reserve with Lua Script

Prevent overselling with an atomic check-and-reserve:

```lua
local key = KEYS[1]
local qty = tonumber(ARGV[1])
local available = tonumber(redis.call('HGET', key, 'available'))
if available == nil or available < qty then
    return 0
end
redis.call('HINCRBY', key, 'available', -qty)
redis.call('HINCRBY', key, 'reserved', qty)
return 1
```

Load and call:

```bash
redis-cli EVAL "local key = KEYS[1]; local qty = tonumber(ARGV[1]); local available = tonumber(redis.call('HGET', key, 'available')); if available == nil or available < qty then return 0 end; redis.call('HINCRBY', key, 'available', -qty); redis.call('HINCRBY', key, 'reserved', qty); return 1" 1 inventory:product:1001 2
```

## Python Example - Inventory Service

```python
import redis

r = redis.Redis(decode_responses=True)

RESERVE_SCRIPT = """
local key = KEYS[1]
local qty = tonumber(ARGV[1])
local available = tonumber(redis.call('HGET', key, 'available'))
if available == nil or available < qty then
    return 0
end
redis.call('HINCRBY', key, 'available', -qty)
redis.call('HINCRBY', key, 'reserved', qty)
return 1
"""

RELEASE_SCRIPT = """
local key = KEYS[1]
local qty = tonumber(ARGV[1])
redis.call('HINCRBY', key, 'available', qty)
redis.call('HINCRBY', key, 'reserved', -qty)
return 1
"""

reserve_lua = r.register_script(RESERVE_SCRIPT)
release_lua = r.register_script(RELEASE_SCRIPT)

def initialize_product(product_id: str, total: int):
    r.hset(f"inventory:product:{product_id}", mapping={
        "total": total,
        "reserved": 0,
        "available": total
    })

def reserve_stock(product_id: str, qty: int = 1):
    result = reserve_lua(keys=[f"inventory:product:{product_id}"], args=[qty])
    return result == 1

def release_stock(product_id: str, qty: int = 1):
    return release_lua(keys=[f"inventory:product:{product_id}"], args=[qty])

def confirm_purchase(product_id: str, qty: int = 1):
    r.hincrby(f"inventory:product:{product_id}", "reserved", -qty)
    r.hincrby(f"inventory:product:{product_id}", "total", -qty)

def get_inventory(product_id: str):
    return r.hgetall(f"inventory:product:{product_id}")

initialize_product("1001", 100)
print("Reserved:", reserve_stock("1001", 2))
print("Inventory:", get_inventory("1001"))
confirm_purchase("1001", 2)
print("After purchase:", get_inventory("1001"))
```

## Reservation Expiry

Automatically release reservations that are not confirmed within a timeout. Use a sorted set to track expiry:

```python
import time

def reserve_with_expiry(product_id: str, order_id: str, qty: int, ttl_seconds: int = 300):
    success = reserve_stock(product_id, qty)
    if success:
        expiry_ts = time.time() + ttl_seconds
        r.zadd("reservations:expiry", {f"{product_id}:{order_id}:{qty}": expiry_ts})
    return success

def expire_reservations():
    now = time.time()
    expired = r.zrangebyscore("reservations:expiry", "-inf", now)
    for entry in expired:
        product_id, order_id, qty = entry.split(":")
        release_stock(product_id, int(qty))
        r.zrem("reservations:expiry", entry)
```

## Low Stock Alerts

Use Pub/Sub to broadcast low stock events:

```python
def check_and_alert_low_stock(product_id: str, threshold: int = 10):
    available = int(r.hget(f"inventory:product:{product_id}", "available") or 0)
    if available <= threshold:
        r.publish("alerts:low_stock", f"product:{product_id}:available:{available}")
```

## Summary

Redis models inventory systems using hashes for rich stock data and Lua scripts for atomic check-and-reserve operations that prevent overselling under high concurrency. Reservation expiry with sorted sets automatically releases unconfirmed holds, while Pub/Sub enables real-time low-stock alerting. This approach handles thousands of concurrent purchase requests reliably.
