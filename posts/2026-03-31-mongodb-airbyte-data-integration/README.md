# How to Use MongoDB with Airbyte for Data Integration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Airbyte, ELT, Data Integration, Connector

Description: Learn how to configure MongoDB as a source or destination in Airbyte to sync documents to data warehouses or between databases.

---

## Why Airbyte with MongoDB?

Airbyte is an open-source ELT platform that syncs data between hundreds of sources and destinations. Using MongoDB with Airbyte allows you to replicate collections to Snowflake, BigQuery, Redshift, or PostgreSQL for analytics - without building custom pipelines.

## Installing Airbyte

```bash
git clone https://github.com/airbytehq/airbyte.git
cd airbyte
docker compose up
```

Access the UI at `http://localhost:8000`.

## Configuring MongoDB as a Source

### Source Setup in the UI

1. Go to Sources and click New Source
2. Search for and select MongoDB
3. Fill in the connection details:

```text
Host:              cluster.mongodb.net
Port:              27017
Database:          myDatabase
Username:          airbyte-reader
Password:          ********
Authentication Source: admin
SSL:               Enabled (for Atlas)
```

### Replica Set Requirement

Airbyte's MongoDB source connector requires MongoDB to be running as a replica set (or Atlas cluster). This is necessary for CDC-based incremental syncing using the oplog.

### Creating a Read-Only User for Airbyte

```javascript
db.createUser({
  user: "airbyte-reader",
  pwd: "AirbyteReadOnly123",
  roles: [
    { role: "read", db: "myDatabase" },
    { role: "read", db: "local" }
  ]
})
```

## Sync Modes

Airbyte supports several sync modes for MongoDB collections:

- **Full Refresh - Overwrite** - deletes all destination data and re-syncs
- **Full Refresh - Append** - appends all documents on each sync
- **Incremental - Append** - uses a cursor field to sync only new/changed documents
- **Incremental - Deduped + History** - CDC-based, maintains a deduplicated current state

For incremental sync, specify the cursor field:

```text
Cursor Field: updatedAt
```

## Configuring MongoDB as a Destination

Airbyte can also write data into MongoDB from other sources:

1. Go to Destinations and click New Destination
2. Select MongoDB
3. Configure:

```text
Host:      localhost
Port:      27017
Database:  airbyte_output
Username:  airbyte-writer
Password:  ********
```

Airbyte will create a collection per stream using the stream name.

## Setting Up a Connection

After creating source and destination:

1. Click Connections and then New Connection
2. Select your MongoDB source
3. Select your destination (e.g., BigQuery or Snowflake)
4. Choose which collections (streams) to sync
5. Set the sync schedule:

```text
Schedule: Every 6 hours
Sync Mode: Incremental - Append
```

## Handling Document Schema

MongoDB documents are schemaless, so Airbyte infers the schema by sampling documents. You can override this in the connection settings and specify a custom JSON schema per stream if the inferred schema is inconsistent.

## Monitoring Sync Status

```bash
# Check Airbyte server logs
docker logs airbyte-server

# Or use the Airbyte API
curl http://localhost:8001/api/v1/jobs/list \
  -H "Content-Type: application/json" \
  -d '{"configId": "<connection-id>", "configTypes": ["sync"]}'
```

## Example: MongoDB to BigQuery Pipeline

```text
Source:      MongoDB (myDatabase.orders)
Destination: BigQuery (analytics.orders)
Sync Mode:   Incremental - Append
Cursor:      createdAt
Schedule:    Every hour
```

After the first full sync, Airbyte only fetches documents where `createdAt > last_sync_time`, keeping the pipeline efficient as the collection grows.

## Summary

Airbyte makes it straightforward to replicate MongoDB collections to data warehouses or other databases. Configure a read-only MongoDB user, select incremental sync mode with a timestamp cursor for efficiency, and set a schedule. For CDC-based sync that captures deletes and updates, use the Incremental - Deduped + History mode with a replica set source.
