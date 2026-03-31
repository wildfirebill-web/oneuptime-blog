# How to Use MongoDB Compass for Performance Profiling

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Compass, Performance, Profiling

Description: Learn how to use MongoDB Compass to profile query performance, analyze explain plans, identify slow operations, and use the Performance tab for real-time server monitoring.

---

## Overview

MongoDB Compass includes several performance analysis tools that make it easy to understand why queries are slow, whether indexes are being used, and what the server is currently doing. This guide covers using Compass for explain plan analysis, the Performance tab for real-time monitoring, and the built-in query profiler.

## Analyzing Query Performance with Explain Plans

Every query in Compass can be analyzed using the Explain Plan view:

1. Open a collection in Compass
2. Enter your query in the Filter bar
3. Click the "Explain" button (or the lightning bolt icon)
4. Compass displays the execution plan in two formats:
   - **Visual Tree** - shows the plan as a graph with stage costs
   - **Raw Output** - shows the full explain document

### Reading the Visual Explain Plan

The Visual Tree shows each stage as a node:

```text
[PROJECTION_COVERED]
    |
[IXSCAN]
  Index: status_1_total_-1
  Keys Examined: 45
  Docs Examined: 0   <- 0 means index-covered (no document reads)
  Returns: 45
  Time: 1ms
```

```text
[SORT]
    |
[COLLSCAN]
  Docs Examined: 50000
  Returns: 23
  Time: 145ms   <- Much slower without an index
```

### Key Metrics to Check

- **Docs Examined** vs **Docs Returned** - a large ratio indicates a slow query; ideal is 1:1
- **COLLSCAN** - indicates no index is being used
- **IXSCAN** - index scan; good
- **FETCH** - additional document reads needed after index scan

## The Performance Tab

The Performance tab provides a real-time dashboard showing:

1. Open the Performance tab from the left sidebar
2. Key charts displayed:
   - **Operations** - ops/sec for read, write, command, getmore
   - **Read/Write** - active and queued reads and writes
   - **Network** - bytes in/out
   - **Memory** - resident and virtual memory
   - **Hottest Collections** - which collections are most active
   - **Slowest Operations** - operations taking the most time

### Using Real-Time Charts

The charts update every second. Watch for:
- Spike in queued reads/writes - indicates lock contention
- Steady climb in memory - potential memory leak
- Hottest collections - identify the busiest collections instantly

## The Current Operations Panel

In the Performance tab, scroll down to see "Current Operations":

- Shows all active operations with their duration
- Click any operation to see full details
- Click the X button to kill a specific operation directly from Compass

## Setting Up the Profiler

Enable the database profiler to capture slow operations:

1. Navigate to the database in Compass
2. Click the "Performance" tab
3. Click "Enable Profiler" in the slow query panel
4. Set profiling level:
   - Level 0: Off
   - Level 1: Slow ops only (above slowMs threshold)
   - Level 2: All operations

Or set profiling via the shell:

```javascript
// Profile operations slower than 100ms
db.setProfilingLevel(1, { slowms: 100 })

// Profile all operations
db.setProfilingLevel(2)

// Check current profiling settings
db.getProfilingStatus()
```

## Viewing Profiler Output in Compass

1. Go to the collection `system.profile` in the database
2. Query for slow operations:

```javascript
// Find the slowest queries
{ "op": "query", "millis": { "$gte": 100 } }

// Sort by execution time
// Sort bar: { "millis": -1 }

// Find collection scans specifically
{ "planSummary": "COLLSCAN" }
```

## Analyzing a Slow Query Workflow

Here is a complete workflow for investigating a slow query in Compass:

**Step 1 - Identify the slow query**

In the Performance tab, look at "Slowest Operations." Note the namespace and command.

**Step 2 - Reproduce in the Documents tab**

Navigate to the collection and enter the filter from the slow query:

```javascript
{ "status": "pending", "region": "US-WEST" }
```

**Step 3 - Run Explain**

Click Explain and check the execution plan.

**Step 4 - Identify the problem**

```text
Issue: COLLSCAN - no index on status + region
Docs Examined: 50,000
Docs Returned: 234
Ratio: 213:1 (very bad)
```

**Step 5 - Create an index**

1. Go to the Indexes tab
2. Click "Create Index"
3. Add fields: `status` (asc), `region` (asc)
4. Click "Create Index"

**Step 6 - Re-run Explain**

```text
After index: IXSCAN on status_1_region_1
Docs Examined: 234
Docs Returned: 234
Ratio: 1:1 (optimal)
Time: 2ms (down from 145ms)
```

## Comparing Index Usage

The Indexes tab shows usage statistics for each index:

- **Usage** - how many operations used this index
- **Size** - index size on disk

Indexes with 0 usage are unused and can be candidates for removal.

## Summary

MongoDB Compass provides multiple performance profiling tools in a single visual interface. The Explain Plan view reveals whether queries use indexes or resort to collection scans, with the Visual Tree making it easy to spot bottlenecks at a glance. The Performance tab shows real-time server metrics and highlights the hottest collections and slowest operations. Combining explain analysis with index creation and the database profiler gives you a complete workflow for diagnosing and fixing query performance problems.
