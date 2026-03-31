# How to Analyze Query Performance in MongoDB Compass

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Compass, Query Performance, Explain Plan, Indexing

Description: Learn how to use MongoDB Compass to analyze query performance with visual explain plans, index recommendations, and the query profiler.

---

## Why Use Compass for Query Performance

MongoDB Compass provides a visual interface for running `explain()` without writing shell commands. You get a graphical query plan tree, color-coded stage indicators, and key metrics like execution time and documents examined - all in one view.

## The Explain Plan Tab

### Running an Explain in Compass

1. Open a collection in Compass
2. Click the **Explain Plan** tab
3. Enter your filter, sort, project, skip, and limit
4. Click **Explain**

### Reading the Visual Plan

Compass displays the execution plan as a tree with stages color-coded:

```text
Green stage  - used an index (IXSCAN, FETCH with index)
Orange stage - in-memory sort (SORT)
Red stage    - full collection scan (COLLSCAN)
```

### Key Metrics Displayed

```text
Returned Documents  - number of documents returned to the application
Examined Documents  - number of documents loaded (should equal returned for good indexes)
Examined Keys       - number of index keys scanned
Execution Time      - total query time in milliseconds
Index Used          - which index was selected (if any)
```

### Identifying Performance Problems

```text
Red COLLSCAN box -> No index used, add one
Orange SORT box  -> In-memory sort, add compound index with sort fields
Examined >> Returned -> Index is not selective, refine or add compound index
```

## Example: Slow Query Investigation

Suppose this query is slow:

```javascript
// Filter: { status: "pending", region: "west" }
// Sort: { createdAt: -1 }
```

Compass explain shows:
```text
Stage: SORT (orange) - in-memory sort
  Stage: FETCH
    Stage: IXSCAN - using "status_1"
      Index Bounds: status: ["pending", "pending"]

Returned: 150
Examined: 150
Time: 145ms
```

The SORT stage means the index does not cover the sort. Fix: create a compound index:

```javascript
db.orders.createIndex({ status: 1, region: 1, createdAt: -1 });
```

After index creation, re-run the explain - the SORT stage should disappear.

## The Query Profiler

The Query Profiler shows actual slow query history from your server:

### Enabling the Profiler

```javascript
// In the shell - enable profiling for queries taking > 100ms
db.setProfilingLevel(1, { slowms: 100 });
```

### Using Compass Query Profiler

1. Navigate to your database in Compass
2. Click the **Performance** tab
3. Click **Query Profiler**
4. Set the time range
5. Compass lists slow queries with duration, namespace, and operation type

### Reading Query Profiler Entries

```text
Operation    - find, update, aggregate, etc.
Namespace    - database.collection
Duration     - execution time in ms
Timestamp    - when the query ran
Keys Examined / Docs Examined / Docs Returned - efficiency metrics
```

Click any entry to see the full query, explain plan, and execution stats.

## Index Recommendations

Compass can suggest indexes based on query patterns:

1. Navigate to the **Indexes** tab of a collection
2. Click **Create Index** 
3. Compass shows existing indexes and their usage statistics
4. The **Performance Advisor** (Atlas-connected instances) suggests new indexes based on slow query logs

## Using the Query Bar Profiling Mode

In the **Documents** tab, Compass shows query time after each query run:

```text
Query result: 142 documents returned in 234ms
```

To improve, add the query to the **Explain Plan** tab for full plan analysis.

## Schema-Aware Performance Tips from Compass

After running Schema analysis, correlate findings with query performance:

```text
High-cardinality fields -> Good index candidates (userId, orderId)
Date fields used in filters -> Add date range index
Nested fields in frequent queries -> Use dot-notation index
Fields always queried together -> Create compound index
```

```javascript
// Based on Compass schema and slow query findings
db.events.createIndex({ "userId": 1, "timestamp": -1 });
db.events.createIndex({ "metadata.region": 1, "type": 1 });
```

## Real-Time Server Stats (Connected to Atlas)

When connected to a MongoDB Atlas cluster, Compass shows:

```text
Operations/sec  - current read and write operation rates
Network I/O     - bytes in/out per second
Memory usage    - resident and virtual memory
Connections     - current active connections
Opcounters      - inserts, queries, updates, deletes per second
```

These help correlate application-level performance issues with server load.

## Summary

MongoDB Compass provides visual query performance analysis through color-coded explain plan trees that clearly identify COLLSCAN (red), in-memory SORT (orange), and efficient IXSCAN (green) stages. Use the Query Profiler tab to review historical slow queries with full explain output, and the Schema tab to identify high-cardinality fields that would benefit from indexing. The visual interface makes it faster to diagnose index gaps than writing and interpreting raw explain() output in the shell.
