# How to Sync Edge Data to MongoDB Atlas

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, Edge Computing, Sync, Data Pipeline

Description: Learn how to sync data collected at the edge to MongoDB Atlas using Atlas Device Sync, Atlas Edge Server, and custom change stream-based pipelines.

---

## Overview

Edge devices collect data locally and need to propagate it to MongoDB Atlas for centralized analytics, alerting, and long-term storage. There are three main patterns for syncing edge data to Atlas: Atlas Device Sync (for Realm-based apps), Atlas Edge Server (for MongoDB-compatible apps), and custom pipelines using change streams.

## Pattern 1 - Atlas Device Sync with Realm SDK

If your edge application uses the Realm SDK, Device Sync handles bi-directional sync automatically:

```javascript
const app = new Realm.App({ id: 'YOUR-APP-ID' });
const user = await app.logIn(Realm.Credentials.apiKey('edge-device-api-key'));

const config = {
  schema: [SensorReadingSchema],
  sync: {
    user,
    flexible: true,
    initialSubscriptions: {
      update(subs, realm) {
        subs.add(realm.objects('SensorReading').filtered('deviceId == $0', DEVICE_ID));
      },
    },
  },
};

const realm = await Realm.open(config);

// Write sensor data locally - syncs to Atlas automatically
realm.write(() => {
  realm.create('SensorReading', {
    _id: new Realm.BSON.ObjectId(),
    deviceId: DEVICE_ID,
    temperature: 22.5,
    humidity: 65.0,
    recordedAt: new Date(),
  });
});
```

## Pattern 2 - Atlas Edge Server with Bulk Upload

If running Atlas Edge Server, data written to the local MongoDB-compatible endpoint syncs to Atlas automatically. For batch collection scenarios:

```python
from pymongo import MongoClient
from datetime import datetime

# Connect to local Edge Server
edge_client = MongoClient("mongodb://localhost:27021")
edge_db = edge_client["iot_data"]
readings = edge_db["sensor_readings"]

# Collect and buffer readings
buffer = []
for i in range(100):
    buffer.append({
        "deviceId": "sensor-001",
        "value": read_sensor(),
        "timestamp": datetime.utcnow()
    })

# Insert all buffered readings at once
readings.insert_many(buffer)
print(f"Inserted {len(buffer)} readings - Edge Server will sync to Atlas")
```

## Pattern 3 - Custom Sync via Change Streams

For edge devices running a standalone MongoDB instance without Device Sync, watch for new documents and forward them to Atlas:

```javascript
const edgeClient = new MongoClient('mongodb://localhost:27017');
const atlasClient = new MongoClient(process.env.ATLAS_URI);

await edgeClient.connect();
await atlasClient.connect();

const edgeCollection = edgeClient.db('iot').collection('readings');
const atlasCollection = atlasClient.db('iot').collection('readings');

const changeStream = edgeCollection.watch([
  { $match: { operationType: 'insert' } }
]);

changeStream.on('change', async (change) => {
  const doc = change.fullDocument;
  try {
    await atlasCollection.insertOne(doc);
    // Mark as synced on the edge
    await edgeCollection.updateOne(
      { _id: doc._id },
      { $set: { syncedAt: new Date() } }
    );
  } catch (err) {
    console.error('Sync failed:', err.message);
  }
});
```

## Handling Sync Failures and Retry

Add a dead-letter queue for documents that fail to sync:

```javascript
async function syncWithRetry(atlasCollection, doc, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      await atlasCollection.replaceOne({ _id: doc._id }, doc, { upsert: true });
      return true;
    } catch (err) {
      if (attempt === maxRetries) {
        await dlqCollection.insertOne({ doc, error: err.message, failedAt: new Date() });
        return false;
      }
      await new Promise(r => setTimeout(r, 1000 * attempt));
    }
  }
}
```

## Monitoring Sync Lag

Track how far behind the edge is by comparing the latest synced document timestamp with the current time:

```javascript
const latest = await edgeCollection.findOne({ syncedAt: { $exists: true } }, { sort: { syncedAt: -1 } });
const lagSeconds = (Date.now() - latest.syncedAt.getTime()) / 1000;
console.log(`Sync lag: ${lagSeconds}s`);
```

## Best Practices

- Use `upsert: true` when writing to Atlas to safely handle duplicate deliveries.
- Buffer edge writes in batches and use `insertMany` to Atlas rather than one insert per document.
- Add an index on `syncedAt` on the edge collection so you can efficiently query unsynced documents.
- Store a `deviceId` and `locationId` on every document so Atlas queries can filter by edge source.

## Summary

Syncing edge data to MongoDB Atlas can be achieved with Atlas Device Sync for Realm apps, Atlas Edge Server for MongoDB-compatible apps, or a custom change stream forwarder for standalone edge MongoDB instances. Add retry logic and monitoring to ensure data integrity across intermittent connections.
