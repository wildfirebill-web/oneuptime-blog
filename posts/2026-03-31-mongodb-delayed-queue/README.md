# How to Implement a Delayed Queue with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Queue, Delayed Job, Scheduling, Node.js

Description: Build a delayed job queue in MongoDB that dequeues jobs only after a specified future time, enabling scheduled tasks without an external scheduler.

---

## Overview

A delayed queue holds jobs until a designated run-after time has passed. Use cases include sending a confirmation email 5 minutes after signup, retrying a failed webhook after 30 seconds, or scheduling a report for the end of the business day. MongoDB's atomic `findOneAndUpdate` handles this without external cron or scheduler services.

## Collection Design and Index

```javascript
// Delayed job document schema
{
  _id: ObjectId,
  payload: Object,
  status: String,      // 'scheduled' | 'processing' | 'done' | 'failed'
  runAt: Date,         // Do not process before this time
  createdAt: Date,
  attempts: Number,
  lockedAt: Date,
  lockedBy: String
}
```

Create a compound index so the dequeue query is efficient.

```javascript
// Index for fetching jobs that are ready to run
await col.createIndex({ status: 1, runAt: 1 });

// TTL index to auto-delete completed jobs after 7 days
await col.createIndex(
  { completedAt: 1 },
  { expireAfterSeconds: 604800, sparse: true }
);
```

## Enqueueing Delayed Jobs

```javascript
async function scheduleJob(col, payload, delayMs) {
  const runAt = new Date(Date.now() + delayMs);

  const result = await col.insertOne({
    payload,
    status: 'scheduled',
    runAt,
    createdAt: new Date(),
    attempts: 0,
    lockedAt: null,
    lockedBy: null
  });

  console.log(`Job scheduled to run at ${runAt.toISOString()}`);
  return result.insertedId;
}

// Schedule a job to run in 5 minutes
await scheduleJob(col, { type: 'send_reminder', userId: 'u42' }, 5 * 60 * 1000);

// Schedule a job to run tomorrow at a fixed timestamp
const tomorrow = new Date();
tomorrow.setDate(tomorrow.getDate() + 1);
tomorrow.setHours(9, 0, 0, 0);

await col.insertOne({
  payload: { type: 'daily_report' },
  status: 'scheduled',
  runAt: tomorrow,
  createdAt: new Date(),
  attempts: 0
});
```

## Dequeuing Ready Jobs

```javascript
const WORKER_ID = `worker-${process.pid}`;
const LOCK_TIMEOUT_MS = 60_000;

async function dequeueReady(col) {
  const now = new Date();

  return col.findOneAndUpdate(
    {
      status: 'scheduled',
      runAt: { $lte: now }, // Only pick up jobs whose time has come
      $or: [
        { lockedAt: null },
        { lockedAt: { $lt: new Date(Date.now() - LOCK_TIMEOUT_MS) } }
      ]
    },
    {
      $set: { status: 'processing', lockedAt: now, lockedBy: WORKER_ID },
      $inc: { attempts: 1 }
    },
    {
      sort: { runAt: 1 }, // Process earliest-scheduled jobs first
      returnDocument: 'after'
    }
  );
}
```

## Worker Loop with Delay-Aware Polling

Instead of polling at a fixed interval, calculate how long until the next job is due and sleep that amount.

```javascript
async function runWorker(col) {
  while (true) {
    const job = await dequeueReady(col);

    if (job) {
      try {
        await processJob(job.payload);
        await col.updateOne(
          { _id: job._id },
          { $set: { status: 'done', completedAt: new Date() } }
        );
      } catch (err) {
        const maxAttempts = 3;
        if (job.attempts >= maxAttempts) {
          await col.updateOne({ _id: job._id }, { $set: { status: 'failed', error: err.message } });
        } else {
          // Reschedule with exponential back-off
          const retryIn = Math.pow(2, job.attempts) * 10_000;
          await col.updateOne(
            { _id: job._id },
            { $set: { status: 'scheduled', runAt: new Date(Date.now() + retryIn), lockedAt: null } }
          );
        }
      }
      continue;
    }

    // No ready job - find when the next one is due
    const next = await col.findOne(
      { status: 'scheduled' },
      { sort: { runAt: 1 }, projection: { runAt: 1 } }
    );

    const sleepMs = next
      ? Math.max(0, next.runAt.getTime() - Date.now())
      : 10_000; // Default poll every 10 seconds if queue is empty

    await new Promise((r) => setTimeout(r, Math.min(sleepMs, 10_000)));
  }
}
```

## Summary

A MongoDB delayed queue stores a `runAt` timestamp on each job and uses `findOneAndUpdate` with `{ runAt: { $lte: now } }` to fetch only jobs whose time has passed. A smart worker sleeps until the next scheduled job is due rather than polling at a fixed rate, and failed jobs can be rescheduled with exponential back-off by simply updating `runAt` to a future time.
