# How to Troubleshoot MongoDB Cursor Leaks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Cursor, Performance, Debugging, Memory

Description: Diagnose and fix MongoDB cursor leaks that cause memory exhaustion and the 'cursor id not found' error in long-running applications.

---

## What Is a Cursor Leak?

A cursor leak occurs when your application opens MongoDB cursors but never explicitly closes them. MongoDB holds server-side state for each open cursor, consuming memory on the `mongod` process. Over hours or days, leaked cursors accumulate, eventually causing out-of-memory conditions or the error:

```text
MongoServerError: cursor id not found
```

## Detect Open Cursors

Check how many cursors are currently open on the server:

```javascript
db.serverStatus().metrics.cursor
```

Sample output:

```json
{
  "timedOut": 42,
  "open": {
    "noTimeout": 3,
    "pinned": 0,
    "multiTarget": 0,
    "singleTarget": 17,
    "total": 20
  }
}
```

A growing `open.total` over time without a corresponding query spike indicates leaks.

## Find the Leaking Operations

Use `currentOp` to inspect long-running cursor operations:

```javascript
db.adminCommand({
  currentOp: true,
  op: "getmore",
  secs_running: { $gt: 60 },
});
```

This lists all `getMore` operations running longer than 60 seconds, which points to cursors being iterated slowly or held open.

## Common Causes and Fixes

### Cause 1 - Not Iterating to Exhaustion

If you break out of a cursor loop early, the cursor is never automatically closed:

```javascript
// Leaky - breaks early without closing
const cursor = db.orders.find({ status: "pending" });
for await (const doc of cursor) {
  if (someCondition) break; // cursor leaked!
}

// Fixed - always close in a finally block
const cursor = db.orders.find({ status: "pending" });
try {
  for await (const doc of cursor) {
    if (someCondition) break;
  }
} finally {
  await cursor.close();
}
```

### Cause 2 - Using toArray() vs Streaming

`toArray()` fetches all results and automatically closes the cursor. Use it when result sets are small:

```javascript
// Safe for small result sets
const docs = await db.collection("orders").find({ status: "pending" }).toArray();
```

For large datasets, stream with explicit close:

```javascript
const cursor = db.collection("orders").find({});
cursor.stream().on("data", processDoc).on("end", () => cursor.close());
```

### Cause 3 - No Timeout on Idle Cursors

By default, MongoDB times out idle cursors after 10 minutes. Confirm `cursorTimeoutMillis` is not set to 0:

```javascript
db.adminCommand({
  getParameter: 1,
  cursorTimeoutMillis: 1,
});
```

If it returns 0, cursors never time out. Reset it:

```javascript
db.adminCommand({ setParameter: 1, cursorTimeoutMillis: 600000 });
```

### Cause 4 - Cursor Created in Aggregation Without Batch Size

Aggregation pipelines that produce large outputs without a batch size can hold the cursor open:

```javascript
// Set a batch size to control memory and close cursors incrementally
const cursor = db.collection("events").aggregate([{ $group: { _id: "$type", count: { $sum: 1 } } }], { cursor: { batchSize: 100 } });
```

## Monitor Cursor Metrics Over Time

Track cursor metrics in your monitoring system with a periodic scraper:

```python
import pymongo
import time

client = pymongo.MongoClient("mongodb://localhost:27017")

while True:
    status = client.admin.command("serverStatus")
    cursor_total = status["metrics"]["cursor"]["open"]["total"]
    cursor_timed_out = status["metrics"]["cursor"]["timedOut"]
    print(f"Open cursors: {cursor_total}, Timed out: {cursor_timed_out}")
    time.sleep(30)
```

Alert when `open.total` exceeds a threshold relevant to your workload (e.g., 100 for a typical web app).

## Summary

MongoDB cursor leaks stem from not closing cursors after early loop exits, missing `finally` blocks, disabled cursor timeouts, or unbounded aggregation cursors. Use `db.serverStatus().metrics.cursor` to detect the problem, `currentOp` to identify the offending queries, and proper try-finally patterns in your application code to prevent leaks from accumulating.
