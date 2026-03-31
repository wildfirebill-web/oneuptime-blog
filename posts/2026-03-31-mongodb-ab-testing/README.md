# How to Implement A/B Testing with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, A/B Testing, Experiment, Analytics, Feature

Description: Learn how to build an A/B testing system with MongoDB to manage experiments, assign users to variants, and track conversion metrics with aggregation.

---

A/B testing lets you compare variants of a feature by exposing different user groups to each version and measuring outcomes. MongoDB stores experiment configurations, handles user-to-variant assignments, and tracks conversion events for analysis.

## Experiment Schema

```javascript
const mongoose = require('mongoose');

const experimentSchema = new mongoose.Schema({
  key: { type: String, required: true, unique: true, index: true },
  name: { type: String, required: true },
  status: { type: String, enum: ['draft', 'running', 'paused', 'completed'], default: 'draft' },
  variants: [{
    key: { type: String, required: true },
    name: String,
    weight: { type: Number, required: true }, // e.g., 50 for 50%
    config: mongoose.Schema.Types.Mixed,       // Variant-specific config
  }],
  targetAudience: {
    percentage: { type: Number, default: 100, min: 0, max: 100 },
  },
  startDate: Date,
  endDate: Date,
  createdAt: { type: Date, default: Date.now },
});

const Experiment = mongoose.model('Experiment', experimentSchema);
```

## Assignment Schema

Track which variant each user is assigned to:

```javascript
const assignmentSchema = new mongoose.Schema({
  userId: { type: String, required: true, index: true },
  experimentKey: { type: String, required: true, index: true },
  variantKey: { type: String, required: true },
  assignedAt: { type: Date, default: Date.now },
});

assignmentSchema.index({ userId: 1, experimentKey: 1 }, { unique: true });

const Assignment = mongoose.model('Assignment', assignmentSchema);
```

## Event (Conversion) Schema

```javascript
const conversionEventSchema = new mongoose.Schema({
  userId: { type: String, required: true, index: true },
  experimentKey: { type: String, required: true, index: true },
  variantKey: { type: String, required: true },
  eventType: { type: String, required: true }, // e.g., 'click', 'purchase', 'signup'
  value: Number, // for revenue tracking
  metadata: mongoose.Schema.Types.Mixed,
  createdAt: { type: Date, default: Date.now, index: true },
});

const ConversionEvent = mongoose.model('ConversionEvent', conversionEventSchema);
```

## Variant Assignment Logic

Deterministic assignment ensures a user always gets the same variant:

```javascript
function assignVariant(userId, experiment) {
  // Hash user + experiment for deterministic assignment
  const hash = simpleHash(`${userId}:${experiment.key}`);
  const bucket = hash % 100;

  // Check if user is in the target audience
  if (bucket >= experiment.targetAudience.percentage) {
    return null; // Not in experiment
  }

  // Assign based on cumulative weights
  const totalWeight = experiment.variants.reduce((s, v) => s + v.weight, 0);
  const variantBucket = hash % totalWeight;
  let cumulative = 0;

  for (const variant of experiment.variants) {
    cumulative += variant.weight;
    if (variantBucket < cumulative) return variant.key;
  }

  return experiment.variants[0].key;
}

function simpleHash(str) {
  let h = 0;
  for (let i = 0; i < str.length; i++) {
    h = Math.imul(31, h) + str.charCodeAt(i) | 0;
  }
  return Math.abs(h);
}
```

## Getting or Creating an Assignment

```javascript
async function getOrAssignVariant(userId, experimentKey) {
  // Check existing assignment first
  const existing = await Assignment.findOne({ userId, experimentKey }).lean();
  if (existing) return existing.variantKey;

  const experiment = await Experiment.findOne({ key: experimentKey, status: 'running' }).lean();
  if (!experiment) return null;

  const variantKey = assignVariant(userId, experiment);
  if (!variantKey) return null;

  await Assignment.findOneAndUpdate(
    { userId, experimentKey },
    { $setOnInsert: { userId, experimentKey, variantKey } },
    { upsert: true }
  );

  return variantKey;
}
```

## Tracking Conversions

```javascript
async function trackConversion(userId, experimentKey, eventType, value = null) {
  const variantKey = await getOrAssignVariant(userId, experimentKey);
  if (!variantKey) return;

  await ConversionEvent.create({ userId, experimentKey, variantKey, eventType, value });
}
```

## Analyzing Results with Aggregation

```javascript
// Conversion rate per variant
const results = await ConversionEvent.aggregate([
  { $match: { experimentKey: 'checkout_button_color', eventType: 'purchase' } },
  { $group: { _id: '$variantKey', conversions: { $sum: 1 }, revenue: { $sum: '$value' } } },
]);

// Unique users assigned per variant
const assignments = await Assignment.aggregate([
  { $match: { experimentKey: 'checkout_button_color' } },
  { $group: { _id: '$variantKey', users: { $sum: 1 } } },
]);

console.log('Results:', results);
console.log('Assignments:', assignments);
```

## Summary

MongoDB's flexible document model handles experiment configurations, deterministic variant assignments, and conversion tracking in a single, queryable store. Use a hash of `userId + experimentKey` for deterministic, sticky assignment without storing assignments upfront. Aggregate conversion events by variant to compute conversion rates, and compound index on `userId + experimentKey` to make assignment lookups fast.
