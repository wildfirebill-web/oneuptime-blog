# How ClickHouse Pipeline Execution Model Works

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Pipeline, Execution Model, Processor, Internal

Description: Learn how ClickHouse's push-based pipeline model connects processors with queues, enables parallel execution, and flows data as columnar blocks through a query plan.

---

## From Query Plan to Pipeline

ClickHouse compiles each query into a pipeline of processors connected by bounded queues. Unlike the traditional Volcano (pull) model where each operator calls `getNext()` on its child, ClickHouse uses a push model where processors push blocks of data into output ports as fast as they can.

Inspect a pipeline:

```sql
EXPLAIN PIPELINE
SELECT user_id, count()
FROM events
WHERE event_date = today()
GROUP BY user_id;
```

## Processors and Ports

Each processor has one or more input ports and output ports. Data flows as `Block` objects (columnar chunks of typically 8192 rows) through these ports. Processors include:

- `MergeTreeThread` - reads granules from disk
- `FilterTransform` - applies WHERE/PREWHERE conditions
- `AggregatingTransform` - builds hash tables for GROUP BY
- `MergingAggregatedTransform` - merges partial aggregations
- `LimitsCheckingTransform` - enforces `LIMIT` and quotas
- `OutputFormatProcessor` - serializes blocks to the wire format

## Parallel Reading

The `ReadFromMergeTree` step spawns multiple `MergeTreeThread` processors, each reading a separate set of granules in parallel:

```sql
SET max_threads = 8;
```

Each thread has its own reader and processes its assigned granules independently, then pushes blocks into a shared queue consumed by the aggregation stage.

## Thread Pool and Scheduling

ClickHouse uses an async pipeline scheduler. Processors are scheduled on a thread pool when their input port has data available. The scheduler avoids busy-waiting - a processor sleeps until its upstream processor pushes a block.

```sql
-- See thread pool usage
SELECT name, value
FROM system.metrics
WHERE name LIKE '%Thread%';
```

## Back-Pressure

Queues between processors are bounded. If a downstream processor is slow (e.g., writing to the network), the upstream processor blocks when the queue is full. This back-pressure mechanism prevents unbounded memory growth in streaming queries.

## Pipeline for Distributed Queries

For distributed tables, the pipeline has an additional `RemoteSource` processor that opens connections to remote shards and reads their results. Results from all shards are merged by a `UnionTransform` before aggregation.

```sql
EXPLAIN PIPELINE
SELECT count() FROM distributed_events;
```

## Debugging Pipeline Execution

The `ProfileEvents` for a query show time spent in each processor class:

```sql
SELECT
    ProfileEvents['MergeTreeDataSelectReadRows'] AS rows_read,
    ProfileEvents['NetworkReceiveBytes'] AS net_bytes
FROM system.query_log
WHERE query_id = 'your-query-id';
```

## Summary

ClickHouse's pipeline execution model connects processors - each performing a single operation like filtering, aggregating, or sorting - through bounded queues that carry columnar blocks. Parallel processors read granules independently, back-pressure prevents memory overuse, and the async scheduler efficiently maps processors to CPU threads.
