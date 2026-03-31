# How to Monitor Notification Queue Health with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Monitoring, Queue, Notification, Observability

Description: Monitor a MongoDB notification queue for backlog depth, dead-letter accumulation, delivery rates, and processing latency to detect queue health issues early.

---

## Overview

A notification queue that processes emails, SMS, and webhooks needs continuous monitoring to detect backlogs, stuck jobs, and delivery failures before they affect users. MongoDB's aggregation pipeline provides all the tools needed to compute queue depth, success rates, processing latency, and dead-letter counts directly from the queue collection.

## Queue Depth by Status and Channel

Check how many jobs are in each status for each channel.

```javascript
async function getQueueDepth(db) {
  return db.collection("notification_queue").aggregate([
    {
      $group: {
        _id: { channel: "$channel", status: "$status" },
        count: { $sum: 1 },
      },
    },
    {
      $group: {
        _id: "$_id.channel",
        statuses: {
          $push: { status: "$_id.status", count: "$count" },
        },
        total: { $sum: "$count" },
      },
    },
    { $sort: { _id: 1 } },
  ]).toArray();
}
```

## Backlog Age - Oldest Pending Job

Detect if jobs are stuck by finding the oldest pending item.

```javascript
async function getOldestPendingJob(db) {
  const oldest = await db.collection("notification_queue").findOne(
    { status: "pending" },
    { sort: { createdAt: 1 }, projection: { channel: 1, createdAt: 1, attempts: 1 } }
  );

  if (!oldest) return { backlogAgeMinutes: 0 };

  const ageMs = Date.now() - oldest.createdAt.getTime();
  return {
    channel: oldest.channel,
    createdAt: oldest.createdAt,
    attempts: oldest.attempts,
    backlogAgeMinutes: Math.round(ageMs / 60000),
  };
}
```

## Delivery Success Rate (Last 1 Hour)

```javascript
async function getDeliveryStats(db, windowHours = 1) {
  const since = new Date(Date.now() - windowHours * 60 * 60 * 1000);

  return db.collection("notification_queue").aggregate([
    { $match: { createdAt: { $gte: since } } },
    {
      $group: {
        _id: "$channel",
        total: { $sum: 1 },
        sent: { $sum: { $cond: [{ $eq: ["$status", "sent"] }, 1, 0] } },
        failed: { $sum: { $cond: [{ $eq: ["$status", "failed"] }, 1, 0] } },
        dead: { $sum: { $cond: [{ $eq: ["$status", "dead"] }, 1, 0] } },
        pending: { $sum: { $cond: [{ $eq: ["$status", "pending"] }, 1, 0] } },
      },
    },
    {
      $project: {
        channel: "$_id",
        total: 1,
        sent: 1,
        failed: 1,
        dead: 1,
        pending: 1,
        successRate: {
          $cond: [
            { $eq: ["$total", 0] },
            null,
            { $multiply: [{ $divide: ["$sent", "$total"] }, 100] },
          ],
        },
      },
    },
  ]).toArray();
}
```

## Processing Latency (Time from Created to Sent)

```javascript
async function getProcessingLatency(db, windowHours = 1) {
  const since = new Date(Date.now() - windowHours * 60 * 60 * 1000);

  return db.collection("notification_queue").aggregate([
    { $match: { status: "sent", sentAt: { $gte: since } } },
    {
      $addFields: {
        latencyMs: { $subtract: ["$sentAt", "$createdAt"] },
      },
    },
    {
      $group: {
        _id: "$channel",
        avgLatencyMs: { $avg: "$latencyMs" },
        p95LatencyMs: { $percentile: { input: "$latencyMs", p: [0.95], method: "approximate" } },
        maxLatencyMs: { $max: "$latencyMs" },
        count: { $sum: 1 },
      },
    },
  ]).toArray();
}
```

## Health Check Endpoint

Expose queue health as an HTTP endpoint for uptime monitoring.

```javascript
app.get("/health/queue", async (req, res) => {
  const depth = await getQueueDepth(db);
  const oldest = await getOldestPendingJob(db);
  const stats = await getDeliveryStats(db);

  const unhealthy = oldest.backlogAgeMinutes > 30
    || stats.some((s) => s.dead > 10)
    || stats.some((s) => s.successRate < 90);

  res.status(unhealthy ? 503 : 200).json({
    status: unhealthy ? "degraded" : "healthy",
    depth,
    oldestPendingAgeMinutes: oldest.backlogAgeMinutes,
    deliveryStats: stats,
  });
});
```

## Summary

Monitoring a MongoDB notification queue requires tracking queue depth by status and channel, detecting aged pending jobs that signal a stuck worker, computing delivery success rates over rolling time windows, and measuring processing latency. Expose these metrics via a health endpoint so your uptime monitoring system can alert on queue degradation automatically.
