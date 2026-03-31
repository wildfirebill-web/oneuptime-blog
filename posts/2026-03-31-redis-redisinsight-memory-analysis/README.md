# How to Use RedisInsight Memory Analysis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RedisInsight, Memory Analysis, Performance, Optimization

Description: Learn how to use RedisInsight Memory Analysis to identify memory hotspots, analyze key distributions, and optimize Redis memory usage in production.

---

## What Is RedisInsight Memory Analysis

RedisInsight Memory Analysis is a built-in tool that scans your Redis keyspace and generates a detailed report on memory consumption. It helps you identify which keys, data types, or namespaces are consuming the most memory, so you can take targeted action to reduce footprint.

Memory analysis is especially useful when Redis is approaching its `maxmemory` limit or when you want to proactively audit memory before scaling up.

## Running a Memory Analysis Report

To run an analysis, open RedisInsight, connect to your Redis instance, and navigate to the **Memory Analysis** section in the left sidebar.

Click **Analyze** to trigger a full keyspace scan. RedisInsight uses `SCAN` internally, so it is non-blocking and safe to run in production.

```text
Analysis Status: Running...
Scanned: 1,250,000 / 1,250,000 keys
Elapsed: 14.3s
```

After the scan completes, the report is saved and you can revisit it without re-scanning.

## Understanding the Summary Dashboard

The summary view shows:

- **Total keys** scanned
- **Total memory** used (estimated)
- **Top key types** by memory (Strings, Hashes, Lists, Sets, Sorted Sets)
- **Top namespaces** by memory (based on the `:` delimiter pattern)

Example summary output:

```text
Total Keys: 1,250,000
Estimated Memory: 4.2 GB

Top Types:
  Hash    - 2.1 GB (50%)
  String  - 1.4 GB (33%)
  List    - 0.5 GB (12%)
  ZSet    - 0.2 GB (5%)

Top Namespaces:
  session:   - 1.8 GB
  cache:     - 1.1 GB
  user:      - 0.9 GB
```

## Identifying Large Keys

The **Top Keys** tab lists individual keys with the highest memory usage. For each key you can see:

- Key name
- Data type
- Memory in bytes
- TTL (if set)
- Number of elements (for collections)

```text
Key                          Type   Memory    TTL     Elements
-----------------------------+-------+---------+-------+--------
session:user:bulk_export     Hash   12.4 MB   300s    84,200
cache:report:2025-Q4         String 8.1 MB    -       -
queue:jobs:pending           List   5.3 MB    -       210,500
```

Keys with no TTL and high memory are candidates for cleanup or expiry enforcement.

## Analyzing Namespace Distribution

RedisInsight extracts namespaces from key names by splitting on `:`. The **Namespaces** tab shows a treemap visualization of memory by prefix.

To customize the delimiter used for namespace splitting, update the delimiter setting:

```text
Settings > Memory Analysis > Key Separator = ":"
```

This helps when using a different separator like `|` or `.` in your key naming convention.

## Detecting Suboptimal Encodings

Redis uses compact encodings (listpack, ziplist, intset) for small collections, and switches to full encodings when thresholds are exceeded. RedisInsight flags keys that have been promoted to full encodings where compact encodings were expected.

Check the **Encoding Recommendations** section for entries like:

```text
Key: cart:user:9821
Type: Hash
Current Encoding: hashtable
Recommended: listpack
Elements: 520 (threshold: 128)
Memory Wasted: ~18KB per key
```

To keep more hashes in listpack encoding, adjust your config:

```bash
redis-cli CONFIG SET hash-max-listpack-entries 256
redis-cli CONFIG SET hash-max-listpack-value 128
```

## Filtering and Searching Keys

Use the filter bar in the Memory Analysis view to narrow results:

```text
Filter by type: Hash
Filter by namespace: session:
Filter by memory: > 1MB
```

You can also search by key pattern:

```text
Pattern: cache:product:*
```

This is useful when investigating a specific service's memory footprint.

## Scheduling Recurring Analysis

RedisInsight allows you to schedule automatic memory snapshots at a defined interval. Go to **Settings > Memory Analysis** and enable the schedule:

```text
Schedule: Every 24 hours
Retention: Keep last 7 snapshots
```

You can then track memory growth over time and correlate spikes with deployment events or traffic increases.

## Exporting Memory Analysis Reports

To share findings, export the report as a CSV:

```text
Memory Analysis > Export > Download CSV
```

The CSV includes key name, type, memory, TTL, and element count. You can import this into a spreadsheet or data pipeline for further analysis.

## Acting on Analysis Results

Common actions after a memory analysis:

**1. Delete stale keys with no TTL:**

```bash
redis-cli --scan --pattern "cache:temp:*" | xargs redis-cli DEL
```

**2. Set expiry on keys missing TTL:**

```bash
redis-cli EXPIRE session:user:12345 3600
```

**3. Reduce collection sizes by trimming lists:**

```bash
redis-cli LTRIM queue:jobs:archive 0 9999
```

**4. Compress large string values at the application level before storing.**

## Summary

RedisInsight Memory Analysis gives you a comprehensive view of how memory is distributed across key types, namespaces, and individual keys. By identifying large keys, suboptimal encodings, and keys without TTL, you can take targeted steps to reduce Redis memory consumption. Running analysis on a schedule and tracking trends over time is a best practice for any production Redis deployment.
