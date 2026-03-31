# How to Implement a Job Queue with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Job Queue, Worker, Background Job, Node

Description: Learn how to build a reliable job queue backed by MongoDB using atomic find-and-modify operations for job claiming, retry logic, and dead letter handling.

---

A MongoDB-backed job queue uses atomic operations to ensure each job is processed exactly once, even when multiple workers run in parallel. The key operation is an atomic claim: find a pending job and mark it as "running" in a single atomic step.

## Job Schema

```javascript
const mongoose = require('mongoose');

const jobSchema = new mongoose.Schema({
  queue: { type: String, required: true, index: true },
  type: { type: String, required: true },
  payload: mongoose.Schema.Types.Mixed,
  status: {
    type: String,
    enum: ['pending', 'running', 'completed', 'failed'],
    default: 'pending',
    index: true,
  },
  priority: { type: Number, default: 0, index: true },
  attempts: { type: Number, default: 0 },
  maxAttempts: { type: Number, default: 3 },
  lastError: String,
  scheduledAt: { type: Date, default: Date.now },
  startedAt: Date,
  completedAt: Date,
  lockedUntil: { type: Date, index: true },
  createdAt: { type: Date, default: Date.now },
});

// Compound index for the worker query
jobSchema.index({ queue: 1, status: 1, priority: -1, scheduledAt: 1 });

// Auto-delete completed jobs after 7 days
jobSchema.index(
  { completedAt: 1 },
  { expireAfterSeconds: 7 * 24 * 60 * 60, partialFilterExpression: { status: 'completed' } }
);

const Job = mongoose.model('Job', jobSchema);
```

## Enqueuing Jobs

```javascript
async function enqueue(queue, type, payload, { priority = 0, delay = 0 } = {}) {
  return Job.create({
    queue,
    type,
    payload,
    priority,
    scheduledAt: new Date(Date.now() + delay),
  });
}

// Usage
await enqueue('emails', 'send_welcome', { userId: '123', email: 'user@example.com' });
await enqueue('reports', 'generate_monthly', { month: '2026-03' }, { delay: 5000 });
```

## Atomically Claiming a Job

The claim operation must be atomic to prevent two workers from picking the same job:

```javascript
const LOCK_DURATION_MS = 5 * 60 * 1000; // 5 minutes

async function claimNextJob(queue) {
  const now = new Date();

  return Job.findOneAndUpdate(
    {
      queue,
      status: 'pending',
      scheduledAt: { $lte: now },
      $or: [
        { lockedUntil: null },
        { lockedUntil: { $lte: now } }, // Reclaim stale locks
      ],
    },
    {
      $set: {
        status: 'running',
        startedAt: now,
        lockedUntil: new Date(now.getTime() + LOCK_DURATION_MS),
      },
      $inc: { attempts: 1 },
    },
    {
      sort: { priority: -1, scheduledAt: 1 },
      new: true,
    }
  );
}
```

## Worker Loop

```javascript
const handlers = {
  send_welcome: async (payload) => {
    await sendWelcomeEmail(payload.email, payload.userId);
  },
  generate_monthly: async (payload) => {
    await generateReport(payload.month);
  },
};

async function processJobs(queue) {
  while (true) {
    const job = await claimNextJob(queue);

    if (!job) {
      await new Promise((resolve) => setTimeout(resolve, 2000)); // Wait 2s before polling again
      continue;
    }

    try {
      const handler = handlers[job.type];
      if (!handler) throw new Error(`No handler for job type: ${job.type}`);

      await handler(job.payload);

      await Job.findByIdAndUpdate(job._id, {
        $set: { status: 'completed', completedAt: new Date(), lockedUntil: null },
      });

      console.log(`Job ${job._id} (${job.type}) completed`);
    } catch (err) {
      const failed = job.attempts >= job.maxAttempts;
      await Job.findByIdAndUpdate(job._id, {
        $set: {
          status: failed ? 'failed' : 'pending',
          lastError: err.message,
          lockedUntil: null,
          // Exponential backoff for retries
          scheduledAt: failed ? undefined : new Date(Date.now() + Math.pow(2, job.attempts) * 1000),
        },
      });

      console.error(`Job ${job._id} ${failed ? 'failed permanently' : 'will retry'}:`, err.message);
    }
  }
}

// Start multiple workers
processJobs('emails');
processJobs('reports');
```

## Monitoring the Queue

```javascript
// Dashboard query
db.jobs.aggregate([
  { $group: { _id: { queue: "$queue", status: "$status" }, count: { $sum: 1 } } },
  { $sort: { "_id.queue": 1 } }
])
```

## Summary

A MongoDB job queue relies on atomic `findOneAndUpdate` to claim jobs without race conditions. Use a `lockedUntil` timestamp to automatically reclaim stale jobs from crashed workers. Implement exponential backoff for retries and cap attempts with `maxAttempts`. Use compound indexes on `queue`, `status`, `priority`, and `scheduledAt` to make the worker poll query fast even with millions of documents.
