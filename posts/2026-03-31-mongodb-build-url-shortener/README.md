# How to Build a URL Shortener with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, URL Shortener, Schema Design, TTL, Node.js

Description: Learn how to build a URL shortener with MongoDB including slug generation, click tracking, expiry using TTL indexes, and analytics aggregation.

---

## Schema Design

```javascript
// URL document
{
  _id: ObjectId(),
  slug: "abc123",                    // unique short code
  originalUrl: "https://example.com/very/long/path",
  userId: ObjectId("..."),           // null for anonymous links
  title: "Example Page",             // optional, for display
  clicks: 0,
  createdAt: ISODate("2026-01-01"),
  expiresAt: ISODate("2026-04-01"),  // null for permanent links
  isActive: true
}

// Click event (for analytics)
{
  _id: ObjectId(),
  slug: "abc123",
  ip: "192.168.1.1",
  userAgent: "Mozilla/5.0...",
  referrer: "https://twitter.com",
  country: "US",
  createdAt: ISODate()
}
```

## Setting Up Collections and Indexes

```javascript
async function setupUrlShortener(db) {
  // Main URLs collection
  await db.createCollection('urls');
  const urls = db.collection('urls');
  await urls.createIndex({ slug: 1 }, { unique: true });
  await urls.createIndex({ userId: 1, createdAt: -1 });
  await urls.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 }); // TTL index

  // Click events collection
  await db.createCollection('clicks');
  const clicks = db.collection('clicks');
  await clicks.createIndex({ slug: 1, createdAt: -1 });
  // Auto-delete click events older than 90 days
  await clicks.createIndex({ createdAt: 1 }, { expireAfterSeconds: 90 * 86400 });
}
```

## Generating a Unique Slug

```javascript
const crypto = require('crypto');

function generateSlug(length = 6) {
  return crypto.randomBytes(length).toString('base64url').slice(0, length);
}

async function createShortUrl(db, { originalUrl, userId = null, ttlDays = null }) {
  let slug;
  let attempts = 0;

  // Retry until we get a unique slug
  while (attempts < 5) {
    slug = generateSlug(6);
    const existing = await db.collection('urls').findOne({ slug });
    if (!existing) break;
    attempts++;
  }

  const doc = {
    slug,
    originalUrl,
    userId,
    clicks: 0,
    isActive: true,
    createdAt: new Date(),
    expiresAt: ttlDays ? new Date(Date.now() + ttlDays * 86400 * 1000) : null,
  };

  await db.collection('urls').insertOne(doc);
  return slug;
}
```

## Resolving a Slug (Redirect)

```javascript
async function resolveSlug(db, slug, { ip, userAgent, referrer } = {}) {
  const url = await db.collection('urls').findOne(
    { slug, isActive: true, $or: [{ expiresAt: null }, { expiresAt: { $gt: new Date() } }] },
    { projection: { originalUrl: 1 } }
  );

  if (!url) return null;

  // Increment click counter and record analytics event in parallel
  await Promise.all([
    db.collection('urls').updateOne({ slug }, { $inc: { clicks: 1 } }),
    db.collection('clicks').insertOne({
      slug, ip, userAgent, referrer: referrer || null, createdAt: new Date(),
    }),
  ]);

  return url.originalUrl;
}
```

## Click Analytics

```javascript
async function getClickAnalytics(db, slug, days = 30) {
  const since = new Date(Date.now() - days * 86400 * 1000);

  return db.collection('clicks').aggregate([
    { $match: { slug, createdAt: { $gte: since } } },
    {
      $facet: {
        daily: [
          {
            $group: {
              _id: { $dateToString: { format: '%Y-%m-%d', date: '$createdAt' } },
              count: { $sum: 1 },
            },
          },
          { $sort: { _id: 1 } },
        ],
        topReferrers: [
          { $group: { _id: '$referrer', count: { $sum: 1 } } },
          { $sort: { count: -1 } },
          { $limit: 5 },
        ],
        topCountries: [
          { $group: { _id: '$country', count: { $sum: 1 } } },
          { $sort: { count: -1 } },
          { $limit: 5 },
        ],
      },
    },
  ]).next();
}
```

## Summary

A MongoDB URL shortener uses a unique index on `slug` for fast redirects, a TTL index on `expiresAt` for automatic expiry of short-lived links, and a separate `clicks` collection for analytics. Use `$inc` for atomic click counting, `$facet` for multi-dimensional analytics queries, and `base64url` encoding for URL-safe slug generation.
