# How to Implement Feature Flags with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Feature Flag, Configuration, Toggle, Deployment

Description: Learn how to build a MongoDB-backed feature flag system with percentage rollouts, user targeting, environment-specific flags, and real-time toggle updates.

---

Feature flags let you deploy code to production without enabling it for users. A MongoDB-backed feature flag system gives you dynamic runtime control - you can enable or disable features without redeploying the application.

## Feature Flag Schema

```javascript
const mongoose = require('mongoose');

const featureFlagSchema = new mongoose.Schema({
  key: { type: String, required: true, unique: true, index: true },
  name: { type: String, required: true },
  description: String,
  enabled: { type: Boolean, default: false },
  environments: {
    production: { type: Boolean, default: false },
    staging: { type: Boolean, default: false },
    development: { type: Boolean, default: true },
  },
  // Percentage rollout (0-100)
  rolloutPercentage: { type: Number, default: 0, min: 0, max: 100 },
  // Specific user IDs that always get the flag
  allowedUsers: [{ type: String }],
  // User attributes for targeting
  targeting: [{
    attribute: String,
    operator: { type: String, enum: ['equals', 'contains', 'in'] },
    value: mongoose.Schema.Types.Mixed,
  }],
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now },
});

module.exports = mongoose.model('FeatureFlag', featureFlagSchema);
```

## Feature Flag Service

```javascript
const FeatureFlag = require('./models/FeatureFlag');

// Cache flags in memory, refresh every 60 seconds
let flagCache = new Map();
let lastCacheTime = 0;
const CACHE_TTL_MS = 60000;

async function getAllFlags() {
  const now = Date.now();
  if (now - lastCacheTime < CACHE_TTL_MS && flagCache.size > 0) {
    return flagCache;
  }

  const flags = await FeatureFlag.find().lean();
  flagCache = new Map(flags.map((f) => [f.key, f]));
  lastCacheTime = now;
  return flagCache;
}

async function isEnabled(flagKey, user = null) {
  const flags = await getAllFlags();
  const flag = flags.get(flagKey);

  if (!flag) return false;

  const env = process.env.NODE_ENV || 'development';
  if (!flag.environments[env]) return false;
  if (!flag.enabled) return false;

  // Specific user override
  if (user && flag.allowedUsers.includes(user.id)) return true;

  // Targeting rules
  if (user && flag.targeting.length > 0) {
    const matches = flag.targeting.every((rule) => {
      const attrValue = user[rule.attribute];
      if (rule.operator === 'equals') return attrValue === rule.value;
      if (rule.operator === 'contains') return String(attrValue).includes(rule.value);
      if (rule.operator === 'in') return rule.value.includes(attrValue);
      return false;
    });
    if (matches) return true;
  }

  // Percentage rollout - deterministic by user ID
  if (flag.rolloutPercentage > 0 && user) {
    const hash = hashUserId(user.id);
    return (hash % 100) < flag.rolloutPercentage;
  }

  return flag.rolloutPercentage === 100;
}

function hashUserId(userId) {
  let hash = 0;
  for (let i = 0; i < userId.length; i++) {
    hash = ((hash << 5) - hash) + userId.charCodeAt(i);
    hash |= 0;
  }
  return Math.abs(hash);
}
```

## Express Middleware

```javascript
async function featureFlagMiddleware(req, res, next) {
  req.isFeatureEnabled = (flagKey) => isEnabled(flagKey, req.user);
  next();
}

app.use(featureFlagMiddleware);

app.get('/api/checkout', async (req, res) => {
  const useNewCheckout = await req.isFeatureEnabled('new_checkout_flow');
  if (useNewCheckout) {
    return newCheckoutController(req, res);
  }
  return legacyCheckoutController(req, res);
});
```

## Admin API to Manage Flags

```javascript
// Enable a flag for all users
app.patch('/admin/flags/:key', async (req, res) => {
  const flag = await FeatureFlag.findOneAndUpdate(
    { key: req.params.key },
    { $set: { ...req.body, updatedAt: new Date() } },
    { new: true }
  );

  // Invalidate cache
  flagCache.clear();
  lastCacheTime = 0;

  res.json(flag);
});

// Create a new flag
app.post('/admin/flags', async (req, res) => {
  const flag = await FeatureFlag.create(req.body);
  res.status(201).json(flag);
});
```

## Using Change Streams for Instant Updates

```javascript
function watchFlagChanges() {
  const stream = FeatureFlag.watch();
  stream.on('change', () => {
    flagCache.clear();
    lastCacheTime = 0;
    console.log('Feature flags cache invalidated');
  });
}

mongoose.connection.once('open', watchFlagChanges);
```

## Summary

A MongoDB-backed feature flag system requires only one collection and provides environment-specific flags, percentage rollouts, user targeting, and instant updates via change streams. Cache flags in memory with a TTL to keep latency low on every request, and use change streams to invalidate the cache immediately when a flag is updated in MongoDB.
