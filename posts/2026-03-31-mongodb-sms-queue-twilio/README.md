# How to Build an SMS Queue with MongoDB and Twilio

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, SMS, Queue, Twilio, Notification

Description: Build a durable SMS queue backed by MongoDB with Twilio delivery, retry logic, and per-recipient rate limiting to avoid carrier throttling.

---

## Overview

A MongoDB-backed SMS queue stores outgoing messages as documents and processes them with a worker that calls the Twilio API. Using MongoDB as the queue store provides durability, retry logic, and a full delivery audit trail. Rate limiting per recipient prevents carrier throttling and ensures compliance with messaging regulations.

## SMS Queue Document Schema

```javascript
const smsQueueItem = {
  to: "+14155552671",
  body: "Your verification code is 123456",
  from: "+14155550100",
  status: "pending",
  attempts: 0,
  maxAttempts: 3,
  scheduledAt: new Date(),
  lockedAt: null,
  lockedBy: null,
  twilioSid: null,
  sentAt: null,
  lastError: null,
  tags: { type: "verification", userId: "user_001" },
  createdAt: new Date(),
};
```

## Enqueueing an SMS

```javascript
async function enqueueSms(db, { to, body, from, delayMs = 0, tags = {} }) {
  return db.collection("sms_queue").insertOne({
    to,
    body,
    from: from || process.env.TWILIO_FROM_NUMBER,
    status: "pending",
    attempts: 0,
    maxAttempts: 3,
    scheduledAt: new Date(Date.now() + delayMs),
    lockedAt: null,
    lockedBy: null,
    twilioSid: null,
    sentAt: null,
    lastError: null,
    tags,
    createdAt: new Date(),
  });
}
```

## Per-Recipient Rate Limiting

Prevent sending more than N messages to the same number within a time window.

```javascript
async function isRateLimited(db, to, maxPerHour = 5) {
  const oneHourAgo = new Date(Date.now() - 60 * 60 * 1000);
  const recentCount = await db.collection("sms_queue").countDocuments({
    to,
    status: { $in: ["sent", "pending", "processing"] },
    createdAt: { $gte: oneHourAgo },
  });
  return recentCount >= maxPerHour;
}
```

## Atomic Job Claiming and Twilio Delivery

```javascript
const twilio = require("twilio");
const client = twilio(process.env.TWILIO_ACCOUNT_SID, process.env.TWILIO_AUTH_TOKEN);
const workerId = `sms-worker-${process.pid}`;

async function processNextSms(db) {
  const lockExpiry = new Date(Date.now() - 5 * 60 * 1000);

  const job = await db.collection("sms_queue").findOneAndUpdate(
    {
      status: "pending",
      scheduledAt: { $lte: new Date() },
      $or: [{ lockedAt: null }, { lockedAt: { $lt: lockExpiry } }],
    },
    {
      $set: { status: "processing", lockedAt: new Date(), lockedBy: workerId },
      $inc: { attempts: 1 },
    },
    { returnDocument: "after", sort: { scheduledAt: 1 } }
  );

  if (!job) return false;

  try {
    const message = await client.messages.create({
      to: job.to,
      from: job.from,
      body: job.body,
    });

    await db.collection("sms_queue").updateOne(
      { _id: job._id },
      {
        $set: {
          status: "sent",
          twilioSid: message.sid,
          sentAt: new Date(),
          lockedAt: null,
          lockedBy: null,
        },
      }
    );
  } catch (err) {
    const isDead = job.attempts >= job.maxAttempts;
    const backoffMs = Math.pow(2, job.attempts) * 60000;
    await db.collection("sms_queue").updateOne(
      { _id: job._id },
      {
        $set: {
          status: isDead ? "dead" : "pending",
          lastError: err.message,
          lockedAt: null,
          lockedBy: null,
          scheduledAt: isDead ? job.scheduledAt : new Date(Date.now() + backoffMs),
        },
      }
    );
  }

  return true;
}
```

## Indexes

```javascript
await db.collection("sms_queue").createIndex(
  { status: 1, scheduledAt: 1 },
  { name: "idx_sms_worker" }
);
await db.collection("sms_queue").createIndex(
  { to: 1, createdAt: -1 },
  { name: "idx_sms_rate_limit" }
);
await db.collection("sms_queue").createIndex(
  { sentAt: 1 },
  { expireAfterSeconds: 60 * 24 * 3600, partialFilterExpression: { status: "sent" } }
);
```

## Summary

A MongoDB SMS queue uses atomic `findOneAndUpdate` for race-condition-free job claiming, exponential backoff for Twilio API failures, and a per-recipient rate limiter to prevent carrier throttling. Index on `status` and `scheduledAt` for efficient worker polling, and add a TTL index on `sentAt` to auto-expire delivered messages after a retention period.
