# How to Enable Query Tracing in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Query Tracing, Profiling, Performance, Debugging

Description: Learn how to enable query tracing in ClickHouse to collect detailed execution timings, thread activity, and performance profiles for slow queries.

---

## What Is Query Tracing?

Query tracing in ClickHouse captures detailed execution information: which threads ran, how much time was spent in each function, and where memory was allocated. It helps diagnose slow queries that look fine on paper but perform poorly in practice.

## Enabling the Query Profiler

ClickHouse uses a sampling profiler that collects stack traces at a configurable interval. Enable it per query:

```sql
SET query_profiler_real_time_period_ns = 10000000;   -- sample every 10ms (wall clock)
SET query_profiler_cpu_time_period_ns  = 10000000;   -- sample every 10ms (CPU time)
```

## Running a Query with Tracing

```sql
SET query_profiler_real_time_period_ns = 10000000;
SET query_profiler_cpu_time_period_ns  = 10000000;
SET log_queries = 1;

SELECT
    event_type,
    count()
FROM events
WHERE toDate(created_at) >= today() - 7
GROUP BY event_type
ORDER BY count() DESC;
```

## Viewing Trace Data in system.trace_log

After running the query, query the trace log:

```sql
SELECT
    trace_type,
    count() AS sample_count,
    arrayStringConcat(
        arrayMap(x -> demangle(addressToSymbol(x)), trace),
        '\n'
    ) AS stack_trace
FROM system.trace_log
WHERE query_id = 'your-query-id'
GROUP BY trace_type, trace
ORDER BY sample_count DESC
LIMIT 10;
```

Replace `'your-query-id'` with the ID from `system.query_log`.

## Finding Your Query ID

```sql
SELECT query_id, query_duration_ms, query
FROM system.query_log
WHERE event_date = today()
  AND query LIKE '%event_type%'
ORDER BY event_time DESC
LIMIT 5;
```

## Flame Graph Export

Export trace data for flame graph visualization:

```bash
clickhouse-client --query "
  SELECT
    arrayStringConcat(
      arrayMap(x -> demangle(addressToSymbol(x)), trace),
      ';'
    ) AS stack,
    count() AS samples
  FROM system.trace_log
  WHERE query_id = 'your-query-id'
  GROUP BY trace
  FORMAT TabSeparated
" > profile.txt

# Generate flame graph using Brendan Gregg's flamegraph.pl
./flamegraph.pl profile.txt > flame.svg
```

## Enabling OpenTelemetry Tracing

ClickHouse also supports OpenTelemetry for distributed tracing:

```sql
SET opentelemetry_start_trace_probability = 1.0;
SET opentelemetry_trace_processors = 1;
```

Traces are written to `system.opentelemetry_span_log` and can be exported to Jaeger or Tempo.

## Text Log for Per-Step Timings

For a simpler view of query execution steps:

```sql
SET send_logs_level = 'trace';
```

With this setting, ClickHouse returns log messages as part of the query response showing each execution step with timing.

## Summary

Enable ClickHouse query tracing using `query_profiler_real_time_period_ns` and `query_profiler_cpu_time_period_ns`. After running the query, inspect `system.trace_log` for sampled stack traces. Export to a flame graph format to visually identify hot code paths. For distributed workloads, use OpenTelemetry tracing to trace queries across multiple nodes.
