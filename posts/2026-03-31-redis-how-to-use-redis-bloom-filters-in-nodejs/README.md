# How to Use Redis Bloom Filters in Node.js

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Node.js, Bloom Filter, Probabilistic, ioredis

Description: Learn how to use Redis Bloom Filters in Node.js with ioredis for memory-efficient membership testing, deduplication, and spam filtering use cases.

---

## What Is a Bloom Filter?

A Bloom filter is a space-efficient probabilistic data structure that tests set membership:

- "Definitely not in the set" - 100% accurate
- "Possibly in the set" - small false positive probability

Redis provides native Bloom filter support via the RedisBloom module (included in Redis Stack).

## Setup

```bash
docker run -p 6379:6379 redis/redis-stack:latest
npm install ioredis
```

## Creating a Bloom Filter

```javascript
const Redis = require('ioredis');
const redis = new Redis();

async function createBloomFilter() {
  // Create with desired false positive rate and expected capacity
  await redis.call(
    'BF.RESERVE', 'emails:blacklist',
    '0.001',    // 0.1% false positive rate
    '1000000'   // Expected 1M items
  );
  console.log('Bloom filter created');
}

createBloomFilter();
```

## Adding and Checking Items

```javascript
const Redis = require('ioredis');
const redis = new Redis();

async function bloomOps() {
  // Add a single item
  const isNew = await redis.call('BF.ADD', 'emails:blacklist', 'spam@evil.com');
  console.log('New item?', isNew); // 1 (new) or 0 (already seen)

  // Add multiple items
  const results = await redis.call(
    'BF.MADD', 'emails:blacklist',
    'phishing@bad.net',
    'scam@fraud.io',
    'noreply@spam.org'
  );
  console.log('New items:', results); // [1, 1, 1]

  // Check single item
  const exists = await redis.call('BF.EXISTS', 'emails:blacklist', 'spam@evil.com');
  console.log('spam@evil.com exists:', exists); // 1

  const clean = await redis.call('BF.EXISTS', 'emails:blacklist', 'alice@valid.com');
  console.log('alice@valid.com exists:', clean); // 0

  // Check multiple items
  const checks = await redis.call(
    'BF.MEXISTS', 'emails:blacklist',
    'spam@evil.com',        // Known blacklisted
    'alice@valid.com',      // Clean
    'phishing@bad.net'      // Known blacklisted
  );
  console.log('Batch check:', checks); // [1, 0, 1]
}

bloomOps();
```

## Bloom Filter Helper Class

```javascript
const Redis = require('ioredis');
const redis = new Redis();

class BloomFilter {
  constructor(key, options = {}) {
    this.key = key;
    this.errorRate = options.errorRate || 0.001;
    this.capacity = options.capacity || 1000000;
  }

  async init() {
    try {
      await redis.call('BF.RESERVE', this.key, String(this.errorRate), String(this.capacity));
    } catch (err) {
      if (!err.message.includes('item exists')) throw err;
    }
    return this;
  }

  async add(item) {
    return redis.call('BF.ADD', this.key, String(item));
  }

  async addMany(items) {
    if (items.length === 0) return [];
    return redis.call('BF.MADD', this.key, ...items.map(String));
  }

  async exists(item) {
    const result = await redis.call('BF.EXISTS', this.key, String(item));
    return result === 1;
  }

  async existsMany(items) {
    if (items.length === 0) return [];
    const results = await redis.call('BF.MEXISTS', this.key, ...items.map(String));
    return items.map((item, i) => ({ item, exists: results[i] === 1 }));
  }

  async filterNew(items) {
    const checks = await this.existsMany(items);
    return checks.filter(c => !c.exists).map(c => c.item);
  }

  async info() {
    const raw = await redis.call('BF.INFO', this.key);
    const info = {};
    for (let i = 0; i < raw.length; i += 2) {
      info[raw[i]] = raw[i + 1];
    }
    return info;
  }
}

// Email blacklist
const blacklist = await new BloomFilter('emails:blacklist', {
  errorRate: 0.001,
  capacity: 500000
}).init();

await blacklist.addMany(['spam@evil.com', 'phishing@bad.net', 'scam@fraud.io']);

const incoming = ['alice@valid.com', 'spam@evil.com', 'bob@clean.org', 'phishing@bad.net'];
const newEmails = await blacklist.filterNew(incoming);
console.log('New (clean) emails:', newEmails);
// ['alice@valid.com', 'bob@clean.org']
```

## Event Deduplication

```javascript
const Redis = require('ioredis');
const redis = new Redis();
const crypto = require('crypto');

class EventDeduplicator {
  constructor(windowMinutes = 60) {
    this.windowMs = windowMinutes * 60 * 1000;
    this.currentKey = () => {
      const bucket = Math.floor(Date.now() / this.windowMs);
      return `events:dedup:${bucket}`;
    };
  }

  async ensureFilter() {
    const key = this.currentKey();
    try {
      await redis.call('BF.RESERVE', key, '0.001', '1000000');
      await redis.expire(key, Math.ceil(this.windowMs / 1000) * 2);
    } catch {
      // Already exists
    }
    return key;
  }

  fingerprint(event) {
    const str = `${event.type}:${event.userId}:${event.resourceId}`;
    return crypto.createHash('md5').update(str).digest('hex');
  }

  async isDuplicate(event) {
    const key = await this.ensureFilter();
    const fp = this.fingerprint(event);
    const isNew = await redis.call('BF.ADD', key, fp);
    return isNew === 0; // 0 means item already existed
  }
}

const deduplicator = new EventDeduplicator(60);

const events = [
  { type: 'view', userId: '1', resourceId: 'post:42' },
  { type: 'view', userId: '1', resourceId: 'post:42' },  // Duplicate
  { type: 'like', userId: '1', resourceId: 'post:42' },
];

for (const event of events) {
  const isDup = await deduplicator.isDuplicate(event);
  console.log(`${event.type} by user ${event.userId}: ${isDup ? 'SKIP (duplicate)' : 'PROCESS'}`);
}
```

## Bloom Filter Info

```javascript
const Redis = require('ioredis');
const redis = new Redis();

const info = await redis.call('BF.INFO', 'emails:blacklist');
for (let i = 0; i < info.length; i += 2) {
  console.log(`${info[i]}: ${info[i + 1]}`);
}
```

## Summary

Redis Bloom Filters in Node.js use ioredis `call()` to invoke `BF.RESERVE`, `BF.ADD`, `BF.MADD`, `BF.EXISTS`, and `BF.MEXISTS`. Create a helper class to encapsulate the raw command API and provide typed methods. Bloom filters are ideal for email blacklisting, event deduplication, and cache miss prevention where memory efficiency is critical and occasional false positives are acceptable. Use `BF.MADD` and `BF.MEXISTS` for batch operations to minimize round trips.
