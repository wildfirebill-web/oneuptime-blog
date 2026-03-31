# How to Build a Trading Order Book with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Finance, Trading, Sorted Set, Lua

Description: Implement a high-performance trading order book with Redis sorted sets to maintain bid and ask queues, match orders atomically, and publish trades in real time.

---

An order book tracks open buy and sell orders for a trading pair. It must handle thousands of order submissions per second, match orders at the correct price, and broadcast trades instantly. Redis sorted sets are a natural fit for bid and ask queues where price is the sort key.

## Order Book Structure

Use two sorted sets per trading pair: one for bids (buy orders, descending) and one for asks (sell orders, ascending):

```bash
# Bids: score = price (higher is better for buyers)
ZADD orderbook:BTC-USD:bids 42500.00 "order:101"
ZADD orderbook:BTC-USD:bids 42499.50 "order:102"

# Asks: score = price (lower is better for sellers)
ZADD orderbook:BTC-USD:asks 42501.00 "order:201"
ZADD orderbook:BTC-USD:asks 42502.50 "order:202"
```

## Storing Order Details

Keep order metadata in a hash:

```python
import redis
r = redis.Redis()

def place_order(order_id, side, price, quantity, user_id, symbol):
    r.hset(f"order:{order_id}", mapping={
        "side": side, "price": price, "quantity": quantity,
        "user_id": user_id, "symbol": symbol, "status": "open"
    })
    book_key = f"orderbook:{symbol}:{side}s"
    r.zadd(book_key, {f"order:{order_id}": float(price)})
```

## Best Bid and Ask

Retrieve the top of book in O(log N):

```python
def best_bid(symbol):
    result = r.zrevrange(f"orderbook:{symbol}:bids", 0, 0, withscores=True)
    return result[0] if result else None

def best_ask(symbol):
    result = r.zrange(f"orderbook:{symbol}:asks", 0, 0, withscores=True)
    return result[0] if result else None
```

## Order Matching with Lua

Match a market buy against the best asks atomically:

```lua
local asks_key = KEYS[1]
local trades_key = KEYS[2]
local quantity_left = tonumber(ARGV[1])
local buyer_order_id = ARGV[2]
local matched = {}

while quantity_left > 0 do
  local best = redis.call("ZRANGE", asks_key, 0, 0, "WITHSCORES")
  if #best == 0 then break end
  local ask_order_id = best[1]
  local ask_price = best[2]
  local ask_data = redis.call("HGETALL", ask_order_id)
  -- simplified: assume full fill
  redis.call("ZREM", asks_key, ask_order_id)
  redis.call("HSET", ask_order_id, "status", "filled")
  local trade = buyer_order_id .. ":" .. ask_order_id .. ":" .. ask_price
  redis.call("RPUSH", trades_key, trade)
  quantity_left = 0
end
return quantity_left
```

## Publishing Trades

Publish each matched trade to subscribers:

```python
import json, time

def publish_trade(symbol, buyer_order, seller_order, price, quantity):
    trade = json.dumps({
        "symbol": symbol, "price": price, "quantity": quantity,
        "buyer": buyer_order, "seller": seller_order, "ts": time.time()
    })
    r.publish(f"trades:{symbol}", trade)
    r.rpush(f"trade:history:{symbol}", trade)
    r.ltrim(f"trade:history:{symbol}", 0, 9999)
```

## Order Cancellation

Remove an order from the book:

```python
def cancel_order(order_id, symbol, side):
    r.zrem(f"orderbook:{symbol}:{side}s", f"order:{order_id}")
    r.hset(f"order:{order_id}", "status", "cancelled")
```

## Summary

Redis sorted sets provide the exact data structure needed for a trading order book: price-ordered bid and ask queues with O(log N) insertion and retrieval. Lua scripts enable atomic matching, and Pub/Sub delivers trade notifications to connected clients in real time, making Redis a capable engine for high-frequency trading systems.
