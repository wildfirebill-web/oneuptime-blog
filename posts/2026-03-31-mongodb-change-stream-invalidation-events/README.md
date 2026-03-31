# How to Handle Change Stream Invalidation Events in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Change Stream, Invalidation, Error Handling, Resilience

Description: Learn what causes MongoDB Change Stream invalidation events, how to detect them, and how to build self-healing consumers that reopen streams automatically.

---

A Change Stream invalidation event signals that the stream can no longer continue. After an invalidation event is received, the cursor is closed and any further call to `next()` raises an error. Understanding the causes and writing code to handle them gracefully keeps your event consumers running reliably.

## What Triggers an Invalidation Event?

| Operation | Scope |
|-----------|-------|
| `drop` collection | Collection or database streams |
| `rename` collection | Collection stream |
| `dropDatabase` | Database or deployment streams |
| `invalidate` (synthetic) | Emitted after the above events |

The event document looks like this:

```json
{
  "_id": { "_data": "..." },
  "operationType": "invalidate"
}
```

After this event, the stream cursor is exhausted. No more events will be emitted.

## Detecting Invalidation in Node.js

```javascript
const { MongoClient } = require("mongodb");

async function watchWithRecovery(client, dbName, collName) {
  const collection = client.db(dbName).collection(collName);
  let resumeToken = await loadSavedToken();

  while (true) {
    const options = resumeToken ? { startAfter: resumeToken } : {};
    const stream = collection.watch([], options);

    try {
      for await (const event of stream) {
        if (event.operationType === "invalidate") {
          console.warn("Stream invalidated. Waiting for collection to reappear...");
          resumeToken = null; // token is no longer usable after invalidation
          break; // exit the for-await loop to reopen the stream
        }

        await processEvent(event);
        resumeToken = event._id;
        await saveToken(resumeToken);
      }
    } catch (err) {
      if (err.code === 40585 || err.message.includes("closed")) {
        console.error("Stream closed unexpectedly:", err.message);
      } else {
        throw err;
      }
    }

    // Wait before reopening to avoid rapid reconnection loops
    await new Promise((r) => setTimeout(r, 2000));
  }
}
```

## Detecting Invalidation in Python

```python
import time
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017/?replicaSet=rs0")
collection = client["shop"]["orders"]

resume_token = load_saved_token()

while True:
    options = {"start_after": resume_token} if resume_token else {}
    with collection.watch(**options) as stream:
        for event in stream:
            if event["operationType"] == "invalidate":
                print("Stream invalidated - collection may have been dropped")
                resume_token = None
                break

            process_event(event)
            resume_token = event["_id"]
            save_token(resume_token)

    time.sleep(2)  # Back off before reopening
```

## Using startAfter to Resume After Invalidation

`resumeAfter` raises an error when given a token from an invalidation event. Use `startAfter` instead, which handles invalidation tokens safely:

```javascript
// Safe: startAfter works even if the last token was an invalidate event
const stream = collection.watch([], { startAfter: lastToken });
```

## Distinguishing Transient Errors from Invalidation

```javascript
stream.on("error", (err) => {
  const INVALIDATION_CODE = 40585;       // CursorKilled
  const NETWORK_ERROR_CODES = [6, 89, 91, 10107];

  if (err.code === INVALIDATION_CODE) {
    handleInvalidation();
  } else if (NETWORK_ERROR_CODES.includes(err.code)) {
    handleTransientError(err); // retry with backoff
  } else {
    throw err; // unknown error - crash loudly
  }
});
```

## Handling Collection Recreation

If your application drops and recreates collections as part of a schema migration, your consumer will receive an invalidation event. The recommended pattern is:

1. Receive invalidation event.
2. Poll until the collection exists again (or wait for an external signal).
3. Open a fresh Change Stream without a resume token.

```javascript
async function waitForCollection(client, dbName, collName, maxWaitMs = 30000) {
  const deadline = Date.now() + maxWaitMs;
  while (Date.now() < deadline) {
    const collections = await client.db(dbName).listCollections({ name: collName }).toArray();
    if (collections.length > 0) return;
    await new Promise((r) => setTimeout(r, 1000));
  }
  throw new Error(`Collection ${dbName}.${collName} did not reappear within ${maxWaitMs}ms`);
}
```

## Summary

Change Stream invalidation events close the cursor and require the consumer to reopen the stream. Use `startAfter` instead of `resumeAfter` when the last saved token may be from an invalidation event, implement retry loops with backoff, and distinguish transient network errors from permanent invalidation to build resilient event consumers.
