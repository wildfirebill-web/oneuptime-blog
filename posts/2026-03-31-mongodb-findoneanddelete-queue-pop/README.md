# How to Use findOneAndDelete to Implement a Queue Pop in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, findOneAndDelete, Queue, Worker, Atomic

Description: Learn how to use MongoDB's findOneAndDelete to implement an atomic queue pop pattern for reliable job processing without duplicate execution.

---

## Overview

`findOneAndDelete` atomically finds a document matching a filter, deletes it, and returns the deleted document. This atomic behavior makes it the ideal tool for implementing a work queue in MongoDB - a worker claims and removes a job in one operation, ensuring no other worker can claim the same job.

## Job Schema

```javascript
// models/Job.js
const mongoose = require('mongoose');

const jobSchema = new mongoose.Schema({
  type:      { type: String, required: true, index: true },
  payload:   { type: mongoose.Schema.Types.Mixed },
  priority:  { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now, index: true },
  runAt:     { type: Date, default: Date.now },  // Support deferred jobs
});

jobSchema.index({ type: 1, priority: -1, createdAt: 1 });

module.exports = mongoose.model('Job', jobSchema);
```

## Popping the Next Job

```javascript
async function popNextJob(type) {
  // Atomically claim and remove the highest-priority oldest job
  const job = await Job.findOneAndDelete(
    {
      type,
      runAt: { $lte: new Date() }, // Only pick up jobs ready to run
    },
    {
      sort: { priority: -1, createdAt: 1 }, // Highest priority, oldest first
    }
  );

  return job; // null if no job is available
}
```

Because the delete is atomic, no two workers can claim the same job even if they run simultaneously.

## Enqueuing Jobs

```javascript
async function enqueue(type, payload, options = {}) {
  const job = await Job.create({
    type,
    payload,
    priority: options.priority ?? 0,
    runAt: options.runAt ?? new Date(),
  });
  return job;
}

// Usage
await enqueue('send-email', { to: 'user@example.com', subject: 'Welcome' });
await enqueue('resize-image', { imageId: '123' }, { priority: 5 });
await enqueue('send-report', { reportId: '456' }, {
  runAt: new Date(Date.now() + 60 * 60 * 1000), // Run in 1 hour
});
```

## Worker Loop

```javascript
async function worker(jobType, handler, pollIntervalMs = 1000) {
  console.log(`Worker started for job type: ${jobType}`);

  while (true) {
    const job = await popNextJob(jobType);

    if (!job) {
      // No jobs available - wait before polling again
      await new Promise(resolve => setTimeout(resolve, pollIntervalMs));
      continue;
    }

    console.log(`Processing job ${job._id}`);

    try {
      await handler(job.payload);
      console.log(`Job ${job._id} completed`);
    } catch (err) {
      console.error(`Job ${job._id} failed:`, err.message);
      // Re-enqueue with a delay for retry
      await enqueue(job.type, job.payload, {
        priority: job.priority - 1,
        runAt: new Date(Date.now() + 30_000), // Retry in 30 seconds
      });
    }
  }
}

// Start workers
worker('send-email', async (payload) => {
  await sendEmail(payload);
});
```

## Dead Letter Queue

For jobs that fail repeatedly, move them to a dead letter queue instead of re-enqueuing indefinitely:

```javascript
const deadLetterSchema = new mongoose.Schema({
  type:       String,
  payload:    mongoose.Schema.Types.Mixed,
  error:      String,
  failedAt:   { type: Date, default: Date.now },
});

const DeadLetter = mongoose.model('DeadLetter', deadLetterSchema);

async function moveToDeadLetter(job, error) {
  await DeadLetter.create({
    type:    job.type,
    payload: job.payload,
    error:   error.message,
  });
}
```

## Monitoring Queue Depth

```javascript
async function queueDepth(type) {
  return Job.countDocuments({ type, runAt: { $lte: new Date() } });
}
```

## Summary

`findOneAndDelete` is the cornerstone of a simple, reliable MongoDB-backed work queue. Its atomic find-and-delete semantics guarantee that each job is processed by exactly one worker. Use `sort` to implement priority queues, `runAt` for deferred execution, and re-enqueue on failure with decremented priority to implement retry logic. For a production queue at scale, consider a dedicated message broker, but for moderate workloads MongoDB handles this pattern well.
