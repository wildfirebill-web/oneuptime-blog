# How to Use Redis as a Cache in TypeScript Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, TypeScript, Cache, Node.js

Description: Learn how to implement a type-safe Redis cache layer in TypeScript, including generic cache helpers, TTL management, and cache-aside pattern.

---

Redis is the de facto caching layer for Node.js applications. With TypeScript, you can build a type-safe cache client that enforces consistent serialization, TTL policies, and cache-aside logic throughout your application.

## Setup

```bash
npm install redis
```

Redis is bundled with TypeScript types - no separate `@types/redis` needed.

## Typed Redis Client

```typescript
import { createClient, RedisClientType } from 'redis';

const client: RedisClientType = createClient({
  url: process.env.REDIS_URL ?? 'redis://localhost:6379',
});

client.on('error', (err: Error) => console.error('Redis error:', err));
await client.connect();
```

## Generic Cache Helper

```typescript
interface CacheOptions {
  ttl?: number; // seconds
}

async function cacheGet<T>(key: string): Promise<T | null> {
  const raw = await client.get(key);
  if (!raw) return null;
  return JSON.parse(raw) as T;
}

async function cacheSet<T>(key: string, value: T, options: CacheOptions = {}): Promise<void> {
  const serialized = JSON.stringify(value);
  if (options.ttl) {
    await client.setEx(key, options.ttl, serialized);
  } else {
    await client.set(key, serialized);
  }
}

async function cacheDel(key: string): Promise<void> {
  await client.del(key);
}
```

## Cache-Aside Pattern

The cache-aside pattern: read from cache first; on miss, load from source and populate cache.

```typescript
interface User {
  id: string;
  name: string;
  email: string;
}

async function getUserById(id: string): Promise<User | null> {
  const cacheKey = `user:${id}`;

  // 1. Check cache
  const cached = await cacheGet<User>(cacheKey);
  if (cached) {
    console.log('Cache hit');
    return cached;
  }

  // 2. Load from database
  const user = await db.users.findById(id); // your DB call
  if (!user) return null;

  // 3. Store in cache for 10 minutes
  await cacheSet<User>(cacheKey, user, { ttl: 600 });
  return user;
}
```

## Cache Invalidation on Update

```typescript
async function updateUser(id: string, updates: Partial<User>): Promise<User> {
  const user = await db.users.update(id, updates);

  // Invalidate cache after write
  await cacheDel(`user:${id}`);

  return user;
}
```

## Bulk Operations with Generics

```typescript
async function mGetCached<T>(keys: string[]): Promise<(T | null)[]> {
  const values = await client.mGet(keys);
  return values.map((v) => (v ? (JSON.parse(v) as T) : null));
}

async function mSetCached<T>(entries: { key: string; value: T; ttl?: number }[]): Promise<void> {
  const pipeline = client.multi();
  for (const { key, value, ttl } of entries) {
    const serialized = JSON.stringify(value);
    if (ttl) {
      pipeline.setEx(key, ttl, serialized);
    } else {
      pipeline.set(key, serialized);
    }
  }
  await pipeline.exec();
}
```

## Cache Key Factory

```typescript
const CacheKeys = {
  user: (id: string) => `user:${id}`,
  userPermissions: (id: string) => `user:${id}:perms`,
  product: (id: string) => `product:${id}`,
  productList: (page: number) => `products:page:${page}`,
} as const;

// Usage
const user = await cacheGet<User>(CacheKeys.user('123'));
```

## Summary

Building a Redis cache layer in TypeScript involves a generic get/set wrapper that handles JSON serialization and deserialization with typed return values. The cache-aside pattern - check cache, fallback to database, populate cache - keeps your source of truth in the database while Redis handles read acceleration. Centralizing cache key definitions in a factory object prevents key typos and makes invalidation straightforward.
