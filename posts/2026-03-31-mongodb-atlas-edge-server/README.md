# How to Use MongoDB at the Edge with Atlas Edge Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, Edge Server, Edge Computing, Sync

Description: Learn how to deploy MongoDB Atlas Edge Server at the network edge to enable local data access, offline operation, and selective sync with Atlas clusters.

---

## Overview

Atlas Edge Server is a lightweight MongoDB-compatible server that runs at the network edge - inside a factory, retail location, or remote site. It syncs data bidirectionally with MongoDB Atlas, allowing edge applications to operate with full read/write capabilities even when the WAN connection is intermittent.

## Atlas Edge Server Architecture

```text
[Edge Devices / Apps]
        |
        v
[Atlas Edge Server]  <-- Local MongoDB-compatible API
        |
        | (Device Sync over HTTPS)
        v
[MongoDB Atlas Cluster]  <-- Source of truth
```

Edge applications connect to the local Edge Server using the standard MongoDB driver or Realm SDK. The Edge Server handles sync with Atlas automatically.

## Installing Atlas Edge Server

Atlas Edge Server runs as a Docker container:

```bash
docker pull mongodb/atlas-edge-server:latest

docker run -d \
  --name atlas-edge-server \
  -p 27021:27021 \
  -e EDGE_SERVER_APP_ID="your-app-services-app-id" \
  -e EDGE_SERVER_AUTH_SECRET="your-secret" \
  -v /data/edge:/data \
  mongodb/atlas-edge-server:latest
```

## Connecting an Application to Edge Server

Edge Server exposes a MongoDB wire protocol endpoint. Connect using the standard driver:

```javascript
const { MongoClient } = require('mongodb');

// Point to Edge Server instead of Atlas
const client = new MongoClient('mongodb://localhost:27021');
await client.connect();

const db = client.db('myapp');
const orders = db.collection('orders');

// Write locally - Edge Server syncs to Atlas
await orders.insertOne({
  _id: new ObjectId(),
  productId: 'prod-001',
  quantity: 5,
  locationId: 'warehouse-seattle',
  createdAt: new Date(),
});
```

## Configuring Sync Rules

In Atlas App Services, configure which collections the Edge Server syncs:

```json
{
  "sync": {
    "state": "enabled",
    "database_name": "myapp",
    "collection_name": "orders",
    "queryable_fields_names": ["locationId", "status"]
  }
}
```

The `queryable_fields_names` list controls which fields can be used in Flexible Sync subscriptions.

## Subscribing to a Subset of Data

Edge Server supports Flexible Sync subscriptions so each edge location only syncs its own data:

```javascript
// Only sync orders for this edge location
const session = client.startSession();
await edgeClient.db('__realm_sync').collection('subscriptions').insertOne({
  query: { locationId: 'warehouse-seattle' },
  collection: 'orders',
});
```

## Monitoring Edge Server Status

Check sync status via the Edge Server management API:

```bash
curl http://localhost:27021/__admin/v1/status
```

Response example:

```json
{
  "sync_state": "connected",
  "last_synced_at": "2026-03-31T09:55:00Z",
  "pending_upload_count": 0,
  "pending_download_count": 0
}
```

## Handling Offline Operation

When the Edge Server loses WAN connectivity, local reads and writes continue uninterrupted. Changes are queued locally and synced to Atlas when the connection is restored. Monitor the `pending_upload_count` in the status API to know how far behind the edge is.

## Best Practices

- Deploy Edge Server on a dedicated host or VM at each edge location, not on the same machine as application servers, to isolate failures.
- Use `locationId` or `siteId` as the primary sync filter field to ensure each edge location only syncs its own data subset.
- Monitor the edge server's sync lag using the management API and alert when `pending_upload_count` exceeds a threshold.
- Back up the local Edge Server data directory (`/data/edge`) to a local NAS for recovery if the edge host fails before syncing.

## Summary

Atlas Edge Server brings MongoDB-compatible data access to the network edge with automatic bidirectional sync to Atlas. Applications connect locally for low-latency reads and writes, while sync handles data propagation when connectivity is available.
