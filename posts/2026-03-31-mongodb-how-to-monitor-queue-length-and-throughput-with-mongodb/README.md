# How to Monitor Queue Length and Throughput with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Job Queue, Monitoring, Aggregation, Throughput

Description: Learn how to monitor job queue depth, throughput, and latency for a MongoDB-backed job queue using aggregation pipelines and dashboards.

---

## Key Queue Metrics to Track

For a healthy job queue, monitor these metrics:
- **Queue depth** - pending jobs by type and priority
- **Throughput** - jobs completed per minute/hour
- **Processing latency** - time from creation to completion
- **Error rate** - failed jobs as a percentage of total
- **Stuck jobs** - jobs in processing state too long

## Queue Depth by Status

```javascript
// Get current queue depth broken down by status and type
db.jobs.aggregate([
  {
    $group: {
      _id: { status: "$status", type: "$type" },
      count: { $sum: 1 },
      oldestJob: { $min: "$createdAt" }
    }
  },
  { $sort: { "_id.status": 1, "_id.type": 1 } }
])
/* Output:
[
  { _id: { status: "failed", type: "sendEmail" }, count: 3, oldestJob: ISODate(...) },
  { _id: { status: "pending", type: "sendEmail" }, count: 142, oldestJob: ISODate(...) },
  { _id: { status: "pending", type: "generateReport" }, count: 8, oldestJob: ISODate(...) },
  { _id: { status: "processing", type: "sendEmail" }, count: 5, oldestJob: ISODate(...) }
]
*/
```

## Throughput: Jobs Completed Per Hour

```javascript
// Jobs completed in the last 24 hours, grouped by hour
db.jobs.aggregate([
  {
    $match: {
      status: "completed",
      completedAt: { $gte: new Date(Date.now() - 24 * 3600 * 1000) }
    }
  },
  {
    $group: {
      _id: {
        year: { $year: "$completedAt" },
        month: { $month: "$completedAt" },
        day: { $dayOfMonth: "$completedAt" },
        hour: { $hour: "$completedAt" }
      },
      completedCount: { $sum: 1 },
      avgProcessingMs: { $avg: {
        $subtract: ["$completedAt", "$processingStartedAt"]
      }}
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1, "_id.day": 1, "_id.hour": 1 } },
  {
    $project: {
      hour: "$_id",
      completedCount: 1,
      avgProcessingSeconds: { $round: [{ $divide: ["$avgProcessingMs", 1000] }, 2] }
    }
  }
])
```

## Processing Latency Percentiles

```javascript
// P50, P95, P99 processing latency for each job type
db.jobs.aggregate([
  {
    $match: {
      status: "completed",
      completedAt: { $gte: new Date(Date.now() - 3600 * 1000) },
      processingStartedAt: { $ne: null }
    }
  },
  {
    $addFields: {
      processingMs: { $subtract: ["$completedAt", "$processingStartedAt"] },
      queueWaitMs: { $subtract: ["$processingStartedAt", "$createdAt"] }
    }
  },
  {
    $group: {
      _id: "$type",
      p50ProcessingMs: {
        $percentile: { input: "$processingMs", p: [0.5], method: "approximate" }
      },
      p95ProcessingMs: {
        $percentile: { input: "$processingMs", p: [0.95], method: "approximate" }
      },
      p99ProcessingMs: {
        $percentile: { input: "$processingMs", p: [0.99], method: "approximate" }
      },
      p95WaitMs: {
        $percentile: { input: "$queueWaitMs", p: [0.95], method: "approximate" }
      },
      count: { $sum: 1 }
    }
  }
])
```

## Error Rate by Job Type

```javascript
// Failure rate over the last hour
db.jobs.aggregate([
  {
    $match: {
      updatedAt: { $gte: new Date(Date.now() - 3600 * 1000) },
      status: { $in: ["completed", "failed"] }
    }
  },
  {
    $group: {
      _id: "$type",
      total: { $sum: 1 },
      failed: { $sum: { $cond: [{ $eq: ["$status", "failed"] }, 1, 0] } },
      completed: { $sum: { $cond: [{ $eq: ["$status", "completed"] }, 1, 0] } }
    }
  },
  {
    $project: {
      type: "$_id",
      total: 1,
      failed: 1,
      completed: 1,
      failureRate: {
        $round: [{ $multiply: [{ $divide: ["$failed", "$total"] }, 100] }, 2]
      }
    }
  },
  { $sort: { failureRate: -1 } }
])
```

## Detecting Stuck Jobs

```javascript
// Jobs in "processing" state longer than expected
function findStuckJobs(collection, thresholdMinutes = 10) {
  const cutoff = new Date(Date.now() - thresholdMinutes * 60 * 1000)
  return collection.find({
    status: "processing",
    processingStartedAt: { $lt: cutoff }
  }).toArray()
}

// Get stuck job summary
db.jobs.aggregate([
  {
    $match: {
      status: "processing",
      processingStartedAt: { $lt: new Date(Date.now() - 600000) }
    }
  },
  {
    $group: {
      _id: "$type",
      stuckCount: { $sum: 1 },
      workersAffected: { $addToSet: "$workerId" },
      oldestStuck: { $min: "$processingStartedAt" }
    }
  }
])
```

## Real-Time Queue Monitor Function

```javascript
async function queueDashboard(collection) {
  const now = new Date()
  const oneHourAgo = new Date(now - 3600000)

  const [depthByStatus, hourlyThroughput, failureRate] = await Promise.all([
    // Queue depth
    collection.aggregate([
      { $group: { _id: "$status", count: { $sum: 1 } } }
    ]).toArray(),

    // Last hour completed
    collection.countDocuments({
      status: "completed",
      completedAt: { $gte: oneHourAgo }
    }),

    // Failure rate (last hour)
    collection.aggregate([
      { $match: { updatedAt: { $gte: oneHourAgo }, status: { $in: ["completed", "failed"] } } },
      { $group: {
          _id: null,
          total: { $sum: 1 },
          failed: { $sum: { $cond: [{ $eq: ["$status", "failed"] }, 1, 0] } }
      }}
    ]).toArray()
  ])

  const depth = Object.fromEntries(depthByStatus.map(d => [d._id, d.count]))
  const fr = failureRate[0] || { total: 0, failed: 0 }

  console.log("=== Queue Dashboard ===")
  console.log("Pending:", depth.pending || 0)
  console.log("Processing:", depth.processing || 0)
  console.log("Completed:", depth.completed || 0)
  console.log("Failed:", depth.failed || 0)
  console.log("Throughput (last hour):", hourlyThroughput, "jobs/hour")
  console.log("Failure rate (last hour):",
    fr.total > 0 ? ((fr.failed / fr.total) * 100).toFixed(1) + "%" : "N/A"
  )
}

// Run every 30 seconds
setInterval(() => queueDashboard(db.collection("jobs")), 30000)
```

## Summary

Monitor a MongoDB job queue by querying the jobs collection with aggregation pipelines for queue depth by status and type, hourly throughput, processing latency percentiles, and failure rates. Build a dashboard function that runs periodically to surface key health indicators. Use the stuck job query to alert when workers stop processing and jobs pile up in the processing state.
