# How to Store Application Configuration in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Configuration, Settings, Change Stream, Cache

Description: Learn how to store dynamic application configuration in MongoDB with real-time updates via change streams, type coercion, and environment-specific overrides.

---

Some application settings need to change at runtime without a deployment: email templates, rate limits, maintenance windows, or pricing tiers. Storing configuration in MongoDB gives you a queryable, auditable, live-updatable configuration store.

## Configuration Schema

```javascript
const mongoose = require('mongoose');

const configSchema = new mongoose.Schema({
  key: { type: String, required: true, index: true },
  value: mongoose.Schema.Types.Mixed,
  type: { type: String, enum: ['string', 'number', 'boolean', 'json'], default: 'string' },
  environment: { type: String, default: 'all', index: true },
  description: String,
  updatedBy: String,
  updatedAt: { type: Date, default: Date.now },
  history: [{
    value: mongoose.Schema.Types.Mixed,
    changedAt: { type: Date, default: Date.now },
    changedBy: String,
  }],
});

// Unique per key + environment
configSchema.index({ key: 1, environment: 1 }, { unique: true });

module.exports = mongoose.model('Config', configSchema);
```

## Seeding Default Configuration

```javascript
const Config = require('./models/Config');

const defaults = [
  { key: 'max_file_upload_mb', value: 10, type: 'number', description: 'Max file upload size in MB' },
  { key: 'maintenance_mode', value: false, type: 'boolean', description: 'Enable maintenance page' },
  { key: 'supported_currencies', value: ['USD', 'EUR', 'GBP'], type: 'json', description: 'Accepted currencies' },
  { key: 'welcome_message', value: 'Welcome!', type: 'string', description: 'Homepage greeting' },
];

async function seedDefaults() {
  for (const item of defaults) {
    await Config.findOneAndUpdate(
      { key: item.key, environment: 'all' },
      { $setOnInsert: item },
      { upsert: true }
    );
  }
}
```

## Configuration Service with In-Memory Cache

```javascript
class ConfigService {
  constructor() {
    this.cache = new Map();
    this.lastLoad = 0;
    this.TTL_MS = 30000; // 30 second cache
  }

  async loadAll() {
    const env = process.env.NODE_ENV || 'development';

    const configs = await Config.find({
      environment: { $in: ['all', env] },
    }).lean();

    this.cache.clear();

    // Environment-specific values override 'all' values
    for (const item of configs) {
      if (item.environment === 'all' && !this.cache.has(item.key)) {
        this.cache.set(item.key, item);
      } else if (item.environment === env) {
        this.cache.set(item.key, item); // Override
      }
    }

    this.lastLoad = Date.now();
  }

  async get(key, defaultValue = null) {
    if (Date.now() - this.lastLoad > this.TTL_MS) {
      await this.loadAll();
    }

    const item = this.cache.get(key);
    if (!item) return defaultValue;

    return this.coerce(item.value, item.type);
  }

  coerce(value, type) {
    switch (type) {
      case 'number': return Number(value);
      case 'boolean': return Boolean(value);
      case 'json': return typeof value === 'string' ? JSON.parse(value) : value;
      default: return String(value);
    }
  }
}

const configService = new ConfigService();
```

## Real-Time Updates via Change Streams

```javascript
function watchConfigChanges() {
  const stream = Config.watch();
  stream.on('change', async () => {
    console.log('Configuration changed - reloading cache');
    await configService.loadAll();
  });
}

mongoose.connection.once('open', () => {
  configService.loadAll();
  watchConfigChanges();
});
```

## Updating Configuration with History

```javascript
async function updateConfig(key, newValue, changedBy) {
  const current = await Config.findOne({ key, environment: 'all' });

  await Config.findOneAndUpdate(
    { key, environment: 'all' },
    {
      $set: { value: newValue, updatedBy: changedBy, updatedAt: new Date() },
      $push: {
        history: {
          $each: [{ value: current?.value, changedAt: new Date(), changedBy }],
          $slice: -20, // Keep last 20 history entries
        },
      },
    },
    { upsert: true }
  );
}
```

## Using Configuration in Routes

```javascript
app.use(async (req, res, next) => {
  const maintenance = await configService.get('maintenance_mode', false);
  if (maintenance && !req.path.startsWith('/admin')) {
    return res.status(503).json({ error: 'Service is under maintenance' });
  }
  next();
});

app.post('/api/upload', async (req, res) => {
  const maxMB = await configService.get('max_file_upload_mb', 10);
  const fileSizeMB = req.headers['content-length'] / (1024 * 1024);

  if (fileSizeMB > maxMB) {
    return res.status(413).json({ error: `File too large. Max ${maxMB}MB` });
  }
  // ... handle upload
});
```

## Summary

MongoDB is a practical configuration store when you need runtime updates without redeployments. Use an in-memory cache with a short TTL for performance, and change streams for instant cache invalidation when a value changes. Store change history with `$push` + `$slice` to keep an audit trail, and support environment-specific overrides with a compound index on `key + environment`.
