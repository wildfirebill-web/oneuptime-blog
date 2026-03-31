# How to Design a Shopping Cart Using Redis in a System Design Interview

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, System Design, Shopping Cart, Interview, E-Commerce, Hashes

Description: A system design walkthrough for building a Redis-backed shopping cart, covering data modeling, TTL management, merging guest and logged-in carts.

---

## Problem Statement

Design a shopping cart system that:
- Stores items for both guest and authenticated users
- Persists carts for 30 days for authenticated users
- Merges guest cart with user cart on login
- Supports inventory checks and price lookups
- Handles 500K concurrent active carts

## Why Redis for Shopping Carts?

```text
Requirement             Why Redis
-----------             ---------
Fast read/write         O(1) hash field access
TTL support             Automatic cart expiration
Atomic operations       HINCRBY for quantity changes
Low latency             Sub-ms for all cart operations
Temporary data          Redis is ideal for session-like data
```

## Cart Data Model

Use Redis Hashes to store cart items:

```text
Key: cart:{userId or sessionId}
Type: Hash
TTL: 2592000 (30 days for users), 86400 (1 day for guests)

Field structure:
  item:{productId} -> JSON string with quantity and price snapshot
```

## Cart Operations

```javascript
const Redis = require('ioredis');
const redis = new Redis({ host: 'localhost', port: 6379 });

const USER_CART_TTL = 2592000;  // 30 days
const GUEST_CART_TTL = 86400;   // 1 day

function getCartKey(userId, sessionId) {
  return userId ? `cart:user:${userId}` : `cart:guest:${sessionId}`;
}

async function addItem(userId, sessionId, productId, quantity, price) {
  const cartKey = getCartKey(userId, sessionId);
  const itemKey = `item:${productId}`;

  const existing = await redis.hget(cartKey, itemKey);
  let item;

  if (existing) {
    item = JSON.parse(existing);
    item.quantity += quantity;
  } else {
    item = { productId, quantity, price, addedAt: Date.now() };
  }

  const ttl = userId ? USER_CART_TTL : GUEST_CART_TTL;

  await redis.pipeline()
    .hset(cartKey, itemKey, JSON.stringify(item))
    .expire(cartKey, ttl)
    .exec();

  return item;
}

async function updateQuantity(userId, sessionId, productId, quantity) {
  const cartKey = getCartKey(userId, sessionId);
  const itemKey = `item:${productId}`;

  if (quantity <= 0) {
    await redis.hdel(cartKey, itemKey);
    return null;
  }

  const existing = await redis.hget(cartKey, itemKey);
  if (!existing) return null;

  const item = JSON.parse(existing);
  item.quantity = quantity;
  item.updatedAt = Date.now();

  await redis.hset(cartKey, itemKey, JSON.stringify(item));
  return item;
}

async function removeItem(userId, sessionId, productId) {
  const cartKey = getCartKey(userId, sessionId);
  await redis.hdel(cartKey, `item:${productId}`);
}

async function getCart(userId, sessionId) {
  const cartKey = getCartKey(userId, sessionId);
  const fields = await redis.hgetall(cartKey);

  if (!fields) return { items: [], total: 0, itemCount: 0 };

  const items = Object.values(fields).map(v => JSON.parse(v));
  const total = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  const itemCount = items.reduce((sum, item) => sum + item.quantity, 0);

  return { items, total: total.toFixed(2), itemCount };
}

async function clearCart(userId, sessionId) {
  const cartKey = getCartKey(userId, sessionId);
  await redis.del(cartKey);
}
```

## Merging Guest Cart on Login

```javascript
async function mergeCartsOnLogin(userId, sessionId) {
  const guestKey = `cart:guest:${sessionId}`;
  const userKey = `cart:user:${userId}`;

  const guestFields = await redis.hgetall(guestKey);
  if (!guestFields || Object.keys(guestFields).length === 0) return;

  const pipeline = redis.pipeline();

  for (const [field, value] of Object.entries(guestFields)) {
    const guestItem = JSON.parse(value);
    const existingValue = await redis.hget(userKey, field);

    if (existingValue) {
      // Merge quantities
      const existingItem = JSON.parse(existingValue);
      existingItem.quantity += guestItem.quantity;
      pipeline.hset(userKey, field, JSON.stringify(existingItem));
    } else {
      // Add new item from guest cart
      pipeline.hset(userKey, field, value);
    }
  }

  // Delete guest cart and set user cart TTL
  pipeline.del(guestKey);
  pipeline.expire(userKey, USER_CART_TTL);

  await pipeline.exec();
}
```

## Python Implementation

```python
import redis
import json
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def add_item(cart_key: str, product_id: str, quantity: int, price: float, ttl: int):
    item_key = f"item:{product_id}"
    existing = r.hget(cart_key, item_key)

    if existing:
        item = json.loads(existing)
        item['quantity'] += quantity
    else:
        item = {'productId': product_id, 'quantity': quantity, 'price': price}

    pipe = r.pipeline()
    pipe.hset(cart_key, item_key, json.dumps(item))
    pipe.expire(cart_key, ttl)
    pipe.execute()

    return item

def get_cart_summary(cart_key: str) -> dict:
    fields = r.hgetall(cart_key)
    if not fields:
        return {'items': [], 'total': 0.0, 'itemCount': 0}

    items = [json.loads(v) for v in fields.values()]
    total = sum(i['price'] * i['quantity'] for i in items)
    count = sum(i['quantity'] for i in items)

    return {'items': items, 'total': round(total, 2), 'itemCount': count}
```

## Price Freshness and Inventory Checks

```javascript
async function validateCartPrices(userId, sessionId) {
  const cart = await getCart(userId, sessionId);
  const warnings = [];

  for (const item of cart.items) {
    const currentPrice = await getProductPrice(item.productId); // From your catalog service
    const currentStock = await getProductStock(item.productId);

    if (currentPrice !== item.price) {
      // Update price snapshot
      item.price = currentPrice;
      item.priceUpdatedAt = Date.now();
      await redis.hset(
        getCartKey(userId, sessionId),
        `item:${item.productId}`,
        JSON.stringify(item)
      );
      warnings.push({ productId: item.productId, type: 'price-changed', newPrice: currentPrice });
    }

    if (currentStock < item.quantity) {
      warnings.push({ productId: item.productId, type: 'insufficient-stock', available: currentStock });
    }
  }

  return warnings;
}
```

## Capacity Estimation

```text
500K concurrent carts:
- Average cart: 5 items * 200 bytes = 1KB per cart
- 500K carts = 500MB Redis memory
- Easily fits in a single Redis instance

Peak write rate:
- Add/update item: 10K ops/sec -> Redis handles easily

TTL-based cleanup:
- Guest carts expire in 1 day automatically
- User carts expire in 30 days if inactive
- No manual cleanup needed
```

## Saving Cart to Database

```javascript
async function persistCartToDatabase(userId) {
  const cartKey = `cart:user:${userId}`;
  const cartData = await redis.hgetall(cartKey);

  if (!cartData) return;

  const items = Object.values(cartData).map(v => JSON.parse(v));
  await db.upsert('carts', { userId, items, updatedAt: new Date() });
}
```

## Summary

Redis Hashes are the ideal data structure for shopping carts, providing O(1) field access for add/remove/update operations and atomic quantity changes. Use separate key prefixes for guest (session-based) and authenticated carts with different TTLs. The cart merge operation on login ensures a seamless transition from anonymous to authenticated browsing. With automatic TTL expiration, Redis handles cart cleanup without cron jobs.
