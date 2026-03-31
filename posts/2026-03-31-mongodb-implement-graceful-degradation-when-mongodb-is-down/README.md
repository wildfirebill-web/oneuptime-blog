# How to Implement Graceful Degradation When MongoDB Is Down

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Resilience, Cache

Description: Build applications that degrade gracefully when MongoDB is unavailable using circuit breakers, Redis caching, and read-only fallback modes.

---

## Why Graceful Degradation

A complete application failure when MongoDB goes down is avoidable. Many features can continue operating in a read-only or cached mode, allowing users to browse content, view dashboards, or complete non-write operations while the database recovers. This approach keeps your application partially functional instead of showing a 503 to everyone.

## The Circuit Breaker Pattern

A circuit breaker detects repeated failures and stops hitting MongoDB until it recovers:

```javascript
class MongoCircuitBreaker {
  constructor(threshold = 5, timeout = 30000) {
    this.failureCount = 0
    this.threshold = threshold
    this.timeout = timeout
    this.state = 'CLOSED' // CLOSED, OPEN, HALF_OPEN
    this.nextAttempt = null
  }

  async call(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker OPEN - MongoDB unavailable')
      }
      this.state = 'HALF_OPEN'
    }

    try {
      const result = await fn()
      this.onSuccess()
      return result
    } catch (err) {
      this.onFailure()
      throw err
    }
  }

  onSuccess() {
    this.failureCount = 0
    this.state = 'CLOSED'
  }

  onFailure() {
    this.failureCount++
    if (this.failureCount >= this.threshold) {
      this.state = 'OPEN'
      this.nextAttempt = Date.now() + this.timeout
      console.error(`Circuit OPEN for ${this.timeout / 1000}s`)
    }
  }
}

const mongoBreaker = new MongoCircuitBreaker(5, 30000)
```

## Redis as a Fallback Cache

Cache read results in Redis so they remain available when MongoDB is down:

```javascript
const redis = require('redis')
const client = redis.createClient({ url: process.env.REDIS_URL })

async function getProduct(productId) {
  const cacheKey = `product:${productId}`

  try {
    // Try MongoDB through circuit breaker
    const product = await mongoBreaker.call(async () => {
      return db.collection('products').findOne({ _id: productId })
    })

    // Cache the result for 5 minutes
    await client.set(cacheKey, JSON.stringify(product), { EX: 300 })
    return { data: product, source: 'mongodb' }
  } catch (err) {
    // Fallback to Redis cache
    const cached = await client.get(cacheKey)
    if (cached) {
      return { data: JSON.parse(cached), source: 'cache', stale: true }
    }
    throw new Error('Service unavailable - no cached data')
  }
}
```

## Read-Only Mode Flag

Use a feature flag to switch the app into read-only mode:

```javascript
let readOnlyMode = false

async function checkMongoHealth() {
  try {
    await mongoBreaker.call(() => db.admin().command({ ping: 1 }))
    if (readOnlyMode) {
      console.log('MongoDB recovered, disabling read-only mode')
      readOnlyMode = false
    }
  } catch {
    if (!readOnlyMode) {
      console.warn('MongoDB unavailable, enabling read-only mode')
      readOnlyMode = true
    }
  }
}

// Check every 15 seconds
setInterval(checkMongoHealth, 15000)

// Middleware to block writes in read-only mode
app.use((req, res, next) => {
  const writeMethods = ['POST', 'PUT', 'PATCH', 'DELETE']
  if (readOnlyMode && writeMethods.includes(req.method)) {
    return res.status(503).json({
      error: 'Service temporarily in read-only mode',
      retryAfter: 30,
    })
  }
  next()
})
```

## User-Facing Feedback

Return informative responses rather than generic errors:

```javascript
app.get('/api/dashboard', async (req, res) => {
  const result = await getProduct(req.query.id).catch(err => null)

  if (!result) {
    return res.status(200).json({
      data: null,
      notice: 'Dashboard data temporarily unavailable. We are working to restore service.',
      degraded: true,
    })
  }

  res.json({ ...result, degraded: result.source === 'cache' })
})
```

## Summary

Graceful degradation when MongoDB is down requires three components: a circuit breaker to stop cascading failures, a Redis fallback cache to serve stale reads, and a read-only mode flag with user-facing messaging. Together these patterns keep your application partially functional during database outages, protect MongoDB during recovery by reducing connection pressure, and give users clear feedback instead of opaque 500 errors.
