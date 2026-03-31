# How to Use the Preallocation Pattern in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema Design, Performance, Pattern, Write Optimization

Description: Use the preallocation pattern in MongoDB to avoid document growth and fragmentation by reserving space for future array elements at document creation.

---

## What Is the Preallocation Pattern

The preallocation pattern involves creating documents with placeholder data so that the document's on-disk size is reserved upfront. When the document is later updated with real data, it fits within its pre-reserved space rather than causing a document move on disk.

This pattern is most relevant with the MMAPv1 storage engine (now deprecated), but the concept still applies to array-based schemas in WiredTiger when you want predictable document sizes.

## Classic Preallocation for Time-Series

A common use case is a daily metrics document where all hours are pre-populated with zero values.

```javascript
// Pre-allocate a daily stats document at midnight
async function createDailyDoc(date) {
  const hours = Array.from({ length: 24 }, (_, h) => ({
    hour: h,
    pageviews: 0,
    uniqueVisitors: 0,
    revenue: 0
  }));

  await db.dailyStats.insertOne({
    _id: `stats_${date}`,
    date: new Date(date),
    hours: hours,
    totals: { pageviews: 0, uniqueVisitors: 0, revenue: 0 }
  });
}
```

Increments use positional updates rather than `$push`, so the document never grows:

```javascript
async function recordPageview(date, hour) {
  await db.dailyStats.updateOne(
    { _id: `stats_${date}`, "hours.hour": hour },
    {
      $inc: {
        "hours.$.pageviews": 1,
        "totals.pageviews": 1
      }
    }
  );
}
```

## Bucket Pattern with Preallocation

The bucket pattern groups time-series events into pre-allocated bucket documents to limit document count while maintaining fast writes.

```javascript
// Pre-allocate a metrics bucket for a sensor
async function createBucket(sensorId, bucketStart) {
  const measurements = Array.from({ length: 60 }, (_, i) => ({
    offset: i,
    value: null
  }));

  await db.sensorBuckets.insertOne({
    sensorId: sensorId,
    bucketStart: bucketStart,
    count: 0,
    measurements: measurements
  });
}

// Write a measurement into existing slot
async function writeMeasurement(sensorId, bucketStart, offset, value) {
  await db.sensorBuckets.updateOne(
    { sensorId, bucketStart },
    {
      $set: { [`measurements.${offset}.value`]: value },
      $inc: { count: 1 }
    }
  );
}
```

## Querying Preallocated Documents

Query efficiently by using the bucket's identifier and filtering out null slots:

```javascript
const bucket = await db.sensorBuckets.findOne({ sensorId, bucketStart });
const actualReadings = bucket.measurements.filter(m => m.value !== null);
```

## When Preallocation Helps

Preallocation helps when: writes update existing array slots (not push new elements), write rate is high and document growth causes performance issues, and document size is predictable and bounded.

```javascript
// Check document size to understand growth
db.dailyStats.aggregate([
  { $project: { docSize: { $bsonSize: "$$ROOT" } } },
  { $sort: { docSize: -1 } },
  { $limit: 5 }
]);
```

## Summary

The preallocation pattern reserves document space at creation time so subsequent updates fit in-place rather than growing the document. It works best for time-series and bucketed data where you know the structure upfront. Combined with the bucket pattern, it dramatically reduces collection document counts and improves write throughput for high-frequency metric recording.
