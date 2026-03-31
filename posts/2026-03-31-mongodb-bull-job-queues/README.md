# How to Use Bull with MongoDB for Job Queues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Bull, Queue, Node.js, Redis

Description: Learn how to combine Bull job queues (backed by Redis) with MongoDB for persistent job data storage and result tracking in Node.js applications.

---

## Bull and MongoDB: Complementary Roles

Bull is a Node.js queue library that uses Redis for fast job dispatch and state management. MongoDB complements Bull by storing job payloads, results, and business data that outlive the queue's Redis TTL. The pattern is: Bull handles queue mechanics (rate limiting, retry, priority), MongoDB stores the actual business data and long-term audit history.

## Architecture

```text
Application --> Bull Queue (Redis) --> Worker Process
                                           |
                              MongoDB: job results, business data
```

## Installation

```bash
npm install bull mongoose ioredis
```

## Setting Up Bull with MongoDB Result Storage

```javascript
// queues/emailQueue.js
const Bull = require('bull');
const mongoose = require('mongoose');

// Bull queue backed by Redis
const emailQueue = new Bull('email-queue', {
  redis: {
    host: process.env.REDIS_HOST || 'localhost',
    port: process.env.REDIS_PORT || 6379
  },
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 5000
    },
    removeOnComplete: 100,  // keep last 100 completed jobs in Redis
    removeOnFail: 200
  }
});

module.exports = emailQueue;
```

## MongoDB Model for Job Tracking

```javascript
// models/JobRecord.js
const mongoose = require('mongoose');

const JobRecordSchema = new mongoose.Schema({
  bullJobId: { type: String, index: true },
  queueName: String,
  type: String,
  payload: mongoose.Schema.Types.Mixed,
  status: {
    type: String,
    enum: ['queued', 'processing', 'completed', 'failed'],
    default: 'queued'
  },
  result: mongoose.Schema.Types.Mixed,
  error: String,
  attempts: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now },
  completedAt: Date
});

module.exports = mongoose.model('JobRecord', JobRecordSchema);
```

## Worker with MongoDB Result Persistence

```javascript
// workers/emailWorker.js
const emailQueue = require('../queues/emailQueue');
const JobRecord = require('../models/JobRecord');

emailQueue.process('send-email', 5, async (job) => {
  const { to, subject, body, recordId } = job.data;

  // Update MongoDB status to processing
  await JobRecord.findByIdAndUpdate(recordId, {
    status: 'processing',
    $inc: { attempts: 1 }
  });

  // Perform the actual work
  const result = await sendEmail({ to, subject, body });

  // Persist result to MongoDB
  await JobRecord.findByIdAndUpdate(recordId, {
    status: 'completed',
    result,
    completedAt: new Date()
  });

  return result;
});

// Handle failures
emailQueue.on('failed', async (job, err) => {
  const { recordId } = job.data;
  if (job.attemptsMade >= job.opts.attempts) {
    await JobRecord.findByIdAndUpdate(recordId, {
      status: 'failed',
      error: err.message
    });
  }
});

console.log('Email worker started');
```

## Enqueuing Jobs with MongoDB Record Creation

```javascript
// services/emailService.js
const emailQueue = require('../queues/emailQueue');
const JobRecord = require('../models/JobRecord');

async function queueEmail({ to, subject, body }) {
  // Create MongoDB record first
  const record = await JobRecord.create({
    queueName: 'email-queue',
    type: 'send-email',
    payload: { to, subject, body }
  });

  // Enqueue in Bull with MongoDB record ID
  const job = await emailQueue.add('send-email', {
    to,
    subject,
    body,
    recordId: record._id
  });

  // Update record with Bull job ID
  await JobRecord.findByIdAndUpdate(record._id, {
    bullJobId: job.id
  });

  return record;
}

module.exports = { queueEmail };
```

## Querying Job History from MongoDB

```javascript
// Query recent failed email jobs
const failures = await JobRecord.find({
  queueName: 'email-queue',
  status: 'failed',
  createdAt: { $gte: new Date(Date.now() - 24 * 60 * 60 * 1000) }
}).sort({ createdAt: -1 }).limit(50);

// Aggregate job success rate by type
const stats = await JobRecord.aggregate([
  { $match: { createdAt: { $gte: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000) } } },
  { $group: {
    _id: { type: '$type', status: '$status' },
    count: { $sum: 1 }
  }},
  { $sort: { '_id.type': 1 } }
]);
```

## Summary

Bull and MongoDB serve complementary roles in a job queue system. Bull provides fast Redis-backed queue management with retry, rate limiting, and job prioritization. MongoDB stores the authoritative record of job payloads, results, and audit history with rich querying capabilities. Create a MongoDB record when enqueueing, update it at each lifecycle event, and use MongoDB aggregation to track queue health metrics over time.
