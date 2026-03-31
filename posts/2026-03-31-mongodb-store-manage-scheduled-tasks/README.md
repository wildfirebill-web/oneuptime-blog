# How to Store and Manage Scheduled Tasks in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Scheduling, Task, Database, Automation

Description: Learn how to design a MongoDB collection for storing and managing scheduled tasks, with CRUD operations, status tracking, and distributed locking.

---

## Designing the Scheduled Tasks Collection

A well-designed task schema must capture the schedule, execution state, locking, and history to support both single-instance and distributed deployments.

```javascript
// models/ScheduledTask.js
const mongoose = require('mongoose');

const ScheduledTaskSchema = new mongoose.Schema({
  // Identity
  name: { type: String, required: true },
  description: String,

  // Schedule
  scheduleType: {
    type: String,
    enum: ['once', 'recurring', 'interval'],
    required: true
  },
  cronExpression: String,          // for recurring (e.g., "0 9 * * 1")
  intervalMs: Number,              // for interval (e.g., 300000 = 5 min)
  runAt: Date,                     // for once
  nextRunAt: { type: Date, index: true },

  // Execution Config
  handler: { type: String, required: true },  // handler name to invoke
  payload: mongoose.Schema.Types.Mixed,        // data to pass to handler
  timeoutMs: { type: Number, default: 60000 },
  maxRetries: { type: Number, default: 3 },

  // Status
  enabled: { type: Boolean, default: true, index: true },
  status: {
    type: String,
    enum: ['pending', 'running', 'completed', 'failed', 'disabled'],
    default: 'pending'
  },

  // Locking (for distributed execution)
  lockedAt: Date,
  lockedBy: String,

  // History
  lastRunAt: Date,
  lastRunStatus: { type: String, enum: ['success', 'failure'] },
  lastRunDurationMs: Number,
  lastRunError: String,
  runCount: { type: Number, default: 0 },
  failCount: { type: Number, default: 0 },

  // Metadata
  createdAt: { type: Date, default: Date.now },
  updatedAt: Date,
  createdBy: String,
  tags: [String]
});

// Index for fast polling
ScheduledTaskSchema.index({ enabled: 1, nextRunAt: 1, lockedAt: 1 });

module.exports = mongoose.model('ScheduledTask', ScheduledTaskSchema);
```

## CRUD Operations for Task Management

```javascript
// services/taskManager.js
const parser = require('cron-parser');
const ScheduledTask = require('../models/ScheduledTask');

class TaskManager {
  async create(taskDef) {
    const nextRunAt = this._computeNextRun(taskDef);
    return await ScheduledTask.create({ ...taskDef, nextRunAt });
  }

  async update(taskId, updates) {
    if (updates.cronExpression || updates.runAt || updates.intervalMs) {
      updates.nextRunAt = this._computeNextRun(updates);
    }
    updates.updatedAt = new Date();
    return await ScheduledTask.findByIdAndUpdate(taskId, updates, { new: true });
  }

  async disable(taskId) {
    return await ScheduledTask.findByIdAndUpdate(
      taskId,
      { enabled: false, status: 'disabled' },
      { new: true }
    );
  }

  async enable(taskId) {
    const task = await ScheduledTask.findById(taskId);
    const nextRunAt = this._computeNextRun(task);
    return await ScheduledTask.findByIdAndUpdate(
      taskId,
      { enabled: true, status: 'pending', nextRunAt },
      { new: true }
    );
  }

  async delete(taskId) {
    return await ScheduledTask.findByIdAndDelete(taskId);
  }

  async list({ enabled, handler, tags, page = 1, limit = 20 } = {}) {
    const filter = {};
    if (enabled !== undefined) filter.enabled = enabled;
    if (handler) filter.handler = handler;
    if (tags?.length) filter.tags = { $in: tags };

    const [tasks, total] = await Promise.all([
      ScheduledTask.find(filter)
        .sort({ nextRunAt: 1 })
        .skip((page - 1) * limit)
        .limit(limit),
      ScheduledTask.countDocuments(filter)
    ]);

    return { tasks, total, page, pages: Math.ceil(total / limit) };
  }

  _computeNextRun(taskDef) {
    if (taskDef.scheduleType === 'once') return taskDef.runAt;
    if (taskDef.scheduleType === 'interval') {
      return new Date(Date.now() + taskDef.intervalMs);
    }
    if (taskDef.cronExpression) {
      return parser.parseExpression(taskDef.cronExpression).next().toDate();
    }
    return null;
  }
}

module.exports = new TaskManager();
```

## Querying Task Health

```javascript
// Get overdue tasks (should have run but haven't)
const overdue = await ScheduledTask.find({
  enabled: true,
  nextRunAt: { $lt: new Date(Date.now() - 10 * 60 * 1000) },  // 10 min overdue
  lockedAt: null
});

// Get tasks with high failure rates
const problematic = await ScheduledTask.aggregate([
  { $match: { runCount: { $gt: 5 } } },
  { $addFields: {
    failureRate: { $divide: ['$failCount', '$runCount'] }
  }},
  { $match: { failureRate: { $gt: 0.2 } } },  // >20% failure
  { $sort: { failureRate: -1 } }
]);

// Tasks running longer than expected (potential hangs)
const LOCK_TIMEOUT_MIN = 10;
const hung = await ScheduledTask.find({
  status: 'running',
  lockedAt: { $lt: new Date(Date.now() - LOCK_TIMEOUT_MIN * 60 * 1000) }
});
```

## Summary

A MongoDB-backed scheduled task store provides flexibility beyond simple cron jobs - each task carries its own configuration, payload, retry policy, and full execution history. Design your schema with appropriate indexes for fast polling, implement atomic locking for distributed safety, and expose CRUD management operations so tasks can be enabled, disabled, and updated without code deployments. Monitor overdue and high-failure-rate tasks as leading indicators of scheduler health.
