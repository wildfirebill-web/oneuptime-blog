# How to Implement Caching for ClickHouse API Responses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Caching, Redis, API Design, Performance

Description: Cache ClickHouse API responses with Redis to reduce database load and improve response times for frequently requested analytics queries and dashboards.

---

## When to Cache ClickHouse Results

Not all queries benefit from caching. Cache results when:
- The same query is run frequently (dashboards refreshing every 30 seconds)
- The data doesn't change in real time (daily reports, historical analytics)
- Query execution is expensive (multi-second aggregations)

For real-time counters or queries where data freshness is critical, skip caching.

## Redis Cache Middleware

```javascript
// middleware/cache.js
const redis = require('redis');
const crypto = require('crypto');

const cacheClient = redis.createClient({ url: process.env.REDIS_URL });
cacheClient.connect();

function cacheKey(prefix, params) {
  const hash = crypto
    .createHash('sha256')
    .update(JSON.stringify(params))
    .digest('hex')
    .substring(0, 16);
  return `ch_cache:${prefix}:${hash}`;
}

function withCache(prefix, ttlSeconds) {
  return async (req, res, next) => {
    const key = cacheKey(prefix, { ...req.query, ...req.params });

    try {
      const cached = await cacheClient.get(key);
      if (cached) {
        res.setHeader('X-Cache', 'HIT');
        res.setHeader('X-Cache-TTL', await cacheClient.ttl(key));
        return res.json(JSON.parse(cached));
      }
    } catch (err) {
      // Cache miss or Redis error - continue to handler
    }

    // Store original json method to intercept response
    const originalJson = res.json.bind(res);
    res.json = async (data) => {
      try {
        await cacheClient.setEx(key, ttlSeconds, JSON.stringify(data));
      } catch (err) {
        // Cache write failure is non-fatal
      }
      res.setHeader('X-Cache', 'MISS');
      return originalJson(data);
    };

    next();
  };
}

module.exports = { withCache };
```

## Apply Caching to Routes

```javascript
// app.js
const express = require('express');
const { withCache } = require('./middleware/cache');
const client = require('./db');

const app = express();

// Cache daily metrics for 5 minutes
app.get('/api/metrics/daily',
  withCache('daily_metrics', 300),
  async (req, res) => {
    const result = await client.query({
      query: `
        SELECT
          toString(toDate(event_time)) AS date,
          count() AS events,
          uniq(user_id) AS users
        FROM events
        WHERE event_time >= today() - 30
        GROUP BY date
        ORDER BY date DESC
      `,
      format: 'JSONEachRow',
    });
    const data = await result.json();
    res.json({ data, generated_at: new Date().toISOString() });
  }
);

// Cache top products for 1 minute
app.get('/api/products/top',
  withCache('top_products', 60),
  async (req, res) => {
    const { limit = 10 } = req.query;
    const result = await client.query({
      query: `
        SELECT product_id, sum(revenue) AS total_revenue
        FROM orders
        WHERE order_time >= today() - 7
        GROUP BY product_id
        ORDER BY total_revenue DESC
        LIMIT {limit:UInt8}
      `,
      query_params: { limit: parseInt(limit) },
      format: 'JSONEachRow',
    });
    res.json({ data: await result.json() });
  }
);
```

## ClickHouse Native Query Cache

ClickHouse 23.5+ also has a built-in query cache:

```sql
-- Enable query cache per query
SELECT count(), uniq(user_id)
FROM events
WHERE event_time >= today()
SETTINGS use_query_cache = 1, query_cache_ttl = 60;
```

```xml
<!-- Enable globally in config -->
<query_cache>
  <max_size_in_bytes>1073741824</max_size_in_bytes>
  <max_entries>1024</max_entries>
  <max_entry_size_in_bytes>1048576</max_entry_size_in_bytes>
  <max_entry_size_in_rows>30000000</max_entry_size_in_rows>
</query_cache>
```

## Cache Invalidation

For time-sensitive data, use short TTLs. For daily reports, invalidate at midnight:

```javascript
// Compute TTL to next midnight
function secondsUntilMidnight() {
  const now = new Date();
  const midnight = new Date(now);
  midnight.setHours(24, 0, 0, 0);
  return Math.floor((midnight - now) / 1000);
}
```

## Summary

ClickHouse API caching uses Redis to store query results with a TTL, reducing database load for frequently-run dashboard queries. Implement caching as middleware that intercepts JSON responses, stores them in Redis with appropriate TTLs, and returns cached results on subsequent identical requests. Combine API-level caching with ClickHouse's built-in query cache for maximum effectiveness.
