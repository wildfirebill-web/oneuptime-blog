# How to Implement CDN Origin Caching with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cache, CDN

Description: Use Redis as an origin cache layer between your CDN and backend to reduce database load and serve CDN misses faster.

---

CDNs cache responses at the edge but they always miss on the first request or after expiration, sending traffic back to your origin server. By placing Redis between your CDN and your application backend, you can serve CDN misses from memory instead of hitting the database every time.

## Architecture Overview

```text
Browser -> CDN Edge -> Origin Server (Redis cache) -> Database
              |              |
         (cache hit)   (Redis hit - fast)
                       (Redis miss - DB query + populate)
```

The origin server first checks Redis. If data is present, it responds immediately. If not, it queries the database, writes to Redis, and returns the response to the CDN which then caches at the edge.

## Implementation with Node.js and Express

```javascript
const redis = require('redis');
const express = require('express');
const app = express();

const client = redis.createClient({ url: 'redis://localhost:6379' });
client.connect();

const CDN_TTL = 3600;    // CDN edge cache TTL (1 hour)
const ORIGIN_TTL = 7200; // Redis origin cache TTL (2 hours)

async function getOrFetch(key, fetchFn, ttl) {
    const cached = await client.get(key);
    if (cached) {
        return { data: JSON.parse(cached), source: 'redis' };
    }
    const data = await fetchFn();
    await client.setEx(key, ttl, JSON.stringify(data));
    return { data, source: 'db' };
}

app.get('/api/articles/:slug', async (req, res) => {
    const key = `article:${req.params.slug}`;
    const { data, source } = await getOrFetch(
        key,
        () => fetchArticleFromDB(req.params.slug),
        ORIGIN_TTL
    );

    // Tell CDN how long to cache this response
    res.set('Cache-Control', `public, max-age=${CDN_TTL}, s-maxage=${CDN_TTL}`);
    res.set('X-Cache-Source', source);
    res.json(data);
});
```

## Cache-Control Headers for CDN Coordination

```javascript
function setCDNCacheHeaders(res, options = {}) {
    const { maxAge = 3600, staleWhileRevalidate = 60 } = options;
    res.set('Cache-Control',
        `public, max-age=${maxAge}, stale-while-revalidate=${staleWhileRevalidate}`
    );
    res.set('Vary', 'Accept-Encoding');
}
```

## Invalidation on Content Update

When content changes, invalidate both the Redis key and purge the CDN edge cache:

```python
import redis
import requests

r = redis.Redis(host='localhost', port=6379)

def invalidate_article(slug: str):
    # Remove from Redis origin cache
    r.delete(f"article:{slug}")

    # Purge from CDN (Cloudflare example)
    purge_cdn_cache(f"https://example.com/api/articles/{slug}")

def purge_cdn_cache(url: str):
    requests.post(
        "https://api.cloudflare.com/client/v4/zones/ZONE_ID/purge_cache",
        headers={"Authorization": "Bearer CF_TOKEN"},
        json={"files": [url]}
    )
```

## Monitoring Origin Cache Efficiency

```bash
# Check how many origin requests hit Redis vs DB
redis-cli INFO stats | grep keyspace_hits
redis-cli INFO stats | grep keyspace_misses

# Monitor key TTLs for CDN-cached content
redis-cli TTL "article:getting-started-with-redis"
```

## Summary

Redis as a CDN origin cache layer dramatically reduces database load when CDN edges experience misses. The pattern is straightforward - check Redis first on every origin request, populate on miss, and set coordinated TTLs so your CDN and Redis cache stay in sync. Add targeted invalidation on content updates to ensure users always receive accurate data.
